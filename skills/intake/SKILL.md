---
name: intake
description: "Use when documents need a way into Kodexa — authoring intakes/*.yaml, pointing an intake at a document store, wiring it to an activity plan, writing the Goja upload script, issuing upload tokens, choosing a sourceType, or diagnosing an INTAKE_* upload rejection."
---

# Kodexa Intake Authoring

An **intake** is an ingestion endpoint: it names a **document store**, optionally an **activity plan**
to start per upload, and optionally a **script** that inspects each file before it lands. Intakes are
org-scoped rows in `kdxa_intakes`, addressed as `intake://<orgSlug>/<intakeSlug>`; the upload URL is
built from those two slugs, so **the slug is public surface** — anything holding an upload token
knows it.

## The file

One YAML per intake at `intakes/<slug>.yaml`. Pushed at tier **55** — after knowledge features (40)
and after activity plans (50), because the plan reference is resolved against the destination.

```yaml
# intakes/invoice-drop.yaml
type: intake                    # for `kdx apply`; the server ignores it
slug: invoice-drop              # required, 3–100 chars
name: Invoice Drop              # required, 1–255
orgSlug: acme-corp              # for `kdx apply`; the server ignores it
description: Vendor invoices arriving by upload
active: true                    # DEFAULTS TO false; false ⇒ every upload 400s "not active"
store:
  ref: "${org}/invoices"        # ALWAYS a document store — see below
activityPlanId: "${activityPlan.invoice-posting}"
allowMultipleFiles: false
largeUploadEnabled: false
knowledgeFeatureIds: []
metadata: {}
script: null
scriptEnabled: false
sourceType: http
```

An intake is **not** built on the shared metadata envelope: no `publicAccess`, `template`,
`deprecated`, `extensionPackRef`, `ref`, `uri`, no soft delete — `DELETE` removes the row outright.
`type` and `orgSlug` are CLI-side keys only; the API ignores both and `kdx sync pull` writes neither,
so applying a pulled file needs `kdx apply -f … --type intake --org-slug acme-corp`. Omit `slug` and
the server derives it from `name`; supply one and it is lower-cased and must be lowercase letters,
digits and hyphens. `organizationId` is stamped by `kdx` and is not client-writable on update.

**Nothing rejects a duplicate slug.** The create-time uniqueness check only fires for resources built
on the shared envelope, so it silently skips intakes, and the migrated schema carries no unique index
on `(organization_id, slug)`. Two intakes can share a slug in one org; the upload lookup then picks
one arbitrarily and `kdx sync` resolves the other. (Databases carried over from the older Hibernate
schema may still hold such an index and 409 instead — rely on neither.)

## The store binding is create-only through `ref`

`store` must be a **document store**, and **nothing enforces it at create**: a store-less intake
saves fine, then 400s every upload with *intake has no document store configured*.

- **On create**, `store: {ref: "orgSlug/storeSlug"}` is resolved server-side. A ref matching no live
  document store fails the create with *cannot resolve store.ref … store not found*. Write it
  `${org}/invoices` for portability: pull rewrites the source org slug to that token and push restores
  it — but only as a whole value or an `${org}/` prefix; any other use is a hard push error.
- **On update, `ref` is silently ignored** — the update path writes the store column only for an
  explicit `store: null` or an object carrying a non-empty `id`, so pushing a changed `ref` reports
  success and leaves the old store bound. Repoint with `store: {id: "<destination store id>"}`, or
  recreate. `store: null` clears the binding, and every upload then 400s as above.

A pulled intake nests the whole store record under `store:`; keep `ref` and `slug`, the rest is noise.

## Wiring to an activity plan

`activityPlanId` is a **raw plan id** with no foreign key and no create-time validation, so authored
files use the sigil: `activityPlanId: "${activityPlan.invoice-posting}"`. Pull rewrites a stored id
into that sigil; push resolves it against the destination org. If the plan genuinely does not exist
there the field is **dropped** (warning only); a transient resolve failure leaves the sigil in place,
which the server stores verbatim and every upload then rejects with `INTAKE_PLAN_NOT_FOUND`.

**The plan and the intake's store must both be bound to exactly one common project.** The upload path
intersects the two `project-resource` binding sets: exactly one project ⇒ the activity starts there;
an empty intersection ⇒ 400 `INTAKE_PROJECT_BINDING_MISSING`; two or more ⇒ 400
`INTAKE_PROJECT_BINDING_AMBIGUOUS`. (A second check inside the activity start re-asserts the plan's
binding to the resolved project and raises the same `…MISSING` reason.) Both roll the whole upload
back — **the document is not stored** — and this is the commonest way a working-looking intake
silently refuses every file. Bind both with the manifest `linked:` block (see `project-resource`).

