---
name: assistant
description: "Use when creating, editing or reviewing a Kodexa assistant — the project-scoped record that holds options.pipeline steps, taxonomy refs and agent config, and that supplies the identity a project's executions run as. Also read this before wiring anything event-driven or scheduled, because that is authored as an activity-plan plus a trigger, not as an assistant."
---

# Kodexa Assistant Authoring

## Status: legacy, still supported, no longer an execution path

An assistant is a **project-scoped record** (`kdxa_assistants`) holding configuration, and the identity
a project's executions are attributed to. In current platform versions it does not run anything itself:

- `subscription` is stored but no evaluator reads it. Assistants never self-trigger.
- Nothing reads `options.pipeline` to build an execution. It persists and round-trips faithfully; it is
  never dispatched.
- Assistant connections were removed (rows drained, no entity, no endpoint, no project-template field)
  and the assistant-to-store join table was dropped outright.
- **Nothing validates an assistant file.** Activity plans, triggers, task templates, data forms and
  service bridges all have server-side validators; assistants have none, and unknown keys — top level
  or inside `options` — are dropped by a typed unmarshal that never rejects. A wrong key is a silent
  no-op, never an error — which is why everything under "Declared but inert" below fails quietly.

If you want work to actually happen, author an `activity-plan` and a `trigger`; both have skills here.

## Assistants vs activity plans vs triggers

| What you want | Author this |
|---|---|
| Something to start when an event happens | A **trigger** (`kdxa_triggers`) pointing at an activity plan. The six accepted event kinds are `task_created`, `task_status_changed`, `activity_completed`, `manual`, `document_locked`, `knowledge_set_updated`; filtering is a JSONata `eventFilter`. See the `trigger` skill. |
| A module to actually run over a document | An **activity-plan** step of `type: EXECUTION` — the only path that builds a runnable pipeline |
| Branching, tasks, approvals, scripts, a service-bridge HTTP call | An **activity-plan**. The step discriminator key is `type:` (not `kind:`); accepted values are `EXECUTION`, `BRIDGE_CALL`, `CREATE_TASK`, `SCRIPT`, `APPROVAL`, `LLM`, `AI_PROMPT`, `AGENT` |
| Cron / periodic work | Not available in-platform. `schedule` is rejected as a trigger event kind and there is no scheduled-jobs API. Drive it from outside. |
| A named project-scoped record that *declares* taxonomy refs, a module pipeline or agent config — configuration that persists but does not run | An **assistant** (this skill) |
| An identity for plan-run executions | The auto-provisioned system Task Assistant — never author one |
| To read runtime state | The rows are `Activity` (`kdxa_activities`) and `Step` (`kdxa_steps`); an activity's runtime field is `lifecycleState`, not `status` |

## Where the file lives

`<metadata_dir>/projects/<projectSlug>/assistants/<slug>.yaml`, pushed with `kdx sync push`. Assistants
are project-scoped, so `kdx` reads and writes them under `projects/<projectSlug>/`, never flat.

```yaml
# .../projects/invoice-processing/assistants/invoice-extractor.yaml
slug: invoice-extractor          # ^[a-zA-Z0-9_-]+$, max 255
name: Invoice Extractor          # required; also the soft-delete unique column
description: Parses and extracts invoice data
active: true                     # nullable bool, defaults true when omitted
assistantRole: extractor         # free text — 'TASK' is RESERVED, see below
chatEnabled: true
showInTraining: true
runOnExistingContent: true
priorityHint: 0
color: "#4F46E5"                 # max 25 chars
options:
  taxonomies: [acme-corp/invoice-data]
  pipeline:
    steps:
      - ref: kodexa/fast-pdf-model     # module://, model://, model-runtime:// or bare org/slug
        name: Parse PDF                # `ref` is the only required key on a step
        stepType: MODULE               # see below
        options:                       # module options belong HERE, not on the assistant
          min_words_per_page: 10
      - ref: kodexa/llm-taxonomy-model
        name: Extract fields
        stepType: MODULE
        options:
          taxonomy: acme-corp/invoice-data
        conditional: "metadata.documentType == 'invoice'"
        executionPolicy:               # all five keys optional — defaults below
          timeoutSeconds: 900
          onExhausted: fail            # fail | skip
  complete_label: reviewed             # snake_case — the only such key on this resource
```

