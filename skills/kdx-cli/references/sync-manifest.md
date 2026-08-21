# `kdx sync` — config, manifests and resource types

`kdx sync` moves an organization's metadata between a git repository and a Kodexa environment.
Three commands: `pull` (environment → repo), `push` (repo → environment), `deploy` (push, driven by
git branch or tag mappings).

## Start with discovery

Do not hand-roll your first manifest. Point the CLI at an org that already has content and let it
write one:

```bash
kdx sync pull --discover --target acme --env prod --discover-dir resources
```

Discovery writes `manifest_version: "2"`, keys every list by the type's URI scheme, fills in every
slug it finds, and then pulls the files into the layout below. Edit from there.

Pass `--env` even here. The API version is resolved from the named environment and from nothing
else: with no `--env` there is no environment to read, so the run falls back to `legacy` no matter
which profile, URL or key you authenticated with.

## Repository layout

```
repo/
  sync-config.yaml                                  # environments, targets, branch/tag mappings
  manifest.yaml                                     # what to sync (slugs)
  .sync-state/                                      # NOT under metadata_dir — see below
    prod/
      acme-corp.yaml                                # per-(environment, org) change-sequence watermark
  resources/                                        # = metadata_dir, relative to the manifest file
    data-definitions/
      invoice-taxonomy.yaml                         # <type-dir>/<slug>.yaml
    data-forms/
      invoice-review-form.yaml
    knowledge-sets/
      vendor-list.yaml
      vendor-list-attachments/                      # attachments land beside the YAML
    projects/
      invoice-processing/                           # project-scoped types nest under the project
        assistants/
          invoice-assistant.yaml
        triggers/
          auto-review.yaml
```

- Org-scoped: `<metadata_dir>/<type-dir>/<slug>.yaml`
- Project-scoped: `<metadata_dir>/projects/<project-slug>/<type-dir>/<slug>.yaml`
- `metadata_dir` is the manifest's own key, resolved relative to the **manifest file's** directory.
- Filenames are the slug **verbatim** — case is deliberately preserved, because the server allows
  two resources whose slugs differ only in case. Spaces become `-`. Push reads `<slug>.yaml` and
  falls back to `<slug>.yml`; pull always writes `.yaml`.
- **`.sync-state/` does NOT follow the manifest's `metadata_dir`.** It is written under the CLI's
  own metadata directory — `--metadata-dir` if you pass one, otherwise the directory containing
  `sync-config.yaml`. Two different knobs with confusingly similar names; if you move
  `metadata_dir`, the state files stay where they were. The file is
  `.sync-state/<env>/<org-slug>.yaml`, the conflict watermark `--force` overrides — decide
  deliberately whether to commit it or gitignore it.
- `task-template` lands flat under `task-templates/` against a v2 backend and under
  `projects/<slug>/task-templates/` against a legacy backend — the CLI consults the backend's
  effective scope when choosing the path. (`task-status` uses the same rule, but legacy skips the
  type entirely; see the api_version note below.)

## `sync-config.yaml`

```yaml
schema_version: "2.0"          # convention only — nothing reads it

environments:
  dev:
    url: https://dev.kodexa.example.com
    api_key_env: KODEXA_DEV_API_KEY
    api_version: v2
  prod:
    url: https://platform.kodexa.example.com
    profile: prod              # take the API key from this named kdx profile instead
    api_version: v2

targets:
  acme:
    organization: acme-corp
    manifests:
      - manifest.yaml

branch_mappings:
  - pattern: main
    target: acme
    environment: prod
  - pattern: "release/*"       # shell glob, NOT a regex
    targets:
      - target: acme
        environment: dev       # per-entry environment override

tag_mappings:
  - pattern: "v*"
    target: acme
    environment: prod
```

