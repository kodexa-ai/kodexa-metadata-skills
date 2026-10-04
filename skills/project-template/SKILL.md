---
name: project-template
description: "Use when authoring or editing a Kodexa project-template YAML — the org-scoped blueprint that provisions a new project's stores, assistants, taxonomies, data forms, document and task status workflows, task templates, knowledge sets, activity-plan bindings, triggers, project options and parent/child lineage config"
---

# Project templates

A project template is an **org-scoped** YAML resource. Applying it to a **brand-new** project creates
and binds that project's resources in one pass. Four rules decide whether yours works.

## Rule 1 — the template body is a closed struct

The YAML you push is decoded into a fixed Go struct and the stored `metadata` is re-serialised **from
that struct**. Any top-level key not in the list below (and not a server-set envelope field such as
`organizationId`/`organization`, `id`, `uuid`, `createdOn`, `changeSequence`) is accepted — 201, no
warning, no log — then discarded: it never reaches the database and is absent from the next `GET`.

Every top-level key that exists:

```
id ref template orgSlug slug type name description version publicAccess imageUrl icon overviewMarkdown
provider providerUrl providerImageUrl deleteProtection deprecated checksum extensionPackRef helpUrl
projectNamePlaceholder projectDescriptionPlaceholder launchActivity stores assistants taxonomies
dataForms documentStatuses taskStatuses attributeStatuses taskTemplates activityPlans serviceBridges
triggers knowledgeSets tags options memory linkedProjects
```

Keys widely seen in older templates and docs that this rule kills:

| Key | What happens |
|---|---|
| `scheduledJobs:` | Vanishes. There is no such field. Timed work is a `schedule` trigger under `triggers:`. |
| `assistants[].connections:` | Vanishes. Assistants carry a flat `subscription:` string instead. |
| `options.taskOptions.showTakeNext` | Vanishes. `taskOptions` has exactly one field, `showNewTask`. |
| unknown keys under `options:` or `assistants[].options:` | Vanish — both are closed structs too (e.g. `confidence_threshold`, `write_back_to_store`). |

Nested collection items are closed structs as well. Two things do raise a 400 when the template is
written: a **known** key holding a wrong-typed value (`deleteProtection: "yes"`, `invalid metadata
field`), and a schedule that cannot run — any `assistants[].schedules[].cronExpression`, or a
trigger `triggerMetadata` that is invalid on a `schedule` trigger or present on any other kind.

## Rule 2 — a template is only ever applied at project CREATE

Materialization runs once, in the project-create path, and only when the **request body** carries
`projectTemplateRef`:

```json
POST /api/projects
{ "name": "Invoice Processing",
  "organization": { "id": "00000000-0000-4000-8000-000000000001" },
  "projectTemplateRef": "acme-corp/invoice-template" }
```

- Editing a template **never** retro-applies to projects already created from it. Re-create, or edit
  the project's resources directly.
- A `?templateRef=` **query parameter is ignored** — the create handler reads the body only. That is
  exactly what `kdx project create --template` sends, so it yields a bare, unprovisioned project.
- `kdx sync push` deliberately strips `projectTemplateRef` from project creates and restores it
  afterwards as lineage, so syncing a project YAML does not materialize its template.

## Rule 3 — most ref failures are warnings, not errors

| Outcome | Which cases |
|---|---|
| **Warn + skip** — project is created "successfully" with the resource missing | any unresolvable `ref:`/`templateRef:` (store, taxonomy, data form, task template, activity plan); `activityPlans:` entry with no `ref:`; a trigger entry with no `slug`; any store, taxonomy, assistant or knowledge-set create failure |
| **Hard fail — whole create rolls back, no project** | template not found; document-status create; task-status lookup/create; inline data-form create; inline task-template create; **any trigger that fails validation** (a 400 naming it); a schedule trigger over the cap (409 `TRIGGER_SCHEDULE_LIMIT`) |

A typo in a `ref:` produces a project that looks fine and is quietly missing a store or a trigger —
always verify what was actually bound after creating from a new template.

## Rule 4 — the six template variables, and where they do not work

