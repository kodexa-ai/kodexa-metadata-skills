# Service Bridge — Worked Examples

Each file below is complete and applyable with `kdx apply -f <file>`. Conventional location in a
sync repository: `service-bridges/<slug>.yaml`.

## 1. Unauthenticated reference lookup, cached

No credentials, no parameters — the endpoint returns the same list to everyone, which is exactly
when caching is safe.

```yaml
type: service-bridge
slug: currency-reference
orgSlug: acme-corp
name: Currency Reference
description: Public currency code list used to populate dropdowns.

baseUrl: https://reference.kodexa.example.com

endpoints:
  - name: list-currencies
    description: All active ISO currency codes
    path: /v1/currencies
    method: GET
    cacheEnabled: true          # GET is cache-eligible by default
    cacheTtlSeconds: 3600
```

## 2. API key + parameterised lookup for a form dropdown

The pattern most bridges want. Note `method: POST`: every in-platform caller sends its arguments as
a JSON body, and a `GET` endpoint would have that body dropped and reach the upstream API with no
parameters.

```yaml
type: service-bridge
slug: vendor-lookup
orgSlug: acme-corp
name: Vendor Lookup
description: Vendor master search for invoice review.

baseUrl: https://vendors.kodexa.example.com/api/v2
agentCallable: false

defaultHeaders:
  - name: Accept
    value: application/json
  - name: X-API-Key
    value: "${secrets.VENDOR_API_KEY}"

endpoints:
  - name: search-vendors
    description: Search vendors by partial name, scoped to a country
    path: /vendors/search
    method: POST
    cacheEnabled: true
    cacheableVerbs: [POST]      # required — without this a POST endpoint is never cached
    cacheTtlSeconds: 300
    requestSchema:
      type: object
      required: [country]       # until `country` is supplied, the call returns 200 []
      properties:
        country: { type: string }
        nameFragment: { type: string }
```

Provision the secret first, in the same organization as the bridge:

```bash
kdx secret set acme-corp VENDOR_API_KEY        # prompts, hidden input
kdx apply -f service-bridges/vendor-lookup.yaml
```

Consume it from a data form declaratively — form scripts have no bridge API:

```yaml
- component: v2:serviceBridgeView
  props:
    bridgeRef: acme-corp/vendor-lookup   # or service-bridge://acme-corp/vendor-lookup
    endpoint: search-vendors
    params: { country: "US" }   # becomes the request body
    transform: "$.results"      # optional JSONata applied before rendering
  children: [ ... ]             # children read the result as ctx.$bridgeResult
```

The component renders its children with the result injected as `ctx.$bridgeResult` (plus
`$bridgeLoading` / `$bridgeError`), so a `serviceBridgeView` with no children shows nothing. Move
`params` into the node's `bindings` instead of `props` to make it react to form data; the view
re-calls the bridge whenever the resolved params change.

## 3. OAuth2 client credentials, agent-callable

The platform owns the whole token lifecycle here — do not hand-roll a token in a header or a hook.

```yaml
type: service-bridge
slug: purchase-orders
orgSlug: acme-corp
name: Purchase Order Service
description: Creates and reconciles purchase orders in the Globex ERP.

baseUrl: https://erp.kodexa.example.com/api
agentCallable: true             # plus: bind this bridge to the agent's project

auth:
  type: oauth2_client_credentials
  tokenUrl: /oauth2/token                       # leading "/" => resolved against baseUrl
  clientId: "${secrets.ERP_CLIENT_ID}"
  clientSecret: "${secrets.ERP_CLIENT_SECRET}"
  requestFormat: custom                         # this IdP wants JSON, not form-encoding
  requestBody:
    grant_type: client_credentials
    client_id: "${auth.clientId}"
    client_secret: "${auth.clientSecret}"
    audience: https://erp.kodexa.example.com
  responseMapping:
    accessToken: token                          # non-standard field names
    expiresIn: ttl
  # tokenCacheBufferSeconds: 60   headerName: Authorization   headerPrefix: Bearer  (defaults)

defaultHeaders:
  - name: Accept
    value: application/json

endpoints:
  - name: create-purchase-order
    description: Create a purchase order from an approved invoice
    path: /v1/purchase-orders
    method: POST
    requestSchema:
      type: object
      properties:
        status:
          enum: [draft, submitted, approved]    # declaration only — see the note on the
        vendorId: { type: string }              #   status-enum rule in references/schema.md
        amount: { type: number }

  - name: get-purchase-order
    description: Fetch a purchase order by id
    path: /v1/purchase-orders/{poId}            # filled from a BRIDGE_CALL step's requestPath map;
    method: GET                                 #   the proxy does NOT substitute it, so a UI,
                                                #   SDK or agent caller sends the literal {poId}
```

