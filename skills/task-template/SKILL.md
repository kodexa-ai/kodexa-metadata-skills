---
name: task-template
description: "Use when creating or editing a Kodexa task template — the org-scoped YAML behind a human review/approval task: action buttons and their status transitions, attached data forms, document upload groups, AI task naming, chat prompt, agent shortcuts, panel and button visibility, team/priority defaults, and task locking. Also use when an activity-plan CREATE_TASK step references a task template by slug."
---

# Kodexa Task Template Authoring

A task template is the **organization-scoped** definition of a human task's shape: the buttons a
reviewer sees, the data forms that open, the documents that attach, how the task is titled, which
panels are visible. One YAML file per template.

Projects do not own templates — they **bind** to them (the binding lives in `kdxa_project_resources`).
The same template is reused by many projects.

Automation is not here. Step graphs are a separate `activity-plan` resource. `planned:` and
`planTemplate:` are **gone** — the fields and their `kdxa_task_templates` columns were both removed,
and the decoder discards them silently, so authoring them looks fine and does nothing.

## The file that works

```yaml
type: taskTemplate                # required by `kdx apply` (or pass --type)
orgSlug: acme-corp                # required (or pass --org-slug)
slug: invoice-review              # required — unique in the org, 3–100 chars
name: "Invoice Review"            # canonical display name
title: "Invoice Review"           # legacy column, still read at runtime — set BOTH (see below)
description: "Manual review of extracted invoice data"
template: false                   # true only for starter/library templates
deprecated: false                 # badge only; nothing is gated on it
metadata:
  teamSlug: finance-reviewers
  priority: 2                     # 1 Highest … 5 Lowest
  properties:
    hideDefaultCancelButton: true
  actions:
    - slug: approve               # slug is the action's IDENTITY
      label: "Approve"
      properties:
        statusSlug: approved      # the key that actually transitions the task
```

```bash
kdx apply -f task-templates/invoice-review.yaml
```

Without `type:` `kdx apply` errors with *"resource file must contain a 'type' field"*; without
`orgSlug:` it errors with *"cannot determine the target organization"*; without `slug:` it refuses to
push. Accepted `type:` spellings: `taskTemplate`, `task-template`, `task-templates`, `taskTemplates`,
`tasktemplate`. Do **not** put `organizationId:` in the file — the CLI strips it before the push, so
it is inert noise.

## Six things that silently break

**1 — `slug` is an action's identity; never let `uuid` disagree with it.**
An action's `slug` is what an activity plan's `dependsOn: "<stepSlug>:<actionSlug>"` edge points at,
and what is recorded as the step's completion token when the task finishes. `uuid` is the deprecated
legacy spelling; readers alias it when `slug` is absent. But if one action carries **both** and they
differ, the platform refuses: every activity start (and plan validation) whose `dependsOn` names an
action on a `CREATE_TASK` step using this template fails with *"conflicting identities (uuid … vs
slug …) — uuid is the legacy spelling of slug; keep the slug"*. One bad action poisons all of them.
To rename a button, change `label` and keep the `slug`.

**2 — the status transition key is `properties.statusSlug`.**
`targetStatus` appears nowhere in the platform. An action with only `targetStatus` is a no-op button:
it saves the task, never transitions it, so no completion token is written and the downstream
activity branch never fires. (`properties.statusId` is read only when `statusSlug` is absent.)

```yaml
properties:
  statusSlug: approved      # ✅ works
  # targetStatus: approved  # ❌ read by nothing
```

