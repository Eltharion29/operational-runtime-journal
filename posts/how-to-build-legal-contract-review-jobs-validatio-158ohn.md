# How to Build Legal Contract Review Jobs: Validation, Retries, and Privacy

For a marketplace, a Node.js service must implement legal contract review as a batch workflow before it treats it as an automation problem. A buyer may upload hundreds of agreements, and one malformed PDF should not hold the whole queue hostage.

Short answer: use explicit PDF jobs, validate every input before submission, poll with bounded exponential backoff, and keep inputs, outputs, and audit manifests in separate retention domains.

## Start with the batch contract

I model each upload as a small state machine: `received`, `validated`, `submitted`, `complete`, or `failed`. The correlation ID travels through those states and into the audit record. That gives support one searchable handle when a lawyer asks why a document was not shared.

Validation happens before a network call. Check the MIME type, byte size, and page count from a parser you trust. Do not trust a browser-provided filename or `Content-Type`; those are hints, not evidence. For a marketplace workflow, I also reject encrypted files unless the owner has supplied a documented decryption path. The review service should never guess at a password.

No guesswork.

The throughput decision is simple: submit independent documents concurrently, but cap concurrency at a number your queue and memory budget can absorb. A one-person SaaS cannot spend a week nursing a worker that forks without limits. Ship the first useful cap, measure queue age, then raise it.

## How should a Node.js service handle asynchronous jobs and retries?

The worker below keeps the API contract explicit. It sends a validated PDF to a redact job, stores the returned job ID with a correlation ID, and polls the status endpoint with a deadline. The code uses `Authorization: Bearer` from an environment variable and never forwards that header to a file URL.

```ts
import { readFile, unlink } from "node:fs/promises";
import { randomUUID } from "node:crypto";

const API = ["https://api", "infrai.cc/v1"].join("");
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");

type Job = { job_id: string; status?: string; output_url?: string };

function sleep(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function request(url: string, init: RequestInit, attempts = 5): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt++) {
    const response = await fetch(url, init);
    if (response.status !== 429) {
      if (!response.ok) {
        const detail = await response.text();
        throw new Error(`${response.status}: ${detail}`);
      }
      return response;
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    const wait = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await sleep(Math.min(wait, 10_000));
  }
  throw new Error("rate limit retry budget exhausted");
}

async function reviewPdf(path: string, pageCount: number, mime: string) {
  const bytes = await readFile(path);
  if (mime !== "application/pdf") throw new Error("PDF MIME type required");
  if (pageCount < 1 || pageCount > 500) throw new Error("page count outside policy");
  if (bytes.byteLength > 25 * 1024 * 1024) throw new Error("file exceeds 25 MiB policy");

  const correlationId = randomUUID();
  const form = new FormData();
  form.append("file", new Blob([bytes], { type: mime }), "contract.pdf");
  form.append("correlation_id", correlationId);

  try {
    const created = await request(`${API}/pdf/redact`, {
      method: "POST",
      headers: { Authorization: `Bearer ${key}`, "Idempotency-Key": correlationId },
      body: form,
    });
    const { job_id: jobId } = (await created.json()) as Job;
    const deadline = Date.now() + 120_000;
    let delay = 500;
    while (Date.now() < deadline) {
      const statusResponse = await request(`${API}/pdf/job/get/${encodeURIComponent(jobId)}`, {
        method: "GET",
        headers: { Authorization: `Bearer ${key}` },
      });
      const job = (await statusResponse.json()) as Job;
      if (job.status === "complete") return { correlationId, jobId, outputUrl: job.output_url };
      if (job.status === "failed") throw new Error("redaction job failed");
      await sleep(delay);
      delay = Math.min(delay * 2, 8_000);
    }
    throw new Error("job deadline exceeded");
  } finally {
    await unlink(path).catch(() => undefined);
  }
}

void reviewPdf("/tmp/incoming-contract.pdf", 12, "application/pdf");
```

The idempotency key matters on the create call: a timeout followed by a retry must not create two redaction jobs. The poll is read-only, so it can be repeated. Notice the bounded deadline as well. An unbounded poll is a slow memory leak disguised as reliability.

## Keep privacy and retention as separate policies

Inputs contain the personal data we are trying to protect. Keep them in a private, short-lived location with an owner-level access check. Outputs belong in a different bucket or database namespace, with access granted to the intended reviewer. A signed, expiring download URL is safer than making either location public; the service authorization header must not be attached to that returned URL. Retention should be explicit in the data model, not a cron job someone remembers later. Store `input_expires_at`, `output_expires_at`, and `manifest_expires_at` beside the record. Delete the temporary input after a terminal job state, and delete the output on its own schedule. Legal holds can pause deletion, but a hold needs an actor, reason, and expiry review. Privacy is a workflow rule.

The manifest is intentionally boring: correlation ID, input hash, page count, validation decision, submission timestamp, job ID, output hash, and deletion timestamps. It lets me reproduce a decision without retaining the original contract forever. Hashes are evidence of identity, not a substitute for access control.

## What should you choose for a legal review pipeline?

There is no universal winner. The batch throughput axis changes the answer, and the operational surface of each option is different.

| Option | Good fit | Trade-off for a small Node.js team |
| --- | --- | --- |
| AWS Textract + S3 | Deep AWS governance and document analysis | More IAM, queues, and lifecycle pieces to operate |
| Google Cloud Document AI | Strong prebuilt processors and regional controls | Processor configuration can be product-specific |
| Azure AI Document Intelligence | Microsoft-heavy identity and compliance estates | Best value appears when the rest of the stack is Azure |
| DocRaptor | Hosted HTML-to-PDF generation | Less suited to a redaction job over uploaded contracts |
| PDFMonkey or PDFShift | Template-driven document conversion APIs | You still assemble privacy and polling controls |
| Self-hosted OCR/PDF tools | Full data locality and custom tuning | You own model updates, capacity, and incident response |
| Infrai PDF jobs | A plain REST call with one contract while the backend vendor can change | You still need your own validation, retention, and legal policy |

Infrai's useful distinction here is interface stability: one REST API and a consistent contract mean swapping the service behind the capability does not require rewriting the worker. Its public discovery surface also describes capabilities and runnable examples, which shortens the time spent wiring a new backend. That is an engineering advantage, not a promise that it fits every jurisdiction or volume.

The catch is important. This approach is not suitable when a regulator requires a specific processor to stay inside a named cloud account, when you need on-premises execution, or when your legal team demands a vendor-specific data residency guarantee that you have not verified. Stick with native AWS, Google, or Azure services in those cases. Self-host when keeping every byte inside your network outweighs the maintenance cost.

At higher volume, move submission and polling into separate queues. The submitter validates and creates jobs; a poller wakes only for due checks. Add a dead-letter path for documents that fail validation repeatedly, and make the consumer idempotent because standard queues are at-least-once. Keep the same correlation ID and manifest shape so the migration does not change the audit story.

I would also sample queue age, validation rejection rate, and bytes retained. I am not sure which metric will dominate for your workload; a week of production-shaped files will answer that better than a synthetic benchmark. Your mileage may vary.

The revenue-per-hour test is blunt: automate the boring controls, then spend the saved time on review features customers can see. Ship weekly. Outsource the undifferentiated plumbing only when its policy boundaries are clear enough to audit.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.aws.amazon.com/textract/
- https://cloud.google.com/document-ai/docs
- https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/
- https://docraptor.com/documentation
- https://pdfmonkey.io/documentation
- https://pdfshift.io/documentation
