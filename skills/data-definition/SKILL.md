---
name: data-definition
description: "Use when creating, editing, or debugging a Kodexa data-definition (taxonomy) — the YAML that defines taxons, data types, extraction prompts (semanticDefinition), typeFeatures, repeating groups and line items, KEXL formulas and validation rules, conditional formatting, selection options and event subscriptions. Covers data-definition:// and taxonomy:// resources applied with kdx."
---

# Kodexa data-definition authoring

A **data-definition** ("taxonomy" is an accepted alias) is a document's extraction schema: a tree of
**taxons** saying what to capture, how to compute it, how to validate it and how to colour it.
Org-scoped, stored at `<metadata-dir>/data-definitions/<slug>.yaml`, referenced as
`data-definition://<orgSlug>/<slug>` or `taxonomy://<orgSlug>/<slug>` — both resolve against
`kdxa_taxonomies` to `/api/data-definitions/{id}`. See `references/schema.md` (fields),
`references/formulas.md` (KEXL) and `references/examples.md`.

## The envelope — three keys, or `kdx apply` never reaches the server

```yaml
type: dataDefinition        # REQUIRED. aliases: data-definition, dataDefinition, taxonomy, taxonomies
slug: invoice-extraction    # REQUIRED, unique within the org
orgSlug: acme-corp          # REQUIRED
name: Invoice Extraction
version: 1.0.0
description: Structured data captured from supplier invoices.
taxonomyType: CONTENT       # CONTENT | PROCESSING | MODEL — free-form string, only these are read
enabled: true
externalDataTaxonomyRefs: []
taxons: []
```

`type` and `orgSlug` may instead come from `--type` / `--org-slug`, but `slug` must be in the file.
`CONTENT` is what data grids, tree views and taxon pickers look for; `PROCESSING`/`MODEL` mark a
taxonomy non-content and hide it from labelling — **there is no `MODULE`**. And **a taxonomy has no
top-level `validationRules:` or `conditionalFormats:`** — unknown top-level keys are ignored on
decode, so a rules array written there is accepted with a 2xx and then silently vanishes. Rules and
formats live on taxons only: put a cross-field rule on whichever taxon should own the exception and
navigate with `{../Sibling}` / `{Group/Child}`.

## Naming — four fields, four subsystems

`name`, `externalName` and `valuePath` are the **required** taxon properties; `label` is not.

| Field | What it actually drives |
|---|---|
| `name` | the data-attribute tag, and therefore `path`. Activity-plan script helpers resolve against it. |
| `externalName` | **the key every `{...}` formula reference resolves against** — validation rules, conditional-format conditions, selection-option formulas — and the JSON-Schema export key. Falls back to `name` when blank. |
| `label` | display, **and** the extraction property name: the schema handed to the model keys each taxon by a snake_cased form of its label. Label text is prompt surface. |
| `path` | **derived.** Recomputed server-side from the `name` chain on every read and write, and stripped as derived on push. Never author it. |

The editor derives `externalName` from `label` by stripping non-alphanumerics ("Line Total" →
"LineTotal"). Set it on **every** taxon — a formula naming a taxon that does not exist never errors.

## A taxon

```yaml
- name: invoice_number            # required
  externalName: InvoiceNumber     # required — the formula key
  label: Invoice Number
  taxonType: STRING               # 14 wire values, 12 in the editor (references/schema.md)
  valuePath: VALUE_OR_ALL_CONTENT # required — 8 wire values, 4 in the editor
  description: "Human documentation. The extraction model NEVER sees this."
  semanticDefinition: "The AI-facing extraction instruction — this IS the prompt."
  typeFeatures: { expected: true }   # adds the property to the extraction schema's `required` list
```

Under `valuePath: FORMULA`, `semanticDefinition` holds **the formula text itself**; it is skipped
entirely for `METADATA`, `DATA_PATH`, `FORMULA` and `REVIEW`.

## Groups and repeating groups

**Hang everything under one root group.** A data object exists per *group* taxon, and a rule or
formula runs against the data object at its own taxon's path — so a bare `{Sibling}` reference only
resolves between taxons that share a group. Give the definition one `group: true` root.

```yaml
- name: line_items
  externalName: LineItems
  label: Line Items
  valuePath: VALUE_OR_ALL_CONTENT
  group: true                     # container — holds children, stores no value of its own
  typeFeatures: { cardinality: multiple }   # single | multiple — ALWAYS set this on a group
  children: [...]
```

`typeFeatures.cardinality` decides whether a group produces one object or an array. Three call sites
disagree about what an *unset* value means, so **state it explicitly on every group**; `multiValue`
is the leaf equivalent and its unset default also differs by layer, so state that too.

