---
name: task-template
description: "Use when creating or editing Kodexa task templates — YAML definitions for reusable task configurations including fields, assignment rules, document groups, actions, planned activities, and custom options"
---

# Kodexa Task Template Authoring

## Overview

Task templates are **project-scoped** blueprints for creating tasks. They pre-configure how work is structured: data forms, action buttons, document groups, custom options, AI chat assistance, AI-generated task titles, and automated multi-step workflows. A project can have zero or more task templates — they are created and managed within the project, either via the Studio UI or deployed through project templates via CLI.

Reference: https://developer.kodexa.ai/studio/project/task-templates

## When to Use

- Creating a new task template for review/approval workflows
- Configuring data forms with form-specific actions
- Adding custom fields and options to tasks
- Configuring assignment rules (auto-assign, role-based, team)
- Setting up document groups for task-based document management
- Adding action buttons with status transitions and keyboard shortcuts
- Configuring AI chat prompts for automatic reviewer guidance
- Configuring AI task naming for descriptive auto-generated titles
- Configuring planned sub-task workflows

## Interactive Wizard

1. **Task purpose** — What is this task for? (document review, approval, data correction, classification)
2. **Fields** — What information should the task capture? (notes, approval status, priority)
3. **Assignment** — How are tasks assigned? (auto-assign by role, manual, team)
4. **Documents** — How should documents be organized? (groups, filters, upload rules)
5. **Actions** — What actions can users take? (approve, reject, escalate)
6. **Forms** — What data forms should be attached? (with form-specific actions)
7. **Chat Prompt** — Should AI provide guidance when a task opens?
8. **AI Naming** — Should task titles be auto-generated from document content?
9. **Planning** — Should tasks auto-generate multi-step plans?

## Editor Tabs

The task template editor in Studio organizes configuration across eight tabs:

| Tab | Function |
|-----|----------|
| **Details** | Name, slug, description, AI naming config, sub-task settings |
| **Forms** | Attach data forms with form-specific actions and panel visibility |
| **Options** | Custom fields, visibility controls, LLM settings |
| **Actions** | Toolbar buttons with status transitions and keyboard shortcuts |
| **Chat Prompt** | Automatic AI assistance when a task is opened |
| **Documents** | Document groups with filtering and file management rules |
| **Planned** | Visual workflow editor (replaces Forms/Actions/Chat when enabled) |
| **YAML** | Complete configuration in raw format |

**Note:** When "Enable Planned Sub-Tasks" is activated on the Details tab, the Forms, Actions, and Chat Prompt tabs are replaced by the Planned tab.

## Task Template Structure

```yaml
taskTemplates:
  - name: "Review Task"                # Required: display name
    slug: review-task                   # Required: unique identifier
    description: "Task purpose"         # Optional description
    teamSlug: data-review-team          # Default team assignment
    metadata:                           # Task configuration
      forms: []                         # Attached data forms with actions
      fields: []                        # Custom fields
      assignmentRules: {}               # Assignment configuration
      documentFamilyGroups: []          # Document organization
      actions: []                       # Action buttons
      options: []                       # Custom options
      priority: 5                       # Default priority level
      properties:                       # Template properties
        defaultToDataForm: true         # Open data form view by default
        hideDefaultButtons: true        # Hide built-in task buttons
        subTaskOnly: false              # Hide from main task creation
      chatPrompt: {}                    # AI chat prompt configuration
      aiNaming: {}                      # AI task naming configuration
      planned: false                    # Enable planned sub-tasks
      planTemplate: {}                  # Plan template structure
```

## Custom Fields

```yaml
fields:
  - name: approval_status
    type: select                       # text, textarea, number, date, select, boolean
    label: "Approval Status"
    required: true
    options:
      - value: approved
        label: "Approved"
      - value: rejected
        label: "Rejected"
      - value: escalated
        label: "Escalated to Manager"

  - name: review_notes
    type: textarea
    label: "Review Notes"
    required: false

  - name: priority_score
    type: number
    label: "Priority Score"
    required: false

  - name: due_date
    type: date
    label: "Due Date"
    required: true

  - name: is_urgent
    type: boolean
    label: "Urgent"
    required: false
```

## Assignment Rules

```yaml
assignmentRules:
  autoAssign: true                     # Automatically assign tasks
  assignToRole: reviewer               # Role to assign to
  assignToTeam: review-team            # Team slug
```

## Forms

Forms attach data forms to the task template with form-specific actions and panel visibility:

