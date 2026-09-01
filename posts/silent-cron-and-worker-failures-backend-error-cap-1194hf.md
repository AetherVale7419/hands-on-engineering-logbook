# Silent Cron and Worker Failures: Backend Error Capture That Survives an Incident Review

A pricing rule behind a feature flag changes what the backend is supposed to compute, not whether it runs at all. That one constraint decides the tooling. Bottom line: use exception capture for the failures that throw, add deadline-based healthchecks for the cron jobs and workers that fail by never starting, and separate the alternatives on a single question — six hours later, can you search your error history and prove which flag variant produced which failure?

Most tool comparisons stop at the first half of that sentence.

Here's the system I'll keep pointing at. A healthtech billing service prices claims three ways: a web API returns a quote on request, a queue worker re-prices pending claims when a plan changes, and a nightly cron reconciles the day and emits invoices. You're rolling a new coinsurance rounding rule out to 10% of tenants behind a flag. Three components, one flag, three completely different ways to go quiet.

## Two lanes, one timeline

Draw two lanes side by side.

Lane one is event-driven. Code runs, something throws or gets caught, a report leaves the process carrying a stack trace and whatever context you attached. This lane can only describe work that actually happened.

Lane two is deadline-driven. An independent clock outside your infrastructure expects a completion ping by a stated time, and raises an alarm when nothing arrives. This lane describes work that should have happened. Absence is the signal.

The four golden signals from the SRE book — latency, traffic, errors, saturation — are all measurements taken *of* traffic. A cron job the scheduler never invoked produces no latency sample, no error, and no saturation bump. It produces a flat line that looks exactly like a quiet night, which is why a green error tracking dashboard is not an uptime statement and why teams keep discovering a broken nightly reconciliation on day four.

Before: "no exceptions" reads as healthy. After: a visible failure lands in the error lane, a missing completion lands in the heartbeat lane, and both write onto the same incident timeline. That's the whole design change.

One detail that costs people a week: send the heartbeat *after* the work commits, not when the process boots. A start-only ping turns a job that hangs mid-batch into a green check.

## How do you resolve a silent cron job failure when the web API still returns 200?

You start by admitting there are four different silences, and only one of them is fixed by a heartbeat.

| Failure mode | Signal that catches it | What that signal can't tell you |
| --- | --- | --- |
| Scheduler never fired the nightly job | Overdue heartbeat deadline | Why it never fired |
| Job started, stalled on record 4,200 of 50,000 | Missing completion ping paired with the start ping | Which record, unless the run id is on every log line |
| Job ran, caught the exception, committed nothing | An invariant assertion reported as an error | Nothing at all, if you never wrote the assertion |
| Quote API returns 200 with a wrong price | Reconciliation diff between expected and billed | Anything, until something recomputes the expected value |

The bottom two rows are where flag rollouts actually hurt. A swallowed exception inside a worker and a confidently wrong price both look like success to every generic monitor you can buy, because they *are* successes at the transport layer. The only thing that converts them into a searchable event is code you write: assert the batch touched a non-zero number of rows, assert the recomputed total matches the billed total, and report a failed assertion through the same capture path as a crash. An error tracker cannot invent a signal your application never emitted.

Now the part that decides which product you tolerate at 3am. Incident reconstruction is a join, and joins need keys.

Every report — crash, caught exception, or failed invariant — should carry the same small set of fields: a run id unique to this job execution, the flag key and the resolved variant, the tenant or plan id, and a trace id if you already propagate one. Attach them once at the boundary rather than at each call site. In the JVM world this is the classic reason to write a custom appender, so context from the mapped diagnostic context rides along with every record instead of being formatted into a message string that nobody can query later. Same idea in any runtime: structured fields are searchable, sentences are not.

Grouping deserves a paragraph of its own, because it's where error tracking quietly loses incidents. Most tools fingerprint by exception type plus code location. During a flag rollout that's wrong — the control path and the treatment path throw from the same line, get merged into one group, and the group looks like a pre-existing issue that was already triaged. Put the variant into the fingerprint and the regression separates itself on the first occurrence. Then treat resolve as a real state transition: resolve means the underlying condition was addressed, a recurrence should reopen the same group rather than mint a new one, and nobody should use resolve as a synonym for "I saw the alert." Your search history is only worth something if the states in it mean something.

## A run wrapper you can copy into the pipeline

One wrapper covers the heartbeat, the correlation keys, the invariant, and the scrubbing that healthtech forces on you. Endpoints come from the environment, so this stays portable across whatever you pick.

