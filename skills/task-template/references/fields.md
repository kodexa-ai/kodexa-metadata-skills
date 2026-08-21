# Task template fields

Every key below is live unless the SKILL's "Declared but inert" table says otherwise.

## Top level

| Key | Notes |
|---|---|
| `type` | CLI resource-type discriminator. `taskTemplate`, `task-template`, `task-templates`, `taskTemplates` and `tasktemplate` are all accepted. Required unless you pass `--type`. |
| `orgSlug` | Target organization slug. Required unless you pass `--org-slug`. Not project-scoped — there is no `projectSlug` for task templates. |
| `slug` | Identity within the org. Validated `min=3, max=100`. (The column is wider only to absorb the suffix appended on soft delete.) |
| `name` | Canonical display name. |
| `title` | Legacy column. Still read directly for the `{templateName}` AI-naming placeholder (activity `CREATE_TASK` and intake) and for the activity `CREATE_TASK` no-title fallback — set it to the same value as `name`. |
| `description` | Free text. |
| `template` | `true` marks a starter/library template; renders a badge in Studio. |
| `publicAccess` | Standard flag; renders a badge. |
| `deprecated` | Standard flag; renders a "Deprecated" badge. Nothing is gated on it — existing tasks keep working. |
| `metadata` | Everything below. |

The canonical key order the CLI writes on pull is `slug, name, description, metadata`.

## `metadata.properties` — the behaviour switchboard

A bare map the server stores and echoes untouched, and the **only** region where deleting a key
locally is pushed as a deletion — provided the file still carries a `properties:` node. Drop the
whole block and the region stops counting as authored, so nothing is removed; spell a full clear
as `properties: {}`. This is where the day-to-day "why is that button still there" switches live.

```yaml
metadata:
  properties:
    hideDefaultSaveButton: false
    hideDefaultCancelButton: true
    hideLock: false
    lockStatusSlug: signed-off
    hideTaskPanels: false
    collapseTaskPanelsOnOpen: true
    panels: { documentStores: true, formula: false }
    visiblePanels: { documentStores: true, exceptions: true }
    subTaskOnly: false
    enableChat: true
    saveShortcut: "meta+s"
    cancelShortcut: "escape"
    enableSummarization: true
    summarizePrompt: "Summarise this invoice in three bullets."
    enableValidation: false
    showAlert: true
    alertType: WARNING
    alertTitle: "Check the vendor code"
    alertBody: "This template feeds downstream payment."
```

| Key | Type | Effect |
|---|---|---|
| `hideDefaultSaveButton` | bool | Removes the toolbar's Save button. |
| `hideDefaultCancelButton` | bool | Removes the toolbar's Cancel button. |
| `hideLock` | bool | Hides both Lock and Unlock buttons. |
| `hideFloatingHelper` | bool | Hides the floating workspace helper on tasks driven by this template (both the single-task and task-group workspaces). |
| `lockStatusSlug` | string | Turns Lock into a status transition to this slug instead of the plain lock call. On a `DONE`-typed status that also completes the owning activity. Unlock never reverts the status, and re-locking a task already at that slug falls back to the plain lock call. |
| `hideTaskPanels` | bool | Removes the panels group and its toggle. |
| `collapseTaskPanelsOnOpen` | bool | Panels start collapsed but stay reachable from the toggle. |
| `panels` | map of panel id → bool | Per-panel availability override applied on workspace load. |
| `visiblePanels` | map of panel id → bool | Opt-in **whitelist** — when present, only panels set `true` are shown. Absent means no filtering. |
| `subTaskOnly` | bool | Hides the template from the New Task wizard. |
| `enableChat` | bool | Shows the chat input on the task's channel. |
| `saveShortcut`, `saveShortcutAltKey` | string | Keyboard shortcut for Save. Ignored while `hideDefaultSaveButton` is set. |
| `cancelShortcut`, `cancelShortcutAltKey` | string | Same for Cancel. |
| `enableSummarization` + `summarizePrompt` | bool + string | Runs the prompt over uploaded PDFs during task creation. The prompt is ignored unless the flag is on. |
| `enableValidation` + `validatePrompt` | bool + string | Same pairing for validation. |
| `showAlert`, `alertType`, `alertTitle`, `alertBody` | bool/string | Banner on the task creation form. `alertType` is `INFORMATION` (default look) or `WARNING`. |

Panel ids usable in `panels` / `visiblePanels` / `forms[].availablePanels`: `documentStores`,
`properties`, `exceptions`, `navigation`, `auditNotes`, `projectNotes`, `taskNotes`, `taskTimeline`,
`projectOverview`, `projectOptions`, `formula`, `taskDocuments`.

## `metadata.forms[]`

```yaml
metadata:
  forms:
    - dataFormRef: "${org}/invoice-form"          # or a concrete acme-corp/invoice-form
      availablePanels:
        documentStores: true
        exceptions: true
        properties: false
```

