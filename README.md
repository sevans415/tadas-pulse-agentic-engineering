# Agentic Engineering

## Principles

Agentic coding is often only as good as the feedback loops it can use to inspect and verify completion of work. 

### Verification in an agentic loop: assertion and observability

There are two oft-missing components to a strong agentic loop:

1. **Assertion** — the agent proves its work is correct. This means running tests, type checkers, linters, build steps, and any other pass/fail gate before moving on. The goal is a tight loop: write code, run checks, fix failures, repeat — all without human intervention.

2. **Observability** — the agent can see what's actually happening at runtime - you empower it with debugging capabilities. Logs, error traces, deployment status, metrics — when an agent has access to this data, it can diagnose the real problem instead of guessing.

### MCP's role: connecting and constraining

MCP is the standard interface for giving agents access to tools and data. It matters here for two reasons:

**Connecting.** MCP servers plug agents into your real infrastructure — databases, cloud APIs, CI/CD pipelines, monitoring systems — through a standardized protocol. Instead of fragile bash scripts and ad-hoc tool wrappers, you get structured tool definitions that are easy to reason about and designed for an LLM-forward era. This is what makes verification loops practical: the agent can call `create_database_index`, `check_deployment_status`, or `query_logs` as it identifies and fixes problems and builds net new functionality.

**Constraining.** MCP configurations let you control exactly what an agent can do in a given session. You can scope a server to read-only mode, filter which tools are exposed, limit access to specific resources, and template secrets through environment variables so credentials never touch the context window. This is how you make agents safe enough to run autonomously — not by hoping the model behaves, but by making dangerous actions structurally impossible in that session's configuration.

With this combination, agents can iterate their way to completion, and you constrain their access so the blast radius of mistakes is bounded.

## Introduction

This repo contains three things:

- **`mcp-servers/`** — a personal catalog of MCP server definitions (`servers.json`) and client configurations (`mcp.json`), plus setup docs and env var templates. This is where you define what servers exist and how to run them.
- **`agents-md/`** — example `AGENTS.md` files that show how to configure agent behavior per project and per directory. These are the instructions agents read before touching your code: what commands to run, what patterns to follow, what mistakes to avoid.
- **`skills/`** — example Claude Code skills for common development workflow steps: waiting for CI, reviewing PRs, testing in staging, performing releases. These are the kind of reusable, agent-executable procedures that tie everything together.

The MCP server catalog is the core of the repo, so the rest of this section focuses on the two file formats that drive it.

### server.json — what a server is

