# Intake — reference

Lookup companion to `SKILL.md`. Read `SKILL.md` first — the facts that fail silently live there.

## Wire format

`GET /api/intakes/{id}` returns the whole record. Authored files are a small subset of it.

```json
{
  "id": "00000000-0000-4000-8000-000000000001",
  "uuid": "00000000-0000-4000-8000-000000000002",
  "createdOn": "2026-05-18T00:00:00Z",
  "updatedOn": "2026-08-01T09:14:22Z",
  "changeSequence": 4,

  "organizationId": "00000000-0000-4000-8000-000000000003",
  "organization": { "id": "00000000-0000-4000-8000-000000000003" },

  "slug": "invoice-drop",
  "name": "Invoice Drop",
  "description": "Vendor invoices arriving by upload",
  "active": true,

  "store": {
    "id": "00000000-0000-4000-8000-000000000004",
    "slug": "invoices",
    "name": "Invoices",
    "ref": "acme-corp/invoices",
    "orgSlug": "acme-corp"
  },

  "activityPlanId": "00000000-0000-4000-8000-000000000005",
  "allowMultipleFiles": false,
  "largeUploadEnabled": false,
  "knowledgeFeatureIds": [],
  "metadata": { "channel": "upload" },

  "script": null,
  "scriptEnabled": false,

  "sourceType": "http",
  "sourceConfig": null,
  "sourceScript": null
}
```

There is no `type`, `ref`, `uri`, `orgSlug`, `publicAccess`, `template`, `deprecated`,
`extensionPackRef`, `deleted` or `yamlSource` on an intake — it is built on the plain entity base,
not the shared metadata envelope. `largeUploadEnabled`, `sourceConfig` and `sourceScript` are
`omitempty`, so an unset one is **absent** from the response rather than `null`.

## Fields

| Field | Type | Default | Notes |
|---|---|---|---|
| `slug` | string | derived from `name` | 3–100 chars, lower-cased, `[a-z0-9-]` only. Part of the public upload URL. **Not** checked for uniqueness: the generic check only applies to shared-envelope models, and the migrated schema has no unique index on `(organization_id, slug)` (databases carried over from the older Hibernate schema may still have one, and 409 there). |
| `name` | string | — | 1–255 chars. Required. |
| `description` | string | `""` | Free text. |
| `active` | bool | `false` | `false` ⇒ uploads 400; the poller also skips inactive non-HTTP intakes. |
| `store` | object | — | Document stores only. `{ref: "orgSlug/storeSlug"}` on create, `{id: "…"}` on update; a `ref` on update is ignored and an id-less object never clears the binding — only an explicit `null` does. **Not enforced at create**: a store-less intake saves, then 400s every upload. |
| `activityPlanId` | string \| null | `null` | Plan id, or `${activityPlan.<slug>}` in an authored file. No FK, no create-time validation. |
| `allowMultipleFiles` | bool | `false` | Allows more than one `file` part per request; a second file without it is a 400. |
| `largeUploadEnabled` | bool | `false` | Gates the presigned and multipart endpoints; without it they 403. |
| `knowledgeFeatureIds` | string[] | `[]` | Raw knowledge-feature ids attached to every uploaded family (provenance recorded as `intake:<slug>`). Not portable. |
| `metadata` | object | `{}` | Merged into each document family's metadata at lowest priority. |
| `script` | string \| null | `null` | Per-upload JavaScript. |
| `scriptEnabled` | bool | `false` | Must be `true` for `script` to run. |
| `sourceType` | enum | `http` | `http` \| `s3` \| `azure_blob` \| `email` \| `script`. |
| `sourceConfig` | object \| null | `null` | Per-source-type JSON. `null` for `http`. `${secrets.NAME}` refs are resolved by the scripted poller (whole config) and by the email poller (`credentials.refreshToken` only); the bucket pollers resolve none. |
| `sourceScript` | string \| null | `null` | Poll script for `sourceType: script` only. Different contract from `script`. |
| `organizationId` / `organization` | — | — | Server-stamped. Not client-writable on update, so an intake cannot be moved between orgs. |
| `changeSequence` | int | `0` | Optimistic lock. Send it and a stale value returns 409; omit it and the check is skipped. |

## Sync and filesystem

| Fact | Value |
|---|---|
| On-disk path | `<metadata_dir>/intakes/<slug>.yaml` |
| Manifest key / aliases | `intake`, `intakes` |
| Push order | 55 (after knowledge features 40, after activity plans 50) |
| Scope | organization |
| URI scheme | `intake://<orgSlug>/<slug>` |
| Soft delete | none — `DELETE` is permanent, and the resolver does no deleted-filtering |

