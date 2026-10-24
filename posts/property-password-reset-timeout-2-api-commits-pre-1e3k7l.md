# Property Password Reset Timeout: 2 API Commits Prevent Duplicate Email

| Choice | API response waits for | Duplicate-send risk | Integration effort |
| --- | --- | --- | --- |
| Send inside the request | Provider acceptance | Higher after an ambiguous timeout | Low at first, costly under failure |
| Durable handoff | Local job commit | Controlled by one idempotency record | Moderate, then predictable |
| Scheduled sweep | Periodic database scan | Controlled, but delivery starts later | Moderate and operationally simple |

TL;DR: For a property-management password reset with a short expiry, return after a durable local handoff, not after an email provider call. Use two commit points: one transaction creates the reset challenge and one notification job; a worker then claims that job and records the provider result. Repeating the API request must reuse the active challenge, while repeating worker execution must reuse the same notification key. This costs more integration work than calling email inline, but it stops a network timeout from becoming a tenant's second or third reset message.

**Recommendation:** choose the durable handoff. A scheduled sweep is the runner-up when adding a queue would consume more founder-hours than its latency saves. The revenue-per-hour test is blunt: keep authentication rules in the application, outsource transport, and spend custom engineering on the tiny state machine between them.

## How should a password reset API handle an email request timeout?

A timeout describes what the caller observed. It does not prove what the downstream system did. The function may lose its connection before submission, during submission, or after the transport accepted the message but before the response reached the function. Retrying the whole handler treats those different states as one state.

That ambiguity is the real bug boundary. A tenant who manages several units may tap “reset” again, a browser may repeat a request, or the platform may run an event more than once. If each attempt creates a fresh token and sends immediately, inbox order can invert: the newest email can arrive first, then be invalidated by an older request that finished late. A short expiry makes the confusion sharper, not safer.

One click should mean one intent.

Keep the public response boring. Return the same generic acknowledgement for an existing and a missing account, and avoid exposing whether an email address belongs to a property manager. The reset token should be random, single-use, stored in a protected form, and invalidated after use. Those are application invariants. Delivery status must not weaken them.

## The two criteria that decide the design

The first criterion is **where uncertainty is committed**. An HTTP deadline should cover authentication, validation, and a local database transaction. It should not stretch across DNS, a remote connection, provider throttling, or a delayed response. Once the transaction commits, the API can truthfully say the request was accepted for processing. It cannot truthfully promise inbox delivery.

The second is **how much machinery one person can operate**. A queue with leases, dead-letter handling, dashboards, and another deployment can be correct and still be the wrong weekly shipping trade. A database-backed outbox often wins because the account row, reset challenge, and notification intent can commit together. One datastore is easier to inspect at 2 a.m. and easier to delete when the product outgrows it.

The limitation is real: a database outbox adds schema, a polling or dispatch process, cleanup, and operational ownership. It is not suitable when the application cannot atomically write business state and an outbox row, and it is a poor fit when extremely low dispatch latency matters more than minimizing moving parts. In those cases, a transactional broker or a queue-native workflow is the better fit. The trade-off goes the other way for a small property-management SaaS with modest reset volume: a separate broker adds credentials, permissions, deployment configuration, monitoring, and a failure surface before it adds user value. That is why integration effort is the primary decision axis here. The choice should change when measured queue delay, database load, or burst concurrency crosses a stated service objective, not when the team encounters a fashionable architecture diagram.

The minimum useful record is small: a stable operation key, account identifier, challenge identifier, template type, status, attempt count, next-attempt time, lease expiry, and provider receipt when available. Put a unique constraint on the semantic operation, such as `challengeId + template`. Do not deduplicate on email address alone. A later legitimate reset for the same manager is a different operation.

Expiry deserves its own clock. The challenge validity starts when the application creates it, while delivery can lag behind. Before each attempt, the worker should check whether enough usable lifetime remains. If not, mark the job expired and create nothing automatically. Sending a nearly dead link is technically successful transport and failed product behavior.

Dead links waste trust.

## A small state machine in TypeScript

This example keeps transport behind a generic interface and puts the irreversible decisions in storage. The repository methods are expected to use atomic compare-and-set updates. Exact SQL varies, but the ownership rule does not.

