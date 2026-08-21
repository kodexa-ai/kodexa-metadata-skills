# Runtimes, bridges and module kinds

## Runtime catalogue

Every shipped runtime lives under the `kodexa` organization and is referenced without a version.

| `moduleRuntimeRef` | Purpose |
|---|---|
| `kodexa/base-model-runtime` | Default Python runtime. The dispatcher falls back to this when `moduleRuntimeRef` is empty. |
| `kodexa/base-cloud-model-runtime` | Python plus cloud/LLM API dependencies. |
| `kodexa/base-agent-model-runtime` | Agentic execution — Claude Code plus the Python/TypeScript SDKs. |
| `kodexa/azure-model-runtime` | Azure SDK dependencies. |
| `kodexa/google-model-runtime` | Google SDK dependencies. |
| `kodexa/textract-model-runtime` | AWS Textract dependencies. |
| `kodexa/excel-model-runtime` | Excel manipulation. |
| `kodexa/uno-model-runtime` | Office-document conversion. |
| `kodexa/go-scripting-runtime` | Inline JavaScript (`scriptLanguage` / `script`). |
| `kodexa/claude-code-agent`, `kodexa/claude-code-agent-java` | Modules runnable inside the companion agent container. |
| `kodexa/activity-step-agent`, `kodexa/knowledge-loop-agent` | Headless activity-step and knowledge-loop agents. |

Omitting `moduleRuntimeRef` is not an error — the dispatcher logs the fallback and uses
`kodexa/base-model-runtime`. Declare it explicitly unless that is exactly what you want.

Runtimes are packaged as `type: extensionPack` manifests whose `services[]` entries carry
`type: modelRuntime` and `deploymentType: LOCAL`, so opening one of these files will not show a
top-level `modelRuntime` resource.

## Where compute settings live

A module never declares CPU, memory, replicas or containers. Those belong to the **model runtime**
row it references: `executionType` (`LAMBDA` or `K8S_JOB`), `memoryMb`, `cpuMillicores`,
`timeoutSeconds`, `ephemeralStorageMb`, `environmentVars`, `dockerImageUri` — with column defaults
of `LAMBDA`, 1024 MB, 2000 millicores, 300 s and 512 MB when the runtime does not set them. Pick a
runtime with `moduleRuntimeRef`; do not try to size it from the module.
`deploymentDefaults` under a module's `metadata:` writes nothing at all.

## Python modules

### Entry point resolution

1. The implementation ZIP is downloaded and extracted.
2. `moduleRuntimeParameters.module` picks the package directory (`<dir>/__init__.py`). Without it
   the runtime walks the extracted directory and takes the first `__init__.py` it finds — a coin
   flip as soon as the ZIP holds more than one package, so always set it.
3. `moduleRuntimeParameters.function` names the function to call; without it the runtime calls
   `infer`. If the package has no attribute under that name, it falls back to `handle_event`.
4. The working directory is changed to the extracted module directory for the duration of the
   call, so relative paths to packaged resources resolve.

### Injected parameters

Values are matched **by parameter name**, and only against ordinary positional-or-keyword
parameters — anything you put after a bare `*` (keyword-only) is never filled in. The runtime
passes only the names it actually has, so anything it does not supply falls back to your signature
default, and a required positional it does not supply raises `TypeError`. Give every injected
parameter a default.

| Parameter | Value |
|---|---|
| *(each configured option name)* | The option value set on the step, after placeholder substitution. This is the step's option map, not the module's `inferenceOptions` block — the orchestrator never reads `inferenceOptions`. |
| `document` | The `Document` being processed. Always passed; `None` when the dispatch carried no content object (event-driven runs), which overrides any default in your signature. |
| `pipeline_context` | A `PipelineContext` — see below |
| `model_base` | Absolute path to the extracted module directory |
| `execution_id` | The current execution id |
| `module_ref` | `orgSlug/slug` of the module being executed |
| `project` | Project endpoint for the execution's project; always passed, `None` when there is no project context |
| `assistant` | Assistant endpoint; always passed, `None` when the dispatch has no assistant |
| `assistant_id` | The assistant id, or `None` |
| `model_data` | Always `None` (legacy slot) |
| `training_id` | Always `None` (legacy slot) |
| `channel_id`, `message_id`, `task_id`, `event_type` | Only when the dispatch context carries them |

**Not injected**, however often you see them in older code or docs: `document_family` (use
`pipeline_context.document_family`), `model_store`, `status_reporter`, `event`.

`status_reporter` in particular: first-party modules declare `status_reporter=None` and guard every
call, and that guard is always false today because the runtime never passes a reporter. A module
that declares it *without* a default fails immediately. Go-WASM modules have a real status API
(`sdk.ReportStatus`).

