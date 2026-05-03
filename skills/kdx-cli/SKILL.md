---
name: kdx-cli
description: "Use when working with the Kodexa CLI (kdx) — kubectl-style commands for managing platform resources, profiles, sync/deploy workflows, and metadata repositories"
---

# Kodexa CLI (kdx)

## Overview

`kdx` is a kubectl-style CLI for the Kodexa AI Platform. It dynamically discovers resource types from the platform's OpenAPI spec and provides commands to list, describe, create, update, delete, and deploy resources. It also manages profiles (environment connections), metadata sync workflows, and Python dataclass generation from taxonomies.

## When to Use

- Running any `kdx` command to interact with the Kodexa platform
- Setting up or switching between platform profiles (dev, staging, production)
- Deploying metadata (modules, taxonomies, forms, project templates) to environments
- Pulling or pushing metadata between local repos and the platform
- Listing, describing, or deleting platform resources
- Generating Python dataclasses from taxonomy YAML
- Troubleshooting CLI connectivity, authentication, or sync issues

## Safety: Always Dry-Run First

**IMPORTANT:** When executing any command that supports `--dry-run` (such as `kdx store reprocess`, `kdx sync push`, or `kdx sync deploy`), you MUST run with `--dry-run` first to verify the filter/selection is correct before running the actual command. Review the dry-run output with the user and only proceed with the real execution after confirmation.

## Command Reference

### Global Flags

Every command accepts these flags:

```
--api-key string            API key (overrides profile)
--debug                     Enable debug logging (shorthand for --log-level=debug)
--log-level string          Log level: debug|info|warn|error|silent (default "info")
--log-timestamps            Include timestamps in log output
-o, --output string         Output format: table|json|yaml|markdown (default "table")
--profile string            Profile to use (overrides current profile)
--skip-production-confirm   Skip production confirmation prompts
--url string                API URL (overrides profile)
```

---

### config — Profile Management

Profiles store connection settings (URL + API key) for platform instances.

```bash
# Create or update a profile
kdx config set-profile dev2 --url https://dev2.kodexa.example.com --api-key <key>

# Mark a profile as production (enables safety confirmations)
kdx config set-profile prod --url https://prod.kodexa.example.com --api-key <key> --production

# List all profiles
kdx config list-profiles

# Show current active profile
kdx config current-profile

# Switch to a different profile
kdx config use-profile dev2

# Delete a profile
kdx config delete-profile old-env
```

**Key points:**
- Production profiles trigger confirmation prompts on destructive operations
- Use `--skip-production-confirm` to bypass (logs the bypass for audit)
- Profile can be overridden per-command with `--profile <name>` or `--url`/`--api-key`

---

### get — List Resources

```bash
# List all resources of a type
kdx get workspaces
kdx get projects
kdx get modules

# Get a specific resource by UUID
kdx get data-definition 00000000-0000-4000-8000-000000000001

# Output as JSON or YAML
kdx get projects -o json
kdx get modules -o yaml

# Apply a named preset filter from configuration (NOT an inline filter expression)
kdx get projects --filter-name my-filter
```

Resource types are dynamically discovered — use `kdx api-resources` to see all available types.

**Important:** `kdx get <resource> <name>` looks up by UUID, not slug. To filter by slug, name, org, or other fields use `kdx run` with the list operation and `--filter` (see below).

---

### describe — Resource Details

```bash
# Show detailed info about a resource
kdx describe workspace my-workspace
kdx describe project my-project
kdx describe module kodexa/my-module
```

**Note:** `describe` uses the same resource type names as `get` (e.g., `data-definition`, not `taxonomy`). Use `kdx api-resources` if unsure of the correct type name.

---

### apply — Create or Update Resources

```bash
# Apply a resource definition from YAML
kdx apply -f data-store.yaml
kdx apply -f taxonomy.yaml

# Apply a module (auto-detects implementation and uploads code)
kdx apply -f model.yml
```

**Key points:**
- For modules, `apply` automatically uploads the implementation code — no separate deploy step needed
- The YAML file must contain `type` and `slug` fields for resource identification
- Creates the resource if it doesn't exist, updates if it does

---

### delete — Remove Resources

