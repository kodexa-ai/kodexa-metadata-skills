---
name: store
description: "Use when authoring or editing a Kodexa document store or data store — the org-scoped containers documents and extracted data live in. Covers the flattened wire shape and the inner keys that vanish if you write them flat, storeType and storePurpose values, the legacy type: store remapping, and the binding a store needs before anything can see it."
---

# Kodexa Store Authoring — `document-store` and `data-store`

## Two resources, two tables, three schemes

A **document store** holds document families — the uploaded file and every processed version of it.
A **data store** holds the data objects extraction writes. Both are first-class **org-scoped**
resources with their own YAML file, endpoint and table, not just a block in a project template.

| | Document store | Data store |
|---|---|---|
| `type:` | `document-store` | `data-store` |
| Endpoint | `/api/document-stores` | `/api/data-stores` |
| Table | `kdxa_document_stores` | `kdxa_data_stores` |
| URI | `document-store://acme-corp/invoices`, or `store://acme-corp/invoices` | `data-store://acme-corp/invoice-data` |
| Sync directory | `<metadata_dir>/document-stores/` | `<metadata_dir>/data-stores/` |
| Upload endpoints | yes (`/upload`, presigned, multipart) | none |

`store://` is an alias for **document stores only** — it resolves in `kdxa_document_stores` and
never falls through to the data-store table. Slugs are unique per organization *per table*, so
`invoices` can name a document store and a data store at once; only the scheme separates them.
Both push at order 20, with data definitions, data forms, modules and service bridges.

## The wire shape: flat coming out, one level flat going in

This is the single fact that decides whether a hand-written store file works.

The Go model nests a content object inside the store record, and **that** content object nests
another. On serialize — every `GET`, and the file `kdx sync pull` writes — **both** levels are
merged up to the top level, the `metadata` key is deleted, and the inner object is *also*
re-emitted intact under `contentMetadata`. A store read back is completely flat:

```yaml
type: document-store            # envelope, alongside publicAccess / deprecated / template
slug: invoices
name: Invoices
orgSlug: acme-corp
storeType: DOCUMENT             # outer content object
storePurpose: OPERATIONAL       # outer
deleteProtection: false         # outer
supportsScheduling: false       # real column, like eventAware
indexed: true                   # flat copy of an INNER key — see below
contentMetadata:                # the inner object, intact
  type: document
  indexed: true
```

Deserialize is **not** the mirror image, and that asymmetry is where data is lost silently: a flat
body is folded into the **outer** content object only.

| Key | Lives at | Settable flat? |
|---|---|---|
| `storeType`, `storePurpose`, `deleteProtection` | outer content object | **yes** |
| `indexed`, `documentProperties`, `labelExpressions` | inner content object | **no — silently dropped** |
| `supportsScheduling`, `eventAware`, and every envelope field | real columns | yes |

Write `indexed: true` at the top level of a store file and the server returns 201/200 and stores
nothing for it — it resolves to no field at the outer level and is discarded like any unknown key.
The inner object can only be set **whole**, either as `contentMetadata:` or as the fully nested
`metadata: { metadata: { … } }`. `contentMetadata` **replaces** the inner object; it is never
merged field-by-field, so a partial `contentMetadata` drops whatever it omits.

The practical answer is to leave the inner object out of authored YAML entirely: `indexed`,
`documentProperties` and `labelExpressions` are legacy round-trip keys **no platform code reads**.
Set only the outer keys, and let `contentMetadata` ride along untouched on pulled files.

Two edges of the same mechanism: a top-level `metadata:` key with no inner `metadata:` of its own
resets the inner object to `{}`, so pick one form per file; and a wrong-typed **outer** key fails
the write with `400 invalid JSON: invalid metadata field: …` while the same mistake **inside**
`contentMetadata` is swallowed and stores a zero value.

## Shape that works