| Variable | Resolves to |
|---|---|
| `${project.id}` | new project UUID |
| `${project.name}` | project display name |
| `${orgSlug}` | organization slug |
| `${org}` | organization slug (alias) |
| `${documentStatus.<Status label>.id}` | created document-status UUID — keyed by the `status` **label**, not the slug |
| `${assistant.<Name>.id}` | created assistant UUID — keyed by `name` |

There is **no `${project.slug}`**. An unknown `${...}` is left in place literally, not blanked.

Substitution does **not** run inside `documentStatuses:`, `taskStatuses:` or `memory:` — those are
copied verbatim, so `orderedDashboards: ["${orgSlug}/x"]` stores the dollar-brace text intact.
`${documentStatus.<label>.id}` enters the map only after `dataForms:` has run — usable from
`taskTemplates:` on, **never inside `dataForms:`** — and `${assistant.<Name>.id}` is added as each
assistant is created, so only a *later* entry in the same `assistants:` list can use it.

Materialization order: options → documentStatuses → taskStatuses → dataForms → taskTemplates →
activityPlans → triggers → knowledgeSets → stores → taxonomies → assistants → memory.

## Shape that works

```yaml
slug: invoice-template
orgSlug: acme-corp
type: project-template          # kdx also accepts projectTemplate / project-templates
name: "Invoice Processing"
description: "Extract and review vendor invoices"

stores:
  - slug: "${project.id}-documents"      # ${project.id}-prefix: store slugs share the ORG namespace
    name: "Documents"
    storeType: DOCUMENT                  # DOCUMENT or omitted -> document store; ANYTHING else -> data store
    storePurpose: OPERATIONAL            # OPERATIONAL | TRAINING
    deleteProtection: false
  - ref: "${orgSlug}/shared-intake"      # bind an existing org store instead of creating one

taxonomies:
  - ref: "${orgSlug}/invoices"           # BIND an existing taxonomy
  - slug: "${project.id}-categories"
    name: "Categories"
    taxons:                              # MUST be a list — a mapping decodes to zero taxons, silently
      - { name: vendor, externalName: Vendor, label: "Vendor", valuePath: VALUE_OR_ALL_CONTENT }

dataForms:
  - ref: "${orgSlug}/invoice-review"     # `ref:` binds. `templateRef:` on a data form does NOTHING.

documentStatuses:
  - { status: "New",      slug: new,      statusType: UNRESOLVED, color: "#6B7280" }
  - { status: "Complete", slug: complete, statusType: RESOLVED,   color: "#10B981" }

taskStatuses:                            # see "Task statuses" below — these are ORG rows
  - { label: "To Do", slug: todo, statusType: OPEN }

taskTemplates:
  - ref: "task-template://${orgSlug}/invoice-review"   # preferred: org template stays source of truth

activityPlans:                           # REF-ONLY. An entry without `ref:` is skipped with a warning.
  - ref: "activity-plan://${orgSlug}/invoice-review-flow"

triggers:                                # yes, triggers CAN be embedded here
  - slug: weekly-review-sweep
    name: "Weekly review sweep"
    eventKind: schedule                  # triggerMetadata is copied verbatim — no ${...} inside it
    triggerMetadata: { cron: "0 6 * * MON", timezone: America/New_York, jitterSeconds: 900 }
    activityPlanRef: "activity-plan://${org}/invoice-review-flow"
```

Field tables per collection: `references/schema.md`. Worked templates, `options:`, troubleshooting: `references/examples.md`. Standalone store YAML — the flat wire shape, `storeType` values and the legacy `type: store` remapping — is the **store** skill. The **project** YAML is a syncable resource of its own, and its `documentStatuses:` and `projectTemplateRef` behave nothing like the template's: `references/project-yaml.md`.

## Task statuses are org rows, not a project-private workflow

`taskStatuses:` entries are looked up in the org's task-status table by `(organization, slug)`:

- **`slug:` is required.** It is never derived from the label (unlike `documentStatuses`, where an
  omitted slug is derived from `status` by lowercasing and hyphenating non-alphanumerics). Omit it and
  every such entry collapses onto one shared empty-slug row.
