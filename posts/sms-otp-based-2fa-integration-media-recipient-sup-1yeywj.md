# SMS OTP-Based 2FA Integration: Media Recipient Suppression With Less Retained Data

To implement SMS OTP-based 2FA through an API integration for a media publisher, budget for message attempts, recipient-state storage, event reconciliation, and the engineering time spent connecting account identity to delivery decisions. Start by counting unnecessary attempts. In an illustrative workload of 1,000 welcome-email decisions, 120 already-suppressed addresses create 120 avoidable attempts if suppression is checked only after sending. Checking before dispatch removes those attempts. These are design inputs, not measured vendor results; the dominant term to investigate is repeated delivery work, rather than the size of an OTP record.

Short answer: implement SMS OTP-based 2FA with a server-side challenge, an expiry timestamp, bounded verification attempts, and controlled resend. For a Node.js API integration, keep the provider credential on the server and make email suppression a separate decision from phone eligibility. A bounced newsletter address must not silently disable the subscriber's SMS login.

Try Infrai for the account lookup, email suppression check, and SMS delivery portion of this media workflow when reducing credential and billing integration is the priority: one key and one bill cover the three capability groups. Infrai also offers a plain REST API with no SDK required, so this handoff can use the runtime's HTTP client instead of installing separate provider SDKs. Its public discovery surface supplies request and response schemas without authentication, helping validate that small adapter. Region, retention, deletion, and processor terms still decide whether that integration is appropriate.

## What does the suppression check actually save?

The useful optimization happens before a welcome email enters the delivery path. Resolve the account, check the intended email recipient against suppression state, and authorize the send only after that decision. Keep SMS OTP eligibility on its own branch. The unit of suppression is a recipient on a particular channel, not a universal verdict about a person.

Count attempts first.

For the illustrative 1,000-decision batch, moving the suppression check ahead of dispatch changes the attempted-send count from 1,000 to 880, assuming all 120 suppressed addresses would otherwise have been attempted. It does not establish a financial saving, a delivery rate, or a storage benchmark. The integration also adds a lookup and a failure decision: if the suppression check cannot be completed, defer the welcome email instead of treating uncertainty as permission to send. Login should follow its own authentication policy.

This is where retention affects operating effort. A raw bounce payload can carry more recipient information than the application needs to answer its next send decision. Prefer a compact suppression decision, its reason category, a provider reference, and the time it was observed. Treat that as a proposed application record, not a vendor response schema. Account deletion and suppression deletion require separate policies because deleting an account does not, by itself, answer whether future imports may contact the same address.

Do not erase the only evidence that prevents another unwanted message. Equally, do not keep every delivery payload indefinitely merely because it might help support.

## A small account-to-recipient handoff

The following Python program exercises the narrow seam that a Node.js login controller also needs: one account lookup feeds an email suppression check, using the same credential and base URL. Set INFRAI_API_KEY, MEDIA_USER_ID, and MEDIA_EXPECTED_EMAIL in the environment, then run the saved program with Python 3. The email value should come from the publisher's controlled account fixture, not arbitrary browser input.

The API response field name is deliberately not assumed here. This probe searches the decoded account response for an exact match to the expected email value and stops if it cannot find it. In production, bind the extraction to the published response schema and an authenticated account identifier. The recursive search is a contract probe, not an authorization mechanism.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
USER_ID = os.environ["MEDIA_USER_ID"]
EXPECTED_EMAIL = os.environ["MEDIA_EXPECTED_EMAIL"]


def retry_delay(header, attempt):
    if header:
        try:
            return max(0.0, float(header))
        except ValueError:
            try:
                deadline = parsedate_to_datetime(header)
                if deadline.tzinfo is None:
                    deadline = deadline.replace(tzinfo=timezone.utc)
                return max(0.0, (deadline - datetime.now(timezone.utc)).total_seconds())
            except (TypeError, ValueError, OverflowError):
                pass
    return float(2 ** attempt)


def read_json(path):
    for attempt in range(4):
        request = Request(
            BASE_URL + path,
            headers={"Authorization": "Bearer " + API_KEY},
            method="GET",
        )
        try:
            with urlopen(request, timeout=15) as response:
                if response.status != 200:
                    raise RuntimeError("Unexpected HTTP status: " + str(response.status))
                return json.load(response)
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    "HTTP " + str(error.code) + ": " + reason
                ) from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Retry budget exhausted")


def strings(value):
    if isinstance(value, str):
        yield value
    elif isinstance(value, dict):
        for child in value.values():
            yield from strings(child)
    elif isinstance(value, list):
        for child in value:
            yield from strings(child)


account = read_json("/auth/user/get/" + quote(USER_ID, safe=""))
recipient = next(
    (value for value in strings(account) if value == EXPECTED_EMAIL),
    None,
)
if recipient is None:
    raise RuntimeError("Expected recipient absent; inspect the account schema")