```yaml
# <metadata_dir>/document-stores/invoices.yaml
type: document-store
slug: invoices
name: "Vendor Invoices"
orgSlug: acme-corp
storeType: DOCUMENT
storePurpose: OPERATIONAL
deleteProtection: false
```

```yaml
# <metadata_dir>/data-stores/invoice-data.yaml — same shape, TABLE instead of DOCUMENT
type: data-store
slug: invoice-data
name: "Extracted Invoice Data"
orgSlug: acme-corp
storeType: TABLE
storePurpose: OPERATIONAL
```

That is the whole authorable surface. There is no `description`, `icon`, `overviewMarkdown`,
`version` or `templateRef` on a store — see **Declared but inert**.

`slug` is generated from `name` when the API body has none, but `kdx apply` refuses a file without
it. A document store's `orgSlug/slug` is baked into the object-storage key of every content object
uploaded under it, and each object keeps the ref it was written with — treat the slug as permanent
once documents exist. A duplicate slug in the same organization is a `409`.

## `storeType` and `storePurpose`

Neither is validated by the API: any string persists. What consumes them:

- **`storeType`** — the studio recognises `DOCUMENT` ("Documents"), `TABLE` ("Extracted Data") and
  `MODEL` ("Trainable Model"); on anything else its store card shows neither label nor name. Write
  `DOCUMENT` for a document store and **`TABLE`** for a data store — project-template materialization
  writes `DATA`, so both spellings exist in the wild, and only `TABLE` is recognised anywhere.
- **`storePurpose`** — `OPERATIONAL` or `TRAINING`. `TRAINING` changes the store's icon in the
  project navigation and makes the document viewer's tag popup show raw tags instead of data
  objects. Any other value behaves as `OPERATIONAL`.

`storeType` does **not** route the write. The endpoint decides which table the row lands in: for
`kdx sync push` the manifest key picks it, for `kdx apply` the file's `type:` does. A data store
posted to `/api/document-stores` is a document store carrying a misleading label.

## Legacy `type: store` is remapped by `storeType`

Older files use one generic type with `storeType` as the discriminator. Both `kdx apply` and
`kdx validate` rewrite it before resolving the resource, and `apply` prints the mapping it made:
`storeType: DOCUMENT` → `document-store`, `TABLE` → `data-store`, `MODEL` → `module`.

Any other `storeType` — **including `DATA`** — leaves `type: store` alone, and `store` is a
registered alias of *document-store*, so `kdx apply` pushes the file as a document store. A legacy
data store carrying `storeType: DATA` therefore lands in the wrong table: change it to `TABLE`, or
write `type: data-store` explicitly. `type:` is also stored verbatim and drives the computed `uri`
on every read, so a file that keeps `type: store` reports `uri: store://…` forever.

## Nothing sees a store until it is bound

A store is org-scoped; a project reaches it only through a **project-resource binding**. Project
store lists are loaded from the binding rows by `resourceType`, not from `storeType`, so:

- An unbound store is absent from the project's store lists and pickers — no error, just missing.
- An activity, step or script sees document families **only** from document stores bound to its
  project; families in unbound stores read as not found.
- A `document_locked` event carries the project resolved from the store's binding (oldest first).
  Lock a document in an unbound store and no trigger can match the event.
- Binding a document store as `resourceType: data-store` (or the reverse) creates a row that
  resolves to nothing: the id is looked up in the other table, misses, and the store never appears.
- Deleting a bound store is `409 resource is linked to N project(s); unlink it before deleting`.

Bind with a `linked: document-store: [...]` / `linked: data-store: [...]` block in the manifest, a
`stores:` entry in a project template, or the project's Resources panel. See **project-resource**.

## Where a store lives

- **Synced file** — `<metadata_dir>/document-stores/<slug>.yaml` or `…/data-stores/<slug>.yaml`,
  listed by slug under the manifest's `organization:` block, and again under a project's `linked:`.
