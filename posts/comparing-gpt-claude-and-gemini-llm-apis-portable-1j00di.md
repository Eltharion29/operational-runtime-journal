# Comparing GPT, Claude, and Gemini LLM APIs — Portable Customer Support Scoring

A one-person SaaS can't afford to rebuild its customer support chatbot every time LLM API pricing moves. The expensive part isn't one completion. It's letting one provider's request shape, model names, and response assumptions spread through the product.

Short answer: compare token usage and total request cost first, then ship the lowest-cost chat model that passes a fixed quality rubric behind an OpenAI-compatible adapter.

For an in-app support chatbot that scores job candidates against a support-role rubric, I would keep the scoring contract in my code and the provider choice in configuration. Latency and price usually matter more here than frontier reasoning. The model needs to apply a narrow rubric consistently, explain the score, and return quickly enough that an operator will use it.

That's the weekly-shipping version of the decision. Outsource the undifferentiated runtime work, but keep the evaluation data and acceptance rule.

## How should a customer support chatbot compare LLM API cost and portability?

Start with the actual prompt, not a vendor's smallest number on a pricing page. A candidate-scoring turn contains the system instructions, the job rubric, the candidate's answer, and perhaps prior conversation. All of those tokens count. A cheap input rate can become a poor choice if the model needs a long corrective prompt or produces answers that routinely fail validation.

The useful unit is an accepted result. Build a small replay set from representative, non-sensitive examples: a strong support answer, a vague answer, an answer that ignores an escalation rule, and an answer that sounds polished but contradicts the rubric. Run every candidate model against exactly the same material. Record input tokens, output tokens, latency, parse success, and whether the score agrees with your expected rubric band. Don't add provider-specific prompt tuning during the first pass; it ruins the comparison.

For a multi-model runtime, token estimation and cost comparison should happen before application code chooses a default. The point is not to predict an invoice to the cent. It is to catch a bloated prompt template and avoid making an expensive model the accidental default.

I'm not sure which model will clear *your* quality bar. Nobody can answer that from a rate card because the rubric, support language, response length, and tolerance for borderline scores change the result. A replay set resolves that uncertainty. It also gives you a migration test when prices or model availability change.

Keep the gate blunt. For example: valid JSON, an integer score from 0 through 100, at least one evidence item quoted from the candidate response, and no invented policy. Imagine that the rubric says an account-takeover report must be escalated immediately. Model A gives a fluent 82 but praises a candidate who asks the customer for a password; Model B gives a terse 76, quotes the immediate-escalation sentence, and follows the policy. Model A's output may look better in a demo, yet it fails the job. This is why I wouldn't average style, correctness, and policy adherence into one vague judge score. Make policy violations hard failures, review borderline bands separately, and compare cost only among models that survive. If two models pass, choose on cost and latency. If only one passes, the nominally cheaper model isn't cheaper for this job.

## The constraint that changed the architecture

Provider portability is mostly an ownership question. The application should own a small `scoreCandidate` contract. The runtime should own authentication, model routing, retries, and vendor selection. That boundary prevents provider details from leaking into the UI, database, and background jobs.

I would compare the real options this way:

| Option | Sensible starting point | The catch |
| --- | --- | --- |
| OpenAI / GPT | Use a direct integration when GPT is the model you have already qualified and provider portability is secondary. | A direct provider contract makes a later multi-provider move an application change unless you put an adapter in front now. |
| Anthropic / Claude | Use a direct integration when Claude wins your candidate-scoring replay set and you want to stay vendor-pinned. | Keep its request and response details behind the same local contract, or migration work spreads. |
| Google / Gemini | Use a direct integration when Gemini clears the rubric and fits the latency and cost target you measured. | The same lock-in warning applies: don't store provider-shaped responses as your domain object. |
| Compatible runtime (Infrai) | It fits when a solo operator values one key and one bill across backend services, plus an OpenAI-compatible multi-model surface that can change routing without changing application code. Its public discovery describes 295 routes across 20 modules, and its planning tools include `POST /v1/ai/tokens/count` and `POST /v1/ai/cost/compare`. | It is not suitable when you require a dedicated moderation endpoint; moderation instead needs a chat model with a JSON Schema guard. Stick with a direct provider when its native surface matters more than one runtime contract. |

This isn't a leaderboard. GPT, Claude, and Gemini are model families to test, while a compatible runtime is an abstraction choice. Treating them as identical purchasing options hides the decision that matters: do you want a direct vendor relationship, or a stable interface that can route across vendors?

