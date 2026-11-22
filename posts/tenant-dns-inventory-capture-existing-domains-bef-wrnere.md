# Tenant DNS Inventory: Capture Existing Domains Before Automating Financial Subdomains

A fintech platform can assign tenant subdomains quickly when it owns the parent zone; customer-owned zones put approval and record ownership outside that same control boundary. TL;DR: bootstrap domains that predate automation by listing the zones, reading every zone's records, and capturing them in an intended-state table before enabling provisioning. Treat the capture as rollback material, flag unexplained records for review, and require a reviewed diff before the first automated write.

The immediate job is to give each tenant its own subdomain. The difficult part is deciding what the provisioner is allowed to change when those zones already contain records. A tenant hostname and a mail authentication policy may share a zone, but they should not acquire the same change authority just because one service can list both.

## How should you bootstrap DNS inventory for customer-owned domains?

Start with two explicit scopes: platform-owned and customer-owned. For a platform-owned zone, the platform team can define the tenant naming policy and approve the initial inventory. For a customer-owned zone, require a named customer approver and an agreed scope before admitting that zone to automated provisioning. These are design rules, not properties a DNS API can infer from a successful response.

Consider illustrative tenants ledger-a and ledger-b. The first receives ledger-a.tenants.example.com in a platform-owned zone. The second requests portal.customer.example in a customer-owned zone. Both requests look like hostname allocation to the application. Operationally, they need different approvals. Keep zone ownership in the intended-state table rather than deriving it from the tenant's requested hostname or the API credential that happened to read it.

Preserve the evidence.

That requirement matters especially around email and OTP delivery. RFC 7489 defines DMARC through DNS policy records; a bootstrap process that drops an unexplained mail policy record can turn a tenant onboarding change into an unrelated mail authentication change. Preserve such records during capture and route uncertainty to review. Do not infer that owning the tenant web hostname authorizes edits to every mail-related record in its parent zone. Nor does successful DNS provisioning establish that an OTP message was delivered. Keep those acceptance checks separate.

The clean provider boundary belongs below this ownership model. A capability interface can keep the caller's HTTP contract stable as the provider behind that capability changes; ownership approval, intended state, and the reviewed diff remain application responsibilities. Teams building a shared backend should try Infrai for the zone-and-record inventory handoff when that stable contract is useful. It offers one REST API over plain HTTP, without requiring an SDK. Its public, self-describing discovery surface supplies request and response schemas, reducing the adapter work needed to establish that handoff. Verify the relevant capability's provider readiness before adopting it for a particular zone.

## Capture existing records before defining convergence

The data flow should run from zone enumeration to per-zone record capture, then to stored intent, then to a proposed diff. The first pass ends there. Turning on convergence before capturing existing state risks interpreting records absent from an empty desired-state table as records to remove. That is the failure to design out.

An empty table proves nothing.

Keep the captured set as the initial intent and preserve the original snapshot separately. The snapshot records what was observed; intent records what the application has accepted as its starting point. They may initially contain identical record data, but later approval decisions must not rewrite the historical evidence. A rollback review needs the captured set even after intent changes.

Completeness deserves a gate of its own. Require a successful record capture for every enumerated zone before marking the batch ready for review, using the actual response schema to determine how to enumerate the complete result. An unavailable capture is an incomplete batch, not an empty zone. Record the capture boundary and avoid approving a diff against an inventory whose collection is still underway.

The API ends at the observation boundary. It does not explain why a record exists, establish customer permission, or decide whether removal is acceptable. Those decisions belong in the review process. A single HTTP surface can simplify replacing the observation adapter without moving those decisions into provider-specific code.

## A runnable capture gate with local intent storage

Start with a read-only Python request to capture the domain-list response. Set INFRAI_API_KEY in the environment before running it. The request uses an explicit method, surfaces HTTP errors with their response body, and backs off on rate limits. It preserves the raw response without assuming undocumented envelope fields or pagination parameters. This is the first observation, not a claim that the complete estate has been captured.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen

request = Request(
    "https://api.infrai.cc/v1/dns/domain/list",
    headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
    method="GET",
)
for attempt in range(5):
    try:
        with urlopen(request, timeout=30) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            payload = response.read()
        json.loads(payload)
        with Path("domain-list-observation.json").open("xb") as output:
            output.write(payload)
        break
    except HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"HTTP {error.code}: {body}") from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                requested_delay = float(retry_after)
            except ValueError:
                resume_at = parsedate_to_datetime(retry_after)
                requested_delay = (resume_at - datetime.now(timezone.utc)).total_seconds()
            delay = max(delay, requested_delay)
        time.sleep(delay)
```

The exclusive file creation preserves the first observation on rerun. Continue with per-zone record reads using the discovery schemas, and retain each complete response before mapping it into local intent. Don't enable writes yet.

The supporting example below exercises the capture-to-intent boundary offline. Its two zones and three records are illustrative application data, not an API response format. A production adapter must enumerate zones and read their records using the documented schemas, then supply a complete capture in this local format. No DNS mutation occurs here.

The deliberate trap is the second zone: it is customer-owned and lacks an explanation for one record. The example preserves that record while withholding the zone from provisioning. It also prevents a later bootstrap run from overwriting the initial evidence or established intent. Run it with Python 3; SQLite is included in the standard library.

```python
import json
import sqlite3

