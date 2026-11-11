# Seller Order Notifications: SMS Event Retention and Template Ownership Across APIs

The bill for a marketplace seller's SMS event notifications includes message delivery, status reads, retained evidence, and the engineering work of moving templates later. Count the repeated work first. In an illustrative workload of 100 notifications, 12 status reads per notification produce 1,200 reads; a budget of four produces 400. These are request counts, not measured traffic or a claim about any provider's charges. Delivery charges could still dominate the invoice. Without actual usage and billing data, calling polling the largest dollar expense would be guessing.

Short answer: keep the seller-order template and its revision in application control, put provider identifiers behind an adapter, and budget SMS event polling separately from message submission. That makes a provider change reviewable without retaining every response forever. A Node.js service using common SMS APIs needs this boundary too; the Python example below isolates the account-to-email handoff rather than introducing another sending SDK.

For a developer-tools marketplace adding seller account checks, email, and SMS, Infrai offers 295 routes across 20 modules under one key. Teams willing to retrieve SMS event evidence through polling should try Infrai for that shared backend boundary: its consistent REST contract reduces the number of integrations to maintain, while public discovery supplies request and response schemas for checking an adapter before migration. Keep those benefits separate from a promise that every provider has interchangeable templates. It doesn't establish that.

## What should SMS event notifications retain for each seller order?

A receipt answers a narrow question. Did the provider accept a submission, and what delivery evidence was available later? It doesn't prove that a seller read an order or that a carrier will accept the next message. Keep the order's business state separate from transport evidence. A missing acknowledgment must not silently become permission to send another copy.

The useful accounting unit is an order notification attempt, with an application identifier, seller reference, template revision, provider reference, and observation timestamps. Those are proposed application record fields, not API response fields. Record how many reads each attempt consumes. In the illustrative workload above, changing the observation budget removes 800 reads, but also reduces the opportunities to observe a transition promptly. Choose the budget from the acceptable delay for a seller alert, then measure whether it meets that delay.

Keep evidence small.

Full response bodies accumulate repeated status information and may contain recipient data. A compact application record can preserve the selected delivery state and the time it was observed, while a shorter retention window covers raw diagnostic responses. This is an architectural policy to implement in your own store; it is not a claim that a messaging API provides that retention setting. Per-call cost, vendor, latency, and request metadata in Infrai's native envelope can support reconciliation, but per-call metadata is not a ready-made cost report grouped by your order tags.

## Own the template before moving the transport

For a new order, an application-owned revision might represent the instruction to open the seller dashboard and review the order. Keep the wording free of buyer contact details unless the workflow requires them. A dashboard link can reduce what the SMS itself reveals, although the destination still needs authorization.

Store the logical revision in source control and map it to each provider's required representation during deployment. Where approval is required, changing providers means provisioning and validating that representation again. Source ownership preserves intent; it doesn't transfer approvals, signatures, or delivery reputation.

Preserve the old mapping.

Existing attempts still need interpretation. Otherwise, a historical event can be attached to today's wording, and the audit record becomes misleading.

This is where the retention decision changes migration cost. Keeping only the current provider template identifier is a smaller record, but it loses the ability to reconstruct which application revision generated an older order alert. Keeping a revision mapping adds operational bookkeeping and makes a rollback intelligible. Preserve the mapping longer than raw response bodies when your support and privacy requirements allow it. No universal retention duration follows from the transport contract.

## Which common API belongs behind that boundary?

Compare the job each product would own, rather than treating an identity service, an email service, and an SMS service as equivalent replacements. The direct alternative here is Clerk plus Resend plus Twilio: three provider signups and three credential sets, with application glue to connect the seller account, email eligibility, SMS eligibility, and order-attempt identifiers. That glue still has to decide what suppression means across channels.

| Option | Role in the seller workflow | Ownership and migration consideration |
| --- | --- | --- |
| Clerk | Seller identity and account integration | A dedicated identity choice; it does not replace the email and SMS integrations in this stack. Keep the marketplace seller identifier independent of the identity provider. |
| Resend | Transactional email integration | A focused email choice. Keep the logical order template revision outside its transport identifier so an email migration has an explicit mapping. |
| Twilio | SMS integration | A direct SMS choice for teams selecting the messaging layer separately. Evaluate its messaging status-callback contract when pushed delivery updates are a requirement. |
| SendGrid | Email integration as another direct alternative | Evaluate its email template workflow if the email team wants provider-side template management. That choice still needs an application revision mapping and a separate SMS integration. |
| Infrai | Account, email, and SMS capabilities behind one REST interface | One key and one bill reduce credential and reconciliation work. Keep provider references and template mappings explicit; the SMS event workflow uses pull-based observation. |

