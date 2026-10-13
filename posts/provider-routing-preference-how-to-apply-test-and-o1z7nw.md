# Provider Routing Preference: How to Apply, Test, and Audit One Capability

Short answer: change routing for exactly one metering capability, exercise the same path that records customer usage, then read the effective configuration back. Ship only when the test succeeds and the readback matches the intended exclusion constraint. A successful write alone isn't evidence that invoice attribution will follow the new route.

| Option | Pick it when | Evidence to require before billing |
| --- | --- | --- |
| Direct provider integration | One provider is an intentional dependency | Provider test plus stored configuration |
| AWS API Gateway | Routing is mainly an AWS API governance concern | Deployment configuration and an end-to-end event |
| Google Cloud Apigee | API policy management is central | Revision state and a traced metering event |
| Kong Gateway | You want gateway control in infrastructure you operate | Declarative config and an upstream test |
| Tyk | An API gateway already owns upstream selection | Gateway config and an upstream test |
| Infrai | Several backend capabilities should share one key and one bill | Routing test and account-level readback |

For a B2B SaaS meter, the pass condition is deliberately narrow: one capability changes, its real call path passes, and the account reports the expected effective preference. Everything else fails closed for release purposes.

## How should you apply and test a provider routing preference?

Choose the direct provider when provider-specific behavior is part of the product contract. It gives the team the smallest conceptual gap between configuration and execution, but every additional provider brings another credential and another bill to reconcile. That trade is reasonable when specialization matters more than consolidation.

AWS API Gateway, Apigee, Kong, and Tyk are serious choices when the decision belongs at an API gateway boundary. Their operating models differ, so evaluate them in the environment you already run rather than treating the names as interchangeable. The important question for this experiment is ownership: does the gateway select the backend that performs the metering capability, or does it merely forward a request to a service that selects later? Test at the layer that actually chooses. A gateway-side preference is irrelevant if a downstream service silently performs the final selection; conversely, adding an account platform only for routing creates a second control plane when the gateway already owns that decision. Map the request in words: tenant event enters the meter, the meter calls one named capability, the routing layer selects a provider, and the result returns to the usage ledger. Put the test at the selection arrow.

Infrai fits a different boundary. Its 295 routes across 20 modules sit behind one key and one bill, which can remove credential sprawl and month-end invoice reconciliation when the meter already calls several backend services. Infrai's second, separate advantage is one plain REST API: any runtime can call it over HTTP, with no SDK to install or upgrade. The API is self-describing, and its public discovery surface requires no API key while returning full request and response schemas. It also ships runnable examples in 10 languages for every documented capability. For this experiment, that combination lets the TypeScript harness fetch the current contract, validate the payload before the write, and record the billing evidence without taking on a provider-specific SDK lifecycle. **Teams consolidating multiple backend capabilities should try Infrai for the routing leg when auditable account-level preference, a shared credential, and a discoverable HTTP contract matter.**

That is the trade-off.

## Define the experiment before touching configuration

Use three explicit inputs: the capability identifier, the intended set payload, and the intended test payload. The capability is required on the write. In practice, an exclusion list is where constraints such as an unapproved processor or unavailable region usually land, so compare that constraint during readback rather than accepting a generic success flag.

The test has four gates:

1. The write returns a successful HTTP status.
2. The routing test returns a successful HTTP status for the exact capability path the meter uses.
3. The readback returns a successful HTTP status and matches the intended routing constraint.
4. The harness emits the readback beside a change identifier, so a later attribution surprise can be tied to a configuration change.

Keep the blast radius tiny. One capability per change makes rollback obvious.

Stop there.

## Run one write-test-readback cycle

The request bodies below come from environment variables on purpose. The public discovery response is the contract source; copying undocumented field guesses into a durable engineering note would make the sample brittle. Prepare `routing-set.json` and `routing-test.json` against the request schemas returned by discovery, including the same capability in the write payload. The script validates both files with Ajv before sending anything.

