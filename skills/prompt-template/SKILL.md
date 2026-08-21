---
name: prompt-template
description: "Use when creating or editing a Kodexa prompt resource — flat YAML with type: prompt-template in kdxa_prompts, carrying promptTemplate + templateType (FSTRING or MUSTACHE) and/or a plain prompt body, referenced by activity-plan LLM steps via promptTemplateRef, by the organization LLM endpoint via promptRef, by Python via Prompt.render(), and by the Studio chat prompt picker. Covers which key each consumer reads, brace syntax, and the fields that do not exist."
---

# Kodexa prompt resources

A prompt resource is org-scoped, reusable **prompt text** stored in `kdxa_prompts`. That is all it
is. It carries no model choice, no temperature, no token limit and no output schema — those belong to
whoever invokes it. It is also *not* how you steer document extraction (see "Extraction prompts" below).

Its metadata column is opaque JSON. The server stores whatever you send and hands it back flattened
to the top level, so **any key you invent will persist and round-trip while being read by nothing.**
Round-tripping is not evidence a field works. Only the keys below are read by anything.

Consumer detail, variable tables, overrides and troubleshooting: `references/consumers.md`.

## The shape that works

```yaml
type: prompt-template                  # see "Deploying" — never `promptTemplate`
slug: invoice-extraction-prompt        # required, 3–100 chars; derived from name if omitted
orgSlug: acme-corp                     # target org (or pass --org-slug)
name: Invoice Extraction Prompt
description: Extracts header fields from a vendor invoice

prompt: |                              # read by the Go consumers and the chat picker
  Extract invoiceNumber, invoiceDate and totalAmount from the invoice below.
  Return JSON. Use null for anything not explicitly present.

  {invoiceText}

promptTemplate: |                      # read by the Python consumers
  Extract invoiceNumber, invoiceDate and totalAmount from the invoice below.
  Return JSON. Use null for anything not explicitly present.

  {invoiceText}

templateType: FSTRING                  # FSTRING | MUSTACHE — required by the Python renderer
```

Everything is **flat at the top level**. The whole request body is stored as the metadata blob, so a
body nested under a `metadata:` block lands one level down inside it. The GET response flattens that
back out, so it still looks right in Studio — but the two Go consumers read the stored blob directly
and never see the key. It saves, returns 201, and breaks only at run time.

## Which key each consumer reads — the failure that ships silently

Two different code paths look for two different keys, and neither errors loudly when its key is absent.

| Consumer | Key it reads | If that key is absent |
|---|---|---|
| Activity-plan `LLM` step via `promptTemplateRef` | top-level `prompt`, then `template` | **the entire metadata JSON blob is sent to the model as the prompt** |
| `POST /api/organizations/{orgId}/llm` via `promptRef` | same | same |
| Python `Prompt.render()` / the SDK's prompt object | `promptTemplate` + `templateType` | raises `ValueError: Unknown template type None` |
| Studio chat "From prompt" picker | `prompt` (plus `context`) | the prompt never appears in the picker |

So a prompt carrying only `promptTemplate` still "works" from an `LLM` step — it just sends the model
`{"slug":"...","name":"...","promptTemplate":"...","templateType":"FSTRING",...}` and you get a
confused answer, not an error. **Anything reached by `promptRef` / `promptTemplateRef` must carry a
top-level `prompt:`.** Author both keys when in doubt.

`template` is not an escape hatch: on every metadata resource `template` is the reserved boolean
"is this a template resource" flag, so the Go fallback's `template` branch can never match a string —
and a string sent there fails the request body decode outright.

## Template dialects

`templateType` selects the renderer. There are exactly two, and **neither is Handlebars** — no
`{{#each}}`, no `{{this.field}}`.

| `templateType` | Placeholder syntax | Notes |
|---|---|---|
| `FSTRING` | `{variableName}` — single braces | Go renderers do a literal replace of `{key}`. The Python renderer is `str.format`, so **double every literal brace** (`{{`, `}}`) in a body Python will render. |
| `MUSTACHE` | `{{variableName}}`, `{{{variableName}}}` unescaped, `{{#name}}…{{/name}}` for a present/absent section | No `#each` helper and no `this.` accessor. |

