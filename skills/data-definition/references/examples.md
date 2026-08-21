# data-definition — worked examples

Every YAML block below has been round-tripped through the decoder the API uses, and every formula
parses. Field order follows the canonical push order, so these files stay stable across pull/push.

## A complete invoice definition

```yaml
type: dataDefinition
slug: invoice-extraction
name: Invoice Extraction
version: 1.0.0
orgSlug: acme-corp
description: Structured data captured from supplier invoices.
publicAccess: false
template: false

taxonomyType: CONTENT
enabled: true
externalDataTaxonomyRefs: []

taxons:
  - name: invoice
    externalName: Invoice
    label: Invoice
    valuePath: VALUE_OR_ALL_CONTENT
    group: true
    typeFeatures:
      cardinality: single
    validationRules:
      - name: Invoice has at least one line
        ruleFormula: "!isblank({LineItems})"
        messageFormula: '"An invoice must have at least one line item"'
        overridable: false
    children:
      - name: invoice_number
        externalName: InvoiceNumber
        label: Invoice Number
        taxonType: STRING
        valuePath: VALUE_OR_ALL_CONTENT
        semanticDefinition: |
          The supplier's own invoice number, printed near the top of the page.
          Look for "Invoice #", "Invoice No." or "Reference".
          Do not return a purchase-order number.
        typeFeatures:
          expected: true
        validationRules:
          - name: Invoice number required
            ruleFormula: "!isblank({InvoiceNumber})"
            messageFormula: '"Invoice number is required"'
            detailFormula: '"Every invoice must carry the supplier reference printed at the top of the page."'
            overridable: false

      - name: invoice_date
        externalName: InvoiceDate
        label: Invoice Date
        taxonType: DATE
        valuePath: VALUE_OR_ALL_CONTENT
        semanticDefinition: "The date the invoice was issued, labelled Date or Invoice Date."
        typeFeatures:
          expected: true
          normalizeDate: true
          dateFormat: "yyyy-MM-dd"

      - name: due_date
        externalName: DueDate
        label: Due Date
        taxonType: DATE
        valuePath: VALUE_OR_ALL_CONTENT
        semanticDefinition: "The date payment is due, labelled Due Date or Payment Due."
        typeFeatures:
          normalizeDate: true
          dateFormat: "yyyy-MM-dd"
        validationRules:
          - name: Due date not before invoice date
            conditional: true
            conditionalFormula: "!isblank({DueDate}) && !isblank({InvoiceDate})"
            ruleFormula: "isafterdate({DueDate}, {InvoiceDate}) || {DueDate} = {InvoiceDate}"
            messageFormula: '"Due date must be on or after the invoice date"'
            overridable: false

      - name: payment_terms
        externalName: PaymentTerms
        label: Payment Terms
        taxonType: SELECTION
        valuePath: VALUE_OR_ALL_CONTENT
        semanticDefinition: "The agreed payment terms, usually printed near the totals."
        selectionOptions:
          - id: due-on-receipt
            label: Due on Receipt
            description: Payment is due as soon as the invoice is received.
            lexicalRelations:
              - type: SYNONYM
                value: Immediate
              - type: SYNONYM
                value: Upon Receipt
          - id: net-30
            label: Net 30
            description: Payment is due 30 days after the invoice date.
          - id: net-60
            label: Net 60
            description: Payment is due 60 days after the invoice date.

      - name: vendor
        externalName: Vendor
        label: Vendor
        valuePath: VALUE_OR_ALL_CONTENT
        group: true
        typeFeatures:
          cardinality: single
        children:
          - name: vendor_name
            externalName: VendorName
            label: Vendor Name
            taxonType: STRING
            valuePath: VALUE_OR_ALL_CONTENT
            semanticDefinition: "The supplier's registered business name, near From or Remit To."
            typeFeatures:
              expected: true
          - name: vendor_email
            externalName: VendorEmail
            label: Vendor Email
            taxonType: EMAIL_ADDRESS
            valuePath: VALUE_OR_ALL_CONTENT
            semanticDefinition: "The supplier's billing contact email address."

      - name: line_items
        externalName: LineItems
        label: Line Items
        valuePath: VALUE_OR_ALL_CONTENT
        group: true
        typeFeatures:
          cardinality: multiple
          chunkingStrategy: record
        additionContexts:
          - type: RECORD_DEFINITION
            context: >-
              Each record is one line of the invoice table: a description, a quantity,
              a unit price and a line total.
        children:
          - name: description
            externalName: Description
            label: Description
            taxonType: STRING
            valuePath: VALUE_OR_ALL_CONTENT
            semanticDefinition: "The goods or services described on this line."
            typeFeatures:
              expected: true
          - name: quantity
            externalName: Quantity
            label: Quantity
            taxonType: NUMBER
            valuePath: VALUE_OR_ALL_CONTENT
            semanticDefinition: "The number of units billed on this line."
            validationRules:
              - name: Quantity is positive
                ruleFormula: "{Quantity} > 0"
                messageFormula: '"Quantity must be greater than zero"'
                overridable: true
          - name: unit_price
            externalName: UnitPrice
            label: Unit Price
            taxonType: CURRENCY
            valuePath: VALUE_OR_ALL_CONTENT
            semanticDefinition: "The price of a single unit on this line."
          - name: line_total
            externalName: LineTotal
            label: Line Total
            taxonType: CURRENCY
            valuePath: FORMULA
            semanticDefinition: "{./Quantity} * {./UnitPrice}"

      - name: subtotal
        externalName: Subtotal
        label: Subtotal
        taxonType: CURRENCY
        valuePath: FORMULA
        semanticDefinition: "sum({LineItems/LineTotal})"

      - name: tax_amount
        externalName: TaxAmount
        label: Tax Amount
        taxonType: CURRENCY
        valuePath: VALUE_OR_ALL_CONTENT
        semanticDefinition: "The total tax charged on the invoice."

      - name: total_amount
        externalName: TotalAmount
        label: Total Amount
        taxonType: CURRENCY
        valuePath: VALUE_OR_ALL_CONTENT
        semanticDefinition: "The final amount payable, labelled Total or Amount Due."
        typeFeatures:
          expected: true
          preferTwoDecimalPlaces: true
        validationRules:
          - name: Total matches subtotal plus tax
            conditional: true
            conditionalFormula: "!isblank({Subtotal})"
            ruleFormula: "abs({TotalAmount} - ({Subtotal} + ifnull({TaxAmount}, 0))) < 0.01"
            messageFormula: '"Total does not match subtotal plus tax"'
            detailFormula: 'concat("Expected ", {Subtotal} + ifnull({TaxAmount}, 0), " but the invoice shows ", {TotalAmount})'
            overridable: true
        conditionalFormats:
          - type: backgroundColor
            condition: "{TotalAmount} > 10000"
            properties:
              color: "#FEF3C7"
          - type: icon
            condition: "{TotalAmount} > 50000"
            properties:
              icon: alert-circle
              color: "#B91C1C"
```

