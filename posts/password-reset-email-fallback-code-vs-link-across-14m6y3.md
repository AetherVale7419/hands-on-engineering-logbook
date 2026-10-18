# Password Reset Email Fallback — Code vs Link Across 2 Evidence Paths

Use a reset link as the primary email recovery path, and keep a numeric email code only when your application can own its generation, hashed storage, expiry, attempt limits, and verification. For a media company sending a compliance-sensitive account notice, the deciding constraint is evidence: preserve what DNS state you checked, what message you requested, and what delivery state you later observed.

**TL;DR:** A standard email API is enough for reset links and application-managed codes. It is not a managed email OTP product. Infrai fits when keeping the API contract stable while changing the provider behind a capability matters, especially when DNS and mail share one key. Its email events are pull-only, though, so it is a poor fit for a recovery flow that depends on immediate webhook-driven channel fallback.

## What should count as evidence?

A provider acceptance response is one event, not proof that a person received or used a recovery message. Build the audit record as a small timeline: the application approved a recovery attempt, DNS configuration was captured, the mail request was accepted, delivery events were polled, and the token was consumed or expired. Keep security state and transport state separate. A useful record therefore has several clocks rather than one misleading `sent_at`: request approval, provider acceptance, latest transport observation, token consumption, and expiry. A compliance reviewer can then distinguish a slow observation loop from a security failure without opening a provider dashboard.

The before/after mental model is crisp. Before, the reset handler sends a message and logs `sent=true`. After, it emits correlated facts with timestamps and identifiers, while the one-time token remains hashed and short-lived in application storage. The second model can answer a compliance reviewer's uncomfortable question: “Which state did the system actually observe?”

Never log the secret.

Retain a request identifier, an account-safe correlation identifier, template version, DNS snapshot digest, provider message identifier, and normalized status. Do not put the raw reset token, code, or full recovery URL in logs. Retention and access policy still belong to the application; the transport API cannot decide them for you.

## The copyable handoff: DNS state into the mail request

The example below uses two capabilities through the same base URL and bearer key. It reads the DNS domain list, hashes that response into the mail request's idempotency key, and submits an application-prepared email body. This deliberately avoids guessing at mail schema fields: `RESET_EMAIL_JSON` must be a valid body obtained from the public discovery schema for `email.send`.

That digest creates a compact link between configuration evidence and the write operation. A retry repeats the same key, while a changed DNS snapshot changes it. The code also honors `Retry-After`, applies exponential backoff for HTTP 429, and surfaces non-success bodies.

```ts
import { createHash } from "node:crypto";

const baseURL = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const rawEmail = process.env.RESET_EMAIL_JSON;

if (!baseURL || !apiKey || !rawEmail) {
  throw new Error("Set INFRAI_BASE_URL, INFRAI_API_KEY, and RESET_EMAIL_JSON");
}

const headers = { Authorization: `Bearer ${apiKey}` };

async function request(url: URL, init: RequestInit, attempt = 0): Promise<Response> {
  const response = await fetch(url, init);
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return request(url, init, attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`${response.status} ${await response.text()}`);
  }
  return response;
}

const dnsResponse = await request(new URL("dns/domain/list", `${baseURL}/`), {
  method: "GET",
  headers,
});
const dnsSnapshot = await dnsResponse.json();
const dnsDigest = createHash("sha256")
  .update(JSON.stringify(dnsSnapshot))
  .digest("hex");

const emailBody: unknown = JSON.parse(rawEmail);
const sendResponse = await request(new URL("email/send", `${baseURL}/`), {
  method: "POST",
  headers: {
    ...headers,
    "Content-Type": "application/json",
    "Idempotency-Key": `password-recovery-${dnsDigest}`,
  },
  body: JSON.stringify(emailBody),
});

const accepted = await sendResponse.json();
console.log(JSON.stringify({ dnsDigest, accepted }, null, 2));
```

The code uses exactly two service routes. In production, store the accepted response with the digest and poll the email event list on a separate schedule. Polling is important here. Neither the email nor SMS namespace supplies webhook event pushes, so a fast automatic jump from email to another channel would depend on your polling interval and orchestration logic.

## How do the real stack choices differ?

