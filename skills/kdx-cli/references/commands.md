# `kdx` command reference

Every command and flag below was read off the CLI's own command definitions. Anything not listed
here does not exist — check `kdx <group> --help` before inventing a flag.

## Global flags

Persistent on the root command, so they work on every subcommand:

| Flag | Default | Notes |
|---|---|---|
| `-o, --output` | `table` | `table` \| `json` \| `yaml` \| `markdown` |
| `--profile` | current profile | Named profile from `~/.kodexa/config.yaml` |
| `--url` | — | Only takes effect **together with** `--api-key` |
| `--api-key` | — | Only takes effect **together with** `--url` |
| `--debug` | `false` | Shorthand for `--log-level=debug` |
| `--log-level` | `info` | `debug` \| `info` \| `warn` \| `error` \| `silent` |
| `--log-timestamps` | `false` | |
| `--skip-production-confirm` | `false` | Bypasses the production prompt and logs the bypass |
| `--strict` | `false` | Escalates server problem-details **warnings** to errors (exit 2) |

Exit codes: **2** when the server returns an RFC 9457 problem-details body containing
error-severity items (or warning-severity items under `--strict`); **1** for every other failure;
**0** on success. A problem body carrying **only warnings** prints them and still exits **0** — do
not let a CI gate treat that as a clean run without also reading the output.

## `kdx login <url>`

Runs a browser (or device-code) authentication flow, mints an API key, saves it as a profile and
makes that profile current.

```bash
kdx login https://platform.kodexa.example.com --profile prod
kdx login https://platform.kodexa.example.com --no-browser     # print the URL instead of opening one
kdx login https://platform.kodexa.example.com --device         # device-code flow, no local callback
kdx login https://platform.kodexa.example.com --timeout 300
```

| Flag | Default |
|---|---|
| `--profile` | prompted, suggested from the URL host |
| `--timeout` | `180` seconds |
| `--no-browser` | `false` |
| `--device` | `false` |

If the chosen profile name already exists for that URL, `login` asks before replacing it. A
config-based profile that receives a 401 during any command triggers this same re-login
interactively.

## `kdx config`

```bash
kdx config set-profile local --url http://localhost:8080 --api-key <key>
kdx config set-profile prod  --url https://platform.kodexa.example.com --api-key <key> --production
kdx config list-profiles        # alias: kdx config get-profiles
kdx config current-profile
kdx config use-profile prod
kdx config delete-profile old
```

`set-profile` flags: `--url` and `--api-key` (both required), `--production`. Config lives in
`~/.kodexa/config.yaml` (directory created `0700`).

`--production` marks the profile so that `kdx apply`, `kdx delete` and `kdx sync push` prompt for
confirmation before acting. **`kdx sync deploy` does not consult this flag.**

## `kdx api-resources`

```bash
kdx api-resources
kdx api-resources --refresh        # re-fetch the OpenAPI document
kdx api-resources --schemes-only   # raw schemes, without OpenAPI metadata
```

## `kdx get`

```bash
kdx get <type>                    # list — walks every page
kdx get <type> <uuid>             # fetch one
kdx get modules --filter-name my-preset
kdx get module <uuid> --download --download-path ./impl.zip
```

Flags: `--filter-name`, `--download`, `--download-path`. **There is no `--filter` flag.**

- **Pagination:** list mode fetches every page. (The loop is disabled only when a `page` parameter
  is pinned, which can happen through a `--filter-name` preset's `params:` — there is no `--page`
  flag.)
- **Interactive table:** with `-o table` (the default) and both stdout *and* stdin attached to a
  terminal, `kdx get <type>` opens a full-screen table instead of printing. Keys: `/` query,
  `f` inline filter, `Enter` apply, `Ctrl+Enter` save filter, `PgUp`/`PgDn` page, `r` refresh,
  `p` preset filters, `e` entity commands, `i` item commands, `s` stream item commands,
  `q`/`Ctrl+C` quit. Piping, or any of `-o json|yaml|markdown`, gives plain output.
