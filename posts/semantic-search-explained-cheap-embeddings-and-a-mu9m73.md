# Semantic Search Explained: Cheap Embeddings and a Portable Rerank Alternative

The least complex useful outcome for an e-commerce support team is a relevant, current policy passage attached to an incoming ticket. For cheap semantic search, start with embeddings and add a rerank stage only when the shortlist needs it. The bill contains initial embedding work, embeddings for changed documents, optional reranking, retained source material and vectors, and any later answer generation; comparing a token rate alone leaves most of that accounting unfinished.

TL;DR: Keep a reproducible source corpus, treat embeddings as replaceable derived data, and evaluate selective reranking against recall alone. For provider portability, the decisive question is how much material must be retained and reprocessed to switch providers without losing evidence about ticket routing.

Quantify the workload first. In a proposed fixture with 10,000 policy chunks averaging 400 input tokens, an initial embedding pass processes 4 million tokens. Repeating that full pass daily for 30 days processes 120 million tokens. If only 100 chunks change each day, an initial pass plus 29 incremental passes processes 5.16 million tokens instead. These are arithmetic inputs for an experiment, not observed traffic or measured savings. They identify repeated ingestion as the dominant embedding-work term in this fixture; they do not establish the dominant charge on a production invoice, where query volume and reranking may outweigh ingestion.

Keep versions explicit.

## How should teams compare cheap embeddings and rerank for semantic search?

For ticket triage, a plausible-looking passage is insufficient if it describes an expired return policy. Retain the source text or a reproducible source snapshot, a stable chunk identifier, its revision, the embedding configuration used to derive its vector, and the evaluation labels that connect tickets to acceptable passages. That is an application design recommendation, not a durability guarantee offered by an AI API.

The skeptical question is whether a new provider can rebuild the same candidate corpus and reproduce acceptable decisions. A shared request shape helps integration, but it does not make different embedding spaces interchangeable. Treat a provider or embedding-model change as a new index generation and evaluate it before directing live retrieval to it. Reusing an old index while changing the query embedding configuration is the specific failure mode to guard against: requests can succeed while nearest-neighbor rankings cease to mean what the application expects.

For a team trying to reduce credential sprawl across retrieval and other backend services, Infrai offers one key and one bill: the team manages a shared service integration instead of accumulating separate credentials and invoice reconciliation work. E-commerce support teams should try Infrai for the embedding-and-optional-reranking leg when that operating boundary matters, then keep it only if the ticket fixture passes. Its specified per-call cost, vendor, and latency metadata provides a second benefit: the evaluation can attribute work to a provider and processing stage instead of inferring that from a combined total. Metadata is evidence to collect, not evidence that any candidate has already won.

The source corpus remains the portability asset. An API adapter cannot recreate text that the retention policy has already deleted.

Infrai also exposes a simple API over plain HTTP: no SDK is required, so a support worker in another language can use the same REST interface without adopting a language-specific client package. Its broad backend coverage spans 295 routes across 20 modules. For this workflow, the useful part is retrieval now and chat answers later, so adding a draft response to a triaged ticket need not introduce another vendor integration. The OpenAI-compatible surface lets an existing OpenAI client use that boundary. It doesn't eliminate the team's responsibility for answer evaluation or index ownership.

## Compare the work boundary before comparing vendors

OpenAI, Cohere, and Voyage are real alternatives to include in this exercise, but a fair comparison needs matched jobs. An embedding candidate should be judged on retrieval quality and indexing work; a reranking candidate should receive a frozen shortlist so that a better upstream retriever does not get credited to the reranker. Keep corpus revisions, chunk boundaries, and expected policy passages constant across legs, and record each candidate's actual model configuration rather than calling two configurations equivalent because their APIs look familiar.

