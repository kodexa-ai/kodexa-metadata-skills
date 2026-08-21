---
name: metadata-envelope
description: "Use when authoring or debugging any Kodexa resource YAML — the shared slug / name / orgSlug / type envelope twelve org-scoped resource types embed, the flatten rule that makes authored YAML flat, what a slug is actually checked against, the changeSequence optimistic lock, the computed ref and uri, and where each file lives in a metadata repo."
---

# Kodexa Metadata Envelope

Twelve org-scoped resource types share one envelope: the same identity keys, the same lifecycle
columns, the same serialization rules. Sibling skills describe a type's own fields; this one covers
the frame they sit in, and the four parts of it that silently break authored YAML — the flatten
rule, the slug rules, `type`, and `changeSequence`.

**Envelope types:** `data-definition`, `data-form`, `document-store`, `data-store`, `module`,
`prompt-template`, `service-bridge`, `activity-plan`, `project-template`, `task-template`,
`task-status`, `knowledge-set`. (`model-runtime` shares it, but is platform-supplied.)

**Not envelope types**, though they look alike: `intake`, `project`, `assistant`, `trigger`,
`workspace`, `knowledge-item-type`, `knowledge-feature-type`, `knowledge-feature`,
`knowledge-item`, `label`. Each carries its own `slug` and `name`, and none emits a computed `ref`
or `uri`. The four knowledge-family ones *do* have an `orgSlug`, but it runs the other way round —
an input you may send in place of an organization id, never filled in on read. `label` has no slug
column at all; a `label://` URI resolves on `name`. `changeSequence`, `id`, `uuid`, `createdOn` and
`updatedOn` are on *every* resource. Full field table: `references/fields.md`.

## The keys

```yaml
type: data-definition            # two different consumers read this — see below
slug: invoice-taxonomy           # identity within the organization
name: "Invoice Taxonomy"         # display name; slugified into `slug` if you omit `slug`
orgSlug: acme-corp               # read by kdx; the server ignores it
publicAccess: false
template: false
deprecated: false
# ...the resource's own fields, FLAT, at this same level
```

| Key | Reality |
|---|---|
| `slug`, `name` | The two you own. See **Slugs**. |
| `orgSlug` | A computed echo of the owning org. `kdx apply` reads it to pick the target org; the **server ignores what you send** for every envelope type except `knowledge-set`, which resolves it to an organization when no id is supplied. `kdx sync pull` strips it — the manifest supplies the org. |
| `organizationId` / `organization: {id}` | The real FK, on create — send the org's **id**, never its slug. Most types let the **nested form win** when both are present, deliberately, so a client that puts a slug in `organizationId` alongside a correct nested id still lands right; `module` and `model-runtime` are the exception and take the flat field. On update both are ignored and the stored org restored, so a PUT can never move a resource between organizations. |
| `publicAccess` | Stored, returned, sortable — and it grants nothing (see **Declared but inert**). |
| `template`, `deprecated` | Not inert — the Studio resource picker reads both. `deprecated: true` filters a resource out unless "include deprecated" is on, and badges it when shown. `template: true` marks a resource as a starting point: the picker offers `kodexa`-org templates alongside your own and sorts them first, and choosing a `template: true` row **copies** the resource into your org under a name and slug you supply instead of binding the original. Nothing else branches on either. |
| `extensionPackRef`, `yamlSource` | Columns with no producer and no consumer. |
| `ref`, `uri` | Computed, read-only. See **ref and uri**. |
| `changeSequence` | Server-owned optimistic lock. See below. |
| `deleted`, `deletedDate`, `deleteUserEmail`, `deleteUserId` | Zeroed out of every create and update body before the write — you cannot delete or undelete through a normal save. `DELETE` sets them and **appends `-<uuid>` to the slug**, freeing the slug for immediate reuse; the old row is then unreachable by its old slug. |
| `id`, `uuid`, `createdOn`, `updatedOn` | Server-owned; never writable through an update. (`searchText` is a server-computed column that never appears on the wire at all.) |

## The flatten rule — why authored YAML is flat

**Eight** of the twelve keep their own fields in a nested `metadata` object and **delete that key on
the way out**: the object's fields are merged up to the top level and the enclosing `metadata` key
is removed. Wire shape and storage shape are not the same shape.

That is why you write `taxons:`, `nodes:`, `baseUrl:`, `promptTemplate:` at the top level of a file
rather than under a `metadata:` heading. Flat is not a convenience; it is the only shape served back.

- **Applies to** `data-definition`, `data-form`, `document-store`, `data-store`, `module`,
  `prompt-template`, `service-bridge`, `project-template`. **The other four do not flatten.**
  `task-template` keeps its real body under a nested `metadata:` and must be authored nested;
  `activity-plan` and `task-status` carry their own fields as top-level keys already with a
  free-form `metadata:` blob beside them; `knowledge-set` has no metadata object at all.
