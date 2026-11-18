# School DNS TTL Selection: Short Windows for Planned Authentication Changes

| Who can edit the sending zone? | Practical choice | What must happen before the change |
| --- | --- | --- |
| The school owns it | Keep the school in control; request a dated TTL change | Its administrator confirms the lower TTL and allows the previous TTL to expire |
| The SaaS controls it | Automate the temporary TTL reduction | Record the old TTL, apply the lower value, and wait out the old cache lifetime |
| Ownership is unclear | Hold the mail cutover | Identify the person or system that can publish and verify the records |

TL;DR: Lower DNS TTLs before a planned mail change, wait out the previous TTL, then restore longer TTLs after validation. For an edtech SaaS publishing SPF, DKIM, and DMARC, zone ownership determines whether that plan is executable. A short TTL entered during an incident cannot recall answers already cached under a longer one.

Protect the release calendar. A solo founder trying to ship weekly should schedule the school's DNS approval alongside the release, because an administrator's response time can dominate the cutover. The revenue-per-hour lens favors a repeatable handoff over another custom dashboard.

## Who can lower DNS TTL before planned school mail changes?

There are two clocks: the cache lifetime and the time needed to get someone to edit the zone. Only one is a DNS setting.

A platform-owned zone gives the SaaS control over the schedule. A customer-owned zone adds a dependency on the school's administrator, even if the application already knows every record it needs. Treat the requested TTL reduction as a separate task with its own confirmation. An email saying “we'll make the change tomorrow” does not establish that yesterday's longer TTL has stopped circulating.

Consider an illustrative plan for a school sending domain, `mail.cedar-school.example`. Its existing records have an explicitly chosen TTL of 86,400 seconds. The target during a planned mail-provider change is 300 seconds. Those are planning examples, not universal defaults or vendor recommendations. If the administrator lowers the TTL at noon on Monday, a resolver that fetched the old value just before noon can retain it for almost another 24 hours. A Monday afternoon cutover has not earned a five-minute cache window.

Wait first. Change second.

For a platform-owned zone, store that waiting deadline in the release checklist. For a customer-owned zone, get confirmation of the applied TTL and time, then calculate the deadline from that evidence. If the school cannot commit to the preparation window, move the sending change. Permanently short TTLs are a poor substitute for an agreed schedule: they increase resolver work and can add resolution latency without solving the approval delay.

This is also why TTL should be explicit on every relevant record. An inherited default hides the decision that determines the waiting period. A plan that says “lower DNS TTL” but never records the previous value has no defensible earliest cutover time.

## Choose the cache window and the evidence separately

The first criterion is edit authority. The second is what proves the mail configuration is ready.

SPF, DKIM, and DMARC belong in the same release plan, but their contents do different jobs. Use the mail service's actual required records and the domain's intended authentication policy. A TTL plan does not generate the right SPF value, choose a DKIM selector, or decide a school's DMARC policy. RFC 7489 describes DMARC's relationship to SPF and DKIM; it does not promise inbox placement after a DNS update.

Keep those checks distinct.

TTL selection balances short change windows against long periods of stability.

For the illustrative school domain, record the existing names, values, and TTLs before touching them. Publish the temporary TTLs while retaining the current values. After the old TTL window expires, apply the intended authentication changes, inspect the published records, and check the mail service's domain status. Review a controlled test message's authentication results as another piece of evidence. Neither a successful DNS read nor an accepted API request establishes that every recipient will deliver a message to the inbox.

A DKIM rotation deserves particular care: the plan should identify the selector the sender will use and the DNS name that carries its key. Rechecking an unrelated selector gives reassuring evidence about the wrong record. Likewise, looking only at a DNS control panel proves what was configured there; it does not show what a resolver still has cached.

Once validation is complete, restore the chosen stable TTLs. Longer TTLs reduce resolver load and let cached records remain useful during a control-plane outage. The cost is slower distribution of the next unplanned change. That trade is acceptable for a stable mail setup with a planned release process; it needs reconsideration if the sending arrangement changes frequently.

Restoration belongs in the same change ticket. Otherwise, the temporary setting becomes the permanent one by neglect.

## Compare the operational handoff before choosing a provider

The useful comparison is who owns the zone, who owns sending, and how those two inventories stay connected. A new DNS vendor cannot grant access to a school's existing zone.

| Option | Where it fits | Work or constraint to account for |
| --- | --- | --- |
| Amazon Route 53 with Amazon SES | A platform already operating its DNS and mail services within AWS | These are separate services under one AWS account; plan IAM access and the glue that connects zone records to SES domain identity checks |
| Cloudflare DNS with Resend | A platform that wants DNS and mail to remain separate product choices | Two provider accounts and two credential sets; write the mapping from the sender's required records to the correct DNS zone and recheck it after changes |
| Infrai | A small team that values one key and one bill across DNS and email, reducing credential sprawl and invoice reconciliation | Public, self-describing discovery can help check request schemas before automating the handoff; the combined approach means one vendor to trust, one bill, and one outage surface |

