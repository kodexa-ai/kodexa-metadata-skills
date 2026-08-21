---
name: project-resource
description: "Use when making an org-scoped Kodexa resource usable inside a project — binding activity-plans, task-templates, task-statuses, data-definitions, data-forms, stores, prompts, service-bridges or intakes, writing `linked:` manifest blocks or project-template ref arrays, or debugging \"not bound to project\", empty pickers and 409-on-delete"
---

# Kodexa Project Resource (Binding) Authoring

## The model

Most Kodexa resources are **org-scoped**: one activity-plan, one task-template, one document store,
shared by the whole organization. A **project resource** is a binding row in `kdxa_project_resources`
saying "this project may use that resource" — the set of bindings *is* the project's resource list.

A binding is `(projectId, resourceType, resourceId)`. Three consequences worth memorising:

- **That triple is unique**, and the same resource can be bound to any number of projects. Re-binding
  through `/bind` returns the existing row; re-binding through the plain create endpoint is a `409`.
- **The row stores a resolved id, not a URI.** Renaming a resource's slug does not break its
  bindings, and you cannot read a binding back as a `scheme://org/slug` URI.
- **Binding does not copy.** The org resource stays the single source of truth for every project
  bound to it. (One exception: a project-template `taskTemplates:` entry authored *inline* rather
  than by `ref:` creates a separate template row and binds that copy.)

## The binding call

```
POST /api/project-resources/bind
{
  "projectId":    "00000000-0000-4000-8000-000000000001",
  "resourceType": "activity-plan",
  "resourceUri":  "activity-plan://acme-corp/invoice-posting"
}
```