- **The outer envelope wins a name collision — when it has a value to win with.** `id`, `slug`,
  `name`, `publicAccess`, `template` and `deprecated` are always serialized, so a blob's copies of
  those are always shadowed. `type`, `ref`, `orgSlug` and `extensionPackRef` are dropped when empty,
  and then the blob's copy shows through. A project template's blob carries its own copy of all ten.
- **Writing is the mirror: one flat body is decoded twice**, into the columns *and* into the
  metadata blob. A key that names both — `type` on a module or a store is the common one — lands in
  both places. You only ever read the envelope copy back, so the blob copy drifts unseen.
- **A body carrying any metadata field replaces the whole blob.** There is no field-level merge: a
  PUT of `{name, slug, baseUrl}` on a service bridge rewrites its metadata to *only* `baseUrl`, and
  every other metadata field reverts to zero. Always send the complete resource. (A PUT of envelope
  keys **only** — `{"name": "New name"}` — leaves the blob untouched.)
- **`document-store` and `data-store` nest twice.** Their inner content object surfaces both
  flattened *and* intact under a `contentMetadata` key, which exists so round-trip writes do not
  lose it. Do not hand-author it.
- A field whose value is the wrong JSON type **fails the whole write with a 400** rather than being
  skipped — `restrictClassification: "true"` where a boolean is expected rejects the request.

## Slugs

`slug` is the resource's identity inside its organization and the second half of every
`orgSlug/slug` ref and `scheme://orgSlug/slug` URI pointing at it. What runs on one you supply:

- **Format** — `^[a-z0-9-]+$`. Mixed case is silently lowercased, not rejected. An underscore, dot,
  space or slash is a `400 invalid slug "…"`. **Uniqueness** — `(organization, slug)` among
  non-deleted rows; a collision is a `409`.
- **Length — nothing checks it by default.** An optional cross-cutting rule set, off unless a
  deployment turns it on, caps a slug at **200** characters and additionally forbids a leading or
  trailing hyphen. With it off, format and uniqueness are the only checks a hand-written slug faces
  — no minimum, no other maximum.

Omit `slug` and the server derives one from `name`: lowercase, every run of non-`[a-z0-9]` collapsed
to a single `-`, leading and trailing `-` trimmed, truncated at 200. **A name that normalizes to
fewer than 3 characters is a hard 400** — `"AI"`, `"#3"` and `"!!!"` all fail with *name … does not
produce a valid slug*. Over-long names truncate instead of failing.

- **Renaming a slug is not cosmetic.** An explicit slug in an update body renames the resource;
  every `scheme://org/old-slug` reference then resolves to nothing and the next repo sync creates a
  stub holding the freed slug. Bindings survive (they store ids); refs do not. Treat a slug as
  frozen after first push. **Omitting `slug` on an update means "keep the stored one"**, not
  "re-derive from name" — the only reason a display-name edit does not re-identify the resource.

## `type` is two fields wearing one key

`type` is read by two consumers that do not agree. **`kdx`** resolves it against its resource
registry to choose the endpoint and the on-disk directory, ignoring case, hyphens and underscores —
`data-definition`, `dataDefinition`, `datadefinition` and the alias `taxonomy` all select the same
type. **The server** stores it verbatim and computes `uri` from it. Leave `type` empty and eight of
the twelve get a server default on create, two of which are not the name you would guess:

| Resource | Stored `type` |
|---|---|
| `data-definition` | **`taxonomy`** |
| `prompt-template` | **`prompt`** |
| `document-store`, `data-store`, `module`, `data-form`, `service-bridge`, `project-template` | same as the resource name |
| `activity-plan`, `task-template`, `task-status` | no default — stays empty unless you set it |
| `knowledge-set` | **not a resource discriminator** — `type` is the *set type*, mirrored to and from `setType`. Never put `knowledge-set` there. |

Both spellings resolve (`taxonomy://` and `data-definition://`, `prompt://` and
`prompt-template://`), so writing either is safe — but the `uri` you get back reflects whichever
string was stored. A `type` the resolver does not know yields a `uri` that resolves to nothing:
`type: dataForm` is a camelCase spelling `kdx` accepts and the server stores verbatim, giving
`dataForm://…`, which is not a scheme. So does an empty column on a flattened type — the metadata
blob's own `type` then shows through.

## `ref` and `uri` — computed, read-only

- `ref` = `orgSlug/slug`, emitted by every envelope type. Any `:version` suffix in stored data is
  stripped on the way out — refs and URIs are unversioned everywhere.