`kdx sync pull` strips `id`, `uuid`, `organization`, `organizationId`, `createdOn`, `updatedOn`,
`changeSequence` at the top level and recursively drops any nested `id` that sits beside a `slug`
(so the nested `store` loses its id and keeps `ref` / `slug`). It then rewrites `activityPlanId` into
`${activityPlan.<slug>}`. It writes no `type:` and no `orgSlug:` key, which is why `kdx apply` on a
pulled file needs `--type intake --org-slug <org>`.

Push sends the YAML body verbatim plus `slug` and `organization: {id}`. Unknown keys such as `type`
and `orgSlug` are ignored by the server. `${org}` resolves only as a whole string value or as an
`${org}/` prefix — any other spelling survives substitution and **fails the push** with
"unresolved ${org} placeholders". `${orgSlug}` is not a token at all and would push as a literal.

## REST surface

Upload routes (`kit_` intake tokens allowed, 256 MB body cap):

| Method | Path |
|---|---|
| POST | `/api/intake/{orgSlug}/{intakeSlug}` |
| POST | `/api/intake/{orgSlug}/{intakeSlug}/presigned-upload-request` |
| POST | `/api/intake/{orgSlug}/{intakeSlug}/presigned-upload-complete` |
| POST | `/api/intake/{orgSlug}/{intakeSlug}/multipart-upload-request` |
| POST | `/api/intake/{orgSlug}/{intakeSlug}/multipart-upload-part-urls` |
| POST | `/api/intake/{orgSlug}/{intakeSlug}/multipart-upload-complete` |
| POST | `/api/intake/{orgSlug}/{intakeSlug}/multipart-upload-abort` |

Management routes (normal auth; `intake` permissions):

| Method | Path | Permission |
|---|---|---|
| GET / POST | `/api/intakes` | `read` / `create` |
| GET / PUT / DELETE | `/api/intakes/{id}` | `read` / `update` / `delete` |
| GET / POST | `/api/intakes/{id}/tokens` | `read` / `create` |
| DELETE | `/api/intakes/{id}/tokens/{tokenId}` | `delete` |
| GET | `/api/intakes/{id}/status` | `read` |
| POST | `/api/intakes/{id}/resume` | `edit` |
| POST | `/api/intakes/{id}/test-connection` | `edit` |
| POST | `/api/intakes/{id}/oauth/start` | `edit` |
| GET | `/api/intakes/oauth/callback` | none — the signed state token is the auth |
| GET | `/api/intakes/{id}/email/messages` | `read` |
| POST | `/api/intakes/{id}/email/messages/{messageId}/retry` | `edit` |
| POST | `/api/intakes/{id}/source-script/dry-run` | `edit` |
| GET | `/api/intakes/{id}/source-script/documents` | `read` |
| POST | `/api/intakes/{id}/source-script/documents/retry` | `edit` |

An upload by a normal user is authorised against the **document store**, not the intake: it needs
`upload` on the target store. An intake-scoped `kit_` token skips that check entirely — the token is
the authorisation — but is confined to `POST` under `/api/intake/` for its own intake.

## Upload parameters

Multipart form fields (each also accepted as a query parameter):

| Field | Meaning |
|---|---|
| `file` | The content. Repeatable only when `allowMultipleFiles: true`. |
| `path` | Destination path in the store. Defaults to the filename. Must not already exist. |
| `metadata` | JSON. Single object for one file; array matched by index for many (a single object is applied to all). |
| `documentVersion` | Written to the content object. |
| `externalData` | JSON injected into the stored document. |
| `labels` | Comma-separated; each is trimmed, upper-cased, and created in the org if new. |
| `statusId` | Document status id; a value that does not exist is warned about and skipped. |
| `knowledgeFeatures` | JSON array of `{id}` objects, appended to the intake's own list. |
| *anything else* | Folded into the **per-upload** metadata tier — it loses to a same-named key inside `metadata`, but still overrides the intake and KDDB tiers. |

Non-KDDB uploads are wrapped into a KDDB; the original bytes are also stored. Uploading a KDDB keeps
it and merges its metadata.

### Presigned flow

1. `presigned-upload-request?path=&contentType=&contentLength=` → `{uploadURL, s3Key, uploadID,
   expiresAt}`. The URL is valid for 15 minutes. `contentLength` must be a positive integer, and a
   colliding `path` is rejected here rather than later.
2. `PUT` the bytes to `uploadURL`.
3. `presigned-upload-complete?s3Key=&path=`, optionally with a JSON body of `metadata`,
   `extraFields`, `externalData`, `labels`, `statusId`, `knowledgeFeatures`, `documentVersion`,
   `filename`. This runs the same pipeline as a direct upload.

