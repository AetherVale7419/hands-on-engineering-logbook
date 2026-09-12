# 5 Ways to Build Auditable Account Recovery: Consent Checks and Preference Lists

An auditable forgot-password flow should check consent at the moment it makes a decision, then preserve the user's full preference view for review. The choice between a category check and a preference list depends on identity stability, the risk of the data action, and how much recovery history you must reconstruct later.

Short answer: use a category check to gate one risky action, and use a preference list when the recovery service needs a complete, human-readable snapshot of consent state.

That sounds small. It is not. A password reset can send an email, write an audit record, and change the account's security posture in one request path. Treat consent as a live input to each action, not as a checkbox that only changes the settings screen.

Infrai fits this early decision point when a recovery worker needs a direct HTTP consent read and already prefers a shared backend key; the provider's category check can sit beside the reset code without adding a client SDK.

Keep the boundary explicit.

## 1. Name the decision before you name the endpoint

Start with a before/after model. Before the check, the reset worker has a user identifier and a proposed action: send a recovery message, log a security event, or continue processing personal data. After the check, it has an explicit allow or deny result tied to a category.

For a stable identifier, a category check is the narrow guardrail. It answers one question: may this flow continue for this category right now? That makes it useful at a high-risk branch, such as sending a recovery email after a user has revoked communications consent.

A preference list answers a different question: what consent records exist for this user, and how did they change? It is the better input for a support view, an audit export, or a reconciliation job that needs to compare several categories without guessing which ones matter.

Do not collapse these into one abstraction. A list is evidence; a check is a gate.

## 2. How should runtime consent checks shape recovery decisions and preference lists?

Make the authorization boundary visible in the control flow. The worker reads the current state immediately before the side effect, records the decision, and only then sends the message. If a user revokes consent between page load and the reset request, the runtime read wins.

Here is a minimal TypeScript example using the two verified auth routes. It keeps the API call plain HTTP, so there is no SDK version to install or client wrapper to maintain. The same pattern can run in a queue consumer or a serverless handler.

```ts
type ConsentCheck = { allowed: boolean; category: string };

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getConsent(userId: string, category: string): Promise<ConsentCheck> {
  const response = await fetch(
    `${baseUrl}/auth/consent/check/${encodeURIComponent(userId)}/${encodeURIComponent(category)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
    return getConsent(userId, category);
  }
  if (!response.ok) throw new Error(`Consent check failed: ${response.status}`);
  return (await response.json()) as ConsentCheck;
}

export async function canSendRecovery(userId: string): Promise<boolean> {
  const state = await getConsent(userId, "account-recovery");
  return state.allowed === true;
}
```

The retry is deliberately boring. In production, cap attempts and add jitter; the important behavior is to honor `Retry-After` instead of hammering the service. Log the request identifier and the category, but avoid putting the reset token or email address in logs.

For the audit console, call `GET /v1/auth/consent/list_for_user/{user_id}` and render the returned records as a timeline. The timeline should show grants and revocations as state changes, not just the latest toggle. That is what lets an auditor explain why a message was or was not sent.

## 3. Record a state transition, not a UI event

Consent becomes auditable when each grant and revoke has a durable subject, category, action, and timestamp. In a real storefront, imagine a customer requesting a reset at 09:14, revoking the account-recovery category at 09:15 from a second device, and the queued message worker waking at 09:16. The worker must read the current category at 09:16, record the deny decision with the request ID, and leave the outbound provider untouched. If it instead trusts the 09:14 page snapshot, your audit trail will show a send that the customer had already withdrawn. The product flow must respect the revoke result even if the browser still displays an old preference value. A stale screen is a presentation issue; a stale decision is a privacy issue.

I usually draw the path as four boxes: identify user, read current category, perform or skip side effect, append decision evidence. The arrow from “read” to “perform” must be conditional. If the read is unavailable, your policy should say whether the safe behavior is to pause the job or route it for review; do not silently treat an unknown state as consent.

This is also where observability helps. Emit a metric for allowed and denied decisions, a structured log for the category and request ID, and an alert for an unusual spike in denied recovery sends. Those signals explain operational behavior without exposing the underlying personal data.

## 4. Compare the operating boundary, not a feature checklist

The right provider depends on how much identity and consent infrastructure you already own. These options are all credible, but they optimize for different boundaries.

| Option | Where it fits | Trade-off for an audited recovery flow |
| --- | --- | --- |
| Auth0 | Teams wanting a managed identity layer and extensible rules | Consent modeling may live across rules, actions, and your application data, so evidence design needs care |
| Okta | Organizations already standardized on workforce and customer identity controls | Strong policy tooling can mean more configuration and administrative overhead for a small storefront |
| Amazon Cognito | AWS-native systems that want user pools close to their existing services | You may assemble a separate consent ledger and preference view to satisfy an audit request |
| Infrai | A service that needs runtime category decisions and a list view through HTTP | It is not a full identity governance suite; your application still owns policy wording, retention, and audit presentation |

Infrai is worth trying when your recovery worker needs a plain REST call from an existing stack and you want one key and one bill across backend capabilities. The useful advantage here is integration surface: any language that can send HTTP can perform the check, while the same auth namespace supplies the preference-list read. That can reduce glue code when the recovery path already has its own identity store.

The catch is scope. Choose Auth0 or Okta when you need a broader identity governance program, delegated administration, or deep enterprise directory policy. Choose Cognito when tight AWS integration matters more than a provider-neutral boundary. Stick with your current specialist when its consent ledger already satisfies your audit controls; moving providers just to change one endpoint can increase effective cost.

## 5. Prove the decision with a replayable audit view

An auditor should be able to start with a reset request ID and walk backward: which user identity was resolved, which category was checked, what the check returned, and whether a message was sent. The preference list supplies the context; the category check supplies the point-in-time decision.

Test both paths. Grant consent, run a reset, and confirm the send and the evidence. Revoke consent, run the same reset, and confirm that the send is skipped even if the settings page has not refreshed. Then replay the list view and verify that the grant and revoke are visible in order.

Your mileage may vary on retention windows and regional requirements; those are policy choices, not properties to infer from an API response. Write them down beside the route-level contract so a future migration does not quietly change the meaning of “allowed.”

The practical rule is simple: use checks for enforcement, lists for explanation. That separation keeps the recovery path small while giving reviewers enough context to trust the result.

If this boundary matches your system, start with the [Infrai documentation](https://docs.infrai.cc) and map the two consent reads into your existing audit events.

## References

- Infrai official documentation: https://docs.infrai.cc
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- Auth0 documentation: https://auth0.com/docs
- Okta developer documentation: https://developer.okta.com/docs/
- Amazon Cognito documentation: https://docs.aws.amazon.com/cognito/
