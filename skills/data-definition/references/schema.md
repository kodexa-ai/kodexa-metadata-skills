# data-definition — field reference

Everything a data-definition YAML can carry. Field order below matches the canonical order the
platform writes on push, so a file authored in this order stays stable across pull/push cycles.

## Taxonomy (top level)

The envelope keys (`type`, `slug`, `name`, `description`, `version`, `publicAccess`, `deprecated`,
`template`, `icon`, `imageUrl`, `overviewMarkdown`, `provider`, `providerUrl`, `providerImageUrl`)
are shared with every org-scoped resource — see the `kdx-cli` skill. On top of those a taxonomy adds
exactly four keys:

| Key | Notes |
|---|---|
| `taxonomyType` | Free-form string. `CONTENT` (author this), `PROCESSING`, `MODEL`. Data grids, tree views and taxon pickers filter on `CONTENT`; a `PROCESSING`/`MODEL` taxonomy is treated as "processing" and kept out of the labelling surfaces. `MODULE` is not a value anything reads. |
| `enabled` | Whether the definition is active. |
| `externalDataTaxonomyRefs` | String array. Usually `[]`. |
| `taxons` | The tree. |

Server-managed keys you will see on a pull and should preserve verbatim: `id`, `uuid`, `ref`,
`orgSlug`, `organizationId`, `createdOn`, `updatedOn`, `changeSequence` (optimistic lock — a stale
value gets the write rejected), `extensionPackRef`.

There is **no** taxonomy-level `validationRules` or `conditionalFormats`. Decoding ignores unknown
top-level keys, so those are accepted and dropped.

## Taxon

```yaml
- name: line_total                # required — attribute tag; feeds the derived `path`
  externalName: LineTotal         # required — the formula/JSON-Schema key
  label: Line Total               # display; snake_cased into the extraction property name
  description: ""                 # human documentation; never seen by the extraction model
  overviewMarkdown: ""            # inert at taxon level
  taxonType: CURRENCY
  group: false
  enabled: true                   # a disabled taxon is left out of extraction and of the JSON-Schema export
  order: 3                        # position among siblings
  valuePath: FORMULA              # required
  dataPath: ""                    # companion to valuePath: DATA_PATH — see "Declared but inert" in SKILL.md
  semanticDefinition: "{./Quantity} * {./UnitPrice}"
  metadataValue: ""               # companion to valuePath: METADATA
  cardinality: ""                 # do not author — see SKILL.md
  multiValue: false               # leaf: may this attribute hold several values
  nullable: false                 # inert
  nullValue: ""                   # inert
  notUserLabelled: false          # hide from labelling / form trees
  userEditable: true              # inert — see "Declared but inert" in SKILL.md
  color: "#4F46E5"                # tag highlight colour
  path: ""                        # DERIVED — never author
  generateName: false             # UI flag: name is a generated UUID
  nodeTypes: []                   # e.g. ["line"] — see below
  synonyms: []                    # inert
  lexicalRelations: []            # inert at taxon level (see selectionOptions)
  additionContexts: []
  typeFeatures: {}
  properties: { width: 200, alignment: right, visible: true }
  selectionOptions: []
  useSelectionOptionFormula: false
  selectionOptionFormula: ""
  options: []                     # extra per-tag properties, surfaced in the tag properties panel
  conditionalFormats: []
  validationRules: []
  eventSubscriptions: []          # group taxons only
  children: []
```

`selectionOptions`, `validationRules`, `conditionalFormats` and `eventSubscriptions` are **taxon-level
keys**, siblings of `typeFeatures` — never members of it. `typeFeatures` has a forward-compatible
catch-all for unknown keys, so anything misfiled there is stored without error and never read.

`nodeTypes` has exactly one consumer: when its first entry is `"line"`, hand-tagging this taxon in
the document viewer tags the whole line node instead of the selected words. Everything else in the
array is ignored. See SKILL.md's "Declared but inert" table for the fields with no consumer at all.

## `taxonType`

