# Project template — field reference

Everything here is subordinate to the four rules in `SKILL.md`. In particular: any key not listed
below is dropped silently, and most `ref:` failures only produce a warning.

## Header fields

| Key | Notes |
|---|---|
| `slug` | Required. Unique per org; the template is addressed as `project-template://<org>/<slug>`. |
| `orgSlug` | Owning organization slug. `kdx sync push` injects the resolved organization id itself — do not author `organizationId:`. |
| `type` | File discriminator. `project-template`, `projectTemplate`, `project-templates` and `projectTemplates` are all accepted, and the value is stored verbatim. When absent the server defaults it to `project-template`. May be omitted entirely if you pass `kdx apply --type`. |
| `name` | Display name in the template picker. |
| `description` | The only other field the picker renders, and what its search box matches on. |
| `version` | Free-form string. Not used for resolution — refs are unversioned. |
| `publicAccess` | `true` grants read access to the template outside the owning organization. |
| `deleteProtection`, `deprecated` | Booleans. |
| `overviewMarkdown`, `icon`, `imageUrl`, `provider`, `providerUrl`, `providerImageUrl` | Presentation metadata carried in the template body. Persisted and round-tripped, but no current platform surface renders them for a project template. |
| `helpUrl` | Copied onto the created project as its help-article id. |
| `extensionPackRef` | Set when the template ships inside an extension pack. |
| `template`, `id`, `ref`, `checksum` | Accepted for round-trip compatibility; not meaningful to author. |

## `stores:`

Two mutually exclusive shapes.

```yaml
stores:
  # (a) CREATE a store owned by this project
  - slug: "${project.id}-documents"
    name: "Documents"
    storeType: DOCUMENT          # DOCUMENT or omitted -> document store; any other value -> data store
    storePurpose: OPERATIONAL    # OPERATIONAL | TRAINING
    deleteProtection: false
  # (b) BIND an existing org store
  - ref: "${orgSlug}/shared-intake"
```

- The `storeType` router is literal: exactly `DOCUMENT` (or empty) routes to the document-store table;
  everything else — `DATA`, and the legacy `TABLE` that older templates carry — routes to the
  data-store table. The persisted value is normalised to `DOCUMENT` / `DATA` either way.
- Only `name`, `slug`, `storeType`, `storePurpose` and `deleteProtection` are applied to a created
  store. `templateRef` is resolved (document stores only) and stamped onto an internal column that
  nothing reads — it copies no documents, metadata or configuration. See the inert list in `SKILL.md`.
- On the `ref:` branch the resolver falls back to the other table if the first lookup misses, so a
  bound store still resolves when `storeType` disagrees with reality.

## `taxonomies:`

Three distinct shapes — `ref` and `templateRef` are **not** synonyms.

```yaml
taxonomies:
  - ref: "${orgSlug}/invoices"               # BIND the existing taxonomy to the project
  - slug: "${project.id}-categories"
    name: "Categories"
    templateRef: "kodexa/standard-taxonomy"  # COPY that taxonomy's metadata into a new project taxonomy
  - slug: "${project.id}-local"
    name: "Local"
    taxons:                                  # inline taxons — MUST be a list
      - name: vendor
        externalName: Vendor
        label: "Vendor"
        valuePath: VALUE_OR_ALL_CONTENT
```

`templateRef:` copies the referenced taxonomy's whole metadata blob (its taxons, `taxonomyType`,
`enabled`, ...) — `ref:` copies nothing and binds the original instead. `ref:` short-circuits the
whole entry; between `templateRef:` and inline `taxons:`, `templateRef` wins.

`taxons:` written as a mapping instead of a list decodes to zero taxons and the error is swallowed:
you get a successfully-created, empty taxonomy with no log line. `taxonomyType` and `description`
authored beside inline `taxons:` are not applied. `name`, `externalName` and `valuePath` are the
required taxon fields (`label` is not) — full taxon structure is in the **data-definition** skill.

## `dataForms:`

