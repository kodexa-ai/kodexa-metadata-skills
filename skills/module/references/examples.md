# Worked examples

## 1. Python model module

`module.yml`, sitting beside the `invoice_extractor/` package:

```yaml
type: module
slug: invoice-extractor
orgSlug: acme-corp
name: "Invoice Data Extractor"
description: "Extracts header fields and line items from invoice PDFs"
moduleType: model
moduleStatus: PUBLISHED
deprecated: false
eventAware: false
supportsScheduling: false
publicAccess: false

metadata:
  moduleRuntimeRef: kodexa/base-cloud-model-runtime
  moduleRuntimeParameters:
    module: invoice_extractor
    function: infer
  moduleSidecars:
    - kodexa/kodexa-llm-model
  inferable: true

  contents:
    - invoice_extractor/**.py
    - invoice_extractor/templates/**
  ignoredContents:
    - "**/__pycache__/**"
    - "**/*.pyc"
    - "**/tests/**"

  inferenceOptions:
    - name: confidence_threshold
      type: number
      default: 0.85
      label: "Confidence threshold"
      description: "Minimum confidence for an extracted value (0.0 – 1.0)"

    - name: extract_line_items
      type: boolean
      default: true
      label: "Extract line items"

    - name: output_format
      type: string
      default: json
      label: "Output format"
      possibleValues:
        - { value: json, label: "JSON" }
        - { value: csv,  label: "CSV" }

    - name: vendor_api_key
      type: password              # unlocks the org secret picker
      label: "Vendor API key"
      required: false
```

`invoice_extractor/__init__.py`:

```python
import logging

from kodexa_document import Document

logger = logging.getLogger(__name__)


def infer(
    document: Document,
    confidence_threshold: float = 0.85,
    extract_line_items: bool = True,
    output_format: str = "json",
    vendor_api_key: str = None,
    pipeline_context=None,
    status_reporter=None,          # never injected today — default and guard
):
    """Process a document and return it with extracted data."""
    logger.info("Processing %s", document.uuid)

    if status_reporter:            # dead today, harmless
        status_reporter.update("Analyzing document", status_type="analyzing")

    content = document.content_node.get_all_content()
    logger.info("%d characters of content", len(content))

    if pipeline_context is not None and pipeline_context.document_family is not None:
        logger.info("Family path: %s", pipeline_context.document_family.path)

    document.add_label("processed")
    return document
```

Every option default is repeated in the signature, because Studio only seeds truthy YAML defaults
into the step's option map and the runtime only passes keys it actually has.

## 2. Event-driven module

```yaml
type: module
slug: stale-invoice-sweeper
orgSlug: acme-corp
name: "Stale Invoice Sweeper"
description: "Flags invoices that have sat in review past their deadline"
moduleType: model
eventAware: true                  # top level — column
supportsScheduling: true          # top level — column
metadata:
  moduleRuntimeRef: kodexa/base-model-runtime
  moduleRuntimeParameters:
    module: stale_invoice_sweeper
    function: handle_event
  inferable: false
  contents:
    - stale_invoice_sweeper/**.py
  ignoredContents:
    - "**/__pycache__/**"
  inferenceOptions:
    - name: document_store_ref
      type: documentStore
      label: "Document store"
      required: true
    - name: overdue_status_id
      type: documentStatus
      label: "Overdue status"
      required: true
```

```python
import logging

from kodexa_document.platform.client import KodexaClient

logger = logging.getLogger(__name__)


def handle_event(
    document_store_ref: str = None,
    overdue_status_id: str = None,
    project=None,
    task_id: str = None,
    event_type: str = None,
    execution_id: str = None,
):
    """Event dispatches carry no document — resolve what you need yourself.

    Note there is no `event` parameter: it is not injected. The identifying
    values arrive as channel_id / message_id / task_id / event_type when the
    dispatch context carries them.
    """
    if not document_store_ref:
        logger.error("document_store_ref option is not set")
        return

    client = KodexaClient()
    # Use the full object type: a bare "store" matches both document and data
    # stores and raises "too many potential matches".
    store = client.get_object_by_ref("document-store", document_store_ref)
    logger.info("Sweeping %s (%s) for event_type=%s", document_store_ref, store.id, event_type)
```

## 3. Go-WASM module

Layout — keep the manifest beside the Go source so the built artifact lands at the ZIP root:

```
archive-publisher/
  module.yml
  main.go
  go.mod
```

```yaml
type: module
slug: archive-publisher
orgSlug: acme-corp
name: "Archive Publisher"
description: "Uploads the current document's KDDB bytes to an archive endpoint"
moduleType: model
eventAware: true
metadata:
  bridgeType: wasm
  moduleRuntimeRef: kodexa/base-model-runtime
  build:
    - lang: go-wasm
      workdir: .
      output: plugin.wasm
  contents:
    - plugin.wasm
  allowedHosts:
    - "*.kodexa.example.com"
  inferenceOptions:
    - name: archive_url
      type: password
      label: "Archive endpoint URL"
      required: true
    - name: overwrite
      type: boolean
      default: true
      label: "Overwrite existing"
```

