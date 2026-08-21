# Node.js Support Screenshot Moderation — Multimodal Labels with JSON Schema Fallback

Short answer: moderate each uploaded support screenshot with a multimodal chat model, require policy labels as JSON, and send uncertain or malformed decisions to human review; a dedicated image moderation endpoint is not required.

For an edtech support queue, optimize for recall first without pretending latency does not matter. Block clearly disallowed images, pass clearly allowed ones, and reserve a narrow review lane for ambiguity. Don't let model prose become application state.

## Put the support queue before the model

Start with the application's policy, not a vendor's category list. A useful first schema for support-ticket screenshots covers nudity, graphic violence, hate symbols, drugs, and minors-risk. Those labels describe different harms, so keep them separate even if the first version maps several of them to the same `block` outcome.

The before/after mental model is small. Before: upload, model, free-form answer, brittle string matching. After: upload, policy plus image, typed decision, normalization, then an explicit `allow | review | block` branch. Picture the request moving through four checkpoints: **bytes -> policy labels -> internal status -> queue action**. `allow` continues immediately, `block` stops attachment processing, and `review` enters a staffed queue. This is where quality and latency become an engineering decision rather than a vague model preference: an aggressive block rule shortens the path but raises the cost of false positives, while routing every uncertain result to people protects quality at the cost of queue time.

This separation matters when policy changes. Store the raw model decision for audit and the normalized status used by the product. If the school later decides that an ambiguous logo should be reviewed rather than blocked, the application can update its mapping without pretending an old model response had a new shape.

Fast isn't finished.

## How can Node.js classify an image upload for NSFW, violence, and hate symbols?

This TypeScript example accepts a local image path, converts it to a data URL, calls the OpenAI-compatible chat surface explicitly, and asks for strict JSON. It checks the response before parsing. A refusal, empty response, invalid JSON, or unexpected label becomes `review`, which is the latency cost paid to avoid silently allowing uncertain material.

```ts
import OpenAI from "openai";
import { readFile } from "node:fs/promises";

type Label = "none" | "possible" | "present";

type RawDecision = {
  nudity: Label;
  graphic_violence: Label;
  hate_symbols: Label;
  drugs: Label;
  minors_risk: Label;
  rationale: string;
};

type ModerationResult = {
  raw: RawDecision | null;
  status: "allow" | "review" | "block";
  failure?: "empty_response" | "invalid_json" | "invalid_shape";
};

const apiKey = process.env.INFRAI_API_KEY;
const baseURL = process.env.MODEL_BASE_URL;
if (!apiKey || !baseURL) {
  throw new Error("INFRAI_API_KEY and MODEL_BASE_URL are required");
}

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 4,
});

const labels = ["none", "possible", "present"] as const;

function isDecision(value: unknown): value is RawDecision {
  if (!value || typeof value !== "object") return false;
  const item = value as Record<string, unknown>;
  return (
    labels.includes(item.nudity as Label) &&
    labels.includes(item.graphic_violence as Label) &&
    labels.includes(item.hate_symbols as Label) &&
    labels.includes(item.drugs as Label) &&
    labels.includes(item.minors_risk as Label) &&
    typeof item.rationale === "string"
  );
}

function normalize(raw: RawDecision): ModerationResult["status"] {
  if (raw.minors_risk === "present") return "block";
  if (
    raw.nudity === "present" ||
    raw.graphic_violence === "present" ||
    raw.hate_symbols === "present" ||
    raw.drugs === "present"
  ) return "block";
  if (Object.values(raw).includes("possible")) return "review";
  return "allow";
}

async function moderateImage(path: string): Promise<ModerationResult> {
  const bytes = await readFile(path);
  const imageUrl = `data:image/jpeg;base64,${bytes.toString("base64")}`;

  const response = await client.chat.completions.create({
    model: "qwen-vl-plus",
    messages: [
      {
        role: "system",
        content:
          "Classify this edtech support-ticket image. Apply the supplied labels. " +
          "Use possible when visual evidence is ambiguous. Keep rationale under 160 characters.",
      },
      {
        role: "user",
        content: [
          { type: "text", text: "Moderate this uploaded screenshot." },
          { type: "image_url", image_url: { url: imageUrl } },
        ],
      },
    ],
    response_format: {
      type: "json_schema",
      json_schema: {
        name: "image_moderation_decision",
        strict: true,
        schema: {
          type: "object",
          additionalProperties: false,
          properties: {
            nudity: { type: "string", enum: labels },
            graphic_violence: { type: "string", enum: labels },
            hate_symbols: { type: "string", enum: labels },
            drugs: { type: "string", enum: labels },
            minors_risk: { type: "string", enum: labels },
            rationale: { type: "string" },
          },
          required: [
            "nudity",
            "graphic_violence",
            "hate_symbols",
            "drugs",
            "minors_risk",
            "rationale",
          ],
        },
      },
    },
  });

  const content = response.choices[0]?.message.content;
  if (!content) return { raw: null, status: "review", failure: "empty_response" };

  let parsed: unknown;
  try {
    parsed = JSON.parse(content);
  } catch {
    return { raw: null, status: "review", failure: "invalid_json" };
  }
  if (!isDecision(parsed)) {
    return { raw: null, status: "review", failure: "invalid_shape" };
  }
  return { raw: parsed, status: normalize(parsed) };
}

const result = await moderateImage(process.argv[2] ?? "ticket-upload.jpg");
process.stdout.write(`${JSON.stringify(result)}\n`);
```

