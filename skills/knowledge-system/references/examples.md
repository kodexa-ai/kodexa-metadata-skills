# Worked examples

Every slug below is real: each feature slug is the SHA-256 of that feature's canonical `properties`
JSON. If you change a property value, recompute — see `SKILL.md`.

Scenario: in org `my-org`, invoices from vendor Acme need a custom extraction prompt for the invoice
number, and Globex invoices need one too — but never on drafts.

```
knowledge/
  knowledge-feature-types/
    vendor.yml
    document-type.yml
  knowledge-feature-instances/
    vendor-acme.yml
    vendor-globex.yml
    document-type-invoice.yml
    document-type-draft.yml
  knowledge-sets/
    acme-invoice-rules.yml
    attachments/
      acme-header.png
  knowledge-items/
    acme-invoice-number-prompt.yml
```

---

## 1. Feature types

```yaml
# knowledge-feature-types/vendor.yml
type: knowledgeFeatureType
orgSlug: my-org
slug: vendor
name: "Vendor"
description: "The supplier a document came from"
active: true

# Core options — these become the instance `properties` that hash into its slug.
# Keep this set small and stable; changing a value later means a new feature.
options:
  - name: vendorId
    type: string
    label: "Vendor ID"
    description: "Stable identifier for the vendor"
    required: true

# Extended options — never hashed, freely editable afterwards.
extendedOptions:
  - name: displayName
    type: string
    label: "Display Name"
  - name: website
    type: string
    label: "Website"

labelJsonPath: '$._extendedProperties.displayName & " (" & vendorId & ")"'
useJSONata: true
color: "#3B82F6"
icon: "building"
```

```yaml
# knowledge-feature-types/document-type.yml
type: knowledgeFeatureType
orgSlug: my-org
slug: document-type
name: "Document Type"
active: true
options:
  - name: documentType
    type: select
    label: "Document Type"
    required: true
    possibleValues:
      - { value: invoice, label: "Invoice" }
      - { value: purchase-order, label: "Purchase Order" }
      - { value: draft, label: "Draft" }
labelJsonPath: "$.documentType"
```

---

## 2. Feature instances

The slug is `{featureTypeSlug}-{sha256(canonicalJSON(properties))[:32]}`. You may omit `slug:`
entirely on a standalone file — the CLI recomputes it and the server computes it — but write it if
you need to name it in an expression.

```yaml
# knowledge-feature-instances/vendor-acme.yml
type: knowledgeFeature
orgSlug: my-org
slug: vendor-45a62423116103b32d28ee02507f2eeb   # sha256 of {"vendorId":"ACME-001"}
featureTypeRef: my-org/vendor
properties:
  vendorId: "ACME-001"          # quoted: an unquoted number hashes differently
extendedProperties:
  displayName: "Acme Corporation"
  website: "https://acme.kodexa.example.com"
active: true
```

```yaml
# knowledge-feature-instances/vendor-globex.yml
type: knowledgeFeature
orgSlug: my-org
slug: vendor-fc0427f8993d7dcaa1156510165a4f21   # sha256 of {"vendorId":"GLOBEX-002"}
featureTypeRef: my-org/vendor
properties:
  vendorId: "GLOBEX-002"
extendedProperties:
  displayName: "Globex"
active: true
```

```yaml
# knowledge-feature-instances/document-type-invoice.yml
type: knowledgeFeature
orgSlug: my-org
slug: document-type-6e49d6c90d404a72f018e53184fa7d5d   # sha256 of {"documentType":"invoice"}
featureTypeRef: my-org/document-type
properties:
  documentType: "invoice"
active: true
```

```yaml
# knowledge-feature-instances/document-type-draft.yml
type: knowledgeFeature
orgSlug: my-org
slug: document-type-827aa3607808acfaa546f6e377922441   # sha256 of {"documentType":"draft"}
featureTypeRef: my-org/document-type
properties:
  documentType: "draft"
active: true
```

Verify any of them:

```bash
printf '%s' '{"documentType":"draft"}' | shasum -a 256 | cut -c1-32
# 827aa3607808acfaa546f6e377922441
```

---

## 3. The knowledge set — standalone style

Items live in their own files; the set carries only the rule and its attachments.

