# V2 component reference

Twenty-six components are registered under the `v2:` namespace. Every entry in a
`nodes` tree must use one of them, spelled `v2:<type>`. An unregistered name
resolves to nothing and the node renders silently as empty.

Props below are taken from each component's declared prop list. A prop not
listed here does not exist — extra keys land on the DOM element or are dropped.

Only `v2:panel`, `v2:tabs`, `v2:row`, `v2:col` and `v2:serviceBridgeView`
declare child support. Children under anything else are discarded.

---

## Layout

### `v2:panel` — container, optionally repeating

| Prop | Type | Notes |
|---|---|---|
| `title` | string | Header text. The header only renders when a title (or another header element) is present, so `icon` alone is a no-op. |
| `icon` | string | Material Design icon name (`mdi-*`). |
| `iconColor` | string | Tailwind colour token (`blue`, `amber`, `emerald`) — renders the icon in a tinted tile. |
| `description` | string | One line under the title. |
| `collapsible` | boolean | Adds the expand/collapse chevron. |
| `colSpan` | number | Width when the panel sits inside a grid-ish parent. |
| `groupTaxon` | string | Repeat the panel's children once per data object at this taxonomy path. |
| `groupDisplay` | `tabs` \| `stacked` \| `lineItems` | Default `tabs`. |
| `groupTintBy` | string | tagPath whose conditional format supplies each instance's background tint. **Stacked mode only** — ignored in tabs. |
| `groupActions` | boolean | Add button in the header plus a remove button per instance. Works in `tabs` and `stacked`; ignored in `lineItems`. New entries are created under the group's **parent** object, so it works on an empty group. |
| `masterBy` | string | `lineItems` only: tagPath of the boolean-ish attribute marking the master instance. Without a match the panel falls back to tabs. |
| `masterLabel` | string | `lineItems` only: heading above the whole master instance. |
| `summaryFields` | array | `lineItems` only: `[{label, tagPath, format?: currency\|number, pageJump?}]` shown on each collapsed row. |
| `subExceptionLabel` | string | `lineItems` only: wording after the open-exception count on sub-rows. |
| `overrideCascade` | object | `lineItems` only: `{taxonPaths, instanceLabel?}` — overriding one of those exceptions on the master also clears the matching one on every sub-instance. One-directional. |
| `aiExtraction` | object | Renders an AI extract button in the header. |
| `mustExpand` | boolean | Completeness gate: starts collapsed and must be expanded once. Requires `collapsible: true`, otherwise warns and is ignored. |
| `mustScroll` | boolean | Completeness gate: the panel's bottom must scroll into view once. |

A grouped panel replaces `ctx.dataObjects` for its children with
`[thisInstance, ...its descendants]`. That scoping is why nested
`v2:attributeEditor` nodes resolve to the right row.

```yaml
- component: v2:panel
  props:
    title: Line Items
    groupTaxon: invoice/line_items
    groupDisplay: stacked
    groupActions: true
  children:
    - component: v2:attributeEditor
      props: { tagPath: invoice/line_items/description }
```

Grid vs stacked panel: a grid is right for many uniform short rows; a stacked
panel is right when each instance needs its own multi-field layout, exceptions
block or nested group.

### `v2:tabs`

`tabs`, `mustView`. It also declares `title` and `icon`, and renders neither —
a tab strip has no header of its own. Put a titled `v2:panel` around it if you
want a heading.

One tab per **child node** — the tab strip is built from the default slot, not
from `slots`. Each tab's label comes from the child's `props.title`, unless the
`tabs` array supplies one at the same index; entries there are either a plain
string or `{label, icon?}`. `mustView` is an array of zero-based indices the
reviewer must open before a completeness-gated task action will fire; the
first-rendered tab counts as already viewed.

### `v2:row` / `v2:col`

`v2:row` takes `gap` (pixels, default 16 — pass `0` to remove it; useful when a
row mixes very uneven spans and the gap maths would otherwise force a wrap).

`v2:col` takes `span` (out of 12) and `gap`. **Omit `span` and the column is
full width**, not an auto-split share — nothing supplies the sibling count the
auto-calculation would need. Always set `span` when you want columns side by
side.

### `v2:divider`

`class` only, and it is **added** to the default rule styling rather than
replacing it — the opposite of `v2:label` below.

### `v2:alert`

`type` (`info` \| `warning` \| `success` \| `error`), `text` (required),
`strong` (bold lead-in rendered before the text).

### `v2:label`

`text` (required), `class`. When `class` is set it **replaces** the default
typography classes rather than adding to them.

### `v2:markdown`

`content`, `size` (default `medium`). Empty content renders a "No content"
placeholder.

### `v2:masterLabel`

`label`, `masterBy`. A heading that renders only inside the master instance of a
`lineItems` panel — use it as a child of the panel to place the heading between
the master's exceptions and its fields.

---

## Data editing

### `v2:attributeEditor` — the workhorse