```yaml
dataForms:
  - ref: "${orgSlug}/invoice-review"   # BIND an existing org data form
  - slug: "${project.id}-review"       # or create one inline
    name: "Invoice Review"
    entrypoints: [documentFamily]      # consumed values: documentFamily, workspace
    cards: []                          # MUST be a list; a mapping yields a card-less form, silently
```

`templateRef:` on a data form is read by nothing — an entry using it falls into the inline branch with
an empty name and slug. `description:` is not persisted for inline forms. Card structure is in the
**data-form** skill.

## `documentStatuses:`

Project-scoped rows, created directly on the new project.

```yaml
documentStatuses:
  - status: "New"          # the label; also the key for ${documentStatus.New.id}
    slug: new              # optional — derived from `status` by lowercasing + hyphenating if omitted
    color: "#6B7280"
    icon: "inbox"
    statusType: UNRESOLVED # UNRESOLVED | RESOLVED — anything else decodes to UNRESOLVED, silently
```

`${...}` is **not** substituted anywhere inside this block. The slug matters beyond display: activity
plans address these rows at run time as `${project.documentStatusId.<slug>}`, while the template-time
variable `${documentStatus.<label>.id}` is keyed by the label. Keep both in mind when naming.

## `taskStatuses:`

See `SKILL.md` for the org-reuse and required-slug rules. Full shape:

```yaml
taskStatuses:
  - label: "To Do"
    slug: todo             # REQUIRED — never derived
    color: "#6B7280"
    icon: "circle"
    statusType: OPEN       # OPEN | IN_PROGRESS | DONE | BLOCKED | PENDING (legacy TODO -> OPEN)
    locked: false          # locks the TASK on entry to this status (read-only), not the transition
```

`lockDocumentFamily` (default true — whether the task lock cascades to attached document families)
and `sequence` are fields of the standalone task-status resource and are **not** on the embedded
shape; author them in a `task-status` YAML and bind it. `oldIdentifier` is accepted so pre-refactor
templates can remap old ids onto the new rows.

## `taskTemplates:`

```yaml
taskTemplates:
  # (a) BIND an org-level template — preferred; the org template stays the source of truth and
  #     edits to it propagate to every bound project
  - ref: "task-template://${orgSlug}/invoice-review"
  # (b) INLINE — creates a per-project COPY (still an org row, bound to this project)
  - name: "Review Task"
    slug: "${project.id}-review"
    description: "Manual review"
    metadata:
      priority: 2
      teamSlug: review-team
      options: []
      forms: []
      actions: []
```

- The inline copy is written straight to the database, bypassing the task-template API handler. The
  shared task-template validator is invoked on this path (a failure there would hard-fail the whole
  project create), but its rule set is currently **empty** — nothing catches a malformed inline
  template. A template authored at org level and bound with `ref:` goes through the normal
  task-template write path instead (org resolution, slug and foreign-key checks).
- `initialStatusSlug` is a column on the task template, **not** part of the embedded metadata shape.
  A template that needs an initial status must be authored at org level and bound with `ref:`.
- Full `metadata` keys: `options`, `forms`, `actions`, `agentShortcuts`, `documentFamilyGroups`,
  `workspaceId`, `properties`, `priority`, `teamSlug`, `aiNaming`, `chatPrompt`, `executionPolicy`,
  `companion`. There is no `planned` or `planTemplate` — orchestration belongs in an activity plan.

## `activityPlans:` and `triggers:`

Covered in `SKILL.md`. Reference shapes:

```yaml
activityPlans:
  - ref: "activity-plan://${orgSlug}/invoice-review-flow"   # scheme optional here
  - ref: "${orgSlug}/invoice-posting"

triggers:
  - slug: review-on-lock
    name: "Review a locked invoice"
    eventKind: document_locked
    eventFilter: { expr: "$exists(documentFamilyId)" }
    inputMapping: { expr: "{ \"documentFamilyId\": documentFamilyId }" }
    activityPlanRef: "activity-plan://${org}/invoice-review-flow"   # scheme REQUIRED here
    enabled: true
    metadata: {}
```

`slug`, `name` and `activityPlanRef` are the only trigger fields that take `${...}` substitution.

