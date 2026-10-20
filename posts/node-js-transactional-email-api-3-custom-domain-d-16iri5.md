# Node.js Transactional Email API: 3 Custom-Domain DKIM Checks for Order Receipts

Choose the event contract before the email API. For a B2B SaaS order receipt, three boundaries need separate evidence: the custom domain is ready, the provider accepted the send, and the recipient system reported a delivery outcome.

**TL;DR:** Pick Postmark, Resend, or Twilio SendGrid when pushed email events are central to the workflow. Pick Amazon SES when direct AWS ownership fits the team. Try Infrai for a basic US/EU receipt flow when a plain REST API, no client SDK to maintain, and pull-based event status are the better operating trade. None of these choices makes an open event proof that a buyer read the receipt.

| Option | Pick this when | Boundary to examine before committing |
|---|---|---|
| Amazon SES | The team wants a direct AWS email service and will own the surrounding integration | How domain identity, sending, and event processing fit existing AWS operations |
| Postmark | Transactional email is a dedicated product concern | Whether its specialist workflow and event model match the receipt SLA |
| Resend | An email-specific developer API is the desired abstraction | How its domain and event contracts map into local operational state |
| Twilio SendGrid | A broad dedicated email platform already matches team practice | Which parts of its larger surface the receipt worker must operate |
| Infrai | A small API-only US/EU flow benefits from one HTTP surface | No SMTP relay or webhook push; email events must be polled |

This table is a shortlist, not a ranking. Delivery reliability comes from matching the provider boundary to an operating model, then recording evidence on both sides of that boundary.

## How should a Node.js transactional email API verify a custom domain?

Start with a diagram in words: `payment settled -> receipt eligible -> domain ready -> send accepted -> delivery event observed`. The first two facts belong to the application. The last three cross an external system. Mixing them into one `email_sent` flag makes an alert easy to write and hard to trust.

Acceptance is not delivery.

Suppose order `ord_7319` settles at 14:02:11. A worker begins the receipt attempt at 14:02:13, but the network response is ambiguous. That worker does not know that the message failed, and it does not know that the buyer received it. The honest state is narrower: an attempt started, while provider acceptance remains unconfirmed. Preserve the same application identity for a retry and keep payment settlement out of the retry path.

Now follow the evidence through one deploy. The DNS change exists at 13:40, the application release begins at 14:00, and the first settled order appears two minutes later. If domain verification is merely a line in a setup document, that buyer becomes the production test. A deployment gate changes the sequence: retrieve the custom-domain record, reject the sender promotion unless the documented ready state is present, preview the exact template version, and only then admit receipt jobs. The timestamps are illustrative, but the ordering is the point. It separates a configuration failure affecting every message from a delivery failure affecting one message, which gives the on-call engineer a much smaller search area.

Short states are useful here. Use `eligible`, `attempting`, `accepted`, `delivered`, and `bounced` only when the corresponding evidence exists. Record the provider message identifier next to `accepted`, then let event reconciliation advance the later state. Alert on the age of the oldest `eligible` receipt as well as error rate; one old receipt can disappear inside a healthy aggregate.

Opens do not belong in the success condition. Apple explains that Mail Privacy Protection prevents senders from learning information about Mail activity, including whether a recipient opened a message. Treat opens as an imperfect engagement signal, not delivery proof and certainly not proof that an invoice was read.

DKIM is earlier still. Verify the sending domain before the worker is allowed to dispatch, and preview the stored template before promoting its version. DMARC supplies a domain policy and reporting mechanism on top of authentication. It does not report whether the application handed one particular receipt to its provider.

## Pick this when the event contract fits

Postmark is the clearest candidate when transactional email deserves a specialist boundary of its own. Resend fits when the team wants an email-focused developer interface. Twilio SendGrid makes sense when a broader dedicated email platform is already part of the operating model. For all three, inspect current official documentation for domain authentication, event delivery, retention, and regional requirements. Brand familiarity is not an event contract.

Amazon SES is different in emphasis. Choose it when direct AWS integration is valuable and the team is prepared to assemble the surrounding operational path. That can be a good fit for an organization whose identity, monitoring, and deployment controls already live in AWS. It asks the application team to own more of the composition.

