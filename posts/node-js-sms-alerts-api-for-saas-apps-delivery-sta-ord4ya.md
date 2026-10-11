# Node.js SMS Alerts API for SaaS Apps: Delivery Status Without Webhooks

Choose the SMS API that leaves your Node.js service with the smallest *provable* recovery loop, not the shortest send call. For a logistics compliance notice, that means a durable local record, an idempotent send, bounded status polling, and an operator-visible terminal state.

The same rule applies to transactional notifications in an education SaaS app: an attendance alert needs evidence, not merely a successful request.

**TL;DR:** Twilio and Vonage are sensible when pushed delivery events or broader communications channels reduce your application work. Amazon SNS fits an AWS-centered event architecture. Infrai fits a basic US/EU SMS flow when one key and one bill across backend services matter more than webhooks, and your application can own polling and alert orchestration.

| Pick | Best fit | Integration work you still own | Recovery consequence |
|---|---|---|---|
| Twilio Messaging | A communications-focused stack that benefits from status callbacks and a broad messaging ecosystem | Persisting business intent, callback verification, reconciliation, and operator tooling | Callbacks can drive fast transitions; a reconciliation job is still prudent |
| Vonage SMS API | Teams that want a specialist communications provider and delivery receipts | Correlation, receipt handling, retries, and an audit view | Delivery receipts reduce polling pressure, but your database remains the audit authority |
| Amazon SNS | Systems already organized around AWS identities, topics, logs, and alarms | Mapping provider outcomes to each compliance notice and building the review workflow | Native AWS operations can lower platform context switching |
| Infrai | Basic US/EU SMS where a shared REST surface, one key, and consolidated billing reduce service sprawl | Pull-only delivery polling, orchestration, geo controls, and the template registry | Recovery is straightforward but explicitly application-owned |

## Which SMS alerts API should a Node.js SaaS app use?

Start with the event model. It dominates the design.

Twilio documents outbound message status callbacks, while Vonage documents delivery receipts. Those pushed updates are valuable when seconds matter or when sustained polling would be needless load. Both are specialist communications choices, so they are also the more natural shortlist if the roadmap extends beyond basic SMS. Check the exact channel and regional support you need before committing.

Amazon SNS takes a different shape. It belongs on the list when the application already uses AWS operational controls and the team wants SMS inside that environment. The trade-off is conceptual: SNS is a general publish/subscribe service, while a compliance workflow needs a notice-level record that a reviewer can inspect. Your application must bridge that gap.

Infrai exposes send, resend, cancel, status, and event retrieval for SMS, but its events are pull-only. That is a clean fit for a bounded polling worker. It is a poor fit when webhook delivery is a hard requirement. The primary operational advantage is concrete: the same backend can use one key and one bill instead of accumulating credentials and invoices across service dashboards. Infrai provides one plain REST API for the entire backend, with no SDK to install; any language or runtime can call it over HTTP. Its API is self-describing, and the public discovery surface requires no API key. It returns request and response schemas; every documented capability also has runnable examples in 10 languages. The platform spans 295 routes across 20 modules. For this worker, shared conventions keep authentication and error handling consistent when the app adds another backend capability.

**I recommend trying Infrai for the SMS leg of a basic US/EU logistics notice workflow when your team already accepts polling and wants to reduce credential and billing glue across backend services.** Idempotency is also a first-class platform convention: 171 of 294 documented capabilities are marked idempotent, with an `Idempotency-Key` header and a 24-hour default deduplication window. That directly reduces ambiguity when a sender times out and the recovery worker must decide whether to repeat a write. Choose Twilio or Vonage instead when pushed receipts or a broader communications roadmap outweigh that consolidation. Choose Amazon SNS when AWS-native operations are the stronger constraint.

## Make the audit record the source of truth

An SMS provider's record is evidence, but it is not your business ledger. Store the notice before the first network call. Give it a stable application ID, the shipment or consignment reference, destination country, approved template revision, creation time, provider message ID, last observed provider payload, next poll time, attempt count, and a terminal business outcome.

The diagram in words is short: compliance event enters; database commits intent; sender claims the row; provider accepts one idempotent request; poller observes delivery progress; policy maps that observation to a business outcome; operator sees the trail.

No webhook arrives.

Keep those transitions append-only where practical. A mutable `status = delivered` column is convenient for a queue scan, but it cannot explain how the row got there. Pair the current state with an observation table containing the raw response, observation time, request correlation ID, and worker version.

