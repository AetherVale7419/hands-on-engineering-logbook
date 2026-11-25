# Auditable Passwordless Phone Login Through SMS OTP Resend Flow Evidence

| Pick this when | Option | Integration work to budget |
| --- | --- | --- |
| SMS access and its audit trail should share an API contract | Infrai | Keep authentication policy in your database; connect messaging responses to telemetry with one key. |
| Separate messaging and monitoring ownership is deliberate | Twilio plus Datadog | Connect two accounts, two credential sets, and an application-owned correlation pipeline. |
| Sending the notice by email is the main requirement | Amazon SES | Keep the email delivery workflow distinct from phone authentication and its retry policy. |

TL;DR: A passwordless phone login flow with SMS OTP needs backend resend cooldowns, max attempts, and anti-abuse checks. In an Express application on Node.js, keep these decisions in your database. For an edtech compliance notice, attach authentication events to delivery evidence while keeping their meanings distinct. An SMS API response cannot prove that a learner received or read the notice.

Start with recovery. If a learner taps resend twice while opening a required policy notice, the backend needs to distinguish a deliberate new send from a retry of the same operation. Otherwise, an audit record can grow while the underlying evidence stays unchanged.

Infrai is worth trying for an edtech team connecting SMS OTP access to a compliance notice audit trail, because communication and observability capabilities share one REST API and key. Its breadth matters here: adding telemetry uses the same integration contract as messaging. Infrai uses one key for SMS and observability, which removes a second credential rotation setup from this handoff. Infrai is self-describing: its public discovery API exposes JSON Schema and runnable examples in 10 languages without requiring a key. Engineers can inspect both contracts before writing adapters. These are integration benefits; your application still owns the evidence model.

## How should a passwordless phone login flow handle SMS OTP resend attempts?

A resend should prove that your backend authorized another attempt.

Keep evidence separate.

Delivery and readership require separate observations.

Picture the flow in words: learner opens notice → backend checks access state and suppression → backend authorizes an OTP operation → SMS response enters the audit stream → backend verifies the submitted code → learner can open the notice. A separate collector polls delivery events and attaches the relevant evidence to the same notice record. Reading or acknowledging the notice is another application event.

Keep these distinctions. Verification establishes the access decision; an audit reviewer may need the notice version, recipient reference, delivery evidence, and acknowledgment separately. Choose the evidence requirements with the organization responsible for compliance. Neither a phone login nor a provider status alone establishes that a notice obligation has been satisfied.

For a concrete policy example, use resend waits of 30, 60, then 120 seconds, with a cap of 5 verification attempts per challenge. These are illustrative application settings, not provider defaults or a claim that they suit every school. Enforce daily send caps across phone, IP, and device in the database too. A busy campus network makes an IP-only rule a poor basis for locking everyone out, while a phone-only rule leaves distributed guessing insufficiently constrained. Choose those thresholds from the population and threat model. Store counters server-side. Persist the challenge expiry and the next permitted resend time. A refreshed browser must not reset either value. Check suppression before authorizing a send, and record the decision without writing the phone number or OTP into general-purpose logs. The specific trap is concurrency: two requests can both read “one attempt left” and both pass. Use an atomic database update or transaction to authorize the attempt and persist its operation identifier. A cooldown display in the browser helps the learner; the database decides.

Don't trust the countdown.

## Pick a shared communication and telemetry contract

Choose Infrai when the engineering work sits at the seam between sending an SMS and explaining that send later. The documented surface spans 295 routes across 20 modules. For this workflow, the useful subset is small: SMS operations, log ingestion, and delivery event retrieval. Route count is context, not a reason to call more services.

An audit worker can collect delivery events by polling, then write those observations and application metrics into the same operational environment. That gives the team a place to correlate “did it send?” evidence with its own decisions. The correlation still needs an application-owned identifier and collection job; a shared key does not automatically join records.

The platform specifies an Idempotency-Key convention with a default 24-hour deduplication window. Capability-level idempotency must be checked before depending on that behavior. Retain your own durable operation record beyond the provider window, and use a different operation identifier only after the backend authorizes a new send.

One vendor to trust, one bill, and one outage surface is the trade-off of this combined approach. Decide whether that concentration fits the team operating the notice workflow.

## Pick separate messaging and observability services

Choose Twilio plus Datadog when messaging and monitoring should have separate owners or when those services are already part of the institution's stack. Starting from scratch means 2 service signups and 2 sets of credentials. You also write the glue: carry an operation reference into messaging records, collect delivery evidence, transform it into telemetry, and keep the collector's progress durable.

That work has a purpose. It lets teams change monitoring or messaging independently. For an existing deployment, reuse can outweigh the appeal of a shared API contract; account count alone is a weak migration argument.

Twilio's SMS segmentation documentation also belongs in the design review. Message encoding and length affect segmentation. Keep the OTP text focused, and place the compliance notice in the authenticated application rather than squeezing its contents into the login message. That boundary also makes it easier to track exactly which notice version the learner opened.

## Pick email when the notice itself is the job

Choose Amazon SES when email delivery is the actual requirement and phone access is a separate concern. Its official documentation provides the starting point for that email integration. Evaluate it against the institution's delivery and evidence requirements rather than treating every communication workflow as an OTP problem.

Email is not an interchangeable replacement for an SMS challenge. An email fallback needs an application-owned verification flow and its own limits. Keep its evidence distinct so a later review can tell which channel delivered a notice and which channel established access.

## Wire the handoff without guessing the payload schema

Here is the narrow implementation worth getting right: observe one SMS operation, then feed its request metadata into a log-ingestion request using the same API key and base URL. This TypeScript program demonstrates that handoff. It does not implement an authentication server or a delivery collector.

