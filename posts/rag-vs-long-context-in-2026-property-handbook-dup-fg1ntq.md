# RAG vs Long Context in 2026 — Property Handbook Duplicate Detection

For a 200-page property-management handbook, start with retrieval when the job is repeated course-material Q&A or finding semantically duplicated rules. Use a long-context prompt when reviewers need a one-off, whole-document reading and can tolerate sending the handbook with every request. The decisive trade-off is retrieval quality versus response latency, not a universal claim that one technique is more accurate.

**TL;DR:** Retrieval-augmented generation (RAG) is the practical default for recurring questions because it narrows the evidence before generation and supports citations. Long context removes the retrieval miss from the pipeline, but irrelevant pages still compete for attention and every request carries the large input. For high-stakes duplicate detection, use a hybrid: retrieve candidates quickly, then compare each candidate with its surrounding section before declaring a match.

| Pick | Pick it when | Main accuracy risk | Main latency work |
| --- | --- | --- | --- |
| Long-context prompt | The handbook is read occasionally as one stable artifact | Relevant clauses can be diluted by unrelated material | Repeatedly processing the full input |
| RAG | Learners ask many targeted questions against changing material | The right passage may never enter the candidate set | Index lookup, reranking, and generation |
| Hybrid retrieval plus expanded context | A missed or false duplicate has operational consequences | Candidate selection and final comparison can fail separately | Two stages must fit one latency budget |

No row wins by itself. Measure the failure that matters.

Start there.

## When does a long-context prompt earn its place?

Long context is attractive because the application path is short: assemble the handbook, add the question, and ask the model to reason over both. There is no chunk size, candidate count, or retrieval index to tune. For an instructor reviewing a newly issued handbook once, that simplicity has real value. The whole document also preserves section order and distant cross-references without a separate expansion step.

The catch appears when the workflow repeats. A 200-page input accompanies every learner question, even when the answer lives in one paragraph. Input volume becomes a recurring operational concern, and latency has to include processing that full prompt. More context also does not prove that the model selected the right clause. A complete input can still produce an answer that needs evidence-level checking.

Pick this route when updates are infrequent, request volume is low, and questions genuinely span many sections. Keep page or section markers in the source text, require quoted support, and test questions whose answers sit far apart. If the system cannot point back to the governing passage, a fluent answer is not enough.

## Pick retrieval for repeated, narrow questions

RAG separates evidence selection from answer generation. The original RAG formulation combines retrieved external material with a generator for knowledge-intensive tasks. In a handbook Q&A service, that separation gives operators two observable stages: did search surface the correct clause, and did generation use it correctly?

This is the better fit for repeated questions such as “Can a resident store a bicycle in a hallway?” or “Does ‘shared passage’ duplicate the fire-egress rule?” Only a small candidate set needs to reach the final prompt. Updates can be indexed at the section level instead of forcing every caller to carry the full artifact.

But retrieval creates a hard failure boundary. If chunking splits a condition from its exception, or vocabulary differs between the question and the policy, the generator never sees the needed evidence. Semantic near-duplicate work makes that boundary sharper: two clauses may express the same restriction with different nouns, while two nearly identical sentences may apply to different building types. Consider a hallway-storage rule that bans “personal items in shared passageways” and an emergency appendix that says “egress routes must remain unobstructed.” Retrieval needs to surface both even though the key nouns differ. It must also preserve an exception that permits a temporary maintenance barrier under staff control. A chunk containing only the exception can look permissive; a chunk containing only the ban can look absolute. The mistake then travels cleanly through the system: retrieval returns incomplete evidence, generation writes a confident summary, and a polished answer hides the first-stage miss. That failure is silent unless the evaluation records the required source section. The useful operating rule is **optimize retrieval recall before polishing answer prose**. Build a labeled question set from the handbook. For each item, record the section that must appear in the candidate set, any confusable section, and the acceptable outcome. Track candidate recall separately from final answer correctness. Also record p50 and p95 latency for retrieval, context assembly, and generation; a single end-to-end average hides the slow stage.

Fluency cannot repair missing evidence.

## Use a hybrid for semantic near-duplicates

Property handbooks often repeat obligations across resident rules, staff procedures, and emergency appendices. Exact text matching misses paraphrases. Blind semantic matching over every possible pair creates too much work and can erase scope differences. A two-stage path is easier to inspect:

1. Split on stable section boundaries and retain the heading, applicability, revision, and page range.
2. Retrieve a broad candidate set for the clause under review.
3. Expand each candidate to include its heading and adjacent policy text.
4. Ask for a structured comparison: equivalent, overlapping, conflicting, or unrelated.
5. Send uncertain or conflicting pairs to a reviewer; do not silently merge policy text.

