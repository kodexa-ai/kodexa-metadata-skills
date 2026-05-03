---
name: activity-plan
description: "Use when creating or editing Kodexa activity plans — org-scoped YAML defining a step graph (CREATE_TASK, EXECUTION, BRIDGE_CALL, SCRIPT, LLM, APPROVAL, AGENT) plus inputs schema and default title/description templates. Replaces the old plan_template inside task-templates."
---

# Kodexa Activity Plan Authoring

## Overview

An **ActivityPlan** is the org-level definition of automated work — a graph of steps, an inputs schema, and default templates for title/description rendering. Activities are *runtime instances* of a plan; the plan itself is metadata, bound to projects via `project_resources`.

Introduced by the **activity refactor (2026-05-02)**. Activity plans replace the inline `planTemplate` that used to live inside task templates. They are first-class resources, syncable via `kdx-cli` (push order 50), and resolvable via `activity-plan://orgSlug/slug`.

## When to Use

- Defining an automated workflow (extraction → review → approval)
- Migrating an old `task_template.planTemplate` block into its new home
- Wiring a service-bridge call into an activity (`BRIDGE_CALL` step)
- Adding a human-in-the-loop step (`CREATE_TASK` with `waitForCompletion`)
- Defining a multi-step LLM/script pipeline tied to a project

## How Activity Plans Fit Together

```
┌─────────────────┐     ┌──────────────────┐     ┌───────────────┐
│   Trigger       │     │   ActivityPlan   │     │  TaskTemplate │
│  (project)      │ →   │   (org)          │ →   │  (org)        │
│  eventKind +    │     │  steps[].kind=   │     │  shape of the │
│  filter         │     │  CREATE_TASK     │     │  human task   │
└─────────────────┘     └──────────────────┘     └───────────────┘
   fires on event       runs steps in order      materialized by
                                                 CREATE_TASK steps
```

A `Trigger` (project-scoped) listens for an event (`task_created`, `task_status_changed`, `activity_completed`, `manual`), then **starts the referenced ActivityPlan as an Activity**. Each `CREATE_TASK` step in the plan materializes a Task from a TaskTemplate.

## Top-Level Structure

```yaml
slug: invoice-processing                      # Required — unique within (orgSlug, slug)
name: "Invoice Processing Workflow"           # Required — display name (from AbstractMetadata)
organizationId: ${orgSlug}                    # Required
description: "Extract → review → approve invoices"

inputsSchema:                                 # Optional — JSON-Schema; validated at start time
  type: object
  required: [documentId]
  properties:
    documentId: { type: string }
    priority:   { type: string, enum: [low, medium, high] }

defaultTitleTemplate: "Process invoice {{ .inputs.documentId }}"
defaultDescriptionTemplate: |
  Automated processing of {{ .inputs.documentId }} (priority: {{ .inputs.priority }})

steps:                                        # Required — ordered graph of step nodes
  - slug: extract
    kind: EXECUTION
    config:
      moduleRef: "${orgSlug}/invoice-extractor"
      options: {}

  - slug: review
    kind: CREATE_TASK
    dependsOn: [extract]
    config:
      taskTemplateRef: invoice-review
      taskStatusSlug: todo
      waitForCompletion: true

metadata:                                     # Optional — description, icon, provider, etc.
  description: "Standard invoice intake → review pipeline"
  icon: "file-invoice"
```

### Template scope (Go text/template)

`defaultTitleTemplate` and `defaultDescriptionTemplate` use Go text/template syntax. Context root: `{ inputs, org, project, caller, now }`. Missing keys render as empty strings. No HTML escaping (titles are plain text).

### Inputs schema

`inputsSchema` is JSON-Schema-shaped. Validation runs at activity start time; failures return HTTP 400 with field-level errors.

## Step Shape (Universal)

Every step in `steps` shares the same envelope:

```yaml
- slug: "string"                              # Unique within the activity plan
  kind: "CREATE_TASK | EXECUTION | BRIDGE_CALL | SCRIPT | LLM | APPROVAL | AGENT"
  conditionExpr: "string"                     # Optional — predicate; step skipped if false
  badges:                                     # Optional — UI-only
    - { icon: "...", color: "...", label: "..." }
  dependsOn: ["other-slug", "other-slug:OUTCOME"]   # Optional
  config: { ... }                             # Per-kind, see below
```

**`dependsOn` notes.** Each entry is either a step slug (run after that step succeeds) or `slug:OUTCOME` to gate on a specific action outcome (relevant for `SCRIPT`, `APPROVAL`, `LLM`).

**`conditionExpr`.** Single string predicate. (Migrated from the old `condition: { expr: "..." }` shape, which is now flattened.)

## Step Kinds

### CREATE_TASK — materialize a Task from a TaskTemplate

```yaml
- slug: review
  kind: CREATE_TASK
  config:
    taskTemplateRef: invoice-review           # Required — TaskTemplate slug (org-scoped)
    taskStatusSlug: todo                      # Optional — defaults to template's initialStatusSlug
    taskData:                                 # Optional — overrides for the materialized task
      title: "Review invoice {{ .inputs.documentId }}"
      description: ""
      priority: 2
      properties: {}
    waitForCompletion: true                   # Optional — default true; step blocks until task reaches a terminal status
```