Fourteen wire values: `STRING`, `DATE`, `DATE_TIME`, `NUMBER`, `BOOLEAN`, `CURRENCY`, `URL`,
`EMAIL_ADDRESS`, `PHONE_NUMBER`, `SELECTION`, `PERCENTAGE`, `FORMULA`, `DERIVED`, `SECTION`.

The editor offers the first twelve. `FORMULA` and `DERIVED` are schema-valid but are not how you
author a computed field — set `valuePath: FORMULA` / `valuePath: DERIVED` instead. The document
runtime additionally defines `INTEGER`, `DECIMAL` and `GROUP`, which are **not** in the API enum;
do not author them. Mark a container with `group: true`, not a type.

`SECTION` is a visual grouping that stores nothing: it is excluded from the extraction schema and
from data grids, and the editor hides its Prompt, Formatting and Validations tabs.

### A typed taxon coerces, it does not reject

Giving a taxon a type does not make its value trustworthy. Extraction stores **two** representations
of an attribute — the literal `value` / `stringValue`, and a typed companion (`decimalValue`,
`dateValue`) — and they can disagree. A value the normalizer cannot parse still lands, with the typed
companion at its zero value and `confidence` untouched:

```json
{ "path": "invoice/total_amount", "value": "<UNKNOWN>", "decimalValue": 0, "confidence": 1 }
```

A consumer reading `decimalValue` sees a confident `0`; a consumer reading `value` sees a non-number;
nothing in the record marks it failed. The same shape applies to every typed taxon — an ambiguous
separator, a number carrying a unit or a stray marker, a date missing a component. A partial date can
land as year `0000` rather than as an error or an absence, which no "is the field populated" check
will catch.

The platform *can* surface this as an `Unable to Normalize Value` content exception at `WARNING`
severity, but whether it fires depends on the extraction path in use, so never read its absence as a
pass.

Two consequences:

- **Assert on the typed value, never on non-emptiness.** In a `data-export`, read `dateValue` /
  `decimalValue` alongside the string form; "all fields populated" is not a passing test.
- Where the source format is genuinely ambiguous, extract to a `STRING` taxon and derive the typed
  one deterministically in a SCRIPT step (**activity-plan**) rather than asking the model to
  normalize it.

## `valuePath`

| Value | Meaning | Companion |
|---|---|---|
| `VALUE_OR_ALL_CONTENT` | default — the tagged value if present, else all tagged content | `semanticDefinition` (the extraction prompt) |
| `VALUE_ONLY` | only the tagged value | `semanticDefinition` |
| `ALL_CONTENT` | all tagged content regardless of value | `semanticDefinition` |
| `DATA_PATH` | nothing reads `dataPath`; the taxon still goes to the extractor with no prompt | `dataPath` (inert) |
| `METADATA` | a document property rather than page content | `metadataValue` |
| `FORMULA` | computed and kept live by the formula engine | `semanticDefinition` holds **the formula text** |
| `DERIVED` | still offered to the extraction model, but the returned value is taken as-is: normalization is skipped and the cell is badged **Derived** in the grid | — |
| `REVIEW` | legacy; excluded from extraction and prompt assembly, absent from the runtime enum | — |

The editor offers four: `VALUE_OR_ALL_CONTENT` (labelled "Document"), `METADATA`, `DERIVED`,
`FORMULA`. `EXTERNAL` and `EXPRESSION` were removed along with the Groovy/SpEL evaluators; they still
decode (the field is an unvalidated string) and then the taxon is silently never extracted.

`metadataValue`: `FILENAME`, `TRANSACTION_UUID`, `CREATED_DATETIME`, `DOCUMENT_LABELS`, `OWNER_NAME`,
`DOCUMENT_STATUS`, `PAGE_NUMBER`.

## `typeFeatures` — display and formatting

Read by the UI. Several ride the forward-compat catch-all rather than typed fields, but all
round-trip cleanly and all are bound in the editor.

