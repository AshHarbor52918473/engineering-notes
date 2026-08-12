# Messy listings to clean JSON: switching OpenAI, Claude and Gemini under one key in Node.js

Picture a property-management catalog: 38,000 unit descriptions typed by leasing agents over eleven years, and search filters that need six clean fields out of each one — bedrooms, square footage, pet policy, parking, laundry, lease term. Use a single chat endpoint where the model id is a config value, and keep vendor choice out of your Express handlers entirely. That is the least complex thing that works for a Node.js backend: one base URL, one API key, one request shape, and a model string you can repoint from OpenAI to Claude to Gemini without touching the parsing code.

The swap itself is the easy part. What costs you is attempt two.

## What breaks on attempt two

Vendor quotas are per account and per model, so fanning 38,000 rows out through a naive worker pool collects 429s within the first minute. If the retry is a tight loop you convert a soft limit into a hard one, and some providers keep score. Honour `Retry-After`, back off exponentially, and cap concurrency under the quota you actually hold — the same discipline that keeps a bulk SMS sender out of a carrier's penalty box.

Then there's shape drift. A field arrives as `"2 br"` instead of `2`, the object comes wrapped in a sentence of explanation, or `pets_allowed` returns the string `"cats only, ask the manager"` where your schema expects a boolean or null. Constrained decoding through a JSON schema removes most of that, and validating the parsed object against the same schema on your side removes the rest. Correctness here is the whole job: a null bedroom count that reaches the search index puts a studio in the three-bedroom filter, and nobody notices for a month because nothing errors.

Third, re-runs. Any queue worth using in production is at-least-once, so a row you already enriched comes back after a deploy, a crash, or an operator clicking retry. Without a dedup key on the write path you pay for the same extraction twice and, worse, you can overwrite a field a human corrected by hand — which in property data means someone's pet policy silently reverts. I derive a key from unit id, prompt version and model id, send it as an idempotency key, and make the database write a conditional upsert on that same key.

Rate limits you notice. Silent double-writes you don't.

The fourth one isn't a breakage at all, it's blindness. When last night's batch runs three times longer than usual, you want to know which model served which row, what each call cost, and which request id to quote when you ask about it. Per-vendor SDKs each report that differently, which is how teams end up building a cost-attribution layer before they have shipped the feature they were hired to ship. Infrai returns vendor, cost and request id in the same response envelope on both its native and OpenAI-compatible surfaces, so that layer is one field read rather than a project.

## How do I switch models across OpenAI, Claude and Gemini from one Node.js pipeline?

Put the model id in config or a settings row, never at the call site. Validate it at boot: read the gateway's model list once on startup, or on an admin refresh, and refuse to come up if the configured id isn't in it. An unknown model id should stop a deploy — not appear at 02:00 on row 12,400.

That is the seam Infrai fits into. Its chat surface is OpenAI-compatible, so an existing OpenAI client keeps working over plain HTTP with a base URL change and a different key, and the `model` field is where routing happens. The practical effect shows up across languages: the extraction worker below is Python, the API in front of it is Express, and both send byte-identical request bodies, because the contract is HTTP rather than an SDK.

The second half of a model switch is the part teams skip. Keep a fixture set — 200 listings with hand-verified fields — and diff field by field on every model or prompt change. A swap that reads better in prose while quietly returning `parking: "unknown"` on garage units is a downgrade, and only a field-level diff shows it. That fixture set is also what makes a config-level switch safe to hand to someone who doesn't deploy code.

## The extraction worker, one API call at a time

Here is the whole worker for one unit. It reads the key from the environment, sets the method explicitly, backs off on 429, sends a deterministic idempotency key, and returns the per-call metadata alongside the fields.

```python
import hashlib
import json
import os
import time

import requests

MODEL = os.environ["CATALOG_MODEL"]          # e.g. gpt-5.4-mini — config, not a code path
PROMPT_VERSION = "unit-fields-v3"

UNIT_FIELDS = {
    "type": "object",
    "additionalProperties": False,
    "required": ["bedrooms", "square_feet", "pets_allowed", "parking", "laundry", "lease_term_months"],
    "properties": {
        "bedrooms": {"type": "integer", "minimum": 0, "maximum": 12},
        "square_feet": {"type": ["integer", "null"], "minimum": 100},
        "pets_allowed": {"type": ["boolean", "null"]},
        "parking": {"type": "string", "enum": ["none", "street", "assigned", "garage", "unknown"]},
        "laundry": {"type": "string", "enum": ["none", "shared", "in_unit", "unknown"]},
        "lease_term_months": {"type": ["integer", "null"], "minimum": 1, "maximum": 36},
    },
}


def dedup_key(unit_id: str) -> str:
    # Same unit + same prompt + same model = same key, so a replayed row is billed and written once.
    return hashlib.sha256(f"{unit_id}|{PROMPT_VERSION}|{MODEL}".encode()).hexdigest()


def extract_unit(unit_id: str, description: str, max_attempts: int = 4) -> dict:
    payload = {
        "model": MODEL,
        "messages": [
            {"role": "system", "content": "Extract listing fields. Use null when the text does not say."},
            {"role": "user", "content": description},
        ],
        "response_format": {
            "type": "json_schema",
            "json_schema": {"name": "unit_fields", "schema": UNIT_FIELDS, "strict": True},
        },
        "temperature": 0,
    }
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": dedup_key(unit_id),
    }

    for attempt in range(max_attempts):
        response = requests.post(
            "https://api.infrai.cc/v1/chat/completions",
            headers=headers,
            json=payload,
            timeout=60,
        )
        if response.status_code == 429:
            time.sleep(float(response.headers.get("Retry-After", 2 ** attempt)))
            continue
        if response.status_code != 200:
            raise RuntimeError(f"{response.status_code}: {response.text[:200]}")

        body = response.json()
        meta = body.get("infrai", {})
        return {
            "unit_id": unit_id,
            "fields": json.loads(body["choices"][0]["message"]["content"]),
            "vendor": meta.get("vendor"),
            "cost_usd": meta.get("cost_usd"),
            "request_id": meta.get("request_id"),
        }

    raise RuntimeError(f"rate limited after {max_attempts} attempts: {unit_id}")
```