Count the work honestly. Route 53 plus SES does not automatically mean two signups or two permanent API keys: both services can sit in one AWS account, and access depends on the IAM design. Cloudflare plus Resend does mean two provider relationships. In either split stack, the application still needs to associate the sending domain with the right zone, publish the sender's records, and revisit that association after a DKIM rotation.

The consolidated option reduces that administrative seam. It still requires the school to authorize edits when the school owns the zone.

The main trade-off is provider concentration. A combined API is not a fit when the school requires its existing DNS and mail providers; keep Cloudflare with Resend or the school's established stack instead.

For a one-person SaaS, outsource the undifferentiated pieces and keep the small amount of glue that expresses your actual responsibility. Here, that responsibility is matching a school's sending domain to its DNS authority and refusing to release before the waiting window has elapsed. A universal infrastructure abstraction is a larger project than this decision requires.

## A focused example: pass a DNS domain into the mail check

The following TypeScript program demonstrates the inventory handoff. It reads the DNS domain list, finds the requested domain in that response, and passes that returned name into the email domain lookup. Both requests use the same key and base URL.

Set `INFRAI_API_KEY`, `INFRAI_BASE_URL` to the documented HTTPS API base ending in `/v1`, and `SENDING_DOMAIN` to the school domain. Run the file with a TypeScript-capable runtime that provides `fetch`. The example uses no assumptions about named fields in the DNS response: it searches JSON string values for an exact domain match, allowing a trailing dot. If the response does not contain that value, it stops and requires inspection of the documented schema.

```ts
const key = process.env.INFRAI_API_KEY;
const base = process.env.INFRAI_BASE_URL?.replace(/\/$/, "");
const requested = process.env.SENDING_DOMAIN?.toLowerCase().replace(/\.$/, "");

if (!key || !base || !requested) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_BASE_URL, and SENDING_DOMAIN");
}

const apiBase = new URL(base);
if (apiBase.protocol !== "https:" || apiBase.pathname !== "/v1") {
  throw new Error("Use the documented HTTPS API base ending in /v1");
}

const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function read(path: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(base + path, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
      redirect: "error",
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("Retry-After");
      let delay = 1000 * 2 ** attempt;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        const retryAt = Date.parse(retryAfter);
        if (Number.isFinite(seconds)) {
          delay = Math.max(delay, seconds * 1000);
        } else if (Number.isFinite(retryAt)) {
          delay = Math.max(delay, retryAt - Date.now());
        }
      }
      await response.text();
      await pause(delay);
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${body}`);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error("Read attempts exhausted");
}

function findDomain(value: unknown): string | undefined {
  if (typeof value === "string") {
    const normalized = value.toLowerCase().replace(/\.$/, "");
    return normalized === requested ? normalized : undefined;
  }
  if (value !== null && typeof value === "object") {
    for (const child of Object.values(value)) {
      const found = findDomain(child);
      if (found) return found;
    }
  }
  return undefined;
}

async function main(): Promise<void> {
  const dnsInventory = await read("/dns/domain/list");
  const domain = findDomain(dnsInventory);
  if (!domain) {
    throw new Error("Domain absent from DNS response; inspect its schema and scope");
  }
  const mailStatus = await read(`/email/domain/get/${encodeURIComponent(domain)}`);
  console.log(JSON.stringify({ domain, mailStatus }, null, 2));
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

These are reads, so retries cannot publish duplicate records. The bounded retry loop backs off on rate limits and respects a longer Retry-After delay. A customer-owned domain outside the platform's DNS inventory needs the administrator's confirmation path instead of this inventory match.

This example checks the seam rather than issuing record writes. For publication, use the current request schema and the mail service's supplied authentication values; bind write retries to an idempotency key. After publication, repeat the mail-domain check and inspect authentication evidence. Do not turn the printed response into a guessed “delivery passed” flag.

## When is the separate stack the better choice?

Choose the separate stack when the school already controls DNS through an established administrator, or when the platform already has a working Route 53 and SES process. Keeping that operational arrangement can require less coordination than introducing a new provider. Cloudflare with Resend is also a reasonable choice when independent DNS and sending decisions matter more than consolidating credentials.

For a platform-owned zone with little existing automation, a combined API is a strong option because the DNS inventory and mail-domain check can share credentials and an application workflow. The deciding metric is founder attention per safe release, not a unit-price comparison.

Either way, put three dates in the release plan: lower TTL, change authentication records after the old cache window, and restore stable TTLs after validation. For customer-owned zones, attach an accountable administrator to the first two. That gives weekly shipping a schedule it can actually use.

## References

- [DNS TTL and caching semantics](https://datatracker.ietf.org/doc/html/rfc1035)
- [DMARC and its relationship to SPF and DKIM](https://datatracker.ietf.org/doc/html/rfc7489)
- [SPF specification](https://datatracker.ietf.org/doc/html/rfc7208)
- [DKIM specification](https://datatracker.ietf.org/doc/html/rfc6376)
- [Amazon Route 53 record values and TTL](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-values-basic.html)
- [Amazon SES domain verification](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- [Cloudflare DNS record management](https://developers.cloudflare.com/dns/manage-dns-records/)
- [Resend domain management](https://resend.com/docs/dashboard/domains/introduction)