**`api_version:` defaults to `legacy`.** Anything but `v2` (trimmed, case-insensitive) selects the
legacy Java protocol, and behaviour diverges from `kdx apply` (which is always v2). On legacy the
backend refuses three registry types outright — `activity-plan`, `trigger` and `task-status` —
logging `Skipping <type> <slug> - not supported by this server (api_version)`, and it never honours
key deletions. (`task-status` is refused because legacy has no standalone endpoint for it; the data
round-trips inside the project YAML's `taskStatuses` array instead.) Conversely `workspace` exists
**only** on legacy. Set `api_version: v2` on every environment running the Go API.

The version is read from the *named* environment only. `--from-profile` / `--from-url` /
`--to-profile` / `--to-url` authenticate but select no environment, so a run without `--env` — and a
`deploy` whose matched mapping omits `environment:` — resolves to `legacy` regardless of the
`environments:` block. Give every mapping an `environment:`.

**Environment credentials.** `api_key_env` names an environment variable holding the API key;
`profile` names a saved `kdx` profile. **`profile:` wins outright** — when it is set, the profile
supplies both the key and (unless `url:` overrides it) the URL, and `api_key_env` is never read.
`api_key_env` only takes effect on an environment with no `profile:`. Do not set both and expect
the variable to take precedence.

**Patterns are shell globs**, matched with the same semantics as filesystem globbing: `*` does not
cross `/`, so `release/*` matches `release/1.0` but `*` alone does not match `feature/x`. They are
not regular expressions.

**Unmatched `--tag` is a hard error** (`no tag mapping found for tag "…"`); an unmatched `--branch`
is a soft no-op that prints "nothing to deploy". A mapping uses either `target:` (single) or
`targets:` (a list, each entry able to override `environment`).

`schema_version` is parsed and never read. There is no validation pass for `sync-config.yaml` at
all — a missing file yields an empty config, not an error, so a typo in the filename looks like
"nothing configured" rather than a failure.

## The manifest

```yaml
manifest_version: "2"
metadata_dir: resources           # relative to this file

organization:                     # org-scoped resources, listed BY SLUG
  data-definition: [invoice-taxonomy, vendor-taxonomy]
  data-form:       [invoice-review-form]
  document-store:  [invoice-store]
  module:          [invoice-extractor]
  activity-plan:   [invoice-flow]
  task-template:   [invoice-review]
  task-status:     [todo, in-review, approved]
  knowledge-set:   [vendor-list]

projects:                         # a project key means "push this project"
  invoice-processing:
    assistant: [invoice-assistant]
    trigger:   [auto-review]
    linked:                       # org-scoped resources to bind to this project after push
      data-definition: [invoice-taxonomy]
      task-template:   [invoice-review]

skip_projects: [archived-pilot]
```

### Entries are slugs, never paths

Every list value is copied verbatim into the resource's slug, and the file is then looked up at
`<metadata_dir>/<type-dir>/<slug>.yaml`.

```yaml
# WRONG — becomes the literal slug "forms/my-form.yml"; the lookup goes under
# resources/data-forms/forms/ and finds nothing
organization:
  data-form: [forms/my-form.yml]

# RIGHT
organization:
  data-form: [invoice-review-form]
```

Manifest validation only rejects **empty** strings, so a path entry passes validation cleanly and
fails much later, at push time, as a missing file.

### Keys are resource types, values are slugs

Keys resolve through the type registry's aliases (see the table). An unknown key is a hard error the
moment the target is resolved: `manifest <path>: invalid resource type "<key>"`. `data-definition`
and `taxonomy` both resolve; discovery always writes the URI scheme (`data-definition`).

There is no `model-runtime` alias. `module`, `modules`, `model`, `models`, `cloudmodule`,
`cloud-module`, `cloudModule`, `cloudmodel`, `cloud-model` and `cloudModel` all resolve to `module`;
a `model-runtime:` key fails as an invalid resource type.

### `linked:` vs inline

Inside `projects.<slug>`, top-level keys are resources the project **owns** (project-scoped types).
The `linked:` sub-block lists **org-scoped** resources to bind to the project after push. Splitting
them makes the distinction readable at a glance.

### Legacy shape

An older shape still parses and is normalized on load:

```yaml
resources:                        # old name for `organization:`
  data-definition: [invoice-taxonomy]
  project: [invoice-processing]   # dropped in the new shape — project keys are the source of truth
modules:                          # dropped in the new shape
  - modules/invoice-extractor.yaml
```

**Mixing the two is a hard error**: `manifest cannot contain both 'resources:' (legacy shape) and
'organization:' (new shape) at the top level; choose one`.

In the new shape there is no top-level `modules:` block — module deploy intent is derived from
`organization.module`, each slug resolving to `<metadata_dir>/modules/<slug>.yaml`.

The legacy `modules:` list is the **one** place a path is accepted: entries resolve relative to
`metadata_dir`, and a bare name falls back to `modules/<name>.yaml`. Nothing else in a manifest
takes a path.

`manifest_version:` is never read: validation ignores it and no sync code path consults it.
Discovery writes `"2"`; treat the key as documentation.

## Syncable resource types

22 entries. The **manifest key** column is the type's URI scheme, which is what discovery writes
and what `kdx sync pull --discover` will hand you. Push order is ascending, so dependencies land
before their dependents.

| Manifest key | Canonical name | Scope | Directory | Push order |
|---|---|---|---|---|
| `label` | `label` | org | `labels/` | 10 |
| `data-definition` | `datadefinition` | org | `data-definitions/` | 20 |
| `data-form` | `dataform` | org | `data-forms/` | 20 |
| `document-store` | `documentstore` | org | `document-stores/` | 20 |
| `data-store` | `datastore` | org | `data-stores/` | 20 |
| `module` | `module` | org | `modules/` | 20 |
| `prompt-template` | `prompttemplate` | org | `prompt-templates/` | 20 |
| `service-bridge` | `servicebridge` | org | `service-bridges/` | 20 |
| `knowledge-item-type` | `knowledgetype` | org | `knowledge-item-types/` | 30 |
| `knowledge-feature-type` | `featuretype` | org | `knowledge-feature-types/` | 30 |
| `knowledge-feature` | `featureinstance` | org | `knowledge-feature-instances/` | 40 |
| `activity-plan` | `activityplan` | org | `activity-plans/` | 50 |
| `intake` | `intake` | org | `intakes/` | 55 |
| `project-template` | `projecttemplate` | org | `project-templates/` | 58 |
| `project` | `project` | org | `projects/` | 60 |
| `workspace` | `workspace` | project | `workspaces/` | 63 (legacy backend only) |
| `knowledge-set` | `knowledgeset` | org | `knowledge-sets/` | 65 |
| `task-template` | `tasktemplate` | org (legacy: project) | `task-templates/` | 65 |
| `task-status` | `taskstatus` | org (legacy: project) | `task-statuses/` | 65 |
| `assistant` | `assistant` | project | `assistants/` | 70 |
| `knowledge-item` | `knowledgeitem` | project | `knowledge-items/` | 70 |
| `trigger` | `trigger` | project | `triggers/` | 75 |

Traps in that table:

- **`knowledge-feature` writes to `knowledge-feature-instances/`**, not `knowledge-features/`.
- **`data-definition`, not `taxonomy`, is the manifest key** discovery emits, and the directory is
  `data-definitions/`. `taxonomy` is only an alias.
- **`project-template` has a real `project-template://` scheme** and sits at 58 — before projects
  (60), so a created project's template lineage resolves.
- **`intake` sits at 55**, between activity-plans (50) and project-templates (58), because an
  intake's `activityPlanId` is rewritten to an `${activityPlan.<slug>}` sigil that must resolve
  against a plan that already exists.
- **`knowledge-set` is 65, after projects (60)**, so a project-level knowledge set can resolve its
  project.
- **`task-template` and `task-status` are org-scoped on v2 but project-scoped on legacy.** On a
  legacy backend they stay inline under `projects.<slug>` in the manifest, and `task-template`
  lands under `projects/<slug>/task-templates/` on disk. `task-status` is skipped on legacy.
- **`workspace` is legacy-only** — the Go API removed it, so a v2 backend refuses the type. It sits
  at 63, before task-templates (65), because task templates carry `${workspace.<slug>}` sigils.

## Portability: `${org}`

Pull rewrites the org slug to `${org}`; push substitutes the destination org slug back.

The contract is anchored on `<orgSlug>/`, plus the bare `orgSlug` field:

| On disk | Resolves to (pushing into `acme-corp`) |
|---|---|
| `${org}/invoice-taxonomy` | `acme-corp/invoice-taxonomy` |
| `orgSlug: ${org}` | `orgSlug: acme-corp` |
| a string that is exactly `${org}` | `acme-corp` |

`${org}-suffix` is **malformed** — the substitution is anchored on `${org}/`, so it never fires.
Unlike the sigils below, this one is caught: after substitution, push scans the payload for any
surviving `${org}` token and **aborts the whole push run** (not just that resource) with

```
<type> <slug> still contains unresolved ${org} placeholders after substitution (<paths>)
  — fix the placeholder or report a kdx bug
```

Modules are scanned on both the create and update paths for the same reason.

## Sigils

Five sigils make environment-specific UUIDs portable. **None of them fails a push.** Every one of
them warns and leaves the sigil in the payload verbatim; the single exception drops a field.

| Sigil | Appears in | Resolved against | On failure |
|---|---|---|---|
| `${taskTemplate.<slug>}` | assistant pipeline step options | the destination **project**'s task templates | warn; sigil left literal |
| `${taskStatus.<slug>}` | assistant pipeline step options | the destination **project**'s task statuses | warn; sigil left literal |
| `${docStatus.<slug>}` | assistant pipeline step options | the destination **project**'s document statuses | warn; sigil left literal |
| `${workspace.<slug>}` | task-template payload | the destination **project**'s workspaces | warn; sigil left literal (silent no-op if the workspace list cannot be fetched) |
| `${activityPlan.<slug>}` | `intake.activityPlanId` | `activity-plan://<org>/<slug>` via the resolver | genuine 404 → the `activityPlanId` field is **dropped**; transient error → sigil left literal |

Only `${activityPlan.…}` uses the URI resolver. The other four are resolved by fetching the
destination project's own statuses / templates / workspaces and matching on slug — so the sigil
resolves per-project, and a project that has not been pushed yet resolves nothing.

Project-templates are not a sigil consumer. Assistants, task-templates and intakes are the only
three payload types any sigil is applied to.

Because nothing aborts, **grep the push log for `⚠️`** before treating a run as clean.

One more reference that warns rather than fails: a `trigger` whose `activityPlanRef` points at an
activity plan that FAILED to push earlier in the same run is still created. Push logs a warning that
the trigger will never fire until the plan exists — the server never validates the ref.

## Pull → edit → validate → push

```bash
# 1. Bring the environment down into the repo
kdx sync pull --target acme --env prod

# 2. Edit YAML. Check one file against the server's schema before writing anything.
kdx validate -f resources/task-templates/invoice-review.yaml

# 3. Rehearse
kdx sync push --target acme --env dev --dry-run

# 4. Push for real
kdx sync push --target acme --env dev
```

Single-file changes can skip the manifest entirely — `kdx apply -f <file>` runs through the same
push engine, so placeholders, sigils and module upload behave identically.

## Deploy

```bash
kdx sync deploy --target acme --env prod --dry-run
kdx sync deploy --branch main
kdx sync deploy --tag v1.2.0
kdx sync deploy --target acme --env prod --confirm-all --output-json report.json
```

With no `--target`/`--branch`/`--tag`, deploy detects the current git branch and matches it against
`branch_mappings`. `--target` requires `--env`.

**Deploy has no production guard.** The `--production` profile flag is read by `kdx apply`,
`kdx delete` and `kdx sync push` only — `deploy` never looks at it, and it authenticates from the
`environments:` block rather than from the current profile anyway. Its only confirmations are the
opt-in `--confirm-all` and `--confirm-each`. Rehearse with `--dry-run`.

`--output-json <path>` writes a machine-readable deployment report — the hook for CI.

## What `--force` does and does not do

`--force` suppresses the change-sequence **conflict skip**, so a resource the server has modified
since your last pull is pushed anyway. It stops there:

- It never reaches the content-equality test, so push still no-ops on content the comparison
  considers unchanged.
- **Removing a key from a resource file is generally a silent no-op**, with or without `--force`.
  Comparison is by containment: a key you deleted is indistinguishable from a key you never
  mentioned. The two declared exceptions, where deletion is honoured, are `data-definition`
  `taxons` and `task-template` `metadata.properties` — and only on a **v2** backend, because the
  legacy backend re-mints defaulted fields inside those regions on the way back out.

To remove something that is not one of those two, delete it through the API or the UI.

## Attachments

`kdx sync pull` writes knowledge-set attachments into `<slug>-attachments/` beside the YAML and
injects `attachmentPath` / `attachmentId` so they round-trip. Unchanged files are skipped by content
hash, so re-pulling does not churn the repo.
