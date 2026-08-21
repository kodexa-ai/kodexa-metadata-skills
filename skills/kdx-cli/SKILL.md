---
name: kdx-cli
description: "Use when running the Kodexa CLI (kdx) — logging in and managing profiles, listing/describing/deleting resources, validating and applying resource YAML, invoking declared API operations with run, pulling/pushing/deploying metadata with a sync manifest, inspecting local KDDB documents, and the project/store/task/secret/intake/document-family/knowledge command groups."
---

# Kodexa CLI (`kdx`)

`kdx` is a kubectl-style CLI for the Kodexa platform. It discovers resource types from the server's
OpenAPI document at runtime, so the available types follow the server you point at. Top-level
commands: `login`, `config`, `api-resources`, `get`, `describe`, `delete`, `validate`,
`apply`, `run`, `sync`, `document`, `knowledge`, `dataclasses`, `version`, plus the plugin groups
`project`, `document-family`, `store`, `task`, `secret`, `intake`.

References: `references/commands.md` (command + flag inventory), `references/sync-manifest.md`
(sync-config, manifests, resource types, sigils), `references/troubleshooting.md` (symptom → fix).

## The `kdx apply` envelope

**This is the canonical envelope every Kodexa metadata resource file carries.** Sibling skills
describe a type's own fields; those fields sit **flat at the top level** beside these keys.

```yaml
type: task-template          # resource type, kebab-case — picks the write path
slug: invoice-review         # identifier, unique within the org
orgSlug: acme-corp           # target organization
name: "Invoice Review"       # display name
projectSlug: invoice-processing   # ONLY for project-scoped types: assistant, trigger, knowledge-item
# ...the resource's own fields, flat, at this same level
```

```bash
kdx validate -f invoice-review.yaml    # schema check only — contacts no write endpoint
kdx apply    -f invoice-review.yaml
```

**A passing `kdx validate` is weak evidence:** an unknown or misspelled key is only a *warning* and
still exits 0 — no schema the server serves sets `additionalProperties: false`. Only a missing
**required** key or a type mismatch is an error. Read the warnings; never gate CI on its exit code.

`kdx apply` is the strict one. Omitting a key fails it with an exact, unmissable error:

| Missing | Error |
|---|---|
| `type` | `resource file must contain a 'type' field (or pass --type)` |
| `slug` | `<canonical-type> files need a 'slug' field to apply through the sync engine` |
| `orgSlug` | `cannot determine the target organization: add 'orgSlug' to the file or pass --org-slug` |
| `projectSlug` on a project-scoped type | `<canonical-type> is project-scoped: add 'projectSlug' to the file` |

Files from `kdx sync pull` are untyped and org-agnostic — pass the missing keys on the command
line (`--type task-template --org-slug acme-corp`) rather than hand-editing. `kdx apply` always
speaks the v2 (Go) API; on a legacy (Java) deployment it fails and points you at `kdx sync push`.
For a `module` it runs `metadata.build`, packages the **top-level** `contents:` globs, and uploads.

## Getting connected

```bash
kdx login https://platform.kodexa.example.com --profile prod   # browser flow; mints + saves an API key
kdx login https://platform.kodexa.example.com --device         # device-code flow, no local callback
kdx config set-profile local --url http://localhost:8080 --api-key <key>   # or set one by hand
```

Profiles live in `~/.kodexa/config.yaml`; a config-based profile that gets a 401 triggers an
interactive re-login automatically. **`--url` and `--api-key` only override the profile when both
are given** — one alone is silently ignored and the current profile is used, an easy way to write to
the wrong environment.

Exit codes: a server problem-details response carrying **error**-severity items renders as a
formatted list and exits **2**; every other failure (including `kdx validate` finding schema errors)
exits **1**. A problem response carrying **only warnings prints them and exits 0** — a CI step
gating on the exit code calls that a pass. `--strict` escalates those warnings to errors (exit 2);
it has no effect on `kdx validate`.

## Reading resources

```bash
kdx get data-definitions -o json        # lists, walking EVERY page
kdx get modules                         # a TTY gets an interactive table; a pipe gets plain output
kdx get module 00000000-0000-4000-8000-000000000001
```

`kdx get <type>` lists; `kdx get <type> <id>` fetches one. The type name is normalized, so singular
and plural both resolve — the number of positional arguments selects list-vs-get. **That positional
identifier is a UUID, not a slug**, for `get`, `describe` and `delete` alike, so
`kdx describe module acme-corp/invoice-extractor` 404s. Resolve the slug first:

```bash
kdx run modules list-modules --filter "slug:'invoice-extractor' and organization.slug:'acme-corp'" -o yaml
kdx describe module 00000000-0000-4000-8000-000000000002
```

**`kdx run` returns one page and stops** — one request, server default page size 20, maximum 1000, so
`kdx run knowledge-sets list-knowledge-set -o yaml > sets.yaml` silently exports at most 20 records.
Pass `--pageSize 1000`, or use `kdx get <type> -o json`. `kdx get` has no `--filter` flag: filter
with `f` in the interactive table, with a `--filter-name` preset, or with `kdx run`.

### Scoping a query to an organization

