# Service Bridge — Field Reference

All keys sit at the **top level** of the YAML file. A nested `metadata:` block is still accepted on
write, but responses and `kdx sync pull` are flat.

## Envelope

| Field | Notes |
|---|---|
| `type` | `service-bridge` — the canonical discriminator, and what the server stamps when `type` is absent. `serviceBridge` is a legacy alias: `kdx` still routes it, but it persists a non-canonical discriminator and makes the response's computed `uri` (`serviceBridge://…`) unresolvable, since the resolver only knows the `service-bridge` scheme. |
| `slug` | Unique within the organization. Addressed as `service-bridge://<orgSlug>/<slug>`. |
| `orgSlug` | What `kdx apply` uses to pick the target organization (or `--org-slug`). The server treats it as a computed echo field — the persisted link is the organization id. `kdx sync` rewrites it to the portable `${org}` placeholder on pull. |
| `name` | Display name. |
| `description` | Display description. |
| `publicAccess`, `deprecated`, `template` | Standard envelope booleans, all defaulting to `false`. |
| `icon`, `imageUrl`, `overviewMarkdown`, `provider`, `providerUrl`, `providerImageUrl` | Display-only. |
| `extensionPackRef` | Set when the bridge arrived in an extension pack. |

A service bridge has **no `version` field** — the envelope key order includes one for other resource
types, but on a bridge it is an unknown key and is dropped on write (see "Keys the model does not
have" in `SKILL.md`).

Canonical key order that `kdx` writes: the envelope keys above, then `baseUrl`, `auth`,
`defaultHeaders`, `endpoints`. Keys outside that list (`orgSlug`, `agentCallable`, `cacheableVerbs`)
follow. Ordering is cosmetic — nothing depends on it.

## Bridge

| Field | Type | Notes |
|---|---|---|
| `baseUrl` | string | Base URL for every endpoint. Supports `${secrets.NAME}`. Must resolve to a public host — see the egress fence in `SKILL.md`. |
| `agentCallable` | bool | Default `false`. Lets a headless activity-plan agent invoke this bridge — **and** the bridge must be bound to the agent's project. Anyone with service-bridge write permission can set it. |
| `defaultHeaders` | list of Header | Applied to every endpoint. The only header collection whose `${secrets.X}` values resolve on *all* call paths. |
| `endpoints` | list of Endpoint | See below. |
| `auth` | object | Optional platform-managed OAuth2 — see below. |
| `deleteProtection`, `checksum` | bool / string | Accepted and echoed; nothing reads them. |

## Endpoint

| Field | Type | Default | Notes |
|---|---|---|---|
| `name` | string | — | **Required.** Unique in the bridge. The URL segment callers address and the identifier every consumer stores. Treat as immutable. |
| `description` | string | — | Surfaced in bridge listings, including to agents choosing an endpoint. |
| `path` | string | — | **Required.** Appended to `baseUrl` by plain concatenation — **must start with `/`**. `${secrets.X}` is *not* resolved here. |
| `method` | string | — | **Required.** The verb sent **upstream**, whatever verb the caller used. Accepted: `GET POST PUT DELETE PATCH HEAD OPTIONS` (case-insensitive). The proxy drops the request body when this is `GET`. |
| `requestSchema` | JSON Schema map | — | Only two keywords are read: `required` (the short-circuit guard) and `properties.status.enum` (see Validation). |
| `responseSchema` | JSON Schema map | — | Persisted only; nothing validates against it. |
| `headers` | list of Header | — | Layered over `defaultHeaders` for this endpoint. `${secrets.X}` here resolves on the proxy path only, and the whole array is redacted from audit snapshots. |
| `cacheEnabled` | bool | `false` | Enables response caching for this endpoint on the proxy path. |
| `cacheTtlSeconds` | int | `0` → 300 s | TTL for a cached response. |
| `cacheableVerbs` | list of string | `["GET"]` | Which declared methods are cache-eligible. Set `[GET, POST]` to cache a POST lookup. Matched against the endpoint's `method`, not the caller's verb. |
| `initScript` | JS string | — | Proxy path only. See Hooks. |
| `preSendScript` | JS string | — | Proxy path only. See Hooks. |
| `postReplyScript` | JS string | — | Proxy path only, **and only when the endpoint is cache-eligible**. See Hooks. |
| `initScope` | string | — | `perCall` / `perDocument` in the UI dropdown; no runtime reader exists. Inert. |

### Caching semantics

- The key is `sha256(endpointName ‖ raw query string ‖ request body)` under a visible
  `org : bridge id : bridge change sequence` prefix. **Request headers are not part of the key.**
- Because the bridge's change sequence is in the prefix, **editing the bridge implicitly invalidates
  every cached response for it**.
- Only **2xx** responses are stored. Bodies over 10 MB are never cached.
- Concurrent identical cacheable misses are coalesced into a single upstream call; all waiters share
  the result.
- Caching assumes a **user-agnostic** upstream (a shared service account). Never enable it on an
  endpoint whose response depends on the calling user.
- Non-deterministic `preSendScript` output changes the key on every call, guaranteeing misses.

## Header

| Field | Type | Notes |
|---|---|---|
| `name` | string | **Required.** |
| `value` | string | The literal value. `${secrets.NAME}` is interpolated here. |
| `secretRef` | string | **Do not use.** Present in the schema and the UI, but a header carrying it makes the call fail with HTTP 400 on the proxy and error out on the activity-plan path. It is never injected. |

Merge order on the proxy path, later wins: the caller's `Content-Type` → `defaultHeaders` →
`endpoints[].headers` → the OAuth token header → headers returned by `initScript` → headers returned
by `preSendScript`. On the activity-plan path the order is `defaultHeaders` → `endpoints[].headers`
→ the step's per-call `requestHeaders` → the OAuth token header (so the token still wins, and a
`requestHeaders` value containing CR, LF or NUL aborts the call).