| Prop | Type | Notes |
|---|---|---|
| `tagPath` | string, **required** | Taxonomy path, e.g. `invoice/total_amount`. Not `taxon`. |
| `label` | string | Defaults to the last `/` segment of `tagPath`. |
| `colSpan` | number | |
| `readonly` | boolean | |
| `valueFrom` | string | Display another field's value; makes this editor read-only. Reads the shared `ctx.formValues` map first, then falls back to the attribute at that path. |
| `editorOptions` | object | Passed to the underlying attribute editor — see below. |
| `dataObject` | object | Explicit row binding. Only hosts that already know the row (e.g. `v2:routeTimeline`) pass this. |

The editor finds its row by filtering `ctx.dataObjects` to the taxon's parent
path, preferring an object that already carries the attribute, else the first
one. That is why `groupTaxon` scoping on the enclosing panel matters.

Useful `editorOptions`:

| Option | Notes |
|---|---|
| `allowDirectExtract` | Copy-from-selection icon. **Defaults to `true`** on `v2:attributeEditor`. Set `false` to suppress it. |
| `aiExtraction` | `{prompt, modelType, targetPaths[]}` — adds the AI sparkle button. **May be combined with `allowDirectExtract`**; both icons render side by side. |
| `displayAsRadio` | Render a SELECTION taxon as radios instead of a dropdown. |
| `displayAs: explanation` | Render a STRING attribute as a read-only AI explanation callout. |
| `isCheckbox` / `onCheckValue` | Checkbox presentation and the value written when checked. |
| `isMaskedText` / `maskedText` | Masked display. |
| `showCalendarPopup`, `maskDateFormat`, `sourceDateFormat` | Date presentation. |
| `hideAttributeMenu`, `hideExceptionPopup`, `hideIndicatorBars`, `hideInsertActions` | Chrome suppression. |
| `placeholder` | Auto-filled when `aiExtraction` or `allowDirectExtract` is on and no placeholder is given. |

### `v2:grid`

| Prop | Notes |
|---|---|
| `groupTaxon` | **Required.** Rows are the data objects at this path. |
| `title` | Heading at the left of the grid toolbar. |
| `limitColumns` | Narrow the auto-derived taxon columns to this list. Entries are **full tag paths** (`invoice/line_items/quantity`), not bare taxon names — a short name matches nothing and silently removes the column. Columns otherwise come from the enabled non-group children of `groupTaxon` in the taxonomy. |
| `columns` | **Custom cell** columns, not taxon columns — see below. |
| `sort` | `[{tagPath \| header, direction: asc\|desc}]`. First entry is primary. `tagPath` matches a taxon column by its **full path**; `header` matches a custom column by its header text, case-insensitively. Entries matching nothing are skipped silently. |
| `height` | Pixel height. |
| `autoHeight` | Size to rows instead of reserving a viewport. Supersedes `height` — set one, not both. Every row renders (no virtualisation), so prefer a bounded `height` for grids that grow. |
| `pagination` | Default `false` on `v2:grid`. |
| `fitColumns` | Columns flex to fill the pane instead of a fixed default width. |
| `groupable`, `showColumnMenu` | ag-grid affordances. |
| `aiExtraction` | AI extract button in the toolbar. |
| `editorOptions` | Same options object as `v2:attributeEditor`, applied to every cell. |
| `movement` | Per-row "move to a different parent" buttons — see below. |

`columns` entries are `{header, component, props?, width?, position?, pinned?,
sortable?}`. **There is no `field` key**, and `component` names a v2 type
**without** the `v2:` prefix. An unresolvable name logs a warning and renders a
dash. `position` is `start` or `end` (default `end`); `width` defaults to 220.

```yaml
- component: v2:grid
  props:
    groupTaxon: invoice/line_items
    limitColumns:
      - invoice/line_items/description
      - invoice/line_items/quantity
      - invoice/line_items/unit_price
      - invoice/line_items/line_total
    autoHeight: true
    sort:
      - { tagPath: invoice/line_items/line_total, direction: desc }
    columns:
      - header: Source
        component: attributeSourceBadge
        props: { tagPath: invoice/line_items/description }
```

`movement` takes the same value shape as V1's `taxonMovement`, but **the key is
different** — V1 spells it `taxonMovement`, V2 spells it `movement` — so a V1
block copied verbatim is silently ignored. Also: `sourceTaxon` must equal the
grid's `groupTaxon` exactly, and the rule belongs on the grid that *displays*
the rows being moved. A mismatch produces no button and no warning.

### `v2:table`

`tagPathPrefix` (declared required, **never read**) and
`columns: [{tagPath, label, width?}]`. Rows are resolved from the parent path of
the *first* column's taxonomy metadata. A lighter alternative to `v2:grid` for
short, fixed column sets.

### `v2:dataTable`

`rows`, `columns`, `emptyMessage`, `filterable`, `pageSize`. Renders arbitrary
row objects — usually bound from a service-bridge result — rather than data
objects. `columns` here is `[{field, title?, width?, type?: string|boolean}]`;
omit it and columns are inferred from the first row. Setting `pageSize` enables
pagination.