- `uri` = `<stored type>://orgSlug/slug`, minted **by the flatten step**, so the four types that do
  not flatten **never emit a `uri`** — even though the OpenAPI document declares one for them and
  generated clients expose the field. Compose those yourself; they resolve fine.
- Both are ignored on write. `kdx sync pull` strips `ref` but not `uri`, so a pulled file may carry
  a stale-looking `uri:` — harmless, and excluded from push comparison.

## `changeSequence` — the optimistic lock

Every successful update increments it; `GET /api/<type>/{id}/sequence` returns the current value.
**Sending it is opt-in, and getting it wrong is a 409.** If your body carries it and the stored row
is at a different number, the write is rejected with *stale changeSequence: expected N but resource
is at M; re-fetch and retry*. Omit the key (or send an explicit `null`) and no check runs. A
non-null `0` **is** enforced — "never updated yet" is not a free pass.

So **never hand-write `changeSequence` into a resource file.** `kdx sync pull` strips it from every
map at every depth for exactly this reason, recording the watermark in
`.sync-state/<env>/<org-slug>.yaml` instead; push compares against that file and skips a resource
whose server sequence has moved on, which `--force` overrides.

## Where the file lives

```
<metadata_dir>/<type-dir>/<slug>.yaml                              # org-scoped
<metadata_dir>/projects/<project-slug>/<type-dir>/<slug>.yaml      # project-scoped
```

Pull always writes `.yaml`; push reads `<slug>.yaml` and falls back to `<slug>.yml`. Full per-type
directory table: `references/repo-layout.md`. Two traps in it: `data-definition` files live in
**`data-definitions/`**, not `taxonomies/`, and `knowledge-feature` files in
**`knowledge-feature-instances/`**, not `knowledge-features/`. `task-template` and `task-status` sit
flat against a current server; on a legacy one `task-template` moves under `projects/<slug>/`.

`${org}` and `${orgSlug}` are **different substitutions**: `${org}` is resolved by `kdx` at push
time (pull writes it into files wherever the org slug appeared, which is what makes a repo
portable; an unresolved one aborts the push), while `${orgSlug}` passes `kdx` untouched and is
resolved by the *server* on activity-plan and task-template writes. Project-template
materialization resolves both.

## Declared but inert

| Field | Reality |
|---|---|
| `publicAccess` | The unauthenticated-read path looks for a capability no model implements, so the flag never grants access. Stored, returned, sortable, inert. |
| `extensionPackRef` | A column and a JSON field with nothing that writes or reads it. `kdx` strips it from pulled files. |
| `yamlSource` | Intended as a round-trip snapshot of the authored YAML. Nothing on the server populates it, and `kdx` reads it off a pulled response and then discards it — the value never reaches disk. |
| `uri` on `activity-plan` / `task-template` / `task-status` / `knowledge-set` | Declared in the OpenAPI document and present on generated client models; never emitted, because those types skip the flatten step that mints it. |

## Common mistakes

| Mistake | What happens |
|---|---|
| Nesting a flattened type's own fields under `metadata:` | The keys are read (a flat body decodes into both places), but you have written a shape the API never returns; the next pull rewrites the file flat and the diff reads as a rewrite. |
| Writing a `task-template`'s metadata flat | Task templates do not flatten. Flat metadata keys resolve to no column and are dropped silently — as are metadata-blob keys written flat on `activity-plan` and `task-status`. |
| A sparse PUT carrying one metadata field | Replaces the entire metadata blob; every unsent metadata field reverts to zero. |
| `slug: Invoice_Taxonomy` | Underscore fails the format check — `400`. (Capitals alone would have been lowercased silently.) |
| Omitting `slug` on a resource named `"AI"` or `"#1"` | `400` — the derived slug is under 3 characters. |
| Renaming a slug to tidy it up | Every `scheme://org/slug` ref pointing at it goes dark, silently. |
| Setting `organizationId` on an update to move a resource | Ignored; the stored organization is restored before the write. |
| `type: knowledge-set` in a knowledge-set file | `type` is that set's *set type*. You have just relabelled the set. |
| Copying `changeSequence` out of a `GET` into a file | Every later push is a `409` until you re-fetch. Strip it. |
| Expecting `uri` back from an activity plan or task template | Never emitted. Compose `activity-plan://orgSlug/slug` yourself. |

## Related skills

**kdx-cli** — the `kdx apply` envelope, manifests, `api_version`, push order and the sigils.
**project-resource** — bindings store resolved ids, so they survive a slug rename when refs do not.
Per-type field detail: **activity-plan**, **data-definition**, **data-form**, **knowledge-system**,
**module**, **project-template**, **prompt-template**, **service-bridge**, **store**, **task-status**, **task-template**.
