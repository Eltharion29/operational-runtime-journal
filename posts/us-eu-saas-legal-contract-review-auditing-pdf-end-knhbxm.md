# US/EU SaaS Legal Contract Review: Auditing PDF Endpoints Against Player Data Exposure

A US/EU gaming SaaS should use PDF endpoints for legal contract review only after it defines how player names, contact details, and account references leave the source document. The shared copy has to hide that data, preserve the pages a lawyer will inspect, and leave a clear record of what was approved and signed.

Short answer: use separate PDF capabilities for inspection, redaction, and final signing, but put them behind one application-owned workflow that controls regional processing, deletion, artifact identity, and audit events. Optimize the reviewer-facing copy for fidelity; keep previews fast by making them disposable derivatives.

This is an operations choice before it is an endpoint choice. For a one-person SaaS, every extra queue, callback, and dashboard competes with shipping the next feature. I want the smallest boundary that can prove which input produced which shared file.

## How should a US/EU SaaS balance PDF endpoint fidelity, latency, privacy, and retention?

Start with two artifacts, because one PDF should not carry every responsibility. The review artifact is a redacted derivative built for quick inspection and annotation. The execution artifact is the final approved document passed to the signing step. They may share source content, but they should never share an ambiguous identity. Give each artifact an opaque ID, record its parent ID, and store its digest with every approval event.

That split makes the trade-off legible. High-fidelity rendering belongs on the path where a reviewer compares page layout, tables, signature boxes, and attached schedules. Low latency matters most for previews and status checks. Privacy and retention apply to both, yet preview artifacts can usually have the shortest application-defined lifetime because they are reproducible. Operational complexity stays bounded when the application owns one state machine instead of teaching every feature about several providers.

The endpoint categories I would evaluate are narrow:

1. An inspection capability that returns page count, document metadata, and enough structure to reject an unexpected input before processing.
2. A transformation capability that creates a redacted review artifact and a stable page preview.
3. A signing capability that accepts only the approved artifact identity and emits an audit record tied to that identity.

These are capabilities, not a demand for three vendors. One service may cover all three. Several services may be necessary when regional processing or signature evidence differs by market. The application contract should stay the same either way.

The catch is latency. A synchronous request looks wonderfully small until a long contract, scanned appendix, or retry occupies a web request. Put transformations behind a job boundary when their duration is uncertain, but don't add a message broker just to feel architectural. A database row with an idempotency key, explicit state transitions, and a worker can be enough at modest volume. Your mileage may vary once concurrent uploads become a real load pattern rather than a planning guess.

## The constraint that changed the build

The signature and audit trail changed the choice. Redaction is a mutation, so the system must make it impossible to approve one byte sequence and later sign another. A filename cannot do that. Neither can a row that merely says `approved: true`. The workflow needs an immutable artifact ID at each boundary, plus an event that connects the source, redaction policy revision, review artifact, approval, and signed result.

Small detail. Big consequence.

For the gaming example, imagine a licensing contract with a player-support export attached as a schedule. The legal team needs the commercial clauses and layout intact, while direct identifiers in the schedule must disappear before the document is shared outside the company. If the redaction step shifts a table onto another page, a reviewer may discuss the wrong clause. If a preview is approved while a separately regenerated PDF is sent for signature, the audit trail answers the wrong question. The workflow therefore freezes the approved review artifact, promotes that exact artifact into the signing stage, and refuses any transition whose parent IDs do not match. This costs a little storage and an extra state transition. It buys a much simpler explanation of what happened.

Retention is part of that state machine — not a weekly cleanup script with no link to document status. Define deadlines for the source, disposable previews, approved artifact, and audit events separately. A deletion request should be idempotent, recorded, and safe to retry. If a contract must remain available under a customer policy, represent that hold explicitly rather than silently skipping deletion. I'm not sure a universal duration can be defensible across every US and EU customer; the contract, data classification, and counsel-approved policy are inputs, so the endpoint evaluation should test configurable retention instead of assuming one magic number.

## The smallest working TypeScript boundary

I keep vendor details outside business logic. The interface below is intentionally boring. It makes artifact identity and region explicit, requires a redaction policy revision, and prevents signing until the caller supplies the approved artifact. The provider adapter can use remote PDF endpoints or an internally operated component; the workflow does not care.

