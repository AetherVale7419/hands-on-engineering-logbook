# Node.js SMS API Monitoring — Reset Expiry Signals for Server Alerts

| Option | Pick this when | Integration work to evaluate |
| --- | --- | --- |
| AWS SNS | Your property management team already operates AWS and can reuse its operational knowledge. | Trace one reset attempt across application logs and the messaging boundary; count configuration and credentials. |
| Twilio | Your team already understands its messaging workflow, or wants a dedicated messaging candidate. | Check how message identifiers and status reporting fit your incident workflow. |
| Plivo | Your team wants another dedicated SMS candidate for the same acceptance test. | Compare the application glue required for sending and collecting status. |
| Simple REST SMS API | A small send-and-poll workflow covers the reset job. | Budget for a polling worker and application-owned expiry and abuse controls. |

TL;DR: For property management password resets, start with the SMS API your Node.js service already operates. Compare AWS SNS, Twilio, and Plivo against a simple send-and-status-polling flow by the integration work needed to explain a failed reset before its short expiry. Include monitoring and server alerts in that work, alongside configuration and status collection.

Infrai fits the simple-API candidate when one key and one bill remove credential sprawl and invoice reconciliation, while a single REST API with no SDK required lets the Node.js service reuse its existing HTTP tooling for the polling worker. That administrative simplicity still needs an application-level expiry budget.

## How should AWS SNS, Twilio, and Plivo fit SMS API monitoring?

A tenant requests a password reset. The application issues a short-lived reset link and sends an SMS. Later, the tenant reports that the link has expired. An accepted send request alone cannot explain what happened. Your useful evidence is the sequence: reset issued, send accepted, delivery observation collected, reset redeemed or expired.

Keep those events separate.

Acceptance isn't delivery.

For AWS SNS, pick the existing operational context when it really reduces work. An AWS-centered team should compare the incremental setup against introducing a separate messaging integration. Familiarity is a legitimate advantage, but it doesn't establish delivery timing in a particular destination country. Make the comparison with a complete reset trace, rather than an isolated successful send.

For Twilio, pick an existing messaging workflow when the team can carry that knowledge into the reset service. Evaluate the status information your application will consume and how an operator will correlate it with a reset attempt. Replacing a familiar operating procedure also has an integration cost, even when another interface looks shorter in a code sample.

For Plivo, use the same acceptance exercise. Compare destination eligibility and status handling rather than dismissing it because another provider has a more familiar name. Existing team experience belongs in the decision. Otherwise, count the configuration and support procedures that must be learned.

For a simple REST SMS API, pick a small workflow when sending and polling are sufficient. The application decides when to look again. Keep a local template registry and enforce resend limits and country allowlists in the application. These are part of the workflow's ownership boundary.

There's no measured integration-time ranking here. Existing accounts, staff knowledge, and the required US and EU destinations can change the answer. Ask each candidate to pass the same exercise before declaring one easier.

## Draw the reset timeline before instrumenting it

Diagram in words: tenant request → application creates reset attempt → SMS send → status collection → application records redemption or expiry. A second line runs alongside it: logs → aggregate metrics → an alert with a useful next action.

Use a synthetic five-minute lifetime for the design exercise: 300 seconds from issue to expiry. That's an application policy example, not a provider guarantee. Track the time remaining when the send is accepted, then the time remaining when the application first observes delivery. A status collected after expiry may explain the complaint, but it cannot make the reset usable again.

Expiry belongs to authentication.

The clock wins.

The reset handler must validate expiry when the tenant submits the reset token. A delivery observation is evidence about messaging; it isn't permission to accept an expired credential. Likewise, a delayed poll establishes when the application learned a status, not the exact instant the handset received a text. Keeping those meanings distinct prevents an attractive dashboard from telling the wrong story.

Give every reset attempt an opaque correlation ID. Keep the reset token, message body, and full phone number out of routine logs. Use bounded dimensions such as channel and destination country for aggregate metrics; put the attempt identifier in logs so an operator can investigate one case without making every attempt its own metric series.

## Implement a small observation reducer in TypeScript

