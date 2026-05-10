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

**Mutability rules** (verified against the platform code):

| Entity | Mutable? | Notes |
|---|---|---|
| **`KnowledgeFeatureType`** | **Yes** — every field can be edited via PUT after creation. Name, color, icon, options, extendedOptions, labelJsonPath, useJSONata, active, slug all updateable. |
| **`KnowledgeItemType`** | **Yes** — every field can be edited via PUT after creation. Name, slug, description, options, supportsAttachment all updateable. |
| **`KnowledgeFeature.properties`** (instance core props) | **Effectively no** — physically writable, but the slug is computed from `properties` at create time and **does not recompute on update**. Editing `properties` after create produces a stale hash that no longer matches the slug. Treat as immutable in practice. |
| **`KnowledgeFeature.extendedProperties`** (instance extended props) | **Yes** — fully mutable. Not part of the slug hash. Use this for any per-instance fact that may change over time. |
| **`KnowledgeItem`** | **Yes** — title, description, properties, sequenceOrder, active, attachmentId all updateable via PUT. Item slug is user-supplied (no hash), so changing it is fine. |

**Practical implications:**

- **Add options to a feature/item type whenever you need to.** No need to version the type or pre-add `extras` blob fields as a safety valve — that advice belonged to a previous, incorrect understanding of immutability.
- **Removing a required option from a type is risky** — existing instances may not satisfy the new schema. Mark options optional first, then drop later.
- **Changing a feature type's `labelJsonPath` retroactively** affects how every existing instance is displayed. Test in a non-prod org first.
- **For instance core properties (the ones that hash into the slug):** structure your initial design carefully because changing them later leaves the slug pointing at the old hash. If the property values must change, **delete the feature and recreate it** with the new values (which will produce a new slug). Anything referencing the old slug needs to be updated to match.

## CRITICAL: Feature Instance Slug Computation

**Feature instance slugs are NOT human-friendly names.** They are deterministic, content-addressable identifiers computed from the feature type slug and the properties hash. You MUST compute them correctly.

### Algorithm

```
slug = "{featureTypeSlug}-{sha256(canonicalJSON(properties))[:32]}"
```

Steps:
1. Take only the `properties` (NOT `extendedProperties`) as a key-value map
2. Serialize to canonical JSON with keys sorted alphabetically (e.g., `{"vendorCode":"ACME-001"}`)
3. Compute SHA-256 hash of the UTF-8 bytes
4. Take the first 16 bytes (32 hex characters) of the hash
5. Prepend the feature type slug with a hyphen: `{featureTypeSlug}-{hash}`

### Examples

| Feature Type | Properties | Correct Slug |
|---|---|---|
| `vendor` | `{"vendorCode":"GLOBEX-002"}` | `vendor-800a90c4cc3aca5d57bd11fde7356d52` |
| `vendor` | `{"vendorCode":"ACME-001"}` | `vendor-d9d23e980a4863d7df0268da7647f64e` |
| `vendor` | `{"vendorId":"ACME-001","vendorCategory":"preferred"}` | `vendor-c410a049207c974c9a4907e1fa008cc0` |

### How to compute

Use this shell command to compute the hash portion:
```bash
echo -n '{"vendorCode":"ACME-001"}' | shasum -a 256 | cut -c1-32
```
Then prepend: `vendor-` + result = `vendor-d9d23e980a4863d7df0268da7647f64e`

**NEVER use human-friendly slugs** like `vendor-acme-preferred` or `acme-corp` for feature instances. The platform auto-computes these slugs and will not match a manually invented slug.

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
slug: vendor                           # Required: unique. Editable later via PUT.
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
slug: prompt-override                  # Required: unique. Editable later via PUT.
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

## Option Types Reference

`options` and `extendedOptions` (on feature types) and `options` (on item types) accept a richer type vocabulary than just primitives. This is the same option machinery used across the platform (assistants, modules, taxonomies, project templates) — anything those support, knowledge types support too.

### Basic types

| `type:` | Use for | Notable extras |
|---|---|---|
| `string` | Single-line text | `password: true` masks input; `possibleValues` constrains it |
| `text` | Multi-line text (textarea) | `lines: N` sets initial height |
| `number` | Numeric input | `min`, `max` for validation |
| `boolean` | Checkbox / toggle | — |
| `select` | Dropdown | `possibleValues: [{value, label}]` required |
| `code` | Syntax-highlighted code editor | Use for regex, JSONata, scripts — much nicer than `string` |
| `script` | Multi-language script editor | — |

### Composite types

| `type:` | Use for | How |
|---|---|---|
| `list` + `listType: string` | Array of strings | UI gives add/remove buttons |
| `list` + `listType: object` | Array of structured rows | Define each row's fields under `groupOptions:` |
| `object` | Single grouped sub-record | Define fields under `groupOptions:`; set `properties.collapsible: true` for collapsible UI |

```yaml
# Array of structured rows — like an editable table
options:
  - name: fieldMappings
    type: list
    listType: object
    label: "Charge type → service mappings"
    groupOptions:
      - name: serviceType
        type: string
        required: true
      - name: regex
        type: code              # gets syntax highlighting
        required: true
```

