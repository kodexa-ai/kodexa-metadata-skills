# Contributing

These skills describe how to author Kodexa platform metadata. They are only useful if they
match the platform. This file records where truth lives, the house rules that keep the skills
publishable, and how to check a change before opening a PR.

## Where truth lives

**The platform source is the only source of truth.** Not this repo, not the developer site, and
not another skill that already says the thing you are about to repeat.

| Question | Answer it from |
|---|---|
| What fields does this resource have? | The GORM model's `json:` tags |
| What values does this enum accept? | The Go constants, plus any validation that rejects at create |
| What endpoint, verb and status code? | The API handler |
| What command, subcommand or flag? | The `kdx` cobra command definitions |
| What column, default or constraint? | The SQL migration |
| What does the runtime actually do with this? | The orchestrator, or the UI renderer |

The generated OpenAPI spec is a good cross-check for wire field names, but it is generated —
when it disagrees with the Go struct, the struct wins.

### Never carry a claim forward

Most errors found in the 2026-08 audit got there by being copied from an earlier version of the
same document, or from a sibling document, without anyone re-checking the code. A statement
already written down is not evidence. If you cannot find it in the platform source today, delete
it rather than preserve it.

The audit's highest-yield check was mechanical: take every fenced YAML example and confirm each
top-level key exists on the target model. That alone catches most of what goes wrong here.

## House rules

**1. This repository is public.** Never commit:

- customer names or customer-specific jargon — including domain vocabulary that identifies one
  engagement rather than a general concept
- internal environments, hostnames, tailnet names or deployment tooling
- internal ticket identifiers
- paths into private or internal repositories, or references to internal design documents by
  section number — a reader cannot open them, and a rule whose justification is unreachable
  reads as arbitrary. State the behaviour directly instead.
- MCP tool names. The audience here authors YAML and applies it with `kdx`.

Database table names (`kdxa_*`) are fine — they help an author debugging URI resolution, and
Kodexa publishes them on its own developer site. Go struct dumps, column types, index names and
constraint names are not: they are internal detail that goes stale on the next schema change.

**2. Use the fixed example cast.** Organizations `acme-corp`, `my-org`, `kodexa`. Companies Acme
and Globex. Subject matter: invoices, purchase orders, vendors, receipts, contracts. Hosts under
`*.kodexa.example.com` (RFC 2606). UUIDs of the form `00000000-0000-4000-8000-0000000000NN`.

Reaching for a real-world shape is the likeliest way to leak something. Reach for the cast instead.

Some examples are content-addressed — knowledge feature slugs are a hash of the feature's
canonical properties. If you change the properties in an example, recompute the hash and verify
it, or the example stops being self-consistent.

**3. Progressive disclosure.** `SKILL.md` stays at or under ~200 lines. Bulk field tables, enum
inventories, long worked examples and troubleshooting go in `references/*.md`.

The split is not by length, it is by consequence: **anything separating "works" from "silently
broken" belongs in `SKILL.md`**, never in a reference file. If a fact will be got wrong and fail
without an error message, it must be in the file that always loads.

**4. Every skill carries a "Declared but inert" section.** Several Kodexa resources persist and
round-trip fields that nothing reads — some are even editable in the UI. Authors meet these keys
in existing YAML and reasonably read silence as endorsement.

Do not simply delete such a field when you discover it is dead. Move it to that section with a
one-line note. If a domain genuinely has none, say so in one line.

**5. Frontmatter.**

```yaml
---
name: <must equal the directory name>
description: "Use when <trigger conditions> — <what the skill covers>"
---
```

The description is the only thing a model sees when deciding whether to load the skill. Write it
so it fires on the right tasks and not on adjacent ones.

## Checking a change

```bash
claude plugin validate . --strict
```

For Codex, validate `.codex-plugin/plugin.json` with `$plugin-creator` and run the
`$skill-creator` validator over every directory under `skills/`. The Codex plugin and skill
validators check different layers; run both.

Then, by hand:

- every changed factual claim traced to platform source
- every fenced example's top-level keys checked against the model
- `SKILL.md` still within budget, with the load-bearing facts inside it
- a grep for customer names, internal hosts, ticket ids and private repo paths

**Bump `version` in both `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json` in the same PR
as any change under `skills/`.** Keep the versions equal. Each field pins the package installed by
that host — a correct fix that never ships is not a fix.

The version lives in each host's `plugin.json`. Marketplace entries deliberately omit it so there
is no third place to forget.

## A known problem: this documentation exists more than once

At the time of the 2026-08 audit, three near-duplicate bodies of Kodexa metadata documentation
had drifted apart — this repository, a partial copy inside the platform monorepo, and a more
actively maintained set shipped in the Kodexa agent container. A fourth, the developer site,
carries overlapping material and is upstream of some of it.

They disagreed, and each was independently wrong about different things. Neither of the other
copies could simply be adopted here.

The durable fix is to make the platform monorepo the single canonical body — written public-safe
from the start, so publishing is a plain copy and `diff` against this repo is itself a drift
check — gated by checks that can only run where the source lives:

- a lint pass that **fails** rather than rewrites, over the denylist in house rule 1
- enum assertions against the Go constants
- backticked field names asserted to exist as `json:` tags, or listed on a documented allow-list
- every fenced example YAML unmarshalled into its model, asserting no top-level key is dropped

The last of these is worth more than the rest combined, and more than any review process, because
the failures here are enumerations, field names and YAML blocks — all machine-checkable.

Until that exists, this repository is maintained by hand, and the rules above are what stands in
for it.