Notes on the shape above:

- Everything hangs off one `invoice` root group. That group's data object is what the leaf taxons'
  attributes live on, which is why `{Subtotal}` and `{TaxAmount}` resolve as bare names from a rule
  authored on `total_amount`.
- Every taxon carries `name`, `externalName` and `valuePath`; nothing carries `path`.
- `vendor` is a single-instance container (`typeFeatures.cardinality: single`); `line_items` repeats
  (`cardinality: multiple`) and declares `chunkingStrategy: record` so its `RECORD_DEFINITION`
  context is actually consulted.
- `line_total` is computed per row with `{./Quantity} * {./UnitPrice}`; `subtotal` aggregates across
  rows with `sum({LineItems/LineTotal})`. Both live in `semanticDefinition` under `valuePath: FORMULA`.
- The "at least one line item" rule sits on the `invoice` group, the parent of `line_items` — the
  only place with a data object to evaluate against when there are no line rows at all.

## Fragments

### Single-instance container

```yaml
- name: remit_to
  externalName: RemitTo
  label: Remit To
  valuePath: VALUE_OR_ALL_CONTENT
  group: true
  typeFeatures: { cardinality: single }
  children:
    - name: address
      externalName: RemitToAddress
      label: Address
      taxonType: STRING
      valuePath: VALUE_OR_ALL_CONTENT
      semanticDefinition: "The full postal address payment should be sent to."
      typeFeatures: { longText: true, maxTextRows: 4 }
```

### Repeating group with regex record markers

Use this instead of `RECORD_DEFINITION` when the table has reliable textual boundaries and you would
rather not pay for a chunking call. Record markers are matched from the **start** of a line; section
markers are matched anywhere in a line.

```yaml
- name: line_items
  externalName: LineItems
  label: Line Items
  valuePath: VALUE_OR_ALL_CONTENT
  group: true
  typeFeatures:
    cardinality: multiple
    chunkingStrategy: record
  additionContexts:
    - { type: RECORD_SECTION_STARTER_MARKER, context: "ITEM\\s+DESCRIPTION" }
    - { type: RECORD_START_MARKER,           context: "^\\s*\\d+\\s+" }
    - { type: RECORD_END_MARKER,             context: "^\\s*TOTAL\\b" }
    - { type: RECORD_SECTION_END_MARKER,     context: "SUBTOTAL" }
  children: [...]
```

