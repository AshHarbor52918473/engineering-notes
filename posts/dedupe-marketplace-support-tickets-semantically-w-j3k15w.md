# Dedupe Marketplace Support Tickets Semantically with an API — Without Model Hosting

The least complex useful design is a hosted embedding API, a vector-capable record store, and a short retention window for ticket vectors. The bill is mostly shaped by how many vectors remain searchable, their dimensions, index overhead, replicas, and query volume. Model hosting is absent from that list.

**TL;DR:** For a marketplace, embed a normalized ticket when it arrives, search only the relevant tenant and recent time partition, and let an agent confirm any proposed duplicate. Keep the original ticket in the system of record, but delete expired vectors and derived search text on a declared schedule. This makes the retrieval-quality-versus-latency decision explicit: a wider candidate history may recover older repeats, while a narrower, filtered index reduces the searchable set and limits retained derived data.

## What are we actually paying to retain?

Start with inventory, not an API comparison. If `N` tickets are retained, each embedding has `D` numeric components, and the representation uses `B` bytes per component, the raw vector payload is `N × D × B`. That is only the floor. Metadata, index structures, replicas, backups, and operational headroom belong in the capacity plan too; their exact multipliers depend on the chosen database and configuration, so treating one universal multiplier as fact is a budgeting mistake.

For an illustrative workload, suppose the marketplace receives 50,000 tickets per day and retains searchable vectors for 30 days. That is 1.5 million active vectors before deletion. Moving to 90 days triples the active vector count to 4.5 million. This arithmetic does not predict a bill, but it identifies the dominant lever: retention multiplies every per-vector storage component and enlarges the candidate corpus.

The useful unit is therefore not “one embedding request.” Track vector-days, queries per ticket, bytes in index and replicas, and the fraction of searches that lead to a confirmed merge. A cheap write path can still support an expensive corpus if deletion never happens.

## How Should an API Dedupe Support Tickets Semantically?

A new marketplace ticket should first be normalized without erasing meaning. Strip transport noise such as repeated email signatures and quoted history, but preserve order identifiers, seller or buyer role, locale, channel, and issue category as separate metadata. OTP delivery failures deserve special care: two messages saying “code never arrived” may be semantically close while belonging to different accounts and security contexts. Retrieval is a candidate generator, not authorization to expose or merge records.

The online path is short:

1. Persist the original ticket and obtain its immutable identifier.
2. Build derived search text, request one embedding, and store it with tenant, locale, category, creation time, and embedding-version metadata.
3. Search a bounded recent partition with mandatory tenant and access filters.
4. Return a small candidate set with stable identifiers and similarity values.
5. Apply deterministic exclusions, then ask a support agent to confirm the duplicate relationship.

Do not turn a similarity score into a universal truth. Calibrate a review threshold against labeled marketplace pairs, including hard negatives such as the same stock complaint for different orders. Measure precision and recall separately, then report latency at representative corpus sizes. The operating point is a policy choice: false merges can hide a distinct fraud, delivery, or account-access case, while missed matches mainly preserve duplicate work.

Scope first.

This is where compliance and deliverability constraints line up. A tenant filter that runs after retrieval is too late because an unauthorized record has already entered the candidate path. Likewise, rate-limit embedding requests and make ingestion idempotent; retry storms should not create several vectors for one ticket.

## Make retention executable

A retention statement that lives only in a document will drift. The following Python sketch keeps vendor details behind two small interfaces. It records an expiry time at ingestion and deletes both expired vectors and derived text in batches. The constants are examples, not recommended universal settings.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from typing import Protocol, Sequence


class EmbeddingAPI(Protocol):
    def embed(self, texts: Sequence[str]) -> list[list[float]]: ...


class VectorStore(Protocol):
    def upsert(self, *, key: str, vector: list[float], metadata: dict) -> None: ...
    def delete_expired(self, *, before: datetime, limit: int) -> int: ...


@dataclass(frozen=True)
class Ticket:
    ticket_id: str
    tenant_id: str
    locale: str
    category: str
    search_text: str


RETENTION_DAYS = 30


def index_ticket(ticket: Ticket, api: EmbeddingAPI, store: VectorStore) -> None:
    now = datetime.now(timezone.utc)
    vector = api.embed([ticket.search_text])[0]
    store.upsert(
        key=ticket.ticket_id,
        vector=vector,
        metadata={
            "tenant_id": ticket.tenant_id,
            "locale": ticket.locale,
            "category": ticket.category,
            "created_at": now.isoformat(),
            "expires_at": (now + timedelta(days=RETENTION_DAYS)).isoformat(),
            "embedding_version": "configured-by-deployment",
        },
    )