- **`--download`** fetches a module's implementation ZIP and errors on any non-module type.
- The positional identifier is a **UUID**.

### `--filter-name` presets

Presets come from a bundled default merged with an optional `~/.kodexa/presentation.yaml`. Matching
is on `name` (case-insensitive); `label` and `description` are for the interactive table only.

The `entities:` key must equal the resource type **as you type it**, lowercased — the preset lookup
runs before the singular/plural normalization, so a block under `data-definitions:` does not apply
to `kdx get data-definition`. An `entities.default:` block covers everything else.

```yaml
# ~/.kodexa/presentation.yaml
entities:
  data-definitions:
    filters:
      - name: acme
        label: Acme org
        description: Everything owned by acme-corp
        filter: "organization.slug:'acme-corp'"     # a real filter expression
        query: "invoice"                            # optional full-text query
        params:                                     # optional extra query params
          pageSize: "500"
```

```bash
kdx get data-definitions --filter-name acme -o yaml
```

An unknown name errors with `filter '<name>' not found for resource type '<type>'`.

## Filter syntax

The `filter` query parameter is a SpringFilter expression:

| | |
|---|---|
| `:` equals | `name:'Invoice Review'` |
| `!` not equals | `status!'ARCHIVED'` |
| `~` like / contains | `name~'*invoice*'` |
| `>` `<` `>=` `<=` | `createdOn>'2025-01-01'` |
| `and` / `or` / `not(...)` | `status:'ACTIVE' and name~'*invoice*'` |

Strings and dates take single quotes, numbers are bare, booleans are `true`/`false`. Sort is
`field:direction`, semicolon-separated for multiple: `sort=name:asc;createdOn:desc`.

## `kdx describe <type> <uuid>` / `kdx delete <type> <uuid>`

Both take exactly two positional arguments and both substitute the second into the `{id}` slot of
the type's get/delete path — so it must be a **UUID**, never a slug or an `org/slug` ref.
`delete` adds `--force` (appended as `?force=true`) and prompts first on a `--production` profile.

## `kdx validate -f <file>`

Checks a resource file against the server's create request schema (falling back to the update
schema). It contacts no write endpoint — but it is not offline: it needs the server's OpenAPI
document, so it authenticates like any other command and fails with
`no OpenAPI specification available — run 'kdx api-resources --refresh' first` if it cannot get one.

Checks, and their severity — the severity is the part that matters:

| Check | Severity |
|---|---|
| Unknown top-level envelope key (anything outside `type`, `slug`, `orgSlug`, `name`, `metadata`, `storeType`, `spec`) | warning, and only when the file uses a `spec:` wrapper |
| Key not present in the schema | **warning** — error only where the schema sets `additionalProperties: false` |
| Missing required key | error |
| Type mismatch (string vs number, etc.) | error |

**No schema in the served OpenAPI document sets `additionalProperties: false`.** In practice that
makes every unknown or misspelled key a warning, so a file full of typos ends on
`Result: 0 error(s), N warning(s)` and exits **0**. Only a missing required key or a type mismatch
exits **1**. Read the warning list; never treat a zero exit as "the file is right".

The file must carry a `type:` field — `validate` has no `--type` flag, and unlike `apply` it will
not take one from a flag. `--strict` has no effect on `validate` (it only escalates *server*
problem-details warnings). `projectSlug` is not an envelope key, so it reports as an unknown-key
warning on project-scoped files; that is expected and harmless.

## `kdx apply -f <file>`

See the envelope section in `SKILL.md`. Flags:

| Flag | Purpose |
|---|---|
| `-f, --filename` | required |
| `--type` | overrides the file's `type` |
| `--org-slug` | overrides the file's `orgSlug` |

Behaviour worth knowing:

- Types the sync registry knows go through the **same push engine as `kdx sync push`**, so `${org}`
  placeholders, sigils, knowledge-set side channels and module archive upload all apply.
- Types the registry does not know (organizations, users, …) fall back to raw create/update against
  the discovered operations.