```yaml
forms:
  - dataFormRef: ${orgSlug}/bill-review:1.0.0  # Data form reference (${orgSlug} resolved at deploy)
    properties: {}                              # Form-specific properties
    availablePanels: {}                         # Panel visibility settings
    actions:                                    # Form-specific action buttons
      - label: "Approve"
        properties:
          statusId: <task-status-uuid>          # Target task status UUID
          automaticallyTakeNextTask: true       # Auto-load next task after action
          onlyEnabledIfNoOpenExceptions: true   # Disable if validation exceptions exist
          color: "#71ae48"                      # Button color (hex)
          icon: check                           # Icon name
          shortcut: "a"                         # Keyboard shortcut
      - label: "Reject"
        properties:
          statusId: <task-status-uuid>
          automaticallyTakeNextTask: true
          color: "#ed7e32"
          icon: cancel
          shortcut: "r"
```

## Document Groups

```yaml
documentFamilyGroups:
  - name: "Source Documents"
    documentFamilyFilter: "*.pdf OR *.xlsx"   # File extension filter
    maxHits: 5                         # Max documents to show
    maxPages: 100                      # Max pages per document
    maxSize: 10485760                  # Max file size (bytes, 10MB)
    hardMaxPages: 200                  # Hard max — documents exceeding this are rejected
    automaticallyAdd: true             # Auto-populate from project store
    editable: true                     # Allow adding/removing docs
    uploadOnly: false                  # Only allow uploads (no selection)
    uniqueFilenames: true              # Enforce unique filenames
    titlePrompt: |                     # AI prompt for task title from document content
      Generate a task title from the document content

  - name: "Supporting Evidence"
    documentFamilyFilter: "*.pdf"
    automaticallyAdd: false
    editable: true
    uploadOnly: true
```

## Actions

```yaml
actions:
  - name: approve
    label: "Approve"
    targetStatus: approved             # Status to transition to
    icon: check                        # Icon name
    color: green                       # Button color
    shortcut: "a"                      # Keyboard shortcut
    setAttributes:                     # Attributes to set on action
      approval_status: approved

  - name: reject
    label: "Reject"
    targetStatus: rejected
    icon: close
    color: red
    shortcut: "r"
    setAttributes:
      approval_status: rejected

  - name: escalate
    label: "Escalate"
    targetStatus: escalated
    icon: arrow-up
    color: orange
```

## Custom Options

```yaml
options:
  - tabName: "Settings"
    name: extractionMode
    label: "Extraction Mode"
    type: select
    required: true
    default: "auto"
    description: "How documents should be processed"
    showIf: "advancedMode == true"     # Conditional visibility
    developerOnly: false               # Developer-only flag
    showOnPopup: true                  # Show in creation popup
    possibleValues:
      - value: auto
        label: "Automatic"
      - value: manual
        label: "Manual"

  - name: notifyOnComplete
    label: "Notify on Completion"
    type: boolean
    default: true
```

## Team Assignment

```yaml
teamSlug: data-review-team            # Assign template to team
```

## AI Chat Prompt

Automatic chat prompts fire when a reviewer opens a task, providing immediate AI guidance:

```yaml
chatPrompt:
  enabled: true
  prompt: |
    You are assisting with bill verification for {templateName}.
    The task contains {documentFamilyCount} document(s): {documentFamilyPaths}.
    Please review the extracted data and:
    1. Summarize the key invoice details
    2. Flag any data quality issues or exceptions
    3. Suggest whether to approve or reject
```

When activated, the system checks for an existing chat session, opens the chat panel, creates a task-scoped channel, substitutes template variables, and sends to the project AI assistant.

### Template Variables

| Variable | Description |
|----------|-------------|
| `{taskTitle}` | The task's title |
| `{taskDescription}` | The task's description |
| `{assignee}` | Assigned user display name |
| `{templateName}` | Task template name |
| `{documentFamilyPaths}` | Comma-separated document file paths |
| `{knowledgeFeatures}` | Comma-separated knowledge feature names |
| `{documentFamilyCount}` | Number of attached documents |

## AI Task Naming

Auto-generates descriptive task titles from document content instead of generic "Task 12" naming:

```yaml
aiNaming:
  enabled: true
  prompt: |
    Generate a concise task title for this {templateName} task.
    Documents: {documentFamilyPaths}
    Include the vendor name and invoice number.
```

