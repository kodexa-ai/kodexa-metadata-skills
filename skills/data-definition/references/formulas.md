# KEXL — the formula language

One language, everywhere a formula appears:

| Field | Where |
|---|---|
| `ruleFormula`, `conditionalFormula`, `messageFormula`, `detailFormula` | `validationRules[]` |
| `condition` | `conditionalFormats[]` |
| `conditionalFormula` | `selectionOptions[]` |
| `selectionOptionFormula` | taxon, with `useSelectionOptionFormula: true` |
| `semanticDefinition` | taxon, when `valuePath: FORMULA` |

Every example below parses against the shipping parser.

## Attribute references

A reference is **always** inside `{ }`, with `/` between segments. A bare word outside braces parses
as an identifier and then fails at evaluation with `unresolved identifier`; a `.` outside `./` is a
parse error.

| Form | Meaning |
|---|---|
| `{InvoiceNumber}` | an attribute reachable from the current data object |
| `{LineItems/LineTotal}` | descend into a group |
| `{./UnitPrice}` | explicitly an attribute on the *same* data object |
| `{../InvoiceNumber}` | walk up one level; `{../../X}` walks two |
| `{/Invoice/TotalAmount}` | absolute from the taxonomy root |
| `{Unit Price}` | a segment name containing spaces |
| `{LineItems[category = "goods"]/LineTotal}` | predicate filter on a **group** segment |
| `@confidence` | a property of the attribute the formula belongs to |

Inside a `[...]` predicate, attribute names are written **bare** — `[category = "goods"]`, not
`[{category} = "goods"]` — because the predicate is evaluated with the candidate row as the current
data object and a bare identifier there resolves to one of its attributes. A braced reference inside
a predicate also resolves against the candidate, so reach outward with `{../X}`. A predicate on a
final **leaf** segment has no rows to filter and is ignored, so put predicates on the group segment:
`count({LineItems[quantity <= 0]})`.

### Which name a segment matches

Segments resolve against the **`externalName`** chain, falling back to `name` only when
`externalName` is blank. `label` is never used. Matching against the stored attribute tag is
case-insensitive, but the chain lookup is not — spell `externalName` exactly.

### How a reference is scoped

Every formula runs against one **data object** — the group instance that owns the attribute the rule
or formula sits on. That, not the taxon tree, is the frame of reference:

- A single-segment `{X}` first matches an attribute on that data object (case-insensitively, on the
  last segment of the attribute tag) — which is how sibling leaves of the same group are reached.
  Failing that it looks for a **child** taxon of that data object's own taxon; a child *group*
  resolves to the list of its child data objects.
- A slash path walks from there: `..` moves to the parent data object, `./` stays on this one. Its
  final segment tries a child of the current data object, then a sibling under the parent's external
  path, then — for multi-segment paths only — an absolute match anywhere in the document.

So a rule on a group reads its direct children by bare name, and a rule on a leaf reads its siblings
by bare name. Neither reaches into a nested group without naming it: `{LineItems/LineTotal}`.

### An unresolved reference is silent

If nothing matches, the reference comes back null or as an empty list — no error, no warning.
`sum({Typo})` returns 0; `isblank({Typo})` returns true; `{Typo} > 0` compares against nothing. Only
walking above the taxonomy root (`{../..}` from a true root object) raises. Check spelling against
`externalName` when a formula "works" but produces nonsense.

### Single values unwrap

A leaf reference that matches exactly one attribute returns that value as a scalar, not a
one-element list; two or more return a list. A **group** reference always returns the list of child
data objects, so `isblank({Group})` is a true row-count test at every row count including one.

## Operators

| Category | Operators |
|---|---|
| Equality | `=` (single, **not** `==`), `!=` |
| Relational | `<`, `>`, `<=`, `>=` |
| Logical | `&&`, `\|\|`, `!` (prefix) |
| Arithmetic | `+`, `-`, `*`, `/`, `^` |
| String concat | `&` |
| Postfix | `[index]` array access |
| Lambda | `=>` |

