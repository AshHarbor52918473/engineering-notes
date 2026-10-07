# Subscription Entitlements: Read Plan Tier Limits Programmatically With Auditable Event Snapshots

Short answer: fetch account entitlements from the subscription system through one backend adapter, cache a versioned snapshot, and record the exact snapshot version with every consequential platform event. During an outage, evaluate against the last verified snapshot under an explicit staleness policy. Do not copy plan limits into application code, and do not retain every raw response forever.

For a B2B SaaS event pipeline, the dominant storage term is usually not the small current entitlement document. It is the repeated decision evidence attached to a high-volume event stream. If one 2 KB snapshot is copied into 50 million event records, the payload alone is about 100 GB before indexes, replicas, or backups. Store the snapshot once under an immutable digest, then put the digest, decision, policy version, and event identifier on each audit record. The change is simple: retention grows with compact decisions, while full snapshots grow only when access actually changes.

## What does the bill contain?

Model the retained bytes before choosing a cache or subscription service. A useful first estimate is `events x decision_record_size + entitlement_changes x snapshot_size`. This separates the high-cardinality term from the low-cardinality term. Measure serialized records from representative data; do not treat the example sizes above as a capacity forecast. Compression, indexes, replicas, and backup retention can change the result substantially.

The decision record needs enough evidence to answer a narrow audit question later: why was event `evt_8f31` accepted when account `acct_204` attempted another export? A compact record can contain the account ID, entitlement key, requested amount, allow or deny result, snapshot digest, policy version, observed snapshot age, event time, and a correlation ID. Avoid secrets, bearer tokens, invoice payloads, and unrelated customer attributes. Retaining less sensitive material reduces both exposure and the scope of later access reviews.

I would keep a short online window for operational investigation and move older decision records to access-controlled archival storage according to the organization's legal and contractual schedule. There is no universal duration. Compliance, dispute windows, customer terms, and incident-response needs determine it. Deletion must cover replicas and backups under their documented lifecycle, not merely the primary table.

## How should a SaaS backend read plan tier subscription entitlements programmatically?

A plan name is poor authorization evidence. Names change, add-ons cross tier boundaries, and negotiated contracts create exceptions. The runtime question is specific: does this account have entitlement `exports.monthly`, what is its effective limit, and which revision supplied that answer?

Use a normalized snapshot shaped around capabilities rather than display labels. The source system remains authoritative, while the backend adapter converts its response into the application's stable contract. Each successful refresh creates an immutable revision. A content digest detects accidental mutation; it is not a substitute for authenticating the source or protecting transport credentials.

```json
{
  "account_id": "acct_204",
  "revision": "rev_1842",
  "effective_at": "2026-10-08T02:15:00Z",
  "entitlements": {
    "exports.monthly": {"enabled": true, "limit": 5000},
    "audit.retention_days": {"enabled": true, "limit": 90}
  }
}
```

The outage rule should be boring and explicit. Low-risk ingestion may continue against a snapshot younger than the approved maximum age. A privilege-increasing action, an unknown entitlement, or an expired snapshot should fail closed or enter a bounded pending state. Do not silently translate a fetch error into an unlimited allowance. Also do not turn every dependency timeout into a denial without considering replay: rejecting an event that cannot be recovered can be more damaging than delaying it.

That trade-off belongs in policy, not in a catch block.

Keep the receipt.

## A focused evaluation boundary

The adapter below uses a generic source interface and a snapshot store. It returns an auditable decision and makes the outage behavior visible. Production code still needs authenticated transport, schema validation, concurrency control, and durable storage, but those concerns should not leak plan constants into business handlers.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from typing import Protocol


class EntitlementSource(Protocol):
    def fetch(self, account_id: str) -> dict: ...


class SnapshotStore(Protocol):
    def latest(self, account_id: str) -> dict | None: ...
    def save_verified(self, snapshot: dict) -> None: ...


