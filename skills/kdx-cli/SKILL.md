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
kdx config set-profile dev2 --url https://dev2.kodexacloud.com --api-key <key>

# Mark a profile as production (enables safety confirmations)
kdx config set-profile prod --url https://prod.kodexa.com --api-key <key> --production

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

# Get a specific resource by name
kdx get workspace my-workspace

# Output as JSON or YAML
kdx get projects -o json
kdx get modules -o yaml

# Apply a named filter from configuration
kdx get projects --filter-name my-filter
```

Resource types are dynamically discovered — use `kdx api-resources` to see all available types.

---

### describe — Resource Details

```bash
# Show detailed info about a resource
kdx describe workspace my-workspace
kdx describe project my-project
kdx describe module kodexa/my-module
```

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

### run — Execute API Operations

Discover and invoke additional API operations beyond standard CRUD.

```bash
# List available operations for a resource type
kdx run projects

# Execute an operation
kdx run projects test --projectId 123 --body '{"hello":"world"}'
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

## Sync & Deploy Workflows

The `sync` command family manages metadata between local Git repos and platform environments using a `sync-config.yaml` configuration file.

### sync-config.yaml Structure

```yaml
schema_version: "2.0"

# Define available Kodexa environments
environments:
  dev:
    url: https://dev.kodexacloud.com
    api_key_env: DEV_TOKEN           # Environment variable name for API key
    profile: dev                      # Profile name to use
  staging:
    url: https://staging.kodexacloud.com
    api_key_env: STAGING_TOKEN
    profile: staging
  production:
    url: https://prod.kodexa.com
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
  # Group by resource type
  data-form:
    - forms/my-form.yml
  taxonomy:
    - taxonomies/my-taxonomy.yaml
  project-template:
    - project-templates/my-project.yml

# Modules (separate section — triggers code upload)
modules:
  - models/my-model/model.yml
  - models/my-other-model/model.yml
```

### sync pull — Download Metadata

```bash
# Pull from default environment
kdx sync pull

# Pull a specific target
kdx sync pull --target my-org-dev

# Pull from a specific environment
kdx sync pull --env dev

# Override source connection
kdx sync pull --from-profile staging
kdx sync pull --from-url https://staging.kodexacloud.com --from-api-key <key>

# Custom config location
kdx sync pull --config ./my-sync-config.yaml --metadata-dir ./metadata
```

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
kdx sync push --to-url https://prod.kodexa.com --to-api-key <key>
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
- Use `--threads` for parallel resource deployment
- The `--output-json` flag produces a machine-readable deployment report for CI/CD pipelines

---

## Common Workflows

### Initial Setup

```bash
# 1. Set up profiles for your environments
kdx config set-profile dev --url https://dev.kodexacloud.com --api-key $DEV_TOKEN
kdx config set-profile prod --url https://prod.kodexa.com --api-key $PROD_TOKEN --production

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
| Reprocess does nothing | `--assistant-id` is required — without it documents get stuck in pending |
| Reprocess no matches | Check filter syntax; use `--dry-run` to debug |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `-f` flag on apply | Must be `kdx apply -f file.yaml`, not `kdx apply file.yaml` |
| Using wrong resource type name | Resource names are plural for listing (`kdx get modules`), singular for specific (`kdx get module my-mod`) |
| Missing `schema_version: "2.0"` in sync-config | Required field — add it at the top of sync-config.yaml |
| Missing `manifest_version: "1.0"` in manifest | Required field — add it at the top of manifest files |
| Relative paths in manifest wrong | Paths are relative to the manifest file location |
| Deploying to wrong env | Always `--dry-run` first; check `kdx config current-profile` |
| Using `--force` delete in production | Skips all safety checks — prefer removing `--force` and confirming manually |
| Reprocessing without `--assistant-id` | Always required — server won't schedule executions without it |
| Wrong store ref format | Must be `org/slug:version`, not `org/slug` or `org:slug:version` |