### `v2:taxonNav`

`taxon` (path of the instances to enumerate, one chip each), `labelFrom` (path
of the attribute supplying the chip text and the scroll target), `tintBy`
(defaults to `labelFrom`), `emptyLabel` (default `"Unnamed"`). It also declares
`colSpan`, which it never applies — unlike `v2:panel` and `v2:attributeEditor`,
where `colSpan` really does emit `grid-column: span N`.

Chip colours come from the taxonomy's conditional formats, not from the form, so
a chip always agrees with the tint on the field itself.

### `v2:autoNavigateToggle`

`defaultEnabled`, `hidden`. `hidden: true` registers and enables
focus-to-navigate without rendering the toggle row.

### `v2:routeTimeline`

A specialised ordered-stops timeline. `groupTaxon` (required), `sequenceTag`
(default `stopSequence`), `typeTag` (default `stopType`), `addressGroupTaxon`
(default `<groupTaxon>/address`), `allowAdd`, `aiExtraction`, `viewId`,
`expandable`, `stopDetailTags`, `addressDetailTags`, `readonly`, `orientation`
(`vertical` \| `horizontal`), `show` (`all` \| `firstLast`).

---

## Exceptions and knowledge

### `v2:exceptions`

`title`, `filterPaths`, `sticky`. It also declares `showResolved`, which
**does nothing** — see below.

Aggregates exceptions from both the data object *and* its attributes, so an
exception that migrated onto an attribute when a reviewer touched the field
still surfaces. The aggregator keeps an exception when it is still open **or**
its closing comment is a custom one (a reviewer override), and drops the rest —
the platform's own auto-close comments. The component then applies that exact
same test again when `showResolved` is false, so the flag cannot add or remove
a single row either way. Auto-resolved exceptions are unreachable from a form;
custom-closed ones always show.

`filterPaths` narrows to specific tag paths. `sticky: true` floats the list at
the top-right of the form pane once the inline block scrolls out of view and
open exceptions remain.

### `v2:knowledgeSection` / `v2:knowledgeTaxonReadonly`

Both take `knowledgeItemType`. `knowledgeSection` renders the highest-priority
knowledge item of that type as an expandable instruction panel with its
attachments. `knowledgeTaxonReadonly` renders the read-only taxon values that
item declares as its dependencies.

---

## Grid cell components

These read a `params` object injected by `v2:grid`'s cell renderer, so they are
only usable as `columns[].component` inside a `v2:grid` — named without the
`v2:` prefix there. `params` is supplied automatically; do not author it.

| Component | Props (besides `params`) |
|---|---|
| `attributeSourceBadge` | `tagPath` — the row attribute whose spatial anchor identifies the source document |
| `attributeCopyButton` | `sourceTagPath`, `targetTagPath`, `tooltip?`, `relatedCopies?: [{sourceTagPath, targetTagPath}]` |
| `attributeRowPromote` | `sourceTagPath`, `targets: [{label, targetTagPath, sourceTagPath?, relatedCopies?}]`, `label?` |
| `attributeRowDeleteButton` | `tooltip?` |

`v2:attributeCopyAction` is the same copy behaviour **without** a `params`
dependency (`sourceTagPath`, `targetTagPath`, `tooltip?`, `relatedCopies?`), so
it can sit anywhere in the tree.

`v2:gridDeleteBySource` is a toolbar companion, not a cell: `groupTaxon` (match
the sibling grid) and `tagPath` (match what the grid's `attributeSourceBadge`
column uses, so badge and toolbar agree). It offers one delete button per
resolved source document.

---

## External data

### `v2:serviceBridgeView`

`bridgeRef` (`service-bridge://acme-corp/vendor-lookup`, or `acme-corp/vendor-lookup`),
`endpoint`, `params`, `transform`.

Calls the endpoint whenever `params` changes and provides its children a scoped
context with `ctx.$bridgeResult`, `ctx.$bridgeLoading` and `ctx.$bridgeError`.
**A `params` value of `null`/`undefined` skips the call entirely** — bind it to
an expression that returns null until the inputs are ready, and the component
stays idle instead of firing a half-formed request. `transform` is a JSONata
expression applied to the response before it reaches `$bridgeResult`; a bad
expression warns and passes the raw response through.

```yaml
- component: v2:serviceBridgeView
  props:
    bridgeRef: service-bridge://acme-corp/vendor-lookup
    endpoint: vendor-names
  bindings:
    params: >-
      ctx.dataObjects?.find(o => o.path === 'invoice')?.attributes
        ?.find(a => a.path === 'invoice/vendor_id')?.resolvedValue
        ? { vendorId: ctx.dataObjects.find(o => o.path === 'invoice').attributes.find(a => a.path === 'invoice/vendor_id').resolvedValue }
        : null
  children:
    - component: v2:dataTable
      props:
        columns:
          - { field: label, title: Vendor }
        emptyMessage: No vendors found
        filterable: true
        pageSize: 10
      bindings:
        rows: ctx.$bridgeResult
```
