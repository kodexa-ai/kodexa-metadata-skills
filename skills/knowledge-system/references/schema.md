# Knowledge schema reference

Field inventories, option-type vocabulary, mutability, attachments, and the read-side model.
The facts that fail *silently* live in `SKILL.md`; this file is the lookup table.

---

## 1. Resource fields

### Knowledge feature type (`type: knowledgeFeatureType`, org-scoped)

| Field | Notes |
|---|---|
| `slug` | `^[a-z0-9-]+$`; `kdx apply` refuses a file without one |
| `name` | the display label; always set it |
| `description` | |
| `options` | bare list — the schema for the **core** properties of an instance. It is the instance's `properties` map that hashes into its slug; `options` only describes it. |
| `extendedOptions` | bare list — metadata that is **not** hashed |
| `color` | hex string, UI badge |
| `icon` | icon identifier |
| `active` | boolean |
| `labelJsonPath` | JSONPath (or JSONata) that renders an instance's label — **optional** |
| `useJSONata` | evaluate `labelJsonPath` with JSONata instead of JSONPath |

**`labelJsonPath` is optional.** With none (or on error) the label falls back through
`feature.value` → feature type `name` → feature `slug` → feature `id` → `"Untitled"`.

The evaluation context is **not** the raw `properties` object. It is:

```js
{ slug, id, ...properties, _extendedProperties: { ... } }
```

so `$.vendorId` reaches a core property, `$.slug` reaches the computed slug, and
`$._extendedProperties.displayName` reaches an extended one.

```yaml
labelJsonPath: '$._extendedProperties.displayName & " (" & vendorId & ")"'
useJSONata: true
```

JSONata evaluation is asynchronous and cached, so the **first** render of a feature shows the
fallback label and the real one appears on the next tick. JSONPath is synchronous.

### Knowledge feature (`type: knowledgeFeature`, org-scoped)

| Field | Notes |
|---|---|
| `slug` | content-addressed; omit it and the server computes it (see `SKILL.md`) |
| `featureTypeRef` | `<orgSlug>/<featureTypeSlug>`. `featureTypeId` or a nested `featureType: {id}` / `featureType: {slug}` also resolve. |
| `properties` | **hashed into the slug**; opaque JSON — nothing validates its keys against the type's `options` |
| `extendedProperties` | not hashed; fully mutable; also feeds feature search |
| `active` | boolean |

There is no `name` or `description` on a feature — its slug is its identity, and its label is
rendered from the feature type.

### Knowledge item type (`type: knowledgeItemType`, org-scoped)

| Field | Notes |
|---|---|
| `slug`, `name`, `description` | `kdx apply` refuses a file without `slug`; always set `name` too |
| `options` | bare list — the per-item configuration schema |
| `impact` | declaration only; see "Declared but inert" in `SKILL.md` |
| `supportsAttachment` | boolean; enables the per-item file attachment |

No `color`, `icon`, `active` or `labelJsonPath` — those are feature-type fields.

### Knowledge item (`type: knowledgeItem`, needs `projectSlug` for `kdx apply`)

| Field | Notes |
|---|---|
| `slug` | auto-derived from `title` **on create only** if omitted; a title that cannot produce a valid 3–200-char slug is a 400 |
| `title`, `description` | |
| `knowledgeItemTypeRef` | `<orgSlug>/<itemTypeSlug>` |
| `knowledgeSetSlug` | write-time reference resolved against the item's `orgSlug` (mandatory alongside it) |
| `properties` | keyed by the item type's `options` names. **Not enforced** — a key with no matching option is stored, hashed nowhere, and read by nothing. |
| `active` | defaults true; `false` means the item is never baked |
| `sequenceOrder` | ordering within the set |
| `attachmentId`, `attachmentPath` | local-only sync directives (see "Attachments" below) |

### Knowledge set (`type: knowledgeSet`, org-scoped, optional project)

| Field | Notes |
|---|---|
| `slug`, `name`, `description` | |
| `setType` | always set it explicitly (see `SKILL.md`) |
| `status` | `ACTIVE` \| `INACTIVE` \| `BUILDING` \| `NEEDS_CLARIFICATION` \| `PENDING_REVIEW` \| `ARCHIVED` \| `ERROR`. Legacy `IN_REVIEW` was migrated to `PENDING_REVIEW`, `ON_HOLD` to `INACTIVE`. |
| `priority` | 0–10, default 5; bake order only |
| `projectSlug` | resolved by the CLI. **The server does not resolve it** — a hand-rolled POST carrying only `projectSlug` stores the string and leaves the project id NULL, so the set silently behaves as org-wide. |
| `featureExpression` | see `references/expressions.md` |
| `features` | the **Feature Palette** join (see "The Feature Palette" below) |
| `knowledgeItems` | has-many over the item foreign key |
| `openQuestions`, `createdByActivityId`, `promotedBy`, `promotedAt`, `currentSnapshotId` | read-mostly; written by the candidate lifecycle (below) |