Uppercase `AND` / `OR` lex as ordinary identifiers and blow up the parse. Lowercase `and` / `or` are
reserved words but are not wired up as operators, so they fail too. Use `&&` and `||`.

Literals: numbers; strings in `"` or `'`; `true`, `false`, `null`; array literals `[a, b]`.

## Function registry

Names are matched case-insensitively; the platform convention is the lowerCamelCase spelling shown.

| Group | Functions |
|---|---|
| Math | `sum` `average` `min` `max` `abs` `ceil` `floor` `round` `stdDeviation` `decimalPlaces` |
| String | `concat` `substring` `lowercase` `uppercase` `trim` `replace` `split` `strlen` `length` `len` |
| Conditional | `if` `ifNull` `ifBlank` `sumifs` `countifs` |
| Validation | `isNull` `isBlank` `contains` `startswith` `endswith` |
| Date | `dateMath` `isdate` `isbeforedate` `isafterdate` `daysBetween` `weeksBetween` `monthsBetween` `validateDate` `formatdate` |
| Other | `regex` `matches` (alias of `regex`) `confidence` `count` `exists` |
| Bridge / selection | `serviceBridgeCall` `selectionContains` `getValue` |
| Side effect | `setDataFeature` |

That is the whole list. Anything else is `unknown function: <name>` at evaluation time.

Two signatures are easy to get wrong. `sumifs(range, criteriaRange, criterion…)` and
`countifs(range, criteriaRange, criterion…)` test **equality** against every criterion (all must
match); they are not Excel criteria strings, so `"<=0"` is compared as the literal text `<=0` — use a
group predicate when you need a comparison. `selectionContains(list, value)` asks whether `value`
appears in `list` (a plain array, or an array of `{label, value}` objects matched on `value`), so its
first argument is a multi-valued attribute or a computed option list, not a single-valued field.
`serviceBridgeCall(bridgeRef, endpointName, key1, value1, key2, value2, …)` takes the remaining
arguments as alternating key/value pairs. `daysBetween(from, to)` (and its week/month siblings)
returns `to - from`, negative when the arguments are reversed, with times truncated to whole days.

### Functions that do not exist, and what to write instead

| Not a function | Write |
|---|---|
| `NOT_EMPTY(x)` | `!isblank({X})` |
| `IS_EMPTY(x)` | `isblank({X})` |
| `NOT(expr)` | `!(expr)` |
| `COALESCE(a, b)` | `ifnull({A}, {B})` — or `ifblank({A}, {B})` when `""` should count as missing |
| `TODAY()` / `NOW()` | there is no clock function; compare two attributes, or use `dateMath({D}, "days", n)` |
| `DATE_ADD(d, 30, 'DAYS')` | `dateMath({D}, "days", 30)` — argument order is `(date, unit, amount)`, units `days` / `weeks` / `months` / `years`, lowercase |
| `REGEX_MATCH(s, p)` | `regex({S}, "p")` (`matches(...)` is the same function) |
| `IN(x, ["A","B"])` | `{X} = "A" \|\| {X} = "B"`, or `selectionContains({X}, "A")` when `{X}` is multi-valued |
| `ALL_VALUES(g.f > 0)` | `count({LineItems[quantity <= 0]}) = 0` — a group predicate, not a criteria string |
| `UNIQUE(g.f)` | no equivalent; enforce uniqueness in a SCRIPT step or a service bridge |

`isBlank` is true for `null`, an empty list, and a whitespace-only string. `ifNull` takes exactly two
arguments and substitutes when the first is null **or an empty list** — which is what an unresolved
reference gives you, so `ifnull({TaxAmount}, 0)` does cover a missing field. It does *not* substitute
for `""`. `ifBlank` is variadic, treats blank strings as missing too, and returns `""` when every
argument is blank.

## Patterns