| Key | Applies to | Effect |
|---|---|---|
| `longText` | STRING | render as a multi-line text cell |
| `maxTextRows` | STRING + `longText` | rows in that cell |
| `markdown` | STRING | only read alongside `summarize: true`, where it tells the summarizer to emit markdown; no UI renderer reads it |
| `showFullOnHover` | STRING | show the full value as a tooltip when the cell is truncated |
| `overrideWidth` | any | boolean — use `displayWidth` instead of the auto-fit column width |
| `displayWidth` | any | pixel width, honoured when `overrideWidth` is true |
| `normalizeDate` | DATE, DATE_TIME | normalize for presentation |
| `normalizeDateInExport` | DATE, DATE_TIME | normalize in JSON/XML exports |
| `dateFormat` | DATE, DATE_TIME | target format, e.g. `yyyy-MM-dd` |
| `truncateDecimal` | NUMBER | round/truncate to `decimalPlaces` |
| `decimalPlaces` | NUMBER | places kept |
| `preferTwoDecimalPlaces` | CURRENCY | on entry, read the last two digits as decimals (`1234` → `12.34`) |
| `currencyCode` | CURRENCY | **inert** — declared and round-tripped, but nothing reads it from the taxon; currency formatting comes from document format hints |
| `stringExtract` | STRING, SELECTION, URL, EMAIL_ADDRESS, PHONE_NUMBER | regex — **keeps only** matching characters |
| `stringReplace` | same | regex — **removes** matching characters |
| `serializeGroupAsObject` | group | export the group as a JSON object rather than an array |

When both string filters are set, `stringExtract` runs first. An invalid regex logs a warning and
leaves the value untouched. The original value is preserved separately.

## `typeFeatures` — extraction and chunking

| Key | Values / default | Effect |
|---|---|---|
| `expected` | bool, `false` | adds this property to the extraction schema's `required` list |
| `skipExtraction` | bool, `false` | leave the taxon out of extraction |
| `cardinality` | `single` \| `multiple`, group taxons only | one object vs an array. **Always set it.** |
| `embedded` | bool, `false` | extract inside the parent's call rather than as its own chunk |
| `chunkingStrategy` | `document`, `page`, `firstNPages`, `record`, `classifiedContent` (default), `pageClassifiedContent`, `consecutiveClassifiedContent`, `groupClassifiedContent` | how the document is split for the model. The editor exposes it only for non-embedded taxons. |
| `nPages` | int | pages taken by `firstNPages` |
| `promptStrategy` | `lines`, `layout` (default), `imageAndBoundingBoxes`, `boundingBoxes` | how content is presented to the model |
| `classificationStrategy` | `none`, `dataElement` (default), `dataElementAndChildren`, `feature`, `pageLabel` | how content is classified before extraction |
| `classificationContent` | `text` (default) \| `image` | what classification looks at |
| `includeExplanation`, `ignoreNonWords`, `restrictClassification`, `rerank`, `maxPagesFromRerank` | classification tuning |
| `includeImages`, `imageWidth` | send page images alongside text |
| `overrideExtractionModel`, `extractionModel`, `enableThinkingMode` | model selection for this taxon |
| `allowTemplating` | bool, `false` | render `semanticDefinition` as a Jinja2 template before the model sees it |
| `tagPage`, `labelDocument` | tag/label side effects during chunking |
| `merge`, `mergeWithAI`, `mergeInstructions`, `mergeInstances`, `instanceBoundaryThreshold` | how results from multiple chunks are combined |
| `enableStructureReview`, `structureReview`, `enableStructureReviewThinkingMode` | post-extraction structure review |
| `summarize`, `enableLineFallback`, `raiseExceptionOnFallback`, `enableAiFallback`, `raiseExceptionOnAiFallback` | normalization and fallback behaviour |
| `contextHeadLines`, `contextTailLines`, `maxChildLines`, `hierarchyMaxLinesPerCall`, `hierarchyWindowOverlap`, `planningModel`, `enablePlanningThinkingMode` | extraction-planner tuning |

### `allowTemplating`

```yaml
typeFeatures:
  allowTemplating: true
semanticDefinition: |
  The invoice total in {{ metadata.currency | default('USD') }}.
  {% if external_data.vendor %}The vendor should be {{ external_data.vendor.name }}.{% endif %}
```

