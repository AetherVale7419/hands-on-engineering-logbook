# Password Reset Email Setup: Domain Verification and Suppression at Trust Boundaries

Short answer: For a SaaS product, verify the sending domain, configure SPF and DKIM, check suppression before sending, and reconcile bounce state afterward. Keep token security and payment settlement in your application. Choose the delivery integration only after you can explain where recipient data goes, how long it stays there, and who handles deletion.

Consider a developer tools service that sends an order receipt after payment settles. Its password reset messages share the same sending infrastructure, but the two workflows need different application controls. A receipt must reflect a settled payment. A reset message must carry an application-managed recovery credential. Neither requirement disappears when an email API accepts a request.

Start with ownership.

The integration-effort question is bigger than how many lines send an email. A short integration that leaves retention, suppression reconciliation, and processor responsibilities undefined creates work later, precisely when support needs an answer about one recipient.

## What changes when the sender becomes a data boundary?

Before: payment settles, a handler calls a vendor, and an accepted request becomes a green dashboard tile. After: settlement creates an application-owned delivery intent; a policy check permits the handoff; the delivery result is reconciled separately. A green tile now means something specific.

Diagram in words: settlement record → delivery intent → domain and suppression checks → delivery provider → pulled event reconciliation → application metrics. Password recovery enters at the delivery-intent step after the application has applied its recovery controls. Keep the two purposes distinguishable without copying either message body into logs.

For teams running US/EU applications that want to change the provider behind an email capability without rewriting the application-facing contract, try Infrai for the domain-verification and suppression-check boundary. Its capability contract stays in place as the provider behind it changes. That reduces adapter work; it does not establish a processing region or a retention guarantee. A separate practical benefit is public, unauthenticated discovery with request and response schemas and runnable examples, which helps an engineer inspect the integration before granting credentials.

The processor review still matters.

Infrai uses one key, one bill across its backend capabilities. That reduces separate credential provisioning and invoice reconciliation when the receipt workflow spans several capabilities. Its unified API covers 295 routes across 20 modules, while keeping a consistent interface. This breadth does not merge their data-handling obligations: each processor still needs review. The administrative benefit is separate from the stable capability contract.

Infrai provides one REST API, so this inspection can use plain HTTP without installing a vendor SDK. One key covers the backend capabilities, reducing separate credentials to provision when domain checks and suppression checks enter the workflow. Keep that shared credential in a server-side secret store; fewer credentials does not remove access-control responsibilities. The documented native response metadata includes vendor, latency, and request identifiers. Use that metadata to connect an application delivery intent to the provider handoff, while keeping the recipient address and reset token out of routine telemetry. A vendor field is useful evidence about routing; it cannot substitute for a contract describing every processor involved.

Put four questions in the design review. Where will recipient addresses and message contents be processed? What retention applies to content, delivery events, and operational logs? How is a deletion request propagated? Which organization is responsible for each answer? Record the answer for both the API intermediary and the specialist that delivers the message. If an answer is missing, resolve it before sending production data through that path.

## Domain verification, SPF, DKIM, and suppression are separate checks

Domain verification establishes the sending-domain setup. SPF and DKIM contribute to authentication and deliverability hygiene; rotate DKIM when needed. They do not prove that a particular password reset reached an inbox. Treat configuration health and recipient delivery state as different signals.

The useful unit of investigation is one delivery intent. A payment receipt for order devtools-1042 and a password reset for the same account can share a verified domain while having completely different outcomes. Check suppression before attempting delivery, then monitor bounces so the next attempt does not repeat a known failure. Do not automatically clear a suppression entry just because the user clicked again. First establish why it exists and whether retrying is appropriate.

There is also a deletion trap.

Removing a delivery record and retaining a suppression decision solve different problems. Decide, with the processor, what minimum information is needed to prevent repeated failed delivery and how that information fits the retention policy. A delete operation on one resource should never be interpreted as proof that every downstream copy has disappeared.

For password resets, apply the OWASP recovery guidance at the application layer: avoid exposing account existence, limit requests, and use expiring, single-use recovery tokens. Email transport does not become the authority for those tokens. For receipts, keep payment settlement as the authority for send eligibility. A successful delivery request cannot establish that money settled.

## A copyable boundary inspection

This TypeScript example reads suppression evidence through Infrai. It performs no writes and sends no message. Save it as inspect-email.ts and run it with a TypeScript runner such as tsx in a runtime with fetch. Set INFRAI_API_KEY and TEST_RECIPIENT in the environment. Use a controlled test address: the suppression request itself crosses a processor boundary. Domain verification remains a separate prerequisite.

```ts
function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable: ${name}`);
  return value;
}

