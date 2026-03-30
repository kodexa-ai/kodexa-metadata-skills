---
name: knowledge-system
description: "Use when creating or editing Kodexa knowledge sets, knowledge feature types, or knowledge item types — YAML definitions for rule-based document processing customization connecting document characteristics (features) to processing behaviors (items)"
---

# Kodexa Knowledge System Authoring

## Overview

The Kodexa knowledge system connects **what you know** about documents (features) to **what you do** with them (items). It has three core concepts:

- **Knowledge Feature Types** — Define categories of document characteristics (vendor, document type, language)
- **Knowledge Item Types** — Define configurable processing behaviors (prompt overrides, validation rules, extraction settings)
- **Knowledge Sets** — Bridge features to items using DNF (Disjunctive Normal Form) expressions

**Important**: Feature types and item types are **immutable** — they cannot be updated or deleted once created. Plan carefully before creating them.

## CRITICAL: Feature Instance Slug Computation

**Feature instance slugs are NOT human-friendly names.** They are deterministic, content-addressable identifiers computed from the feature type slug and the properties hash. You MUST compute them correctly.

### Algorithm

```
slug = "{featureTypeSlug}-{sha256(canonicalJSON(properties))[:32]}"
```

Steps:
1. Take only the `properties` (NOT `extendedProperties`) as a key-value map
2. Serialize to canonical JSON with keys sorted alphabetically (e.g., `{"shipperCode":"1414"}`)
3. Compute SHA-256 hash of the UTF-8 bytes
4. Take the first 16 bytes (32 hex characters) of the hash
5. Prepend the feature type slug with a hyphen: `{featureTypeSlug}-{hash}`

### Examples

| Feature Type | Properties | Correct Slug |
|---|---|---|
| `shipper` | `{"shipperCode":"1420"}` | `shipper-265c8ff7acae5ecf2ff06935d99f7d0b` |
| `shipper` | `{"shipperCode":"1414"}` | `shipper-29d437b464f465e0a09a7fd4de6edfd4` |
| `vendor` | `{"vendorId":"ACME-001","vendorCategory":"preferred"}` | `vendor-c410a049207c974c9a4907e1fa008cc0` |

### How to compute

Use this shell command to compute the hash portion:
```bash
echo -n '{"shipperCode":"1414"}' | shasum -a 256 | cut -c1-32
```
Then prepend: `shipper-` + result = `shipper-29d437b464f465e0a09a7fd4de6edfd4`

**NEVER use human-friendly slugs** like `shipper-1414-sbd` or `acme-corp` for feature instances. The platform auto-computes these slugs and will not match a manually invented slug.

The same computed slug must appear in BOTH:
1. `features[].slug` — the feature instance definition
2. `featureExpression` FEATURE nodes — the `slug` field referencing that feature

## When to Use

- Defining document characteristics that vary across document types or vendors
- Creating processing rules that change based on document features
- Customizing extraction prompts per vendor or document type
- Adding conditional validation rules based on document properties
- Setting up knowledge-driven routing and prioritization
- Building a complete knowledge configuration for a project

## Interactive Wizard

1. **Variations** — What varies across your documents? (vendor, document type, language, department)
2. **Behaviors** — What should change based on those variations? (prompts, validation, processing priority)
3. **Feature types** — Define the characteristics (name, slug, options, extended options)
4. **Item types** — Define the behaviors (name, slug, options)
5. **Knowledge sets** — Wire features to items (which combinations trigger which behaviors)

Generate all three YAML definitions as a coordinated set.

## Knowledge Feature Type

Defines a category of document characteristic.

```yaml
slug: vendor                           # Required: unique, immutable once created
name: "Vendor"                         # Required: display name
description: "The vendor/supplier"     # Description
active: true                           # Active flag

# Core options — required properties of a feature instance
options:
  - name: vendorId
    type: string
    label: "Vendor ID"
    description: "Unique vendor identifier"
    required: true

  - name: vendorCategory
    type: select
    label: "Vendor Category"
    required: false
    possibleValues:
      - value: "preferred"
        label: "Preferred Vendor"
      - value: "standard"
        label: "Standard Vendor"

# Extended options — additional metadata
extendedOptions:
  - name: displayName
    type: string
    label: "Display Name"
    description: "Human-readable vendor name"

  - name: website
    type: string
    label: "Website"

  - name: contactEmail
    type: string
    label: "Contact Email"

# Appearance
color: "#3B82F6"                       # Hex color for UI
icon: "building"                       # Icon identifier

# Label path (how to display instances)
labelJsonPath: "$.vendorId"            # JSONPath for instance label
useJSONata: false                      # Use JSONata instead of JSONPath
```