```bash
# Delete a resource
kdx delete workspace my-workspace

# Force delete (skip confirmation)
kdx delete project my-project --force
```

---

### run — Execute API Operations & Filtered Lookups

Discover and invoke additional API operations beyond standard CRUD. **This is also the way to filter resources by slug, name, org, or other fields** — `kdx get` does not support inline filters. Some resource types (like knowledge sets) only support operations via `kdx run`, not `kdx get`.

```bash
# List available operations for a resource type
kdx run projects

# Execute an operation
kdx run projects test --projectId 123 --body '{"hello":"world"}'
```

#### Filtered Resource Lookups (via `kdx run`)

Most resource types expose a `list-*` operation that supports `--filter` with SpringFilter DSL syntax. Use this instead of `kdx get` when you need to find a specific resource by slug, name, or org.

```bash
# Data definitions (taxonomies)
kdx run data-definitions list-taxonomies --filter "slug:'my-taxonomy-slug'" -o yaml
kdx run data-definitions list-taxonomies --filter "name:'My Taxonomy Name'" -o yaml
kdx run data-definitions list-taxonomies --filter "orgSlug:'my-org'" -o yaml

# Data forms
kdx run data-forms list-data-forms --filter "slug:'my-form-slug'" -o yaml
kdx run data-forms list-data-forms --filter "name:'My Form Name'" -o yaml

# Data stores
kdx run data-stores list-data-stores --filter "slug:'my-store-slug'" -o yaml

# Modules
kdx run modules list-modules --filter "slug:'my-module'" -o yaml
kdx run modules list-modules --filter "orgSlug:'my-org'" -o yaml

# Combine filters with 'and'
kdx run data-definitions list-taxonomies --filter "orgSlug:'my-org' and name~'*Invoice*'" -o yaml

# Pagination and sorting
kdx run data-definitions list-taxonomies --filter "orgSlug:'my-org'" --pageSize 50 --sort "name:asc" -o yaml
```

**Filter operators:** `:` (equals), `!` (not equal), `~` (like/wildcard), `>`, `<`, `>=`, `<=`, `and`, `or`, `not()`.

**Operation naming:** The operation names don't always match the resource type name. For example, `data-definitions` uses `list-taxonomies` (not `list-data-definitions`). Always run `kdx run <resource-type>` with no operation name to discover the correct operation names and their parameters.

#### Knowledge Sets (via `kdx run`)

**Important:** Knowledge sets must be accessed via `kdx run knowledge-sets`, NOT `kdx get knowledge-sets` (the list operation is not available on the get endpoint).

```bash
# List all knowledge sets
kdx run knowledge-sets list-knowledge-set

# Filter by organization
kdx run knowledge-sets list-knowledge-set --orgSlug my-org

# Filter by project (name or slug)
kdx run knowledge-sets list-knowledge-set --filter "project.slug:'my-project'"
kdx run knowledge-sets list-knowledge-set --filter "project.name:'My Project Name'"

# Filter by knowledge set slug (useful for exporting a single set)
kdx run knowledge-sets list-knowledge-set --orgSlug my-org --filter "slug:'my-knowledge-set-slug'"

# Filter by name with wildcards
kdx run knowledge-sets list-knowledge-set --orgSlug my-org --filter "name~'*Water*'"

# Combine org and project filters
kdx run knowledge-sets list-knowledge-set --orgSlug my-org --filter "project.slug:'my-project'"

# Export to YAML
kdx run knowledge-sets list-knowledge-set --orgSlug my-org --filter "slug:'my-slug'" -o yaml > my-knowledge-set.yaml

# Get a single knowledge set by ID
kdx run knowledge-sets get-knowledge-set --id <uuid> -o yaml > file.yaml

# Use a specific profile
kdx run knowledge-sets list-knowledge-set --profile my-prod
```

**Knowledge item types** (organization-level, accessed via `kdx get`):

```bash
# List item types
kdx get knowledge-item-types
kdx get knowledge-item-types -o yaml > item-types.yaml
```

---

### api-resources — Discover Resource Types

```bash
# List all available resource types
kdx api-resources

# Show raw schemes from server
kdx api-resources --schemes-only

# Refresh the resource cache
kdx api-resources --refresh
```

