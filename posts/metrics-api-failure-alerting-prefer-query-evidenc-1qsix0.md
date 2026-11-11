# Metrics API Failure Alerting: Prefer Query Evidence Over Webhook-Only Notifications

| Approach | What remains after a shipment incident | Choose it when |
| --- | --- | --- |
| Metrics polling with evidence capture | A retained query result, evaluation rule, and notification outcome | Support must reconstruct why an alert fired |
| Notification-only polling | The message, plus whatever the underlying system still retains | Another system already preserves the required evidence |

Short answer: prefer metrics API polling with evidence capture when a logistics customer needs an explanation after a shipment failure. Query the metric, retain the observation and decision, then attempt the webhook. Notification-only polling is the smaller implementation when an existing incident record already supplies that history.

For a one-person SaaS, the useful revenue-per-hour question is how much time the next customer investigation will consume. A webhook can interrupt the founder. The retained evidence should explain what happened after the interruption is over. Ship weekly, but make each alert leave something support can actually use.

## How should a small app retain metrics API query evidence for failure alerting?

Consider an illustrative delivery-status workflow: a customer asks why shipment updates stopped, while the application has already resumed processing. This is an example to design against, not a reported incident. A current health check cannot establish what an earlier metrics query returned. If the only surviving artifact says that failures crossed a threshold, support still needs the evaluated time window, the population being counted, and the meaning of failure.

Keep the window.

The first decision criterion is reconstruction completeness. Retain the query identifier or expression, its parameters, the observation timestamp, the returned measurement, the rule version, and the resulting decision. For a shipment workflow, define whether the measurement counts unsuccessful update attempts, distinct affected shipments, or final delivery outcomes. Those definitions answer different customer questions. A retry that succeeds later should not quietly change the meaning of an earlier alert.

The Google SRE monitoring chapter distinguishes symptoms from causes and recommends examining latency, traffic, errors, and saturation. For this workflow, use that distinction to write down what the poll proves: an elevated error measurement is a symptom; the reason a particular shipment update failed requires separate evidence. A query result should therefore link to the relevant application records through a bounded lookup key. Avoid copying customer addresses or whole shipment payloads into every alert.

That boundary matters. An aggregate measurement cannot, on its own, reconstruct an individual customer's sequence of events. If customer reconstruction is the requirement, preserve the application event history as well, with timestamps and correlation identifiers, and make the alert point into it. Set the retention period from the support investigation window; do not invent a universal number of days.

## Separate a failed shipment signal from a failed observation

The second criterion is whether the retained record preserves uncertainty. An empty result, an expired query window, and an exception while fetching metrics must not silently become a healthy observation. Represent observation failure separately from the shipment rule's result, so a later investigation can distinguish a quiet application from a blind monitor.

For a concrete design exercise, suppose the rule evaluates a 5-minute window every 1 minute. Those are example settings, not measured recommendations. Consecutive evaluations overlap. Counting each alert as a new customer incident would exaggerate the event history, while discarding every repeated observation would erase useful evidence about duration. Retain observations individually and group notifications under an incident identity defined by the rule and the affected scope.

The trade-off is extra storage and code, but the benefit is a retained explanation.

Also record the application release and rule version where available. During a weekly deployment, a changed definition of failure can otherwise look like changed shipment behavior. Keep the decision inputs interpretable after the next release, including the unit and aggregation semantics. Support should not need to recover an old source tree just to learn whether a value counted attempts or shipments.

## Put persistence before the webhook attempt

The focused implementation below separates three responsibilities: fetch the observation, retain it, and attempt notification. It uses TypeScript contracts for application-owned adapters. These interfaces are illustrative; they do not claim to match a public metrics API or a particular serverless runtime. A scheduled Node.js or Lambda handler can call the same function after supplying those adapters.