## `assistants:`

```yaml
assistants:
  - name: "Document Processor"
    slug: "${project.id}-processor"      # derived from the name if omitted
    description: "Prepares incoming documents"
    subscription: "hasMixins('spatial') && type == 'content'"   # flat string; stored, not evaluated
    priorityHint: 10
    chatEnabled: false
    showInTraining: false
    options:
      taxonomies: ["${orgSlug}/${project.id}-categories"]
      complete_label: "labeled"
      pipeline:
        steps:
          - ref: "kodexa/fast-pdf-model"
            stepType: MODEL
            conditional: "context.project.use_ocr == False"
          - ref: "kodexa/apply-status"
            stepType: MODEL
            options: { document_status: "${documentStatus.Processing.id}" }
      # agentic assistants:
      prompt: "Help reviewers finish invoice tasks."
      moduleRefs: []
      agentRuntimeId: ""
```

`options:` is a typed struct — its only keys are `taxonomies`, `pipeline`, `complete_label`, `prompt`,
`moduleRefs`, `agentRuntimeId`. A pipeline step takes `ref`, `name`, `stepType`, `options`,
`conditional`, `executionPolicy`. Anything else is dropped.

`assistantRole` is a free-form string; the only value the platform acts on is `TASK`. Do not author
it: materialization always adds one extra assistant per project — a system **Task Assistant**
(`slug: task-assistant`, `assistantRole: TASK`) used for plan-driven executions — and creates it only
when the project has no `TASK`-role assistant yet.

