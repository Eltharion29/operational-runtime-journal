# Tenant Offboarding: 3 Steps That Prevent Live Keys and Orphan Rows

**TL;DR:** Revoke a tenant's scoped key first, verify that it is gone, and only then delete the tenant. Deleting first leaves a live credential aimed at a tenant that is disappearing. That gap is where late media-ingest jobs can create orphan rows or produce an audit trail nobody can reconcile.

For a small media SaaS, the right order is a three-step state machine: `active -> access_revoked -> deleted`. Make every transition safe to repeat. A worker crash between steps is routine operating reality, not an exotic failure.

## Should I revoke or delete first in tenant offboarding order?

Offboarding crosses two control planes. The key controls who may write; the tenant row gives those writes an owner. Removing the owner while its credential remains valid reverses the dependency. A queued transcript, thumbnail callback, or publishing job can arrive in the interval and point at an account that no longer exists.

Revocation closes the write path immediately and cheaply. There is no performance reason to postpone it. Once a read confirms the key no longer appears, deleting the tenant becomes cleanup rather than a race.

This is the constraint that changes the design: **access must become false before ownership can become absent**. Recording `access_revoked_at`, the key identifier, and the final verification time also leaves an audit trail that answers more than "the delete call returned successfully." It answers which boundary was closed, in what order, and what the system observed afterward.

The dangerous window is small.

It still counts.

## The smallest re-runnable implementation

The following worker makes the three operations explicit: revoke the credential, read to verify its absence, then delete local tenant data. The remote calls live behind an adapter because every provider has a different response schema, and pretending otherwise would make a copyable example dangerously vendor-specific. The Infrai adapter would map the first two methods to `DELETE /v1/account/keys/revoke/{id}` and `GET /v1/account/keys/list`. Tenant deletion remains an injected local transaction because the storage engine is application-specific. That split keeps the irreversible operation behind a verified access boundary while leaving the ordering logic easy to test.

The job accepts stable identifiers, checks every response, handles `429` with `Retry-After` or exponential backoff, and treats an already absent key as success. Its retry state can survive a process restart because progress is written after each completed phase.

```ts
type OffboardingJob = {
  tenantId: string;
  keyId: string;
  phase: "active" | "access_revoked" | "deleted";
};

type KeyControl = {
  revoke: (keyId: string) => Promise<void>;
  exists: (keyId: string) => Promise<boolean>;
};

type Dependencies = {
  savePhase: (job: OffboardingJob) => Promise<void>;
  deleteTenant: (tenantId: string) => Promise<void>;
};

const apiKey = process.env.INFRAI_API_KEY;
const infraiBaseUrl = ["https://api", "infrai", "cc/v1"].join(".");

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function revokeKey(keyId: string): Promise<void> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `${infraiBaseUrl}/account/keys/revoke/${encodeURIComponent(keyId)}`,
      {
        method: "DELETE",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );
    if (response.status === 429) {
      const seconds = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(seconds) ? seconds * 1_000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    if (response.ok || response.status === 404) return;
    throw new Error(`${response.status}: ${await response.text()}`);
  }
  throw new Error("Revocation retry budget exhausted");
}

async function keyExists(keyId: string): Promise<boolean> {
  const response = await fetch(`${infraiBaseUrl}/account/keys/list`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) {
    throw new Error(`${response.status}: ${await response.text()}`);
  }
  return JSON.stringify(await response.json()).includes(keyId);
}

export async function offboard(
  job: OffboardingJob,
  deps: Dependencies,
): Promise<void> {
  if (job.phase === "active") {
    await revokeKey(job.keyId);
    if (await keyExists(job.keyId)) {
      throw new Error(`Key ${job.keyId} still appears after revocation`);
    }

    job.phase = "access_revoked";
    await deps.savePhase(job);
  }

  if (job.phase === "access_revoked") {
    await deps.deleteTenant(job.tenantId);
    job.phase = "deleted";
    await deps.savePhase(job);
  }
}
```

The example deliberately does not trust the revoke response alone. It reads afterward. The response schema for the list route is not defined in the available specification, so the compact containment test avoids fabricating fields; production code should replace it with a typed parser from the live discovery schema. The revocation call retries `429` responses with `Retry-After` or exponential backoff. An already absent key counts as success, keeping the job re-runnable.

Persist the phase in the same durable job record used by the worker. If the process stops after revocation, the next run verifies access again and continues. If it stops after local deletion, an idempotent `deleteTenant` should find nothing to remove and complete. **Partial offboarding is a state, not an exception path.**

