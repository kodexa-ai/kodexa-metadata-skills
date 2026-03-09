---
name: data-form
description: "Use when creating or editing Kodexa Data Forms V2 — schema-driven JSON/YAML defining UI components for viewing and editing extracted document data, including UINode trees, data binding, QuickJS scripting, Bridge API, and event handling"
---

# Kodexa Data Form V2 Authoring

## Overview

Data Forms V2 define the UI for viewing and editing extracted document data. They use a component tree (UINode) model with data binding, scripting via QuickJS sandbox, and a Bridge API for interacting with platform services. Forms are defined in JSON and rendered dynamically in the workspace.

## When to Use

- Creating a data form for viewing/editing extracted document data
- Building custom UI layouts with panels, grids, tabs
- Adding scripting for dynamic behavior (formatting, validation)
- Configuring data binding between UI components and taxon data
- Migrating from V1 card-based forms to V2 schema forms

## Interactive Wizard

1. **Data source** — What taxonomy/data definition does this form display?
2. **Layout** — What layout? (single panel, tabs, split view, grid-heavy)
3. **Components** — What UI components? (labels, editors, grids, trees)
4. **Interactivity** — Any dynamic behavior? (formatting, conditional visibility, computed values)
5. **Actions** — Any buttons or event handlers needed?

Generate the complete V2 form JSON.

## V2 Top-Level Schema

```json
{
  "version": "2",
  "nodes": [],
  "scripts": {},
  "scriptModules": {},
  "bridge": {
    "permissions": ["data:read", "data:write", "navigation", "formState"],
    "apiBaseUrl": "/api",
    "maxExecutionMs": 2000
  }
}
```

| Field | Purpose |
|-------|---------|
| `version` | Must be `"2"` |
| `nodes` | Array of root UINode components |
| `scripts` | Named inline functions: `"name": "function(ctx) { ... }"` |
| `scriptModules` | Named modules with metadata: source, description, inputs, returns, debounce |
| `bridge` | Bridge API configuration: permissions, base URL, timeout |

## UINode Schema

```json
{
  "component": "card:cardPanel",
  "props": { "title": "Section Title" },
  "bindings": { "subtitle": "ctx.dataObjects?.length + ' items'" },
  "computed": { "backgroundColor": "deriveColor" },
  "events": {
    "click": { "type": "script", "target": "kodexa.log.debug('clicked')" }
  },
  "children": [],
  "slots": { "tab-1": [] },
  "if": "ctx.dataObjects?.length > 0",
  "show": "ctx.$item?.status !== 'archived'",
  "for": {
    "source": "ctx.dataObjects",
    "itemAs": "$item",
    "indexAs": "$index",
    "key": "$item.uuid"
  },
  "key": "unique-key",
  "class": "mt-4",
  "style": { "backgroundColor": "#ffffff" },
  "ref": "panelRef",
  "meta": {
    "label": "Panel Description",
    "category": "layout"
  }
}
```

## Component Registry

### Layout Components

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `card:cardPanel` | Container panel | `title`, `groupTaxon`, `collapsible` |
| `card:cardGroup` | Group of panels | `columns`, `gap` |
| `card:tabs` | Tabbed interface | `tabs` array |
| `card:horizontalLine` | Divider | `color`, `thickness` |

### Display Components

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `card:label` | Text display | `text`, `variant` |
| `card:singleTaxon` | Single taxon value | `taxon` path |

### Data Components

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `card:dataAttributeEditor` | Edit a data attribute | `taxon` path |
| `card:grid` | Data grid/table | `columns`, `dataSource` |
| `card:taxonGrid` | Taxon-based grid | `groupTaxon`, `columns` |
| `card:dataStoreGrid` | Store data grid | `storeRef` |
| `card:transposedGrid` | Transposed view | `groupTaxon` |

### Tree/Navigation Components

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `card:dataObjectsTree` | Data object tree | `rootPath` |
| `card:taxonTabs` | Tabs per taxon | `groupTaxon` |
| `card:exceptions` | Validation exceptions | `showResolved` |

## Event Handling

```json
{
  "events": {
    "click": {
      "type": "script",
      "target": "kodexa.log.debug('clicked', ctx.$item?.uuid)",
      "condition": "ctx.$item?.status !== 'locked'",
      "debounce": 300
    },
    "change": {
      "type": "scriptRef",
      "target": "validateField",
      "params": { "fieldName": "invoice_number" }
    },
    "submit": {
      "type": "emit",
      "target": "form-submitted"
    }
  }
}
```

Event types: `script` (inline JS), `scriptRef` (named script), `emit` (bubble to parent).

## Bridge API

### kodexa.data (requires `data:read` or `data:write`)

