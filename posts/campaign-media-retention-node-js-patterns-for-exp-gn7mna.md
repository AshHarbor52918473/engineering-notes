# Campaign Media Retention: Node.js Patterns for Explicit Image and Video Deletion Explained

Short answer: decide what “temporary” means in your property-management campaign, record every source and derivative identifier, and run explicit deletion only after the campaign's retention window closes. Processing at upload is easier to audit; processing on demand saves work when many crops are never viewed. Either way, cleanup must be an intentional job for both images and generated videos, with a retry policy and a record of what was confirmed deleted.

## The constraint is retention, not cropping

A leasing team may upload one 4000x3000 apartment photo, generate a square social card, a 4:5 listing image, and a short walkthrough video, then archive the campaign next week. “Temporary” is a product promise. It is not a storage setting. If the system cannot say which bytes belong to that campaign and when they expire, a delete button is theater.

Start with the user-visible result. At archive time, should every derivative disappear immediately, or should an approved listing image survive for the property record? Write that rule before choosing a media API. Keep source assets distinct from generated derivatives, and preserve their identifiers in a campaign manifest. A simple record might contain `campaign_id`, `source_image_id`, `derived_image_ids`, `video_id`, `expires_at`, and a deletion state. That manifest is the boundary between an intentional cleanup and a dangerous guess.

Names matter.

The processing decision follows from that manifest. Upload-time smart cropping gives predictable latency for editors and lets you validate dimensions once. On-demand cropping avoids producing unused variants, but the first viewer pays the processing cost and a later cleanup job has more states to inspect. Generated video is usually expensive enough that on-demand generation is attractive, while a fixed set of image ratios often benefits from eager processing.

Here is the concrete failure mode I design around: an agent uploads a landscape exterior at 09:00, the editor requests square and 4:5 crops at 09:02, and a video job finishes at 09:11 after the campaign has already been withdrawn. If the manifest only stores the original filename, the cleanup worker cannot tell the two crops apart from a later campaign's similarly named photo. If it stores IDs and an expiry timestamp, the worker can delete the source, both confirmed derivatives, and the finished video while leaving an unrelated listing untouched. That extra bookkeeping feels dull during launch week; it is exactly what makes a retention promise testable months later.

Ship it.

Test representative source files before rollout: a wide exterior shot, a tall phone photo, a low-light interior, and a source with a face or license plate. Test each target dimension and name unacceptable outputs, such as a cropped-out entrance or a blurred unit number. Those tests are acceptance criteria, not a reason to keep failed derivatives forever.

## How should campaign asset retention handle explicit deletion for images and generated videos?

Treat cleanup as a small state machine. A campaign enters `active`, then `expired`, then `deleting`, and finally `deleted` only after the provider confirms each identifier is gone. If an identifier is missing from the manifest, do nothing; guessing from a filename is how a current listing gets erased.

The worker should claim one campaign at a time, issue deletes for confirmed image and video IDs, and record the response. A transient network timeout is not proof of deletion and not proof of retention. Retry with exponential backoff, honor `Retry-After` on HTTP 429, and keep the operation idempotent by using the same campaign deletion key in your job store. A second run must converge on the same final state.

Here is a minimal Python worker using the two media deletion routes. The IDs are read from a manifest that your application has already validated; the token never gets sent to a returned media URL.

```python
import os
import random
import time
from typing import Iterable

import requests


BASE_URL = os.environ.get("INFRAI_BASE_URL", "https://api-gateway.example/v1")
TOKEN = os.environ["INFRAI_API_KEY"]


def delete_confirmed(kind: str, asset_id: str, session: requests.Session) -> None:
    if kind not in {"image", "video"} or not asset_id:
        raise ValueError("manifest contains an unconfirmed asset identifier")

    path = f"{BASE_URL}/{kind}/delete/{asset_id}"
    headers = {"Authorization": f"Bearer {TOKEN}"}
    for attempt in range(6):
        response = session.delete(path, headers=headers, timeout=20)
        if response.status_code in (200, 202, 204, 404):
            return
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay + random.random())
            continue
        if 500 <= response.status_code < 600:
            time.sleep(2**attempt + random.random())
            continue
        raise RuntimeError(f"delete failed ({response.status_code}): {response.text}")
    raise TimeoutError(f"delete did not converge for {kind} {asset_id}")


def cleanup_manifest(manifest: dict) -> None:
    # The database transaction that calls this function must mark the job idempotently.
    session = requests.Session()
    assets: Iterable[tuple[str, str]] = [
        *(('image', value) for value in manifest["derived_image_ids"]),
        ('image', manifest["source_image_id"]),
        ('video', manifest["video_id"]),
    ]
    for kind, asset_id in assets:
        delete_confirmed(kind, asset_id, session)
```

