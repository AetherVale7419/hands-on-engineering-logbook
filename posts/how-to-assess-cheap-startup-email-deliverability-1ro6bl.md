# How to Assess Cheap Startup Email Deliverability APIs (Custom Domains Required)

A cheap email deliverability API can serve a healthtech startup, but the support message must still reach the right queue when the channel is delayed, rejected, or deliberately suppressed. That operational constraint changes the choice: select the API only after the application owns routing state, idempotency, and delivery evidence.

TL;DR: accept the contact once, classify it into an internal queue, persist that decision, and treat email as an observable delivery attempt rather than the queue itself. Then test any candidate API for custom-domain verification, overlap during DKIM key rotation, machine-readable delivery events, and suppression controls. Cheap and simple are useful tie-breakers. They are poor reliability models.

The before picture is fragile: form submission -> email request -> shared inbox. The after picture is inspectable: form submission -> durable case -> queue decision -> notification attempt -> delivery event -> operator view. One extra boundary makes failures visible.

## Can a cheap startup email API protect custom-domain deliverability?

An accepted request proves only that the delivery service accepted work. It does not prove that a support mailbox received the message, that an agent saw it, or that a reply entered the case record. Model those as separate facts.

This distinction matters for domain authentication too. DMARC evaluates whether a message aligns with an authenticated domain and gives domain owners a published policy mechanism; it does not replace application-level delivery tracking. A passing authentication result and a completed support handoff answer different questions.

Start with four states: `case_created`, `notification_requested`, `delivery_observed`, and `agent_acknowledged`. Keep the original case available to an operator even if the last three never happen. **Email signals the work; it must not contain the only copy of the work.**

Keep the case.

That also limits exposure. The notification can carry a case ID, queue name, and a link to the controlled case view instead of repeating every form field. The exact field policy belongs to the organization, but the architectural choice is straightforward: routing metadata travels farther than sensitive message content.

## Build the routing boundary first

Here is a small, runnable TypeScript model. It uses no provider-specific routes. The example makes three decisions explicit: a stable idempotency key, a queue chosen before notification, and a suppression check that creates an operator-visible task rather than silently dropping work.

```ts
type Queue = "urgent-clinical" | "billing" | "general-support";
type Contact = {
  submissionId: string;
  subject: string;
  category: "symptom" | "billing" | "other";
  urgent: boolean;
  replyTo: string;
};

type DeliveryRequest = {
  idempotencyKey: string;
  queue: Queue;
  recipient: string;
  subject: string;
  caseId: string;
};

type DeliveryResult =
  | { status: "accepted"; attemptId: string }
  | { status: "suppressed"; reason: string };

interface DeliveryPort {
  send(request: DeliveryRequest): Promise<DeliveryResult>;
}

const route = (contact: Contact): Queue => {
  if (contact.urgent && contact.category === "symptom") {
    return "urgent-clinical";
  }
  if (contact.category === "billing") return "billing";
  return "general-support";
};

const recipients: Record<Queue, string> = {
  "urgent-clinical": "urgent@example.invalid",
  billing: "billing@example.invalid",
  "general-support": "support@example.invalid",
};

async function notify(contact: Contact, delivery: DeliveryPort) {
  const queue = route(contact);
  const caseId = `case_${contact.submissionId}`;
  const request: DeliveryRequest = {
    idempotencyKey: `contact:${contact.submissionId}:queue:${queue}`,
    queue,
    recipient: recipients[queue],
    subject: `[${caseId}] ${contact.subject}`,
    caseId,
  };

  const result = await delivery.send(request);
  const event = {
    event: result.status === "accepted"
      ? "notification_requested"
      : "operator_action_required",
    caseId,
    queue,
    result,
    observedAt: new Date().toISOString(),
  };

  process.stdout.write(`${JSON.stringify(event)}\n`);
  return event;
}

const demoPort: DeliveryPort = {
  async send(request) {
    return request.recipient.startsWith("urgent")
      ? { status: "accepted", attemptId: "attempt_demo_001" }
      : { status: "suppressed", reason: "recipient_policy" };
  },
};

void notify(
  {
    submissionId: "7f3a2",
    subject: "Question about today's visit",
    category: "symptom",
    urgent: true,
    replyTo: "patient@example.invalid",
  },
  demoPort,
);
```

Do not generate a new idempotency key on every retry. A retry with a fresh key looks like new work and can create duplicate notifications. The useful operational question is not “Did the function throw?” It is “Which case, queue, and attempt does this event describe?”