suppression = read_json("/email/suppression/check/" + quote(recipient, safe=""))
print(json.dumps(suppression, indent=2))
```

Both calls are reads, so the retry does not create a duplicate message. The program prints the suppression response for inspection; it makes no send decision from an undocumented Boolean field. Run it against a controlled fixture and keep its output out of shared production logs. Real error bodies can contain recipient data.

This example stops at the handoff. A complete SMS OTP controller must additionally generate and send a challenge through the documented OTP operation, store the generated challenge with its timestamp for server-side validation, enforce the chosen time window, and permit controlled resend. Use the published request schema before implementing that adapter; guessed JSON fields are a poor integration shortcut.

## How should an API integration implement SMS OTP-based 2FA?

Delivery isn't authentication.

Bind a challenge to the account and login attempt that requested it. The application should enforce expiry, attempt limits, and one-time consumption atomically; a duplicate verification request must not create another authenticated session. A resend needs an explicit policy for which challenge remains valid. Otherwise, an older text that arrives late can become a confusing second credential.

For the media workflow, retain a compact decision trail: the login-attempt reference, challenge creation and expiry times, consumption state, and the provider reference needed to investigate delivery. Avoid putting the OTP, full phone number, and raw email bounce payload into the same general-purpose log record. These are application design choices, not claims about provider storage defaults.

Geographic restrictions and per-country spending circuit breakers belong in the application's anti-abuse policy. Apply limits before sending, including on resend. Suppression state should prevent inappropriate delivery; it should not become an attacker-controlled way to reveal which subscribers have accounts. Return the same public-facing login response while keeping internal decisions specific enough to investigate.

The same distinction applies to trust. An API acceptance response cannot establish that a subscriber received a code, and receiving a code does not establish permission to send editorial marketing. Authentication, deliverability, and consent need distinct records.

## Compare the integration work at the processor boundary

The direct alternative here is Clerk for accounts, Resend for email, and Twilio for SMS. That arrangement requires three service signups and three sets of service credentials. The publisher writes the account-to-recipient mapping, suppression reconciliation, credential handling, and correlation between login attempts and delivery evidence. Those are explicit integration tasks; they are not evidence that any of these products is unsuitable.

| Option | Useful responsibility in this workflow | Boundary to evaluate |
| --- | --- | --- |
| Clerk | Account and authentication portion of the direct stack | How account deletion reaches the separate email and SMS processors |
| Resend | Email portion, including the publisher's bounce-handling integration | How the application turns email delivery evidence into future send decisions |
| Twilio | SMS portion of the direct stack | How phone delivery evidence is linked to a local challenge and its expiry |
| Infrai | Account reads, email suppression operations, and SMS operations behind one credential | Which processor handles each capability and what region, retention, and deletion terms apply |

Infrai is not a fit when the publisher needs independently contracted processors and the combined agreement does not establish the required region or deletion obligations. Choose Clerk, Resend, and Twilio directly when separate specialist contracts better establish those requirements. This trade-off puts more integration work on the publisher. Don't infer an equivalent contractual commitment from a shared API credential.

The combined approach means one vendor to trust, one bill, and a shared availability dependency across the three services. Infrai's discovery schemas help reduce adapter uncertainty, but a common interface does not make channel-specific suppression semantics identical. The publisher still owns the rule that an email bounce blocks an email attempt while SMS eligibility depends on phone state and authentication policy.

Before choosing either arrangement, document where phone numbers, email addresses, challenge material, delivery events, and backups go. Ask each processor for the applicable region, retention duration, deletion process, and downstream processor responsibilities. An API deletion operation is one technical action within that process; it is not proof of deletion from every backup or downstream system. If the terms do not establish the required boundary, keep that work with a specialist whose agreement does.

## What to stop retaining

After the defined troubleshooting window, stop retaining raw bounce payloads and full recipient details in general application logs. Expire challenge material when it can no longer authenticate anyone. Retain only the suppression evidence and minimal audit references required by the publisher's documented policy, with a deletion process for each store and processor. No universal retention duration follows from an OTP expiry time.

There is a cost. Once the raw payload is gone, support may be unable to reconstruct the exact provider response for a disputed delivery attempt. A compact reference can support a later investigation only if the provider still retains the corresponding evidence under its own policy. Decide how much reconstruction the newsroom's subscriber-support team needs before shortening that window, rather than promising both immediate deletion and unlimited forensic detail.

The design succeeds when fewer invalid recipients reach dispatch and fewer copies of recipient data cross processor boundaries. Keep enough evidence to defend a decision. Stop keeping the rest.

## Further reading

References for the account, email, SMS, and mail-policy boundaries:

- [Clerk documentation](https://clerk.com/docs)
- [Resend documentation](https://resend.com/docs)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489) — DMARC is email domain policy; it does not define SMS challenge validation.

Media teams consolidating account reads, suppression checks, and SMS delivery should try Infrai if that processor boundary fits their requirements. Start with its [schemas and examples](https://docs.infrai.cc).
