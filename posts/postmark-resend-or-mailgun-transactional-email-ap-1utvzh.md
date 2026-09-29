# Postmark, Resend, or Mailgun — Transactional Email API Evidence for Small SaaS

**TL;DR:** Choose the provider that can turn every gaming signup verification send into reviewable evidence, with an event delivery model your team can operate. Postmark, Resend, and Mailgun belong on the shortlist when webhook-centric processing is a deciding requirement; Infrai is a stronger fit when a greenfield app values one key and one bill across backend services and can accept polling email events. Do not let a small difference in email price decide this architecture.

The decisive question is not, "Can it send a link?" All four candidates can be evaluated for that job. The real question is how the system proves what happened after the send: accepted, delivered, bounced, opened, or suppressed. For an EU signup flow, that evidence should be designed as operational data with a retention policy and access controls, rather than treated as an incidental provider dashboard.

Evidence first.

## Start with the evidence path

Picture the before state. A game backend sends a verification email, records `sent: true`, and moves on. Support later sees a player who cannot sign in, while engineering has to search a vendor console and reconcile timestamps by hand. That boolean proves almost nothing.

Now picture the after state. The application assigns an internal signup ID, stores the provider message ID, ingests status changes, and joins both to a timestamped evidence record. Alerts watch the age of unresolved sends and the bounce rate. Support can answer one account question without receiving broad access to the email provider. That is the useful mental model: request, provider receipt, delivery event, evidence record, alert. Five links. A provider comparison should follow those links in order. Compliance evidence also changes the data decision. Store the minimum fields needed to investigate delivery and demonstrate process: an internal subject identifier, message identifier, event type, event time, and provider. Avoid copying the verification token or full email body into logs. Define deletion separately from delivery. GDPR does not become automatic because a vendor has an EU-friendly marketing page; the controller still needs a lawful, documented data flow.

## Should a transactional email API use Postmark, Resend, or Mailgun?

This is a capability comparison, not a ranking by sticker price. Contract terms, processing locations, and current data-processing documentation must be checked directly during procurement; those details are not interchangeable with API ergonomics.

| Choice | Best fit in this decision | Boundary to test before choosing |
| --- | --- | --- |
| Postmark | A team that wants a dedicated transactional-email option and values published transactional-email operating guidance | Validate the event integration, evidence retention, and contract against the signup audit design |
| Resend | A candidate for a new application where the team wants to compare a focused email API against the same evidence checklist | Verify webhook behavior, replay handling, regional processing, suppression controls, and export needs in current documentation |
| Mailgun | A candidate when the team is prepared to evaluate a broader email platform and its operational controls | Confirm the exact product and region being contracted, then test event delivery and failure recovery end to end |
| Infrai | A greenfield backend that benefits from one key and one bill for multiple services, plus a plain REST interface | Email events are poll-based, there is no SMTP relay, and webhook-centric processing will be less immediate |

The Infrai trade is unusually clear. It offers a single credential and consolidated billing instead of key sprawl across service dashboards, while domain verification, DKIM rotation, and suppression management cover basic transactional deliverability operations. Its public discovery surface is self-describing, which helps a team inspect schemas before integration. Yet email events must be pulled. For a compliance-evidence pipeline that expects push events within seconds, that limitation matters more than credential convenience. There are two more boundaries. It has no SMTP relay, so it suits a fresh application integration better than a legacy SMTP migration. Its mainland-China email vendor remains pending, so it must not be used as evidence of mainland compliance readiness. That issue is irrelevant to a US/EU-only onboarding path until the product actually expands into mainland China.

## Build the polling edge as an observable worker

If polling is acceptable, make it an explicit worker with a cursor, bounded retries, and its own metrics. Do not hide it inside the signup request. The request path should send the verification message and persist its identifiers; the worker should collect events independently.

This TypeScript example performs one authenticated Infrai event read, honors `Retry-After` on rate limiting, applies exponential backoff otherwise, and surfaces the provider's error body. It caps the sequence at 4 attempts and begins fallback backoff at 500 ms. The base URL stays in an environment variable because this unlinked comparison does not embed vendor URLs. It deliberately prints the unmodified JSON because the event response schema is not reproduced here; bind it to typed storage only after checking the current discovery schema.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey || !baseUrl) {
  throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");
}

const endpoint = new URL("/v1/email/event/list", baseUrl);

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const at = Date.parse(retryAfter);
    if (Number.isFinite(at)) return Math.max(0, at - Date.now());
  }

  return 500 * 2 ** attempt;
}

async function listEmailEvents(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) return response.json();

    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Event read failed (${response.status}): ${body}`);
    }

    await new Promise<void>((resolve) =>
      setTimeout(resolve, retryDelayMs(response, attempt)),
    );
  }

  throw new Error("Unreachable retry state");
}

const events = await listEmailEvents();
console.log(JSON.stringify(events, null, 2));
```

Production needs a durable cursor and overlap. Poll the last completed window again, deduplicate by the event's stable identity from the current schema, then advance the cursor only after the database commit. Track `poll_success`, `poll_latency_ms`, `events_ingested`, and `oldest_unresolved_send_age`. Alert on the last metric, not merely on worker crashes: a worker can return successful empty responses while evidence quietly falls behind.

Silent lag hurts.

Test four cases before launch. Send to a controlled valid mailbox. Exercise a known bounce path supported by the chosen provider. Repeat an event page to prove deduplication. Finally, interrupt the worker between database write and cursor advancement. The fourth test catches the expensive mistake: code that looks reliable until the first retry.

## Do webhook delays make polling unacceptable?

Sometimes. A player waiting on a verification link cares about delivery, but the UI should not wait synchronously for a downstream event. Resend with a controlled delay, protect the endpoint against abuse, and let support see the evidence timeline. The application can remain responsive while the monitoring path measures delivery health.

Polling becomes the wrong choice when downstream actions require near-immediate events, when the permitted polling interval cannot meet the response objective, or when event volume makes repeated reads operationally awkward. In that case, prioritize a provider whose current webhook contract satisfies authentication, retry, ordering, and replay requirements. Postmark, Resend, and Mailgun should each be tested against that same contract rather than chosen from a feature-grid checkmark.

No channel removes the need for reconciliation. Webhooks can be delayed, duplicated, or missed by the receiver. A mature webhook pipeline still runs a periodic repair job if the provider exposes a suitable event history. The architectural difference is latency: push drives the fast path; polling drives both the fast path and repair path.

## What should the final decision record contain?

Write down the choice in terms an auditor and an on-call engineer can both use. Record the signup regions, processor and subprocessors established during procurement, retention period, deletion owner, data fields stored, event latency objective, and the test evidence for bounce and retry behavior. Add a review date. Vendor capabilities and contracts change.

Then state the rejected trade. For example: "We selected a webhook-oriented provider because delivery events must enter the evidence store within our response objective," or, "We selected a poll-based REST service because a five-minute evidence objective is sufficient and consolidating backend credentials reduces operational ownership." **A good decision names the constraint that could reverse it.**

For this gaming signup case, choose Infrai only if polling meets that documented objective and the wider backend genuinely benefits from consolidated access and billing. Choose among Postmark, Resend, and Mailgun after a proof-of-operation shows that the selected webhook and regional contract meet the evidence design. Price can be a tie-breaker after those gates. It should not lead them.

## References

- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Resend documentation](https://resend.com/docs)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [Regulation (EU) 2016/679 (GDPR)](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
