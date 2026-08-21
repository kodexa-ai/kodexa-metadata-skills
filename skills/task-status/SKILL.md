---
name: task-status
description: "Use when defining or changing the workflow states Kodexa tasks move through — authoring task-statuses/*.yaml, picking statusType (OPEN / IN_PROGRESS / PENDING / BLOCKED / DONE), configuring locked and lockDocumentFamily, retiring a state, or binding org-level statuses to a project so reviewers can reach them."
---

# Kodexa Task Status Authoring

A **task status** is one workflow state a task can sit in — "To Do", "In Review", "Signed Off".

Since the 2026-05-02 activity refactor, statuses are **org-scoped**: one row per `(organization, slug)`
in `kdxa_task_statuses`, shared by every project in the org. A project's "workflow" is not a list the
project owns — it is **the set of org statuses bound to that project**. A status that is not bound
still exists, but no reviewer in that project can select it.

Tasks store the status as a **bare slug** with no foreign key. That single fact causes most of the
traps below.

## The file

One YAML per status at `task-statuses/<slug>.yaml`. Minimum that works:

```yaml
# task-statuses/in-review.yaml
slug: in-review
name: In Review
label: In Review
statusType: IN_PROGRESS
```

Only `slug` is structurally required by the API (and it is derived from `name` if you omit it — a
create with neither is a 400). Everything else is *practically* required: without `label` the chip a
reviewer sees is blank, and without `statusType` the status silently drops out of every behaviour on
this page.

**Never write `organizationId` or an `${orgSlug}` placeholder.** `kdx` stamps the target organization
onto every org-scoped push itself, and that nested stamp wins over anything flat in the file. The only
substitution `kdx` performs in a task-status file is `${org}`; `${orgSlug}` is left untouched and
pushes as a literal string.

```yaml
organizationId: ${orgSlug}   # WRONG — kdx supplies the org; this value is discarded
```

Slugs are lowercase letters, digits and hyphens (mixed case is downcased). Pushing a file whose slug
already exists updates that row; a create that collides returns 409.

## A realistic workflow

A four-state invoice review is four files: `todo` (OPEN, sequence 10), `awaiting-vendor` (PENDING,
20), `in-review` (IN_PROGRESS, 30), `signed-off` (DONE, 40). `sequence` orders the org admin list
within each `statusType` group and breaks the tie when a task group auto-reopens. It does **not**
reorder a reviewer's status dropdown — that lists whatever the API returns, unsorted.

```yaml
# task-statuses/signed-off.yaml
slug: signed-off
name: Signed Off
label: Signed Off
color: "#10B981"
statusType: DONE
sequence: 40
locked: true
lockDocumentFamily: false
```

## `statusType` — the five values and what each one actually changes

| Value | What it changes |
|---|---|
| `OPEN` | The **only** type take-next hands out. An unassigned, ungrouped task in an OPEN status enters the individual pool; a task group whose own status is OPEN enters the group pool. Counts as "open" in analytics. |
| `IN_PROGRESS` | Excluded from take-next. Preferred target when a task group auto-reopens: the lowest-`sequence` bound IN_PROGRESS status, else the lowest-`sequence` bound OPEN one. |
| `PENDING` | Excluded from take-next, **including as an individual task**. A task parked in a PENDING status reaches a reviewer only through a task group. This is the whole point of the type. |
| `BLOCKED` | Excluded from take-next and never chosen as a reopen target. Neither open nor completed in analytics, but still counts as unfinished for overdue metrics. |
| `DONE` | The completion boundary — see below. |

`DONE` is a cross-resource contract, not a colour:

- The task-completion guard fires on every analyst-driven transition **into** a DONE-typed status and
  can return 409 naming documents whose last save failed server-side.
- Task-group done counters and auto-complete count DONE-typed member tasks.
- Analytics counts the task closed.
- An activity-plan `CREATE_TASK` step advances only when its task reaches a DONE-typed status.
- Every DONE-typed slug in the org becomes a resolvable completion token for the `dependsOn` action
  qualifier on an edge out of a `CREATE_TASK` step. Non-DONE slugs are deliberately excluded — an edge
  to one could never fire, so it is rejected when the plan starts.

**Unknown values are accepted and silently disable all of it.** The column is a plain `varchar(20)`
with no constraint, so `statusType: TODO` (the pre-refactor spelling) or `statusType: REVIEW` stores
fine and then drops the status out of every type-keyed query: no take-next, no counters, no analytics,
no plan advance. Existing `TODO` rows were retyped to `OPEN` by migration; do not author it again.

## `locked` and `lockDocumentFamily`

`locked` is an **entry** lock: arriving in this status flips the task to read-only. It is not a guard
on transitions *away* from the status — nothing stops a reviewer moving the task out again.

