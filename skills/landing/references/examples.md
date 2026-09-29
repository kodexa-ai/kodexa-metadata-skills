# Landing examples

Everything here passes the platform's contract validator and its activation readiness check. The
references must still exist in the organization: the `vendor` project template, the
`invoice-intake` plan and the `invoice-review` task template.

## A three-view landing — `landings/workflow.yaml`

Supervisors (`org-admin`) land on the control room. Everyone else lands on the vendor book and can
switch to the review conveyor. Each vendor is a project, and each invoice batch is an activity.

```yaml
type: landing
slug: workflow
name: Accounts payable workflow
description: Invoice review home for the AP team
metadata:
  schemaVersion: 1
  timezone: America/New_York
  defaultView: vendors
  selection:
    allowUserChoice: true
    allowStandardView: true
    rules:
      - roleNames: [org-admin]
        view: supervisor
  views:
    # Book: one row per vendor project, invoice batches (activities) beneath
    - id: vendors
      surface: book
      title: Vendors
      entity: project
      hierarchy: none
      history: current
      orphanTasks: separate
      scope:
        organizations: { mode: current }
        projects: { mode: allActive }
      identity:
        label: { kind: field, source: project, path: /name }
        key: { kind: field, source: project, path: /options/dataProperties/vendor_id, missingText: No vendor id }
      labels: { singular: vendor, plural: vendors, search: Search vendors }
      columns:
        - id: name
          label: Vendor
          rowType: project
          format: text
          sortable: true
          binding: { kind: field, source: project, path: /name }
        - id: open
          label: Open tasks
          rowType: project
          format: number
          align: right
          sortable: false
          binding: { kind: projection, key: project.openTaskCount, availability: required }
        - id: batch
          label: Invoice batch
          rowType: activity
          format: text
          sortable: false
          binding: { kind: field, source: activity, path: /title }
        - id: review
          label: Review
          rowType: activity
          format: badge
          sortable: false
          binding:
            kind: review
            taskTemplateRefs: ["task-template://${org}/invoice-review"]
            attempts: current
            show: [status, assignee]
            eligibility: hidden
            availability: optional
      filters:
        - { id: active, label: In progress, predicate: { kind: activeActivities } }
        - { id: mine, label: Mine, predicate: { kind: mine } }
        - { id: all, label: All, predicate: { kind: all } }
      defaultFilter: active
      searchFields:
        - { kind: field, source: project, path: /name }
      sort:
        - { columnId: name, direction: asc, nulls: last }
      pagination: { defaultSize: 25, sizes: [25, 50, 100] }
      stats:
        - id: open
          label: Open tasks
          scope: filtered
          format: number
          binding: { kind: metric, key: tasks.open, subject: all, window: { kind: allTime }, availability: required }
      actions:
        - { id: open, label: Open, kind: openProject, availability: required }
        - { id: new-vendor, label: New vendor, kind: createProject, projectTemplateRef: "project-template://${org}/vendor", availability: optional }
        - { id: intake, label: Start invoice batch, kind: startActivity, activityPlanRef: "activity-plan://${org}/invoice-intake", availability: optional }
      refreshSeconds: 60
      navigation: { onComplete: landing, onCancel: landing }

    # Conveyor: one unit of work at a time, platform-ranked
    - id: operator
      surface: conveyor
      title: Review the next invoice
      scope:
        organizations: { mode: current }
        projects: { mode: allActive }
      stats:
        - id: done
          label: Done today
          scope: view
          format: number
          binding: { kind: metric, key: tasks.completedByMeToday, subject: me, window: { kind: today }, availability: optional }
        - id: waiting
          label: Waiting
          scope: view
          format: number
          binding: { kind: metric, key: queue.depth, subject: all, window: { kind: allTime }, availability: required }
      actions:
        - { id: next, label: Start review, kind: takeNext, availability: required }
        - { id: resume, label: Resume held work, kind: resumeHeld, availability: required }
        - id: defer
          label: Cannot do this one
          kind: defer
          requireReason: true
          reasonCodes: [missing-po, needs-help]
          availability: optional
      refreshSeconds: 30
      navigation: { onComplete: takeNext, onCancel: landing }
      unit:
        task:
          badge: { kind: field, source: task, path: /template/name }
          title: { kind: field, source: task, path: /title }
          context: { kind: field, source: project, path: /name }
        group:
          badge: { kind: text, parts: [{ literal: Invoice batch }] }
          title: { kind: field, source: taskGroup, path: /name }
      queue: { workUnits: taskAndGroup, ordering: platform }
      heldWork: { enabled: true, showAge: true }

    # Control room: grouped backlog plus a reviewer roster
    - id: supervisor
      surface: controlRoom
      title: AP operations
      audience: { roleNames: [org-admin] }
      scope:
        organizations: { mode: current }
        projects: { mode: allActive }
      stats:
        - id: open
          label: Open tasks
          scope: view
          format: number
          binding: { kind: metric, key: tasks.open, subject: all, window: { kind: allTime }, availability: required }
        - id: behind
          label: Days behind
          scope: view
          format: number
          tone: warn
          binding: { kind: metric, key: queue.daysBehind, subject: all, window: { kind: lastDays, days: 7 }, availability: required }
      actions:
        - { id: reassign, label: Reassign, kind: reassign, availability: required }
        - { id: release, label: Release held, kind: releaseHeld, availability: optional }
      refreshSeconds: 30
      navigation: { onComplete: landing, onCancel: landing }
      queue:
        title: Queue by review screen
        groupBy: taskTemplate
        columns:
          - id: screen
            label: Screen
            rowType: queue
            format: text
            sortable: false
            binding: { kind: dimension, key: taskTemplate, availability: required }
          - id: waiting
            label: Waiting
            rowType: queue
            format: number
            sortable: true
            binding: { kind: metric, key: queue.depth, subject: all, window: { kind: allTime }, availability: required }
          - id: oldest
            label: Oldest item
            rowType: queue
            format: duration
            sortable: true
            binding: { kind: metric, key: queue.oldestAgeSeconds, subject: all, window: { kind: allTime }, availability: required }
        sort:
          - { columnId: waiting, direction: desc, nulls: last }
        pagination: { defaultSize: 25, sizes: [25, 50] }
      people:
        title: Reviewers
        dataset: assignments
        columns:
          - id: reviewer
            label: Reviewer
            rowType: person
            format: text
            sortable: false
            binding: { kind: dimension, key: reviewer, availability: required }
          - id: assigned
            label: Assigned
            rowType: person
            format: number
            sortable: true
            binding: { kind: metric, key: people.assigned, subject: all, window: { kind: allTime }, availability: required }
        sort:
          - { columnId: assigned, direction: desc, nulls: last }
        pagination: { defaultSize: 25, sizes: [25, 50] }
```

