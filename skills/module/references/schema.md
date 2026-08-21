# Module schema reference

## Top level vs `metadata:`

A module row has real columns and a `metadata` JSONB blob. The API merges the blob up to the top
level on read, and accepts either shape on write — **but the column always wins on a collision**,
so a key that has a column must be authored at the top level or its value is lost.

### Top-level fields (columns)

| Field | Type | Notes |
|---|---|---|
| `type` | string | Always `module`. Also drives the computed `uri` scheme. |
| `slug` | string | Unique within the organization — the database rejects a duplicate. `kdx apply` refuses a file without one; a direct API create derives it from `name`. |
| `orgSlug` | string | Required by `kdx apply` (it refuses the file without one) and used to resolve the owning organization. |
| `organizationId` | uuid | The FK the API actually stores. The nested `organization: { id: … }` form is also accepted and wins over a flat value. |
| `name` | string | Display name. Also the slug source if `slug` is omitted. |
| `description` | string | The only real prose field on a module. |
| `moduleType` | string | `model` (default) or `skill`. The discriminator. |
| `moduleStatus` | string | `DRAFT` (default), `PUBLISHED`, `DEPRECATED`. Inert — see SKILL.md. |
| `eventAware` | bool | Module handles events. Column — must be top level. |
| `supportsScheduling` | bool | Module can be scheduled. Column — must be top level; the schedule picker filters on it. |
| `deprecated` | bool | `true` hides the module from Studio resource pickers. |
| `publicAccess` | bool | Makes the module visible to every organization. |
| `template` | bool | Marks the record as a template. |
| `extensionPackRef` | string | Set when the module ships inside an extension pack. |
| `createdBy` | string | Free string. |
| `ref`, `uri`, `id`, `uuid`, `changeSequence`, `createdOn`, `updatedOn` | — | Server-computed. Never author. |

### `metadata:` keys (JSONB)

| Key | Type | Purpose |
|---|---|---|
| `moduleRuntimeRef` | string | `orgSlug/slug` of the model runtime. Empty falls back to `kodexa/base-model-runtime`. |
| `moduleRuntimeParameters` | object | `module` — the Python package directory inside the ZIP; `function` — the entry point (default `infer`). The pre-rename spelling `modelRuntimeParameters` is read for old rows but is no longer a field: writing it loses the object. |
| `moduleSidecars` | string[] | Other modules unpacked onto `sys.path` before yours imports. Same story for the pre-rename `modelSidecars`. |
| `bridgeType` | string | `python` (default) or `wasm`. |
| `allowedHosts` | string[] | WASM only. Outbound host patterns the plugin may call; wildcards allowed. Empty means no outbound HTTP. |
| `scriptLanguage` | string | `javascript` or `js` (case-insensitive). A non-empty value that is neither is rejected at run time; omitting it is tolerated, but say it anyway. |
| `script` | string | The inline JavaScript body. |
| `inferable` | bool | Metadata-only capability flag; filtered as `metadata.inferable`. |
| `inferenceOptions` | Option[] | User-configurable parameters. See below. |
| `actions` | Action[] | Named actions an agent can discover and invoke. |
| `contents` | string[] | Glob patterns packaged into the implementation ZIP. Also accepted at the top level. |
| `ignoredContents` | string[] | Exclusion globs. Read **only** from `metadata:`. |
| `build` | Step[] | Client-side pre-package build steps. Consumed by `kdx apply`; not stored on the record. |
| `stateHash` | string | Server-managed. Stamped on every implementation upload so the orchestrator notices the code changed. Do not author it. |
| `state`, `baseDir`, `optionTabs`, `messageTemplates`, `taxonomy`, `additionalTaxonOptions`, `taxonFeatures`, `type` | — | Stored and round-tripped, read by nothing. See "Declared but inert" in SKILL.md. |

Fields that are **not** part of the module model and are dropped on write: `version`, `lifecycle`,
`sourceUrl`, `provider`, `providerUrl`, `providerImageUrl`, `overviewMarkdown`, `trainable`,
`deploymentDefaults`. (Several of these are genuine fields on *other* resource types, which is
where the confusion comes from — `deploymentDefaults` belongs to a `modelRuntime`.)

