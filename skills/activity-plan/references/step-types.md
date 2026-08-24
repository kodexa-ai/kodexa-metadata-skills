# Step types and plan-level fields — full surface

Everything here is authored as YAML and applied with `kdx apply -f` or `kdx sync push`. Read
`SKILL.md` first: the envelope is flat, keyed by `type`, and the load-bearing traps live there.

## Universal step fields

| Field | Notes |
|---|---|
| `slug` | Required. Unique within the plan (a duplicate fails the start). Format `^[a-z0-9][a-z0-9_-]*$`. Renaming it means rewriting every `dependsOn` entry that names it, preserving `?` and `:action` qualifiers. |
| `type` | Required. `EXECUTION`, `CREATE_TASK`, `SCRIPT`, `LLM`, `BRIDGE_CALL`, `AGENT`, `APPROVAL`. |
| `name`, `description` | Display only; independent of `slug`. |
| `uuid` | Legacy step identity. If present it also works as a `dependsOn` alias for the slug. Do not add it to new steps. |
| `dependsOn` | Array of strings. Grammar in `SKILL.md`. |
| `conditionExpr` | JSONata string (an object `{expr: "..."}` is also accepted and is the persisted form; an object without a non-empty `expr` is flagged `condition.malformed`). A false, non-boolean **or erroring** result marks the step `NOT_TAKEN` (an unmatched path is an error, so it too settles NOT_TAKEN rather than failing) — and NOT_TAKEN cascades to its plain-dep downstream. |
| `setDocumentStatus` | Project document-status slug, applied on completion. Works on every type. |
| `bypass` | `true` settles the step SKIPPED with no work. Works on every type. |
| `badges` | `[{icon?, label, color?, promote?}]`. Only `promote: true` badges are merged onto the activity when the step completes, deduped on `icon`+`label`+`color` (so `label` is what makes a badge distinct), and the `promote` flag is stripped on the activity copy. SCRIPT and BRIDGE_CALL steps can also emit badges at run time via their mapped output, and those are always promoted. |
| `perDocument`, `maxParallel`, `joinPolicy` | See `SKILL.md`. A join policy is persisted for EXECUTION, LLM, SCRIPT and BRIDGE_CALL (the `join-policy-inert` warning still names only EXECUTION and LLM — the warning text is stale, the field works). |

Plan caps — enforced only by the save-time validator, so only where a deployment enables it: 5 MiB of
`steps` JSON, 2000 steps, 500 input options, 10 000 characters per `conditionExpr`.

## EXECUTION

```yaml
- slug: extract
  type: EXECUTION
  moduleRef: "kodexa/fast-pdf-model"      # orgSlug/moduleSlug or module://orgSlug/moduleSlug
  options:
    inferenceMode: layout
    projectId: "${project.id}"            # top-level string values only
    statusId: "${project.documentStatusId.pending-review}"
    apiKey: "${secrets.VENDOR_API_KEY}"
  perDocument: true                       # default true for EXECUTION
  maxParallel: 5                          # EXECUTION only
  joinPolicy: ALL_SETTLED
```

`options` substitution happens at materialization and covers **top-level string values only** —
nested maps and arrays are left alone. `${project.id}` and `${project.documentStatusId.<slug>}` let an
org-level plan drive project-specific behaviour without baking in project UUIDs; an unresolvable
placeholder is left literal and logged. `${secrets.NAME}` resolves from the org's secret store.
`${orgSlug}` is not handled here — it was already expanded when the plan was saved.

**Nothing in the plan names the resources the module works on.** There is no plan-level taxonomy,
store or prompt key, no project-level default, and no inheritance from a resource merely bound to the
project — the module reads what it needs out of `options`, so `options` is the only wiring. Omit an
option the module wanted and the step still runs, still completes, and produces nothing.

Read the contract off the module rather than copying `options` from an example: each entry under the
module's own `options:` gives the `name` to write, a `type`, and often a `default`.

```bash
kdx run modules list-modules --filter "organization.slug:'acme-corp'" --pageSize 1000 -o yaml
```

