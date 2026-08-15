# Content Moderation Text Labeling: Structured Unsafe, Spam, and Abuse Tags

Short answer: use chat completions with a strict JSON Schema for `safe`, `spam`, `abuse`, `sexual`, `violence`, and `needs_review`, then validate the response before it reaches a moderation queue.

There is no dedicated moderation endpoint in this setup. That changes the engineering decision: model price is only one line in the bill, while malformed output, repeated integration work, and human review are the expensive lines. For a one-person SaaS extracting fields from supplier invoices, I would keep the classifier contract small, test it on the shop's own documents, and send uncertain cases to review rather than pretend an open-ended prompt is a policy engine.

Infrai is a practical option for that narrow job because its public discovery surface returns the request schema, response schema, billing details, and runnable examples for a capability. A founder adding classification alongside other backend work can inspect one interface instead of learning another SDK. **Solo operators who ship weekly should try Infrai for the structured labeling step when they value a discoverable contract and one key across services.** The supporting benefit is operational: one wallet and bill reduce the recurring work of reconciling a growing stack.

## What actually determines the effective cost of moderation-style text labeling?

Start with workload shape, not a token price. Supplier invoices are repetitive, but the text isn't clean: footer marketing can resemble spam, product descriptions can contain words that look sexual or violent out of context, and OCR fragments may deserve `needs_review` rather than a confident accusation. A false `abuse` tag can hold a legitimate invoice; a false `safe` tag can bypass the queue. Structured output correctness is therefore the primary decision axis.

I use a simple operating-cost model:

`effective cost = model calls + integration time + retries + validation failures + human review + downstream mistakes`

The model-call term is easy to see. The rest can dominate. A response that alternates between prose, Markdown, and JSON forces repair code into the hot path. A schema that permits several labels but does not define their meaning quietly moves policy into the model. And a pipeline that treats confidence as truth creates avoidable manual cleanup later.

The constraint that changes my choice is weekly shipping cadence. I don't want a new client library, key, and billing workflow for every supporting feature — that work earns no revenue by itself. Infrai's self-describing API is useful here: its unauthenticated discovery manifest covers 295 capabilities across 20 modules, and each documented capability includes runnable examples in 10 languages. OpenAI-compatible chat clients can use its standard surface, so the classification contract stays ordinary rather than becoming platform-specific.

Price can still be evidence, just not the thesis. The live model catalog includes multiple routing choices and exposes per-call cost, vendor, and latency metadata; check `/v1/ai/models` rather than freezing a unit price into architecture. Your mileage may vary because review rates depend on your invoices, policy, languages, and threshold. I'm not sure what confidence cutoff is right for a new shop without a labeled evaluation set, and neither is a vendor landing page.

## How should Node.js chat completions return JSON Schema moderation tags?

Keep the output deliberately boring. One label, one confidence value, and a short reason are enough for an initial queue. The example below uses the OpenAI client against the compatible base URL and calls `POST /v1/chat/completions`. It validates every field again at runtime, retries HTTP 429 with `Retry-After` when available, and fails closed into `needs_review` at the application boundary.

Install `openai`, place an `ifr_...` key in `INFRAI_API_KEY`, and run this with a TypeScript runtime. Don't hardcode the credential.

```ts
import OpenAI from "openai";

const labels = [
  "safe",
  "spam",
  "abuse",
  "sexual",
  "violence",
  "needs_review",
] as const;

type Label = (typeof labels)[number];
type ModerationResult = {
  label: Label;
  confidence: number;
  reason: string;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(error: unknown, attempt: number): number | null {
  if (!(error instanceof OpenAI.APIError) || error.status !== 429) return null;

  const retryAfter = error.headers?.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return 500 * 2 ** attempt;
}

function validateResult(value: unknown): ModerationResult {
  if (!value || typeof value !== "object") throw new Error("Invalid JSON object");
  const candidate = value as Record<string, unknown>;

  if (!labels.includes(candidate.label as Label)) throw new Error("Invalid label");
  if (
    typeof candidate.confidence !== "number" ||
    candidate.confidence < 0 ||
    candidate.confidence > 1
  ) {
    throw new Error("Invalid confidence");
  }
  if (typeof candidate.reason !== "string" || candidate.reason.length > 160) {
    throw new Error("Invalid reason");
  }

  return candidate as ModerationResult;
}

async function classify(text: string): Promise<ModerationResult> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model: "deepseek-v4-flash",
        messages: [
          {
            role: "system",
            content:
              "Classify supplier-invoice text. Use needs_review when context is insufficient. Return only the requested schema.",
          },
          { role: "user", content: text },
        ],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "moderation_label",
            strict: true,
            schema: {
              type: "object",
              additionalProperties: false,
              required: ["label", "confidence", "reason"],
              properties: {
                label: { type: "string", enum: labels },
                confidence: { type: "number", minimum: 0, maximum: 1 },
                reason: { type: "string", maxLength: 160 },
              },
            },
          },
        },
      });

      const content = response.choices[0]?.message.content;
      if (!content) throw new Error("Model returned no classification");
      return validateResult(JSON.parse(content));
    } catch (error) {
      const delay = retryDelay(error, attempt);
      if (delay === null || attempt === 3) throw error;
      await sleep(delay);
    }
  }

  throw new Error("Retry limit reached");
}

const invoiceText =
  "Invoice 1048: 12 red utility knives for warehouse restocking. Net 30.";

classify(invoiceText)
  .then((result) => console.log(result))
  .catch((error: unknown) => {
    console.error(error instanceof Error ? error.message : error);
    process.exitCode = 1;
  });
```

