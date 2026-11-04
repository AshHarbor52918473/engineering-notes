# Marketplace Domain Ownership: Separating TXT Verification from Email Confirmation

Assign marketplace tenants subdomains in a platform-owned zone through the platform's tenant registry. Require a scoped DNS TXT challenge before activating a hostname in a customer-owned zone. Email confirmation belongs in the contact-verification flow; it should not independently authorize a hostname claim.

TL;DR: Separate three decisions: who can receive an onboarding message, who can publish the requested DNS proof, and which tenant may serve the hostname. This record chooses TXT verification for customer-controlled names and internal allocation for platform-controlled names. Activation remains a separate, transactional step.

The useful precedent is narrower than a general ownership certificate. RFC 7489 section 7.1 uses a DNS TXT check to authorize sending aggregate reports to an external destination. That mechanism supports a specific authorization decision. A marketplace should give its own challenge equally explicit scope, rather than interpreting any successful email exchange as authority over a customer's domain.

## Can email confirmation prove domain ownership without TXT verification?

For this decision, the protected operation is attaching a requested hostname to a marketplace tenant. Imagine two illustrative requests: tenant `seller-42` receives `seller-42.marketplace.example`, while tenant `seller-73` asks to attach `shop.seller.example`. These are design examples, not customer observations. The first name is allocated inside the platform's zone; the second crosses an administrative boundary and needs external proof.

Define the boundary.

A TXT response should authorize only the claim identified by the challenge: one tenant, one canonical hostname, one purpose, and one validity window. It does not establish legal ownership, authority over every sibling hostname, or permission to send mail with that domain in the From header. Treat publication as evidence of the ability to arrange that particular DNS change. Delegated DNS access is sufficient to produce it; a claim about the registrant's identity would require different evidence.

Mailbox confirmation answers a different question. A completed confirmation shows that someone could complete the mailbox workflow at that time. It does not, by itself, demonstrate that the same person can publish a record for `shop.seller.example`. This distinction matters during automated onboarding, where a useful contact address can otherwise become an accidental authorization credential. Keep the contact relationship, but don't give it that extra power.

DMARC is another separate boundary. RFC 7489 sections 3.1 and 6 describe identifier alignment and domain policy; section 6.6.3 describes policy discovery. A marketplace hostname challenge is an application-specific authorization check, not a substitute for those checks. Before enabling tenant email, require the relevant mail configuration and authorization through a separate process. Passing the web-hostname challenge should never silently enable a tenant to use a new sender domain.

## Decision and invariants for the two zone owners

Choose the proof according to who administers the zone, then keep allocation and activation consistent. For platform-owned names, the authoritative application record is the tenant allocation. Asking each seller to publish a TXT record in a zone the platform operates adds a ceremony without introducing independent customer evidence. Customer-owned names need a different path because the platform cannot derive permission from its own registry.

| Candidate | What this design accepts as evidence | Activation boundary | Operational trade-off |
| --- | --- | --- | --- |
| Internal allocation in a platform-owned zone | A permitted tenant action and an unclaimed platform hostname | Commit the hostname allocation to that tenant | Easy to automate; the platform must enforce its namespace and reservation rules |
| TXT challenge in a customer-owned zone | The current token at the exact challenge name | Consume the matching live claim and acquire the hostname | Requires coordination with the customer's DNS administrator |
| Email confirmation for a customer-owned zone | Completion of a contact-verification workflow | Contact status only | Useful for communication; insufficient evidence for the hostname operation |

The invariants are application requirements. A canonical hostname may have at most one active tenant binding. A claim cannot be transferred between tenants by forwarding its token. An expired or superseded challenge cannot activate a new binding. DNS verification and hostname routing must refer to the same canonical name; otherwise a successful check can authorize a different resource from the one actually attached. Canonicalization deserves its own boundary in the implementation: validate and normalize the requested hostname before constructing the challenge, then store that value. Use that stored value throughout verification and activation. Also distinguish an exact hostname claim from a parent-domain claim. This decision authorizes `shop.seller.example` only; it does not grant `billing.seller.example` to the same tenant by implication. A broader grant needs an explicit policy and its own proof scope.

There is a delivery concern here too. Keep the marketplace's own transactional sender authorization separate from tenant web-host routing, and record which tenant operation triggered each onboarding notification. That makes it possible to diagnose an unconfirmed contact without treating a delayed confirmation as a failed DNS claim. Apply separate retry limits to notifications and proof checks so that repeatedly clicking verify does not repeatedly send email.

## The critical path consumes a claim, not just a DNS answer

The following Python example specifies the application boundary. It deliberately accepts parsed TXT values from a resolver adapter instead of pretending to implement a DNS client. The adapter must query the claim's exact challenge name, preserve record boundaries, and combine character strings belonging to a single TXT record before returning each record value. RFC 7489 section 6.1 discusses TXT string concatenation for its own records; the same parsing concern is relevant when defining an application's TXT representation. Never join distinct records into a new token.