```yaml
# Required field
ruleFormula: "!isblank({InvoiceNumber})"
messageFormula: '"Invoice number is required"'
overridable: false

# Cross-field arithmetic with a tolerance
conditional: true
conditionalFormula: "!isblank({Subtotal})"
ruleFormula: "abs({TotalAmount} - ({Subtotal} + ifnull({TaxAmount}, 0))) < 0.01"
messageFormula: '"Total does not match subtotal plus tax"'
detailFormula: 'concat("Expected ", {Subtotal} + ifnull({TaxAmount}, 0), " but the invoice shows ", {TotalAmount})'

# Date ordering
ruleFormula: "isafterdate({DueDate}, {InvoiceDate}) || {DueDate} = {InvoiceDate}"

# Format check
ruleFormula: 'regex({InvoiceNumber}, "^INV-[0-9]{4,}$")'

# Aggregate over a repeating group, authored on the parent
semanticDefinition: "sum({LineItems/LineTotal})"     # with valuePath: FORMULA

# Row-level arithmetic, authored on a child of the group
semanticDefinition: "{./Quantity} * {./UnitPrice}"   # with valuePath: FORMULA

# Filtered aggregate — bare attribute name inside the predicate
semanticDefinition: 'sum({LineItems[category = "goods"]/LineTotal})'

# "every row has a positive quantity"
ruleFormula: 'count({LineItems[quantity <= 0]}) = 0'

# Conditional-format condition
conditionalFormats:
  - type: backgroundColor
    condition: "{TotalAmount} > 10000"
    properties: { color: "#FEF3C7" }

# Dynamic selection options from a service bridge
useSelectionOptionFormula: true
selectionOptionFormula: 'serviceBridgeCall("erp-lookup", "currencies", "country", {../Country})'
```

### Does this group have any rows?

Author the rule on the group that **contains** the repeating group — a group with no rows has no data
object of its own for a rule to run on, but its parent always does.

```yaml
ruleFormula: "!isblank({LineItems})"      # ✅ reference the GROUP by its own external name
ruleFormula: "!isblank({LineItems/Description})"   # ❌ this asks about the children's attributes
```

`{LineItems}` resolves to the list of child data objects and `isblank` returns true for an empty
list, so this is the supported "at least one row" rule and it re-evaluates as rows are added and
removed.

### `messageFormula` and `detailFormula` are formulas, not templates

They are evaluated and the **string result** is shown. A fixed message is therefore a KEXL string
literal, which in YAML means quotes inside quotes:

```yaml
messageFormula: '"Total does not match subtotal plus tax"'   # ✅
messageFormula: "Total does not match subtotal plus tax"     # ❌ parsed as an expression, and fails
```

A `messageFormula` that fails to parse or evaluate is not reported — the exception silently falls
back to the stock message `Validation failed`. Same for `detailFormula`, which simply produces no
detail.

For an interpolated message use `concat(...)`; there is no `${}` or `{{ }}` substitution here.

## Why a broken formula is expensive

Formulas are not validated when the definition is applied. A parse error or an unknown function
surfaces at **document review time**: the validation runner marks the evaluation as errored and
raises a data exception on every affected data object with the message
`Validation rule configuration error` and details naming the underlying error. A conditional formula
that errors is treated the same way — it no longer fails open (a cleanly *false* conditional still
skips the rule as normal).

An unresolvable **reference** is quieter still: it does not error anywhere, so the rule evaluates
against an empty list and reports a perfectly ordinary pass or fail based on nothing. That is the
failure mode to suspect when a rule fires on every document, or never fires at all.

## Pitfalls

| Mistake | Fix |
|---|---|
| `NOT_EMPTY(field)` | `!isblank({Field})` |
| `SUM(line_items.line_total)` | `sum({LineItems/LineTotal})` |
| `==` for equality | `=` |
| `AND` / `OR` / `and` / `or` | `&&` / `\|\|` |
| Bare `total_amount > 100` | `{TotalAmount} > 100` |
| Referencing a taxon's `name` or `label` | reference its `externalName` |
| `messageFormula: "My message"` | `messageFormula: '"My message"'` — otherwise you silently get `Validation failed` |
| `DATE_ADD({D}, 30, "DAYS")` | `dateMath({D}, "days", 30)` |
| Expecting an error from a typo'd reference | there is none — it resolves to an empty list |