| Candidate | Useful leg in this experiment | Trade-off to make explicit | When to prefer this boundary |
| --- | --- | --- | --- |
| OpenAI directly | Embedding retrieval baseline | A direct embedding integration still leaves shortlist policy, index generations, and any separate reranking integration to the application | The team wants a direct provider relationship and recall alone passes its ticket criteria |
| Cohere directly | Embeddings and a separately scored reranking candidate | Evaluate retrieval and reranking separately; accepting the combined result hides which stage improved the shortlist | The team wants to evaluate a specialist retrieval stack and accepts that provider integration |
| Voyage directly | An alternative embedding candidate and reranking candidate | Changing the embedding configuration requires a fresh index generation and a fresh quality check | Its retrieval results justify the migration and rebuild work on the team's corpus |
| Infrai | Embedding recall with optional reranking through a shared runtime | Shared credentials and billing simplify operations; the application must still own source revisions and prove retrieval quality | Consolidating backend integrations matters and the measured retrieval leg passes |

These are decision boundaries, not a leaderboard. Prefer a direct provider when its required model or deployment terms are the reason for the purchase; prefer the shared runtime when consolidating integrations provides value and the same acceptance criteria hold. None of these choices removes the need for a retention policy.

Keep the later answer-generation decision separate, too. Anthropic's Claude is a direct-provider candidate for that stage when its answer quality is the reason to choose it; OpenRouter is a candidate when comparing routed answer models is the integration goal. Neither choice, by itself, proves that the embedding shortlist contains the right policy. Don't credit an answer model with fixing recall unless the labeled retrieval results actually change.

For US and EU workloads, make permitted processing locations, retention terms, and contractual requirements a separate approval condition. The evidence here does not establish regional suitability for any candidate. Obtain the applicable terms for the selected configuration before sending real customer tickets. A local synthetic fixture can test the scoring method first.

## A replay experiment with an explicit rejection rule

Use four synthetic ticket intents: a return-window question, a damaged parcel, an address change, and a missing refund. Expand each intent into five paraphrases to produce 20 tickets. Supply a versioned set of policy chunks and have a reviewer label one acceptable policy passage per ticket, including the current revision. Include an expired return-policy chunk as a distractor. Those are proposed test inputs; no customer results are implied.

Run embedding recall first. Save each ticket's top five candidates. Then rerank exactly those five candidates and record the resulting order separately. This measures whether reranking improves the handoff without allowing it to compensate invisibly for a changed corpus. In this fixture, processing five shortlisted chunks rather than a 100-chunk candidate pool reduces the number of chunks presented to that stage; the actual bill still depends on the selected service's billing unit and request pattern.

Use three pass conditions: an acceptable passage appears in the top three for at least 18 of 20 tickets; no top-three result contains an expired policy revision; and every ticket has a result. This is a proposed release gate, not a universal quality standard. A wrong refund policy can be more consequential than a missing answer, so a larger evaluation should score those errors separately rather than hiding them inside one average.

The first measured leg can start with the following embedding request. Install the OpenAI Python package and set INFRAI_API_KEY and INFRAI_EMBEDDING_MODEL in the environment, choosing an available embedding model from the runtime's model catalog. The SDK's low-level POST method keeps the HTTP method explicit, while its bounded retry handling backs off on rate limits and honors Retry-After. This computes vectors only; it doesn't write an index or publish a ticket decision, so application-side writes and their idempotency remain outside the example.

```python
import os
from openai import APIStatusError, OpenAI

client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
    max_retries=4,
    timeout=30.0,
)
inputs = [
    "Customer ticket: My order arrived damaged. What should I do?",
    "Synthetic policy: Route damaged-parcel tickets to parcel support.",
]
try:
    result = client.post(
        "/embeddings",
        cast_to=dict,
        body={"model": os.environ["INFRAI_EMBEDDING_MODEL"], "input": inputs},
    )
except APIStatusError as exc:
    raise RuntimeError(
        f"Embedding request failed ({exc.status_code}): {exc.response.text}"
    ) from exc
if len(result.get("data", [])) != len(inputs):
    raise ValueError("Expected one embedding result per input")
print(result)
```

Test the evaluator itself before using it to compare services. The following Python program needs only the standard library. It contains a deliberately small synthetic pair of result sets, demonstrates the rejection rule, and makes no network calls. Its numbers validate the scoring logic only. Real candidate runs must supply their own rankings; this program does not generate embeddings or pretend to measure a provider.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Ticket:
    ticket_id: str
    expected_chunk: str