```ts
type Observation = {
  id: string;
  queryId: string;
  ruleVersion: string;
  scope: string;
  observedAt: string;
  windowStart: string;
  windowEnd: string;
  failedUpdates: number;
  threshold: number;
  eventLookupKey: string;
};

type Evidence = Observation & {
  decision: "alert" | "quiet";
};

type Dependencies = {
  query: () => Promise<Observation>;
  retain: (record: Evidence) => Promise<void>;
  notify: (record: Evidence) => Promise<void>;
  recordAttempt: (
    observationId: string,
    outcome: "accepted" | "failed"
  ) => Promise<void>;
};

export async function pollOnce(deps: Dependencies): Promise<void> {
  const observation = await deps.query();
  if (
    !Number.isFinite(observation.failedUpdates) ||
    !Number.isFinite(observation.threshold) ||
    observation.failedUpdates < 0 ||
    observation.threshold <= 0
  ) {
    throw new Error("Invalid shipment metric or threshold");
  }

  const record: Evidence = {
    ...observation,
    decision:
      observation.failedUpdates >= observation.threshold
        ? "alert"
        : "quiet",
  };
  await deps.retain(record);
  if (record.decision === "quiet") return;

  try {
    await deps.notify(record);
  } catch (error) {
    await deps.recordAttempt(record.id, "failed");
    throw error;
  }
  await deps.recordAttempt(record.id, "accepted");
}
```

The query adapter must validate the response schema, timestamps, scope, and freshness before returning an observation; missing data should reject the query rather than manufacture a zero. Define the notification adapter so it rejects transport failures and unacceptable responses. Its success means the receiver accepted the request, not that a human read it. These are contract requirements for the implementation, not guarantees supplied by the function.

Notice the ordering: a persistence failure stops the notification path. That is a deliberate trade-off for a system where every emitted alert must have retained evidence. It also means this small function needs an independent signal for poller failures. Otherwise the evidence store becomes a reason the founder never receives a shipment alert.

There is another boundary. A process can stop after retaining the observation but before notifying, or after the receiver accepts the webhook but before the attempt record is written. This function does not make those operations atomic. For recovery across those gaps, persist pending notification work alongside the evidence, then let a worker deliver it with retries and a stable deduplication identity. Keep the incident identity distinct from each observation ID. Outsource the undifferentiated scheduling and durable delivery work where practical; the shipment rule and its evidence contract still belong to the application.

## Test the record a customer investigation will need

Start with an illustrative shipment update event and a matching aggregate observation. Replay an above-threshold observation, an empty response, a stale window, a rejected webhook, and a persistence failure. Inspect the retained artifacts and pending work. The test should establish whether the eventual investigation can distinguish each outcome, including which decisions were never delivered.

Then change the rule version and repeat. Old records shouldn't lose their old interpretation.

Before each weekly release, check that the deployed scheduler invokes the intended rule and that event lookup keys still resolve within the chosen retention window. Keep a short investigation procedure: locate the observation, inspect its rule and window, follow the application event history, and inspect notification attempts. This is a better acceptance criterion than merely seeing a test message arrive.

## When notification-only polling is enough

Choose notification-only polling when the underlying application already keeps a complete, searchable shipment history and preserves the exact metrics evaluation elsewhere. The webhook can then carry a stable reference to that evidence. A second evidence store may add maintenance without improving reconstruction. Verify that assumption with an old incident-shaped test record, not just today's dashboard.

For a small application needing only a basic failure signal, a scheduled query and webhook can also be a reasonable starting boundary. Human acknowledgment, escalation, and ownership transfer remain separate requirements; adding a webhook does not define them. If those become necessary, reassess the notification workflow independently of the shipment evidence model.

For the logistics case here, retain the decision evidence unless another system demonstrably supplies it. Favor the option that lets a solo founder answer the customer's next question and return to shipping features.

## References

- Google SRE Book, Monitoring Distributed Systems: symptoms versus causes, actionable alerting, and the four golden signals. https://sre.google/sre-book/monitoring-distributed-systems/
