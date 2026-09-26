# Web Search and Continuous Crawling for Marketplace Research Explained

TL;DR: Use a web search API for breadth, and operate your own crawl only for the small set of marketplace sources whose pages must be compared repeatedly. Normalize and deduplicate URLs before indexing either stream, then perform semantic near-duplicate detection after retrieval. This split protects coverage without forcing every query to wait for a crawler you cannot keep comprehensive.

The primary trade-off is retrieval quality versus latency. Search reaches the long tail that a maintained crawl will miss; a focused crawl gives you repeatable snapshots of the same pages. Treating either one as the whole acquisition strategy sacrifices one side of that trade-off.

## Should you choose between a web search API and your own crawl?

A market-research corpus has two different jobs hiding inside it. One is discovery: find unfamiliar sellers, category pages, policy changes, and listings that did not exist when the last crawl plan was written. The other is monitoring: revisit a known set of pages so that changes can be compared over time.

Search fits discovery because it covers the long tail you cannot realistically maintain a crawler for. Your own crawl fits monitoring because you control which known pages return on each run. The crawler's value is not theoretical completeness. It is continuity.

That distinction matters when the product must detect near-duplicate marketplace records. Two sellers may reuse a description while changing a title, tracking parameter, or category path. Search can reveal both records, but it does not promise that the same result set will appear on every research run. A scheduled crawl of selected sources gives you the stable observations needed for longitudinal comparison.

Keep the boundary narrow.

Every extra domain brings scheduling, politeness, parsing, and change-detection work, while still doing nothing for the unknown sites outside the list. Consider a seller page found by search on Monday, then found again through a tracking crawl on Tuesday with `utm_source` appended. Indexing both copies wastes retrieval space and can make one listing look like two independent observations. Canonical URL equality should collapse that pair before embedding, while the Tuesday observation time remains available for genuine content-change analysis. This is a small distinction with a large audit consequence: acquisition duplication and semantic duplication are different events.

## Deduplication belongs before and after retrieval

Both acquisition streams need URL deduplication before anything reaches the index. Strip fragments, normalize host casing, remove tracking parameters you have explicitly classified as non-identifying, and sort the remaining query parameters. Do not blindly delete every query string: on many marketplaces, a product identifier lives there.

URL equality only removes exact acquisition duplicates. Semantic near-duplicate detection is a later retrieval concern: embed or otherwise represent the cleaned record, retrieve likely neighbors, and compare them using a threshold calibrated on marketplace examples. A high threshold reduces false merges but leaves more copied listings separate; a lower threshold catches paraphrases and risks joining distinct variants. There is no honest universal cutoff in the supplied evidence, so label a representative set and choose the threshold from your own precision and recall results.

Use stable marketplace identifiers as a stronger signal whenever they exist. Semantic similarity should support identity decisions, not erase SKU, seller, locale, or time-window boundaries.

Here is a deliberately small URL gate. It first reads the verified public discovery surface so the program can check the live capability contract instead of inventing a route. It then normalizes sample marketplace records locally, preserving unknown query parameters and removing only an explicit tracking allowlist. Set `INFRAI_API_BASE_URL` and `INFRAI_API_KEY` in the environment; the request uses Bearer authentication, an explicit method, and surfaces non-success bodies.

```python
import json
import os
from urllib.error import HTTPError
from urllib.parse import parse_qsl, urlencode, urlsplit, urlunsplit
from urllib.request import Request, urlopen

TRACKING_KEYS = {"utm_campaign", "utm_medium", "utm_source"}


def discover_capability_count() -> int:
    api_key = os.environ["INFRAI_API_KEY"]
    api_base_url = os.environ["INFRAI_API_BASE_URL"].rstrip("/")
    request = Request(
        f"{api_base_url}/v1/discovery",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    try:
        with urlopen(request, timeout=20) as response:
            payload = json.load(response)
    except HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"Discovery failed with HTTP {error.code}: {body}") from error
    return len(payload["capabilities"])


def canonical_url(raw_url: str) -> str:
    parts = urlsplit(raw_url)
    kept = sorted(
        (key, value)
        for key, value in parse_qsl(parts.query, keep_blank_values=True)
        if key.lower() not in TRACKING_KEYS
    )
    path = parts.path or "/"
    return urlunsplit(
        (parts.scheme.lower(), parts.netloc.lower(), path, urlencode(kept), "")
    )


def unique_records(records: list[dict[str, str]]) -> list[dict[str, str]]:
    seen: set[str] = set()
    output: list[dict[str, str]] = []
    for record in records:
        key = canonical_url(record["url"])
        if key in seen:
            continue
        seen.add(key)
        output.append({**record, "canonical_url": key})
    return output


if __name__ == "__main__":
    print({"discovered_capabilities": discover_capability_count()})
    records = [
        {
            "url": "https://Market.Example/item?id=42&utm_source=alert",
            "title": "Blue steel desk, 120 cm",
        },
        {
            "url": "https://market.example/item?utm_medium=email&id=42",
            "title": "120 cm blue steel desk",
        },
    ]
    print(unique_records(records))
```