@dataclass(frozen=True)
class Decision:
    allowed: bool
    reason: str
    snapshot_revision: str | None
    snapshot_age_seconds: int | None


def may_consume(
    account_id: str,
    entitlement: str,
    used: int,
    requested: int,
    source: EntitlementSource,
    snapshots: SnapshotStore,
    now: datetime,
    max_staleness: timedelta,
) -> Decision:
    try:
        snapshot = source.fetch(account_id)
        snapshots.save_verified(snapshot)
    except (TimeoutError, ConnectionError):
        snapshot = snapshots.latest(account_id)

    if snapshot is None:
        return Decision(False, "no_verified_snapshot", None, None)

    observed_at = datetime.fromisoformat(snapshot["observed_at"])
    age = now.astimezone(timezone.utc) - observed_at
    revision = snapshot["revision"]
    if age > max_staleness:
        return Decision(False, "snapshot_expired", revision, int(age.total_seconds()))

    grant = snapshot["entitlements"].get(entitlement)
    if grant is None or not grant["enabled"]:
        return Decision(False, "not_entitled", revision, int(age.total_seconds()))

    allowed = used + requested <= grant["limit"]
    reason = "within_limit" if allowed else "limit_exceeded"
    return Decision(allowed, reason, revision, int(age.total_seconds()))
```

Catch only failures that the fallback policy understands. Malformed data, signature failures, or schema mismatches should be visible errors, because using an old snapshot in response to an integrity failure can conceal an attack or a broken deployment. Credentials for the source belong in a managed secret store with rotation, narrow access, auditing, and a documented break-glass process. The application log should carry a secret identifier or version when useful, never the secret value.

## Proving the fallback before it matters

Test entitlement changes as state transitions rather than isolated HTTP mocks. Create a verified snapshot, authorize an event, change the upstream limit, refresh, and verify that the next audit record points to the new revision. Then make the source time out. Confirm that a fresh snapshot permits only the capability and quantity it contains, while an expired snapshot produces the configured deny or pending outcome.

Include boundary cases: `used + requested` exactly equals the limit; the entitlement key is absent; an entitlement is disabled while a numeric limit remains; clocks disagree; two refreshes race; and a queued event is replayed after the entitlement changes. The replay rule needs a product decision. Evaluating at original receipt time preserves the historical decision, while evaluating at execution time enforces the current contract. Record both times so the choice can be audited. My default is to favor execution-time checks for privilege increases and receipt-time evidence for explaining ingestion, because those are different questions. Make the distinction explicit in the audit schema. Deployment needs the same care. Introduce the adapter in shadow mode, comparing its decisions with the existing path without enforcing them. Count disagreements by reason, not by account name. Once enforcement begins, alert on snapshot age, refresh failures, unknown keys, denied-event rate, and pending-queue age. A global cache-hit ratio can look healthy while one large tenant has stale access. Per-account cardinality can be expensive, so retain detailed traces for sampled or exceptional cases and aggregate the routine path.

Retries complicate this.

## The retention boundary is part of authorization

Keep immutable entitlement revisions while any retained decision points to them. After the decision records and relevant dispute window expire, delete unreferenced snapshots under the same lifecycle policy. What you deliberately stop keeping is the full upstream response, redundant snapshot copies, transport headers, and transient debug bodies. This lowers storage and exposure, but it has a real cost: an investigation cannot reconstruct fields that were never normalized into the snapshot.

Choose that normalized contract carefully. Preserve effective times, capability values, source revision, verification status, and enough provenance to show which adapter produced the record. Do not preserve data merely because it arrived in the response. Auditability is the ability to explain a decision with controlled evidence, not the ability to replay every byte the dependency ever sent.

The practical decision rule is compact: one authoritative source, one stable adapter contract, versioned snapshots, bounded stale reads, and small append-only decision records. Hardcoded tier names disappear from handlers. Outages become policy decisions with observable outcomes, and access reviews can follow an event back to the exact grant used without retaining a duplicate billing catalog.

## Further reading

- OWASP, Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
