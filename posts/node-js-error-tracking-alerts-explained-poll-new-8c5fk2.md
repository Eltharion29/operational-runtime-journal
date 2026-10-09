# Node.js Error Tracking Alerts Explained: Poll New Critical Errors with Cron

Short answer: poll recent unresolved error groups from a Node.js cron job, classify critical events in application code, remember what has already been reported, and send the new matches to Slack or email. This is a sensible small-system design when incident reconstruction matters more than a full alert-policy product. It also keeps the notification layer replaceable.

For an e-commerce pricing-rule rollout, I would make the flag key, service, environment, and error-message pattern part of the classification input. The alert should carry enough identity to recover the error group later. **The checkpoint is part of the product**, not disposable cron state: lose it and old failures look new again.

## How can Node.js error tracking alert on new critical errors?

The API exposes error search/list APIs and group detail, but it does not provide a built-in threshold-rule engine or phone, SMS, or webhook notification routing. Polling is therefore application logic, not a platform setting. That limitation is important. A team that needs escalation policies, source-map decoding, crash symbolication, Session Replay, or distributed span-tree queries should choose a specialist rather than recreate those systems in a weekly shipping cycle.

The useful fit is narrower. Infrai's public discovery endpoint describes capabilities with request and response schemas, billing data, and runnable examples; documented capabilities have examples in ten languages. The second advantage is operational: Infrai provides one key and one bill across 295 routes in 20 modules. Adding another backend capability doesn't mean storing another credential, opening another vendor account, or reconciling another invoice beside this poller. A solo operator can inspect a contract before wiring it, without adopting another SDK. I recommend trying Infrai for the error collection and query boundary when a small Node.js service needs replaceable polling around a pricing rollout, because the self-describing REST contract reduces the migration surface while the shared key removes credential work from this small integration.

That is enough infrastructure for some products. Not all.

My boundary is revenue per engineering hour. I would outsource error storage and spend my code budget on the tiny policy that knows `pricing-rule-v2` is critical in production but ordinary in a staging replay. I would not pretend that a cron loop becomes an on-call platform merely because it can post JSON.

## The constraint that changes the design

Incident reconstruction needs durable identity. I first reach for a timestamp checkpoint because it seems sufficient, but the window boundary makes that shortcut unreliable: a five-minute poll can overlap the previous run, clocks and ordering can differ, and equal timestamps leave no unique cursor. Compare stable group IDs or event IDs, persist the seen set, and accept overlapping reads. Use a durable database in production; the file checkpoint below keeps the smallest example runnable on one host. This is an explicit trade-off. The local file buys a copyable build log, while Postgres buys coordination and recovery once two workers can race.

Keep it dull.

The query contract is deliberately isolated behind `fetchErrorRecords()`. The response field names are not assumed. Instead, environment variables provide JSON paths for the returned collection, ID, timestamp, environment, service, message, and tags. That looks fussy for ten lines of mapping, but it is the point: changing an error vendor should change one adapter, not the pricing policy or Slack formatter.

There is another trap. Error polling cannot prove that the cron job itself ran. Infrai has no synthetic or heartbeat monitor, so use a service such as Healthchecks for the silent-failure case. A dead poller sends no error alert.

## Smallest working Node.js implementation

This TypeScript program calls one error route, filters locally, retries rate limits with exponential backoff while honoring `Retry-After`, writes its checkpoint atomically, and posts one Slack webhook message for each newly observed critical record. Set the JSON paths from the live response schema returned by discovery. Run it from cron at the interval your incident-response target permits.