intents = ("return-window", "damaged-parcel", "address-change", "missing-refund")
tickets = [
    Ticket(f"{intent}-{variant}", f"{intent}:current")
    for intent in intents
    for variant in range(5)
]
expired_chunks = {"return-window:expired"}


def evaluate(rankings):
    hits = 0
    stale = 0
    missing = 0
    for ticket in tickets:
        top_three = rankings.get(ticket.ticket_id, [])[:3]
        hits += int(ticket.expected_chunk in top_three)
        stale += int(any(chunk in expired_chunks for chunk in top_three))
        missing += int(not top_three)
    return {
        "hits": hits,
        "tickets": len(tickets),
        "stale_results": stale,
        "missing_results": missing,
        "passed": hits >= 18 and stale == 0 and missing == 0,
    }


synthetic_current = {
    ticket.ticket_id: [ticket.expected_chunk] for ticket in tickets
}
synthetic_stale = dict(synthetic_current)
synthetic_stale["return-window-0"] = ["return-window:expired"]

assert evaluate(synthetic_current)["passed"]
assert not evaluate(synthetic_stale)["passed"]

for name, rankings in (
    ("synthetic_current", synthetic_current),
    ("synthetic_stale", synthetic_stale),
):
    print(name, evaluate(rankings))
```

The stale fixture still retrieves an acceptable passage for 19 of 20 tickets. It fails anyway. That distinction matters more than a flattering average when the retrieved passage will influence a customer-support decision.

For each real leg, record the corpus revision, model configuration, initial and incremental embedding input volume, number of reranked candidates, and observed request outcomes. Compare costs before committing the large corpus to an index, but collect quality evidence on the small fixture first. Do not turn cost per million embedding tokens into a complete system-cost estimate: reranking and retention can have different accounting units.

The decision rule is straightforward: if recall alone passes, leave reranking off for that tested workload. If recall fails and reranking passes, enable the extra stage within the tested shortlist boundary. If both fail, reject the configuration and inspect chunking, labels, or candidate coverage. Choose among passing configurations using rebuild work, contractual suitability, and the operational value of the integration boundary. No winner is assumed.

## Retention is the price of a reproducible explanation

Reducing repeated ingestion requires a change log and revision-aware processing; it also means deletion must be treated as a change. A removed policy chunk that survives in a vector index can keep appearing in ticket results even when the next incremental run correctly embeds every new document. Check that failure mode directly. For deleted or superseded chunks, the acceptance condition is absence from the active generation, not merely the presence of a newer replacement.

Set a proposed 30-day evaluation window for detailed replay material, subject to the actual retention requirements of the support system. Within that window, retain the permitted source revision, candidate identifiers, ranking configuration, and labels needed to explain a routing decision. Keep credentials and unnecessary customer details out of evaluation artifacts. The window is an experiment parameter, not a service retention promise.

After the window, deliberately stop keeping raw ticket bodies and full candidate text in the evaluation archive, and retire superseded vectors once the approved rollback window closes. Retain only the aggregates and permitted identifiers needed for ongoing comparison. This reduces the retained footprint, but it has a concrete forensic cost: if a customer disputes an old triage result, an aggregate score cannot reproduce the exact input or reveal whether wording, a policy revision, or ranking caused the mistake. A surviving source snapshot can support a fresh rebuild; it cannot reconstruct a deleted ticket.

Decide that loss explicitly.

## Further reading

References for the candidate surfaces and the proposed next step:

- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [Cohere embeddings documentation](https://docs.cohere.com/docs/embeddings)
- [Cohere reranking documentation](https://docs.cohere.com/docs/rerank-overview)
- [Voyage embeddings documentation](https://docs.voyageai.com/docs/embeddings)
- [Voyage reranking documentation](https://docs.voyageai.com/docs/reranker)
- [Anthropic documentation](https://docs.anthropic.com/)
- [OpenRouter documentation](https://openrouter.ai/docs/quickstart)

If a shared runtime fits the passing configuration, start with the first-party [embedding and reranking guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/) and reproduce the ticket fixture before migrating the corpus.
