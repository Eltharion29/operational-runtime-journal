# Cheapest Bulk SMS API? Cost-Testing SaaS Incident Alerts Across US/EU

Short answer: don't choose a bulk SMS alerts API from its headline rate; run the same US and EU incident workload against each no-monthly-minimum quote, then compare delivered-alert cost and operator time.

For a one-person SaaS, that last term matters. Shipping weekly means an hour spent reconciling messaging bills is an hour unavailable for product work. The useful target isn't the smallest number on a pricing page. It is the lowest repeatable cost for an alert that reaches the intended destination, leaves enough evidence to audit, and doesn't turn routine maintenance into a second job.

This is a cost-harness build log, not a vendor ranking. Telnyx, Bandwidth, Twilio, and Sinch can all be rows in the input sheet, but the supplied evidence does not establish comparable current SMS rates, country coverage, account minimums, or delivery behavior for them. Naming a winner anyway would be fiction. Current written quotes, terms, test results, and invoices are what resolve the question.

## The constraint that changes the comparison

Incident traffic is awkward to price because the workload used for evaluation has to represent both silence and a burst. A zero-send period tests the “no monthly minimum” requirement directly. A burst tests the thing the integration exists to do. Combining those cases into one average hides the fixed-cost question, while testing only the burst ignores the quiet months that a small SaaS will actually pay through.

So the comparison unit should be an incident trace, not a nominal SMS. Define one trace before asking for quotes: destinations by country, alert bodies, send times, expected recipient count, and the evidence required to call an alert delivered. Run that trace without changing copy or timing between candidates. Keep US and EU results separate. A blended total can make one region's weak economics disappear inside the other region's volume.

No shortcuts.

The revenue-per-hour calculation also needs a labor column. Record the minutes required to provision credentials, configure the sender, rotate a key, inspect a rejected test, reconcile an invoice, and remove a recipient. These aren't universal constants, and I'm not sure a useful public benchmark could make them universal: account terms, destinations, and an operator's familiarity all change the result. A timed drill on the actual account settles it.

Use a compact scorecard rather than a feature matrix:

| Measurement | US run | EU run | Quiet period |
|---|---:|---:|---:|
| Submitted alerts | Count from outbox | Count from outbox | 0 |
| Accepted requests | Count from adapter records | Count from adapter records | 0 |
| Delivered alerts | Count from verified status records | Count from verified status records | 0 |
| Total invoiced amount | Copy from dated invoice | Copy from dated invoice | Copy from dated invoice |
| Operator minutes | Time the drill | Time the drill | Time reconciliation |

Do not prefill the money cells from memory. Save the quote date and account terms with every run, then calculate `total invoiced amount / delivered alerts` for each region. Keep the quiet-period amount visible rather than distributing it across an imagined volume. This makes “no monthly minimum” an observed account condition instead of a label.

## How should a SaaS compare bulk SMS incident alerts in the US and EU?

Start at the application boundary. The incident engine should create an alert record and an outbox record together, using one stable internal alert ID. A worker submits that outbox item through a narrow transport adapter. The adapter returns an external correlation ID, and later status input appends evidence to the same internal record. Acceptance and delivery stay separate because they answer different questions in the cost harness.

The test dataset should be boring and fixed. Give every candidate the same redacted recipient set, ASCII and non-ASCII bodies, severity mix, destination split, and burst schedule. A useful fixture might contain 40 internal alerts: ten short US bodies, ten longer US bodies, ten short EU bodies, and ten longer EU bodies, each carrying a unique internal ID but no production phone number in the retained copy. Schedule the same fixture once during an ordinary window and once as a compact burst. Before sending, freeze the adapter revision and write down what counts as accepted, delivered, rejected, and unresolved. Afterward, preserve the exact submitted body, the internal ID, timestamps, adapter result, status history, invoice line mapping, and operator notes. Reconcile each internal alert to its external correlation ID, then reconcile the resulting chargeable units to the dated invoice without assuming that one internal alert maps to one billed unit. If a quote uses a unit that differs from an application alert, retain both values rather than silently treating them as equivalent. This one deliberately fussy record turns a vague pricing claim into a result another operator can inspect after the dashboard has changed.

Then run three drills. The first is a normal incident burst. The second repeats the same internal alert ID and verifies that the application does not create an uncontrolled duplicate submission. The third introduces an invalid fixture at the adapter boundary and confirms that it is quarantined with a useful local reason. A concrete regression test might expect `ALERT_PAYLOAD_INVALID` for fixture `eu-0042` when `to` is absent; that code belongs to the application, so it remains stable even if a candidate uses different response fields.

