# Prompt consumers, variables and overrides

Reference detail for the four paths that read a prompt resource, the variables each one supplies,
and the two adjacent mechanisms authors reach for by mistake (knowledge overrides, skill modules).

Read `SKILL.md` first — the rule about which key each consumer reads lives there, and getting it
wrong is the failure that ships silently.

---

## 1. Activity-plan `LLM` step

The primary consumer. Discriminator is `type: LLM` on the step. All fields sit **flat on the step**,
not under a `config:` block.

Only `LLM` and the legacy `AI_PROMPT` dispatch to the prompt path. `AI_PLANNER` / `AIPLANNER` are
older values that were normalised in the data by a one-off migration; there is **no read-time
remapping**, so a step still carrying one falls through the runtime's dispatch switch and is silently
marked SKIPPED. Nothing rejects it at save or at start. Write `LLM`.

| Field | Meaning |
|---|---|
| `promptBody` | Inline FSTRING prompt. Mutually exclusive with `promptTemplateRef`. |
| `promptTemplateRef` | **Bare slug** into `kdxa_prompts`, resolved in the plan's own organization. |
| `llmModelName` | `SMALL`, `LARGE`, or a literal gateway model id. Empty → small. It *is* evaluated — do not trust older notes calling it reserved. |
| `enrichment` | Array of service-bridge calls run **before** rendering; each result lands under `{enrichment}` keyed by its `outputKey`. |
| `includeDocument` | Injects document variables. Only honoured together with `perDocument: true`; otherwise the run logs a warning and nothing is injected. |
| `promptVariables` | `{name: jsonataExpr}` evaluated over the variables below, adding more variables. |
| `outputMapping` | `{field: jsonataExpr}` over the parsed response. The response is `json.Unmarshal`ed best-effort; if it is not JSON it becomes `{"response": "<text>"}`. |
| `promptActions` | `[{slug, name, uuid}]` declared outcomes. The value `outputMapping` writes to `action` is matched against `slug` (exact) first, then `name` (case-insensitive). No match logs a warning and leaves the step with no action — it does not fail the step. |
| `perDocument` | Fan out over the activity's document families. |

Hard failures: a step with neither `promptBody` nor `promptTemplateRef` fails; a `promptTemplateRef`
that resolves to nothing fails the step. `maxParallel` on an `LLM` step, and `perDocument` on
anything other than `EXECUTION` / `LLM` / `SCRIPT` / `BRIDGE_CALL`, are both rejected with a 400 when
the activity starts (and reported ahead of time by the plan-validate endpoint) rather than being
silently ignored — a clean *save* proves nothing, because the save-time rule set is separate.

### Variables available to the prompt body

Single-shot (`perDocument` unset or false):

| Placeholder | Contents |
|---|---|
| `{context}` | `{orgId, projectId, inputs}` — the activity's inputs |
| `{enrichment}` | Map keyed by each enrichment step's `outputKey` |
| `{steps}` | Prior **COMPLETED** steps keyed by step slug. Each step's mapped-output keys are flattened directly onto it, plus `completedActionUuid`. |
| anything in `promptVariables` | JSONata evaluated over the three above |

With `perDocument: true`, add:

| Placeholder | Contents |
|---|---|
| `{documentFamilyId}` | Always present on the per-document path |
| `{extractedData}` | The document family's metadata — **only when `includeDocument` is set** |
| `{documentPath}` | The document family's path — only when `includeDocument` is set |
| `{documentText}` | Raw document text — only when `includeDocument` is set |

A document that fails to load, or exceeds the injection limits, fails **that document**, not the step.

`{context}`, `{enrichment}` and `{steps}` interpolate through Go's `%v`, so dropping one whole into a
prompt produces Go map syntax (`map[orgId:… inputs:map[…]]`), not JSON. Pull the scalars you actually
want through `promptVariables` instead:

```yaml
promptVariables:
  vendorName: "enrichment.vendorRecord.name"
  priorTotal: "steps.`extract-header`.totalAmount"    # mapped keys are flat on the step
```

**Backtick every hyphenated step slug.** JSONata parses `steps.extract-header.totalAmount` as a
subtraction, so the expression yields nothing, the variable is never added, and the `{priorTotal}`
placeholder survives into the prompt with no warning. The plan validator checks `conditionExpr` for
this but not `promptVariables` or `outputMapping`. A leading `$.` is harmless but unnecessary —
`enrichment.x` and `$.enrichment.x` evaluate identically.

---

## 2. `POST /api/organizations/{orgId}/llm`

Direct invocation. Body:

```json
{
  "promptRef": "acme-corp/invoice-extraction-prompt",
  "parameters": { "documentText": "..." },
  "modelType": "LARGE"
}
```