```go
package main

import (
    "encoding/json"

    sdk "github.com/kodexa/kodexa-plugin-sdk"
)

// The export name has to be `infer` — that is the entry point the orchestrator
// stamps on every execution step.
//
//go:wasmexport infer
func infer() int32 {
    input := sdk.Input()
    if len(input) == 0 {
        sdk.Log(sdk.LogError, "no input provided")
        return 1
    }

    // Options arrive as one JSON config entry named module_options. The SDK's
    // GetOption* helpers read a different key that is never set, so they would
    // just hand back the defaults.
    opts := map[string]any{}
    if raw, ok := sdk.GetConfig("module_options"); ok && raw != "" {
        if err := json.Unmarshal([]byte(raw), &opts); err != nil {
            sdk.Log(sdk.LogError, "bad module_options: "+err.Error())
            return 1
        }
    }
    url, _ := opts["archive_url"].(string)
    overwrite, ok := opts["overwrite"].(bool)
    if !ok {
        overwrite = true
    }
    if url == "" {
        sdk.Log(sdk.LogError, "archive_url option is required")
        return 1
    }

    sdk.ReportStatus("uploading archive copy")

    // ... an Extism pdk.HTTPRequest to url, gated by metadata.allowedHosts ...
    _ = overwrite

    sdk.Output(input)
    return 0
}

func main() {}
```

`kdx apply -f module.yml` compiles `plugin.wasm` with your local Go toolchain, packages it, and
uploads it in one step.

## 4. Inline-JavaScript module

```yaml
type: module
slug: threshold-router
orgSlug: acme-corp
name: "Threshold Router"
description: "Routes a document based on a confidence threshold"
moduleType: model
metadata:
  moduleRuntimeRef: kodexa/go-scripting-runtime
  scriptLanguage: javascript
  script: |
    var threshold = parameters.confidence_threshold || 0.8;
    var score = parameters.score || 0;

    log.info("execution " + parameters.execution_id + " score=" + score);

    if (score < threshold) {
      return { status: "completed", route: "manual_review" };
    }
    return { status: "completed", route: "auto_approve" };
  inferenceOptions:
    - name: confidence_threshold
      type: number
      default: 0.8
      label: "Confidence threshold"
    - name: score
      type: number
      default: 0
      label: "Score"
```

No `contents`, no ZIP. The returned object becomes the step's output event (return anything that
is not an object and the output event is empty). Note the `|| 0` guard on `score`: `default: 0`
is falsy, so Studio never seeds it into the step's option map and the key simply is not there.

## 5. Agent skill pack

```
invoice-review-skills/
  module.yml
  SKILL.md              <- must be at the ZIP root
  SYSTEM_PROMPT.md
  prompts/
    triage.md
    escalation.md
  tools/
    lookup.yml
```

```yaml
type: module
slug: invoice-review-skills
orgSlug: acme-corp
name: "Invoice Review Skills"
description: "Prompts and helper tools for invoice review agents"
moduleType: skill                  # top level — without this it stays a runnable model
metadata:
  contents:
    - SKILL.md
    - SYSTEM_PROMPT.md
    - prompts/**
    - tools/**
  ignoredContents:
    - "**/*.pyc"
```

Because `contents` are globbed relative to the manifest's directory, `SKILL.md` lands at the ZIP
root and the agent SDK — which walks exactly one level deep — discovers it at
`<workspace>/.claude/skills/invoice-review-skills/SKILL.md`.

## 6. Action-native module

```yaml
type: module
slug: invoice-export
orgSlug: acme-corp
name: "Invoice Export"
description: "Actions an agent can call to validate and export invoice data"
moduleType: model
metadata:
  moduleRuntimeRef: kodexa/claude-code-agent-java
  moduleRuntimeParameters:
    module: invoice_export
  contents:
    - invoice_export/**.py
  ignoredContents:
    - "**/__pycache__/**"
  actions:
    - name: validate-invoice-data
      entry_point: validate_invoice_data
      label: "Validate invoice data"
      description: "Check that the documents in a task are ready for export"
      inputs:
        - { name: document_store_ref, type: string, required: true }
    - name: export-invoices
      entry_point: export_invoices
      label: "Export invoices"
```

```python
def validate_invoice_data(document_store_ref: str = None, module_ref: str = None,
                          execution_id: str = None):
    """Actions return a JSON-serialisable value as their result."""
    return {"ok": True, "store": document_store_ref}


def export_invoices(document_store_ref: str = None, task_id: str = None):
    return {"exported": 0}
```

The in-container bridge injects only the option values plus `model_base`, `execution_id`,
`module_ref`, and the event-id kwargs — no `document`, no `pipeline_context`. Fetch anything else
through the platform client.
