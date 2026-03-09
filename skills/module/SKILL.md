---
name: module
description: "Use when creating, configuring, or debugging Kodexa modules — YAML metadata for model-type modules (Python ML/AI processing) and skill-type modules (file-based resources), including runtime configuration, sidecars, inference options, and deployment settings"
---

# Kodexa Module Authoring

## Overview

Modules are the execution units of the Kodexa platform. There are two types: **model modules** (Python code that processes documents) and **skill modules** (file packs like prompts, configs, and knowledge). This skill covers the YAML metadata for both types, plus Python entry point patterns for model modules.

## When to Use

- Creating a new module YAML (model or skill)
- Configuring inference options for user-configurable parameters
- Setting up module runtime references and sidecars
- Writing Python entry points for model modules
- Configuring deployment settings (replicas, memory, containers)
- Debugging module loading or execution issues

## Interactive Wizard

1. **Module type** — Model (Python code) or Skill (file pack)?
2. **Purpose** — What does it do? (extraction, classification, transformation, event handling, chat, reporting)
3. **Runtime** — Which runtime? (base-cloud-model-runtime is default for models)
4. **Sidecars** — Need shared dependencies? (kodexa-llm-model for LLM access, etc.)
5. **Options** — What should users be able to configure? (confidence thresholds, modes, etc.)
6. **Events** — Does it handle events? Need scheduling support?

Generate the module YAML and Python entry point skeleton.

## Module YAML Structure

```yaml
slug: my-module                       # Required: unique identifier
orgSlug: my-org                       # Required: organization slug
name: "My Module"                     # Required: display name
type: module                          # Required: must be "module"
description: "What this module does"  # Description
lifecycle: production                 # production, development, deprecated
version: "1.0.0"                      # Semantic version
sourceUrl: "https://github.com/..."   # Source code URL

metadata:
  type: model                         # "model" or "skill"

  # === Content (what files to include) ===
  contents:                           # File glob patterns to include
    - my_module/**.py
  ignored_contents:                   # Patterns to exclude
    - "**/__pycache__/**"
    - "**/*.pyc"

  # === Runtime (model type only) ===
  moduleRuntimeRef: kodexa/base-cloud-model-runtime  # Runtime reference
  modelRuntimeParameters:             # Runtime-specific params
    module: my_module                 # Python module name
    function: infer                   # Entry point function (default: infer)

  # === Sidecars ===
  modelSidecars:                      # Additional modules loaded before execution
    - kodexa/kodexa-llm-model:1.0.0

  # === Capabilities ===
  eventAware: false                   # Handles events (needs handle_event function)
  inferable: true                     # Can run inference
  trainable: false                    # Supports training
  supportsScheduling: false           # Can be scheduled via cron

  # === Inference Options (user-configurable) ===
  inferenceOptions:
    - name: confidence_threshold
      type: number
      default: 0.85
      description: "Minimum confidence score"
    - name: mode
      type: string
      default: "auto"
      description: "Processing mode"

  # === Provider Info ===
  provider: "My Company"
  providerUrl: "https://example.com"

overviewMarkdown: |                   # Detailed description (markdown)
  ## My Module
  Detailed documentation...
```

## Model vs Skill Comparison

| Aspect | Model | Skill |
|--------|-------|-------|
| `metadata.type` | `model` | `skill` |
| Language | Python | Any (files only) |
| Runtime | Required (`moduleRuntimeRef`) | Not needed |
| Entry point | Python function | N/A |
| Execution | Server-side processing | Downloaded to `/home/kodexa/skills/` |
| Use case | Document processing, ML, AI | Prompts, configs, knowledge files |

## Inference Options Schema

```yaml
inferenceOptions:
  - name: option_name                 # Required: parameter name
    type: string                      # string, number, boolean, select
    default: "value"                  # Default value
    description: "What this controls" # User-facing description
    label: "Display Label"            # Optional display label
    required: false                   # Whether required
    possibleValues:                   # For select type
      - value: "opt1"
        label: "Option 1"
      - value: "opt2"
        label: "Option 2"
```

## Python Entry Points

### Basic Inference Function

```python
import logging
from kodexa_document import Document

logger = logging.getLogger(__name__)

def infer(document: Document, confidence_threshold=0.85, status_reporter=None):
    """Process a document and return it with extracted data."""
    logger.info(f"Processing: {document.uuid}")

    if status_reporter:
        status_reporter.update("Analyzing document", status_type="analyzing")

    # Process the document
    content = document.content_node.get_all_content()

    # Add labels, data objects, etc.
    document.add_label("processed")

    return document
```

### Event Handler

