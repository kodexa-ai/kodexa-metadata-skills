---
name: activity-plan
description: "Use when writing or editing Kodexa ActivityPlan YAML — the org-scoped graph of steps (EXECUTION, CREATE_TASK, SCRIPT, LLM, BRIDGE_CALL, AGENT) that runs as a project Activity. Covers the flat step envelope keyed by `type`, dependsOn and action edges, per-document fan-out and routing (perDocument, ANY_BRANCH, await `?` deps), setDocumentStatus, inputOptions vs inputsSchema, per-field reference formats, and the several template languages that coexist in one plan."
---

# Kodexa ActivityPlan authoring

An **ActivityPlan** is an org-scoped directed graph of typed steps; starting one creates a project-scoped **Activity**
plus a runtime step row per step. Plans live in `kdxa_activity_plans`, resolve as `activity-plan://acme-corp/invoice-intake`
(unversioned), sync from an `activity-plans/` directory, and must be **bound to a project** before they run.

## Resource shape

```yaml
slug: invoice-intake                  # required, unique in the org
orgSlug: acme-corp                    # `kdx apply -f` needs it (or --org-slug); ${org} also resolves
name: Invoice Intake
type: activity-plan                   # routes the file — `kdx apply -f` refuses one without it
inputOptions:                         # the ONLY server-enforced input contract
  - { name: vendorId, label: Vendor, type: string, required: true }
  - { name: priority, label: Priority, type: selection, required: false }
inputsSchema: { type: object, required: [vendorId], properties: { vendorId: { type: string } } }
defaultTitleTemplate: "Invoice — {{ filename .documentPath }}"   # Go text/template; the start
defaultDescriptionTemplate: "{{ .documentCount }} document(s)"   # returns 400 if it fails to render
steps: [...]
```

A required `inputOptions` entry missing (or empty) in `inputs` fails the start: `missing required inputs: <names>`.
`inputsSchema` only renders the Studio start form — never validated server-side — so declare every input in both.

## The step envelope is FLAT and keyed by `type`

```yaml
steps:
  - slug: extract                     # required; unique in plan; ^[a-z0-9][a-z0-9_-]*$
    type: EXECUTION                   # required — the discriminator is `type`, never `kind`
    moduleRef: "kodexa/fast-pdf-model"          # kind fields sit at the TOP LEVEL
    conditionExpr: "inputs.priority = 'high'"   # JSONata; false ⇒ step NOT_TAKEN
    setDocumentStatus: extracted      # any type; stamps a project document status on completion
    bypass: false                     # any type; true ⇒ step settles SKIPPED with no work
```

- `kind:` is **not** an alias — a step without `type` fails the start: `step missing required slug or type`.
- There is **no `config: {}` wrapper**: it is silently flattened into the step on save (so a later pull
  looks like a rewrite), and the save-time validator runs *before* that flattening, over the flat keys —
  so wherever that validator is enabled it reports your wrapped fields as missing.
- `bypass: true` settles the step **SKIPPED**, never COMPLETED — and SKIPPED is pass-through for **every**
  dep form, action-qualified ones included, so bypassing a router releases its branches, never strands them.

## Step types

| `type` | Runs | Key fields |
|---|---|---|
| `EXECUTION` | a module over the documents | `moduleRef`, `options`, `perDocument` (**default true**), `maxParallel` (5), `joinPolicy` |
| `CREATE_TASK` | a Task from an org TaskTemplate; waits for it | `taskTemplateRef`, `taskStatusSlug`, `taskData` |
| `SCRIPT` | JavaScript in a sandboxed VM (300 s budget) | `scriptBody`, `scriptActions`, `scriptSidecars`, `perDocument` |
| `LLM` | a prompt via the AI gateway | `promptBody` **or** `promptTemplateRef`, `promptActions`, `outputMapping`, `perDocument` |
| `BRIDGE_CALL` | an HTTP call to a ServiceBridge endpoint | `serviceBridgeRef`, `endpointName`, `request*` maps XOR `requestScript` |
| `AGENT` | dispatches an agent runtime | `agentRuntimeRef` (`orgSlug/runtimeSlug`, must be READY), `prompt`, `moduleRefs` |
| `APPROVAL` | **nothing — see below** | — |