| Option `type` | Value to write |
|---|---|
| `taxonomy`, `dataDefinition`, `documentStore`, `tableStore`, `modelStore` | the resource **`ref`** — `orgSlug/slug`. Not a UUID, not a `taxonomy://` URI |
| `assistant` | the assistant **id** — the one picker that stores an id |
| `password` | `"${secrets.NAME}"`, provisioned with `kdx secret set acme-corp NAME` |

Full option-type inventory: the **module** skill, `references/schema.md`.

An EXECUTION step emits no action, so nothing may route off it with `slug:action`.

## CREATE_TASK

```yaml
- slug: review
  type: CREATE_TASK
  taskTemplateRef: invoice-review         # bare slug, resolved org-scoped (no project binding needed)
  taskStatusSlug: to-do                   # bare slug from the org's task statuses; NO fallback
  taskData:
    title: "Review — ${activity.title}"
    description: "Project ${project.name}"
    priority: 2
    properties:
      costCentre: "${project.options.dataProperties.costCentre}"
```

`taskData.title`, `.description` and top-level string values in `.properties` understand exactly four
placeholders: `${activity.title}`, `${project.id}`, `${project.name}`,
`${project.options.dataProperties.<key>}`. Nothing else resolves — `{{ .inputs.x }}` stays literal in
the created task's title. To get an input into a task title, put it in `defaultTitleTemplate` and
reference `${activity.title}` from the step. An unresolvable `dataProperties` key becomes an empty
string; the other three are left literal with a warning.

Routing off a CREATE_TASK uses the **referenced TaskTemplate's own actions** plus the org's
**DONE-typed task-status slugs** (so `dependsOn: ["review:reviewed"]` works when `reviewed` is a DONE
status). The step's inline `actions:` array is folded into `taskData.actions` and is a documentation
mirror only. `setDocumentStatus` on a CREATE_TASK that also routes on its outcome fires on **any**
outcome (warned `create-task-set-status-outcome`) — for outcome-specific status, use SCRIPT steps
keyed on the action.

## SCRIPT

```yaml
- slug: route
  type: SCRIPT
  scriptBody: |
    var fam = families[0];
    var doc = documents.get(fam.id);
    log.info('status=' + doc.status);
    return {
      action: doc.status === 'reviewed' ? 'skip' : 'process',
      badges: [{ icon: 'flag', label: 'Rush', color: 'amber' }],
      taskProperties: { vendorId: inputs.vendorId }
    };
  scriptActions:
    - { name: Process, slug: process }
    - { name: Skip,    slug: skip }
  scriptSidecars: ["acme-corp/invoice-helpers"]   # orgSlug/moduleSlug, no version suffix
  perDocument: false
```

**Return contract.** The script must return an object with a non-empty `action` that matches a
declared `scriptActions` entry by `slug` (exact) or `name` (case-insensitive). `return {}`,
`return null` and returning an action when `scriptActions` is empty all fail the step. Optional keys
on the same object:

| Key | Effect |
|---|---|
| `features` | `[{documentFamilyId, featureId}]` — assigns existing knowledge features |
| `badges` | `[{icon, label, color}]` — always promoted onto the activity |
| `taskProperties` | merged onto the child task of a downstream CREATE_TASK, overriding static `taskData.properties` |
| `nextActivity` / `nextActivities` | spawn follow-up activities when this activity completes. Mutually exclusive; each entry is `{activityPlanRef, projectId?, inputs?, title?, description?, documentFamilyIds?, features?, priority?, triggerMetadata?}`. Capped at 32. Ignored on a `perDocument` script (logged as a warning). |

**`families` depends on `perDocument`.** A `perDocument: true` script runs once per document against a
**single-element** `families` array, so `families[0]` is *the* current document. A single-shot script
runs once with **every** family in the array, so `families[0]` is merely the first — iterate instead of
indexing, and note it still resolves only **one** action for the whole run.

**Globals** — `inputs` (the activity's inputs, always an object), `families` (the activity's document
families as `{id, name, metadata, mixins, featureIds, contentObjects}`), `task`, `org` (`{id, slug}`),
`documents`, `tasks`, `knowledge`, `projects`, `loadDocument`, `loadTaxonomy`,
`serviceBridge.{call,list}`, `llm.{invoke,invokeWithPromptRef}`, `lookupFeatureType`,
`lookupFeaturesByType`, `log.{info,warn,error,debug}`, `console.log`. There is **no** `ctx`, `context`
or `action` global.

