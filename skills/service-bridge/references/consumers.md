# Who Calls a Bridge, and What Each Caller Actually Honours

The same bridge YAML is executed by **two different engines**. They do not support the same feature
set. Design the bridge for whichever consumers you actually have — and if a bridge is used by both,
author to the intersection.

## The call paths

| # | Caller | Engine | How it addresses the bridge |
|---|---|---|---|
| 1 | UI: the `v2:serviceBridgeView` form component, and any document formula calling `serviceBridgeCall("<org>/<bridge>", "<endpoint>", key, value, …)` — most commonly a `selectionOptionFormula` | platform proxy | resolves the slug, then `POST /api/service-bridges/{id}/proxy/{endpointName}` with the arguments as a **JSON body** |
| 2 | Python (module code, SDK): `from kodexa_document.platform import call_service_bridge`, then `call_service_bridge(client, "acme-corp/vendor-lookup", "lookup-vendor", {...})` | platform proxy | same route, always `POST`, body is the dict you pass (`None` for no body); a non-2xx raises `ServiceBridgeError` |
| 3 | Activity plan: `BRIDGE_CALL` step, and the `enrichment` block on an `LLM` step | orchestrator, direct HTTP | looks the bridge up by slug in the org, then calls the named endpoint |
| 4 | Activity plan: `serviceBridge.call(ref, endpointName, body?)` inside a `SCRIPT` step | orchestrator, direct HTTP | looks the bridge up by slug **joined to the running project's resource bindings** — an unbound bridge is "not found" here even though path 3 finds it |

A headless agent running an `AGENT` step uses path 1's route and therefore gets the proxy's full
feature set — plus two extra gates (`agentCallable: true` **and** a project binding). Unlike paths
1 and 2 it can put parameters in the proxy URL's query string as well as in a body, so it is the
one caller that can drive a `method: GET` endpoint with parameters.

## Feature matrix

| | Proxy (1, 2, agents) | Orchestrator (3, 4) |
|---|---|---|
| `${secrets.X}` in `baseUrl`, `auth.clientId/clientSecret/tokenUrl` | resolved | resolved |
| `${secrets.X}` in `defaultHeaders[].value` | resolved | resolved |
| `${secrets.X}` in `endpoints[].headers[].value` | resolved | **not resolved** — the literal placeholder is sent as the header value |
| `cacheEnabled` / `cacheTtlSeconds` / `cacheableVerbs` | honoured | **ignored** — there is no caching on this engine, and a `BRIDGE_CALL` step's `disableCache` is a documented no-op |
| `initScript`, `preSendScript` | run | **ignored** — never carried into this engine |
| `postReplyScript` | runs **only when the endpoint is cache-eligible** | **ignored** — the raw upstream body is what you get |
| `requestSchema.required` guard | applied | not applied |
| `responseSchema` | not read | not read |
| `{token}` placeholders in `endpoints[].path` | not substituted | **substituted**, URL-escaped, from a `BRIDGE_CALL` step's `requestPath` map — and only there; `LLM` enrichment and `serviceBridge.call()` send the literal |
| OAuth token acquisition + caching | yes | yes |
| OAuth 401/403 → evict token, retry once | yes, on a cache-eligible endpoint | no |
| Per-call header overrides | no | `BRIDGE_CALL` only (`requestHeaders`; CR/LF/NUL rejected) |
| SSRF egress fence | yes | yes for `BRIDGE_CALL` and `LLM` enrichment; **no** for `serviceBridge.call()` in a `SCRIPT` step, which uses a plain HTTP client |

## Per-path specifics

### Proxy (paths 1, 2, agents)

- Route: `/api/service-bridges/{bridgeId}/proxy/{endpointName}`. `{bridgeId}` is the **UUID** — get
  it from `POST /api/resolve?path=service-bridge://<orgSlug>/<slug>`; the slug is not accepted in
  the path.
- The route answers any HTTP method, but the **upstream** verb is always `endpoints[].method`.
- The incoming query string is appended to `baseUrl + path` verbatim. Of the callers above, only an
  agent sets one; paths 1 and 2 send a body only.
- The request body is **dropped** when `endpoints[].method` is `GET` — the single most common cause
  of a lookup that silently returns everything or nothing.
- An unknown `endpointName` is a 404 that lists the endpoints that do exist. An empty one is a 400.
- Response headers the proxy sets: `X-Service-Bridge-Cache: HIT|MISS`, `X-Bridge-Context` (when an
  `initScript` produced one; not set on a cache hit), `X-Service-Bridge-Guard:
  required-fields-missing` (the `requestSchema.required` short-circuit).
- Non-2xx upstream statuses are passed straight back to the caller.
- 30 s per call; bodies read to 10 MB (larger ones stream through un-cached).
- Requires `service-bridge` read permission on the bridge, in the bridge's organization.

Data-form scripts have **no** HTTP or bridge namespace — a form reaches a bridge declaratively
through `v2:serviceBridgeView` (`bridgeRef`, `endpoint`, `params`, optional JSONata `transform`) or
through a `selectionOptionFormula`, not from script code.

### Activity-plan `BRIDGE_CALL` (path 3)

- Configured with `serviceBridgeRef` + `endpointName`; see the `activity-plan` skill for the full
  step schema.
- `requestBody` is sent for **any** method, `requestQuery` becomes the query string, `requestPath`
  fills `{token}` placeholders in the endpoint path.
- `timeoutSeconds` defaults to 30 and is **capped at 120**.
- Non-2xx is not automatically an error: the `treatAsError` JSONata predicate decides, and its
  default is `$._statusCode < 200 or $._statusCode >= 300`.
- An empty `endpointName` is rejected by the step before it reaches the engine.

`LLM`-step `enrichment` shares this engine with two differences: `inputMapping` becomes a JSON
**body** when the endpoint's method is `POST` and a **query string** otherwise, and a non-2xx is
always a failure.

### Activity-plan `SCRIPT` step `serviceBridge.call()` (path 4)

- Signature: `serviceBridge.call(bridgeRef, endpointName, body?)`, plus `serviceBridge.list()`.
- Both resolve against the running **project's** bound bridges, so the bridge must be a project
  resource — `serviceBridge.list()` returning nothing usually means the binding is missing.
- **The body is marshalled only when the endpoint's method is `POST`.** A `PUT` or `PATCH` endpoint
  called from a script silently sends no body.
- Fixed **10 s** timeout, not overridable, and a hard limit of **10 bridge calls per script run**.
- Any non-2xx **throws** a JS error carrying the status and body.
- An unknown endpoint name throws immediately.

## Designing a bridge that serves both engines

1. Keep every credential in `defaultHeaders` — it is the only header collection whose secrets
   resolve on both engines, and endpoint headers are additionally redacted from the audit trail.
2. Don't rely on `postReplyScript` to reshape a response that an activity-plan step will consume;
   the step sees the raw upstream body. Shape it upstream, or read it downstream — a completed
   `BRIDGE_CALL` step always publishes `steps.<slug>._statusCode` and `steps.<slug>._body`, so a
   following `SCRIPT` or `LLM` step can do the reshaping. (`outputMapping` is **not** persisted on a
   plan-authored `BRIDGE_CALL` step, so it is not the lever here.)
3. Don't rely on caching for activity-plan traffic — it isn't there. Size the upstream for the call
   volume.
4. Prefer `method: POST` for parameterised endpoints: it is the only declared method for which
   every caller can hand parameters to the upstream API. Add `cacheableVerbs: [POST]` if you also
   want the proxy to cache it.