- `dataFormRef` is a **reference**, not an embedded form. It is also the one and only field where
  the server-side `${orgSlug}` placeholder is expanded (on save). Prefer `${org}/` anyway: `kdx`
  resolves it before the push and `kdx sync pull` writes it back, so `${org}` round-trips and
  `${orgSlug}` drifts against the file on every pull.
- The reference is matched by string equality against the **project's** data forms (a version suffix
  on either side is tolerated here). If the form is not bound to the project the task lives in, the
  form pane simply never appears — a warning is logged and nothing else happens.
- `availablePanels` overrides workspace panel visibility for that form, using the panel ids above.

## `metadata.documentFamilyGroups[]`

Groups of documents the reviewer attaches. These drive the **New Task wizard** — the upload zones,
filters and limits are applied there. (On an already-open task only `uniqueFilenames` is consulted.)

```yaml
metadata:
  documentFamilyGroups:
    - name: "Source Invoices"
      notes: "Primary documents for review"
      documentFamilyFilter: "*.pdf OR *.tiff"
      maxHits: 5
      sort: "created:desc"
      automaticallyAdd: true
      editable: true
      uploadOnly: false
      uniqueFilenames: true
      maxPages: 100
      hardMaxPages: 500
      maxSize: 10485760
      titlePrompt: "Name this task from the invoice number and vendor."
```

| Key | Behaviour |
|---|---|
| `documentFamilyFilter` | **Extension whitelist only**, OR-joined (`*.pdf OR *.docx`); a bare `.pdf` also works. Anything the parser cannot read as an extension leaves the list empty, which means *allow every file*. Path and glob expressions do not work. |
| `maxHits` | Cap on how many families the group holds; auto-add and upload both respect it. |
| `sort` | `<field>:<dir>` where field is `path`, `created`, `modified` or `size` and dir is `asc` or `desc`. |
| `automaticallyAdd` | Pre-selects every matching existing family (up to `maxHits`). |
| `editable` | Defaults to `true`. `false` stops the user changing the selection. |
| `uploadOnly` | Hides selection of existing documents — upload only. |
| `uniqueFilenames` | Inserts an 8-character UUID before the extension on upload, so the same file can be uploaded to several tasks. It enforces nothing and rejects nothing. |
| `maxSize` | Bytes. Over the limit shows a confirm dialog; the user can proceed. |
| `maxPages` | PDF page count. **Warns** and lets the user proceed. |
| `hardMaxPages` | PDF page count. **Rejects** the upload. |
| `titlePrompt` | Sends this prompt plus the selected file names to the model and uses the reply as the task title, unless the user has typed one. It has **no** `{placeholder}` vocabulary. The first group that declares one wins for the whole wizard, and `aiNaming` takes priority over it entirely. |

`required` and `knowledgeFeatures` exist on the group shape but are only honoured for groups declared
on an **activity plan**. On a task template they do nothing.

## `metadata.aiNaming`

```yaml
metadata:
  aiNaming:
    enabled: true
    prompt: "Title this task using the vendor from {metadata:vendor.name} and {documentFamilyPaths}."
```

The model is wrapped in a JSON-only instruction and must return `{"title": …, "description": …}`; the
description is used only if none was supplied. When it fires depends on the path:

- **Activity `CREATE_TASK`** and the **create-task API** — only when no title was supplied.
- **Intake** — whenever `enabled` is true. The title starts as the uploaded filename and AI naming
  overwrites it; if the model call or the JSON parse fails, the filename stands.
- **New Task wizard** — on template selection and again whenever the document selection changes,
  unless the user has edited *both* the title and the description. It takes priority over a document
  group's `titlePrompt`.

Placeholders are single-brace. Unknown ones pass through verbatim. Which ones resolve depends on
where the task is created:

| Placeholder | Resolves to | Server paths (intake, plan step, create API) | New Task wizard |
|---|---|---|---|
| `{templateName}` (alias `{activityPlanName}`) | The template's name | Yes — from the `title` column on the intake and `CREATE_TASK` paths, `name` falling back to `title` on the create API | Yes, from `name`; no alias |
| `{documentFamilyPaths}` | `", "`-joined paths | Yes | Yes |
| `{documentFamilyCount}` | Count of the above | Yes | Yes |
| `{knowledgeFeatures}` | JSON array of the attached features (`[]` when empty) | Yes | Always renders empty |
| `{documentMetadata}` | JSON array of family metadata, `"No metadata available"` when empty | Intake and create API only — **always `"No metadata available"` on the activity `CREATE_TASK` path** | Left as literal text |
| `{metadata:some.path}` | Query across attached families' metadata, deduped, `"; "`-joined | Intake and create API only — **always empty on the activity `CREATE_TASK` path** | Left as literal text |
| `{externalData:key.some.path}` | Query into the family's external data | Intake and create API only — **always empty on the `CREATE_TASK` path** | Left as literal text |