```yaml
# knowledge-sets/acme-invoice-rules.yml
type: knowledgeSet
orgSlug: my-org
slug: acme-invoice-rules
name: "Acme & Globex invoice rules"
description: "Vendor-specific extraction guidance for invoices"
setType: extraction        # set it explicitly, or `type` leaks into set_type
status: ACTIVE             # NOT `active: true` — that field does not exist here
priority: 5                # bake order only; does not affect matching
# projectSlug: invoice-processing   # uncomment to scope this set to one project

attachments:
  - attachmentId: acme-invoice-header
    attachmentPath: attachments/acme-header.png    # a file, relative to this YAML

featureExpression:
  type: OR
  children:
    - type: AND
      children:
        - { type: FEATURE, slug: vendor-45a62423116103b32d28ee02507f2eeb }
        - { type: FEATURE, slug: document-type-6e49d6c90d404a72f018e53184fa7d5d }
    - type: AND
      children:
        - { type: FEATURE, slug: vendor-fc0427f8993d7dcaa1156510165a4f21 }
        - type: NOT
          children:
            - { type: FEATURE, slug: document-type-827aa3607808acfaa546f6e377922441 }
```

The item, using the shipped `PROMPT_OVERRIDE` item type so the extraction pipeline honours it. A
standalone `knowledgeItem` needs `projectSlug` for `kdx apply`:

```yaml
# knowledge-items/acme-invoice-number-prompt.yml
type: knowledgeItem
orgSlug: my-org
projectSlug: invoice-processing
slug: acme-invoice-number-prompt
title: "Acme invoice number"
description: "Where Acme prints the invoice number"
knowledgeItemTypeRef: kodexa/taxon-semantic-definition-customization
knowledgeSetSlug: acme-invoice-rules       # resolved against orgSlug above
active: true
sequenceOrder: 1
properties:
  taxonomyAndTaxon: "my-org/invoice//invoice/invoiceNumber"
  replace: false
  prompt: |
    Acme prints the invoice number in the top-right header block, in the
    form ACME-YYYY-NNNNN. The reference under the line-item table is a
    purchase order number, not the invoice number.

    ![Acme header](attachment://acme-invoice-header)
```

Apply it:

```bash
kdx apply -f knowledge/knowledge-feature-types/vendor.yml
kdx apply -f knowledge/knowledge-feature-types/document-type.yml
kdx apply -f knowledge/knowledge-feature-instances/vendor-acme.yml
kdx apply -f knowledge/knowledge-feature-instances/document-type-invoice.yml
kdx apply -f knowledge/knowledge-sets/acme-invoice-rules.yml       # uploads the attachment too
kdx apply -f knowledge/knowledge-items/acme-invoice-number-prompt.yml
```

Feature types before feature instances before sets before items.

---

## 4. The same set — inline style

One file, no `projectSlug` needed for the items. The trade-off is the reconcile: **every** item the
set should have must be in the array, because any server-side item whose slug is missing from it is
deleted.

```yaml
type: knowledgeSet
orgSlug: my-org
slug: acme-invoice-rules
name: "Acme & Globex invoice rules"
setType: extraction
status: ACTIVE
priority: 5

# Created if the slug does not already exist; existing features are left alone.
# NOTE: this does not populate the set's Feature Palette on a CLI push.
features:
  - slug: vendor-45a62423116103b32d28ee02507f2eeb
    featureTypeRef: my-org/vendor
    properties:
      vendorId: "ACME-001"
    extendedProperties:
      displayName: "Acme Corporation"
    active: true

# `knowledgeItems`, never `items`. Only slug/title/description/properties/
# active/sequenceOrder survive; attachmentId and attachmentPath are dropped.
knowledgeItems:
  - slug: acme-invoice-number-prompt
    title: "Acme invoice number"
    knowledgeItemTypeRef: kodexa/taxon-semantic-definition-customization
    sequenceOrder: 1
    active: true
    properties:
      taxonomyAndTaxon: "my-org/invoice//invoice/invoiceNumber"
      replace: false
      prompt: |
        Acme prints the invoice number in the top-right header block.

featureExpression:
  type: AND
  children:
    - { type: FEATURE, slug: vendor-45a62423116103b32d28ee02507f2eeb }
    - { type: FEATURE, slug: document-type-6e49d6c90d404a72f018e53184fa7d5d }
```

---

## 5. A custom item type

Only worth writing when you have code that consumes it — processing dispatches on the item type
slug, so a new slug does nothing on its own. This one shows `markdown`, a branching `showIf`, and a
list of structured rows.

