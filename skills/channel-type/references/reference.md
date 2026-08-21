# Channel type reference

## What ships in the box

Five rows are inserted by platform migrations, each only when no row with that slug exists — so an
installation that already has one keeps its own. Later migrations converge `skills` and `mcpServers`
on `task`, `project` and `project_admin`; they deliberately never touch `moduleRefs` or
`systemPromptFragment`, which are the genuinely per-installation keys.

| Slug | `mcpServers` | `skills` |
|---|---|---|
| `task` | `kodexa-document`, `kodexa-module-runner`, `kodexa-platform`, `kodexa-ui` | `document-edit`, `document-read`, `kodexa-document-workflow`, `kodexa-ui` |
| `project` | `kodexa-document`, `kodexa-module-runner`, `kodexa-platform`, `kodexa-ui` | `activity-execution`, `activity-plan`, `data-definition`, `data-form`, `document-edit`, `document-read`, `kodexa-document-workflow`, `kodexa-ui`, `organization`, `project`, `service-bridge`, `task-status`, `task-template`, `trigger` |
| `project_admin` | `kodexa-platform`, `kodexa-ui` | the `project` list **minus** `document-read` and `document-edit` |
| `org_admin` | `kodexa-platform`, `kodexa-ui` | `activity-plan` |
| `workspace_global` | `kodexa-platform`, `kodexa-ui` | *(none)* |

All five ship with `moduleRefs: []`.

Two things to notice:

- The `task` and `project` merges are **additive** — a name you added by hand survives. The
  `project_admin` merge is not: it also strips `document-read`, `document-edit` and
  `kodexa-document`, so that row converges on a document-free surface however you left it.
- As shipped, `workspace_global` mounts the `kodexa-platform` server with **no skills**, so that
  server carries no tools at all — and `org_admin` carries only the activity-plan ones. That is
  exactly the shape a misconfigured row takes too, so never read "the server is listed" as "the chat
  can do things"; read the `skills` list.

## Skill names, and what each one needs mounted

Naming a skill without its server is inert; mounting `kodexa-platform` without at least one of its
eleven gated names leaves that server empty.

| Skill | Needs mounted | Unlocks |
|---|---|---|
| `activity-plan` | `kodexa-platform` | reading and drafting org-scoped activity plans and their steps |
| `activity-execution` | `kodexa-platform` | starting, watching, cancelling and retrying activities (needs `activity-plan` alongside it to discover plans) |
| `data-definition` | `kodexa-platform` | reading and drafting taxonomies |
| `data-form` | `kodexa-platform` | reading and drafting review forms |
| `task-template` | `kodexa-platform` | reading and drafting task templates |
| `task-status` | `kodexa-platform` | reading task statuses (read-only from the agent) |
| `service-bridge` | `kodexa-platform` | reading and drafting service bridges |
| `trigger` | `kodexa-platform` | reading and drafting project triggers |
| `project` | `kodexa-platform` | reading a project and managing its resource bindings |
| `organization` | `kodexa-platform` | discovering org-level assets, including module refs |
| `knowledge-loop` | `kodexa-platform` **and** `kodexa-document` | the knowledge-loop workflow; it is the one name that gates tools on both servers |
| `document-edit` | `kodexa-document` | the additive document-editing tools; without it the document server is read-only |
| `document-read` | `kodexa-document` | gates nothing — guidance only. Document reads are always on once the server is mounted |
| `filing-analysis` | `kodexa-document` | gates nothing — guidance only. Analyst-style questioning of financial filings (10-K, 10-Q, annual reports) attached to a task; builds on `document-read` |
| `kodexa-ui` | `kodexa-ui` | gates nothing, and is added automatically whenever that server mounts |

The drafting skills (`data-definition`, `data-form`, `activity-plan`, `task-template`,
`service-bridge`, `trigger`) carry preview / discard / undo tools and **no save tool**: they stage
changes for the user to save in the UI. That is a property of the skill packs, not of the row.

## API surface

| Call | Notes |
|---|---|
| `GET /api/channel-types` | Paginated, unfiltered by permissions. `?filter=slug:'task'&pageSize=1` is the lookup both the UI and the runtime use |
| `GET /api/channel-types/{id}` | Permission-checked — effectively platform-admin only |
| `POST /api/channel-types` | `409` on a duplicate slug. The schema marks `slug` required; the server does not — it falls back to slugifying `name`, or stores an empty slug |
| `PUT /api/channel-types/{id}` | Whole-`metadata` replacement; honours `changeSequence` when you send it |
| `DELETE /api/channel-types/{id}` | Hard delete. Chats that referenced the row keep working with their pointer cleared |
| `GET /api/channel-types/{id}/sequence` | Current `changeSequence`, for a read-modify-write |

Create, update and delete all require a platform-admin role, because the row has no organization for
an ordinary grant to hang off.

## Editing a row that has an underscore slug

`project_admin`, `org_admin` and `workspace_global` cannot be updated through `PUT` — the slug format
check runs against the stored value even when the request omits `slug`, and underscores fail it. Edit
the JSON in place instead, for example adding one skill name without disturbing the rest of the
object:

```sql
UPDATE kdxa_channel_types
SET metadata = jsonb_set(
      metadata, '{skills}',
      (SELECT COALESCE(jsonb_agg(DISTINCT s ORDER BY s), '[]'::jsonb)
         FROM (SELECT jsonb_array_elements_text(COALESCE(metadata->'skills','[]'::jsonb)) AS s
               UNION SELECT 'data-form') u))
WHERE slug = 'org_admin';
```

Rows whose slug is already hyphen-only (`task`, `project`) can be edited through the API normally —
fetch the row, change one key inside `metadata`, and `PUT` the whole object back.

## How the configuration reaches a chat

1. A chat is created with `channelTypeId` pointing at a row. The binding is permanent. If the slug
   lookup found nothing, the chat is created unbound and stays that way.
2. Each turn, the UI sends the bound row's **slug** with the request.
3. The runtime looks the row up by that slug, once per agent session, and caches it for the life of
   the session. No row, no slug, or a failed lookup all fall back to an empty configuration — zero
   MCP servers, zero skills, no prompt fragment — and the chat still runs.
4. Servers named in `mcpServers` are mounted, skills named in `skills` are copied into the working
   directory and used to gate tools, `moduleRefs` are downloaded into the same directory, and
   `systemPromptFragment` is appended to the standing instructions after the platform's own context
   block and before any module-supplied addendum.
5. Module refs supplied by the caller for that turn are merged after the row's own, de-duplicated,
   row-first.

Headless agent runs started by an activity plan have no chat and therefore no channel type; their
equivalent grant is baked into the runtime image. Nothing you write here affects them.