- **API** — `POST`/`PUT` on `/api/document-stores` and `/api/data-stores`. A `PUT` that sets
  `storeType`, `storePurpose`, `deleteProtection` or `contentMetadata` rewrites the whole stored
  blob from that body, so round-trip the full `GET` body (`contentMetadata` included), not a
  fragment. A body of real columns only (`{"name": "…"}`) leaves the blob alone.
- **`DELETE`** is a soft delete: the row is flagged and its slug gets a UUID suffix, freeing the
  slug for reuse. Its document families are not deleted, and `deleted` is not client-writable on an
  update, so nothing in the API undoes it.
- **Project template** — the `stores:` array creates *or* binds a store at project create. It
  applies `slug`, `name`, `storePurpose` and `deleteProtection`; `storeType` only picks which kind
  to create, and the server stamps `DOCUMENT` or `DATA` itself.

## Declared but inert

| Key | Reality |
|---|---|
| `deleteProtection` | Stored and round-tripped on both store kinds. **Nothing checks it** — no delete path, API or studio, consults it. It does not protect the store. |
| `indexed`, `documentProperties`, `labelExpressions` | Inner content keys kept so legacy rows survive a read-modify-write; nothing reads them. The upload dialog looks for `documentProperties` under a nested `metadata` object the API no longer emits, so it never renders. |
| `supportsScheduling`, `eventAware` | Real columns on both store tables, round-tripped. `supportsScheduling` is only ever read to filter the studio's *module* picker; `eventAware` has no reader anywhere. On a store both are inert. |
| `templateRef`, `templateStoreId` | Not fields on a store. The underlying template-store column is never serialized and cannot be set through the API — only project-template materialization writes it, and nothing reads it. It copies no documents, metadata or configuration. |
| `description`, `version`, `icon`, `imageUrl`, `overviewMarkdown`, `provider`, `providerUrl`, `providerImageUrl`, `checksum`, `projectId` | Not fields on a store: accepted, dropped on decode, absent from the next `GET`. Several appear in the canonical key ordering `kdx sync` writes, which makes them look supported. |
| `type` inside `contentMetadata` (`document` / `data`) | Always shadowed on the wire by the resource `type`, so it is invisible flat. It survives only inside `contentMetadata`, and nothing branches on it. |
| `template: true` | Round-tripped. The studio's project-store creation flow requires it on the source store it copies from; it has no effect on a store you author directly. |

`deprecated: true` is the opposite case — it is honoured: a deprecated store is filtered out of the
resource picker unless "include deprecated" is on.

## Common mistakes

| Mistake | What happens / fix |
|---|---|
| `indexed` / `documentProperties` / `labelExpressions` written flat | Accepted, stored nowhere. Send the inner object whole under `contentMetadata:`, or omit it — nothing reads these. |
| A top-level `metadata:` that has no inner `metadata:` of its own | The inner object is reset to `{}` and `contentMetadata: {}` appears on the next read. Author flat only. |
| Partial `contentMetadata:` on an update | It replaces the inner object wholesale; omitted keys are gone. |
| `deleteProtection: true` expecting the delete to fail | Nothing enforces it. Remove the permission instead. |
| `storeType: DATA` on a data store | Its store card renders blank, and `type: store` files carrying it are pushed as **document** stores. Use `TABLE`. |
| `store://acme-corp/invoice-data` for a data store | `store://` only ever resolves document stores. Use `data-store://`. |
| Binding a document store as `resourceType: data-store` | Accepted, resolves to nothing, store never appears in the project. |

Full key inventory, the four authoring forms, `kdx validate` warnings, lifecycle: `references/reference.md`.

## Related skills

**project-resource** (the binding, and what breaks without it) · **project-template** (the inline
`stores:` block) · **metadata-envelope** (`slug`, `type`, `ref`, `uri`, soft delete) · **kdx-cli**
(manifest layout, push order, `apply` vs `sync push`) · **intake** (points at a document store).
