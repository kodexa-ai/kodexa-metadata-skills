---
name: prompt-template
description: "Use when creating or editing Kodexa prompt templates — YAML definitions for LLM prompt configurations used by assistants and modules for document processing, data extraction, and AI-driven analysis"
---

# Kodexa Prompt Template Authoring

## Overview

Prompt templates define reusable LLM prompts for document processing in Kodexa. They can be standalone metadata resources referenced by assistants and modules, or embedded within skill modules as markdown files. Prompts drive extraction, classification, summarization, and other AI-powered document processing tasks.

## When to Use

- Creating extraction prompts for document processing
- Defining system prompts for AI assistants
- Building reusable prompt templates with variable placeholders
- Embedding prompts in skill modules
- Customizing prompts per vendor/document type via knowledge system

## Interactive Wizard

1. **Purpose** — What should the prompt do? (extract data, classify, summarize, validate, transform)
2. **Target** — What data fields or outcomes? (specific taxon paths, categories, summaries)
3. **Context** — What context does the model need? (document structure, field definitions, examples)
4. **Format** — What output format? (JSON, structured text, labels)
5. **Variations** — Do prompts vary by document type? (use knowledge system for overrides)

## Prompt Template YAML Structure

```yaml
slug: my-prompt                        # Required: unique identifier
orgSlug: my-org                        # Required: organization slug
name: "My Prompt Template"            # Required: display name
type: promptTemplate                   # Required: must be "promptTemplate"
description: "What this prompt does"

metadata:
  # Prompt content
  systemPrompt: |                      # System-level instructions
    You are a document processing agent specializing in invoice extraction.
    Follow these rules strictly:
    1. Extract only data that is explicitly present in the document
    2. Return null for fields that cannot be found
    3. Use the exact format specified for each field

  userPromptTemplate: |                # User prompt with variables
    Extract the following fields from this {{documentType}} document:

    {{#each fields}}
    - {{this.name}}: {{this.description}} (type: {{this.type}})
    {{/each}}

    Document content:
    {{documentContent}}

  # Model configuration
  modelProvider: anthropic             # anthropic, openai, bedrock
  modelId: claude-sonnet-4-6          # Model identifier
  temperature: 0.1                     # 0-1, lower = more deterministic
  maxTokens: 4096                      # Max response tokens

  # Output configuration
  outputFormat: json                   # json, text, structured
  outputSchema:                        # JSON schema for structured output
    type: object
    properties:
      invoice_number:
        type: string
      total_amount:
        type: number
```

## Prompt in Skill Modules

Prompts are commonly stored as markdown files in skill modules:

```
my-skills/
  module.yml
  prompts/
    system.md                          # System prompt
    extraction.md                      # Extraction prompt template
    classification.md                  # Classification prompt
    validation.md                      # Validation prompt
```

### System Prompt (prompts/system.md)

```markdown
You are an invoice extraction agent for the Kodexa platform.

## Rules
1. Extract only data explicitly present in the document
2. Return null for fields not found
3. Preserve original formatting for addresses and names
4. Use ISO 8601 format for dates (YYYY-MM-DD)
5. Use numeric values without currency symbols for amounts

## Output Format
Return a JSON object with the extracted fields.
```

### Extraction Prompt (prompts/extraction.md)

```markdown
Extract the following data from this invoice document:

### Required Fields
- **invoice_number**: The unique invoice identifier (e.g., INV-2024-001)
- **invoice_date**: The date the invoice was issued (YYYY-MM-DD)
- **total_amount**: The final total amount due (numeric, no currency symbol)

### Vendor Information
- **vendor_name**: Legal business name of the vendor
- **vendor_address**: Full mailing address

### Line Items
For each line item, extract:
- **description**: Product/service description
- **quantity**: Number of units (numeric)
- **unit_price**: Price per unit (numeric)
- **line_total**: Total for this line (numeric)

### Document Content
{{content}}
```

## Variable Placeholders

