# Template Ownership for Signup Verification Email: DKIM, SPF, First Node.js Send

Keep the verification email template in your application repo and use the sending API as transport only. That rule holds up against the two things that actually break a B2B SaaS signup: a custom domain nobody authenticated with SPF and DKIM, and a template nobody can diff before the first send.

| Where the template lives | Choose it when | The cost you accept |
| --- | --- | --- |
| Your repo, rendered in Node.js | Engineers are the only people editing copy, and the message carries a security-sensitive link | A copy fix needs a deploy |
| The provider's hosted template store | Non-engineers edit copy, or many locales change on their own schedule | The rendered output is not in your pull request |
| Split — hosted layout, dynamic parts injected at send time | Brand chrome is stable, per-account content is not | Two systems must agree on variable names |

For a one-person SaaS shipping weekly, row one wins almost every time, and the reason has nothing to do with elegance. A verification message is transactional infrastructure with a token in it. It should be reviewed, tested and rolled back the same way the endpoint that mints the token is.

## Authenticate the custom domain, then test what receivers actually see

Nothing about template ownership matters if the message lands in spam. Domain authentication is the setup step that gates everything else, and it's one-time work — the kind of undifferentiated chore worth doing once, carefully, and never touching again.

Three records do the job. SPF, defined in RFC 7208, is a TXT record listing which infrastructure may send for the domain. DKIM, defined in RFC 6376, is a key pair: the sending service signs each message with a private key, receivers fetch the public key from `<selector>._domainkey.yourdomain.com` and validate the signature. DMARC (RFC 7489) is the policy that ties them to the visible `From` header through alignment, and it's the reason a passing DKIM signature on the wrong domain still buys you nothing.

Two practical choices are worth making deliberately. Send transactional mail from a subdomain — `mail.example.com` rather than the apex — so that a future marketing blast can't drag the reputation of your signup mail down with it. And publish DMARC at `p=none` first, read the aggregate reports for a week, then tighten to quarantine or reject once you can see every source that claims your domain.

Do this part first.

Confirm the records resolve from outside your own network before you send anything to a real user:

```bash
dig +short TXT signup._domainkey.mail.example.com
dig +short TXT mail.example.com
dig +short TXT _dmarc.example.com
```

DNS propagation timing depends on the TTL you published and on caches you don't control, so a 3600-second TTL means "up to an hour", not "instantly". I'm not sure any universal wait estimate is useful here. What resolves the uncertainty is the query above returning what you expect, from a machine that isn't yours.

## Should the verification email template live in your Node.js code or in the sending API?

The first criterion is who edits the copy, and how often. In a company with a marketing team, hosted templates earn their keep: someone changes a subject line at 4pm without opening an editor or waiting for CI. In a one-person shop that person is you, and you're already deploying — the hosted editor removes a step you don't have.

The second criterion is reproducibility, and it's the one that decides it after launch. A user writes in saying the verification link didn't work. With the template in the repo you can check out the commit that was live, render the message with that user's token, and read exactly what they received. With hosted templates you're reading a dashboard that shows the current version, not the version that shipped last Tuesday.

That gap widens as soon as you have tests. A rendered template is a pure function from data to HTML plus text, which means a snapshot test in CI catches a broken interpolation before it reaches an inbox. A hosted template can only be tested by sending mail and looking at it.

There's a third argument I'll admit is weaker: reviewability. Copy changes in a diff get read; copy changes in a web form usually don't.

So: repo by default.

## A minimal Node.js example: render in your repo, post to the API

The first send should be small enough to read in one screen. The application owns rendering; the transactional email API receives finished HTML and text. Auth via environment variable, explicit method, one idempotency key per logical message, bounded retries that honor `Retry-After`, and non-success responses surfaced instead of swallowed.

```ts
import { randomUUID, createHash } from "node:crypto";

const MAIL_API = process.env.MAIL_API_URL;   // provider-agnostic transport
const MAIL_KEY = process.env.MAIL_API_KEY;
const FROM = process.env.VERIFY_FROM;        // an address on the authenticated subdomain

if (!MAIL_API || !MAIL_KEY || !FROM) throw new Error("mail transport is not configured");

// Template lives here, in version control, next to the code that mints the token.
function renderVerification(link: string, minutes: number) {
  return {
    subject: "Confirm your email address",
    text: `Confirm your address to finish setting up your workspace:\n${link}\nThis link expires in ${minutes} minutes.`,
    html: `<p>Confirm your address to finish setting up your workspace.</p>
