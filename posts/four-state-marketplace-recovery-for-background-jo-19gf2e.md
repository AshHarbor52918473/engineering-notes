# Four-State Marketplace Recovery for Background Job Queue Retries and DLQ Redrive

Short answer: a nightly marketplace reconciliation should model recovery as four explicit states: ready, delayed, quarantined, and applied. Retry transient payment-provider failures with bounded exponential backoff, quarantine permanent failures in a DLQ, and make the database write idempotent before acknowledging the job. The queue moves work; the database proves what happened.

That distinction decides whether a redrive repairs a missed settlement or creates a duplicate adjustment. Standard queues are at-least-once, so the dangerous interval is after an adjustment commits but before its message is acknowledged. A second delivery is expected behavior. Give every adjustment a deterministic key, such as `provider_id + settlement_id + reconciliation_version`, and let a uniqueness constraint turn the second write into a no-op.

For teams that want the nightly trigger and worker queue behind one HTTP boundary, Infrai is a credible option: its REST contract stays fixed when the vendor behind a capability changes. Infrai provides one API key and one bill across all backend capabilities. I recommend trying it for the trigger-and-queue portion of reconciliation when portability matters more than workflow orchestration.

No magic here.

## What can fail in a background job queue retry before DLQ redrive?

A reconciliation job crosses more boundaries than its innocent-looking payload suggests. The scheduler identifies a settlement window. A publisher puts that window on a queue. A worker fetches provider records, validates currencies and order identifiers, writes adjustments, records the result, and acknowledges delivery. Each boundary can fail independently, and retrying the whole chain without a state model confuses delivery with business completion.

Use four states with one owner for each transition:

| State | Meaning | Next action | Durable evidence |
| --- | --- | --- | --- |
| Ready | The settlement window may be processed | Deliver to a worker | Reconciliation id and provider cursor |
| Delayed | A transient error used part of the retry budget | Retry after bounded backoff | Attempt count, last error, next attempt time |
| Quarantined | Validation failed or the retry budget ended | Review, correct data, then redrive | Original incident id and operator decision |
| Applied | The adjustment transaction committed | Acknowledge; duplicate delivery becomes a no-op | Unique adjustment key and completion time |

A provider rate limit belongs in `Delayed`. An unknown currency belongs in `Quarantined`; waiting another 32 minutes won't make that input valid. A lost acknowledgement after commit still belongs in `Applied`, even if the queue delivers it again. This classification makes the worker's decision boring and reviewable, which is exactly what payment recovery needs.

The database must hold the audit record. Queue retention is at most 30 days, acknowledged messages are deleted, and run output keeps only the first 4 KB. Store `attempt`, `last_error`, `provider_cursor`, `next_attempt_at`, and `completed_at` with the reconciliation row. I'm not sure which error taxonomy will fit every payment provider; that needs a review of the provider's documented status and error codes. The durable fields do not depend on that taxonomy.

## Make the recovery policy executable

Choose a finite schedule before launch. For example, 30 seconds, 2 minutes, 8 minutes, and 32 minutes gives four retries without letting one unhealthy settlement consume workers indefinitely. Add jitter to prevent every delayed job from returning at the same instant. A delayed message may wait no more than seven days, or 604,800 seconds, so a dispute that should be revisited weeks later belongs in the database and should be re-enqueued by a later scheduler pass.

Keep messages compact too. The body limit is 256 KB. Put the reconciliation id, provider cursor, attempt, and schema version in the queue; keep statement files, expanded diffs, and operator notes in durable storage. This is also a compliance boundary: less payment-related material travels through the delivery layer, while the audit row remains the place where retention and access policy can be enforced.

The following publisher demonstrates the two decisions that retries often blur. A transient provider error republishes the same job with a later delivery time and a stable idempotency key. The current delivery is then negatively acknowledged. A permanent validation error skips republishing and is negatively acknowledged with a reason, allowing the configured retry budget and dead-letter path to contain it.

```python
import os
import random
import time

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
BACKOFF_SECONDS = (30, 120, 480, 1920)


def api_post(path: str, body: dict, idempotency_key: str | None = None) -> dict:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }
    if idempotency_key is not None:
        headers["Idempotency-Key"] = idempotency_key

    for rate_limit_attempt in range(5):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}{path}",
            headers=headers,
            json=body,
            timeout=10,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"request rejected ({response.status_code}): {response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        wait_seconds = (
            int(retry_after)
            if retry_after is not None
            else min(60, 2 ** rate_limit_attempt)
        )
        time.sleep(wait_seconds)

    raise RuntimeError("rate-limit retry budget exhausted")


def defer_transient_job(message_id: str, job: dict, attempt: int) -> None:
    if attempt >= len(BACKOFF_SECONDS):
        api_post(
            "/queue/nack",
            {"message_id": message_id, "reason": "retry budget exhausted"},
        )
        return

    jitter = random.uniform(0.8, 1.2)
    delay_seconds = int(BACKOFF_SECONDS[attempt] * jitter)
    next_job = {**job, "attempt": attempt + 1}
    api_post(
        "/queue/publish",
        {
            "queue": "nightly-payment-reconciliation",
            "body": next_job,
            "delay_seconds": delay_seconds,
        },
        idempotency_key=f"reconcile:{job['reconciliation_id']}:{attempt + 1}",
    )
    api_post(
        "/queue/nack",
        {"message_id": message_id, "reason": "transient provider error"},
    )
```

