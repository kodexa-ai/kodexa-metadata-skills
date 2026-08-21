# Complete form examples

Each file below is a whole resource, ready to drop in `data-forms/<slug>.yaml`
and apply with `kdx sync push`. Taxon paths must exist in the project's data
definition — the form does not create them.

---

## 1. Minimal V2 form

The smallest thing that renders and is reachable.

```yaml
type: dataForm
slug: invoice-header
name: Invoice Header
description: Minimal header review panel
version: "2"
entrypoints:
  - documentFamily
nodes:
  - component: v2:panel
    props:
      title: Invoice Header
      groupTaxon: invoice
    children:
      - component: v2:attributeEditor
        props:
          tagPath: invoice/invoice_number
          label: Invoice Number
      - component: v2:attributeEditor
        props:
          tagPath: invoice/invoice_date
          label: Invoice Date
```

---

## 2. Header plus line-item grid, with exceptions

The common review-form shape: a two-column header, a divider, a grid over the
repeating group, and an exceptions block that follows the reviewer down the page.

```yaml
type: dataForm
slug: invoice-review
name: Invoice Review
description: Review and correct extracted invoice data
version: "2"
publicAccess: false
deprecated: false
template: false
entrypoints:
  - documentFamily
nodes:
  - component: v2:panel
    props:
      title: Invoice
      icon: mdi-file-document-outline
      iconColor: blue
      description: Check the header, then reconcile the line items against the total.
      groupTaxon: invoice
    children:
      - component: v2:exceptions
        props:
          title: Needs attention
          sticky: true

      - component: v2:row
        props:
          gap: 16
        children:
          - component: v2:col
            props:
              span: 6
            children:
              - component: v2:attributeEditor
                props:
                  tagPath: invoice/invoice_number
                  label: Invoice Number
              - component: v2:attributeEditor
                props:
                  tagPath: invoice/invoice_date
                  label: Invoice Date
                  editorOptions:
                    showCalendarPopup: true
          - component: v2:col
            props:
              span: 6
            children:
              - component: v2:attributeEditor
                props:
                  tagPath: invoice/vendor/name
                  label: Vendor
              - component: v2:attributeEditor
                props:
                  tagPath: invoice/purchase_order_number
                  label: PO Number

      - component: v2:divider

      - component: v2:alert
        props:
          type: info
          strong: "Tip:"
          text: Select text in the document and use the copy icon to fill a field.

      - component: v2:grid
        props:
          groupTaxon: invoice/line_items
          title: Line Items
          limitColumns:
            - invoice/line_items/description
            - invoice/line_items/quantity
            - invoice/line_items/unit_price
            - invoice/line_items/line_total
          autoHeight: true
          fitColumns: true
          sort:
            - tagPath: invoice/line_items/line_total
              direction: desc

      - component: v2:row
        props:
          gap: 16
        children:
          - component: v2:col
            props:
              span: 4
            children:
              - component: v2:attributeEditor
                props:
                  tagPath: invoice/subtotal
                  label: Subtotal
          - component: v2:col
            props:
              span: 4
            children:
              - component: v2:attributeEditor
                props:
                  tagPath: invoice/tax_amount
                  label: Tax
          - component: v2:col
            props:
              span: 4
            children:
              - component: v2:attributeEditor
                props:
                  tagPath: invoice/total_amount
                  label: Total
                  readonly: true
```

---

## 3. Repeating group as stacked panels

Use a stacked panel (not a grid) when each instance needs its own multi-field
layout. `groupActions` gives the reviewer add/remove controls, and `v2:taxonNav`
renders a chip strip that jumps to each instance.

```yaml
type: dataForm
slug: purchase-order-review
name: Purchase Order Review
description: Review purchase order deliveries line by line
version: "2"
entrypoints:
  - documentFamily
nodes:
  - component: v2:panel
    props:
      title: Purchase Order
      groupTaxon: purchase_order
    children:
      - component: v2:taxonNav
        props:
          taxon: purchase_order/deliveries
          labelFrom: purchase_order/deliveries/delivery_reference
          emptyLabel: Unlabelled delivery

      - component: v2:panel
        props:
          title: Deliveries
          groupTaxon: purchase_order/deliveries
          groupDisplay: stacked
          groupActions: true
          groupTintBy: purchase_order/deliveries/status
        children:
          - component: v2:row
            props:
              gap: 16
            children:
              - component: v2:col
                props:
                  span: 6
                children:
                  - component: v2:attributeEditor
                    props:
                      tagPath: purchase_order/deliveries/delivery_reference
                      label: Reference
              - component: v2:col
                props:
                  span: 6
                children:
                  - component: v2:attributeEditor
                    props:
                      tagPath: purchase_order/deliveries/status
                      label: Status
                      editorOptions:
                        displayAsRadio: true
          - component: v2:grid
            props:
              groupTaxon: purchase_order/deliveries/items
              title: Items
              autoHeight: true
```

Colours for `groupTintBy` and for the nav chips come from the taxonomy's
conditional formats, not from the form — set them in the data definition.

---

## 4. Tabs and a conditional section

`v2:tabs` builds one tab per **child**; the label is the child's `props.title`.
`mustView` blocks a completeness-gated task action until the reviewer has opened
those tabs. `v2:tabs` renders no heading of its own — wrap it in a titled
`v2:panel` if you want one.

