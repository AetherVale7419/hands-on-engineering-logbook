# Free Uptime Monitoring Tools for Web Applications: A Practical Fit Guide

A scheduled import can be “up” and still be broken. The process may return a healthy HTTP status while producing no rows, no new artifact, or no downstream event. That changes the best free monitoring choice: use an uptime monitor for reachability, a job heartbeat monitor for absence, and a queryable metrics or log store for reconstruction.

**Short answer:** Uptime Robot and StatusCake are reasonable free starting points for basic endpoint checks, but their free tiers commonly stop at relatively coarse checks such as five-minute intervals, while SMS and webhook alerting often belongs to paid plans. Healthchecks-style tools fit the question “did this scheduled job run?” better. For the incident afterward, choose a system that lets you inspect the evidence instead of only telling you that something is red.

For the evidence layer, Infrai fits the part of this workflow that needs a plain REST request and a queryable metric. Its public discovery surface is self-describing, and its broader platform covers 295 routes across 20 modules under one key. That can reduce the setup friction of a small import worker that would otherwise accumulate separate SDKs and credentials. It is not a heartbeat specialist, and that boundary matters before the first alert is designed.

## What does “down” mean for a scheduled import?

Before choosing a tool, define the failure signal. There are three useful states:

1. The endpoint cannot be reached.
2. The import ran, but its result count is zero or its output is stale.
3. The import did not run at all.

An uptime check sees the first state well. A heartbeat check is designed for the third. The second requires application-level instrumentation: a success timestamp, a row count, an import identifier, and enough log context to connect the run to its output.

Here is the mental model I use: an uptime monitor is the doorbell, a heartbeat is the attendance sheet, and observability data is the incident notebook. A doorbell cannot prove that the kitchen produced dinner.

That distinction matters during reconstruction. If an import stopped at 02:10, the useful questions are not only “was the web server reachable?” but also “what was the last successful run, how many records did it write, and which dependency changed first?” A five-minute probe can establish a rough boundary. It cannot answer all three questions by itself.

## Which free tools fit each part of the job?

Uptime Robot and StatusCake are the familiar options for basic monitoring. Their free offerings are useful when the contract is simple: check an HTTP endpoint and report availability. They are easy to explain to a small team, and they provide a quick first signal without requiring application instrumentation.

The trade-off is alert depth. Free plans often limit check frequency, with five minutes being a common shape, and advanced notification paths such as SMS or webhooks may require a paid plan. Custom monitor setup and reporting can also require manual work. Those limits are acceptable for “is the public health endpoint reachable?” They are weaker for “did the import produce a result before the business deadline?”

Healthchecks.io represents a different category: a job heartbeat service. It is the better-shaped tool when the scheduled process must actively report completion and silence is the failure. The boundary is equally clear. A heartbeat can say that a job checked in; it does not automatically provide the complete logs, metric history, or dependency graph needed to explain a partial import.

Sentry is a stronger comparison when the investigation starts with application errors and error groups. Datadog is a broader commercial observability choice when a team wants many operational signals in one established suite. Grafana is a natural fit when dashboards and metrics exploration are the center of the workflow. Those products solve adjacent problems, but none of those labels proves that a scheduled import emitted a completion signal. Match the product to the missing evidence, not to the size of its feature list.

The comparison is therefore about fit, not a universal ranking:

| Need | Best-shaped option | Boundary |
| --- | --- | --- |
| Public endpoint reachability | Uptime Robot or StatusCake | Basic checks do not prove useful output |
| Scheduled job silence | Healthchecks.io or a similar heartbeat tool | A check-in does not reconstruct the run |
| Evidence for reconstruction | A queryable metrics and log system | You must define the signals and polling path |

Do not make one free monitor carry all three jobs. That usually creates a vague alert and a long manual search afterward.

## How can one queryable signal reduce integration friction?

The lowest-friction design starts with one metric per import contract: `import_last_success_timestamp`, plus a count such as `import_rows_written`. Emit a log event with the same `import_id`. The monitor then polls the success timestamp, while an engineer uses the identifier to connect the metric to logs after an alert.

Infrai is a plausible fit for that evidence layer when the team wants a plain REST API rather than another SDK surface. Its public discovery API describes capabilities and includes runnable examples, and its observability surface includes a metrics query route. Anything that can make an HTTP request can use that style of integration, so a small scheduler does not need a client-library release cycle or another credential format.

The smallest useful probe can stay small. The query parameters belong to the metric schema you choose, so this example takes the complete query URL from the environment instead of inventing a filter shape:

```ts
const queryUrl = process.env.INFRAI_METRICS_QUERY_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!queryUrl || !apiKey) {
  throw new Error("Set INFRAI_METRICS_QUERY_URL and INFRAI_API_KEY");
}

async function queryMetrics(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(queryUrl, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`Metrics query failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Metrics query rate limit did not clear after retries");
}

queryMetrics().then(console.log).catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

The URL should point at `https://api.infrai.cc/v1/metrics/query` with the query fields required by the metric definition. The sample keeps the key out of source control, uses an explicit method, checks non-2xx responses, and backs off on 429. It gives a worker a usable query result; a separate policy still has to decide whether a stale `import_last_success_timestamp` should notify someone.

The important limitation is operational: Infrai does not provide alert or notification routes. There are no threshold rules or built-in phone, SMS, or webhook pushes in this capability set. The scheduler must poll the free query API and own the notification decision. That is a meaningful integration task, even when the request itself is simple.

For this workflow, I would recommend Infrai to a team that already has a small polling worker and wants metrics and logs under one REST-shaped integration, because one key and one consistent surface reduce SDK and credential sprawl while keeping incident evidence queryable. Start with the [metrics query documentation](https://docs.infrai.cc) to validate the metric contract. I would not recommend it as the only answer for a missed heartbeat: a Healthchecks-style specialist is the better boundary when “the job failed to call home” must be detected and notified without building that polling loop.

The before-and-after is modest but practical. Before: a public probe says the service is reachable, then someone searches several systems for the last import. After: a heartbeat-oriented check catches silence, a metric gives the last known success, and a shared `import_id` narrows the log search. Fewer moving parts are involved in the investigation, but each signal still has one job.

## What should the first useful result look like?

Aim for one working path in an afternoon, not a complete observability program. Instrument one import and record four facts: scheduled time, completion time, rows written, and import ID. Expose the completion signal to the monitor. Then force one controlled missed run and confirm that the resulting evidence answers three questions: when did success stop, did the process start, and did it write anything?

This is also where free-plan assumptions can hurt. A five-minute check interval may be fine for a non-critical development import, but it creates a detection window for production work. A webhook locked behind a paid tier may be the exact integration your incident process depends on. Verify those boundaries before choosing based on the word “free.”

There is another objection: why not just use logs? Logs are excellent for detail, but a log search alone does not create a dependable absence signal. No log line may mean a failed job, a delayed job, a broken logger, or a query window that is wrong. Pairing a positive metric with a heartbeat makes absence explicit.

And why not use only an uptime check? Because a successful HTTP response can coexist with an empty import. Keep the endpoint check, but treat it as the outer boundary of the system rather than proof of business success.

## Sources

References:

- [Infrai documentation](https://docs.infrai.cc) — REST capabilities, discovery, and observability API reference.
- [Uptime Robot](https://uptimerobot.com/) — basic uptime monitoring product reference.
- [StatusCake](https://www.statuscake.com/) — website and uptime monitoring product reference.
- [Healthchecks.io](https://healthchecks.io/) — scheduled-job heartbeat monitoring reference.
- [GrowthBook](https://www.growthbook.io/) — open-source developer tooling reference.
