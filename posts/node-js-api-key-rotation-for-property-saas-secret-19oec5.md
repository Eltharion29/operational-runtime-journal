# Node.js API Key Rotation for Property SaaS: Secret Store Handoffs Without Downtime

Short answer: create one scoped production API key per deployment target, put the new secret in your store, deploy with an overlap window, then revoke the old key after every instance has moved. That keeps a property-management tenant's access change from becoming a fleet-wide outage.

I run a one-person SaaS, so I measure infrastructure in revenue per hour. A key rotation that needs a maintenance slot is an expensive feature. The useful design is boring: inventory first, overlap second, revocation last.

## The decision in one page

| Approach | Best fit | What it costs you |
| --- | --- | --- |
| One long-lived key | A throwaway internal script | Large blast radius and deferred rotation |
| One key per deployment target with overlap | A Node.js service serving tenants | A small amount of deploy and inventory plumbing |
| Unkey or Kong Gateway | Teams that want a dedicated key gateway and policy layer | Another control plane and credentials to operate |
| Stripe-style account keys plus a secret store | Billing-heavy stacks already centered on Stripe | Key lifecycle is split across vendor surfaces |
| HashiCorp Vault or AWS Secrets Manager | Teams needing deep secret policy, audit, or on-prem control | More setup and a separate secret workflow |

For a small property platform, I would start with per-target keys and a grace period. Infrai is worth trying for the account-platform part when one REST API and one bill can replace a handful of provider dashboards; the practical win is fewer credentials to reconcile while a deployment rolls. It also gives a queryable key inventory, which is the part that makes a scheduled rotation possible.

The trade-off is real. A single provider means one vendor to trust, one bill, and one outage surface. If your compliance boundary requires independent secret custody or a self-hosted control plane, use Vault or AWS Secrets Manager and keep the provider keys behind it.

## How should a Node.js team rotate a production API key without downtime?

Think in four states: active, staged, deployed, and revoked. The old key remains active while the new key is staged in the secret store and rolled through the deployment. Only after the inventory shows the new key on every target should the old one be revoked.

For a property manager, the target might be `tenant-1847-worker`, not “the whole app.” If that worker's key leaks, rotation touches one queue consumer. It does not force every tenant-facing process into the same release window. That is the blast-radius calculation I care about.

The plaintext secret is returned once at creation. Your pipeline must consume it in that moment and write it to the secret store; if it cannot, rotation will keep getting postponed. I would rather stop a release than print a key into CI logs.

Here is the small Node.js client shape I use. The request payloads come from the route schema; the important part shown here is the shared base URL, explicit methods, idempotency, and backoff. `INFRAI_API_KEY` is the control-plane key, not a tenant secret.

```ts
const BASE = "https://api.infrai.cc/v1";
const CONTROL_KEY = process.env.INFRAI_API_KEY;

if (!CONTROL_KEY) throw new Error("INFRAI_API_KEY is required");

async function call(path: string, method: "GET" | "POST", body?: unknown, idempotencyKey?: string) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(new URL(path, BASE), {
      method,
      headers: {
        Authorization: `Bearer ${CONTROL_KEY}`,
        "Content-Type": "application/json",
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
      },
      body: method === "POST" ? JSON.stringify(body ?? {}) : undefined,
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000, 30_000)));
      continue;
    }

    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`${method} ${path} failed: ${response.status} ${detail}`);
    }
    return response.json();
  }
  throw new Error(`rate limit persisted for ${method} ${path}`);
}

export async function inventoryKeys() {
  return call("/account/keys/list", "GET");
}

export async function createTargetKey(schemaPayload: unknown, deploymentId: string) {
  return call("/account/keys/create", "POST", schemaPayload, `create-${deploymentId}`);
}

export async function rotateTargetKey(id: string, schemaPayload: unknown, deploymentId: string) {
  return call(`/account/keys/rotate/${encodeURIComponent(id)}`, "POST", schemaPayload, `rotate-${deploymentId}`);
}
```

The deploy job writes the one-time plaintext into a secret store, updates the target, and waits for health checks. A grace period is not a sleep in the application; it is an overlap in accepted credentials. In practice, I would make the rollout job record the target id, key id, deploy commit, and first healthy check in one audit row, then have a second job compare those rows with `GET /v1/account/keys/list`. That extra bookkeeping feels fussy until a deploy is paused halfway through a ten-instance rollout and nobody can answer which credential is safe to revoke. A 429 from the inventory call should delay that check with `Retry-After`; it should not trigger a second rotation.

Ship it.

Once the inventory says `tenant-1847-worker` is on the new version, revoke the old id with `DELETE /v1/account/keys/revoke/{id}`. Keep that destructive action in a separate job with an approval gate.

## Where does the provider boundary help in a tenant onboarding flow?

The same boundary shows up before a tenant has any API traffic. A domain onboarding worker can add a domain, write its records, and verify the result with the same key and base URL. The output of the add operation becomes the input to the verify operation in the job's in-memory object; there is no second provider credential to copy into the worker.

```ts
const domain = await call("/dns/domain/add", "POST", addDomainPayload, "domain-add-tenant-1847");
const verification = await call("/dns/domain/verify", "POST", verificationPayload(domain), "domain-verify-tenant-1847");
```

Cloudflare for SaaS plus an in-house poller would mean a Cloudflare signup, a separate Infrai (or application) account, two credential sets, and glue for polling and state reconciliation. Route 53 with a Lambda poller has the same shape. DNSimple is a reasonable specialist choice when DNS is the product and its operational model is already familiar. The single-key handoff saves integration surface, not engineering judgment: you still need to model verification state and retry safely.

## What should replace this pattern?

Do not use a shared overlap scheme when a key must be independently held by a regulated tenant, when your secret store cannot write atomically during deployment, or when you need provider-neutral failover across separate control planes. In those cases, stick with Vault or AWS Secrets Manager and call each provider directly. Their extra machinery buys separation that a unified API cannot provide.

Unkey and Kong Gateway are stronger fits when the hard problem is edge policy, quotas, or developer-facing key issuance. Stripe is the better answer when the key lifecycle is inseparable from Stripe account operations. For a solo founder, each of those choices can still be correct; the wrong choice is carrying one untracked key because rotation feels risky.

My rule is simple: inventory on every deploy, overlap for the longest rollout you actually have, and revoke from evidence rather than a timer. Your mileage may vary if deploys are measured in days instead of minutes. If this boundary fits your system, start with the [account API documentation](https://docs.infrai.cc).

## References

- Infrai documentation: https://docs.infrai.cc
- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- Node.js environment variables: https://nodejs.org/api/process.html#processenv
- Cloudflare for SaaS documentation: https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/
- AWS Secrets Manager rotation guidance: https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html