## Knowledge Item Type

Defines a configurable processing behavior.

```yaml
slug: prompt-override                  # Required: unique, immutable once created
name: "Prompt Override"                # Required: display name
description: "Custom extraction prompt for specific fields"
active: true

# Options — what can be configured per item instance
options:
  - name: targetField
    type: string
    label: "Target Field"
    description: "Taxon path this prompt applies to"
    required: true

  - name: promptText
    type: text
    label: "Prompt Text"
    description: "Custom extraction prompt"
    required: true

  - name: includeExamples
    type: boolean
    label: "Include Examples"
    description: "Include extraction examples"
    default: true

  - name: confidenceBoost
    type: number
    label: "Confidence Boost"
    description: "Boost to confidence threshold (0-0.2)"
    default: 0

# Appearance
color: "#8B5CF6"
icon: "chat-bubble"

labelJsonPath: "$.targetField"
```

## Knowledge Set

Bridges features to items using DNF expressions.

```yaml
slug: vendor-specific-extraction       # Required: unique identifier
name: "Vendor-Specific Extraction"     # Display name
description: "Customize extraction per vendor"
setType: extraction                    # Set type
active: true
priority: 5                           # 0-10, higher = applied first
status: ACTIVE                        # PENDING_REVIEW, ACTIVE

# Feature instances used in this set
# IMPORTANT: Feature slugs are computed as {featureTypeSlug}-{sha256(properties)[:32]}
features:
  - slug: vendor-c410a049207c974c9a4907e1fa008cc0  # Computed from properties hash
    featureTypeRef: "my-org/vendor"    # Reference to feature type
    properties:                        # Core option values (included in hash)
      vendorId: "ACME-001"
      vendorCategory: "preferred"
    extendedProperties:                # Extended option values (NOT in hash)
      displayName: "Acme Corporation"
      website: "https://acme.com"
    active: true
    uuid: "feat-uuid-1"               # UUID for clause references

  - slug: vendor-fc0427f8993d7dcaa1156510165a4f21  # Computed from properties hash
    featureTypeRef: "my-org/vendor"
    properties:
      vendorId: "GLOBEX-002"
    active: true
    uuid: "feat-uuid-2"

# Item instances — the behaviors to apply
items:
  - slug: acme-invoice-prompt
    title: "Acme Invoice Number Prompt"
    description: "Custom prompt for Acme invoice numbers"
    knowledgeItemTypeRef: "my-org/prompt-override"
    properties:
      targetField: "invoice/invoice_number"
      promptText: |
        Extract the Acme invoice number.
        Acme uses format: ACME-YYYY-NNNNN (e.g., ACME-2024-00123).
        Look in the top-right corner of the first page.
      includeExamples: true
    active: true
    sequenceOrder: 1

# DNF Clauses — connect features to items
# Clauses are OR'd together; features within a clause are AND'd
clauses:
  - features:                          # This clause matches Acme documents
      - featureUuid: "feat-uuid-1"
        positive: true                 # Feature must be present

  - features:                          # This clause matches Globex documents
      - featureUuid: "feat-uuid-2"
        positive: true

# Feature expression (tree-based matching using computed feature slugs)
featureExpression:
  type: OR
  children:
    - type: FEATURE
      slug: vendor-c410a049207c974c9a4907e1fa008cc0
    - type: FEATURE
      slug: vendor-fc0427f8993d7dcaa1156510165a4f21
```

## Set-Level Attachments

Knowledge sets support **set-level file attachments** — images or other files that belong to the knowledge set and can be referenced from any knowledge item's markdown content using `attachment://` URLs.

### When to use set-level vs per-item attachments

| Approach | Use when |
|----------|----------|
| **Set-level attachment** | An image or file is shared across multiple knowledge items, or referenced in `instructionMarkdown` via `attachment://` URLs |
| **Per-item attachment** | A single file is tightly coupled to one specific knowledge item (e.g., a sample document for that item only) |

Per-item attachments (a single file on a knowledge item, with an `attachmentId` field) still work as before. Set-level attachments are preferred for shared images referenced in markdown.

### Attachment properties

Each set-level attachment has:
- **`attachmentId`** — A unique slug within the knowledge set (e.g., `company-logo`, `signature-sample`)
- **`attachmentPath`** — Local file path relative to the YAML manifest (used during sync)
- Files are stored in S3 with content-addressable hashing for deduplication
- Attachments are cascade-deleted when the knowledge set is removed

### YAML format for sync

