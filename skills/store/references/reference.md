# Store reference — `document-store` and `data-store`

## 1. Complete key inventory

Everything the API round-trips on a store, and where each key actually lives. "Column" keys are real
table columns; "outer" and "inner" keys live inside the store's JSON blob at the two nesting levels
described under "The two-level flatten" below.

| Key | Where | Writable flat | Read by |
|---|---|---|---|
| `id`, `uuid`, `createdOn`, `updatedOn`, `changeSequence` | column | server-set | server-managed; `kdx sync pull` strips them from the file it writes |
| `slug` | column | yes | identity; unique per `(organization, table)` |
| `name` | column | yes | display |
| `type` | column | yes | the computed `uri`; also the resource discriminator |
| `orgSlug`, `ref`, `uri` | computed on read | — | recomputed on every serialize |
| `organizationId` / `organization: {id}` | column | create only | ownership; ignored on update |
| `publicAccess` | column | yes | nothing — the unauthenticated-read check tests for a capability no model implements (**metadata-envelope**) |
| `deprecated` | column | yes | filtered out of the resource picker by default |
| `template` | column | yes | the studio's "create project store from template" flow |
| `extensionPackRef` | column | yes | nothing — no producer and no consumer (**metadata-envelope**) |
| `yamlSource` | column | yes | nothing — the CLI reads it off a pull and then never uses it (**metadata-envelope**) |
| `deleted`, `deletedDate`, `deleteUserEmail`, `deleteUserId` | column | server-set | soft delete |
| `supportsScheduling`, `eventAware` | column | yes | nothing, on a store |
| `storeType` | outer | **yes** | display label; the legacy `type: store` remap |
| `storePurpose` | outer | **yes** | training-store icon and tag popup |
| `deleteProtection` | outer | **yes** | nothing |
| `contentMetadata` | top level (the inner object, re-emitted) | **yes** | replaces the inner object wholesale on write |
| `indexed` | inner | **no** | nothing |
| `documentProperties` (document stores only) | inner | **no** | nothing reachable — see the notes at the end |
| `labelExpressions` (document stores only) | inner | **no** | nothing |
| `type` (inside `contentMetadata`) | inner | **no** | nothing; shadowed on the wire |

Keys that are **not** fields on a store and are dropped on decode: `description`, `version`, `icon`,
`imageUrl`, `overviewMarkdown`, `provider`, `providerUrl`, `providerImageUrl`, `checksum`,
`templateRef`, `templateStoreId`, `projectId`, `files`, `hasImage`, `showThumbnails`,
`showStoreInLabeling`, `highQualityPreview`, `allowDataEditing`. Several of these appear in the
canonical key ordering `kdx sync` uses when it writes a store file, and several appear on the
project-template `stores:` item, which is a different (also closed) struct — neither makes them
supported on a store.

## 2. The two-level flatten, in full

The store record has a `metadata` field holding an **outer content object**
(`storeType`, `storePurpose`, `deleteProtection`, plus its own `metadata` field), and that field
holds an **inner content object**: `type` and `indexed` on both store kinds, plus
`documentProperties` and `labelExpressions` on document stores only.

**Serialize** (every `GET`, and what `kdx sync pull` writes to disk):

1. The inner object's keys are merged into the outer object — but only where the outer object does
   not already have that key.
2. The intact inner object is emitted at the top level under **`contentMetadata`** (absent entirely
   when the store has no inner object).
3. The inner `metadata` key is deleted from the outer object.
4. The outer object's keys are merged into the top level — but only where the top level does not
   already have that key. This is why the inner `type` (`document` / `data`) never appears flat: the
   record's own `type` (`document-store` / `data-store`) already occupies that key.
5. The `metadata` key is deleted from the top level, and `ref` and `uri` are recomputed from
   `orgSlug` + `slug` + `type`.

**Deserialize** (`POST`, `PUT`):

1. The body is decoded normally, so a nested `metadata: { … metadata: { … } }` populates both levels.
2. The **whole body** is then decoded a second time into the **outer** content object. This is what
   makes `storeType` / `storePurpose` / `deleteProtection` work when written flat — and what makes
   inner keys written flat resolve to nothing and disappear.
   A type mismatch here is a hard error: `400 invalid JSON: invalid metadata field: …`.
3. Finally, a top-level `contentMetadata` object **replaces** the inner object outright. A
   `contentMetadata` that is `null`, a string, an array or a number is ignored rather than applied;
   a type mismatch *inside* it is swallowed, leaving a zero value.

### The four authoring forms compared