| Variable | Source | Description |
|----------|--------|-------------|
| `{{content}}` | Document | Full document text content |
| `{{documentType}}` | Metadata | Document type classification |
| `{{taxonPath}}` | Taxonomy | Current taxon being extracted |
| `{{taxonDescription}}` | Taxonomy | Semantic definition of the taxon |
| `{{existingData}}` | Data objects | Already extracted data (JSON) |
| `{{pageContent}}` | Document | Content of specific page |
| `{{exampleValues}}` | Knowledge | Example values from knowledge sets |

## Integration with Knowledge System

Use knowledge items to override prompts per vendor/document type:

```yaml
# Knowledge Item Type for prompt overrides
slug: extraction-prompt-override
name: "Extraction Prompt Override"
options:
  - name: taxonPath
    type: string
    label: "Target Taxon"
    required: true
  - name: customPrompt
    type: text
    label: "Custom Prompt"
    required: true
  - name: examples
    type: text
    label: "Example Values"

# Knowledge Set connecting vendor to custom prompt
knowledgeSets:
  - slug: acme-prompts
    features:
      - slug: acme
        featureTypeRef: org/vendor
        properties: { vendorId: "ACME" }
        uuid: "acme-uuid"
    items:
      - slug: acme-invoice-number
        knowledgeItemTypeRef: org/extraction-prompt-override
        properties:
          taxonPath: "invoice/invoice_number"
          customPrompt: |
            Acme uses format ACME-YYYY-NNNNN.
            Located in top-right corner, bold text.
          examples: "ACME-2024-00123, ACME-2024-00456"
    clauses:
      - features:
          - featureUuid: "acme-uuid"
            positive: true
```

## Prompt Engineering Best Practices

### Be Specific About Format
```markdown
# GOOD
Return the date in ISO 8601 format: YYYY-MM-DD
Example: 2024-03-15

# BAD
Return the date
```

### Provide Examples
```markdown
# GOOD
Extract the invoice number. Examples:
- "INV-2024-001" → INV-2024-001
- "Invoice #12345" → 12345
- "Ref: ABC-789" → ABC-789

# BAD
Extract the invoice number from the document.
```

### Define Null Handling
```markdown
# GOOD
If the field cannot be found in the document, return null.
Do not guess or infer values that are not explicitly stated.

# BAD
Extract all fields. (What if a field is missing?)
```

### Structured Output
```markdown
Return your response as a JSON object:
{
  "invoice_number": "string or null",
  "invoice_date": "YYYY-MM-DD or null",
  "total_amount": number or null,
  "confidence": number between 0 and 1
}
```

## Complete Standalone Example

```yaml
slug: invoice-extraction-prompt
orgSlug: acme-corp
name: "Invoice Extraction Prompt"
type: promptTemplate
description: "Extracts structured data from invoice documents"

metadata:
  systemPrompt: |
    You are a precise document extraction agent.
    Rules:
    1. Extract only explicitly present data
    2. Return null for missing fields
    3. Dates in YYYY-MM-DD format
    4. Amounts as numbers without currency symbols
    5. Preserve original text formatting

  userPromptTemplate: |
    Extract the following from this invoice:

    ## Header
    - invoice_number: Unique invoice ID
    - invoice_date: Issue date (YYYY-MM-DD)
    - due_date: Payment due date (YYYY-MM-DD)

    ## Vendor
    - vendor_name: Legal business name
    - vendor_address: Full address

    ## Line Items (array)
    - description: Item description
    - quantity: Number (numeric)
    - unit_price: Price per unit (numeric)
    - line_total: Line total (numeric)

    ## Totals
    - subtotal: Sum before tax (numeric)
    - tax_amount: Tax (numeric)
    - total_amount: Final total (numeric)

    ## Document
    {{content}}

  modelProvider: anthropic
  modelId: claude-sonnet-4-6
  temperature: 0.1
  maxTokens: 4096
  outputFormat: json
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Prompt too vague | Be specific about format, location, and examples |
| No null handling | Always specify what to do when data is missing |
| Temperature too high | Use 0.0-0.2 for extraction; higher for creative tasks |
| Missing output format | Specify JSON schema or structured format |
| Hardcoded vendor-specific logic | Use knowledge system for per-vendor customization |
| Prompt in module code | Store in skill module files or prompt templates for reusability |