**Never write `cardinality: {min: 1, max: 100}`.** Taxon-level `cardinality` is a string; an object
there aborts the whole decode with `cannot unmarshal object into ... TaxonCardinality`, so the request
is rejected and nothing is written. Avoid taxon-level `cardinality` altogether — the API types it
`ONCE_PER_DOCUMENT | MULTIPLE_PER_DOCUMENT | ONCE_PER_SEGMENT | MULTIPLE_PER_SEGMENT`, which is a
different vocabulary from the `single`/`multiple` one that actually drives shape.

`additionContexts` (record-boundary hints) are read **only** when `typeFeatures.chunkingStrategy`
contains `record` *and* a context of the matching type is present — without both the block is dead
configuration, no warning. The four `*_MARKER` values are **regexes**, only `RECORD_DEFINITION` takes
prose (`references/schema.md`).

## Formulas (KEXL) — the shape that parses

One language across `ruleFormula`, `conditionalFormula`, `messageFormula`, `detailFormula`,
`conditionalFormats[].condition`, `selectionOptionFormula` and `semanticDefinition`-under-`FORMULA`.

```yaml
# ✅
ruleFormula: "!isblank({InvoiceNumber})"
ruleFormula: "abs({TotalAmount} - ({Subtotal} + ifnull({TaxAmount}, 0))) < 0.01"
conditionalFormula: "!isblank({DueDate}) && !isblank({InvoiceDate})"
messageFormula: '"Invoice number is required"'   # KEXL string literal inside YAML quotes
# ❌ parse or evaluation failures
ruleFormula: "NOT_EMPTY(invoice_number)"             # no such function; bare identifiers don't resolve
ruleFormula: "line_items.quantity > 0"               # "." outside "./" is a parse error
conditionalFormula: "NOT_EMPTY(a) AND NOT_EMPTY(b)"  # AND/and/OR/or are not operators — use && and ||
ruleFormula: "{Status} == 'PAID'"                    # "==" is a parse error — equality is a single "="
```

- **References are `{Braces}` with `/` separators**, resolved against the `externalName` chain:
  `{InvoiceNumber}`, `{LineItems/LineTotal}`, `{./Sibling}`, `{../Parent}`, `{/Absolute/Path}`.
  **Equality is a single `=`**; `!=`, `<`, `>`, `<=`, `>=` are normal; logic is `&&` / `||` / `!`.
- **`NOT_EMPTY`, `IS_EMPTY`, `COALESCE`, `TODAY`, `DATE_ADD`, `REGEX_MATCH`, `IN`, `ALL_VALUES`,
  `UNIQUE` and `NOT` do not exist.** The registry is fixed — `isBlank`, `ifNull`, `regex`, `dateMath`,
  `count`, `sum`, … matched case-insensitively. Full list and a translation: `references/formulas.md`.
- **A reference that resolves to nothing does not error** — it comes back null or an empty list, so a
  misspelled `externalName` quietly makes `sum(...)` return 0 and `isblank(...)` true. An unknown
  function *does* error, but only per document at review time, as a data exception reading
  `Validation rule configuration error`. Neither is caught at apply time.

**To check a group has rows, reference the group itself from its parent group** — the parent still
has a data object when the child has no rows. `!isblank({LineItems})` there works because
`{LineItems}` resolves to the list of child data objects and `isblank` is true for an empty list;
`{LineItems/SomeChild}` resolves against those children, so it is not an emptiness test.

## Conditional formats — nested, or silently inert

```yaml
conditionalFormats:                                # type: backgroundColor | outlineColor | textColor | icon
  - { type: backgroundColor, condition: "{TotalAmount} > 10000", properties: { color: "#FEF3C7" } }
  - { type: icon, condition: "{TotalAmount} > 50000", properties: { icon: alert-circle, color: "#B91C1C" } }
```

The runtime reads **only** `type`, `condition` and `properties`. A flat entry (`{name, formula,
backgroundColor, textColor, fontWeight, icon}`) decodes without error, the colour/weight keys are then
dropped on ingest and `formula` is stored but never read — nothing renders, nothing complains. An
empty `condition` is skipped; formats paint cells, so use leaf taxons.

## Selection options — the model is constrained to the LABEL

For `taxonType: SELECTION`, the extraction schema's enum is the set of enabled option **labels**. `id`
is an opaque row key and is never the stored value; add `value:` only when the UI must store something
other than the label. Write the label exactly as it should appear in extracted data. Each option's
`description` and `lexicalRelations` are injected into the extraction prompt (only `SYNONYM` and
`ANTONYM` get a gloss) — use them for the wordings real documents use, **one entry per term**, because
a comma-packed value reads as a single phrase. Dynamic dropdowns: `references/schema.md`.

## Declared but inert

Persisted and round-tripped, sometimes editable in the UI, but nothing in the platform reads them.

