# Python Multi-Model API Selection 2026 (Small Team Evidence Contracts)

Choose a multi-model API for private customer support by testing whether its answers can pass an application-owned evidence contract, with retrieval versions and access decisions preserved across a model switch. A shared credential is convenient; the deciding constraint is whether a small team can change the generation backend without changing what counts as an acceptable answer.

TL;DR: Keep the knowledge base, authorization, evidence snapshots, and answer acceptance rules under application control. Put model-specific request translation behind a narrow adapter, then accept structured answers only after checking their shape and citation membership against the authorized snapshot. Treat factual support as a separate review requirement. Valid JSON alone does not settle it.

This decision record takes a data-lifecycle angle: an answer is a proposed derived record whose evidence can expire. The design below is a proposed implementation, and its sample identifiers and revision numbers are illustrative, not production observations or benchmark results.

## Decision: make evidence part of the answer contract

For a support agent answering questions over a private knowledge base, an answer containing a well-formed citation to an inaccessible document is unacceptable, even if the prose sounds right. The same applies when the citation names a document that was revised between retrieval and publication. A model interface should therefore be selected against the acceptance boundary the support application needs, rather than against the appeal of one credential covering several backends.

The proposed boundary has three application-level outcomes: accept a grounded answer, accept an explicit abstention, or reject the candidate. Rejection belongs to the application; it must not be confused with a model returning an abstention. A transport failure is a fourth, operational outcome, recorded separately so that a team does not accidentally count missing responses as successful restraint.

Keep ownership explicit. The retrieval service decides which document revisions the requester may see. The generation adapter receives only that evidence and returns a candidate object. The acceptance layer checks the candidate before anything reaches the support interface, and the publication step checks whether the authorization and evidence context are still valid.

That is the contract.

## How should a small team select a multi-model API for private support?

Start with invariants a reviewer can actually inspect. In this design, every accepted citation must name an exact document revision in the authorized retrieval snapshot; an answered result must contain text and at least one citation; an abstention must carry neither answer text nor citations. These rules are deliberately modest. They establish shape and evidence membership, not whether a sentence follows logically from the cited passage.

Consider a support question about whether a replacement shipment is available. The illustrative retrieval snapshot contains `replacement-policy` at revision `7`. A candidate citing revision `6` fails, even if revision `6` exists elsewhere in storage, because the generation attempt was not authorized to substitute evidence outside its snapshot. Conversely, citing revision `7` does not prove the answer interpreted the policy correctly. A plausible citation can still accompany an invented exception.

Name these failure modes separately: citation substitution, stale evidence, authorization drift, and unsupported interpretation.

Collapsing them into a single “bad JSON” counter would hide which boundary needs attention. An adapter change can repair malformed output; it cannot establish that a requester is entitled to read a document, and it cannot make an old policy current.

For a storage-oriented design, the consequential question is what publication means. If the requirement is “current at publication,” a check followed by an unrelated write leaves a race in which a policy changes between those operations. Require the publication operation to condition acceptance on the expected evidence and authorization versions, or explicitly define a weaker guarantee such as “answered against the authorized snapshot captured at retrieval.” Do not describe the weaker guarantee as current knowledge. The implementation must match the promised boundary.

These are proposed consistency requirements, not claims that an arbitrary object store or API supplies the necessary transaction. The team must verify that its chosen publication mechanism can enforce them.

## Compare where the acceptance boundary lives

The following table compares design options, not commercial offerings. The deciding trade-off is how much work remains stable when a model, prompt, or knowledge-base revision changes.

| Option | What the team owns | Main failure boundary | Appropriate use |
| --- | --- | --- | --- |
| Pass generated text straight to the support interface | Prompt and display behavior | Unsupported interpretations can reach the reader without a structured acceptance step | A supervised drafting surface with mandatory human review |
| Accept any object matching an answer shape | Field names and basic validation | A correctly shaped citation can name unauthorized or unrelated evidence | An internal formatting experiment using nonprivate sample material |
| Validate an answer against an authorized evidence snapshot | Shape, revision membership, publication conditions, and review policy | Semantic support still requires its own check | A proposed boundary for private knowledge-base answers |

The last option creates maintenance work.

Someone must define the answer contract, keep fixtures representative, and decide what happens to an in-flight response after access changes. For a small team, start with one narrow answer type rather than a framework that tries to represent every conceivable model response. Every additional field becomes another promise to test.

The shared API credential should be assessed as a credential boundary: determine where it is held, which services can exercise it, and how it is rotated. Keep it distinct from the end user's knowledge-base permissions. This design never treats possession of the generation credential as permission to read support documents.

## Critical path: reject candidates outside the snapshot