```yaml
locked: true                # entering this status makes the task read-only
lockDocumentFamily: false   # ...but leaves the attached document families editable
```

`lockDocumentFamily` defaults to **true**: the lock cascades to every document family attached to the
task. Set it `false` for a sign-off state that should freeze the human action while downstream
extraction keeps writing to the document family.

Both defaults can be overridden per transition by the caller (a task-template action carries
`lockTask` / `lockDocumentFamily` in its `properties`), and the request value wins over the status.
So a status flagged `locked: false` can still lock, and vice versa — the status is the default, not
the rule. One asymmetry to know: a status change driven by an activity-plan script honours `locked`
but ignores `lockDocumentFamily`, locking the attached document families unconditionally.

## Binding a status to a project

An org status is inert in a project until bound. Three real paths:

1. **Project-template inline `taskStatuses:`** — on project create the platform get-or-creates the org
   row by `(organization, slug)` and binds it. If the slug already exists, the existing row is bound
   **and not updated** — colour/label/type edits in the template are silently ignored.
2. **Manifest `linked:` block** — the path to use for a status authored as its own YAML file.
3. **`POST /api/project-resources/bind`** for a one-off.

```yaml
# manifest — RIGHT: a linked: entry both pushes the file and binds it
organization:
  task-status:
    - escalated              # pushed to the org only; bind it to a project separately
projects:
  invoice-review:
    linked:
      task-status:
        - todo
        - awaiting-vendor
        - signed-off
```

```yaml
# manifest — WRONG: pushes the statuses, creates no binding at all
projects:
  invoice-review:
    task-status:
      - todo
```

The wrong form is silent: the statuses land in the org, the push reports success, and the project's
status picker stays empty.

## Fail-open vs fail-closed on an unknown slug

The two places that consume a status slug behave in **opposite** ways. Know which one you are in.

- **Activity-plan `CREATE_TASK` → `taskStatusSlug` fails OPEN.** An unresolvable slug is not an error.
  The task is materialized with an empty status, and then: it never appears in take-next, it counts as
  neither open nor completed in analytics (only as unfinished, so it can go overdue), and the plan step
  holds forever because the planner cannot judge completion. Nothing surfaces until someone asks where
  the work went.
- **Task groups fail CLOSED on `statusSlug`.** Creating a task group, or updating a group's status,
  with a slug that is not bound to that project (or is soft-deleted) returns **422**: *statusSlug "x"
  is not bound to project …; bind the task-status via project-resources first.* A group's
  `doneStatusSlug` is **not** validated — an unresolvable one is silently replaced at auto-complete
  time by the most common DONE-typed slug among the group's completed member tasks.

## Renaming and deleting

Because tasks hold a bare slug and nothing enforces referential integrity:

- **Renaming a slug orphans every in-flight task holding the old one.** Treat it as an org-wide rename
  and update task templates, activity plans and task groups in the same change.
- **Deleting is a soft delete that appends `-<uuid>` to the slug**, freeing the original for
  re-creation — and orphaning in-flight tasks exactly the same way.
- **Changing `statusType` on a live status retroactively re-categorises every task holding it.**
  Flipping to or from `OPEN` adds or removes all of them from take-next; flipping to `DONE`
  retroactively closes them in analytics and can auto-complete their task groups.

## Declared but inert

Persisted and round-tripped, but nothing in the platform acts on them for a task status:

| Field | Note |
|---|---|
| `icon` | Stored and returned, but no surface renders it — the task chip draws `label` on `color`, the org admin list draws a colour dot. |
| `metadata` | Free-form JSON. Nothing reads it. |
| `deprecated` | Stored; no listing or picker filters on it. Use it as a note to authors, not as a hide switch. |
| `template`, `publicAccess`, `extensionPackRef` | Inherited from the shared metadata base; unused on this resource. |
| `type` | Back-filled to `task-status` by an old migration; nothing sets it on new rows and no platform behaviour reads it. Only `kdx apply` looks at a file-level `type:`, purely to route the file — `--type` replaces it. |
| `oldIdentifier` | Only meaningful on the project-template inline entry, as a legacy id-remap aid. Never author it. |

`projectId` / `project` are **not** inert and were never removed: they are accepted on create as a
shortcut, and project scope wins over an explicit `organizationId`. They have no column, so they never
come back on a read.

## Reference

`references/reference.md` — full field table, the wire-format record, every resource that points at a
status slug, the project-template inline subset, and the REST / `kdx` surface.

Related: `task-template` (which status a task starts in and which its action buttons move it to) ·
`activity-plan` (`CREATE_TASK` and `dependsOn`) · `project-resource` (bindings) · `project-template`
(inline provisioning) · `kdx-cli` (manifests and push).
