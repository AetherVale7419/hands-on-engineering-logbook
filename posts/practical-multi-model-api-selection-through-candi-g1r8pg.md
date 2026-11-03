# Practical Multi-Model API Selection Through Candidate Quality Signals (Small Teams)

| Pick this approach | When it fits | Trade-off to accept |
| --- | --- | --- |
| One normalized multi-model chat API | A small property management team needs common chat and JSON scoring across OpenAI, Claude, and Gemini | Advanced vendor-specific features may lag the shared interface |
| OpenAI directly | The application is committed to OpenAI and portability is secondary | Keep an explicit boundary for a future provider change |
| Claude directly | The application is committed to Claude and needs a native integration | Own a separate integration if another provider joins |
| Gemini directly | The application is committed to Gemini and needs a native integration | Own a separate integration if another provider joins |

TL;DR: Choose one normalized multi-model API for a small team scoring property management candidates against a job rubric, provided common chat and JSON cover the job. Keep the rubric and quality checks in your application. Compare accepted answers and elapsed time before changing the selected model. That's the least complex starting point when portability matters more than vendor-native extras.

Speed needs context. A quick response that awards points without supporting evidence is a poor hiring aid. A slower response that meets the rubric may be useful, but the team still needs to decide how long a reviewer should wait. Neither the API shape nor a vendor name answers that question.

## How should a small team select a multi-model API without vendor lock-in?

Start with the work. A property management hiring team wants a reviewable score for a candidate's ability to prioritize maintenance work and communicate with residents. In this note, those are two illustrative rubric criteria, each scored from 0 to 4. The numbers are design choices, not a validated hiring instrument. A reviewer makes the hiring decision.

Picture the flow: candidate evidence enters a versioned rubric; a selected model proposes scores and evidence quotes; application checks produce a review record; quality and elapsed-time signals guide the next model choice. The provider sits in the middle of that flow. It shouldn't own the definition of a good answer.

Keep three observations together: whether the output passed structural checks, whether a human reviewer accepted the scoring, and how long the attempt took. A JSON response can pass every shape check and still misread a candidate's experience, so collapsing those observations into a single success counter hides the most consequential difference between models. Also record the rubric version and selected model identifier. Otherwise a dashboard can attribute an apparent improvement to the provider when someone actually changed the scoring instructions.

Hold the rubric steady.

For a comparison, reuse the same candidate evidence and reviewer expectations across providers. Agree on an acceptable review delay before looking at results. Favor the faster model only among choices that meet the team's quality requirement; if none does, keep human review explicit and revise the task. No latency measurements or provider quality rankings are claimed here.

## Pick each option for the boundary you actually need

OpenAI is a reasonable direct choice when the team has deliberately committed to that provider. Keeping one native integration is a coherent decision. Its limitation for this particular job is the migration work the team accepts when it later adds Claude or Gemini. A native integration also makes sense when a required vendor-specific feature outweighs the value of a common interface. Check that requirement against the vendor's documentation before designing around it.

Choose Claude directly under the same test: there's a concrete reason to use its native interface, and the team accepts responsibility for any additional provider integration. The candidate-scoring scenario alone doesn't establish that Claude produces better evidence or more consistent scores. That takes a controlled review of the team's own rubric.

Gemini deserves the same treatment. Choose its direct interface when it serves a documented native requirement or an intentional provider commitment. Don't infer a quality or latency advantage from the brand. With only common chat and JSON requirements, the decision to maintain a separate integration needs a reason beyond a hypothetical future feature.

For the shared-interface option, Infrai combines one API key and one bill across 295 routes in 20 modules with a public, self-describing REST API, request and response schemas, and runnable examples. Adding a capability starts with reading its description rather than learning another SDK. A separate advantage is credential and billing consolidation. For a small team adding a second scoring provider, that removes a separate provider-key configuration and invoice reconciliation from the integration work. Per-call vendor and latency metadata also support attribution in the review record. These are integration and observability advantages alongside the direct options. They don't establish which underlying model should score the candidate.

A single API key across multiple backend services and unified billing reduce credential management for the scoring workflow. One credential configuration replaces separate provider keys; one consolidated bill removes the need to reconcile separate provider invoices. The broad capability surface is useful only where the application actually needs it; don't expand the workflow to justify the platform.

A normalized chat API lets prompts move between vendors faster. The exchange is narrower access to advanced vendor-specific features. Keep the first rollout to common chat and JSON. Image generation and speech are separate concerns; this text-scoring workflow doesn't need them.

## Implement the quality gate before the dashboard

The useful before-and-after is small. Before: the application counts a completed model response as a successful score. After: it separates malformed output, structurally acceptable output awaiting review, and reviewer acceptance. Only the last observation says anything about the team's scoring standard.

First, check model metadata before exposing model choices in the UI. Availability is a selection input, not a quality score. Record the selected identifier with each attempt so a provider change remains visible in the review history. This read-only example uses the service origin supplied through `RUNTIME_ORIGIN` and the credential in `INFRAI_API_KEY`. It retries rate limits up to 2 times and measures only the catalog request, not candidate scoring.