<p><a href="${link}">Confirm email address</a></p>
<p>This link expires in ${minutes} minutes.</p>`,
  };
}

export async function sendVerificationEmail(userId: string, to: string) {
  const token = randomUUID();
  const tokenHash = createHash("sha256").update(token).digest("hex");
  await storeVerificationToken({ userId, tokenHash, expiresInMinutes: 30 });

  const body = renderVerification(`https://app.example.com/verify?t=${token}`, 30);
  const idempotencyKey = `verify:${userId}:${tokenHash.slice(0, 16)}`;

  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${MAIL_API}/messages`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${MAIL_KEY}`,
        "content-type": "application/json",
        "idempotency-key": idempotencyKey,
      },
      body: JSON.stringify({ from: FROM, to: [to], ...body }),
    });

    if (res.ok) return res.json();

    const detail = await res.text();
    if (res.status !== 429 && res.status < 500) {
      throw new Error(`verification mail rejected (${res.status}): ${detail}`);
    }

    const retryAfter = Number(res.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter) && retryAfter > 0 ? retryAfter * 1000 : 500 * 2 ** attempt;
    await new Promise((r) => setTimeout(r, waitMs));
  }

  throw new Error("verification mail exhausted retries");
}
```

Three details in there are worth defending. The token is stored as a hash, so a leaked database row can't be replayed as a valid confirmation. The idempotency key is derived from the user and the token rather than generated per attempt, so a timeout followed by a retry produces one message instead of two — which is exactly the failure a nervous user turns into three duplicate welcome mails by hammering the button. And a 4xx that isn't 429 throws immediately, because a malformed payload will not fix itself on the second try.

One warning that catches people out: automated link prefetching. Mail privacy features and corporate security scanners fetch remote content and follow links before a human ever opens the message, so a verification link that flips account state on a plain GET can be consumed by a machine. Apple's Mail Privacy Protection loads remote content through a proxy, which also means an open event tells you very little about whether a person read anything. Make the link land on a page that requires a click to confirm, keep the token single-use with a short expiry, and let a second visit render "already verified" rather than an error. The same thinking applies to whatever metric you report to yourself about this flow: a proxy that fetches every image and follows every link inflates opens, deflates nothing, and quietly turns "90% of users opened the verification mail" into a number that means almost nothing — the only signal worth trusting is the confirmation your own endpoint recorded, keyed to the user, with a timestamp you wrote.

After the send response comes back, the work isn't done. Store the provider's message id against the user, record delivery and bounce state, and suppress future mail to hard-bounced addresses — otherwise you'll pay for it in domain reputation. Log the failure branch loudly; a signup flow that silently drops verification mail looks identical to a signup flow that works, right up until conversion drops.

## Where hosted templates win, and what migrating away costs

Stick with the provider's template store when people who don't deploy need to change copy, or when a dozen locales are maintained by translators on their own cadence. Pushing a translation update through a code review queue is a tax on both sides, and the audit UI most vendors put around hosted templates is genuinely useful when someone else has to sign off on wording.

The catch is that you're accepting a second source of truth for something with a security token in it, so keep the boundary narrow: hosted layout, application-supplied variables, and a rule that the link itself is always built in your code. Narrow boundaries are also what make the reverse move cheap — migrating a hosted template back into the repo is a copy-paste plus a snapshot test when the variables were yours all along, and a small archaeology project when they weren't.

Self-hosting an MTA is the option I'd push back on hardest for a small team. It's technically fine and operationally expensive — IP warming, feedback processing, blocklist monitoring — and none of that work shows up in the product. Buy the transport, own the template, and spend the recovered hours on the feature that made someone sign up in the first place.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://datatracker.ietf.org/doc/html/rfc7208
- https://datatracker.ietf.org/doc/html/rfc7489
- https://datatracker.ietf.org/doc/html/rfc8058
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