---

### dataclasses — Generate Python Code

Generate Python dataclasses from a taxonomy definition file.

```bash
# Generate from taxonomy YAML
kdx dataclasses taxonomy.yaml

# Specify output file and path
kdx dataclasses taxonomy.yaml --output-file my_classes.py --output-path ./src
```

The generated classes mirror the taxonomy structure and work with Kodexa's LLM tooling.

---

### completion — Shell Autocompletion

```bash
# Generate for your shell
kdx completion bash
kdx completion zsh
kdx completion fish
kdx completion powershell
```

---

## Store Commands

The `kdx store` command provides extended operations for document stores. These are plugin commands (not `kdx run` operations).

### store reprocess — Reprocess Document Families

Reprocess document families in a store, filtering by status or other criteria. Requires an assistant ID to specify which assistant handles reprocessing.

```bash
kdx store reprocess <store-ref> --assistant-id <id> [flags]
```

**Store ref format:** `org/store-slug:version` (e.g., `my-org/my-store:1.0.0`)

| Flag | Default | Description |
|------|---------|-------------|
| `--assistant-id` | *(required)* | Assistant ID to use for reprocessing |
| `--filter` | *(required)* | Filter expression for selecting document families |
| `--dry-run` | `false` | Preview matching documents without triggering reprocessing |
| `--page-size` | `100` | Page size when fetching document families |

**Common filters:**

| Filter | Description |
|--------|-------------|
| `statistics.recentExecutions.execution.status: 'FAILED'` | Failed documents |
| `pendingProcessing:true` | Documents stuck in pending state |

```bash
# Reprocess all failed documents
kdx store reprocess my-org/my-store:1.0.0 \
  --assistant-id abc123 \
  --filter "statistics.recentExecutions.execution.status: 'FAILED'"

# Dry run — preview what would be reprocessed
kdx store reprocess my-org/my-store:1.0.0 \
  --assistant-id abc123 \
  --filter "statistics.recentExecutions.execution.status: 'FAILED'" \
  --dry-run

# Reprocess pending documents
kdx store reprocess my-org/my-store:1.0.0 \
  --assistant-id abc123 \
  --filter "pendingProcessing:true"
```

**Typical reprocess workflow:**

```bash
# 1. Find the assistant ID
kdx get assistants -o json

# 2. Preview failed documents (dry run)
kdx store reprocess my-org/my-store:1.0.0 \
  --assistant-id abc123 \
  --filter "statistics.recentExecutions.execution.status: 'FAILED'" \
  --dry-run

# 3. Trigger reprocessing
kdx store reprocess my-org/my-store:1.0.0 \
  --assistant-id abc123 \
  --filter "statistics.recentExecutions.execution.status: 'FAILED'"

# 4. Monitor progress
kdx store watch <document-family-id>
```

**Important:** `--assistant-id` is required. Without it, the server marks documents as pending but never schedules executions, leaving them stuck.

### store upload — Upload Files

```bash
# Upload a file to a store
kdx store upload my-org/my-store:1.0.0 invoice.pdf
```

### store watch — Monitor Processing

```bash
# Watch a document family's processing progress
kdx store watch <document-family-id>
```

### store stats — Store Statistics

```bash
# Show store statistics
kdx store stats my-org/my-store:1.0.0
```

### store reindex — Trigger Reindexing

```bash
# Reindex a store
kdx store reindex my-org/my-store:1.0.0
```

---

## Secret Commands

The `kdx secret` plugin manages organization secrets. The values are passed via prompt or stdin only — they never appear on the command line, in shell history, or in process listings. The underlying API (`/api/organizations/{orgId}/secrets`) is intentionally not in the public OpenAPI spec.

### secret list — List Secret Names

```bash
kdx secret list <org-slug>
```

Returns the names only. Values are never returned over the API.

### secret set — Set a Secret Value

```bash
# Interactive — terminal prompts for the value (hidden input)
kdx secret set <org-slug> <name>

# Non-interactive — read value from stdin
echo -n "<value>" | kdx secret set <org-slug> <name> --stdin
```

The value is sent to the server but never logged or echoed. Existing names are overwritten.

