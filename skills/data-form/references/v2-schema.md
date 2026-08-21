# V2 node and record reference

## Top-level record

Canonical key order (what the platform normalises YAML to on pull, so writing it
this way keeps diffs quiet):

```
type, slug, name, description, version,
publicAccess, deprecated, template,
icon, imageUrl, overviewMarkdown,
provider, providerUrl, providerImageUrl,
version, editable, entrypoints,
nodes, cards, views, actions, options,
scripts, scriptModules, bridge, scriptTriggers, eventTriggers,
tabOrderInitial, tabOrder, tabOrderGroups
```

(`version` appears in both the common envelope and the data-form body; it is one
key, emitted once.) `shortcuts` is absent from the ordering list entirely, so it
sorts lexically after the ordered keys. It is a first-class persisted field
regardless — the placement is cosmetic.

| Field | What it does |
|---|---|
| `type` | `dataForm` in a resource file; stored as `data-form`. |
| `slug` | Identity within the org. Omit it and it is derived from `name` — usually not the slug you wanted, so set it. |
| `name`, `description` | Display. |
| `version` | The form's own version string. Also accepted as a V2 signal when exactly `"2"`. |
| `entrypoints` | `documentFamily` and/or `workspace`. See SKILL.md. |
| `nodes` | The V2 component tree. |
| `cards` | The V1 tree. Mutually exclusive with `nodes` in practice. |
| `scripts` | `{name: source}` — the only executable script map. |
| `bridge` | `{permissions?}`. `apiBaseUrl` and `maxExecutionMs` persist but are never read. |
| `scriptTriggers`, `eventTriggers`, `shortcuts`, `tabOrder*` | See `scripting.md`. |
| `editable`, `views`, `actions`, `options`, `scriptModules` | Persisted, no runtime reader. See the inert table in SKILL.md. |

## The node

| Field | Status | Notes |
|---|---|---|
| `component` | **required** | `v2:<type>`. |
| `props` | works | Static values, spread onto the component. |
| `bindings` | works | `{propName: "<JS expression over ctx>"}`. Evaluated **after** `props` are spread, so a binding overwrites a static prop of the same name. |
| `events` | works | See below. |
| `children` | works | Rendered into the component's default slot — but only five components have one. |
| `if` | works | JS expression over `ctx`. Falsey ⇒ the node is not mounted. |
| `key` | works | Sibling identity when children reorder. |
| `meta` | works | `{label, description, designOnly, category}` — design-time metadata; no runtime effect but preserved. |
| `show` | **inert** | Never read. Use `if`. |
| `for` | **inert** | The loop directive is not implemented; the node renders once. |
| `computed` | **inert** | Never read. |
| `slots` | **inert** for rendering | Walked once to collect bound paths for the completeness gate, never rendered. `v2:tabs` builds its strip from `children`, so `slots: {tab-1: [...]}` yields an empty tab strip. |
| `class`, `style`, `ref` | **inert** | Not merged into what the component receives. Use `props.class` (works on `v2:label` and `v2:divider`) or a `bindings.class`, which does fall through as an attribute. |
| `ifFormula`, `showFormula` | **not persisted** | The renderer honours them (KEXL formulas evaluated reactively in WebAssembly) but they are not in the stored model, and unknown keys are silently dropped on save. Do not author them in YAML. |

The `class` trap is worth restating: `bindings.class` works and node-level
`class` does not, so a form can demonstrate one working while the other quietly
does nothing.

### Conditions

`if` is a JavaScript expression compiled with `ctx` in scope. Anything that
throws evaluates to `undefined`, i.e. the node is not mounted — a typo in an
`if` hides the node rather than erroring.

```yaml
- component: v2:panel
  if: ctx.dataObjects?.some(o => o.path === 'invoice/line_items')
  props: { title: Line Items }
```

### Events

```yaml
events:
  click:
    type: scriptRef            # script | scriptRef | emit | store-action | bus-event
    target: recalcTotals       # script name for scriptRef, expression for script,
                               # event name for emit
    params: { field: total }   # passed to the script as ctx.params
    condition: ctx.dataObjects?.length > 0
```

- An event value may also be an **array** of these configs, executed in order.
- `type: script` evaluates `target` as an expression with **only `ctx` in
  scope** — no `bridge`. Anything that needs the bridge must be a named entry in
  `scripts`, invoked via `type: scriptRef`.
- `type: emit` raises `target` as a component event on the form root. Neither
  place that mounts a form binds a listener, so nothing receives it — treat
  `emit` as inert until something does.
- `store-action` and `bus-event` are reserved and do nothing.
- `debounce` on an event config is ignored. The only place debounce is read is
  `eventTriggers[]`.

## Data context (`ctx`)

Available in `bindings`, `if`, `events[].condition`, `type: script` targets, and
as the first argument to a script invoked by `type: scriptRef` or by a trigger.

**A `shortcuts` script is the exception.** Its `ctx` is `{shortcut: <the
shortcut entry>}` and nothing else — no `dataObjects`, no `formValues`, no
`tagMetadataMap`. A shortcut script reading `ctx.dataObjects` gets `undefined`
with no error. Shortcut scripts should work through `bridge`.

| Variable | Populated? | What it holds |
|---|---|---|
| `ctx.dataObjects` | yes | Data objects in scope. Scoped to the view's documents unless the form's `entrypoints` include `workspace`. A `v2:panel` with `groupTaxon` replaces this for its children with `[thisInstance, ...descendants]`. |
| `ctx.tagMetadataMap` | yes | `Map<tagPath, tagMetadata>` — taxon type, parent path, taxonomy ref. |
| `ctx.formValues` | yes | Shared string map for cross-field values. The only writer is a trigger script (mutate it, or return it); `v2:attributeEditor.valueFrom` reads it. A key containing `/` is also written through to the matching attribute. |
| `ctx.bridgeSelectionOptions` | yes | Bridge-computed dropdown options, keyed by tag path. |
| `ctx.$bridgeResult` / `$bridgeLoading` / `$bridgeError` | yes, inside `v2:serviceBridgeView` only | The endpoint response, its loading flag and its error. |
| `ctx.$item`, `$index`, `$parent`, `$root` | **never** | Declared in old docs and in the type; nothing assigns them. Always `undefined`. |
| `ctx.value` | **never** | There is no per-value formatter hook. A script reading `ctx.value` gets `undefined`. |

Inside a `scriptRef` event handler, `ctx` additionally carries `event` and the
handler's `params`. A `scriptTriggers` script gets `{dataObjects, formValues}`;
an `eventTriggers` script also gets `event`. On the WebAssembly path (the normal
one for both trigger kinds) `dataObjects` is stripped — see `scripting.md`.

## What survives a save

The stored model is a typed structure. Keys it does not declare are dropped
silently on write — no error, no warning, and the loss only shows up when you
pull the form back. Known casualties: `copyRules`, node `ifFormula` /
`showFormula`. If you add something not documented here, pull the form back and
confirm it survived before building on it.