### EXECUTION — run a module

```yaml
- slug: extract
  kind: EXECUTION
  config:
    moduleRef: "${orgSlug}/invoice-extractor" # Required — module URI or slug
    options:                                  # Optional — module-specific config
      confidence_threshold: 0.85
    bypass: false                             # Optional — skip execution and mark COMPLETED
    perDocument: true                         # Optional — iterate over the activity's document family
    maxParallel: 4                            # Optional — parallelism cap when perDocument=true
    joinPolicy: all_complete                  # Optional — all_complete | any_complete
```

### BRIDGE_CALL — call a service bridge HTTP endpoint

```yaml
- slug: post-to-erp
  kind: BRIDGE_CALL
  config:
    serviceBridgeRef: "${orgSlug}/erp-bridge" # Required
    endpointName: createInvoice               # Required — named endpoint within the bridge
    requestBody:                              # JSONata expressions or static body
      number: "$.inputs.invoiceNumber"
      amount: "$.context.extracted.total"
    requestQuery:                             # Optional — query-string template
      mode: "auto"
    requestPath:                              # Optional — path-segment substitution
      vendorId: "$.inputs.vendorId"
    requestHeaders:                           # Optional
      X-Idempotency-Key: "$.context.idempotencyKey"
    requestScript: ""                         # Optional — GoJA preprocessor; mutex with the four request_* maps
    treatAsError: '$.status >= "400"'         # Optional — JSONata predicate that marks response as error
    timeoutSeconds: 30
    disableCache: false
    bridgeActions:                            # Optional — action descriptors emitted by the call
      - { uuid: "ok", name: "submitted" }
```

### SCRIPT — execute JavaScript (GoJA)

```yaml
- slug: classify
  kind: SCRIPT
  config:
    scriptBody: |                             # Required — JS code
      const total = context.extracted.total;
      action(total > 10000 ? 'high' : 'normal');
    scriptActions:                            # Optional — declared action outcomes
      - { uuid: "high",   name: "high" }
      - { uuid: "normal", name: "normal" }
    scriptSidecars:                           # Optional — module refs whose JS is pre-loaded
      - "${orgSlug}/script-helpers"
```

> **Migration note.** Old `planTemplate` items used the key `script:` for the body. The activity-plan canonical key is **`scriptBody:`** — Step A migrations rename automatically; new authoring should use `scriptBody`.

### LLM — call an LLM with a prompt

```yaml
- slug: extract-fields
  kind: LLM
  config:
    promptBody: "Extract invoice number and total from the document."   # Either promptBody…
    promptTemplateRef: "${orgSlug}/invoice-extract-prompt"              # …or promptTemplateRef
    llmModelName: claude-sonnet-4-6           # Optional — overrides default
    enrichment:                               # Optional — service-bridge calls run before the prompt
      - serviceBridgeRef: "${orgSlug}/vendor-lookup"
        operation: lookupVendor
        inputMapping: { vendorName: "$.inputs.vendor" }
        outputKey: vendorRecord
    includeDocument: true                     # Attach activity's document family content
    outputMapping:                            # JSONata mapping LLM output → step outputs
      invoiceNumber: "$.number"
      amount: "$.total"
    promptVariables:                          # JSONata expressions providing variables
      vendor: "$.context.vendorRecord.name"
    promptActions:                            # Optional — declared action outcomes for outputMapping
      - { uuid: "ok", name: "extracted" }
```

> Was named `AI_PROMPT` / `AIPLANNER` in the old plan_template schema. The kind value is now **`LLM`**; migrations remap automatically.

### APPROVAL — human approval gate

```yaml
- slug: approval
  kind: APPROVAL
  config:
    approverRole: finance-manager             # Required — role slug
    approvalCriteria:                         # Optional — predicate over the activity context
      condition: "$.context.extracted.total > 10000"
```

Outcomes are written to the runtime `approval_outcome` column.

### AGENT — invoke a project-scoped Assistant

> **Provisional.** Column wiring not yet finalized. Verify against the orchestrator's agent-step handler before authoring.

```yaml
- slug: agent-review
  kind: AGENT
  config:
    assistantRef: "${orgSlug}/review-agent"   # Assistant slug, resolved within the project
    agentInputs:
      context: "$.inputs"
```

## Quick Reference — Step Kinds

| Kind | Required config keys | Common optional keys |
|---|---|---|
| `CREATE_TASK` | `taskTemplateRef` | `taskStatusSlug`, `taskData`, `waitForCompletion` |
| `EXECUTION` | `moduleRef` | `options`, `perDocument`, `maxParallel`, `joinPolicy`, `bypass` |
| `BRIDGE_CALL` | `serviceBridgeRef`, `endpointName` | `requestBody`, `requestQuery`, `requestPath`, `requestHeaders`, `requestScript`, `treatAsError`, `bridgeActions`, `timeoutSeconds`, `disableCache` |
| `SCRIPT` | `scriptBody` | `scriptActions`, `scriptSidecars` |
| `LLM` | `promptBody` *or* `promptTemplateRef` | `llmModelName`, `enrichment`, `includeDocument`, `outputMapping`, `promptVariables`, `promptActions` |
| `APPROVAL` | `approverRole` | `approvalCriteria` |
| `AGENT` *(provisional)* | `assistantRef` | `agentInputs` |

