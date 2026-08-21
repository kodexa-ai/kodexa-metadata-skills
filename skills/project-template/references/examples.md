# Project template — worked examples and troubleshooting

## A complete template

```yaml
slug: invoice-processing
orgSlug: acme-corp
type: project-template
name: "Invoice Processing"
description: "Extract, review and approve vendor invoices"
version: "1.2.0"
publicAccess: false
deleteProtection: true
icon: "file-invoice"
overviewMarkdown: |
  Creates an intake store, a review workflow and the invoice extraction pipeline.
provider: "Acme"
providerUrl: "https://acme.kodexa.example.com"

stores:
  - slug: "${project.id}-documents"
    name: "Documents"
    storeType: DOCUMENT
    storePurpose: OPERATIONAL
    deleteProtection: false
  - slug: "${project.id}-extracted"
    name: "Extracted Data"
    storeType: DATA
    storePurpose: OPERATIONAL
  - ref: "${orgSlug}/shared-vendor-reference"

taxonomies:
  - ref: "${orgSlug}/invoices"

dataForms:
  - ref: "${orgSlug}/invoice-review"

documentStatuses:
  - { status: "New",        slug: new,        statusType: UNRESOLVED, color: "#6B7280", icon: inbox }
  - { status: "Processing", slug: processing, statusType: UNRESOLVED, color: "#3B82F6" }
  - { status: "Complete",   slug: complete,   statusType: RESOLVED,   color: "#10B981" }

taskStatuses:
  - { label: "To Do",       slug: todo,        statusType: OPEN,        color: "#6B7280" }
  - { label: "In Review",   slug: in-review,   statusType: IN_PROGRESS, color: "#F59E0B" }
  - { label: "Waiting",     slug: waiting,     statusType: PENDING,     color: "#A855F7" }
  - { label: "Done",        slug: done,        statusType: DONE,        color: "#10B981", locked: true }

taskTemplates:
  - ref: "task-template://${orgSlug}/invoice-review"

activityPlans:
  - ref: "activity-plan://${orgSlug}/invoice-review-flow"

triggers:
  - slug: review-on-lock
    name: "Review a locked invoice"
    eventKind: document_locked
    eventFilter: { expr: "$exists(documentFamilyId)" }
    activityPlanRef: "activity-plan://${org}/invoice-review-flow"
    enabled: true

assistants:
  - name: "Prepare Document"
    slug: "${project.id}-prepare"
    description: "Runs extraction on incoming invoices"
    options:
      pipeline:
        steps:
          - { ref: "kodexa/fast-pdf-model", stepType: MODEL }
          - ref: "kodexa/apply-status"
            stepType: MODEL
            options: { document_status: "${documentStatus.Processing.id}" }

options:
  options:
    - { name: use_ocr, type: boolean, label: "Enable OCR", default: true }
  dataOptions:
    - { name: entityId, type: string, label: "Entity ID", required: true }
  taskOptions:
    showNewTask: true
  executionPolicy:
    timeoutSeconds: 900
    maxAttempts: 1
    onExhausted: fail

linkedProjects:
  label: "Linked projects"
  parentChip: "Parent"
  childChip: "Created from"
  countLabel: "Invoices"
  propertyColumns:
    - { key: entityId, label: "Entity" }
```

## Minimal template

Everything below the header is optional. This is a valid template that provisions one store:

```yaml
slug: blank-with-intake
orgSlug: my-org
type: project-template
name: "Blank with intake"
stores:
  - slug: "${project.id}-documents"
    name: "Documents"
    storeType: DOCUMENT
```

## Embedded triggers — event kinds and filters

Only three event kinds are actually emitted today. The other three accepted values persist and never
fire, which looks identical to a working trigger until you notice nothing runs.

| `eventKind` | Fires today | Payload keys the filter sees |
|---|---|---|
| `task_status_changed` | yes | `eventKind`, `taskId`, `projectId`, `organizationId`, `documentFamilyId` |
| `document_locked` | yes | `eventKind`, `documentFamilyId`, `projectId`, `organizationId`, plus `taskId` / `storeId` when present |
| `knowledge_set_updated` | yes | `eventKind`, `projectId`, `organizationId` |
| `task_created`, `activity_completed`, `manual` | **no** — accepted, stored, never dispatched | — |
| `schedule`, `document_arrived`, `data_extracted` | rejected at write time | — |

Each payload also has the event's own extra fields flattened on top, so filters address everything as
flat names.

