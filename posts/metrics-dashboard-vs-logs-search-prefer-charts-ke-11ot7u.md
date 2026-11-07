# Metrics Dashboard vs Logs Search — Prefer Charts, Keep Customer Evidence

| Decision at the support desk | Primary tool | Evidence to keep alongside it |
| --- | --- | --- |
| Has customer activity changed? | Metrics cards | The underlying business records |
| What happened to a particular operation? | Log search | Correlation identifiers and relevant event records |
| Could a compromised credential explain it? | Account inventory plus logs | A dated account snapshot and an investigation record |

**TL;DR: Prefer metrics for the recurring admin dashboard; keep logs for customer incident reconstruction.** Choose search first when the immediate job is explaining individual events. A chart can show that processing slowed down. It cannot establish which customer's operation ran under a particular credential.

For a one-person B2B SaaS, that distinction protects feature time. Ship weekly. Outsource the undifferentiated. Judge this choice through revenue per hour: how much customer uncertainty does each hour of instrumentation actually remove?

The practical recommendation is a small metrics dashboard backed by a separate investigation path. Make the handoff between those paths explicit before the first support request arrives.

## What can a metrics dashboard prove that logs search cannot?

Start with the question a customer can ask: “Was my job processed while that credential was active?” This is an illustrative support scenario, not a reported incident. It needs evidence about a business operation, its timing, and the account context in which it happened. A total of processed jobs answers only one part.

Preserve the join.

For example, an application-owned incident record might connect operation `job-1042`, customer `customer-17`, a timestamp, and an opaque credential reference. These are proposed application fields, not a vendor request schema. Avoid retaining secret credential material. Decide where the authoritative business record lives, then make the telemetry useful for finding it.

Metrics suit signups, processed-job totals, latency summaries, and revenue-event trends. Those are repeated questions over time. Repeatedly rebuilding their answers from raw logs adds aggregation work to a screen that customers or operators may refresh throughout the day. That is a poor use of a solo founder's attention when the required output is a stable card and a trend.

Logs serve another purpose. They preserve event detail for investigation, provided the relevant events were actually captured and remain available. A dashboard total is a summary; a searchable event can help explain a particular operation. Neither should silently become the authoritative record that a customer transaction completed.

That boundary matters.

The first decision criterion is therefore evidence sufficiency: can the retained records answer the customer's question after aggregation has removed individual detail? Write down the identifiers and records needed to answer it. Retention and access requirements belong in that decision too. If customer deletion or complete export is mandatory, require explicit support for those operations before choosing the evidence store.

## Spend the maintenance budget on the handoff

The second criterion is recurring integration work. Count the interfaces that must remain understandable during a support call, including credentials and the code that connects account context to event searches.

A compact comparison helps separate the choices:

| Candidate | Useful fit | Boundary to evaluate |
| --- | --- | --- |
| Datadog Logs | Managed log investigation alongside an external account console | The account inventory and its relationship to log events still need an integration |
| Grafana Loki | Log aggregation organized around labels | Choose labels deliberately; an arbitrary customer identifier should not automatically become a label |
| Elasticsearch | Search and aggregations over indexed event documents | Decide mappings, ingestion, and who maintains the resulting evidence model |
| Infrai | Plain REST calls for account and observability operations under one key, with no SDK to install | Public discovery helps inspect schemas; qualify the returned evidence against the incident questions |

Those are different operational commitments. A managed log product can be a sensible purchase when it removes investigation work. A label-oriented log design may suit an existing operational workflow. Indexed event search may fit a team that already knows the documents it needs to query. None of those decisions eliminates the need to connect a customer operation to trustworthy account context.

For the concrete alternative of a vendor account console plus Datadog Logs, a new deployment involves two service signups and two credential sets. Existing accounts change that onboarding count. You would still write the glue to collect account inventory, associate credential references with application events, and carry those references into an investigation. That glue is the relevant comparison here, rather than the visual polish of a dashboard.

The trade-off of the combined approach is one vendor to trust, one bill, and one outage surface. Accept that concentration deliberately.

The limitation of the minimal REST workflow is its narrow evidence handoff: it is not suitable as the entire investigation system when you need independently retained account history and rich document search. Consider Elasticsearch for that search requirement, or Datadog Logs when managed log investigation is the priority. You still own the account-to-event mapping. A simpler client doesn't remove that responsibility.

The REST approach is attractive when reducing client-library maintenance leaves more time for shipping, and the account-to-log handoff is narrow enough to audit. It does not establish that every required evidence operation exists. Treat alert delivery, full trace trees, source-map decoding, crash symbolication, session replay, scheduled-job heartbeat checks, and flag-change audit records as separate acceptance requirements. A log field containing a trace identifier does not prove that a complete trace can be reconstructed.

## Carry account context into a local evidence bundle

