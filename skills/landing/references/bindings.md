# Bindings, predicates and catalogs

Two lists matter for every key below. The **contract** list decides what saves. The **provider**
list decides what activates and renders. A key the contract admits may still have no provider;
see `availability` in SKILL.md.

## `field` binding

```yaml
{ kind: field, source: project, path: /options/dataProperties/vendor_id, valueType: string, missingText: "—" }
```

| Key | Rule |
|---|---|
| `source` | `organization`, `project`, `activity`, `task` or `taskGroup`. |
| `path` | A JSON Pointer (`/a/b`). Escape `/` inside a key as `~1` and `~` as `~0`. |
| `valueType` | Optional: `string`, `number`, `boolean`, `date` or `datetime`. It casts for comparison and sorting. A value that does not parse becomes null, sorts last and matches no comparison. `date` accepts only `YYYY-MM-DD`; `datetime` accepts only ISO 8601 with seconds and an offset (`2026-09-30T14:00:00Z`). |
| `missingText` | Optional, at most 80 characters. Shown for a legitimately absent value only, never for a broken binding. |

### Paths with a provider

A path outside this table **saves** but fails **activation** (`landing.provider-unavailable`).

| `source` | Fixed paths | Open-ended JSON paths (up to 6 segments) |
|---|---|---|
| `organization` | `/id`, `/name`, `/slug` | — |
| `project` | `/id`, `/name`, `/slug`, `/description`, `/createdOn`, `/updatedOn` | `/options/<…>` (e.g. `/options/dataProperties/<key>`), `/workflowContext/<…>` |
| `activity` | `/id`, `/title`, `/description`, `/createdOn`, `/updatedOn`, `/lifecycleState` | `/inputs/<…>`, `/workflowContext/<…>` |
| `task` | `/id`, `/title`, `/description`, `/statusSlug`, `/qualityState`, `/template/name`, `/priority`, `/dueDate`, `/createdOn`, `/effectiveCreatedOn`, `/assigneeId`, `/teamId` | `/properties/<…>` |
| `taskGroup` | `/id`, `/name`, `/statusSlug`, `/priority`, `/createdOn`, `/effectiveCreatedOn`, `/assigneeId`, `/teamId` | `/properties/<…>` |

Sources are checked against the row. `project` and `organization` resolve under any root. `activity`
resolves on activity rows, and on task rows through the task's creating activity. `task` and
`taskGroup` resolve only on their own rows. A `project` `/workflowContext/…` path reads the
project's latest current activity.

**Book cells are stricter than the server.** The Workflow home renderer reads a book field only from
the row's own source, plus `organization` on project rows. An activity row or activity-book
identity bound to `source: project` activates, and then the whole view renders as a configuration
error. Conveyor units have no such limit: a task unit may show `project` and `activity` fields.

## `projection` binding

`{kind: projection, key, availability}`. These are server-computed values from a closed catalog.

| Key | Row | Reads |
|---|---|---|
| `project.openTaskCount` | project | unfinished, non-retired tasks (unknown statuses count as unfinished) |
| `project.lastActivityOn` | project | the latest activity update |
| `activity.invoiceCount` | activity | distinct families in the context's `invoiceFamilyIds` that are still attached |
| `task.screenPosition`, `task.screenCount` | task | position within a task group sharing one document; blank otherwise |
| `activity.riskScore`, `.riskReason`, `.riskVersion`, `.reviewDisposition`, `.reviewReason`, `.confidence`, `.currentStage`, `.filingDueDate`, `.campaign`, `.periodKind`, `.fiscalYear`, `.fiscalQuarter`, `.filingType` | activity | the activity's **workflow context** |
| `project.currentStage`, `.filingDueDate`, `.periodKind`, `.fiscalYear`, `.fiscalQuarter`, `.filingType` | project | the workflow context of the project's latest current activity |

A projection used on the wrong row has no provider.

**Workflow context** is a small, versioned, typed record per activity:

- `period {kind: annual|quarterly|comparative, fiscalYear, quarter?, filingType?}`
- `dueDate`, `stage`, `disposition`, `campaign`
- `invoiceFamilyIds`
- `risk {score 0–100, version, reason}`
- `review {reason, confidence 0–1, ruleVersion, attemptedActions, evidenceFamilyIds}`

It is written **only** through `PUT /api/activities/{id}/workflow-context`, which the book's
`editActivityContext` action uses, and each write is kept as history. An activity plan's
`workflowContextSchema` can narrow the editor's vocabulary; it cannot add fields.

## `text` binding (identities only)

```yaml
{ kind: text, parts: [ { literal: "PO " }, { kind: field, source: project, path: /options/dataProperties/po_number } ] }
```

1–12 parts. A literal part has **no** `kind`; it is recognised by `literal` (at most 200
characters). Other parts are `field` or `projection`. It is a fixed concatenation: nothing is
evaluated, and a missing part contributes an empty string.