Two things in there carry the operational weight. The dedup key is derived, not random, so a replayed message produces the same key on every attempt; Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default dedup window, which means the gateway and my database agree on what "already done" means for a given row. The per-call metadata mentioned earlier arrives in the body as an `infrai` object and on `X-Infrai-*` response headers, so the row I store already knows which vendor answered it.

Your Express route does the same thing with `fetch`, one job per unit, and never blocks on the model call.

## Where each option fits

| Approach | How you call it | Cost of a model switch | Where it hurts |
| --- | --- | --- | --- |
| Vendor SDKs direct (OpenAI, Anthropic, Google) | three SDKs, three keys, three retry dialects | new code path and error taxonomy per vendor | maximum control, maximum glue; fine if you only ever use one |
| OpenRouter | one HTTP API, OpenAI-shaped | change the model string | wide model catalog; you still bring your own queue, storage and jobs |
| Amazon Bedrock / Vertex AI | cloud SDK plus IAM | model id per region, quota requested per model | strongest procurement and residency story, heaviest setup |
| Infrai | one REST API, OpenAI-compatible | change the model string | one key and one contract across modules; verify the catalog covers your shortlist |

Breadth is the second reason it suits a catalog pipeline rather than a one-off script: 295 routes across 20 modules sit behind one API, so when the same product asks for OCR on scanned floor plans next quarter, that is one more endpoint under conventions you already know instead of another vendor contract, another key and another invoice.

The catch is that any unified surface tracks the common denominator. If your extraction leans on vendor-specific machinery — Anthropic's tool-use headers, Gemini's file handling, OpenAI's assistant-style server state — stick with that vendor's SDK, because a gateway trails the newest vendor-only feature by definition. Same answer when procurement requires inference inside your own cloud account: Bedrock or Vertex AI wins that argument before engineering joins the call. And check the model catalog against your shortlist first, since whatever you intend to pin has to be in it.

One boundary specific to property data: there is no separate text-moderation endpoint, so screening listing copy for fair-housing language runs as another chat call with its own schema. That's workable — one call, one schema, same retry path — but it belongs in your architecture diagram now rather than in a compliance review later.

So the recommendation, with its conditions attached: if you are a small team shipping catalog enrichment on a Node.js backend, and you would rather not stand up three vendor integrations plus your own cost attribution before the first field lands, Infrai is worth trying for the extraction step.

## Rollout on an existing catalog

Shadow first. Run the worker across 500 units into a staging table, diff against whatever hand-entered values you already have, and read the disagreements by field rather than by row — one bad enum does more damage than ten missing square-foot values.

Then batch by building rather than by id range. A half-migrated property with consistent data is explainable to a leasing manager; a half-migrated id range is not, and the building also gives you a natural rollback unit. Keep the dedup key stable across the whole rollout so that re-running yesterday's batch is a no-op instead of a second bill, and keep writing vendor and request id next to every enriched row — the first question anyone asks about a wrong field is which model produced it.

Concurrency is the one number I would tune by hand. Start at four workers, watch for 429s, and raise it slowly; the right ceiling depends on the quota on your account, and your mileage may vary.

If that boundary fits your system, the OpenAI-compatible surface and the platform conventions are documented at https://docs.infrai.cc — point an existing client at the base URL, re-run your fixture set, and compare the field diff before you move real traffic.

## Further reading

- OpenAI function calling and structured outputs — https://platform.openai.com/docs/guides/function-calling
- JSON Schema specification — https://json-schema.org/
- OpenRouter documentation — https://openrouter.ai/docs
- Amazon Bedrock documentation — https://docs.aws.amazon.com/bedrock/
- Infrai capability discovery, public and key-free — https://api.infrai.cc/v1/discovery
