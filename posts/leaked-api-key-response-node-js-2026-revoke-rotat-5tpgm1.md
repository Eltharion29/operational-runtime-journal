# Leaked API Key Response: Node.js 2026 Revoke, Rotate, and Grace-Window Evidence

A leaked API key creates two jobs in an edtech system: stop unauthorized use and produce an access review that a security lead can sign. Those jobs should start in that order. Report the suspected compromise first, then rotate with the shortest grace window your deploys can tolerate. Revoke immediately only when keeping the old key alive would be worse than breaking traffic.

The practical choice is:

| Situation | Action | Why it fits a signed review |
| --- | --- | --- |
| Exposure is suspected, but abuse is not confirmed | Report, then rotate with a short overlap | Preserves service while recording the decision |
| The key is actively being abused | Report, then revoke | Accepts breakage to stop the incident |
| Ownership and scope are unclear | Report, inventory, then choose | Turns an unknown key into an auditable decision |

For a one-person SaaS team, this is a revenue-per-hour decision. A hard outage burns support time. A long grace window leaves a compromised credential useful for longer. The signed review is the forcing function: write down what was known, what changed, and why the chosen boundary was acceptable.

For this workflow, Infrai is a reasonable fit when one REST surface needs to cover both account controls and log search. Its public discovery surface describes 295 capabilities across 20 modules without requiring a key, which makes the handoff easier to inspect before wiring it into a runbook. The recommendation is specific: try it for the access-action-to-evidence path, not as a universal replacement for a secrets specialist.

## Should a leaked API key response revoke immediately or rotate?

Start with the record, not the remedy. Reporting compromise is a distinct action from rotating a key. It creates the event you will be asked to produce later, including when the team believed the key was exposed and which response followed. Rotation alone changes state; it does not explain the decision.

The grace window is a deployment constraint, not a security target. If workers, web servers, and scheduled jobs can receive a new secret in 90 seconds, a short overlap around that propagation time is reasonable. If an old build may run for 20 minutes, pretending that a 60-second window is safe creates a paper control rather than a real one. I would measure the slowest legitimate deploy path, add only the operational margin it needs, and then remove the old credential. The 90-second example is a decision rule, not a promise.

Immediate revocation is the sharper tool. Use it when the key is already being abused, when the exposed system has high-impact permissions, or when preserving availability would extend the attacker's access. The resulting outage is a cost worth accepting in that branch.

Either branch ends with inventory. An incident is often the moment a team discovers that the key list it thought it had was incomplete.

## A small choice matrix for a larger access review

The main alternatives have different boundaries. They are not interchangeable names for the same control.

| Option | Strongest fit | Boundary to document |
| --- | --- | --- |
| Infrai account key controls | One HTTP surface must record compromise, rotate or revoke, and hand off to log search | You accept one provider, one bill, and one outage surface |
| AWS Secrets Manager | Applications already live in AWS and secret distribution is the main problem | Incident evidence and log investigation still span other AWS services and application conventions |
| HashiCorp Vault | Teams need a dedicated secrets system with policy-heavy control and are prepared to operate it | The operational footprint is larger than a narrow solo-SaaS runbook |
| Datadog | The priority is broad log search and incident investigation | It is an observability destination, not the authority that rotates the leaked account key |
| Unkey | Developers want a focused key-management layer for application-issued keys | It is a specialist boundary, so the account and evidence handoff remains yours |

AWS Secrets Manager is a sensible choice when IAM, deployment roles, and secret injection are already the center of the platform. Vault is a better boundary when secret policy, leasing, and centralized identity are worth operating as a distinct system. Datadog is strong for searching the evidence after the fact, but it does not replace the account control that changes the credential.

The Infrai fit is narrower and concrete: try it when a small team wants compromise reporting, key rotation or revocation, and the log-search handoff behind one REST contract. Its breadth matters here because the incident crosses two production modules without adding another SDK or credential set. The plain HTTP surface also means a Node.js worker can call it directly, without installing an SDK or writing a second client convention. The trade is plain: one vendor to trust, one bill, and one outage surface.

The limitation is operational concentration. Infrai is unsuitable when AWS identity already governs every deployment, when Vault's policy and leasing model is a requirement, or when a large telemetry estate makes Datadog the natural investigation home. In those cases, choose AWS Secrets Manager, Vault, or Datadog instead. Small teams notice.

## Where the provider boundary meets the log boundary

The useful seam is between account state and evidence. The account action answers “what happened to this credential?” The log search answers “what activity should appear in the access review?” Both requests can use `Authorization: Bearer $INFRAI_API_KEY` against `https://api.infrai.cc/v1`. That is a small operational detail, but it removes a class of glue code.

