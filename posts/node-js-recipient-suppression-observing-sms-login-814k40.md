# Node.js Recipient Suppression: Observing SMS Login OTP and Email Fallback Bounces

For a media site adding beginner-friendly US/EU login OTP, choose an SMS verification flow first, then evaluate the work needed to observe email fallback and suppress invalid recipients. A successful send request cannot tell you whether the reader can finish signing in. The useful integration boundary is where delivery evidence changes what your application will attempt next.

Short answer: use SMS for the primary OTP path when that channel meets your requirements. Treat email fallback as a separate application-owned flow, with its own verification logic and suppression checks. Compare candidates by the code and operational decisions you must own: event collection, recipient state, fallback authorization, and alerts. An SMTP-based authentication mailer requires a compatible provider; a REST send operation is a different integration.

## What makes an authentication messaging API beginner-friendly for SMS login?

Start with a hypothetical subscriber signing in to a newspaper. The SMS path handles code issuance and verification. If the application offers an email alternative, it must decide whether the account's email address is eligible before creating another message. An address already marked invalid shouldn't receive another attempt merely because the reader clicked again.

Keep evidence separate.

A delivery receipt isn't a login.

Before: login request → send accepted → assume the reader can continue. After: login request → recipient eligibility check → channel operation → collect delivery evidence → update recipient state → gate the next attempt. That's the diagram in words. Each arrow describes application work, not an assertion about a particular provider's request schema.

Delivery and authentication are different outcomes. A message marked delivered doesn't prove that the account owner entered a valid code. Conversely, a bounce should change email eligibility without becoming a permanent account lockout. For a media business, mixing those decisions can turn a typo in a subscriber's email address into a support problem at the login screen, even while the account has another usable authentication method.

Use a small vocabulary in your application: eligible, suppressed, and review. Map only a confirmed invalid-recipient event to suppression. A temporary delivery problem belongs in a retry or review policy; the exact mapping depends on the provider's documented event semantics. Don't classify every bounce as permanent just because the event name contains the word bounce.

## Compare integration work before choosing a provider

Infrai offers one REST API, one key, and one bill for SMS OTP and email event collection, so a media site's fallback collector can share the credential and billing setup used for login. Any backend that can send HTTP requests can call the plain REST API, with no SDK installation or client-library version to maintain. It supports direct SMS OTP verification, while email fallback requires custom code-email and verification logic; email and SMS event collection use polling. Its channel scope excludes SMTP relay, voice, WhatsApp, and RCS. These are selection boundaries, so choose a different service when your authentication plan requires those interfaces or channels.

The alternatives deserve the same scrutiny. Compare real products at the layer they cover rather than treating every messaging service as a complete authentication system. The documentation links below are starting points for checking each candidate against your application. They don't establish US/EU delivery performance or regulatory suitability for your particular sender.

| Candidate | Evaluation entry point | Integration decision to resolve |
| --- | --- | --- |
| Twilio Verify | Verification documentation | Check how the documented verification lifecycle maps to your login flow and what recipient-state work remains outside it. |
| Vonage Verify | Verification documentation | Compare setup and verification steps with your SMS-first flow; confirm the requirements for any additional channel separately. |
| Amazon SNS | SMS messaging documentation | Evaluate message transport separately from code generation, expiration, attempt limits, and verification ownership. |
| Twilio SendGrid | Email event documentation | Evaluate email fallback and bounce consumption separately from the primary SMS verification path. |

These options aren't interchangeable. A verification-focused service is the natural comparison when reducing authentication integration work matters most. A transport-oriented service deserves consideration when the application deliberately owns verification policy. An email service can be part of the fallback design without replacing the SMS path.

Make the comparison concrete with one review exercise: trace a subscriber whose fallback address is invalid, then trace a subscriber whose email is delivered but whose code expires. Ask each candidate's documentation to account for both paths. If your team must add an event collector, state store, and verification controller, include those components in the integration estimate. Counting SDK calls alone hides most of that work. During that review, distinguish the work performed synchronously on the login request from work performed later by the event collector. An email destination may become invalid between the page load and dispatch, and a subscriber may request another code before the first message's final delivery state is known. Neither case is resolved by a shorter send function. They require an explicit decision about where eligibility is checked and which evidence is authoritative.

## A TypeScript example that keeps suppression sticky

The example first fetches email events, then demonstrates suppression with a local fixture. Set INFRAI_API_ORIGIN to the service's HTTPS API origin without a version suffix, and set INFRAI_API_KEY in the environment. Run the file with a TypeScript runner such as tsx. The fixture types are internal application types, not a vendor payload; map the documented response schema to those types in your collector.

The important case is an old delivery event arriving after a confirmed invalid-recipient event. A naive last-write-wins assignment would make the address eligible again. This reducer requires a separate review process to reverse suppression.