`setName` / `setType` are deprecated aliases for `name` / `type`, kept for the current API
deprecation window; whichever side is empty is filled from the other.

---

## 2. The Feature Palette (`features:` on a set)

Direct `POST` / `PUT /api/knowledge-sets` reconciles the `kdxa_knowledge_set_features` join table
from the payload's `features` array:

| Payload | Effect |
|---|---|
| key absent or `null` | links untouched |
| `[]` | **all links removed** |
| `[...]` | links reconciled exactly |

Every entry must carry `id` **or** `slug`; both are validated against the set's own organization —
an unknown slug is `400 unknown feature slug "..." in this organization`.

On read, the set's `features` is the **union** of palette links and the features named by
`featureExpression`, deduplicated by id.

**The CLI does not use this path.** `kdx apply` / `kdx sync push` strip `features` from the
metadata payload and create the entries through a side channel instead, so the palette is not
written by a CLI push — only expression-referenced features appear.

---

## 3. Option vocabulary

### The Option object

The only keys an option has: `aliases`, `default`, `description`, `developerOnly`,
`displayProperties`, `falseLabel`, `featureFlag`, `groupOptions`, `hint`, `label`,
`listDescription`, `listLabel`, `listType`, `name`, `overviewMarkdown`, `possibleValues`,
`properties`, `required`, `showIf`, `showOnPopup`, `subType`, `supportArticle`, `tabName`, `type`.

There is no `lines`, `min`, `max`, or `password` key — see `SKILL.md`.

| Want | Write |
|---|---|
| multi-line text box | `type: text` + `properties: { lines: 8 }` |
| masked secret | `type: password` |
| numeric bounds | *unsupported — validate in your consumer* |
| dropdown | `type: select` with `possibleValues: [{value, label}]` (also: `type: string` **with** `possibleValues` renders as a select) |
| developer-only field | `developerOnly: true` |
| false-state caption on a toggle | `falseLabel: "..."` |

### Type names

Resolved either through an alias table or by exact match. Anything else renders a red
**"Unknown Option Type"** banner in place of the field.

**Direct types:** `alert`, `article`, `attributeStatus`, `boolean`, `chart`, `cloudEmbedding`,
`cloudModel`, `code`, `componentLookup`, `dataForm`, `documentLookup`, `documentStatus`,
`knowledgeFeature`, `knowledgeSet`, `label`, `markdown`, `number`, `select`, `taskStatus`,
`taskTemplates`, `taxon`, `taxonLookup`, `taxonWithProperties`, `taxonomyTaxon`,
`taxonomyTaxonSelection`, `text`, `workspace`.

**Aliases** (routed to a direct type):

| Alias | Renders as |
|---|---|
| `python`, `javascript`, `script`, `simpleExpression` | `code` |
| `tableStore`, `documentStore`, `modelStore`, `taxonomy`, `dataDefinition`, `assistant` | `componentLookup` |
| `regex`, `password`, `string` | `text` |
| `groupTaxon` | `taxon` (groups only) |

`moduleStore`, `taxonomyStore`, `document`, `taxon_label`, `taxon_with_properties`, `taxon-lookup`,
`cloud-model`, `cloud-embedding` and `data-form` are **not** valid names.

### Lists and groups

The renderer branches on **the presence of `listType`**, then on `type: object`:

```yaml
- name: fieldMappings
  type: list                # convention; the row renderer comes from listType
  listType: object          # a list of structured rows
  label: "Field mappings"
  properties:
    useTable: true          # editable table instead of stacked cards
  groupOptions:
    - { name: source, type: string, required: true }
    - { name: pattern, type: code, required: true }

- name: bounds
  type: object              # single grouped sub-record
  properties: { collapsible: true }
  groupOptions:
    - { name: x1, type: number }
    - { name: y1, type: number }
```

`listType: string` gives a simple add/remove list of scalars.

### `properties.*` modifiers

| Key | Effect |
|---|---|
| `lines: N` | textarea with N rows (`text` / `string` / `password`) |
| `height: "600px"` | editor height — `markdown` (default `200px`) and `code` (default `300px`) |
| `language: python` | `code` editor language |
| `collapsible: true` | `object` group (and list) renders collapsed |
| `useTable: true` | `listType: object` renders as an editable table |
| `markdown: "..."` | "Click for more information" help panel beside the field |
| `hideLabel: true` / `hideDescription: true` | suppress label / description |
| `pattern`, `patternMessage` | client-side regex validation on text |
| `valueField`, `textField` | override the `value` / `label` keys when reading `possibleValues` |

