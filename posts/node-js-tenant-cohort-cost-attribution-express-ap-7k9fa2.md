# Logistics Cohort Cost Attribution for Express Server Failures

**Short answer:** Use Express error middleware to capture normalized server exceptions, expose recent grouped counts through a small polling API, and alert only when one fingerprint repeats; keep tenant cohort and cost data beside the event rather than inside its identity.

| Situation | Better first choice | Why |
| --- | --- | --- |
| A service needs a small repeated-failure alert | Grouped events plus a polling worker | The threshold is visible, testable, and easy to change |
| The team needs request-level investigation | Structured logs with trace correlation | Logs preserve context that an alert should omit |
| The team needs end-to-end latency analysis | Metrics and traces with a sampling policy | A thrown exception is only one part of a request path |
| The concern is a silent scheduled job | A heartbeat or completion signal | No exception is emitted when the job never starts |

The decision is about signal shape, not dashboard decoration. A raw stack trace is useful evidence. It is a poor alert key.

## What should an Express Node.js polling API capture before repeated-error alerts?

Capture the exception type, a normalized message, the route pattern, event time, tenant cohort, and an opaque correlation ID. Record the experiment arm and cost center separately when the alert is used to compare cohorts. Those fields let an operator ask, "Did failures rise in cohort B, and what did each failure cost?" without changing the identity of the underlying failure.

Do not place request IDs, timestamps, customer email addresses, shipment IDs, or database row IDs in the fingerprint. They change on every request. A message such as `warehouse 1842 timed out` should group with `warehouse 1843 timed out` after numeric values are replaced with a stable token.

Prometheus recommends naming metrics around the thing being measured and using consistent units; that principle applies here too. Name a counter for an event, such as `http_server_exceptions_total`, and keep dimensions bounded. A tenant ID for every customer is usually a high-cardinality cost attribution field, not a safe alert label. Store it in an event record or an offline aggregation path.

## A practical field guide for choosing the signal

Pick grouped exception events when the question is, "Is this same server failure repeating now?" The worker can query a five-minute window and page once after a threshold. This is a good fit for a small service where the team owns the rule and can tolerate polling delay.

Pick metrics when the important comparison is a rate across tenant cohorts. A counter for failed requests and a histogram for request duration make the denominator visible. For cost attribution, also record the experiment arm and a normalized cost unit in the event or measurement pipeline. Never infer cost from alert count alone: one failed batch may consume more compute than many failed health checks.

Pick traces when a failure crosses service boundaries. Sampling matters. Head sampling decides at trace start; tail sampling can use information collected later to retain interesting traces. The OpenTelemetry sampling model documents both approaches. Keep exception alerts independent from trace retention, because a sampled-out trace must not erase an alertable failure.

A heartbeat belongs beside these signals. It answers a different question: did the scheduled import or carrier-rate refresh run at all? An exception endpoint cannot answer that if the process died before the job began.

## How can you capture server exceptions and poll a simple error API?

The flow is intentionally plain: Express middleware turns failures into grouped records; the read endpoint returns only unresolved groups seen in the requested window; the worker applies the threshold and deduplicates notifications. The cohort is retained for cost analysis, while the fingerprint stays stable across tenants.

