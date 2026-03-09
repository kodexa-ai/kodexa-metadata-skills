---
name: task-template
description: "Use when creating or editing Kodexa task templates — YAML definitions for reusable task configurations including fields, assignment rules, document groups, actions, planned activities, and custom options"
---

# Kodexa Task Template Authoring

## Overview

Task templates define reusable task configurations in Kodexa projects. They specify the fields, assignment rules, actions, document groups, and optional automated planning for tasks that are created during document processing workflows.

## When to Use

- Creating a new task template for review/approval workflows
- Adding custom fields to tasks
- Configuring assignment rules (auto-assign, role-based)
- Setting up document groups for task-based document management
- Adding action buttons with status transitions
- Configuring automated plan generation

## Interactive Wizard

1. **Task purpose** — What is this task for? (document review, approval, data correction, classification)
2. **Fields** — What information should the task capture? (notes, approval status, priority)
3. **Assignment** — How are tasks assigned? (auto-assign by role, manual, round-robin)
4. **Documents** — How should documents be organized? (groups, filters, upload rules)
5. **Actions** — What actions can users take? (approve, reject, escalate)
6. **Planning** — Should tasks auto-generate plans?

## Task Template Structure

```yaml
taskTemplates:
  - name: "Review Task"                # Required: display name
    slug: review-task                   # Required: unique identifier
    description: "Task purpose"         # Optional description
    metadata:                           # Task configuration
      fields: []                        # Custom fields
      assignmentRules: {}               # Assignment configuration
      documentGroups: []                # Document organization
      actions: []                       # Action buttons
      options: []                       # Custom options
      planned: false                    # Enable auto-planning
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

## Document Groups

```yaml
documentGroups:
  - name: "Source Documents"
    documentFamilyFilter: "*.pdf OR *.xlsx"   # File filter
    maxHits: 5                         # Max documents to show
    maxPages: 100                      # Max pages per document
    maxSize: 10485760                  # Max file size (bytes, 10MB)
    automaticallyAdd: true             # Auto-add matching documents
    editable: true                     # Allow adding/removing docs
    uploadOnly: false                  # Only allow uploads (no selection)
    uniqueFilenames: true              # Enforce unique filenames
    titlePrompt: |                     # AI prompt for task title generation
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

## Planned Activities

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
| Missing `metadata` wrapper | Fields, actions, etc. must be inside `metadata` |
| Action without `targetStatus` | Each action needs a status to transition to |
| `options` vs `fields` confusion | `fields` are task data; `options` are configuration settings |
| Document group filter too broad | Use specific patterns: `*.pdf` not `*` |
| Plan steps without `order` | Always number steps for correct sequencing |
| Missing `slug` on template | Required for API references and CLI operations |
