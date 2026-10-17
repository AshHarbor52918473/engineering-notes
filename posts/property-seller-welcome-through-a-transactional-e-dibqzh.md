# Property Seller Welcome Through a Transactional Email API (3 Custom-Domain Gates)

The least complex reliable design is a three-stage pipeline: commit the seller's order, enqueue one notification intent, then let a worker render and submit the email through a custom-domain sending identity. Do not make the order request wait for an email provider. The bill is made of accepted submissions, retry attempts, event ingestion, log volume, and retained message data; in a retry-heavy incident, attempts are the term that can grow while order volume stays flat.

Acceptance is not delivery.

**TL;DR:** Authenticate the visible sending domain with SPF, DKIM, and DMARC; use an idempotent outbox; retry only transient submission failures; and retain delivery metadata longer than rendered bodies. A provider's accepted response means the next system took custody. It does not prove that the seller received, opened, or even had the message placed in an inbox.

For a concrete planning model, suppose a property marketplace creates 100,000 seller orders in a month. One initial submission plus a 2% retry rate means 102,000 billable submission attempts, before later delivery events are counted. If each rendered message averages 25 KB and bodies are retained for 30 days, that cohort occupies about 2.5 GB at steady state; three stored copies across the job payload, event archive, and application log make it about 7.5 GB. Those are example inputs, not a benchmark or price claim. Put your measured values into the same equation before changing architecture.

## How should a Node.js API send custom transactional email?

SPF, DKIM, and DMARC answer related but different questions. SPF lets a receiving system evaluate whether the connecting sender is authorized for the envelope domain. DKIM adds a cryptographic signature tied to a signing domain. DMARC then evaluates alignment between the domain visible to the recipient in the `From` field and a domain authenticated by SPF or DKIM, and it defines reporting and requested handling policy. RFC 7489 is the normative source for that policy model.

That distinction matters for seller notifications. A message can have a passing DKIM signature while still failing DMARC alignment if the signing domain is unrelated to the visible `From` domain. Likewise, SPF is sensitive to the envelope path rather than the friendly address a seller sees. Configure the exact records supplied by the sending system, verify them through DNS, and treat verification as deployment state. Do not paste guessed record values into infrastructure code.

The runtime does not change those mail rules. A Node.js order service can write the outbox record and call the same narrow gateway contract demonstrated below; the example is Python so the delivery boundary stays visible instead of turning this note into framework setup. The API call needs a sender, recipient, subject, rendered alternatives, and stable idempotency identity. DNS authentication belongs to deployment configuration, not to a per-message option.

Start DMARC policy changes with evidence from aggregate reports. A strict policy applied before every legitimate sender is identified can reject valid mail. The operational sequence is inventory, authenticate, observe, then tighten. Keep marketing and order traffic on distinct subdomains when their operators and risk differ; this narrows the effect of a bad campaign or configuration mistake without pretending that a subdomain repairs poor sending practices.

## Make submission idempotent, not synchronous

The order transaction should create both the order and an outbox row in one database commit. A worker claims the row, renders a versioned template, and submits it with a stable idempotency key such as `seller-order:{order_id}:v1`. The exact transport contract varies, so put it behind a small adapter that returns a normalized submission identifier and error class.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class SellerOrderNotice:
    order_id: str
    seller_email: str
    property_name: str
    template_version: str = "seller-order-v1"


class MailGateway(Protocol):
    def submit(self, *, sender: str, recipient: str, subject: str,
               html: str, idempotency_key: str) -> str: ...


def send_notice(notice: SellerOrderNotice, gateway: MailGateway) -> str:
    # Values are escaped by the template engine before this function is called.
    html = render_template(
        notice.template_version,
        {"order_id": notice.order_id, "property_name": notice.property_name},
    )
    return gateway.submit(
        sender="orders@notify.example-property.test",
        recipient=notice.seller_email,
        subject=f"New order {notice.order_id}",
        html=html,
        idempotency_key=f"seller-order:{notice.order_id}:v1",
    )