### `pipeline_context`

The only route to the document family. Attributes: `.document_family` (a document-family endpoint
with a client attached), `.content_object`, `.document_store` (resolved from the family's store
ref), and `.context` (the raw dispatch context map).

### Return value

If the function returns a `Document`, or mutates the one it was given, the runtime writes a **new
derived content object** on the family so lineage is preserved and plan-level reprocessing works.
The step's output event then carries the new content-object id forward to the next step.

Any other return value is **discarded** on this path — the orchestrator bridge reports only the
output event. (A return value does become the result payload for an action invoked inside the
companion agent; see "Action-native modules" below.) To hand data to the rest of a plan, write it
onto the document or into a store.

### Sidecars

```yaml
metadata:
  moduleSidecars:
    - kodexa/kodexa-llm-model      # LLM helpers — package kodexa_llm
    - kodexa/fast-pdf-model
    - kodexa/text-parser
```

Each sidecar's implementation is downloaded and prepended to `sys.path` before your package is
imported, so `from kodexa_llm.utils import get_bedrock_client` works. Refs are bare `orgSlug/slug`;
a `:version` suffix is parsed off and discarded. Sidecar packages are resolved one level deep on
this path — a sidecar's own `moduleSidecars` are not fetched. The pre-rename key `modelSidecars` is
still read for rows written before the rename, but it is no longer part of the module model, so
authoring it now loses the list.

## Go-WASM modules

```yaml
type: module
moduleType: model
slug: archive-publisher
orgSlug: acme-corp
name: "Archive Publisher"
metadata:
  bridgeType: wasm
  moduleRuntimeRef: kodexa/base-model-runtime
  build:
    - lang: go-wasm          # the only builder today
      workdir: .             # relative to the manifest's directory; required
      output: plugin.wasm    # relative to workdir; required
      # optional: args: []   env: ["KEY=VALUE"]
  contents:
    - plugin.wasm
  allowedHosts:
    - "*.kodexa.example.com"
```

`kdx apply` runs the build steps *before* globbing `contents`, shelling out to your locally
installed `go` with `GOOS=wasip1 GOARCH=wasm GOWORK=off go build -buildmode=c-shared -o <output> .`
— no toolchain is embedded, and `go` must be on `PATH`. The `build` block is a client-side
instruction; it is not stored on the module record.

The bridge loads the **first `.wasm` file at the root of the ZIP**. ZIP entry names are the file
paths relative to the manifest's directory, so a manifest one level above its plugin produces a
nested entry that the bridge will not find.

Plugin contract:

```go
package main

import (
    "encoding/json"

    sdk "github.com/kodexa/kodexa-plugin-sdk"
)

//go:wasmexport infer
func infer() int32 {
    in := sdk.Input() // raw KDDB bytes

    // The step's option values arrive as ONE Extism config entry holding a
    // JSON object. Unmarshal it yourself — see the note below.
    opts := map[string]any{}
    if raw, ok := sdk.GetConfig("module_options"); ok && raw != "" {
        _ = json.Unmarshal([]byte(raw), &opts)
    }

    sdk.ReportStatus("processing")
    sdk.Log(sdk.LogInfo, "started")
    sdk.Output(in) // modified KDDB bytes
    return 0
}

func main() {}
```

**Export `infer`.** The orchestrator stamps `entry_point: "infer"` on every execution step it
materialises, and the WASM bridge calls that export name directly. `moduleRuntimeParameters.function`
is a Python-bridge setting and never reaches the WASM bridge, so there is no way to point a step at
a differently-named export. (The bridge does fall back to `process_document` when a step names no
entry point, but the current step builder always names one.)