Notes:

- The `selection.rules` entry opens `supervisor` for `org-admin` holders. Everyone else opens
  `defaultView: vendors`, and conveyor users can switch to `operator`. To send reviewers straight
  to the conveyor, add a rule for their role.
- The book's cells follow the renderer's rules. `project` columns read `source: project`, the
  `batch` column reads `source: activity`, and the rest are a projection and a review.
- `tasks.open` and `queue.daysBehind` are `required`: both have providers.
  `tasks.completedByMeToday` is `optional`, so a gap renders as an explained blank rather than
  blocking the view.

## The task templates the conveyor dispatches

Policy lives on the task template, nested under `metadata:`.

```yaml
type: task-template
slug: invoice-review
name: Invoice review
metadata:
  workPolicy:
    maxHeldWork: 1
    leaseMinutes: 15
    requiredSkills: [invoice-review]
    reviewBudgetMs: 300000
    requireOutcome: true
    allowedTransitions:
      ready: [reviewing]
      reviewing: [approved, rejected]
    sla:
      businessMinutes: 480
      dueSource: arrival
      escalateAfterMinutes: 30
      calendar:
        timezone: America/New_York
        weekdays: [1, 2, 3, 4, 5]
        startMinute: 540
        endMinute: 1020
        holidays: ["2026-12-25"]
    routing:
      agingMinutes: 60
      urgentWithinMinutes: 30
      urgencyPriority: -100
    quality:
      sampleBasisPoints: 1000
      reviewers: 2
      consensus: majority
      outcomeSlugs: [approved, rejected]
```

A second review that the first reviewer may never take:

```yaml
type: task-template
slug: second-review
name: Second review
metadata:
  workPolicy:
    distinctFromTaskTemplateRefs: ["task-template://${orgSlug}/invoice-review"]
```

Before either template is used, **grant `invoice-review` to the reviewers** from the control room's
person row. Also make sure every status a task can start in has a key in `allowedTransitions`.

## Deploying it

```yaml
# manifest.yaml
manifest_version: "2"
metadata_dir: resources
organization:
  project-template: [vendor]
  activity-plan:    [invoice-intake]
  task-template:    [invoice-review, second-review]
  task-status:      [ready, reviewing, approved, rejected]
  landing:          [workflow]            # the resource: resources/landings/workflow.yaml
landing:
  ref: landing://${org}/workflow          # activate it after a clean, unfiltered push
```

```bash
kdx sync push --target acme --env prod --dry-run   # validates the landing against this push
kdx sync push --target acme --env prod             # stages every resource, then activates
kdx landing get --org-slug acme-corp               # active vs latest revision
```

Pushing a later edit to `landings/workflow.yaml` re-activates automatically, as long as the
manifest keeps `landing.ref` and the push is complete and unfiltered. Without the block, run
`kdx landing set landing://acme-corp/workflow --org-slug acme-corp`.