- Send **`resourceUri`** (resolved server-side against the project's organization) **or**
  `resourceId` if you already hold the UUID. One is required.
- **Do not send `organizationId`** — it is derived from the project.
- **URIs are unversioned.** `activity-plan://acme-corp/invoice-posting:1.0.0` fails to resolve:
  `version-suffixed URIs not supported`.
- Idempotent: an existing binding returns `200 {"status":"existing"}`; a new one returns `201`.
- Requires the `bind` permission on `project-resource` for that project.

Unbind with `DELETE /api/project-resources/{id}`. It is a hard delete.

There is also a plain `POST /api/project-resources` (what the CLI and the UI actually call): no URI
and no resolution, so it needs `projectId`, `organizationId`, `resourceType` **and** `resourceId`.

## Authoring bindings in YAML

### In a `kdx` metadata-repo manifest — the `linked:` block

Anything under `linked:` is pushed at org scope and then bound. Org-scoped types listed *inline*
under the project are folded into `linked:` when the manifest loads, so they bind too — with the one
carve-out below. Project-scoped types (`assistant`, `trigger`) stay inline and never bind.

```yaml
manifest_version: "1.0"
metadata_dir: metadata

organization:
  activity-plan: [invoice-posting]
  task-template: [invoice-review]

projects:
  invoice-ops:
    assistant: [invoice-helper]        # project-owned — pushed, never bound
    trigger:   [review-on-lock]        # project-owned — pushed, never bound
    linked:                            # org-scoped — pushed, then BOUND to invoice-ops
      activity-plan:   [invoice-posting]
      task-template:   [invoice-review]
      task-status:     [todo, in-review, approved]
      data-definition: [invoices]
      document-store:  [invoice-inbox]
      service-bridge:  [erp]
```

Two silent traps here:

- **`task-template` and `task-status` are carved out of that folding.** Listed as a plain project key
  instead of under `linked:` they are pushed and never bound — the resource exists, the project
  cannot use it, nothing errors. Always give them their own `linked:` entry.
- **Deleting a slug from `linked:` does not unbind it.** The CLI only ever creates bindings; there is
  no unbind command. Remove the binding through the API or the project's Resources panel.

### In a project template — typed arrays, not a `resources:` block

Project templates have **no `resources:` list**. Each resource kind is its own array, and only some
of them create bindings when a project is materialized from the template:

```yaml
slug: invoice-ops-template
orgSlug: acme-corp
type: project-template
name: "Invoice Operations"

activityPlans:                                          # binds activity-plan  (ref: REQUIRED)
  - ref: "activity-plan://${orgSlug}/invoice-posting"
taskTemplates:                                          # binds task-template
  - ref: "task-template://${orgSlug}/invoice-review"
taskStatuses:                                           # creates org rows, then binds task-status
  - { label: "To Do", slug: todo, statusType: OPEN }
taxonomies:                                             # binds data-definition
  - ref: "${orgSlug}/invoices"
dataForms:                                              # binds data-form
  - ref: "${orgSlug}/invoice-review-form"
stores:                                                 # binds document-store / data-store
  - ref: "${orgSlug}/invoice-inbox"
```

- `${orgSlug}` resolves to the **new project's** organization at materialization time.
- `ref:` accepts `scheme://orgSlug/slug`, `orgSlug/slug`, or a bare `slug` (that org). **The scheme
  is stripped and never checked** — a `ref:` under `taxonomies:` is looked up as a taxonomy.
- `assistants:`, `triggers:` and `knowledgeSets:` create project-owned rows — **no** binding.
- **An unresolvable `ref:` is logged and skipped.** The project is created "successfully" with the
  binding missing, and the failure only surfaces later as a start-time or empty-picker problem.
- **The template is applied once, when a project is created from it.** Editing the template later
  binds nothing to projects that already exist.

## Valid `resourceType` values

Exactly these 21 strings are accepted; anything else is rejected at create with
`invalid resourceType`. All kebab-case.

```
activity-plan         task-template           task-status       trigger
intake                document-store          data-store        data-definition
data-form             module                  prompt            prompt-template
service-bridge        knowledge-set           knowledge-item-type
knowledge-feature-type  project-template      assistant         action
workflow              label
```

Two type traps that fail in opposite directions:

| You write | What happens |
|---|---|
| `resourceType: taxonomy` | **Rejected outright** — data definitions bind as `data-definition`. (A `taxonomy://` *URI* is fine; it is only the type string that must be `data-definition`.) |
| `resourceType: prompt-template` | **Accepted, and no project-team access follows** — prompt visibility is keyed on `prompt`. A manifest `linked: prompt-template:` entry also writes `prompt-template`, so a `prompt` binding has to come from the API. |

The `resourceUri` scheme and the `resourceType` are resolved independently and **never cross-checked**.
`resourceType: prompt` with `resourceUri: "activity-plan://acme-corp/invoice-posting"` creates a
binding that points at a plan and is labelled a prompt. Nothing complains; nothing works.

## What a binding actually controls

Enforced — a missing binding is the bug:

| Behaviour | Failure without the binding |
|---|---|
| Starting an activity from a plan | `400` — `activity-plan "…" is not bound to project …; create a project-resource binding first` |
| A trigger firing a plan | The same start call fails; the fire is recorded with reason `binding_missing` |
| An intake auto-starting a plan | The plan **and** the intake's document store must be bound to **exactly one** common project. Zero common projects → `400` "not bound to the same project"; two or more → `400` ambiguous |
| An intake creating a task from a task-template | The owning project is read off that template's binding (oldest first); with no binding the task is silently not created |
| Which document families an activity, step or script sees | Only families in **document stores bound to the project** — others are filtered out at start and read as "not found in project" |
| A script calling `serviceBridge.*`, and agent-initiated bridge calls | Bridge not found / "not linked to this agent's project" |
| Setting a task group's status slug | `422` — "is not bound to project …; bind the task-status via project-resources first" |
| Listing task-templates or task-statuses filtered by project | The project filter resolves through the binding table, so an unbound row is missing from every project-scoped list and picker — empty, not an error |
| Access for a user whose role comes from a **team-project assignment** | The org resource is invisible in lists and denied on read |
| Deleting the org resource itself | `409` — "resource is linked to N project(s); unlink it before deleting". Unbind everywhere first |

**Not** enforced — binding these is good hygiene for visibility, but a missing binding will not be
the cause of your failure:

- A `CREATE_TASK` step's `taskTemplateRef` — resolved against the organization only.
- That step's `taskStatusSlug` — resolved against the organization only.
- An LLM step's `promptTemplateRef` — resolved against the organization only.
- A `BRIDGE_CALL` step's `serviceBridgeRef` — only the script `serviceBridge.*` API and the agent
  proxy check a binding.

## Declared but inert

Fields you will meet on a binding row that nothing in the platform reads:

- **`bindSource`** — provenance only; no behaviour branches on it. The `X-Bind-Source` header that
  would set `UI` / `KDX_SYNC` / `PROJECT_TEMPLATE` is not sent by any shipped tool, so API-made
  bindings read `API`; `MIGRATION` and `LEGACY` appear only on rows written by schema upgrades;
  `INTAKE_AUTO` is declared and never written. Treat any value as "someone bound this".
- **`createdById`** — stamped from the caller by `/bind`, never read.
- **`resourceType: assistant` and `resourceType: trigger`** — accepted, but assistants and triggers
  already carry their own project id, and no code path consults a binding row for them.
- **`resourceType: action` and `resourceType: workflow`** — accepted, no consumer, and no URI scheme,
  so they can only be bound by raw `resourceId`.
- **`project_resource.bound` / `project_resource.unbound` platform events** — defined but never
  emitted. Do not expect a bind or unbind to show up in an event feed.

## Common mistakes

| Mistake | Fix |
|---|---|
| `resourceType: activityPlan` (camelCase) | kebab-case: `activity-plan` |
| `resourceUri: "invoice-posting"` (bare slug) | Full URI `activity-plan://acme-corp/invoice-posting`, or pass `resourceId` |
| Expecting a resource in a project picker after `kdx sync push` | Check the manifest's `linked:` block, not the resource file — and mind the task-template / task-status carve-out |
| Removing a `linked:` entry to unbind | Delete the binding row; the CLI never unbinds |
| Assuming a local alias or version pin exists | Neither exists — resources keep their org slug and are unversioned |

## Cross-references

- `activity-plan` (its start is binding-gated), `task-template` and `task-status` (the two types
  authors most often forget to bind), `trigger` (project-owned, never bound — but its plan must be)
- `project-template` — where most bindings are authored; `kdx-cli` — manifest layout and push order