The multipart flow mirrors it: `multipart-upload-request` → `multipart-upload-part-urls` (body is a
JSON array of part numbers) → `multipart-upload-complete`, with `multipart-upload-abort` to discard.

## `sourceConfig` by source type

"Read" means the polling service actually consumes the key today. Consuming the key is not the
same as delivering a document: only `http` and `script` complete an upload — the `s3`,
`azure_blob` and `email` pollers run their lifecycle around a stub (see `SKILL.md`).

**`http`** — no config. `test-connection` answers *HTTP intake: ready to accept uploads*. For `s3`,
`azure_blob` and `email` it answers `ok: false` ("not yet implemented"); for `script` it checks that
`sourceScript` is non-empty and `allowedHosts` parses — the "script compiles" wording in its summary
overstates what runs, as no compile step is wired.

**`s3`**

| Key | Read | Note |
|---|---|---|
| `bucket`, `region` | yes | Both required. |
| `credentials.accessKeyId` / `.secretAccessKey` | yes | Optional; without them the ambient cloud credential chain is used. |
| `pollIntervalSeconds` | yes | Falls back to the service default. |

Bucket layout is fixed: `{orgSlug}/{intakeSlug}/inbox/` is watched, files move through
`processing/` to `completed/YYYY/MM/DD/` or `failed/YYYY/MM/DD/`. A `<file>.metadata.json` sidecar in
`inbox/` supplies per-file metadata; failures get a `<file>.error.json` sibling. Zero-byte files go
straight to `failed/` as `FILE_EMPTY`.

**`azure_blob`** — `container` and `credentials.connectionString` are required and read;
`storageAccount` is not read. Same folder lifecycle as `s3`.

**`email`**

| Key | Read | Note |
|---|---|---|
| `provider` | yes | `microsoft-graph` or `gmail-api`; anything else fails the cycle. |
| `credentials.refreshToken` | yes | Always a `${secrets.NAME}` ref in practice — the OAuth callback writes one. |
| `filter` | yes | Passed to the provider's message query. |
| `processing.attachmentsOnly` | yes | Default `true`. |
| `processing.includeEmailAsEml` | yes | Default `false`. |
| `processing.attachmentMimeTypes` | yes | Allowlist; empty means all. |
| `processing.minAttachmentBytes` | yes | Default `0`. |
| `pollIntervalSeconds` | yes | |
| `mailbox`, `tenantId` | no | `mailbox` is written by the OAuth callback for display; the tenant comes from server configuration. |

Each cycle reads from the newest terminal `received_at` minus a 5-minute overlap, pages 50 at a time,
and stops at 200 messages. Per-message dedup rows are defined for `completed`, `failed`,
`skipped_no_attachment`, `skipped_oversized` and `skipped_encrypted`, though only the first three are
written today. Any of them suppresses a re-fetch; a `failed` row keeps doing so until the retry
endpoint deletes it.

**`script`**

| Key | Read | Note |
|---|---|---|
| `allowedHosts` | yes | Required, non-empty. Anything not listed is denied by the egress guard. |
| `scriptTimeoutSeconds` | yes | Default 5 minutes, clamped to 15. |
| `pollIntervalSeconds` | yes | |
| `maxEmittedDocsPerCycle` | no | Caps are fixed: 1 000 soft (warn), 10 000 hard (abort), 5 GB per cycle. |
| `intakeToken` | no | The upload back to the API is currently unauthenticated over the internal network. |

`${secrets.NAME}` references anywhere in a scripted intake's `sourceConfig` are resolved before the
script runs, so `config` never contains the reference syntax.

## Upload-script contract (`script`)

Input globals: `filename` (the destination `path`, which defaults to the upload filename),
`fileSize`, `mimeType`, `metadata`, `labels`, `statusId`, `externalData`, `documentVersion`,
`document.text` (PDF only, first 5 pages), `document.pageCount`, `document.metadata` (KDDB uploads
only), `log.{debug,info,warn,error}`, `console.{log,warn,error}`, `springFilterQuote(value)` — which
returns a **complete** single-quoted literal, so concatenate it, do not re-quote it. 5-second
timeout, no module loading, no network.

Return keys:

| Key | Effect |
|---|---|
| `metadata` | Replaces the merged metadata map. Reserved keys (`uuid`, `version`, `labels`, `mixins`, `source`, `statusId`) abort the upload with a 500. |
| `reject` / `rejectReason` | 400 with `{"error":"SCRIPT_REJECTED","rejectReason":…}`; nothing is stored. |
| `activityPlan` | Bare slug or `activity-plan://org/slug`. Overrides `activityPlanId`. |
| `title`, `description`, `inputs`, `priority` | Passed to the started activity. `inputs` is validated against the plan's input options. |
| `skipActivityPlan` | `true` stores the file and starts nothing. Must be boolean. Combining it with `activityPlan` is `INTAKE_SCRIPT_CONFLICT`. |
| `documentFamilyFilter` | SpringFilter expression evaluated over the current multi-file request's families only. Read only alongside `activityPlan`. |
| `documentFamilyFilterLimit` | Positive integer; default 500. A non-integer, zero or negative value is rejected inside the script parser, which surfaces as a **500** — not the `INTAKE_FILTER_LIMIT_INVALID` 400. |
| `taskTemplates[]` | Deprecated. `{slug, priority, teamSlug, assigneeEmail, title, description, metadata, properties}`. Mutually exclusive with `activityPlan` (the plan wins). |

