# Complete ActivityPlan examples

Five plans that start cleanly. Every SCRIPT declares its actions and returns one; every BRIDGE_CALL
literal is single-quoted JSONata; every await dep carries a guard or is an `ANY_BRANCH` join.

## 1. Linear pipeline

Extract, classify, post to an external system.

```yaml
slug: invoice-intake
orgSlug: acme-corp
name: Invoice Intake
type: activity-plan
description: Extract an invoice, decide whether it is high value, and post it.

inputOptions:
  - { name: vendorId, label: Vendor, type: string, required: true }

inputsSchema:
  type: object
  required: [vendorId]
  properties:
    vendorId: { type: string }

defaultTitleTemplate: "Invoice {{ .inputs.vendorId }} — {{ .documentCount }} document(s)"

steps:
  - slug: extract
    type: EXECUTION
    name: Extract invoice
    moduleRef: "kodexa/fast-pdf-model"
    options: { inferenceMode: layout }
    setDocumentStatus: extracted

  - slug: grade
    type: SCRIPT
    dependsOn: [extract]
    scriptBody: |
      var total = Number(families[0].metadata.total || 0);
      log.info('total=' + total);
      return { action: total > 10000 ? 'high-value' : 'standard' };
    scriptActions:
      - { name: High value, slug: high-value }
      - { name: Standard,   slug: standard }

  - slug: post-invoice
    type: BRIDGE_CALL
    dependsOn: [grade]
    serviceBridgeRef: erp
    endpointName: submit-invoice
    requestBody:
      vendor:   "inputs.vendorId"          # path lookup
      grade:    "steps.grade.action"       # the action the SCRIPT emitted
      source:   "'kodexa'"                 # literal — single quotes required
    treatAsError: "$._statusCode >= 300"
    timeoutSeconds: 30
    setDocumentStatus: posted
```

`steps.grade.action` works because a SCRIPT publishes its returned `action`. `steps.extract.total`
would **not** work — EXECUTION steps publish nothing.

## 2. Conditional human review

Route to a review task only when the extraction looks weak, then post either way.

```yaml
steps:
  - slug: extract
    type: EXECUTION
    moduleRef: "kodexa/fast-pdf-model"

  - slug: score
    type: SCRIPT
    dependsOn: [extract]
    scriptBody: |
      var c = Number(families[0].metadata.confidence || 0);
      return { action: c < 0.85 ? 'needs-review' : 'clean' };
    scriptActions:
      - { name: Needs review, slug: needs-review }
      - { name: Clean,        slug: clean }

  - slug: review
    type: CREATE_TASK
    dependsOn: ["score:needs-review"]      # only when the run scored needs-review
    taskTemplateRef: invoice-review        # bare slug, resolved in the plan's own org
    taskStatusSlug: to-do                  # always set it — there is no fallback
    taskData:
      title: "Review — ${activity.title}"
      priority: 2

  - slug: post-invoice
    type: BRIDGE_CALL
    dependsOn: [extract, "review?"]        # await: proceed whether or not review ran
    conditionExpr: "true"                  # an await dep REQUIRES a guard; this is "always proceed"
    serviceBridgeRef: erp
    endpointName: submit-invoice
    requestBody:
      vendor: "inputs.vendorId"
      # `steps` carries COMPLETED steps only, so `review` is ABSENT when it was NOT_TAKEN — and
      # an unmatched path FAILS the step. Guard the full path: $exists(steps.review) is not
      # enough, because a task that completed without an action token leaves steps.review = {}.
      reviewed: "$exists(steps.review.completedActionUuid) ? steps.review.completedActionUuid : ''"
```

Route on the review outcome with `dependsOn: ["review:approved"]` only when `approved` is an action
on the **invoice-review task template**, or a DONE-typed task status slug in the org. An `actions:`
array on the CREATE_TASK step itself will not satisfy that edge.

## 3. Per-document routing with an ANY_BRANCH re-join

Each document is classified on its own, takes its own branch, and re-joins.

```yaml
steps:
  - slug: classify
    type: SCRIPT
    perDocument: true                      # one run per document
    scriptBody: |
      var t = (families[0].metadata.docType || '').toLowerCase();
      return { action: t === 'receipt' ? 'receipt' : 'invoice' };
    scriptActions:
      - { name: Receipt, slug: receipt }
      - { name: Invoice, slug: invoice }

  - slug: receipt-extract
    type: EXECUTION
    dependsOn: ["classify:receipt"]
    moduleRef: "acme-corp/receipt-extractor"
    setDocumentStatus: receipt-extracted

  - slug: invoice-extract
    type: EXECUTION
    dependsOn: ["classify:invoice"]
    moduleRef: "acme-corp/invoice-extractor"
    setDocumentStatus: invoice-extracted

  - slug: finalize
    type: SCRIPT
    perDocument: true
    joinPolicy: ANY_BRANCH                 # converge each document via the branch it took
    dependsOn: ["receipt-extract?", "invoice-extract?"]   # await-only — required deps converge nothing
    scriptBody: "return { action: 'done' };"
    scriptActions: [{ name: Done, slug: done }]
    setDocumentStatus: ready-for-review
```

