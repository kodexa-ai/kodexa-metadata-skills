---
name: knowledge-system
description: "Use when authoring or debugging Kodexa knowledge YAML — knowledge sets, knowledge feature types, knowledge item types, feature instances and knowledge items — including featureExpression trees, content-addressable feature slugs, option schemas, attachment:// references, set status and promotion, and diagnosing a set that never matches a document."
---

# Kodexa knowledge system

Knowledge connects **what is true about a document** (features) to **what should change when processing it** (items), through a **knowledge set** that owns a
boolean rule over features.

| Resource | `type:` | What it is |
|---|---|---|
| Knowledge feature type | `knowledgeFeatureType` | Schema for a kind of fact (vendor, document type, language) |
| Knowledge feature | `knowledgeFeature` | One instance of that fact. **Slug is a hash of its properties.** |
| Knowledge item type | `knowledgeItemType` | Schema for a kind of behaviour change |
| Knowledge item | `knowledgeItem` | One configured behaviour change, living inside a set |
| Knowledge set | `knowledgeSet` | `featureExpression` (which features) + its items (what changes) |

More: `references/schema.md` (fields, option types, lifecycle, attachments), `references/expressions.md` (matching and runtime), `references/examples.md`
(complete file sets). **Related skills:** `project-template` owns the `knowledgeSets:` embedding (where `active` and `showOnNewProject` do exist), `kdx-cli`
owns `apply` / `sync push`, `data-definition` owns the taxonomies and taxon paths every prompt-override item points at.

## A set only reaches a document if all four hold

Top cause of "my set is perfect and nothing happens". Nothing in a set file does step 1.

1. **The feature is attached to the document family.** Attachment is a separate act: `POST /api/knowledge-features/assignToDocumentFamily {featureId,
   documentFamilyId}`, `POST /api/document-families/{id}/add-knowledge-feature`, or feature ids configured on an intake.
2. **The set resolves to `status: ACTIVE`** and is not deleted. **Leave it org-wide unless a plan step will process the document.** `POST
   /api/document-families/{id}/assess` and every upload/store event match **org-wide sets only** — a set carrying a project is invisible there, because the
   project is never derived from the document or its store. A project-scoped set matches only when a plan step runs it inside that project.
3. **`featureExpression` evaluates true** over the attached features.
4. **The item has `active: true`.** Inactive items are never baked, though they still appear in the set's snapshot.

`POST /api/document-families/{id}/assess` records the match; items then bake into the document in set-priority order.

## Feature slugs are content-addressed — compute them

```
slug = "{featureTypeSlug}-{sha256(canonicalJSON(properties))[:32]}"
```

Canonical JSON = `properties` only (never `extendedProperties`), keys sorted alphabetically, compact, no spaces; take the first 16 bytes = 32 hex characters.

```bash
printf '%s' '{"vendorId":"ACME-001"}' | shasum -a 256 | cut -c1-32   # prints the hash; prepend "vendor-"
```

| Feature type | `properties` | Slug |
|---|---|---|
| `vendor` | `{"vendorId":"ACME-001"}` | `vendor-45a62423116103b32d28ee02507f2eeb` |
| `vendor` | `{"vendorId":"GLOBEX-002"}` | `vendor-fc0427f8993d7dcaa1156510165a4f21` |
| `document-type` | `{"documentType":"invoice"}` | `document-type-6e49d6c90d404a72f018e53184fa7d5d` |

Four ways a hand-computed hash silently diverges:

- **Types matter.** `vendorId: 1001` hashes `{"vendorId":1001}`; `"1001"` hashes `{"vendorId":"1001"}` — different slug. Quote anything you mean as a string.
- **`<`, `>` and `&` are JSON-escaped** to `\u003c`, `\u003e`, `\u0026` before hashing, so `properties: {vendorId: "A&B"}` really hashes the bytes `{"vendorId":"A\u0026B"}` (= `459f390f1d07cda1082c3b8e85cde201`, not the hash of the literal text). Let the server compute these.
- Absent or empty `properties` hashes `{}`; an empty feature-type slug falls back to `feature`.
- The server **never checks a supplied slug against the hash** — it computes one only when you send none, and never recomputes on update. A wrong slug is accepted and simply never matches.

`kdx apply` / `kdx sync push` recompute and overwrite the slug on a **standalone** `knowledgeFeature` file (`Recomputed feature slug: X -> Y`), so a wrong
value there is harmless — but the key must be present, because `kdx apply` refuses any file with no `slug`. They do **not** rewrite slugs inside a
hand-authored `featureExpression`; those are yours to get right. Safest route: POST the feature with no `slug`, then copy back the one the server computed.

Editing `properties` after create leaves a stale slug forever: delete and recreate the feature (new slug), then update every expression naming the old one.
Anything mutable — display names, contacts, URLs — belongs in `extendedProperties`, which is not hashed.

