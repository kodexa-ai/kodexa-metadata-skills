# Schedule triggers — reference

The load-bearing rules are in `SKILL.md`. This file holds the definition rules in full, the
schedule's state and endpoints, delivery, permissions and caps.

## The definition: `triggerMetadata`

```yaml
eventKind: schedule
triggerMetadata:
  cron: "0 6 * * MON"
  timezone: America/New_York
  jitterSeconds: 900
```

Exactly these three keys. An unknown key, a second JSON value, or a missing `triggerMetadata` is a
400. Every rule is checked on create and on every update, against the row the update produces — a
PUT that only flips `eventKind` to `schedule` still needs a valid `triggerMetadata`. The 400's
message names the rule that failed:

| Rule | Refused example | Message (abridged) |
|---|---|---|
| Five fields: minute, hour, day-of-month, month, day-of-week | `0 0 6 * * MON` | `has 6 fields; exactly 5 are required` |
| No descriptors | `@daily`, `@every 1h` | `descriptors such as @every and @daily are not accepted` |
| Zone outside the expression | `CRON_TZ=UTC 0 6 * * *` | `set the zone in triggerMetadata.timezone` |
| An explicit IANA zone | `""`, `Local`, `" UTC"`, `Eastern` | `must be an explicit IANA zone` / `is not a known IANA zone` |
| `jitterSeconds` 0 to 3600 | `7200` | `must be between 0 and 3600` |
| A next slot exists | `0 0 30 2 *` | `never fires` |
| At least 15 minutes between each of the next ten slots | `*/5 * * * *` | `fires 5m0s apart …; slots must be at least 15m0s apart` |

Every other event kind refuses a non-empty `triggerMetadata` (`null` and `{}` count as empty).

The definition is stored canonical: runs of whitespace in `cron` fold to one space and
`jitterSeconds` is written even when you left it out (as `0`). A re-sent definition that differs
only in spelling therefore compares equal and does not re-arm.

Cron syntax is the standard five-field form: lists (`1,15`), ranges (`MON-FRI`), steps (`*/30`),
and month and weekday names. One classic trap: when **both** day-of-month and day-of-week are
restricted, a day matching **either** fires. `0 6 1 * MON` fires on the 1st *and* on every Monday.

`POST /api/triggers/schedule-preview` validates a definition exactly as a write does and lists its
next runs from database time, without saving anything. Any authenticated user may call it.

```json
{ "cron": "0 6 * * MON", "timezone": "America/New_York", "jitterSeconds": 900, "count": 3 }
```

It answers `{ "triggerMetadata": {canonical}, "runs": [{ "at", "local", "key" }] }` — `count`
defaults to 3, at most 10 — or the same 400 a write would give. `at` is the UTC instant, `local`
the wall time in the zone, and `key` the slot's identity (`2026-10-05T06:00 America/New_York`).

## Slots, arming and missed runs

- **A slot is its local wall time plus zone**, never the instant. A wall time that a DST change
  skips does not exist, so it never fires. A wall time that a DST change repeats fires once.
- **Arming.** When a schedule trigger is created, enabled, or its `eventKind` or `triggerMetadata`
  changes, the schedule is re-armed: the next tick sets the next fire to the **first slot after
  now**. A slot that passed before the change never fires, so enabling a trigger at 06:05 does not
  run the 06:00 slot. Each re-arm bumps `scheduleVersion` and records who armed it (`armedBy`).
- **What does not re-arm:** renames, `eventFilter` and `inputMapping` edits, `activityPlanRef`
  changes, and a full-body save that re-sends the same definition.
- **Fire once.** After an outage of any length a due trigger fires once, for the slot that was
  due, and its next fire becomes the first slot after now. Slots in between are not replayed.
- **Jitter.** A fire is delivered `hash(triggerId) mod jitterSeconds` seconds after its slot: the
  same offset every time for one trigger, spread across the window for many. Run now has no jitter.
- The scheduler ticks every 20 seconds on every orchestrator replica, and Postgres row locks decide
  which replica fires a slot. Any number of replicas fires each slot at most once.

## State: `GET /api/triggers/{id}/schedule`

Needs `trigger:read`. A trigger that is not a schedule trigger answers 409 `TRIGGER_NOT_SCHEDULED`.
None of this is part of the trigger resource or its YAML; it lives in its own table.

| Field | Meaning |
|---|---|
| `enabled`, `schedule` | the trigger's flag and its stored definition |
| `nextFireAt` | the armed slot; absent while arming (up to one tick) or while disabled |
| `nextRuns` | the next three slots after now, as the scheduler will fire them |
| `lastSlot`, `lastStatus`, `lastError`, `lastActivityId` | the latest fire and how it ended |
| `scheduleVersion`, `armedBy` | re-arm count and the user whose change last armed it |
| `runRequestedAt`, `runRequestedBy` | a Run now waiting for the next tick |
| `fires` | the 20 newest fires: `id`, `spawnKey`, `runNow`, `state`, `scheduledFor`, `nextAttemptOn`, `attemptCount`, `errorCode`, `childActivityId` |