- A legacy `type: store` file with `storeType: TABLE|DOCUMENT|MODEL` is remapped to
  `data-store` / `document-store` / `module` with a warning.
- For a module with a top-level `contents:` block, apply runs `metadata.build` steps, packages and
  uploads the implementation, then re-pushes the metadata. A failing build step is the most common
  cause of "code didn't upload". `metadata.contents` is accepted as a legacy fallback location.
- `kdx apply` always requests the v2 (Go) API. On legacy it fails and directs you to `kdx sync push`
  with `api_version: legacy`.

## `kdx run <resource> [operation] [--flags]`

Invokes an operation declared in the server's OpenAPI document for that resource.

```bash
kdx run projects                                  # list this resource's operations + example calls
kdx run modules list-modules --filter "slug:'invoice-extractor'" --pageSize 1000 -o yaml
kdx run document-families data-export --id <uuid> -o json > extracted.json
kdx run document-families reprocess --id <uuid>
kdx run activities retry  --id <uuid>             # retry FAILED/CANCELLED steps, keep completed work
kdx run activities replan --id <uuid>             # supersede with a fresh activity
```

- Path and query parameters become `--<name>` flags. **Any flag the operation does not declare is a
  hard error**: `unknown parameter(s): --foo`. `--orgSlug` is a *path* parameter on the intake
  endpoints only; it is a query parameter nowhere.
- Request bodies go in `--body '<json>'`.
- **No pagination.** One request, one page. Server default `pageSize` is 20, maximum 1000.
- Operation names derive from the path segment after the resource prefix; when that segment is
  empty the sanitized `operationId` is used. Verified examples:
  `data-definitions` → `list-taxonomies`, `data-forms` → `list-data-forms`,
  `data-stores` → `list-data-store-metadata`, `modules` → `list-modules`,
  `knowledge-sets` → `list-knowledge-set` / `get-knowledge-set`, `activities` → `list-activity`,
  `document-families` → `list-document-family` / `data-export` / `reprocess`.
- **When two paths derive the same name, one keeps it and the other gets a `-<method>` suffix — and
  which is which is not stable between runs.** (`POST .../labels` and `DELETE .../labels/{labelId}`
  both derive `labels`; the tie is broken by Go map iteration order.) Always run
  `kdx run <resource>` bare and use the name it prints in that run, or use the dedicated
  `kdx document-family add-label` / `remove-label` commands.

## `kdx version`

Prints the CLI version, git commit and build date.

## `kdx dataclasses <taxonomy-file>`

Generates Python dataclasses from a data-definition/taxonomy file (JSON or YAML).
Flags: `--output-path` (default `.`), `--output-file` (default `data_classes.py`).

## `kdx knowledge`

Attachments on knowledge items. `<knowledge-item-ref>` may be a UUID or a slug.

```bash
kdx knowledge attach <ref> --file ./vendors.csv
kdx knowledge attach <ref> --file ./logo.png --id logo   # id used in markdown references
kdx knowledge attach <ref> --folder ./model-data/        # zipped, then uploaded
kdx knowledge download <ref> --output ./downloaded.zip
```

`attach` requires exactly one of `--file` or `--folder`; `--id` defaults to the sanitized filename.
`download` takes `-o, --output`.

## `kdx document` — local KDDB files

Operates on `.kddb` files on disk. Nothing here talks to the platform. 23 subcommands:

**Overview** — `info`, `stats`, `schema`, `print`

```bash
kdx document info doc.kddb
kdx document print doc.kddb --depth 3
```

**Content extraction** — `text`, `page`, `lines`, `natives (list|extract)`

```bash
kdx document text doc.kddb --pages 1:5
kdx document page 3 doc.kddb
kdx document lines doc.kddb --page 2
kdx document natives list doc.kddb
```

**Content search** — `grep`, `select`, `find`, `locate`

```bash
kdx document grep "total" doc.kddb
kdx document select "//line" doc.kddb
kdx document find doc.kddb --contains "invoice" --type line
kdx document locate doc.kddb --pattern "Total\\s+Due" --type line
```

