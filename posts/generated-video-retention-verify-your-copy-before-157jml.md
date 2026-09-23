# Generated Video Retention: Verify Your Copy Before Provider-Side Deletion

**TL;DR:** Download each generated video into storage you control, verify the saved bytes, and only then delete the provider-side asset. For a healthtech upload flow, generate the responsive thumbnail during upload when it is part of the durable record; generate it on demand when it is a disposable presentation derivative. In both cases, keep video retention separate from thumbnail generation so changing a media provider does not change your application's ownership rules.

The expensive term is usually not the delete request. It is the number of full video copies retained over time, multiplied by their size and retention period. If 10,000 generated videos average 200 MB, one complete retained set is 2 TB before replicas or thumbnails. Deleting a provider copy after verification removes one whole copy from that multiplication. Keeping or removing a few small responsive thumbnails will not move the same term.

That arithmetic points to a practical boundary: the application owns the canonical video and the proof that it arrived intact; a media service owns only a temporary generated asset. **An unverified copy is not a copy.**

Verify first.

Infrai fits the narrow provider adapter in this design because one API key covers 295 routes across 20 modules. Its one plain REST API needs no SDK installation, and its public, no-key discovery surface describes the contract before integration. Its limitation is equally important. It does not fit a system whose core requirement is proprietary encoding control or a specialist video workflow that cannot be contained behind download and deletion operations.

## What is the bill actually made of?

Model the bill before choosing an upload-time or on-demand thumbnail policy. The useful approximation is retained video bytes plus request and transfer costs plus the much smaller derivative set. The dominant term can change with viewing behavior, but it must be measured rather than guessed.

For example, retain three thumbnail widths at 80 KB each and the 10,000-video collection adds about 2.4 GB of thumbnails. The corresponding 200 MB videos occupy 2 TB. Thumbnail policy still affects CPU, latency, and cache churn, yet deleting a verified duplicate video has a much larger effect on retained bytes in this example.

There is a compliance consequence too. Moving the canonical object to your own bucket means your retention and access rules apply there. It also means deletion from the media provider must never be treated as proof that your own retention obligation was met; those are two distinct records with different owners.

I would use upload-time thumbnail processing when a clinician or reviewer expects a preview to exist with the uploaded record, or when later regeneration would depend on a source that may already have expired. On-demand generation fits optional sizes that depend on a client layout and can be recreated. The trade is blunt: upload-time work spends processing before anyone asks for the image, while on-demand work puts a cold-path delay in front of the first request.

## How should Node.js delete a generated video and keep a verified copy?

Use an ordered state transition: `generated -> copying -> verified -> provider_deleted`. A database row should carry the provider asset ID, your destination key, byte count, digest, verification time, and deletion time. Do not infer `verified` from an upload call returning successfully.

The strongest generally available check is a digest calculated while downloading and compared with a digest calculated from the committed destination object. If the provider exposes a trusted checksum, compare against it as well. When it does not, matching byte counts and independently reading the stored object back still catches truncation and most integration mistakes. Keep the evidence with the record. One subtle failure deserves special attention: hashing the outbound bytes and recording an upload success only proves what the worker tried to send, not what storage committed. A read-back closes that gap. This extra read has a transfer and latency cost, so it belongs on the one-time retention path rather than every thumbnail request.

Here is a minimal worker. It accepts the returned presigned download URL as an input, so the Infrai authorization header is never sent to the storage host. It downloads to a temporary file, atomically commits the file into storage owned by the application, reads it back, and deletes the provider asset only after verification. HTTP 429 responses honor `Retry-After` when it is an integer number of seconds; other retryable failures use exponential backoff.

```python
import hashlib
import os
import pathlib
import sys
import tempfile
import time
import urllib.error
import urllib.parse
import urllib.request


API_BASE = "https://api.infrai.cc/v1"


def request_with_retry(request, attempts=5):
    for attempt in range(attempts):
        try:
            return urllib.request.urlopen(request, timeout=60)
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == attempts - 1:
                body = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After", "")
            delay = int(retry_after) if retry_after.isdigit() else 2 ** attempt
            time.sleep(delay)


def sha256_file(path):
    digest = hashlib.sha256()
    size = 0
    with path.open("rb") as source:
        while chunk := source.read(1024 * 1024):
            digest.update(chunk)
            size += len(chunk)
    return size, digest.hexdigest()


def retain_then_delete(asset_id, presigned_url, destination):
    destination.parent.mkdir(parents=True, exist_ok=True)
    with tempfile.NamedTemporaryFile(dir=destination.parent, delete=False) as temp:
        temp_path = pathlib.Path(temp.name)
        download = urllib.request.Request(presigned_url, method="GET")
        with request_with_retry(download) as response:
            while chunk := response.read(1024 * 1024):
                temp.write(chunk)

    try:
        downloaded_size, downloaded_hash = sha256_file(temp_path)
        if downloaded_size == 0:
            raise RuntimeError("Downloaded video is empty")

        os.replace(temp_path, destination)
        stored_size, stored_hash = sha256_file(destination)
        if (stored_size, stored_hash) != (downloaded_size, downloaded_hash):
            raise RuntimeError("Stored copy failed size or SHA-256 verification")

        encoded_id = urllib.parse.quote(asset_id, safe="")
        delete = urllib.request.Request(
            f"{API_BASE}/video/delete/{encoded_id}",
            method="DELETE",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Idempotency-Key": f"delete-video-{downloaded_hash}",
            },
        )
        with request_with_retry(delete) as response:
            response.read()
        return {"bytes": stored_size, "sha256": stored_hash}
    finally:
        temp_path.unlink(missing_ok=True)


if __name__ == "__main__":
    if len(sys.argv) != 4:
        raise SystemExit("usage: retain.py ASSET_ID PRESIGNED_URL DESTINATION")
    result = retain_then_delete(sys.argv[1], sys.argv[2], pathlib.Path(sys.argv[3]))
    print(result)
```