`eventFilter` and `inputMapping` are JSONata carriers. Three accepted shapes:

```yaml
eventFilter: "$exists(documentFamilyId)"                  # bare string
eventFilter: { expr: "$exists(documentFamilyId)" }        # canonical wrapper
eventFilter: { taskTemplateRef: invoice-review }          # convenience: rewritten to an equality AND
```

The third form is rewritten on write into `{"expr": "taskTemplateRef = \"invoice-review\""}`. It takes
**scalars only** — a nested object or array in that position is an error. `{ expr: "" }` is an error.
An absent, null, `{}` or `""` filter means "match every event". An empty `inputMapping` passes the raw
event payload through as the activity inputs.

The **trigger** skill covers the standalone `triggers/*.yaml` form, JSONata patterns and the event
contract in full.

## Common mistakes

| Mistake | What actually happens |
|---|---|
| `scheduledJobs:` for cron work | Silently dropped. There is no scheduler; time-based automation is not available from a template. |
| `connections:` under an assistant | Silently dropped. Use the flat `subscription:` string — and note nothing evaluates it yet. |
| `options.taskOptions.showTakeNext` | Silently dropped. `showNewTask` is the only field. |
| `${project.slug}` in a slug or ref | Left as literal text, producing a slug containing `${project.slug}`. Use `${project.id}`. |
| `${orgSlug}` inside `documentStatuses:`, `taskStatuses:` or `memory:` | Left literal — substitution does not run in those blocks. |
| Store or taxonomy slug without a `${project.id}` prefix | Collides with the first project made from the template; component slugs are not auto-suffixed. |
| `taskStatuses:` entry with no `slug:` | All such entries collapse onto one shared empty-slug org row. |
| Expecting inline `taskStatuses:` to be project-private | They are org rows keyed on `(organization, slug)`; an existing slug is reused with its original label and colour. |
| `templateRef:` on a data form | Never read — the entry falls into the inline branch and creates an empty, nameless form. |
| `templateRef:` on a store, expecting seeded content | Resolved and stamped onto an internal column nothing reads. No documents, metadata or configuration are copied. |
| A `documentStatuses` entry with a `statusType` outside `UNRESOLVED`/`RESOLVED` | Decodes to `UNRESOLVED` with no error. |
| An inline taxon with only a `label` | `name`, `externalName` and `valuePath` are the required taxon fields; a label-only taxon is created but addresses nothing. |
| `taxons:` or `cards:` written as a mapping | Decodes to zero taxons / cards; the error is swallowed and the resource is created empty. |
| Embedding a trigger whose plan is not in `activityPlans:` | The trigger is created but never runs; start-activity rejects the unbound plan with HTTP 400. |
| Leftover `${...}` in a trigger's `activityPlanRef` | Write-time validation error — and it only warns, so the trigger is silently not created. |
| `eventKind: task_created` | Persists, never fires. Use `task_status_changed`, `document_locked` or `knowledge_set_updated`. |
| Editing a template and expecting existing projects to change | Materialization is create-only. Re-create the project, or edit its resources directly. |
| Creating a project with `?templateRef=` — including `kdx project create --template` | Ignored: the create handler reads the body only, so you get a bare project with none of the template's resources. |
| Expecting `kdx sync push` of a project YAML to apply its template | It strips `projectTemplateRef` on create and restores it afterwards as lineage only. |
| `tags:` expecting them on the created project | Nothing in the platform reads a template's tags, and projects have no tags field. |
| `attributeStatuses:` | Stored, never materialized — no attribute-status table or endpoint exists. |
| Authoring `parentProjectId` | Stripped from every public POST; only the fan-out ensure path may set it. |
| Authoring `organizationId:` in a synced YAML | `kdx` injects the resolved organization; author `orgSlug:` as the file's routing identity instead. |

## Verifying a new template

Because most ref failures only warn, create one throwaway project from a new template and check that
each expected resource actually exists and is bound:

1. `POST /api/projects` with `projectTemplateRef` in the body.
2. List the project's bound resources and confirm one entry per `ref:` you authored — a missing entry
   means that ref did not resolve.
3. List the project's triggers and confirm one per `triggers:` entry.
4. Confirm no store, taxonomy or data form was created empty (the swallowed-unmarshal cases).

If the create returns an error instead, it came from one of the hard-fail cases: template not found,
a document- or task-status write, an inline data-form create, or an inline task-template create. The
project is rolled back entirely, so fix and retry rather than hunting for a partial project.