## Uploading

`POST /api/intake/{orgSlug}/{intakeSlug}` — `multipart/form-data`.

Form fields: `file` (repeatable when `allowMultipleFiles: true`), `path` (defaults to the filename),
`metadata` (JSON — an object, or an array matched to files by index), plus `documentVersion`,
`externalData`, `labels` (comma-separated, upper-cased), `statusId`, `knowledgeFeatures`. Family
metadata merges **intake `metadata` < metadata from an uploaded KDDB < per-upload `metadata`**, and
those five named keys are then deleted — putting any of them in the intake's `metadata` block is a
no-op. Every **other** form field joins the *per-upload* tier: it loses to a `metadata` key of the
same name but still overrides intake and KDDB metadata. It is not the lowest tier.

Two limits to design around: uploading to a `path` that already exists in that store is a 400, and
the request-body cap on these routes is **256 MB**. Above that set `largeUploadEnabled: true` and use
the presigned or multipart endpoints. Presigned completions handle one file per request, which is why
a script `documentFamilyFilter` is refused there with `INTAKE_FILTER_UNSUPPORTED_PRESIGNED`.

**Tokens.** `POST /api/intakes/{id}/tokens` mints a `kit_…` value, returned **once**, stored hashed;
send it as `X-API-Key`. Such a token may only `POST` under `/api/intake/` and only for its own intake
— anything else 403s. A normal user token instead needs `upload` on the target **document store**.

## The upload script

`script` is JavaScript run in a sandboxed Goja VM, only when `scriptEnabled: true` — both fields must
be set. Timeout is **5 seconds**; there is no `require`, no `fetch`, no network.

Globals in: `filename` (really the destination `path`, which defaults to the upload filename),
`fileSize`, `mimeType`, `metadata` (the merged map), `labels`, `statusId`, `externalData`,
`documentVersion`, `document.{text,pageCount,metadata}`, `log.*`, `console.*`, `springFilterQuote()`.
`document.text` is PDF-only (first 5 pages); `document.metadata` is populated only for a KDDB upload.

The body is wrapped in a function, so you `return` an object:

```js
if (fileSize === 0) return { reject: true, rejectReason: "empty file" };
metadata.vendor = filename.split("_")[0];
if (!/pack\.pdf$/.test(filename)) return { metadata: metadata, skipActivityPlan: true };
return {
  metadata: metadata,
  activityPlan: "invoice-posting",                       // bare slug or activity-plan:// URI
  inputs: { vendor: metadata.vendor },
  documentFamilyFilter: "path~" + springFilterQuote(metadata.vendor + "*"), // needs multi-file
  documentFamilyFilterLimit: 8
};
```

- `reject: true` fails the upload with a `SCRIPT_REJECTED` 400 body and stores nothing.
- `activityPlan` overrides `activityPlanId`; `skipActivityPlan: true` stores the file and starts
  nothing; returning **both** is a 400 with `INTAKE_SCRIPT_CONFLICT`. Returning neither (or nothing
  at all) falls back to the static `activityPlanId`.
- `documentFamilyFilter` groups one activity over several files of the **same request**; read only
  alongside `activityPlan`, and grouping across files also needs `allowMultipleFiles: true`. Wrap
  interpolated filenames in `springFilterQuote` (it returns a complete quoted literal): a raw
  apostrophe breaks the filter and a crafted filename can inject predicates. Matching none of this
  request's files is `INTAKE_FILTER_NO_MATCH` and kills the upload — even a single-file intake must
  match its one file; over the limit (default 500) is `INTAKE_FILTER_OVER_LIMIT`. Neither truncates.
  A non-positive or non-integer limit fails the script with a **500**, not `…FILTER_LIMIT_INVALID`.
- Returning `metadata` with `uuid`, `version`, `labels`, `mixins`, `source` or `statusId` aborts the
  upload with a **500** — those back structural document fields. Rename yours (`documentSource`).
- `taskTemplates: [...]` is the deprecated pre-activity-plan contract. Still parsed, still creates
  tasks; do not write new scripts against it.

## Rejection reasons

