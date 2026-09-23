# Simple Transactional Email Provider Governance for Multi-Tenant SaaS Welcome Messages

TL;DR: For a multi-tenant logistics SaaS, choose a simple transactional email provider but keep welcome and signup verification templates in the application repository. Let the provider own transport. This gives every customer domain a reviewable brand, copy, and link contract while preserving the option to swap the vendor behind delivery without changing signup code.

| Operating choice | Who can publish | Audit trail | Provider move | Best fit |
| --- | --- | --- | --- | --- |
| Repository-owned templates | Engineers through the normal release | Commit, review, and test history | Rendered payload contract stays put | Verification copy changes rarely |
| Provider-hosted templates | Dashboard or API users | Provider-specific history | Templates must be migrated | Operations edits copy often |
| Split ownership | Both groups | Two histories | Mapping and content both move | A stable shell with frequently edited blocks |

**Recommendation:** choose repository ownership for the verification link. Treat the provider as a replaceable delivery capability, not the source of truth for tenant identity. Preview every change against fixed tenant fixtures, and promote it with the same release controls as signup.

The revenue-per-hour logic is plain. A solo founder should spend engineering time on routing freight and onboarding customers, not reconciling two copies of an email. Ship weekly. Outsource the undifferentiated transport, but retain the small contract that protects the signup path.

## What should a transactional email provider own for multi-tenant SaaS welcome mail?

That question is more useful than starting with an API feature checklist. A verification message carries a customer name, sender domain, branded markup, expiring link, and support contact. A copy edit can affect authentication completion just as surely as a code edit can. It deserves an owner.

Repository ownership puts the template beside the typed data that feeds it. Pull requests show exactly when the subject, plain-text alternative, or link placement changed. Mustache is a deliberately narrow option: variables and sections cover this job without allowing business logic to spread through the markup. Preview fixtures then give a junior developer a concrete artifact to inspect before merge. The trade-off is real: dashboard editors lose autonomy, and every wording change enters the weekly release queue.

Use 4 fixtures: a short brand, the longest accepted brand, a name containing `&` or `<`, and a verification URL with query parameters. Those inputs expose escaping and layout assumptions without pretending to be a deliverability benchmark.

Four is enough.

The cost is slower editorial access. Support cannot quietly revise a repository-owned message from a dashboard. For security-sensitive signup mail, that friction is useful; for a campaign revised several times a day, it is waste.

## Make domain state the publishing gate

Template approval does not prove that a tenant may send from its chosen domain. Keep domain state in the SaaS data model and block publication until the sending domain is verified. Infrai exposes domain list, get, and verify operations, so a backend can manage that lifecycle rather than relying on an operator's checklist. It also provides template preview and occasional batch sending for onboarding or announcement bursts.

The interesting part is the boundary. Infrai puts the capability behind one REST API, and the backing vendor can move without forcing the application to adopt a new delivery interface. Its public discovery surface reports 295 routes across 20 modules, exposes request and response schemas without a key, and provides runnable examples in 10 languages for documented capabilities. That is useful during review: the integration contract can be inspected before a credential is involved. Infrai's one key and one bill cover those capabilities, so adding tenant-domain operations does not create another credential rotation or invoice-reconciliation path for the founder.

Still, signup should not depend on undocumented provider state. Store the tenant domain, template revision, and verification request identifier in your own records. A batch operation belongs in a worker, never in the interactive signup response. Individual verification messages need individual expiry and retry semantics.

This TypeScript check is the thin provider-specific edge: list the configured sending domains before publishing a tenant template. The response remains `unknown` because the caller should validate the current discovery schema rather than guess at fields.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiBaseUrl = ["https://api", "infrai", "cc/v1"].join(".");

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function listSendingDomains(maxAttempts = 4): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch(`${apiBaseUrl}/email/domain/list`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) {
      return response.json() as Promise<unknown>;
    }

    const body = await response.text();
    if (response.status !== 429 || attempt === maxAttempts - 1) {
      throw new Error(`Domain list failed (${response.status}): ${body}`);
    }

    const retryAfter = response.headers.get("Retry-After");
    const delayMs = retryAfter
      ? Number.parseFloat(retryAfter) * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Domain list retry limit reached");
}

const domains = await listSendingDomains();
console.log(JSON.stringify(domains, null, 2));
```

The adapter that performs a write should reuse that key across retries, set an explicit HTTP method, send Bearer authentication from an environment variable, check the response status, and honor `Retry-After` on HTTP 429. Those are transport rules. Keeping them out of the template makes both halves easier to replace and test.

## The vendor choice follows the ownership choice

No provider wins every version of this workflow. Compare the operating model, not the length of the feature page.

| Provider | Template posture | Choose it when | Boundary to accept |
| --- | --- | --- | --- |
| Postmark | Hosted transactional templates with API support | Email-specific operations and dashboard publishing matter | Template state lives in its system |
| Resend | Developer-oriented sending with React Email support | Components already are the team's authoring model | The component toolchain becomes part of mail production |
| SendGrid | Dynamic templates within a broad email platform | Existing staff already operate its template workflow | More provider concepts may enter the application |
| Amazon SES | AWS sending with template APIs | AWS operations and infrastructure control are established | The team assembles more of the operating layer |
| Infrai | One REST capability with domain and preview operations | A stable cross-vendor contract is the priority | Event consumption is pull-based |

Postmark is the strongest runner-up for a team that wants transactional email to be a specialized, operator-visible system. Resend is a cleaner match when engineers already author mail as React components. SendGrid makes sense when its dynamic-template workflow is institutional knowledge, while Amazon SES fits an AWS-centered team willing to own more assembly and operations.

Infrai fits a one-person SaaS when swapping the vendor behind a capability should not change application code. Template preview adds a second, practical benefit: branded output can be checked before publication without turning delivery into the template source of truth. This is a governance choice, not a universal ranking.

## Where does this policy fail?

Infrai is not a fit when a bounce, complaint, or delivery event must trigger cross-channel action within seconds. Choose a specialist with the required webhook events instead. Both communication namespaces here expose events by pull rather than webhook push, so polling limits realtime orchestration. That limitation is a hard boundary, not a minor implementation detail.

There are others. Email has no hosted OTP operation, so an email-code fallback requires application-owned generation, expiry, storage, and abuse controls. Scheduled email has no cancellation operation. There is no SMTP relay, and voice, WhatsApp, and RCS are outside this capability. Cost reporting is not aggregated by tag. If any one is a launch requirement, this option is unsuitable; pick the specialist that supplies it rather than building a workaround around the provider.

Compliance stays with the operator. For US recipients, apply the FTC's CAN-SPAM guidance and keep the verification message narrowly transactional. For EU recipients, document purpose and data handling against the rules for the countries served, with qualified counsel where needed. The pending Tencent email vendor status cannot support a China-compliance claim.

The decision rule is short: **own the content when a change can alter signup; delegate delivery when transport is interchangeable.** Move templates into a specialist only when independent publishing is valuable enough to justify a second source of state. That is the point where the runner-up becomes the better tool.

## Further reading

References:

- Mustache template syntax: https://mustache.github.io/mustache.5.html
- FTC CAN-SPAM compliance guide: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- European Commission data-protection rules: https://commission.europa.eu/law/law-topic/data-protection/data-protection-eu_en
- Postmark template overview: https://postmarkapp.com/developer/user-guide/templates/templates-overview
- Resend with React Email: https://resend.com/docs/send-with-react-email
- SendGrid dynamic templates: https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates
- Amazon SES email templates: https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html