Uses the same template variables as chat prompts. Produces titles like "Acme Corp Q4 2025 Invoice #4821".

## Sub-Task Templates

Templates marked as sub-task-only are hidden from the main task creation dialog:

```yaml
properties:
  subTaskOnly: true                    # Hidden from main task creation
```

These activate exclusively when Planning Mode plans create child tasks.

## Planned Activities

When enabled, the Forms, Actions, and Chat Prompt tabs are replaced by the Planned tab with a visual workflow editor. In Planning Mode, subsequent steps can depend on which action the reviewer selected.

```yaml
planned: true                          # Enable auto-planning
planTemplate:
  steps:
    - name: "Initial Review"
      description: "Review extracted data for accuracy"
      order: 1
    - name: "Validation"
      description: "Validate against business rules"
      order: 2
    - name: "Final Approval"
      description: "Manager approval for high-value items"
      order: 3
      conditional: true
      condition: "total_amount > 10000"
```

## Complete Example

```yaml
taskTemplates:
  - name: Invoice Review
    slug: invoice-review
    description: "Review and approve extracted invoice data"
    metadata:
      fields:
        - name: approval_status
          type: select
          label: "Approval Status"
          required: true
          options:
            - value: approved
              label: "Approved"
            - value: rejected
              label: "Rejected for Correction"
            - value: escalated
              label: "Escalate to Manager"

        - name: review_notes
          type: textarea
          label: "Review Notes"
          required: false

        - name: confidence_override
          type: number
          label: "Manual Confidence Score"
          required: false

      assignmentRules:
        autoAssign: true
        assignToRole: reviewer

      documentGroups:
        - name: "Invoice Documents"
          documentFamilyFilter: "*.pdf"
          maxHits: 3
          automaticallyAdd: true
          editable: false
          titlePrompt: "Generate a review task title from the invoice"

      actions:
        - name: approve
          label: "Approve"
          targetStatus: approved
          icon: check
          color: green
          shortcut: "a"
          setAttributes:
            approval_status: approved

        - name: reject
          label: "Reject"
          targetStatus: rejected
          icon: close
          color: red
          shortcut: "r"

        - name: escalate
          label: "Escalate"
          targetStatus: escalated
          icon: arrow-up
          color: orange

      options:
        - name: auto_approve_threshold
          label: "Auto-Approve Threshold"
          type: number
          default: 0.95
          description: "Auto-approve above this confidence"

      planned: true
      planTemplate:
        steps:
          - name: "Data Verification"
            description: "Verify extracted fields match document"
            order: 1
          - name: "Validation Check"
            description: "Run business rule validation"
            order: 2
          - name: "Approval Decision"
            description: "Approve or reject the invoice"
            order: 3

  - name: Exception Resolution
    slug: exception-resolution
    description: "Resolve processing exceptions"
    metadata:
      fields:
        - name: resolution_type
          type: select
          label: "Resolution"
          required: true
          options:
            - value: corrected
              label: "Data Corrected"
            - value: reprocessed
              label: "Reprocessed"
            - value: rejected
              label: "Document Rejected"
        - name: resolution_notes
          type: textarea
          label: "Resolution Notes"

      assignmentRules:
        autoAssign: true
        assignToRole: data_specialist

      documentGroups:
        - name: "Exception Documents"
          documentFamilyFilter: "*"
          automaticallyAdd: true

      actions:
        - name: resolve
          label: "Mark Resolved"
          targetStatus: resolved
          icon: check-circle
          color: green
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Missing `metadata` wrapper | Fields, actions, forms, etc. must be inside `metadata` |
| Action without `statusId` | Each form action needs a `statusId` referencing a task status UUID |
| Using `targetStatus` instead of `statusId` | Form actions use `properties.statusId`, not `targetStatus` |
| `options` vs `fields` confusion | `fields` are task data; `options` are configuration settings |
| `documentGroups` vs `documentFamilyGroups` | The correct key is `documentFamilyGroups` |
| Document group filter too broad | Use specific patterns: `*.pdf` not `*` |
| Plan steps without `order` | Always number steps for correct sequencing |
| Missing `slug` on template | Required for API references and CLI operations |
| Enabling planned mode with form actions | When `planned: true`, Forms/Actions/Chat tabs are replaced by the Planned tab |
| Chat prompt variables without braces | Use `{taskTitle}` not `taskTitle` for template variables |
| Referencing non-existent task statuses | Verify `statusId` UUIDs exist in the project's task statuses before using them |
