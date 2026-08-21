---
name: module
description: "Use when authoring or debugging a Kodexa module.yml / model.yml — Python, Go-WASM or inline-JavaScript model modules and agent skill packs. Covers moduleType, moduleRuntimeRef, moduleSidecars, inferenceOptions, actions, contents/ignoredContents packaging, the parameters the runtime injects, and applying with kdx."
---

# Kodexa Module Authoring

A module is a unit of executable work owned by an organization. `moduleType: model` modules run
against documents — a Python package, a Go-WASM plugin, or an inline JavaScript snippet;
`moduleType: skill` modules are file packs an agent loads as a Claude skill. Both live in
`kdxa_modules` and are created and updated with `kdx apply -f module.yml`.

## The envelope that works

```yaml
type: module                  # resource type — always "module"
slug: invoice-extractor
orgSlug: acme-corp
name: "Invoice Extractor"
description: "Extracts header and line-item data from invoices"
moduleType: model             # model | skill — THE discriminator, top level
eventAware: false             # top level — real column
supportsScheduling: false     # top level — real column
deprecated: false
publicAccess: false

metadata:
  moduleRuntimeRef: kodexa/base-model-runtime
  moduleRuntimeParameters:
    module: invoice_extractor  # Python package directory inside the ZIP
    function: infer            # entry point (default: infer)
  inferable: true              # metadata-only (no column)
  contents:
    - invoice_extractor/**.py
  ignoredContents:
    - "**/__pycache__/**"
    - "**/*.pyc"
  inferenceOptions:
    - name: confidence_threshold
      type: number
      default: 0.85
      label: "Confidence threshold"
      description: "Minimum confidence for an extracted value"
```

Modules are **unversioned**. `ref` (`acme-corp/invoice-extractor`) and `uri`
(`module://acme-corp/invoice-extractor`) are computed server-side — never author them, and never put
a `:1.0.0` suffix on a `moduleRuntimeRef` or `moduleSidecars` entry (loaders parse it off and throw
it away). `GET /api/modules/{id}` serves the record **flat**: `moduleRuntimeRef`, `contents`,
`inferenceOptions` and friends come back at the top level with no `metadata` object. Writes accept
either shape, so your YAML is not "losing" its metadata block when a GET looks different.

## Six things that fail silently

**1. The model/skill switch is top-level `moduleType`, not `metadata.type`.**
The API decodes the whole request body into the metadata struct after the nested block, so
`metadata.type` is overwritten with the resource type (`"module"`) on every single write — it can
never hold `model` or `skill`, and nothing reads it. The `module_type` column defaults to `model`,
so a Python module survives the omission. **A skill pack does not**: without top-level
`moduleType: skill` it stays `model` and gets offered as a runnable execution target.

**2. `eventAware` and `supportsScheduling` must be top level.** They are real columns *and*
metadata keys, and the column always wins on read. Nested under `metadata:` your value is lost and
the module never appears in the scheduling picker (which filters the column). `inferable` is the
opposite — metadata-only, filtered as `metadata.inferable`.

```yaml
eventAware: true            # ✅ column, honoured
metadata:
  eventAware: true          # ❌ shadowed by the column, value lost
  inferable: true           # ✅ correct place for this one
```

**3. `ignoredContents`, not `ignored_contents` — and only under `metadata:`.** The wire format is
camelCase everywhere except `entry_point` inside `actions[]`. `kdx apply` reads `ignoredContents`
from `metadata:` alone, so a snake_case key *or* the same key at the top level means no exclusions
apply and `__pycache__/`, `.pyc` and `tests/` all ship in the ZIP. (`contents` is lenient — top level or
`metadata:` both work.) The pre-rename spellings `modelSidecars` and `modelRuntimeParameters` are
still *read* by the runtimes for old rows, but they are no longer fields of the module model:
author them today and the API drops them. Write `moduleSidecars` / `moduleRuntimeParameters`.

**4. Only listed parameters are injected, and un-injected ones with no default crash the call.**
The runtime hands your function exactly: the option values configured on the step (keyed by your
option `name`s), plus `document`, `pipeline_context`, `model_base`, `execution_id`, `module_ref`,
`project`, `assistant`, `assistant_id`, `model_data` and `training_id` (both always `None`), and
`channel_id` / `message_id` / `task_id` / `event_type` when the dispatch carries them. Nothing else
— notably **not** `document_family` (reach it via `pipeline_context.document_family`),
`model_store`, `status_reporter`, or `event`. The runtime passes only the keys it has, so a
parameter it does not supply falls back to your signature default — and a *required positional* it
does not supply raises `TypeError`. Give every injected parameter a default. (`document` is always
passed; on an event dispatch with no content object its value is `None`, which overrides your
default — guard for it.)

**5. `default:` in an option is a form-seeding hint, not the runtime default.** Studio seeds it
into the step's option map only when the value is truthy, so `default: false`, `default: 0` and
`default: ""` never reach the module at all. The value that actually applies at execution time is
the one in your function signature. Mirror them.

**6. `listType` — not `type: list` — switches on the list editor.** The renderer computes the
effective type as `listType ?? type ?? "string"`. `type: list` alone matches no component and
renders an "unknown option type" alert.

## Implementation kinds

| Kind | Declares | Entry point |
|---|---|---|
| Python package (default) | `moduleRuntimeParameters.module` / `.function` | `infer` (or the named function), falling back to `handle_event` |
| Go-WASM plugin | `bridgeType: wasm`, `allowedHosts`, `build:` | the WASM export named `infer` |
| Inline JavaScript | `moduleRuntimeRef: kodexa/go-scripting-runtime`, `scriptLanguage: javascript`, `script:` | the script body itself |
| Agent skill pack | `moduleType: skill` at top level | none — the agent reads `SKILL.md` |