### secret delete — Remove a Secret

```bash
kdx secret delete <org-slug> <name>
```

`<org-slug>` may be a slug *or* a UUID. Slugs resolve via `filter=slug:'...'` (exact match — the `query=` endpoint searches display names and would miss orgs whose slug differs from the display name).

---

## Task Commands

The `kdx task` command provides extended operations for tasks.

> **Activity refactor compatibility note (2026-05-02).** The platform's `Plan` concept has been renamed to `Activity` (backing tables: `kdxa_plans` → `kdxa_activities`, `kdxa_planned_items` → `kdxa_steps`), and the `/api/plans` back-compat shim has been removed. The `kdx task failures` and `kdx task reprocess` commands still surface "plan" terminology and call legacy paths — confirm against the installed CLI version before relying on them on a post-refactor server. If those commands fail, use the new `/api/activities` endpoints directly via `kdx run`.

### task failures — List Tasks with Failed Plans

Find tasks whose plans contain a FAILED step, showing the error for quick diagnosis.

```bash
kdx task failures --project <project-slug> [flags]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--project` | | Project slug (shorthand for `project.slug:'...'` filter) |
| `--filter` | | Additional SpringFilter expression |
| `--since` | `24h` | How far back to look (e.g., `2h`, `30m`, `1d`, `7d`) |
| `--page-size` | `100` | Page size when fetching tasks |

At least one of `--project` or `--filter` is required.

```bash
# Show failures in a project in the last 2 hours
kdx task failures --project invoice-processing --since 2h

# Show failures in the last 7 days
kdx task failures --project invoice-processing --since 7d

# Filter by template
kdx task failures --project invoice-processing --filter "template.name:'Invoice Document Pack'"
```

Output shows a table with TITLE, TEMPLATE, PLAN CREATED, FAILED STEP, and ERROR columns.

### task reprocess — Reprocess Failed Plans

Reprocess one or more failed plans by restarting them from the beginning via the orchestrator.

```bash
# Single plan
kdx task reprocess <plan-id>

# Bulk — reprocess all failed plans matching filters
kdx task reprocess --project <project-slug> --since <duration> [flags]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--project` | | Project slug for bulk mode |
| `--filter` | | Additional SpringFilter expression |
| `--since` | `24h` | Time window for bulk mode (e.g., `2h`, `30m`, `1d`) |
| `--dry-run` | `false` | Preview which plans would be reprocessed |
| `--page-size` | `100` | Page size when fetching tasks |

```bash
# Preview what would be reprocessed
kdx task reprocess --project invoice-processing --since 2h --dry-run

# Reprocess all failed plans in the last 2 hours
kdx task reprocess --project invoice-processing --since 2h

# Reprocess a single plan by ID
kdx task reprocess 00000000-0000-4000-8000-000000000002
```

**Typical workflow:**

```bash
# 1. Find failures
kdx task failures --project invoice-processing --since 2h

# 2. Preview reprocessing (dry run)
kdx task reprocess --project invoice-processing --since 2h --dry-run

# 3. Reprocess
kdx task reprocess --project invoice-processing --since 2h
```

---

## Document-Family Commands

The `kdx document-family` command provides operations for working with document families.

### document-family data — Extract Data

Retrieve extracted/transformed data from a document family in JSON format.

```bash
kdx document-family data <document-family-id> [flags]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--output, -o` | stdout | Output file path |
| `--include-ids` | `true` | Include element IDs in output |
| `--friendly-names` | `false` | Use friendly names for fields |
| `--inline-audits` | `false` | Include inline audit information |
| `--include-exceptions` | `false` | Include exception information |

```bash
# Print extracted data to stdout
kdx document-family data abc123

# Save to file with friendly names
kdx document-family data abc123 --friendly-names --output extracted.json
```

### document-family content — Content Object Operations

```bash
# List content objects in a document family
kdx document-family content list <document-family-id>

# Download a specific content object
kdx document-family content download <document-family-id> <content-object-id> --output doc.kddb

# Download the latest content object
kdx document-family content download <document-family-id> --latest --output doc.kddb
```

### document-family add-label / remove-label / set-status

