# `kdx` troubleshooting

## Commands that are currently broken

Each of these is still registered and still runs. Three fail loudly with a 404; the last fails
**silently**, which is worse.

### `kdx store reprocess` / `kdx store stats` / `kdx store reindex` → 404

They call `/api/stores/{ref}/…`. The Go API registers **no `/api/stores/…` routes at all** — only
`/api/document-stores/{id}/…` and `/api/data-stores/…`.

Working replacement for reprocessing:

```bash
# 1. Find the families you want (walks every page)
kdx get document-families -o json

#    …or filter server-side (one page; raise --pageSize)
kdx run document-families list-document-family \
    --filter "store.slug:'invoice-store'" --pageSize 1000 -o json

# 2. Reprocess one family — this endpoint is live
kdx run document-families reprocess --id <document-family-id>
```

`kdx store upload` and `kdx store watch` are unaffected and work — they use the document-store
upload route and the document-family route respectively.

### `kdx get stores` / `kdx describe store <id>` → 404

Same cause: the `store` resource type is hard-wired to the `/api/stores` prefix and overrides
anything discovery finds. Use the concrete types instead:

```bash
kdx get document-stores -o json
kdx get data-stores -o json
```

### `kdx document-family data` → 404

It resolves the latest content object and then GETs a content-object export path that no longer
exists. Even once repaired, its four boolean flags would be inert — the live endpoint reads no
query parameters.

```bash
kdx run document-families data-export --id <document-family-id> -o json > extracted.json
```

### `kdx task failures` and bulk `kdx task reprocess --project …` → silently empty

Both read `plan.plannedItems` from each task. The API renamed that structure: an activity's steps
serialize as `steps`, and `plannedItems` no longer appears anywhere on the platform. The extraction
therefore never matches, so `kdx task failures` always prints "No failed tasks found." and bulk
reprocess always prints "No failed plans found matching the filter" — **exit code 0 in both cases.**
An empty result here means nothing.

Working replacement:

```bash
# Failed activities in a project
#   NOTE: /api/activities has no `project` relationship — use project.id, not project.slug
kdx run activities list-activity \
    --filter "lifecycleState:'FAILED' and project.id:'<project-uuid>'" \
    --pageSize 1000 -o yaml

# Retry one — this half of the CLI is current
kdx task reprocess <activity-id>
```

