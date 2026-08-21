# The envelope, field by field

Every key below belongs to the shared envelope carried by all twelve types (individual notes call
out the handful that behave differently on one type): `data-definition`, `data-form`,
`document-store`, `data-store`, `module`, `prompt-template`, `service-bridge`, `activity-plan`,
`project-template`, `task-template`, `task-status`, `knowledge-set` (and the platform-supplied
`model-runtime`).

## Identity

| Key | Type | Writable | Notes |
|---|---|---|---|
| `id` | string (UUID) | create only, and normally server-minted | Never writable through an update. Stripped from pulled files. |
| `uuid` | string | create only | A second identifier, minted on create when absent. Distinct per environment for the same slug, so it is pure noise in a repo; `kdx` strips it from every nested map on pull. |
| `slug` | string | yes | Identity within the organization. Format `^[a-z0-9-]+$`, lowercased on the way in. Unique per `(organization, slug)` among non-deleted rows. |
| `name` | string | yes | Display name. Required in practice: with no `slug` and no `name` a create fails *invalid metadata: name is required*. Also the source the server slugifies when `slug` is absent. |
| `orgSlug` | string | effectively no | Computed from the organization FK on read. Ignored on write for every envelope type except `knowledge-set`. `kdx apply` reads it locally to pick the target org; `kdx sync pull` strips it. |
| `organizationId` | string (UUID) | create only | The FK. On update it is overwritten from the stored row before validation runs. |
| `organization` | `{ "id": "…" }` | create only | The nested spelling. It **wins** over `organizationId` when both are supplied — deliberately, so a client that emits the org *slug* under `organizationId` alongside a correct nested id still lands right. Two exceptions: `module` and `model-runtime` shadow the nested field with a typed relation and have no bridging resolver, so on those the flat `organizationId` wins. |
| `ref` | string | no | Computed `orgSlug/slug`. Any `:version` suffix in stored data is stripped on the way out. Stripped from pulled files. |
| `uri` | string | no | Computed `<stored type>://orgSlug/slug` — minted by the flatten step, so absent on the four non-flattening types. Not stripped from pulled files, but excluded from push comparison. |

## Classification

| Key | Type | Notes |
|---|---|---|
| `type` | string | Resource-type discriminator on eleven of the twelve. Server-defaulted on create for eight (see the table in `SKILL.md`). On `knowledge-set` it is the set type, mirrored to and from `setType`. On `activity-plan` / `task-template` / `task-status` there is no default, so it stays empty unless authored. |
| `publicAccess` | bool | Default `false`. Inert — see **Inert** below. |
| `template` | bool | Default `false`. Read by the Studio resource picker: a `template: true` resource in the `kodexa` organization is offered to every organization, and choosing it copies the resource into yours under a name and slug you supply (identity and lifecycle keys stripped) rather than binding the original. Also the picker's primary sort. |
| `deprecated` | bool | Default `false`. Read by the same picker: deprecated resources are filtered out unless "include deprecated" is on, and carry a Deprecated badge when shown. Nothing else in the platform branches on it — no resolver or API refuses a deprecated resource. |
| `extensionPackRef` | string | Inert. |

## Lifecycle

| Key | Type | Notes |
|---|---|---|
| `createdOn`, `updatedOn` | timestamp | Server-managed; never client-writable through an update. `updatedOn` is stamped explicitly on every update. Both are stripped from every nested map on pull. |
| `changeSequence` | int | Optimistic lock, incremented on every successful update. Enforced only when the request body supplies it (an explicit `null` opts out; an explicit non-null `0` does not). Mismatch is a `409 stale changeSequence: expected N but resource is at M; re-fetch and retry`. Stripped from every nested map on pull. |
| `deleted` | bool | Set by `DELETE`, never by a save — the four soft-delete fields are zeroed out of every create and update body before the write. |
| `deletedDate` | timestamp | Stamped by `DELETE`. |
| `deleteUserEmail`, `deleteUserId` | string | Stamped from the calling user by `DELETE`. |
| `yamlSource` | string | Inert. |
| `searchText` | — | Not on the wire at all. Server-computed from `name` + `slug` (types may override) and never client-writable. |

### What `DELETE` actually does

For every envelope type the delete is soft: `deleted` goes true, the audit fields are stamped, and
**`-<uuid>` is appended to the slug**. That last step is what frees the slug for immediate re-use —
and it adds 37 characters, so keep hand-written slugs comfortably short. Consequences:

- A soft-deleted resource is no longer reachable by its old slug, through the resolver or otherwise.
- Re-creating a resource with the same slug straight after a delete works.
- Lists and reads filter deleted rows out, so a soft-deleted resource simply looks gone.

## Common metadata-blob fields

One shared block — `description`, `icon`, `imageUrl`, `overviewMarkdown`, `provider`, `providerUrl`,
`providerImageUrl`, `deleteProtection`, `checksum` — is embedded in the metadata blob of
`data-definition`, `data-form` and `service-bridge`, and `project-template` declares its own copies
of the same names. Because those four flatten, the block appears **flat** on the wire for them.

It is not an envelope column and it is not universal. `module`, `document-store` and `data-store`
blobs do **not** carry it: a module's `description` is a column of its own, and the store content
objects carry only `type`, `indexed`, `storeType`, `storePurpose` and `deleteProtection`. Of the
non-flattening four, `activity-plan` and `task-status` have a `description` column and a free-form
`metadata:` blob nothing schematizes; `task-template` has a typed metadata block plus a top-level
`description`; `knowledge-set` has a `description` column and no metadata object at all. Check the
per-type skill rather than assuming a field exists.

## Which types flatten, and what they emit

| Resource | `metadata` flattened on the wire | Emits `uri` | Server default for `type` |
|---|---|---|---|
| `data-definition` | yes | yes | `taxonomy` |
| `data-form` | yes | yes | `data-form` |
| `document-store` | yes (two levels) | yes | `document-store` |
| `data-store` | yes (two levels) | yes | `data-store` |
| `module` | yes | yes | `module` |
| `prompt-template` | yes | yes | `prompt` |
| `service-bridge` | yes | yes | `service-bridge` |
| `project-template` | yes | yes | `project-template` |
| `activity-plan` | **no** | **no** | none |
| `task-template` | **no** | **no** | none |
| `task-status` | **no** | **no** | none |
| `knowledge-set` | **n/a — no metadata object** | **no** | none (`type` is the set type) |

The two-level cases emit their inner content object twice: flattened into the top level *and*
intact under a `contentMetadata` key, so a round-trip PUT can restore it. Nothing else uses that
key; do not author it by hand.

`knowledge-set` is the odd one out — it has no nested metadata blob at all, so every field it
carries is a column in its own right and the update rules below do not apply to it. Note that the
"no" column means only that the `metadata` key is *not* flattened away: `activity-plan` and
`task-status` still carry their own real fields (`steps`, `inputOptions`, `statusType`, `locked`, …)
as top-level keys, because those are columns rather than metadata. Only `task-template` keeps its
substantive body inside `metadata:`.

## Update semantics on a flattened type

The update path writes only the columns the request body actually mentioned, which produces two
rules worth knowing:

- A body of envelope keys only (`{"name": "New name"}`) leaves the metadata blob **untouched**.
- A body carrying **any** key that is not an envelope column marks the whole metadata column as
  written, and it is written from the decoded body — so every metadata field the body omitted
  reverts to its zero value. There is no field-level merge. Send the complete resource.

A round-trip body always carries `ref` and `uri`, which are unresolvable keys and do satisfy the
first condition. They cannot trigger a rewrite on their own because a second condition applies: the
decoded metadata struct must be non-zero. A body of envelope keys plus `ref`/`uri` decodes to an
empty metadata struct and is left alone.

## Inert

Fields that persist, round-trip and look meaningful, but that nothing in the platform reads.
(`template` and `deprecated` look like members of this list and are **not** — see **Classification**
above.)

- **`publicAccess`** — the unauthenticated-read check tests for a capability that no model
  implements, so the assertion never succeeds and the flag never grants access to anything. It is
  stored, returned and sortable, which makes it look live.
- **`extensionPackRef`** — a column and a JSON field with neither a producer nor a consumer.
  `kdx sync pull` strips it from files.
- **`yamlSource`** — intended as a round-trip snapshot of the authored YAML text. Nothing on the
  server writes it, and `kdx` reads it off a pulled response, carries it as far as the file writer
  and then drops it: pulled files never contain it. Treat it as a dead column.
- **`uri` on `activity-plan` / `task-template` / `task-status` / `knowledge-set`** — declared in
  the OpenAPI document (so generated clients expose the field) and never emitted, because the
  serializer that mints it does not run for those types.