Both Go paths (the `LLM` step and the organization LLM endpoint) render **FSTRING only** — they do a
plain `{key}` substitution regardless of what `templateType` says. `templateType` matters to the
Python renderer, which raises on any value other than `FSTRING` or `MUSTACHE`, including absent.

There is no global variable vocabulary. A placeholder resolves only if the caller puts that key in
the parameter map it passes. See `references/consumers.md` for the variables each caller supplies.

## Using a prompt from an activity plan

`promptTemplateRef` is a **bare slug**, resolved against `kdxa_prompts` within the plan's own
organization. It is not `org/slug` and not a `prompt-template://` URI.

```yaml
- slug: classify-invoice
  type: LLM                            # legacy AI_PROMPT also dispatches; AI_PLANNER does not
  name: Classify the invoice
  promptTemplateRef: invoice-extraction-prompt   # bare slug — OR promptBody, never both
  llmModelName: LARGE                  # SMALL | LARGE | a literal gateway model id
  promptVariables:                     # JSONata over {context, enrichment, steps}
    invoiceText: "steps.`load-document`.text"    # backticks — see below
  outputMapping:
    invoiceNumber: "invoiceNumber"     # JSONata over the parsed response
```

A step with neither `promptBody` nor `promptTemplateRef` fails outright, and so does a
`promptTemplateRef` that resolves to nothing. **Backtick any hyphenated step slug inside a
`promptVariables` / `outputMapping` expression** — JSONata reads the bare hyphen as subtraction, the
variable resolves to nothing, and the `{placeholder}` is left in the prompt with no error (only
`conditionExpr` is checked for this). Full field list in `references/consumers.md`; the step graph
itself is the `activity-plan` skill's subject.

## Chat pre-built prompts

An org-wide prompt appears in the Studio chat panel's "From prompt" picker when it carries a
non-empty `prompt` body plus a `context`:

```yaml
type: prompt-template
slug: summarise-this-task
orgSlug: acme-corp
name: Summarise this task              # doubles as the created chat channel's name
prompt: |
  Summarise the current task, listing open exceptions first.
context: task                          # "task" | "project" — which screen offers it
category: Triage                       # picker grouping; omitted → "General"
```

The picker lists prompts from the current organization only — `publicAccess` does not surface a
prompt in another org's picker.

## Extraction prompts do not live here

Per-data-element extraction guidance is the taxon's `semanticDefinition` on a data definition, not a
prompt resource. `typeFeatures.allowTemplating: true` on a taxon additionally renders that definition
as a **Jinja2** template — a third dialect, with exactly three context roots (`external_data`,
`metadata`, `knowledge`); undefined lookups render empty rather than raising. Per-vendor overrides are
knowledge items of the seeded `kodexa/taxon-semantic-definition-customization` type, not a bespoke
item type of your own — see `references/consumers.md`, then `data-definition` and `knowledge-system`.

## Deploying

- Files live in `<metadata_dir>/prompt-templates/<slug>.yaml`; the manifest key is `prompt-template:`.
- `kdx apply -f <file>` needs `type:` (or `--type`) plus `slug` and `orgSlug` (or `--org-slug`).
- CRUD is `/api/prompts` (+ `/{id}`). Org-scoped, unique on `(organization, slug)`, soft-deleted.
- `${org}` in the file is substituted with the target org on push; leftovers are a hard error.
- `type` is excluded from the push comparison: a type-only edit never re-pushes. Change something else too.

**Only `prompt` and `prompt-template` are valid `type` values.** The CLI also *accepts*
`promptTemplate` / `prompttemplate` / `prompt-templates` / `promptTemplates` as file-type aliases,
but the value is persisted verbatim and the resource's own `uri` is built from it — so a camelCase
type yields `promptTemplate://acme-corp/my-prompt`, which `POST /api/resolve` rejects as an
unsupported scheme, and any client that follows that `uri` fails. `type: prompt` is what the server
defaults to and what the shipped `kodexa` prompts carry, but it is not a CLI alias, so a file typed
`prompt` bypasses the sync engine and loses `${org}` resolution. **`prompt-template` is the only
value correct on both sides. Use it.**