Called from an activity plan (see the `activity-plan` skill for the full step schema — the step
fields are flat, and every `request*` value is a JSONata expression, so a string literal has to be
single-quoted):

```yaml
- slug: post-to-erp
  type: BRIDGE_CALL
  serviceBridgeRef: purchase-orders        # bare slug; an orgSlug/ prefix is stripped
  endpointName: create-purchase-order
  requestBody:
    status: "'submitted'"                  # literal — bare `submitted` is a path lookup
    vendorId: "inputs.vendorId"
    amount: "steps.extract.total"
  timeoutSeconds: 30                       # default 30, capped at 120
```

## 4. Reshaping an upstream response

`postReplyScript` runs only on the proxy path **and only when the endpoint is cache-eligible**, so
this endpoint sets both `cacheEnabled: true` and a matching `cacheableVerbs`. The script is wrapped
in an IIFE, receives a global `input`, must `return`, and gets 10 s.

```yaml
type: service-bridge
slug: receipt-classifier
orgSlug: acme-corp
name: Receipt Classifier
description: Third-party receipt classification, flattened for the review form.

baseUrl: https://classify.kodexa.example.com

defaultHeaders:
  - name: Authorization
    value: "Bearer ${secrets.CLASSIFIER_TOKEN}"

endpoints:
  - name: classify-receipt
    description: Classify a receipt and return a flat option list
    path: /v3/classify
    method: POST
    cacheEnabled: true
    cacheableVerbs: [POST]
    cacheTtlSeconds: 900
    requestSchema:
      type: object
      required: [text]
      properties:
        text: { type: string }
    preSendScript: |
      // Deterministic only — this body feeds the cache key. A timestamp or a
      // request id here would make every call a permanent cache miss.
      var text = (input.body && input.body.text) || "";
      return { body: { text: String(text).trim().toLowerCase() } };
    postReplyScript: |
      var rows = (input.body && input.body.predictions) || [];
      return {
        body: rows.map(function (r) {
          return { value: r.code, label: r.label + " (" + r.score + ")" };
        })
      };
```

## Wrong beside right

| Wrong | What happens | Right |
|---|---|---|
| `headers:` as a map at bridge level | unknown key, silently dropped; the bridge saves and sends no headers | `defaultHeaders:` as a list of `{name, value}` |
| a `caching:` block on the bridge | unknown key, silently dropped; nothing is ever cached | `cacheEnabled` / `cacheTtlSeconds` / `cacheableVerbs` **per endpoint** |
| `endpoints[].targetPath` | not a field; silently dropped | `path` — it *is* the remote path, appended to `baseUrl` |
| an endpoint with no `name` | unroutable: the name is the URL segment every caller uses | always set `name`, and treat it as immutable |
| `path: vendors/search` | concatenated as `https://vendors.kodexa.example.comvendors/search` | `path: /vendors/search` |
| `method: GET` on a parameterised lookup | the caller's JSON body is dropped; upstream is called with no parameters | `method: POST` (add `cacheableVerbs: [POST]` to keep caching) |
| `secretRef: MY_KEY` on a header | HTTP 400 at call time; the header is never injected | `value: "${secrets.MY_KEY}"` on a `defaultHeaders` entry |
| a credential in `endpoints[].headers` | not secret-resolved on the activity-plan path; also redacted from audit diffs | put credentials in `defaultHeaders` |
| `postReplyScript` on an endpoint with `cacheEnabled: false` | never runs; the raw upstream body is returned | make the endpoint cache-eligible, or shape the response upstream / in the consuming step |
| `cacheEnabled: true` on a per-user endpoint | one user's response is served to another | leave caching off unless the upstream is user-agnostic |
| `baseUrl: http://localhost:8080` | HTTP 400 "service bridge egress denied" | a publicly resolvable host |
| `agentCallable` omitted on a bridge an agent must call | permission denied at call time | `agentCallable: true` **and** bind the bridge to the agent's project |
| `type: serviceBridge` | legacy alias; persists a non-canonical discriminator and an unresolvable computed `uri` | `type: service-bridge` |
| secret referenced but never created in the org | the call fails with a server error instead of sending the placeholder (the secret name is in the server log, not the response) | `kdx secret set <org-slug> <name>` first |
| `${secrets.my.api.key}` — a dot in the secret name | no match against `[a-zA-Z0-9_-]+`, so the literal `${secrets.my.api.key}` is sent as the credential | rename the secret to `MY_API_KEY` |
