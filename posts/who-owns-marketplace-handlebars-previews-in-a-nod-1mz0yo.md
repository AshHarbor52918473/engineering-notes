# Who Owns Marketplace Handlebars Previews in a Node.js Transactional Email API?

Short answer: keep marketplace contact-routing rules in the Node.js application, keep approved wording in versioned Handlebars templates, and require a preview of typed variables before the transactional confirmation email can be released.

That boundary lets a support team revise queue-specific guidance without redeploying routing code, while engineering retains control of recipient selection, privacy, retry behavior, and the data contract. It also prevents a copy edit from quietly changing which queue receives a message. The template owns presentation. The application owns decisions.

## How does privacy governance divide routing and template ownership?

A marketplace contact form usually serves several parties at once: buyers asking about an order, sellers disputing a payout, and visitors reporting trust or safety concerns. A single `topic` field may look sufficient until a submission combines two concerns, an account role is stale, or free text appears to contradict the selected category. Routing therefore belongs in a policy function with tests and an explicit fallback queue. Handlebars should never infer the destination from prose or decide which internal address receives a copy.

Template ownership is narrower. An authorized support or content owner may own the subject, greeting, expected response window, and queue-specific next steps. Engineering should own the allowed variable names, their types, escaping rules, and which template revision is active for each route. A revision is deployable only when it consumes the same approved contract or arrives with an application change that introduces a new one. Keep the public confirmation separate from the internal support notification: the confirmation can acknowledge receipt and expose a case reference, while the internal message can contain operational context appropriate for the assigned queue. Reusing one template for both audiences is tempting, but it increases the chance that internal notes or sensitive marketplace data reach the submitter.

Don't do it.

This is also where compliance enters the design. Collect only fields needed to route and resolve the request, redact raw message bodies from routine logs, and use a stable case identifier for support lookups. Sender authentication is a deployment concern rather than a template feature: DMARC evaluates alignment and policy around authenticated mail, so a beautiful preview cannot establish that the production sending domain is ready.

## How can a Node.js API create a Handlebars preview with email variables?

Treat preview as a pure operation over a template revision and a typed data object. The Node.js endpoint can call a renderer behind an internal interface, but it should validate the request before rendering and it should never send from the preview path. That separation makes the authorization story legible: a reviewer may preview draft copy without receiving permission to dispatch mail.

The variable set for this scenario can stay small: `case_reference`, `requester_name`, `marketplace_role`, `queue_label`, and `support_path`. Values such as the internal queue address, risk score, or routing explanation don't belong in the customer-facing contract. Normal Handlebars interpolation escapes HTML; triple-brace output bypasses that protection, so customer-controlled text should remain in an escaped interpolation context.

Use fixtures that make rendering uncomfortable. Include a long display name, an ampersand, an apostrophe, a missing optional name, and the longest queue label the interface permits. Include a support URL that already has a query string. One pleasant record proves almost nothing — the edge record is the preview that earns its keep.

Preview is evidence.

The following Python model is deliberately offline. It documents the boundary that the Node.js service should enforce without inventing a commercial email API or a network route.

```python
from dataclasses import dataclass
from html import escape
from urllib.parse import urlparse


ALLOWED_ROLES = {"buyer", "seller", "visitor"}
ALLOWED_QUEUES = {"orders", "payouts", "trust-and-safety", "general"}


@dataclass(frozen=True)
class ConfirmationData:
    case_reference: str
    requester_name: str | None
    marketplace_role: str
    queue_label: str
    support_path: str

    def validate(self) -> None:
        if not self.case_reference.strip():
            raise ValueError("case_reference is required")
        if self.marketplace_role not in ALLOWED_ROLES:
            raise ValueError("marketplace_role is not recognized")
        if self.queue_label not in ALLOWED_QUEUES:
            raise ValueError("queue_label is not recognized")

        parsed = urlparse(self.support_path)
        if parsed.scheme != "https" or not parsed.netloc:
            raise ValueError("support_path must be an absolute HTTPS URL")


def preview_context(data: ConfirmationData) -> dict[str, str]:
    data.validate()
    return {
        "case_reference": escape(data.case_reference),
        "requester_name": escape(data.requester_name or "there"),
        "marketplace_role": escape(data.marketplace_role),
        "queue_label": escape(data.queue_label),
        "support_path": escape(data.support_path, quote=True),
    }


edge_fixture = ConfirmationData(
    case_reference="MKT-80421",
    requester_name="D'Arcy & Partners",
    marketplace_role="seller",
    queue_label="trust-and-safety",
    support_path="https://support.example.test/cases/MKT-80421?source=confirmation",
)
preview_context(edge_fixture)
```