```bash
# Add a label to a document family
kdx document-family add-label <document-family-id> <tag-id>
kdx document-family add-label <document-family-id> <tag-id> --path /some/path

# Remove a label
kdx document-family remove-label <document-family-id> <label-id>

# Set document status
kdx document-family set-status <document-family-id> <status-id>
```

---

## Sync & Deploy Workflows

The `sync` command family manages metadata between local Git repos and platform environments using a `sync-config.yaml` configuration file.

### sync-config.yaml Structure

```yaml
schema_version: "2.0"

# Define available Kodexa environments
environments:
  dev:
    url: https://dev.kodexa.example.com
    api_key_env: DEV_TOKEN           # Environment variable name for API key
    profile: dev                      # Profile name to use
  staging:
    url: https://staging.kodexa.example.com
    api_key_env: STAGING_TOKEN
    profile: staging
  production:
    url: https://prod.kodexa.example.com
    api_key_env: PROD_TOKEN
    profile: prod

# Define deployment targets (organization + manifests)
targets:
  my-org-dev:
    organization: my-org-dev
    manifests:
      - resources-manifest.yaml       # Path to manifest file
  my-org-prod:
    organization: my-org-prod
    manifests:
      - resources-manifest.yaml

# Automatic deployment based on Git branch
branch_mappings:
  - pattern: "main"
    targets:
      - target: my-org-dev
        environment: dev
      - target: my-org-dev
        environment: staging
  - pattern: "production"
    targets:
      - target: my-org-prod
        environment: production
```

### Resource Manifest Structure

Manifests define which resources and modules to deploy:

```yaml
manifest_version: "1.0"

# Directory containing resource definitions
metadata_dir: resources

resources:
  # Group by resource type — see "Syncable Resource Types" below
  data-form:
    - forms/my-form.yml
  taxonomy:
    - taxonomies/my-taxonomy.yaml
  project-template:
    - project-templates/my-project.yml
  activity-plan:                    # Org-scoped (push order 50)
    - activity-plans/invoice-flow.yaml
  task-template:                    # Org-scoped (push order 65)
    - task-templates/invoice-review.yaml
  task-status:                      # Org-scoped (push order 65)
    - task-statuses/todo.yaml
    - task-statuses/in-review.yaml
  trigger:                          # Project-scoped (push order 75)
    - triggers/auto-review.yaml

# Modules (separate section — triggers code upload)
modules:
  - models/my-model/model.yml
  - models/my-other-model/model.yml
```

#### Syncable Resource Types (post-activity-refactor)

| Canonical name | URI scheme | Scope | FilesystemDir | Push order | Notes |
|---|---|---|---|---|---|
| `module` | `module://` | org | `modules/` | 10 | Plus `model`, `model-runtime` aliases |
| `taxonomy` | `taxonomy://` | org | `taxonomies/` | 20 | Aka `data-definition` |
| `data-store` | `data-store://` | org | `data-stores/` | 30 | |
| `data-form` | `data-form://` | org | `data-forms/` | 40 | |
| `service-bridge` | `service-bridge://` | org | `service-bridges/` | 45 | |
| `prompt-template` | `prompt-template://` | org | `prompt-templates/` | 45 | |
| `knowledge-set` | `knowledge-set://` | org | `knowledge-sets/` | 45 | |
| `activity-plan` | `activity-plan://` | **org** | `activity-plans/` | **50** | **NEW** — replaces inline `planTemplate` |
| `project-template` | (no scheme) | org | `project-templates/` | 60 | |
| `task-template` | `task-template://` | **org** | `task-templates/` | **65** | **Flipped from project to org scope** |
| `task-status` | `task-status://` | **org** | `task-statuses/` | **65** | **Flipped from project to org scope** |
| `assistant` | `assistant://` | project | `assistants/` | 70 | |
| `trigger` | `trigger://` | **project** | `triggers/` | **75** | **NEW** — references an `activity-plan://` URI |

> Push order matters: lower numbers push first. Triggers (75) are pushed after the activity-plans (50) they reference and the projects (60) / assistants (70) they live in.

### sync pull — Download Metadata

