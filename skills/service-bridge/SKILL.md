---
name: service-bridge
description: "Use when authoring, reviewing or debugging a Kodexa service bridge — the org-scoped YAML that proxies an external HTTP API: baseUrl, named endpoints, defaultHeaders carrying ${secrets.NAME}, OAuth2 client-credentials auth, per-endpoint response caching, requestSchema guards, initScript/preSendScript/postReplyScript hooks, and the agentCallable flag that lets activity-plan BRIDGE_CALL steps and agents invoke it."
---

# Kodexa Service Bridge Authoring

A service bridge is an **organization-scoped** resource holding one external API's base URL,
its credentials, and a list of **named endpoints**. Callers never name a URL — they name the
bridge and an endpoint *name*. The platform appends `endpoints[].path` to `baseUrl`, layers on
the configured headers, resolves secrets, and returns the upstream response.

## The shape that works

```yaml
type: service-bridge                       # canonical discriminator
slug: vendor-lookup
orgSlug: acme-corp                         # picks the target org for `kdx apply`
name: Vendor Lookup
description: Looks up vendor records in the Globex vendor master.

baseUrl: https://vendors.kodexa.example.com/api/v2
agentCallable: false                       # absent/false = agents are denied

defaultHeaders:                            # a LIST of {name, value}, not a map
  - name: Accept
    value: application/json
  - name: X-API-Key
    value: "${secrets.VENDOR_API_KEY}"

endpoints:
  - name: lookup-vendor                    # REQUIRED, unique, and a cross-resource contract
    description: Look up a vendor by code
    path: /vendors/search                  # appended to baseUrl — must start with "/"
    method: POST                           # verb sent UPSTREAM (see below)
    cacheEnabled: true
    cacheableVerbs: [POST]                 # empty => only GET endpoints are ever cached
    cacheTtlSeconds: 600                   # 0 or omitted => 300s
    requestSchema:
      type: object
      required: [vendorCode]
      properties:
        vendorCode: { type: string }
```

Apply with `kdx apply -f service-bridges/vendor-lookup.yaml`. The file needs `type`, `slug` and
`orgSlug` (or pass `--type` / `--org-slug`).

**Author flat, as above.** A nested `metadata:` block still decodes on write, but every API
response and every `kdx sync pull` is flat, and pull never deletes on-disk keys it did not send —
so a nested file re-pulled later carries a stale `metadata:` block *and* flat duplicates.

## Credentials

Put every credential in a **`defaultHeaders` entry's `value`** as `${secrets.NAME}`:

```yaml
defaultHeaders:
  - name: Authorization
    value: "Bearer ${secrets.VENDOR_API_TOKEN}"     # Bearer
  - name: X-API-Key
    value: "${secrets.VENDOR_API_KEY}"              # API key
  # Basic: name: Authorization, value: "Basic ${secrets.VENDOR_BASIC_B64}"  (pre-encoded)
```

- `secretRef` is in the wire schema and in the UI, but **using it fails the call with HTTP 400**
  ("…use secretRef, which is not supported and would be sent empty…"). It is never injected.
  `${secrets.NAME}` in `value` is the only supported placement.
- Secret names must match `[a-zA-Z0-9_-]+`. Anything else (a dot, a space) simply won't match the
  pattern and the literal `${secrets.…}` string is sent as the credential.
- Secrets are organization-scoped. Provision them with `kdx secret set <org-slug> <name>`
  (`kdx secret list` returns names only — values are never readable back).
- `${secrets.X}` is resolved in exactly four kinds of place: `baseUrl`, header **values**,
  `auth.clientId` / `auth.clientSecret`, and `auth.tokenUrl`. **Not** in `endpoints[].path`, not in
  `auth.requestBody`, and not in request bodies.
- For platform-managed OAuth2 client credentials, use the `auth:` block — see
  `references/schema.md`.

## What fails silently

**`method` is the verb sent upstream, and a `GET` endpoint discards its parameters.**
The proxy always issues `endpoints[].method` upstream regardless of how the proxy was called, and
it **drops the request body when that method is GET**. Data forms, selection-option formulas and
the Python SDK helper all send their arguments as a JSON body with no query string — so a
parameterised lookup declared `method: GET` reaches the upstream API with no parameters at all,
returns whatever the unfiltered call returns, and logs nothing. Declare a parameterised lookup
`method: POST`. (Only a caller that puts parameters in the proxy URL's own query string, which
none of those three do, gets them through to a GET endpoint.) `POST` is also the only method for
which `serviceBridge.call()` in an activity-plan `SCRIPT` step marshals a body at all — a `PUT` or
`PATCH` endpoint called from a script silently sends none.

**`path` must begin with `/`.** The target is naive concatenation — `path: vendors/search` against
`baseUrl: https://api.kodexa.example.com` produces `https://api.kodexa.example.comvendors/search`.
Nothing validates this.

**`postReplyScript` only runs on a cache-eligible endpoint.** On the proxy the response-transform
branch is reached only when `cacheEnabled: true` **and** the endpoint's `method` appears in
`cacheableVerbs` (default `["GET"]`). With `cacheEnabled: false`, or on a POST endpoint without
`cacheableVerbs: [POST]`, the raw upstream body is returned and the script never executes. The
OAuth 401/403 evict-and-retry-once behaviour sits in that same branch.

**None of the hooks, and none of the caching, exist on the activity-plan path.** A `BRIDGE_CALL`
step, `serviceBridge.call()` in a `SCRIPT` step and `LLM`-step `enrichment` run on a different
engine that never loads `initScript`, `preSendScript`, `postReplyScript`, `cacheEnabled`,
`cacheTtlSeconds`, `cacheableVerbs` or `requestSchema` — a step gets the raw upstream response and a
live call every time (`disableCache` on the step is a no-op). Shape the response upstream, or read
`steps.<slug>._body` in a downstream step. Full matrix in `references/consumers.md`.

