# Where a resource file lives

## The rule

```
<metadata_dir>/<type-dir>/<slug>.yaml                              # org-scoped
<metadata_dir>/projects/<project-slug>/<type-dir>/<slug>.yaml      # project-scoped
```

- `<metadata_dir>` is the manifest's own `metadata_dir:` key, resolved relative to the **manifest
  file's** directory.
- `<type-dir>` is fixed per resource type — the table below. It is not derived from the type name,
  and two of them are not what the type name suggests.
- `<slug>.yaml` — pull always writes `.yaml`; push reads `<slug>.yaml` and falls back to
  `<slug>.yml`.
- There is **no organization directory level**. One metadata directory holds one organization's
  resources; the org comes from the manifest and the environment, which is also why `orgSlug` is
  stripped out of pulled files and `${org}` is substituted into them.

A knowledge set's binary attachments land next to its YAML, in `<slug>-attachments/`.

## Per-type directories

22 syncable types. "Scope" decides whether the file sits flat under `<type-dir>/` or nested under
`projects/<project-slug>/`.

| Type | Directory | Scope | Envelope? |
|---|---|---|---|
| `label` | `labels/` | org | no |
| `data-definition` | `data-definitions/` | org | **yes** |
| `data-form` | `data-forms/` | org | **yes** |
| `document-store` | `document-stores/` | org | **yes** |
| `data-store` | `data-stores/` | org | **yes** |
| `module` | `modules/` | org | **yes** |
| `prompt-template` | `prompt-templates/` | org | **yes** |
| `service-bridge` | `service-bridges/` | org | **yes** |
| `knowledge-item-type` | `knowledge-item-types/` | org | no |
| `knowledge-feature-type` | `knowledge-feature-types/` | org | no |
| `knowledge-feature` | `knowledge-feature-instances/` | org | no |
| `activity-plan` | `activity-plans/` | org | **yes** |
| `intake` | `intakes/` | org | no |
| `project-template` | `project-templates/` | org | **yes** |
| `project` | `projects/` | org | no |
| `workspace` | `workspaces/` | project | no |
| `knowledge-set` | `knowledge-sets/` | org | **yes** |
| `task-template` | `task-templates/` | org (legacy: project) | **yes** |
| `task-status` | `task-statuses/` | org (legacy: project) | **yes** |
| `assistant` | `assistants/` | project | no |
| `knowledge-item` | `knowledge-items/` | project | no |
| `trigger` | `triggers/` | project | no |

### The two directory names that catch people

- **`data-definition` → `data-definitions/`**, never `taxonomies/`. `taxonomy` remains a valid
  spelling for the `type:` key and a valid URI scheme; it is not the directory.
- **`knowledge-feature` → `knowledge-feature-instances/`**, never `knowledge-features/`. The API
  endpoint *is* `/api/knowledge-features`, which is where the mismatch comes from.

Two more worth knowing:

- **`prompt-template` → `prompt-templates/`** on disk, while the API endpoint is `/api/prompts` and
  the stored `type` defaults to `prompt`. Three spellings, all correct in their own layer.
- **`project` → `projects/`** collides visually with the `projects/<project-slug>/` tree that holds
  project-scoped resources. They are different things: `projects/<slug>.yaml` is a project's own
  definition, `projects/<slug>/assistants/x.yaml` is a resource inside that project.

### Scope caveats

- `task-template` and `task-status` are org-scoped and sit flat against a current server. Against a
  legacy one they are project-scoped: `task-template` moves to
  `projects/<project-slug>/task-templates/`, and `task-status` is not supported at all.
- `workspace` is project-scoped and legacy-only.
- `label` rows have no `slug` column — a `label://` URI resolves on `name` instead.

## What is not under `metadata_dir`

The sync state file — the change-sequence watermark push compares against — is written under the
sync root (`--metadata-dir`, auto-discovered when omitted), **not** the manifest's `metadata_dir`:

```
.sync-state/<environment>/<org-slug>.yaml
```

Move `metadata_dir` and the state files stay put. Decide deliberately whether to commit that
directory or ignore it: committing it makes conflict detection shared across a team, ignoring it
makes every first push on a fresh clone unconditional.

## Which keys a pulled file does and does not carry

`kdx sync pull` writes the server's resource with the environment-specific and server-owned keys
removed, so a pulled tree diffs cleanly across environments. Removed: `id`, `uuid`,
`organization` / `organizationId`, `project` / `projectId`, `orgSlug`, `projectSlug` (kept on
knowledge sets, where it is real data), `ref`, `searchText`, `createdOn`, `updatedOn`,
`changeSequence`, `yamlSource`, `deleted`, `extensionPackRef`, and `owner`. `uuid`, `createdOn`,
`updatedOn` and `changeSequence` are additionally stripped from **every nested map at every depth**,
as is a nested `id` wherever a non-empty sibling `slug` makes the slug the portable identity.

Kept: `type` when the server returned one, and the resource's own fields. That is why a pulled
data-definition file carries `type: taxonomy` — the server's default — while a pulled activity-plan
file carries no `type:` unless someone authored one, since the server never mints it for that type.
`kdx apply` needs both keys, so applying a pulled file stand-alone means passing `--org-slug`
always, and `--type` as well for the four types the server gives no default. `kdx apply` also
refuses a file with no `slug:`.