```bash
# Pull from default environment
kdx sync pull

# Pull a specific target
kdx sync pull --target my-org-dev

# Pull from a specific environment
kdx sync pull --env dev

# Filter resources by slug (regex)
kdx sync pull --filter "knowledge.*liability"

# Override source connection
kdx sync pull --from-profile staging
kdx sync pull --from-url https://staging.kodexa.example.com --from-api-key <key>

# Custom config location
kdx sync pull --config ./my-sync-config.yaml --metadata-dir ./metadata
```

#### Discover Mode

Use `--discover` to auto-generate a manifest from the platform. When an existing manifest is found, new resources are merged in without overwriting existing entries.

```bash
# Discover resources and generate manifest
kdx sync pull --discover

# Set the metadata directory in the generated manifest
kdx sync pull --discover --discover-dir resources
```

**Smart manifest merge behavior:**
- Newly discovered resources are appended without overwriting existing entries
- Commented-out slugs (`# - slug`) are treated as intentional exclusions and preserved
- Commented-out type headers (`# type-name:`) exclude the entire type from discovery
- Warnings are shown for manifest entries not found on the server
- Comments and formatting are preserved

### sync push — Upload Metadata

```bash
# Push to default environment
kdx sync push

# Push a specific target
kdx sync push --target my-org-dev

# Push to a specific environment
kdx sync push --env staging

# Dry run — show what would happen
kdx sync push --dry-run

# Filter resources by slug (regex)
kdx sync push --filter "my-module.*"

# Override destination connection
kdx sync push --to-profile prod
kdx sync push --to-url https://prod.kodexa.example.com --to-api-key <key>
```

### sync deploy — GitOps Deployment

Deploy resources using Git branch mappings or explicit targets.

```bash
# Deploy based on current Git branch (uses branch_mappings)
kdx sync deploy

# Deploy to a specific environment
kdx sync deploy --env staging

# Deploy a specific target
kdx sync deploy --target my-org-dev --env dev

# Override branch detection
kdx sync deploy --branch main

# Dry run
kdx sync deploy --dry-run

# Force push (override server conflicts)
kdx sync deploy --force

# Filter resources
kdx sync deploy --filter "taxonomy.*"

# Parallel deployment
kdx sync deploy --threads 4

# Confirmation modes
kdx sync deploy --confirm-all      # One confirmation for all targets
kdx sync deploy --confirm-each     # Confirm each target individually

# CI/CD integration — write deployment report
kdx sync deploy --output-json deploy-report.json

# Tag-based deployment
kdx sync deploy --tag v1.2.0
```

**Key points:**
- Branch mappings in `sync-config.yaml` determine which targets/environments to deploy to
- Production environments trigger safety confirmations
- Use `--dry-run` to preview changes before deploying
- Use `--force` to push even when the server has changed since last pull (overwrites conflicts)
- Use `--threads` for parallel resource deployment
- The `--output-json` flag produces a machine-readable deployment report for CI/CD pipelines

### Portable YAML — `${org}` Placeholder

Pulled YAML files use `${org}` as a portable placeholder for the organization slug. This enables reuse of sync repositories across different organizations.

- **During pull:** The actual org slug (e.g., `acme-corp`) is replaced with `${org}` in cross-org references
- **During push/deploy:** `${org}` is resolved back to the target organization slug

No flags needed — this is automatic.

### Resource Reference Sigils

Project-templates and assistants can reference other org-level resources by slug. These sigils resolve to UUIDs at push/deploy time so the final payload is environment-agnostic:

| Sigil | Resolves to |
|---|---|
| `${taskTemplate.<slug>}` | UUID of the org's `task-template` with the given slug |
| `${activityPlan.<slug>}` | UUID of the org's `activity-plan` with the given slug |
| `${taskStatus.<slug>}` | UUID of the org's `task-status` with the given slug |

The CLI's resolver looks up the resource via the platform's `/api/resolve?path=<scheme>://<orgSlug>/<slug>` endpoint before substituting. Unresolved sigils fail the push with a clear error pointing at the missing resource.

### Knowledge Attachments

Knowledge set and knowledge item attachments are automatically downloaded during `sync pull`. Attachments are saved to `<slug>-attachments/` directories alongside the YAML files.

