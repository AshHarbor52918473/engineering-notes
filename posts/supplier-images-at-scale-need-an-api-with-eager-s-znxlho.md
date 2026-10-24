# Supplier Images at Scale Need an API (With Eager Smart Crops)

**TL;DR:** To import supplier images at scale, make the API persist each source once, dedupe and normalise it, then batch the required property-listing crops before publication. Store an immutable master plus crop intent. Reserve on-demand rendering for uncommon sizes and later policy changes; this split keeps duplicate detection ahead of expensive work while preventing a transient renderer failure from leaving a new listing without its primary photo.

The constraint is not raw image conversion. It is preserving what a leasing agent meant to show when the same kitchen photo appears as a 4:3 search tile, a 16:9 gallery frame, and a 1:1 syndication thumbnail. A face detector or saliency model can suggest a focal region, but the import contract must retain the original and make every derived crop reproducible. Never let the first derivative become the master.

## How should an API import supplier images at scale?

Upload-time processing gives publication a crisp invariant: every active property has the required variants, or activation waits. On-demand processing weakens that invariant because the first viewer can become the unlucky caller who discovers a corrupt source, a decoder limit, or a missing crop decision.

There is still a place for lazy work. A partner may request an unusual ratio months later, and rebuilding every possible width during intake creates storage and compute with no reader benefit. **Eagerly create required ratios; lazily create the long tail from the immutable master.** Cache those lazy results under a transformation-version key so a new crop policy cannot silently reuse an old file.

The limitation is extra storage and work for images that never reach publication. A small catalogue with unpredictable ratios may fit pure on-demand rendering better; a bulk property feed with three mandatory placements usually does not.

This is also a compliance boundary. Supplier authorization, retention class, and takedown state belong to the asset record, not to a filename copied into several folders. Deleting or suppressing an image must fan out to derivatives and caches. Property photos are less sensitive than OTP content, but the operational lesson is similar: delivery success does not erase consent, expiry, or audit requirements.

## Make intake a state machine, not a download loop

Treat the supplier feed as claims. Its URL, MIME label, dimensions, and external identifier are useful routing data, not proof. Fetch through an outbound service with allowlisted schemes and network ranges, cap bytes while streaming, and decode in an isolated worker. OWASP's SSRF guidance explains why accepting arbitrary remote URLs without network controls is dangerous; image parsing also belongs outside the API process that coordinates the batch.

The durable record should move through explicit states such as `received`, `validated`, `deduplicated`, `normalized`, `cropped`, and `ready`, with a terminal rejection state carrying a stable reason code. Keep the source object before transformation. Record the detected media type, decoded dimensions, orientation handling, hashes, crop model version, transform version, and each output object's checksum.

Ordering matters:

1. Stream the remote bytes into quarantine while calculating a cryptographic source hash.
2. Verify the decoded format and dimensions, apply orientation, and convert into a canonical pixel representation.
3. Hash that representation to catch files whose containers or metadata differ but whose displayed pixels match.
4. Choose or validate one focal region, then derive the required aspect ratios.
5. Publish the asset record only after every mandatory output is durable.

Retries happen.

A byte hash is exact and cheap, but metadata changes defeat it. A normalized-pixel hash catches those container-level variations, yet it should not be confused with perceptual similarity. Two resizes or recompressions may still represent the same room while producing different pixels. Perceptual matching can flag candidates for review; using it as an automatic identity key risks merging distinct photos with similar layouts. That is a bad trade in a property catalogue, where two beige bedrooms can look remarkably alike. I would accept a missed automatic match before accepting a false merge, because the former spends processing capacity while the latter can attach the wrong room evidence to a listing.

## Put identity and retries in the API contract

Batch endpoints should accept client-generated item keys and return per-item outcomes. One malformed panorama must not roll back 499 valid records, and retrying a timed-out request must not create another asset. HTTP defines idempotent methods, but an import operation still needs an application-level key because job creation is commonly expressed as a non-idempotent request. A Node.js API can own this coordination even when the isolated media workers use another runtime; the contract and state transitions matter more than sharing a language across every process.

The following Python sketch keeps orchestration separate from image implementation. The repository methods are conditional writes; the queue carries identifiers, never image bytes.