### Kind-Value Migration Map (pre- vs post-refactor)

| Old `item_type` | New `kind` |
|---|---|
| `TASK` | `CREATE_TASK` |
| `EXECUTION` (no `service_bridge_ref`) | `EXECUTION` |
| `EXECUTION` (with `service_bridge_ref`) | `BRIDGE_CALL` |
| `BRIDGE_CALL` | `BRIDGE_CALL` |
| `SCRIPT` | `SCRIPT` |
| `AI_PROMPT` / `AIPLANNER` | `LLM` |
| `AGENT` | `AGENT` |
| `APPROVAL` | `APPROVAL` |

## Complete Example

```yaml
slug: invoice-processing
name: "Invoice Processing Workflow"
organizationId: ${orgSlug}
description: "Extract invoice fields, run rules, optionally route to human review."

inputsSchema:
  type: object
  required: [documentId]
  properties:
    documentId: { type: string }
    priority:   { type: string, enum: [low, medium, high] }

defaultTitleTemplate: "Process invoice {{ .inputs.documentId }}"
defaultDescriptionTemplate: "Automated processing of {{ .inputs.documentId }}"

steps:
  - slug: extract
    kind: EXECUTION
    config:
      moduleRef: "${orgSlug}/invoice-extractor"
      options:
        confidence_threshold: 0.85

  - slug: classify
    kind: SCRIPT
    dependsOn: [extract]
    config:
      scriptBody: |
        const total = context.extracted.total;
        action(total > 10000 ? 'high' : 'normal');
      scriptActions:
        - { uuid: "high",   name: "high" }
        - { uuid: "normal", name: "normal" }

  - slug: post-to-erp
    kind: BRIDGE_CALL
    dependsOn: ["classify:normal"]
    config:
      serviceBridgeRef: "${orgSlug}/erp-bridge"
      endpointName: createInvoice
      requestBody:
        number: "$.context.extracted.number"
        amount: "$.context.extracted.total"
      treatAsError: '$.status >= "400"'

  - slug: review
    kind: CREATE_TASK
    dependsOn: ["classify:high"]
    config:
      taskTemplateRef: invoice-review
      taskStatusSlug: todo
      waitForCompletion: true
      taskData:
        title: "High-value invoice — {{ .inputs.documentId }}"

  - slug: post-after-review
    kind: BRIDGE_CALL
    dependsOn: [review]
    config:
      serviceBridgeRef: "${orgSlug}/erp-bridge"
      endpointName: createInvoice
      requestBody:
        number: "$.context.extracted.number"
        amount: "$.context.extracted.total"

metadata:
  description: "Auto-process invoices; route high-value to human review."
  icon: "file-invoice"
```

## Launching an ActivityPlan

Activity plans don't run on their own. They are launched by:

1. **A `Trigger`** (project-scoped) firing on a project event — see the `project-template` skill (Triggers section).
2. **A `CREATE_TASK` step** in another activity plan, when its child task is configured to spawn one.
3. **A direct API call** to `POST /api/activities` with the plan slug.
4. **An Intake** YAML returning an `activityPlanId` from its GoJA script (replaces old `taskTemplateId`).

## Project Binding

For an ActivityPlan to be usable in a project, its URI must be bound in `kdxa_project_resources`. Three ways to bind:

- **kdx-cli sync/deploy**: include the plan slug in the project's resource manifest.
- **Project template**: list the plan in the template's resource refs (when authoring projects from a template).
- **UI**: bind via the Studio resources panel.

If a project tries to start an ActivityPlan that isn't bound, the orchestrator returns **422 Unprocessable Entity**.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Using old kind values (`TASK`, `AI_PROMPT`, `AIPLANNER`) | Use `CREATE_TASK`, `LLM`. |
| Putting plan steps inside a task-template's `planTemplate:` | Move to a separate `activity-plan` resource and reference via `taskTemplateRef`/`Trigger`. |
| Using `script:` instead of `scriptBody:` in a SCRIPT step | Authoritative key is `scriptBody`. |
| `condition: { expr: "..." }` on a step | Flatten to a top-level `conditionExpr: "..."` string. |
| `dependencies:` on a step | Source key is **`dependsOn`** — `dependencies` is silently ignored. |
| `EXECUTION` step with `serviceBridgeRef` set | Use `kind: BRIDGE_CALL` instead. EXECUTION is module-only. |
| Authoring an activity plan directly under a project | Plans are **org-scoped**. Bind to projects via project resources. |
| Starting an unbound plan in a project | Bind the plan to the project first (UI / template / kdx). The orchestrator returns 422 otherwise. |
| Forgetting `waitForCompletion` on a `CREATE_TASK` you intend to gate on | Default is `true`, but be explicit when the next step depends on the task's outcome. |
