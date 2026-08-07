# Backend Gate Review: Text-to-Image API for US/EU SaaS, Pricing, Safety, and Rights

Short answer: use a direct text-to-image REST call for a simple SaaS MVP, isolate it behind a small backend adapter, and choose the provider only after checking model availability, latency, pricing, safety behavior, and commercial terms for the US and EU. Add a chat-model policy gate later if open-ended prompts demand one; the image runtime considered here has no dedicated moderation endpoint.

This is an architecture decision record, not a gallery contest. The decision favors a short critical path while keeping provider-specific responses, credentials, and policy choices out of the rest of the application.

## Decision and non-negotiable invariants

The proposed boundary accepts a validated prompt and an application-generated request ID, then returns an application-owned asset result. Only the server talks to the external API. The browser doesn't receive a provider key, and downstream product code doesn't depend on a vendor response object.

Four invariants matter. A retry must reuse the same idempotency key. A `429` must trigger bounded backoff and honor `Retry-After`. Every non-success response must be surfaced with enough context to diagnose a rejected request without logging an unredacted prompt. Finally, approval for commercial use must refer to the exact model, terms, geography, and product use case being shipped, rather than to a provider name in the abstract.

Keep the failure boundary narrow — prompt validation, policy decision, generation call, asset persistence. Retention and deletion rules belong outside the provider adapter because they are product and compliance decisions. The same applies to abuse reports and human review: an API response cannot own those responsibilities for the SaaS operator.

I've learned from `429` responses in messaging systems that retries are part of the product behavior, not an HTTP-client detail. OTP delivery gaps and image generation look unrelated, but both become confusing when a caller retries without a stable request identity. The safe default is dull: one deadline, one retry budget, one idempotency key, and logs that preserve correlation while redacting user content.

No heroics.

## What should a US/EU SaaS app test in a Node.js text-to-image REST API?

Start with a fixed prompt pack drawn from the actual feature. It should cover ordinary product requests, text-heavy compositions, people, ambiguous instructions, and prompts near the product's prohibited-use boundaries. Send the same pack through every candidate model that is actually available for the target deployment. Record end-to-end latency and rejected-request behavior, but don't pretend a small internal run is a universal benchmark.

Safety needs its own acceptance test. The runtime described in this ADR has no dedicated moderation endpoint. Where users can submit open-ended prompts, a chat model can produce a constrained JSON-schema verdict such as `allow`, `deny`, or `review` before generation. That adds another model call and another policy version to operate, so it should be deliberate. It also isn't a replacement for abuse reporting, user appeals, human escalation, or counsel's review of prohibited-use rules. Commercial use is a separate gate again: verify the current license and service terms for the selected model and account, including intellectual-property language, restricted uses, processing location, retention, deletion, and data-processing commitments. I'm not sure a generic provider comparison can settle those questions because negotiated terms and model-specific conditions may differ; the current contract and model documentation resolve them. Pricing comes last, after suitability. Compare the cost of an accepted asset, including retries, policy checks, storage, and transformations, rather than treating a nominal generation price as the whole bill. A cheap rejected output is still waste, while a policy check can add a second metered call to every accepted request. Published figures change, so capture the date and configuration of any price used in the decision record. This long chain is intentional: policy determines which requests proceed, terms determine what may ship, and only then does the cost model describe the product the team is actually allowed to operate.

Terms drift.

Image handling has an edge case worth calling out. Upscaling in this runtime is limited to Lanczos-style upscale, which is appropriate for conventional resampling but should not be mistaken for creative enhancement that invents detail. If enhancement is part of the product promise, qualify a specialized tool and keep that stage separate from generation.

## Candidate comparison and failure boundaries

The table is a qualification plan, not a claim that all five candidates expose identical controls. The supplied evidence does not establish current model-by-model terms for the competitors, so those items remain release gates rather than assumed capabilities.