```python
def handle_event(event: dict, document: Document, assistant=None, status_reporter=None):
    """Handle platform events (requires eventAware: true)."""
    event_type = event.get("type")
    logger.info(f"Handling event: {event_type}")

    if status_reporter:
        status_reporter.update("Processing event", status_type="processing")

    return document
```

### Magic Parameter Injection

Parameters are injected by name. Available parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `document` | Document | The document being processed |
| `document_family` | DocumentFamily | The document family |
| `model_base` | str | Path to model's base directory |
| `pipeline_context` | PipelineContext | Pipeline execution context |
| `model_store` | ModelStore | For persisting/loading artifacts |
| `assistant` | Assistant | Associated assistant (for LLM) |
| `assistant_id` | str | The assistant ID |
| `project` | Project | Project this execution belongs to |
| `execution_id` | str | Current execution ID |
| `status_reporter` | StatusReporter | For live status updates |
| `event` | dict | Triggering event (event handlers only) |

### StatusReporter

```python
status_reporter.update(
    title="Processing page 3/10",
    subtitle="Extracting line items",
    status_type="processing"  # thinking, searching, planning, reviewing,
                               # processing, analyzing, writing, waiting
)
```

## Deployment Defaults

```yaml
metadata:
  deploymentDefaults:
    deploymentType: CONTAINER          # CONTAINER, AWS_LAMBDA
    minReplicas: 1
    maxReplicas: 3
    memoryAssigned: "1000"             # MB
    serviceName: my-service
    containerName: my-container
    containerUrl: registry.example.com/image
    childProcess: true
    environment:
      ENV_VAR: value
```

## Sidecar Pattern

Sidecars are additional modules loaded into `sys.path` before your module executes.

```yaml
metadata:
  modelSidecars:
    - kodexa/kodexa-llm-model:1.0.0   # LLM utilities
    - kodexa/kodexa-langchain-module   # LangChain integration
```

Usage in Python:
```python
# Import from sidecar after it's loaded
from kodexa_llm.utils import get_bedrock_client
from kodexa_langchain.utils import create_chain
```

## Skill Module Example

```yaml
slug: my-llm-skills
orgSlug: my-org
name: "LLM Skills Pack"
type: module
description: "Prompts and tools for LLM processing"

metadata:
  type: skill
  contents:
    - prompts/**
    - tools/**
    - config.yml
  ignored_contents:
    - "**/*.pyc"
```

Directory structure:
```
my-llm-skills/
  module.yml
  prompts/
    system.md
    extraction.md
  tools/
    search.yml
  config.yml
```

## Complete Model Example

```yaml
slug: invoice-extractor
orgSlug: acme-corp
name: "Invoice Data Extractor"
type: module
description: "Extracts structured data from invoice PDFs using AI"
lifecycle: production
version: "2.1.0"

metadata:
  type: model
  contents:
    - invoice_extractor/**.py
    - invoice_extractor/templates/**
  ignored_contents:
    - "**/__pycache__/**"
    - "**/*.pyc"
    - "**/tests/**"

  moduleRuntimeRef: kodexa/base-cloud-model-runtime
  modelRuntimeParameters:
    module: invoice_extractor
    function: infer

  modelSidecars:
    - kodexa/kodexa-llm-model:1.0.0

  eventAware: true
  inferable: true
  trainable: false
  supportsScheduling: true

  inferenceOptions:
    - name: confidence_threshold
      type: number
      default: 0.85
      description: "Minimum confidence for extraction"
    - name: use_ocr
      type: boolean
      default: true
      description: "Enable OCR for scanned documents"
    - name: extract_line_items
      type: boolean
      default: true
      description: "Extract individual line items"
    - name: output_format
      type: string
      default: "json"
      description: "Output format for extracted data"
      possibleValues:
        - value: json
          label: "JSON"
        - value: csv
          label: "CSV"

  provider: "Acme Corp"
  providerUrl: "https://acme-corp.com"

overviewMarkdown: |
  ## Invoice Data Extractor

  Extracts structured data from invoice PDFs including:
  - Header fields (invoice number, date, vendor)
  - Line items (description, quantity, price, total)
  - Totals and tax calculations

  ### Requirements
  - PDF documents (scanned or digital)
  - Base Cloud Model Runtime
  - LLM Model sidecar for AI extraction
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Missing `moduleRuntimeRef` for model type | Models must reference a runtime |
| Wrong `modelRuntimeParameters.module` | Must match the Python package directory name |
| Sidecar without version | Include version: `kodexa/sidecar:1.0.0` |
| `eventAware: true` without `handle_event` function | Add `handle_event()` to your Python code |
| `contents` patterns missing files | Use `**` for recursive glob: `my_module/**.py` |
| Inference options not matching function params | Option `name` must match the Python function parameter name |