A malformed preview request should fail locally with a client-side validation result such as `400`, before any recipient or delivery system is involved. More important, the preview artifact should record the template revision and a fixture hash. Approval of “the latest template” is too vague because latest can change between review and dispatch.

## Where do retries and delivery failures belong?

Follow one seller dispute through the boundary. The request handler validates the form, classifies it through deterministic policy, creates case `MKT-80421`, and stores both `trust-and-safety` and the policy version that produced that route. In the same database transaction it adds an outbox record for the public confirmation, including the approved template revision and a stable application event ID. Only after that transaction commits may a worker render and attempt delivery. This order matters: sending inline before commit can give the seller a reference that support cannot find, while committing the case and then losing an in-process send can leave no durable intent to retry. Retrying the entire form request is worse because it may duplicate both case and mail. The outbox turns those ambiguous outcomes into inspectable states, lets support trace one reference across routing and delivery, and prevents a later policy edit from silently rerouting an old dispute.

The routing fallback deserves special attention. Unknown categories should go to a staffed general queue, not disappear and not guess at a sensitive destination from free text. Record why the fallback happened using a low-cardinality reason code such as `unknown_topic` or `role_conflict`; avoid copying the submitter's message into telemetry. If a trust-and-safety report needs restricted access, enforce that in the case system and internal notification path, outside the public template.

Retries must preserve the application event ID. Back off on rate limiting, honor an explicit retry delay when one is supplied, and stop automated retries for permanent address or policy failures. The exact retry ceiling depends on the delivery service and the support promise; I'm not sure a universal number would be defensible. What matters is that exhaustion produces an observable state and does not rerun route selection against newer policy.

No template should contain an authentication secret. If the contact flow later grows account recovery or one-time-code behavior, give that flow separate expiry, replay, abuse, and recovery controls. NIST's authenticator guidance is a useful baseline for that security boundary; a support confirmation template is not an authenticator merely because it can interpolate a code.

## How can teams test each template ownership model?

Once the contract and outbox exist, the ownership choice becomes concrete. There is no universally best model.

| Ownership model | Useful when | Main trade-off |
|---|---|---|
| Templates in the application repository | Engineering controls every release and copy changes are infrequent | Wording changes require the code review and deployment path |
| Templates in a separate versioned repository | Content and engineering need distinct review rules with auditable revisions | The release process must bind an application contract to a template revision |
| Templates in a managed editor | Authorized operators need frequent preview and approval without application deployment | Access control, revision export, rollback, and environment promotion need verification |

Repository ownership is the conservative choice for a small team with rare changes. The catch is queue guidance may change faster than application code, especially during marketplace policy updates. A separate versioned repository fits when review independence matters, but it is not suitable if the organization cannot operate a reliable promotion pipeline between staging and production. A managed editor can reduce copy-release friction; stick with repository-controlled templates when procurement, audit export, or fine-grained authorization requirements cannot be met.

The comparison should use evidence, not screenshots. Give each model the same fixtures and ask whether it can pin a revision, preview without sending, require approval, roll back, restrict sender changes, and expose an audit trail. Then test the application boundary: removing `queue_label` must block release, while changing punctuation must not require a routing deployment.

Observability follows the same ownership split. The application should expose counts for cases created, fallback routes, outbox age, delivery attempts, permanent failures, and duplicate events suppressed. Template metrics should identify revisions, not arbitrary copy fragments. Alert on a growing oldest-outbox age and on a sudden change in fallback rate; neither signal requires storing the contact message itself.

## How can the team migrate without losing contact mail?

Start by inventorying current templates and variables, then assign an owner to routing policy, data contracts, copy approval, domain authentication, and delivery operations. Freeze variable renames during migration. Import one approved revision, render the edge fixtures, and send only to controlled seed inboxes while checking both content and authenticated-domain behavior.

Next, shadow the new router without delivering from it. Compare its queue decision with the existing path, investigate disagreements, and version the final policy. Move a small cohort to the outbox worker, watch queue fallback and outbox age, then widen traffic. Rollback should select the prior approved template revision and worker release; it should not recreate cases or mint new event identifiers.

Keep the final gate short: a pinned revision, a validated variable contract, an approved preview, authenticated sending, a durable outbox, and a traceable case reference.

Ship only when all six agree.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B: Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