A `server.json` is a structured definition of an MCP server: how to install it, how to run it, what arguments it takes, and what environment variables it needs. It follows the [MCP Registry schema](https://raw.githubusercontent.com/modelcontextprotocol/static/main/schemas/2025-12-11/server.schema.json) and captures everything a client needs to know to launch a server correctly.

Here's the Postgres MCP server definition from this repo's catalog (`mcp-servers/servers.json`), trimmed for readability:

```json
{
  "name": "io.github.crystaldba/postgres-mcp",
  "title": "PostgreSQL MCP Pro",
  "packages": [{
    "registryType": "pypi",
    "identifier": "postgres-mcp",
    "runtimeHint": "uvx",
    "packageArguments": [{
      "name": "--access-mode",
      "choices": ["restricted", "unrestricted"],
      "default": "restricted"
    }],
    "environmentVariables": [{
      "name": "DATABASE_URI",
      "isRequired": true,
      "isSecret": true
    }]
  }]
}
```

The definition is declarative and machine-readable. It says: install from PyPI via `uvx`, pass `--access-mode` to control read/write access, and inject credentials through `DATABASE_URI`. A client (or a skill that generates client configs) can consume this directly.

### mcp.json — how a server runs in a session

An `mcp.json` is the client-side configuration that actually launches servers. It's a standardized representation of what Claude Code (or any MCP client) reads at startup to know which servers to connect to and how.

Here's what the Postgres server looks like in `mcp-servers/mcp.json`, configured for two access levels:

```json
"pg-postgres-mcp-ro": {
  "command": "uvx",
  "args": ["--from", "postgres-mcp==0.3.0", "postgres-mcp", "--access-mode=restricted"],
  "env": {
    "DATABASE_URI": "postgresql://${PG_USER}:${PG_PASSWORD}@${PG_HOST}:${PG_PORT}/${PG_DATABASE}"
  }
},
"pg-postgres-mcp-rw": {
  "command": "uvx",
  "args": ["--from", "postgres-mcp==0.3.0", "postgres-mcp", "--access-mode=unrestricted"],
  "env": {
    "DATABASE_URI": "postgresql://${PG_USER}:${PG_PASSWORD}@${PG_HOST}:${PG_PORT}/${PG_DATABASE}"
  }
}
```

Same server, two configurations. The read-only variant is safe to point at production — resource-limited queries, no writes. The read-write variant is for development. You pick which one to activate for a given session. Credentials use `${VAR}` interpolation so secrets stay in your `.env` file, never in the config itself.

### Claude Code's .mcp.json — one more hop (for now)

`mcp.json` is not yet an official standard — it's a format we use in this repo to represent client configurations in a client-agnostic way. Claude Code has its own `.mcp.json` format that wraps servers under an `mcpServers` key and drops fields like `title` and `description`. So there's one more transformation step: from the catalog's `mcp.json` to the project-scoped `.mcp.json` that Claude Code actually reads.

The differences are minor:

```json
// mcp-servers/mcp.json (catalog format)
{
  "pg-postgres-mcp-ro": {
    "title": "PostgreSQL MCP Pro (Read-Only)",
    "description": "PostgreSQL tuning and analysis...",
    "type": "stdio",
    "command": "uvx",
    "args": ["--from", "postgres-mcp==0.3.0", "postgres-mcp", "--access-mode=restricted"],
    "env": { "DATABASE_URI": "postgresql://${PG_USER}:${PG_PASSWORD}@${PG_HOST}:${PG_PORT}/${PG_DATABASE}" }
  }
}

// .mcp.json (Claude Code format)
{
  "mcpServers": {
    "pg-postgres-mcp-ro": {
      "command": "uvx",
      "args": ["--from", "postgres-mcp==0.3.0", "postgres-mcp", "--access-mode=restricted"],
      "env": { "DATABASE_URI": "postgresql://${PG_USER}:${PG_PASSWORD}@${PG_HOST}:${PG_PORT}/${PG_DATABASE}" }
    }
  }
}
```

The `/adjust-claude-mcp-json` skill handles this transformation. Long term, the hope is these formats converge — there's no fundamental reason the catalog format and the client format need to differ. Until then, the skill bridges the gap.

### What this looks like in practice

The principles above — assertion, observability, connecting, constraining — become concrete when you see what agents can actually do with the right servers configured.

**Closing the verification loop.** With Playwright configured as an MCP server, the agent can start a dev server, navigate to pages, interact with elements, and screenshot the results. When building a search feature, the agent writes the code, runs it in a browser, checks the output, and fixes failures — iterating until the feature [verifiably works](https://www.pulsemcp.com/posts/closing-the-agentic-loop-mcp-use-case). No manual QA step required.

**Observability that drives debugging.** An ECS deployment is failing intermittently. With the ECS MCP server and CloudWatch Log Analyzer both active, the agent can pull task failure events, correlate them with log streams, identify an image pull failure from a specific ECR digest, and trace it back to a missing platform architecture — all without you running a single `aws` command. The agent has the same read access to production telemetry that you'd use to debug the issue manually.

**Constraining access per session.** The DynamoDB server in this catalog supports three tool groups: `readonly`, `readwrite`, and `admin`. For a data migration task, you might activate `readwrite` but leave `admin` (create/delete table) disabled. For investigating a production issue, `readonly` only. The S3 server goes further — set `S3_BUCKET=my-specific-bucket` and the agent can only touch that one bucket; tools like `create_bucket` and `delete_bucket` are automatically hidden. These aren't IAM policies (though those matter too) — they're structural constraints on what tools the agent even knows about.

**Composing servers for multi-system workflows.** A data engineering task: build a model that estimates download counts by joining data from multiple sources. With a BigQuery MCP server and dbt MCP server both active, the agent writes the transformation, runs it, queries the output table, spot-checks the numbers against known values, and iterates on the formula — all in one session. Or consider alert triage: an AppSignal MCP server surfaces the error trace, the agent writes a reproducing test, fixes the code, and opens a PR via GitHub MCP tools. The servers compose because MCP is a shared protocol, and the agent orchestrates across them naturally.

### The workflow

The skills in this repo automate the path from "I found a useful MCP server" to "it's active in my coding session":

1. **`/create-server-json`** — research a server and produce a validated `server.json` definition, added to `mcp-servers/servers.json`
2. **`/create-mcp-json`** — transform that definition into a ready-to-use client config in `mcp-servers/mcp.json`, with env var placeholders and access-mode variants
3. **`/adjust-claude-mcp-json`** — pick servers from your catalog and generate the `.mcp.json` for a specific project

The catalog (`servers.json`) is your source of truth for what servers exist. The client configs (`mcp.json`) are your source of truth for how to run them. The per-project `.mcp.json` is what actually gets loaded. Each layer is a deliberate narrowing: all known servers, to all configured servers, to the servers active right now.

## Running Claude Code

Create a `.env` file at the project root (it's gitignored) with your secrets. Then add this function to your shell profile (`~/.zshrc` or `~/.bashrc`):

```sh
clauder() {
  [ -f .env ] && set -a && source .env && set +a
  ENABLE_TOOL_SEARCH=false claude --dangerously-skip-permissions "$@"
}
```

Then start a session with:

```bash
clauder
```

All flags pass through, so `clauder --continue`, `clauder --resume`, etc. work as expected.

**Why source `.env`?**

MCP server configurations in `mcp-servers/mcp.json` use `${VAR}` interpolation for secrets (API keys, tokens, AWS credentials, etc.). These variables must be present in your shell environment before launching Claude Code so they're available to child processes (i.e., MCP servers). `set -a` / `set +a` tells the shell to automatically export every variable that gets sourced. The function conditionally sources `.env` only if one exists in the current directory. See `mcp-servers/SETUP.md` for what credentials each server needs and how to obtain them.

**Why `ENABLE_TOOL_SEARCH=false`?**

MCP works best when you [pre-emptively configure](https://www.pulsemcp.com/posts/agentic-mcp-configuration) what MCP servers should be active in a session. It _is_ more up front work than simply installing all your servers and ignoring the setup on a session-per-session basis, but we believe it's worth building some personal tooling around that dynamic rather than hoping automated "search" gets it right on the fly. This makes building reliable workflows and debugging issues easier to grok and reason through.

**Why `--dangerously-skip-permissions`?** To effectively agentically engineer, you don't want to be pinged for permissions approvals any more than is absolutely necessary. `--dangerously-skip-permissions` makes this the default, albeit with obvious associated risks. An alternative if you are working in a sensitive environment: slowly build up an always-allow-list of commands that get checked into `.claude/settings.json`.

## Useful skills

This repo ships Claude Code skills for managing a personal catalog of MCP servers — from discovery through to session configuration.

- **`/create-server-json`** — Research an MCP server (from a URL, name, or repo) and produce a validated `server.json` definition. Downloads the official schema and validates the output with `ajv` before saving. Results go into `mcp-servers/servers.json`.

- **`/create-mcp-json`** — Transform a `server.json` into a ready-to-use client configuration. Writes actionable descriptions, templates all env vars as `${ENV_VAR_NAME}` placeholders, and validates against `docs/mcp.schema.json`. Results go into `mcp-servers/mcp.json`.

- **`/adjust-claude-mcp-json`** — Pick servers from `mcp-servers/mcp.json` and generate (or update) a `.mcp.json` for a specific project. Use this to activate a subset of your catalog for a Claude Code session.
