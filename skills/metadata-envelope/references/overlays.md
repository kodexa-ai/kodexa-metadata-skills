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
| `slug`, `name`, `description`, `publicAccess`, `deprecated`, `template`, `type`, `extends` | every content field: a plan's `steps`, `inputOptions`, `inputsSchema`, `documentFamilyGroups`, `metadata`, title templates, prompts, `failureHandler`; a task template's `metadata` and `initialStatusSlug`; a data form's and project template's flat fields |

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
| `merge: <anchor>` + `with: <fields>` | merges fields into the item (maps recurse, `null` removes, lists replace) |
| `remove: <anchor>` or `remove: [anchors]` | removes the item |

A change has exactly one verb. A list that mixes changes with plain items is a 400.

**Anchors.** A string names an item by its key; a map of `{field: value}` names the one item whose
fields match, and a dotted field reaches into it (`{properties.title: Total}`). In a list of plain
values (`dependsOn`, `entrypoints`) a string names the value. When keys are compared, `${org}`,
`${orgSlug}` and a `kind://` prefix are ignored, and a bare slug names a ref by its last segment.
An anchor that matches nothing, or more than one item, is a 400 naming the change
(`overlay.steps[2]: no step is "extract"`). So is an `insert` whose key the list already has.

| Kind | Keyed lists (key) |
|---|---|
| `activity-plan` | `steps` (slug), `inputOptions` (name), `documentFamilyGroups` (name) |
| `task-template` | `metadata.actions` (slug, uuid or label), `metadata.forms` (dataFormRef) and each form's `actions`, `metadata.options` (name), `metadata.documentFamilyGroups` (name), `metadata.agentShortcuts` (id) |
| `data-form` | `cards` (id) and `nodes` (key or ref) as trees, searched at any depth; `options`, `views`, `actions`, `tabOrderGroups` (name); `tabOrder` (path); `shortcuts` (key) |
| `project-template` | `activityPlans`, `taskTemplates`, `dataForms`, `serviceBridges`, `stores`, `taxonomies` (ref, or slug for an inline entry); `knowledgeSets` and their `knowledgeItems` (slug); `triggers` (slug); `assistants` (slug or name); statuses (slug); `tags` (label); `options.options`, `options.dataOptions` (name) |

A component with no key (a V2 node without `key` or `ref`) is named by a map of its fields.

## Resolution rules

- **Same organization only.** `extends` naming another organization's resource is a 400.
- **Chains** are followed up to 8 deep; a **cycle** (including a resource extending itself) is a 400.
- **A missing base** is a 400 when the overlay is saved.
- **Validation runs on the resolved resource.** An activity plan's findings say which step the
  overlay inserted or changed (`step "po-audit" was inserted by the overlay of activity plan
  "invoice-review-plan-audit" (overlay.steps[1])`) and point at the change. The plan editor's
  validate endpoint accepts `extends` + `overlay` and validates them as they resolve.
- **When the base is saved**, its overlays are re-resolved in the same transaction. One the change
  breaks keeps its **last good resolution** and records the reason in
  `overlayResolution.error`; the base's save still succeeds, with a warning. **An activity plan in
  that state cannot start (409 `OVERLAY_STALE`)** until its overlay or the base is fixed and saved.
  This is what lets one deploy change a base and its overlays in either order.
- **Deleting a base** that live overlays extend is a 409. `extends: null` detaches an overlay: it
  keeps the content it was last resolved to and stops following the base.
- **Content beside `extends` is derived.** Steps (or any content) sent next to `extends` are not
  used; the response warns when they differed. Change the overlay instead.
- `overlayResolution` is server-written (base id and ref, chain, time, error) and never accepted.

## kdx

- `kdx sync pull` writes an overlay resource as `slug`, `name`, `description`, flags, `extends`,
  `overlay` — never the resolved content or `overlayResolution`. `sync push` and `apply` send that
  form; content beside `extends` in a file is not sent, with a warning.
- The overlay is compared **as written**: reordering its changes, or editing an `id` inside it, is a
  change. Elsewhere, ids and timestamps are environment noise.
- `kdx overlay resolve <file> --base-dir <dir>` resolves offline against the files under `<dir>`;
  `--check <file>` fails when the result differs from a given file.

## Existing projects

A project template applies only at project create. For projects made from a base template, apply
what an overlay template adds: `kdx project apply-template-delta <project> --template <overlay>`
(or `POST /api/projects/{id}/template-delta`). It binds the plans, task templates, data forms,
service bridges and stores the overlay adds, creates its inline stores, knowledge sets and triggers,
and adds or updates its option definitions and data properties. It only adds; a data property the
project set to its own value is kept unless `overwriteDataProperties`. Idempotent, one transaction,
`--dry-run` reports without writing. Assistants, statuses and tags are reported as not applied.