Practical consequence: a prompt that leans on `{metadata:…}` works from an intake and silently
degrades to nothing when the same template is driven by an activity plan.

## `metadata.chatPrompt`

```yaml
metadata:
  chatPrompt:
    enabled: true
    prompt: "Help me review {taskTitle}. The attached documents are {documentFamilyPaths}."
```

Rendered client-side with a **different**, smaller vocabulary than `aiNaming`: `{taskTitle}`,
`{taskDescription}`, `{assignee}` (display name, else first name, else `"Unassigned"`), `{templateName}`,
`{documentFamilyPaths}`, `{documentFamilyCount}`, and `{knowledgeFeatures}` — which is hard-coded to
render as an empty string here. There is no `{metadata:…}` or `{externalData:…}` support.

It fires when the task opens and **only if the current user has no channel on this task yet**. It
creates a private channel named "Task Assistance" and posts the rendered prompt as the first message.

## `metadata.agentShortcuts[]`

```yaml
metadata:
  agentShortcuts:
    - id: summarise-invoice
      label: "Summarise"
      icon: text-box-outline
      prompt: "Summarise the attached invoice and flag anything unusual."
```

Each entry renders a button in the task header. Clicking it saves any pending changes (after a
confirm), opens a **new private channel** named after the label, and seeds it with `prompt`. `id` and
`label` and `prompt` are required; `icon` is a Material Design icon name defaulting to `robot`. The
buttons are hidden while the task is locked, and disabled while the task is loading or another
shortcut is still running.

## `metadata.teamSlug` and `metadata.priority`

```yaml
metadata:
  teamSlug: finance-reviewers
  priority: 2
```

`teamSlug` is looked up in the org's teams and becomes the new task's team — but **only when the
caller supplied no explicit team**. It applies on the create API and the intake path, and the New Task
form pre-selects the matching team. The activity `CREATE_TASK` insert does not write a team at all, so
activity-created tasks ignore `teamSlug` entirely. There is no auto-assign-to-a-user mechanism.

`priority` is `1` Highest, `2` High, `3` Medium, `4` Low, `5` Lowest; `0`/unset is "Not Set". **Lower
is ranked earlier.** The template's value is only read by the New Task form (defaulting to `3`); the
create API and intake take priority from the request, and an activity `CREATE_TASK` step takes it from
the step's `taskData.priority` or the activity's own priority.

## `metadata.companion`

```yaml
metadata:
  companion:
    moduleRefs:
      - "${org}/review-helpers"
```

`moduleRefs` is the only key the server acts on: the refs are merged (de-duplicated, after the agent's
own) into the module set for agent invocations on tasks from this template. `agentRuntimeRef` and
`prompt` are persisted and shown in Studio but are not read — runtime selection comes from the
project's companion configuration.

## How tasks get created

Three paths, with different behaviour:

1. **Manually / via the create-task API.** Status, priority, team and title come from the request; the
   template supplies `teamSlug` and `aiNaming` fallbacks. The Studio New Task wizard is this path, and
   it is also where document groups, the alert banner and summarize/validate run.
2. **Intake.** The target project is resolved *from* a project→task-template binding; with no binding
   the intake logs a warning and creates **no task**. Status is inserted as NULL.
3. **An activity plan's `CREATE_TASK` step.** The template is resolved by `(slug, organization)` — no
   binding is checked, so an unbound template still materialises tasks; it just never appears in the
   project UI. The step's `taskStatusSlug` sets the status, looked up against the org's task
   statuses; omit it — or name a slug the org does not have — and the task is inserted with no
   status, silently.

**A task with no status is invisible.** Take-next joins tasks to org task statuses and only considers
`OPEN`-typed ones, so a NULL status or a `PENDING`/`IN_PROGRESS`/`DONE`/`BLOCKED` status drops the task
out of the individual take-next pool entirely. Candidates are ranked by `priority` ascending, then
document arrival time, then creation time.

## Locking

- Task statuses drive it: a status can lock the task on entry, and a separate status flag (defaulting
  to **on**) cascades the lock to attached document families.
- An action's `properties.lockTask` / `properties.lockDocumentFamily` override those defaults for that
  transition — e.g. lock the task but leave the documents editable.
- A locked task is immutable: edits return "Cannot edit a locked task" and deletes are refused.
- `metadata.properties.hideLock` hides the buttons; `metadata.properties.lockStatusSlug` turns Lock
  into a status transition.

## Resolving a template

```
POST /api/resolve?path=task-template://acme-corp/invoice-review
→ { "path": "/api/task-templates/00000000-0000-4000-8000-000000000001" }
```

Two segments: org slug and template slug. A third segment is accepted but silently discarded (task
templates are org-scoped). A `:version` suffix is rejected outright.