**The cache lookup runs *after* `initScript` and `preSendScript`, not instead of them.** Those two
always run, and the body `preSendScript` produces is what the cache key is hashed from — so a
`preSendScript` that injects a timestamp or request id makes every call a permanent miss. It is
`postReplyScript` that is skipped on a hit.

**`requestSchema.required` short-circuits the call.** If any listed field is absent or explicitly
`null` in both the JSON body and the query string, the proxy answers **HTTP 200 with body `[]`**
and header `X-Service-Bridge-Guard: required-fields-missing`, makes no upstream call, and records
no bridge event. Presence is the test — `""`, `0` and `false` all count as provided. Callers cannot
distinguish this from a genuine empty result, so only declare `required` where an empty list is a
sensible "not filled in yet" answer (this is what data-form dropdowns want).

**Endpoint `name` is a contract, not a label.** It is the URL segment callers address, and it is
named by activity-plan `BRIDGE_CALL` steps (`endpointName`), by `serviceBridge.call(...)` in
`SCRIPT` steps, and by data-form / selection-option bridge references. Renaming or deleting one
breaks every consumer at call time with no warning at save time. Add a new endpoint, migrate
callers, then remove the old one.

**Keys the model does not have are dropped without an error.** Decoding ignores unknown keys, so a
bridge-level `headers:` map, a `caching:` block, `endpoints[].targetPath` or `moduleRefs` all save
200/201 and then do nothing. (A key that *does* exist but has the wrong value type — e.g.
`defaultHeaders` written as a map — is rejected with 400 `invalid metadata field`.)

**Endpoint-level header secrets are not resolved on the activity-plan path.** `BRIDGE_CALL`,
`SCRIPT` and `LLM` enrichment resolve `${secrets.X}` in `defaultHeaders` only; a `${secrets.X}` in an
`endpoints[].headers` value is transmitted upstream as the literal placeholder. Keep credentials in
`defaultHeaders`. (Audit snapshots also redact the whole `endpoints[].headers` array, so
endpoint-header changes are invisible in the audit trail.)

## Agent access

A headless agent (an activity-plan `AGENT` step) may invoke a bridge only when **both** hold:

1. the bridge sets `agentCallable: true` — absent or `false` denies; and
2. the bridge is bound to the agent's project as a project resource of type `service-bridge`
   (see the `project-resource` skill).

Either gate missing yields a permission error naming which one failed. Human callers are unaffected
by both. Agents also cannot create, update or delete bridges, and their listings are narrowed to
project-bound bridges.

The project binding matters for one non-agent caller too: `serviceBridge.call()` / `.list()` inside
an activity-plan `SCRIPT` step resolve **only bridges bound to the running project**, so an unbound
bridge throws "not found" there — while a `BRIDGE_CALL` step in the same plan finds it org-wide.

## Egress and limits

Calls made through the platform proxy — and by activity-plan `BRIDGE_CALL` steps and `LLM`
enrichment — go out through an SSRF-fenced HTTP client: loopback, private, link-local (including the
cloud metadata address), multicast and unspecified addresses are refused, as are `localhost` and
known cluster-internal hostnames; DNS is pinned against rebinding, and every redirect hop is
re-validated (max 5). A bridge aimed at `localhost`, an RFC1918 address or a cluster-internal name
returns **HTTP 400 "service bridge egress denied: …"** from the proxy, not a 500 — point `baseUrl`
at a publicly resolvable host unless an operator has explicitly exempted it. (`serviceBridge.call()`
in a `SCRIPT` step is the one path that does not go through the fence, so a private `baseUrl` can
appear to work there and then fail everywhere else — don't take that as a green light.)

Limits are **path-dependent**, not one global timeout: 30 s per proxied call; 15 s per OAuth token
fetch; 10 s per JS hook script; `BRIDGE_CALL` steps default to 30 s and are capped at 120 s;
`serviceBridge.call()` inside a `SCRIPT` step is a fixed 10 s and is capped at 10 calls per script
run. Response bodies are read to 10 MB. Full matrix in `references/consumers.md`.

## Declared but inert

Persisted, round-tripped, visible in the UI — and read by nothing:

| Field | Note |
|---|---|
| `endpoints[].initScope` (`perCall` / `perDocument`) | No runtime reader anywhere. `initScript` runs on every proxy call unless the caller replays the `X-Bridge-Context` header it was given. |
| `endpoints[].responseSchema` | Nothing validates or reshapes a response against it. Documentation only. |
| `requestSchema` keywords other than `required` | `required` drives the guard above and is the only keyword with runtime effect. `properties.status.enum` is read solely by an activity-plan save-time check that is disabled by default and cannot match a working plan (`references/schema.md`). Field types, `minLength`, etc. are documentation. |
| `deleteProtection`, `checksum` | Accepted and echoed on a bridge; nothing enforces or verifies them. |

Not inert but worse: `headers[].secretRef` is refused with 400 at call time (see Credentials).

## References

- `references/schema.md` — every field, defaults, the `auth` block, server-side validation rules.
- `references/consumers.md` — the call paths and exactly which features each one honours.
- `references/examples.md` — four complete, applyable bridges.

**Related skills:** `activity-plan` (`BRIDGE_CALL` steps, `LLM` enrichment, `SCRIPT` bridge calls),
`data-form` (calling a bridge from a form), `project-resource` (binding the bridge to a project),
`kdx-cli` (`kdx apply`, `kdx sync`, `kdx secret`).
