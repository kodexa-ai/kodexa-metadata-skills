---
name: landing
description: "Use when authoring, deploying or debugging a Kodexa organization landing — the org-scoped Workflow home page made of book, conveyor and control-room views. Covers the nested landing document, why saving only stages it and activation freezes it, the three validation checkpoints, bindings and the field/metric catalogs, required vs optional availability, role-selected views, the task-template workPolicy that drives take-next, and `kdx landing get/set/clear`."
---

# Kodexa Organization Landings

A **landing** replaces an organization's standard Workflow tabs with authored views. There are
three surfaces:

| Surface | What it is | Actions it may carry |
|---|---|---|
| `book` | Paginated table of **project** or **activity** roots, with child activities, tasks and groups | `openProject`, `openTask`, `openEvidence`, `editProject`, `editActivityContext`, `createProject`, `startActivity`, `proposeKnowledge` |
| `conveyor` | One unit of work at a time, claimed by platform ranking | `takeNext` (**exactly one**), `resumeHeld`, `resumeDeferred`, `defer` |
| `controlRoom` | Supervisor backlog grouped by a dimension, plus a people roster | `reassign`, `reprioritize`, `releaseHeld`, `createTaskGroup`, `setDueDate`, `resumeDeferred` |

A landing is **org-scoped and single-organization**. Every view's scope is its own organization, and
every reference must point into it. A view is presentation only: rows and actions are authorized
per caller, so a landing never widens anyone's access.

## Saving stages; activation publishes

1. `kdx apply` / `kdx sync push` create or update the **landing resource** (`kdxa_landings`). That
   **stages** a definition. Nobody sees it yet.
2. **Activation** points the organization at one landing. It captures the landing's exact
   `changeSequence` and **freezes a snapshot** of the document into the organization's assignment.
   The Workflow home renders **the snapshot**, never the live resource.
3. Edits made after activation are invisible until you activate again. `kdx landing get` reports this
   as `diverged: revision N is staged but not active`.

Activate with `kdx landing set landing://acme-corp/workflow --org-slug acme-corp` (add `--dry-run`
to preview), or declare it in the sync manifest — see **kdx-cli**. `kdx landing clear` returns the
organization to the standard tabs and leaves the resource alone. Activation needs `landing:publish`
at **organization** scope; a project-level admin grant does not satisfy it.

**A landing an organization is assigned to cannot be deleted** (409). Clear or repoint the
assignment first. Translations are the one exception to the snapshot: they come from the live
resource's catalog, so they reach users without re-activation.

## Three checkpoints, three different rule sets

| When | What runs | What it misses |
|---|---|---|
| **Save** (create/update) | Strict decode + the contract validator: shapes, enums, limits, per-surface rules, reference *format* and ownership | Whether referenced projects, plans, templates and statuses **exist**; whether a field path has a **provider** |
| **Validate** — `POST /api/landings/validate`, which `kdx sync push --dry-run` calls | Contract + reference **existence** + provider **readiness** | Nothing further. Resources in the same push count as `staged`. |
| **Activate** — `kdx landing set` / manifest `landing.ref` | Contract + existence + readiness, against persisted rows only | — |

So **a landing can save cleanly and then refuse to activate** with `landing is not ready for
activation`. A field path such as `/dataProperties/x` on a project (the right one is
`/options/dataProperties/x`) is the classic case. Always dry-run before activating.

There is a fourth gate none of the three runs: **the Workflow home renderer**. It refuses to draw a
**book** whose cells it cannot render, and shows the whole view as a configuration error instead.
That covers three cases:

- a column with `rowType: task` or `taskGroup`;
- a field `source` that differs from its row, e.g. `source: project` in an activity row or in an
  activity book's identity (`organization` is fine on project rows);
- any `metric` column, even an `optional` one.

All three save and activate cleanly. Keep book cells to `project` and `activity` rows, bound to a
same-source `field`, a `projection` or a `review`.

## The resource

Landings **do not flatten**. The document lives under a nested `metadata:`, and its **whole tree
decodes strictly**: an unknown key anywhere (`allowUserChoise`) is a 400, not a silently dropped
setting.