Routing switched on because `classify` is perDocument, declares actions, and is the target of
`classify:receipt` / `classify:invoice`. Nothing sets a routing flag. Declaring an action with no
outgoing branch would be warned as `action-unrouted`, and documents taking it would stop there.

## 4. Skip documents that are already done

A perDocument root gated on the document's current status. `document.*` is only populated under
per-document routing, so this pattern belongs in a plan shaped like example 3.

```yaml
  - slug: prepare
    type: EXECUTION
    perDocument: true
    moduleRef: "kodexa/fast-pdf-model"
    conditionExpr: "$not(document.status in ['pending-review', 'reviewed', 'completed'])"
```

Documents whose status matches become `NOT_TAKEN` at `prepare`, and the NOT_TAKEN cascade skips the
subtree below it. For a *visible* two-way skip, use a perDocument SCRIPT router instead and gate each
branch with an action-qualified dep:

```yaml
  - slug: triage
    type: SCRIPT
    perDocument: true
    scriptBody: |
      var d = documents.get(families[0].id);
      var done = d && ['pending-review', 'reviewed', 'completed'].indexOf(d.status) >= 0;
      return { action: done ? 'skip-to-review' : 'process' };
    scriptActions:
      - { name: Process,        slug: process }
      - { name: Skip to review, slug: skip-to-review }
```

## 5. LLM classification with bridge enrichment

```yaml
steps:
  - slug: enrich-and-classify
    type: LLM
    perDocument: true                      # required for includeDocument to do anything
    includeDocument: { maxPages: 5, maxCharacters: 40000 }
    llmModelName: LARGE
    enrichment:
      - serviceBridgeRef: vendor-api
        operation: lookup-vendor
        inputMapping: { vendorId: "inputs.vendorId" }     # dot-path, not JSONata
        outputKey: vendorRecord
    promptVariables:
      vendorName: "enrichment.vendorRecord.name"          # JSONata, root has `enrichment`
    promptBody: |
      Vendor: {vendorName}
      Document:
      {documentText}

      Reply with JSON: {"decision": "receipt" or "invoice"}
    outputMapping:
      action: "$.decision"                 # MUST be named `action` for promptActions to resolve
      decision: "$.decision"
    promptActions:
      - { name: Receipt, slug: receipt }
      - { name: Invoice, slug: invoice }
```

`promptTemplateRef` is the alternative to `promptBody` — and it takes a **bare slug**, never an
`orgSlug/` prefix, unlike almost every other ref in a plan.

## Anti-patterns, side by side

```yaml
# WRONG — the discriminator is `type`, and kind fields do not live under config
- slug: extract
  kind: EXECUTION
  config: { moduleRef: "kodexa/fast-pdf-model" }

# RIGHT
- slug: extract
  type: EXECUTION
  moduleRef: "kodexa/fast-pdf-model"
```

```yaml
# WRONG — no action() global, and `total` would not be readable downstream anyway
scriptBody: |
  action(ctx.context.total > 10000 ? 'high' : 'low');

# RIGHT
scriptBody: |
  return { action: Number(families[0].metadata.total || 0) > 10000 ? 'high' : 'low' };
scriptActions: [{ name: High, slug: high }, { name: Low, slug: low }]
```

```yaml
# WRONG — bare strings are JSONata path lookups; `$.status` does not exist; and an
# unmatched path in a request* map fails the step rather than sending null
requestQuery:  { mode: "auto", tier: "inputs.tier" }
treatAsError:  '$.status >= "400"'

# RIGHT
requestQuery:  { mode: "'auto'", tier: "$exists(inputs.tier) ? inputs.tier : 'standard'" }
treatAsError:  "$._statusCode >= 300"
```

```yaml
# WRONG — an EXECUTION step cannot emit actions, so this edge can never fire
- slug: post
  type: BRIDGE_CALL
  dependsOn: ["extract:done"]
  serviceBridgeRef: erp
  endpointName: submit
  requestBody: { vendor: "inputs.vendorId" }

# RIGHT — route from the SCRIPT that follows the extraction
- slug: post
  type: BRIDGE_CALL
  dependsOn: ["grade:standard"]
  serviceBridgeRef: erp
  endpointName: submit
  requestBody: { vendor: "inputs.vendorId" }   # a BRIDGE_CALL with neither request* maps
                                               # nor requestScript fails the step at run time
```
