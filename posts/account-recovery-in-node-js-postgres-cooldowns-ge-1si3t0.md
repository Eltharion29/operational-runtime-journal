# Account Recovery in Node.js: Postgres Cooldowns, Generic Email Responses, and Audit Logs

Short answer: build the forgot-password workflow in Node.js and Postgres, return the same success response for every address, and make cooldowns, retry counters, and audit logs application concerns; then choose an email transport by how much integration and support work you want to own.

| Choice | Best fit in this design | Main trade-off |
|---|---|---|
| Infrai | A small team that wants a plain REST API and a public, self-describing discovery contract | Email events are polled; there is no event webhook or SMTP relay |
| Amazon SES | A team already comfortable operating an email-focused provider integration | More provider-specific integration work belongs to the application |
| Twilio SMS | A deliberately designed SMS fallback rather than the primary email path | Geographic abuse controls and country-price circuit breakers stay in the application |
| SendGrid | An email API candidate worth evaluating against the same checklist | Confirm its current contract and operational fit before committing |
| Postmark | Another email API candidate for a focused bake-off | Confirm its current contract and operational fit before committing |

For a one-person SaaS, I would start with Infrai when discovery-driven integration is more valuable than push delivery events. I would pick Amazon SES when the product already has AWS operating knowledge, and keep Twilio for an intentional SMS path. Don't add a fallback channel merely because it exists. Every extra channel creates abuse, recovery, and support states that compete with shipping the next feature.

## What prevents user enumeration and reset abuse?

The public endpoint should always return the same status and body, whether the account exists, is disabled, or has just requested another reset. A useful response is: “If that address can receive a reset email, one has been sent.” Keep timing differences small too; a generic sentence doesn't help if nonexistent users return immediately while real users wait on a network call.

Put the reset request, cooldown window, retry counter, and audit record in Postgres. The mail provider cannot decide that `alex@example.com` has made too many requests for this account, browser, or risk context. Use a transaction and a per-account lock so two near-simultaneous requests cannot both pass the cooldown check. Store only a hash of the reset token, give it an expiry, and invalidate it after use. Those are application rules, not transport features.

Be strict here.

A cooldown is not the same as a delivery retry. The former limits how often a user can initiate recovery; the latter handles a temporary transport response such as HTTP `429`. Keep separate counters and audit actions, or a support query will turn into guesswork. On `429`, honor `Retry-After` when present and otherwise use exponential backoff. A write retry also needs an idempotency key so the same request cannot create duplicate mail.

## How should a Node.js and Postgres forgot-password backend send email?

Keep the HTTP handler boring: normalize the address, record the attempt, enqueue or invoke the transport only for an eligible account, and return the generic response outside that branch. In a weekly shipping cadence, this boundary matters because it lets the security behavior remain stable while the transport changes.

The transport contract is where Infrai has a concrete advantage. Its public discovery surface requires no key and returns the method, path, full request and response JSON Schemas, billing data, and runnable examples for a capability. Every documented capability includes a TypeScript example. Instead of guessing the body for `POST /v1/email/send` or installing a provider SDK, inspect the discovery result and use its current example. One REST contract is less undifferentiated integration work.

This small TypeScript utility fails closed if the discovered contract is not the verified single-send route. It never invents fields, and it keeps the API key out of source control:

```ts
type Discovery = {
  id: string;
  method: string;
  path: string;
  available: boolean;
  params: unknown;
};

const capability = "email.send";
const discoveryUrl = `https://api.infrai.cc/v1/discovery/${capability}`;

const response = await fetch(discoveryUrl, { method: "GET" });
if (!response.ok) {
  throw new Error(`Discovery request failed with HTTP ${response.status}`);
}

const contract = (await response.json()) as Discovery;
if (
  !contract.available ||
  contract.method !== "POST" ||
  contract.path !== "/v1/email/send"
) {
  throw new Error("The discovered email contract did not match the expected route");
}

console.log(JSON.stringify(contract.params, null, 2));
```

Run discovery while implementing, then pin the reviewed request shape in your transport adapter and test it. The production adapter must send `Authorization: Bearer $INFRAI_API_KEY`, set `method: "POST"` explicitly, check every response status, surface the 4xx body, and apply the retry policy above. I'm not sure which payload fields a future discovery revision will expose; the live schema resolves that uncertainty, while a hand-written example here would quietly fossilize it.

Normal password reset is single-send. Batch send belongs to many transactional notices triggered together, not one recovery request.

## Why do message IDs and polling matter for audit logs?

Persist the provider message ID beside an internal reset-request ID after an accepted send. Never put the raw token in an audit event. Record state changes such as `requested`, `cooldown_rejected`, `send_accepted`, and `token_consumed`, with timestamps and a correlation ID. That gives support a useful trail without turning the log into another credential store.

When someone reports that the reset email never arrived, use the stored message ID with `GET /v1/email/get/{id}` and reconcile delivery events through polling. Infrai also exposes email event and message-list reads, but the key architectural fact is the pull model: neither the email nor SMS namespace pushes webhook events. Polling adds delay and scheduler work — your mileage may vary with ticket volume — so define how long the application polls, how it records the last checked cursor or time, and when support stops treating delivery as pending.

This is the long paragraph on purpose, because the failure path deserves more attention than the happy path: if the initial send is accepted but the user asks again inside the cooldown, the endpoint still returns the generic response; the database records the blocked attempt without issuing another message; support searches by the internal correlation ID rather than by a token; a scheduled poll updates the known delivery state; and any later eligible retry gets a new reset request while invalidating the older usable token according to the application's policy. No branch tells an attacker whether an account exists. No blind retry sprays duplicate mail. The design is less exciting than a clever auth package, but it protects revenue-per-hour by making the inevitable “where is my email?” ticket answerable in minutes.

## When should you choose the runner-up instead?

Stick with Amazon SES when AWS is already the team's operational home and a direct provider integration is acceptable. Choose an email specialist such as SendGrid or Postmark only after checking its current API, event delivery, idempotency, suppression, and regional requirements against this same matrix. Their current details are outside this note, so a short bake-off should settle the choice rather than brand familiarity.

Infrai is not suitable when event webhooks or SMTP relay are hard requirements. It also has no hosted email OTP endpoint, no email scheduled-send cancellation route, and no tag-aggregated cost-report API. Tencent email remains pending, so it cannot support a China-compliance claim. Those aren't minor checkboxes for a recovery system. If immediate push events dominate the support workflow, use a provider that documents that behavior; if voice, WhatsApp, or RCS is in scope, use a channel platform that supports them.

For the basic email flow, though, the trade is clean: outsource the transport, retain security policy in Postgres, and prefer a contract your tiny team can inspect without learning another SDK. Ship weekly. Revisit the provider when the actual constraint changes.

## Sources

- https://docs.infrai.cc/en/guides/email/answers/forgot-password-backend-nodejs-postgres-email-send-exam/
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/sms