Context variables: `external_data`, `metadata`, `knowledge`. Undefined values chain to `''` instead
of raising; a template error logs a warning and the raw string is used. This is the **only**
templating mechanism in the extraction path.

## `additionContexts` — record chunking

Consulted only when `typeFeatures.chunkingStrategy` contains `record`, and only when a context of
the strategy's own trigger type is present. The two strategies are mutually exclusive: keep one kind
per taxon, because the definition strategy concatenates every context it finds into its prompt.

| `type` | Consumed by | `context` holds |
|---|---|---|
| `RECORD_DEFINITION` | the LLM record-definition strategy (needs at least one entry of this type) | **prose** describing what one record is |
| `RECORD_SECTION_STARTER_MARKER` | the marker strategy (needs at least one `RECORD_START_MARKER`) | regex, matched **anywhere** in a line — narrows to a section first |
| `RECORD_SECTION_END_MARKER` | same | regex, matched anywhere in a line |
| `RECORD_START_MARKER` | same | regex, matched **anchored at the start** of a line |
| `RECORD_END_MARKER` | same | regex, matched anchored at the start of a line |

A context with no `type` defaults to `RECORD_DEFINITION`. Both strategies apply to spatial documents.

```yaml
# (a) let the model find the boundaries
typeFeatures: { cardinality: multiple, chunkingStrategy: record }
additionContexts:
  - type: RECORD_DEFINITION
    context: "Each record is one row of the invoice table: description, quantity, unit price, line total."

# (b) deterministic boundaries — regexes, and the record markers must match from column 0
typeFeatures: { cardinality: multiple, chunkingStrategy: record }
additionContexts:
  - { type: RECORD_SECTION_STARTER_MARKER, context: "ITEM\\s+DESCRIPTION" }
  - { type: RECORD_START_MARKER,           context: "^\\s*\\d+\\s+" }
  - { type: RECORD_END_MARKER,             context: "^\\s*TOTAL\\b" }
  - { type: RECORD_SECTION_END_MARKER,     context: "SUBTOTAL" }
```

## `validationRules`

Eleven keys per rule, in canonical order:

| Key | Notes |
|---|---|
| `name` | human label; also the rule's identity — with `exceptionId` empty the exception key is derived as `<taxon path>/<rule name>` |
| `description` | optional documentation |
| `disabled` | `true` skips evaluation entirely |
| `conditional` | **must be `true` for `conditionalFormula` to be consulted at all** |
| `conditionalFormula` | KEXL. Cleanly false ⇒ the rule is skipped. An *erroring* conditional is treated as a configuration error, not a skip. |
| `ruleFormula` | KEXL. Truthy ⇒ pass. Empty ⇒ the rule is skipped. |
| `messageFormula` | KEXL producing the user-facing message. A plain message is a KEXL **string literal**: `'"Total does not match"'` — outer YAML quotes, inner KEXL quotes. |
| `detailFormula` | KEXL producing the longer detail text shown with the exception |
| `exceptionId` | optional exception-type id; when empty the derived key above is used, so two rules on one taxon do not collide |
| `supportArticleId` | optional help-article link on the exception |
| `overridable` | defaults to `false` (hard block). `true` ⇒ a reviewer can dismiss it. |

A failing rule creates a **data exception** on each affected data object carrying the message,
details, `overridable` and the support-article link; it closes again when the rule passes. A rule
whose formula errors still produces an exception, but with the message
`Validation rule configuration error` and details naming the evaluation error — that is how a bad
function name or a mistyped reference surfaces, per document, at review time.

Rules are legal on group taxons as well as leaves, which is where cross-field rules belong.

## `conditionalFormats`

Only `type`, `condition` and `properties` are read; see SKILL.md for the shape and the flat-form
trap. Four types, and what each does to the cell:

| `type` | Rendering | `properties` |
|---|---|---|
| `backgroundColor` | `background-color` | `color` |
| `outlineColor` | a 2px solid border | `color` |
| `textColor` | text `color` | `color` |
| `icon` | the attribute's status icon | `icon`, `color` |