```ts
import express, { NextFunction, Request, Response } from "express";
import { createHash } from "node:crypto";

type ExceptionEvent = {
  fingerprint: string;
  name: string;
  message: string;
  route: string;
  cohort: string;
  costCenter: string;
  seenAt: string;
};

const app = express();
const events: ExceptionEvent[] = [];
const notified = new Set<string>();

function fingerprint(error: Error, route: string): string {
  const message = error.message
    .replace(/\b\d+\b/g, ":n")
    .replace(/[0-9a-f]{8}-[0-9a-f-]{27,}/gi, ":uuid");
  return createHash("sha256")
    .update(`${error.name}|${message}|${route}`)
    .digest("hex")
    .slice(0, 16);
}

function record(error: Error, request: Request): void {
  const cohort = String(request.header("x-experiment-cohort") ?? "unknown");
  const costCenter = String(request.header("x-cost-center") ?? "unknown");
  events.push({
    fingerprint: fingerprint(error, request.route?.path ?? request.path),
    name: error.name,
    message: error.message,
    route: request.route?.path ?? request.path,
    cohort,
    costCenter,
    seenAt: new Date().toISOString(),
  });
}

app.use((error: Error, request: Request, response: Response, _next: NextFunction) => {
  record(error, request);
  response.status(500).json({ error: "internal_server_error" });
});

app.get("/internal/exception-groups", (request, response) => {
  const since = Date.parse(String(request.query.since));
  if (!Number.isFinite(since)) {
    response.status(400).json({ error: "since must be an ISO-8601 timestamp" });
    return;
  }

  const recent = events.filter((event) => Date.parse(event.seenAt) >= since);
  const groups = new Map<string, { fingerprint: string; count: number; cohorts: Set<string> }>();
  for (const event of recent) {
    const group = groups.get(event.fingerprint) ?? {
      fingerprint: event.fingerprint,
      count: 0,
      cohorts: new Set<string>(),
    };
    group.count += 1;
    group.cohorts.add(event.cohort);
    groups.set(event.fingerprint, group);
  }

  response.json([...groups.values()].map((group) => ({
    fingerprint: group.fingerprint,
    count: group.count,
    cohorts: [...group.cohorts],
  })));
});

async function poll(): Promise<void> {
  const windowMs = 5 * 60_000;
  const since = new Date(Date.now() - windowMs).toISOString();
  const response = await fetch(
    `http://127.0.0.1:3000/internal/exception-groups?since=${encodeURIComponent(since)}`,
  );
  if (!response.ok) {
    throw new Error(`exception query returned HTTP ${response.status}`);
  }

  const groups = await response.json() as Array<{
    fingerprint: string;
    count: number;
    cohorts: string[];
  }>;
  const bucket = Math.floor(Date.now() / windowMs);
  for (const group of groups) {
    const key = `${group.fingerprint}:${bucket}`;
    if (group.count >= 5 && !notified.has(key)) {
      console.log(`ALERT ${group.fingerprint}: ${group.count} failures`);
      notified.add(key);
    }
  }
}

app.get("/demo/shipments", (_request, _response) => {
  throw new Error("carrier rate service timed out for warehouse 1842");
});

app.listen(3000, () => {
  setInterval(() => void poll().catch(console.error), 60_000);
});
```

There is a deliberate before-and-after in this example. Before grouping, five requests produce five noisy notifications. After normalization, the same failure produces one group with a count and a cohort set. That is enough to start an investigation, while the original events remain available for cost attribution.

The in-memory array is for teaching, not durability. In production, write events to a bounded store, authenticate the internal read endpoint, redact secrets, and make retention explicit. Aggregate counts by event time, not lifetime count: otherwise an old group can trigger immediately after one new event. Test the rule with a burst, a slow trickle, two message variants, and a request from each experiment cohort.

One trap deserves its own paragraph.

I've seen the tempting shortcut: poll every few seconds and alert on every non-200 response. Don't. A busy service turns that into notification load, and a rate limit becomes a second incident. Consider a logistics experiment in which cohort A sends ordinary shipment requests while cohort B exercises a new carrier-rate path. If the carrier path fails five times during one window, the alert should identify the stable failure, show that cohort B is affected, and preserve the cost center for the later comparison. It should not send five pages because five warehouse IDs appeared in five messages. Use a measured interval, inspect response status, retry with backoff where the transport contract allows it, and expose poller failures through the same operational system. A transient failure in the read API must not be mistaken for zero application exceptions, so keep transport errors distinct from an empty result. The poller is part of the alerting path, and its own silence is a signal to monitor; record the last successful poll and the time range it covered so an operator can tell the difference between no failures and no observation.

## Where does this setup stop being suitable?

The catch is detection latency. A worker that polls once per minute cannot promise second-level paging, and a failed notifier needs a durable queue or another delivery guarantee. Choose a push-based incident path when alert delivery time is a hard requirement.

This pattern also does not explain a distributed failure by itself. If an operator needs a span tree, service dependency view, or source-map processing, add the relevant tracing or error-analysis system. Keep the exception group as one signal among them.

It is not suitable when the primary question is "did the job run?" Use a heartbeat for that. It is also a poor fit for unbounded per-tenant labels; keep high-cardinality cohort and cost data in event storage and aggregate it separately. Your mileage may vary on the threshold: five repeats in five minutes is an example rule, not a universal SLO.

The useful boundary is clear: capture failures centrally, group them with stable identity, alert on recent counts, and use metrics, traces, and heartbeats for questions that exceptions cannot answer.

## References

- https://prometheus.io/docs/practices/naming/
- https://opentelemetry.io/docs/concepts/sampling/

## Further reading

- https://prometheus.io/docs/practices/naming/
- https://opentelemetry.io/docs/concepts/sampling/
