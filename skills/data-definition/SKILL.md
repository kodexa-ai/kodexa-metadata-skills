---
name: data-definition
description: "Use when creating, editing, or debugging Kodexa data definitions (taxonomies) — YAML files defining taxon hierarchies, data types, validation rules, conditional formatting, and semantic extraction definitions for document processing"
---

# Kodexa Data Definition Authoring

## Overview

Data definitions (also called taxonomies) define the structured data schema for document extraction in Kodexa. They specify what data to extract, how to validate it, and how to display it. This skill provides the complete reference and an interactive wizard for authoring data definition YAML files.

## When to Use

- Creating a new data definition YAML for document extraction
- Adding taxons (fields) to an existing taxonomy
- Configuring validation rules for extracted data
- Setting up repeating groups (line items, tables)
- Debugging extraction issues related to taxonomy configuration
- Adding conditional formatting for visual indicators

## Interactive Wizard

When the developer wants guided creation, walk through these steps:

1. **Document type** — What kind of document? (invoice, contract, form, purchase order, custom)
2. **Key fields** — What data needs to be extracted? List field names and types
3. **Groups** — Are there repeating sections? (line items, parties, addresses)
4. **Validation** — What business rules apply? (required fields, cross-field checks, range constraints)
5. **Formatting** — Any conditional formatting needed? (overdue highlighting, high-value warnings)

Generate the complete YAML with correct hierarchy, slugs, types, and validation.

## Top-Level Structure

```yaml
slug: unique-identifier              # Required: alphanumeric, hyphens, underscores
name: "Display Name"                  # Required: human-readable name
description: "Purpose description"    # What this definition extracts
taxonomyType: CONTENT                 # CONTENT (typical), PROCESSING, MODULE
enabled: true                         # Whether definition is active
taxons: []                            # Root-level taxons array
validationRules: []                   # Top-level validation rules
conditionalFormats: []                # Top-level conditional formats
```

## Data Types Reference

| Type | Use For | Key TypeFeatures |
|------|---------|-----------------|
| `STRING` | Names, addresses, text | `longText`, `maxTextRows`, `markdown`, `expected` |
| `NUMBER` | Quantities, counts | `truncateDecimal`, `decimalPlaces` |
| `CURRENCY` | Monetary amounts | `preferTwoDecimalPlaces` |
| `DATE` | Calendar dates | `normalizeDate`, `dateFormat` (e.g., `yyyy-MM-dd`) |
| `DATE_TIME` | Timestamps | `normalizeDate`, `dateFormat` |
| `SELECTION` | Dropdowns, categories | `selectionOptions` array |
| `BOOLEAN` | Yes/no flags | — |
| `EMAIL_ADDRESS` | Email with validation | — |
| `PHONE_NUMBER` | Phone with normalization | — |
| `URL` | Web addresses | — |
| `PERCENTAGE` | Percent values | — |
| `SECTION` | Visual grouping only (no data) | — |

## Complete Taxon Field Reference

