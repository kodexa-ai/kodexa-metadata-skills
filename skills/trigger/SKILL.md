---
name: trigger
description: "Use when creating, editing, or debugging Kodexa triggers — project-scoped YAML rules that start an activity plan when a platform event fires. Covers which event kinds actually dispatch, the exact payload each one carries, JSONata eventFilter and inputMapping, and the project binding a trigger needs before it can fire."
---

# Kodexa Trigger Authoring

## Overview

A **trigger** is a project-scoped rule: *when event X happens in this project, start activity plan Y*. It is the automated counterpart to starting an activity by hand.

Triggers are the one resource in this family that **lives inside a project** — a trigger row carries its own `projectId`. What it points at (an activity plan) is **org-scoped and bound** to the project. So the plan is org-level metadata, a project-resource binding makes it usable in the project, and the trigger decides when it runs:

```
event in project P  →  trigger (in P, enabled, filter matches)  →  activity plan (org-scoped, bound to P)  →  activity (runtime, in P)
```

## Event kinds — three of the six never fire

`eventKind` is required and must be one of six lowercase values. Anything else — `TASK_CREATED`, `task.statusChanged`, `schedule`, `document_arrived`, `data_extracted` — is rejected with a 400.

| `eventKind` | Dispatched today? | Raised when |
|---|---|---|
| `task_status_changed` | **yes** | any task update (see below) |
| `document_locked` | **yes** | a document family is locked / marked finished |
| `knowledge_set_updated` | **yes** | the last open question on a knowledge set is answered, or the set is explicitly validated |
| `task_created` | **no — nothing emits it** | never |
| `activity_completed` | **no — nothing emits it** | never |
| `manual` | **no — nothing emits it** | never |

The bottom three are accepted by the API, offered in the UI's event-kind dropdown, and stored happily — but no code path ever dispatches them, so a trigger authored on one is inert and silent. The UI's create dialog even *defaults* to `task_created`. Choose one of the top three. There is no "run this trigger now" endpoint, so `manual` cannot be fired by hand either.

`task_status_changed` is a misnomer worth internalising: it is dispatched from the generic task-updated event, which is published on status change, assignee change, team change, lock, unlock, take-next assignment, and batch task saves. An empty filter on this kind fires on **all** of them.

## Resource shape

```yaml
slug: on-invoice-locked                                   # unique in the project; also the file name
name: On invoice locked
enabled: true                                             # omit for true; false = dormant, config kept
eventKind: document_locked                                # required, one of the six
eventFilter: '$exists(lockedByUserId) and $not($exists(reconciled))'
inputMapping: '{ "documentFamilyId": documentFamilyId, "projectId": projectId }'
activityPlanRef: activity-plan://acme-corp/invoice-review # required
```

- `projectId` — required on the wire, but you do **not** write it in a synced file: `kdx` derives it from the `projects/<project-slug>/` directory the file sits in and sends it for you. Write it only when POSTing directly.
- `organizationId` — **never set it.** It is derived server-side from the owning project on every write; a value you supply is overwritten.
- `activityPlanRef` — must be `activity-plan://<orgSlug>/<planSlug>`, unversioned. A bare slug (`invoice-review`), a `:1.0.0` suffix, another scheme, or a leftover `${...}` fails the write — a 400 on update, but on create it surfaces as a **500**, so do not read a server error from a trigger create as a platform fault; check the ref first. `activity-plan://${org}/invoice-review` is fine in a synced file — `${org}/` is substituted before the API sees it.
- `slug` — unique per project; a second trigger with the same slug in the same project is rejected. Nothing in the platform freezes it after create, but it is the trigger's identity in synced files and in `trigger://` URIs, so treat it as stable.

## What a filter can actually see

Payloads are **flat**: the event's extra fields are lifted to the top level, so a filter reads `lockedByUserId`, never `extra.lockedByUserId`. `eventKind` is stamped onto every payload.

| Kind | Payload keys |
|---|---|
| `task_status_changed` | `eventKind`, `taskId`, `projectId`, `organizationId`, `documentFamilyId`, and `planAdvancement: true` only on the updates raised when a task completes and its activity advances |
| `document_locked` | `eventKind`, `documentFamilyId`, `projectId`, `organizationId`, `storeId`, `lockedAt`, `lockedByUserId` (when a person did the lock), plus `reconciled: true` on a replay |
| `knowledge_set_updated` | `eventKind`, `projectId`, `organizationId`, `knowledgeSetId` |

That is the whole contract. There is **no** status, task-template, plan, activity or result data in any payload. A filter on `newStatusSlug`, `oldStatusSlug`, `taskTemplateSlug`, `activityPlanSlug`, `result` or `triggeredById` references keys that do not exist, so the trigger silently never fires.

Two traps in that table:

- On `task_status_changed`, `documentFamilyId` is **always present and always empty** — no emitter populates it. `$exists(documentFamilyId)` is therefore true and useless; `documentFamilyId = ""` matches every event.
- `document_locked` only reaches triggers when the locked document's store is bound to the project. Lock a document in an unbound store and the event is published and then discarded.

`knowledge_set_updated` is deliberately suppressed when the caller is an assistant (execution) user — the identity agents and module runs use — so an agent updating a knowledge set cannot re-fire its own loop.

## eventFilter is JSONata, not a key/value map

`eventFilter` is a JSONata predicate evaluated against the payload. Write it as a **string**:

```yaml
eventFilter: 'planAdvancement = true'                      # task_status_changed: only plan advances
eventFilter: 'storeId = "00000000-0000-4000-8000-000000000001"'
eventFilter: 'storeId in ["00000000-0000-4000-8000-000000000001", "00000000-0000-4000-8000-000000000002"]'
```

`{ "expr": "…" }` is the same thing wrapped. Absent, `null`, `{}` and `""` all mean **no filter — fire on every event of that kind**; an explicit `{ "expr": "" }` is an error at write time. Start with no filter and narrow once you can see which events arrive.

A **bare YAML mapping** is also accepted, and is silently rewritten into an equality-AND predicate with keys sorted:

```yaml
eventFilter:                    # becomes: projectId = "…" and storeId = "…"
  storeId: "00000000-0000-4000-8000-000000000001"
  projectId: "00000000-0000-4000-8000-000000000002"
```

Scalars only. A list value (`storeId: [a, b]`) or a nested object is rejected outright at write time — express alternation as a JSONata string (`in [...]`, or `or`) instead. The mapping form has no array support.

Semantics that decide whether you get a fire:

- `false`, `null`, and an expression that resolves to nothing → no match. Any other value → match, so a bare `storeId` matches whenever the key has a value.
- Any comparison against a key the payload does not carry evaluates **false** — including `!=`. `reconciled != true` never matches on an event with no `reconciled` key. Guard with `$exists(...)` / `$not($exists(...))` instead.
- A filter that will not compile is **fail-closed** — it never fires.

## inputMapping: the only way to reshape the payload

`inputMapping` is a second JSONata expression, same wire shapes, that produces the **inputs object** handed to the activity. Leave it empty and the raw event payload becomes the inputs verbatim. Write it as a string, and quote the *keys* but not the *field references*:

```yaml
inputMapping: '{ "documentFamilyId": documentFamilyId, "lockedBy": lockedByUserId }'
```

Three ways it goes quietly wrong:

- **A key whose field is not in the payload is silently dropped.** `'{ "a": newStatusSlug }'` evaluates to `{}` — no error, no warning. The activity starts with inputs you did not intend.
- **A result that is not an object aborts the fire** (`inputs_invalid`) — an array, a string, a number. But an expression that resolves to nothing yields empty inputs `{}` and the fire proceeds.
- **The bare-mapping form does not do what it looks like.** This:

```yaml
inputMapping:                              # WRONG
  documentFamilyId: documentFamilyId
```

is stored as the JSONata expression `{"documentFamilyId":"documentFamilyId"}`, which builds an object whose value is the literal **string** `"documentFamilyId"` — not the field. The activity starts, with garbage inputs. Use the quoted-string form for `inputMapping`; reserve the mapping form for `eventFilter` equality checks.

If the plan declares required entries in its `inputOptions`, the fire is rejected unless the inputs carry them — an absent key, `null` and `""` all count as missing. (A plan's `inputsSchema` is *not* enforced at start; it drives the UI's start form only.)

## Bind the plan before the trigger

A trigger fires only if its `activityPlanRef` resolves to a plan **bound to the same project** through a project-resource binding. Unbound, the first fire fails and **is not retried** — the event is gone. Bind the plan (project-resource YAML, the project's Resources tab, or the template's activity-plan list) first.

## Where a trigger lives

**Synced file** — `<metadata_dir>/projects/<project-slug>/triggers/<slug>.yaml`, one file per trigger, no org-slug directory level. Reference it from the manifest under the project:

```yaml
projects:
  invoice-processing:
    trigger:
      - on-invoice-locked
    linked:
      activity-plan:
        - invoice-review          # org-scoped: pushed, then bound to the project
```

`kdx` pushes triggers at order 75 — after activity plans (50), projects (60) and assistants (70) — so the plan and the project exist by the time the trigger lands. `linked:` bindings are applied in a post-push pass, which is fine: the binding is only checked when the trigger fires.

**Inline in a project template** — a `triggers:` block is a first-class part of a project template and is materialized when a project is created from it, straight after the plan bindings. Same fields minus `projectId`, `organizationId` and `triggerMetadata`, and re-applying is a no-op per `(project, slug)`:

```yaml
triggers:
  - slug: on-invoice-locked
    name: On invoice locked
    eventKind: document_locked
    activityPlanRef: activity-plan://${org}/invoice-review
    inputMapping: '{ "documentFamilyId": documentFamilyId }'
```