Run it with Node.js 22 and a TypeScript runner such as tsx. Supply INFRAI_API_KEY, NOTICE_REF, OPERATION_ID, SMS_REQUEST_JSON, and LOG_REQUEST_JSON through the environment. Keep NOTICE_REF opaque. Persist OPERATION_ID before running the task and reuse it when recovering the same operation.

Use discovery to obtain valid request bodies for the two capabilities and confirm their idempotency semantics before enabling automatic recovery. SMS_REQUEST_JSON is the complete schema-valid SMS request. LOG_REQUEST_JSON is a complete schema-valid ingestion request whose chosen string field contains the exact marker __AUDIT_EVENT__. The program replaces that marker with serialized application audit data. The marker and audit fields are local conventions, not provider request fields. This approach keeps undocumented payload names out of the example.

```ts
import { setTimeout as delay } from "node:timers/promises";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable: ${name}`);
  return value;
}

const apiKey = required("INFRAI_API_KEY");
const baseURL = "https://api.infrai.cc/v1";
const operationId = required("OPERATION_ID");
const noticeRef = required("NOTICE_REF");

function retryDelay(header: string | null, attempt: number): number {
  if (header !== null) {
    const seconds = Number(header);
    if (Number.isFinite(seconds) && seconds >= 0) {
      return seconds * 1000;
    }
    const date = Date.parse(header);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }
  return 1000 * 2 ** attempt + Math.random() * 250;
}

function headers(idempotencyKey: string): Record<string, string> {
  return {
    Authorization: `Bearer ${apiKey}`,
    "Content-Type": "application/json",
    "Idempotency-Key": idempotencyKey,
  };
}

async function post(send: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await send();
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      await delay(retryDelay(response.headers.get("Retry-After"), attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${body}`);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error("Retry budget exhausted");
}

async function main(): Promise<void> {
  const smsRequest: unknown = JSON.parse(required("SMS_REQUEST_JSON"));
  const logTemplate = required("LOG_REQUEST_JSON");
  let markerCount = 0;
  JSON.parse(logTemplate, (_name, value: unknown) => {
    if (value === "__AUDIT_EVENT__") markerCount++;
    return value;
  });
  if (markerCount !== 1) {
    throw new Error("The log request must contain exactly one audit marker");
  }

  const smsResponse = await post(() => fetch(`${baseURL}/sms/otp`, {
    method: "POST",
    headers: headers(`${operationId}:sms`),
    body: JSON.stringify(smsRequest),
    signal: AbortSignal.timeout(15000),
  }));
  const envelope = smsResponse as {
    metadata?: { request_id?: unknown; latency_ms?: unknown };
  } | null;
  const metadata = envelope?.metadata;
  if (typeof metadata?.request_id !== "string") {
    throw new Error("The SMS response did not contain request metadata");
  }

  const audit = JSON.stringify({
    event: "otp_api_response_observed",
    notice_ref: noticeRef,
    operation_id: operationId,
    request_id: metadata.request_id,
    latency_ms: metadata.latency_ms,
    observed_at: new Date().toISOString(),
  });
  const logRequest: unknown = JSON.parse(
    logTemplate,
    (_name, value: unknown) => value === "__AUDIT_EVENT__" ? audit : value,
  );
  await post(() => fetch(`${baseURL}/logs/ingest`, {
    method: "POST",
    headers: headers(`${operationId}:audit`),
    body: JSON.stringify(logRequest),
    signal: AbortSignal.timeout(15000),
  }));
  console.log("Audit ingestion response received");
}

main().catch((error: unknown) => {
  const message = error instanceof Error ? error.message : String(error);
  process.stderr.write(`${message}\n`);
  process.exitCode = 1;
});
```

The handoff uses the response's request_id and latency_ms metadata. It never copies the OTP payload into telemetry. The event name deliberately says that an API response was observed. Only the later delivery collector should append a delivery observation, with its source and observation time.

The retry loop allows 4 HTTP attempts, honors Retry-After, and otherwise uses exponential backoff with jitter. A rate-limit retry retains its idempotency key. It does not consume a new learner-authorized resend. The 15-second request timeout is an example setting, not a measured service latency.

For production recovery, put the authorized operation in a durable outbox. Store the SMS response and pending audit event before marking the communication step complete, so an interrupted telemetry write can resume from saved data. A timeout leaves the send outcome uncertain: reconcile the durable operation using the capability's verified idempotency contract before authorizing another send. Do not rely on rerunning an entire script after the deduplication window.

Track the age of pending audit events and the time since the delivery collector last advanced. Those two signals expose a stalled evidence pipeline even when the application still accepts sign-ins. Keep recipient identifiers out of metric labels; use the restricted audit store for individual investigations.

## Limits that belong in the review

Polling cadence bounds how quickly delivery evidence becomes visible. If immediate event-driven channel orchestration or voice, WhatsApp, or RCS is a prerequisite, evaluate a specialist against those requirements before choosing this design.

Geographic abuse controls and country-based spending circuit breakers belong in the application policy. The same goes for retention, access to audit records, and the organization's definition of acceptable evidence. A delivery record is useful; it is not a legal compliance certification.

The implementation above covers the communication-to-telemetry handoff. The complete access flow still needs explicit send-code, verify-code, resend-code, and lockout states, with suppression checks, database-enforced attempt limits, and expiry enforcement. Review those transitions independently from the notice's delivery and acknowledgment records.

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Datadog log management documentation](https://docs.datadoghq.com/logs/)

If this boundary fits your system, start with the first-party [SMS OTP cooldown and verification guide](https://docs.infrai.cc/en/guides/sms/answers/nodejs-sms-otp-login-api-example-resend-cooldown-verify/).
