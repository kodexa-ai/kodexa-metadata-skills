# Landing views — field reference

Every limit here is enforced when the landing is **saved**. The contract validator reports every
problem at once, as JSON Pointers into the request body (`/metadata/views/0/...`). "Required" means
the save fails without it.

## Root (`metadata`)

| Key | Rule |
|---|---|
| `schemaVersion` | Required, `1`. |
| `timezone` | Required. An IANA name of at most 80 characters. `today` windows, relative dates and header shifts resolve here. |
| `defaultView` | Required. Must name a view. |
| `selection.allowUserChoice` | `false` hides the view switcher and ignores a saved choice. It does not block an eligible deep link. |
| `selection.allowStandardView` | Allows a `viewId=standard` deep link to the ordinary Workflow tabs. It never restores tabs the organization's features disabled. |
| `selection.rules[]` | At most 20. Each has `roleNames` (1–30 tokens `^[a-zA-Z0-9_-]{1,100}$`) and a `view` that must exist. |
| `views[]` | 1–20. Ids are unique and match `^[a-z][a-zA-Z0-9-]{0,63}$`; `standard` is reserved. |
| `localization` | Server-managed. Do not author it. |

## Shared by every view

| Key | Rule |
|---|---|
| `id`, `surface`, `title` | Required. `surface` is `book`, `conveyor` or `controlRoom`. `title` is at most 500 characters. |
| `subtitle` | Optional, at most 500 characters. |
| `icon` | Optional, `^[a-z0-9-]{1,80}$` (a Material Design icon name). Shown in the view switcher. |
| `audience.roleNames` | Optional, 1–30 role-name tokens. No audience means everyone who can read the Workflow home. |
| `scope.organizations` | `{mode: current}`. It is the only mode. |
| `scope.projects` | `{mode: allActive}`, or `{mode: include, refs: [project://…]}` with 1–100 unique refs. A missing referenced project narrows the scope; it never widens back to every project. |
| `where[]` | Optional, at most 20 predicates. They are ANDed with scope and with any selected filter. |
| `stats[]` | At most 8. See **Stats**. |
| `actions[]` | At most 12. `kind` must be one the surface allows (see SKILL.md). Each has `id`, `label` and `availability` (`required`/`optional`). |
| `refreshSeconds` | Required, 15–300. |
| `navigation` | Required. `onComplete` is `landing` or `takeNext` (conveyor only). `onCancel` is `landing`. |
| `headerContext` | Optional. `greeting`, `teams` and `campaigns` are required booleans, and `shifts` is a required array (≤20) of `{id, label, weekdays: [1–7 ISO], startMinute, endMinute}` with distinct minutes within one day. Present keys must be non-null, even when false or empty. |

### Stats

`{id, label, binding, scope, format, tone?, where?}`. `binding` is always a `metric` binding
(`references/bindings.md`). `scope` is `view` (the whole view population) or `filtered` (the
current filter), independent of the page. `format` is `number`, `percent` or `duration`. `tone` is
`neutral`, `warn`, `info` or `escalate`. `where` (≤20 predicates) is allowed **only on a book**.

### Action variants

| `kind` | Extra keys |
|---|---|
| `createProject` | `projectTemplateRef` (optional; without it the project marketplace opens) |
| `startActivity`, `proposeKnowledge` | `activityPlanRef` (required) |
| `defer` | `requireReason: true` (must be true) and `reasonCodes` (1–30 tokens) |
| every other kind | none |

`proposeKnowledge` starts the named plan in the selected activity's project. Its inputs are exactly
`proposalId`, `sourceActivityId`, `sourceActivityVersion`, `reason`, `proposedRule` and
`evidenceFamilyIds`. If the plan declares `inputsSchema` or required `inputOptions`, they must
accept that object. Evidence must be sealed revisions linked to the source activity.

## `book`

| Key | Rule |
|---|---|
| `entity` | `project` or `activity`: the root row type. |
| `hierarchy` | `projectLinks` (parent/child projects; project books only) or `none`. |
| `history` | `current` (drops replaced or superseded attempts and historical tasks) or `all`. |
| `orphanTasks` | `separate`: tasks with no activity are shown on their own. |
| `identity.label` | Required identity value. `key` and `context` are optional. Values are `field`, `projection` or `text` bindings. |
| `labels` | Required `singular`, `plural` and `search` texts (at most 500 characters each). |
| `columns[]` | At most 20, with unique ids. See **Columns**. |
| `filters[]` | 1–12 `{id, label, predicate}`. They are named presets ANDed with the view. |
| `defaultFilter` | Required. Must name a filter. |
| `facets[]` | At most 8 `{id, label, field, multiple, options: [{label, value}]}`, with 1–50 options of unique values. Each facet is checked as an `in` predicate on its field. |
| `searchFields[]` | 1–12 `field` bindings. |
| `sort[]` | At most 3 `{columnId, direction: asc|desc, nulls: last}`. The column must be `sortable`, and its `rowType` must equal `entity`: a root cannot sort by a child column. |
| `pagination` | `{defaultSize, sizes}`: 1–4 unique sizes, with `defaultSize` among them. |