## Auth — platform-managed OAuth2

```yaml
auth:
  type: oauth2_client_credentials     # the only value that does anything
  tokenUrl: /oauth/token              # absolute, or a leading "/" => resolved against baseUrl
  clientId: "${secrets.VENDOR_CLIENT_ID}"
  clientSecret: "${secrets.VENDOR_CLIENT_SECRET}"
  # requestFormat: standard           # default: form-encoded grant_type=client_credentials
  # requestFormat: custom             # JSON POST built from requestBody
  # requestBody:                      #   top-level string values support
  #   grant_type: client_credentials  #   ${auth.clientId} / ${auth.clientSecret}
  #   key: "${auth.clientId}"
  # responseMapping:                  # only for non-standard token responses
  #   accessToken: token              #   default "access_token"
  #   expiresIn: ttl                  #   default "expires_in"
  # tokenCacheBufferSeconds: 60       # default 60
  # headerName: Authorization         # default
  # headerPrefix: Bearer              # default, space-joined with the token
```

`type`, `tokenUrl`, `clientId` and `clientSecret` are required when `auth` is present.

Behaviour: the token endpoint must answer **HTTP 200**; the response is read to 1 MB. A missing
`expires_in` is treated as **3600**. Cached TTL is `expiresIn - tokenCacheBufferSeconds`, floored at
30 s, keyed per bridge. On the proxy path a **401 or 403 from the upstream evicts the token and
retries once** with a fresh one — but only on a cache-eligible endpoint (see `SKILL.md`).

**Any `auth.type` other than `oauth2_client_credentials` is silently inert** — no token is fetched
and no header is injected.

## Hooks

Three optional JS snippets run by the platform around a **proxy-path** call. Each gets a fresh VM
with a **10 s** cap, is wrapped in an IIFE that receives a global `input`, and must `return` an
object (returning nothing/`null` means "no changes"). The only other globals are `log` and
`console`; there is no HTTP, bridge-calling, LLM or module-loading facility.

| Hook | `input` | May return |
|---|---|---|
| `initScript` | `{headers, config: {baseUrl, endpointName, method, path}}` | `{headers, context}` |
| `preSendScript` | `{headers, body?, context?}` | `{headers, body}` — a returned body replaces the outbound body and feeds the cache key |
| `postReplyScript` | `{body, statusCode}` | `{body, statusCode}` — a returned body forces `Content-Type: application/json`; a **string** body is used verbatim (no double-encoding); a returned `statusCode` overrides the status the caller sees |

`initScript` / `X-Bridge-Context` handshake: when a caller sends no `X-Bridge-Context` request
header, `initScript` runs and any `context` it returns is echoed back in the `X-Bridge-Context`
**response** header. If the caller replays that header on the next call, `initScript` is skipped and
the decoded context is injected into `preSendScript` as `input.context`. The response header is not
set on a cache hit.

## Server-side structural validation

An environment can enable structural validation on bridge create/update. It is **disabled by
default**; in shadow mode findings are logged and emitted, in enforced mode they return 400. Rule
codes:

| Code | Checks |
|---|---|
| `service-bridge.endpoint.name-required` | every endpoint has a non-blank `name` |
| `service-bridge.endpoint.name-unique` | no duplicate endpoint names |
| `service-bridge.endpoint.path-required` | every endpoint has a non-blank `path` |
| `service-bridge.endpoint.method-known` | `method` is one of the seven accepted verbs |
| `service-bridge.endpoint.status-enum-wellformed` | if `requestSchema.properties.status.enum` is declared, it is a non-empty array of non-empty strings |

That last one exists so activity-plan validation can reject a `BRIDGE_CALL` whose literal
`requestBody.status` is outside the endpoint's declared enum. Treat that enum as a **declaration,
not an enforced contract**: the consuming rule compares the raw `requestBody.status` string, but a
`BRIDGE_CALL` `requestBody` value is JSONata, so the form that actually works at run time is
`"'submitted'"` — which the comparison sees complete with its quotes and never matches an entry.
Both validators are off unless the environment opts in, so neither blocks a save by default.

Note what is **not** validated: a `path` without a leading `/`, a header carrying both `value` and
`secretRef`, an unsupported `auth.type`, and an unreachable `baseUrl` all save cleanly.

## Reference appendix

- Table `kdxa_service_bridges`, unique on `(organization_id, slug)`. Soft delete appends a UUID
  suffix to the slug.
- Resolve a slug to an id with `POST /api/resolve?path=service-bridge://<orgSlug>/<slug>`, which
  returns `{"path": "/api/service-bridges/{id}"}`.
- Bridge metadata is cached briefly on the serving side but revalidated against the row's change
  sequence on every call, so an edit takes effect on the next call rather than after a TTL.
- Audit snapshots redact `auth.clientSecret`, `auth.requestBody`, `defaultHeaders[].value`,
  `defaultHeaders[].secretRef`, and the entire `endpoints[].headers` array.