`documents.get(familyId)` returns `{id, path, size, metadata, locked, pendingProcessing, storeSlug,
storeRef, summary, status, statusLabel, mixins, labels, features, featureIds, contentObjects,
createdOn, updatedOn}` — `status` is the project document-status slug or null.

**Limits** — 300 s wall clock by default (deployment-configurable), 5 document loads per script, 10
service-bridge calls per script, 10 s per bridge call, 10 MB response cap, and a 256 KB cap on any one
`documents.*` / `tasks.*` payload. Do not call `close()` on a loaded document — the runtime closes and
persists them itself when the step finishes.

**`documents.get()` is metadata; `loadDocument()` is the data.** The record above carries no
attributes and no data objects. Extracted data hangs off `loadDocument(familyId)`, and three things
about that surface cost the most time — none of them checked, none inferable from the YAML:

- **Rows of a repeating group are reached by path, not by descent.** `getChildren()` on the root data
  object returns `[]` however many rows exist; rows are stored flat. Use
  `findDataObjectsByPath('invoice/line_items')`, or `findFirstDataObjectByPath(...)` for a single
  object. `select(...)` is a **content-node** selector and returns `[]` for a taxon path.
- **A path here is the `/`-joined chain of taxon `name` values** — the same string a data form binds
  as `tagPath`, and *not* the `externalName` chain a KEXL formula resolves (**data-definition**,
  "Naming — four fields, four subsystems"). The two chains diverge the moment `externalName` is set.
- **Writes are type-strict, and the error names the parser rather than your value.**
  `setAttribute(name, value)` parses against the taxon's `taxonType` and fails the whole step on a
  mismatch. A `DATE` taxon wants RFC3339 — stricter than what extraction itself stores — and refuses
  a bare `YYYY-MM-DD`:
  `setAttribute(invoice_date): date attribute parse: parsing time "2026-05-01" as
  "2006-01-02T15:04:05Z07:00": cannot parse "" as "T"`.

**Debugging a script.** `log.*` and `console.log` reach no CLI-visible sink —
`kdx run activities logs --id <activity-id>` returns each step's skeleton with an empty `logs` array
and exits 0, which reads as "the step logged nothing". The failure itself is retrievable:
`kdx run activities get-activity --id <activity-id> -o json` puts the exception on
`steps[].errorDetails.error` with its class, message and `<eval>:line:col`, and a `perDocument` step
repeats it per document under `steps[].mappedOutput` — which tells you *which* document failed. Since
a script fails whole and only its recognised return keys are published, put a diagnostic in an
attribute or on the action object rather than in a log line.

## LLM

```yaml
- slug: classify
  type: LLM
  promptBody: |                           # supply this OR promptTemplateRef (body wins if both)
    Classify {documentText} for vendor {vendorName}.
    Answer JSON: {"decision": "receipt"|"invoice"}
  llmModelName: LARGE                     # SMALL (default) | LARGE
  perDocument: true
  includeDocument: { maxPages: 5, maxCharacters: 40000 }
  enrichment:
    - serviceBridgeRef: vendor-api
      operation: lookup-vendor            # the endpoint name
      inputMapping: { vendorId: "inputs.vendorId" }   # dot-paths, NOT JSONata
      outputKey: vendorRecord
  promptVariables: { vendorName: "enrichment.vendorRecord.name" }   # JSONata
  outputMapping: { action: "$.decision" }                           # JSONata over the reply
  promptActions:
    - { name: Receipt, slug: receipt }
    - { name: Invoice, slug: invoice }
```

- The prompt body is rendered with **FString** substitution (`{name}`), not Go templates or Mustache.
  Available names: `context`, `enrichment`, `steps`, plus everything `promptVariables` adds; the
  per-document path adds `documentFamilyId`, and adds `documentText`, `documentPath` and
  `extractedData` **only when `includeDocument` is set**. Substitution is a plain replace of each known
  `{name}`, so an unknown one is left literal and JSON braces in the prompt are safe.
- `llmModelName` is a model *type*. `SMALL` and `LARGE` resolve to the environment's configured
  models; any other value is passed to the gateway verbatim and fails if the gateway does not know it.