```ts
type Region = "us" | "eu";
type ArtifactId = string;
type JobId = string;

type PdfArtifact = {
  id: ArtifactId;
  parentId: ArtifactId | null;
  digest: string;
  pageCount: number;
};

type AuditEvent = {
  action: "redacted" | "approved" | "signed" | "deletion_requested";
  artifactId: ArtifactId;
  occurredAt: string;
  actorId: string;
};

interface PdfCapabilities {
  inspect(input: Blob, region: Region): Promise<PdfArtifact>;
  redact(inputId: ArtifactId, policyRevision: string): Promise<JobId>;
  awaitArtifact(jobId: JobId): Promise<PdfArtifact>;
  sign(approvedArtifactId: ArtifactId): Promise<PdfArtifact>;
  requestDeletion(artifactId: ArtifactId): Promise<void>;
}

interface AuditSink {
  append(event: AuditEvent): Promise<void>;
}

async function prepareForExternalReview(
  pdf: PdfCapabilities,
  audit: AuditSink,
  input: Blob,
  region: Region,
  actorId: string,
  policyRevision: string,
): Promise<PdfArtifact> {
  const source = await pdf.inspect(input, region);
  const jobId = await pdf.redact(source.id, policyRevision);
  const reviewCopy = await pdf.awaitArtifact(jobId);

  if (reviewCopy.parentId !== source.id) {
    throw new Error("Artifact lineage mismatch");
  }

  await audit.append({
    action: "redacted",
    artifactId: reviewCopy.id,
    occurredAt: new Date().toISOString(),
    actorId,
  });

  return reviewCopy;
}
```

The browser can hold an upload or downloaded preview as a `Blob`, which MDN defines as an immutable, file-like object of raw data. That is useful at the UI edge. It is not the system of record. Don't put approval state, retention deadlines, or signing authority in an object URL or a browser tab.

The adapter should classify failures without leaking document contents into logs. A rejected input is different from a transient transport failure, and both are different from a completed job whose artifact identity does not match the expected parent. Retry only the operations defined as idempotent by your own boundary. For everything else, stop the transition and require a deliberate decision.

I would also keep observability sparse: job ID, artifact ID, region, transition, duration, and a normalized outcome. No extracted contract text. No player identifiers. Logs are another retention surface, and they are easy to forget because they sit outside the document database.

## What I would change at scale

At low volume, one worker and a relational state table are enough. At higher volume, split preview generation from execution-document processing so a burst of page thumbnails cannot delay a signing handoff. Add per-region queues only after residency requirements or measured contention justify them. Ship weekly; don't prepay the complexity tax.

I would add a fixed evaluation corpus before adding infrastructure: digitally generated contracts, scans, rotated pages, dense schedules, embedded fonts, tables crossing page boundaries, and documents that contain no target data. For each sample, compare the redacted artifact visually and structurally, then verify that forbidden strings cannot be recovered through the application path. A pretty black rectangle is not sufficient evidence of removal. The corpus should contain synthetic data rather than copied customer documents, and its expected results should be versioned alongside the redaction policy.

Then measure the workflow, not an isolated request. Capture time to first preview, time to review-ready artifact, queue wait, transformation duration, retry count, and deletion completion. Percentiles matter more than a single demo run, but the thresholds belong to the product: a lawyer waiting in an interactive screen has a different latency budget from a nightly back-office batch.

There is a point where the simple adapter stops being enough. If volume creates sustained queue contention, add explicit backpressure and dead-letter handling. If customers require independently controlled encryption keys or specialized evidence formats, the storage and signing boundaries may need separate ownership. If contracts routinely depend on handwriting, unusual annotations, or image-heavy scans, a human review checkpoint may remain mandatory even after automated tests improve. Outsource the undifferentiated processing, but keep policy, lineage, and release authority in the application.

## Trade-offs I would accept

A single integrated service reduces adapters, credentials, invoices, and operational surfaces. It is the sensible default for a small team when its regional controls, deletion semantics, rendering fidelity, and signing evidence meet the written acceptance tests. The limitation is concentration: one service boundary now sits on the whole review path, and migrating requires a tested export of artifacts and audit records.

Separate specialists can isolate signing from transformation and let each region use a different processing boundary. Stick with that approach when a customer obligation or evidence requirement cannot be represented by the integrated option. The cost is real: more credentials, more callbacks, more failure combinations, and more reconciliation code. This is not suitable when the expected benefit is merely hypothetical and the operator is still one person.

Self-operated processing offers direct control over deployment and deletion. Choose it when that control is a hard requirement and there is capacity to patch, monitor, and test the document stack. It is a poor fit when maintenance steals the revenue-producing hours that should go into product work.

My final gate is deliberately plain: can the system reproduce which source and policy produced the reviewed bytes, prove that those same bytes entered signing, and show when every temporary copy became eligible for deletion? If any answer depends on a dashboard screenshot or a filename, the endpoint choice is unfinished.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