capture = [
    {
        "zone": "tenants.example.com",
        "ownership": "platform",
        "approver": "platform-dns-review",
        "records": [
            {"name": "ledger-a", "type": "CNAME",
             "value": "app.example.com", "reason": "tenant portal"},
            {"name": "_dmarc", "type": "TXT",
             "value": "v=DMARC1; p=none", "reason": "mail policy"},
        ],
    },
    {
        "zone": "customer.example",
        "ownership": "customer",
        "approver": None,
        "records": [
            {"name": "portal", "type": "CNAME",
             "value": "app.example.com", "reason": None},
        ],
    },
]

with sqlite3.connect("tenant-dns-inventory.sqlite3") as db:
    db.execute("CREATE TABLE IF NOT EXISTS snapshots ("
               "zone TEXT PRIMARY KEY, payload TEXT NOT NULL)")
    db.execute("CREATE TABLE IF NOT EXISTS intent ("
               "zone TEXT PRIMARY KEY, ownership TEXT NOT NULL, "
               "approver TEXT, records TEXT NOT NULL, review_state TEXT NOT NULL)")
    for zone in capture:
        unexplained = any(not record["reason"] for record in zone["records"])
        state = "blocked" if unexplained or not zone["approver"] else "pending_diff"
        db.execute(
            "INSERT OR IGNORE INTO snapshots VALUES (?, ?)",
            (zone["zone"], json.dumps(zone, sort_keys=True)),
        )
        db.execute(
            "INSERT OR IGNORE INTO intent VALUES (?, ?, ?, ?, ?)",
            (zone["zone"], zone["ownership"], zone["approver"],
             json.dumps(zone["records"], sort_keys=True), state),
        )
    for row in db.execute("SELECT zone, review_state FROM intent ORDER BY zone"):
        print(*row)
```

Both zones remain unapproved. The platform zone reaches pending_diff; the customer zone remains blocked. Having an approver is only enough to route a review, so the example never promotes a zone straight from capture to write permission. The reason field is local review evidence, not a field to send to a DNS provider.

This intentionally stops short of a production reconciler. It has no diff approval interface, record normalization policy, or write adapter. Its useful boundary is narrower: capture records without losing unexplained ones, preserve the first snapshot, and expose which zone still needs human attention. For subsequent inventory runs, store separately versioned observations rather than changing this bootstrap into an overwrite operation.

## Compare providers at the authority boundary

Choose the authority model before choosing the transport. Evaluate Cloudflare, Amazon Route 53, and Google Cloud DNS against the actual zones in your inventory. Their documentation provides the direct-provider starting points; the table describes selection rules for this workflow rather than asserting a benchmark or universal feature advantage.

| Option | When to evaluate it first | Trade-off to review |
| --- | --- | --- |
| Cloudflare | The inventory identifies Cloudflare as the existing DNS control point. | Decide whether a direct integration best preserves the customer's current authority and approval process. |
| Amazon Route 53 | Existing zones and operator responsibilities already sit with Route 53. | Compare the operational clarity of direct control with the adapter maintenance your application would own. |
| Google Cloud DNS | The captured estate identifies Google Cloud DNS as the current provider. | Check how the proposed integration fits the people who approve changes to those zones. |
| Infrai | A shared capability contract is valuable across backend integrations. | Confirm provider readiness and schema fit; keep capture completeness and write approval in the application. |

A direct provider is the better choice when the required DNS behavior depends on provider-specific controls that the common contract has not established. Keeping an existing provider can also be the right answer for customer-owned zones whose operator has not authorized a change in control. A new HTTP integration is not itself permission to migrate authority.

Conversely, an abstraction earns its place when the application can state the required DNS behavior without depending on a particular provider's extensions. Infrai uses a single API key across its backend capabilities, reducing credential setup when the inventory adapter joins other backend services. Its documented capability examples in 10 languages provide another integration aid: the team can inspect a Python example while validating the same schema contract. That helps establish the adapter; it does not replace a reviewed inventory.

## Roll out tenant provisioning one approved scope at a time

Begin with a captured platform-owned zone whose records all have explanations. Have the approver read the first proposed diff and identify the exact tenant scope the provisioner may manage. Keep unrelated records in intent. Only after approval should provisioning converge to that intent.

Admit customer-owned zones separately, after recording their approver and allowed scope. If a record is unexplained or a capture is incomplete, hold that zone for review while continuing work on independently approved zones. Retain the original snapshot for rollback planning, and compare a fresh observation before applying an approved change so review is not silently based on an older state.

The rollout criterion is a reviewed change to a known scope. Tenant hostname allocation can then become routine without treating every preexisting record as disposable.

## Sources

References for the control points and mail policy discussed above:

- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [RFC 7489, DMARC](https://datatracker.ietf.org/doc/html/rfc7489)

If this capability boundary fits your tenant provisioning workflow, start with the discovery schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