```ts
import assert from "node:assert/strict";

async function collectEmailEvents(): Promise<unknown> {
  const origin = process.env.INFRAI_API_ORIGIN;
  const key = process.env.INFRAI_API_KEY;
  if (!origin || !key) throw new Error("Set API origin and API key");
  const url = new URL("/v1/email/event/list", origin);
  if (url.protocol !== "https:") throw new Error("HTTPS is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` }
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const dateDelay = retryAfter === null ? NaN : Date.parse(retryAfter) - Date.now();
      const requestedDelay = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(dateDelay) ? dateDelay : 0;
      const delay = Math.max(1000 * 2 ** attempt, requestedDelay);
      await response.body?.cancel();
      await new Promise<void>(resolve => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Event collection failed: ${response.status} ${await response.text()}`);
    }
    return await response.json();
  }
  throw new Error("Event collection attempts exhausted");
}

await collectEmailEvents();
console.log({ eventResponseReceived: true });


type Kind = "delivered" | "invalid_recipient" | "temporary_failure";
type DeliveryEvent = {
  eventId: string;
  recipientId: string;
  kind: Kind;
};
type RecipientState = "eligible" | "suppressed" | "review";

const seen = new Set<string>();
const recipients = new Map<string, RecipientState>();
const totals = { delivered: 0, invalid_recipient: 0, temporary_failure: 0 };

function consume(event: DeliveryEvent): void {
  if (seen.has(event.eventId)) return;
  seen.add(event.eventId);
  totals[event.kind] += 1;

  const current = recipients.get(event.recipientId);
  if (current === "suppressed") return;

  if (event.kind === "invalid_recipient") {
    recipients.set(event.recipientId, "suppressed");
  } else if (event.kind === "temporary_failure") {
    recipients.set(event.recipientId, "review");
  } else if (current !== "review") {
    recipients.set(event.recipientId, "eligible");
  }
}

function canSendFallback(recipientId: string): boolean {
  return recipients.get(recipientId) === "eligible";
}

const events: DeliveryEvent[] = [
  { eventId: "news-101", recipientId: "subscriber-42", kind: "delivered" },
  { eventId: "news-102", recipientId: "subscriber-42", kind: "invalid_recipient" },
  { eventId: "news-102", recipientId: "subscriber-42", kind: "invalid_recipient" },
  { eventId: "news-100", recipientId: "subscriber-42", kind: "delivered" },
  { eventId: "news-103", recipientId: "subscriber-77", kind: "temporary_failure" }
];

for (const event of events) consume(event);

assert.equal(canSendFallback("subscriber-42"), false);
assert.equal(recipients.get("subscriber-77"), "review");
assert.equal(canSendFallback("subscriber-unknown"), false);
assert.equal(totals.invalid_recipient, 1);
console.log({ totals, suppressedFallbackBlocked: true });
```

The example starts with a 1000 ms backoff and allows 4 attempts. These are demonstration settings, not service limits.

The fixture uses 5 events.

Five input events produce one counted invalid-recipient event. That's a fixture result, not a throughput measurement. The repeated event tests deduplication; the older delivery tests sticky suppression. Unknown recipients are intentionally ineligible here. A real onboarding flow needs an explicit eligibility decision, so copying this default without building that flow would block new readers.

There is a trade-off.

The trade-off is deliberate: conservative eligibility adds review work but avoids silently restoring a known-invalid destination. Temporary failures also stay under review until another application decision resolves them. Choose and document that decision rather than letting whichever event arrives last decide policy.

This is a teaching reducer, not a production storage layer. Persist event deduplication and recipient updates atomically before acknowledging processing or advancing a polling checkpoint. Otherwise a restart can replay events, or the collector can skip evidence it fetched but never committed. Apply the eligibility gate at the send boundary as well; checking it only when a login page loads leaves a gap before dispatch.

## Does polling make this too much work for a beginner?

It adds a component.

Whether that component is acceptable depends on the fallback promise your product makes. With pull-based event collection, the reader-facing flow shouldn't depend on a delivery update arriving immediately. If immediate event-driven orchestration is a requirement, evaluate candidates' documented delivery-event mechanisms before choosing.

For the polling design, instrument three separate concerns: collector freshness, normalized event outcomes, and suppressed send attempts. A collector that has stopped can otherwise make a quiet dashboard look healthy. Track the last successful collection time and processing errors alongside delivery counts. Define the collection interval and alert threshold from your login experience requirements; there is no measured latency here from which to derive a universal number.

Keep raw addresses and OTP values out of metric labels. Log an opaque subscriber reference and a correlation identifier where needed, with access and retention appropriate to the application. One counter for invalid recipients and another for blocked fallback attempts explain different things: the first measures incoming evidence, while the second shows whether the send gate is enforcing it. Neither measures completed authentication.

## Can email fallback replace a managed authentication flow?

Only if your application owns the verification work. Sending a custom code email still leaves code generation, expiration, attempt limits, and successful verification to implement. Suppression protects the destination decision; it doesn't prove ownership of an email address or authorize an account recovery. Keep a verified alternative available through the authentication design rather than bypassing suppression to make a failed login appear successful.

For US/EU rollout, assess destination coverage and sender requirements for the actual countries you intend to serve. An API choice alone isn't a compliance finding. Geographic abuse controls and country-based spend circuit breakers also belong in the integration review when the application must implement them. These controls matter before a public login form becomes a sending trigger.

Finally, distinguish invalid-recipient suppression from a reader's subscription preferences. RFC 8058 addresses one-click unsubscribe for list email. It's useful background for a media publisher, but it isn't an OTP verification protocol or a definition of bounce semantics. Keep the login workflow, recipient validity, and newsletter preference decisions explicit. The easiest integration to maintain is the one whose ownership you can explain before the first subscriber needs help.

## References

- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify documentation](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [SendGrid event reference](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058)

The verification references support evaluating authentication workflows. The messaging and email-event references support evaluating transport and recipient evidence. Check the specific product contract before implementing an adapter; the TypeScript types above intentionally describe only local application policy.

## Further reading

- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Vonage Verify documentation: https://developer.vonage.com/en/verify/overview
- Amazon SNS SMS documentation: https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- Twilio SendGrid event webhook reference: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Twilio SMS documentation: https://www.twilio.com/docs/sms
- RFC 8058, One-Click Unsubscribe: https://datatracker.ietf.org/doc/html/rfc8058
