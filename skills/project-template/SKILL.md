---
name: project-template
description: "Use when creating or editing Kodexa project templates — YAML blueprints that define complete project configurations including stores, assistants, taxonomies, data forms, workspaces, knowledge sets, task templates, status workflows, and scheduled jobs"
---

# Kodexa Project Template Authoring

## Overview

Project templates are complete blueprints for Kodexa projects. They define every component — stores, assistants, taxonomies, data forms, workspaces, knowledge sets, task templates, scheduled jobs, and status workflows — in a single YAML file. When a user creates a project from a template, all components are provisioned automatically.

## When to Use

- Creating a new project template from scratch
- Adding components to an existing template
- Setting up status workflows (document, task, attribute)
- Configuring workspace layouts and panels
- Defining project options (user-configurable settings)
- Wiring up assistant connections and event subscriptions

## Interactive Wizard

1. **Use case** — What does this project do? (document processing, data extraction, classification, review workflow, custom)
2. **Document types** — What documents are processed? What volume?
3. **Stores** — What stores are needed? (intake, processed, exceptions, training, archive)
4. **Processing pipeline** — Which modules/assistants process documents? In what order?
5. **Data capture** — What data forms are needed for manual entry/review?
6. **Workspace** — What panels should the workspace show?
7. **Automation** — Any scheduled jobs? Knowledge-based routing?

Generate the complete project template YAML with all cross-references wired correctly.

## Top-Level Structure

```yaml
slug: my-template                     # Required: unique identifier
orgSlug: my-org                       # Required: organization slug
version: "1.0.0"                      # Semantic version
name: "My Project Template"           # Required: display name
type: projectTemplate                 # Required: must be "projectTemplate"
description: "Template purpose"       # Description

# Optional metadata
publicAccess: false                   # Public availability
deleteProtection: true                # Prevent accidental deletion
icon: "file-invoice"                  # Icon identifier
helpUrl: "https://docs.example.com"   # Help documentation URL
overviewMarkdown: |                   # Markdown overview
  Description shown in template selector.

# Component collections
stores: []                            # Document and data stores
assistants: []                        # Processing assistants
taxonomies: []                        # Data definitions / taxonomies
dataForms: []                         # UI data forms
workspaces: []                        # Workspace configurations
taskTemplates: []                     # Reusable task definitions
scheduledJobs: []                     # Cron-based automation
knowledgeSets: []                     # Knowledge-based routing

# Status workflows
documentStatuses: []                  # Document lifecycle states
taskStatuses: []                      # Task workflow states
attributeStatuses: []                 # Data attribute states

# Configuration
options: {}                           # User-configurable settings
```

## Template Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `${project.id}` | Unique project ID | `abc-123-def-456` |
| `${project.name}` | Project display name | `Invoice Processing` |
| `${project.slug}` | Project slug | `invoice-processing` |
| `${orgSlug}` | Organization slug | `acme-corp` |

Use these in slugs and references to ensure uniqueness across projects.

## Component Schemas

### Stores

```yaml
stores:
  - slug: "${project.id}-documents"   # Unique slug (use template vars)
    name: "Documents"                  # Display name
    storeType: DOCUMENT                # DOCUMENT or DATA
    storePurpose: OPERATIONAL          # OPERATIONAL, TRAINING
    description: "Purpose"             # Optional description
    deleteProtection: true             # Prevent deletion
    highQualityPreview: true           # High-quality doc previews
    showThumbnails: true               # Show thumbnails in list
    showStoreInLabeling: true          # Show in labeling UI
    allowDataEditing: false            # Allow in-place editing
    documentProperties:                # Metadata columns
      - name: vendor_name
        type: string
        label: "Vendor Name"
    labelExpressions:                  # Auto-labels
      - label: "processed"
        expression: "status == 'completed'"
```

### Assistants

```yaml
assistants:
  - name: "Document Processor"
    slug: doc-processor
    description: "Processes incoming documents"
    assistantDefinitionRef: kodexa/pdf-extractor  # Module reference
    priorityHint: 10                   # Execution priority (higher = more priority)
    loggingEnabled: true               # Detailed execution logging
    chatEnabled: false                 # Enable chat interface
    connections:                       # Event subscriptions
      - sourceType: STORE              # STORE, CHANNEL, DOCUMENT_FAMILY
        sourceRef: "${orgSlug}/${project.id}-intake"
        subscription: "!hasMixins('processed')"  # Filter expression
    stores:                            # Store access list
      - "${orgSlug}/${project.id}-intake"
      - "${orgSlug}/${project.id}-output"
    options:                           # Module-specific options
      confidence_threshold: 0.85
    schedules:                         # Optional cron schedules
      - cronExpression: "0 0 8 * * *"
```

