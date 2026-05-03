---
name: project-template
description: "Use when creating or editing Kodexa project templates — YAML blueprints that define complete project configurations including stores, assistants, taxonomies, data forms, workspaces, knowledge sets, task templates, status workflows, scheduled jobs, triggers, and activity-plan bindings"
---

# Kodexa Project Template Authoring

## Overview

Project templates are blueprints for Kodexa projects. They define stores, assistants, taxonomies, data forms, status workflows, knowledge sets, scheduled jobs, and the project's option schema. When a user creates a project from a template, all components are provisioned and bound together.

Since the **activity refactor (2026-05-02)** the platform also exposes three first-class resources that compose with project templates:

- **`task-template`** — org-scoped human-task definition (was project-scoped). Templates can be defined inline in a project-template *or* defined once at the org level and bound to a project.
- **`task-status`** — org-scoped status definition. The project-template's inline `taskStatuses:` array continues to work as a project-level workflow, but org-level `task-status` resources can be bound for reuse across projects.
- **`activity-plan`** + **`trigger`** — replace the old inline `planTemplate:` inside task templates. `Trigger` is a project-scoped resource (separate YAML) that listens for events and starts a referenced ActivityPlan.

See the `task-template`, `task-status` (this skill), `activity-plan`, and `trigger` (this skill) sections for details.

## When to Use

- Creating a new project template from scratch
- Adding stores, assistants, or data forms to an existing template
- Setting up document/task/attribute status workflows
- Defining project options (user-configurable settings) and task options
- Wiring assistant connections and event subscriptions
- Configuring a scheduled job
- Wiring a trigger to launch an ActivityPlan

## Top-Level Structure

```yaml
slug: my-template                     # Required — unique identifier
orgSlug: my-org                       # Required
type: projectTemplate                 # Required — must be "projectTemplate"
name: "My Project Template"           # Required — display name
version: "1.0.0"                      # Semantic version
description: "Template purpose"
publicAccess: false
deleteProtection: true
icon: "file-invoice"
imageUrl: "https://..."
helpUrl: "https://docs.example.com"
overviewMarkdown: |
  Markdown shown in the template selector.
provider: "Acme"
providerUrl: "https://acme.com"
providerImageUrl: "https://acme.com/logo.png"
deprecated: false
extensionPackRef: ""                  # Set when distributed via an extension pack

# Component collections
stores: []
assistants: []
taxonomies: []
dataForms: []
documentStatuses: []
taskStatuses: []
attributeStatuses: []
taskTemplates: []
scheduledJobs: []
knowledgeSets: []
tags: []
options: {}                           # ProjectOptions (see below)
memory: {}                            # ProjectMemory (filters/queries/dashboards)
```

## Template Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `${project.id}` | Unique project ID | `abc-123-def-456` |
| `${project.name}` | Project display name | `Invoice Processing` |
| `${project.slug}` | Project slug | `invoice-processing` |
| `${orgSlug}` | Organization slug | `acme-corp` |

Use these in store slugs and `sourceRef`/`stores` references to keep them unique per project.

## Component Schemas

### Stores

```yaml
stores:
  - slug: "${project.id}-documents"
    name: "Documents"
    storeType: DOCUMENT                 # DOCUMENT, DATA
    storePurpose: OPERATIONAL           # OPERATIONAL, TRAINING
    description: "Purpose"
    deleteProtection: true
    highQualityPreview: true
    showThumbnails: true
    showStoreInLabeling: true
    allowDataEditing: false
    documentProperties:                 # Metadata columns (free-form)
      - name: vendor_name
        type: string
        label: "Vendor Name"
    labelExpressions:
      - label: "processed"
        expression: "status == 'completed'"
    files: []                           # Optional bootstrap files
```

### Assistants

```yaml
assistants:
  - name: "Document Processor"
    slug: doc-processor
    description: "Processes incoming documents"
    assistantDefinitionRef: kodexa/pdf-extractor
    priorityHint: 10
    loggingEnabled: true
    chatEnabled: false
    showInTraining: false
    assistantRole: ""                   # Optional role tag
    connections:
      - sourceType: STORE               # STORE, CHANNEL, DOCUMENT_FAMILY, etc.
        sourceRef: "${orgSlug}/${project.id}-intake"
        subscription: "!hasMixins('processed')"
    stores:
      - "${orgSlug}/${project.id}-intake"
      - "${orgSlug}/${project.id}-output"
    options:
      confidence_threshold: 0.85
    schedules:
      - { type: cron, cronExpression: "0 0 8 * * *" }
```

### Taxonomies

```yaml
taxonomies:
  - slug: categories
    name: "Document Categories"
    taxonomyType: CONTENT               # CONTENT, TASK, DOCUMENT
    taxons: { ... }                     # See data-definition skill
  - templateRef: "kodexa/standard-taxonomy"   # Or reference an existing taxonomy
```

