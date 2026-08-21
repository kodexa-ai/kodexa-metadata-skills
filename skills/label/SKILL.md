---
name: label
description: "Use when creating, syncing or debugging Kodexa labels — the org-scoped tags Studio calls Document Tags — covering the missing slug that makes `kdx apply` refuse a label file, the name-matching rules that decide whether a push updates or silently no-ops, and the four runtime paths that create label rows behind your back"
---

# Kodexa Label Authoring

## A label has no slug — and that breaks `kdx apply`

Every other org-scoped resource is keyed by `slug`. A label is not. The row carries `name`, `label`,
`color` and its organization, and nothing else — there is no `slug` column, no `type`, no version, no
soft delete.

`kdx apply` routes every registry type through the sync push engine, and that engine's entry point
hard-requires a `slug` key before it starts. So a faithful label file is rejected:

```
label files need a 'slug' field to apply through the sync engine
```

`kdx validate` will not catch it. Add the `type: label` key both commands require and validate exits
clean: the label create schema has no required fields and does not forbid extra keys, so `type`,
`slug` and `orgSlug` come back as *warnings* — `key not found in schema` — and nothing is an error.
The refusal lands only at apply time. **Labels are pushable through `kdx sync push` from a
manifest.** Adding `slug:` and `orgSlug:` does get apply through — apply needs both, uses the slug
purely as the lookup key, and the server ignores it — but the value must then match the label's name
under the rule below, and the file is now carrying two keys the resource does not have.

## Shape

```yaml
# metadata/labels/high-priority.yaml
name: High Priority       # identity — unique per organization, stored verbatim
label: High Priority      # display text
color: "#3B82F6"          # swatch, presentation only
```

That is the whole resource. A synced file carries no `slug`, no `type` and no `orgSlug`: the push
sends the organization for you, and `kdx sync pull` writes exactly these three keys.

**`name` is the identity everywhere it matters** — the resolver (`label://acme-corp/High Priority`,
matched exactly, case- and space-sensitive), the join a plan script or condition reads, and the
uniqueness rule. There is no slugification step: whatever you put in `name` is what is stored, spaces
and capitals included. Creating a second label with the same `name` in the same organization is a
`409` — *a label with the same name already exists in this organization*.

`label` and `color` are free text. Nothing derives one from the other, and nothing in the platform
branches on either — Studio's **Document Tags** editor writes `name` and `label` from a single input
box, so labels created there always have the two in sync. Renaming a tag there rewrites `name` too,
which **breaks** every by-name reference — a `documents.find({ label: … })`, a condition testing
`document.labels`, a manifest entry — while the documents already carrying it keep it, because those
links follow the id.

## Syncing

```yaml
organization:
  label:
    - high-priority       # entries are names, never paths; file = metadata/labels/<entry>.yaml
```

Labels push at **order 10 — the lowest in the registry, so they land before every other type.** They
are the registry's no-dependency primitive: a label needs only its organization, and no other
resource type resolves a label reference during a push, so nothing is ever waiting on one. The
practical effect is that label results are the first lines of a push log, before a single store,
plan or project has been touched.

**The manifest entry has to identify the label by name, loosely:** the push looks up
`label://<org>/<entry>` exactly, and on a miss falls back to listing the organization's labels and
matching on name with case folded and spaces treated as dashes. So `high-priority` finds
`High Priority`, and `High Priority` finds it directly. A stray `slug:` left in the file **overrides
the manifest entry** as that lookup key — the push prints a *slug mismatch* warning and uses the
file's value.

When neither match hits, the push **creates** — and a create that collides with an existing name
comes back as *already exists*, which the CLI reports as `⏭️ already exists` and counts as skipped,
not failed. That is the failure mode to watch for: a manifest entry that does not normalize to the
label's `name` makes every push report "already exists" while your `label:` and `color:` edits are
never written, and the run still exits clean.

## Where label rows come from without you

Four platform paths create labels on the fly, so an organization's label list fills with rows nobody
authored:

| Path | Name written | Notes |
|---|---|---|
| A new KDDB content object — `POST /api/document-families/{id}/new-content` | each entry of the document's `labels` metadata, **uppercased** | applied and un-applied to match the document, but only when the document carries at least one label — an empty list leaves existing links alone. A non-KDDB upload syncs nothing |
| Intake upload with a `labels` parameter | each comma-separated entry, **uppercased** | form field or query param on the upload |
| `documents.addLabel(familyId, name)` in a plan script | the string you pass, **verbatim** | creates the row with no `label` and no `color` |
| An execution completing with a `completeLabel` | that value, verbatim | the row's `label` is set to the same value |

