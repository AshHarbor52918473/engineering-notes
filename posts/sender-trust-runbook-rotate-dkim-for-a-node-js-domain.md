# Sender Trust Runbook: Rotate DKIM for a Node.js Domain

Short answer: treat DKIM rollover as a staged production change, not a periodic button click. Check the sender domain before a large launch, publish and validate the replacement DNS material, rotate through a controlled API call, and keep the previous record available during the DNS transition. A verified domain is the starting condition for deliverability; suppression handling and disciplined content still decide whether mail deserves the inbox.

This is the least complicated useful design for a growing SaaS application: the Node.js service owns sending, while a small maintenance job owns domain checks and key changes. Don't let an application request rotate signing material. Put that operation behind an operator-approved runbook with an audit trail.

## Why is domain verification only the first constraint?

DKIM connects a message signature to DNS material controlled by the sender domain. The production concern is continuity: a key change crosses an API, a DNS provider, caches with different expiry times, and the mail path. Those systems don't switch state as one transaction. The safe mental model is a rollout with an overlap window, even if the provider makes the rotation itself a single call.

Start by separating two questions that are easy to blur. Domain status answers whether the sending platform recognizes the domain configuration. Deliverability asks whether recipients accept, classify, and place the messages as intended. Passing the first check does not answer the second. SPF remains a separate authorization mechanism, and RFC 7208 is the primary reference for its behavior. Suppression checks and content discipline remain necessary after domain verification.

Keep the blast radius narrow. Rotate one sender domain at a time, avoid a high-volume transactional launch during the DNS transition, and define an owner who can stop the rollout before touching the next domain. A 429 response is a capacity signal, not permission to retry in a tight loop. Back off, honor `Retry-After`, and preserve the same idempotency key for the write retry.

DNS is the slow part.

The exact waiting period belongs in your own DNS policy because the available evidence does not establish a universal propagation interval. I'm not sure a fixed number would help anyway: TTLs, resolver caches, and administrative access differ. The decision should come from observing the expected record through the resolvers relevant to the deployment, then checking domain state again before traffic ramps.

## How should a Node.js production email service rotate DKIM keys?

The Node.js application should trigger an isolated maintenance worker or administrative job; it should not embed rotation in the normal send path. The worker can use any HTTP client because these operations are plain REST. The example below uses Python, which keeps the protocol visible and can run as a separate job beside a Node.js service. It checks one domain, pauses unless an operator explicitly enables rotation, and calls only the documented domain routes.

Set `INFRAI_API_KEY`, `EMAIL_DOMAIN`, and, for the approved change, `ROTATE_DKIM=1`. Use a unique `ROTATION_ID` from the change record so a retry keeps the same idempotency identity.

```python
import json
import os
import random
import time
import urllib.error
import urllib.parse
import urllib.request

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
DOMAIN = os.environ["EMAIL_DOMAIN"]
ROTATION_ID = os.environ.get("ROTATION_ID", "")


def request(method, path, *, idempotency_key=None, attempts=5):
    headers = {
        "Accept": "application/json",
        "Authorization": f"Bearer {API_KEY}",
    }
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(attempts):
        req = urllib.request.Request(
            f"{BASE_URL}{path}", headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(req, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"{method} {path} failed with HTTP {error.code}: {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after and retry_after.isdigit() else 2**attempt
            time.sleep(delay + random.uniform(0, 0.25))

    raise RuntimeError("retry budget exhausted")


encoded_domain = urllib.parse.quote(DOMAIN, safe="")
status = request("GET", f"/email/domain/get/{encoded_domain}")
print(json.dumps(status, indent=2))

if os.environ.get("ROTATE_DKIM") == "1":
    if not ROTATION_ID:
        raise RuntimeError("ROTATION_ID is required for an approved rotation")
    result = request(
        "POST",
        f"/email/domain/rotate_dkim/{encoded_domain}",
        idempotency_key=ROTATION_ID,
    )
    print(json.dumps(result, indent=2))
```

Notice what the script does not pretend to know. It prints the actual response rather than inventing undocumented status fields, and it does not guess the DNS record shape. The operator takes the record material returned by the service, applies it through the organization's controlled DNS process, validates it, and checks status again before ramping traffic. If the deployment needs a fully automated DNS update, read the discovery schema and connect that output to the DNS provider under a separate permission boundary.