### `taxonomyTaxon` / `taxonomyTaxonSelection`

Stored value: `orgSlug/taxonomySlug//taxonPath` (just `orgSlug/taxonomySlug` when
`taxonMode: none`).

```yaml
- name: taxonomyAndTaxon
  type: taxonomyTaxon
  label: "Data Definition"
  required: true
  properties:
    scope: global          # global (org lookup) | project (project content taxonomies)
    taxonMode: leaves      # leaves (default) | groups | all | none
    taxonomyType: CONTENT  # CONTENT | CLASSIFICATION | PROCESSING | MODEL — global scope only
    enabledOnly: true
    organizationSlug: ...  # override the active org, global scope only
```

### Built-in item types shipped in the `kodexa` org

| Slug | `impact` | Options |
|---|---|---|
| `taxon-semantic-definition-customization` | `PROMPT_OVERRIDE` | `taxonomyAndTaxon` (taxonomyTaxon), `prompt` (markdown), `replace` (boolean) |
| `base-prompt-customization` | `BASE_PROMPT_OVERRIDE` | `taxonomyAndTaxon` (taxonomyTaxon), `prompt` (markdown) |
| `taxon-enabled` | `TAXON_ENABLE` | `baseTaxonomyAndTaxon` (taxonomyTaxon), `enabled` (boolean) |
| `taxon-name-customization` | `NAME_OVERRIDE` | `baseTaxonomyAndTaxon` (taxonomyTaxon), `override_name` (string) |
| `taxon-selection-option-enabled` | `SELECTION_OVERRIDE` | `taxonomyAndTaxonAndSelectionOption` (taxonomyTaxonSelection), `enabled` (boolean) |
| `project-extraction-model` | *(none)* | `extraction_model` (string) |
| `document-note` | *(none)* | `content` (markdown) |

Also shipped: the feature type `kodexa/document-note`.

---

## 4. Attachments

### Set-level (shared, `attachment://`-addressable)

Declared on the set file and pushed by `kdx apply` / `kdx sync push`:

```yaml
attachments:
  - attachmentId: acme-invoice-header    # slug used in markdown; unique within the set
    attachmentPath: attachments/acme-header.png   # a FILE, relative to this YAML
```

Reference from any markdown-typed item property:

```markdown
![Acme header](attachment://acme-invoice-header)
```

| Method | Endpoint | Behaviour |
|---|---|---|
| `GET` | `/api/knowledge-sets/{id}/attachments` | list — `id, attachmentId, contentHash, fileName, size, contentType, uploadedAt, uploadedBy, createdOn, updatedOn` |
| `POST` | `/api/knowledge-sets/{id}/attachments` | multipart upload; **upserts** on `(set, attachmentId)`; requires the `upload` permission on the set |
| `GET` | `/api/knowledge-sets/{id}/attachments/{attachmentId}` | **streams the file bytes** through the API — not a JSON URL |
| `DELETE` | `/api/knowledge-sets/{id}/attachments/{attachmentId}` | remove |

Uploading a set attachment mints a snapshot (`SET_ATTACHMENT_UPLOADED`).

### Per-item (one file per item)

On a **standalone** `knowledgeItem` file only — the keys are stripped from the payload and treated
as local sync directives; `attachmentPath` may be a file **or a directory, which is zipped**:

```yaml
attachmentId: sample-doc
attachmentPath: samples/acme-invoice.pdf
```

Or from the CLI, where the ref may be a UUID or a slug:

```bash
kdx knowledge attach <item-ref> --file ./image.png --id my-id
kdx knowledge attach <item-ref> --folder ./sample-data/
kdx knowledge download <item-ref> -o ./out.zip
```

There is **no** `kdx knowledge sync`. `kdx sync pull` writes `attachmentPath` / `attachmentId` back
out and downloads files into `<set-slug>-attachments/`.

**Item attachments are not `attachment://`-addressable.** That scheme resolves only against the
**set-level** attachment list, whatever the CLI's success message says.

---

## 5. Snapshots, sources, applications, and the candidate lifecycle

### Snapshots (`/api/knowledge-set-snapshots`)

