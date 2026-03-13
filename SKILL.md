---
name: pckle
description: Manage Pinecone Knowledge Agents and workflows via the pckle CLI. Use when the user mentions "knowledge agent", "Pinecone Knowledge Agent", "pckle", or wants to create, list, run, monitor, or cancel workflows or agents. Recognizes requests like "run a workflow", "create a knowledge agent", "check workflow status", "list my agents", "cancel a workflow", "search my knowledge base", or "run a pckle task".
license: SSPL-1.0
compatibility: Requires the pckle binary in PATH and network access to a PCKLE API instance.
metadata:
  author: pinecone-io
  version: "0.1.0"
allowed-tools: Bash(pckle:*)
---

# PCKLE CLI Skill

PCKLE powers **Pinecone Knowledge Agents** — also referred to as just "Knowledge Agents". Use the `pckle` CLI to interact with the PCKLE API. The CLI manages **agents** (reusable configuration templates with instructions, setup scripts, and a Pinecone index) and **workflows** (AI-powered execution units that run inside agents).

This skill should be activated whenever the user mentions:
- "Knowledge Agent" or "Pinecone Knowledge Agent"
- "pckle" or "PCKLE"
- Creating, running, or managing agents or workflows against a Pinecone index

## Prerequisites

1. The `pckle` binary must be in PATH
2. Authenticate before any other command:

```bash
pckle login --api-key <PINECONE_API_KEY>
```

Or set `PINECONE_API_KEY` as an environment variable:

```bash
export PINECONE_API_KEY=pcsk_...
pckle login
```

You can also override the API URL:

```bash
pckle --api-url https://my-pckle-server.example.com login --api-key <KEY>
```

## Global Flags

| Flag | Description |
|------|-------------|
| `--json` | Output machine-readable JSON (always use this when parsing output) |
| `--api-url <URL>` | Override API base URL (also: `PCKLE_API_URL` env var) |

**Always use `--json` when calling from an agent.** This ensures structured, parseable output.

## Agent Selection

All workflow commands require an agent. Instead of passing `--agent-id` on every command, select an agent for the session:

```bash
# Select an agent (persists to ~/.pckle/cli.json)
pckle agent select <AGENT_ID>

# Check which agent is selected
pckle agent which --json
```

Agent ID is resolved in this priority order:
1. `--agent-id` flag (explicit override)
2. `PCKLE_AGENT_ID` environment variable
3. Persisted selection from `pckle agent select`

Once an agent is selected, all workflow commands work without `--agent-id`:

```bash
pckle agent select abc-123
pckle workflow list --json           # uses agent abc-123
pckle workflow create --input "..." --json  # uses agent abc-123
```

## Authentication

```bash
# Login (saves token to ~/.pckle/cli.json)
pckle login --api-key <KEY>

# Check current identity
pckle whoami --json
```

## Agent Management

Agents are reusable templates that define instructions, setup scripts, and configuration for workflows. Each agent gets its own Pinecone index.

```bash
# Create an agent
pckle agent create --name "my-agent" --description "Analysis agent" --json

# Create with instructions from a file
pckle agent create --name "my-agent" --instructions @instructions.md --setup @setup.sh --json

# List all agents (includes per-agent workflow stats)
pckle agent list --json

# Get a specific agent
pckle agent get --id <AGENT_ID> --json

# Update an agent
pckle agent update --id <AGENT_ID> --name "new-name" --description "updated" --json

# Delete an agent (fails if active workflows exist)
pckle agent delete --id <AGENT_ID>

# Select an agent for the session
pckle agent select <AGENT_ID>

# Show currently selected agent
pckle agent which --json
```

## Workflow Management

Workflows are execution units that run within an agent. Each workflow takes natural language input and executes autonomously using AI.

### Workflow Types

| Type | Description |
|------|-------------|
| `find_many` | Autonomous execution with AI agent (default) |
| `find_one` | Conversational single-query AI |
| `find_all` | Search/retrieval AI |
| `upload_files` | File ingestion (multipart) |
| `import_huggingface` | Hugging Face dataset ingestion |
| `update_sourcemaps` | Sourcemap regeneration |

### Creating Workflows

```bash
# Create a workflow (find_many by default, returns immediately)
pckle workflow create --input "Find all documents about pricing" --json

# Specify workflow type
pckle workflow create --name find_all --input "Search for pricing docs" --json

# Create and wait for completion (up to 15 min)
pckle workflow create --input "Analyze the dataset" --wait --json

# Set compute tier (0=Lite, 1=Base, 2=Pro, 3=Max)
pckle workflow create --input "Deep analysis" --level 2 --json

# Set timeout
pckle workflow create --input "Process records" --timeout 300 --json

# Read input from a file
pckle workflow create --input @prompt.txt --json

# Override agent for a single command
pckle workflow create --agent-id <OTHER_AGENT_ID> --input "..." --json
```

### Listing and Filtering Workflows

```bash
# List all workflows
pckle workflow list --json

# Filter by state
pckle workflow list --state running --json

# Filter by workflow type
pckle workflow list --name find_many --json

# Paginate
pckle workflow list --limit 10 --offset 20 --json
```

### Getting Workflow Details

```bash
# Get a specific workflow (includes output, steps, token counts)
pckle workflow get --id <WORKFLOW_ID> --json
```

### Updating Workflows

```bash
# Update workflow payload (merged into existing payload)
pckle workflow update --id <WORKFLOW_ID> --payload '{"tags": ["reviewed"]}' --json
```

### Cancelling Workflows

```bash
# Cancel a running workflow
pckle workflow cancel --id <WORKFLOW_ID>
```

### Workflow Statistics

```bash
# Get resource stats (CPU, memory, tokens, runtime)
pckle workflow stats --id <WORKFLOW_ID> --json
```

## Global Statistics

```bash
# View platform-wide workflow stats
pckle stats --json
```

## API Version

```bash
pckle version --json
```

## Workflow States

Workflows progress through these states:

| State | Description |
|-------|-------------|
| `starting` | Workflow is being provisioned |
| `running` | Workflow is actively executing |
| `stopping` | Workflow is shutting down |
| `completed` | Workflow finished successfully |
| `cancelled` | Workflow was cancelled by user |
| `failed` | Workflow encountered an error |

## Common Patterns

### Quick start: create agent, select it, run workflow

```bash
AGENT=$(pckle agent create --name "analysis" --json | jq -r '.id')
pckle agent select "$AGENT"
pckle workflow create --input "Find pricing documents" --wait --json
```

### Monitor a background workflow

```bash
WF=$(pckle workflow create --input "Long analysis" --json | jq -r '.id')
pckle workflow get --id "$WF" --json
```

### Batch operations

```bash
# List all running workflows
pckle workflow list --state running --json

# Cancel all running workflows
pckle workflow list --state running --json | \
  jq -r '.items[].id' | \
  xargs -I{} pckle workflow cancel --id {}
```

## Output Format

With `--json`, all commands output structured JSON matching the PCKLE API response schemas.

Without `--json`, output is formatted as human-readable tables.

## Error Handling

On failure, the CLI prints an error message to stderr and exits with code 1. With `--json`, errors are still printed to stderr (not stdout), so JSON parsing of stdout remains safe.