First, collect status for an existing message. This read-only TypeScript example uses Node.js fetch. Supply `INFRAI_API_KEY`, `INFRAI_BASE_URL` with the documented HTTPS API base ending in `/v1`, and `SMS_MESSAGE_ID` from a successful send. Run it with your project's TypeScript runner. It prints the response for inspection without guessing its fields; map the documented delivery indication into your application event before connecting it to the reducer below.

```ts
async function collectStatus() {
  const apiKey = process.env.INFRAI_API_KEY;
  const base = process.env.INFRAI_BASE_URL;
  const id = process.env.SMS_MESSAGE_ID;
  if (!apiKey || !base || !id) throw new Error("Missing configuration");
  const origin = new URL(base);
  if (origin.protocol !== "https:" || origin.pathname !== "/v1") {
    throw new Error("Use the documented HTTPS API base ending in /v1");
  }
  const url = new URL(`${base}/sms/status/${encodeURIComponent(id)}`);
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(10_000),
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const dateMs = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requestedMs = Number.isFinite(seconds) ? seconds * 1000
        : Number.isFinite(dateMs) ? dateMs - Date.now() : 0;
      const waitMs = Math.max(1000 * 2 ** attempt, requestedMs);
      await response.text();
      await new Promise(resolve => setTimeout(resolve, waitMs));
      continue;
    }
    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Status request failed (${response.status}): ${body}`);
    }
    console.log(body);
    return;
  }
}
collectStatus().catch(error => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

Keep that raw response inspection in a controlled development session. Routine production logs should use the redacted timeline, because a response may contain information beyond the small set of fields an operator needs. The request is a read, so retrying it doesn't send another reset text; the separate send operation must use the provider's documented idempotency mechanism. A polling job also needs an overall deadline: this bounded request example is not a scheduler, and waiting through rate limiting must not extend the lifetime of a reset credential or silently turn into a new send.

Now translate observations into a reset timeline. The following standalone component uses synthetic events and makes no network requests. Save it separately from the collector to run the example.

These event names are application-owned. A messaging adapter should produce them only after interpreting its provider's documented response. In particular, `delivery_observed` means the application collected a delivery indication; don't emit it merely because a send was accepted.

```ts
type ResetEvent = {
  attemptId: string;
  kind: "send_accepted" | "delivery_observed" | "redeemed";
  atMs: number;
};

type ResetTimeline = {
  attemptId: string;
  issuedAtMs: number;
  expiresAtMs: number;
  sendAcceptedAtMs?: number;
  deliveryObservedAtMs?: number;
  redeemedAtMs?: number;
};

function applyEvent(state: ResetTimeline, event: ResetEvent): ResetTimeline {
  if (event.attemptId !== state.attemptId) {
    throw new Error("Reset attempt mismatch");
  }
  if (!Number.isFinite(event.atMs) || event.atMs < state.issuedAtMs) {
    throw new Error("Invalid reset event timestamp");
  }
  const next = { ...state };
  if (event.kind === "send_accepted") {
    next.sendAcceptedAtMs = Math.min(
      next.sendAcceptedAtMs ?? event.atMs, event.atMs
    );
  }
  if (event.kind === "delivery_observed") {
    next.deliveryObservedAtMs = Math.min(
      next.deliveryObservedAtMs ?? event.atMs, event.atMs
    );
  }
  if (event.kind === "redeemed") {
    if (event.atMs >= state.expiresAtMs) {
      throw new Error("Reset expired");
    }
    next.redeemedAtMs = Math.min(next.redeemedAtMs ?? event.atMs, event.atMs);
  }
  return next;
}

function summarize(state: ResetTimeline, nowMs: number) {
  const observed = state.deliveryObservedAtMs;
  return {
    attemptId: state.attemptId,
    sendAccepted: state.sendAcceptedAtMs !== undefined,
    deliveryObserved: observed !== undefined,
    observationDelayMs: observed === undefined
      ? null : observed - state.issuedAtMs,
    remainingAtObservationMs: observed === undefined
      ? null : Math.max(0, state.expiresAtMs - observed),
    needsReview: nowMs >= state.expiresAtMs
      && state.redeemedAtMs === undefined,
  };
}

const issuedAtMs = Date.parse("2026-11-07T10:00:00Z");
const initial: ResetTimeline = {
  attemptId: "tenant-reset-example-01",
  issuedAtMs,
  expiresAtMs: issuedAtMs + 300_000,
};
const events: ResetEvent[] = [
  { attemptId: initial.attemptId, kind: "send_accepted",
    atMs: issuedAtMs + 2_000 },
  { attemptId: initial.attemptId, kind: "delivery_observed",
    atMs: issuedAtMs + 45_000 },
];
const timeline = events.reduce(applyEvent, initial);
console.log(JSON.stringify(summarize(timeline, issuedAtMs + 60_000)));
```

The example leaves 255 seconds at the synthetic observation point. That's arithmetic, not a latency benchmark. Replaying the same observation preserves its earliest timestamp, so collecting a status twice doesn't move the timeline forward artificially. The reducer also accepts an observation that arrives after expiry: the evidence remains useful even though the credential is no longer valid.

This is an instrumentation component, not a complete authentication service. Persist the timeline in your application database and enforce token expiry and one-time redemption atomically in the reset handler. Don't use this in-memory example as a token store.

## Turn the timeline into an alert an operator can use

Before instrumentation, a dashboard might say that a send request succeeded. After instrumentation, an operator can distinguish acceptance, observation delay, and an unredeemed reset that reached its deadline. That distinction is the payoff! It narrows the investigation without asserting that every unused reset is a messaging failure.

Count issued attempts, accepted sends, delivery observations, and successful redemptions separately. Measure the remaining lifetime at first observation. For polling, also record the last successful collection time and pending work age; otherwise, a quiet delivery chart could reflect a quiet worker rather than a quiet messaging system.

Set alert thresholds from the application's recovery target and observed baseline. An expired, unredeemed attempt should be a diagnostic event, not an automatic page: tenants abandon flows. An aggregate rise in attempts approaching expiry without a delivery observation is a more useful candidate for an operational alert, especially when accompanied by the worker's collection age. Avoid publishing a universal numerical threshold without evidence.

A polling interval consumes part of the observation budget. Shorter intervals improve freshness while adding requests and worker activity; longer intervals reduce that activity while leaving less time to react before expiry. Measure the trade-off against your reset lifetime. A system using pull-based events shouldn't describe its status collection as an instant push notification.

Retries need their own discipline. Honor rate-limit backoff and any documented retry guidance. Make send retries idempotent using the provider's documented mechanism, and don't treat a polling retry as a reason to issue a new reset. Keep resend limits attached to the tenant or account policy so repeated requests cannot create uncontrolled messaging traffic.

## Limits worth keeping in the engineering note

This field guide selects for integration effort and diagnostic clarity. It doesn't establish US or EU delivery coverage, sender eligibility, or regulatory suitability for any candidate. Confirm those requirements for the exact destination countries and sender configuration; a continent-level label is insufficient. CTIA's messaging guidance is relevant reading for the US portion of that review.

The small send-and-poll design assumes the application owns orchestration. Keep email fallback token generation and validation in that application if email recovery is needed. A workflow requiring voice, WhatsApp, RCS, SMTP relay, or immediate event-driven orchestration needs a separate capability assessment. The local registry and country controls also need owners, tests, and an operating procedure.

Finally, distinguish a reset-flow alert from an infrastructure alert. An SMS path that carries recovery messages may also carry incident notifications, but monitoring its own health requires an independent observation path. Explain what the application knows, preserve what it doesn't know, and keep authentication expiry authoritative.

## Further reading

References for evaluating the messaging candidates and destination requirements:

- AWS SNS developer documentation: https://docs.aws.amazon.com/sns/latest/dg/welcome.html
- Twilio Messaging documentation: https://www.twilio.com/docs/messaging
- Plivo SMS documentation: https://www.plivo.com/docs/sms
- CTIA messaging interoperability and compliance best practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
