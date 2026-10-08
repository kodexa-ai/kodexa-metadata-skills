# `extends` + `overlay` — a resource as a delta over a base

An `activity-plan`, `task-template`, `data-form` or `project-template` can **extend** a base
resource of its own kind in its own organization. It names the base in `extends` and writes what it
changes in `overlay`, and nothing else. The server resolves the overlay against the base **when the
resource is saved**, stores the resolved resource in the ordinary fields beside `extends` and
`overlay`, and resolves it again **whenever the base is saved**. Every reader — activity start,
triggers, intake, project creation, the editors, `kdx` — sees a complete resource.

No other resource type accepts `extends`. A resource without it is stored and read exactly as one
was before the feature existed.

```yaml
slug: invoice-review-plan-audit
name: Invoice review with audit
description: Invoice review that also audits each invoice against its purchase order.
extends: activity-plan://${org}/invoice-review-plan
overlay:
  inputOptions:
    - name: auditTolerance
      type: number
      default: 0.01
  steps:
    - merge: extract
      with:
        moduleRef: ${orgSlug}/invoice-extractor-v2
        options:
          includeLineItems: true
    - insert:
        - slug: po-audit
          type: EXECUTION
          dependsOn: [extract]
          moduleRef: ${orgSlug}/po-auditor
      after: extract
    - remove: legacy-export
```

## What belongs where

| Top level (the resource's own) | Inside `overlay` (changes to the base's content) |
|---|---|
| `slug`, `name`, `description`, `publicAccess`, `deprecated`, `template`, `type`, `extends` | every content field: a plan's `steps`, `inputOptions`, `inputsSchema`, `documentFamilyGroups`, `metadata`, title templates, prompts, `failureHandler`; a task template's `metadata` and `initialStatusSlug`; a data form's and project template's flat fields — a data form's `version` included, so an overlay of a V2 form is V2 (and may set `version`) |

An envelope key inside `overlay` (`overlay.name`) is a 400. An overlay key the kind does not have
(`overlay.stepz`) is a 400.

## The overlay language

Each overlay key mirrors the field it changes:

- **a map** merges into the base's map, field by field and recursively; `null` removes a field;
  fields left out are the base's (JSON merge patch, RFC 7386);
- **a scalar** replaces;
- **a list of plain items** replaces the whole list (`[]` empties it);
- **a list of changes** edits the base's list in place, in order:

| Change | Effect |
|---|---|
| `insert: [items]` + `after: <anchor>` / `before: <anchor>` | inserts beside the anchor |
| `insert: [items]` + `into: <anchor>` | appends as the last children of a tree node (data form cards and nodes) |
| `insert: [items]` alone | appends at the end |
| `replace: <anchor>` + `with: <item>` | replaces the item wholly; a replacement without a key keeps the replaced item's |
| `merge: <anchor>` + `with: <fields>` | merges fields into the item: maps recurse, `null` removes, and a list is read as any overlay list is — a list of changes edits the item's list in place (`dependsOn`, a card's `children`), a list of plain items replaces it |
| `remove: <anchor>` or `remove: [anchors]` | removes the item |

An item that carries a verb key (`insert`, `replace`, `merge`, `remove`) **is** a change, with exactly
one verb, and every other key on it must be that verb's modifier: `before`, `after` or `into` for
`insert`, `with` for `replace` and `merge`, none for `remove`. A list that mixes changes with plain
items is a 400 that says which items are which. A list value whose items have a verb field cannot be
set by merging into the item that holds it — `replace` that item.

**Typos are errors, never items.** A misspelt modifier is a 400 naming the key, the verb, the list,
what the verb takes and the closest one: `overlay.steps[1]: "wiht" is not a modifier of merge (the
merge change in steps takes with) (did you mean "with"?)`. An item with no verb but a key one or two
edits from one — beside a modifier (`mege: b` + `with`), or keyless in a keyed list (`remvoe: b` in
`steps`) — is a 400 too (`"mege" is not a change verb (did you mean "merge"?)`). Read as items, either
would have replaced the whole list.