Only these seven are safe. `TASK`, `BRIDGE` and `AI_PLANNER` sit outside the accepted type set and
materialize with none of their type-specific fields (`AI_PLANNER` settles SKIPPED); `AI_PROMPT` is
accepted but loses its prompt fields at start; **APPROVAL is not implemented** — the orchestrator settles
it `SKIPPED` at once, so it never reaches an approver and can never emit an action (use `CREATE_TASK`
with a review template); and **AGENT** ignores `assistantRef`/`agentInputs`, then fails the step with
`planned agent <id> has no agent_runtime_id`.

## Reference formats differ per field

| Field | Form | If you get it wrong |
|---|---|---|
| `moduleRef` (EXECUTION) | `orgSlug/moduleSlug`, or `module://orgSlug/moduleSlug` | module not found |
| `scriptSidecars[]` | `orgSlug/moduleSlug` **required**, no `:version` | `must be 'orgSlug/moduleSlug'` |
| `agentRuntimeRef` | `orgSlug/runtimeSlug` **required** | `expected 'orgSlug/runtimeSlug'` |
| `serviceBridgeRef`, `enrichment[].serviceBridgeRef` | bare slug; an `orgSlug/` prefix is stripped | — |
| `taskTemplateRef`, `taskStatusSlug`, `setDocumentStatus` | bare slug, resolved in the plan's org | silently unresolved — an unknown `taskStatusSlug` leaves the task with **no status at all** (there is no fallback) |
| `promptTemplateRef` | **bare slug — an `orgSlug/` prefix never resolves** | `prompt "acme-corp/x" not found` |

`${orgSlug}` is expanded when the plan is **saved**, inside `steps`, `metadata`, `inputOptions`, `inputsSchema`,
`documentFamilyGroups` and the two templates — so `moduleRef: "${orgSlug}/helpers"` works while
`promptTemplateRef: "${orgSlug}/…"` does not. `${org}/` is a different placeholder, expanded by `kdx` at push.

## Dependency and action edges

| `dependsOn` entry | Ready when the upstream is |
|---|---|
| `"slug"` | COMPLETED or SKIPPED |
| `"slug?"` | also NOT_TAKEN (await). **Requires a `conditionExpr`** unless it is a valid `ANY_BRANCH` join, else the start is rejected (`await-no-condition`) |
| `"slug:action"` | COMPLETED with that action token — or SKIPPED |
| `"slug?:action"` | await, plus warned `action-qualifier-on-await-ignored`; the qualifier only means anything on an `ANY_BRANCH` branch dep. Qualify a plain dep or gate with `conditionExpr` instead |

Only `SCRIPT`, `LLM`, `BRIDGE_CALL` and `CREATE_TASK` can emit actions — an action edge off an `EXECUTION`,
`AGENT` or `APPROVAL` step is an error at start (`action-edge-upstream-cannot-emit`). Actions are declared
`{name, slug}` (`uuid` is the legacy spelling of `slug`; both, differing, is rejected). The **live** source is
`scriptActions` / `promptActions` / `bridgeActions` — and for `CREATE_TASK`, the **referenced TaskTemplate's own
actions plus the org's DONE-typed task-status slugs**; the step's inline `actions:` array is a mirror only, and an
edge matching only it fails the start.

**Every SCRIPT step must declare `scriptActions` and return one of them.** The runtime needs a returned object
with a non-empty `action` matching one by `slug` (exact) or `name` (case-insensitive); `return {}`, or returning an
action with nothing declared, fails the step. There is no `action()` function and no `ctx`/`context` global — the
globals are `inputs`, `families`, `task`, `org`, `documents`, `tasks`, `knowledge`, `loadDocument`, `serviceBridge`, `llm`, `log`.

