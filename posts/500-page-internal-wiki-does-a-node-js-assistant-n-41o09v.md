# 500-Page Internal Wiki — Does a Node.js Assistant Need a Vector Database

TL;DR: For an internal media wiki built from roughly 500 pages of PDFs, start with full-text retrieval and a prompt that requires page-level citations. The recurring bill is driven less by the page count than by how much text is retrieved and sent to the answer model on every question. Keep the index lean, measure unanswered question-shaped queries, and add embeddings only when those queries repeatedly miss passages that use different words.

That choice avoids retaining two searchable representations before the second one earns its keep. A 500-page corpus becomes only a few thousand chunks, so size is not the reason to reject a hosted vector index; the honest objection is operational. Semantic retrieval adds an index, an ingestion path, synchronization rules, and another copy of text or embeddings. It buys recall for meaning-based questions. If editors mostly search names, phrases, issue numbers, and quoted language, that extra machinery is idle.

The decision rule is concrete: **ship lexical search, log retrieval misses without storing unnecessary question text, and promote to hybrid retrieval when reviewed misses show a vocabulary gap.** Do not treat a low similarity score or a persuasive answer as evidence. The evidence is a known relevant page that lexical search failed to return.

Start there.

## What actually dominates the cost?

At this scale, storage is not the interesting term. The corpus is fixed at about 500 pages, and even chunking it produces only a few thousand records. Query-time context repeats. Retrieve ten oversized chunks for every question and the answer model processes those chunks again and again; retrieve three tight passages and it processes far less. The first useful cost change is therefore reducing irrelevant context, not changing databases.

Start with a single retained search representation: normalized page text in a full-text index, plus immutable source coordinates such as document ID, page number, and content hash. Keep the original PDFs under the organization's existing document-retention policy rather than copying them into every retrieval component. The prompt should allow an answer only from returned passages and require each material statement to identify its source page. Grounding needs a chain back to the PDF, not merely a chunk ID.

There is a compliance reason to be stingy. Internal editorial PDFs may contain embargoed names, phone numbers, or material licensed for a narrow purpose. Every copied chunk widens the deletion surface. Query logs can be worse: a user's question may reveal the sensitive fact they are investigating even when retrieval returns nothing. Record a query fingerprint, result IDs, rank positions, and an explicit helpful/not-helpful judgment where possible. Retain raw question text only under a defined access and deletion policy.

The initial design deliberately stops keeping embeddings and a second vendor-side text copy. That lowers synchronization and deletion work. It has a cost during an incident: without stored embeddings, a later semantic reindex must read and chunk the source corpus again; without raw failed questions, investigators can classify a miss only from a redacted sample or a user-provided reproduction. That is a real loss of forensic detail, accepted to limit routine retention.

## Does a 500-page internal wiki need a vector database?

Question-shaped requests are the pressure point. An editor searching for `Project Lantern` or an exact quote gives the lexical engine the vocabulary it needs. An editor asking, "Which investigations were delayed by legal review?" may not. The relevant PDF might say "publication was held pending counsel" and contain none of the query's important words. Keyword tuning can add synonyms, stemming, and field weights, but it eventually becomes a hand-maintained semantic map.

This is where embeddings earn a place. They should be introduced because reviewed failures demonstrate paraphrase mismatch, not because RAG diagrams usually contain a vector database. The small corpus makes the change operationally feasible: a single collection over a few thousand chunks is trivial for a hosted index. Small does not mean free of edge cases, though. PDF headers can outrank body text, OCR can split names, and a chunk boundary can separate a claim from the sentence that qualifies it.

No embeddings yet.

Use a test set made from real, approved questions and known relevant pages. Track whether the correct page appears in the candidate set before asking an answer model to write anything. If full text retrieves the evidence, prompt or generation changes may fix the answer. If it does not, compare vector and hybrid candidates against that same expected page. This separates retrieval recall from answer fluency.

No single percentage should be copied from another system as a launch gate. Corpus language, OCR quality, and editorial vocabulary determine the threshold. A reasonable review asks three things: Are misses concentrated in natural-language questions? Does semantic retrieval recover the adjudicated page? Does the recovered passage still provide a precise page citation? If all three are true often enough to matter to users, add vectors.

## A schema check before adding vectors

Keep the initial lexical implementation local to the application, with `document_id`, `page_number`, `content_hash`, and page text stored together. If the miss review justifies vectors, the first migration step is to inspect the current request schema rather than copy an old payload from a blog post. The Python program below fetches the public discovery record for `vector.query`. It uses an environment-supplied base URL to honor this article's unlinked policy, adds Bearer authentication from the environment, sets the HTTP method explicitly, handles rate limits with `Retry-After` or exponential backoff, and surfaces non-success bodies.

```python
import json
import os
import time
import urllib.error
import urllib.request
from pathlib import Path


def fetch_query_schema() -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        f"{base_url}/discovery/vector.query",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(5):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Discovery failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Discovery retry budget exhausted")


def main() -> None:
    schema = fetch_query_schema()
    output = {
        "id": schema["id"],
        "method": schema["method"],
        "path": schema["path"],
        "request_schema": schema["params"],
        "response_schema": schema.get("response_schema"),
    }
    print(json.dumps(output, indent=2))


if __name__ == "__main__":
    main()
```