| Key | Note |
|---|---|
| `typeFeatures.formulaExpression`, `typeFeatures.currencyCode` | No consumer. `valuePath: FORMULA` reads `semanticDefinition`; currency formatting comes from document format hints. |
| taxon `userEditable`, `nullable`, `nullValue`, `synonyms`, `overviewMarkdown` | Carried, and some are editable in the taxonomy editor, but nothing reads them. `userEditable` in particular does **not** make a cell read-only — the grid looks for it under a key a taxon does not have, so editability comes from the grid context alone. |
| taxon `properties` (`width`, `alignment`, `visible`) | Round-trips, no consumer. Column width comes from `typeFeatures.overrideWidth` + `displayWidth`. |
| taxon-level `lexicalRelations` | Editable in the Classification tab, but only `selectionOptions[].lexicalRelations` reaches the prompt. |
| `eventSubscriptions[].dependsOn` | A pre-regex shorthand. The runtime indexes subscriptions on `on` alone and never expands `dependsOn`, so write the `on` regex yourself. |
| taxon `dataPath` (+ `valuePath: DATA_PATH`) | Never read. The taxon still goes to the extractor, with no prompt. |
| `typeFeatures.skipExtraction` | Declared as "leave the taxon out of extraction", and `kodexa/llm-taxonomy-model` extracts the taxon anyway — observed populating reviewer-input taxons, including a `SELECTION` holding the reviewer's own conclusion, on every document of a corpus. The prompt is the lever that works: give the taxon a `semanticDefinition` telling the model the field is computed after extraction and to return no value. |
| `valuePath: EXTERNAL` / `EXPRESSION` / `REVIEW` | `EXTERNAL`/`EXPRESSION` are removed value paths (the Groovy/SpEL evaluators are gone); `REVIEW` is legacy and absent from the runtime enum. All three save cleanly, then the field is never extracted, never normalized and gets no prompt. `REVIEW` is **not** a template mechanism — that is `typeFeatures.allowTemplating`. |
| conditional-format `name`/`formula`/`style`/`color`/`background`/`icon` | Round-trip compatibility only; the evaluator reads `condition`, the renderer reads `properties`. |
| `apiVersion:` in the file | Present in the shipped templates; `kdx apply` does not read it. |

## Common mistakes

| Mistake | What happens |
|---|---|
| Applying a hand-authored taxonomy whose taxons have no `id` | Extraction works, the activity completes green and the data export looks perfect — and **every field in a data form renders `No data`**. `kdx apply` never mints taxon ids, and once ids are the only difference it reports `unchanged` and sends nothing, so they have to go through a direct PUT. `references/schema.md`, "Taxon `id`". |
| An unquoted short `selectionOptions` id — `no`, `yes`, `on`, `off` | Applies through `kdx` (Go YAML 1.2 reads them as strings) and then any Python round-trip of the file rewrites it as `false` (PyYAML, YAML 1.1), after which the server rejects the whole write. Quote option ids. |
| No `type:`/`orgSlug:` in the file and no `--type`/`--org-slug` | The CLI refuses before contacting the server. |
| `cardinality: {min, max}` on a group | Hard decode failure — the request is rejected outright. |
| Leaf taxons at the root, outside any group | No shared data object, so bare `{Sibling}` references between them do not resolve. Put one `group: true` root above everything. |
| Rules or formats at taxonomy level | Accepted, then silently discarded. Move them onto a taxon. |
| Flat `conditionalFormats` (`backgroundColor:`, `fontWeight:`) | Accepted, renders nothing. |
| `conditionalFormula` without `conditional: true` | Never consulted; the rule fires unconditionally. |
| `typeFeatures: { selectionOptions: [...] }` | It is a taxon-level key. Stored in the forward-compat catch-all and never read — a SELECTION field with no options. |
| Dotted group paths (`line_items.quantity`) | Parse error; segments are `/`-separated inside braces. |
| `additionContexts` with no `chunkingStrategy: record` | Dead configuration, no warning. |
| Authoring `path:` on a taxon | Overwritten server-side from the `name` chain, and stripped on push. |
| `taxonType: GROUP`/`INTEGER`/`DECIMAL` | Not in the API enum. Mark containers with `group: true`. |

## Verifying

`kdx validate -f <file>` checks the file against the API's create schema without sending anything —
but that schema nests the taxonomy body one level down, so **the taxons are never type-checked** (they
surface as unrecognized-key warnings). Validate confirms the envelope, not the tree. Anything inside
`taxons` — the `cardinality` shape above most of all — is caught only when `kdx apply -f <file>` posts
it and the server refuses the body; formulas are checked at neither point. `kdx dataclasses <file>`
generates Python dataclasses from the taxonomy, and `GET /api/data-definitions/{id}/json-schema`
returns a draft-07 schema of the **enabled** taxons keyed by `externalName` ("Download JSON Schema").

Related skills: `kdx-cli` (apply envelope, `kdx get`/`kdx run`), `project-resource` (binding a
definition into a project), `data-form` (forms bind to taxon paths and consume selection options),
`activity-plan` (SCRIPT steps write into taxon paths — see its `references/step-types.md`),
`service-bridge` (`serviceBridgeCall`), `knowledge-system` (runtime overrides of names and prompts).