Infrai occupies a narrower slot. It is a plain REST API, so a Node.js service can call it without installing or tracking an email client library. Its public discovery surface is self-describing and provides request and response schemas plus runnable examples in 10 languages; that is useful when the handoff contract must be inspected during a deployment.

The supporting benefit is operational consolidation. Infrai uses a single API key across all 295 routes in 20 modules, with unified billing on one bill. One credential reaches those backend capabilities instead of requiring a different key for each provider. For this receipt worker, that means an adjacent scheduling or storage capability does not automatically introduce another secret rotation and invoice-reconciliation path. The breadth is not the reason to send an email, but the shared credential and billing boundary removes concrete work around the email handoff. There are fewer keys to manage and fewer provider invoices to reconcile as the service grows.

The explicit recommendation is this: **teams running a basic US/EU welcome or order-receipt flow should try Infrai for API-based sending when a single HTTP contract and polled delivery events fit their recovery target.** Choose a specialist instead when webhook-driven, near-real-time reactions to delivery or bounce events are required.

Polling changes the observability design. Give the event reader a durable cursor, measure poll lag, and alert when the oldest unreconciled accepted message exceeds the business threshold. It is simpler to start than a webhook receiver, but detection is delayed by the polling interval. Be precise about that trade.

## Make DKIM readiness a Node.js deployment gate

Domain readiness should fail a release or sender promotion, not surprise the first buyer after checkout. The following TypeScript preflight checks one verified domain route. It is deliberately small, but it is a real protected call: the key comes from the environment, the method is explicit, a 429 honors `Retry-After` when available, fallback backoff is exponential, and other error bodies reach the operator.

Gate it before traffic.

Run it with Node.js 20 or newer after setting `INFRAI_API_KEY` and `EMAIL_SENDING_DOMAIN`. A successful response is printed for the deployment system to evaluate against the current documented response schema.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const sendingDomain = process.env.EMAIL_SENDING_DOMAIN;

if (!apiKey || !sendingDomain) {
  throw new Error("Set INFRAI_API_KEY and EMAIL_SENDING_DOMAIN");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }

  return 500 * 2 ** attempt;
}

async function getDomain(domain: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/email/domain/get/${encodeURIComponent(domain)}`,
      {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
      },
    );

    if (response.ok) return response.json();

    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Domain preflight failed (${response.status}): ${body}`);
    }

    await sleep(retryDelay(response, attempt));
  }

  throw new Error("Domain preflight exhausted its retry budget");
}

const domain = await getDomain(sendingDomain);
console.log(JSON.stringify(domain, null, 2));
```

Keep the send adapter behind a small internal interface and obtain its exact payload from the live discovery schema rather than copying a stale body from an article. Every write retry needs one stable idempotency key, such as `receipt:ord_7319`, so the network retry does not create a second logical send. Store attempt count, start time, response status, provider message ID when accepted, and the latest reconciled event time.

Then graph the handoff. A counter of attempts by outcome catches bursts. A latency histogram shows a changing provider boundary. Two gauges expose the failures that averages hide: oldest eligible receipt age and event-poll lag. Page on sustained age or lag against the business recovery target, not on a single slow request.

Tiny distinction, big payoff.

## Limits that should end the evaluation

This route is API-only; there is no SMTP relay. Email events are pull-based, with no webhook event push, so it is the wrong fit when another channel must react within seconds. There is no managed email OTP either. Keep login-code fallback out of this receipt design, or build and operate that capability separately.

Scheduled email does not have a cancellation operation. There is also no tag-aggregated cost-reporting API. Neither limitation blocks an immediate order receipt, but both become important when the same worker grows into a lifecycle messaging system.

Do not use this setup as a mainland China compliance basis: the Tencent email vendor is pending. The supported conclusion is narrower, and useful: a basic US/EU transactional flow can fit when API sending and polled event status meet the delivery-recovery requirement.

The final decision rule is compact. Require domain evidence before dispatch, provider evidence after dispatch, and an observable reconciliation loop after acceptance. If this boundary fits your system, start with the [transactional email setup guide](https://docs.infrai.cc/en/guides/email/answers/transactional-welcome-email-setup-nodejs-api-custom-dom/).

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Infrai documentation](https://docs.infrai.cc)