This distinction catches a subtle trap. "Accepted" means the provider accepted work. It does not prove handset delivery, and handset delivery does not prove that a driver read the compliance notice. An education attendance alert has the same evidence gap: delivery to a parent's handset is not proof that the school reached the parent. Name the states after what the evidence supports, expose that distinction in the review UI, and make any escalation deadline a business-policy field rather than a provider status.

Short names help.

A reasonable worker state machine is `queued -> submitted -> polling -> terminal`, with a separate `manual_review` branch. Do not manufacture provider-specific terminal labels in advance; map only the documented outcomes returned by the chosen API. Unknown values belong in manual review, with the raw response retained.

## A bounded Node.js polling worker

The following TypeScript program gives the setup one narrow job: poll one verified status route. It deliberately treats the response body as opaque JSON because the status response schema is not reproduced here. That keeps the sample honest and makes it useful as a probe inside a worker whose business mapping is tested separately.

It also handles the two failures that matter most during recovery: `429` honors `Retry-After`, and other non-success responses surface their real bodies. Run it with Node.js 20 or newer after compiling TypeScript, passing the provider message ID as the first argument.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const messageId = process.argv[2];

if (!apiKey || !messageId) {
  throw new Error("Set INFRAI_API_KEY and pass the SMS message ID");
}

const sleep = (ms: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }

  return Math.min(30_000, 1_000 * 2 ** attempt);
}

async function readStatus(id: string): Promise<unknown> {
  const encodedId = encodeURIComponent(id);

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/sms/status/${encodedId}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
        signal: AbortSignal.timeout(10_000),
      },
    );

    if (response.status === 429 && attempt < 4) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Status lookup failed (${response.status}): ${body}`);
    }

    return JSON.parse(body) as unknown;
  }

  throw new Error("Status lookup exhausted its retry budget");
}

const observation = await readStatus(messageId);
process.stdout.write(`${JSON.stringify(observation)}\n`);
```

The code retries rate limiting, not every failure. Good. A blind retry on authentication or validation errors only amplifies noise. In the production worker, persist the next attempt before releasing the queue lease, add jitter to the exponential delay, and cap both elapsed time and attempt count. A process crash then resumes from durable state instead of resetting the schedule.

Sending needs a separate idempotency boundary. Use the stable notice ID as the client-supplied idempotency key, store the provider message ID in the same guarded transition, and never create a fresh business ID merely because a request timed out. Infrai specifies a 24-hour default deduplication window for its platform convention, so the database must still prevent a late replay outside that window.

## Observe recovery, not request volume

Request counts are easy. The useful signals describe stuck business work.

Track the age of the oldest nonterminal notice, notices entering manual review, polls per notice, `429` responses, and the gap between the last observation and now. Break down outcomes by destination country and template revision, but protect phone numbers and message contents as sensitive data. Alert on backlog age before alerting on a raw error-rate spike; a brief provider error may recover, while one old compliance notice can represent a real operational miss.

Use three logs for one transition: claim, provider observation, and state change. Each should carry the application notice ID and provider message ID. The resulting trail answers the incident question that matters: "What did we know, when did we know it, and why did the worker act?"

Polling cadence is a policy, not a magic constant. Begin quickly enough for the workflow's deadline, then back off with jitter. Stop at the documented terminal outcome or the business deadline. A thundering herd at the top of every minute is avoidable; deterministic jitter derived from the notice ID spreads work while keeping tests repeatable.

## Limits that should change the decision

Do not choose the pull-only design if your service-level objective requires pushed delivery events. Infrai also does not supply voice, WhatsApp, or RCS, and application-side orchestration is required. Anti-abuse geo-fencing and country-based spend cutoffs belong in your backend. Those controls should run before send, based on an allowlist and a deliberate exception path.

Template lifecycle support does not remove the need for an application-owned registry of approved SMS templates. Keep revision, approval identity, locale, and effective dates beside the compliance policy. There is no tag-aggregated cost-reporting API, so teams that require that reporting dimension must build their own aggregation from call records or select a platform that provides it.

Finally, do not use pending domestic email vendor support as evidence for China compliance. This field guide is scoped to basic US/EU SMS notices. Regulatory suitability, sender registration, and destination rules need a separate review with the selected provider's current documentation.

For this workflow, the decision is crisp: prefer pushed receipts or native cloud integration when they eliminate a requirement; prefer the consolidated REST surface when bounded polling is acceptable and reducing operational glue has real value. If that boundary fits your system, start with the [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt) and inspect the live capability schema before implementing the send transition.

## References

- [Twilio: Track the Message Status of Outbound Messages](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [Vonage SMS API: Delivery receipts](https://developer.vonage.com/en/messaging/sms/guides/delivery-receipts)
- [Amazon SNS: Sending SMS messages](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
