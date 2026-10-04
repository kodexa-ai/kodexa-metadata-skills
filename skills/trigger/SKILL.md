---
name: trigger
description: "Use when creating, editing, or debugging Kodexa triggers — project-scoped YAML rules that start an activity plan when a platform event fires or on a cron schedule. Covers which event kinds actually dispatch, the exact payload each one carries, schedule triggers (cron, timezone, missed runs, DST, Run now), JSONata eventFilter and inputMapping, and the project binding a trigger needs before it can fire."
---

# Kodexa Trigger Authoring

## Overview

A **trigger** is a project-scoped rule: *when event X happens in this project — or when the clock reaches a slot — start activity plan Y*. It lives inside a project (the row carries its own `projectId`); the plan it starts is **org-scoped and bound** to the project:

```
event or schedule slot in project P  →  trigger (in P, enabled, filter matches)  →  activity plan (org-scoped, bound to P)  →  activity (in P)
```

## Event kinds — three of the seven never fire

`eventKind` is required and must be one of seven lowercase values. Anything else — `TASK_CREATED`, `task.statusChanged`, `document_arrived`, `data_extracted` — is a 400.

| `eventKind` | Dispatched? | Raised when |
|---|---|---|
| `task_status_changed` | **yes** | any task update (see below) |
| `document_locked` | **yes** | a document family is locked / marked finished |
| `knowledge_set_updated` | **yes** | the last open question on a knowledge set is answered, or the set is explicitly validated |
| `schedule` | **yes** | a slot of its cron schedule comes due, or someone uses Run now |
| `task_created` | **no — nothing emits it** | never |
| `activity_completed` | **no — nothing emits it** | never |
| `manual` | **no — nothing emits it** | never |

The bottom three are accepted and stored, but no code path dispatches them, so a trigger on one is inert and silent. Run now works only on schedule triggers, so `manual` cannot be fired by hand either.

`task_status_changed` is a misnomer: it is dispatched from the generic task-updated event — status change, assignee change, team change, lock, unlock, take-next, batch saves. An empty filter fires on **all** of them.

## Resource shape

```yaml
slug: on-invoice-locked                                   # unique in the project; also the file name
name: On invoice locked
enabled: true                                             # omit for true; false = dormant, config kept
eventKind: document_locked                                # required, one of the seven
eventFilter: '$exists(lockedByUserId) and $not($exists(reconciled))'
inputMapping: '{ "documentFamilyId": documentFamilyId, "projectId": projectId }'
activityPlanRef: activity-plan://acme-corp/invoice-review # required
```

- `projectId` — required on the wire, but not in a synced file: `kdx` derives it from the `projects/<project-slug>/` directory. `organizationId` — **never set it**; it is derived from the project and overwritten.
- `activityPlanRef` — `activity-plan://<orgSlug>/<planSlug>`, unversioned. A bare slug, a `:1.0.0` suffix, another scheme or a leftover `${...}` is a 400. `activity-plan://${org}/invoice-review` is fine in a synced file — `kdx` substitutes `${org}/` first.
- `slug` — unique per project; treat it as stable — it is the identity in synced files and `trigger://` URIs.
- `triggerMetadata` — the schedule definition, on `schedule` triggers only. Any other kind refuses a non-empty value with a 400.

## Schedule triggers

```yaml
slug: weekly-vendor-statements
name: Weekly vendor statements
eventKind: schedule
triggerMetadata:
  cron: "0 6 * * MON"              # minute hour day-of-month month day-of-week, read in the zone below
  timezone: America/New_York       # an explicit IANA zone; write "UTC" for UTC
  jitterSeconds: 900               # deliver up to 15 min after the slot; the same offset every week
inputMapping: '{ "scheduledFor": scheduledFor, "lookbackDays": 7 }'
activityPlanRef: activity-plan://acme-corp/vendor-statement-review
```

**The definition is strict.** Exactly five cron fields; no `@daily` / `@every`; no `TZ=` or `CRON_TZ=` in the expression; an explicit IANA zone (not `""` or `Local`); `jitterSeconds` 0 to 3600; a next slot must exist; and at least 15 minutes between each of the next ten slots (`*/5 * * * *` is refused). A failure is a 400 whose message names the rule. Check a definition before saving with `POST /api/triggers/schedule-preview`, which returns the next runs or that same 400.

What decides whether a schedule fires when you expect:

- **It fires once after missed runs.** After any outage the trigger fires one time for the slot that was due, then jumps to the first slot after now. Missed slots are never replayed.
- **Arming starts from now.** Creating, enabling, or changing `eventKind` or `triggerMetadata` re-arms the schedule to the first slot **after now**. Enabling at 06:05 does not run the 06:00 slot. Renames, filter, mapping and plan edits do not re-arm.
- **A slot is its local wall time plus zone.** When DST skips an hour, a slot inside it **never fires** (`30 2 * * *` in `America/New_York` skips a day each March). When DST repeats an hour, the slot fires once.
- **At most once per slot**, however many replicas or orchestrators run; Postgres is the only clock and lock. A fire queued for a trigger that is then disabled, re-armed or re-planned is cancelled, not delivered.
- **Jitter only delays.** Each trigger keeps a fixed offset into `[0, jitterSeconds)`. Run now has no jitter.
- Schedule state (next fire, last status, recent fires) is not on the trigger and not in its YAML. Read it from `GET /api/triggers/{id}/schedule`: `lastStatus` says why a slot did not start (`FILTER_FALSE`, `PLAN_UNBOUND`, `PROJECT_INACTIVE`, …).

**Run now** — `POST /api/triggers/{id}/run` answers 202, and the next tick (within about 20 seconds) fires through the same filter, mapping and binding checks, with `scheduledFor` = the request time. It is a 409 while the trigger is disabled, a request or fire is still queued, a run it started is live, or within 5 minutes of the last Run now (`Retry-After`). A blocked fire is retried or cancelled by trigger, under `/api/triggers/{id}/spawn-requests/{requestId}/`.

**Authority.** Creating a schedule trigger, or changing a schedule trigger's kind, definition, plan or `enabled`, needs the project-scoped `trigger:schedule`; Run now, retry and cancel need `trigger:run`. project-editor and above hold both; org-member holds neither, so it can author event triggers but gets a 403 on schedule ones. **Caps:** 5 schedule triggers per project, 500 per organization — one more is 409 `TRIGGER_SCHEDULE_LIMIT`.

Codes, statuses, the state fields and delivery outcomes are in [references/schedule.md](references/schedule.md).

## What a filter can actually see

Payloads are **flat**: extra fields sit at the top level, so a filter reads `lockedByUserId`, never `extra.lockedByUserId`. `eventKind` is on every payload.

| Kind | Payload keys |
|---|---|
| `task_status_changed` | `eventKind`, `taskId`, `projectId`, `organizationId`, `documentFamilyId`, and `planAdvancement: true` only on the updates raised when a task completes and its activity advances |
| `document_locked` | `eventKind`, `documentFamilyId`, `projectId`, `organizationId`, `storeId`, `lockedAt`, `lockedByUserId` (when a person did the lock), plus `reconciled: true` on a replay |
| `knowledge_set_updated` | `eventKind`, `projectId`, `organizationId`, `knowledgeSetId` |
| `schedule` | `eventKind`, `triggerId`, `projectId`, `organizationId`, `scheduledFor` (the slot, or the Run now request time, RFC 3339 UTC) |

That is the whole contract. No payload carries status, task-template, plan, activity or result data. A filter on `newStatusSlug`, `taskTemplateSlug`, `activityPlanSlug`, `result` or `triggeredById` references keys that do not exist, so the trigger silently never fires.

- On `task_status_changed`, `documentFamilyId` is **always present and always empty**. `$exists(documentFamilyId)` is useless; `documentFamilyId = ""` matches everything.
- `document_locked` reaches triggers only when the document's store is bound to the project. Otherwise the event is discarded.
- `knowledge_set_updated` is suppressed when the caller is an assistant (execution) user, so an agent cannot re-fire its own loop.
- A schedule payload holds only the clock and the trigger's own ids, so narrow a schedule in the cron, not the filter. A filtered-out slot records `FILTER_FALSE` — and so does a Run now the filter rejects.

## eventFilter is JSONata, not a key/value map

`eventFilter` is a JSONata predicate over the payload. Write it as a **string**:

```yaml
eventFilter: 'planAdvancement = true'                      # task_status_changed: only plan advances
eventFilter: 'storeId in ["00000000-0000-4000-8000-000000000001", "00000000-0000-4000-8000-000000000002"]'
```

`{ "expr": "…" }` is the same thing wrapped. Absent, `null`, `{}` and `""` all mean **no filter**; an explicit `{ "expr": "" }` is an error. A **bare YAML mapping** is rewritten into an equality-AND predicate (`storeId: "…"` becomes `storeId = "…"`), scalars only — a list value or nested object is rejected; write alternation as a JSONata string.

- `false`, `null`, and an expression that resolves to nothing → no match. Any other value → match.
- A comparison against a key the payload lacks is **false** — including `!=`. `reconciled != true` never matches when `reconciled` is absent. Guard with `$exists(...)` / `$not($exists(...))`.
- A filter that will not compile is **fail-closed** — it never fires.

## inputMapping: the only way to reshape the payload

`inputMapping` is a second JSONata expression that produces the **inputs object** for the activity. Empty, the raw payload becomes the inputs. Write it as a string; quote the *keys*, not the *field references*:

```yaml
inputMapping: '{ "documentFamilyId": documentFamilyId, "lockedBy": lockedByUserId }'
```

