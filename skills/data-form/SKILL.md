---
name: data-form
description: "Use when creating or editing a Kodexa data form — the schema-driven review panel that shows and corrects extracted document data. Covers the V2 `nodes` tree and its `v2:*` components, legacy V1 `cards`, `tagPath` binding, `entrypoints`, form scripts and the bridge, keyboard shortcuts and declarative tab order."
---

# Kodexa data forms

A data form is an org-scoped resource describing the panel a reviewer works in
beside a document: which extracted fields appear, in what layout, which are
editable, and which exceptions surface. Author it as YAML in
`data-forms/<slug>.yaml` and apply it with `kdx sync push`.

## Which schema am I in?

Two renderers ship side by side. **The array you use picks the renderer:**

| Your YAML has | Renderer | Component names |
|---|---|---|
| `nodes: [...]` | V2 schema renderer | `v2:panel`, `v2:attributeEditor`, … |
| `cards: [...]` | V1 card renderer | `cardPanel`, `dataAttributeEditor`, … (bare, no prefix) |

Detection is `version == "2"` **OR** a non-empty `nodes` array. So:

- `version` is the form's own domain version string, not the schema selector.
  A form with `version: 1.0.0` and a `nodes` array renders as V2 — that is the
  common shape in the wild.
- **Setting `version: "2"` on a `cards` form blanks it.** Detection flips to V2,
  V2 reads `nodes` (empty), and the form renders nothing. No error.
- V1 forms get **no** `scripts`, `shortcuts`, `scriptTriggers`, `eventTriggers`
  or `tabOrder` — those are only handed to the V2 renderer. Adding them to a
  `cards` form is silently ignored.

Author new forms as V2. Edit existing V1 forms in place — see
`references/v1-legacy.md`.

## The record

```yaml
type: dataForm                 # resource discriminator; stored as "data-form"
slug: invoice-review           # identity; derived from `name` if you omit it
name: Invoice Review
description: Review extracted invoice data
version: "2"
publicAccess: false
deprecated: false
template: false
editable: true                 # persisted, but nothing reads it (see below)
entrypoints:
  - documentFamily
nodes:
  - component: v2:panel
    props:
      title: Invoice
      groupTaxon: invoice
    children:
      - component: v2:attributeEditor
        props:
          tagPath: invoice/invoice_number
          label: Invoice Number
```

`entrypoints` is the field authors most often leave off, and it changes where
the form is reachable and how much data it sees. Only two values are read:

| Value | Effect |
|---|---|
| `documentFamily` | Listed in the document viewer's "open form" menu. In a task, one form view per document, scoped to that document's data objects. |
| `workspace` | One form view spanning every document in the task; data objects are **not** scoped to a single document. |

Anything else (`task` appears in shipped forms) round-trips but no code branches
on it. **Always set at least one.** An empty `entrypoints` is omitted from the
API response entirely, and one surface reads the array without a null guard — so
such a form can break the document viewer's form menu for the whole project.

## Scope and shipping

Data forms are **org-scoped**: `kdxa_data_forms`, `/api/data-forms`, resolved as
`data-form://<orgSlug>/<slug>`. A saved form is invisible in a project until
bound there (manifest `projects.<slug>.linked['data-form']`, or Project Settings
→ Resources). A task template's `dataFormRef` is a plain `<orgSlug>/<slug>` ref.

## The V2 node

```yaml
- component: v2:attributeEditor      # required, always `v2:`-prefixed
  props:                             # static props, passed straight through
    tagPath: invoice/total_amount
    label: Total
  bindings:                          # JS expressions over `ctx`, evaluated per render
    readonly: ctx.dataObjects?.length > 1
  if: ctx.dataObjects?.length > 0    # JS expression; false ⇒ node not mounted
  key: total                         # v-for key when siblings reorder
  children: []                       # only meaningful on container components
```

Rules that fail silently when broken:

- **`tagPath`, not `taxon`.** `taxon` is the V1 property name. A `v2:*` node
  with `props.taxon` leaves `tagPath` undefined and the editor throws while
  computing its label — the node renders nothing.
- **The taxon at that path must carry an `id`.** An editor resolves through the
  taxon, so a taxonomy applied from YAML with no taxon ids binds nothing and
  every field reads `No data` — however correct the `tagPath` is. See
  **data-definition**.
- **Only five components accept `children`:** `v2:panel`, `v2:tabs`, `v2:row`,
  `v2:col`, `v2:serviceBridgeView`. Children nested under any other component
  are dropped without a warning.
- **`bindings` win over `props`** on a key collision. A binding whose expression
  throws yields `undefined` — it does *not* fall back to the static prop.
- **Never put a `card:`-prefixed component under `nodes`.** It resolves to a V1
  card, which needs a `card` object and `viewId` the V2 renderer never supplies.
- Repetition comes from components, not from a loop directive: `v2:panel` with
  `groupTaxon` repeats its children per data object; `v2:grid` / `v2:table`
  render one row each.

Node fields: `references/v2-schema.md`. All 26 components: `references/v2-components.md`.

## Scripts

Most forms have no `scripts` and no `bridge` block, and are better for it. When
you do need one, four facts matter more than the API surface:

- **Form scripts are plain browser JavaScript.** They are compiled with
  `new Function` on the main thread. There is no sandbox, no isolation and no
  timeout. Trigger scripts additionally get a WebAssembly path with a hard 2s
  interrupt. Treat form scripts as trusted code.
