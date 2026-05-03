---
name: task-template
description: "Use when creating or editing Kodexa task templates — org-scoped YAML defining reusable task configurations (options, forms, actions, document family groups) that projects bind to via project resources"
---

# Kodexa Task Template Authoring

## Overview

Task templates define reusable task configurations. Since the **activity refactor (2026-05-02)** they are **organization-scoped** metadata resources — one YAML file per template, synced at the org level. Projects bind to specific templates via the `kdxa_project_resources` table; the same template can be reused across many projects.

**Automation that used to live inside a task template (the old `planned: true` + `planTemplate.steps:` fields) has moved into a separate resource: `activity-plan`.** Task templates are now purely about the human-task shape — fields, forms, actions, document groups, assignment, behavior. See the `activity-plan` skill for orchestration.

## When to Use

- Defining a reusable human task (review, approval, exception resolution)
- Adding custom fields/options to a task type
- Attaching one or more data forms to a task
- Configuring document family groups
- Setting team ownership, AI naming, chat prompt, execution policy
- Defining action buttons users can click in the task UI

## What Has Changed

| Old (pre-refactor) | New |
|---|---|
| Project-scoped (one per project) | **Org-scoped** (`task-template://orgSlug/slug`) |
| `planned: true` + `planTemplate.steps:` for auto-orchestration | **Removed.** Use a separate `activity-plan` resource and a `trigger` to launch it |
| `assignmentRules: { autoAssign, assignToRole, assignToTeam }` | **Not part of the schema.** Use `metadata.teamSlug`; assignment behavior is platform-level |
| `documentGroups:` | Renamed to **`documentFamilyGroups:`** |
| Top-level `fields:` (sometimes confused with `options:`) | Single **`metadata.options:`** array (the field/option distinction is gone) |
| Action `targetStatus` / `icon` / `color` / `setAttributes` | **Folded into `properties` map** on each action |
| — | **New top-level `initialStatusSlug:`** — initial task status, resolved against the project's bound `task-status` resources |

## Top-Level Structure

```yaml
slug: invoice-review                         # Required — unique within (orgSlug, slug)
name: "Invoice Review"                       # Required — display name (DB column: title)
organizationId: my-org-uuid                  # Required — the owning org
description: "Manual review of extracted invoices"
initialStatusSlug: todo                      # Optional — must match a task-status slug bound to the project at runtime
deprecated: false                            # Optional — mark legacy plan-vehicle templates
metadata:
  options: []                                # Field definitions (see below)
  forms: []                                  # Attached data forms
  actions: []                                # Action buttons
  documentFamilyGroups: []                   # Document grouping config
  workspaceId: ""                            # Optional — workspace binding (UUID)
  properties: {}                             # Custom map (free-form)
  priority: 2                                # Default priority (int)
  teamSlug: finance                          # Default team
  aiNaming: { enabled: true, prompt: "..." } # Optional — AI-generated title/description
  chatPrompt: { enabled: false, prompt: "" } # Optional — auto chat prompt on task open
  executionPolicy: {}                        # Optional — session.ExecutionPolicy
  companion: {}                              # Optional — workspace companion agent config
```

> The `planned` and `planTemplate` fields still exist on the Go struct as **Phase 2 transition stubs** (not mapped to the database). **Do not author them in new YAML.** Phase 4 removes them outright.

## Options (Custom Fields)

The `options` array is the unified field/option list. There is no longer a separate `fields:` section.

```yaml
metadata:
  options:
    - name: approval_status
      type: select                  # text, textarea, number, date, select, boolean
      label: "Approval Status"
      required: true
      possibleValues:
        - value: approved
          label: "Approved"
        - value: rejected
          label: "Rejected"
        - value: escalated
          label: "Escalate to Manager"

    - name: review_notes
      type: textarea
      label: "Review Notes"
      required: false
      hint: "Capture rationale for the decision"

    - name: priority_score
      type: number
      label: "Priority Score"
      default: 0

    - name: due_date
      type: date
      label: "Due Date"
      required: true

    - name: is_urgent
      type: boolean
      label: "Urgent"
      falseLabel: "Normal"          # Custom label for false state

    # Conditional / advanced
    - name: extraction_mode
      type: select
      label: "Extraction Mode"
      tabName: "Settings"            # Group under a tab in the UI
      showIf: "advancedMode == true" # Conditional visibility
      developerOnly: true            # Hide unless developer-mode enabled
      showOnPopup: false             # Show in task creation popup
      possibleValues:
        - { value: auto, label: "Automatic" }
        - { value: manual, label: "Manual" }

    # Nested / list-of-objects
    - name: line_items
      type: list
      listType: object               # object, primitive
      listLabel: "Line Items"
      listDescription: "Itemized invoice rows"
      groupOptions:                  # Nested option definitions (each list entry is an object)
        - { name: description, type: text, label: "Description", required: true }
        - { name: quantity,    type: number, label: "Qty" }
        - { name: amount,      type: number, label: "Amount" }
```

| Option key | Purpose |
|---|---|
| `name` | Field identifier (required) |
| `type` | `text`, `textarea`, `number`, `date`, `select`, `boolean`, `list`, etc. |
| `subType` | Optional refinement of `type` |
| `listType` | For `type: list` — `object` for nested editor, `primitive` for scalar list |
| `groupOptions` | For `type: list` with `listType: object` — nested option defs |
| `label` | UI label |
| `falseLabel` | Custom label for `boolean` false state |
| `hint` | Inline help text |
| `description` | Longer description |
| `required` | Boolean |
| `default` | Default value |
| `possibleValues` | Array of `{ value, label, description? }` for `select` |
| `tabName` | UI tab grouping |
| `showIf` | Expression toggling visibility |
| `developerOnly` | Hidden unless developer mode |
| `showOnPopup` | Show in creation popup |
| `featureFlag` | Hide unless flag enabled |
| `displayProperties` | Free-form UI hints |
| `aliases` | Alternate names for migration/back-compat |

