# Idempotency Keys and Retries: Proving One Email or SMS per Seller Order Event

Use one durable ledger row per (event, recipient, channel) as both the deduplication key and the compliance record: write it before the send, commit it, and make every retry read it first. Duplicate email and SMS notifications are almost never a Node.js problem — they're a bookkeeping problem, and the fix that survives an audit is the one where the idempotency keys you deduplicate on are the same rows you hand to a reviewer 30 days later. Exactly once, in a notification backend, means exactly one provable attempt per logical event.

That's the whole design. The rest of this note is about where it leaks.

## The constraint: reconstruct any notification 30 days later

The system I keep coming back to is a logistics marketplace that tells a seller a new order landed. Two channels, because sellers live on their phones and the warehouse team lives in a shared mailbox: one transactional email, one SMS with the pickup window. Nothing exotic. What makes it hard is what happens weeks later, when the pickup was missed and three parties disagree about who knew what. The seller says no message ever arrived. The carrier has a driver log showing a wasted stop. Someone in operations has to decide who eats the fee, and the only admissible answer is a record — which event triggered the notification, which consent record covered that phone number at that moment, which template version rendered, what the provider returned, and whether the second attempt created a second message or correctly did nothing.

Most deduplication designs fail that question, not the technical one. An in-memory Set in the worker deduplicates fine until the pod restarts. A Redis key with a one-hour TTL deduplicates fine and then evaporates, which means that a month later you can't tell a suppressed retry from a send that never happened. Both prevent duplicates. Neither produces evidence.

So invert the usual order and let the retention requirement pick the mechanism: the artifact that prevents the duplicate has to be the same artifact that proves the send. One row, two jobs. Once you accept that, most of the architecture is forced — the row must be in a transactional store, it must be written before you call anyone, and its primary key has to be derivable from the event rather than generated per attempt.

If it isn't durable, it isn't evidence.

## Three windows where a second message is born

The first window is upstream. Order services publish `order.created` at least once, and a webhook redelivery or a rebalanced consumer group hands you the same business event under a new delivery id. If your deduplication key is the delivery id, you have deduplicated nothing.

The second is the crash window, and it's the one people design around badly. Your worker gets a 202, starts writing the result, and the process dies before the commit. There is no transaction spanning your database and a message carrier, so on restart a blind retry sends twice and a blind skip loses the only notification. Both failure modes are real, and only one of them is visible in your metrics.

The third window is identity drift. The same seller is stored as `+8613800138000` in the account record and `008613800138000` in the shipping profile; the same mailbox appears as `Ops@Seller.example` in one table and `ops@seller.example` in another. Two hashes, two keys, two messages, one very annoyed seller — and a compliance record that now looks like two separate consent events. Normalize to E.164 for phone numbers and lowercase the domain for email before hashing. Leave the email local part alone: dot and plus-sign handling is a provider-specific behavior, not a standard, so folding `a.b@` into `ab@` will eventually merge two different people.

## How should a Node.js backend deduplicate email and SMS notifications across retries?

The runtime is the least interesting part of this. In a Node.js backend, the practical rule is that the guard lives in the same module the queue worker calls — not in an HTTP controller that a retrying load balancer can invoke twice, and not in middleware that a batch job bypasses. Whatever your worker library, the sequence is: claim, send, resolve.

The claim is an insert with a unique constraint, and the constraint is the real arbiter. Application-level "check, then insert" is a race with a very generous window; two workers processing the same event will both read "no row" and both send. Let the database say no. An insert that conflicts returns nothing, and the worker that got nothing simply stops — it doesn't retry, doesn't wait, doesn't send.

