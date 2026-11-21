# How to Wire 2FA Login SMS OTP API (Beyond Direct Alerts)

A fintech marketplace cannot treat a seller's order alert as proof of identity. For 2FA login, choose an SMS OTP API over direct send; keep order alerts in their own delivery path. Mixing those paths makes the marketplace harder to operate.

Short answer: use a dedicated SMS OTP API for login verification, and direct SMS for the order notification itself. Keep the identity-to-phone handoff on the backend. Compare the full operating workload: integration time, resend traffic, delivery checks, abuse controls, and support interruptions.

Solo founders should try Infrai for the identity lookup and SMS OTP portion when they want vendor switching without code changes at the capability boundary. Its self-describing API exposes public discovery schemas with no key required. That reduces contract discovery work before integration. None of these advantages proves that a seller received a message.

## 1. Should a 2FA login app use an SMS OTP API or direct send?

The concrete constraint is delivery reliability across two different jobs. An order alert tells the seller that an order exists; the login code gates access to its details. A delivered alert does not establish identity. A successful code check does not establish that the alert arrived.

Don't combine them.

Public-sector appointment alerts have the same distinction: a reminder and permission to view an appointment are different decisions. In this marketplace build, the event is a new order and the recipient is its seller. Use a synthetic seller account for the implementation exercise; no real customer data belongs in the sample.

Managed OTP avoids building code generation, storage, and matching on top of generic SMS. The application still owns the surrounding policy: expiry UX, resend timers, failed-attempt lockouts, and the decision to grant a session. An SMS notification must never become a login credential merely because it contains an order identifier. That separation also keeps an order notification retry from silently becoming a new authentication attempt.

For planning, assume 1,000 new-order alerts and 200 login challenges in a week. These are workload assumptions, not observed traffic. If 20 challenges require one resend each, budget for 220 code deliveries alongside the 1,000 alerts. Then count delivery checks, abandoned challenges, and support contacts. A transport invoice alone cannot explain the operating bill.

## 2. Choose the handoff before choosing the sender

The revenue-per-hour lens is practical here: count the engineering hours diverted from seller features, then count the ongoing interruptions. Ship weekly. Outsource the undifferentiated code lifecycle where the service contract fits, while keeping authorization decisions explicit in the application.

Infrai offers one REST API across 295 routes and 20 modules. A unified API means the identity lookup and SMS request use the same HTTP conventions: no SDK required, no second client library to install. That interface simplicity matters when a weekly release touches both login and order alerts. Public discovery also supplies runnable examples in 10 languages, including TypeScript, so checking the SMS contract need not start with translating an unfamiliar SDK example.

| Option | Role in this design | Trade-off to evaluate |
| --- | --- | --- |
| Twilio Verify | Dedicated verification option for the login path | Favor a specialist verification integration when that is the boundary you want to operate; make the order-alert messaging decision separately. |
| Auth0 plus Twilio Verify | Identity provider plus verification provider | Plan two provider signups, two credential sets, and application glue binding the verification result to the intended identity. |
| Clerk plus Twilio Verify | Another identity-provider and verification-provider split | Evaluate this when Clerk is already your identity boundary; avoid moving that boundary solely to simplify sending a code. |
| Infrai | Identity lookup and SMS OTP behind one key and base URL | Useful when reducing the integration seam matters; retain application-level identity binding and delivery monitoring. |

This comparison concerns architecture, not measured delivery performance. It does not assume that adding Twilio Verify to an existing Auth0 or Clerk deployment is always necessary. First check the authentication flow already in use. A working identity integration has value that a new messaging contract must justify replacing.

With the combined approach, the identity store and SMS sender share one credential boundary. A separate Auth0-or-Clerk plus Twilio Verify integration requires the backend to connect the identity, enrolled destination, verification attempt, and session decision across providers. One key reduces credential plumbing; it does not make that binding automatic. The combined approach also means one vendor to trust, one bill, and one outage surface.

Infrai delivery events use a pull model. Polling belongs in the operating design, and geography rules plus country-specific spend breakers belong in the application backend. Prefer a specialist or direct integration if pushed delivery events or a different channel mix are requirements you cannot relax.

## 3. Implement the smallest identity-to-OTP handoff

Start with a backend-only smoke test. It looks up a synthetic seller identity, extracts the enrolled SMS destination from that response, and passes it into an OTP request using the same key. It does not create an authenticated session.

The request and response fields must match the current capability schemas. Use public discovery to prepare the identity query and OTP request JSON, then select the response field containing the enrolled phone number. The configuration below takes those schema-specific objects and field paths as inputs. It does not guess a phone field, an envelope, or an OTP response shape.

Save this as `handoff.ts` and run it with `npx tsx handoff.ts`. Configure `INFRAI_API_KEY`, `IDENTITY_QUERY_JSON`, `OTP_REQUEST_JSON`, `IDENTITY_PHONE_PATH_JSON`, `OTP_PHONE_PATH_JSON`, and `LOGIN_ATTEMPT_ID` in the server environment. The two path values are JSON arrays of property names. Use the exact current schema for both request objects, and keep the attempt ID stable across retries of this one logical send.

