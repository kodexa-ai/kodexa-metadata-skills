# The `project` YAML — the thing a template creates

A template is the blueprint. The **project** is a syncable resource in its own right, with its own
file, its own push order and its own traps. `kdx sync pull` writes one per project and `kdx sync
push` sends it — no template is involved on either leg.

## Where it lives

`<metadata_dir>/projects/<project-slug>.yaml` — a **file**, sitting beside the **directory**
`<metadata_dir>/projects/<project-slug>/` that holds the project's own project-scoped children
(`assistants/`, `triggers/`, `knowledge-items/`). Same name, two different things.

You do not list a project under `organization:` with the other org-scoped types. Its key under
`projects:` *is* the instruction to push it:

```yaml
projects:
  invoice-ops:                       # this key alone means "push projects/invoice-ops.yaml"
    assistant: [invoice-helper]      # project-owned children, pushed into this project
    trigger:   [review-on-lock]
    linked:                          # org-scoped resources, pushed then BOUND to this project
      activity-plan: [invoice-review-flow]
      task-status:   [todo, in-review, approved]
```

`skip_projects: [invoice-ops]` excludes one without deleting the key — and it excludes the whole
subtree: no project YAML push, no project-scoped children, no `linked:` bindings. Org-scoped entries
at the top level still push. `kdx sync pull --discover` writes a `# <project name>` comment above
each key, because project slugs are not always self-describing — and it omits any project whose
`statusType` is `ARCHIVED`.

Push order **60**: after activity-plans (50), intakes (55) and project-templates (58), before
task-templates and task-statuses (65), assistants (70) and triggers (75) — so the project exists
before anything that has to resolve into it.

## Shape

```yaml
name: "Invoice Operations"
slug: invoice-ops
description: "Vendor invoice intake and review"
statusType: ACTIVE                     # ACTIVE | ARCHIVED | PENDING_DELETE — see the traps
color: "#10B981"
notes: ""
projectTemplateRef: acme-corp/invoice-template   # lineage only; never re-applied
options:
  options: []                          # option DEFINITIONS (a template seeds these)
  dataOptions: []
  properties: { region: "EMEA" }       # option VALUES
  dataProperties: {}
  taskOptions: { showNewTask: true }   # exactly one field
  companion:
    moduleRefs: ["acme-corp/invoice-review-skills"]
documentStatuses:                      # project-owned rows — created after the project, see below
  - { status: "New",      slug: new,      statusType: UNRESOLVED, color: "#6B7280" }
  - { status: "Complete", slug: complete, statusType: RESOLVED,   color: "#10B981" }
```

| Key | Notes |
|---|---|
| `name` | Required by the API. |
| `slug` | Unique per organization. Omit it and the server derives one from `name`; supply it and it is honoured — unless it collides, and then it is silently auto-suffixed (see the traps). A name that does not slugify at all (symbols or emoji only) falls back to a generated `project-<random>` slug instead of a 400. |
| `description`, `notes`, `color` | Plain columns. |
| `statusType` | Defaults to `ACTIVE` on create. |
| `projectTemplateRef` | `orgSlug/templateSlug`. Lineage only — see the traps. |
| `options` | A **closed struct**, like the template's: `options`, `dataOptions`, `properties`, `dataProperties`, `taskOptions`, `companion`, `executionPolicy`, `taxonTypeFeatures`, `groupTaxonTypeFeatures` and nothing else — an unrecognised key is accepted, dropped, and gone from the next `GET`. `properties` / `dataProperties` hold the values a project actually runs on; `options` / `dataOptions` hold the definitions the UI renders. `taskOptions` has one field, `showNewTask`. `companion` (`agentRuntimeRef`, `moduleRefs`, `prompt`) is the project-level agent baseline: a task template's own `metadata.companion` unions its `moduleRefs` onto this list and overrides the prompt and runtime ref. |
| `documentStatuses` | Project-owned status rows (`status` is the label, plus `slug`, `color`, `icon`). `slug` is what the CLI's post-push hydration keys on — a row without one never lands (see the traps). `statusType` is `UNRESOLVED` or `RESOLVED` only. |
| `dataFlow` | The data-flow canvas (`nodes`, `edges`, `viewPort`). A push writes the column, but Studio's canvas no longer saves to it — its save path logs a warning and updates only the in-memory copy. |
| `memory` | `recentFilters`, `recentQueries`, `orderedDashboards`. Stored, returned, round-tripped, and read by nothing — the UI keeps its recent filters in browser storage. `kdx` drops the key from the file when it is empty. |

Not in the file, and not for you to author: `id`, `organization`/`organizationId`/`orgSlug`, `ref`,
`searchText`, `createdOn`/`updatedOn`, `changeSequence`, `owner`, and the `id` of any nested row
that carries a `slug` — pull strips all of them. Push additionally drops `ownerId`, `statusId`,
`status`, `templateArticleId` and `parentProjectId` before sending. `parentProjectId` is
platform-stamped provenance and never yours to set: the create path clears whatever you send and the
update path refuses to write the column — silently, in both directions.

## Traps

**`projectTemplateRef` is lineage, not an instruction.** Only `POST /api/projects` with the ref in
the *request body* materializes a template. On create, `kdx sync push` deliberately withholds the
ref and re-applies it afterwards as a plain metadata update, so a synced project is never
provisioned by its template — you get an empty project with a template name attached. Provision it
by creating it through the API (SKILL.md, Rule 2), or push the resources yourself.

**`documentStatuses:` only ever creates — it never edits.** The project create and update both omit
association fields, so the array in the body writes nothing; `kdx` POSTs each row to
`/api/document-statuses` after the project lands, skipping any slug already present on the target.
That hydration runs on every push — created, updated, even unchanged — so new statuses do appear,
but edits to an existing one (label, colour, `statusType`) are silently ignored forever, and a row
with no `slug:` is skipped entirely. Change an existing status through its own endpoint, not the
project YAML.

**`documentStatuses[].statusType` has two values.** `UNRESOLVED` and `RESOLVED`. Anything else —
`OPEN`, `DONE`, a typo — decodes to `UNRESOLVED` with no error and no warning. Task statuses use a
different enum entirely; see the **task-status** skill.

**`taskStatuses:` does not belong in a project file any more.** Against the Go API the project
record carries no task-status collection at all, so a pull never writes one. If a legacy-era file
still has the array, push creates each entry as an **org-level** task status (the create resolves
the organization from the project ref it sends) and **binds nothing** — the project still cannot use
them. Move them to the project's `linked: task-status:` block instead; see the **project-resource**
skill.

**Archiving mangles the name and the slug.** Setting `statusType` from `ACTIVE` to `ARCHIVED`
rewrites the name to `<name> [archived-xxxxxxxx]` and the slug to `<slug>-archived-xxxxxxxx`, so the
slug in your file no longer names anything on the server, and discovery drops the project from a
regenerated manifest.

**Deleting a project keeps its slug forever.** Delete is a soft delete: `statusType` becomes
`PENDING_DELETE`, the name gains a UUID suffix, the slug is untouched, and the row vanishes from
both `GET` and list. The next push therefore does not find it and creates a new project — and
because slug collisions are auto-suffixed **counting soft-deleted rows**, the new project is minted
as `invoice-ops-1` while your manifest still says `invoice-ops`.

## Related

- **project-resource** — the `linked:` block, and what a binding actually gates.
- **kdx-cli** — manifest shape, repository layout, push order, `${org}` and the sigils.
- `SKILL.md` here — the template that provisions a project at create time, and only then.