The uppercasing is the trap: an intake `labels=urgent` and a script `addLabel(id, "urgent")` produce
**two different rows** — `URGENT` and `urgent` — and a hand-authored `Urgent` is a third. Pick one
casing convention per organization and hold to it; uppercase matches what the document and intake
paths do.

## Reading labels back

- **Per-document `conditionExpr`** in an activity plan sees `document.labels` — an array of label
  **names**, next to `document.status`, `statusLabel`, `locked` and `path`.
- **Plan scripts** see the same array on `documents.get(...)` — `[]` when there are none — and can
  filter with `documents.find({ label: "URGENT" })`, which matches `name` exactly within the
  organization.
- **A document family's API payload** carries `labels: [{id, name, color, label}]`.
- `GET /api/organizations/{orgId}/labels` returns the organization's labels as a plain array,
  name-ordered — handier than the paged `GET /api/labels`.

## Two things that are not what they look like

`POST /api/document-families/{id}/labels` does **not** attach a label. It requires a `tagId` and
writes a tag-metadata row — a different table, unrelated to `kdxa_labels`, invisible to
`document.labels`, conditions and `documents.find`. `DELETE .../labels/{labelId}` removes that same
tag row. There is no REST endpoint that links a `kdxa_labels` row to a document family; use the four
paths above.

Likewise, `kdx intake upload --label NAME=value` packs its flags into a JSON array and posts that as
the `labels` field, which the API then splits on commas and uppercases — producing labels named after
fragments of JSON. Pass the field directly instead: `--metadata labels=NEEDS-REVIEW,URGENT`.

## Deleting

`DELETE /api/labels/{id}` is a **hard delete** — labels have no soft-delete state, so there is no
`deleted: true` row and nothing to restore. The join rows cascade, so the label vanishes from every
document family that carried it, silently and everywhere at once. How many documents carry it does
not block the delete; the only guard is the generic one, a `409` when the label is still bound to a
project.

Reprocessing a document family also clears labels: a full reprocess removes **every** label link on
that family, including ones an intake or a script added; a reprocess scoped to particular assistants
removes only the names those assistants applied as a `completeLabel`. The label rows themselves
survive.

## Declared but inert

| Field / feature | Reality |
|---|---|
| `slug` / `orgSlug` in a label file | Both are sent on every push and discarded — the resource has neither field. `slug` earns its keep only as the CLI's lookup key (where it beats the manifest entry); `orgSlug` only as the key `kdx apply` demands. |
| `color` | Stored, returned, rendered as a swatch in Studio. No platform behaviour reads it; labels created by scripts have none at all. |
| A `labelExpressions` array on a document store or project template | Accepted and persisted; no service evaluates it. It does not create or apply labels. |
| `resourceType: label` project-resource binding | Valid and accepted, but it does not scope a label to a project — every label stays organization-wide. It buys two things only: a user whose access comes from a team-project assignment can see labels bound to that project, and the label can no longer be deleted until it is unbound. |

## Common mistakes

| Mistake | What happens / fix |
|---|---|
| `kdx apply -f label.yaml` | Refused — it wants `type`, then `slug`, then `orgSlug`, and the resource has none of them. Push from a manifest instead. |
| Adding `slug: urgent` while the file says `name: High Priority` | The lookup misses, the create collides, the push logs *already exists* and skips. Make the entry normalize to the name. |
| Expecting `name` to be slugified | It is not. `High Priority` stays `High Priority`, and the resolver only matches it exactly. |
| Authoring `Urgent` when documents apply `URGENT` | Two separate labels. The document and intake paths uppercase; scripts do not. |
| `POST /api/document-families/{id}/labels` to tag a document | That writes a tag row, not a label. Nothing that reads labels will see it. |
| `kdx intake upload --label X=Y` | Posts JSON into a comma-separated field. Use `--metadata labels=X`. |
| Deleting a label to tidy the list | Hard delete plus cascade — it disappears from every document that had it. |
| Reprocessing and expecting labels to survive | A full reprocess drops every label link on the family. |
| Editing `label:`/`color:` and seeing no change after a push | Almost always the manifest-entry-vs-`name` mismatch above; check the push log for *already exists*. |

## Related skills

- **kdx-cli** — manifests, push order and the sync commands labels depend on.
- **intake** — the `labels` upload parameter and how it reaches a document family.
- **activity-plan** — `documents.addLabel` / `removeLabel` / `find` in plan scripts, and per-document
  `conditionExpr` reading `document.labels`.
- **project-resource** — the binding model behind `resourceType: label`.