### Data Forms

```yaml
dataForms:
  - slug: invoice-review
    name: "Invoice Review Form"
    description: "Review extracted fields"
    entrypoints:
      - documentFamily
      - task
    cards: { ... }                      # V2 component tree (see data-form skill)
  - templateRef: "${orgSlug}/shared-form"
```

### Document Statuses

```yaml
documentStatuses:
  - status: "New"
    slug: new
    color: "#6B7280"
    icon: "inbox"
    statusType: UNRESOLVED              # UNRESOLVED, RESOLVED
  - status: "Processing"
    slug: processing
    color: "#3B82F6"
  - status: "Complete"
    slug: complete
    color: "#10B981"
    statusType: RESOLVED
```

### Task Statuses (inline, project-level)

The inline `taskStatuses:` array remains supported for project-template-driven projects.

```yaml
taskStatuses:
  - label: "To Do"
    slug: todo
    color: "#6B7280"
    icon: "circle"
    statusType: OPEN                    # OPEN, IN_PROGRESS, DONE, BLOCKED
    locked: false
  - label: "In Progress"
    slug: in-progress
    color: "#3B82F6"
    statusType: IN_PROGRESS
  - label: "Done"
    slug: done
    color: "#10B981"
    statusType: DONE
```

> **Activity refactor note.** As of 2026-05-02 there is also an org-scoped `task-status` resource (one YAML per status, synced to `kdxa_task_statuses`, resolvable via `task-status://orgSlug/slug`). Projects bind to org-level statuses via the `kdxa_project_resources` table — there is no inline `refs:` block in project-template YAML; binding happens at sync/deploy time. The inline form above remains valid for templated project provisioning.

### Task Statuses (org-scoped resource)

Authored as a separate YAML file at the org level:

```yaml
# task-statuses/in-review.yaml
slug: in-review
name: "In Review"
organizationId: ${orgSlug}
label: "In Review"
color: "#F59E0B"
icon: "hourglass"
statusType: IN_PROGRESS                 # OPEN, IN_PROGRESS, DONE, BLOCKED
sequence: 2                             # Ordering hint within statusType group
locked: false                           # If true, transitions away are blocked
metadata:
  description: "Awaiting reviewer action"
```

Push order is **65** (with task-templates and activity-plans), so org statuses exist before assistants/triggers reference them.

### Attribute Statuses

```yaml
attributeStatuses:
  - status: "Pending"
    color: "#F59E0B"
    icon: "hourglass"
    statusType: UNRESOLVED
  - status: "Validated"
    color: "#10B981"
    statusType: RESOLVED
```

### Task Templates (inline, project-level)

The inline `taskTemplates:` block remains valid for project-template-driven projects; it produces the same downstream task-template rows that org-level templates would.

```yaml
taskTemplates:
  - name: "Review Task"
    slug: review-task
    description: "Manual review task"
    metadata:
      priority: 2
      teamSlug: review-team
      options: []                       # See task-template skill
      forms: []
      actions: []
      documentFamilyGroups: []
```

> **Do not author inline `planned: true` / `planTemplate.steps: ...` blocks.** Those fields are Phase 2 transition stubs (no longer mapped to the database). Move orchestration into a separate `activity-plan` resource and use a `trigger` to launch it.

For org-level reuse, define one `task-template` YAML per template at the org level (see the **task-template** skill) and bind via project resources.

### Triggers (project-scoped automation)

A **`Trigger`** is a project-scoped resource that listens for an event and starts a referenced ActivityPlan. Triggers are **separate YAML files** (one per trigger), not embedded in project-template.

```yaml
# triggers/auto-review-on-task-created.yaml
slug: auto-review-on-task-created
name: "Auto-trigger review activity"
projectId: ${project.id}
enabled: true
eventKind: task_created                 # task_created | task_status_changed | activity_completed | manual
eventFilter:
  taskTemplateRef: invoice-review       # JSONata predicate, event-kind specific
activityPlanRef: "activity-plan://${orgSlug}/invoice-review-flow"
triggerMetadata: {}                     # Event-kind-specific config (e.g. cron in v2)
metadata:
  description: "Run review activity whenever an invoice-review task is created"
```

**Event kinds (v1):**

| `eventKind` | `eventFilter` shape | Fires when |
|---|---|---|
| `task_created` | `{ taskTemplateRef? }` | A task is created (optionally filtered to a template slug) |
| `task_status_changed` | `{ taskTemplateRef?, fromStatusSlug?, toStatusSlug? }` | A task transitions status |
| `activity_completed` | `{ parentActivityPlanRef? }` | A parent activity completes |
| `manual` | `{}` | Only via explicit API call to start the trigger |

> Future (v2): `schedule`, `document_arrived`, `data_extracted`. Not yet implemented.