**3 — attribute writes use `properties.attributes`, an array.**
`setAttributes` is read by nothing. The real key writes taxon-path values onto the task's extracted
data attributes (stamped with the reviewer's `user://` owner), not onto task fields. See
`references/actions.md`.

**4 — set `title:` as well as `name:`.**
`name` is the canonical column; `title` is a separate legacy column still read *directly* in two
places: the `{templateName}` placeholder in AI naming, on both the activity `CREATE_TASK` path and
the intake path; and the `CREATE_TASK` "no title supplied" fallback. A template authored with only
`name:` renders `{templateName}` as an empty string and gives activity-created tasks a **blank
title**. Only the create-task API reads `name` first and falls back to `title`.

**5 — the org placeholder is `${org}`, not `${orgSlug}`.**
In a `kdx sync` tree the CLI substitutes `${org}/` across the whole payload and hard-fails the push if
any `${org}` survives. `${orgSlug}` is a *server-side* placeholder expanded **only** inside
`metadata.forms[].dataFormRef`. Anywhere else — `companion.moduleRefs`, anything — `${orgSlug}` is
stored and used as a literal string that will never resolve.

**6 — deleting a key usually does not delete it.**
`metadata.properties` is the one region where the local file is the desired state: remove a key from
it and `kdx sync push` pushes the removal — but only while the file still *has* a `properties:` node.
Delete the whole block and the region stops counting as authored, so the push keeps every server-side
key; spell a full clear as `properties: {}`. Everywhere else in the file an absent key is
indistinguishable from a key you never mentioned — push silently no-ops on the removal, `--force`
included — so clear those by setting an explicit empty value instead.

There is also **no server-side task-template validation**: the rule set is empty and the handler
defaults to disabled. Nothing will catch any of the above for you.

## Declared but inert

Persisted, round-tripped, visible in existing YAML and sometimes editable in Studio — and read by
nothing at runtime. Do not delete them from files that already have them; do not add them to new ones.

| Field | Note |
|---|---|
| `initialStatusSlug` (top level) | No reader anywhere. Set the starting status on the activity-plan `CREATE_TASK` step's `taskStatusSlug`, or `statusSlug` in the create request. A task with no status is excluded from take-next entirely. |
| `metadata.options[]` | The whole option list. Studio's "Options" tab edits `metadata.properties`, not this. No task-runtime renderer exists. |
| `metadata.executionPolicy` | Editable in Studio; retries/timeouts actually come from the activity-plan step's own `executionPolicy`. |
| `metadata.workspaceId` | No reader. `kdx sync` still templates the UUID as `${workspace.<slug>}` on pull and re-resolves it against the destination project on push — but nothing on the task path reads the result, and the `workspace` resource itself is one type `kdx sync` refuses to carry (see **kdx-cli**). |
| `metadata.forms[].actions` | Task buttons come from the template-level `metadata.actions` only. Buttons declared under a form never render. |
| `metadata.forms[].properties` | Studio parks the referenced data form's own option values here; no runtime path reads it. |
| `metadata.forms[].dataForm` (inline form) | The server does not accept this field, so it is dropped on every save. Use `dataFormRef`. |
| `metadata.companion.agentRuntimeRef`, `.prompt` | Only `companion.moduleRefs` is acted on. Runtime selection comes from the **project's** companion config. |
| `documentFamilyGroups[].required`, `.knowledgeFeatures` | Enforced/rendered only for groups declared on an **activity plan**, not on a task template. |
| Action `type` | Part of the action shape and common in older YAML, but no task path reads it; Studio does not write it. Harmless to keep, pointless to add. |
| Action `properties.automaticallyTakeNextTask` | Written by the Studio action editor; no reader. |
| Action `properties.keybind` / `.altKeybinds` | Studio's *Keybind* field writes these; the task toolbar reads `shortcut` / `shortcutAltKey`. Author those. |
| `metadata.properties.defaultToDataForm`, `openNotesAutomatically`, `requireExplanations` | Studio checkboxes with no consumer on the task path. `hideFloatingHelper`, sitting beside them in the same panel, *is* live — see `references/fields.md`. |
| Option-level `subType`, `showOnPopup`, `featureFlag` | Moot given `metadata.options` itself is inert; `showOnPopup` is a project-template mechanic. |

## What changed from older YAML

| Old | Now |
|---|---|
| Project-scoped, one copy per project | Org-scoped; projects bind via project resources. URI is `task-template://acme-corp/invoice-review` — two parts. A third segment is silently ignored; a `:version` suffix is rejected. |
| `planned: true` + `planTemplate.steps:` | Removed from the resource and from the database. Author an `activity-plan` instead. |
| `assignmentRules:` | Never part of this schema. Use `metadata.teamSlug`; there is no auto-assign-to-user. |
| `documentGroups:` | Read by nothing. The live key is `documentFamilyGroups:`. |
| Separate `fields:` and `options:` | Neither is live — see "Declared but inert". |
| Action `targetStatus` / `setAttributes` | `properties.statusSlug` / `properties.attributes[]` |
| Action `uuid:` | `slug:` — `uuid` is deprecated and aliased |

## Common mistakes

| Mistake | What happens |
|---|---|
| One action declares `uuid` and `slug` with different values | Activity starts referencing this template's actions fail loudly. |
| `properties.targetStatus` | Button saves but never transitions; downstream steps never fire. |
| `properties.setAttributes` | Nothing is written. Use the `attributes` array. |
| `name:` without `title:` | Activity-created tasks get a blank title, and `{templateName}` renders empty on both the activity and intake AI-naming paths. |
| `${orgSlug}` outside `forms[].dataFormRef` | The literal string is stored and never resolves. |
| `sort: "createdOn desc"` on a document group | Not a recognised value. Grammar is `path\|created\|modified\|size` + `:asc\|:desc`. |
| A path- or glob-style `documentFamilyFilter` | Silently degrades to "allow every file". It parses extensions only. |
| Expecting `initialStatusSlug` to set the status | Never read. Tasks with no status are invisible to take-next. |
| Expecting `teamSlug`, `priority` or `documentFamilyGroups` to shape an activity-created task | An activity `CREATE_TASK` step writes no team, takes priority from the step or the activity, and never reads the document groups. `teamSlug` applies on the create-task API and intake; `priority` only pre-fills the New Task form. |
| An AI-naming prompt built on `{metadata:…}` / `{externalData:…}` | Both render empty on the activity `CREATE_TASK` path — that context carries no document metadata. They resolve on intake and the create-task API only. |
| Re-creating the same template per project | Define once at org level and bind each project to it. |
| Editing `metadata.actions` on a live template | Tasks carry no action data of their own — buttons are hydrated from the template on every read, so open tasks change instantly, including the `slug` an in-flight activity is waiting on. |

## References

- `references/fields.md` — every live `metadata` key: the `properties` switchboard, `forms[]`,
  `documentFamilyGroups[]`, `aiNaming` and `chatPrompt` placeholder tables, `agentShortcuts[]`,
  `teamSlug`/`priority`, `companion`, panel ids, plus how tasks get created, assignment/take-next and
  locking.
- `references/actions.md` — the action model end to end: identity, the activity-plan contract, and
  the full `properties` vocabulary.
- `references/examples.md` — three complete files with their `kdx` invocations.

Related skills: `activity-plan` (CREATE_TASK steps and `dependsOn` edges), `task-status` (what
`statusSlug` points at, plus the `locked` / `lockDocumentFamily` defaults an action overrides),
`data-form` (what `dataFormRef` points at), `project-resource` and `project-template` (where the
project→template binding is authored), `kdx-cli` (`kdx apply`, `kdx sync push`).