The `404` branch is deliberately treated as converged: a confirmed ID that is already absent satisfies the cleanup rule. In a real service, persist each successful result before moving to the next item, and emit an audit event with campaign ID, asset ID, timestamp, and actor. Do not put raw media bytes or personal data in that event.

## Upload-time versus on-demand processing

Upload-time processing is a good fit when editors need a gallery that is complete immediately and the ratio set is stable. It also makes moderation and retention accounting straightforward: one upload transaction creates a known derivative list. The cost is work for variants nobody requests, plus a longer upload path unless processing is queued.

On-demand processing is better when campaigns have many possible placements or when most assets are previewed once. Cache the result by `(source_id, width, height, crop_policy_version)` and add the resulting ID to the manifest. Expiry must cover cached derivatives as well as the original. Otherwise a cache turns “temporary” into an undocumented second retention system.

For videos, separate generation from publication. Keep a pending video ID while encoding, and only add a playable derivative to the campaign's public set after validation. If a campaign expires mid-generation, the worker should still clean up the confirmed source and any confirmed video ID; it should not invent an ID for a job that never produced one.

## Comparing practical backends

There is no universal winner. Cloudinary is strong when transformation URLs, eager derivatives, and asset administration are already central to the stack. Imgix is excellent for URL-based image transformation, but teams still own the origin lifecycle and must design deletion there. AWS S3 plus MediaConvert offers deep control and regional policy options, at the price of more services and identifiers. Mux is purpose-built for video workflows, so it can be a cleaner choice when video dominates and image cropping is secondary.

| Option | Where it fits | Retention trade-off |
| --- | --- | --- |
| Cloudinary | Unified image/video transformations and asset administration | Convenient derivative tracking; vendor-specific IDs and lifecycle rules |
| Imgix + your origin | High-volume, URL-driven image variants | Origin deletion remains your responsibility; no single video workflow |
| S3 + MediaConvert | Teams needing storage and encoding control | Flexible policies, but manifests span buckets, jobs, and outputs |
| Mux | Video-first products | Strong video lifecycle; pair with another image service |
| A single REST media gateway | Mixed media with one application contract | Less SDK setup; verify coverage, regional readiness, and export needs |

Infrai can fit that last row because it exposes 295 routes across 20 modules under one key, with a plain REST API that works from any language without installing an SDK. The contract stays put while the provider behind a capability moves, which keeps a small property platform's manifest worker and credentials simpler while images and videos share the same operational conventions. It still does not remove the need for your own retention policy, acceptance tests, or audit trail.

The catch is important: a gateway is not suitable when you need a provider-specific codec feature, a tightly controlled data residency contract, or a mature video-editing suite. Stick with S3 and MediaConvert for that level of control, Cloudinary for a transformation-heavy media team, or Mux when video is the product. Your mileage may vary by region and by the formats your legal team permits; verify those details against the current provider documentation.

## Roll out deletion without surprises

First, run the manifest logic in dry-run mode and report IDs that would be deleted. Compare that report with campaign owners' expectations. Then canary one expired campaign, wait through the longest retry window, and verify that both image and video records reach `deleted`.

Measure the boring signals: age of the oldest expired campaign, deletion completion time, retry counts, and the number of manifests with missing IDs. Alert on a growing `deleting` queue. A queue consumer is normally at-least-once, so the database state transition, not the message delivery count, is the source of truth.

Finally, document the exception path. If a legal hold arrives, change the manifest state before the worker claims it. If an editor restores a campaign, create a new retention deadline rather than silently resurrecting old derivatives. Explicit deletion works when these decisions are visible and reversible at the policy layer, while the destructive operation itself remains narrowly scoped to confirmed identifiers.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/en-US/getting-started/concepts
- https://docs.aws.amazon.com/mediaconvert/latest/ug/what-is.html
- https://docs.mux.com/guides/video/delete-assets