- Uses SHA-256 hash comparison to skip unchanged files
- Injects `attachmentPath` and `attachmentId` directives into YAML for round-trip push
- Uploads and downloads are parallelized for performance

### Module Slug Shorthand in Manifests

Modules can be listed by bare slug in manifest files instead of full path:

```yaml
modules:
  - transformer                    # resolves to modules/transformer.yaml
  - models/my-model/model.yml     # explicit path also works
```

The CLI resolves through multiple formats: `<slug>`, `<slug>.yaml`, `<slug>.yml`, `modules/<slug>.yaml`, `modules/<slug>.yml`.

---

## Common Workflows

### Initial Setup

```bash
# 1. Set up profiles for your environments
kdx config set-profile dev --url https://dev.kodexa.example.com --api-key $DEV_TOKEN
kdx config set-profile prod --url https://prod.kodexa.example.com --api-key $PROD_TOKEN --production

# 2. Verify connection
kdx config current-profile
kdx get projects

# 3. Discover available resource types
kdx api-resources
```

### Develop & Deploy a Module

```bash
# 1. Apply the module (uploads code + metadata)
kdx apply -f model.yml

# 2. Check it's deployed
kdx get modules
kdx describe module my-org/my-module

# 3. Deploy to staging via sync
kdx sync deploy --env staging --target my-org
```

### Pull Remote Changes Locally

```bash
# Pull all metadata from dev environment
kdx sync pull --env dev --target my-org-dev

# Review changes in Git
git diff
```

### Diagnose and Reprocess Failed Task Plans

```bash
# 1. Find failures in a project
kdx task failures --project invoice-processing --since 2h

# 2. Preview what would be reprocessed (dry run)
kdx task reprocess --project invoice-processing --since 2h --dry-run

# 3. Reprocess all failed plans
kdx task reprocess --project invoice-processing --since 2h

# 4. Or reprocess a single plan by ID
kdx task reprocess 00000000-0000-4000-8000-000000000002
```

### Reprocess Failed Documents

```bash
# 1. Find the store and assistant
kdx get data-stores -o json
kdx get assistants -o json

# 2. Preview what failed
kdx store reprocess my-org/my-store:1.0.0 \
  --assistant-id abc123 \
  --filter "statistics.recentExecutions.execution.status: 'FAILED'" \
  --dry-run

# 3. Reprocess
kdx store reprocess my-org/my-store:1.0.0 \
  --assistant-id abc123 \
  --filter "statistics.recentExecutions.execution.status: 'FAILED'"

# 4. Watch progress
kdx store watch <family-id>
```

### Export & Import Knowledge Sets Between Environments

Knowledge sets have dependencies that must be applied in order:

```
1. Knowledge feature types   (org-level, no dependencies, immutable once created)
2. Knowledge item types      (org-level, no dependencies)
3. Knowledge set             (includes features, items, and clauses)
```

```bash
# 1. Export from source environment
kdx config use-profile my-uat
kdx run knowledge-sets list-knowledge-set \
  --orgSlug my-org \
  --filter "slug:'my-knowledge-set'" \
  -o yaml > my-knowledge-set.yaml

# 2. Export dependencies (item types)
kdx get knowledge-item-types -o yaml > item-types.yaml

# 3. Apply to target environment (in dependency order)
kdx config use-profile my-prod
kdx apply -f item-types.yaml          # Item types first
kdx apply -f my-knowledge-set.yaml    # Knowledge set second

# 4. Verify
kdx run knowledge-sets list-knowledge-set --profile my-prod
```

**Moving to a different project:** Edit the exported YAML to update the `project` section (id and name) and `organization` section. Remove server-generated fields (`id`, `changeSequence`, `createdOn`, `updatedOn`, `searchText`) from the top level and from each nested feature/item before applying.

**Tip:** Find the target project ID with `kdx get projects -o yaml`.

### CI/CD Pipeline Deploy