You will still meet them in first-party manifests. That is because the server keeps the authored
YAML text verbatim alongside the record so `kdx sync pull` round-trips cleanly — the keys survive in
that stored copy, they are simply absent from the module the API serves and from everything that
reads it. Studio's module panel reads `overviewMarkdown` off the module response and always gets
nothing. Put human-facing prose in `description`, which is a real column.

Unknown metadata keys are ignored on write. A *known* key carrying the wrong JSON type is not — for
example `contents: "invoice_extractor/**.py"` instead of a list rejects the whole request with an
"invalid metadata field" error rather than saving half the record.

## Glob patterns

`contents` and `ignoredContents` are evaluated on your machine, relative to the directory holding
the manifest file. ZIP entry names are those same relative paths, so directory structure is
preserved.

```yaml
metadata:
  contents:
    - invoice_extractor/**          # everything under the package, recursively
    - invoice_extractor/**.py       # only .py files, recursively
    - templates/**
    - config.yml                    # a single file
  ignoredContents:
    - "**/__pycache__/**"
    - "**/*.pyc"
    - "**/tests/**"
```

An empty match set is a hard error, not a warning.

## Inference options

```yaml
inferenceOptions:
  - name: confidence_threshold      # must match the function parameter name exactly
    type: number
    default: 0.85                   # Studio form seed only, and only when truthy
    label: "Confidence threshold"
    description: "Minimum confidence for an extracted value"
    required: false
    showIf: "this.use_advanced"     # JS expression; `this` is the current values map
    developerOnly: false            # hidden unless the user has developer tools on

  - name: output_format
    type: string
    label: "Output format"
    possibleValues:                 # turns a string/regex option into a select
      - { value: json, label: "JSON" }
      - { value: csv,  label: "CSV" }

  - name: extraction_notes
    type: text
    label: "Notes"
    properties:                     # per-type render options, see below
      lines: 6
```

### Option types

Types resolve either by direct match against the shipped renderers or through an alias map.
Anything else renders an "unknown option type" alert.

| `type` | Renders as |
|---|---|
| `string` | Text input; with `possibleValues`, a select |
| `regex` | Text input; with `possibleValues`, a select |
| `select` | Select. Usually reached via `string` + `possibleValues`, but settable directly |
| `componentLookup` | Generic component picker; the resource-specific aliases below are friendlier |
| `password` | Text input with masking **and the org secret picker** — the UI stores `${secret.<name>}` and the runtime resolves it. Use this for API keys. |
| `text` | Text input; multi-line via `properties: { lines: 6 }` |
| `number` | Numeric input. No client-side range enforcement — validate in your function |
| `boolean` | Toggle (`falseLabel` sets the off-state label) |
| `markdown` | Markdown editor |
| `code`, `python`, `javascript`, `script` | Code editor |
| `simpleExpression` | Compact code editor |
| `label` | Document-label picker |
| `documentStatus` | Document-status picker |
| `taskStatus` | Task-status picker |
| `taskTemplates` | Task-template picker |
| `documentStore`, `tableStore`, `modelStore`, `taxonomy`, `dataDefinition` | Component picker for that resource type. The value stored is the resource **`ref`** (`orgSlug/slug`), so resolve it with a ref lookup, not an id lookup. |
| `assistant` | Assistant picker. Unlike the others, the value stored is the assistant **id**. |
| `cloudModel` | AI model picker |
| `cloudEmbedding` | Embedding model picker |
| `knowledgeSet`, `knowledgeFeature` | Knowledge pickers |
| `dataForm` | Data-form picker |
| `workspace` | Workspace picker |
| `documentLookup` | Document lookup |
| `attributeStatus` | Attribute-status picker |
| `taxon`, `groupTaxon`, `taxonLookup`, `taxonWithProperties`, `taxonomyTaxon`, `taxonomyTaxonSelection` | Taxon pickers of varying shape |
| `alert`, `article`, `chart` | Display-only, no value captured |
| `object` | Nested sub-form driven by `groupOptions` |

There is no `textarea` type — use `text` with `properties: { lines: N }`.

