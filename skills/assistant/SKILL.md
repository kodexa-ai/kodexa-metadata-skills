---
name: assistant
description: "Use when creating or editing Kodexa assistants — YAML configuration for event-driven processing pipelines including step definitions, connections to stores, event subscriptions, conditionals, and module orchestration"
---

# Kodexa Assistant Authoring

## Overview

Assistants are event-driven processing pipelines in Kodexa. They connect to stores and channels, subscribe to events (new documents, status changes), and execute modules in sequence to process documents. Assistants can be reactive (triggered by events), schedulable (run on cron), or both.

## When to Use

- Creating a new assistant for document processing
- Configuring event subscriptions and store connections
- Setting up multi-step processing pipelines
- Adding conditional logic to processing flows
- Debugging assistant event handling or execution issues

## Interactive Wizard

1. **Purpose** — What does the assistant do? (extraction, classification, validation, transformation, routing)
2. **Trigger** — What triggers it? (new document uploaded, status change, schedule, manual)
3. **Stores** — Which stores does it read from / write to?
4. **Steps** — What processing steps? (which modules, in what order)
5. **Conditions** — Any conditional logic? (skip if already processed, route by type)
6. **Error handling** — What happens on failure? (retry, move to exceptions, notify)

Generate the assistant YAML configuration.

## Extension Pack Assistant Definition

```yaml
name: "My Assistant"
slug: my-assistant
description: "What this assistant does"
type: assistant
reactive: true                         # Triggered by events
schedulable: false                     # Can run on schedule
publicAccess: true                     # Available to all orgs
template: true                         # Is a template

A:
  package: my_package                  # Python package name
  class: MyAssistant                   # Python class name

options:                               # User-configurable options
  - name: confidence_threshold
    label: "Confidence Threshold"
    type: number
    default: 0.85
    required: false
    description: "Minimum confidence for auto-processing"
```

## Project-Level Assistant Configuration

When used in a project or project template:

```yaml
assistants:
  - name: "Document Processor"
    slug: doc-processor
    description: "Processes incoming documents"

    # Module reference
    assistantDefinitionRef: kodexa/pdf-extractor

    # Execution settings
    priorityHint: 10                   # Higher = more priority
    loggingEnabled: true               # Detailed execution logs
    chatEnabled: false                 # Enable chat interface
    assistantRole: "extractor"         # Role identifier

    # Store access
    stores:
      - "${orgSlug}/${project.id}-intake"
      - "${orgSlug}/${project.id}-output"

    # Event connections
    connections:
      - sourceType: STORE
        sourceRef: "${orgSlug}/${project.id}-intake"
        subscription: "!hasMixins('processed')"

    # Module options
    options:
      use_ocr: true
      confidence_threshold: 0.85

    # Optional scheduling
    schedules:
      - cronExpression: "0 0 8 * * *"  # 8 AM daily
```

## Connection Types

```yaml
connections:
  # Store connection - triggered when documents arrive or change
  - sourceType: STORE
    sourceRef: "${orgSlug}/${project.id}-documents"
    subscription: "!hasMixins('processed')"

  # Channel connection - triggered by upstream assistant output
  - sourceType: CHANNEL
    targetRef: "${orgSlug}/${project.id}/upstream-assistant"
    subscription: "status == 'completed'"

  # Document family connection
  - sourceType: DOCUMENT_FAMILY
    sourceRef: "${orgSlug}/${project.id}-store"

  # Workspace connection
  - sourceType: WORKSPACE
    sourceRef: "${orgSlug}/${project.id}/workspace-slug"
```

### ConnectableType Values

| Type | Trigger | Use Case |
|------|---------|----------|
| `STORE` | Document added/changed in store | Primary input processing |
| `CHANNEL` | Upstream assistant completes | Pipeline chaining |
| `DOCUMENT_FAMILY` | Document family events | Fine-grained document processing |
| `WORKSPACE` | Workspace events | User-initiated processing |

## Subscription Expressions

Filter which events trigger the assistant:

```yaml
# Process only unprocessed documents
subscription: "!hasMixins('processed')"

# Process only PDFs
subscription: "contentType == 'application/pdf'"

# Process documents with specific status
subscription: "status == 'new'"

# Combine conditions
subscription: "!hasMixins('processed') AND contentType == 'application/pdf'"

# Process completed upstream results
subscription: "status == 'completed'"
```

## Multi-Step Pipelines

Chain assistants together using channels:

```yaml
assistants:
  # Step 1: OCR/Parsing
  - name: Document Parser
    slug: parser
    assistantDefinitionRef: kodexa/pdf-parser
    connections:
      - sourceType: STORE
        sourceRef: "${orgSlug}/${project.id}-intake"
        subscription: "!hasMixins('parsed')"

  # Step 2: Extraction (triggered by parser completion)
  - name: Data Extractor
    slug: extractor
    assistantDefinitionRef: kodexa/data-extractor
    connections:
      - sourceType: CHANNEL
        targetRef: "${orgSlug}/${project.id}/parser"
        subscription: "status == 'completed'"

  # Step 3: Validation (triggered by extractor completion)
  - name: Data Validator
    slug: validator
    assistantDefinitionRef: kodexa/validator
    connections:
      - sourceType: CHANNEL
        targetRef: "${orgSlug}/${project.id}/extractor"
        subscription: "status == 'completed'"
```

## Extension Pack Structure

For creating assistant packages:

```yaml
name: "My Extension Pack"
slug: my-extension-pack
type: extensionPack
description: "Collection of processing assistants"
source:
  type: docker
  location: kodexa://extension-core:{version}
services:
  - slug: my-assistant
    name: "My Assistant"
    type: assistant
    assistant:
      package: my_package
      class: MyAssistant
    schedulable: true
    reactive: true
    options:
      - name: option_name
        label: "Option Label"
        type: string
        required: true
```

## Python Assistant Implementation

```python
from kodexa import Assistant, PipelineContext

class MyAssistant(Assistant):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.confidence = kwargs.get('confidence_threshold', 0.85)

    def process(self, document, context: PipelineContext):
        """Main processing method."""
        # Process the document
        context.log.info(f"Processing {document.uuid}")

        # Access options
        use_ocr = context.get_option('use_ocr', True)

        # Update status
        context.status_reporter.update("Extracting data", status_type="analyzing")

        return document

    def handle_event(self, event, document, context):
        """Handle platform events (if reactive)."""
        event_type = event.get('type')
        return document
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| No connections defined | At least one connection needed to trigger processing |
| Wrong `sourceRef` format | Must be `orgSlug/storeSlug` or use template vars `${orgSlug}/${project.id}-slug` |
| Channel ref without project context | Channel refs need project: `${orgSlug}/${project.id}/assistant-slug` |
| Missing stores list | Add all stores the assistant reads from or writes to |
| Subscription syntax errors | Use `hasMixins('label')` not `hasMixin('label')` |
| Scheduling without `schedulable: true` | Extension pack definition must set `schedulable: true` |