## `metric` binding

```yaml
{ kind: metric, key: tasks.completedToday, subject: all, window: { kind: today }, availability: optional }
```

`subject` is `all`, `me` or `myTeams`. `window` is `{kind: allTime}`, `{kind: today}` (a local day
in the landing timezone), or `{kind: lastDays, days: 1–366}`.

**Contract catalog.** Anything else is a 400:

- `tasks.`: `count`, `open`, `unfinished`, `completed`, `completedToday`, `completedByMeToday`,
  `rejectionRate`, `avgActiveReviewMs`, `medianActiveReviewMs`, `p95ActiveReviewMs`,
  `avgWallReviewMs`, `reviewActiveRatio`, `reviewBudgetMetRate`, `avgLoginToFirstDecisionMs`
- `queue.`: `depth`, `oldestAgeSeconds`, `deepest`, `daysBehind`, `behindCount`
- `people.`: `total`, `active`, `working`, `away`, `assigned`, `heldAgeSeconds`
- `workSessions.`: `avgActiveMs`, `medianActiveMs`, `avgWallMs`, `activeRatio`
- `workObservations.`: `documentTouchesPerCompletion`, `claimedEntryRate`, `avgHandoffMs`
- `quality.`: `reviews`, `goldReviews`, `goldAccuracy`
- `activities.`: `active`, `awaitingTask`, `invoiceCount`
- `projects.notStarted`, `book.rows`, `book.linkedFamilies`

**Serving rules.** A `required` binding that breaks them is refused at save:

- `queue.behindCount` has **no provider**. Bind it `optional` or not at all.
- `queue.daysBehind` needs `window: {kind: lastDays, days: N}` and `subject: all`. With no measured
  throughput it reads unavailable, never zero.
- `people.*` are current observations and need `window: {kind: allTime}`.
- `activities.*`, `projects.*` and `book.*` are served only for `subject: all` over `allTime`.
- On a **conveyor or control room**, a required stat must come from the operational families:
  `tasks.`, `queue.`, `people.`, `workSessions.`, `workObservations.` or `quality.`. This is
  checked at activation.
- `projects.notStarted` needs a project book.
- A `metric` column on a book row is never served. Required fails activation, and the book renderer
  refuses any metric column, so the view shows a configuration error either way.

## `review` binding (book `activity` columns)

```yaml
{ kind: review, taskTemplateRefs: ["task-template://${org}/invoice-review"], attempts: current,
  show: [status, assignee], eligibility: hidden, availability: optional }
```

- `taskTemplateRefs`: 1–30 refs.
- `attempts`: `current` or `all`.
- `show`: 1–5 of `status`, `assignee`, `completedBy`, `completedOn`, `eligibility`. Other values
  are accepted and draw nothing.
- `eligibility`: `server` or `hidden`. `eligibility` may appear in `show` only when this is
  `server`.
- A review column is never `sortable`.

## `dimension` binding (control-room columns)

`{kind: dimension, key, availability}`. `key` is one of `taskTemplate`, `project`, `team`,
`property`, `reviewer`, `presence` or `role`. Dimensions group by stable platform ids, so a rename
does not split or merge a group.

## Predicates

Used in `where`, `filters[].predicate` and book `stats[].where`.

| `kind` | Shape | Matches |
|---|---|---|
| `all` | `{kind: all}` | everything |
| `mine` | `{kind: mine}` | project rows: the caller owns the project or holds a task in it. Activity rows: the caller holds a task **that activity** created. |
| `activeActivities` | `{kind: activeActivities}` | activity lifecycle `RUNNING` or `PENDING` (for projects: has one) |
| `linkedProjects` | `{kind: linkedProjects}` | has a parent or child project in the same organization |
| `taskStatusType` | `{kind: taskStatusType, values: [OPEN, …]}` | **any** attached visible task has one of 1–5 of `OPEN`, `PENDING`, `IN_PROGRESS`, `BLOCKED`, `DONE`. Unknown statuses never count as `DONE`. |
| `taskStatus` | `{kind: taskStatus, refs: [task-status://…]}` | any attached task is in one of 1–30 statuses |
| `field` | `{kind: field, field: <field binding>, operator, values, relativeDate?}` | see below |

Field operators and their arity:

- `exists` takes no values.
- `in` takes 1–50 values.
- `eq`, `gte` and `lte` take exactly one value.
- `gte`/`lte` may instead take `relativeDate: {anchor: today, offsetDays: -3660…3660}`, with no
  values, on a field whose `valueType` is `date` or `datetime`. The bounds are inclusive, and the
  date resolves in the landing timezone at query time.

Values must be scalars of **one** JSON type; mixing a string and a number is a 400, and nothing is
coerced.