```yaml
type: knowledgeItemType
orgSlug: my-org
slug: line-item-locator
name: "Line item locator"
description: "How to find the line-item table on this vendor's invoices"
supportsAttachment: false

options:
  - name: mode
    type: select
    label: "Capture mode"
    required: true
    possibleValues:
      - { value: line, label: "Line marker" }
      - { value: column, label: "Column marker" }

  - name: lineMarker
    type: code
    label: "Line regex"
    showIf: "this.mode === 'line'"     # `this` is the item's properties map
    properties:
      language: javascript

  - name: columnHeader
    type: string
    label: "Column header text"
    showIf: "this.mode === 'column'"

  - name: notes
    type: text
    label: "Notes"
    properties:
      lines: 6                          # without this, `text` is a single-line input

  - name: exclusions
    type: list
    listType: object
    label: "Rows to skip"
    properties:
      useTable: true
    groupOptions:
      - { name: pattern, type: code, required: true }
      - { name: reason, type: string }

  - name: guidance
    type: markdown
    label: "Guidance"
    properties:
      height: 400px
```

A flag-only type — the item's presence is the whole signal:

```yaml
type: knowledgeItemType
orgSlug: my-org
slug: no-invoice-date
name: "No invoice date"
description: "This vendor never prints an invoice date"
options: []          # bare empty list, not {options: []}
```

---

## 6. Two organising patterns

**One set per rule.** Good when the item takes no parameters: every affected vendor is a branch of
one `OR`, and the set holds a single shared item with `properties: {}`.

```yaml
type: knowledgeSet
orgSlug: my-org
slug: rule-no-invoice-date
name: "Rule: no invoice date"
setType: validation
status: ACTIVE
featureExpression:
  type: OR
  children:
    - { type: FEATURE, slug: vendor-45a62423116103b32d28ee02507f2eeb }
    - { type: FEATURE, slug: vendor-fc0427f8993d7dcaa1156510165a4f21 }
knowledgeItems:
  - slug: rule-no-invoice-date-apply
    title: "No invoice date"
    knowledgeItemTypeRef: my-org/no-invoice-date
    properties: {}
    active: true
```

**One set per vendor.** Good when the items are parameterised: the set is scoped by that vendor's
feature and holds one item per rule, each with vendor-specific `properties`. A vendor with both
kinds of rule appears as a branch in several rule-sets *and* owns one parameterised set.

---

## 7. Embedding in a project template

Different schema, different owner — this block lives inside a project-template file, not a
standalone knowledge set. Note `active` and `showOnNewProject` exist **here** and nowhere else, and
the item key is still `knowledgeItems`.

```yaml
knowledgeSets:
  - slug: acme-invoice-rules
    name: "Acme invoice rules"
    setType: extraction
    active: true
    showOnNewProject: false
    features:
      - slug: vendor-45a62423116103b32d28ee02507f2eeb
        featureTypeRef: my-org/vendor
        properties:
          vendorId: "ACME-001"
        active: true
    knowledgeItems:
      - slug: acme-invoice-number-prompt
        title: "Acme invoice number"
        knowledgeItemTypeRef: kodexa/taxon-semantic-definition-customization
        sequenceOrder: 1
        properties:
          taxonomyAndTaxon: "my-org/invoice//invoice/invoiceNumber"
          prompt: "Top-right header block, form ACME-YYYY-NNNNN."
          replace: false
    featureExpression:
      type: FEATURE
      slug: vendor-45a62423116103b32d28ee02507f2eeb
```

See the `project-template` skill for the surrounding file.

---

## 8. Common mistakes

| Mistake | What happens |
|---|---|
| Invented human-friendly feature slug (`vendor-acme`) | The expression resolves nothing; the set never matches. Slugs are hashes. |
| `items:` instead of `knowledgeItems:` | Silently ignored — the set is created with no items. |
| Partial `knowledgeItems:` array | Every server item not in the array is **deleted**. |
| `active: true` on a knowledge set | Ignored; the set keeps whatever `status` it had. |
| `setType:` omitted | `set_type` is persisted as the envelope value, e.g. `"knowledgeSet"`. |
| `featureUuid` / `clauses` in a set file | Not knowledge-set fields; dropped. Use `featureExpression`. |
| `options:` wrapped as `{options: [...]}` | Editor shows "No configuration options available" and hides every field. |
| `type: text` for a prompt body | One-line input, no `attachment://` rendering, no server-side image repair. Use `markdown`. |
| Underscore in a slug | `400 invalid slug` — only `[a-z0-9-]`. |
| Editing a feature's `properties` after create | Slug stays at the old hash forever. Delete and recreate. |
| Custom item type expected to change extraction | Nothing dispatches on it. Use a `kodexa/*` type, or write the consumer. |
| Standalone `knowledgeItem` with no `projectSlug` | `kdx apply` refuses the file. |
| `attachment://` pointing at a per-item attachment | Resolves only against **set-level** attachments. |