```yaml
# === Core Identity ===
name: field_name                      # Required: snake_case identifier
label: "Display Label"                # Required: human-readable label
description: "Field explanation"      # Detailed description
enabled: true                         # Active flag (default: true)
color: "#4F46E5"                      # Hex color for UI
externalName: "externalFieldName"     # Name for external systems (camelCase)

# === Type Configuration ===
taxonType: STRING                     # One of the 12 data types above
typeFeatures:                         # Type-specific settings
  expected: true                      # Field is expected/required in extraction
  longText: true                      # Multi-line text (STRING)
  maxTextRows: 10                     # Max display rows (STRING)
  markdown: true                      # Enable markdown (STRING)
  normalizeDate: true                 # Normalize dates (DATE/DATE_TIME)
  dateFormat: "yyyy-MM-dd"            # Target format (DATE/DATE_TIME)
  truncateDecimal: true               # Round decimals (NUMBER)
  decimalPlaces: 2                    # Decimal places (NUMBER)
  preferTwoDecimalPlaces: true        # Interpret last 2 digits as decimal (CURRENCY)
  overrideWidth: true                 # Override display width
  displayWidth: 150                   # Width in pixels

# === Data Source ===
valuePath: VALUE_OR_ALL_CONTENT       # How to get the value (see Value Paths below)
semanticDefinition: |                 # Extraction prompt - be specific!
  Clear guidance on what to extract and where to find it.
metadataValue: FILENAME               # For METADATA valuePath only

# === Group Configuration ===
group: true                           # Mark as container (no data stored itself)
children: []                          # Array of child taxons
cardinality:                          # For repeating groups
  min: 1                              # Minimum instances
  max: 100                            # Maximum instances

# === Selection Options ===
selectionOptions:                     # For SELECTION type only
  - label: "Display Text"
    id: "option_id"
    description: "Option description"
    lexicalRelations:                 # Synonyms for AI matching
      - type: SYNONYM
        value: "syn1, syn2"

# === Validation ===
validationRules:
  - name: "Rule Name"
    ruleFormula: "NOT_EMPTY(field)"   # Must evaluate true to pass
    messageFormula: '"Error message"' # Shown on failure
    overridable: true                 # Can user override?
    conditional: false                # Only apply if conditionalFormula is true
    conditionalFormula: ""            # When to apply this rule

# === Conditional Formatting ===
conditionalFormats:
  - name: "Format Name"
    formula: "field > 1000"           # Boolean condition
    backgroundColor: "#FEE2E2"        # Hex background
    textColor: "#991B1B"              # Hex text
    fontWeight: "bold"                # Font weight
    icon: "warning"                   # Icon name

# === Display ===
userEditable: true                    # Can user edit in forms
notUserLabelled: false                # Hide from labeling interface
multiValue: false                     # Allow multiple values
nullable: true                        # Allow null/empty
```

## Value Paths

| Value Path | Purpose | Key Field |
|-----------|---------|-----------|
| `VALUE_OR_ALL_CONTENT` | Extract from document | `semanticDefinition` — extraction prompt |
| `VALUE_ONLY` | Extract from value region only | `semanticDefinition` |
| `ALL_CONTENT` | Use all page content | `semanticDefinition` |
| `METADATA` | Document properties | `metadataValue`: FILENAME, TRANSACTION_UUID, CREATED_DATETIME, DOCUMENT_LABELS, OWNER_NAME, DOCUMENT_STATUS, PAGE_NUMBER |
| `FORMULA` | Calculated value | `semanticDefinition` — formula expression |
| `REVIEW` | Human review template | `semanticDefinition` — Jinja2 template |
| `EXTERNAL` | External API/database | `expression` — Groovy code |
| `EXPRESSION` | Spring Expression | `expression` — SpEL code |

## Validation Formula Functions

| Function | Example | Description |
|----------|---------|-------------|
| `NOT_EMPTY(field)` | `NOT_EMPTY(invoice_number)` | Field has value |
| `IS_EMPTY(field)` | `IS_EMPTY(notes)` | Field is empty |
| `NOT(expr)` | `NOT(IS_EMPTY(date))` | Logical NOT |
| `SUM(group.field)` | `SUM(line_items.amount)` | Sum within group |
| `COUNT(group)` | `COUNT(line_items)` | Count group instances |
| `ABS(value)` | `ABS(difference)` | Absolute value |
| `COALESCE(a, b, default)` | `COALESCE(tax, 0)` | First non-null |
| `TODAY()` | `date <= TODAY()` | Current date |
| `DATE_ADD(date, n, unit)` | `DATE_ADD(date, 30, 'DAYS')` | Date arithmetic |
| `LENGTH(str)` | `LENGTH(name) > 2` | String length |
| `REGEX_MATCH(str, pat)` | `REGEX_MATCH(id, "^INV-")` | Regex validation |
| `IN(field, [vals])` | `IN(status, ["A","B"])` | Value in list |
| `CONTAINS(field, sub)` | `CONTAINS(name, "Inc")` | Substring check |
| `IF(cond, t, f)` | `IF(qty>0, "ok", "err")` | Conditional logic |
| `ALL_VALUES(grp.f)` | `ALL_VALUES(items.qty > 0)` | All group values match |
| `UNIQUE(grp.f)` | `UNIQUE(items.id)` | Values are unique |