The contract lets a project book declare `project`, `activity`, `task` and `taskGroup` columns, and
an activity book every one of those except `project`. `dimension` bindings are refused in a book.
**The renderer draws only `project` and `activity` columns bound to a same-source `field`, a
`projection` or a `review`.** A `task`/`taskGroup` column, a cross-source field, or any `metric`
column saves and activates, and then the view renders as a configuration error.

## `conveyor`

| Key | Rule |
|---|---|
| `unit.task` | Identity for a task unit. `title` is required; `badge`, `subtitle` and `context` are optional. Field sources must be `task`, `activity`, `project` or `organization`. |
| `unit.group` | Identity for a task-group unit. Field sources must be `taskGroup`, `project` or `organization`, because a group is never shown as one of its member tasks. |
| `queue` | `{workUnits: taskAndGroup, ordering: platform}`. Both values are fixed. |
| `heldWork` | `{enabled, showAge}`. `enabled` governs the held-work panel only; recovering an uncertain claim always works. `showAge` shows the actual assignment time. |
| `actions` | Exactly one `takeNext`. |

The server ranks eligible tasks and groups across the scope, applying the view's predicates to both
the preview and the atomic claim. Eligibility per reviewer comes from the task template's
`workPolicy` (`references/work-policy.md`).

## `controlRoom`

| Key | Rule |
|---|---|
| `queue.title` | Required. |
| `queue.groupBy` | `taskTemplate`, `project`, `team` or `property`. |
| `queue.groupField` | Required exactly when `groupBy: property`. It must be a `field` binding on `project` `/options/dataProperties/<…>`, or on `task`/`taskGroup` `/properties/<…>`. |
| `queue.columns[]` | 1–12. Every one has `rowType: queue` and is bound to a `metric` or `dimension`. |
| `people.title` | Required. |
| `people.dataset` | `assignments` (users holding unfinished work in scope), `workSessions` (actors with recorded sessions in the metric windows) or `organizationMembers`. |
| `people.columns[]` | 1–12. Every one has `rowType: person` and is bound to a `metric` or `dimension`. |
| `queue.sort`, `people.sort`, `*.pagination` | As for a book, without the root-row rule. |

At activation, a **required** dimension column must supply its table: the queue's `groupBy` key in
`queue`, and `reviewer`, `presence`, `team` or `role` in `people`. `queue.*` metrics cannot be
attributed to a person. `people.total/active/working/away` cannot be attributed to a queue.

## Columns

`{id, label, rowType, binding, format, currency?, align?, sortable}`.

| Key | Rule |
|---|---|
| `rowType` | `project`, `activity`, `task`, `taskGroup`, `queue` or `person`, limited per surface as above. |
| `format` | `text`, `number`, `currency`, `percent`, `date`, `datetime`, `duration` or `badge`. |
| `currency` | Required with `format: currency` and forbidden otherwise. A three-letter ISO 4217 code. |
| `align` | `left` or `right`. |
| `sortable` | `false` on a `review` binding. |
| `binding` | `field`, `projection`, `metric`, `review` or `dimension` (`references/bindings.md`). `review` only on `rowType: activity`. |

## Project data editing (`editProject`)

A book's `editProject` action opens a form over the project's name, description, status and
`options.dataProperties`. The server builds the form's schema:

- **From the project's `options.dataSchema`** when its template supplies one. It must be a JSON
  Schema with `type: object`.
- **Otherwise from `options.dataOptions`.** Supported types are `string`, `text`, `select`,
  `integer`, `number`, `boolean`, `date` and `datetime`; `possibleValues` become an `enum` and
  `required` is honoured. **Any other type makes the editor refuse to open** (`data option … requires
  a dataSchema for this editor`). Declare a `dataSchema` instead.

The derived schema is **closed** (`additionalProperties: false`), and a save replaces
`dataProperties` wholesale. The form sends back every current value, so a project holding a
`dataProperties` key its template does not declare **cannot be saved** from this editor (`project
data does not match its schema`) until the key is declared. Saves are optimistic on the project's
`changeSequence` (409 on a concurrent edit), and each save is kept as history.