`${org}`, `${orgSlug}`, `${project.id}` and `${project.name}` are substituted in **`slug`, `name` and `activityPlanRef` only** — not in `eventFilter` or `inputMapping`. The referenced plan must also appear in the template's activity-plan list, or the trigger's first fire fails for want of a binding. A trigger entry that fails validation is **skipped**, not fatal: the project is created without it and nothing in the response says so, so list the project's triggers after materializing a template.

**API** — `POST /api/triggers` with `projectId` in the body; `PUT /api/triggers/{id}` re-runs the same validation. Pause and resume with `POST /api/triggers/{id}/enable` and `/disable` rather than deleting: triggers have no soft delete, so a `DELETE` is permanent.

**URI** — `trigger://<orgSlug>/<projectSlug>/<triggerSlug>`, unversioned, resolvable to `/api/triggers/{id}`.

## Delivery: at-least-once, no dedupe, no retry

Delivery is at-least-once with no deduplication, no ordering guarantee and no cycle detection. `document_locked` additionally has a reconciler that replays locks from the last 24 hours (at least 5 minutes old) that appear to have produced no activity, marking the replay `reconciled: true` — so the same lock can legitimately arrive twice. **Author the plan to be idempotent.** A plan that must not run concurrently in a project can set `serializePerProject: true` in its own `metadata`: concurrent starts then coalesce onto the in-flight run and queue one follow-up.

A failed fire is not retried. Failures surface as `trigger.fire_failed` with a reason (`binding_missing`, `activity_plan_not_found`, `inputs_invalid`, `strict_binding_violation`); every evaluation also emits `trigger.evaluated` with `matched` / `no-match` / `eval-error`, which is how you tell "filter did not match" from "filter did not compile". Both are platform telemetry, so they appear wherever your deployment collects it.

## Declared but inert

| Field | Reality |
|---|---|
| `triggerMetadata` | Persisted, returned, round-tripped — and never read. The evaluator does not even load the column. It is reserved for event kinds that do not exist yet. It is **not** a way to pass config to the plan: use `inputMapping`, or defaults in the plan's inputs. The `triggerMetadata` you see on a started activity is a different field, written by the platform as `{"sourceTriggerId": "<trigger id>"}`. |
| `metadata` | Free-form. Stored and returned; nothing in the platform interprets it. Safe for your own annotations. |
| `eventKind: task_created` / `activity_completed` / `manual` | Accepted, listed in the UI, stored — never dispatched. |

`inputMapping` is the opposite case: fully honoured by the evaluator, but the UI's trigger editor does not expose it. Author it in YAML or via the API.

## Common mistakes

| Mistake | What happens / fix |
|---|---|
| `eventKind: TASK_CREATED` or `task.statusChanged` | 400. Lowercase snake_case, one of the six. |
| Authoring on `task_created` / `activity_completed` / `manual` | Accepted and silent forever. Use `task_status_changed`, `document_locked` or `knowledge_set_updated`. |
| Filtering on `newStatusSlug`, `taskTemplateSlug`, `activityPlanSlug`, `result` | Those keys are in no payload. The trigger never fires. |
| `eventFilter: {storeId: [a, b]}` | Rejected — scalars only in the mapping form. Write `'storeId = "a" or storeId = "b"'`. |
| `eventFilter: 'extra.lockedByUserId = "…"'` | Payloads are flat. Use `lockedByUserId`. |
| `eventFilter: 'reconciled != true'` | Evaluates false when the key is absent, so it never matches. Use `$not($exists(reconciled))`. |
| `inputMapping` written as a YAML mapping | Values become literal strings, not field references. Use `'{ "key": fieldName }'`. |
| `inputMapping` naming a key the payload lacks | That key is dropped from the inputs, silently. Map only keys from the table above. |
| Empty filter on `task_status_changed` | Also fires on assignment, team change, lock, unlock and take-next. Narrow with `planAdvancement = true` if you mean completion. |
| `activityPlanRef: invoice-review` or `…:1.0.0` | Write fails. Must be `activity-plan://orgSlug/planSlug`, unversioned. |
| Plan not bound to the project | First fire fails `binding_missing` and is never retried. Bind first. |
| Setting `organizationId` | Silently overwritten from the project. Omit it. |
| Deleting a trigger to pause it | Hard delete, no recovery. Use `/disable` or `enabled: false`. |
| Assuming one event → one run | At-least-once, no dedupe, plus lock replays. Make the plan idempotent. |

## Related skills

- **activity-plan** — the plan a trigger starts, its steps, and its `inputOptions` (which decide whether your `inputMapping` output is accepted).
- **project-resource** — the binding that makes an org-scoped plan usable in the project. Without it a trigger cannot fire.
- **project-template** — the `triggers:` block, and getting the plan binding into the same template.
- **task-template** / **knowledge-system** — the tasks whose updates raise `task_status_changed`, and the knowledge sets whose validation raises `knowledge_set_updated`.
