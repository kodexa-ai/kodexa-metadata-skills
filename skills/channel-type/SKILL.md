---
name: channel-type
description: "Use when configuring the Kodexa Studio chat agent — choosing which MCP servers, built-in skills, module skill packs and system-prompt fragment a chat loads, editing a channel-type row, or debugging a chat that has no platform tools at all. Covers the metadata payload, the strict opt-in mount rules, and the bind-once channel binding."
---

# Kodexa Channel Type Authoring

## Nothing else in this plugin works like this

A **channel type** is the entire configuration surface for the Studio chat agent: which MCP servers a
chat mounts, which built-in skills land in its working directory, which module skill packs are
downloaded, and what extra instructions get appended to its prompt. A chat with no channel type has
**no platform tools at all**.

| | |
|---|---|
| **Not a synced resource** | No YAML file, no `metadata_dir` directory, no manifest key. `kdx` has no `channel-type` type — pull and push both ignore it. |
| **Not resolver-addressable** | There is no `channel-type://` scheme. Rows are found by slug through the list endpoint: `GET /api/channel-types?filter=slug:'task'&pageSize=1`. |
| **Not org-scoped** | The table has no organization column. One set of rows serves every organization in the database. |
| **No editor screen** | The UI reads channel types (to bind a new chat) and never writes them. |
| **Writes need a platform-admin role** | Because a row carries no organization, no org or project grant can authorize the write; only a platform-admin role passes. Listing is unfiltered and open to any authenticated caller — but `GET /api/channel-types/{id}` runs the same check as a write, so a non-admin can list rows and still get `403` fetching one by id. |

**Five rows ship already**, inserted idempotently by the platform's migrations — `task`, `project`,
`project_admin`, `workspace_global`, `org_admin` — so your job is almost always to *edit* one of
those, not to invent a new one. Later migrations re-converge their `skills` and `mcpServers`:
additively for `task` and `project`, but `project_admin` also gets `kodexa-document`,
`document-read` and `document-edit` **stripped**, so a hand edit giving that row document access
does not survive an upgrade. `moduleRefs` and `systemPromptFragment` are never touched by a
migration. Shipped contents are in `references/reference.md`.

## The slugs the UI actually asks for

The UI picks a slug from where the user is, resolves it to a row id, and sends that id when it
creates the chat. A slug outside this set is never selected, so a row named anything else is dead
weight unless a caller binds it by id itself.

| Slug | Chosen when |
|---|---|
| `task` | there is a task in scope — beats everything else |
| `project` | any route under a project — workspace, document view, reporting, settings |
| `project_admin` | the project administration screen |
| `org_admin` | the organization home screen |
| `workspace_global` | anywhere else — the fallback |

Three of those slugs contain **underscores**, which the API's slug rules forbid (see below). They
exist because they were inserted by migration, not through the API.

## Payload

```json
POST /api/channel-types
{
  "slug": "task",
  "name": "Task chat",
  "description": "Chat opened from inside a task.",
  "metadata": {
    "moduleRefs": ["acme-corp/invoice-extraction"],
    "mcpServers": ["kodexa-document", "kodexa-ui"],
    "systemPromptFragment": "## Role: Task assistant\n\nYou are helping the user...",
    "skills": ["document-read", "document-edit"]
  }
}
```

- **`slug`** — the row's identity, and the field you must actually set. Unique across the whole
  database, not per organization. Lowercase letters, digits and hyphens only (mixed case is
  lowercased for you, anything else is a `400`), 100 characters. The API schema marks it required on
  create but **the server does not enforce that**: omit it and the server slugifies `name`; omit both
  and you get a row with an empty slug that nothing can ever select. A duplicate returns `409`
  `a channel type with the same name already exists in this organization` — the slug collided, and no
  organization is involved.
- **`name`** / **`description`** — display only. The runtime never reads either.
- **`metadata`** — a **closed struct of exactly four keys**. Anything else you put in it is dropped
  silently on write. A `PUT` that provides `metadata` **replaces the whole object**, so a body
  carrying only `skills` wipes `moduleRefs`, `mcpServers` and `systemPromptFragment`. Always send all
  four keys. Omit the key entirely and the stored object is left untouched. Send `changeSequence`
  too if you want the update to fail rather than clobber a concurrent one: a mismatch returns `409`
  `stale changeSequence: expected N but resource is at M; re-fetch and retry`.

## `mcpServers` — strict opt-in, and empty means nothing

A server mounts only if this list names it. **An empty or absent list mounts nothing** — the chat is
left with the agent SDK's own read-only file tools and can touch no part of the platform. Empty does
not mean "mount everything": a row with an empty list is a broken row, not a permissive one.

| Name | Mounts |
|---|---|
| `kodexa-document` | document reads, plus the edit tools when `skills` includes `document-edit` |
| `kodexa-platform` | the configuration tools — **all** of them skill-gated, see below |
| `kodexa-ui` | the workspace-driving and ask-the-user tools; needs a channel id, so it never mounts for a chat that has none |
| `kodexa-module-runner` | running module actions; also requires the runtime's callback listener |
| `kodexa-report` / `kodexa-bridge` | single-purpose write / external-call grants. No shipped row lists them — headless plan-step runtimes grant them through the image instead |

An unrecognised name is simply never matched — no error, no warning. Every mounted server is also
added to the agent's tool allow-list, and a guard denies anything outside it, so this list is the
real boundary rather than a hint.

## `skills` — it does two things, and one of them is invisible

A name in `skills` (a) copies that built-in skill pack into the agent's working directory so the
model can read it, and (b) **unlocks the tools gated behind that name**. Consequences:

- **`kodexa-platform` with no matching skill name mounts an empty server.** Every tool on it is
  gated. Listing the server and forgetting the skills produces a chat that looks configured and can
  do nothing.
- **`kodexa-document` is the other way round**: its read tools are always on once the server is
  mounted; only `document-edit` and `knowledge-loop` gate anything.
- `kodexa-ui` is added automatically whenever that server mounts; listing it is redundant.
- **Naming a skill without mounting its server is inert** — the skill file sits in the working
  directory advertising tools that are not there.
- An unknown name is logged and skipped.

Accepted names: `activity-execution`, `activity-plan`, `data-definition`, `data-form`,
`document-edit`, `document-read`, `filing-analysis`, `knowledge-loop`, `kodexa-ui`, `organization`,
`project`, `service-bridge`, `task-status`, `task-template`, `trigger`. Which server each one needs
is in `references/reference.md`.

## `systemPromptFragment`

Plain markdown, appended **after** the platform's own per-turn context and before any module-supplied
addendum, so it wins on recency — which is how the shipped admin rows steer the agent away from
document work they mount no tools for. Use it for role, tone and prohibitions, not for facts.

## `moduleRefs`

Module skill packs, downloaded into the session's working directory on its first turn.

- Form is `orgSlug/moduleSlug`; a leading `module://`, `model://` or `model-runtime://` is stripped.
- **`{org}`** (single braces, no `$`) is substituted with the slug of the organization the chat is
  running in — the way to write a row that suits every org in the database. A ref containing `{org}`
  when no organization slug reaches the runtime is dropped.
- **A `:version` suffix is parsed and then discarded.** Resolution is unversioned; you always get the
  current module.
- The on-disk cache is keyed by module slug alone, so two modules with the same slug in different
  organizations collide — the second is skipped with a warning rather than overwriting the first.
- A ref that does not resolve, or whose download fails, is logged and skipped. The chat still runs.

## Binding a chat, and why it is permanent

A chat carries `channelTypeId`. Set it at create (`POST /api/channels`), as the flat `channelTypeId`
or as `channelType: {"id": "..."}` — **the slug is not accepted as a reference**, resolve it first. A
bad id is not checked up front: it reaches the insert, trips the foreign key and returns a `500`, so
read a server error on a channel create as a bad `channelTypeId` first.

It is **bind-once**. Update rejects a change:

```
channelTypeId is immutable: cannot change from "<old>" to "<new>" once the channel has been created.
```

and rejects an explicit clear:

```
channelTypeId is immutable: cannot clear once the channel has been bound to a type.
```

Omitting the key entirely is a no-op, and setting a type on a chat that never had one is allowed —
that is the upgrade path for older chats. Deleting a channel type does not delete its chats: their
pointer is cleared, and they drop to the no-tools fallback.

## When an edit takes effect

Both caches are **session-scoped**: the resolved row is cached per agent session by slug, and module
packs are extracted into that session's working directory and reused while it is non-empty. A session
expires on an idle timeout — just under an hour by default — and its working directory goes with it.
So an edit, to the row or to a module you republished, lands on the next session; a chat someone is
mid-conversation in keeps the old configuration until it goes idle.

## Declared but inert

| Thing | Reality |
|---|---|
| `:version` on a `moduleRefs` entry | Parsed and thrown away. Nothing pins a module version here. |
| `kodexa-document-workflow` in `skills` | Present in the shipped rows; no skill pack of that name exists. Ignored — harmless, and deliberately left in place. |
| Extra keys inside `metadata` | Dropped on write. Only the four documented keys survive a round trip. |
| `name`, `description` | Never read by the runtime. Pure documentation for whoever opens the table next. |
| `slug` declared required (and min-length) on the create schema | Nothing enforces it. A body with only `name` derives the slug; a body with neither stores an empty one. |

## Common mistakes

| Mistake | What happens / fix |
|---|---|
| `PUT` with a partial `metadata` object | The other three keys are wiped. Send all four, every time. |
| `PUT` against `project_admin`, `org_admin` or `workspace_global` | `400` `invalid slug "project_admin": slugs must contain only lowercase letters, numbers, and hyphens` — even when you omit `slug`, because the stored value is validated after being backfilled. Those rows can only be changed in the database. |
| Creating a row with an underscore slug | Same `400`. Hyphens only through the API. |
| `mcpServers: []` (or omitted) | Zero servers. The chat can read files and nothing else. |
| `mcpServers: ["kodexa-platform"]` with no `skills` | The server mounts with no tools at all. Add the skill names for the tools you want. |
| Listing `document-edit` without `kodexa-document` | Nothing mounts; the skill file just advertises absent tools. |
| Inventing a slug like `invoice-chat` | The UI never asks for it. Edit one of the five known slugs, or bind by id from your own caller. |
| `${org}/my-module` in `moduleRefs` | Not substituted — the placeholder is `{org}`. |
| Expecting `kdx` to sync it | It has no `channel-type` type. Use the API or the database. |
| Editing a row and testing in an open chat | The old config is cached for the session's life. Start a new chat. |
| Rebinding a chat to a different type | Refused; the binding is permanent. Start a new chat. |

## Related skills

- **module** — what a module skill pack is, and the ref form `moduleRefs` expects.
- **project-resource** — the org-scoped resources those skills act on still need binding to a project.
- **activity-plan**, **task-template**, **data-definition**, **data-form**, **service-bridge**,
  **trigger**, **task-status** — the resources the like-named built-in skills read and edit.