Two more top-level keys round-trip, after `options`: `testOptions` then `subscription`. Keeping the
whole file in this key order stops `kdx` reformatting it on the next pull.

## Step shape: refs, stepType, conditionals, execution policy

These are the semantics the runtime applies to an execution step of this shape. An assistant's copy is
stored and never dispatched, so author it correctly — you cannot observe it by firing the assistant.

**Refs are unversioned** — `ref: kodexa/fast-pdf-model:1.0.0` is wrong; a legacy `:version` suffix is
stripped, and version-suffixed URIs are rejected by the resolver outright. **Module options go on the
step**, as a sibling of `ref`; that map is the only part of the file where `kdx` rewrites sigils and
variables on pull/push. **`stepType`** is a free string; `MODULE` is the platform's only defined value
and the one the runtime emits, while older YAML often carries `MODEL`. Write `MODULE`.

**`conditional`** is a `simpleeval` **Python** expression, not the older filter syntax. Three names are
in scope, all with dot access: `context` (the execution context), `metadata` (shorthand for
`context['metadata']`) and `input_event` (the previous step's output). Use Python operators — `and`,
`or`, `not`, `True`/`False`; there is no `hasMixins()` helper and `AND` is a syntax error. Empty or
absent means the step runs. Two silent-skip traps: an expression that **errors is treated as false and
the step is skipped**, and a **name that is not in the payload resolves to `None`** rather than
raising — so `metadata.documentTyp == 'invoice'` is simply false and the step vanishes with no error.

**`executionPolicy`** keys are all optional; unset ones fall back to `timeoutSeconds: 900`,
`maxAttempts: 1`, `backoffStrategy: immediate`, `backoffBaseSeconds: 0`, `onExhausted: fail`. `skip`
lets the rest of the pipeline continue past a failed step. These five are the only failure knobs in
this shape — there is no "move to exceptions" or "notify" behaviour.

## Assistants inside a project template

A project template can carry an `assistants:` list. Project creation copies exactly `name, slug,
description, subscription, showInTraining, chatEnabled, assistantRole, priorityHint, options`, and
forces `active` true. Two traps:

**Flat `options` are destroyed.** A template's `options` is parsed as a free-form map, then re-parsed
into the typed shape, keeping only `taxonomies`, `pipeline`, `complete_label`, `prompt`, `moduleRefs`,
`agentRuntimeId`. Anything else is gone before the row is written and cannot come back.

```yaml
options:                     # WRONG — the key silently vanishes
  min_words_per_page: 10

options:                     # RIGHT
  pipeline:
    steps:
      - ref: kodexa/fast-pdf-model
        stepType: MODULE
        options: { min_words_per_page: 10 }
```

**Booleans invert their default inside a template.** On the standalone resource `showInTraining`,
`chatEnabled`, `active` and `runOnExistingContent` are nullable and default to **true** when omitted;
in a template `showInTraining` and `chatEnabled` are plain booleans wrapped unconditionally, so
**omitting them persists `false`**. Write every boolean you care about explicitly in template YAML.
(`runOnExistingContent` and `color` have no template field at all.)

`${assistant.<Name>.id}` is keyed on the assistant's **name** (spaces included), not its slug, and is
registered only as each assistant is created — late in project creation. So only assistants further
down the same `assistants:` list can use it; anything earlier gets the literal string back. The
variables seeded up front are `${project.id}`, `${project.name}`, `${orgSlug}` and its alias `${org}`.

## Referencing one, and the reserved system assistant

`assistant://<orgSlug>/<projectSlug>/<assistantSlug>` — **three** segments. A two-segment
`assistant://acme-corp/invoice-extractor` is rejected with "project slug required". Version-suffixed
URIs are rejected outright, and soft-deleted assistants are excluded from resolution.

`assistantRole` is free text with one reserved value: **`TASK`**. Every project gets an
auto-provisioned system assistant — name `Task Assistant`, slug `task-assistant`, role `TASK` — created
with the project and re-created if missing when an activity is materialized. Activity-plan `EXECUTION`
steps are attributed to it, and it backs the assistant-type platform user and scoped token used to
dispatch step work. Never author that slug or that role; you will collide with it.

## What actually executes

Executions get their pipeline from an activity-plan `EXECUTION` step, which builds a single-`MODULE`
pipeline attributed to the Task Assistant. Your assistant's `options.pipeline` is not consulted.

`POST /api/projects/{id}/assistants/{assistantId}/events` and
`PUT /api/projects/{id}/assistants/{assistantId}/schedule` both create an execution with **no
pipeline**, which the scheduler immediately marks `FAILED`. Firing an assistant from the UI or the API
produces a failed execution, not a run — current behaviour, not a bug in your YAML.

`kdx` writes through `/api/assistants` and `/api/assistants/{id}`; the activate, deactivate, events,
executions and schedule routes hang off `/api/projects/{id}/assistants/...`. Permissions are
`assistant:` `read` / `create` / `update` / `delete` / `activate` / `deactivate` / `trigger`, plus
`execution:read` to list executions. Deletes are soft — the row stays with `deleted = true` and a
mangled `name`, and the resolver skips it. A stale `changeSequence` on update is rejected with 409;
omitting the key opts out of the check.

## Declared but inert

**Parsed and discarded — never persisted.** These still appear in older YAML and in the key order
`kdx` writes back, so they look endorsed. They are not.

| Key | What happens |
|---|---|
| `connections:` | Removed. No entity, no endpoint, no template field; the backing rows were deleted. Migrate to an activity-plan plus a trigger. |
| `stores:` | Removed. The backing table was dropped; the store-list endpoint is in the spec but has no route and 404s. Scope stores with project-resource bindings and step options instead. |
| `schedules:` / `schedulable:` | Removed. No column, no scheduled-jobs API, and `schedule` is not a valid trigger event kind. |
| `assistantDefinitionRef:` | Parses, never copied onto the created assistant. There is no assistant-definition entity or endpoint. |
| `loggingEnabled:` / `deleteLoggingOnSuccess:` / `definition:` | Columns dropped or never existed on the model. Silent no-op. |
| `options.properties:`, `options.data_store:`, `options.write_back_to_store:`, any other flat `options` key | Not one of the six typed `options` keys, so dropped on write. Common in older templates. `kdx` even rewrites sigils inside `options.properties` on pull/push — that is formatting, not support. |
| `reactive:` / `publicAccess:` / `template:` / `type:` | Belong to org-scoped metadata resources. An assistant is not one — it has no organization, no public-access flag and no type. |

**Persisted and round-tripped, but nothing reads them.**

| Key | Note |
|---|---|
| `subscription` | No evaluator anywhere. Does not cause the assistant to fire. |
| `options.pipeline` / `options.taxonomies` / `options.complete_label` | Stored intact; no code path reads them to run or label anything. |
| `options.prompt` / `options.moduleRefs` / `options.agentRuntimeId` | Agent configuration in waiting; the live agent path takes its prompt and module refs from channel types, not from here. |
| `priorityHint` | Never mapped onto execution priority. If it is ever wired, note that scheduling orders priority **ascending** — lower is more urgent, the opposite of "higher wins". |
| `chatEnabled` / `runOnExistingContent` | No reader in the backend or the UI. |
| `showInTraining` / `color` / `testOptions` | UI-only: they filter the workspace assistants panel, colour a swatch, and supply options to the events endpoint whose executions fail as described above. |

## Common mistakes

| Mistake | What actually happens |
|---|---|
| Expecting the assistant to fire on document arrival | Nothing fires. Author an activity-plan plus a trigger. |
| Module options directly under the assistant's `options:` | Dropped by the typed unmarshal. Put them on `options.pipeline.steps[].options`. |
| Omitting `showInTraining` / `chatEnabled` in a project template | Persists `false`, not the resource default `true`. |
| `completeLabel` | The wire key is `complete_label` — snake_case. |
| `:version` on a step `ref`, or a two-segment `assistant://org/slug` | Versions were stripped platform-wide; assistant URIs need three segments. |
| `AND` or `hasMixins()` in a `conditional` | It is Python: `and`, `or`, `not`. A bad expression skips the step silently. |
| A Python `Assistant` subclass with `process()` | Modules are plain functions — the runtime imports `infer()`, falling back to `handle_event()`. See the `module` skill. |