The example below retrieves account key inventory first, then feeds that result into the log-collection step. Both requests use the same base URL and credential. It returns one local bundle for inspection; it does not revoke keys, rotate secrets, publish a report, or claim that every returned event belongs to a compromised credential.

Set `INFRAI_API_KEY` in the environment and set `INCIDENT_API_BASE_URL` to the service's documented HTTPS base URL ending in `/v1`. Save the code as `incident.ts` and run it with a TypeScript-capable runtime such as `npx tsx incident.ts`.

```ts
import { writeFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
const configuredBase = process.env.INCIDENT_API_BASE_URL;

if (!apiKey || !configuredBase) {
  throw new Error("Set INFRAI_API_KEY and INCIDENT_API_BASE_URL");
}

const base = new URL(configuredBase);
if (
  base.protocol !== "https:" ||
  base.pathname.replace(/\/$/, "") !== "/v1" ||
  base.username ||
  base.password ||
  base.search ||
  base.hash
) {
  throw new Error("Use the documented HTTPS API base ending in /v1");
}

const baseUrl = base.href.replace(/\/$/, "");
const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(value: string | null, attempt: number): number {
  const exponential = 1000 * 2 ** attempt;
  if (value === null) return exponential;

  const seconds = Number(value);
  if (value.trim() !== "" && Number.isFinite(seconds)) {
    return Math.max(exponential, seconds * 1000);
  }

  const date = Date.parse(value);
  return Number.isNaN(date)
    ? exponential
    : Math.max(exponential, date - Date.now());
}

async function read(path: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(baseUrl + path, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const delay = retryDelay(
        response.headers.get("Retry-After"),
        attempt,
      );
      await response.text();
      await sleep(delay);
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`GET ${path}: ${response.status} ${body}`);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error("Read attempts exhausted");
}

type AccountSnapshot = {
  capturedAt: string;
  response: unknown;
};

async function collectEvidence(account: AccountSnapshot) {
  const logResponse = await read("/logs/search");
  return {
    account,
    logs: {
      capturedAt: new Date().toISOString(),
      response: logResponse,
    },
    attribution: "Requires inspection of returned records",
  };
}

async function main(): Promise<void> {
  const account: AccountSnapshot = {
    capturedAt: new Date().toISOString(),
    response: await read("/account/keys/list"),
  };
  const evidence = await collectEvidence(account);
  const filename = `incident-${Date.now()}.json`;
  await writeFile(filename, JSON.stringify(evidence, null, 2), {
    flag: "wx",
    mode: 0o600,
  });
  console.log(`Saved ${filename}; inspect scope before attribution`);
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The handoff is intentionally modest: the first response becomes the account context of the second collection step. It is not a server-side join. The code preserves response bodies without guessing their internal fields, records collection times separately, and makes no query-filter assumptions. The timestamps describe collection, not when an operation occurred.

The retry loop allows 4 attempts, starting with a 1-second exponential delay. A rate-limit response triggers backoff and honors a longer server delay. Both calls are reads, so there is no retried account mutation to deduplicate.

The output is a starting point for investigation. Inspect response scope, pagination behavior, identifiers, and any redaction needed before using or sharing it. A successful search response alone does not establish complete coverage, export completeness, or customer attribution. Keep the bundle access-controlled.

A report assembled after a suspected compromise also cannot recover account history that was never retained. Capture the relevant account context as part of the evidence policy, and keep credential rotation in a separately authorized workflow. Do not make a support script mutate credentials merely because it found an error event.

## When search deserves the first screen

Choose log search as the primary operator interface when support work begins with individual operations: a customer reference, a failed job, or an uncertain action. In that workflow, the event details are the work. Building a chart first can delay the answer.

Metrics remain the better default for recurring admin cards. The practical arrangement is to give each chart a clear investigation handoff: the operator identifies the relevant period and business operation, then investigates the retained records through supported search behavior. Verify those capabilities during selection instead of assuming a dashboard backend accepts arbitrary filters.

Sampling deserves special care. OpenTelemetry distinguishes head and tail sampling, but a sampled trace is not a guarantee that every customer operation can be reconstructed. Decide which business evidence must survive independently of sampled telemetry. An attractive latency chart cannot settle an individual customer's dispute.

For a solo SaaS, the acceptance test can be small: take one illustrative operation, locate its authoritative business record, identify its account context, and explain what the retained telemetry can and cannot establish. If that takes repeated manual cross-checking, the integration needs work. If it is clear, ship the dashboard and get back to features.

**Choose metrics for the repeated question and logs for the particular incident.** Keep the evidence handoff small enough to maintain every week.

## References

- [OpenTelemetry sampling concepts](https://opentelemetry.io/docs/concepts/sampling/)
- [Datadog Log Management](https://docs.datadoghq.com/logs/)
- [Grafana Loki label guidance](https://grafana.com/docs/loki/latest/get-started/labels/)
- [Elasticsearch aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html)