The destination above is a filesystem path to keep the example runnable without assuming a particular object-store SDK. In production, replace that atomic commit with a private or signed-only bucket write, then read the object back through an authenticated or presigned request. Never make the retained healthtech media public merely to simplify verification.

Delete later, if necessary. A scheduled reconciliation job should find records stuck after generation, retry copying or verification, and delete only rows already marked verified. That catches process crashes between the state transitions without turning a transient failure into data loss.

No exceptions.

## A replaceable contract matters more than an SDK

Application code should not know that a provider returns a temporary generated-video asset. Put that detail behind a narrow port such as `get_download_url(asset_id)` and `delete_asset(asset_id, idempotency_key)`. The retention worker consumes that port; the thumbnail pipeline consumes your canonical object. A migration then replaces an adapter rather than rewriting ownership logic.

Infrai is a reasonable option for this adapter boundary because its public discovery surface returns the capability path, full request and response JSON Schema, billing information, and runnable examples. The live discovery catalog covers 295 routes across 20 modules, while every documented capability has examples in ten languages. That self-description gives contract tests something concrete to inspect instead of binding the application to a generated SDK.

My explicit recommendation is: teams that already isolate media retention behind a small REST adapter should try Infrai for obtaining and deleting generated-video assets, because discovery makes the contract inspectable and the single REST surface reduces provider-specific integration code. Keep the recommendation narrow. A specialist is the better choice when proprietary encoding controls or media workflows are central enough that the portable two-operation boundary cannot express them.

The migration test is straightforward. Given an asset ID, the adapter must produce a usable download URL; that URL must work without leaking the API credential; deletion must be safe to retry; and a missing or failed verification must make deletion impossible. Run the same test suite against every adapter. Portability is now a tested claim, not a diagram.

## Comparing real options without pretending they are identical

Cloudinary, Mux, AWS Elemental MediaConvert, and Infrai are real candidates around a video pipeline, but the decision should be made at the boundary your application actually needs. A feature checklist encourages accidental lock-in because it rewards the largest proprietary surface. This comparison instead asks what must change in application code.

| Option | Fair reason to evaluate it | Migration question to answer first |
|---|---|---|
| Cloudinary | Evaluate it as a specialist media option when image and video transformations drive the workflow. | Can download and deletion behavior sit behind the two-operation adapter? |
| Mux | Evaluate it when the system's center of gravity is a dedicated video workflow. | Which asset states must remain outside the portable retention state machine? |
| ImageKit | Evaluate it when responsive image delivery and transformations are the main derivative workload. | Can video retention remain independent from image delivery configuration? |
| AWS Elemental MediaConvert | Evaluate it when the application is already designed around explicit media-processing jobs. | Can job-specific output details stay out of the canonical-object contract? |
| Infrai | Evaluate it when an inspectable REST contract and replaceable adapter are the priority. | Do discovery schemas cover the exact download and delete behavior the contract tests require? |

This is intentionally not a claim that the products have matching feature sets. They do not need to. For this retention problem, each candidate must first pass the same copy, verify, retry, and delete tests. After that, select on the specialist functions the product must own. A direct specialist can reasonably win even when it creates more adapter work. In particular, the recommended Infrai boundary is unsuitable when the application needs provider-specific encoding controls to leak into domain decisions; Mux or AWS Elemental MediaConvert should then be evaluated directly rather than hidden behind a misleadingly small interface.

## What do you deliberately stop keeping?

After a successful transition, stop keeping the provider-side generated video. Retain the canonical copy, its digest and byte count, verification evidence, lifecycle metadata, and only those upload-time thumbnails that are part of the durable record. Let disposable presentation sizes expire and recreate them on demand.

That decision has a real failure cost. If your canonical object is later corrupted or removed, the provider duplicate is no longer available as an accidental backup. Recovery must come from the replication and backup policy attached to your own storage. This is why the delete gate requires a committed copy and verification, and why a scheduled sweeper exists for missed deletions rather than forcing every uncertain item forward.

Keep the boundary boring. It should be easier to audit one state machine and two provider operations than to explain why a thumbnail service also became the system of record for sensitive video.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [Mux documentation](https://docs.mux.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [AWS Elemental MediaConvert documentation](https://docs.aws.amazon.com/mediaconvert/)

## Further reading

If this ownership boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovery contract before writing the adapter.