`locate` takes the pattern as `--pattern` (required; not a positional) and the file path as the one
positional. It returns `nodeId`, `matchStart`, `matchEnd` and `matchText` — the shape `tag` needs.

**Annotations** — `tags`, `features`, `node`, `tag`

```bash
kdx document tags doc.kddb
kdx document node 42 doc.kddb --tags
kdx document tag doc.kddb --node-id 42 --name TOTAL --value "1234.00" --confidence 0.9
```

`locate` is the intended way to find the `--node-id` for `tag`.

**Spatial** — `spatial find`, `spatial bbox`

```bash
kdx document spatial find --region 0,0,300,100 --page 1 doc.kddb
kdx document spatial bbox --page 1 doc.kddb
```

**Data objects** — `data objects`, `data attributes`, `data exceptions`, `data create`,
`data set-attribute`

```bash
kdx document data objects doc.kddb
kdx document data attributes 1 doc.kddb
kdx document data exceptions doc.kddb
```

**Stored values** — `metadata (get|set)`, `external (list|get|set|delete)` — `set` takes valid JSON

**Audit and comparison** — `audit`, `delta`, `diff`

```bash
kdx document audit doc.kddb --object-id 1
kdx document delta changes.bin --data --json --max-ops 50
kdx document diff expected.kddb produced.kddb --output diff.json --severity warn
kdx document diff expected.kddb produced.kddb --layer data --layer steps
```

`diff` compares two KDDB files across four layers — `data`, `steps`, `validations`, `audit` — each
emitting Added / Closed / Preserved / Changed findings. All four are implemented and all four run
by default; `--layer` is repeatable to narrow the run. (The command's own `Long` help text still
describes three of them as stubs — that text is stale, the layers are wired.) Exit code is 0 when
no finding reaches the `--severity` threshold (default `error`), non-zero otherwise.

## `kdx sync`

`pull`, `push`, `deploy`. Full treatment in `sync-manifest.md`. Flag summary:

Persistent on the `sync` group: `--metadata-dir`, `--config`.

| Command | Flags |
|---|---|
| `pull` | `--target` (repeatable), `--env`, `--from-profile`, `--from-url`, `--from-api-key`, `--skip-missing`, `--discover`, `--discover-dir`, `-f/--filter` (regex), `--threads` (4) |
| `push` | `--target` (repeatable), `--env`, `--to-profile`, `--to-url`, `--to-api-key`, `--dry-run`, `-f/--filter` (regex), `--force`, `--threads` (4) |
| `deploy` | `--target` (repeatable), `--env`, `--branch`, `--tag`, `--dry-run`, `--force`, `--confirm-each`, `--confirm-all`, `-f/--filter` (regex), `--output-json`, `--threads` (4) |

`pull`/`push` also accept deprecated `--organization` / `--project`; prefer `--target`.
`--env` names an entry in `sync-config.yaml`'s `environments:` block. `--target` requires `--env`
on `deploy`.

---

# Plugin command groups

Six plugins are registered, each becoming a top-level command: `project`, `document-family`,
`store`, `task`, `secret`, `intake`.

## `kdx project`

```bash
kdx project create --template kodexa/invoice-processor --org acme-corp --name "Invoice Processing"
kdx project create --template kodexa/invoice-processor --org acme-corp --name "Invoice Processing" \
    --description "AP invoice intake and extraction"
```

`--template`, `--org` and `--name` are all required. The server derives the project slug from the
name and auto-suffixes collisions — read the created slug back from the output.

## `kdx intake`

Uploads through an intake endpoint, so the intake's script runs. (`kdx store upload` bypasses it.)

```bash
kdx intake upload acme-corp/invoice-intake ./invoice.pdf \
    --metadata VendorCode=ACME-001 \
    --metadata DocumentType=invoice \
    --label NEEDS-REVIEW=true

kdx intake upload acme-corp/invoice-intake ./invoice.pdf --metadata-file ./metadata.yaml
```

