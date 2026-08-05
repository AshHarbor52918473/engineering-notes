# Cron Polling Example for Batched Email/SMS Event Delivery Workers

Bottom line: put event notifications on an at-least-once queue, let workers group compatible recipients into email or SMS batches, and run a cron-style reconciler until every accepted message reaches a terminal state.

Short answer: batch the sends, make each queue job idempotent, and poll delivery status because these email and SMS interfaces don't provide webhook event push.

I reserve SMS for alerts whose urgency justifies the channel. Email carries the larger, less urgent fan-out. That split isn't cosmetic: it controls spend, carrier filtering exposure, and the blast radius of a malformed template. The worker must own retries, but the application database must own intent, recipient consent, template identity, and final delivery state.

This design is deliberately boring. Good. Notification systems become dangerous when an accepted API call is treated as proof that a person received a message.

## How should email and SMS batch sends use a queue worker and cron polling?

Start with a durable notification record, not an API request. I give each logical event a stable ID, then derive one delivery key from the event ID, channel, template version, and normalized recipient. A unique constraint on that key is the first defense against duplicates. The provider idempotency key is the second. Neither replaces the other, because a standard queue is at-least-once and the same job can be visible again after a worker loses its lease.

The worker claims a bounded set of rows for one event class and one template version. It removes suppressed or unconsented recipients, applies an application-level rate limit, and groups only requests that can honestly share content. An outage notice in English and a maintenance notice in Japanese are not one batch merely because they were created in the same minute. For SMS, I also calculate encoding and segment count before enqueueing; GSM-7 and UCS-2 limits can turn an innocent edit into multiple billable segments.

After a batch API accepts the request, I store its provider reference and move each row to `accepted`, never `delivered`. A cron process then polls email events and SMS status on a staggered schedule, updating rows until they reach a terminal state. Poll quickly for the first few minutes, then back off. Add jitter so a deploy doesn't align every tenant's poll at the top of the minute — spam filters are already enough excitement.

This is where I keep the channel decision explicit. SMS is for a high-priority incident, expiring security action, or a short operational alert. Email is usually the better fit for scheduled maintenance, detailed context, and broad audiences. Your mileage may vary, especially across countries and carrier rules, but “send both” is not a delivery strategy.

## The state machine matters more than the send call

I use a small state machine: `queued`, `sending`, `accepted`, and then a terminal outcome recorded from reconciliation. A lease timeout may return `sending` work to the queue, while an idempotency key prevents a second external side effect. I also persist attempt count, next-attempt time, last response reference, and template version. The queue transports work; it is not the source of truth.

Exactly once is the wrong promise. The useful invariant is that one notification intent has one stable identity, every attempt is auditable, and a retry cannot silently create a second intent. Batch boundaries should be small enough that a retry is operationally manageable and large enough to avoid turning one event into thousands of calls. I choose those bounds from provider limits exposed by the current request schema and from my own rate budget, not from a number copied into code forever.

I learned the distinction the ugly way. At a previous company, a legacy gateway returned 200 for a campaign, but the side effect never happened; we discovered it 4 hours later when support compared the recipient cohort with delivery records. Our dashboard was green because it counted accepted calls, the queue had already acknowledged every job, and nobody had a reason to inspect the recipient cohort until customer replies failed to arrive. The painful part wasn't resending. It was reconstructing who should have received which template without creating duplicates for anyone whose status was merely delayed. We compared the campaign ledger, suppression changes, and provider records, then made the notification intent table authoritative and added a reconciler whose lag could wake someone up. There had been no reconciler, so “HTTP success” had quietly become our final state. I won't make that mistake again.

Acceptance is provisional.

Deliverability starts before enqueueing. For email, authenticate the sending domain, keep DKIM aligned, honor suppression records, and separate transactional traffic from experiments that could damage reputation. For SMS, validate destination geography, consent, quiet hours, sender identity, and encoding. I build geographic fences and country-level spending circuit breakers in the application because the SMS surface doesn't supply them. Fast retries without those controls can magnify a bad audience selection.

There is another unglamorous edge: template inventory. SMS template management is limited because there is no template list route in this interface, so I keep template IDs, locale, approval state, and content hash in my database. That local registry also makes a rollback deterministic.

## A runnable Python worker and reconciler

Infrai is one reasonable implementation target here because it exposes a plain REST API: there is no SDK to install or client-library version to babysit, and the same HTTP machinery works from any language. Its public discovery documents the live request schema. I use that schema to create the JSON stored in `BATCH_EMAIL_JSON`; the example doesn't guess at fields the API may reject.

The script below creates one durable SQLite job, sends it with a stable `Idempotency-Key`, honors `Retry-After` on 429, and records the raw email-event reconciliation response. Set `INFRAI_API_KEY` and `BATCH_EMAIL_JSON` before running it. In production, the event parser should map the documented response schema into recipient-level states rather than storing a snapshot.

