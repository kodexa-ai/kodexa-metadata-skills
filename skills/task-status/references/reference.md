# Task Status — reference

Lookup companion to `SKILL.md`. Read `SKILL.md` first — the facts that fail silently live there.

## Wire format

A `GET` on a single status returns the whole record. Fields you author are a small subset.

```json
{
  "id": "00000000-0000-4000-8000-000000000001",
  "uuid": "00000000-0000-4000-8000-000000000001",
  "createdOn": "2026-05-02T00:00:00Z",
  "updatedOn": "2026-06-11T09:14:22Z",
  "changeSequence": 7,
  "organizationId": "00000000-0000-4000-8000-000000000002",

  "slug": "in-review",
  "name": "In Review",
  "label": "In Review",
  "statusType": "IN_PROGRESS",
  "color": "#F59E0B",
  "icon": "hourglass",
  "sequence": 30,
  "locked": false,
  "lockDocumentFamily": true,
  "metadata": {},
  "deprecated": false,
  "template": false,
  "publicAccess": false,

  "ref": "acme-corp/in-review",
  "orgSlug": "acme-corp",
  "deleted": false
}
```

Two absences that surprise people:

- **No `uri`.** The generated OpenAPI schema declares a computed `uri` on every metadata resource, but
  it is only injected for resources whose metadata is flattened at serialization time, and task status
  is not one of them. Build `task-status://<orgSlug>/<slug>` yourself when you need to resolve one.
- **No nested `organization` object.** `organization.id` is accepted on write and resolved to
  `organizationId`; it is never echoed back. `orgSlug` and `ref` are what a read gives you.

## Field reference

### Authorable

| Field | Type | Default | Notes |
|---|---|---|---|
| `slug` | string | — | The only API-required field. Unique per organization. Lowercase letters, digits, hyphens; mixed case is downcased. This is what every other resource points at. |
| `name` | string | — | Full name. If `slug` is omitted it is derived from `name`; a create with neither is a 400. A status provisioned by a project template is inserted with no `name` at all, so do not rely on it being populated. |
| `label` | string | — | The short text on the task chip and in the pickers. This is what a reviewer actually sees. |
| `statusType` | enum | — | `OPEN` \| `IN_PROGRESS` \| `PENDING` \| `BLOCKED` \| `DONE`. Stored as a plain `varchar(20)` with no constraint — anything else is accepted and disables every behaviour below. |
| `color` | string | — | Hex. The chip background; the org admin list draws it as a dot. |
| `icon` | string | — | Stored and returned; nothing renders it. See "Declared but inert" in `SKILL.md`. |
| `sequence` | int | `0` | Orders the org admin list within each `statusType` group, and picks the auto-reopen target for a task group (lowest sequence among bound IN_PROGRESS, then OPEN). The reviewer-facing status dropdowns request no sort, so they do **not** honour it. |
| `locked` | bool | `false` | Entry lock: entering this status makes the **task** read-only. Not a transition guard. |
| `lockDocumentFamily` | bool | `true` | Whether the entry lock cascades to the task's attached document families. Set `false` to freeze the action but keep the document family editable. Only consulted when the lock actually applies. |
| `deprecated` | bool | `false` | Advisory only — nothing filters on it. |
| `metadata` | object | `{}` | Free-form; nothing reads it. |

### Accepted on create only

| Field | Notes |
|---|---|
| `projectId`, `project.id` | An org-resolution shortcut for callers that only know their project: the owning organization is resolved through the project, and **project scope wins over an explicit `organizationId`**. Neither has a column, so neither comes back on a read. Not needed when pushing with `kdx`, which supplies the organization. |
| `organizationId`, `organization.id` | Supplied by `kdx` on every org-scoped push; the nested `organization.id` wins over a flat `organizationId` in the same body. Do not author either. If none of these resolve, the create is a 400. |

### Server-managed and computed