```yaml
type: dataForm
slug: contract-review
name: Contract Review
version: "2"
entrypoints:
  - documentFamily
nodes:
  - component: v2:tabs
    props:
      mustView: [1]
    children:
      - component: v2:panel
        props:
          title: Parties
          groupTaxon: contract
        children:
          - component: v2:attributeEditor
            props:
              tagPath: contract/counterparty_name
              label: Counterparty
      - component: v2:panel
        props:
          title: Terms
          groupTaxon: contract
        children:
          - component: v2:attributeEditor
            props:
              tagPath: contract/term_months
              label: Term (months)
          - component: v2:alert
            if: ctx.dataObjects?.some(o => o.path === 'contract/renewal')
            props:
              type: warning
              strong: "Auto-renewal:"
              text: This contract renews unless notice is served.
```

---

## 5. External lookup with a service bridge

`params` bound to `null` until the inputs are ready keeps the call from firing
prematurely; children see the response on `ctx.$bridgeResult`.

```yaml
type: dataForm
slug: vendor-lookup
name: Vendor Lookup
version: "2"
entrypoints:
  - documentFamily
nodes:
  - component: v2:panel
    props:
      title: Vendor
      groupTaxon: invoice
    children:
      - component: v2:attributeEditor
        props:
          tagPath: invoice/vendor/tax_id
          label: Tax ID
      - component: v2:serviceBridgeView
        props:
          bridgeRef: service-bridge://acme-corp/vendor-directory
          endpoint: lookup-by-tax-id
          transform: "$.records"
        bindings:
          params: >-
            (() => {
              const inv = ctx.dataObjects?.find(o => o.path === 'invoice');
              const id = inv?.attributes?.find(a => a.path === 'invoice/vendor/tax_id')?.resolvedValue;
              return id ? { taxId: id } : null;
            })()
        children:
          - component: v2:label
            props:
              text: Matching vendors
          - component: v2:dataTable
            props:
              columns:
                - field: name
                  title: Vendor
                - field: city
                  title: City
              emptyMessage: No vendor matched that tax ID
              filterable: true
              pageSize: 10
            bindings:
              rows: ctx.$bridgeResult
```

---

## 6. Keyboard shortcuts, tab order and a recalculation trigger

Note the two script shapes: the shortcut script is an arrow, the trigger script
**must** be a `function` expression or it silently does nothing.

```yaml
type: dataForm
slug: invoice-fast-entry
name: Invoice Fast Entry
version: "2"
editable: true
entrypoints:
  - documentFamily
nodes:
  - component: v2:panel
    props:
      title: Invoice
      groupTaxon: invoice
    children:
      - component: v2:attributeEditor
        props: { tagPath: invoice/invoice_number, label: Invoice Number }
      - component: v2:attributeEditor
        props: { tagPath: invoice/invoice_date, label: Invoice Date }
      - component: v2:attributeEditor
        props: { tagPath: invoice/subtotal, label: Subtotal }
      - component: v2:attributeEditor
        props: { tagPath: invoice/tax_amount, label: Tax }
      - component: v2:attributeEditor
        props: { tagPath: invoice/total_amount, label: Total, readonly: true }
scripts:
  rotateRight: |
    (ctx, bridge) => bridge.navigation.rotatePage("right")
  nextPage: |
    (ctx, bridge) => bridge.navigation.nextPage()
  clearField: |
    (ctx, bridge) => bridge.data.clearFocusedValue()
  recalcTotal: |
    function (ctx, bridge) {
      const sub = Number(ctx.formValues["invoice/subtotal"] || 0);
      const tax = Number(ctx.formValues["invoice/tax_amount"] || 0);
      ctx.formValues["invoice/total_amount"] = (sub + tax).toFixed(2);
      return ctx.formValues;
    }
bridge:
  permissions:
    - data:read
    - data:write
    - navigation
scriptTriggers:
  - script: recalcTotal
    triggerOn:
      - invoice/subtotal
      - invoice/tax_amount
tabOrderInitial: first
tabOrder:
  - path: invoice/invoice_number
  - path: invoice/invoice_date
  - path: invoice/subtotal
  - path: invoice/tax_amount
shortcuts:
  - key: "alt+r"
    description: Rotate page right
    group: document
    scriptRef: rotateRight
  - key: "alt+n"
    description: Next page
    group: navigation
    scriptRef: nextPage
  - key: "alt+c"
    description: Clear the focused field
    group: data-entry
    scriptRef: clearField
```

Declaring `tabOrder` here means `invoice/total_amount` is removed from the tab
chain — that is the intent (it is read-only), but it is also the trap: every
field you forget to list becomes untabbable.

---

## Applying and binding

```bash
kdx sync push --dry-run     # always dry-run first
kdx sync push
```

Data forms are org-scoped. A pushed form is not visible inside a project until
it is bound there — add it under the project's `linked['data-form']` entries in
the manifest, or bind it from Project Settings → Resources. A task template then
names it in `forms[].dataFormRef` as `acme-corp/invoice-review` — the plain
`<orgSlug>/<slug>` ref, not the `data-form://` URI the resolver and the binding
manifest use.