## Declared but inert

Persisted, returned by the API, present in existing files — and read by nothing:

| Key | Note |
|---|---|
| `version` | Prompt version columns were dropped; the shipped `kodexa` prompt files still carry `version: 1.0.0` and it means nothing. |
| `org` | The organization comes from `orgSlug` / `--org-slug`. A separate `org:` key is stored and echoed only. |
| `provider`, `providerUrl`, `providerImageUrl`, `imageUrl`, `icon`, `overviewMarkdown` | Shared decoration fields the platform defines for metadata resources generally. Studio renders them for modules and models; no prompt surface reads any of them. The shipped `kodexa` prompts still set several. |
| `template`, `deprecated` | Real booleans you can filter a list query on. Nothing in the prompt path acts on them — the Go extractor's `template` branch only ever sees the boolean and skips it. |
| `systemPrompt`, `userPromptTemplate`, `modelProvider`, `modelId`, `temperature`, `maxTokens`, `outputFormat`, `outputSchema` | Never existed. Older prompt documentation taught all eight. Because the metadata column is opaque JSON they save, round-trip and look real. Model choice comes from the caller; sampling parameters are fixed by the platform LLM client; output shape comes from the step's `outputMapping`. |

## Common mistakes

| Mistake | What happens |
|---|---|
| `type: promptTemplate` | Persisted verbatim; the resource's `uri` becomes `promptTemplate://…`, which `POST /api/resolve` rejects. Use `prompt-template`. |
| Body nested under `metadata:` | Reads back flat over the API, so it looks fine — but the Go consumers read the stored blob and never see it. |
| Only `promptTemplate`, reached via `promptTemplateRef` / `promptRef` | The whole metadata JSON is sent to the model. Add a top-level `prompt:`. |
| `templateType` omitted | Python rendering raises `Unknown template type None`. |
| Handlebars `{{#each}}` / `{{this.x}}` | Renders nothing under MUSTACHE; left literal or raises under FSTRING. |
| Literal `{` in an FSTRING body rendered by Python | Double it: `{{` / `}}`. |
| `promptTemplateRef: acme-corp/my-prompt` | Not found — it is a bare slug in the plan's org. |
| Both `promptBody` and `promptTemplateRef` on one step | Mutually exclusive; pick one. |
| `steps.load-document.text` in `promptVariables` | Hyphen parses as subtraction → resolves to nothing, placeholder left in the prompt. Backtick the slug. |
| `type: AI_PLANNER` on the step | Not a working alias for `LLM`. There is no read-time remap, so the step is silently SKIPPED at run time. (`AI_PROMPT` does still dispatch.) |
| `maxParallel` on an `LLM` step | Rejected with a 400 when the activity starts: per-document LLM work runs sequentially. |
| `includeDocument` without `perDocument: true` | Inert — the document variables only exist on the per-document path, and the run logs a warning. |

## Prompt engineering that holds up

- **Name the format, don't imply it.** "Return the date as YYYY-MM-DD, e.g. 2026-03-15".
- **Show two or three worked examples**, including a messy one: `"Invoice #12345"` → `12345`.
- **Say what to do when the value is absent.** "Return null; do not infer or guess." Without this the
  model invents plausible values.
- **Ask for JSON explicitly and show the object** you want back, then shape it downstream with the
  step's `outputMapping`. There is no schema-constrained decoding to lean on.
- **One prompt, one job.** Extraction, classification and validation as separate prompts are easier
  to evaluate and to override than one prompt doing all three.
- **Keep vendor-specific quirks out of the body** — put them in knowledge overrides.

## Cross-references

- `activity-plan` — the `LLM` step that invokes a prompt, its dependencies and outputs
- `knowledge-system` — knowledge sets, feature expressions, and prompt-override items
- `data-definition` — `semanticDefinition`, `allowTemplating`, `promptStrategy`
- `module` — packaging `SKILL.md` / `SYSTEM_PROMPT.md` into a skill module
- `kdx-cli` — `kdx apply`, `kdx sync push/pull`, manifests