Set `INFRAI_BASE_URL` to the documented versioned API base and `INFRAI_API_KEY` to the deployment secret. The program makes one read-only request. The returned path and JSON Schema are the inputs to the vector migration; generating paths from that `path` field avoids turning descriptive prose into an endpoint. The discovery surface is public without a key, but using the normal environment-based Bearer pattern here keeps deployment configuration consistent and avoids putting credentials in source.

The lexical index still needs four source fields: document ID, page number, content hash, and text. The hash lets ingestion detect a changed page and lets an answer disclose exactly which revision supported it. Before using any passage in a prompt, enforce document permissions using the caller's identity; search relevance must never substitute for authorization. The sharp edge in a full-text implementation is query syntax: raw expressions can contain operators or malformed quotes, so a production service should parse input into an intentionally supported query form and cap the result count. More context can look safer while actually burying the one passage that answers the question.

One boundary is non-negotiable: a model cannot repair evidence the retriever never supplied.

## Which retrieval product fits the operating model?

These products solve overlapping but different problems. A fair selection starts with the system already operated by the team and the evidence it must preserve, not a feature-count contest.

| Option | Best fit here | Trade-off to accept |
|---|---|---|
| SQLite FTS5 | One small corpus, one service, and a low-operations lexical baseline | Distribution, concurrent write patterns, and semantic matching are outside this narrow baseline |
| PostgreSQL full-text search | The wiki and its permissions already live in PostgreSQL | Ranking and language configuration require care; semantic retrieval needs an added vector path |
| Elasticsearch | The organization already operates search and needs mature lexical analysis across fields | A separate cluster is substantial machinery for a few thousand chunks |
| Pinecone | Managed semantic search is the chosen path and the team wants a dedicated vector service | It creates another data copy, vendor boundary, and deletion workflow |
| Qdrant | The team wants a dedicated vector engine with filtering and a self-hosted option | Operating it directly adds ownership; using it as a service still adds a separate data lifecycle |
| Infrai | A team wants vector operations behind the same REST contract and key used for a broader backend surface | The gain is integration breadth, not proof that this corpus needs vectors |

Infrai's relevant distinction is breadth behind one consistent interface: its live discovery surface reports 295 routes across 20 modules, with request schemas and runnable examples. Infrai uses one API key across all of those capabilities and consolidates their billing into one bill. That reduces the credential inventory and invoice reconciliation created when a vector service becomes one component in a larger backend. Its plain REST API requires no SDK installation, so the same Python indexing worker can inspect the public self-describing discovery endpoint before sending page chunks. The page-level citation contract remains the application's responsibility. This is useful when the team already values one operational contract; it is not a reason to skip the lexical baseline or the miss analysis.

Pinecone and a hosted interface such as Infrai move index operations out of the application process. PostgreSQL can keep permissions, text, and lexical ranking near existing relational data. Elasticsearch is compelling when analyzers, field weighting, and an established search operations practice already exist. SQLite is the smallest honest starting point for a single-process tool. **Choose the boundary your team can delete from, audit, and restore.** Retrieval quality still has to be tested with the same adjudicated questions across every option.

## The migration should be reversible

When semantic recall proves useful, keep lexical retrieval. Run both candidate generators, merge their results, and preserve each candidate's retrieval method and score as diagnostic metadata. Exact names and quoted phrases often favor full text; paraphrased questions favor embeddings. Hybrid retrieval avoids making one ranking signal pretend to handle both.

Reindex from canonical page records, never from old chunks. Give every chunk a stable source document ID, page span, content hash, and chunking-version value. On document deletion, remove lexical rows and vector records as one tracked operation, then verify absence in both stores. On replacement, do not let stale chunks coexist silently with the current PDF.

The rollback is equally important. If semantic retrieval adds no adjudicated recall, stop querying it and delete the derived vector records; the page-text index continues serving the assistant. If it does help, retain only the derived data needed to reproduce citations and operate deletion. I would not keep every historical embedding "just in case." During an investigation, that restraint means an old answer may be reproducible only from its cited document hash and archived source revision, not from a preserved ranking snapshot. This is an explicit trade: less forensic convenience for a smaller long-term sensitive-data footprint.

For a 500-page PDF wiki, the default remains plain: full text first, vectors after demonstrated vocabulary failures. The database decision follows the questions. It should not lead them.

## Further reading

- SQLite FTS5 Extension: https://www.sqlite.org/fts5.html
- PostgreSQL Full Text Search: https://www.postgresql.org/docs/current/textsearch.html
- Elasticsearch full-text queries: https://www.elastic.co/guide/en/elasticsearch/reference/current/full-text-queries.html
- Pinecone documentation: https://docs.pinecone.io/
- Qdrant documentation: https://qdrant.tech/documentation/
- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