**Anchors.** A string names an item by its key; a map of `{field: value}` names the one item whose
fields match, and a dotted field reaches into it (`{properties.title: Total}`). In a list of plain
values (`dependsOn`, `entrypoints`) a string names the value. Before keys are compared, `${org}` and
`${orgSlug}` are **expanded** to the organization's slug and a `kind://` prefix is dropped (so
`${org}/invoice-review-plan`, `acme-corp/invoice-review-plan` and
`activity-plan://acme-corp/invoice-review-plan` name one binding); a bare slug names a ref by its last
segment. An anchor that matches nothing, or more than one item, is a 400 naming the change
(`overlay.steps[2]: no step is "extract"`). So is an `insert` — or a change nested in a `merge` —
that gives the list a key it already has, anywhere in a tree (a card's `children` included).

| Kind | Keyed lists (key) |
|---|---|
| `activity-plan` | `steps` (slug), `inputOptions` (name), `documentFamilyGroups` (name) |
| `task-template` | `metadata.actions` (slug, uuid or label), `metadata.forms` (dataFormRef) and each form's `actions`, `metadata.options` (name), `metadata.documentFamilyGroups` (name), `metadata.agentShortcuts` (id) |
| `data-form` | `cards` (id) and `nodes` (key or ref) as trees, searched at any depth; the rows of a card a V2 node wraps (`props.card.children`, id); `options`, `views`, `actions`, `tabOrderGroups` (name); `tabOrder` (path); `shortcuts` (key) |
| `project-template` | `activityPlans`, `taskTemplates`, `dataForms`, `serviceBridges`, `stores`, `taxonomies` (ref, or slug for an inline entry); `knowledgeSets` and their `knowledgeItems` (slug); `triggers` (slug); `assistants` (slug or name); `documentStatuses` (slug or status); `taskStatuses` (slug or label); `attributeStatuses` (status); `tags` (label); `options.options`, `options.dataOptions` (name) |

A component with no key (a V2 node without `key` or `ref`) is named by a map of its fields — a
node that wraps a card by `{props.card.id: <card id>}`.

## Resolution rules

- **Same organization only.** `extends` naming another organization's resource is a 400.
- **Chains** are followed up to 8 deep; a **cycle** (including a resource extending itself) is a 400.
- **A missing base** is a 400 when the overlay is saved.
- **Validation runs on the resolved resource.** An activity plan's findings say which step the
  overlay inserted or changed (`step "po-audit" was inserted by the overlay of activity plan
  "invoice-review-plan-audit" (overlay.steps[1])`, whatever the change's anchor) and point at the
  change. The plan editor's validate endpoint accepts `extends` + `overlay` and validates them as
  they resolve; it needs `activity-plan:read` in the `organizationId` sent — an organization the
  caller cannot read, or one that does not exist, gets one 404 whatever is asked, and a base the
  caller cannot read is reported as one that does not exist.
- **Saves of a base and its overlays serialize.** An overlay's save locks its bases (root first) and
  then itself before resolving; a base's save locks its overlays parent before child. The stored
  content is always the stored overlay resolved against the stored base, and every write moves
  `changeSequence` on by one.
- **When the base is saved**, its overlays are re-resolved in the same transaction. One the change
  breaks keeps its **last good resolution** and records the reason in
  `overlayResolution.error`; the base's save still succeeds, with a warning naming the overlay's own
  base (and, for an overlay of an overlay, the stale base in between). With auditing on, each
  overlay the cascade rewrites gets its own audit entry, attributed to whoever saved the base.
  **An activity plan in that state keeps running its last good resolution**: each activity started
  from it records `metadata.overlayStale` (the plan, the base and its change, the resource whose save
  made it stale, the error), carries a *Stale overlay* badge, and the start's response warns
  (`overlay.stale-resolution`). The validate endpoint (given the plan's `id`) and the plan editor
  keep reporting it until the overlay or the base is fixed and saved. An edit that leaves `extends`
  and `overlay` as they are (a rename) still saves on a stale plan, or one whose base is stale,
  keeping its last good resolution.
  **Strict mode** (409 `OVERLAY_STALE` instead of running stale) is decided when the plan goes stale
  and recorded (`overlayResolution.staleMode`): `metadata.overlayStale` in the plan's own overlay
  when it sets one (`refuse`, or anything else to run), else the base being saved, as that save
  leaves it — a base that adds `overlayStale: refuse` in the save that breaks its overlay makes it
  refuse. An overlay of a stale overlay inherits its base's decision.
  This is what lets one deploy change a base and its overlays in either order.
- **Deleting a base** that live overlays extend is a 409 (an overlay being saved holds its base, so
  the delete waits and sees it). `extends: null` — alone, or with `overlay: null` — detaches an
  overlay: the overlay goes with it, the resource keeps the content it was last resolved to and
  stops following the base.
- **Content beside `extends` is derived.** Steps (or any content) sent next to `extends` are not
  used; the response warns when they differed. Change the overlay instead.
- `overlayResolution` is server-written (base id and ref, chain, time, the list changes the overlay
  made, and when stale: the error, the save that caused it, the strict mode) and never accepted. An
  activity's plan snapshot keeps only the base, its `changeSequence`, the resolution time and whether
  it was stale — never the overlay.

## kdx

- `kdx sync pull` writes an overlay resource as `slug`, `name`, `description`, flags, `extends`,
  `overlay` — never the resolved content or `overlayResolution`. A resource the server holds plain
  (detached with `extends: null`) is pulled as the plain resource: the file loses `extends` and
  `overlay`, so the next push cannot attach it again. `sync push` and `apply` send the overlay form;
  content beside `extends` in a file is not sent, with a warning.
- The overlay is compared **as written**: reordering its changes, or editing an `id` inside it, is a
  change. Elsewhere, ids and timestamps are environment noise. `extends` is compared as the server
  stores it (`kind://org/slug`), so a short `extends: invoice-review-plan` pushes once.
- `kdx overlay resolve <file> --base-dir <dir>` resolves offline against the files under `<dir>`,
  with the same key checks and change language a save applies; `--check <file>` fails when the
  result differs from a given file.

## Existing projects

A project template applies only at project create. For projects made from a base template, apply
what an overlay template adds: `kdx project apply-template-delta <project> --template <overlay>`
(or `POST /api/projects/{id}/template-delta`). It binds the plans, task templates, data forms,
service bridges and stores the overlay adds, creates its inline stores, knowledge sets and triggers,
and adds or updates its option definitions and data properties. It only adds; a data property the
project set to its own value is kept unless `overwriteDataProperties`. Idempotent, one transaction,
`--dry-run` reports without writing. Assistants, statuses and tags are reported as not applied, as
is a knowledge set whose slug the organization already uses for another project's set (give a
template's set a per-project slug such as `invoices-${project.id}`). A stale overlay template is a
409 `OVERLAY_STALE`.

It needs `project:update`, and each change is authorized as its own endpoint authorizes it:

| Change | Needs |
|---|---|
| bind a plan, task template, data form, service bridge, data definition or store | `project-resource:bind` on the project, and a resource of the project's organization (as `POST /api/project-resources/bind`) |
| create an inline store | `document-store:create` / `data-store:create`, and `project-resource:bind` |
| create a knowledge set | `knowledge-set:create` |
| create a trigger | `trigger:create` (and `trigger:schedule` for a schedule trigger) |

The whole delta is planned first; one change the caller may not make refuses it all — dry run
included — with **403 `FORBIDDEN`**, nothing applied, `details.refused` listing each refused change
and the permission it needs. A binding to another organization's resource is refused the same way.