- **The bridge is a function parameter, not a global.** Write
  `(ctx, bridge) => bridge.navigation.rotatePage("right")`. There is no
  `kodexa.*` object anywhere; referencing one throws, and the runner swallows
  the throw and returns `undefined` — a silent no-op.
- **Omitting `bridge.permissions` grants everything.** Permissions are a
  narrowing list, not an opt-in. Declaring a list that omits a capability makes
  that call throw, which is likewise swallowed mid-script.
- **Trigger scripts must be `function (ctx, bridge) { … }`, never arrows.**
  `scriptTriggers` / `eventTriggers` cross into a WebAssembly VM through a
  transform that only recognises a `function` expression; an arrow is discarded
  and the trigger does nothing. Shortcut and event scripts accept either form.

Everything else — the real bridge surface, `shortcuts`, `scriptTriggers`,
`eventTriggers`, tab order — is in `references/scripting.md`.

## Declared but inert

These persist, round-trip, and appear in existing forms and in the UI model, but
**nothing in the platform reads them**. Preserve them when editing a form you
did not write; do not author new ones.

| Field | Note |
|---|---|
| `editable` (form level) | No reader anywhere. Read-only fields are set per node via `props.readonly`. |
| `views[]`, `actions[]` | No renderer behaviour. `views` is stored untouched; `actions` gets a name check only. |
| `options[]` | Persisted and name-validated; no renderer behaviour. |
| `scriptModules` | Only `scripts` is executable. A `scriptRef` naming a module fails validation as an unknown script. |
| `bridge.maxExecutionMs`, `bridge.apiBaseUrl` | Never read. No script timeout is configurable; `apiBaseUrl` is left from a removed HTTP helper. |
| node `for`, `ctx.$item`/`$index`/`$parent`/`$root` | The loop directive is not implemented; the node renders once and those variables are always `undefined`. |
| node `show` | Not read. CSS visibility comes from `showFormula`. Use `if`. |
| node `computed`, `slots` | Not read; `slots` is walked once for path collection but never rendered. |
| node `class`, `style`, `ref` | Not applied. Set them through `props`/`bindings` instead (`props.class` works on `v2:label` and `v2:divider`). |
| `debounce` on node events and on `scriptTriggers[]` | Read only on `eventTriggers[]` (default 300ms). Everywhere else it is stored and ignored. |
| event `type: emit`, `type: store-action`, `type: bus-event` | `store-action` and `bus-event` are empty cases. `emit` raises a component event that no mount site listens for. |
| Component props `v2:table.tagPathPrefix`, `v2:tabs.title`/`.icon`, `v2:exceptions.showResolved`, `v2:taxonNav.colSpan` | Declared and accepted, never acted on. See `references/v2-components.md`. |
| `copyRules` (form level), node `ifFormula`/`showFormula` | Not in the stored model. The API drops unknown keys silently, so these vanish on save. |

## Server-side validation

Four structural rules run on create and update. Depending on configuration they
are disabled, logged, or returned as a 400:

- `data-form.option.name-required` — every `options[]` entry needs a `name`.
- `data-form.option.name-unique` — `options[].name` must be unique.
- `data-form.action.name-required` — every `actions[]` entry needs a `name`.
- `data-form.script-ref-unknown` — **the one you will hit**: when the form
  declares any inline `scripts`, every `scriptTriggers[].script`,
  `eventTriggers[].script` and `shortcuts[].scriptRef` must name a key in
  `scripts`. Skipped entirely when `scripts` is absent or empty.

## Common mistakes

| Mistake | What happens |
|---|---|
| **Every field renders `No data`** while the data export is full | Not a form problem. The taxons have no `id`, and a form cannot bind an id-less taxon — see **data-definition**, `references/schema.md`, "Taxon `id`". The chrome mounting normally is what makes this read as a form bug. |
| `card:*` components under `nodes` | Resolves to a V1 card with no `card`/`viewId` → broken or empty render |
| `props.taxon` on a `v2:*` node | `tagPath` undefined → editor throws, node disappears |
| `kodexa.data...` inside a script | `ReferenceError`, swallowed → silent no-op. Use the `bridge` parameter |
| Calling the bridge from an inline `type: script` handler | Only `ctx` is in scope there. Use a named script via `type: scriptRef` |
| Assuming an omitted `bridge.permissions` locks things down | It grants every capability |
| No `entrypoints` | Saves fine, then appears on no surface — and can break the viewer's form menu outright |
| Both `tabOrder` and `tabOrderGroups` | Mutually exclusive; `tabOrderGroups` wins and `tabOrder` is discarded |
| `version: "2"` on a `cards` form | Renders blank |
| Reading `ctx.dataObjects` in a `shortcuts` script | A shortcut's `ctx` is only `{shortcut}` — reach data through `bridge` |

## References

- `references/v2-components.md` — all 26 `v2:*` components with verified props
- `references/v2-schema.md` — node fields, data context, precedence, conditions
- `references/scripting.md` — runtimes, bridge surface, triggers, shortcuts, tab order
- `references/v1-legacy.md` — the `cards` schema and the 22 card types
- `references/examples.md` — complete, applyable form YAML

## Related skills

`data-definition` owns the taxon paths `tagPath` / `groupTaxon` must match (and
the conditional formats that colour panels and nav chips). `service-bridge` owns
`v2:serviceBridgeView.bridgeRef`. `task-template` names forms in
`forms[].dataFormRef`. `project-template` carries forms inline under `dataForms:`.