### The `properties` bag

These are the keys a renderer actually reads:

| Key | Effect |
|---|---|
| `lines` | Multi-line text input with this many rows |
| `pattern` / `patternMessage` | Client-side regex validation and its message |
| `hideLabel` / `hideDescription` | Suppress the label / description chrome |
| `markdown` | Inline help popover content |
| `valueField` / `textField` | Which keys of a select item supply value and label (default `value` / `label`) |
| `useTable` | Render a `listType` option's rows as a table of its `groupOptions` |

Studio's option builder also offers `placeholder`, `password`, `min`, `max` and `step` when you
edit an option definition, but no option renderer reads them — setting them does nothing. Masking
comes from `type: password`, not `properties: { password: true }`, and there is no numeric range
enforcement. For placeholder-style guidance use the option's own `hint` field, which several of the
resource pickers use as their placeholder; `description` renders as helper text under the field.

### Lists and nested objects

`listType` is the switch. Setting it turns the option into a repeating editor whose rows are of
that type; `type: list` on its own does nothing useful.

```yaml
inferenceOptions:
  # A list of sub-forms
  - name: label_actions
    listType: object
    label: "Label actions"
    listLabel: "Action"              # heading used per row
    listDescription: "One rule per row"
    groupOptions:
      - name: action
        type: string
        label: "Action"
        required: true
        possibleValues:
          - { value: add_label,    label: "Add label" }
          - { value: remove_label, label: "Remove label" }
      - name: label
        type: label
        label: "Label"
        required: true

  # A list of picker values
  - name: allowed_statuses
    listType: documentStatus
    label: "Allowed statuses"
```

### Other `Option` fields

`label`, `description`, `required`, `showIf`, `developerOnly`, `default`, `possibleValues`,
`groupOptions`, `listType`, `listLabel`, `listDescription`, `falseLabel`, `hint` (used as
placeholder text by several pickers), `properties`. The remaining declared fields — `tabName`,
`featureFlag`, `subType`, `aliases`, `displayProperties`, `overviewMarkdown`, `supportArticle`,
`showOnPopup` — are accepted and stored but no module-option renderer consumes them.

## Placeholders in option values

Option values authored on an activity-plan execution step (or an assistant step) go through
substitution before your module sees them.

| Placeholder | Resolved by | When |
|---|---|---|
| `${secrets.<name>}` | Orchestrator | Slice materialisation. Top-level string option values only. |
| `${secret.<name>}` | The module runtime itself | At execution, recursively through the whole value tree. This is what the `password` option's secret picker writes. |
| `${project.id}` | Orchestrator | Slice materialisation (project-scoped, so it cannot happen earlier). |
| `${project.documentStatusId.<slug>}` | Orchestrator | Slice materialisation — resolves the project's per-status id. |
| `${org}/<slug>` in any string, or a value that is exactly `${org}` | `kdx` | Apply/push time, against the target organization. `kdx sync pull` writes it back in, which is why pulled files are full of `${org}/…` refs. Only those two forms are substituted — a leftover `${org}` anywhere in the payload aborts the push with an error rather than shipping the literal. |
| `{org}` in `moduleRefs` / `moduleSidecars` | Agent runtime | Chat start; substituted with the running organization's slug. Note the brace style differs from `${org}`. |

An unresolved `${project.*}` placeholder is left literal with a warning, so the module fails with
the placeholder visible rather than silently receiving an empty string.

## Endpoints

| Endpoint | Purpose |
|---|---|
| `GET`/`POST` `/api/modules` | List / create |
| `GET`/`PUT`/`DELETE` `/api/modules/{id}` | Fetch / update / delete (flat wire format) |
| `GET`/`POST` `/api/modules/{id}/implementation` | Download / upload the implementation ZIP |
| `GET` `/api/modules/{id}/audit` | Audit trail |
| `GET` `/api/modules/{id}/sequence` | Current `changeSequence` for optimistic concurrency |
| `POST` `/api/resolve?path=module://acme-corp/invoice-extractor` | Slug → id path. Schemes `module`, `model` and `model-runtime` all resolve here. |