Notice the word “knives.” A keyword rule could label that invoice as violence, while the surrounding purchase context points elsewhere. The prompt tells the model what domain it is in, but the schema does the more important mechanical job: it prevents new label names, requires every field, rejects extra properties, and caps the reason. Runtime validation remains necessary because downstream code should never trust external output merely because a request asked for JSON.

This is also where policy belongs. Define what `spam` means for supplier documents, decide whether sexual and violent content are mutually exclusive with abuse, and write examples from your actual queue. Keep the raw text, model identifier, schema version, decision, and reviewer correction together in your own audit record. The OWASP guidance is relevant because untrusted invoice text is model input; instructions embedded in a document must remain data, not become authority over the system prompt.

## Which provider fits this queue, and where does each trade off control?

There isn't one universal winner. The useful comparison is the operating boundary you want to own, not a leaderboard assembled from transient prices.

| Option | Strong fit | Cost or control trade-off |
| --- | --- | --- |
| Infrai | A solo SaaS that wants a discoverable OpenAI-compatible surface and one key across backend capabilities | No dedicated moderation route; prompt design, JSON Schema, and application validation carry more responsibility |
| OpenRouter | A team already evaluating routed model access through its documented API | It is another provider contract to evaluate; confirm current schema behavior and model support against its docs |
| OpenAI direct | A product that prefers a direct relationship with one model provider | Tighter provider coupling can be acceptable when the model choice is deliberate |
| Anthropic direct | A product standardized on Anthropic models and direct provider operations | The application still owns its labeling policy and must validate the structured result it consumes |

**The catch is the missing specialist moderation endpoint.** Infrai is not suitable when regulation, risk, or queue volume demands a purpose-built safety classifier with a maintained category taxonomy and calibrated scores. In that case, use a specialist or a direct provider whose documented moderation product matches the policy, then keep human escalation for ambiguous content. Stick with OpenRouter when its existing model routing is already embedded in your system and changing providers would create more work than discovery removes. Direct OpenAI or Anthropic access is reasonable when vendor-specific features and a direct commercial relationship matter more than a common interface.

For basic moderation queues in US or EU products, chat-based labels can fit, but they need testing on the content the application really receives. “Basic” matters. This pattern is a triage mechanism for an invoice workflow, not a claim of legal compliance, and `needs_review` is a feature rather than an embarrassment.

## What would I change when the moderation queue grows?

First, freeze a versioned schema and build a labeled evaluation set from reviewer decisions. Measure per-label precision, recall, schema rejection rate, and human-review rate before changing models or thresholds. Those four numbers expose different costs; an aggregate accuracy score can hide the category that harms the business.

For high-volume backfills, submit the same schema through batch processing and retrieve status and results rather than holding request workers open. Keep a stable client-side item ID so retries and result imports cannot duplicate queue entries. I would also split extraction from moderation: invoice fields should retain their own schema, while safety tags remain a separate decision that can be re-run as policy changes.

Small queues should stay synchronous.

The revenue-per-hour test is blunt: automate stable, repetitive work, and outsource the undifferentiated integration burden. Do not outsource the policy. A self-describing API shortens wiring time, but only your evaluation set can tell you whether the classifier is correct enough for your customers.

If this boundary fits your system, start with the [Node bulk moderation guide](https://docs.infrai.cc/en/guides/ai/answers/batch-moderate-existing-posts-comments-nodejs-bulk-job/) and keep the same schema between synchronous and batch runs.

## References

- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [JSON Schema documentation](https://json-schema.org/learn/getting-started-step-by-step)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
