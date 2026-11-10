# Seller SMS OTP 2FA — Suppression List Telemetry for Blocked Numbers

A gaming marketplace seller needs to learn about a new order. If opening the seller dashboard requires SMS OTP 2FA, a suppressed phone number can interrupt that journey before the seller ever sees the order. Short answer: check the suppression list before sending an authentication code, expose a useful blocked-number state, and issue a session only after successful server-side verification. Keep order notifications separate from authentication challenges.

The useful integration question is how much application logic remains after choosing a messaging service. A send operation alone does not supply challenge expiry, attempt limits, account recovery, or the explanation a support agent needs. Make those responsibilities visible before picking an API.

## Turn the login journey into observable states

Before: an application records “SMS requested,” then treats the seller's inability to continue as one undifferentiated login problem. After: the application records suppression eligibility, challenge creation, send outcome, verification outcome, and session issuance as separate transitions. Now support can distinguish an ineligible destination from an expired challenge without reading a message body or asking the seller to disclose a code.

Name the states.

Diagram in words: new order → seller notification → seller opens dashboard → authentication challenge → suppression check → eligible destination → OTP send → server verifies code → session issued. A blocked destination branches to recovery. An unreachable destination branches to a delivery explanation. Neither branch grants access, and an order alert never doubles as proof that its recipient completed authentication. This separation matters because a seller can receive an order notification yet still need a fresh authentication challenge before seeing protected order details.

Keep blocked number, too many attempts, expired code, and retry later distinct in application state. The user-facing wording can be less specific where detailed responses would reveal account existence. Support visibility and public error text serve different audiences; OWASP's recovery guidance is useful for deciding how to preserve that distinction.

## What should an SMS OTP 2FA suppression list record for blocked numbers?

Record the decision and its place in the journey. Useful application fields include an opaque challenge ID, purpose, transition, and outcome. These are application-owned fields, not a proposed provider request schema. Avoid recording raw OTPs, full phone numbers, session credentials, or recovery codes.

Use a concrete example: order `game-order-1042` belongs to seller `seller-27`. Its notification job concerns the order; challenge `challenge-73` concerns a login attempt. Keep those identifiers separate so an order-notification retry cannot accidentally become a request to create another authentication challenge. Maintaining the suppression list also matters: a prior opt-out or repeated delivery failure should lead to an explicit eligibility decision instead of an endless resend loop.

A useful metric counts challenge outcomes by state. Another counts how often eligible challenges reach a verified result. Track delivery outcomes separately; an accepted send is not a verified login. For alerts, choose thresholds from your own traffic and operational response capacity. No universal failure percentage tells a small gaming marketplace when to wake someone up.

Start with counts.

## A copyable application-side transition example

This TypeScript example demonstrates the policy boundary with synthetic inputs. Save it as `seller-auth.ts` and run it in a TypeScript-capable Node.js environment. It executes locally, makes no network requests, and prints the resulting audit events. It is a transition example, not a complete authentication server: a production adapter must obtain suppression and verification results on the server, persist challenge state, and enforce attempt limits.

```ts
import { strict as assert } from "node:assert";

type State = "created" | "awaiting_code" | "blocked_number"
  | "too_many_attempts" | "expired_code" | "retry_later"
  | "invalid_code" | "verified";
type Event = "suppressed" | "sent" | "unreachable"
  | "limited" | "expired" | "invalid" | "verified";
type Challenge = { id: string; state: State };

function transition(c: Challenge, event: Event): Challenge {
  const allowed: Partial<Record<State, Partial<Record<Event, State>>>> = {
    created: {
      suppressed: "blocked_number",
      sent: "awaiting_code",
      unreachable: "retry_later",
      limited: "too_many_attempts",
    },
    awaiting_code: {
      expired: "expired_code",
      invalid: "invalid_code",
      limited: "too_many_attempts",
      verified: "verified",
    },
  };
  const next = allowed[c.state]?.[event];
  if (!next) throw new Error(`Invalid transition: ${c.state}/${event}`);
  console.info(JSON.stringify({
    challenge_id: c.id, purpose: "seller_login",
    from: c.state, to: next,
  }));
  return { ...c, state: next };
}

const blocked = transition(
  { id: "challenge-73", state: "created" }, "suppressed",
);
assert.equal(blocked.state, "blocked_number");
assert.throws(() => transition(blocked, "verified"));

const sent = transition(
  { id: "challenge-74", state: "created" }, "sent",
);
const expired = transition(sent, "expired");
assert.throws(() => transition(expired, "verified"));

const confirmed = transition(
  transition({ id: "challenge-75", state: "created" }, "sent"),
  "verified",
);
assert.equal(confirmed.state, "verified");
```