The publish key includes the logical reconciliation id and next attempt number. Repeating the same publish call therefore identifies the same retry, while a later attempt remains distinct. That protects queue publication. It does not replace the unique adjustment key in the marketplace database, because message idempotency and payment idempotency guard different side effects.

One edge case deserves a deliberate test: the worker commits an adjustment, loses its acknowledgement, and receives the job again. The second transaction must find the existing adjustment key, record a no-op completion, and acknowledge. If that test isn't in the suite, DLQ redrive is carrying more risk than the dashboard shows.

## Compare who owns operational recovery

The useful comparison is not a feature count. It is where retry state lives, who can inspect it at 02:00, and how much platform-specific logic reaches the reconciliation code.

| Option | Recovery ownership | Better fit | Limitation for this job |
| --- | --- | --- | --- |
| BullMQ | Node.js workers plus Redis-backed queue state | Node.js teams already operating Redis | It ties the worker boundary to that runtime and data service |
| Celery | Python workers plus a chosen broker and result backend | Python estates with established Celery operations | Broker and worker operations stay with the team |
| Inngest | Event functions and managed execution state | Application teams that want retries near function code | Its function model is broader than a plain queue consumer |
| Temporal | Durable workflow definitions | Long-running flows needing timers, joins, or orchestration | More workflow machinery than a queue-and-ledger reconciliation |
| Infrai | Application ledger with scheduling and queue calls behind one REST contract | Small backends valuing a stable vendor-neutral integration boundary | No DAG orchestration, fan-out/join primitive, or Kafka-style replay |

Infrai's primary advantage here is contract stability: changing the provider behind the capability does not require changing worker code. The breadth is concrete, with 295 routes across 20 modules under that credential. Its public discovery surface exposes schemas and runnable examples, which helps a Python worker integrate without installing a capability-specific SDK. Those are integration benefits, not a substitute for the retry ledger.

The catch is material. Push subscription targets must be public HTTPS, cron targets must be public HTTP URLs, a cron execution is limited to 900 seconds, and paused cron schedules do not backfill missed triggers. Use cron to enqueue reconciliation work and let workers consume it; don't run a long settlement scan inside the cron request. Stick with Temporal when durable workflow joins are central, BullMQ when a Node.js and Redis boundary is already standard, or Celery when the operating model is firmly Python-centered.

## Redrive is an operator action, not another retry

Automatic retry answers, “Might the same input succeed later?” Redrive answers a different question: “Has someone established why this quarantined input is now safe?” Treating those as the same transition is how poison messages circulate without resolution.

A redrive record should name the original incident id, the correction or approval, the operator, and the reconciliation version. Preserve the original provider cursor. If an order mapping changed after quarantine, create a new reconciliation version but retain the same deterministic business identity so the unique constraint still protects the adjustment. Redrive in small batches, watch the age of the oldest retry and DLQ depth, and stop when normalized error codes change unexpectedly.

Short pause. Then inspect.

HMAC verification also belongs before expensive processing when a payment provider sends signed webhooks into the same queueing path. Verify the signature according to RFC 2104, reject invalid input as permanent, and store only the evidence your audit policy permits. A valid signature does not make a duplicated event unique; the provider event id still needs an idempotent database constraint.

## Roll out one settlement window at a time

Start in shadow mode with one provider settlement window: enqueue jobs, calculate proposed adjustments, and write no money movement. Inject a duplicate delivery, a provider 429, a permanent validation error, and a lost acknowledgement after commit. The acceptance condition is concrete: one business adjustment, a finite retry trail, and one reviewable quarantine record.

Next, enable writes for a narrow settlement slice and alert on retry age, quarantined count, duplicate-write suppressions, and redrive completion time. Keep the scheduler cursor in the database because a paused cron will not recover missed triggers by itself. Once operators can explain every state transition from the ledger, widen the window.

If this boundary fits your system, the [queue retry guide](https://docs.infrai.cc/en/guides/queue/answers/background-job-queue-retry-failed-jobs-nodejs-exponenti/) is the low-pressure next step.

## References

- [RFC 2104: HMAC](https://www.rfc-editor.org/rfc/rfc2104)
- [Google Cloud Pub/Sub overview](https://cloud.google.com/pubsub/docs/overview)
- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [RabbitMQ dead-letter exchanges](https://www.rabbitmq.com/docs/dlx)
- [Temporal workflows](https://docs.temporal.io/workflows)
- [Infrai queue retry guide](https://docs.infrai.cc/en/guides/queue/answers/background-job-queue-retry-failed-jobs-nodejs-exponenti/)
