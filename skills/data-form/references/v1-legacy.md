# V1 forms (`cards`)

The V1 card renderer is legacy but still carries a lot of production traffic.
Edit V1 forms in place; author new forms as V2. Migrating a V1 form is a rewrite
of the tree, not a rename — see the mapping table at the end.

## The card envelope

```yaml
cards:
  - id: header-panel                # required — cards are looked up by id
    type: cardPanel                 # bare type name; NO `card:` or `v2:` prefix
    properties:
      title: Invoice
      groupTaxon: invoice
      row: 0
      colSpan: 12
    children:
      - id: invoice-number
        type: dataAttributeEditor
        properties:
          attributeLabel: Invoice Number
          taxon: invoice/invoice_number
          row: 0
          col: 0
          colSpan: 6
```

Differences from V2 that bite when copying between the two:

| V1 | V2 |
|---|---|
| `type: dataAttributeEditor` | `component: v2:attributeEditor` |
| `properties:` | `props:` |
| `taxon:` | `tagPath:` |
| `attributeLabel:` | `label:` |
| `taxonMovement:` | `movement:` |
| `id` required on every card | `key` optional |

## Layout

V1 has no row/col components. Layout comes from each card's own properties, and
the renderer buckets siblings into rows:

| Property | Default | Meaning |
|---|---|---|
| `row` | `0` | Row bucket within the parent. |
| `col` | `0` | Column hint. |
| `colSpan` | `2` | Grid columns consumed. |
| `rowSpan` | `2` | Grid rows consumed. |
| `order` | `0` | Sort within the row. |

## The 22 card types

`cardGroup` · `cardPanel` · `comingSoon` · `dataAttributeEditor` ·
`dataObjectsTree` · `dataStoreGrid` · `excelExportConfig` · `exceptions` ·
`grid` · **`horizonalLine`** · `label` · `selectionRoster` · `singleTaxon` ·
`tabs` · `taxonGrid` · `taxonTabs` · `transposedGrid` ·
`transposedGridAggregatedCell` · `transposedGridDataPropertyRollup` ·
`transposedGridRollup` · `workspaceDataGrid` · `workspaceDataTreeView`

**`horizonalLine` is spelled without the second `t`.** The type name is derived
from the component filename, and the filename carries the typo. `horizontalLine`
matches no card: the renderer logs a browser-console warning and draws nothing,
which on the page is indistinguishable from a card you forgot to add.

Six card types declare child support: `cardPanel`, `cardGroup`, `tabs`,
`singleTaxon`, `taxonTabs` and `transposedGridDataPropertyRollup`.

Three further type names appear *inside* the transposed rollup grids —
`rollupColumn`, `rollupChildRow`, `rollupSpacer`. They are not registered card
types; they are nested entries parsed by the rollup grid from its own children.
Preserve them verbatim.

## Card properties that actually exist

Only these cards declare option metadata. Anything not listed either has no
declared options or is configured entirely from the taxonomy.

| Card | Properties |
|---|---|
| `label` | `label` (not `text`) |
| `singleTaxon` | `taxonId` (not `taxon`) |
| `dataStoreGrid` | `dataStoreRef` |
| `exceptions` | `exceptionMessage` |
| `cardGroup` | `showHeader`, `title`, `subtitle` |
| `tabs` | `tabsSticky` |
| `comingSoon` | `title`, `description`, `releaseDate`, `features` |
| `dataAttributeEditor` | `attributeLabel`, `taxon`, `editorOptions` (`hideAttributeMenu`, `isCheckbox`, `onCheckValue`, `isMaskedText`, `maskedText`, `maskDateFormat`, `sourceDateFormat`, `showCalendarPopup`) |
| `taxonGrid` | `showHeader`, `title`, `subtitle`, `groupTaxon`, `hideAdd`, `hideExceptions`, `taxonMovement`, `taxonCopy` |
| `cardPanel` | `showHeader`, `title`, `subtitle`, `groupTaxon`, `useTabs`, `hideAdd`, `hideExceptions`, `showChildExceptions`, `exceptionSticky`, `isJumpOn`, `overrideException`, `isEmptyAutoAdd`, `pageSize`, `maxHeight`, plus the `taxonMovement` / `taxonCopy` / `taxonNavigation` / `backgroundColorMapping` blocks |
| `selectionRoster` | `groupTaxon`, `selectTaxon`, `titleTaxon`, `metaTaxons`, `badgeTaxons`, `stateBadges`, `detailTaxons`, `requiredWhenSelected`, `prefilledReadonly`, `detailNoteWhen` |
| `transposedGridDataPropertyRollup` | `groupTaxon`, `groupByDataProperty`, `categoryTaxon`, `groupBy` |

`taxonMovement`, `taxonCopy`, `taxonNavigation` and `backgroundColorMapping` are
deep nested configuration blocks (rules, allowed destinations, attribute
mappings, colour mappings). **Preserve them verbatim** when editing a V1 form —
hand-rewriting them is the fastest way to break a working reviewer flow.

## What V1 forms do not get

`scripts`, `shortcuts`, `scriptTriggers`, `eventTriggers`, `tabOrder`,
`tabOrderGroups` and `tabOrderInitial` are only handed to the V2 renderer. Add
them to a `cards` form and they persist, round-trip, and do nothing.

Conversely, setting `version: "2"` on a `cards` form flips schema detection to
V2, which then renders an empty `nodes` array — the form goes blank with no
error.

## V1 → V2 mapping

| V1 card | V2 equivalent |
|---|---|
| `cardPanel` | `v2:panel` (`groupTaxon` + `groupDisplay: tabs` replaces `useTabs`; `exceptionSticky` becomes `v2:exceptions` `sticky: true`; `backgroundColorMapping` becomes `groupTintBy` reading the taxonomy's conditional formats; `taxonNavigation` becomes a sibling `v2:taxonNav`) |
| `cardGroup` | `v2:row` + `v2:col` |
| `label` | `v2:label` (`label` → `text`) |
| `horizonalLine` | `v2:divider` |
| `dataAttributeEditor` | `v2:attributeEditor` (`taxon` → `tagPath`, `attributeLabel` → `label`) |
| `taxonGrid` | `v2:grid` (`taxonMovement` → `movement`) |
| `tabs` | `v2:tabs` (one tab per child node) |
| `exceptions` | `v2:exceptions` |
| `singleTaxon` | `v2:attributeEditor` with `readonly: true`, or `v2:masterLabel` |
| `dataStoreGrid` | no direct equivalent |
| `dataObjectsTree`, `taxonTabs`, `workspaceDataTreeView` | no V2 equivalent — there is no tree component in V2 |
| `transposedGrid*` family | **no V2 equivalent.** Leave these forms on V1. |