Do not mix `RECORD_DEFINITION` with the marker types on one taxon — the definition strategy
concatenates every context it finds into its prompt, so stray regexes end up in the instructions.

### A computed field

```yaml
- name: balance_due
  externalName: BalanceDue
  label: Balance Due
  taxonType: CURRENCY
  valuePath: FORMULA
  semanticDefinition: "{TotalAmount} - ifnull({AmountPaid}, 0)"
```

### A field read from document metadata

```yaml
- name: source_file
  externalName: SourceFile
  label: Source File
  taxonType: STRING
  valuePath: METADATA
  metadataValue: FILENAME
```

`semanticDefinition` is ignored for `METADATA` taxons — there is no prompt to write.

### A templated extraction prompt

```yaml
- name: total_amount
  externalName: TotalAmount
  label: Total Amount
  taxonType: CURRENCY
  valuePath: VALUE_OR_ALL_CONTENT
  typeFeatures:
    allowTemplating: true
  semanticDefinition: |
    The final amount payable, in {{ metadata.currency | default('USD') }}.
    {% if external_data.vendor %}Expect the supplier to be {{ external_data.vendor.name }}.{% endif %}
```

Available context: `external_data`, `metadata`, `knowledge`. Undefined values chain to `''` rather
than raising; a template error logs a warning and the raw string is used.

### A cross-field rule authored on a group

Put the rule on the taxon that should own the exception, and reach the other fields by name.

```yaml
- name: totals
  externalName: Totals
  label: Totals
  valuePath: VALUE_OR_ALL_CONTENT
  group: true
  typeFeatures: { cardinality: single }
  validationRules:
    - name: Totals reconcile
      ruleFormula: "abs({GrandTotal} - ({NetTotal} + ifnull({TaxTotal}, 0))) < 0.01"
      messageFormula: '"Grand total does not reconcile with net plus tax"'
      detailFormula: 'concat("Net ", {NetTotal}, " + tax ", ifnull({TaxTotal}, 0), " should equal ", {GrandTotal})'
      overridable: true
  children: [...]     # NetTotal, TaxTotal, GrandTotal
```

A rule on a group runs against that group's data object, so the bare names reach its direct children.
The same rule authored on one of those leaves would also work — its siblings are attributes of the
same data object — but the exception would then surface on that leaf instead of on the group.

### A rule that only applies sometimes

```yaml
validationRules:
  - name: Purchase order required for large invoices
    conditional: true                  # REQUIRED for conditionalFormula to be consulted
    conditionalFormula: "{TotalAmount} > 5000"
    ruleFormula: "!isblank({PurchaseOrderNumber})"
    messageFormula: '"Invoices over 5,000 need a purchase order number"'
    overridable: false
```

### Conditional formatting on a leaf

```yaml
- name: payment_window_days
  externalName: PaymentWindowDays
  label: Payment Window (days)
  taxonType: NUMBER
  valuePath: FORMULA
  semanticDefinition: "daysBetween({InvoiceDate}, {DueDate})"   # (from, to) — negative if reversed
  conditionalFormats:
    - type: backgroundColor
      condition: "{PaymentWindowDays} > 30"
      properties: { color: "#FEF3C7" }
    - type: icon
      condition: "{PaymentWindowDays} > 60"
      properties: { icon: alert-box, color: "#991B1B" }
```

### A SELECTION with a dynamic option list

```yaml
- name: currency
  externalName: Currency
  label: Currency
  taxonType: SELECTION
  valuePath: VALUE_OR_ALL_CONTENT
  useSelectionOptionFormula: true
  selectionOptionFormula: 'serviceBridgeCall("erp-lookup", "currencies", "country", {../Country})'
```

The formula must return `[{ "label": "US Dollar", "value": "USD" }, …]` or a plain string array. The
`{../Country}` reference puts the formula in the reactive graph, so the options refresh when the
sibling changes.

### An event subscription on a group

```yaml
- name: vendor
  externalName: Vendor
  label: Vendor
  valuePath: VALUE_OR_ALL_CONTENT
  group: true
  typeFeatures: { cardinality: single }
  eventSubscriptions:
    - name: derive-region
      on: "changed:dataAttribute:(vendor_city|vendor_state)"
      disabled: false
      script: |
        if (!currentObject) return;
        var city = currentObject.getFirstAttributeValue("vendor_city");
        var state = currentObject.getFirstAttributeValue("vendor_state");
        currentObject.setAttribute("region", city + ", " + state);
  children: [...]     # vendor_city, vendor_state, region
```

Write attributes through `currentObject.setAttribute(name, value)`; there is no `bridge.data.*`.
None of the structural rules — group taxons only, unique `name`, non-empty `script`, an `on` that
compiles as a regex — are checked at save time. Break one and the handler is simply never indexed,
so it never fires and nothing tells you why.