## Forms

Attach data forms to the task. Each form can have its own action buttons and panel visibility.

```yaml
metadata:
  forms:
    - dataFormRef: "${orgSlug}/invoice-form"   # Reference an org-level data-form
      actions:
        - { label: "Save Draft", type: save }
        - { label: "Submit",     type: submit }
      availablePanels:
        documentStores: true
        exceptions: true
        properties: false
      properties: {}                            # Free-form per-form config
```

## Actions

Action buttons in the task UI. Schema is intentionally lean — extra behavior lives in `properties`.

```yaml
metadata:
  actions:
    - uuid: "approve-action"
      type: approve                # Free-form discriminator the UI/orchestrator interprets
      label: "Approve"
      properties:
        targetStatus: approved     # Status transition (free-form key)
        icon: check
        color: green
        shortcut: "a"
        setAttributes:
          approval_status: approved
    - uuid: "reject-action"
      type: reject
      label: "Reject"
      properties:
        targetStatus: rejected
        icon: close
        color: red
```

> `TaskTemplateAction` only declares `uuid`, `type`, `label`, `properties` at the schema level. UI conventions for icon/color/targetStatus are stored inside `properties`.

## Document Family Groups

Renamed from `documentGroups`. Group documents that should be associated with the task.

```yaml
metadata:
  documentFamilyGroups:
    - name: "Source Invoices"
      notes: "Primary documents for review"
      documentFamilyFilter: "*.pdf"          # Filter expression
      maxHits: 5                              # Max families to surface
      sort: "createdOn desc"                  # Optional ordering
      automaticallyAdd: true                  # Auto-attach matching families
      editable: true                          # User may add/remove
      uploadOnly: false                       # If true, only allow uploads
      uniqueFilenames: true                   # Enforce unique filenames
      maxPages: 100
      hardMaxPages: 500
      maxSize: 10485760                       # Bytes (10 MB)
      titlePrompt: "Generate task title from this invoice"
```

## Team, Priority, AI Naming, Chat Prompt

```yaml
metadata:
  teamSlug: finance-reviewers
  priority: 2

  aiNaming:
    enabled: true
    prompt: "Generate a concise task title from the document headers."

  chatPrompt:
    enabled: true
    prompt: "Help me complete this invoice review."
```

## Execution Policy & Companion

`executionPolicy` (from `session.ExecutionPolicy`) controls how the task interacts with sessions and the orchestrator. `companion` configures the workspace companion agent surfaced when the task is open.

```yaml
metadata:
  companion:
    agentRuntimeRef: "${orgSlug}/review-agent"
    moduleRefs:
      - "${orgSlug}/review-helpers"
    prompt: "Help the reviewer complete this invoice."
```

## Complete Example

```yaml
slug: invoice-review
name: "Invoice Review"
organizationId: ${orgSlug}
description: "Manual review and approval of extracted invoice data"
initialStatusSlug: todo
metadata:
  priority: 2
  teamSlug: finance-reviewers

  options:
    - name: approval_status
      type: select
      label: "Approval"
      required: true
      possibleValues:
        - { value: approved, label: "Approved" }
        - { value: rejected, label: "Rejected" }
        - { value: escalated, label: "Escalate to Manager" }
    - name: review_notes
      type: textarea
      label: "Review Notes"
    - name: confidence_override
      type: number
      label: "Manual Confidence Score"

  forms:
    - dataFormRef: "${orgSlug}/invoice-form"
      actions:
        - { label: "Submit", type: submit }

  actions:
    - uuid: approve
      type: approve
      label: "Approve"
      properties:
        targetStatus: approved
        icon: check
        color: green
        shortcut: "a"
        setAttributes:
          approval_status: approved
    - uuid: reject
      type: reject
      label: "Reject"
      properties:
        targetStatus: rejected
        icon: close
        color: red
        shortcut: "r"

  documentFamilyGroups:
    - name: "Invoice"
      documentFamilyFilter: "*.pdf"
      maxHits: 3
      automaticallyAdd: true
      editable: false
      titlePrompt: "Generate a review task title from the invoice"

  aiNaming:
    enabled: true
    prompt: "Title from invoice number and vendor."
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Authoring `planned: true` + `planTemplate.steps:` | Remove. Use a separate `activity-plan` resource and a `trigger` to launch it on task creation. |
| Using `assignmentRules:` | Not in the schema. Use `metadata.teamSlug` for team ownership. |
| Using `documentGroups:` | Renamed to `documentFamilyGroups:`. |
| Splitting fields into separate `fields:` and `options:` arrays | Use a single `metadata.options:` array. |
| Putting `targetStatus` / `icon` / `color` directly on an action | Move them into the action's `properties:` map. |
| Project-scoped slug references like `task-template://org/project/slug` | Templates are org-scoped now: `task-template://orgSlug/slug` (2 parts only). |
| Hardcoding a status UUID | Use `initialStatusSlug:` and ensure the project has the matching `task-status` bound. |
| Re-creating the same template per project | Define once at org level; each project binds to it via project resources. |