Icon ids: `alert-box`, `alert-box-outline`, `alert-circle`, `alert-circle-outline`.
Suggested `properties.color` values: `#FEE2E2` red, `#FEF3C7` amber, `#D1FAE5` green, `#DBEAFE` blue.
A format created in the editor starts with a placeholder type and stays inert until you pick one of
the four.

## `selectionOptions`

For `taxonType: SELECTION`. Canonical key order: `id`, `label`, `value`, `description`, `hint`,
`hintMarkdown`, `isConditional`, `conditionalFormula`, `disabled`, `lexicalRelations`.

| Key | Notes |
|---|---|
| `id` | opaque row key used by the editor. **Not** the stored value. |
| `label` | the extraction schema constrains the model to the set of enabled option labels, so this is what gets extracted |
| `value` | what the form stores when it must differ from the label; the UI stores `value` if present, else `label` |
| `description` | appended to the option in the extraction prompt |
| `hint`, `hintMarkdown` | UI hint text, optionally rendered as markdown |
| `isConditional` + `conditionalFormula` | show the option only while the KEXL formula evaluates to `true`; an evaluation error leaves the option visible |
| `disabled` | string flag; `"true"` disables the option |
| `lexicalRelations` | `{type, value, weight}`. Types: `SYNONYM`, `ANTONYM`, `HYPERNYM`, `HYPONYM`, `ABBREVIATION`, `VARIANT` — the editor offers the first two and only those two are glossed into the prompt. **One entry per term.** |

### Dynamic options

```yaml
taxonType: SELECTION
useSelectionOptionFormula: true
selectionOptionFormula: 'serviceBridgeCall("erp-lookup", "currencies", "country", {../Country})'
```

`serviceBridgeCall` takes the bridge ref, the endpoint name, then alternating argument key/value
pairs. The formula must return an array of `{label, value}` objects, or an array of plain strings
used as both label and value. The result is stored on the owning data object and re-evaluated when a
referenced attribute changes — which is why `{../Sibling}` references are the point of the pattern.
Use this rather than a `semanticDefinition` with a `setDataFeature` wrapper.

## `eventSubscriptions` — group taxons only

```yaml
group: true
eventSubscriptions:
  - name: derive-region                                     # unique within the taxon
    on: "changed:dataAttribute:(vendor_city|vendor_state)"  # regex, compiled anchored as ^(?:…)$
    disabled: false
    script: |
      if (!currentObject) return;
      var city = currentObject.getFirstAttributeValue("vendor_city");
      var state = currentObject.getFirstAttributeValue("vendor_state");
      currentObject.setAttribute("region", city + ", " + state);
```

- **Nothing here is checked when the definition is saved.** The API stores whatever you write; the
  runtime indexes subscriptions only from **group** taxons, and only when `on` is non-empty and
  compiles as a regex. A subscription on a leaf taxon, or with a broken `on`, is simply never
  indexed — it fails by never firing, not by refusing the save.
- The `on` regex is matched against an event string. Recognised forms:
  `changed:dataAttribute:<tag>`, `focus:dataAttribute:<tag>`, `blur:dataAttribute:<tag>`,
  `formLoaded:<form>`, `trigger:<name>`, `loaded:dataObject`, `created:dataObject`.
- Write `on` explicitly. `dependsOn: [a, b]` is a pre-regex shorthand that the runtime index does not
  expand, so a subscription carrying only `dependsOn` never matches anything.
- `disabled: true` skips execution.
- Scripts are JavaScript with `document`, `currentObject` and `event` pre-bound. Read and write
  attributes through `currentObject.getFirstAttributeValue(name)` and
  `currentObject.setAttribute(name, value)` — **there is no `bridge.data.*`**. Also available:
  `log.*` / `console.*`, `bridge.notify(title, message)`, `bridge.events.fire(eventString, [objId])`
  and `serviceBridge.call(...)`. Self-recursion, once-per-cascade and maximum-depth guards stop
  runaway loops.

## `properties`

A small display block, separate from `typeFeatures`: `width` (int), `alignment` (string),
`visible` (bool), plus a catch-all that round-trips unknown keys. Nothing in the platform reads it —
column width comes from `typeFeatures.overrideWidth` + `displayWidth`.