| Field | Notes |
|---|---|
| `id`, `uuid` | Assigned on create. |
| `createdOn`, `updatedOn` | Timestamps. |
| `changeSequence` | Incremented on every write; used for optimistic concurrency. |
| `ref` | `orgSlug/slug`, recomputed on read. |
| `orgSlug` | Recomputed on read. There is no `uri` on the wire — see above. |
| `deleted`, `deletedDate`, `deleteUserEmail`, `deleteUserId` | Soft-delete bookkeeping. The nullable ones are omitted from a live row's JSON. |
| `yamlSource` | The authored YAML, preserved for round-trip when the row came from a push. |
| `template`, `publicAccess`, `extensionPackRef` | Present from the shared metadata base; unused here. |
| `type` | Stored, never populated by the server on new rows, never read by the platform. `kdx apply` reads a file-level `type:` only to route the file. |

## What each `statusType` drives, in detail

| Behaviour | OPEN | IN_PROGRESS | PENDING | BLOCKED | DONE |
|---|---|---|---|---|---|
| Individual take-next pool | yes | no | no | no | no |
| Group take-next pool (the **group's** own status) | yes | no | no | no | no |
| Task-group auto-reopen target | 2nd choice | 1st choice | never | never | never |
| Task-group done counter / auto-complete | — | — | — | — | counted |
| Completion guard on entry (can 409) | — | — | — | — | fires |
| Analytics "open" | yes | no | no | no | no |
| Analytics "completed" | no | no | no | no | yes |
| Analytics "unfinished" (drives overdue) | yes | yes | yes | yes | no |
| Activity-plan `CREATE_TASK` step advances | no | no | no | no | yes |
| Usable as a `dependsOn` action qualifier | no | no | no | no | yes |

Notes:

- Take-next also requires the task (or group) to be unassigned and not soft-deleted, and an individual
  task additionally to be ungrouped. It joins through `kdxa_task_statuses`, so a status slug that does
  not resolve removes the row from the pool entirely, regardless of type.
- A task group auto-completes only when every live member is DONE-typed and the group is not already
  DONE-typed. The slug it lands on is its configured `doneStatusSlug` when that still resolves to a
  non-deleted DONE-typed org status; otherwise the most common DONE-typed slug among its completed
  member tasks (alphabetical tiebreak). Auto-reopen fires only from a DONE-typed group status.
- The completion guard applies to analyst-driven writes. Machine writers (an activity-plan script
  setting a status) are deliberately unguarded.
- The analytics rows above are the status-type half of the expression. A task that carries a
  completion date counts as completed whatever its status type, and drops out of both "open" and
  "unfinished".

## Who points at a status slug

| Source | Field | Resolution and failure mode |
|---|---|---|
| Task | `status_slug` on the task row | Bare slug, no foreign key; joined to the org's statuses on every read. A miss drops the task out of take-next entirely and out of every DONE-typed check (guard, group counters, plan advance); in analytics it lands as neither open nor completed, only unfinished. |
| Task template | `initialStatusSlug` | Status stamped on tasks created from the template. |
| Task template | `metadata.actions[].properties.statusSlug` | The status an action button transitions the task to. The same `properties` block may carry `lockTask` / `lockDocumentFamily` overrides. |
| Task template | `metadata.properties.lockStatusSlug` | Opt-in: makes the task's Lock button a transition into that status (with both lock overrides forced on) instead of a plain lock. Only fires when the task is not already in it. |
| Activity plan | `CREATE_TASK` step's `taskStatusSlug` | **Fails open.** An unknown slug is not an error — the task is created with an empty status and the step never advances. |
| Activity plan | `dependsOn` action qualifier on an edge out of a `CREATE_TASK` step | Every DONE-typed slug in the org is a valid completion token there, alongside the referenced task template's declared action slugs. A qualifier matching neither fails the plan start loudly. |
| Activity plan | `SCRIPT` step task bindings | Scripts can read one status or list the org's statuses, and set a task's status by slug. Setting an unknown slug throws. A script-driven set honours the status's `locked` flag but **ignores `lockDocumentFamily`** — when the status locks, the attached document families are locked unconditionally. |
| Task group | `statusSlug` | **Fails closed.** Creating a group or updating its status with a slug that is not bound to the group's project, or that is soft-deleted, returns 422. |
| Task group | `doneStatusSlug` | **Not validated on write.** At auto-complete it is used only if it still resolves to a non-deleted DONE-typed status in the org (project binding deliberately not required); otherwise the mode of the members' completed slugs wins, and if there is none the group is left un-flipped. |
| Assistant | `${taskStatus.<slug>}` sigils in pipeline options | Resolved by `kdx` at push time against the destination project's statuses. An unresolved slug logs a warning and leaves the sigil in place. |

## Project-template inline `taskStatuses:`

Still supported, and a strict **subset** of the resource. The inline entry accepts only:

```yaml
taskStatuses:
  - slug: todo
    label: To Do
    color: "#6B7280"
    icon: circle
    statusType: OPEN
    locked: false
```

`name`, `sequence`, `lockDocumentFamily`, `deprecated` and `metadata` cannot be set inline.

Semantics on project create: for each entry the platform **get-or-creates** the org row keyed
`(organization, slug)`, then binds it to the new project.

- If the slug already exists in the org, the existing row is bound and **not updated**. Colour, label
  and type changes in the template are silently ignored for that project onward.
- `statusType: TODO` on an inline entry is mapped to `OPEN` on the fly; anywhere else it is stored
  verbatim and breaks.
- `oldIdentifier` on an inline entry is a legacy id-remap aid consumed during create. Do not author it.

Use org-level `task-status` files, plus a manifest `linked:` block, whenever a status is shared across
projects or you want to change it after the project exists.

## Surfaces

| Surface | Shape |
|---|---|
| List | `GET /api/task-statuses` — standard query DSL. `?filter=project.id:'<projectId>'` narrows to the statuses **bound** to that project, which is what the reviewer-facing picker uses. |
| Read / create / update / delete | `POST /api/task-statuses`, then `GET`/`PUT`/`DELETE /api/task-statuses/{id}` — full CRUD, permission-checked per organization. Creating a slug that already exists in the org is a **409**. `DELETE` is a soft delete that also suffixes the slug with `-<uuid>`, so the original slug becomes free to re-create. |
| Resolve a URI | `POST /api/resolve?path=task-status://acme-corp/in-review` returns the ID-based API path. Soft-deleted rows do not resolve. |
| Bind to a project | `POST /api/project-resources/bind` with `{"projectId": "…", "resourceType": "task-status", "resourceUri": "task-status://acme-corp/in-review"}` (`resourceId` works in place of `resourceUri`). Idempotent — re-binding returns the existing row with `"status": "existing"`, never a second row. |
| Push a whole repo | `kdx sync push --dry-run` first, then `kdx sync push`. Statuses push at order 65 — after project templates (58) and projects (60), before assistants (70) and triggers (75). Manifest `linked:` bindings are applied after every resource has been pushed, so a status and its binding land in one run. |
| Push one file | `kdx apply -f task-statuses/in-review.yaml --type task-status --org-slug acme-corp`. `kdx sync pull` strips every organization reference from the file it writes, so `--org-slug` is always needed; `--type` supplies the routing a pulled file usually has no `type:` for. Both flags override whatever the file says. |
| Org admin UI | Organization → task statuses. Edits name, label, slug, status type, colour, icon, sequence and the two lock switches. It does not expose `deprecated` or `metadata`. |

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| A reviewer's status picker is empty for a project | The statuses exist in the org but are not bound. Check the manifest uses `projects.<slug>.linked.task-status:`, not a bare `task-status:` key under the project. |
| Take-next says there is no work, but tasks are visibly waiting | Their status is not OPEN-typed, or its slug no longer resolves (renamed, soft-deleted, or never created). PENDING tasks are reachable only through a task group. |
| An activity plan sits on a `CREATE_TASK` step forever | The child task's status slug does not resolve to a status in the task's organization, so the planner cannot judge completion and holds. Set the task to a resolvable status. |
| Creating a task group returns 422 about `statusSlug` | The slug is not bound to that project, or the status is soft-deleted. Bind it via project-resources first. |
| Completing a task returns 409 naming documents | The completion guard: the most recent save for those documents failed server-side. Reopen, redo, save, then complete. |
| A status shows in analytics as neither open nor completed | It is IN_PROGRESS-, PENDING- or BLOCKED-typed. Only OPEN counts as open and only DONE as completed; the middle three still count as unfinished, which is what drives overdue. |
| A template's colour change never reaches an existing project | Inline `taskStatuses:` is get-or-create; an existing org row is bound, never updated. Edit the org-level status instead. |
| A status disappeared from every query after an edit | `statusType` was set to a value outside the five, so every type-keyed join now misses. |