```ts
type ResetNotice = {
  jobId: string;
  challengeId: string;
  recipient: string;
  expiresAt: Date;
  attempt: number;
};

type ClaimResult =
  | { kind: "claimed"; notice: ResetNotice }
  | { kind: "busy" | "sent" | "expired" };

interface ResetStore {
  createOrReuseChallenge(input: {
    accountId: string;
    requestKey: string;
    expiresAt: Date;
  }): Promise<{ challengeId: string; jobId: string }>;
  claim(jobId: string, leaseUntil: Date): Promise<ClaimResult>;
  markSent(jobId: string, receipt: string): Promise<void>;
  retryLater(jobId: string, attempt: number, nextAttemptAt: Date): Promise<void>;
  markExpired(jobId: string): Promise<void>;
}

interface MailTransport {
  send(input: {
    operationKey: string;
    to: string;
    template: "property-manager-reset";
    challengeId: string;
  }): Promise<{ receipt: string }>;
}

export async function deliverReset(
  jobId: string,
  store: ResetStore,
  mail: MailTransport,
  now: Date,
): Promise<void> {
  const leaseUntil = new Date(now.getTime() + 30_000);
  const claim = await store.claim(jobId, leaseUntil);
  if (claim.kind !== "claimed") return;

  const { notice } = claim;
  const minimumUsefulLifeMs = 120_000;
  if (notice.expiresAt.getTime() - now.getTime() < minimumUsefulLifeMs) {
    await store.markExpired(jobId);
    return;
  }

  try {
    const result = await mail.send({
      operationKey: notice.jobId,
      to: notice.recipient,
      template: "property-manager-reset",
      challengeId: notice.challengeId,
    });
    await store.markSent(jobId, result.receipt);
  } catch {
    const nextAttempt = notice.attempt + 1;
    const cappedDelayMs = Math.min(60_000, 2 ** nextAttempt * 1_000);
    await store.retryLater(
      jobId,
      nextAttempt,
      new Date(now.getTime() + cappedDelayMs),
    );
  }
}
```

The 30-second lease and two-minute minimum useful life are example policy values, not universal constants. Set them from the function deadline, observed delivery latency, and the actual reset window. The important part is their relationship: the lease must recover from a dead worker, and the worker must refuse work that no longer leaves a useful interval for the tenant.

There is still an unavoidable gap between remote acceptance and `markSent`. If the worker dies there, it may submit again after the lease expires. A transport that honors an idempotency key can close that gap. If it does not, stable message metadata plus reconciliation can reduce confusion, but an exactly-once claim would be false. Design for at-least-once execution and make the effect as close to once as the transport permits.

No wrapper can erase that gap.

Ship the first version with three observable timestamps: job creation, first claim, and sent or terminal failure. Add a counter for claims, a counter for ambiguous transport failures, and an alert on jobs that remain pending beyond their useful lifetime. Never log reset tokens or reset URLs. The useful debugging unit is the operation key.

## When the scheduled sweep is the better runner-up

A scheduled database sweep is sensible at low volume when notification latency of up to one polling interval is acceptable. It avoids a separate broker and keeps recovery legible: select due rows, claim a bounded batch, then run the same worker. For a solo SaaS, fewer control planes can beat lower theoretical latency.

It loses when resets arrive in bursts, the polling interval consumes too much of a short expiry, or concurrent sweepers create database contention. Move to a managed queue when those measured conditions appear. Do not adopt one because a diagram looks mature.

Inline sending remains reasonable only when a duplicate message has little consequence and the caller can tolerate ambiguity. Password recovery fails that test. Email links can invalidate one another, tenants interpret repeats as suspicious, and support work lands on the same person who should be shipping the next feature.

Test the state machine, not just the happy-path function. Run cases where the transport accepts and then throws, the worker dies before `markSent`, two workers claim together, the same browser request arrives twice, and the challenge expires while a job waits. The release gate is simple: every run leaves one active challenge and at most one notification intent for that challenge.

## A practical shipping rule

Own two commits and one key. The API owns creation of the challenge plus notification intent. The worker owns the transition from pending to a recorded outcome. The job ID crosses the transport boundary unchanged.

Everything else is replaceable plumbing. That boundary keeps a short-expiry reset understandable during a timeout, cheap enough for a one-person operation, and ready to move from a sweep to a queue without rewriting authentication. Ship it, observe the pending age, and change infrastructure only when the numbers demand it.

## Sources

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/glossary/what-sms-character-limit
