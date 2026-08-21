# Kodexa Metadata Skills

A [Claude Code](https://claude.ai/claude-code) plugin providing skills for authoring [Kodexa platform](https://kodexa.ai) metadata — the YAML behind data definitions, activity plans, task templates, forms, modules, knowledge sets and the rest of the resource model.

Each skill is written against the platform source: field names from the entity models, enum values from the constants that validate them, endpoints and status codes from the handlers, CLI flags from the command definitions. Where a field is accepted and then silently ignored, the skill says so rather than leaving you to find out.

## Skills

| Skill | Covers |
|-------|--------|
| **activity-plan** | Org-scoped step graphs — `EXECUTION`, `CREATE_TASK`, `SCRIPT`, `LLM`, `BRIDGE_CALL`, `AGENT`; the flat step envelope, dependency grammar, per-document routing, document status control |
| **assistant** | Project-scoped pipelines — `options.pipeline` steps, taxonomy refs, agent config, and when to reach for an activity plan instead |
| **data-definition** | Taxonomies — taxons, data types, `semanticDefinition` extraction prompts, type features, repeating groups, validation and formulas |
| **data-form** | Review panels — the V2 `nodes` tree and its `v2:*` components, legacy V1 `card:*` forms, scripting and the bridge calling convention |
| **intake** | Getting documents in — pointing an intake at a store, wiring it to an activity plan, upload scripts, tokens, and the source types that actually deliver |
| **kdx-cli** | The `kdx` CLI — profiles and login, resource CRUD, `validate`/`apply`, `run`, and sync/deploy with manifests |
| **knowledge-system** | Knowledge sets, feature types, item types and feature instances — `featureExpression` trees, computed slugs, options |
| **module** | Python, Go-WASM and inline-JavaScript model modules plus agent skill packs — `moduleType`, runtime refs, sidecars, inference options |
| **project-resource** | Making an org-scoped resource usable in a project — the binding model, and the silent failures when a binding is missing |
| **project-template** | Blueprints that provision a project — stores, assistants, taxonomies, forms, status workflows, and the closed-struct rule that drops unknown keys |
| **prompt-template** | Prompt resources — `promptTemplate` with `FSTRING` or `MUSTACHE` templating, and the four paths that consume one |
| **service-bridge** | Proxying an external HTTP API — `baseUrl`, named endpoints, secret interpolation, OAuth2, caching, and the egress fence |
| **task-status** | The workflow states tasks move through — `statusType`, locking, and which mechanics fail open versus closed |
| **task-template** | Human review/approval tasks — action buttons and their transitions, attached forms, document groups, AI naming |
| **trigger** | Starting an activity plan when a platform event fires — event kinds, JSONata filters and input mapping |

## Installation

```bash
claude plugin marketplace add kodexa-ai/kodexa-metadata-skills
claude plugin install kodexa-metadata-skills
```

The skills then load automatically when Claude Code detects a matching task.

To try a local checkout without installing:

```bash
claude --plugin-dir /path/to/kodexa-metadata-skills
```

## Usage

Ask for what you want and the relevant skill loads on its own:

```
> Create a data definition for extracting purchase order data
> Why is my trigger not firing?
> Add an approval step to this activity plan
```

You can also invoke one explicitly:

```
> /kodexa-metadata-skills:data-definition
```

## How the skills are organised

Each skill follows the progressive-disclosure layout:

```
skills/<name>/
  SKILL.md              # ≤200 lines — always loaded
  references/*.md       # loaded on demand
```

`SKILL.md` carries the shape that works plus everything separating "works" from "silently broken". Bulk field tables, enum inventories, long examples and troubleshooting live in `references/`, pulled in only when needed.

Every skill also carries a **Declared but inert** section. Several Kodexa resources persist and round-trip fields that nothing reads — some are editable in the UI. Those fields are listed rather than omitted, because you will meet them in existing YAML and silence reads as endorsement.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for where truth lives, the house rules that keep this repo publishable, and how to check a change.

Two things worth knowing before you edit:

- **Verify against platform source, never against another document.** Most errors here got in by being copied forward without re-checking.
- **Bump `version` in `.claude-plugin/plugin.json` in the same PR as any change under `skills/`.** That field pins the plugin — installed users receive nothing until it moves.

## Documentation

Full Kodexa platform documentation is at [developer.kodexa.ai](https://developer.kodexa.ai).

## License

Apache 2.0 — see [LICENSE](LICENSE).