```python
from dataclasses import dataclass
from hashlib import sha256
from typing import BinaryIO, Protocol


@dataclass(frozen=True)
class ImportItem:
    supplier_id: str
    property_id: str
    source_url: str
    idempotency_key: str


class IntakePorts(Protocol):
    def fetch_limited(self, url: str, byte_limit: int) -> BinaryIO: ...
    def store_quarantine(self, stream: BinaryIO) -> tuple[str, str]: ...
    def create_once(self, key: str, values: dict) -> tuple[str, bool]: ...
    def enqueue(self, asset_id: str) -> None: ...


def accept(item: ImportItem, ports: IntakePorts) -> dict:
    stream = ports.fetch_limited(item.source_url, byte_limit=25_000_000)
    object_key, source_digest = ports.store_quarantine(stream)
    identity = sha256(
        f"{item.supplier_id}\0{item.property_id}\0{item.idempotency_key}".encode()
    ).hexdigest()
    asset_id, created = ports.create_once(identity, {
        "property_id": item.property_id,
        "source_object": object_key,
        "source_sha256": source_digest,
        "state": "received",
    })
    if created:
        ports.enqueue(asset_id)
    return {"asset_id": asset_id, "accepted": created}
```

The 25,000,000-byte value is an example policy limit, not a universal recommendation. Put the real limit in configuration, version it, and reject over-limit streams before full buffering. More important, make `create_once` atomic. A read-then-insert sequence can race when the batch client retries while the original request is still completing.

Workers should checkpoint stage outputs with compare-and-set transitions. Retry network fetches and temporary storage errors with bounded backoff, but send deterministic failures such as an unsupported decoded type to a review queue immediately. Log asset IDs and reason codes. Do not log signed source URLs; they can carry credentials.

## Smart crop intent belongs beside the master

Store crop intent in source coordinates, preferably as a focal point plus an optional protected rectangle. Normalized coordinates from 0 through 1 survive source-resolution changes and make the record independent of a particular output size. The renderer can then compute the largest rectangle of the requested ratio that stays within the image while favoring the protected region.

Automation needs a human override. Leasing teams care about ovens, balconies, accessibility features, and floor-plan labels that generic saliency can miss. An override should create a new crop-intent version and regenerate derivatives; it should not destructively edit the source. Keep the automatic proposal too, since it helps explain why an earlier crop was produced.

Format normalization, or normalise in a feed contract that uses British spelling, is a policy decision rather than a synonym for converting everything to one extension. MDN documents different browser support and characteristics across JPEG, PNG, WebP, AVIF, and other image types. Select output formats from the clients you actually support, retain alpha only where the catalogue allows it, and test color and orientation using decoded results rather than trusting filename suffixes.

The acceptance suite needs awkward fixtures: portrait images with orientation metadata, transparent logos accidentally placed in photo slots, very wide panoramas, truncated files, animated inputs, duplicate pixels with changed metadata, and a protected rectangle touching each edge. Tiny images deserve an explicit policy too. Upscaling can satisfy dimensions while making a listing look worse.

## Compare the two processing moments honestly

| Concern | Upload-time derivatives | On-demand derivatives |
|---|---|---|
| Listing activation | Can require all standard crops first | May expose a missing variant on first view |
| Policy changes | Requires a controlled rebuild | Can apply at the next cache miss |
| Compute shape | Predictable batch work after intake | Traffic-driven bursts and cache coordination |
| Storage | Holds known variants in advance | Holds only requested variants plus the master |
| Failure ownership | Import workflow records the failure | Read path must define fallback behavior |

For this catalogue, the first column wins for 4:3, 16:9, and 1:1 because those ratios are publication requirements. On demand wins for uncommon partner dimensions. The trade-off is explicit: the upload-time approach consumes capacity before demand is known, while the on-demand approach moves failure and latency into the read path. Mandatory presentation should not depend on request-time side effects.

Observe the pipeline by stage and reason: queue age, time in state, retry count, rejected-byte count, duplicate decisions, manual crop overrides, and derivative verification failures. Alert on stuck work rather than a single end-to-end average. A healthy mean can hide one supplier partition that has stopped progressing.

Cost belongs in capacity planning, not in the headline. Measure decoded megapixels and output pixels in addition to file count because a batch of panoramas is not equivalent to the same number of thumbnails. Set concurrency from measured memory and decoder behavior, then isolate suppliers with per-tenant limits so one oversized feed cannot starve routine listing updates.

## Roll out without recropping the catalogue blindly

Start in shadow mode: ingest masters, calculate hashes and crop proposals, but leave the current published images untouched. Compare duplicate decisions and have property staff inspect a deliberately varied sample across room types and source orientations. Define rejection and override thresholds before promotion.

Next, enable the new path for one bounded supplier partition. Backfill by immutable asset ID, record the transform version, and switch reads only after required outputs pass decode and checksum verification. Keep rollback simple: the listing points back to the prior derivative set while the master and audit record remain intact.

Finally, expand in batches with pause controls. **The migration unit is an asset record, not a folder of files.** That makes retries, takedowns, crop corrections, and later format changes traceable without guessing which filename was authoritative.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://www.rfc-editor.org/rfc/rfc9110.html
- https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- https://c2pa.org/specifications/specifications/2.2/specs/C2PA_Specification.html