The adapter behind `DeliveryPort` is intentionally boring. Keep it that way. Mapping a provider response into `accepted` or `suppressed` belongs there; deciding which team owns an urgent message does not.

## Test the controls that protect continuity

Evaluate candidates with a sandbox domain and a written scorecard. Avoid a feature-checkbox contest. Ask each implementation to produce evidence for the same scenario.

| Exercise | Evidence to retain | Failure decision |
|---|---|---|
| Verify a custom sending domain | Current verification state and its timestamp | Block production sending until the state is ready |
| Rotate a DKIM key | Old and new selector observations during a planned overlap | Pause removal of the old key until the new path is verified |
| Submit the same case twice | One logical notification identity | Reject or coalesce the duplicate |
| Suppress a recipient | Reason, scope, time, and review path | Create operator work; preserve the case |
| Receive a delivery event twice | One state transition and two raw observations | Deduplicate by event identity |
| Delay an event | Case remains searchable before the event arrives | Alert on age, not on a guessed final outcome |

DKIM rotation deserves a rehearsal because DNS publication, signing configuration, and observation are separate steps. The safer decision rule is an overlap: publish the new selector, verify messages using it, then retire the old selector according to the team's change window. Do not turn rotation into an instant swap whose only validation is “the configuration call succeeded.” For DMARC, record the organizational choice separately from application configuration. RFC 7489 defines `none`, `quarantine`, and `reject` as requested handling policies, and also defines aggregate reporting through `rua`. Those are domain-level controls, while the routing service retains its own case and attempt evidence regardless of that policy. Keep SMS fallback out of the happy path unless the product truly needs it. If it does, test message encoding as part of the delivery contract: GSM-7 messages have a 160-character single-message limit, while UCS-2 messages have a 70-character limit; concatenated messages use smaller per-segment limits. A tiny copy edit can therefore change segmentation. The fallback event should record the chosen channel and segment count without pretending that submission equals human acknowledgment. The trade-off is explicit: a second channel creates another observable route, but it also adds encoding, policy, and acknowledgment states that the team must operate.

Prove the overlap.

## What should the dashboard and alerts show?

Show the work funnel, not a vanity success rate. A useful view starts with cases created by queue, then shows notification requests, observed outcomes, suppressions, retry age, and agent acknowledgments. Operators need counts and the oldest outstanding case. Engineers need a trace from `caseId` to every attempt.

Use separate alerts for separate owners. A rise in domain-authentication failures belongs with the sending-domain owner. A growing oldest-unacknowledged age belongs with support operations. A suppression increase may need both teams, because the delivery control is working while the human workflow still requires attention.

Three timers are enough for the first version: age since case creation, age since notification request, and age since last agent action. Choose thresholds from the support commitment, not from a generic email benchmark. **An alert must point to recoverable cases, not merely announce a percentage.**

Log identifiers and transitions. Avoid logging unrestricted form bodies or full recipient data just to make debugging convenient. A compact event with `caseId`, `queue`, `attemptId`, outcome, and timestamps supports correlation while keeping the observability surface focused.

## Resolve the two common objections

“Isn't this too much machinery for a startup?” The minimum version is a case table, one delivery adapter, structured events, and an operator list for old or suppressed cases. No elaborate orchestration is required. The trade-off is a little application state in exchange for retries that can be explained and support work that survives a channel failure.

“Can't the provider own suppressions and retries?” It can enforce its delivery policy, but the application still owns the consequence. A suppressed address may be correct from a sending perspective while an urgent health support case remains unresolved. Import suppression events, retain their reason and scope, and send the case to a reviewed path. Never “fix” a suppression by blindly resending.

Only after these exercises pass should procurement compare operational burden and price shape. Check access controls, event retention, export paths, domain separation between environments, and how a team rolls credentials. Then compare total fit. A low entry price cannot compensate for missing evidence during an incident, and a long feature list cannot repair a routing model that loses the original case.

The durable choice is the API that fits behind a narrow adapter and supplies enough evidence for your team to operate the workflow. **Own the case, observe the channel, and make every failure leave a next action.**

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- Twilio, SMS character limits and segmentation (GSM-7/UCS-2): https://www.twilio.com/docs/glossary/what-sms-character-limit

## Sources

- https://datatracker.ietf.org/doc/html/rfc7489
- https://www.twilio.com/docs/glossary/what-sms-character-limit