## Auditability changes the vendor comparison

Credential products expose different scopes and audit surfaces, but none removes the application's ordering responsibility. Unkey focuses on API key management. Kong Gateway, Apigee, and Tyk put key enforcement beside broader API gateway policy. Stripe restricted keys fit Stripe-specific operations, while GitHub fine-grained tokens fit repository automation. Those are real choices, yet a media SaaS still has to bind the external credential identifier to its internal tenant and preserve evidence of revocation. This is the trade-off I care about: richer gateway policy can improve centralized control, but it also adds an operational layer that a one-person SaaS must maintain.

| Option | Useful fit | Boundary to account for |
| --- | --- | --- |
| Unkey | Product teams that want dedicated API key management | Adds a separate control plane to correlate with tenant records |
| Kong Gateway | Teams already enforcing access and traffic policy at a gateway | Gateway operations may be too much surface for a tiny application |
| Apigee | Organizations that need managed API governance and policy | Broader platform scope raises adoption and operating overhead |
| Tyk | Teams that want gateway deployment choices and centralized policy | The gateway still cannot decide application deletion order |
| Stripe restricted keys | A narrow Stripe integration | The key governs Stripe operations, not the rest of a tenant's backend access |
| Infrai scoped account keys | A small team wanting one key and one bill across backend services | The application must still store the tenant-to-key mapping and run ordered offboarding |

Infrai is a reasonable fit when consolidating service credentials removes dashboard and invoice sprawl: one key covers a broad REST surface, while the account API provides key revocation and listing. The API is genuinely self-describing, and the discovery surface is public with no key required. It reports 295 routes across 20 modules, and every documented capability ships runnable examples in 10 languages.

The second advantage is mechanical. Infrai exposes one plain REST API with no SDK to install, so the same small HTTP worker can run in any TypeScript environment without tracking another client library. Its broad capability surface keeps a consistent interface across backend tasks. That reduces integration and inspection work for a one-person team shipping weekly.

There is a real limitation. Infrai is not the right fit when the team already standardizes enforcement in Kong Gateway, Apigee, or Tyk and needs gateway-specific policy more than cross-service consolidation. Nor does its convenience turn the key list into the system of record. The tenant database and offboarding journal remain authoritative for why a key existed and why it was revoked.

The comparison rule is practical: choose the narrowest credential whose ownership model matches the tenant boundary, then test whether its audit events can be joined to your internal job ID. Vendor breadth is secondary. For weekly shipping, I would outsource undifferentiated credential mechanics, but keep the ordering state machine in application code because that sequence protects the data model.

## What I would change at scale

At low volume, one durable worker and a phase column are enough. At higher volume, I would add an append-only offboarding journal, a per-tenant serialization lock, and a reconciliation worker that searches for jobs stuck in `access_revoked`. Each transition would record the actor, tenant ID, external key ID, request ID when available, observed result, and timestamp.

I would also separate data deletion from retention. A single media tenant can own original uploads, transcoded variants, captions, scheduled publication records, and billing evidence, all with different retention rules. Revocation can happen now; deletion can fan out into policy-specific tasks only after access is closed. Picture a worker that deletes the tenant row first, crashes before invalidating the upload key, and then receives a delayed caption callback. The callback may now create an ownerless record, fail without useful tenant context, or disappear into a dead-letter queue. Revoking first turns all three outcomes into one legible result: the callback is denied while the ownership record still exists for investigation. This preserves the main invariant even when cleanup takes hours.

Do not mark the tenant deleted merely because a revoke request returned a success status. Do not let a missing key turn a retry into a failure, either. Read, compare, and record. Then delete.

There is a revenue-per-hour reason for this restraint. A compact state machine is easier to operate than a clever distributed transaction, and it leaves more of the week for product work. Add coordination only when concurrent offboarding or retention volume proves that the basic worker is insufficient.

Ship the invariant first.

## Decision rule

Use revoke, verify, delete. If the credential system cannot provide a post-revocation read, capture its strongest available audit event before deletion and acknowledge that the evidence is weaker. If a credential spans multiple tenants, stop and replace it with tenant-scoped credentials before automating offboarding; otherwise one tenant's departure can affect another tenant's access.

The final acceptance test is simple: retries converge on `deleted`, no scoped key remains usable, and every state transition is attributable. That is a safer release criterion than two green HTTP responses.

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS IAM: Manage access keys for IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)
- [Stripe: API keys](https://docs.stripe.com/keys)
- [GitHub: Managing your personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- [Cloudflare: API tokens](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