Diagram in words: handbook sections flow into an index; a submitted clause fans out to candidates; candidate neighborhoods enter a comparer; labeled pairs and citations reach the reviewer. Logs attach the same request ID to all four stops.

This design pays a small latency cost for the second comparison stage. It earns that cost by preserving a distinction retrieval scores cannot settle alone: similar wording is not the same as equivalent policy. **The index proposes; the evidence-bearing comparison decides.**

Keep those roles separate.

The focused TypeScript below keeps that boundary explicit. It does not assume a particular database or model. The caller supplies retrieval and comparison implementations, while the orchestration enforces metadata, deduplicates candidates, and returns timings that can feed metrics.

```ts
type Section = {
  id: string;
  heading: string;
  body: string;
  appliesTo: string[];
  revision: string;
  pageStart: number;
  pageEnd: number;
};

type Candidate = Section & { retrievalScore: number };
type Verdict = "equivalent" | "overlapping" | "conflicting" | "unrelated";

type Comparison = {
  candidateId: string;
  verdict: Verdict;
  evidence: string[];
  needsReview: boolean;
};

interface Retriever {
  search(query: string, limit: number): Promise<Candidate[]>;
}

interface Comparer {
  compare(input: {
    clause: string;
    candidate: Section;
  }): Promise<Omit<Comparison, "candidateId">>;
}

export async function findNearDuplicates(
  clause: string,
  retriever: Retriever,
  comparer: Comparer,
  candidateLimit = 12,
): Promise<{ comparisons: Comparison[]; retrieveMs: number; compareMs: number }> {
  const retrieveStarted = performance.now();
  const retrieved = await retriever.search(clause, candidateLimit);
  const retrieveMs = performance.now() - retrieveStarted;

  const unique = [...new Map(retrieved.map((item) => [item.id, item])).values()];
  const compareStarted = performance.now();
  const comparisons = await Promise.all(
    unique.map(async ({ retrievalScore: _score, ...candidate }) => ({
      candidateId: candidate.id,
      ...(await comparer.compare({ clause, candidate })),
    })),
  );

  return {
    comparisons,
    retrieveMs,
    compareMs: performance.now() - compareStarted,
  };
}
```

Twelve candidates in this example are a testable starting parameter, not a universal optimum. Sweep the limit against the labeled set. Stop increasing it when recall gains flatten or comparison latency exceeds the service objective. That is the trade-off in plain sight.

Error handling belongs at the stage boundary. A retrieval timeout should produce a retriable, evidence-free result, not an answer composed from guesswork. A comparison failure should retain the candidate and mark it for review. Log document revision, candidate IDs, stage timings, and verdict counts; do not log private resident data that never belonged in the handbook query.

## Test quality and latency together

An evaluation set should represent the ugly cases, not only easy questions. Include paraphrased duplicates, identical language with different applicability, a rule plus its exception, superseded revisions, and questions with no supported answer. Keep the expected source section beside each case. This makes retrieval recall mechanically checkable before anyone debates writing style.

Run both architectures over the same questions and the same handbook revision. For long context, record answer correctness, citation correctness, unsupported-answer rate, input size, and latency. For RAG, add candidate recall and reranker latency. For the hybrid, score the four duplicate labels and reviewer-escalation rate. Cost follows measurable workload such as input volume, index maintenance, and review effort; a changing per-token price should not drive the architecture.

Set alerts on regressions that map to action. A drop in candidate recall points toward indexing, chunking, or query formulation. Stable recall with worse citation correctness points toward context assembly or generation. A p95 retrieval spike and a p95 generation spike have different owners. Crisp stage metrics keep the response useful.

Do not choose a threshold by intuition. Select it on held-out labeled pairs, document the false-positive and false-negative consequences, and rerun the suite whenever segmentation, embeddings, comparison instructions, or handbook revisions change. The threshold is part of the release artifact.

## Limits and decision rule

RAG adds indexing, evaluation, and observability work. Long context adds repeated input processing and can make evidence selection harder to diagnose. The hybrid adds another stage and reviewer workflow. None guarantees factual answers merely by fitting the handbook into a model input.

For recurring Q&A over a 200-page handbook, choose retrieval first. Choose long context for occasional whole-document synthesis. Choose the hybrid when semantic duplicate decisions affect policy and need auditable evidence. Then hold the choice to two gates: the required passage must be present, and the full request must meet the latency objective. If either gate fails, change the architecture or its parameters before changing the prose.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