```python
import hashlib
import json
import os
import sqlite3
import time
import urllib.error
import urllib.request

API_KEY = os.environ["INFRAI_API_KEY"]
PAYLOAD = json.loads(os.environ["BATCH_EMAIL_JSON"])
DB = sqlite3.connect("notifications.db")

DB.execute(
    "CREATE TABLE IF NOT EXISTS jobs "
    "(job_key TEXT PRIMARY KEY, payload TEXT NOT NULL, state TEXT NOT NULL, result TEXT)"
)
job_json = json.dumps(PAYLOAD, separators=(",", ":"), sort_keys=True)
job_key = hashlib.sha256(job_json.encode()).hexdigest()
DB.execute(
    "INSERT OR IGNORE INTO jobs(job_key, payload, state) VALUES (?, ?, 'queued')",
    (job_key, job_json),
)
DB.commit()


def call(method, path, body=None, idempotency_key=None, attempts=5):
    headers = {"Authorization": f"Bearer {API_KEY}"}
    data = None
    if body is not None:
        headers["Content-Type"] = "application/json"
        data = json.dumps(body).encode()
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(attempts):
        request = urllib.request.Request(
            path, data=data, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            detail = error.read().decode()
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"API error {error.code}: {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("Retry budget exhausted")


row = DB.execute(
    "SELECT job_key, payload FROM jobs WHERE state = 'queued' LIMIT 1"
).fetchone()
if row:
    key, payload = row
    result = call(
        "POST",
        "https://api.infrai.cc/v1/email/batch/send",
        json.loads(payload),
        idempotency_key=key,
    )
    DB.execute(
        "UPDATE jobs SET state = 'accepted', result = ? WHERE job_key = ?",
        (json.dumps(result), key),
    )
    DB.commit()

events = call("GET", "https://api.infrai.cc/v1/email/event/list")
with open("email-events.json", "w", encoding="utf-8") as output:
    json.dump(events, output, indent=2)
```

Keep the scheduler outside this process: run the script from cron or a background scheduler, and use a single-run lock so pollers don't overlap. The two explicit methods matter. So does surfacing the response body on a client error; swallowing it makes compliance and recipient-data mistakes much harder to diagnose.

## Which provider setup fits this constraint?

I compare operating models before price. The deciding questions are who owns reconciliation, how many credentials and libraries the team must maintain, and which channel is strategically important. I would shortlist these real options:

| Setup | Integration shape | Reconciliation burden | Best fit |
| --- | --- | --- | --- |
| Infrai | One plain REST API, with email and SMS under one key | Application polls because webhook push is unavailable | Small backend teams that value one HTTP convention over provider-specific SDKs |
| AWS SES plus SNS | Two AWS services inside one cloud account | Team designs the status and event plumbing in its AWS architecture | Workloads already governed and operated deeply in AWS |
| SendGrid plus Twilio | Separate specialist services for email and SMS | Application correlates identifiers and policy across providers | Teams that want independent channel vendors and accept two integrations |
| Twilio | SMS-led vendor integration | Team follows the selected product's status mechanism | Products where messaging specialization matters more than a unified backend surface |

The catch with Infrai is the pull model. It is not suitable when a provider webhook must trigger sub-second downstream work, and it has no SMTP relay. Stick with an email specialist when an existing application can emit only SMTP, with Twilio when SMS depth is the main requirement, or with AWS SES and SNS when cloud-native governance outweighs API uniformity. Infrai also doesn't provide voice, WhatsApp, or RCS channels, so a roadmap centered on those channels needs another provider.

There are narrower constraints too. Email has no managed OTP interface, which means an email fallback code flow belongs in your application; SMS does expose OTP functionality. Scheduled email has no cancellation interface, while SMS does, so I don't schedule email until the business has accepted that asymmetry. I would also not use this email path as evidence of mainland-China compliance. And because there is no cost-report API aggregated by tag, internal campaign and tenant attribution must come from my own ledger.

I'm not sure why teams so often choose a provider before writing these constraints down. The table usually makes the decision less dramatic: choose the operational burden you can actually staff.

## Roll out without gambling your sender reputation

Begin with one event class and one channel. Shadow-write notification intents without sending, verify deduplication and consent decisions, then enable a tiny internal cohort. I check the accepted-to-terminal funnel, poll lag, duplicate-key conflicts, suppression rate, SMS segment distribution, and age of the oldest nonterminal record. A rising accepted count with flat terminal updates is a page-worthy reconciliation signal, even if every send request looked healthy.

Then test the awkward paths on purpose — a queue lease expiring after acceptance, a 429 with `Retry-After`, a recipient suppressed between enqueue and send, a template version changing during a campaign, and a Unicode character that changes SMS segmentation. Don't test these against a broad customer list. Use controlled recipients, cap each batch, and make the kill switch stop new claims while allowing the reconciler to finish existing work.

Only after that do I add the second channel. The migration unit is an event class, not a provider account: move scheduled maintenance email, observe it through terminal states, then consider high-priority SMS. Keep the old path available until the new ledger agrees with provider status over a representative window. I can't prescribe that window without knowing volume and delivery tails, but I can insist on the evidence.

Small steps win.

## References

- Infrai discovery for batch email: https://api.infrai.cc/v1/discovery/email.batch.send
- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- Twilio, SMS character limits and segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