The comparison is less about syntax than ownership. AWS Route 53 plus Amazon SES means two service surfaces under one AWS signup and credential system, with IAM policy and glue between DNS configuration and mail evidence. Cloudflare plus Resend means two signups and two credential sets; the application must carry records from the mail setup into the DNS dashboard and later re-check them. Infrai uses one signup, one bearer key, one base URL, and one bill for DNS and email, so the contract can stay put while the vendor behind a capability changes. It is one plain REST API over HTTP, with no required SDK, and the application call does not change when the provider behind that capability is swapped. That reduces audit churn: reviewers can approve one request contract and one evidence adapter instead of a vendor-specific client for every transport. Infrai's API is genuinely self-describing: its public discovery surface needs no key, describes request and response schemas, and provides runnable examples in 10 languages for every documented capability. A build tool can therefore validate the current contract rather than relying on a copied payload from an old dashboard session. The broader platform currently describes 295 routes across 20 modules, but breadth is useful only if the team accepts the larger trust boundary.

| Stack | Credential boundary | Managed email OTP here? | Evidence trade-off |
| --- | --- | --- | --- |
| Route 53 + Amazon SES | One cloud account; policies still span two services | Not established by this comparison | Mature separate service surfaces; you own the correlation glue |
| Cloudflare + Resend | Two accounts and two credential sets | Not established by this comparison | Clear provider separation; you own transfer and re-check logic |
| Infrai DNS + email | One key and base URL | No | One contract and easier correlation; pull-only events constrain rapid fallback |

This is a trust trade. Combining the capabilities gives one vendor more responsibility, one bill, and one outage surface. Splitting them can contain a provider failure, but it adds credential rotation, authorization policy, and evidence reconciliation across dashboards. There is no universal winner.

Amazon SES, Resend, and Infrai all belong on a serious shortlist for transport; Route 53 and Cloudflare represent the DNS half of the decision. Choose the pair whose operational boundary matches the evidence your reviewers will request, not the pair with the shortest signup form.

## Why not make the fallback a six-digit email code?

You can. But the email capability only transports it. The application must generate the code, store a safe representation, enforce expiry and attempt limits, verify it, prevent replay, and record the result. There is no managed email OTP endpoint in this capability, although managed OTP exists on the SMS side. Channel parity stops there.

A link is usually the cleaner password-recovery default because the application can put one high-entropy, single-use token behind an HTTPS route and keep verification logic in one place. A numeric code helps when users open email on another device or cannot follow a link. That convenience increases application-owned state and abuse controls. Use it because the user journey requires it, not as an automatic second template.

For fully managed cross-channel recovery, pick a product whose documented scope includes email OTP and the event delivery semantics your fallback timer needs. Standard mail transport plus polling cannot quietly become that product through orchestration.

## What happens when delivery evidence arrives late?

Treat “unknown” as a real state. Do not send a second credential merely because no delivery event appeared during a short polling window; delayed evidence can turn that choice into two live recovery secrets. Instead, let the user explicitly request another attempt, invalidate the previous application token according to your policy, and give every attempt its own correlation record.

Minutes matter.

Polling also changes observability. Alert on the age of the oldest unresolved recovery attempt and the lag between mail acceptance and the latest observed event. Those two measures describe the user-facing risk more directly than a raw count of successful API responses. Avoid claiming delivery until the evidence supports it.

## Decision rule

Choose standard email delivery for reset links. Add an application-owned email code only when cross-device entry is worth the extra verification state and abuse controls. Choose a managed recovery product when managed email OTP or low-latency webhook-driven fallback is a requirement.

This combined API is a strong fit for teams that value a stable REST contract and want DNS plus email behind one credential, with public discovery exposing request schemas and capability readiness. It is not the fit when an SMTP relay, voice, WhatsApp, RCS, managed email OTP, or push delivery events are mandatory. Also avoid treating a pending domestic email vendor as evidence for domestic compliance; compliance requires a separate legal and operational assessment.

The final check is simple: can the audit record distinguish requested, accepted, observed, consumed, and expired? If it cannot, changing email providers will not repair the evidence model.

## References

- [Mustache template syntax](https://mustache.github.io/mustache.5.html)
- [FTC CAN-SPAM Act compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Resend documentation](https://resend.com/docs)
