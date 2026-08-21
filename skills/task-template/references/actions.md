# Actions

`metadata.actions[]` declares the buttons in a task's toolbar.

```yaml
metadata:
  actions:
    - slug: approve
      label: "Approve"
      properties:
        statusSlug: approved
        icon: check
        color: green
        shortcut: "a"
        gatedByCompleteness: true
        attributes:
          - taxon: { taxonomySlug: invoice-taxonomy, taxonPath: invoice/approval_status }
            valueType: string
            value: approved
          - taxon: { taxonomySlug: invoice-taxonomy, taxonPath: invoice/reviewed_by }
            valueType: metadata
            metadataKey: currentUserEmail
    - slug: reject
      label: "Reject"
      properties:
        statusSlug: rejected
        icon: close
        color: red
        requireComment: true
        commentPrompt: "Why is this invoice being rejected?"
```

The declared shape is `slug`, `uuid` (deprecated), `type`, `label`, `properties`. `type` is persisted
but no task path reads it; the Studio editor does not even write it. Everything that matters is
`slug`, `label` and `properties`.

## Identity

`slug` is the action's identity. It is:

- the qualifier an activity plan's `dependsOn: "<stepSlug>:<actionSlug>"` edge points at, and
- the completion token recorded on the owning step when the task is finished with that button.

`uuid` is the deprecated legacy spelling. When an action declares `uuid` but no `slug`, the uuid value
*becomes* the slug, so old templates keep working — and the completion token is `uuid || slug`, so the
two spellings share one identity space.

Rules that bite:

- **Never set `uuid` and `slug` to different values on one action.** The platform refuses with
  *"conflicting identities (uuid … vs slug …) — uuid is the legacy spelling of slug; keep the slug"*.
  This is raised while resolving any action-qualified dependency onto a `CREATE_TASK` step that
  references this template, so a single bad action breaks every such activity start and fails plan
  validation too.
- **Never give two actions the same slug.** A dependency naming it fails as *ambiguous*.
- **To rename a button, change `label` only.** Changing the slug orphans every plan edge pointing at it.
- **Author `slug`, not `uuid`, on new actions.** Studio derives the slug from the label (kebab-cased,
  uniqued against siblings) and persists slug only.

Tasks carry no action data of their own: buttons are hydrated from the template on every read. Editing
`metadata.actions` therefore changes the toolbar of every open task immediately.

## The activity-plan contract

A plan step depends on a specific button like this:

```yaml
# in the activity plan
- slug: approve-step
  type: EXECUTION
  dependsOn: ["review:approve"]     # <stepSlug>:<actionSlug>
```

`review` must be a `CREATE_TASK` step whose `taskTemplateRef` names this template, and `approve` must
be a declared action slug (or the legacy uuid value). If it matches nothing, the activity start fails
with the list of live identities — *unless* the reference is RFC-4122 UUID-shaped and matches no
declaration anywhere, which passes through verbatim as a legacy escape hatch for actions the
resolver cannot see. Do not rely on that: a readable slug is checked, a UUID is not.

Two extras worth knowing:

- The org's **`DONE`-typed task-status slugs are also valid completion tokens** on a `CREATE_TASK`
  step, because finishing a task from the status dropdown records the status slug rather than an
  action token. So `dependsOn: "review:reviewed"` resolves if `reviewed` is a DONE status, even with
  no matching button.
- Only the referenced template's `metadata.actions` are live for a `CREATE_TASK` step. An inline
  `actions:` array on the step itself is a documentation mirror and never renders — a dependency on it
  fails the start with an explicit message.

## `properties` vocabulary

### Status transition

| Key | Type | Effect |
|---|---|---|
| `statusSlug` | string | The status the task moves to. **This is the one that works.** |
| `statusId` | string | Legacy fallback, read only when `statusSlug` is absent. |

`targetStatus` is read nowhere. An action with only `targetStatus` saves the task without moving it,
so no completion token is written and downstream steps stall.

### Locking