That separation saves investigation time. Without it, a generic “send failed” log forces the operator to rediscover whether the application rejected its own input, the request was accepted, or later delivery evidence never matched the internal record. With it, the decision sheet can be regenerated from durable records rather than screenshots.

For the commercial pass, request a dated answer to the exact same questions for every candidate: recurring account commitments, sender-related recurring charges, chargeable message unit, destination-specific additions, taxes or regulatory items, and cancellation conditions. Treat an unanswered field as unknown. Don't convert unknown into zero.

## The smallest TypeScript cost harness

The application needs one interface and one recorder. Provider-specific authentication, request paths, signatures, and response shapes stay inside adapters because those details must come from the current contract and documentation for the account being tested. The sample uses an injected function, so it invents no commercial endpoint.

```ts
type Region = "US" | "EU";

interface AlertInput {
  alertId: string;
  region: Region;
  to: string;
  body: string;
}

interface Submission {
  externalId: string;
  acceptedAt: string;
}

interface TrialRow {
  alertId: string;
  region: Region;
  startedAt: string;
  finishedAt: string;
  accepted: boolean;
  externalId?: string;
  localError?: "ALERT_PAYLOAD_INVALID" | "ADAPTER_REJECTED";
}

type SubmitAlert = (input: AlertInput) => Promise<Submission>;

function validate(input: AlertInput): void {
  if (!input.alertId || !input.to || !input.body) {
    throw new Error("ALERT_PAYLOAD_INVALID");
  }
}

export async function runTrial(
  input: AlertInput,
  submit: SubmitAlert,
): Promise<TrialRow> {
  const startedAt = new Date().toISOString();

  try {
    validate(input);
    const result = await submit(input);
    return {
      alertId: input.alertId,
      region: input.region,
      startedAt,
      finishedAt: new Date().toISOString(),
      accepted: true,
      externalId: result.externalId,
    };
  } catch (error: unknown) {
    const message = error instanceof Error ? error.message : "ADAPTER_REJECTED";
    return {
      alertId: input.alertId,
      region: input.region,
      startedAt,
      finishedAt: new Date().toISOString(),
      accepted: false,
      localError:
        message === "ALERT_PAYLOAD_INVALID"
          ? "ALERT_PAYLOAD_INVALID"
          : "ADAPTER_REJECTED",
    };
  }
}
```

This function records submission evidence only. A separate status-ingestion path should authenticate input according to the selected service's current specification, parse the body as `unknown`, map it to the internal alert ID, and append the result. Keeping status processing out of `runTrial` avoids pretending that all candidates share one callback contract.

The outbox worker should own retry scheduling and use the stable internal ID when the selected API offers an idempotency mechanism. Put retry counts, queue age, and the final adapter classification into the trial export. Redact phone numbers and credentials before retaining fixtures. The harness should make a fresh adapter cheap to test, but it should make accidental behavioral differences obvious.

## What changes at scale, and where does this method stop?

At low volume, one adapter and a separately tested fallback channel minimize ownership work. Automatic multi-provider routing adds credentials, sender setup, status mappings, billing reconciliation, and more branches to exercise during every release. That machinery earns its place only when measured regional volume or a defined availability requirement pays back the maintenance. Until then, outsource the undifferentiated transport and keep routing policy in application code.

At higher volume, split transport selection from incident policy. Retain dated regional cohorts, compare delivered-alert cost over the same observation window, and replay a fixed canary set after an adapter change. A weekly release should not quietly alter the evidence vocabulary or invoice mapping. Version those mappings like code.

The catch is that a no-minimum API is not suitable when the organization needs contractual capacity, procurement controls, sender-management support, or regional commitments that a usage-only account cannot supply. In that case, stop optimizing for the absence of a monthly commitment and evaluate the contract that satisfies those operational requirements. Likewise, stick with one transport when it passes the defined drills and its operator cost is acceptable; add routing complexity only after measured evidence justifies it.

SMS also should not carry the entire incident workflow. The application still needs escalation state and human acknowledgement; transport evidence alone does not prove that a person acted. An email path may be part of a separate fallback design, but email has its own API and compliance context. Resend's introduction documents an email API, and the FTC guide addresses CAN-SPAM requirements for commercial email. Neither source substantiates SMS pricing, SMS delivery, or the four-candidate comparison, so neither can fill those evidence gaps.

The final selection is deliberately mechanical: discard candidates that fail a required regional drill or account-term constraint, then compare the remaining delivered-alert cost and operator minutes. Re-run the harness when terms, destinations, message bodies, or operational requirements change. Cheapest is a result tied to a workload and a date, not a permanent vendor attribute.

## References

- https://resend.com/docs/introduction
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