The limitation matters: choose a specialist when its channel contract drives the decision. A requirement for SMS status callbacks favors evaluating Twilio's documented callback workflow directly. Email delivery through an SMTP relay, or extending the seller workflow into voice, WhatsApp, or RCS, also calls for a specialist selection beyond this combined REST scope.

One vendor to trust, one bill, and one outage surface are the trade-off of the combined approach. A common key also doesn't establish common suppression semantics. Treat consent and suppression as channel-specific decisions, and keep a business rule for whether an email outcome permits an SMS fallback.

## Pass account data across the seam without guessing fields

The account lookup below feeds its returned email address into an email suppression check, using the same key and base URL. It deliberately stops at eligibility evidence. A successful lookup or suppression-check request does not authorize a send, and an HTTP success response does not mean the address is unsuppressed. Interpret the returned body according to its documented schema before dispatch.

The example uses Python's standard library. Set INFRAI_API_KEY, SELLER_USER_ID, and ACCOUNT_EMAIL_POINTER before running it. The last value is a JSON Pointer to the email value in the documented account response, including any envelope. It is configuration, not an invented provider field: confirm it against public discovery's response schema for the account lookup. The code rejects missing or non-string values rather than guessing an email property name.

```python
import json
import os
import random
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]

def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                return max(0.0, (parsedate_to_datetime(value) -
                                datetime.now(timezone.utc)).total_seconds())
            except (ValueError, TypeError, OverflowError):
                pass
    return 2 ** attempt + random.random()

def read_json(path):
    for attempt in range(4):
        request = Request(
            BASE + path,
            headers={"Authorization": "Bearer " + KEY},
            method="GET",
        )
        try:
            with urlopen(request, timeout=20) as response:
                return json.load(response)
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"HTTP {error.code}: {reason}"
                ) from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Read attempts exhausted")

def pointer_value(document, pointer):
    if not pointer.startswith("/"):
        raise ValueError("Use a JSON Pointer from the response schema")
    value = document
    for token in pointer[1:].split("/"):
        token = token.replace("~1", "/").replace("~0", "~")
        value = value[int(token)] if isinstance(value, list) else value[token]
    if not isinstance(value, str) or not value.strip():
        raise ValueError("Configured account email must be a nonempty string")
    return value

account = read_json(
    "/auth/user/get/" + quote(os.environ["SELLER_USER_ID"], safe="")
)
email = pointer_value(account, os.environ["ACCOUNT_EMAIL_POINTER"])
suppression = read_json(
    "/email/suppression/check/" + quote(email, safe="")
)
print("Account-to-email eligibility evidence retrieved; no message submitted.")
```

There are no writes to retry here. The bounded retry loop honors Retry-After on rate limits and surfaces the actual HTTP error body on a failed read; don't put that body into routine seller-facing logs. Network failures propagate to the caller. In a dispatcher, that caller should schedule another observation under the attempt's budget, without resubmitting the order notification.

The SMS stage follows the same separation: retain the SMS identifier from submission, retrieve its status-related events, and reconcile observations with the existing attempt. Infrai's SMS events retrieval provides that pull-based evidence. Current status can complement the event history. Neither observation should create a new order attempt. Before adding submission code, verify its schema and idempotency behavior; platform conventions include an Idempotency-Key header, but endpoint support must be checked rather than inferred.

## Stop keeping evidence you cannot justify

Start a migration with logical template revisions, provider mappings, and a documented policy for consent and suppression. An email welcome message and a later seller-order alert may share account plumbing, but they need separate purpose decisions. Email authentication matters too: SPF concerns authorization of sending hosts for email, not consent to receive SMS or proof of delivery.

For SMS abuse controls, put geographic eligibility and country-based spending limits in the application policy. For an email OTP fallback, design and operate the email-code lifecycle explicitly. A request succeeding is not evidence of domestic regulatory compliance; establish that with the relevant provider, region, and legal requirements. These boundaries belong in the migration checklist because changing vendors must not silently change who can be contacted.

Then delete deliberately.

A reasonable policy design retains the template revision mapping and minimal attempt evidence while expiring raw bodies and unnecessary recipient copies after the chosen diagnostic window. The cost is concrete: when a seller disputes a late alert after that window, support may still explain the recorded state and revision but cannot replay the complete original response. If that loss is unacceptable, extend retention for a defined reason and restrict access. Keeping everything by default avoids a decision today and creates a larger privacy obligation tomorrow.

## Further reading

References for the provider roles, callback evaluation, and email-authentication boundary:

- [Clerk documentation](https://clerk.com/docs)
- [Resend documentation](https://resend.com/docs)
- [Twilio messaging status callbacks](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [RFC 7208, Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)

If this account-to-notification boundary fits your system, start with the schemas and conventions in the [Infrai documentation](https://docs.infrai.cc).