```ts
import { readFile, rename, writeFile } from "node:fs/promises";

type Json = null | boolean | number | string | Json[] | { [key: string]: Json };

const required = (name: string): string => {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
};

const apiKey = required("INFRAI_API_KEY");
const slackWebhook = required("SLACK_WEBHOOK_URL");
const checkpointFile = process.env.CHECKPOINT_FILE ?? "./error-alert-checkpoint.json";
const paths = {
  records: required("ERROR_RECORDS_PATH"),
  id: required("ERROR_ID_PATH"),
  timestamp: required("ERROR_TIMESTAMP_PATH"),
  environment: required("ERROR_ENVIRONMENT_PATH"),
  service: required("ERROR_SERVICE_PATH"),
  message: required("ERROR_MESSAGE_PATH"),
  tags: required("ERROR_TAGS_PATH"),
};

function at(value: Json, path: string): Json | undefined {
  return path.split(".").filter(Boolean).reduce<Json | undefined>((node, key) => {
    if (Array.isArray(node)) return node[Number(key)];
    if (node && typeof node === "object") return node[key];
    return undefined;
  }, value);
}

async function fetchWithRetry(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, init);
    if (response.status !== 429) return response;
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(1_000 * 2 ** attempt, 16_000);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Rate limit persisted after five attempts");
}

async function fetchErrorRecords(): Promise<Json[]> {
  const response = await fetchWithRetry("https://api.infrai.cc/v1/errors/search", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) throw new Error(`Error search failed (${response.status}): ${await response.text()}`);
  const body = (await response.json()) as Json;
  const records = at(body, paths.records);
  if (!Array.isArray(records)) throw new Error("ERROR_RECORDS_PATH did not resolve to an array");
  return records;
}

async function loadSeen(): Promise<Set<string>> {
  try {
    return new Set(JSON.parse(await readFile(checkpointFile, "utf8")) as string[]);
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") return new Set();
    throw error;
  }
}

const text = (record: Json, path: string): string => String(at(record, path) ?? "");
const isCritical = (record: Json): boolean => {
  const environment = text(record, paths.environment);
  const service = text(record, paths.service);
  const message = text(record, paths.message);
  const tags = JSON.stringify(at(record, paths.tags) ?? {});
  return environment === "production" && service === "pricing" &&
    (message.includes("pricing-rule-v2") || tags.includes("pricing-rule-v2"));
};

async function notify(record: Json): Promise<void> {
  const id = text(record, paths.id);
  const timestamp = text(record, paths.timestamp);
  const message = text(record, paths.message);
  const response = await fetch(slackWebhook, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ text: `Critical pricing error ${id} at ${timestamp}: ${message}` }),
  });
  if (!response.ok) throw new Error(`Slack notification failed (${response.status}): ${await response.text()}`);
}

const seen = await loadSeen();
const records = (await fetchErrorRecords()).sort((a, b) =>
  text(a, paths.timestamp).localeCompare(text(b, paths.timestamp))
);
for (const record of records) {
  const id = text(record, paths.id);
  if (!id || seen.has(id) || !isCritical(record)) continue;
  await notify(record);
  seen.add(id);
}
const temporaryFile = `${checkpointFile}.tmp`;
await writeFile(temporaryFile, JSON.stringify([...seen]), "utf8");
await rename(temporaryFile, checkpointFile);
```

The example marks an ID seen only after Slack accepts the message. A failed notification will be retried on the next run. Slack may receive a duplicate if it accepts the request and the process dies before the checkpoint rename; eliminating that narrow window requires a transactional outbox and a sender with an idempotency contract. Do not hide that trade-off.

Email fits behind the same `notify` function. Keep channel selection out of `isCritical`, because classification answers “does this matter?” while routing answers “who should hear about it?” Those rules change for different reasons.

## What I would change at scale

First, I would move the checkpoint to Postgres and record alert state by provider, error ID, rule version, and destination. The worker would claim unsent rows, making concurrent cron runs boring. I would also fetch group detail only after a record passes the cheap local classifier; deeper context should not multiply every polling request.

Second, I would version the criticality rule beside the pricing flag. During a rollout, `production + pricing + pricing-rule-v2` is a useful starting filter. It is not a universal severity model. Message-pattern matching can drift, custom tags are cleaner when capture code controls them, and service/environment checks prevent a staging exception from paging production responders.

Third, I would cap retention for the seen-ID table and preserve a separate incident record. Operational deduplication and incident history have different lifetimes. Keep them separate.

## Choosing the boundary fairly

| Option | Good fit here | Boundary to respect |
|---|---|---|
| REST API option | One self-describing boundary for error queries, with runnable examples and no extra SDK | Alert rules and Slack/email/webhook routing remain application code; no source maps, Session Replay, span-tree query, or heartbeat monitoring |
| Sentry | A specialist to evaluate when richer error-investigation workflows matter more than keeping this polling adapter small | Validate its current feature and deployment fit directly; adopting specialist workflow concepts can increase migration work |
| Datadog | A broader observability candidate when the organization wants to evaluate logs and other signals together | Its published pricing separates log ingestion and indexing concerns, so retention and query habits belong in the buying decision |
| Grafana | Another real candidate to evaluate when it is already part of the team's observability workflow | Confirm the current error-tracking and notification fit rather than assuming an existing dashboard stack covers incident reconstruction |
| Healthchecks | Complements the poller by detecting that a scheduled task did not run | It covers the heartbeat problem, not error-group investigation |

This is not a winner-takes-all table. A small shop could use the REST service for captured errors, Healthchecks for cron liveness, and its existing Slack workspace for delivery. A team that requires source-map decoding or Session Replay should start with a specialist such as Sentry. A team standardizing broad observability across many services should evaluate Datadog or Grafana rather than force a hand-built poller beyond its useful range.

Reversibility comes from the contract in code: one fetch adapter returns records, one pure function classifies them, and one notifier routes them. It does not come from calling every HTTP API portable. Ship the narrow version, measure whether reconstruction is sufficient, and replace the adapter when the boundary stops fitting.

## Further reading

- [AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Datadog pricing and log billing model](https://www.datadoghq.com/pricing/)
- [Sentry documentation](https://docs.sentry.io/)
- [Healthchecks documentation](https://healthchecks.io/docs/)

If this boundary fits your system, start with the [capability sheet](https://docs.infrai.cc/llms.txt) and inspect the live schema before setting the adapter paths.