| Flag | Notes |
|---|---|
| `--metadata key=value` | repeatable |
| `--metadata-file <yaml\|json>` | merged **first**; `--metadata` flags override it |
| `--label name=value` | repeatable; bare `name` allowed |
| `--status-id` | initial document-family status |
| `--external-data` | JSON string |
| `--document-version` | |

The ref must be `<org>/<intake-slug>`. Anything that is not a known parameter reaches the intake
script as metadata.

## `kdx store`

```bash
kdx store upload acme-corp/invoice-store ./invoice.pdf
kdx store watch <document-family-id> --label PROCESSED --timeout 600 --poll-interval 3
```

**Store ref format is `org/slug` — unversioned.** `org:slug`, a full `document-store://org/slug`
URI, and a bare document-store UUID also work. Adding `:version` makes the resolver read it as a
three-segment project-scoped URI and fail; version-suffixed URIs are explicitly rejected.

`watch` flags: `--label` (default `PROCESSED`), `--timeout` (default 600s), `--poll-interval`
(default 3s).

`reindex`, `reprocess` and `stats` are registered but call `/api/stores/…`, which no longer exists
— see `troubleshooting.md`.

## `kdx document-family`

```bash
kdx document-family content list <document-family-id>
kdx document-family content download <document-family-id> <content-object-id> -o out.kddb
kdx document-family content download <document-family-id> --latest
kdx document-family add-label <document-family-id> <tag-id> [--path <node-path>]
kdx document-family remove-label <document-family-id> <label-id>
kdx document-family set-status <document-family-id> <status-id>
```

`kdx document-family data` is registered but calls a removed endpoint — use
`kdx run document-families data-export --id <uuid> -o json` instead. That reads the family's latest
KDDB content object out of object storage, so it returns 503 on a deployment with no object store
configured. Do **not** reach for the similarly named `kdx run document-families data` — despite the
spec calling it "Export document family data", it returns the family's `metadata` column reformatted
(`--format json|csv|xml|ndjson`), not the extracted data objects.

## `kdx task`

```bash
kdx task reprocess <activity-id>              # works
kdx task reprocess <activity-id> --dry-run
```

Single-ID `reprocess` retries an activity's **FAILED/CANCELLED steps and preserves completed work**.
To start over from scratch instead, use `kdx run activities replan --id <activity-id>`, which
supersedes the activity with a fresh one and marks the original `REPLANNED`.

`kdx task failures` and bulk `kdx task reprocess --project … --since …` read a plan key the API
renamed and return empty results with exit 0 — see `troubleshooting.md`.

On `/api/tasks`, `activity.*` is the canonical filter prefix; `plan.*` survives as a deprecated
alias. `project` **is** a registered relationship there, so `project.slug:'invoice-processing'`
works against tasks (it does not work against `/api/activities` — use `project.id`).

## `kdx secret`

Organization secrets, backed by AWS Secrets Manager. These endpoints return **503** on a deployment
where that service is not configured.

```bash
kdx secret list acme-corp                                    # names only
kdx secret set  acme-corp MY_API_KEY                         # no-echo prompt (needs a TTY)
kdx secret set  acme-corp MY_API_KEY --from-file ./secret.txt
kdx secret set  acme-corp MY_API_KEY --from-env  SOURCE_ENV_VAR
some-secret-tool read ... | kdx secret set acme-corp MY_API_KEY --from-stdin
kdx secret delete acme-corp MY_API_KEY                       # idempotent
```

**The flag is `--from-stdin`, not `--stdin`.** `--from-file`, `--from-env` and `--from-stdin` are
mutually exclusive; giving two errors out, and giving none without a TTY errors with
`no value source: stdin is not a terminal`. `--from-file` and `--from-stdin` strip trailing
newlines (`--from-env` does not). An empty resulting value is rejected. The command logs the secret
name and a byte count, never the value.

The secrets API (`/api/organizations/{orgId}/secrets`) is published in the OpenAPI document,
including a reveal endpoint at `…/secrets/{name}/value`. `kdx secret list` returns names only;
reading a value is restricted server-side to execution tokens and platform admins.