## Conditional Formatting Colors

| Purpose | Background | Text |
|---------|-----------|------|
| Red alert | `#FEE2E2` | `#991B1B` |
| Yellow warning | `#FEF3C7` | `#92400E` |
| Green success | `#D1FAE5` | `#065F46` |
| Blue info | `#DBEAFE` | `#1E40AF` |

## Group Patterns

**Single-instance group** (organizational container):
```yaml
- name: vendor
  label: Vendor Information
  group: true
  children:
    - name: name
      label: Name
      taxonType: STRING
    - name: address
      label: Address
      taxonType: STRING
```

**Repeating group** (line items, table rows):
```yaml
- name: line_items
  label: Line Items
  group: true
  cardinality:
    min: 1
    max: 500
  additionContexts:                        # Helps AI find record boundaries
    - type: RECORD_DEFINITION
      context: "Each line item has description, quantity, unit price, total"
    - type: RECORD_START_MARKER
      context: "Description, Item, Product"
    - type: RECORD_END_MARKER
      context: "Subtotal, Total"
  children:
    - name: description
      taxonType: STRING
    - name: quantity
      taxonType: NUMBER
    - name: unit_price
      taxonType: CURRENCY
    - name: line_total
      taxonType: CURRENCY
      valuePath: FORMULA
      semanticDefinition: "line_items.quantity * line_items.unit_price"
```

## Semantic Definition Best Practices

```yaml
# GOOD - specific, tells AI exactly where to look
semanticDefinition: |
  The vendor's legal business name as registered with tax authorities.
  Look for names near "Bill To", "Vendor", or "From" sections.
  Should be the company name, not an individual's name.
  If multiple names appear, choose the one on official letterhead.

# BAD - too vague
semanticDefinition: "vendor name"
```

## Complete Invoice Example