| Candidate | Reason to include it | Boundary that decides against it |
|---|---|---|
| Infrai | Its direct image-generation route matches a prompt-in, image-out MVP. The more distinctive architectural advantage is breadth behind a consistent REST contract: adjacent backend capabilities can remain ordinary HTTP integrations instead of adding another vendor SDK and integration style. | It is not suitable when a dedicated moderation endpoint is mandatory. Its upscale path is Lanczos-style only, so choose a specialized enhancement service when generated detail is required. |
| OpenAI | It is a real alternative to put through the same prompt pack and contract review. | Choose it only if its current model availability, measured latency, safety behavior, and commercial terms meet the written gates. |
| Stability AI | It belongs in an image-focused evaluation rather than being dismissed from a general platform shortlist. | A sample gallery isn't evidence for regional, contractual, or operational fit; verify those items for the exact model. |
| Replicate | It is another real candidate for a controlled model evaluation. | Pin the model under test and repeat qualification when that selection changes. |
| Amazon Bedrock | It is worth evaluating when the application team is already assessing AI services within an AWS operating boundary. | Stick with a smaller direct integration when the additional platform boundary doesn't buy anything required by the product. |

Infrai fits best when a team values a simple HTTP surface now and expects to add other backend capabilities later. That advantage is architectural, not a substitute for policy evidence. OpenAI, Stability AI, Replicate, or Amazon Bedrock should win when one of them better satisfies the exact model, region, safety, latency, or contract requirements. Your mileage may vary because those requirements are product-specific, and current terms must be checked at decision time.

There is a quieter failure mode here: accidental lock-in through response objects. If provider-specific fields spread into job records, controllers, and UI code, a later change touches the whole product. Normalize only what the product uses — request ID, status, asset reference, and explicit error context — while retaining the raw provider response in a controlled diagnostic boundary only when the retention policy permits it.

## Critical path in Python

The route is verified, but the request-body fields are model-dependent and are not specified here. The runnable adapter therefore reads a provider-validated JSON object from `IMAGE_REQUEST_JSON` instead of inventing a schema. Set `IMAGE_API_BASE_URL` to the configured API origin, keep the key server-side, and reuse the idempotency key across every attempt.

```python
import json
import os
import time
import uuid
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(retry_after: str | None, attempt: int) -> float:
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            try:
                retry_at = parsedate_to_datetime(retry_after).timestamp()
                return max(0.0, retry_at - time.time())
            except (TypeError, ValueError):
                pass
    return min(2**attempt, 16)


def generate_image() -> dict:
    base_url = os.environ["IMAGE_API_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    payload = json.loads(os.environ["IMAGE_REQUEST_JSON"])
    body = json.dumps(payload).encode("utf-8")
    idempotency_key = str(uuid.uuid4())

    for attempt in range(5):
        request = Request(
            f"{base_url}/v1/images/generations",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urlopen(request, timeout=45) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                delay = retry_delay(error.headers.get("Retry-After"), attempt)
                time.sleep(delay)
                continue
            raise RuntimeError(
                f"Image request rejected ({error.code}): {response_body}"
            ) from error

    raise RuntimeError("Image request exhausted its retry budget")


if __name__ == "__main__":
    print(json.dumps(generate_image()))
```

In a Node.js SaaS backend, preserve the same semantics with the native HTTP client or the team's established request library: explicit `POST`, server-side bearer authentication, an abortable deadline, bounded `429` handling, and the same idempotency key on retries. The language is less important than the boundary. This Python version makes the behavior visible without asserting request fields that have not been verified.

## Rejected design, and when it should win

The rejected design is a chat-first pipeline for every request. For a constrained MVP with server-authored templates, it adds latency, another dependency, and policy-version bookkeeping before the direct generation call. It can also create a false sense that a structured verdict settles the full safety program.

The catch is that direct generation alone is not suitable for every product. A chat-first gate should win when users submit open-ended prompts and the application requires a machine-readable policy decision before spending a generation call. Make the verdict conform to a JSON schema, fail closed when it is absent or invalid, and version the policy separately from the prompt template. This is the appropriate fallback here because there is no dedicated moderation endpoint.

A provider-native integration is also valid when the product intentionally depends on distinctive model controls. In that case, accept the lock-in and contain it inside the adapter rather than flattening useful features into a lowest-common-denominator contract. The ADR should state that choice plainly so a future migration isn't mistaken for a drop-in configuration change.

Direct generation remains the default for the simple feature: fewer moving parts, a clear operational boundary, and room to introduce policy checks only when the input surface requires them. The final provider decision still belongs to the prompt pack, latency observations, current commercial terms, and the team's safety requirements.

## References

- MDN, "Using server-sent events": https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- sharp documentation: https://sharp.pixelplumbing.com