## featureExpression is rewritten on every write

```yaml
featureExpression:            # FEATURE carries `slug`, never children. AND / OR / NOT carry `children`.
  type: AND                   # NOT reads children[0] and ignores the rest. Nothing else is a node type.
  children:
    - { type: FEATURE, slug: vendor-45a62423116103b32d28ee02507f2eeb }
    - { type: FEATURE, slug: document-type-6e49d6c90d404a72f018e53184fa7d5d }
```

The tree is rebuilt on save and **invalid nodes are removed, not rejected**, so the file you wrote and the row that got stored differ. A `FEATURE` with no
`slug`, an unknown or missing `type`, and a `NOT` whose children all vanished are each **deleted**; an `AND`/`OR` left with no children is **kept and
evaluates false**; a tree that sanitizes to nothing is stored as NULL. A `FEATURE` slug matching no feature reads as "feature not present" — false, silently.
When a set stops matching after an edit, read it back and diff the stored expression against your file.

## File envelope for `kdx apply`

One resource per file. Required: `type`, `slug`, `orgSlug` — plus `projectSlug` for `knowledgeItem`, which the CLI scopes to a project (`knowledgeitem is
project-scoped: add 'projectSlug' to the file`) even though the entity itself hangs off its set.

```yaml
type: knowledgeSet
orgSlug: my-org
slug: vendor-extraction-rules
name: "Vendor Extraction Rules"
setType: extraction        # ALWAYS set this — see below
```

- **`setType:` is not optional in practice.** Every key in the file is forwarded to the API, and a set has a real persisted `type` column that mirrors into
  the legacy `setType`. Omit `setType:` and the envelope value lands there, persisting `set_type: "knowledgeSet"`.
- `version:` and `ref:` are accepted and ignored — neither is a writable field on any knowledge model.
- Never send `organizationId`, `featureTypeId`, `knowledgeItemTypeId` or `knowledgeSetId`. Use `orgSlug`, `featureTypeRef: <org>/<typeSlug>`,
  `knowledgeItemTypeRef: <org>/<typeSlug>` and `knowledgeSetSlug: <setSlug>`. The CLI deliberately drops `featureTypeId:` — a UUID from another environment
  breaks the foreign key on arrival.
- Slugs must match `^[a-z0-9-]+$`. Mixed case is silently lowercased; **underscores are a 400**. `(organization, slug)` is unique per org for features,
  feature types and item types.

## Inline vs standalone: what each key does

Both styles work. `kdx apply` and `kdx sync push` route three keys on a set file through side channels instead of the metadata payload.

| Key on a `knowledgeSet` file | What actually happens |
|---|---|
| `features: [...]` | Each entry needs its own **computed `slug`** plus `featureTypeRef` — the CLI never computes an inline feature's slug and skips an entry missing either. It is **created only if that slug does not already exist**; an existing feature is skipped, so inline features are effectively immutable. The array is stripped from the metadata payload, so **the CLI does not populate the set's Feature Palette** — only features your expression names appear there. |
| `knowledgeItems: [...]` | Created/updated by `slug`, **then reconciled: any item on the server whose slug is not in the array is DELETED.** Omit the key entirely to leave server items alone. Never ship a partial array. |
| `items: [...]` | **Not a key.** Silently ignored. |
| `attachments: [{attachmentId, attachmentPath}]` | Each file is SHA-256 compared with the server copy and uploaded on mismatch. `attachmentPath` must be a **file** — set-level attachments are not zipped. |

**Inline items are rebuilt from a fixed key list.** An inline `knowledgeItems[]` entry keeps `slug`, `title`, `description`, `properties`, `active`,
`sequenceOrder` and the resolved set/type ids — nothing else. `attachmentId` and `attachmentPath` on an inline item are dropped silently, so per-item
attachments only work from a **standalone** `knowledgeItem` file. Standalone files stay the better choice when one feature is shared by many sets, or when
items need attachments.

## Knowledge set fields that bite

```yaml
status: ACTIVE     # ACTIVE | INACTIVE | BUILDING | NEEDS_CLARIFICATION | PENDING_REVIEW | ARCHIVED | ERROR
priority: 5        # 0-10, default 5.   projectSlug: <slug> is optional — omit for an org-wide set
```

- **There is no `active:` on a knowledge set.** It exists on features, items, feature types and the project-template embedding — not here. Use `status:`. Only
  `ACTIVE` (or empty, which coalesces to `ACTIVE`) is ever matched against a document.
- **`priority` does not decide what matches.** It orders how items bake into a document — higher first, ties broken by set id — so higher-priority items lead
  every ordered read. Matching ignores it entirely.
