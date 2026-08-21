# Scripting, triggers, shortcuts and tab order

Most production forms have neither a `scripts` map nor a `bridge` block. Reach
for a script only when a declarative component genuinely cannot do the job.

## The two runtimes

There is no sandbox and no isolation. Two engines run form scripts:

| What runs it | Engine | Limits |
|---|---|---|
| `bindings`, `if`, `events[].condition`, `type: script` targets, `type: scriptRef` scripts, `shortcuts` | Browser `new Function`, on the main thread, with full page privileges | **None.** No timeout, no memory cap, no restricted globals. |
| `scriptTriggers` and `eventTriggers`, whenever the form's document is loaded in WebAssembly (the normal case) | A Go JavaScript VM compiled to WebAssembly, inside the document worker | Hard **2-second** interrupt and a **10-call** service-bridge budget, neither configurable. Falls back to browser `new Function` when no WebAssembly document reference is available. |

`bridge.maxExecutionMs` is parsed and stored but never read by either runner —
it changes nothing. Nothing in the platform runs form scripts under QuickJS.

## Script shape

A named script is a function expression. The runner wraps it and calls it:

```yaml
scripts:
  rotateRight: |
    (ctx, bridge) => bridge.navigation.rotatePage("right")
```

**The bridge is the second positional parameter.** There is no `kodexa` global —
`kodexa.data.setAttribute(...)` raises a `ReferenceError`, the runner catches it,
logs it and returns `undefined`. The script appears to do nothing.

Two shapes, and they are not interchangeable:

| Used by | Accepted source shapes |
|---|---|
| `shortcuts`, `type: scriptRef` events | Arrow `(ctx, bridge) => …` **or** `function (ctx, bridge) { … }` |
| `scriptTriggers`, `eventTriggers` | **`function (ctx, bridge, helpers) { … }` only** (or `async function`) |

Trigger scripts are transformed before crossing into WebAssembly: the transform
extracts the body of a `function (...) { ... }` expression. **An arrow function
does not match that pattern**, so the whole arrow is treated as the body,
evaluated as a dead expression statement, and the trigger silently does nothing.
Write triggers in `function` form.

The same transform also strips every `await` (the WebAssembly VM is synchronous)
and rewrites `bridge.serviceBridge.call(` to `serviceBridge.call(`.

A third argument, `helpers`, is supplied **only** on the browser-fallback path
for triggers. It carries `setSelectionOptions(tagPath, options)`, which populates
a field's dropdown. `shortcuts` and `scriptRef` events get two arguments only.

The `ctx` a script receives depends on who called it, and a `shortcuts` script
gets the thinnest one of all: **`{shortcut: <the entry that fired>}` and nothing
else** — no `dataObjects`, no `formValues`, no `tagMetadataMap`. Reading
`ctx.dataObjects` there yields `undefined` with no error. Work through `bridge`
in a shortcut; use `ctx` in `scriptRef` events and triggers, which do get the
form's data context.

### What a trigger script actually sees

When the WebAssembly path is taken (the normal case), the environment is not the
browser bridge:

- `ctx` arrives with **`dataObjects` stripped** — only `formValues` and `event`
  survive the crossing. A trigger reading `ctx.dataObjects` is a no-op in
  production and only "works" in the browser fallback.
- Globals are `loadDocument()`, `serviceBridge.call(bridgeRef, endpoint, body?)`,
  `log(level, message)`, `console.log` / `console.error`, and a **minimal**
  `bridge` carrying only `bridge.data.setAttribute(dataObjectId, path, value)`
  and `bridge.data.getAttribute(dataObjectId, path)` — numeric object ids, not
  UUIDs. Nothing else from the browser bridge exists there.
- The only channel back is `ctx.formValues`: mutate it, or return an object, and
  the result is merged into the form's shared values.

```yaml
scripts:
  recalcTotal: |
    function (ctx, bridge) {
      const sub = Number(ctx.formValues["invoice/subtotal"] || 0);
      const tax = Number(ctx.formValues["invoice/tax_amount"] || 0);
      ctx.formValues["invoice/total_amount"] = (sub + tax).toFixed(2);
      return ctx.formValues;
    }
```

## Permissions

```yaml
bridge:
  permissions: [data:read, data:write, navigation, viewer, formState]
```

Those five are the complete vocabulary. Anything else is an inert string that
narrows nothing — `http:get` and `http:post` still appear in older forms and
have never been consulted.

**Omitting `permissions` (or the whole `bridge` block) allows everything.** The
check only fires when the list is present. A present list that omits a
capability makes that call *throw*, and the runner swallows the throw, so the
script stops silently at that line with everything before it already applied.
The nastiest version of this: a list containing **only** inert strings passes no
real check, so every bridge call in every script throws and the form's scripting
looks simply dead.

## The bridge surface

### `bridge.data`

| Method | Permission | Behaviour |
|---|---|---|
| `getDataObjects(filter?)` | `data:read` | `filter` is `{path?, parentId?}`. |
| `getDataObject(uuid)` | `data:read` | `null` when missing. |
| `getAttributes(uuid)` | `data:read` | `[]` when missing. |
| `getAttribute(uuid, path)` | `data:read` | Resolved value, or `undefined`. |
| `setAttribute(uuid, path, value)` | `data:write` | Updates an existing attribute in place. To *create* one, the taxonomy metadata for `path` must be loaded; otherwise it warns and does nothing. |
| `addDataObject(parentUuid, path, taxonomyRef?)` | `data:write` | **async.** Resolves `taxonomyRef` from taxonomy metadata when omitted. Returns `null` when no document family or taxonomy ref can be resolved. |
| `deleteDataObject(uuid)` | `data:write` | **async.** |
| `getTagMetadata(path)` | `data:read` | |
| `getTaxonomies()` | `data:read` | |
| `clearFocusedValue()` | `data:write` | Blanks the focused field, keeping the attribute record. |
| `deleteFocusedValue()` | `data:write` | Removes the focused field's attribute record entirely. |
| `addDataGroup()` | `data:write` | Adds a sibling data object after the focused field's own. |
| `deleteDataGroup()` | `data:write` | Deletes the data object the focused field belongs to. |
| `blurField()` | `navigation` | Drops focus. Changes no data. |
| `showInDocument()` | `navigation` | Reveals the focused value in the viewer. |