```javascript
kodexa.data.getDataObjects(filter)       // Get data objects
kodexa.data.getDataObject(uuid)          // Get single object
kodexa.data.getAttributes(uuid)          // Get attributes
kodexa.data.getAttribute(uuid, path)     // Get single attribute
kodexa.data.setAttribute(uuid, path, v)  // Set attribute value
kodexa.data.addDataObject(parentUuid, p) // Add child object
kodexa.data.deleteDataObject(uuid)       // Delete object
kodexa.data.getTaxonomies()              // Get available taxonomies
```

### kodexa.navigation (requires `navigation`)

```javascript
kodexa.navigation.focusAttribute(uuid, path)  // Focus on attribute
kodexa.navigation.scrollToNode(ref)            // Scroll to component
kodexa.navigation.switchView(viewName)         // Switch workspace view
```

### kodexa.form (requires `formState`)

```javascript
kodexa.form.get(key)                     // Get form state
kodexa.form.set(key, value)              // Set form state
kodexa.form.getNodeRef(ref)              // Get component reference
```

### kodexa.http (requires `http:get` or `http:post`)

```javascript
kodexa.http.get(path)                    // GET request
kodexa.http.post(path, body)             // POST request
```

### kodexa.log (no permission needed)

```javascript
kodexa.log.debug(...args)
kodexa.log.warn(...args)
kodexa.log.error(...args)
```

## Data Context Variables

| Variable | Description |
|----------|-------------|
| `ctx.dataObjects` | Array of data objects at current scope |
| `ctx.tagMetadataMap` | Map of taxonomy path to tag metadata |
| `ctx.$item` | Current item in `for` loop |
| `ctx.$index` | Current index in `for` loop |
| `ctx.$parent` | Parent data context |
| `ctx.$root` | Root data context |

## Complete Example

```json
{
  "version": "2",
  "nodes": [
    {
      "component": "card:cardPanel",
      "props": {
        "title": "Invoice Summary",
        "groupTaxon": "invoice",
        "collapsible": true
      },
      "children": [
        {
          "component": "card:cardGroup",
          "props": { "columns": 2 },
          "children": [
            {
              "component": "card:dataAttributeEditor",
              "props": { "taxon": "invoice/invoice_number" }
            },
            {
              "component": "card:dataAttributeEditor",
              "props": { "taxon": "invoice/invoice_date" }
            },
            {
              "component": "card:dataAttributeEditor",
              "props": { "taxon": "invoice/vendor/name" }
            },
            {
              "component": "card:dataAttributeEditor",
              "props": { "taxon": "invoice/total_amount" },
              "bindings": {
                "class": "ctx.dataObjects?.[0]?.attributes?.total_amount > 10000 ? 'highlight-warning' : ''"
              }
            }
          ]
        },
        {
          "component": "card:horizontalLine"
        },
        {
          "component": "card:taxonGrid",
          "props": {
            "groupTaxon": "invoice/line_items",
            "columns": [
              { "field": "description", "header": "Description", "width": 300 },
              { "field": "quantity", "header": "Qty", "width": 80 },
              { "field": "unit_price", "header": "Unit Price", "width": 120 },
              { "field": "line_total", "header": "Total", "width": 120 }
            ]
          }
        },
        {
          "component": "card:cardPanel",
          "props": { "title": "Totals" },
          "children": [
            {
              "component": "card:dataAttributeEditor",
              "props": { "taxon": "invoice/subtotal" }
            },
            {
              "component": "card:dataAttributeEditor",
              "props": { "taxon": "invoice/tax_amount" }
            },
            {
              "component": "card:dataAttributeEditor",
              "props": { "taxon": "invoice/total_amount" }
            }
          ]
        },
        {
          "component": "card:exceptions",
          "props": { "showResolved": false }
        }
      ]
    }
  ],
  "scripts": {
    "formatCurrency": "function(ctx) { return '$' + Number(ctx.value).toFixed(2); }",
    "highlightOverdue": "function(ctx) { return new Date(ctx.value) < new Date() ? 'overdue' : ''; }"
  },
  "bridge": {
    "permissions": ["data:read", "data:write", "navigation", "formState"],
    "maxExecutionMs": 2000
  }
}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Missing `version: "2"` | Required to use V2 schema |
| Bridge permissions not declared | Scripts fail silently without required permissions |
| Wrong taxon path in `props.taxon` | Must match taxonomy hierarchy: `group/field` |
| `if` vs `show` confusion | `if` removes from DOM; `show` hides with CSS |
| Missing `key` in `for` loops | Always provide unique key for list rendering |
| Script exceeding `maxExecutionMs` | Optimize or increase timeout in bridge config |