def expire_search_records(store: VectorStore, batch_size: int = 1_000) -> int:
    cutoff = datetime.now(timezone.utc)
    deleted = 0
    while True:
        count = store.delete_expired(before=cutoff, limit=batch_size)
        deleted += count
        if count < batch_size:
            return deleted
```

The production contract also needs bounded retries, timeouts, and a dead-letter path for ingestion failures. A failed embedding call must leave the original ticket usable; it should mark semantic matching as pending rather than block support intake. Never log ticket bodies, vectors, authentication tokens, or unrestricted candidate payloads. Log identifiers, durations, result counts, version labels, and classified error codes with access controls appropriate to the ticket system.

Deletion must finish.

Prove that it does by counting due records, deleted records, retry age, and the oldest expired record still present. Test backup expiration and restoration behavior as well as the live index, because deleting a serving record says nothing by itself about replicas, snapshots, evaluation exports, or a dead-letter payload waiting for replay. The GDPR storage-limitation principle requires personal data to be kept no longer than necessary for its purpose, and NIST describes media sanitization as a program-level process rather than a single database command. Those principles do not dictate a 30-day window; the organization must document the purpose and choose its own defensible period. The trade-off is deliberate: keeping derived text longer can make an incident easier to reconstruct, but it also preserves another copy of customer content after its matching purpose has ended. Prefer structured deletion receipts and aggregate counters over retained payloads.

## Quality and latency share one control panel

Do not tune approximate search parameters in isolation. Build a time-sliced labeled set from agent-confirmed duplicate and non-duplicate pairs, excluding labels that would not have existed at evaluation time. Include terse messages, multilingual tickets, copied templates, changed order numbers, and recurring OTP or notification complaints. Spammy repetition is a particularly sharp test because lexical overlap can be high while the responsible transaction is different.

| Control | Likely quality effect | Likely latency or retention effect |
| --- | --- | --- |
| Longer history | Can recover older recurring issues | More retained vectors and a larger search space |
| Tenant, locale, and category filters | Remove impossible candidates when metadata is reliable | Reduce the eligible set; bad metadata can hide matches |
| Larger candidate count | Gives later rules more alternatives | Adds retrieval and payload work |
| New embedding version | May change neighborhood quality | Requires version isolation and a deliberate re-index plan |
| Human confirmation | Prevents automatic false merges | Adds queue time and review labor |

Evaluate p50 and p95 retrieval latency beside precision at the chosen review threshold, recall within the returned candidate count, no-result rate, confirmation rate, and false-merge reversals. Segment by tenant, language, category, and ticket age. One global average can conceal a weak path for a small locale just as an aggregate SMS delivery rate can conceal one failing destination.

Roll out with shadow queries first: retrieve and measure, but do not show or merge suggestions. Then expose candidates to a limited agent group, retaining an audit record of the query version, filters, candidate identifiers, and decision. An embedding change is a schema migration. Store the version, prevent silent comparison across incompatible vector spaces, backfill deliberately, and compare old and new paths on the same frozen evaluation set.

## What we deliberately stop keeping

When the search-retention window closes, delete the embedding, normalized search text, and disposable index metadata. Keep the authoritative ticket only under the support system's separately documented retention policy; semantic-search convenience does not justify extending it. Also expire evaluation exports and temporary backfill artifacts. They are easy to overlook because they sit outside the serving index.

The cost is real. After deletion, an old ticket cannot be recalled by vector search, historical investigations have less derived evidence, and rebuilding requires the source record to still exist and a compatible embedding process to remain available. A deletion fault may also be harder to diagnose once transient payloads are gone. Accept those limits deliberately: retain aggregate, non-content operational metrics and auditable deletion outcomes, not a shadow archive of customer text.

**The decision rule is simple:** choose the shortest searchable window that meets measured duplicate-recall needs, then widen it only when a labeled evaluation shows that older candidates materially improve the support outcome within the latency budget. Capacity arithmetic sets the guardrail; confirmation data earns an exception.

## Further reading

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- GDPR Article 5, principles relating to processing of personal data: https://eur-lex.europa.eu/eli/reg/2016/679/art_5/oj
- NIST SP 800-88 Rev. 1, Guidelines for Media Sanitization: https://csrc.nist.gov/pubs/sp/800/88/r1/final
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
