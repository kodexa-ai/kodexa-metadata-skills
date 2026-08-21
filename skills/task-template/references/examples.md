# Examples

Three complete, copy-pasteable files. Org is `acme-corp`; adjust `orgSlug` and the refs.

## 1 — Minimal review task

`task-templates/invoice-review.yaml`

```yaml
type: taskTemplate
orgSlug: acme-corp
slug: invoice-review
name: "Invoice Review"
title: "Invoice Review"
description: "A reviewer checks an extracted invoice and approves or rejects it."
template: false
deprecated: false
metadata:
  teamSlug: finance-reviewers
  priority: 2
  properties:
    hideDefaultCancelButton: true
  actions:
    - slug: approve
      label: "Approve"
      properties:
        statusSlug: approved
        icon: check
        color: green
        shortcut: "a"
    - slug: reject
      label: "Reject"
      properties:
        statusSlug: rejected
        icon: close
        color: red
        shortcut: "r"
        requireComment: true
        commentPrompt: "Why is this invoice being rejected?"
```

```bash
kdx apply -f task-templates/invoice-review.yaml
```

An activity plan reaches the Approve button with `dependsOn: ["review:approve"]`, where `review` is the
`CREATE_TASK` step whose `taskTemplateRef: invoice-review`.

## 2 — Form-driven approval with gating, locking and an agent shortcut

`task-templates/purchase-order-approval.yaml`

```yaml
type: taskTemplate
orgSlug: acme-corp
slug: purchase-order-approval
name: "Purchase Order Approval"
title: "Purchase Order Approval"
description: "Approver signs off a purchase order against the vendor contract."
metadata:
  teamSlug: procurement
  priority: 1

  properties:
    hideDefaultSaveButton: true
    hideDefaultCancelButton: true
    lockStatusSlug: signed-off
    collapseTaskPanelsOnOpen: true
    visiblePanels:
      documentStores: true
      exceptions: true
      taskNotes: true
    enableChat: true
    saveShortcut: "meta+s"
    showAlert: true
    alertType: WARNING
    alertTitle: "Check the vendor before signing off"
    alertBody: "Approval here releases the order to the supplier."

  forms:
    - dataFormRef: "${org}/purchase-order-form"
      availablePanels:
        documentStores: true
        exceptions: true
        properties: false

  actions:
    - slug: sign-off
      label: "Sign off"
      properties:
        statusSlug: signed-off
        icon: check-decagram
        color: green
        shortcut: "s"
        gatedByCompleteness: true
        onlyEnabledIfNoOpenExceptions: true
        lockTask: true
        lockDocumentFamily: false
        attributes:
          - taxon: { taxonomySlug: purchase-order, taxonPath: order/approval_status }
            valueType: string
            value: approved
          - taxon: { taxonomySlug: purchase-order, taxonPath: order/approved_by }
            valueType: metadata
            metadataKey: currentUserEmail
          - taxon: { taxonomySlug: purchase-order, taxonPath: order/approved_at }
            valueType: metadata
            metadataKey: currentTimestamp
        takeOwnershipForPaths:
          - { taxonomySlug: purchase-order, taxonPath: order/total }

    - slug: query-vendor
      label: "Query vendor"
      properties:
        statusSlug: awaiting-vendor
        icon: email-alert-outline
        color: orange              # a CSS colour — see actions.md
        requireComment: true
        commentPrompt: "What are you asking the vendor?"

  agentShortcuts:
    - id: compare-to-contract
      label: "Compare to contract"
      icon: file-compare
      prompt: "Compare this purchase order against the vendor contract and list any line that differs."

  chatPrompt:
    enabled: true
    prompt: "Help me approve {taskTitle}. The attached documents are {documentFamilyPaths}."
```

```bash
kdx apply -f task-templates/purchase-order-approval.yaml
```

Notes on this file:

- `${org}/` is resolved by `kdx` before the push. `${orgSlug}` would also work here — `dataFormRef`
  is the one field the server expands it in — but `kdx sync pull` writes `${org}`, so `${org}` is
  the spelling that round-trips. The referenced data form must also be bound to the project the
  task lives in, or the form pane silently never appears.
- `lockStatusSlug: signed-off` makes the toolbar's Lock button perform that transition. Because
  `sign-off` also targets `signed-off`, the two routes converge.
- `lockTask: true` with `lockDocumentFamily: false` locks the task while leaving the documents editable
  for whatever runs next, regardless of what the `signed-off` status itself is configured to do.

## 3 — Document intake with AI naming

`task-templates/contract-intake.yaml`

```yaml
type: taskTemplate
orgSlug: acme-corp
slug: contract-intake
name: "Contract Intake"
title: "Contract Intake"
description: "Reviewer uploads a signed contract and its supporting receipts, then files it."
metadata:
  teamSlug: legal-ops
  priority: 3

  documentFamilyGroups:
    - name: "Signed Contract"
      notes: "The executed contract PDF."
      documentFamilyFilter: "*.pdf"
      maxHits: 1
      sort: "created:desc"
      automaticallyAdd: true
      editable: false
      hardMaxPages: 300

    - name: "Supporting Receipts"
      notes: "Anything the counterparty sent alongside."
      documentFamilyFilter: "*.pdf OR *.png OR *.jpg"
      uploadOnly: true
      uniqueFilenames: true
      maxSize: 10485760
      maxPages: 50

  aiNaming:
    enabled: true
    prompt: >-
      Produce a short task title for a {templateName} covering
      {documentFamilyCount} document(s): {documentFamilyPaths}.
      Use the counterparty from {metadata:party.name} when it is available.

  properties:
    subTaskOnly: false
    enableSummarization: true
    summarizePrompt: "Summarise the key obligations and dates in this contract."

  actions:
    - slug: file-contract
      label: "File contract"
      properties:
        statusSlug: filed
        icon: archive-arrow-down-outline
```

```bash
kdx apply -f task-templates/contract-intake.yaml
```

Notes on this file:

- `documentFamilyFilter` is an extension whitelist. `"path:contracts/*"` or any glob would leave the
  extension list empty and silently allow every file type.
- `maxPages: 50` warns and lets the user continue; `hardMaxPages: 300` rejects the upload outright.
- `{metadata:party.name}` resolves on the intake and create-task paths but is **always empty** when the
  same template is driven by an activity plan's `CREATE_TASK` step. Keep the prompt readable when it
  renders to nothing.
- `{templateName}` reads the `title` field here — which is why `title` is set alongside `name`.

## Applying a file the CLI pulled

Files produced by `kdx sync pull` are untyped and org-agnostic — the source org slug is rewritten to
`${org}` everywhere it appears. Supply the identity on the command line:

```bash
kdx apply -f pulled/invoice-review.yaml --type task-template --org-slug acme-corp
```

To push a whole tree, keep the files under a `task-templates/` directory and name a target from your
sync config:

```bash
kdx sync push --target <target-name> --dry-run
kdx sync push --target <target-name>
```

Push resolves `${org}/` against the destination org and **fails the push** if any `${org}` is left
unresolved. Remember that outside `metadata.properties`, removing a key from a file is not pushed as a
deletion.