- **A key whose field is not in the payload is silently dropped.** `'{ "a": newStatusSlug }'` evaluates to `{}`.
- **A result that is not an object aborts the fire** (`inputs_invalid`). An expression that resolves to nothing yields `{}` and the fire proceeds.
- **The bare-mapping form is wrong here.** `inputMapping: {documentFamilyId: documentFamilyId}` builds an object whose value is the literal **string** `"documentFamilyId"`. Use the quoted-string form.

If the plan declares required `inputOptions`, the fire is rejected unless the inputs carry them (absent, `null` and `""` count as missing). A plan's `inputsSchema` is *not* enforced at start.

## Bind the plan before the trigger

A trigger fires only if `activityPlanRef` resolves to a plan **bound to the same project**. For an event trigger an unbound plan fails the fire (`binding_missing`) and **is not retried** — the event is gone. For a schedule trigger the slot records `PLAN_UNBOUND` and is lost; the next slot tries again. Bind the plan first.

## Where a trigger lives

**Synced file** — `<metadata_dir>/projects/<project-slug>/triggers/<slug>.yaml`, listed under the project in the manifest (`trigger: [on-invoice-locked]`, with the plan under `linked: activity-plan:`). `kdx` pushes triggers at order 75, after plans (50), projects (60) and assistants (70).

**Inline in a project template** — a `triggers:` block, materialized at project create. `triggerMetadata` is copied verbatim; a trigger that fails validation fails the create with a 400. See the **project-template** skill.

**API** — `POST /api/triggers` with `projectId`; `PUT /api/triggers/{id}` validates the row the update produces. Pause with `POST /api/triggers/{id}/disable` — a `DELETE` is permanent. **URI** — `trigger://<orgSlug>/<projectSlug>/<triggerSlug>`.

## Delivery

Event kinds deliver **at least once**: no dedupe, no ordering, no cycle detection, no retry of a failed fire. `document_locked` replays locks from the last 24 hours that produced no activity, marked `reconciled: true`. **Make the plan idempotent**; a plan that must not overlap can set `serializePerProject: true` in its `metadata`. Schedule triggers deliver **at most once per slot**, retry transient delivery failures, and leave a refused fire `BLOCKED` for retry or cancel.

The started activity carries `triggerKind` (`DOCUMENT_LOCKED`, `SCHEDULE`, …) and `triggerMetadata` `{"sourceTriggerId": "<trigger id>"}`; a schedule fire adds `scheduledFor` and `armedBy`. Failures surface as `trigger.fire_failed` (`binding_missing`, `activity_plan_not_found`, `inputs_invalid`, `strict_binding_violation`, `project_inactive`); every evaluation emits `trigger.evaluated` (`matched` / `no-match` / `eval-error`).

## Declared but inert

| Field | Reality |
|---|---|
| `triggerMetadata` on any kind but `schedule` | Refused with a 400. It is not a way to pass config to the plan: use `inputMapping`, or defaults in the plan's inputs. |
| `metadata` | Free-form. Stored and returned; nothing interprets it. |
| `eventKind: task_created` / `activity_completed` / `manual` | Accepted and stored — never dispatched. |

`inputMapping` is the opposite case: fully honoured, but the UI's trigger editor does not expose it. Author it in YAML or via the API.

## Common mistakes

| Mistake | What happens / fix |
|---|---|
| `eventKind: TASK_CREATED` or `task.statusChanged` | 400. Lowercase snake_case, one of the seven. |
| Authoring on `task_created` / `activity_completed` / `manual` | Silent forever. Use a dispatched kind. |
| Filtering on `newStatusSlug`, `taskTemplateSlug`, `result` | Those keys are in no payload. Never fires. |
| `eventFilter: 'reconciled != true'` | False when the key is absent. Use `$not($exists(reconciled))`. |
| `inputMapping` as a YAML mapping | Values become literal strings. Use `'{ "key": fieldName }'`. |
| Empty filter on `task_status_changed` | Also fires on assignment, team change, lock, unlock and take-next. |
| `cron: "@daily"`, six fields, or `timezone: Local` | 400 naming the rule. Five fields, explicit zone. |
| Expecting missed slots to catch up | One fire after an outage; enabling never runs a past slot. |
| A slot inside the DST-skipped hour | Never fires that day. Schedule outside 02:00–03:00 local. |
| Restricting both day fields (`0 6 1 * MON`) | Fires on the 1st **and** every Monday — standard cron OR. |
| Run now on an event trigger | 409 `TRIGGER_NOT_SCHEDULED`. |
| Plan not bound to the project | Event: the fire is lost. Schedule: `PLAN_UNBOUND`, every slot. Bind first. |
| Deleting a trigger to pause it | Hard delete. Use `/disable` or `enabled: false`. |

## Related skills

- **activity-plan** — the plan a trigger starts, and its `inputOptions`.
- **project-resource** — the binding that makes an org-scoped plan usable in the project.
- **project-template** — the `triggers:` block and getting the plan binding into the same template.
- **assistant** — why scheduled work is a schedule trigger, not an assistant schedule.