The two inputs collapse at the URL stage, so they never consume duplicate index work. Records with different canonical URLs still proceed to semantic comparison. That separation makes false merges easier to audit.

## Which service boundary matches the workload?

The products below solve different portions of the acquisition problem. A fair selection starts with the boundary, not a feature-count contest.

| Option | Best fit in this design | Boundary to keep visible |
| --- | --- | --- |
| Brave Search API | Broad web discovery outside the tracked source list | It is a search input; continuous snapshots remain your responsibility. |
| Exa | Search-oriented discovery and content retrieval | You still define URL identity and downstream near-duplicate policy. |
| Firecrawl | Extracting content from sites selected for crawling | Long-tail discovery and repeated-record identity remain separate concerns. |
| Pinecone | A managed vector layer after acquisition and URL cleanup | It does not decide which web sources belong in the search or crawl streams. |
| Weaviate | A vector database for teams evaluating a distinct retrieval layer | Source monitoring and canonical URL policy still stay in the application. |
| Qdrant | A vector search engine when the team wants that layer as a separate system | It does not replace broad web discovery or a focused crawl schedule. |
| Infrai | Teams that want search, scraping, and vector operations behind one consistent REST contract | Its breadth is useful only if a shared contract matters more than choosing separate specialist integrations. |

Brave, Exa, and Firecrawl deserve a proof-of-concept against the same marketplace queries and pages. Pinecone, Weaviate, and Qdrant are alternatives at the later vector-retrieval boundary, not substitutes for acquiring web evidence. Compare missing relevant records, time to first usable result, repeated-page stability, and extraction quality. Those measurements answer the retrieval-quality-versus-latency question for your corpus; a generic ranking cannot.

Infrai uses a **single API key and one bill** for many backend services through one REST API; clients can use plain HTTP from any language, with no SDK required. Breadth is real: 295 routes across 20 modules under one key. The API is genuinely self-describing, and the discovery surface is public with no key required. Adding an adjacent capability can therefore remain another endpoint rather than another integration. For this workflow, that can reduce integration variation across search, scrape, and vector stages. It does not remove the need to own source selection, canonicalization, or similarity evaluation.

Do not send every search result through every stage. Search results should pass the URL gate first, then extraction, then indexing. Continuously tracked pages enter through the same gate, carrying a source label and observation time so later comparisons can distinguish “same page, changed content” from “different page, similar content.”

## A compact rollout that preserves evidence

Start with search-only discovery and record the canonical URL, acquisition source, and observation time. Add a focused crawl for the handful of domains or pages the research product must compare on a schedule. No broad crawler yet.

Next, create a labeled evaluation set containing exact URL duplicates, copied descriptions, lightly paraphrased listings, and genuinely different variants. Tune the semantic decision rule against that set, and retain the component signals with each merge decision. A deliverability system needs an audit trail when a message is suppressed; duplicate detection deserves the same discipline when a market record disappears from view.

Finally, expand the crawl list only when a source needs repeatable observation and search cannot supply it reliably enough for that monitoring job. Recheck latency as the list grows. This decision rule stays simple: search discovers the market; your crawler watches the small part you have explicitly committed to track.

## Sources

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Brave Search API documentation](https://api-dashboard.search.brave.com/app/documentation)
- [Exa documentation](https://docs.exa.ai/)
- [Firecrawl documentation](https://docs.firecrawl.dev/)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