### Taxonomies

```yaml
taxonomies:
  - slug: categories
    name: "Document Categories"
    taxonomyType: CONTENT              # CONTENT, TASK, DOCUMENT
    taxons:
      - name: Category A
        children:
          - name: Sub-Category 1
          - name: Sub-Category 2
      - name: Category B

  # Or reference an existing taxonomy
  - ref: "kodexa/standard-taxonomy"
```

### Data Forms

```yaml
dataForms:
  - slug: metadata-form
    name: "Metadata Entry Form"
    description: "Manual data entry form"
    cards:                             # V1 card-based definition
      - type: form
        fields:
          - name: field_name
            type: text                 # text, date, number, select, textarea
            label: "Field Label"
            required: true
```

### Workspaces

```yaml
workspaces:
  - name: "Main Workspace"
    slug: main-workspace
    description: "Primary processing workspace"
    workspaceStorage:
      availablePanels:
        documentStores: true           # Document stores panel
        dataForms: true                # Data forms panel
        taxonomies: true               # Taxonomy panel
        assistants: true               # Assistants panel
        exceptions: true               # Exceptions panel
        properties: true               # Properties panel
        channels: true                 # Chat/channels panel
        auditEvents: true              # Audit trail panel
        navigation: true               # Navigation panel
      overview: |                      # Markdown overview
        # Workspace Guide
        Instructions for users...
```

### Status Configurations

```yaml
documentStatuses:
  - status: "New"
    slug: new
    color: "#6B7280"                   # Gray
    statusType: UNRESOLVED
  - status: "Processing"
    slug: processing
    color: "#3B82F6"                   # Blue
  - status: "Complete"
    slug: complete
    color: "#10B981"                   # Green
    statusType: RESOLVED

taskStatuses:
  - label: "To Do"
    slug: todo
    color: "#6B7280"
    statusType: TODO
  - label: "In Progress"
    slug: in-progress
    color: "#3B82F6"
    statusType: IN_PROGRESS
  - label: "Done"
    slug: done
    color: "#10B981"
    statusType: DONE

attributeStatuses:
  - status: "Pending"
    slug: pending
    color: "#F59E0B"
    statusType: UNRESOLVED
  - status: "Validated"
    slug: validated
    color: "#10B981"
    statusType: RESOLVED
```

### Task Templates

```yaml
taskTemplates:
  - name: "Review Task"
    slug: review-task
    description: "Manual review task"
    metadata:
      fields:
        - name: approval_status
          type: select
          label: "Approval"
          options:
            - value: approved
              label: "Approved"
            - value: rejected
              label: "Rejected"
      assignmentRules:
        autoAssign: true
        assignToRole: reviewer
```

### Knowledge Sets

```yaml
knowledgeSets:
  - slug: routing-rules
    name: "Document Routing"
    description: "Route documents based on features"
    setType: extraction
    active: true
    features:
      - slug: is-high-value
        featureTypeRef: "kodexa/numeric-feature"
        properties:
          field: "total_amount"
          operator: ">"
          threshold: 10000
    clauses:
      - features:
          - featureUuid: "uuid-1"
            positive: true
```

### Scheduled Jobs

```yaml
scheduledJobs:
  - name: "Daily Report"
    description: "Generate processing report"
    modelRef: kodexa/report-generator  # Module reference
    active: true
    schedules:
      - cronExpression: "0 0 8 * * *"  # 8 AM daily
    properties:
      report_format: pdf
```

## Project Options

```yaml
options:
  options:                             # User-facing settings
    - name: use_ocr
      type: boolean                    # boolean, string, number, select
      label: "Enable OCR"
      description: "Use OCR for scanned documents"
      default: true
      hint: "Disable for digital PDFs"

    - name: confidence
      type: number
      label: "Confidence Threshold"
      default: 0.85
      min: 0
      max: 1

    - name: mode
      type: select
      label: "Processing Mode"
      default: "auto"
      possibleValues:
        - value: auto
          label: "Automatic"
        - value: manual
          label: "Manual Review"

    - name: advanced_option
      type: string
      label: "Advanced Setting"
      showIf: "mode == 'manual'"       # Conditional visibility
      developerOnly: true              # Only for developers

  taskOptions:
    showTakeNext: true                 # Show "Take Next" button
    showNewTask: true                  # Show "New Task" button
```