```yaml
- slug: classify
  type: SCRIPT
  perDocument: true
  scriptBody: |
    return { action: families[0].metadata.docType === 'receipt' ? 'receipt' : 'invoice' };
  scriptActions: [{ name: Receipt, slug: receipt }, { name: Invoice, slug: invoice }]
```

## Per-document processing, routing and document status

`perDocument: true` fans a step out over the activity's document families. Valid **only** on `EXECUTION`
(default `true`), `LLM`, `SCRIPT`, `BRIDGE_CALL` (default `false`); anywhere else is an error
(`per-document-unsupported`). `maxParallel` is **EXECUTION-only** (default 5) — on LLM/SCRIPT/BRIDGE_CALL it
is an error, because those per-document paths run sequentially.

**Routing is auto-detected, never a flag**: it turns on when a `perDocument` step declaring actions is the target
of an action-qualified dep. Branch with `dependsOn: ["classify:receipt"]` / `["classify:invoice"]`, then re-join on
a `perDocument` step with `joinPolicy: ANY_BRANCH` whose branch deps are await-only (`["receipt-extract?",
"invoice-extract?"]`) — plain branch deps AND-combine per document and converge nothing. `joinPolicy` is
`ALL_SETTLED` (default), `ALL_COMPLETED`, `ANY_COMPLETED` or `ANY_BRANCH`; a document whose action matches no
edge is NOT_TAKEN on every branch and counted unrouted.

`setDocumentStatus: <slug>` works on **any** step type, resolving against the **project's** document statuses
(`kdxa_project_document_status`, not task statuses): a `perDocument` step stamps the families that COMPLETED
there, any other step stamps every family in the activity. Best-effort — an unknown slug is logged, never fatal.
Under routing, `conditionExpr` is evaluated per document and gains a `document` object (`.status` slug-or-null,
`.statusLabel`, `.locked`, `.labels`, `.path`), so a root step can gate with
`"$not(document.status in ['reviewed', 'completed'])"`; elsewhere `document.*` is inert.

## Template languages — four, in one file

| Where | Language | Right | Wrong |
|---|---|---|---|
| `defaultTitleTemplate`, `defaultDescriptionTemplate` | Go text/template | `{{ .inputs.vendorId }}` | `${inputs.vendorId}` |
| `conditionExpr`, BRIDGE_CALL `request*` values, `treatAsError`, `promptVariables`, `outputMapping` | JSONata | `"inputs.vendorId"`, literal `"'active'"` | `"$.context.vendorId"`, bare `"active"` |
| LLM `promptBody` | FString | `Classify {docType}` | `{{ docType }}` |
| `taskData.title/description/properties`, EXECUTION `options` | fixed `${…}` placeholders only | `"${activity.title}"` | `"{{ .inputs.vendorId }}"` — stays literal |
| `enrichment[].inputMapping` values | plain dot-paths | `"inputs.vendorId"` | `"$.inputs.vendorId"` — resolves to null |

JSONata roots: `conditionExpr` and BRIDGE_CALL maps see `{orgId, projectId, inputs, documentFamilyId,
extractedData, steps}`, where upstream output is `steps.<slug>.<key>`; LLM `promptVariables` see
`{context: {orgId, projectId, inputs}, enrichment, steps}`, so an enrichment result is at `enrichment.<outputKey>`,
never `context.<outputKey>`. Backtick hyphenated slugs: ``steps.`a-b`.action``. **`steps` holds COMPLETED steps
only, and in a BRIDGE_CALL `request*` map a path that matches nothing fails the step** (`eval: no results found`) —
guard the **full** path: `"$exists(steps.review.completedActionUuid) ? steps.review.completedActionUuid : ''"`.

## Validate, bind, start