| Body | Outer keys | Inner object | Notes |
|---|---|---|---|
| Flat outer only — `storeType: DOCUMENT` | set | untouched | **the form to author** |
| Flat inner — `indexed: true` at top level | — | **not set** | silently lost |
| `contentMetadata: {type: document, indexed: true}` | — | replaced | the only flat way to set inner keys |
| Nested — `metadata: {storeType: …, metadata: {…}}` | set | set | works, but nothing writes it back this way |
| Mixed — `metadata: {storeType: …}` plus flat keys | flat wins | reset to `{}` | leaves `contentMetadata: {}` on the wire |

`PUT` semantics follow from step 2. The stored JSON blob is rewritten wholesale from the decoded
body when **both** hold: the body carries some key that resolves to no column (any flat content key,
or even the computed `ref`/`uri` a `GET` body carries), **and** that body decodes to a non-empty
outer content object. So `{"name": "Invoices", "ref": "acme-corp/invoices"}` leaves the blob
untouched, while the same body plus `"storeType": "DOCUMENT"` replaces it with exactly what the body
said — omitted content keys included. Round-trip the full `GET` body, `contentMetadata` and all.

## 3. `kdx validate` on a store file

The published OpenAPI schema for both store types still models the **nested** form: it declares
`metadata` as an object and declares no `storeType`, `storePurpose`, `deleteProtection` or
`contentMetadata` at the top level. Validating a correct flat store file therefore produces
`key not found in schema` **warnings** for those keys. They are warnings, not errors — the schema
does not forbid additional properties — and the flat form is what the server accepts. `slug` is the
one key the create schema marks required.

`kdx validate` applies the same legacy `type: store` → `storeType` remapping as `kdx apply` before
it looks the resource type up, so a legacy file validates against the type it would actually be
pushed as.

## 4. Lifecycle and identity

- **Slug uniqueness** is per `(organization, table)`. Nothing stops the same slug naming a document
  store and a data store; the URI scheme is what distinguishes them.
- **Slug generation**: the API derives a URL-safe slug from `name` when the body has none, and
  rejects a body carrying neither with a 400. `kdx apply` refuses a file with no `slug` before it
  ever reaches the API, so the generation only helps direct API callers.
- **Soft delete**: `DELETE` flags the row, stamps the deleting user, and appends a UUID suffix to
  the slug so the original slug is immediately reusable. The store vanishes from lists; the
  documents are not recovered by it.
- **Delete is blocked while bound**: a store with any `kdxa_project_resources` row returns
  `409 resource is linked to N project(s); unlink it before deleting`. Unbind everywhere first.
- **Object-storage keys**: a document family records `storeRef` as `orgSlug/storeSlug`, and every
  content object's storage key is built from that ref. Each object keeps the ref it was written
  with, so renaming a store's slug does not orphan existing content — but new content lands under a
  different prefix. Treat the slug as permanent once documents exist.
- **Data objects** are keyed to a data store by id, and the foreign key cascades on a hard row
  delete.

## 5. How a store is reached

| Consumer | How it names the store |
|---|---|
| Intake | `store: {ref: "orgSlug/storeSlug"}` — always a **document** store, resolved at create only |
| Activity / step / script | document families from **bound** document stores only |
| Trigger (`document_locked`) | the event's project comes from the store's oldest project binding |
| Studio project panels | the project-resource binding rows, loaded by `resourceType` |
| Uploads | `POST /api/document-stores/{id}/upload`, plus presigned and multipart variants |

A binding whose `resourceType` does not match the table the store lives in resolves to nothing: the
id is looked up in the other table, misses, and the store is silently absent from the project. Bind
document stores as `document-store` and data stores as `data-store`.

## 6. Notes on individual inert keys

- **`deleteProtection`** — present on both store kinds and on the project-template `stores:` item.
  No delete path consults it. It is not a safety net; use permissions.
- **`documentProperties`** — the studio's upload dialog renders a configuration option per entry,
  but it reads them from a nested `metadata` object on the store record, and the API deletes that
  key on every serialize. The block therefore never renders.
- **`templateStoreId`** — written only when a project template materializes a document store with a
  `templateRef:`. It is never serialized (so it cannot be set or read through the API) and nothing
  consumes it. It copies no documents, metadata or configuration.
- **`supportsScheduling`** — a real column on both store tables, but only the module resource type
  is ever filtered on it. Its neighbour **`eventAware`** has no reader anywhere in the platform.
- **`template`** — the studio's "create a project store from this one" flow requires the *source*
  store to carry it. Setting it on a store you author has no other effect.

## 7. The legacy unified store table

`kdxa_stores` still exists from before document stores, data stores and modules were split into
their own tables. It is not served by any endpoint and no URI scheme resolves to it. If you meet a
file whose `type:` is `store`, treat it as legacy input to the `storeType` remapping (SKILL.md), not
as a resource type to author.