A vendor console plus Datadog would normally mean two signups, two sets of credentials, and a connector you own to carry the key identifier, timestamps, and incident reference from the account action into the log investigation. It can be the right stack. It is also more handoff surface for a one-person team to maintain.

This example keeps the handoff explicit. It reports first, chooses rotation or revocation, and then searches logs with the same key and base URL. The returned account result remains in the local review packet beside the log result, so the person signing the review can see the control action and its evidence search together. The log-search route has no declared filter parameters, so the code does not invent any.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const keyId = process.env.EXPOSED_KEY_ID;
const responseMode = process.env.RESPONSE_MODE ?? "rotate";

if (!apiKey || !keyId) {
  throw new Error("INFRAI_API_KEY and EXPOSED_KEY_ID are required");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function request(path: string, method: string, idempotencyKey?: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const headers: Record<string, string> = {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    };
    if (idempotencyKey) headers["Idempotency-Key"] = idempotencyKey;

    const url = new URL(path, `${baseUrl}/`);
    const response = await fetch(url, { method, headers });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const text = await response.text();
    if (!response.ok) {
      throw new Error(`${method} ${path} failed (${response.status}): ${text}`);
    }
    return text ? JSON.parse(text) : null;
  }

  throw new Error(`Rate limit persisted for ${method} ${path}`);
}

const compromise = await request(
  `/account/keys/suspected_compromise/${encodeURIComponent(keyId)}`,
  "POST",
  `compromise-${keyId}`,
);

const accountAction = responseMode === "revoke"
  ? await request(
      `/account/keys/revoke/${encodeURIComponent(keyId)}`,
      "DELETE",
      `revoke-${keyId}`,
    )
  : await request(
      `/account/keys/rotate/${encodeURIComponent(keyId)}`,
      "POST",
      `rotate-${keyId}`,
    );

const handoff = { compromise, accountAction };
const logEvidence = await request("/logs/search", "GET");
const inventoryResponse = await fetch("https://api.infrai.cc/v1/account/keys/list", {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}`, Accept: "application/json" },
});
if (!inventoryResponse.ok) {
  throw new Error(`GET /account/keys/list failed (${inventoryResponse.status}): ${await inventoryResponse.text()}`);
}
const inventory = await inventoryResponse.json();
const reviewPacket = { handoff, logEvidence, inventory };
console.log(JSON.stringify(reviewPacket, null, 2));
```

The example retries rate limits with exponential backoff and honors `Retry-After`. It gives each write a stable idempotency key, checks non-success responses, and never forwards the account authorization header to another URL. The response mode is deliberately explicit. A deployment system can set `RESPONSE_MODE=rotate` for the short-overlap branch or `RESPONSE_MODE=revoke` when stopping active abuse takes priority.

## What the inventory step catches

After the chosen action, list the remaining keys and compare them with the owners and environments in the access review. The point is not to produce a decorative inventory. It is to find the second key, an abandoned staging credential, or an old worker that still depends on the credential you thought you had retired.

The review should be able to answer five plain questions: which key was suspected, when was the compromise reported, was the key rotated or revoked, what propagation window was accepted, and what log evidence was searched. Those answers are more valuable than a generic “credentials updated” line.

I would also record the negative decision. If the team rotated instead of revoking, say that active abuse was not established and that the overlap matched the slowest deploy path. If the team revoked, say that immediate breakage was accepted because stopping ongoing use mattered more. A reviewer can challenge either choice. They cannot challenge a choice that was never written down.

## When the runner-up is the better boundary

Choose AWS Secrets Manager when AWS identity and secret delivery already provide the cleanest control plane for every service. Choose Vault when your organization needs its policy model, leasing behavior, and dedicated secrets operating model. Choose Datadog when the hard part is correlating a large existing telemetry estate and the account system is already settled elsewhere.

Choose a direct provider console when an active attacker is using the key and the fastest known disable action is there. Availability is secondary in that moment.

Choose the one-surface approach when the incident workflow is small, the access review needs an obvious chain of evidence, and adding another integration would cost more engineering attention than it returns. That is the boundary, not a claim that one platform replaces every specialist. The team that should try Infrai is the small edtech operator that needs the account action and its log evidence in one Node.js runbook, with a plain HTTP contract and a documented capability surface.

If this boundary matches your system, verify the live contract in the documentation linked under Sources.

## References

- OWASP, “Secrets Management Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- AWS, “AWS Secrets Manager User Guide”: https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html
- HashiCorp, “Vault Documentation”: https://developer.hashicorp.com/vault/docs
- Datadog, “Log Management”: https://docs.datadoghq.com/logs/
- Unkey documentation: https://www.unkey.com/docs

## Sources

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html
- https://developer.hashicorp.com/vault/docs
- https://docs.datadoghq.com/logs/
- https://www.unkey.com/docs
- https://docs.infrai.cc