Nothing in the platform currently evaluates an assistant's `subscription`, and nothing in the Go
services reads `options.pipeline`; both are stored configuration consumed elsewhere. Omitting
`showInTraining` / `chatEnabled` stores `false` (not the entity's own `true` default).

## `knowledgeSets:`

Created (never bound by ref) and always with status `ACTIVE`.

```yaml
knowledgeSets:
  - slug: "${project.id}-routing"
    name: "Routing rules"
    description: "Route invoices by vendor"
    setType: extraction          # free-form discriminator; no enum server-side
    features:
      - uuid: "00000000-0000-4000-8000-000000000010"   # local handle, referenced by clauses below
        slug: high-value                               # LOOKUP key only, not the created slug
        featureTypeRef: "kodexa/numeric-feature"
        properties: { field: total_amount, operator: ">", threshold: 10000 }
        active: true
    clauses:
      - features:
          - { featureUuid: "00000000-0000-4000-8000-000000000010", positive: true }
    knowledgeItems: []
    featureExpression: {}
```

A clause feature is matched against the `uuid` values declared in this template's own `features:`
list — a `featureUuid` with no matching entry is dropped without a warning. A feature is first looked
up by `slug` in the org; when that misses, `featureTypeRef` + `properties` create one whose slug is
**computed from the feature type and properties**, not from the `slug` you authored. Feature, clause
and item semantics are in the **knowledge-system** skill.

## `options:`

```yaml
options:
  options:              # project settings shown in the UI
    - name: use_ocr
      type: boolean
      label: "Enable OCR"
      default: true
      hint: "Disable for digital PDFs"
    - name: mode
      type: select
      label: "Processing mode"
      default: auto
      possibleValues:
        - { value: auto,   label: "Automatic" }
        - { value: manual, label: "Manual review" }
    - name: advanced
      type: string
      label: "Advanced"
      showIf: "mode == 'manual'"
      developerOnly: true

  dataOptions:          # see "Data options" below
    - name: entityId
      type: string
      label: "Entity ID"
      required: true

  properties: {}        # free-form values
  dataProperties: {}    # free-form values; the ones activity plans read at run time
  groupTaxonTypeFeatures: {}
  taxonTypeFeatures: {}

  taskOptions:
    showNewTask: true   # the ONLY field on taskOptions

  executionPolicy:
    timeoutSeconds: 900        # default 900
    maxAttempts: 1             # default 1 (no retry)
    backoffStrategy: immediate # immediate | linear | exponential
    backoffBaseSeconds: 0
    onExhausted: fail          # fail | skip

  companion:
    agentRuntimeRef: "${orgSlug}/companion"
    moduleRefs: []
    prompt: "Help users complete project tasks."
```

`showOnPopup: true` puts an option on the New Project dialog's details step; without it the option
sits behind a separate tab. `developerOnly` options under `options:` are only rendered (and so only
enforced) in the full Studio UI.

An `Option` (used by both `options:` and `dataOptions:`) accepts: `name`, `type`, `subType`,
`listType`, `label`, `falseLabel`, `listLabel`, `listDescription`, `description`, `hint`, `tabName`,
`aliases`, `required`, `default`, `showIf`, `developerOnly`, `showOnPopup`, `featureFlag`,
`supportArticle`, `overviewMarkdown`, `possibleValues`, `groupOptions`, `displayProperties`,
`properties`.

### Data options and project data properties

`dataOptions` defines the option set whose **values** live on the created project under
`options.dataProperties`. It is the only option list with server-side enforcement, and only on the
orchestrator's fan-out project-ensure path: an ensure whose `dataProperties` omits any `dataOption`
that is `required: true` and has no `default` is rejected with `requires dataProperties [...]`. The
regular New Project form enforces required options client-side only.

Activity plans read the values at run time as `${project.options.dataProperties.<name>}`, substituted
anywhere inside a string (task titles, properties). An unset or non-scalar key resolves to the **empty
string**, not to a literal — a missing value produces a quietly wrong title rather than an error.

Values a user enters at project-create are overlaid on top of the template's `properties` /
`dataProperties` seeds, so the form wins over the template per key.

## `tags:` and `memory:`

```yaml
tags:
  - { label: "Production", color: "#10B981" }

memory:
  recentFilters: {}
  recentQueries: {}
  orderedDashboards: []
  changeSequence: 0
```

`tags` is template-catalogue metadata only (see the inert list). `memory` is copied verbatim onto the
project with no `${...}` substitution.

## `linkedProjects:`

Display vocabulary for parent/child project navigation on projects created by plan-driven fan-out.
Read by the platform UI's lineage panel and chip; every key optional, each falling back to a neutral
default.

```yaml
linkedProjects:
  label: "Linked projects"      # panel title
  parentChip: "Parent"          # chip prefix on the parent project
  childChip: "Created from"     # chip prefix on a child project
  countLabel: "Activities"      # header for the activity-count column
  propertyColumns:              # options.dataProperties keys rendered as columns
    - { key: entityId, label: "Entity" }
```

The lineage link itself is `parentProjectId` on the project. It is **server-set only**: it is stripped
from every public project POST and written solely by the fan-out ensure path. Never author it.

## Refs and resolution

`ref:`/`templateRef:` values accept three forms, all resolved against the *target table* for that
collection:

| Form | Resolves as |
|---|---|
| `scheme://org/slug` | scheme stripped, then org + slug |
| `org/slug` | as written |
| `slug` | slug in the new project's own organization |

A trailing `:version` is stripped. The one place a bare form is **not** accepted is a trigger's
`activityPlanRef`, which is validated to be a full `activity-plan://org/slug` URI.

## `kdx sync` layout and push order

A template is an org-scoped resource: it lives in the sync tree at `project-templates/<slug>.yaml`
and is addressed as `project-template://<org>/<slug>`.

Push order (lower pushes first):

```
data-definition / data-form / document-store / data-store / module 20 → activity-plan 50 → intake 55
→ project-template 58 → project 60 → workspace 63
→ knowledge-set / task-template / task-status 65 → assistant / knowledge-item 70 → trigger 75
```

Templates land at 58, before projects at 60, so a project's `projectTemplateRef` lineage resolves.
Note that task-templates, task-statuses and knowledge-sets push *after* projects — a fresh-org sync
cannot materialize a template anyway (`projectTemplateRef` is stripped on project create), so this
ordering only matters when you create the project yourself once the sync has finished.

Workspaces are a separate project-scoped resource with their own endpoint; there is no `workspaces:`
key on a project template and no panel list anywhere in the model.