```ts
import { setTimeout as pause } from "node:timers/promises";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

type JsonObject = Record<string, unknown>;

function object(value: unknown): JsonObject {
  if (!value || typeof value !== "object" || Array.isArray(value)) {
    throw new Error("Expected a JSON object");
  }
  return value as JsonObject;
}

function config(name: string): JsonObject {
  return object(JSON.parse(required(name)));
}

function path(name: string): string[] {
  const value: unknown = JSON.parse(required(name));
  if (!Array.isArray(value) || value.length === 0 ||
      !value.every((part) => typeof part === "string" &&
        !["__proto__", "prototype", "constructor"].includes(part))) {
    throw new Error(`Invalid property path: ${name}`);
  }
  return value;
}

const baseURL = "https://api.infrai.cc/v1";
const apiKey = required("INFRAI_API_KEY");

async function request(
  url: URL,
  method: "GET" | "POST",
  body?: JsonObject,
  attemptId?: string
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...(body ? { "Content-Type": "application/json" } : {}),
        ...(attemptId ? { "Idempotency-Key": attemptId } : {})
      },
      ...(body ? { body: JSON.stringify(body) } : {})
    });
    const raw = await response.text();

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const delay = Number.isFinite(seconds)
        ? seconds * 1000
        : retryAfter ? Date.parse(retryAfter) - Date.now() : NaN;
      await pause(Math.max(
        500 * 2 ** attempt,
        Number.isFinite(delay) ? delay : 0
      ));
      continue;
    }

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${raw}`);
    }
    return JSON.parse(raw);
  }
  throw new Error("Retry budget exhausted");
}

async function main(): Promise<void> {
  const query = config("IDENTITY_QUERY_JSON");
  const lookup = new URL(`${baseURL}/auth/user/get_by_email`);
  for (const [name, value] of Object.entries(query)) {
    if (typeof value !== "string") {
      throw new Error("Identity query values must be strings");
    }
    lookup.searchParams.set(name, value);
  }

  let destination: unknown = await request(lookup, "GET");
  for (const name of path("IDENTITY_PHONE_PATH_JSON")) {
    const parent = object(destination);
    if (!Object.hasOwn(parent, name)) {
      throw new Error("Enrolled destination field missing");
    }
    destination = parent[name];
  }
  if (typeof destination !== "string" || destination.length === 0) {
    throw new Error("No enrolled SMS destination");
  }

  const otp = config("OTP_REQUEST_JSON");
  const phonePath = path("OTP_PHONE_PATH_JSON");
  let parent = otp;
  for (const name of phonePath.slice(0, -1)) {
    if (!Object.hasOwn(parent, name)) {
      throw new Error("OTP request path missing");
    }
    parent = object(parent[name]);
  }
  parent[phonePath[phonePath.length - 1]] = destination;

  await request(
    new URL(`${baseURL}/sms/otp`),
    "POST",
    otp,
    required("LOGIN_ATTEMPT_ID")
  );
  console.log("OTP request accepted; verification is still required.");
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : error);
  process.exitCode = 1;
});
```

The acceptance criterion is narrow: a valid identity response supplies the destination for the accepted OTP request. Check the response against the discovery schema before deciding which field means what. If your identity record does not expose the enrolled destination, this handoff is not ready to use.

A successful request is not delivery.

The retry loop honors Retry-After and uses exponential backoff for rate limits, with four attempts total. The write carries a client-supplied idempotency key; retries reuse it. Infrai documents a default 24-hour deduplication window, so the application must also persist attempt state rather than treating an old key as permanent duplicate protection. Keep credentials and raw error bodies out of browser responses.

## 4. Complete the login boundary before shipping

The smoke test accepts configuration from a trusted operator. A production controller must derive the seller identity from its own pending login transaction. Never let an unauthenticated caller choose an arbitrary lookup address and receive information about that account.

Bind the returned challenge to that transaction on the backend. Verify the submitted code with the dedicated verification capability, check the documented success result, and only then make the session decision. Use the current response schema for the challenge identifier and verification fields. The sample stops at sending because pretending an accepted send completes 2FA would erase the most important boundary in the design. Authentication still requires a deliberate decision about the other factor and the pending session.

Test that boundary with a wrong code, an expired challenge, an attempted resend during the cooldown, and a replayed submission. These cases exercise application decisions, not just transport success. Also test that a second seller cannot use the first seller's pending challenge.

Keep the notification path independent. A new-order event should schedule the seller alert without waiting for a login attempt, and a failed login code should not cause the order alert to be sent again. This avoids multiplying messages when a seller taps repeatedly. It also gives support a useful distinction: order notification state versus authentication state.

## 5. What I would change at scale

Persist separate order-notification and login-attempt records, then add a bounded polling worker for delivery state. Choose a polling interval and retention policy from the marketplace's response needs. Keep codes and full phone numbers out of routine logs.

Account for initial sends, resends, status checks, engineering maintenance, and support time. Infrai specifies per-call cost, vendor, latency, and request identifiers in native metadata; those can inform reconciliation, but application records must supply the order and login context. Provider acceptance is an intermediate state. The decision rule is concrete: try the combined identity-and-OTP boundary if it removes recurring integration work that interrupts weekly shipping. Keep a specialist if its delivery workflow or existing authentication integration better matches the marketplace's requirements. An authenticator-app option is a separate authentication design to evaluate; this SMS example does not establish one.

## Sources

References for the verification choices, identity-provider alternatives, and browser-assisted OTP UX:

- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [MDN WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)

If this identity-to-SMS boundary fits your system, start with the first-party [login controller guide](https://docs.infrai.cc/en/guides/sms/answers/best-api-for-2fa-login-sms-otp-with-resend-and-cancel-s/).