This Python example implements the structural and citation-membership portion of the boundary. An adapter must supply a decoded object to this function; the function intentionally rejects extra fields and does not coerce types. The snapshot must come from the authorized retrieval step, never from a citation list returned by the model.

```python
def validate_candidate(candidate, snapshot):
    """Validate shape and citation membership, not semantic support."""
    if type(candidate) is not dict:
        raise ValueError("candidate must be an object")
    if set(candidate) != {"status", "answer", "citations"}:
        raise ValueError("unexpected answer fields")

    status = candidate["status"]
    answer = candidate["answer"]
    citations = candidate["citations"]
    if type(status) is not str or status not in {"answered", "abstain"}:
        raise ValueError("invalid status")
    if type(answer) is not str or type(citations) is not list:
        raise ValueError("invalid answer or citations type")

    if status == "abstain":
        if answer != "" or citations != []:
            raise ValueError("abstention must contain no answer or citations")
        return {"status": status, "answer": "", "citations": []}

    if not answer.strip() or not citations:
        raise ValueError("answered result requires text and evidence")

    checked = []
    seen = set()
    for citation in citations:
        if type(citation) is not dict:
            raise ValueError("citation must be an object")
        if set(citation) != {"document_id", "revision"}:
            raise ValueError("unexpected citation fields")
        document_id = citation["document_id"]
        revision = citation["revision"]
        if type(document_id) is not str or type(revision) is not int:
            raise ValueError("invalid citation identifier or revision")
        key = (document_id, revision)
        if key not in snapshot or key in seen:
            raise ValueError("citation outside snapshot or duplicated")
        seen.add(key)
        checked.append({"document_id": document_id, "revision": revision})

    return {"status": status, "answer": answer, "citations": checked}


authorized_snapshot = frozenset({("replacement-policy", 7)})
candidate = {
    "status": "abstain",
    "answer": "",
    "citations": [],
}
accepted_shape = validate_candidate(candidate, authorized_snapshot)
```

The exact integer check is intentional: this contract doesn't accept a Boolean where a revision number is expected. Copying validated citation values into a fresh result also avoids returning the original mutable citation dictionaries. Neither detail substitutes for the missing semantic check or the conditional publication step.

`accepted_shape` is named accordingly.

Before this function, impose explicit response-size and parsing limits in the adapter. After it, apply the separate support-review policy and the publication condition. Do not put the support answer into an automated workflow merely because this function returns successfully.

## Rejected option: treat a common API shape as the exit plan

A common request format is useful for an adapter, but this decision rejects it as the entire portability strategy. If the application leaves its knowledge-base ownership, evidence versioning, or acceptance rules implicit, moving a request to another backend does not demonstrate that the resulting answer satisfies the same support obligations. The exit test should replay the same authorized evidence and apply the same acceptance rules. This is the practical vendor lock-in test: can the team retain the evidence and acceptance machinery when it changes who generates the candidate?

There is a valid smaller use case. For a human-reviewed drafting tool using nonprivate examples, a common interface can be enough to explore response styles before investing in the publication boundary. Keep that experiment's guarantee narrow.

For deployment, construct illustrative fixtures covering an answer with valid evidence, an abstention, a missing citation, a revision outside the snapshot, an extra field, and an access change before publication. Add unsupported interpretations that pass structural validation, because those cases reveal the limit of the checker rather than a formatting defect. Run the fixtures through each candidate adapter and record structural rejection, semantic rejection, abstention, transport outcome, and publication outcome separately. No measured pass rate is asserted here; those results must come from the team's own corpus and review policy.

Retries deserve the same discipline. Bound the number of attempts and elapsed time, and make an explicit choice about whether a retry reuses its snapshot or begins a fresh authorized retrieval. If evidence changes, record a new attempt context. Avoid retaining entire private prompts in routine telemetry; decide what evidence must be retained for review, who can access it, and when it should expire.

Budget comparisons should include review effort and repeated generation alongside usage. Token counting also has a boundary: the official tiktoken repository describes a BPE tokenizer library for OpenAI models, which is evidence for its stated purpose, not a basis for assuming an identical count across unrelated model families. Where health information is involved, assess the applicability of 45 CFR Part 164; an answer validator does not establish compliance with its security and privacy requirements.

The selection rule is concrete: accept a candidate interface only after the team can replace its generation adapter while keeping evidence ownership, answer acceptance, and publication conditions intact. Prefer the smallest contract the support workflow can defend. Expand it when a real requirement demands more.

## References

- Official tiktoken repository; tokenizer scope and BPE implementation: https://github.com/openai/tiktoken
- Electronic Code of Federal Regulations, 45 CFR Part 164; security and privacy requirements relevant to assessing regulated health-information workflows: https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