- **A plain `PUT` that flips a set into `ACTIVE` is rejected 409** when the stored status is `PENDING_REVIEW` or `BUILDING` — however the set was authored —
  or when the set came from an agent run (`createdByActivityId`) and sits in any non-ACTIVE state, `ARCHIVED` included. Agent-run sets also default to
  `PENDING_REVIEW` instead of `ACTIVE`. The only sanctioned path to ACTIVE from those states is `POST /api/knowledge-sets/{id}/promote`.
- `knowledgeSetSlug` on an item is a **write-time** reference resolved against that item's `orgSlug` (so `orgSlug` is mandatory whenever you use it). The
  set's `knowledgeItems` in API responses come from the foreign key, not from slug matching.

## Options: what silently renders nothing

`options` and `extendedOptions` are **bare YAML lists**, never `{options: [...]}` — the editor reads `options.length` and iterates it, so a wrapper hides
every field behind "No configuration options available". (Older `kdx` builds rejected bare arrays with `Error at "/options": value must be an object`; current
builds emit no such constraint. If you hit that, upgrade `kdx`.)

```yaml
options:
  - name: prompt
    type: markdown          # rich editor + server-side image-escape repair
    label: "Prompt"
    required: true
    properties:
      height: 600px
```

- **An unrecognised `type:` renders a red "Unknown Option Type" banner** instead of the field. Names are exact and camelCase: `taxonomyTaxon`, `taxonLookup`,
  `taxonWithProperties`, `cloudModel`, `dataForm`, `documentLookup`, `modelStore`, `label`. `taxon_label`, `taxon-lookup`, `cloud-model`, `data-form`,
  `document`, `moduleStore` and `taxonomyStore` are **not** types. Full list in `references/schema.md`.
- **`type: text` is a single-line input.** For a textarea add `properties: { lines: 8 }`. For prose in a knowledge item prefer `type: markdown` — the only
  type the server guards, repairing the escaped image syntax rich-text editors reintroduce, and the one that makes `attachment://` images render.
- **`min` / `max` do not exist** on an option and are read by nothing; validate bounds in your consumer. Masking is `type: password`, not `password: true`.
- **`showIf` is JavaScript evaluated with `this` bound to the current values object** — the item's `properties` map (or the row object inside a list of
  objects), not the option. So `showIf: "this.mode === 'line'"` reads the sibling *value* `mode`. An expression that throws is treated as `true`.

## Item types: use the built-ins before inventing one

Processing dispatches on the item type's **slug**. A type you invent is inert until something you wrote consumes it. These five `kodexa/` slugs have a
consumer in the processing pipeline today:

| `kodexa/` slug | Effect | Options |
|---|---|---|
| `taxon-semantic-definition-customization` | Replace/append a data element's prompt | `taxonomyAndTaxon`, `prompt` (markdown), `replace` |
| `taxon-enabled` | Enable/disable a data element | `baseTaxonomyAndTaxon`, `enabled` |
| `taxon-name-customization` | Override a data element's label | `baseTaxonomyAndTaxon`, `override_name` |
| `taxon-selection-option-enabled` | Enable/disable one selection option | `taxonomyAndTaxonAndSelectionOption`, `enabled` |
| `project-extraction-model` | Override the extraction model | `extraction_model` |

`kodexa/base-prompt-customization` and `kodexa/document-note` also ship, but nothing reads their slugs — an item of either type is stored and baked into the
document, then ignored.

A `taxonomyTaxon` value is the string `orgSlug/taxonomySlug//taxonPath`. Knowledge **item** types have `slug`, `name`, `description`, `options`, `impact` and
`supportsAttachment` — nothing else. `color`, `icon`, `active` and `labelJsonPath` belong to **feature** types; set on an item type they are dropped without a
word.

## Declared but inert

Persisted and round-tripped, read by nothing that changes behaviour:

- **`impact`** on an item type (`PROMPT_OVERRIDE`, `BASE_PROMPT_OVERRIDE`, `TAXON_ENABLE`, `NAME_OVERRIDE`, `SELECTION_OVERRIDE`, reserved `LOOKUP`/`DERIVE`)
  — a declaration for tools that catalogue item types. Processing keys on the type **slug**, so `impact` does not make a custom type effective.
- **Option keys `featureFlag`, `subType`, `tabName`, `displayProperties`, `aliases`, `overviewMarkdown`, `supportArticle`** — accepted, stored, never rendered
  in a knowledge editor. `showOnPopup` is read only by the project-template dialogs, not by knowledge.
- **`version:` and `ref:`** in the file envelope — neither is a writable field on any knowledge model.
- **Set metadata `template`, `deprecated`, `publicAccess`, `extensionPackRef`** — inherited columns, no knowledge behaviour.
- **`AppliedKnowledge` (`/api/applied-knowledges`, per item)** — a CRUD surface nothing in the platform's own processing path writes. Per-application tracking
  is `AppliedKnowledgeSet`.
- **`clauses` / `featureUuid`** — the pre-expression matching form, a fallback only when `featureExpression` is null; `featureUuid` is a project-template
  shape. Author neither.