Code below is Python (that's the runtime our notification worker runs on), but the three writes port to a Node.js worker unchanged, because the guarantees live in the schema rather than the language:

```python
import hashlib
import os
import time

import requests

SEND_URL = os.environ["NOTIFY_SEND_URL"]        # your provider's send route
LOOKUP_URL = os.environ["NOTIFY_LOOKUP_URL"]    # message lookup, for reconciliation


def dedupe_key(event_id: str, channel: str, address: str, template_version: str) -> str:
    # Normalize identity first: +8613800138000 and 008613800138000 are one seller.
    ident = normalize_address(channel, address)
    return hashlib.sha256(f"{event_id}|{channel}|{ident}|{template_version}".encode()).hexdigest()


def claim(conn, key, event_id, channel, address, consent_id):
    """Durable claim before anything leaves the process. Returns False if someone else owns it."""
    row = conn.execute(
        """
        insert into notification_ledger (dedupe_key, event_id, channel, address, consent_id, state)
        values (%s, %s, %s, %s, %s, 'claimed')
        on conflict (dedupe_key) do nothing
        returning id
        """,
        (key, event_id, channel, address, consent_id),
    ).fetchone()
    conn.commit()
    return row is not None


def notify(conn, event, recipient, consent_id, payload, template_version):
    key = dedupe_key(event["id"], recipient["channel"], recipient["address"], template_version)
    if not claim(conn, key, event["id"], recipient["channel"], recipient["address"], consent_id):
        conn.execute("update notification_ledger set suppressed_count = suppressed_count + 1 "
                     "where dedupe_key = %s", (key,))
        conn.commit()
        return "suppressed"

    headers = {"Authorization": f"Bearer {os.environ['NOTIFY_API_KEY']}", "Idempotency-Key": key}
    for attempt in range(4):
        try:
            resp = requests.post(SEND_URL, headers=headers, json=payload, timeout=10)
        except requests.RequestException:
            mark(conn, key, "unresolved")           # network verdict unknown: reconcile, never re-send blind
            raise
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", 2 ** attempt)))
            continue
        if resp.ok:
            mark(conn, key, "sent", provider_id=resp.json()["id"])
            return "sent"
        mark(conn, key, "failed", detail=resp.text[:500])
        resp.raise_for_status()

    mark(conn, key, "unresolved")
    return "unresolved"


def reconcile(conn, key):
    """Crash recovery: ask the provider what it did with our key before deciding anything."""
    found = requests.get(LOOKUP_URL, params={"idempotency_key": key}, timeout=10).json()
    if found.get("messages"):
        mark(conn, key, "sent", provider_id=found["messages"][0]["id"])
        return "sent"
    mark(conn, key, "claimed")                      # nothing went out; the send path may run again
    return "retryable"
```

Four states carry the whole state machine: `claimed`, `sent`, `failed`, `unresolved`. Resist adding more. The one that earns its keep is `unresolved` — it's the honest answer to the crash window, and it turns an unanswerable question into a queue an operator can drain. Reconciliation asks the provider what it did with your key, which is why the key must be stable across attempts; generating a fresh UUID per try defeats every layer at once, including the provider's own duplicate suppression. Keep `suppressed_count` on the row too. A sudden climb in suppressions is almost always an upstream producer replaying a partition, and I'd rather find that in a counter than in a seller's inbox.

## Where the ledger stops paying, and what to reach for instead

Deduplication can live in four places, and they protect against different things:

| Guard | Stops | Can't prove | Reasonable when |
| --- | --- | --- | --- |
| Provider `Idempotency-Key` header | Repeated HTTP attempts inside the provider's retention window | Anything after failover to a second provider or channel | You send through one vendor and one channel |
| Cache key with TTL | Fast replays inside the window | That a suppression happened, once the key expires | High volume, low consequence, no audit duty |
| Transactional ledger row | Producer replays, worker crashes, cross-channel fan-out | Nothing — this is the record | Consequences are legal, financial, or contractual |
| Recipient-side collapsing | Nothing upstream; it only hides symptoms | Which attempt the recipient actually saw | Never as a primary mechanism |

The catch is cost, in two currencies. Every send now waits on a committed write, so your database becomes a hard dependency of notification delivery and your P99 grows by a round trip; and every notification becomes a row you retain for as long as your dispute window, which for regulated shipping paperwork can be over a year. For bulk marketing sends at millions per day, that's a bad trade — stick with the provider header plus a cache key and accept the occasional duplicate, because nobody litigates a repeated newsletter. The ledger is for messages someone might have to defend.

Two more boundaries worth stating plainly. A provider-issued idempotency key doesn't support cross-channel or cross-vendor reasoning: if SMS fails over to a backup route, the backup has never seen that key and will happily send again, so the key has to be yours and travel with the payload. And none of this reduces rate-limit pressure — I've spent more hours on 429 storms and spam-folder forensics than on any send call, and a ledger makes retry storms safe without making them smaller. That still needs a per-recipient token bucket in front of the worker.

## Rolling this out without changing notification semantics

Dual-write first: compute and store the key while the existing sender stays authoritative, and compare what the ledger *would* have suppressed against what actually went out. A week of that data tells you whether your duplicate rate is a producer problem or a crash-window problem, and those have different fixes.

Then test the window on purpose. In staging, kill the worker between the provider response and the commit, restart it, and assert that reconciliation resolves the row instead of creating a second message. If you can't make that test fail before the fix and pass after it, you haven't verified the property you claim to have.

Watch the agent-triggered path specifically, because it's newer than most runbooks. When an LLM tool call fires the notification, the tool invocation can be retried by the orchestration layer for reasons that have nothing to do with delivery — a timeout, a parse failure, a re-planned step. Derive the key from the business event, never from the tool call or the model's turn id. Cost lands here too: an SMS over 160 GSM-7 characters bills as multiple segments, so a duplicated two-segment alert to a seller fleet is four segments of real money plus a carrier reputation hit.

Migrate one event type at a time, starting with the one that shows up in disputes. Order notifications first; shipment status updates later. Honestly, I'm not sure the ledger is worth it for internal ops digests, and your mileage may vary depending on how your legal team reads retention — but for any message that a third party might later claim never arrived, the row you write before the send is the cheapest insurance in the stack.

## Further reading

- The Idempotency-Key HTTP Header Field (IETF HTTPAPI draft) — https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header
- RFC 5322, Internet Message Format (Message-ID uniqueness) — https://www.rfc-editor.org/rfc/rfc5322
- RFC 6585, Additional HTTP Status Codes (429 Too Many Requests) — https://www.rfc-editor.org/rfc/rfc6585
- RFC 9110, HTTP Semantics (Retry-After, idempotent methods) — https://www.rfc-editor.org/rfc/rfc9110
- ITU-T E.164, international public telecommunication numbering plan — https://www.itu.int/rec/T-REC-E.164
- PostgreSQL INSERT ... ON CONFLICT documentation — https://www.postgresql.org/docs/current/sql-insert.html
- Resend, transactional email API documentation — https://resend.com/docs/introduction
- Anthropic tool use overview (agent tool definitions and retries) — https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