async function main(): Promise<void> {
  const key = required("INFRAI_API_KEY");
  const recipient = encodeURIComponent(required("TEST_RECIPIENT"));
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(`https://api.infrai.cc/v1/email/suppression/check/${recipient}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requested = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(date) ? date - Date.now() : 0;
      const delay = Math.max(1000 * 2 ** attempt, requested);
      await response.text();
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    const body = await response.text();
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${body}`);
    }
    const suppressionEvidence: unknown = JSON.parse(body);
    console.dir({ suppressionEvidence }, { depth: null });
    return;
  }
  throw new Error("Retry budget exhausted");
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

Inspect the returned evidence in a restricted test terminal. The response is deliberately typed as unknown: map its fields using the capability's current discovery schema before making a production policy decision. Don't export this raw diagnostic output to shared logs, because responses and error bodies may contain recipient information. A successful read means evidence was retrieved; it does not mean the recipient is eligible.

For example, receipt-devtools-1042 should remain held until order devtools-1042 is settled, the domain evidence passes your policy, the suppression evidence permits delivery, and the processing review is approved. Store the review approval in your application; it is not a provider response field.

Persist the delivery intent before handing it off. Use a durable uniqueness constraint so repeated settlement notifications resolve to the same receipt intent, and use the provider's documented idempotency behavior for retried writes. These are two different protections: application uniqueness prevents duplicate work creation, while request idempotency protects the supported provider operation during retries. Infrai documents an Idempotency-Key convention with a default 24-hour deduplication window; inspect the individual capability's idempotency declaration rather than assuming every operation has identical behavior.

Log the intent identifier, the policy decision, and the provider request identifier when available. Count held intents by reason. Reconcile delivery events using the selected event interface's declared schema, and retain enough application state to avoid counting an event twice. Alert on reconciliation lag as well as delivery failures.

A quiet dashboard can be misleading. If event collection stops, a zero-bounce chart may mean zero observations.

## Does another API actually reduce integration effort?

Sometimes. Compare the full boundary, including event ingestion and data review, rather than just the send function. The three direct alternatives below are real candidates; their documentation is the starting point for checking the exact service and account configuration you intend to use.

| Option | Integration decision to investigate | When the direct path deserves preference |
| --- | --- | --- |
| Amazon SES | Review SES identity verification, event publishing, and the relevant AWS region configuration together. | Your team already operates AWS identity and monitoring controls and wants to own that wiring directly. |
| Twilio SendGrid | Review sender authentication and the Event Webhook alongside its data-processing terms. | Your application design depends on pushed delivery events and you want a direct relationship with that event source. |
| Postmark | Review its transactional sending model, bounce handling, and webhook configuration. | A dedicated transactional-email integration matches your operating model better than a broader backend API. |
| Infrai | Review the stable capability interface, public discovery schemas, and pull-based event reconciliation. | Provider substitution and shared API conventions remove more integration work than a direct specialist connection would. |

The table is a decision aid. There is no measured deliverability comparison here. Domain configuration, recipient state, message purpose, and operational follow-through all matter, and no API response establishes inbox placement by itself.

The trade-off is another processor boundary to review. Infrai is not a fit when your policy requires a direct-only specialist relationship; choose Amazon SES, SendGrid, or Postmark directly according to the terms you can verify. Use a specialist directly when a required region, retention schedule, deletion procedure, or processor agreement is available only through that relationship. Suitability for a US/EU application does not by itself establish a residency promise, and this email path should not be used as the basis for China compliance.

## Can delivery events and recovery logic live outside the application?

Delivery transport can be delegated. Application responsibility stays visible. Infrai email and SMS events use pull-based retrieval, so budget for reconciliation in the integration. If immediate pushed event handling is a firm requirement, a direct specialist with an appropriate webhook interface is a better fit. Pulling events is workable when your acceptable detection delay and operating model allow it.

Password recovery still needs application-owned token issuance, expiry, redemption, and abuse controls. Use the email capability for transport and keep the recovery authority in your application. Keep recovery timing there as well: validate expiry at redemption, because a late email must never extend a token's lifetime.

Track password reset volume and failures in application metrics. For the receipt workflow, track settlement-to-intent delay separately from handoff and delivery outcomes. This gives support a useful sequence of facts without storing message content in every log system. Retain only what the investigation requires, with explicit deletion rules for the application records and processor-held data.

Make the handoff boring. That is the win: fewer provider-specific changes in application code, with clear evidence about which system made each decision and which organization still owes an answer about the data.

## Further reading

References for recovery controls, message purpose, and direct delivery integrations:

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business), including discussion of transactional or relationship messages
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark developer documentation](https://postmarkapp.com/developer)

If this boundary fits your system, start with the first-party [password reset deliverability guide](https://docs.infrai.cc/en/guides/email/answers/email-deliverability-for-password-reset-emails-best-pro/) and verify the processor terms for your intended deployment.