```yaml
type: knowledge-set
slug: vendor-extraction-rules
attachments:
  - attachmentId: company-logo             # Unique slug within the set
    attachmentPath: attachments/logo.png   # Relative path to the file
  - attachmentId: signature-sample
    attachmentPath: attachments/signature.jpg
knowledgeItems:
  - slug: extraction-instructions
    # ...
```

### Referencing attachments in markdown

Use the `attachment://` URL scheme in any knowledge item's `instructionMarkdown`:

```markdown
![Company Logo](attachment://company-logo)

Use the signature sample below as a reference:
![Signature](attachment://signature-sample)
```

### API endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/knowledge-sets/{id}/attachments` | List all set-level attachments |
| `POST` | `/api/knowledge-sets/{id}/attachments` | Upload attachment (multipart: file + attachmentId) |
| `GET` | `/api/knowledge-sets/{id}/attachments/{attachmentId}` | Get presigned download URL |
| `DELETE` | `/api/knowledge-sets/{id}/attachments/{attachmentId}` | Remove an attachment |

### CLI

```bash
# Attach a file to a specific knowledge item (per-item attachment)
kdx knowledge attach <ref> --file ./image.png --id my-id

# Set-level attachments sync automatically via the attachments array in YAML
kdx knowledge sync
```

### Complete example with attachments

```yaml
# === Knowledge Set with Set-Level Attachments ===
# Feature slugs are computed: {featureTypeSlug}-{sha256(canonicalJSON(properties))[:32]}
# echo -n '{"vendorId":"ACME"}' | shasum -a 256 | cut -c1-32 → 73182a7521dfac20a3a18c2a4ec549c8
knowledgeSets:
  - slug: vendor-prompts-with-images
    name: "Vendor-Specific Prompts with Reference Images"
    setType: extraction
    active: true
    priority: 7

    # Set-level attachments — shared across all items in this set
    attachments:
      - attachmentId: acme-invoice-header
        attachmentPath: attachments/acme-header.png
      - attachmentId: acme-format-guide
        attachmentPath: attachments/acme-format.png

    features:
      - slug: vendor-73182a7521dfac20a3a18c2a4ec549c8
        featureTypeRef: "my-org/vendor"
        properties:
          vendorId: "ACME"
        extendedProperties:
          vendorName: "Acme Corporation"
        uuid: "acme-uuid"

    items:
      - slug: acme-inv-num
        knowledgeItemTypeRef: "my-org/extraction-prompt"
        title: "Acme Invoice Number"
        properties:
          taxonPath: "invoice/invoice_number"
          prompt: |
            Extract Acme's invoice number in format ACME-YYYY-NNNNN.
            Located in the top-right header area.
          instructionMarkdown: |
            ## Acme Invoice Number Extraction

            The invoice number appears in the header area shown below:

            ![Acme Header](attachment://acme-invoice-header)

            Follow the format guide for edge cases:

            ![Format Guide](attachment://acme-format-guide)
        sequenceOrder: 1

    featureExpression:
      type: AND
      children:
        - type: FEATURE
          slug: vendor-73182a7521dfac20a3a18c2a4ec549c8
```

## DNF Expression Logic

Knowledge sets use Disjunctive Normal Form:
- **Clauses** are OR'd together (any clause matching triggers the set)
- **Features within a clause** are AND'd (all features must match)

```yaml
# Example: Apply rules when (Vendor=Acme AND Type=Invoice) OR (Vendor=Globex)
clauses:
  - features:                          # Clause 1: Acme + Invoice
      - featureUuid: "acme-uuid"
        positive: true
      - featureUuid: "invoice-type-uuid"
        positive: true
  - features:                          # Clause 2: Globex (any type)
      - featureUuid: "globex-uuid"
        positive: true

# Negative features (exclude)
clauses:
  - features:
      - featureUuid: "vendor-uuid"
        positive: true                 # Must have this vendor
      - featureUuid: "draft-type-uuid"
        positive: false                # Must NOT be draft type
```

## Common Feature Type Patterns

### Document Type
```yaml
slug: document-type
name: "Document Type"
options:
  - name: typeId
    type: string
    label: "Type ID"
    required: true
extendedOptions:
  - name: typeName
    type: string
    label: "Type Name"
labelJsonPath: "$.typeId"
```

### Language
```yaml
slug: language
name: "Document Language"
options:
  - name: languageCode
    type: string
    label: "Language Code"
    required: true
    description: "ISO 639-1 code (en, es, fr)"
labelJsonPath: "$.languageCode"
```

### Customer/Department
```yaml
slug: department
name: "Department"
options:
  - name: departmentCode
    type: string
    label: "Department Code"
    required: true
extendedOptions:
  - name: departmentName
    type: string
    label: "Department Name"
  - name: approvalThreshold
    type: number
    label: "Approval Threshold"
labelJsonPath: "$.departmentCode"
```