| Key | Type | Effect |
|---|---|---|
| `lockTask` | bool | Overrides the destination status's lock default for this transition. |
| `lockDocumentFamily` | bool | Overrides the status's document-family lock default — e.g. lock the task while leaving documents editable. |

Omit either key to keep the status default. The document-family default is **on**.

### Gating (when the button is disabled)

| Key | Type | Effect |
|---|---|---|
| `gatedByCompleteness` | bool | Disabled while the form-completeness gate has outstanding items; an info popover lists them. |
| `onlyEnabledIfNoOpenExceptions` | bool | Disabled while any open exception exists. |
| `onlyEnabledIfNoOpenExceptionsForPaths` | array of `{taxonomySlug, taxonPath}` | Scopes the exception gate to specific paths, so unrelated exceptions elsewhere in the form do not block it. Legacy bare path strings are accepted. |

Every action is also disabled while the task is locked, loading, or another action is running.

### Comment capture

| Key | Type | Effect |
|---|---|---|
| `requireComment` | bool | Prompts for a comment **before any mutation**. Cancelling aborts the action and leaves the task untouched. The comment is persisted as a `COMMENT` task activity tagged with the action token. |
| `commentPrompt` | string | Dialog text. Defaults to *Add a comment to record why you ran "&lt;label&gt;"*. |

### Data writes

```yaml
properties:
  attributes:
    - taxon: { taxonomySlug: invoice-taxonomy, taxonPath: invoice/approval_status }
      valueType: string
      value: approved
    - taxon: { taxonomySlug: invoice-taxonomy, taxonPath: invoice/reviewed_at }
      valueType: metadata
      metadataKey: currentTimestamp
    - taxon: { taxonomySlug: invoice-taxonomy, taxonPath: invoice/is_clean }
      valueType: boolean
      booleanValue: true
  takeOwnershipForPaths:
    - { taxonomySlug: invoice-taxonomy, taxonPath: invoice/total }
```

| Key | Type | Effect |
|---|---|---|
| `attributes` | array | Writes values onto the task's **extracted data attributes**, not onto task fields. `taxon` is `{taxonomySlug, taxonPath}` (a bare path string is the legacy form). `valueType` is `string` (use `value`), `boolean` (use `booleanValue`) or `metadata` (use `metadataKey`: `currentUserEmail` or `currentTimestamp`). A path with no matching taxon is skipped with a warning. |
| `takeOwnershipForPaths` | array of `{taxonomySlug, taxonPath}` | Stamps the reviewer as owner of the listed attributes even when their value did not change. Runs after `attributes`. |

`setAttributes` is not a key; it is read nowhere.

### Presentation

| Key | Type | Effect |
|---|---|---|
| `icon` | string | Material Design icon name. Defaults to `check`. |
| `color` | string | Set as the button's raw CSS `background`, so it must be a **CSS colour** — `green`, `#0f766e`, `rgb(15 118 110)`. Tailwind-style names like `amber` or `primary` are not CSS colours: they are ignored and the button keeps its default styling. Omit it to get the default. |
| `shortcut` | string | Keyboard shortcut, e.g. `"a"`, `"control+enter"`, `"meta+s"`. |
| `shortcutAltKey` | string or array | Alternate key(s). |

The Studio action editor's *Keybind* field writes `keybind` / `altKeybinds`, which the task toolbar
does not read. Author `shortcut` / `shortcutAltKey`. `automaticallyTakeNextTask` is likewise written
by the editor and read by nothing.

## Execution order

When a reviewer clicks an action:

1. If `requireComment`, prompt — cancelling aborts everything.
2. Apply `statusSlug` (or `statusId`) and any `lockTask` / `lockDocumentFamily` overrides.
3. Record the completion token (`uuid || slug`).
4. Apply `attributes`.
5. Apply `takeOwnershipForPaths`.
6. Save everything in one batch, then navigate away.

The token reaches the owning activity step on the save, which is what releases any
`"<stepSlug>:<actionSlug>"` dependency waiting on it.