```bash
# WRONG — orgSlug is not a query parameter on any list endpoint
kdx run data-definitions list-taxonomies --orgSlug acme-corp
#   → error: unknown parameter(s): --orgSlug

# WRONG — orgSlug is a computed, non-persisted field; this compiles to a column that
# does not exist and the query fails at the database
kdx run data-definitions list-taxonomies --filter "orgSlug:'acme-corp'"

# RIGHT — organization.slug
kdx run data-definitions list-taxonomies --filter "organization.slug:'acme-corp'" -o yaml

# RIGHT — organization.id works everywhere
kdx run data-forms list-data-forms --filter "organization.id:'00000000-0000-4000-8000-000000000003'"
```

`organization.slug` is wired up only on data-definitions, data-stores, document-stores, modules,
prompts, project-templates and the knowledge family. Everywhere else (data-forms, task-templates,
task-statuses, activity-plans, triggers, service-bridges, intakes, assistants) resolve the org UUID
once with `kdx get organizations -o json` and filter on `organization.id`.

Operation names come from the path segment after the resource prefix, falling back to the sanitized
`operationId` when that is empty — rarely the plural you expect (`list-data-stores` does not exist;
it is `list-data-store-metadata`). Run `kdx run <resource>` bare to list what it actually offers.

## Sync in one screen

`kdx sync pull` / `push` / `deploy` move a whole org's metadata between a git repo and an
environment. Two things decide whether it works at all:

**1. Manifest entries are SLUGS, never file paths.** Each list value becomes a resource slug, and
the file is looked up at `<metadata_dir>/<type-dir>/<slug>.yaml`. A path value silently becomes a
nonsense slug — validation only rejects empty strings, so nothing complains until push time.

```yaml
# WRONG — the entry becomes the literal slug "forms/my-form.yml"; the lookup goes
# under resources/data-forms/forms/ and finds nothing, so this deploys nothing
organization:
  data-form: [forms/my-form.yml]

# RIGHT
organization:
  data-form: [invoice-review-form]
```

**2. `api_version:` defaults to `legacy`** — and so does *any* run without `--env`, whatever
credentials you pass. Always sync with `--env <name>`, and put `api_version: v2` on every Go-API
environment, or `activity-plan`, `trigger` and `task-status` are skipped and deletions stop working.

```yaml
environments:
  prod:
    url: https://platform.kodexa.example.com
    api_key_env: KODEXA_PROD_API_KEY
    api_version: v2
```

`kdx sync pull --discover` writes a correct manifest for you — start there rather than hand-rolling
one. Manifest shape, the 22 syncable types and their push order, repository layout, `${org}` and
the sigils, branch/tag mappings: `references/sync-manifest.md`. Three more facts to carry:

- **`kdx sync deploy` has no production guard.** The `--production` profile flag is honoured only by
  `kdx apply`, `kdx delete` and `kdx sync push`. For deploys, run `--dry-run` first, or pass
  `--confirm-all` / `--confirm-each` explicitly.
- **`--force` overrides the conflict skip only.** It never reaches the content-equality test, so
  *removing* a key from a resource file is a silent no-op — comparison is by containment. Deletion is
  honoured only on v2 and only for `data-definition` `taxons` and `task-template` `metadata.properties`.
- **Sigil resolution never fails a push.** `${taskTemplate.…}`, `${taskStatus.…}`, `${docStatus.…}`,
  `${workspace.…}` and `${activityPlan.…}` warn and are left in the payload verbatim when
  unresolvable. Grep the push log for `⚠️` lines before believing a green run.

## Currently broken against the Go API

These still exist and still run, so they look healthy. `references/troubleshooting.md` has the fix for each.

| Command | What happens |
|---|---|
| `kdx store reprocess` / `store stats` / `store reindex` | Call `/api/stores/…`, which the server no longer registers → 404 |
| `kdx get stores` / `kdx describe store …` | Same prefix, same 404. Use `document-stores` / `data-stores` |
| `kdx document-family data` | Calls a removed content-object export path → 404 |
| `kdx task failures`, bulk `kdx task reprocess --project …` | Read a plan key the API renamed; **fail silently** — always "no failures found", exit 0 |
| `kdx sync pull` / `push` on a `workspace:` entry | Still a registry type, but the v2 backend refuses it: every one is skipped, the run exits 0, nothing moves. `/api/workspaces` itself is alive, so `kdx get workspaces` still works |

## Declared but inert

Fields that persist, round-trip and look meaningful, but that nothing reads:

- **`schema_version:`** in `sync-config.yaml` — parsed and never read. There is no validation pass
  for sync-config at all; a missing file yields an empty config, not an error.
- **`manifest_version:`** in a manifest — never inspected by validation or any sync path.
  `--discover` writes `"2"`; the value is documentation for humans only.
- **`--include-ids` / `--friendly-names` / `--inline-audits` / `--include-exceptions`** on
  `kdx document-family data` — the live data-export endpoint reads no query parameters at all.
- **`filters[].label` / `filters[].description`** in `~/.kodexa/presentation.yaml` — interactive
  table only; `--filter-name` matches on `name`.
- **`productionEnvironment` on a profile** is inert for `kdx sync deploy` — never read there.

## Related skills

Each documents one resource type's own fields; the envelope above is shared, and `kdx validate` →
`kdx apply` is where they all terminate: **activity-plan**, **assistant**, **data-definition**,
**data-form**, **intake**, **knowledge-system**, **module**, **project-resource**, **project-template**,
**prompt-template**, **service-bridge**, **task-status**, **task-template**, **trigger**.