### Knowledge Sets

```yaml
knowledgeSets:
  - slug: routing-rules
    name: "Document Routing"
    description: "Route documents based on features"
    setType: extraction
    active: true
    showOnNewProject: false
    features:
      - slug: is-high-value
        featureTypeRef: "kodexa/numeric-feature"
        properties:
          field: "total_amount"
          operator: ">"
          threshold: 10000
    clauses:
      - features:
          - { featureUuid: "uuid-1", positive: true }
    knowledgeItems: []                  # Items pre-attached to the project
    featureExpression: { ... }          # Optional KnowledgeExpression
```

See the **knowledge-system** skill for full schemas.

### Scheduled Jobs

```yaml
scheduledJobs:
  - name: "Daily Report"
    description: "Generate processing report"
    modelRef: kodexa/report-generator
    active: true
    deleted: false
    properties:
      report_format: pdf
    schedules:
      - { type: cron, cronExpression: "0 0 8 * * *" }
```

### Tags

```yaml
tags:
  - { label: "Production", color: "#10B981" }
  - { label: "PII",        color: "#EF4444" }
```

### Memory

`memory` seeds the project's user-state defaults — recent filter sets, query history, dashboard ordering. Usually empty in a freshly-authored template.

```yaml
memory:
  recentFilters: {}
  recentQueries: {}
  orderedDashboards: []
```

## Project Options

`options` is a `ProjectOptions` blob: user-facing options shown in the project's settings UI plus runtime configuration.

```yaml
options:
  options:                              # Project-level options exposed in UI
    - name: use_ocr
      type: boolean
      label: "Enable OCR"
      description: "Use OCR for scanned documents"
      default: true
      hint: "Disable for digital PDFs"
    - name: confidence
      type: number
      label: "Confidence Threshold"
      default: 0.85
    - name: mode
      type: select
      label: "Processing Mode"
      default: "auto"
      possibleValues:
        - { value: auto,   label: "Automatic" }
        - { value: manual, label: "Manual Review" }
    - name: advanced_option
      type: string
      label: "Advanced Setting"
      showIf: "mode == 'manual'"
      developerOnly: true

  dataOptions: []                       # Data-extraction-related options
  properties: {}                        # Free-form properties
  dataProperties: {}                    # Free-form data properties
  groupTaxonTypeFeatures: {}            # Per-taxon-type-feature group config
  taxonTypeFeatures: {}                 # Per-taxon-type-feature config

  taskOptions:
    showTakeNext: true
    showNewTask: true

  executionPolicy: {}                   # session.ExecutionPolicy

  companion:                            # Project-level workspace companion baseline
    agentRuntimeRef: "${orgSlug}/companion"
    moduleRefs: []
    prompt: "Help users complete project tasks."
```

Each entry in `options:` and `dataOptions:` follows the same `Option` shape as in task-templates (see **task-template** skill).

## Quick Reference — What Lives Where

| Concern | Where to author |
|---|---|
| Stores, assistants, taxonomies, data forms | Inline in project-template |
| Document/attribute status workflows | Inline `documentStatuses` / `attributeStatuses` |
| Task statuses | Inline `taskStatuses` (project) **or** org-level `task-status` resources, bound via project resources |
| Task shape (fields/forms/actions) | Inline `taskTemplates` **or** org-level `task-template` resources |
| Multi-step orchestration (extract → review → approve) | Org-level **`activity-plan`** resource — *not* inline `planTemplate` |
| Event-driven activity launches | Project-scoped **`trigger`** resources |
| Cron-driven module runs | `scheduledJobs:` inline |
| Project-level UI options | `options.options[]` |
| Project workspace companion | `options.companion` |

## Common Mistakes

| Mistake | Fix |
|---|---|
| Authoring `planned: true` / `planTemplate:` under inline `taskTemplates` | Drop those fields. Define an `activity-plan` and a `trigger` instead. |
| Hardcoded store slugs | Use `${project.id}` prefix for uniqueness across projects |
| Missing store refs in assistant connections | Refs must be `${orgSlug}/${project.id}-slug` |
| No UNRESOLVED/RESOLVED status types | First doc status should be UNRESOLVED, terminal should be RESOLVED |
| Mixing OPEN/CLOSED with task statusType | Task statusType is **OPEN, IN_PROGRESS, DONE, BLOCKED** (not RESOLVED/UNRESOLVED) |
| Workspace missing key panels | Include `documentStores`, `dataForms`, `exceptions` at minimum |
| Assistant without connections | Need at least one connection to trigger processing |
| Embedding triggers in project-template | Triggers are separate project-scoped YAML resources |
| Missing `type: projectTemplate` | Required for the platform to recognize this as a template |
| Using inline `taskStatuses` and trying to reference them across projects | Use org-level `task-status` resources for cross-project reuse |
