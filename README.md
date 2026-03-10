# Kodexa Metadata Skills

A [Claude Code](https://claude.ai/claude-code) plugin providing skills for authoring [Kodexa platform](https://kodexa.ai) metadata. Each skill includes complete reference schemas, interactive wizards, YAML examples, and common mistake guides.

## Skills

| Skill | Description |
|-------|-------------|
| **data-definition** | Taxon hierarchies, data types, validation rules, conditional formatting, semantic extraction definitions |
| **project-template** | Complete project blueprints — stores, assistants, taxonomies, data forms, workspaces, knowledge sets, status workflows |
| **module** | Model modules (Python ML/AI) and skill modules (file packs) — runtime config, sidecars, inference options, deployment |
| **data-form** | V2 schema-driven UI forms — UINode component trees, data binding, QuickJS scripting, Bridge API |
| **assistant** | Event-driven processing pipelines — connections, subscriptions, multi-step orchestration, scheduling |
| **task-template** | Task workflows — custom fields, assignment rules, document groups, actions, automated planning |
| **knowledge-system** | Feature types, item types, knowledge sets — rule-based processing customization with DNF expressions |
| **service-bridge** | External API proxy endpoints — centralized authentication, header management, caching |
| **prompt-template** | LLM prompt configurations — extraction prompts, system prompts, knowledge-driven overrides |

## Installation

Install the plugin in Claude Code:

```bash
claude install kodexa-ai/kodexa-metadata-skills
```

After installation, the skills are automatically available in all Claude Code sessions.

## Usage

Skills are invoked automatically when Claude Code detects a matching task, or you can invoke them explicitly:

```
# Ask Claude Code to create metadata
> Create a data definition for extracting purchase order data

# Claude Code will automatically use the data-definition skill

# Or invoke directly via the Skill tool
> /kodexa-metadata-skills:data-definition
```

### Interactive Wizard

Each skill offers a guided wizard mode. When you ask to create metadata from scratch, the skill walks you through:

1. Purpose and use case questions
2. Field and component selection
3. Validation and business rule configuration
4. Complete YAML generation

### Reference Mode

When editing existing metadata or debugging issues, the skill loads the complete schema reference into context to assist with any modification.

## Skills Detail

### Tier 1 — Core

**data-definition** — Author taxonomy YAML for document extraction. Covers all 12 data types (STRING, NUMBER, CURRENCY, DATE, SELECTION, etc.), value paths, validation formula functions, conditional formatting, repeating groups, and semantic definitions.

**project-template** — Author complete project blueprint YAML. Covers all 9 component collections (stores, assistants, taxonomies, data forms, workspaces, task templates, scheduled jobs, knowledge sets, status configs), template variables, and project options.

**module** — Author module YAML for both model (Python) and skill (file pack) types. Covers runtime configuration, sidecar dependencies, inference options, deployment defaults, Python entry point patterns, and magic parameter injection.

### Tier 2 — Project Components

**data-form** — Author V2 schema-driven forms in JSON. Covers UINode component tree, layout/display/data components, Bridge API namespaces (kodexa.data, kodexa.navigation, kodexa.form, kodexa.http, kodexa.log), event handling, and scripting.

**assistant** — Author assistant pipeline YAML. Covers connection types (STORE, CHANNEL, DOCUMENT_FAMILY, WORKSPACE), subscription expressions, multi-step pipeline chaining, extension pack structure, and Python implementation.

**task-template** — Author task template YAML. Covers custom fields, assignment rules, document groups, action buttons with status transitions, custom options, and planned activity configuration.

### Tier 3 — Advanced

**knowledge-system** — Author knowledge feature types, item types, and knowledge sets as a coordinated set. Covers DNF expression logic, immutability constraints, common patterns (vendor, document type, language), and feature-to-item wiring.

**service-bridge** — Author service bridge YAML for external API integration. Covers authentication patterns (API key, Bearer, Basic), endpoint mapping, caching configuration, and usage from modules and forms.

**prompt-template** — Author LLM prompt templates. Covers standalone YAML prompts and skill module embedded prompts, variable placeholders, output format configuration, and integration with the knowledge system for per-vendor prompt overrides.

## Documentation

Full Kodexa platform documentation is available at [developer.kodexa.ai](https://developer.kodexa.ai).

## License

This project is licensed under the Apache License 2.0 — see the [LICENSE](LICENSE) file for details.
