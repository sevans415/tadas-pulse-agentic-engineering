# Agentic Engineering

## Running Claude Code

Create a `.env` file at the project root (it's gitignored) with your secrets, then use this to start a session:

```bash
set -a && source .env && set +a && ENABLE_TOOL_SEARCH=false claude --dangerously-skip-permissions
```

**Why `set -a && source .env && set +a`?**

MCP server configurations in `mcp-servers/mcp.json` use `${VAR}` interpolation for secrets (API keys, tokens, AWS credentials, etc.). These variables must be present in your shell environment before launching Claude Code so they're available to child processes (i.e., MCP servers). `set -a` / `set +a` tells the shell to automatically export every variable that gets sourced. See `mcp-servers/SETUP.md` for what credentials each server needs and how to obtain them.

**Why `ENABLE_TOOL_SEARCH=false`?**

MCP works best when you [pre-emptively configure](https://www.pulsemcp.com/posts/agentic-mcp-configuration) what MCP servers should be active in a session. It _is_ more up front work than simply installing all your servers and ignoring the setup on a session-per-session basis, but we believe it's worth building some personal tooling around that dynamic rather than hoping automated "search" gets it right on the fly. This makes building reliable workflows and debugging issues easier to grok and reason through.

**Why `--dangerously-skip-permissions`?** To effectively agentically engineer, you don't want to be pinged for permissions approvals any more than is absolutely necessary. `--dangerously-skip-permissions` makes this the default, albeit with obvious associated risks. An alternative if you are working in a sensitive environment: slowly build up an always-allow-list of commands that get checked into `.claude/settings.json`.

## Useful skills

This repo ships Claude Code skills for managing a personal catalog of MCP servers — from discovery through to session configuration.

- **`/create-server-json`** — Research an MCP server (from a URL, name, or repo) and produce a validated `server.json` definition. Downloads the official schema and validates the output with `ajv` before saving. Results go into `mcp-servers/servers.json`.

- **`/create-mcp-json`** — Transform a `server.json` into a ready-to-use client configuration. Writes actionable descriptions, templates all env vars as `${ENV_VAR_NAME}` placeholders, and validates against `docs/mcp.schema.json`. Results go into `mcp-servers/mcp.json`.

- **`/adjust-claude-mcp-json`** — Pick servers from `mcp-servers/mcp.json` and generate (or update) a `.mcp.json` for a specific project. Use this to activate a subset of your catalog for a Claude Code session.
