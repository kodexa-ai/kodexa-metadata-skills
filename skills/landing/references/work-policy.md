# Task-template `workPolicy` — who may take, hold and finish work

A conveyor has no eligibility settings of its own. The rules live on each **task template**, under
`metadata.workPolicy`; task templates keep their body nested under `metadata:` (see
**task-template**). The policy is saved with the template. It is then enforced by a **database
guard on every task and task-group write** — take-next, a supervisor reassign, the ordinary task
screen, the API — not only on landings.

```yaml
type: task-template
slug: invoice-review
name: Invoice review
metadata:
  workPolicy:
    maxHeldWork: 1                 # 1–100 units held at once
    leaseMinutes: 15               # 1–1440; renewable assignment lease
    requiredSkills: [invoice-review]
    reviewBudgetMs: 300000         # 100–3600000; feeds tasks.reviewBudgetMetRate
    requireOutcome: true
    allowedTransitions:            # from-status slug → allowed next status slugs
      ready: [reviewing]
      reviewing: [approved, rejected]
    sla:
      businessMinutes: 480         # 1–43200 working minutes
      dueSource: arrival           # arrival | activityFilingDate
      escalateAfterMinutes: 30     # 0–43200
      calendar:
        timezone: America/New_York # IANA, not "Local"
        weekdays: [1, 2, 3, 4, 5]  # ISO 1=Monday … 7=Sunday, distinct
        startMinute: 540           # opening hours within ONE day, at least 60 minutes long
        endMinute: 1020
        holidays: ["2026-12-25"]   # YYYY-MM-DD, at most 366
    routing:
      agingMinutes: 60             # 1–43200: one rank better per elapsed interval
      urgentWithinMinutes: 30      # these two come as a pair
      urgencyPriority: -100        # lower ranks are claimed first
    quality:
      sampleBasisPoints: 1000      # 0–10000; 1000 = a 10% sample
      reviewers: 2                 # 1–9
      consensus: majority          # majority | unanimous
      outcomeSlugs: [approved, rejected]   # 2–20 distinct status slugs
```

## Save-time checks

Violating a range above is a 400 (`workPolicy.maxHeldWork must be between 1 and 100`, and so on).
The save also checks:

- **Only `quality` is strict.** Its four keys are all required, and an unknown key under it is a
  400. **Every other key is decoded leniently: a misspelt key (`leaseMinute`) is dropped without an
  error.** Read the template back after saving.
- **`distinctFromTaskTemplateRefs`** holds up to 30 refs `task-template://<org>/<slug>`, where the
  org is the literal slug or `${orgSlug}`. Every ref must exist and must not name the template
  itself.
- Skills, reasons and slugs are tokens matching `^[A-Za-z0-9_-]{1,100}$`. `requiredSkills` holds at
  most 30; `allowedTransitions` at most 50 keys, each with 1–50 targets.

## What the guard refuses at run time

Every refusal is a constraint error on the write, carrying the reason:

| Setting | Refused with | When |
|---|---|---|
| `requiredSkills` | `required_skill_missing` | the assignee lacks an unexpired grant of every listed skill |
| `maxHeldWork` | `work_capacity_reached` | the assignee already holds that many unfinished units (tasks or groups) in the organization |
| `distinctFromTaskTemplateRefs` | `prerequisite_review_missing` / `distinct_reviewer_required` / `review_history_unavailable` | no completed review of the named template exists in this activity's attempt lineage, or the same person did it, or its actor is unknown |
| `leaseMinutes` | `active_work_lease_required` | the completer does not hold an unexpired lease on the unit |
| `allowedTransitions` | `status_transition_not_allowed` | the status change is not listed, **including a change out of a status that has no key** |
| `requireOutcome` | `completion_outcome_required` | completion without a task signal `outcome` of `APPROVED`, `REJECTED`, `NEEDS_REWORK` or `ESCALATED` |
| any identity-bound rule | `reviewer_identity_required` | the write has no acting user |

Two further consequences of the guard:

- **Moving governed work is blocked.** Once its template has a policy, a task's project, template
  and creating activity are immutable (`governed_work_identity_is_immutable`).
- **Grants come first.** Skills are granted per organization member, with an optional expiry, from
  the control room's person row (`PUT /api/organizations/{orgId}/members/{userId}/work-skills`).
  Adding `requiredSkills` before anyone holds the skill empties every queue that template feeds.

## Leases, SLA, routing

- **Leases.** A lease starts when a unit is assigned. The Workflow app's presence heartbeat renews it
  while the unit is open there. A background sweep reclaims an expired lease from unlocked work and
  records the reclaim. A group uses the smallest `leaseMinutes` among its tasks.
- **SLA.** The deadline counts `businessMinutes` inside the calendar's opening hours, skipping
  non-working days and holidays in the calendar's timezone. It is stored apart from the date-only
  task due date. A breach is recorded once per task and deadline, and shown, with who acknowledged
  it, in the control room. It sends no email. `dueSource: activityFilingDate` follows the activity
  workflow context's `dueDate` instead of arrival time.
- **Routing.** Take-next orders by a routing score, lowest first. The score starts at the unit's
  `priority`:
  - `agingMinutes` subtracts one per elapsed interval since arrival;
  - `urgentWithinMinutes` caps the score at `urgencyPriority` once the SLA deadline (or due date)
    is that close;
  - ties break on effective arrival, then creation time, then id.

  **A task with no `priority` sorts after every prioritised one, and aging never promotes it.**
  Urgency still applies. A group scores as the best of its members, computed with the group's own
  priority under each member's template policy.

## Sampled quality review

`quality` enrols a stable `sampleBasisPoints` share of **newly created or newly bound** tasks.
Existing tasks are not enrolled by a later edit. Each case freezes its policy, so editing the
template cannot waive a pending review.

- Every `outcomeSlugs` entry must be an active, project-bound `DONE` status.
- `reviewers` distinct people vote. A strict majority (or everyone, for `unanimous`) settles the
  case, and a tie needs an adjudicator. Only the agreed `DONE` status can then complete the task.
- A task in a sampled case cannot join a task group.
- `task:quality-manage` governs gold answers, adjudication and the accuracy metrics. No role is
  granted it by name, so only a `*:*` wildcard holds it: `org-owner`, `org-admin`, and
  `project-admin` on its projects.

Surface quality in a landing with `source: task, path: /qualityState`, and with the
`quality.reviews`, `quality.goldReviews` and `quality.goldAccuracy` metrics.
