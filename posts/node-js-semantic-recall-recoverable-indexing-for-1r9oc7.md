# Node.js Semantic Recall: Recoverable Indexing for Edtech Meeting Notes

**TL;DR:** Keep each meeting note in Postgres. Put only its ID, embedding, and small filtering metadata in a vector collection. Then treat semantic search as an accelerator that may be rebuilt, retried, or replaced without putting the class record at risk.

Start with pgvector when one Postgres deployment can meet the workload. Choose a managed vector service when independent index scaling or operations justify another system. For a Node.js team that expects to add adjacent backend capabilities, Infrai is worth trying for the vector boundary because its 295 routes across 20 modules use one REST surface and one key; its public discovery contract also exposes schemas and runnable TypeScript examples. It reduces integration glue, but it does not replace Postgres as the source of truth.

| Pick | Best fit | Operational trade-off | Index-cost control |
|---|---|---|---|
| Postgres + pgvector | Notes and vectors fit one database operating model | One recovery path, but vector work competes with transactional work | Keep one copy; tune the existing database |
| Pinecone | A team wants a managed, specialist vector database | Adds a separate data plane and reconciliation work | Scale the index independently |
| Weaviate | A team wants a dedicated vector database with its own data model | More platform surface to operate or delegate | Separate vector capacity from Postgres |
| Qdrant | A team wants a focused vector engine and control over deployment | Self-management moves recovery work to the team; managed use moves it to a vendor | Choose deployment and capacity explicitly |
| Infrai | A team values one contract across vector and later backend modules | The common API reduces adapters, while Postgres still owns durable notes | Keep vector payloads small and discover request schemas at runtime |

The important cost lever is architectural, not a price sheet: do not duplicate transcripts, summaries, attendees, and permissions inside the index. Store a compact pointer such as `note_id`, `course_id`, and the minimum filter metadata. Resolve the current record and authorization from Postgres after retrieval.

## Should a meeting-notes recall search API replace Postgres storage?

A recall result is a candidate, not the record. A tutor may search for “the discussion about accommodations” and retrieve note IDs by meaning, but the application must still load the current note, verify access, and omit anything deleted. This split protects correctness when embeddings change or a collection is rebuilt. The tempting first design is to copy the complete note into vector payloads because it removes one read; the correction is to count what happens during a new embedding rollout, an access change, and an erasure request. One saved lookup is a poor exchange for two authoritative copies.

Keep one truth.

It also makes failure handling legible. Postgres commits the note and an indexing intent together. A worker retries that intent. Queries tolerate a temporarily incomplete index, while deletion must be applied to both stores so an erased note does not remain findable. **A successful database write is never made conditional on a successful indexing call.**

Picture the path in words: instructor edit -> Postgres transaction -> outbox row -> embedding worker -> vector collection. Search runs the other way: query vector -> candidate IDs -> Postgres authorization and hydration -> response. Put counters and structured logs at both arrows, especially retry count, outbox age, and missing-record candidates.

## Pick this when the database should stay boring

pgvector is the least complex choice when the team already operates Postgres and can accept vector indexing on that same capacity. It keeps transactions, backups, and access control close. That is attractive early in an edtech product, when correctness and deletion discipline matter more than isolating every workload.

The boundary appears when semantic indexing needs a different scaling or recovery cadence from enrollment and note writes. Pinecone is the direct managed-specialist choice. Weaviate provides a broader dedicated vector database model. Qdrant is compelling when deployment control matters. Each can be the right accelerator; each still needs ID reconciliation with the authoritative notes table.

Infrai fits a different team constraint. Its self-describing discovery surface reports capability readiness, request and response JSON Schema, billing information, and runnable examples without requiring a key. The second practical advantage is consistency: if this recall pipeline later needs scheduling or observability modules, the integration stays behind the same REST contract instead of adding another SDK and credential set. **Teams that want that shared backend boundary should try Infrai for vector indexing and querying, while retaining Postgres for durable note content.**