- `POST /api/activity-plans/validate` with `{steps, organizationId}` returns `{valid, issues:[{stepSlug,
  code, severity, message}]}`, always 200 — the **same** checks the start enforces, but all of them; every
  `severity: "error"` means start returns 400. The separate save-time rule set is off unless a deployment
  enables it, so a clean save proves nothing.
- The plan must be bound to the project — a `kdxa_project_resources` binding of type `activity-plan` —
  else `POST /api/activities` returns **400**:
  `activity-plan "<slug>" is not bound to project <id>; create a project-resource binding first`.
- Start body: `projectId` + `activityPlanRef` required, plus optional `title`, `inputs`, `triggerKind`
  (default `MANUAL`), `documentFamilyIds`, `documentFamilyFilter`; success is 201. Also launchable from a
  Trigger, an intake script returning `{ activityPlan: 'invoice-intake', … }`, or a SCRIPT `nextActivity`.

## Declared but inert

Persisted, round-tripped, present in existing YAML — and read by nothing.

| Field | Reality |
|---|---|
| `waitForCompletion` (CREATE_TASK) | never read; the step always waits for its task to reach a DONE status |
| `disableCache` (BRIDGE_CALL) | plumbed to the request then ignored; there is no caching layer |
| `outputMapping` (BRIDGE_CALL) | accepted in YAML but never carried onto the runtime step, so `bridgeActions` never resolve and action edges off a BRIDGE_CALL never fire. Branch instead with a `conditionExpr` on the downstream steps, reading the always-present `steps.<slug>._statusCode` |
| `inputsSchema` | drives the Studio start form only |
| `approverRole`, `approvalCriteria`, all of APPROVAL | the step settles SKIPPED before anyone can act |
| CREATE_TASK inline `actions:` | folded into `taskData.actions`, never rendered, never routable |
| `badges[]` without `promote: true` | stays on the step; only promoted badges reach the activity |
| `joinPolicy` on CREATE_TASK / AGENT / APPROVAL | dropped; behaves as ALL_SETTLED (warned `join-policy-inert`) |
| `documentSummarizationPrompt`, `documentValidationPrompt` | browser upload flow only; API and intake starts bypass both, and an unreachable model fails open |
| `maxParallel` on CREATE_TASK / AGENT / APPROVAL | never persisted and never flagged — only EXECUTION reads it (on LLM/SCRIPT/BRIDGE_CALL it is a start-time error) |

## Common mistakes

| Mistake | What happens |
|---|---|
| `kind: SCRIPT` | start rejected: `step missing required slug or type` |
| `config: { moduleRef: … }` | flattened away on save; where the save-time validator runs it reports them missing |
| `action('high')` in `scriptBody` | `ReferenceError` — `return { action: 'high' }` instead |
| `return {}` from a SCRIPT | step fails: the return needs a non-empty `action` matching a `scriptActions` entry |
| `ctx.` / `context.` in `scriptBody` | no such global; use `inputs`, `families`, `documents`, `task`, `org` |
| `treatAsError: '$.status >= "400"'` | always false, so a 500 counts as success; use `$._statusCode >= 300` |
| unguarded `"steps.x.y"` / `"inputs.optional"` in a `request*` map | step fails `eval: no results found` when the path is absent; wrap in `$exists(…) ? … : …` |
| `joinPolicy: all_complete` | stored verbatim, then reduced as `ALL_SETTLED` with a warning |
| `promptTemplateRef: "${orgSlug}/x"` | prompt not found at run time; use the bare slug |
| `maxParallel` on LLM/SCRIPT/BRIDGE_CALL, `perDocument` on CREATE_TASK/AGENT/APPROVAL | error at start |
| await dep `slug?` with no `conditionExpr` | error at start (`await-no-condition`) |

See `references/step-types.md` (per-type fields, runtime contexts, limits, plan-level keys),
`references/validation.md` (every issue code) and `references/examples.md` (complete plans). Related
skills: `task-template`, `task-status`, `trigger`, `service-bridge`, `prompt-template`, `module`.