Every set create/update, every item create/update/delete, every attachment upload or removal (set-
or item-level), every assessment, and every promote mints a snapshot **if the content hash changed**.
Payload: `{name, description, status, priority, items[], featureSlugs[], metadata, snapshotAt}`,
with `status` and `snapshotAt` zeroed before hashing so a pure lifecycle flip does not mint one.
Fields: `version`, `contentHash`, `data`, `createdBy`, `changeReason` (`INITIAL_CREATE`, `UPDATE`,
`ITEM_UPDATE`, `ATTACHMENT_UPLOADED`, `ATTACHMENT_REMOVED`, `SET_ATTACHMENT_UPLOADED`,
`SET_ATTACHMENT_REMOVED`, `ASSESSMENT`, `PROMOTE`), `buildContext`, `reproductionScore`,
`validatedAt`. The set's `currentSnapshotId` points at the newest. Inactive items still appear in
the snapshot, flagged `optional: true`.

Dedupe is against the **current** snapshot only, so a content revert mints a new snapshot rather
than no-opping.

### Sources (`/api/knowledge-set-sources`)

Provenance: which content object a set was derived from, plus `executionId` and free-form
`metadata`.

### Applied knowledge sets (`/api/applied-knowledge-sets`)

One row per (document family, set) application, pinned to `snapshotId`.
`status ∈ ACTIVE | SUPERSEDED | REMOVED`; `trigger` (`ASSESSMENT`, `MANUAL`, `INGESTION`, …);
`appliedBy`, `confidence`, `supersededAt`, `supersededById`, `sourceContentObjectId`. Sub-routes:
`/knowledge-sets/{id}/stats`, `/knowledge-sets/{id}/outdated`, `/{applicationId}/snapshot`.

### Candidate lifecycle

Sets produced by an agent run carry `createdByActivityId`, default to `PENDING_REVIEW`, and may
carry `openQuestions`. All routes under `/api/knowledge-sets/{id}`:

| Verb | Path | Purpose |
|---|---|---|
| POST | `/questions/{questionId}/answer` | answer one open question |
| POST | `/validate` | validate a candidate |
| PUT | `/snapshots/{snapshotId}/validation` | write `buildContext` / `reproductionScore` / `validatedAt` |
| GET | `/promote-collisions` | same-element collision pre-flight for review |
| POST | `/promote` | the **only** path to `ACTIVE`; requires `PENDING_REVIEW` plus a validated current snapshot; body `{allowTaxonCollisions?: bool}` |
| POST | `/demote` | reverse a promotion → `ARCHIVED` |
| POST / GET | `/tests` | reprocess-proof runs |

A test run records a two-arm proof: `reproductionScore` (candidate applied) against
`controlReproductionScore` (candidate absent), with `correctedPaths` / `matchedPaths` /
`controlMatchedPaths`, `status ∈ PENDING | RUNNING | PASSED | FAILED | ERROR`, and
`validationMode`.

---

## 6. Mutability

| Entity | Mutable? |
|---|---|
| `knowledgeFeatureType` | **Yes** — every field, including `slug`, `options`, `extendedOptions`, `labelJsonPath`, `useJSONata`, `color`, `icon`, `active`. |
| `knowledgeItemType` | **Yes** — `name`, `slug`, `description`, `options`, `impact`, `supportsAttachment`. |
| `knowledgeFeature.properties` | **Effectively no.** Writable, but the slug was hashed at create and never recomputes. Delete + recreate instead. |
| `knowledgeFeature.extendedProperties` | **Yes** — not hashed. Put anything that may change here. |
| `knowledgeItem` | **Yes** — `title`, `description`, `properties`, `sequenceOrder`, `active`, `attachmentId`. The slug is user-supplied, so renaming is fine. |
| `knowledgeSet.status` | **Guarded** — a `PUT` into `ACTIVE` from a candidate state is a 409; use `promote`. |
| Set attachments | **Upsert** by `attachmentId`. |

Adding options to a type at any time is safe. **Removing a `required` option is not** — existing
instances may no longer satisfy the schema. Mark it optional first, drop it later. Changing a
feature type's `labelJsonPath` retroactively changes how every existing instance displays.
Renaming a feature type's `slug` does **not** rewrite the feature slugs already computed from it —
existing instances keep the old prefix and keep matching; only new instances get the new one.

---

## 7. Referencing knowledge from elsewhere

`POST /api/resolve?path=<uri>` turns a slug URI into an id-based API path. Knowledge schemes:
`knowledge-set`, `knowledge-item-type`, `knowledge-feature-type`, `knowledge-feature`, and
`knowledge-item` (resolved through its parent set, since items have no organization column).

```
knowledge-set://my-org/vendor-extraction-rules  ->  /api/knowledge-sets/<id>
```

URIs are unversioned. In refs inside a YAML file, `${org}` is a placeholder the CLI expands to the
target org — useful for files produced by `kdx sync pull`. `kdx apply` still needs a real org, so
pass `--org-slug my-org` when the file carries `${org}`.