- **An existing slug is reused as-is.** Your `label`, `color`, `icon` and `statusType` are ignored
  when a row with that slug already exists in the org. Two templates declaring `slug: todo` with
  different labels share whichever was created first.
- `statusType` is `OPEN | IN_PROGRESS | DONE | BLOCKED | PENDING`. Legacy `TODO` is rewritten to
  `OPEN`. `PENDING` is load-bearing: tasks in a PENDING-type status are excluded from the individual
  take-next pool and reach reviewers only via a task group.

The status is then bound to the project as a project-resource; see the **task-status** and **project-resource** skills.

## Triggers embedded in a template

A trigger fires an activity plan, so **the plan must also be listed in `activityPlans:`** — otherwise
it is created but never runs (`binding_missing` per event, `PLAN_UNBOUND` per slot). Specifics:

- `activityPlanRef` must be a full, unversioned `activity-plan://<org>/<slug>` URI with **no leftover
  `${...}`** after substitution. (In `activityPlans:` the scheme is optional; here it is mandatory.)
- **A trigger that fails validation fails the project create** with a 400 naming it — it is no longer
  skipped. A duplicate slug, or a re-apply, is a no-op per `(project, slug)`.
- No `projectId` (the server sets it). `triggerMetadata` is copied **verbatim** — no `${...}`.
- Schedule triggers count against the caps (5 per project, 500 per organization); `trigger:schedule`
  is not checked against the user creating the project — the template is their authority.
- Only `task_status_changed`, `document_locked`, `knowledge_set_updated` and `schedule` are
  dispatched. `task_created`, `activity_completed` and `manual` persist and never fire.

Event payload keys, `eventFilter`/`inputMapping` shapes and the standalone trigger YAML are in the **trigger** skill.

## Slugs

The project's own slug is unique per organization and auto-suffixed on collision (`x` → `x-1`),
counting soft-deleted projects too. Component slugs are **not** auto-suffixed: stores, taxonomies,
data forms and task templates are minted into the shared org namespace — prefix them with
`${project.id}`.

## Declared but inert

Round-tripped and visible in existing templates and the UI, but read by nothing:

| Key | Note |
|---|---|
| `attributeStatuses:` | No attribute-status table, entity or endpoint exists; never materialized. |
| `tags:` | Round-tripped in the template body, but nothing in the platform reads them and projects have no tags field. |
| `stores[].description`, `templateRef`, `files`, `hasImage`, `showThumbnails`, `showStoreInLabeling`, `highQualityPreview`, `allowDataEditing`, `documentProperties`, `labelExpressions` | Only `storeType`, `storePurpose`, `deleteProtection`, `name` and `slug` reach a created store. A store has no field at all for most of the rest; `documentProperties` and `labelExpressions` exist on a document store only as inert legacy keys (**store** skill). `templateRef` stamps an internal column nothing reads — it copies no documents, metadata or configuration. |
| `assistants[].assistantDefinitionRef`, `stores`, `schedules`, `loggingEnabled` | Never copied to the created assistant; no such columns exist. A `schedules[].cronExpression` is refused at write. |
| `assistants[].subscription` | Written onto the created assistant, but no current platform service evaluates it — confirm before relying on event-driven assistants. |
| `dataForms[].templateRef`, `dataForms[].description` | Never read; `templateRef` here silently falls into the inline-create branch. |
| `taxonomies[].taxonomyType`, `taxonomies[].description` | Dropped for inline taxonomies; set them on a standalone data-definition and bind with `ref:`. |
| `knowledgeSets[].active`, `showOnNewProject`, `ref` | Never read; status is hard-coded to `ACTIVE` and there is no bind-by-ref path. |
| `activityPlans[]` fields other than `ref` | Inline activity-plan creation is deliberately unsupported. |
| `overviewMarkdown`, `icon`, `imageUrl`, `provider`, `providerUrl`, `providerImageUrl`, `checksum` | Stored and round-tripped. The template picker renders only `name` and `description`; `checksum` is never computed or verified. |

## Related skills

`activity-plan` · `trigger` · `task-status` · `task-template` · `project-resource` · `store` · `data-definition` · `data-form` · `knowledge-system` · `kdx-cli`