**Read options from `module_options`.** The bridge flattens the step config into the Extism plugin
config, which gives you the entries `module_options` (a JSON object of the step's option values),
`module_ref`, `assistant_id` and `runtime_parameters`. The SDK's `GetOptionString` / `GetOptionInt`
/ `GetOptionFloat` / `GetOptionBool` helpers look up a *differently named* config key that the
orchestrator does not set, so they return the default you passed and nothing else — use
`sdk.GetConfig("module_options")` and unmarshal.

`sdk.Log(level, msg)` (levels `sdk.LogDebug` / `LogInfo` / `LogWarn` / `LogError`) writes to the
platform log; `sdk.ReportStatus(msg)` emits a progress message. Outbound HTTP is only permitted to
hosts matching `allowedHosts` — an empty list means none.

The plugin SDK is distributed with the platform source rather than through a public module proxy,
so `go get github.com/kodexa/kodexa-plugin-sdk` will not resolve. Your plugin's `go.mod` needs the
`require` plus a `replace github.com/kodexa/kodexa-plugin-sdk => <path to your local copy>`.

Output bytes, when present and the dispatch has a document family, are uploaded as a new derived
content object, exactly as in the Python path.

## Inline-JavaScript modules

The whole implementation lives in the manifest — no ZIP, no `contents`.

```yaml
type: module
moduleType: model
slug: threshold-router
orgSlug: acme-corp
name: "Threshold Router"
metadata:
  moduleRuntimeRef: kodexa/go-scripting-runtime
  scriptLanguage: javascript          # "javascript" or "js"
  script: |
    var threshold = parameters.confidence_threshold || 0.8;
    log.info("execution " + parameters.execution_id);
    return { status: "completed", threshold: threshold };
  moduleSidecars:
    - acme-corp/js-helpers            # other inline-JS modules, run in order in the
                                      # global scope before your script; max 10
```

The body is wrapped in a self-invoking function, so a top-level `return` is valid; the returned
object becomes the step's output event (a non-object return is discarded). Globals: `parameters`
(the step's option values plus `execution_id` and `pipeline_context`, the latter being the raw
dispatch context map, not the Python `PipelineContext` object), `log.debug/info/warn/error`, and
`console.log/warn/error`. An empty `script` is a run-time error, as is a non-empty `scriptLanguage`
that is anything other than `javascript`/`js` (case-insensitive).

The script must be inline in `metadata.script`. `kdx apply` sends the manifest as authored, and
`scriptPath` is not a field of the module model — a manifest that points at a `.js` file instead of
carrying the source lands with an empty `script` and fails at run time.

## Agent skill packs

```yaml
type: module
moduleType: skill                 # top level — this is what stops the pack being run as a model
slug: invoice-review-skills
orgSlug: acme-corp
name: "Invoice Review Skills"
description: "Prompts and helper tools for invoice review"
metadata:
  contents:
    - SKILL.md                    # REQUIRED at the ZIP root
    - SYSTEM_PROMPT.md            # optional prompt addendum
    - prompts/**
    - tools/**
  ignoredContents:
    - "**/*.pyc"
```

- The pack is extracted to `<workspace>/.claude/skills/<slug>/`, falling back to
  `/home/kodexa/skills/<slug>/`. **Slug only — no organization folder.**
- The agent SDK walks exactly one level deep, so `SKILL.md` must sit at the ZIP root. An extra
  wrapping directory hides it and the skill is never discovered.
- `SYSTEM_PROMPT.md`, if present, is appended to the agent's system prompt.
- Two packs with the same slug from different organizations collide in the cache; the second is
  skipped with a warning.
- A pack is loaded because something lists it in `moduleRefs` (a channel type or workspace
  context). A ref may use the literal `{org}` placeholder — `{org}/invoice-review-skills` — which
  is substituted with the running organization's slug.
- Declared `moduleSidecars` are fetched recursively into the same cache for code imports; they are
  not surfaced as separate skill directories.

## Action-native modules

A module can expose named actions an agent discovers and invokes.

```yaml
metadata:
  moduleRuntimeRef: kodexa/claude-code-agent-java
  actions:
    - name: validate-invoice-data        # the action name callers use (hyphen form)
      entry_point: validate_invoice_data # Python function in the package
      label: "Validate invoice data"
      description: "Check that the documents in a task are ready for export"
      inputs:
        - { name: document_store_ref, type: string, required: true }
    - name: list-templates
      entry_point: list_templates
```

`entry_point` is snake_case — the one snake_case key in a module manifest, because the agent reads
`actions[]` as opaque author-owned JSON rather than through the module model. Write `entryPoint`
and it is ignored: the agent falls back to the action's `name` **verbatim**, hyphens included, and
`getattr` on a hyphenated name never resolves.

Without an `actions[]` list the action name is treated as the entry point directly, with hyphens
mapped to underscores.

Only these runtime refs can run inside the companion agent container: the empty string (module
declares none), `kodexa/claude-code-agent`, `kodexa/claude-code-agent-java`,
`kodexa/kodexa-llm-model`, `kodexa/excel-model-runtime`. Anything else is refused with a message
pointing you at the standard execution flow.

The in-container bridge is deliberately thinner than the orchestrator one: it injects only the
option values plus `model_base`, `execution_id`, `module_ref`, and the event-id kwargs. There is no
`document`, no `pipeline_context`, no `project`/`assistant`. Actions fetch what they need through
the platform client themselves, and return a JSON-serialisable value as the action's result.