```yaml
type: landing                 # aliases: landings, organization-landing, workflow-landing
slug: workflow
name: Accounts payable workflow
metadata:
  schemaVersion: 1            # the only accepted value
  timezone: America/New_York  # IANA name; "today" windows and relative dates resolve here
  defaultView: vendors        # must name a view; "standard" is reserved
  selection:
    allowUserChoice: true     # false hides the view switcher and ignores a saved choice
    allowStandardView: true   # allows a viewId=standard deep link back to the ordinary tabs
    rules:                    # first rule whose roles match AND whose target is eligible wins
      - { roleNames: [org-admin], view: supervisor }
  views: [...]                # 1–20 views; ids unique, ^[a-z][a-zA-Z0-9-]{0,63}$
```

Do not author `localization` or any `textKeys` map. Both are server-managed, and pull/push compare
ignores them. A complete, validated three-view document is in `references/examples.md`.

**Every view** carries `id`, `surface`, `title`, `scope`, `refreshSeconds` (15–300; there is no
default, so an omitted value fails) and `navigation`. It may also carry `subtitle`, `icon`,
`audience`, `where`, `stats` (≤8), `actions` (≤12) and `headerContext`. `navigation.onCancel` is
always `landing`. `navigation.onComplete: takeNext` is legal **only on a conveyor**; books and
control rooms must say `landing`.

```yaml
scope:
  organizations: { mode: current }           # the only mode
  projects: { mode: allActive }              # or { mode: include, refs: [project://${org}/ap-east] }
```

Per-surface fields and limits: `references/views.md`.

## References: `${org}`, never `${orgSlug}`

Every reference is `scheme://org/slug`, where `org` is the literal owning slug or `${org}`.
`project://`, `project-template://`, `activity-plan://`, `task-template://` and `task-status://`
appear in `scope.projects.refs`, `taskStatus` predicates, review `taskTemplateRefs`, and the
`activityPlanRef` / `projectTemplateRef` of actions. **`${orgSlug}` is a contract error in a
landing**, although task-template `workPolicy` accepts it. A reference into another organization is
refused at save.

`startActivity` and `proposeKnowledge` buttons are enabled per row only where the plan is **bound
to that row's project**. Otherwise they render disabled (`activity_plan_not_bound_to_project`).

## Bindings — how a cell gets its value

| `kind` | Where | Reads |
|---|---|---|
| `field` | columns, identities, predicates, facets, search | `source` (`organization`, `project`, `activity`, `task`, `taskGroup`) + a **JSON Pointer** `path` |
| `projection` | columns, identities | a named server value from a closed catalog (`project.openTaskCount`, `activity.riskScore`, …) |
| `metric` | stats; control-room columns | a closed metric catalog, with `subject` (`all`/`me`/`myTeams`) and `window` |
| `review` | book **activity** columns only | task state for `taskTemplateRefs`: `status`, `assignee`, `completedBy`, `completedOn`, `eligibility` |
| `dimension` | control-room columns only | a grouping identity: `taskTemplate`, `project`, `team`, `property`, `reviewer`, `presence`, `role` |
| `text` | identities only | a fixed concatenation of `literal`, `field` and `projection` parts, not a template |

Paths are JSON Pointers, so escape a `/` inside a key as `~1`. Only a fixed catalog of paths has a
**provider**. For example, `project` has `/name`, `/slug`, `/options/<…>`, `/workflowContext/<…>`;
`activity` has `/title`, `/inputs/<…>`; `task` has `/title`, `/template/name`, `/properties/<…>`.
A path outside the catalog passes save and fails activation. The full catalogs, and which keys a
provider actually serves, are in `references/bindings.md`.

**The `activity.*` and `project.*` projections read the activity's *workflow context*.** Only the
`editActivityContext` action (or `PUT /api/activities/{id}/workflow-context`) writes it; nothing in
a plan run does. Bind them `optional` unless your process records that context.

## `availability: required` vs `optional`

Stats, metrics, projections, dimensions, reviews and actions each declare an `availability`.

- `optional`: an unserved value renders as an explained blank or a disabled control.
- `required`: an unserved value is **refused**. A required metric with no provider fails at
  **save**. A required projection or dimension with no provider fails at **activation**, and so does
  a `field` with no provider (fields have no `availability`). A required binding or action the
  renderer cannot draw makes **the whole view** render as a configuration error.