```bash
# In CI: deploy based on branch, with JSON report
kdx sync deploy --dry-run                        # Preview first
kdx sync deploy --output-json deploy-report.json  # Then deploy
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `connection refused` / timeout | Check `--url` or profile URL is correct; verify network access |
| `401 Unauthorized` | API key expired or wrong — run `kdx config set-profile` to update |
| `resource type not found` | Run `kdx api-resources --refresh` to update the cache |
| `no sync-config.yaml found` | Create one in your repo root or use `--config` to specify path |
| `no metadata directory found` | Use `--metadata-dir` flag or ensure `metadata_dir` is set in manifest |
| Profile not switching | Run `kdx config use-profile <name>` — verify with `kdx config current-profile` |
| Deploy doing nothing | Check `branch_mappings` pattern matches current branch; use `--branch` to override |
| Production confirmation blocking CI | Use `--skip-production-confirm` (logs bypass for audit) |
| Module code not uploading | Ensure `metadata.contents` globs in module YAML match your files |
| Stale resource types | Run `kdx api-resources --refresh` to re-fetch from server |
| `list operation not available` for knowledge sets | Use `kdx run knowledge-sets list-knowledge-set` instead of `kdx get knowledge-sets` |
| Knowledge set apply fails with FK violation | Apply dependencies first: feature types, then item types, then the knowledge set |
| Reprocess does nothing | `--assistant-id` is required — without it documents get stuck in pending |
| Reprocess no matches | Check filter syntax; use `--dry-run` to debug |
| `kdx get` returns all resources, need just one | Use `kdx run <resource> list-<operation> --filter "slug:'my-slug'"` for filtered lookups |
| Task failures not showing | Check `--since` window is wide enough; use `--since 7d` to search further back |
| Task reprocess returns 404 | Verify the plan ID (not task ID) — get it from `kdx task failures` output |
| Sync pull missing attachments | Upgrade CLI — attachment download was added recently |
| Sync deploy conflicts | Use `--force` to override server conflicts, or pull first to reconcile |
| `${org}` in YAML not resolving | Ensure sync push/deploy target has a valid organization configured |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `-f` flag on apply | Must be `kdx apply -f file.yaml`, not `kdx apply file.yaml` |
| Using `--filter` with `kdx get` | `kdx get` has no `--filter` flag — use `kdx run <resource> list-<op> --filter` instead |
| Trying to `kdx get` a resource by slug | `kdx get <resource> <name>` expects UUID, not slug — use `kdx run` with `--filter "slug:'...'"` for slug-based lookups |
| Using `kdx get` for knowledge sets | Knowledge sets require `kdx run knowledge-sets list-knowledge-set`, not `kdx get knowledge-sets` |
| Using wrong resource type name | Resource names are plural for listing (`kdx get modules`), singular for specific (`kdx get module <uuid>`) |
| Using `taxonomy` instead of `data-definition` | The resource type is `data-definition`/`data-definitions`, not `taxonomy`. Similarly use `data-form`, not `form`. Check `kdx api-resources` for correct names |
| Using `--filter-name` for inline filters | `--filter-name` looks up a named preset from config, not an inline filter expression — use `kdx run` with `--filter` instead |
| Wrong `kdx run` operation name | Operation names don't always match the resource type — e.g., `data-definitions` uses `list-taxonomies`. Run `kdx run <resource>` to discover operation names |
| Applying knowledge set before its dependencies | Feature types and item types must exist in the target env before applying a knowledge set that references them |
| Missing `schema_version: "2.0"` in sync-config | Required field — add it at the top of sync-config.yaml |
| Missing `manifest_version: "1.0"` in manifest | Required field — add it at the top of manifest files |
| Relative paths in manifest wrong | Paths are relative to the manifest file location |
| Deploying to wrong env | Always `--dry-run` first; check `kdx config current-profile` |
| Using `--force` delete in production | Skips all safety checks — prefer removing `--force` and confirming manually |
| Reprocessing without `--assistant-id` | Always required — server won't schedule executions without it |
| Wrong store ref format | Must be `org/slug:version`, not `org/slug` or `org:slug:version` |
| Using task ID instead of plan ID for reprocess | `kdx task reprocess` takes a plan ID, not a task ID — use `kdx task failures` to find plan IDs |
| Sync pull `--filter` not working | `--filter` uses regex against slug, not SpringFilter DSL — e.g., `--filter "knowledge.*"` |
| Expecting `--discover` to overwrite manifest | Discovery merges into existing manifests — comment out entries to exclude them |