## Complete Example

```yaml
slug: invoice-processing
orgSlug: acme-corp
version: "1.0.0"
name: Invoice Processing System
type: projectTemplate
description: Complete invoice processing with extraction, validation, and approval
deleteProtection: true

stores:
  - slug: "${project.id}-intake"
    name: "Invoice Intake"
    storeType: DOCUMENT
    storePurpose: OPERATIONAL
    deleteProtection: true
    highQualityPreview: true
    showThumbnails: true

  - slug: "${project.id}-processed"
    name: "Processed Invoices"
    storeType: DOCUMENT
    storePurpose: OPERATIONAL

  - slug: "${project.id}-exceptions"
    name: "Exception Queue"
    storeType: DOCUMENT
    storePurpose: OPERATIONAL

documentStatuses:
  - status: "New"
    slug: new
    color: "#6B7280"
    statusType: UNRESOLVED
  - status: "Processing"
    slug: processing
    color: "#3B82F6"
  - status: "Extracted"
    slug: extracted
    color: "#8B5CF6"
  - status: "Validated"
    slug: validated
    color: "#10B981"
  - status: "Approved"
    slug: approved
    color: "#059669"
    statusType: RESOLVED
  - status: "Exception"
    slug: exception
    color: "#EF4444"

taskStatuses:
  - label: "To Review"
    slug: to-review
    color: "#6B7280"
    statusType: TODO
  - label: "In Review"
    slug: in-review
    color: "#3B82F6"
    statusType: IN_PROGRESS
  - label: "Completed"
    slug: completed
    color: "#10B981"
    statusType: DONE

assistants:
  - name: Invoice Extractor
    slug: invoice-extractor
    assistantDefinitionRef: kodexa/pdf-invoice-extractor
    priorityHint: 10
    loggingEnabled: true
    connections:
      - sourceType: STORE
        sourceRef: "${orgSlug}/${project.id}-intake"
        subscription: "!hasMixins('processed')"
    stores:
      - "${orgSlug}/${project.id}-intake"
      - "${orgSlug}/${project.id}-processed"
      - "${orgSlug}/${project.id}-exceptions"
    options:
      use_ocr: true
      confidence_threshold: 0.85

taxonomies:
  - slug: invoice-categories
    name: Invoice Categories
    taxonomyType: CONTENT
    taxons:
      - name: Professional Services
        children:
          - name: Consulting
          - name: Legal
      - name: Goods
        children:
          - name: Office Supplies
          - name: Equipment

dataForms:
  - slug: invoice-review
    name: Invoice Review Form
    cards:
      - type: form
        fields:
          - name: invoice_number
            type: text
            label: "Invoice Number"
            required: true
          - name: total_amount
            type: number
            label: "Total Amount"
            required: true

workspaces:
  - name: Invoice Processing
    slug: invoice-workspace
    workspaceStorage:
      availablePanels:
        documentStores: true
        dataForms: true
        taxonomies: true
        assistants: true
        exceptions: true
        auditEvents: true
      overview: |
        # Invoice Processing
        1. Upload invoices to **Invoice Intake**
        2. Automatic extraction runs via **Invoice Extractor**
        3. Review results in **Processed Invoices**
        4. Handle exceptions in **Exception Queue**

taskTemplates:
  - name: Invoice Approval
    slug: invoice-approval
    metadata:
      fields:
        - name: approval_status
          type: select
          label: "Approval"
          options:
            - value: approved
              label: "Approved"
            - value: rejected
              label: "Rejected"

scheduledJobs:
  - name: Daily Report
    modelRef: kodexa/report-generator
    active: true
    schedules:
      - cronExpression: "0 0 8 * * *"
    properties:
      report_format: pdf

options:
  options:
    - name: auto_validation
      type: boolean
      label: "Auto Validation"
      default: true
    - name: confidence_threshold
      type: number
      label: "Confidence Threshold"
      default: 0.85
  taskOptions:
    showTakeNext: true
    showNewTask: true
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Hardcoded store slugs | Use `${project.id}` prefix for uniqueness |
| Missing store refs in assistant connections | Store refs must be `${orgSlug}/${project.id}-slug` format |
| No UNRESOLVED/RESOLVED status types | First status should be UNRESOLVED, final should be RESOLVED |
| Workspace missing key panels | Include at minimum: documentStores, dataForms, exceptions |
| Assistant without connections | At least one connection is needed to trigger processing |
| Missing `type: projectTemplate` | Required for the platform to recognize this as a template |