`lifecycleState` values: `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `CANCELLED`, `REPLANNED`.

Single-ID `kdx task reprocess` posts to the activity retry endpoint and **retries the FAILED and
CANCELLED steps while preserving completed work**. It does not restart from the beginning. To
supersede an activity with a fresh one instead:

```bash
kdx run activities replan --id <activity-id>     # original is marked REPLANNED
```

## Symptom → cause → fix

| Symptom | Cause | Fix |
|---|---|---|
| `unknown parameter(s): --orgSlug` | `orgSlug` is not a query parameter on any list endpoint | Scope with `--filter "organization.slug:'…'"` or `--filter "organization.id:'…'"` |
| A `--filter "orgSlug:'…'"` query errors at the database | `orgSlug` is a computed, non-persisted field; it compiles to a column that does not exist | Use `organization.slug` (where registered) or `organization.id` (everywhere) |
| `operation 'list-data-stores' not found` | Operation names fall back to the sanitized `operationId` | `kdx run data-stores list-data-store-metadata`; run `kdx run <resource>` bare to list real names |
| An export written with `kdx run … -o yaml` has exactly 20 records | `kdx run` issues one request; server default `pageSize` is 20 | Add `--pageSize 1000`, or use `kdx get <type> -o json`, which walks every page |
| `kdx describe`/`kdx delete`/`kdx get <type> <x>` 404s on a slug | The positional identifier substitutes into `{id}` and must be a UUID | Resolve first: `kdx run <plural> list-<op> --filter "slug:'…' and organization.slug:'…'"` |
| `unknown flag: --stdin` on `kdx secret set` | The flag is `--from-stdin` | `--from-stdin`, or `--from-file` / `--from-env` |
| `kdx secret …` returns 503 | Secrets are backed by AWS Secrets Manager and it is not configured on that deployment | Nothing CLI-side; the deployment needs the service wired up |
| Store commands 404 with a `:version` ref | Store URIs are unversioned; `org/slug:1.0.0` parses as a three-segment project-scoped URI and the resolver rejects version suffixes outright | Use `org/slug` (or `org:slug`, a `document-store://org/slug` URI, or a bare store UUID) |
| Wrote to the wrong environment despite passing `--url` | `--url` and `--api-key` override the profile **only together**; one alone is silently ignored | Pass both, or use `--profile` |
| `kdx sync` deployed nothing, no error | Manifest entries were file paths, not slugs — validation only rejects empty strings | Rewrite as slugs; `<metadata_dir>/<type-dir>/<slug>.yaml` is derived for you |
| `Skipping <type> <slug> - not supported by this server (api_version)` for activity-plans / triggers / task-statuses | The environment defaulted to `api_version: legacy`, which refuses those three types | Add `api_version: v2` to that environment in `sync-config.yaml` |
| Same message even though the environment declares `api_version: v2` | No environment name reached the run, so the version fell back to `legacy` — `--from-profile` / `--from-url` / `--to-*` do not select one | Pass `--env <name>` on `pull` / `push`, and give every `branch_mappings` / `tag_mappings` entry an `environment:` for `deploy` |
| Same message for `workspace`, and `api_version: v2` does not fix it | The v2 backend declares `workspace` unsupported, so `kdx sync` skips every one and still exits 0. The `/api/workspaces` endpoints themselves are alive — `kdx get workspaces` and `kdx describe workspace <id>` still work | Drop `workspace` from the manifest; manage workspaces through the API, not through sync |
| `kdx apply` fails with a pointer to `kdx sync push` | `kdx apply` always speaks the v2 (Go) API | Use `kdx sync push` with `api_version: legacy`, or target a v2 server |
| `manifest cannot contain both 'resources:' … and 'organization:' …` | Legacy and current manifest shapes mixed at the top level | Pick one; the current shape is `organization:` |
| `manifest <path>: invalid resource type "<key>"` | Manifest key is not a registered type or alias (e.g. `model-runtime:`) | Use the manifest key from the resource-type table in `sync-manifest.md` |
| `no tag mapping found for tag "…"` | `--tag` matches against `tag_mappings:`, a separate block from `branch_mappings:` | Add a `tag_mappings:` entry; patterns are shell globs, not regexes |
| `kdx sync deploy --branch <x>` prints "nothing to deploy" | Unmatched branch is a soft no-op (unlike `--tag`) | Check the glob — `*` does not cross `/` |
| A pushed assistant/task-template/intake still contains a literal `${…}` sigil | Sigil resolution warns and passes through; it never aborts a push | Read the push log for `⚠️`; push the referenced project/plan first, then re-push |
| `<type> <slug> still contains unresolved ${org} placeholders after substitution` | The contract is `${org}/<slug>`; a malformed spelling such as `${org}-suffix` never substitutes, and push aborts the resource rather than shipping the literal | Fix the file to `${org}/<slug>` — the message names the offending key paths |
| Removed a key from a resource file, push says "up to date" | Comparison is by containment; removals are a no-op, `--force` included | Only `data-definition` `taxons` and `task-template` `metadata.properties` honour deletion, and only on a v2 backend — otherwise delete via the API/UI |
| `⚠️  CONFLICT <org>/<slug> — server changeSequence is N but was M when last pulled on <date>. Skipping` | The server's change sequence moved past the `.sync-state` watermark | Pull first and merge, or `--force` to overwrite. (`kdx apply` does no conflict tracking at all.) |
| Cannot find `.sync-state/` under the manifest's `metadata_dir` | It is written under the CLI's metadata directory — `--metadata-dir`, else the directory holding `sync-config.yaml` — which is a different setting from the manifest's `metadata_dir` key | Look beside `sync-config.yaml` for `.sync-state/<env>/<org-slug>.yaml` |
| Module code did not upload | Top-level `contents:` globs matched nothing, or a `metadata.build` step failed | Check the globs (legacy files nest them under `metadata.contents`) and read the build output — apply runs build steps before packaging |
| `filter '<name>' not found for resource type '<type>'` | `--filter-name` takes a preset name, not a filter expression | Define the preset in `~/.kodexa/presentation.yaml`, or use `kdx run … --filter` |
| `kdx get` opened a full-screen table in CI | Interactive mode triggers when `-o table` and both stdout and stdin are terminals | Pass `-o json` (or pipe) for scripted use |
| A command printed `⚠` validation warnings and still exited 0 | Only error-severity problem-details items exit 2; warnings alone exit 0 | Pass `--strict` to escalate warnings to errors (exit 2) |
| `kdx validate` exited 0, but a key in the file has no effect after `kdx apply` | `validate` reports an unknown or misspelled key as a **warning**, not an error — no served schema sets `additionalProperties: false` | Read the `Result: N error(s), M warning(s)` line and the warning list; `--strict` does not change this (it only escalates *server* problem-details) |
| `required flag(s) "pattern" not set` on `kdx document locate` | The search pattern is the `--pattern` flag, not a positional; the positional is the KDDB path | `kdx document locate doc.kddb --pattern "…"` |
| `kdx run document-families data …` returned the family's metadata, not extracted values | `data` and `data-export` are different endpoints despite similar spec summaries | Use `data-export` for extracted data objects |

## Common mistakes

- **Treating an empty `kdx task failures` as good news.** It is broken; see above.
- **Assuming `kdx get` filters.** There is no `--filter` flag. Filter with the interactive `f` key,
  a `--filter-name` preset, or `kdx run`.
- **Assuming `kdx run` paginates.** It does not.
- **Assuming plural means list.** `kdx get modules` and `kdx get module` resolve identically; the
  number of positional arguments is what selects list-vs-get.
- **Trusting a green `kdx sync deploy` against production.** Deploy has no production guard and no
  confirmation unless you ask for one. Run `--dry-run` first.
- **Putting file paths in a manifest.** They pass validation and deploy nothing.
- **Omitting `api_version: v2`.** Sync silently speaks the legacy protocol.
- **Adding `:version` to a resource URI.** Version-suffixed URIs are rejected by the resolver.
- **Hand-editing `kdx sync pull` output to add `type:`/`orgSlug:`.** Pass `--type` and `--org-slug`
  to `kdx apply` instead.
- **Skipping `kdx validate`.** It writes nothing, is fast, and surfaces unknown keys (warnings) and
  missing required keys (errors) before a write reaches the server.
- **Reading only `kdx validate`'s exit code.** Unknown and misspelled keys are warnings; exit 0
  means "nothing fatal", not "the file is right".