The important boundary sits outside this function: a browser must never be allowed to submit a trusted `verified` event. The authentication service derives it from successful server-side code verification for the correct challenge and account. Session issuance then consumes that successful result once, with expiry and attempt limits checked against authoritative stored state. Serialize concurrent updates so two requests cannot both consume the same challenge.

This small example stops after an invalid code. A production retry policy needs additional transitions and a persisted attempt counter. That deliberate boundary keeps the teaching example readable without quietly claiming to implement a complete OTP protocol.

## Does a managed verification product remove this work?

It changes the integration boundary. It doesn't remove the need to explain what happened to the seller. Compare the following options by the work your application must own, and check the linked documentation for the precise service contract before implementation.

| Option | Integration boundary | Work to evaluate for this seller journey |
| --- | --- | --- |
| Twilio Verify | A dedicated verification product | Map its verification lifecycle into local login states and recovery decisions. |
| Vonage Verify | A dedicated verification product | Evaluate its verification workflow against your suppression policy and support-state vocabulary. |
| Amazon SNS SMS | Messaging delivery | Budget application work for challenge creation, verification, expiry, and session issuance. |

Infrai offers a plain REST API, one key, and one bill for the email and SMS integrations. An HTTP-capable backend can call it without installing an SDK or maintaining a client-library version. The self-describing API has public discovery with no key required, so a developer can inspect the suppression-check JSON Schema before wiring up the seller-login adapter. That helps separate contract questions from local policy decisions; the application's suppression policy, challenge lifecycle, and session boundary still need explicit design.

The trade-off is application ownership. This approach is not a good fit if your requirement is to delegate the verification lifecycle; evaluate Twilio Verify or Vonage Verify instead. The discovery surface supplies schemas, not your account-recovery policy.

To inspect the suppression-check contract before implementing the adapter, the following read-only example fetches its discovery schema. Set `INFRAI_API_BASE_URL` to the platform's versioned API base and `INFRAI_API_KEY` through server environment configuration. It uses built-in HTTP support. A maximum of 3 attempts bounds rate-limit retries; other HTTP errors surface their response body. This is schema inspection, not an OTP send, so it creates no challenge or message.

```ts
const base = process.env.INFRAI_API_BASE_URL;
const key = process.env.INFRAI_API_KEY;
if (!base || !key) throw new Error("Missing API configuration");

async function inspectSuppressionSchema(): Promise<unknown> {
  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch(
      `${base!.replace(/\/$/, "")}/discovery/sms.suppression.check`,
      { method: "GET", headers: { Authorization: `Bearer ${key}` } },
    );
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requested = Number.isFinite(seconds) ? seconds * 1000
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      await response.text();
      const wait = Math.max(requested, 1000 * 2 ** attempt);
      await new Promise(resolve => setTimeout(resolve, wait));
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Retry budget exhausted");
}

console.info(JSON.stringify(await inspectSuppressionSchema()));
```

For an application that already owns authentication state, a messaging-oriented integration may fit better. Compare a blocked-number journey and an expired-code journey during evaluation, rather than judging integration effort from the length of a successful-send snippet. Neither a short snippet nor a long feature list establishes the amount of support work you will inherit.

## Can email recovery and delivery telemetry close the gap?

Recovery needs its own design. For the plain REST option discussed above, plan recovery codes or an email fallback; voice, WhatsApp, and RCS are outside its channel scope. An email fallback requires application-managed email-code verification. Enforce expiry and attempt limits there too, and avoid automatically moving a seller onto an unverified recovery destination.

Delivery events for that integration are retrieved by polling. Build the polling cadence into your operational design, since the support view's freshness depends on it. Keep polling off the critical path for granting sessions: the decisive event remains successful code verification, not a delivery update. Geographic abuse controls and country-based spending circuit breakers also belong in the application policy. These responsibilities count toward integration effort even when the HTTP call is small.

The practical acceptance check is compact: a suppressed destination reaches recovery; an expired code cannot authorize a session; a verified challenge can be consumed only once. Then make sure support can tell those outcomes apart. That is the useful before-and-after for the seller waiting to handle an order.

## Further reading

References for verification boundaries, recovery controls, and messaging integration:

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify documentation](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