The last six take no arguments — they act on whichever field has focus. Having
no focused field is the *common* case (a form shortcut fires app-wide), and they
warn and no-op rather than throwing. There is **no `deleteAttribute`** on the
bridge.

### `bridge.navigation` (permission `navigation`)

`focusAttribute(uuid, path, viewId?)` · `setPage(page, dfId?)` (**1-based**,
clamped) · `nextPage(dfId?)` · `previousPage(dfId?)` ·
`rotatePage("left"|"right", dfId?)` (relative ±90°, current page only) ·
`getCurrentPage(dfId?)` (1-based) · `getPageCount(dfId?)` · `getTabOrder()` ·
`focusNext("forward"|"backward")` · `nextSection()` · `nextException(dfId?)`.

`scrollToNode(ref)` and `switchView(name)` exist but are empty stubs — they
check the permission and return. Do not build on them.

Every method warns and no-ops when the underlying document view is not mounted.

### `bridge.viewer` (permission `viewer`)

All take an optional trailing document-family id.

`scroll("up"|"down"|"left"|"right")` · `zoom("in"|"out")` ·
`showRegion("top"|"middle"|"bottom")` · `fit("width"|"height")` ·
`copySelection()` · `detach()` (pop the viewer into a new tab) · `dock()` (dock
a popped-out viewer back).

When the viewer is detached, these are forwarded to the pop-out so a shortcut
fired from the form still moves the visible viewer.

### `bridge.form` (permission `formState`)

`get(key)` and `set(key, value)` back an in-memory map that is **not**
`ctx.formValues` — `bridge.form.set("x", 1)` is invisible to
`ctx.formValues.x`. Use `ctx.formValues` for anything a component should see.
`getNodeRef(ref)` returns a stub whose `setProps` does nothing.

### `bridge.log` (no permission)

`debug`, `warn`, `error` — console output prefixed `[DataFormV2]`.

## Keyboard shortcuts

Registered under a per-form scope on mount, cleared on unmount (so remounting
resets them), and forwarded to a detached viewer pop-out.

```yaml
shortcuts:
  - key: "alt+r"
    altKey: ["control+r"]        # string or string[]; optional
    description: "Rotate page right"
    group: document              # navigation | zoom | data-entry | document | ui
    scriptRef: rotateRight       # must name a key in `scripts`
scripts:
  rotateRight: |
    (ctx, bridge) => bridge.navigation.rotatePage("right")
```

`ctrl` normalises to `control`, `cmd`/`command` to `meta`, `option` to `alt`.
`description` is what appears in the app's keyboard-help dialog; `group` sorts
it there. An entry missing `key` or `scriptRef` is skipped silently.

## Declarative tab order

`tabOrder` (flat) and `tabOrderGroups` (with per-instance expansion) are
**mutually exclusive — if both are present `tabOrderGroups` wins and `tabOrder`
is discarded.** Declaring either changes the whole form: fields inside the chain
get `tabindex="0"`, and **every other field on the form is removed from the tab
chain** with `tabindex="-1"`. Grids manage their own cell focus and are skipped.

`tabOrderInitial` is `first` or `none` (default `none`); `first` focuses the
first chain entry on mount.

```yaml
tabOrderInitial: none
tabOrderGroups:
  - name: header
    paths: [invoice/invoice_number, invoice/invoice_date]
    wrap: next-group
    nextGroup: lines
  - name: lines
    forEach: { path: invoice/line_items }
    paths: [description, quantity, unit_price]   # relative to forEach.path
    wrap: next-instance
```

`wrap` fires only at a chain boundary (past the last entry going forward, past
the first going back): `next-group` jumps to `nextGroup`'s edge, `repeat`
restarts this group, `next-instance` wraps within a `forEach` group, `stop` (the
default) hands control back to the browser.

Flat form:

```yaml
tabOrder:
  - { path: invoice/invoice_number }
  - { path: invoice/invoice_date }
```

## Triggers

### `scriptTriggers` — watch attribute paths

```yaml
scriptTriggers:
  - script: recalcTotal
    triggerOn: [invoice/subtotal, invoice/tax_amount]
```

The watched paths are joined into one composite value; the script runs when that
value changes. The first evaluation only seeds the cache, so a trigger does not
fire on initial load. Writes made by a running script do not re-trigger it.

`scriptTriggers[].debounce` is persisted but **never read** — a script trigger
always fires immediately. Only `eventTriggers` debounce.

### `eventTriggers` — react to a document event

```yaml
eventTriggers:
  - script: onTotalChanged
    on: "changed:dataAttribute"
    filter: { path: invoice/total_amount }
    debounce: 300          # default 300; this one really is honoured
```

Only `on: "changed:dataAttribute"` **combined with `filter.path`** is wired up
today. A trigger missing either one is stored, round-trips, and is never
registered as a listener at all — it simply never runs. Each script named by a
registered trigger also fires once about a second after the form loads, with an
event type of `load`.

Every `scriptTriggers[].script`, `eventTriggers[].script` and
`shortcuts[].scriptRef` must name a key in `scripts` whenever the form declares
any inline scripts — otherwise the `data-form.script-ref-unknown` validation rule
rejects the save. `scriptModules` does not count; only `scripts` is consulted.