```

The domain above is example data, not a DNS prescription. In production, the sender must be a domain the organization controls and has authenticated. The adapter also needs a plain-text alternative even though the focused example shows only the HTML render path.

This adapter has limits.

It is appropriate for a team that owns an order database, a worker, and delivery-event processing. It is a poor fit when the application cannot run background jobs, when every message must be composed by a nontechnical operator in a visual workflow, or when local policy forbids the external processing needed by the chosen transport. A direct SMTP submission can reduce adapter code for a tiny system, but it still needs durable retry state and authentication; a managed email API can expose clearer request and event identifiers, but introduces a remote dependency and contract-specific event mapping. Neither choice removes the outbox problem.

Retry policy is where duplicate notifications are born. Retry timeouts, connection failures, and explicitly transient responses with exponential backoff and jitter. Do not retry permanent address or content rejections as though time will fix them. When a timeout leaves submission outcome unknown, query by the stable key if the gateway supports that operation; otherwise record the ambiguity and reconcile events before resubmitting.

Blind retry is convenient. It is also how one order becomes two alerts.

Retries are a trade-off, not a repair strategy.

## Delivery state is a ledger

Store state transitions as append-only events keyed by your notification ID and the remote submission ID. `queued`, `submitted`, `delivered`, `deferred`, `bounced`, and `complained` are useful internal concepts, but map them from the actual event semantics of the transport. A webhook arriving twice must have the same effect as one arrival. A late event must not move a terminal failure backward to a hopeful state merely because its timestamp is newer.

Track at least three ratios separately: outbox-to-submission success, submission-to-delivery success, and time from order commit to delivery event. A single "sent" counter collapses application health, transport acceptance, and mailbox delivery into a misleading green light. Alert on queue age and retry amplification as well as failure counts. A low raw error count can still hide one seller whose notification has been stuck for hours.

One seller still matters.

Template deployment deserves the same discipline. Render fixtures containing long property names, missing optional fields, HTML-sensitive characters, and Unicode addresses before release. Snapshot the subject, text, and HTML outputs by template version. Then run a small authenticated-domain canary through real receiving mailboxes you control and inspect authentication results in the received headers. Unit tests can prove escaping and required fields; they cannot prove DNS publication or receiver evaluation.

Keep SMS out of the automatic fallback path unless the seller has provided a valid number and the organization has a lawful, documented basis for that use. Email failure does not create consent for another channel. If an OTP is ever added to a web flow, MDN documents WebOTP as an experimental API with limited browser availability and a specific SMS formatting convention; it should be progressive enhancement, never the only way to complete authentication.

## Retain evidence, discard content

Retention should follow the question each datum answers. Keep the notification ID, order ID, template version, recipient hash or protected lookup key, submission ID, timestamps, attempt count, normalized result, and authentication configuration version long enough to investigate delivery and satisfy applicable recordkeeping rules. Access must be restricted because even metadata can identify business activity.

Rendered bodies are different. They duplicate data already held by the order system and may contain a seller's address, property details, or other personal information. Prefer deterministic re-rendering from a versioned template and an audited order snapshot over indefinite body storage. Redact bodies and authorization headers from application logs; cap payload capture in error tooling; expire raw webhook payloads after their normalized fields have been validated and recorded. Exact retention periods are a policy decision for legal, security, and operations, not a universal engineering constant.

This is the deliberate loss: after the short diagnostic window closes, you cannot reproduce the exact MIME bytes or inspect an obsolete raw event payload. That makes a rare forensic investigation slower. In exchange, routine logs hold less sensitive content, storage grows more slowly, and deletion requests have fewer copies to chase. Preserve hashes, template versions, transition history, and authentication evidence so the system can still answer the usual questions: what was intended, what was submitted, and what the receiving system reported.

## A release gate for reliable alerts

Before enabling seller traffic, verify SPF and DKIM results on received test messages and confirm DMARC alignment for the visible sending domain. Exercise duplicate outbox delivery, worker crashes after remote acceptance, transient throttling, permanent rejection, delayed webhooks, duplicate events, and events received out of order. Measure queue age under a controlled backlog. Confirm that a template rollback does not alter the idempotency identity of an already submitted notice.

Then review the cost equation with production measurements. Separate unique notifications from attempts, event count, bytes retained, and days retained. The useful optimization is usually the one that reduces retry amplification or unnecessary body copies without deleting the evidence needed to explain a missed order alert. Reliability comes from explicit state and bounded ambiguity, not from calling an API inside the request handler.

## Further reading

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- MDN, WebOTP API: https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