```yaml
# Go-WASM: kdx runs the build step, then packages the artifact
metadata:
  bridgeType: wasm                # "python" (default) | "wasm"
  moduleRuntimeRef: kodexa/base-model-runtime
  build:
    - lang: go-wasm               # lang / workdir / output all required
      workdir: .
      output: plugin.wasm
  contents:
    - plugin.wasm                 # must land at the ZIP ROOT
  allowedHosts:                   # wasm only; wildcards allowed
    - "*.kodexa.example.com"
```

`build` runs on your machine at apply time (it shells out to your local `go`); it is not stored on
the module record. The WASM bridge takes the first `.wasm` file **at the root** of the ZIP, and ZIP
entry paths are relative to the manifest's own directory — so keep the manifest beside the plugin.

Two WASM-only traps: export **`infer`** (the orchestrator stamps that entry-point name on every
execution step, and `moduleRuntimeParameters.function` is a Python-bridge setting that never
reaches the WASM bridge); and read your options out of the Extism config entry **`module_options`**
(a JSON object) — the plugin SDK's `GetOption*` helpers look for a differently-named key and
silently hand back the default you passed them.

See `references/runtimes.md` for the runtime catalogue, the WASM plugin contract, inline-JS
globals, skill-pack layout, and action-native modules.

## Packaging and applying

```bash
kdx apply -f module.yml            # upserts metadata, runs build steps, zips contents, uploads
kdx get modules --output json
kdx get module invoice-extractor --download --download-path ./impl.zip
kdx delete module invoice-extractor
```

A glob that matches nothing is a hard error ("no files found matching the patterns"). Use `**` for
recursion (`invoice_extractor/**.py`). There is no separate deploy step.

## Declared but inert

Persisted and round-tripped; except where noted, nothing in the platform reads them:

| Key | Note |
|---|---|
| `metadata.type` | Overwritten with `"module"` on every write; no reader. Use top-level `moduleType`. |
| `moduleStatus` (`DRAFT`/`PUBLISHED`/`DEPRECATED`) | Stored, never gated on. `deprecated: true` is what actually hides a module from pickers. |
| `metadata.state` | Free string; first-party modules write `TRAINED` out of habit. No reader. |
| `metadata.stateHash` | Read, but server-managed: stamped on every implementation upload. Never author it. |
| `metadata.baseDir` | No reader on the module record; `kdx apply` globs relative to the manifest's own directory. |
| `metadata.optionTabs`, `messageTemplates`, `taxonomy`, `additionalTaxonOptions`, `taxonFeatures` | Round-trip slots that keep legacy rows intact. Nothing consumes them. |
| Option `tabName` | Stored, but the option renderer draws one flat list — no tab grouping exists. |
| Option `featureFlag`, `subType`, `aliases`, `displayProperties`, `overviewMarkdown`, `supportArticle` | Accepted by the API; no renderer consumes them. |

Not fields at all — the API drops them on write: `version`, `lifecycle`, `sourceUrl`, `provider`,
`providerUrl`, `providerImageUrl`, `overviewMarkdown`, `trainable`, `deploymentDefaults`. Put
human-facing prose in `description`, which is a real column.

## Common mistakes

| Mistake | What happens |
|---|---|
| `metadata: { type: skill }` for a skill pack | `moduleType` stays `model`; the pack is offered as a runnable module. Set `moduleType: skill` at the top level. |
| `eventAware` / `supportsScheduling` under `metadata:` | Column shadows the nested value; the module never shows in the scheduling picker. |
| `ignored_contents:`, or `ignoredContents:` at the top level | Silently ignored — `__pycache__` and tests ship in the ZIP. It must be `metadata.ignoredContents`. |
| `version:` or `lifecycle:` on a module | Dropped. Modules are unversioned; use `moduleStatus` / `deprecated`. |
| `deploymentDefaults:` under a module | Dropped. CPU/memory/timeout belong to the model runtime, not the module. |
| Sidecar or runtime ref with `:1.0.0` | Suffix is parsed off and discarded; it just contradicts every shipped manifest. |
| `def infer(document, status_reporter):` | `TypeError` — never injected. Declare `status_reporter=None` and guard. |
| `type: list` with no `listType` | Renders an "unknown option type" alert. Set `listType`. |
| Skill pack ZIP with no root `SKILL.md` | The agent SDK walks one level deep and never discovers the skill. |
| Go-WASM artifact nested in a subdirectory of the ZIP | Bridge fails with "no .wasm file found at root of ZIP archive". |
| Go-WASM plugin exporting only `handle_event` or `process_document` | The bridge calls the export named `infer` and the call fails. Export `infer`. |
| Option `name` not matching the function parameter | The value is passed under a key your signature does not declare, so it is dropped. |

## References

- `references/schema.md` — top-level fields and metadata keys, the option-type inventory, the
  option `properties` bag, placeholder substitution, endpoints.
- `references/runtimes.md` — runtime catalogue, injected parameters, WASM contract, inline-JS
  globals, skill packs, action-native modules.
- `references/examples.md` — worked manifests for all four kinds.

Related skills: `kdx-cli` (apply, sync, secrets), `activity-plan` (an `EXECUTION` step references a
module by `moduleRef` and supplies its option values), `assistant` (pipeline steps and
`moduleRefs`), `prompt-template`.