In a multi-file request every file is stored first and activity starts run afterwards, so filter
grouping does not depend on part order. Mixing scripted and static plan starts in one request is
allowed but logs a cohort-overlap warning.

## Poll-script contract (`sourceScript`)

The script body is executed and then `poll()` is called, so declare `function poll() { … }`.

Globals: `config` (resolved `sourceConfig`), `state` (previous cycle's returned object, `null` on the
first run), `secrets.get(name)`, `log.{debug,info,warn,error}(msg, fields?)`,
`http.{get,post,put,delete,head}(url, opts?)` and `emit(doc)`.

`http` options: `headers`, `body` (string, bytes, or any object — JSON-encoded), `timeoutMs`,
`responseType` (`text` default, `json`, `bytes`). The response is `{status, headers, body}`; a denied
host throws, which the script can catch. Every call is traced.

`emit({filename, bytes, contentType?, metadata?, externalData?, path?, scriptDocumentId?})` queues a
document; `filename` and `bytes` are required. `contentType` is captured but never sent on — the
upload endpoint derives the MIME type from the path. Queued documents are uploaded through the ordinary
intake endpoint after `poll()` returns, so scripts, metadata merging and activity starts all apply.

`poll()` returns the next cycle's `state` object (≤ 256 KB serialised). Cycles are atomic: state and
dedup rows persist only if `poll()` returns cleanly **and** every upload succeeded. A 4xx from the
intake API aborts the cycle and leaves state untouched, so the next cycle repeats the work.

## Rejection reasons

| Reason | Cause |
|---|---|
| `INTAKE_PROJECT_BINDING_MISSING` | Raised at two sites: the plan and the intake's document store share no common project, or the activity start's own check finds the plan unbound from the resolved project. |
| `INTAKE_PROJECT_BINDING_AMBIGUOUS` | They are bound together in more than one project. |
| `INTAKE_PLAN_NOT_FOUND` | `activityPlanId` points at nothing in this org, or the script named a plan that does not resolve. |
| `INTAKE_PLAN_INVALID` | The plan's steps failed validation while being materialised. |
| `INTAKE_PLAN_INPUTS_INVALID` | Script `inputs` violated the plan's declared input options. |
| `INTAKE_REQUIRED_GROUP_UNSATISFIED` | The plan marks a document group required and no in-scope family resolved. |
| `INTAKE_TEMPLATE_RENDER_FAILED` | The plan's default title or description template failed to render. |
| `INTAKE_SCRIPT_CONFLICT` | The script returned both `activityPlan` and `skipActivityPlan: true`. |
| `INTAKE_FILTER_INVALID` | `documentFamilyFilter` does not parse, or names an unknown field. |
| `INTAKE_FILTER_NO_MATCH` | The filter matched none of the files in this request. |
| `INTAKE_FILTER_OVER_LIMIT` | It matched more than the limit — the run is refused rather than truncated. |
| `INTAKE_FILTER_LIMIT_INVALID` | A non-positive `documentFamilyFilterLimit` reached the activity start. Unreachable from an intake script today — the script parser rejects such values first (as a 500). |
| `INTAKE_FILTER_UNSUPPORTED_PRESIGNED` | A filter was used on a presigned or presigned-multipart upload. |
| `INTAKE_ACTIVITY_START_REJECTED` | Catch-all for any other deterministic 4xx out of the activity start. |

Failures without an `INTAKE_*` reason — inactive intake, missing store, duplicate path, permission
denial — are ordinary errors that emit no rejection event. A script `reject` is its own case: a 400
carrying `{"error":"SCRIPT_REJECTED"}` plus an `intake.rejected` event.

## Instrumentation

`intake.upload_completed` (per stored family; carries `activityStarted`), `intake.upload_rejected`
(exactly once per `INTAKE_*` rejection, with reason, status, plan/project/family context and whether
storage cleanup completed), `intake.rejected` (script `reject`), plus
`intake.document_family_insert_failed`, `intake.content_object_insert_failed`,
`intake.s3_upload_failed` and `intake.kddb_wrap_failed` for internal failures.