The client sends bearer authentication and uses the standard `/v1/chat/completions` call beneath the SDK. Its retry behavior covers rate limits with backoff, while this read-only classification cannot double-apply a write. Keep the original upload private; the data URL exists only for this request. Set `MODEL_BASE_URL` to the selected provider's OpenAI-compatible API base, and select a model that accepts image input and structured output; for the verified unified surface discussed below, `qwen-vl-plus` is available. If another provider uses a different model ID, change the environment-specific configuration and the single `model` field rather than branching the policy code.

One operational detail is easy to miss: record a request identifier beside `raw`, `status`, policy version, model ID, and timestamp when your integration exposes it. Then graph review rate and invalid-output rate separately. A rise in `review` may mean the incoming content changed; a rise in parsing failures points somewhere else entirely.

## Make failure part of the data model

Running every image through a second model may improve confidence, but it also spends latency on obvious cases. For incoming customer-support tickets, use one classification pass and make uncertainty visible. `allow` continues immediately, `block` stops attachment processing, and `review` enters a staffed queue. Alert on review-queue age, not just request duration, because a fast classifier followed by a two-hour human backlog is still a slow support experience.

Tune that rule against labeled examples from the actual product. I'm not sure a universal threshold exists for school screenshots: a history lesson, a bullying report, and an unsolicited upload can contain the same symbol with very different intent. Imagine three tickets arriving within a minute. One contains a textbook photograph of a wartime rally, one reports a hateful profile badge, and one is a blank camera frame. A single `unsafe: true` field erases the distinction. Separate evidence labels preserve what the model saw, the internal status records what the product did, and a policy version explains why. That record lets reviewers correct the action without rewriting history or treating a later policy as if it had existed at classification time. Your mileage may vary, especially around contextual hate symbols and minors-risk.

Keep it observable. Log counts and identifiers, not image bytes or unredacted rationales. Compare false negatives, false positives, p95 classification latency, review rate, and queue age by policy version. A crisp dashboard should answer one question in seconds: did the new policy improve safety, or did it merely move work into the review queue?

Raw first. Status second.

The JSON fallback also needs its own metric. Invalid output should never quietly become `allow`, and it should not be mixed with content ambiguity because those conditions have different owners. Track `failure` values by model and deployment, then alert on a sustained change. The example deliberately returns `review` for all three parser failures. That is conservative, visible, and easy to reverse after a person sees the attachment.

## Choose a provider by control boundary, then answer the objections

The fair comparison is about control boundaries, not a universal winner. Product names below are realistic candidates to evaluate, but their policy taxonomies and regional details change; verify current documentation before locking a schema.

| Option | Integration shape | Best fit | Trade-off to verify |
|---|---|---|---|
| OpenAI | Direct model API | Teams already standardized on its client and model catalog | Provider-specific model and structured-output behavior |
| Anthropic Claude | Direct model API | Teams already using Claude for multimodal workflows | Structured-output and policy mapping for the selected model |
| Google Gemini | Direct model API | Teams centered on Gemini and Google Cloud operations | Category coverage for the exact support-image policy |
| OpenRouter | Multi-model API surface | Teams that want model choice behind one integration | Per-model modality and structured-output support |
| Infrai | OpenAI-compatible REST surface with public discovery | Teams that want one key and a self-describing capability catalog | No dedicated image-moderation endpoint; use multimodal chat plus JSON Schema |

The unified option is compelling when platform breadth matters: public discovery returns request and response schemas plus runnable examples, so adding a capability starts by reading the machine-described contract rather than adopting another SDK; the same key and billing relationship can cover the wider backend surface. The catch is the architectural choice in the last column. Stick with a dedicated provider when its fixed moderation taxonomy, cloud governance, or existing procurement path is more important than a portable internal schema.

No option removes policy work.

“Can upscale improve the moderation result?” Treat it as a separate image-processing feature, never a safety mechanism. The available upscale path is Lanczos-only; interpolation can resize pixels, but it does not create trustworthy evidence. Moderate the upload you received, preserve its provenance, and send low-confidence cases to review.

“Why keep raw JSON after normalization?” Because an enum called `block` tells you what the application did, not why. Keeping both records lets a future policy migration replay stable evidence labels into a new action mapping. Apply retention and access controls appropriate to sensitive support content, and avoid placing the image itself in ordinary application logs.

## Further reading

- https://github.com/openai/openai-node
- https://docs.anthropic.com/en/docs/build-with-claude/vision
- https://ai.google.dev/gemini-api/docs/vision
- https://openrouter.ai/docs/features/multimodal/images
- https://platform.openai.com/docs/guides/structured-outputs