```yaml
slug: invoice-extraction
name: Invoice Data Extraction
description: Extract structured data from invoices with validation
taxonomyType: CONTENT
enabled: true

taxons:
  - name: invoice_number
    label: Invoice Number
    taxonType: STRING
    valuePath: VALUE_OR_ALL_CONTENT
    semanticDefinition: |
      The unique invoice number, found at the top of the document.
      Look for "Invoice #", "Invoice No.", or "Ref #".
    typeFeatures:
      expected: true
    validationRules:
      - name: "Invoice number required"
        ruleFormula: "NOT_EMPTY(invoice_number)"
        messageFormula: '"Invoice number is required"'
        overridable: false

  - name: invoice_date
    label: Invoice Date
    taxonType: DATE
    valuePath: VALUE_OR_ALL_CONTENT
    semanticDefinition: |
      The date the invoice was issued. Look for "Date" or "Invoice Date".
    typeFeatures:
      normalizeDate: true
      dateFormat: "yyyy-MM-dd"
      expected: true
    validationRules:
      - name: "Date not in future"
        ruleFormula: "invoice_date <= TODAY()"
        messageFormula: '"Invoice date cannot be in the future"'
        overridable: true

  - name: vendor
    label: Vendor Information
    group: true
    children:
      - name: name
        label: Vendor Name
        taxonType: STRING
        valuePath: VALUE_OR_ALL_CONTENT
        semanticDefinition: |
          The vendor's legal business name.
          Look near "From", "Vendor", or "Bill From" sections.
        typeFeatures:
          expected: true
      - name: address
        label: Address
        taxonType: STRING
        valuePath: VALUE_OR_ALL_CONTENT
        typeFeatures:
          longText: true
      - name: email
        label: Email
        taxonType: EMAIL_ADDRESS
        valuePath: VALUE_OR_ALL_CONTENT
        nullable: true

  - name: payment_terms
    label: Payment Terms
    taxonType: SELECTION
    valuePath: VALUE_OR_ALL_CONTENT
    selectionOptions:
      - label: "Due on Receipt"
        id: "due_on_receipt"
        lexicalRelations:
          - type: SYNONYM
            value: "Immediate, Upon Receipt, COD"
      - label: "Net 30"
        id: "net_30"
      - label: "Net 60"
        id: "net_60"

  - name: line_items
    label: Line Items
    group: true
    cardinality:
      min: 1
      max: 500
    children:
      - name: description
        label: Description
        taxonType: STRING
        valuePath: VALUE_OR_ALL_CONTENT
        typeFeatures:
          expected: true
      - name: quantity
        label: Quantity
        taxonType: NUMBER
        valuePath: VALUE_OR_ALL_CONTENT
        validationRules:
          - name: "Positive quantity"
            ruleFormula: "line_items.quantity > 0"
            messageFormula: '"Quantity must be greater than zero"'
      - name: unit_price
        label: Unit Price
        taxonType: CURRENCY
        valuePath: VALUE_OR_ALL_CONTENT
      - name: line_total
        label: Line Total
        taxonType: CURRENCY
        valuePath: FORMULA
        semanticDefinition: "line_items.quantity * line_items.unit_price"

  - name: subtotal
    label: Subtotal
    taxonType: CURRENCY
    valuePath: FORMULA
    semanticDefinition: "SUM(line_items.line_total)"

  - name: tax_amount
    label: Tax Amount
    taxonType: CURRENCY
    valuePath: VALUE_OR_ALL_CONTENT
    nullable: true

  - name: total_amount
    label: Total Amount Due
    taxonType: CURRENCY
    valuePath: VALUE_OR_ALL_CONTENT
    semanticDefinition: |
      The final total amount due, including taxes.
      Look for "Total", "Amount Due", or "Balance Due".
    typeFeatures:
      expected: true
    validationRules:
      - name: "Total required"
        ruleFormula: "NOT_EMPTY(total_amount)"
        messageFormula: '"Total amount is required"'
        overridable: false
      - name: "Total matches calculation"
        conditional: true
        conditionalFormula: "NOT_EMPTY(subtotal)"
        ruleFormula: "ABS(total_amount - (subtotal + COALESCE(tax_amount, 0))) < 0.01"
        messageFormula: '"Total does not match subtotal + tax"'
        overridable: true
    conditionalFormats:
      - name: "High value"
        formula: "total_amount > 10000"
        backgroundColor: "#FEF3C7"
        textColor: "#92400E"
        icon: "warning"

validationRules:
  - name: "Due date after invoice date"
    conditional: true
    conditionalFormula: "NOT_EMPTY(due_date) AND NOT_EMPTY(invoice_date)"
    ruleFormula: "due_date >= invoice_date"
    messageFormula: '"Due date must be on or after invoice date"'
    overridable: false
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `required: true` instead of validation rule | Use `typeFeatures.expected: true` + a `validationRules` entry with `NOT_EMPTY()` |
| Vague `semanticDefinition` | Be specific about where to find the data and what format to expect |
| Missing `cardinality` on repeating groups | Always set `min`/`max` on groups that repeat |
| Using `FORMULA` without proper formula syntax | Formulas reference field names, use SUM/COUNT/IF functions |
| Wrong validation formula reference in groups | Use `group_name.field_name` (e.g., `line_items.quantity`) |
| Missing `overridable` on validation rules | Always specify — `false` for hard requirements, `true` for warnings |