```yaml
# Bounded numeric tuple — modeled as an object with named fields
options:
  - name: bbox
    type: object
    label: "Bounding box (page coords)"
    properties:
      collapsible: true
    groupOptions:
      - name: x1
        type: number
        min: 0
      - name: y1
        type: number
        min: 0
      - name: x2
        type: number
      - name: y2
        type: number
```

### Platform-aware types

Specialised pickers that resolve against existing org resources:

| `type:` | Picks |
|---|---|
| `documentStore` / `moduleStore` / `tableStore` / `taxonomyStore` | Resource of that kind |
| `document` | Specific document (by search) |
| `workspace` | Workspace |
| `taxon` | Single taxonomy element |
| `taxon_label` | Taxonomy label |
| `taxon_with_properties` | Taxon + configured properties |
| `taxon-lookup` | Hierarchical taxonomy search |
| `documentStatus` / `attributeStatus` / `taskStatus` | Status enum |
| `cloud-model` | Cloud LLM (with caching) |
| `cloud-embedding` | Embedding model |
| `data-form` | Data form configuration |

Use `taxon` rather than `string` whenever the option references a taxonomy element — the UI gives users a real picker instead of free-text typing the path.

### Conditional visibility (`showIf`)

Hide an option until another option's value is set. The expression is JavaScript evaluated against the option object:

```yaml
options:
  - name: mode
    type: select
    label: Capture mode
    possibleValues:
      - value: line
        label: "Line marker"
      - value: column
        label: "Column marker"

  - name: lineMarker
    type: code
    label: "Line regex"
    showIf: "this.mode === 'line'"

  - name: columnHeader
    type: string
    label: "Column header"
    showIf: "this.mode === 'column'"
```

This is the canonical way to model rules whose schema branches based on a mode flag — no need to invent multiple item types just because parameters differ across modes.

### Other UX modifiers

- `password: true` — masks the value (use for API keys, secrets)
- `developerOnly: true` — only visible when the user toggles "Show Developer Tools"
- `properties.collapsible: true` — `object` group renders as a collapsible card
- `default:` — must structurally match the type (string for `string`, array for `list`, object for `object`); mismatched defaults silently break the editor

### Anti-patterns

- **Don't use `text` for what should be a `list`.** Users get a multiline blob and your code has to parse it. Use `list` + `listType: string` instead.
- **Don't use `text` for structured config.** Use `object` + `groupOptions` so the UI gives proper inputs and your code receives a typed map.
- **Don't `string` a regex.** Use `code` — the UI gets syntax highlighting and the value is still a plain string in code.
- **Don't `string` a taxon path.** Use `taxon` — eliminates an entire class of typo bugs.

### Why this matters

When migrating any hardcoded JSON config to knowledge items, the platform's option types let you keep the structure typed and editable rather than dumping JSON blobs into `text` fields. Almost every JSON shape you'd want to migrate maps cleanly:

| JSON shape | Use |
|---|---|
| `["a", "b", "c"]` | `list` + `listType: string` |
| `{"x": "regex"}` (string→string map with dynamic keys) | `list` + `listType: object` with `{key, value}` row |
| `[1.5, 3.5, 2.0, 4.0]` (fixed-length tuple) | `object` with named numeric fields |
| `{flag: true, mode: "line", marker: "..."}` | `object` with `select` + `boolean` + branchy `showIf` fields |
| Nested config | `object` containing `list`/`object` (≤3 levels deep) |

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
| **Using human-friendly feature slugs** | Feature instance slugs MUST be computed: `{typeSlug}-{sha256(properties)[:32]}`. Never use names like `acme-corp` or `vendor-acme-preferred` |
| **Feature expression using wrong field** | Expression FEATURE nodes use `slug` (the computed hash slug), NOT `featureUuid` |
| **Including extendedProperties in hash** | Only `properties` are included in the slug hash, NOT `extendedProperties` |
| **Unsorted JSON keys in hash** | Keys must be sorted alphabetically before hashing (JSON canonical form) |
| Editing a feature instance's `properties` after creation | Don't — the slug was hashed from the original `properties` and **does not recompute on update**, leaving a stale slug. Edit `extendedProperties` instead, or delete + recreate the feature with the new properties (which produces a new slug). |
| Treating feature / item types as immutable | They're not — every field is editable via PUT. Add options as needed. (Removing a `required` option is still risky for existing instances; soften to optional first.) |
| Missing UUID on features in clauses | Features need `uuid` field for clause references |
| Feature type without `labelJsonPath` | Required for displaying feature instances |
| Missing `featureTypeRef` on features | Must reference `orgSlug/typeSlug` |
| Knowledge set without expression | Need a `featureExpression` to connect features to items |
| Options vs extendedOptions confusion | Options = required core properties (included in slug hash); extendedOptions = additional metadata (NOT in hash) |
| **Using per-item attachments for shared images** | Use set-level attachments when images are referenced in markdown across multiple items; per-item attachments are for single-item files only |
| **Wrong attachment URL scheme** | Use `attachment://attachmentId` (not `http://` or relative paths) to reference set-level attachments in markdown |