Use a fresh, unpredictable token for each new claim and a dedicated challenge label, such as `_marketplace-claim.shop.seller.example`. Do not replace the customer's existing mail-policy records. Here, the proposed application format is `marketplace-claim=` followed by the token. That prefix is a local protocol choice, not a DNS or DMARC standard.

```python
from dataclasses import dataclass
from datetime import datetime
from hmac import compare_digest
from typing import Protocol, Sequence


@dataclass(frozen=True)
class Claim:
    claim_id: str
    tenant_id: str
    hostname: str
    challenge_name: str
    token: str
    expires_at: datetime


class ClaimStore(Protocol):
    def activate_if_current(self, claim: Claim) -> bool:
        # Atomically check expiry and current claim status,
        # enforce hostname uniqueness, and consume the claim.
        ...


def activate_hostname(
    claim: Claim,
    authenticated_tenant: str,
    now: datetime,
    txt_values: Sequence[str],
    store: ClaimStore,
) -> bool:
    # Datetimes must be timezone-aware; TXT values must come
    # from a completed query for claim.challenge_name.
    if authenticated_tenant != claim.tenant_id:
        return False
    if now >= claim.expires_at:
        return False

    expected = (
        "marketplace-claim=" + claim.token
    ).encode("utf-8")
    matched = any(
        compare_digest(value.encode("utf-8"), expected)
        for value in txt_values
    )
    if not matched:
        return False

    return store.activate_if_current(claim)
```

The store method is a required interface contract, not an implemented database transaction. It must check the persisted claim against current state, including expiration at commit time, because the earlier time check cannot protect a slow lookup or a delayed transaction. It must also verify that the supplied token and hostname still belong to the current claim. Put the uniqueness constraint on the canonical hostname in persistent storage. Two concurrent workers can both observe a matching TXT value; only one tenant binding may win.

A match is evidence.

It is not a transaction. That is the failure boundary this design protects: the external observation can succeed while the application rejects an expired, replaced, or conflicting claim. Keep that rejection explicit in the result model used by the real service. The compact Boolean above only illustrates the authorization path; a production interface should distinguish pending proof, resolver failure, expired claim, and conflicting allocation without revealing another tenant's account details.

## Deployment, retries, and recovery preserve the same boundary

Treat an unsuccessful DNS lookup as an observation with a reason, not an invitation to fall back to email authorization.

A missing value leaves the claim pending. A resolver timeout leaves the observation incomplete. A matching old token leaves the current claim unverified. Give customers the exact challenge name and required value, and show when the last check ran; do not promise a fixed activation delay that the application cannot establish.

Test the boundaries before enabling automatic activation. The focused cases are a contact confirmation with no TXT proof, a token published for a different tenant's claim, an expired token that still appears in DNS, multiple TXT records where only one matches, and two simultaneous attempts to bind the same hostname. Include a test where the claim expires after DNS succeeds but before the database commits. Those cases exercise authority and timing, rather than merely confirming that the happy-path string comparison works.

For deployment, make the current claim and its state visible to both the verifier and the routing controller. Drive routing changes from committed bindings. Record the claim identifier, canonical hostname, lookup outcome, and activation outcome in restricted operational logs; keep raw tokens and contact addresses out of routine metrics. This is a proposed data-minimization boundary, not a claim that the design alone satisfies a compliance framework.

Recovery needs the same discipline as initial onboarding. Removing a binding should stop its routing authorization. Reassigning the name to a different tenant should require a fresh claim, even if an old TXT token remains published. Decide explicitly when to revalidate an existing binding and what event suspends it; the initial proof cannot establish future control indefinitely. Keep historical evidence for the documented retention period so support can explain an activation without treating those old records as reusable credentials.

The operational cost is mainly coordination and diagnosis in this decision: customers may need a DNS administrator, while the platform needs a verifier, transactional state, and actionable status messages. Email confirmation reduces that coordination for contact collection.

It cannot remove the missing authorization evidence.

## Rejected option: email-only hostname activation

Reject email-only activation for customer-owned marketplace hostnames. It couples contact delivery to a domain authorization decision and makes a successful mailbox interaction carry more authority than this design intends. An OTP or confirmation link can be useful without authorizing a hostname; keep its completion in the contact record.

Email remains a valid choice when the protected operation is confirming an onboarding contact or delivering a notification. Platform-owned subdomain allocation also remains valid without customer DNS proof, provided the tenant is authorized and the requested name is available in the platform's registry. These are distinct operations with distinct evidence.

The decision is to automate platform-owned allocation and require scoped TXT proof for customer-owned claims, with a transactional activation gate after verification. Preserve that distinction during retries, deployment, and recovery. It is the boundary that keeps a contact workflow from becoming a hostname takeover path.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- RFC 7489, section 6.1, General Record Format: https://datatracker.ietf.org/doc/html/rfc7489#section-6.1
- RFC 7489, section 7.1, Verifying External Destinations: https://datatracker.ietf.org/doc/html/rfc7489#section-7.1