## Common Item Type Patterns

### Validation Rule
```yaml
slug: validation-rule
name: "Validation Rule"
options:
  - name: fieldPath
    type: string
    label: "Field Path"
    required: true
  - name: ruleFormula
    type: text
    label: "Rule Formula"
    required: true
  - name: errorMessage
    type: string
    label: "Error Message"
    required: true
  - name: severity
    type: select
    label: "Severity"
    default: "warning"
    possibleValues:
      - value: error
        label: "Error (blocking)"
      - value: warning
        label: "Warning (overridable)"
```

### Processing Priority
```yaml
slug: processing-priority
name: "Processing Priority"
options:
  - name: priorityLevel
    type: select
    label: "Priority"
    required: true
    possibleValues:
      - value: high
        label: "High (process first)"
      - value: normal
        label: "Normal"
      - value: low
        label: "Low (batch)"
  - name: slaHours
    type: number
    label: "SLA (hours)"
    default: 24
```

## Complete Coordinated Example

```yaml
# === Feature Type: Vendor ===
featureTypes:
  - slug: vendor
    name: "Vendor"
    description: "Invoice vendor/supplier"
    options:
      - name: vendorId
        type: string
        label: "Vendor ID"
        required: true
    extendedOptions:
      - name: vendorName
        type: string
        label: "Vendor Name"
    labelJsonPath: "$.vendorId"
    color: "#3B82F6"
    icon: "building"

# === Item Type: Prompt Override ===
itemTypes:
  - slug: extraction-prompt
    name: "Extraction Prompt Override"
    description: "Custom prompt for a specific taxon"
    options:
      - name: taxonPath
        type: string
        label: "Taxon Path"
        required: true
      - name: prompt
        type: text
        label: "Custom Prompt"
        required: true
    labelJsonPath: "$.taxonPath"
    color: "#8B5CF6"
    icon: "chat"

# === Knowledge Set ===
# Feature slugs are computed: {featureTypeSlug}-{sha256(canonicalJSON(properties))[:32]}
# echo -n '{"vendorId":"ACME"}' | shasum -a 256 | cut -c1-32 → 73182a7521dfac20a3a18c2a4ec549c8
knowledgeSets:
  - slug: vendor-prompts
    name: "Vendor-Specific Prompts"
    setType: extraction
    active: true
    priority: 7
    features:
      - slug: vendor-73182a7521dfac20a3a18c2a4ec549c8  # Computed from properties
        featureTypeRef: "my-org/vendor"
        properties:
          vendorId: "ACME"
        extendedProperties:
          vendorName: "Acme Corporation"
        uuid: "acme-uuid"
    items:
      - slug: acme-inv-num
        knowledgeItemTypeRef: "my-org/extraction-prompt"
        title: "Acme Invoice Number"
        properties:
          taxonPath: "invoice/invoice_number"
          prompt: |
            Extract Acme's invoice number in format ACME-YYYY-NNNNN.
            Located in the top-right header area.
        sequenceOrder: 1
    featureExpression:
      type: AND
      children:
        - type: FEATURE
          slug: vendor-73182a7521dfac20a3a18c2a4ec549c8
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| **Using human-friendly feature slugs** | Feature instance slugs MUST be computed: `{typeSlug}-{sha256(properties)[:32]}`. Never use names like `acme-corp` or `shipper-1414-sbd` |
| **Feature expression using wrong field** | Expression FEATURE nodes use `slug` (the computed hash slug), NOT `featureUuid` |
| **Including extendedProperties in hash** | Only `properties` are included in the slug hash, NOT `extendedProperties` |
| **Unsorted JSON keys in hash** | Keys must be sorted alphabetically before hashing (JSON canonical form) |
| Trying to update a feature/item type | Types are immutable — create a new version instead |
| Missing UUID on features in clauses | Features need `uuid` field for clause references |
| Feature type without `labelJsonPath` | Required for displaying feature instances |
| Missing `featureTypeRef` on features | Must reference `orgSlug/typeSlug` |
| Knowledge set without expression | Need a `featureExpression` to connect features to items |
| Options vs extendedOptions confusion | Options = required core properties (included in slug hash); extendedOptions = additional metadata (NOT in hash) |
| **Using per-item attachments for shared images** | Use set-level attachments when images are referenced in markdown across multiple items; per-item attachments are for single-item files only |
| **Wrong attachment URL scheme** | Use `attachment://attachmentId` (not `http://` or relative paths) to reference set-level attachments in markdown |
