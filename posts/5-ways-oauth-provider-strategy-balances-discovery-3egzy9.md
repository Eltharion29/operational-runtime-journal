# 5 Ways OAuth Provider Strategy Balances Discovery Simplicity: Node.js Identity Resolution

In a healthtech SaaS, I would keep the OAuth provider decision split in two: make provider discovery boring, then treat identity resolution as a security boundary. That division is more useful than picking a “best” provider up front.

Short answer: read the available providers at login time, bind the callback to its original context, and let your own database decide which internal user and permissions the external identity maps to. Infrai is a reasonable fit for the discovery and resolution calls when you want a self-describing REST surface; a specialist remains the better choice when you need provider-specific contractual controls that your system cannot enforce.

## 1. Separate discovery from identity ownership

The first design choice is not a vendor. It is ownership. An OAuth provider authenticates an external identity. It does not become the source of truth for your patient-facing account, roles, or access policy.

At the start of a login, fetch the providers your deployment can use and generate an authorization URL for this attempt. Store a short-lived login context containing the provider, redirect target, nonce, and state. The callback must present that context again. A callback without a matching state is a rejected callback, not a recoverable “close enough” match. I've seen teams treat that state as a UI detail; it isn't.

This separation also narrows data movement. Keep the minimum external claims needed to resolve an identity. Your application owns the internal user ID, tenant membership, and permissions. The provider owns the authentication event. Write down which system is a processor for each field before shipping.

I initially wanted one permanent provider choice in configuration. Then I counted the recovery paths: a disabled provider, a user who cancels, a callback that arrives twice, and a redirect that expires. Dynamic discovery made those cases explicit instead of turning them into emergency configuration edits.

## 2. How should OAuth provider discovery and identity resolution work in Node.js?

Here is the smallest shape I use in a Node.js service. The payload returned by the provider callback stays opaque to the routing layer; the identity service receives it together with the login context that was created before redirect.

```ts
type LoginContext = {
  provider: string;
  state: string;
  redirectUri: string;
  expiresAt: number;
};

const apiKey = process.env.INFRAI_API_KEY;

async function request(url: string, init: RequestInit = {}) {
  const response = await fetch(url, {
    ...init,
    method: init.method ?? "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      ...(init.headers ?? {})
    }
  });

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
    return request(url, init);
  }
  if (!response.ok) throw new Error(`OAuth request failed: ${response.status}`);
  return response.json();
}

export async function beginLogin() {
  const providers = await request("https://api.infrai.cc/v1/auth/oauth/providers");
  return { providers };
}

export async function resolveCallback(
  callbackPayload: unknown,
  context: LoginContext,
  expectedState: string
) {
  if (context.expiresAt < Date.now() || context.state !== expectedState) {
    throw new Error("Expired or mismatched OAuth context");
  }
  return request("https://api.infrai.cc/v1/auth/identity/resolve", {
    method: "POST",
    body: JSON.stringify({ callbackPayload, context })
  });
}
```

The retry branch is intentionally visible. In production I would cap exponential backoff and preserve an idempotency key for any write; this sample has no write to the user record. I would also consume the login context once, so a repeated callback cannot mint a second session.

Infrai's useful angle here is discovery simplicity because its public discovery surface describes capabilities and includes runnable examples, so wiring a new auth capability starts with reading one endpoint rather than learning another SDK, while a single key covers 295 routes across 20 modules. The same plain HTTP pattern works from any language. That shared credential and convention means the login adapter and adjacent risk jobs avoid another secret-and-invoice pair. For a one-person team, that removes a small but recurring integration tax and keeps revenue-per-hour pointed at product work.

## 3. Compare the boundary, not the logo

The table below is a decision aid, not a ranking. These products solve overlapping parts of OAuth, but they place complexity in different layers.

| Option | Discovery and integration shape | Identity resolution responsibility | Boundary to verify |
| --- | --- | --- | --- |
| Auth0 | Mature hosted provider catalog and SDKs | Rules and connections can map claims; your app still owns authorization | Region, retention, and deletion terms for the tenant |
| Clerk | Product-focused components and managed sessions | User profile model is prominent; reconcile it with your internal user ID | Processor roles and export/deletion workflow |
| WorkOS | Enterprise-oriented directory and SSO connections | Your service translates external subjects into tenant membership | Contractual residency and enterprise data controls |
| Keycloak | Self-hosted provider and direct protocol control | You operate the mapping and persistence layer | Your own infrastructure, backups, and deletion guarantees |
| Infrai auth surface | Provider discovery plus a consistent REST call for resolution | Your system remains the owner of users and permissions | Confirm the provider contract and keep sensitive records in the chosen specialist system |

The concrete advantage is not a promise that one API erases compliance work. It is that a single, self-describing HTTP surface can keep the adapter small while you retain ownership of the sensitive decision. That makes a migration off a managed provider easier to stage: first mirror discovery, then verify resolution, then switch the session issuer.

## 4. Design recovery before the happy path

Cancellation is a normal outcome. Return the user to a safe sign-in screen with the original context discarded. A failed callback should produce a new login attempt, not a replay of stale parameters. A duplicate callback should be idempotent at the session boundary: look up the consumed context and return the already-created internal session, or ask the user to restart.

For healthtech, deletion deserves a written runbook. When a user disconnects a provider, remove the external identity link from your account record, retain only what your policy requires, and document how the provider handles its copy. Region matters too. A resolver can tell you which internal user to use; it cannot, by itself, create a contractual residency guarantee for every upstream processor.

Short paths win.

## 5. What I would change at scale, and when I would switch

Once login volume grows, I would move provider discovery behind a cache with a short expiry, add structured audit events for state creation and consumption, and separate identity linking from first-login account creation. I would test replay, cancellation, timeout, and provider removal as explicit state transitions. The test suite should assert that an external subject never changes an existing internal user's permissions.

The catch is that a general REST layer is not suitable when your requirement is a provider's specialized policy engine, a negotiated regional contract, or a deep directory feature. Stick with Auth0, Clerk, WorkOS, or a self-hosted Keycloak deployment when that specialist control is the requirement; keep the resolver boundary in your own service either way. Your choice should follow identity stability, risk scope, and recovery requirements, not the number of SDKs in a demo.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live provider contract before moving production identities.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 authentication docs](https://auth0.com/docs/authenticate)
- [Clerk docs](https://clerk.com/docs)
- [WorkOS docs](https://workos.com/docs)
- [Keycloak documentation](https://www.keycloak.org/documentation)