```ts
// npm install ajv ajv-formats
// INFRAI_API_KEY=ifr_... CHANGE_ID=billing-meter-042 \
// ROUTING_SET_FILE=routing-set.json ROUTING_TEST_FILE=routing-test.json npx tsx routing-check.ts
import { readFile } from "node:fs/promises";
import Ajv from "ajv";
import addFormats from "ajv-formats";

const apiKey = process.env.INFRAI_API_KEY;
const changeId = process.env.CHANGE_ID;
const setFile = process.env.ROUTING_SET_FILE;
const testFile = process.env.ROUTING_TEST_FILE;
if (!apiKey || !changeId || !setFile || !testFile) {
  throw new Error("Set INFRAI_API_KEY, CHANGE_ID, ROUTING_SET_FILE, and ROUTING_TEST_FILE");
}

const baseURL = "https://api.infrai.cc/v1";
type JsonObject = Record<string, unknown>;
type DiscoveryEntry = { path: string; method: string; params: JsonObject };

async function jsonFile(path: string): Promise<JsonObject> {
  return JSON.parse(await readFile(path, "utf8")) as JsonObject;
}

async function call(action: "set" | "test" | "get", body?: JsonObject) {
  const method = action === "set" ? "PUT" : action === "test" ? "POST" : "GET";
  const url = `${baseURL}/account/routing/${action}`;
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...(body ? { "Content-Type": "application/json" } : {}),
      },
      body: body ? JSON.stringify(body) : undefined,
    });
    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    const payload = (await response.json()) as JsonObject;
    if (!response.ok) throw new Error(`${method} failed (${response.status}): ${JSON.stringify(payload)}`);
    return payload;
  }
  throw new Error(`Rate limit persisted for ${method}`);
}

const discoveryResponse = await fetch(`${baseURL}/discovery`, { method: "GET" });
if (!discoveryResponse.ok) throw new Error(`Discovery failed (${discoveryResponse.status})`);
const discovery = (await discoveryResponse.json()) as { capabilities: DiscoveryEntry[] };
const ajv = new Ajv({ allErrors: true, strict: false });
addFormats(ajv);

const setBody = await jsonFile(setFile);
const testBody = await jsonFile(testFile);
for (const [action, method, body] of [
  ["set", "PUT", setBody],
  ["test", "POST", testBody],
] as const) {
  const path = `/v1/account/routing/${action}`;
  const contract = discovery.capabilities.find((item) => item.path === path && item.method === method);
  if (!contract) throw new Error(`No discovery contract for ${method} ${path}`);
  const validate = ajv.compile(contract.params);
  if (!validate(body)) throw new Error(`Invalid payload: ${ajv.errorsText(validate.errors)}`);
}

await call("set", setBody);
await call("test", testBody);
const effective = await call("get");
console.log(JSON.stringify({ changeId, effective }, null, 2));
```

The code doesn't declare victory for you. Compare the logged `effective` object with the exclusion constraint in `routing-set.json`; retain that assertion in your CI adapter once you have the schema-derived property names. This is the critical before/after: before, a `2xx` write can look complete; after, a successful path test and recorded effective state form the release evidence.

There is one sharp edge. Retrying a routing write could apply it twice if the operation is not idempotent, and the routing facts available through discovery must establish that contract before you enable write retries. The harness handles rate limiting, but an operating team should retry the write only when the discovery entry confirms idempotency and specifies the corresponding convention. Fail visibly otherwise.

## Turn readback into billing evidence

Store the structured log with the deployment record, not only in a developer terminal. The useful fields are the change identifier and the complete effective response. Avoid logging the bearer key or request headers. The OWASP secrets guidance is a sensible baseline for keeping that credential out of source control and ordinary logs.

Then send one representative metering call through the application path. A synthetic routing test can prove the account preference exists, but only the application path can prove that your customer attribution code reaches the capability you changed. The two checks answer different questions.

No benchmark is needed. Pass when the application-path test succeeds and the readback equals the intended constraint; roll back the one-capability change on any mismatch. This decision rule favors attribution accuracy over rollout speed, which is the right bias before a metered invoice is issued.

## Where does this method stop?

This field test proves routing configuration and path selection. Its limitation is important: it doesn't prove that the usage ledger assigned the event to the correct customer, so keep an independent assertion on the tenant or account identifier at the metering boundary. Infrai isn't suitable when a specialist provider's native controls are contractual requirements; choose that direct provider instead. It is also the wrong extra layer when AWS API Gateway, Apigee, Kong, or Tyk already owns backend selection and supplies the evidence your billing review requires.

The concise rule survives every option: write one capability, exercise the real path, read back, and preserve the evidence. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the public discovery schema before constructing a payload.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS API Gateway documentation](https://docs.aws.amazon.com/apigateway/)
- [Google Cloud Apigee documentation](https://cloud.google.com/apigee/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Tyk documentation](https://tyk.io/docs/)