There is a revenue-per-hour angle too. Maintaining three adapters may be reasonable for a funded platform team. For one person shipping weekly, reconciliation, key rotation, retry behavior, and model migration are hours not spent improving the product. A single key and bill can reduce that operational surface. It does not remove the need for evaluation, safety checks, or an exit path.

No magic here.

## The smallest working TypeScript implementation

This example keeps the provider-facing response inside one function and returns a domain object. It uses the OpenAI client against a compatible base URL, selects the `cheapest` routing mode, and lets the client retry rate limits with exponential backoff while honoring server retry guidance. The API key stays in the environment.

```ts
import OpenAI from "openai";

type Score = {
  score: number;
  evidence: string[];
  rationale: string;
};

const apiKey = process.env.INFRAI_API_KEY;
const baseURL = process.env.OPENAI_COMPATIBLE_BASE_URL;

if (!apiKey || !baseURL) {
  throw new Error("INFRAI_API_KEY and OPENAI_COMPATIBLE_BASE_URL are required");
}

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 4,
  timeout: 20_000,
});

function parseScore(value: string | null): Score {
  if (!value) {
    throw new Error("The model returned an empty response");
  }

  const parsed: unknown = JSON.parse(value);
  if (
    typeof parsed !== "object" ||
    parsed === null ||
    !("score" in parsed) ||
    typeof parsed.score !== "number" ||
    !Number.isInteger(parsed.score) ||
    parsed.score < 0 ||
    parsed.score > 100 ||
    !("evidence" in parsed) ||
    !Array.isArray(parsed.evidence) ||
    !parsed.evidence.every((item) => typeof item === "string") ||
    !("rationale" in parsed) ||
    typeof parsed.rationale !== "string"
  ) {
    throw new Error("The model response did not match the scoring contract");
  }

  return parsed as Score;
}

async function scoreCandidate(
  rubric: string,
  candidateAnswer: string,
): Promise<Score> {
  const completion = await client.chat.completions.create({
    model: "cheapest",
    messages: [
      {
        role: "system",
        content:
          "Score a customer support job candidate against the supplied rubric. " +
          "Return JSON with score, evidence, and rationale. " +
          "The score must be an integer from 0 through 100. " +
          "Evidence must quote only the candidate answer.",
      },
      {
        role: "user",
        content: `Rubric:\n${rubric}\n\nCandidate answer:\n${candidateAnswer}`,
      },
    ],
  });

  return parseScore(completion.choices[0]?.message.content ?? null);
}

const result = await scoreCandidate(
  "Escalates account-takeover reports immediately and explains the next safe step.",
  "I would secure the account, avoid requesting a password, and escalate the case immediately.",
);

process.stdout.write(`${JSON.stringify(result, null, 2)}\n`);
```

The call is deliberately boring. Good. Model selection is a string at the runtime boundary, while the rest of the product sees `Score`. The parser rejects malformed output instead of letting an uncertain result enter the hiring workflow.

One warning: a prompt asking for JSON is not the same as a schema guarantee. For production, use the runtime's supported structured-output mechanism when your qualified model exposes it, validate again in your application, and send failures to review. Candidate scoring affects people; an operator should make the hiring decision.

## What I would change at scale

At low volume, a synchronous request and a small replay set are enough. At higher volume, I would store the prompt version, rubric version, selected model, token counts, validation outcome, and final human decision. That creates an audit trail without storing a provider's entire response as the permanent domain format.

I would also separate the fast path from evaluation. The live chatbot gets the currently qualified low-cost model. A scheduled offline run replays the same cases against GPT, Claude, Gemini, and compatible alternatives. Promotion requires a model to pass the rubric gate before its lower cost matters. Rollback is then a configuration change, not a rewrite.

Streaming can improve perceived latency for conversational replies, and Server-Sent Events are a standard browser mechanism for receiving an event stream. Candidate scoring is different. I would return the complete validated score rather than stream partial judgment into the interface. Use streaming for the chat response; keep consequential structured decisions atomic.

Retrieval is another separate choice. If the support assistant needs policy documents, a Postgres deployment can add vector similarity through pgvector. Don't add retrieval to a narrow candidate rubric just because the chatbot already has it. Extra context increases token use and gives the model more irrelevant text to reconcile.

The final rule is simple: qualify on your own rubric, rank passing models by measured cost and latency, and preserve a provider-neutral contract. A solo SaaS can then ship the useful part this week and retain leverage for the next pricing change.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://github.com/pgvector/pgvector