Mark `required` only what the view is meaningless without.

## Audiences and selection

`audience.roleNames` and `selection.rules[].roleNames` match **effective role names**, such as
`org-admin` and `project-admin`, held on the organization or on a project in the view's scope. One
listed role is enough. Role names are only checked for shape, so **a misspelled role is accepted
and silently hides the view** from everyone who lacks a real match. A view with no audience is
offered to every Workflow reader. The server opens the first matching rule's view, then
`defaultView`, then the first eligible view. A deep link or saved choice wins over all three.

## Take-next behaviour lives on the task template

A conveyor has no ordering or eligibility settings (`queue` is fixed to
`{workUnits: taskAndGroup, ordering: platform}`). **Who may claim what is the task template's
`metadata.workPolicy`**: `maxHeldWork`, `leaseMinutes`, `requiredSkills`,
`distinctFromTaskTemplateRefs`, `allowedTransitions`, `sla`, `routing` and `quality`. It is enforced
by a **database guard on every assignment and completion write**, not only on landings.

Adding `requiredSkills` before any member has been granted the skill (from the control room's person
row) empties the queue for **everyone** (`required_skill_missing`). Once `allowedTransitions` is
present, **every** status change it does not list is refused, including changes out of a status it
has no key for (`status_transition_not_allowed`). Every key except `quality` is decoded leniently, so
a misspelt policy key is **silently dropped**. See `references/work-policy.md`.

## Declared but inert

None found: the validator, query engine or renderer reads every authored field. Some accept one
value only (`schemaVersion: 1`, `orphanTasks: separate`, `queue.workUnits: taskAndGroup`,
`queue.ordering: platform`, `nulls: last`, `onCancel: landing`, `organizations.mode: current`).

## Common mistakes

| Mistake | What happens / fix |
|---|---|
| `metadata` fields written flat at the top level | Landings do not flatten. The empty document fails validation. |
| `activity-plan://${orgSlug}/…` in a landing | Contract error. Use `${org}` or the literal slug. |
| Project data path `/dataProperties/x` | Saves; fails activation (`provider-unavailable`). Use `/options/dataProperties/x`. |
| A required metric no provider serves (`queue.behindCount`) | 400 at save. Bind it `optional`. |
| `queue.daysBehind` without `window: {kind: lastDays, days: N}` and `subject: all` | Unserved. People metrics need `window: {kind: allTime}`. |
| A `metric` column on a book (required or optional) | Required fails activation; either way the book renders as a configuration error. Use a stat or a projection. |
| A book column with `rowType: task`/`taskGroup`, or a field `source` that differs from its row | Saves and activates; the view renders as a configuration error. |
| Two `takeNext` actions, or none, on a conveyor | Contract error: exactly one. |
| `onComplete: takeNext` on a book or control room | Contract error. Only conveyors auto-advance. |
| `review` binding on a project column, or `sortable: true` on it | Contract error. Reviews belong to activity rows and have no scalar sort. |
| `show: [eligibility]` with `eligibility: hidden` (or the reverse) | Contract error. Showing eligibility requires `eligibility: server`. |
| A `show` value that is not one of the five | Accepted, draws nothing. Only `status`, `assignee`, `completedBy`, `completedOn`, `eligibility`. |
| Misspelt role in `audience` / `selection.rules` | Accepted; the view is offered to nobody who lacks a real match. |
| `requiredSkills` added before any grants exist | Conveyor is empty for everyone. Grant skills first. |

## Related skills

- **kdx-cli**: `kdx landing get/set/clear`, the `landing` sync type, the manifest's `landing: {ref}`
  block and `--landing-revision` on pull.
- **task-template**: the templates a conveyor dispatches and a review column reports on. Their
  `workPolicy` is covered here.
- **project-template**: `options.dataSchema` / `dataOptions`, which drive the `editProject` form.
- **activity-plan**: the plans `startActivity` and `proposeKnowledge` launch.
- **project-resource**: binding those plans to each project in scope.
- **task-status**: the `statusType` values `taskStatusType` predicates match.