- `enrichment[].inputMapping` values are plain dot-paths over `{orgId, projectId, inputs}` —
  `"inputs.vendorId"`. A `$.`-prefixed value resolves to nil: sent as `null` on a POST endpoint, and
  the query parameter silently disappears on a GET. `operation` must name a real endpoint on the
  bridge — an unknown name fails the call with `has no endpoint`. Enrichment treats any non-2xx as a
  step failure.
- `includeDocument` is honoured **only when `perDocument: true`**. On a single-shot LLM step it logs a
  warning and injects nothing. Shape: `true`, or `{documentTypes?, includeExtractedData?, maxPages?,
  maxCharacters?}`. Exceeding `maxCharacters` fails that document.
- `outputMapping` JSONata is evaluated against the **parsed reply JSON**, or `{response: "<text>"}`
  when the reply is not JSON. `promptActions` only resolve if `outputMapping` produces a key literally
  named `action`; without it no action is recorded and every `slug:action` edge off this step is dead.

## BRIDGE_CALL

```yaml
- slug: post-invoice
  type: BRIDGE_CALL
  serviceBridgeRef: erp                   # bare slug (an orgSlug/ prefix is stripped)
  endpointName: submit-invoice            # must exist on the bridge — no fallback to the first
  requestBody:
    grade:    "steps.grade.action"        # every value is JSONata; only SCRIPT/LLM/BRIDGE_CALL
                                          # publish anything — steps.<execution-slug>.x is empty
    vendor:   "inputs.vendorId"
    source:   "'kodexa'"                  # a literal MUST be single-quoted
  requestQuery: { mode: "'auto'" }
  requestHeaders: { X-Trace: "documentFamilyId" }
  treatAsError: "$._statusCode >= 300"    # default: $._statusCode < 200 or $._statusCode >= 300
  timeoutSeconds: 30                      # default 30, capped at 120
```

- A step must supply **either** the four `request*` maps **or** `requestScript`, never both and never
  neither — both is `bridge.request-source-conflict` at save and a step failure at run time; neither
  fails the step.
- Every value in `requestBody`, `requestQuery`, `requestPath` and `requestHeaders` is compiled and
  evaluated as JSONata, so a bare word is a **path lookup**. `"hello"` fails the step with an eval
  error (`no results found`); write `"'hello'"`.
- **Any path that matches nothing fails the whole step the same way** — `"steps.review.action"` when
  `review` was NOT_TAKEN, or `"inputs.tier"` when that optional input was omitted. Guard every
  optional reference: `"$exists(inputs.tier) ? inputs.tier : 'standard'"`. Only an expression that
  *evaluates* to null is tolerated, and it drops the key from the query, path and header maps.
- `requestBody` keyed on the single key `"$"` replaces the whole body with that expression's result
  (use it to send a JSON array).
- `requestScript` is JavaScript wrapped as `(function(ctx){ … })(ctx)` and must return
  `{body?, query?, path?, headers?}`. This is the **only** place a `ctx` global exists; it is the
  same context the JSONata maps see.
- Header values containing CR, LF or NUL abort the call.
- `outputMapping` is not carried onto a plan-authored BRIDGE_CALL step, so `bridgeActions` never
  resolve — see the inert table in `SKILL.md`. The step's mapped output still always carries
  `_statusCode` and `_body`, so a downstream step's `conditionExpr` can branch on
  `steps.<slug>._statusCode`. A SCRIPT *body* cannot: there is no `steps` global in the script VM.

## AGENT

```yaml
- slug: agent-review
  type: AGENT
  agentRuntimeRef: "acme-corp/review-agent"   # resolved at start; the runtime must be READY
  prompt: "Review the extracted invoice and flag anomalies."
  moduleRefs: ["acme-corp/invoice-helpers"]   # handed to the runtime in the agent metadata
```

`agentRuntimeId` is the resolved form and may appear in plans exported from a running system. An
unresolvable or non-READY `agentRuntimeRef` rejects the start.

## APPROVAL

Accepts `approverRole` and `approvalCriteria` and persists them, then the orchestrator settles the
step `SKIPPED`. Do not build a gate on it.

## What each step publishes into `steps.<slug>`

Downstream JSONata (`conditionExpr`, BRIDGE_CALL request maps, LLM `promptVariables`) reads prior
COMPLETED steps as `steps.<slug>.<key>` — a SCRIPT body cannot, there is no `steps` global in the VM.
Only COMPLETED steps appear, and only these keys exist:

| Step type | Keys under `steps.<slug>` |
|---|---|
| `SCRIPT` | the returned object's recognised keys only — `action`, `features`, `badges`, `taskProperties`, `nextActivity`/`nextActivities`. **Any other key you return is dropped**, so `return {total: 42}` is not readable downstream |
| `LLM` | the `outputMapping` results, exactly |
| `BRIDGE_CALL` | `_statusCode` and `_body`, always |
| `EXECUTION`, `CREATE_TASK`, `AGENT`, `APPROVAL` | nothing |

Every step that completed with an action also exposes `completedActionUuid`. Under per-document
routing a perDocument step's entry is narrowed to the current document, exposing that document's
`action` and `response` rather than the aggregate.

## Plan-level fields

| Field | Behaviour |
|---|---|
| `inputOptions` | Array of option objects. `name` is required and must be unique. `type` must be one of `string`, `text`, `number`, `integer`, `boolean`, `date`, `datetime`, `selection`, `object`, `array`, `cloudModel`, `cloudEmbedding`. Only required-presence is enforced at start; types are not checked. |
| `inputsSchema` | JSON Schema draft 7. Renders the Studio start form. Never validated server-side. |
| `defaultTitleTemplate`, `defaultDescriptionTemplate` | Go text/template. Context: `inputs`, `org{id,slug,name}`, `project{id,slug,name}`, `caller{userId,email}`, `now` (RFC3339), `documentPath`, `metadata` (first family's metadata), `documentCount`, `documentNames`, `documentPaths`, `documentMetadata`, `features`, `featureNames`. Helpers: `filename`, `join`, `default`. Missing keys render empty; a template that fails to *render* rejects the start with 400. |
| `documentFamilyGroups` | Upload zones for the New Activity wizard. Per group: `name`, `notes`, `required`, `maxHits`, `maxPages`, `hardMaxPages`, `maxSize`, `uniqueFilenames`, `automaticallyAdd`, `documentFamilyFilter`, `editable`, `uploadOnly`, `sort`, `titlePrompt`, `knowledgeFeatures: [{featureTypeRef, required}]`. **A `required: true` group with zero resolved document families rejects the start with 400** (`this activity requires at least one document in "<group>" before it can start`) — including for API, trigger, intake and spawned starts that never see the wizard. |
| `knowledgeAgent` | Boolean, copied onto each Activity at start so knowledge runs can be listed with one filter. |
| `metadata.serializePerProject` | `true` allows at most one activity of this plan per project at a time; a concurrent start folds its event into the in-flight run and returns that activity. |
| `metadata.aiNaming` | `{enabled, prompt}`. When enabled and the caller supplied no title, a small model renames the activity after commit. Placeholders are single-brace: `{templateName}` / `{activityPlanName}`, `{documentFamilyPaths}`, `{documentFamilyCount}`, `{knowledgeFeatures}`, `{documentMetadata}`, `{metadata:some.path}`, `{externalData:key.path}`. |
| `documentValidationPrompt` | Enforced in the browser upload flow only. API, trigger and intake starts bypass it, and an unreachable model fails open. |
| `documentSummarizationPrompt` | Upload-time summary written to the family's `metadata.summary`. Decoration only. |
| `slug`, `name`, `description` | `slug` is the identity (unique in the org, and what `activity-plan://` resolves); `name` and `description` are display only. |
| `type`, `orgSlug` | File-routing identity, not runtime behaviour — but `kdx apply -f` **refuses a file without both** (`--type` / `--org-slug` override them). `type` accepts `activity-plan`, `activityPlan`, `activity-plans` or `activityPlans`. Nothing in the start or run path reads either. |
| `template`, `deprecated`, `publicAccess`, `extensionPackRef` | Standard metadata flags shared with other org resources. |

## Sync layout

```
acme-corp/
  activity-plans/
    invoice-intake.yaml
  task-templates/
    invoice-review.yaml
```

`activity-plans` is pushed before project templates so `${activityPlan.<slug>}` references resolve.
`kdx` accepts `activity-plan`, `activityPlan`, `activity-plans` and `activityPlans` as type aliases.