- `promptRef` is `"promptSlug"` (resolved in the URL's org) or `"orgSlug/promptSlug"`. Supply
  `prompt` instead to send an inline body; one of the two is required.
- `parameters` — every key becomes an FSTRING variable. This is the whole variable vocabulary for
  this path; there are no platform-supplied variables.
- `modelType` — `SMALL` (default) or `LARGE`; any other non-empty value is passed through as a
  literal gateway model id.
- The response is `text/plain`. No JSON mode, no schema constraint, no `response_format`.

Temperature (`0.1`) and max tokens (`4096`) are fixed by the platform LLM client for every call on
this path and on the `LLM` step. They are not authorable anywhere.

---

## 3. Python: `Prompt.render()`

Used from modules and scripts, either by constructing a prompt object directly or by fetching one
through the platform client and calling `.render({...})`.

```python
prompt = client.get_object_by_ref("prompt", "acme-corp/invoice-extraction-prompt")
text = prompt.render({"invoiceText": doc_text})   # see the gotcha below before relying on this
```

- `FSTRING` renders with `str.format`, so **every literal brace in the body must be doubled**
  (`{{`, `}}`). A single stray `{` raises `KeyError` or `ValueError` at render time.
- `MUSTACHE` renders with `chevron`.
- Any other `templateType` — including missing — raises `ValueError: Unknown template type ...`.

**Known gotcha — the fetch-then-render path is currently broken.** The bundled Python prompt model
declares `prompt_template` and `template_type` in snake_case with **no camelCase alias**, and the
shared base model sets `populate_by_name` but installs no alias generator. The API serves
`promptTemplate` / `templateType`, so those wire keys land in the model's extras, both attributes
stay `None`, and `render()` falls into the `else` branch: `ValueError: Unknown template type None` —
on a prompt that is perfectly well-formed. Until it is fixed, set `prompt_template` /
`template_type` explicitly on the object you construct. Do **not** rename the keys in the stored
resource; every other consumer reads the camelCase names, and renaming breaks all of them.

---

## 4. Studio chat "From prompt" picker

Reads three flattened top-level keys:

| Key | Effect |
|---|---|
| `prompt` | The body prefilled into the chat input. Must be non-empty or the prompt is filtered out. |
| `context` | `"task"` or `"project"` — which screen offers the prompt. Anything else never matches. |
| `category` | Picker grouping. Absent or blank → `"General"`. |

`name` is both the picker title and the name of the chat channel the picker creates. The picker lists
prompts from the current organization only.

---

## Per-vendor prompt overrides — the recipe that actually applies

Do **not** define your own knowledge item type for this. The extraction pipeline filters knowledge
items by the exact type slug `taxon-semantic-definition-customization`; an item of any other type is
never inspected, so a bespoke override type is inert no matter how it is shaped.

One seeded `kodexa` item type carries prompt text into extraction:

| Type slug | `impact` | Options | Effect |
|---|---|---|---|
| `taxon-semantic-definition-customization` | `PROMPT_OVERRIDE` | `taxonomyAndTaxon`, `prompt`, `replace` | Replaces or appends to one data element's `semanticDefinition` |

A sibling type `kodexa/base-prompt-customization` (`impact: BASE_PROMPT_OVERRIDE`, options
`taxonomyAndTaxon` + `prompt`) is seeded alongside it, but **no extraction code reads it**. The
pipeline applies exactly four item-type slugs — `taxon-semantic-definition-customization`,
`taxon-enabled`, `taxon-name-customization`, `taxon-selection-option-enabled` — and that is not one
of them, so items of it persist and go nowhere.

```yaml
type: knowledge-set
slug: acme-prompts
orgSlug: acme-corp
name: Acme prompt overrides
featureExpression:                       # eligibility rule; a `clauses:` list is legacy and ignored
  type: FEATURE
  slug: acme
features:
  - slug: acme
    featureTypeRef: acme-corp/vendor      # a feature type you define; orgSlug/slug
    properties: { vendorId: "ACME" }
knowledgeItems:                          # the has-many is knowledgeItems, not `items`
  - slug: acme-invoice-number
    title: Acme invoice number format
    knowledgeItemTypeRef: kodexa/taxon-semantic-definition-customization
    properties:
      taxonomyAndTaxon: "acme-corp/invoice//Invoice/InvoiceNumber"
      prompt: |
        Acme invoice numbers use the form ACME-YYYY-NNNNN, printed top-right in bold.
        Examples: ACME-2026-00123, ACME-2026-00456
      replace: false                     # false appends to the existing definition; true replaces it
```

Details that decide whether it applies:

- `taxonomyAndTaxon` is `orgSlug/taxonomySlug//TaxonPath`, split on the **double** slash. The taxon
  part is the chain of taxon **names**, which is what the Studio taxon picker persists.
- The properties read are exactly `taxonomyAndTaxon` (or the legacy `taxon`), `prompt` and `replace`.
  Names such as `customPrompt`, `taxonPath` or `examples` are read by nothing.
- If several eligible items target the same taxon, one wins deterministically by created date, then
  `sequenceOrder`, then id — not by file order.
- Set authoring (features, feature expressions, activation) is the `knowledge-system` skill's subject.

### Images in override prompts

A knowledge item property whose type is declared `markdown` on its item type — which `prompt` is on
`taxon-semantic-definition-customization` — may embed images as `![](attachment://<attachmentId>)`.
Upload the file against the item or the set, giving it a stable `attachmentId`, then reference that
id in the markdown; the CLI prints the ready-made reference after upload. This is a **knowledge**
mechanism only: a prompt resource has no attachment support.

---

## Prompts inside skill modules

A skill module does not carry a `prompts/` directory — nothing reads one. The contract is two
filenames at the **package root** of the module archive:

```
my-skill-module/
  module.yml            # moduleType: skill
  SKILL.md              # discovered by the agent runtime
  SYSTEM_PROMPT.md      # optional; appended verbatim to the agent's system prompt
  references/…          # anything SKILL.md points at
```

The runtime walks one level deep, so `SKILL.md` must sit at the archive root — an extra wrapping
folder hides it. Module slugs share one cache namespace across organizations, so two modules with the
same slug collide and the last one downloaded wins. See the `module` skill for packaging.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| The model answers about JSON structure, or echoes field names like `slug` and `templateType` | The prompt has no top-level `prompt:` key and was reached by `promptRef` / `promptTemplateRef`, so the whole metadata blob was sent. |
| `ValueError: Unknown template type None` | Python path: either `templateType` is missing, or the camelCase/snake_case alias gap described above. |
| `KeyError` / `ValueError` at render time in Python | An undoubled literal `{` or `}` in an `FSTRING` body. |
| Placeholder appears verbatim in the model's input | Go FSTRING leaves unknown `{keys}` untouched. The caller did not supply that variable — check `promptVariables` (backticks on hyphenated step slugs), `parameters`, or `includeDocument` + `perDocument`. |
| A `{{#section}}` block renders nothing | MUSTACHE only emits a section when the key is present and truthy; and `#each` is not a Mustache helper at all. |
| Step fails with "neither prompt_body nor prompt_template_ref" | Both were omitted, or `promptTemplateRef` was nested under a `config:` block instead of flat on the step. |
| `promptTemplateRef` not found | It is a bare slug in the plan's organization — strip any `orgSlug/` prefix or `prompt-template://` scheme. |
| Activity start returns 400 citing `maxParallel` | Not supported on `LLM` steps; the check runs at start, not at save. |
| A knowledge prompt override changes nothing | The item's type is not `kodexa/taxon-semantic-definition-customization`, or its properties use invented names, or `taxonomyAndTaxon` does not resolve to a real taxon name chain. |
| Editing `type:` in a file does not re-push | `type` is excluded from the push comparison; change something else in the file too. |
| A `yamlSource` field appears in the GET response | The server retains the YAML text a push sent and echoes it back. Server-managed — never author it, and never treat it as the live shape. |

---

## A complete MUSTACHE example

```yaml
type: prompt-template
slug: contract-clause-summary
orgSlug: acme-corp
name: Contract Clause Summary

prompt: |
  Summarise the termination and renewal clauses in the contract below.
  Quote the clause number for each point.

promptTemplate: |
  Summarise the termination and renewal clauses in the contract below.
  Quote the clause number for each point.

  {{#priorSummary}}
  A previous summary exists; note only what has changed since:

  {{{priorSummary}}}
  {{/priorSummary}}

  ## Contract text
  {{{contractText}}}

templateType: MUSTACHE
```

`{{{triple}}}` emits the value unescaped, which is what you want for document text. `{{double}}`
HTML-escapes, which mangles quotes and angle brackets in extracted text.

Note what the `prompt:` copy does here: it is deliberately placeholder-free. Only the Python renderer
honours `templateType`; the two Go consumers read `prompt` and always do FSTRING `{key}` replacement,
so a MUSTACHE body placed under `prompt:` would reach the model with its `{{…}}` untouched. Keep a
MUSTACHE body in `promptTemplate` and give `prompt` a self-contained FSTRING version, or leave
`prompt` off entirely if the prompt is only ever rendered from Python.