Rejections carry a stable `details.reason`, are emitted as `intake.upload_rejected`, and any objects
the request already wrote are best-effort deleted. `references/reference.md` maps each to its cause.
`INTAKE_PROJECT_BINDING_MISSING` · `INTAKE_PROJECT_BINDING_AMBIGUOUS` · `INTAKE_PLAN_NOT_FOUND` ·
`INTAKE_PLAN_INVALID` · `INTAKE_PLAN_INPUTS_INVALID` · `INTAKE_REQUIRED_GROUP_UNSATISFIED` ·
`INTAKE_TEMPLATE_RENDER_FAILED` · `INTAKE_SCRIPT_CONFLICT` · `INTAKE_FILTER_INVALID` ·
`INTAKE_FILTER_NO_MATCH` · `INTAKE_FILTER_OVER_LIMIT` · `INTAKE_FILTER_LIMIT_INVALID` ·
`INTAKE_FILTER_UNSUPPORTED_PRESIGNED` · `INTAKE_ACTIVITY_START_REJECTED` (catch-all)

## Source types: what actually runs

`sourceType` is one of `http`, `s3`, `azure_blob`, `email`, `script`. **`http` and `script` are the
two that deliver documents end to end today.** The bucket and email pollers fetch and shuffle files
but their upload step is still a stub that fails *silently forward*: `s3`/`azure_blob` move the file
on into `completed/` as though it had landed, and `email` writes a `completed` dedup row that stops
the message ever being re-fetched. `test-connection` answers `ok: false` for both. The editor locks
`sourceType` and `slug` after create; **the API does not** — a changed slug moves the upload URL.

`sourceConfig` is free-form JSON whose shape depends on `sourceType` (per-type keys, and which are
actually read, in `references/reference.md`). It is `null` for `http`. **Never put a raw credential
in it** — write `"${secrets.NAME}"`. Only the scripted poller resolves those across the whole config;
the email poller resolves `credentials.refreshToken` alone and the bucket pollers resolve none, so a
`${secrets.…}` bucket key is sent verbatim. The email OAuth flow **rewrites `sourceConfig`
server-side** (`mailbox` + a `credentials.refreshToken` ref), so a later push from an older file wins.

## Before you push

`kdx sync push --dry-run` compares against the destination and, on a capable v2 server, asks it to
validate without persisting. `kdx validate -f` checks the file against the OpenAPI schema, but it
validates the nested `store` against the full store schema, so the portable `store: {ref: …}` form
reports a dozen missing-required **errors that are not real** (`id`, `uuid`, `createdOn`,
`publicAccess`, …); `type` and `orgSlug` "key not found in schema" warnings are likewise expected. `POST /api/intakes/{id}/test-connection` is the live probe; the scripted
**dry-run** endpoint runs static checks only, then returns `503 DRY_RUN_NOT_IMPLEMENTED`.

## Declared but inert

| Field / surface | Note |
|---|---|
| `sourceScript` when `sourceType != script` | Stored, never executed. |
| `sourceConfig.maxEmittedDocsPerCycle` | Never read; caps are fixed at 1 000 soft / 10 000 hard docs and 5 GB per cycle. |
| `sourceConfig.intakeToken` | Never read — the scripted poller uploads without the header. |
| `sourceConfig.storageAccount`, `.tenantId`, `.mailbox` | Never read by the poller; the Azure client uses `credentials.connectionString`, the tenant comes from server configuration, and `mailbox` exists for display. |
| `emit({ scriptDocumentId })` dedup rows | Written and listable, but nothing consults them, so a re-emitted document is re-uploaded. |
| `POST /api/intakes/{id}/source-script/documents/retry` | Only matches rows with status `failed`, and nothing ever writes that status — it always 404s. |
| `GET /api/intakes/{id}/status` | `lastPoll`, `consecutiveFailures` and `warnings` are always null/0/empty; `isDegraded`/`degradedReason` read a `sourceConfig.degraded` marker that nothing ever writes, so `POST .../resume` is a no-op. |
| `emit({ contentType })` | Captured but never sent to the upload endpoint; the MIME type is derived from the path. |
| `test-connection` on a scripted intake | Reports "script compiles"; only presence of `sourceScript` and the `allowedHosts` parse are actually checked. |
| `knowledgeFeatureIds` (*not* inert, but not portable) | Raw feature ids, never rewritten to slugs on pull. Ids from another environment fail to attach, and the failure is logged rather than returned to the uploader. |

## Related

`activity-plan` (what `activityPlanId` points at) · `project-resource` (the bindings that make the
plan startable) · `kdx-cli` (push order, `apply` vs `sync`). Details: `references/reference.md`.