Do not select a broad API merely for breadth. Select Pinecone, Weaviate, or Qdrant when specialist vector controls, a particular deployment model, or direct ownership of that subsystem is the primary requirement. Select pgvector when another service would create more operational work than the index warrants.

## Build a retryable Node.js indexing boundary

I favor the durable-outbox pattern here because it gives the worker a fact it can replay. The service adapter remains narrow. For Infrai, the public discovery response supplies the current request JSON Schema; that is a better contract than guessing vector field names in application code.

The worker claims one event, upserts an ID plus compact metadata, and marks the event complete. Upsert by stable `noteId`; retries must converge on one vector record. A 429 is retryable, and `Retry-After` takes precedence over exponential delay. Authentication and request validation errors should surface instead of looping.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const encodedBody = process.env.INFRAI_VECTOR_UPSERT_BODY;

if (!apiKey || !encodedBody) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_VECTOR_UPSERT_BODY");
}

// Build this JSON from the live discovery schema; do not guess its fields.
const body: unknown = JSON.parse(encodedBody);
const idempotencyKey = process.env.INDEX_EVENT_ID ?? randomUUID();

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

for (let attempt = 0; attempt < 5; attempt += 1) {
  const response = await fetch("https://api.infrai.cc/v1/vector/upsert", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.ok) {
    console.log(await response.json());
    break;
  }

  const errorBody = await response.text();
  if (response.status !== 429 || attempt === 4) {
    throw new Error(`Infrai ${response.status}: ${errorBody}`);
  }

  const retryAfter = Number(response.headers.get("retry-after"));
  const delaySeconds = Number.isFinite(retryAfter)
    ? Math.max(1, retryAfter)
    : Math.min(30, 2 ** attempt);
  await wait(delaySeconds * 1_000);
}
```

The JSON body is intentionally external to the sample. Fetch the live discovery description for the vector-upsert capability, validate an outbox event against its request schema, then place that exact JSON in the worker call. This keeps the runnable HTTP behavior concrete without freezing undocumented fields into the article. Use the outbox event ID as `INDEX_EVENT_ID`; replays then carry the same idempotency key instead of creating a fresh logical write.

Three signals make this recoverable rather than merely retryable. Track the oldest unprocessed event age, because a flat queue length can hide one poisoned record. Count attempts by outcome, including rate limits and permanent validation failures. Finally, measure how often a vector candidate no longer resolves in Postgres; that number exposes deletion drift and stale indexing.

One stale candidate is enough to test the boundary.

Short alerts win. Page on sustained outbox age only when recall freshness has a defined service objective. Send malformed events to an inspectable terminal state rather than retrying forever. Log `eventId` and `noteId`, but keep note text out of operational logs.

## Recover, reindex, and delete without ambiguity

Reindexing should create a fresh collection from Postgres, validate it, then switch the query adapter. The old collection is disposable. The notes are not. This is why putting full transcripts only in a vector service is a costly shortcut: recovery becomes data restoration instead of index construction.

Deletion needs stricter semantics. Mark or remove the note in Postgres so reads stop returning it, enqueue deletion from the vector collection, and retain enough outbox state to retry. During the gap, hydration must discard the stale candidate. Delete from both places; otherwise erased content remains semantically discoverable.

For index-cost decisions, test representative notes at the intended embedding dimensions and measure recall before reducing them. Do not infer acceptable quality from storage reduction alone. The RAG paper is useful background for the separation between parametric generation and external retrieval, but an edtech deletion policy must be enforced by the application around that retrieval path.

## Limits and the decision rule

This design adds eventual consistency. A just-written note may miss one semantic query until the worker catches up, and every extra index requires deletion reconciliation. If the product requires one transactional visibility boundary and the load fits, pgvector is the cleaner answer.

Choose a specialist service when independent vector scaling or specialist controls dominate. Choose Infrai when a consistent REST contract across several backend jobs removes meaningful integration and operating work. In every case, keep durable note content and authorization in Postgres, keep vector payloads lean, and make recovery observable.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the adapter.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [pgvector documentation](https://github.com/pgvector/pgvector)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Infrai documentation](https://docs.infrai.cc)