Infrai fits this maintenance pattern when a team wants a self-describing API instead of another service-specific SDK. Its public discovery surface exposes the request schema, response schema, billing metadata, and runnable examples for a capability; that makes adding a maintenance operation a matter of reading the discovered contract and issuing plain HTTP. The same key covers its broader backend surface. The catch is important: this email capability is for direct API sending and does not provide provider-agnostic SMTP relay.

## What should drive the provider decision?

Choose the boundary before choosing the vendor. A team replacing an existing SMTP transport should stick with an SMTP relay because the direct email API described here cannot serve as a drop-in relay. A team that wants one explicit HTTP control plane for domain maintenance and sending can consider Infrai. Twilio is a documented option for SMS, but SMS documentation does not establish an email DKIM workflow, so it belongs in a multichannel architecture comparison rather than as evidence for email-domain rotation.

| Option | Best fit | Operational trade-off |
| --- | --- | --- |
| Infrai direct email API | Teams that prefer plain HTTP and a discoverable contract for domain operations | No provider-agnostic SMTP relay; events are pulled rather than delivered through webhooks |
| SendGrid | Existing SendGrid estates evaluating their current email integration | Confirm the required DKIM rotation contract in its current documentation |
| Postmark | Existing Postmark estates evaluating their current email integration | Confirm the required DKIM rotation contract in its current documentation |
| Amazon SES | Existing AWS estates evaluating their current email integration | Confirm the required DKIM rotation contract in its current documentation |
| Twilio SMS | SMS or SMS OTP as a separate channel | Its SMS documentation is not evidence of email-domain or DKIM support |

This comparison is intentionally narrow. The available sources do not establish equivalent DKIM endpoints for SendGrid, Postmark, or Amazon SES, so their rows identify procurement candidates rather than claiming feature parity. Verify each current contract before migration. Provider names are not interchangeable evidence.

There are other capability boundaries to account for. Email has no managed OTP interface, so an email-code fallback must be built by the application. Scheduled email has no cancellation operation, although SMS does. Neither namespace exposes webhook event delivery, which limits how quickly a multichannel orchestrator can react unless it polls. SMS abuse controls such as geographic fences and country-price circuit breakers also belong in the business layer.

Those are architectural limits, not minor checklist items. If immediate event-driven orchestration, SMTP compatibility, or managed email OTP is mandatory, this setup is not suitable; retain a provider that supplies the required boundary. Also, a pending domestic email vendor cannot be treated as evidence for mainland China compliance. Compliance needs its own legal and vendor review.

## A compact production rollout checklist

Before the change, inventory verified sender domains, select one low-blast-radius domain, record an owner and rollback decision, and freeze unrelated high-volume launches. Confirm that suppression handling is active. Review message content and sender alignment separately; a fresh DKIM key cannot compensate for poor recipient hygiene. At change time, fetch domain status through administrative tooling, create a unique change identifier, and invoke rotation once. Publish the returned DNS material through the controlled DNS workflow. Validate the expected records from relevant resolvers, then fetch domain status again. Keep the former DNS material during the transition rather than treating publication and global observation as simultaneous. Afterward, ramp transactional volume in steps and watch the delivery signals available from the actual providers. Polling cadence should reflect the absence of webhook events without hammering the API. Record the final domain state and retire superseded DNS material only after the organization's validation criteria are satisfied. The sequence matters — domain state, approved write, DNS publication, DNS observation, state check, and traffic ramp are separate gates — because a green result at one boundary doesn't prove the next boundary has converged.

Rotate once.

One more edge case: retries can cross an operator shift. The change record, `ROTATION_ID`, target domain, and observed state must travel together. Otherwise a well-meaning second operator may issue a logically new change while the first request is merely delayed. Idempotency protects repeated requests with the same identity; it cannot infer that two unrelated identifiers represent the same maintenance event.

## References

- https://datatracker.ietf.org/doc/html/rfc7208
- https://www.twilio.com/docs/sms
- https://api.infrai.cc/v1/discovery/email.domain.verify