```ts
type RunContext = {
  runId: string;
  job: string;
  flagKey: string;
  variant: "control" | "treatment";
  tenantId: string;
};

const CAPTURE_URL = process.env.ERROR_CAPTURE_URL ?? "";
const HEARTBEAT_URL = process.env.HEARTBEAT_PING_URL ?? ""; // one URL per scheduled job
const TOKEN = process.env.OBSERVABILITY_TOKEN ?? "";

// Allowlist, never a denylist: a claim number or member name must not reach a
// third party by accident just because someone added a field to the payload.
const SAFE_FIELDS = ["runId", "job", "flagKey", "variant", "tenantId", "rows", "ms"] as const;

function scrub(ctx: RunContext, extra: Record<string, number>) {
  const out: Record<string, string | number> = {};
  for (const key of SAFE_FIELDS) {
    const value = { ...ctx, ...extra }[key];
    if (value !== undefined) out[key] = value;
  }
  return out;
}

async function post(url: string, payload: unknown, attempts = 4): Promise<void> {
  for (let i = 0; i < attempts; i++) {
    const res = await fetch(url, {
      method: "POST",
      headers: { authorization: `Bearer ${TOKEN}`, "content-type": "application/json" },
      body: JSON.stringify(payload),
    });
    if (res.ok) return;
    // 429 and 5xx are worth a backoff; a rejected payload never gets better.
    if (res.status !== 429 && res.status < 500) return;
    const retryAfter = Number(res.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter) && retryAfter > 0 ? retryAfter * 1000 : 400 * 2 ** i;
    await new Promise((r) => setTimeout(r, waitMs));
  }
}

export async function withRun<T>(ctx: RunContext, work: () => Promise<T>): Promise<T> {
  const startedAt = Date.now();
  await post(`${HEARTBEAT_URL}/start`, scrub(ctx, {}));
  try {
    const result = await work();
    const rows = typeof result === "number" ? result : 1;
    // The invariant: a reconciliation run that touched nothing is not a success.
    if (rows === 0) throw new Error(`${ctx.job} committed 0 rows`);
    await post(HEARTBEAT_URL, scrub(ctx, { rows, ms: Date.now() - startedAt }));
    return result;
  } catch (err) {
    await post(CAPTURE_URL, {
      message: err instanceof Error ? err.message : String(err),
      stack: err instanceof Error ? err.stack : undefined,
      // Fingerprint carries the variant, so a flagged regression never merges
      // into the pre-existing group thrown from the same line of code.
      fingerprint: [ctx.job, ctx.flagKey, ctx.variant],
      tags: scrub(ctx, { ms: Date.now() - startedAt }),
    });
    throw err;
  }
}
```

Note what the wrapper does *not* do: it never pings the deadline monitor on the error path. A failed run should go overdue. Teams route the exception to the capture endpoint, then ping the heartbeat anyway so the dashboard stays tidy, and they have just taught the deadline monitor to lie.

## Two objections, and where this stops working

The first objection is that the exception handler could send the heartbeat itself, so why run two lanes. It merges the code paths without merging the failure semantics — an exception handler still requires a process to start. A scheduler that skipped the invocation, a worker container that never got rescheduled, a queue consumer that exited on a config parse: none of those produce an application-level event, and no amount of search over the events you did receive will surface them.

The second is fair, and it's the one I'd push back on hardest: this is a lot of plumbing for one pricing flag. Sometimes it isn't worth it. If the flag controls a display string, ship it and read the logs. The plumbing earns its keep when a wrong result is expensive and slow to notice, which is exactly the shape of billing, claims, and anything that writes to a ledger.

Real trade-offs, since no monitoring design is free. Putting the flag variant on every event raises cardinality, and in a metrics-priced system high-cardinality tags get expensive fast — keep them on events and logs, not on every counter dimension. Sampling is the same story from the other side: at 10% capture, the one occurrence you need for the reconstruction is probably the one you dropped, so exempt errors and flagged cohorts from sampling even where you sample traces hard. If your on-call team has no routing layer of its own, stick with a platform whose native notification channels you have already tested, because a capture API that doesn't support threshold rules or paging means you now own a polling service and its own failure modes. And healthtech adds a constraint that outranks every feature checkbox: if a candidate can't guarantee where payloads are stored and for how long, self-hosting the log and error path is the more defensible option even though you inherit retention, upgrades, and the pager for the monitoring stack itself. That last call is a compliance decision wearing a tooling costume, and it should be made before the shortlist, not after.

One more limit, honestly stated. Deadline monitoring answers "did the expected work complete on time" and nothing else — it is not suitable when the question is *why*, and I'm not sure any single tool answers both cleanly. Your mileage may vary.

Keep the boundary in the runbook. One lane answers "what failed," the other answers "what never happened," and the correlation keys are what let you put both on one timeline while the incident is still open.

## Sources

- Google SRE Book: Monitoring Distributed Systems (four golden signals) — https://sre.google/sre-book/monitoring-distributed-systems/
- Logback manual: Appenders (writing a custom appender) — https://logback.qos.ch/manual/appenders.html