`lastStatus` values:

| Status | Meaning |
|---|---|
| `PENDING` | a fire is queued for delivery |
| `STARTED` | the activity started; `lastActivityId` names it |
| `RETRYING` | delivery failed transiently and will be retried |
| `BLOCKED` | delivery was refused; retry or cancel the fire |
| `CANCELLED` | the trigger changed before delivery, or a user cancelled the fire |
| `FILTER_FALSE` | `eventFilter` did not match this slot |
| `FILTER_TIMEOUT` | `eventFilter` or `inputMapping` ran past its 2-second deadline |
| `FILTER_ERROR` | `eventFilter` failed to evaluate |
| `INPUTS_INVALID` | `inputMapping` failed or did not produce an object |
| `PLAN_UNBOUND` | `activityPlanRef` is not bound to the project |
| `PROJECT_INACTIVE` | the project is archived or pending deletion |
| `SCHEDULE_INVALID` | the stored definition could not be armed, or has no further slot |
| `ERROR` | the fire failed and stays due for the next tick |

Only `ERROR` leaves the slot due, and `SCHEDULE_INVALID` stops the schedule. Every other status is
recorded after the schedule has already moved on to its next slot, so the next slot fires on time
whatever the last one did. An older slot's late outcome never overwrites a newer slot's status.

## Run now: `POST /api/triggers/{id}/run`

Needs `trigger:run`. Answers 202 `{ triggerId, runRequestedAt, runRequestedBy }`; the next tick
(within about 20 seconds) fires it through the same filter, mapping, plan and project checks as a
slot, with `scheduledFor` set to the request time. Refusals are 409, with the reason in
`details.code`:

| `details.code` | When |
|---|---|
| `TRIGGER_NOT_SCHEDULED` | the trigger is not a schedule trigger |
| `TRIGGER_DISABLED` | the trigger is disabled |
| `RUN_PENDING` | a Run now is already waiting, or a fire of this trigger is still queued, retrying or blocked |
| `RUN_LIVE` | an activity this trigger started is still `PENDING` or `RUNNING` |
| `RUN_COOLDOWN` | the last Run now was under five minutes ago; `Retry-After` and `details.retryAfterSeconds` say how long to wait |

A blocked fire therefore blocks Run now until you retry or cancel it.

## Retry and cancel a fire

`POST /api/triggers/{id}/spawn-requests/{requestId}/retry` and `…/cancel`, where `requestId` is a
`fires[].id`. Both need `trigger:run`.

- **Retry** moves a `BLOCKED` or `RETRYABLE` fire to `RETRYABLE`, due now. Any other state is 409
  `INVALID_STATE`.
- **Cancel** moves a fire that has not started to `CANCELLED`. It is idempotent.
- A fire that already started is 409 `ALREADY_STARTED` — act on the activity instead. Another
  trigger's fire is a 404.

## Delivery

A queued fire is delivered like any other child start, and the activity is a root run: no parent,
`startedByTriggerId` set, `triggerKind: SCHEDULE`, and `triggerMetadata`
`{ sourceTriggerId, scheduledFor, armedBy }`.

| Outcome at delivery | Result |
|---|---|
| The trigger was deleted, disabled, re-armed, re-kinded or re-planned since the fire was queued, or its project archived | fire `CANCELLED`, `trigger.fire_failed` |
| The target is busy, or the API predates schedule fires | retried with back-off (`RETRYING`) |
| The start was refused | fire `BLOCKED`, `trigger.fire_failed`; retry or cancel it |
| Accepted | fire `STARTED`, `trigger.fired` |

Re-enabling a trigger does not revive a cancelled fire.

## Permissions

| Permission | Gates | Held by |
|---|---|---|
| `trigger:schedule` | creating a schedule trigger; changing a schedule trigger's `eventKind`, `triggerMetadata`, `activityPlanRef` or `enabled`; making any trigger a schedule trigger; `/enable` and `/disable` on a schedule trigger | org-owner, org-admin, project-admin, project-editor |
| `trigger:run` | Run now, retry, cancel | the same four roles |
| `trigger:enable-disable` | flipping `enabled` on any trigger, by PUT or `/enable` `/disable` | the same four roles |

org-member does not hold `trigger:schedule` or `trigger:run`: it can author event triggers but not
schedule ones. The checks are scoped to the trigger's project. A refusal is a 403 naming the
permission. Templates are the exception: a project template's schedule triggers are not checked
against the user creating the project.

## Caps

At most **5 schedule triggers per project** and **500 per organization** (deployment defaults). One
more is refused with 409, `code: TRIGGER_SCHEDULE_LIMIT`, `details: { scope, limit }`. The caps
apply to creates, to updates that make a trigger a schedule trigger, and to template
materialization.

## Telemetry

Every evaluation emits `trigger.evaluated` (`matched`, `no-match`, `eval-error`, `eval-timeout`).
A slot that cannot start emits `trigger.fire_failed` with `binding_missing`, `project_inactive` or
`inputs_invalid`. Delivery emits `trigger.fired` or `trigger.fire_failed`.