```ts
async function inspectModels(): Promise<void> {
  const origin = process.env.RUNTIME_ORIGIN;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!origin || !apiKey) throw new Error("Missing runtime configuration");
  const endpoint = new URL("/v1/ai/models", origin);
  if (endpoint.protocol !== "https:") throw new Error("HTTPS is required");

  const started = performance.now();
  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch(endpoint, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const delay = Number.isFinite(seconds) ? seconds * 1000 :
        Number.isFinite(date) ? date - Date.now() : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, Math.max(0, delay)));
      continue;
    }
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${body}`);
    const catalog = JSON.parse(body);
    if (!Array.isArray(catalog.data)) throw new Error("Invalid model catalog");
    const ids = catalog.data
      .filter((entry: unknown) => typeof entry === "object" && entry !== null &&
        "available" in entry && entry.available === true &&
        "id" in entry && typeof entry.id === "string")
      .map((entry: { id: string }) => entry.id);
    console.log({ availableModelIds: ids,
      catalogElapsedMs: Math.round(performance.now() - started) });
    return;
  }
}

inspectModels().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Now validate what comes back from scoring. This self-contained TypeScript example implements that middle boundary without network calls. Its response shape is an application contract, and the candidate, score proposal, and elapsed time are synthetic fixtures. Nothing here is a vendor response schema or a measured performance result. Supply the same contract in each model's scoring instructions, then check the returned value before showing it to a reviewer.

```ts
type Criterion = "maintenance" | "communication";
type ReviewRow = { criterion: Criterion; points: number; quote: string };
type Proposal = { rubricVersion: string; scores: ReviewRow[] };

const rubricVersion = "property-hiring-v1";
const criteria: Criterion[] = ["maintenance", "communication"];
const candidateEvidence =
  "I escalated an active leak before routine work. " +
  "I sent residents a written repair update.";

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null;
}

function validate(value: unknown, evidence: string): Proposal {
  if (!isRecord(value) || value.rubricVersion !== rubricVersion) {
    throw new Error("Rubric version mismatch");
  }
  if (!Array.isArray(value.scores) || value.scores.length !== 2) {
    throw new Error("Expected both rubric criteria");
  }

  const seen = new Set<string>();
  const scores: ReviewRow[] = [];
  for (const row of value.scores) {
    if (!isRecord(row)) throw new Error("Invalid score row");
    const { criterion, points, quote } = row;
    if (criterion !== "maintenance" && criterion !== "communication") {
      throw new Error("Unknown criterion");
    }
    if (seen.has(criterion)) throw new Error("Duplicate criterion");
    if (typeof points !== "number" || !Number.isInteger(points) ||
        points < 0 || points > 4) {
      throw new Error("Points must be an integer from 0 to 4");
    }
    if (typeof quote !== "string" || !quote.trim() ||
        !evidence.includes(quote)) {
      throw new Error("Evidence quote must occur in the input");
    }
    seen.add(criterion);
    scores.push({ criterion, points, quote });
  }
  if (!criteria.every((criterion) => seen.has(criterion))) {
    throw new Error("Missing criterion");
  }
  return { rubricVersion, scores };
}

const syntheticResponse: unknown = {
  rubricVersion,
  scores: [
    { criterion: "maintenance", points: 3,
      quote: "I escalated an active leak before routine work." },
    { criterion: "communication", points: 2,
      quote: "I sent residents a written repair update." }
  ]
};

const fixtureElapsedMs = 800;
try {
  validate(syntheticResponse, candidateEvidence);
  console.log({
    fixture: true, rubricVersion, elapsedMs: fixtureElapsedMs,
    structuralCheck: "passed", reviewerDecision: "pending"
  });
} catch (error) {
  console.error({
    fixture: true, rubricVersion, elapsedMs: fixtureElapsedMs,
    structuralCheck: "failed",
    reason: error instanceof Error ? error.message : String(error)
  });
  throw error;
}
```

Notice the duplicate check. Checking that an array contains two scores would also accept two maintenance rows and no communication row. That's a concrete pitfall: a superficially complete answer can silently omit half the rubric. Version checking catches another boundary mistake, where a response generated against old instructions reaches a newer review screen.

The quote check is deliberately modest. Finding an exact substring establishes that the quoted text exists. It doesn't establish that the text warrants 3 points, or that the rubric measures the job fairly. The fixture's points demonstrate the contract; they are not recommended hiring judgments.

Instrument that distinction.

For real attempts, measure elapsed time around the scoring operation and label it separately from any catalog lookup. Keep structural failures, reviewer rejections, and reviewer acceptances as separate counters. Compare latency distributions within the same rubric version and review outcome; a quick stream of unusable proposals shouldn't improve the dashboard's headline. Include retries in the user-visible elapsed time, because reviewers wait through them too. Keep raw candidate evidence out of aggregate metrics and use a review-record identifier to connect an investigation to the application record.

Start with a review queue. A structural failure should produce a clear validation reason rather than a score displayed as complete. An acceptable structure should enter the queue as pending, exactly as the fixture does. Human acceptance is a later event. This separation makes an alert actionable: an increase in malformed output calls for inspecting the response boundary, while an increase in reviewer rejection calls for examining scoring quality and rubric interpretation.

## Limits of the selection

A shared API reduces integration work; it doesn't make prompts, quality, or response time identical across OpenAI, Claude, and Gemini. Advanced native features may outgrow the common interface. Reassess the boundary when a required feature does, and keep image generation and speech outside this text-only decision.

This note supplies no measured quality ranking or latency target. The next decision needs paired candidate examples, a fixed rubric, and reviewer judgments. Until those exist, choose the integration shape for portability and treat model selection as provisional.

## Further reading

References for checking the direct-interface options and understanding tokenization; none establishes a candidate-scoring quality ranking:

- OpenAI API reference: https://platform.openai.com/docs/api-reference
- Claude API documentation: https://docs.anthropic.com/en/api/overview
- Gemini API documentation: https://ai.google.dev/gemini-api/docs
- Official BPE tokenizer library: https://github.com/openai/tiktoken
