---
name: adjust-claude-mcp-json
description: Generate or update a Claude Code .mcp.json file from the project's mcp-servers/mcp.json. Use when the user wants to configure Claude Code's MCP servers from the trusted server set.
user-invocable: true
argument-hint: [servers and/or constraints e.g. "github, sentry --read-only"]
---

# adjust-claude-mcp-json

## Sequencing Checklist

- [ ] Verify `mcp-servers/mcp.json` exists and has server entries
- [ ] Parse `$ARGUMENTS` for server selection and constraints
- [ ] For each selected server, transform the flat entry into Claude Code format (wrap under `mcpServers`, adjust fields, map transport types, apply constraints)
- [ ] Check if `.mcp.json` already exists at project root; merge if exists, create if not
- [ ] Run the validation checklist (see bottom of this file)
- [ ] Present the result to the user before writing
- [ ] Write the `.mcp.json` file

Generate or update a Claude Code `.mcp.json` file using entries from this project's trusted server configurations in `mcp-servers/mcp.json`.

The user provided this context:

```
$ARGUMENTS
```

## Prerequisites

- The file `mcp-servers/mcp.json` must exist in the project and contain at least one server entry in flat mcp.json format (see `mcp-servers/README.md`)
- If it's empty or missing, tell the user and suggest running `/create-server-json` then `/create-mcp-json` first

## What this skill does

This skill bridges the gap between our project's canonical `mcp-servers/mcp.json` (flat format, transport-specific) and Claude Code's `.mcp.json` (wrapped in `mcpServers`, project-scoped).

The two formats differ:

| `mcp-servers/mcp.json` (flat) | Claude Code `.mcp.json` |
|---|---|
| Servers at root level | Servers nested under `mcpServers` key |
| `type` field required | `type` optional for stdio (defaults to stdio) |
| `${VAR}` interpolation for secrets | `${VAR}` and `${VAR:-default}` supported |

Reference: https://code.claude.com/docs/en/mcp

## Instructions

1. Read `mcp-servers/mcp.json` to see all available trusted server entries
2. Parse `$ARGUMENTS` to determine:
   - **Which servers** the user wants (by name, or "all")
   - **Constraints** like "read only", "no write tools", "minimal" — these may affect env vars or args
3. For each requested server, transform the flat entry into Claude Code format
4. Check if a `.mcp.json` already exists at the project root:
   - If yes, read it and **merge** the new entries into the existing `mcpServers` object (don't clobber existing servers unless the user asks to replace)
   - If no, create a new `.mcp.json`
5. Present the result to the user before writing

## Transformation rules

### Wrapping

Every entry goes under the `mcpServers` key:

**Input** (flat mcp.json):
```json
{
  "github": {
    "title": "GitHub",
    "type": "stdio",
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-github"],
    "env": {
      "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_PERSONAL_ACCESS_TOKEN}"
    }
  }
}
```

**Output** (Claude Code .mcp.json):
```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_PERSONAL_ACCESS_TOKEN}"
      }
    }
  }
}
```

### Field adjustments

- **`type`**: Omit for `stdio` servers (Claude Code defaults to stdio). Keep for `sse` and `streamable-http` — but note Claude Code uses `"http"` not `"streamable-http"` for HTTP transport.
- **`title`** and **`description`**: Drop these — Claude Code's `.mcp.json` does not use them.
- **`command`**, **`args`**, **`env`**: Copy directly for stdio servers.
- **`url`**, **`headers`**: Copy directly for remote servers.
- **Secret interpolation**: Keep the `${ENV_VAR_NAME}` format as-is. Claude Code resolves env vars from the shell environment. The variable name inside `${}` should always be UPPER_SNAKE_CASE matching the env var key (e.g., `${GITHUB_PERSONAL_ACCESS_TOKEN}`, `${AWS_ACCESS_KEY_ID}`). For headers, use a descriptive `UPPER_SNAKE_CASE` name.

### Transport type mapping

| Flat mcp.json `type` | Claude Code `.mcp.json` |
|---|---|
| `stdio` | Omit `type` (or include `"stdio"`) |
| `streamable-http` | `"type": "http"` |
| `sse` | `"type": "sse"` |

### Applying constraints from `$ARGUMENTS`

If the user provides constraints, apply them to the selected servers:

- **"read only"** / **"read only tools"**: If the server supports tool filtering (e.g., via `--read-only` flag, `ENABLED_TOOLS` env var, or similar), configure it for read-only access. Add a comment noting what was restricted.
- **"no env vars"**: Only include env vars that are strictly required. Drop optional ones.
- **"minimal"**: Bare minimum fields only.
- **Server selection**: If the user names specific servers (e.g., "github, sentry"), only include those. If no servers are named, include all available entries.

## Scope guidance

When presenting the result, remind the user about Claude Code's MCP scopes:

- **Project scope** (`.mcp.json` at project root): Shared with the team via version control. Best for servers the whole team needs. Created by this skill by default.
- **Local scope** (`~/.claude.json`): Private to you, project-specific. Use `claude mcp add --scope local` for personal servers.
- **User scope** (`~/.claude.json`): Available across all your projects. Use `claude mcp add --scope user` for personal utilities.

If the `.mcp.json` contains secrets that shouldn't be committed, suggest the user use `${ENV_VAR}` interpolation and set the actual values in their shell environment.

## Examples

### Select specific servers

User runs: `/adjust-claude-mcp-json github, sentry`

If `mcp-servers/mcp.json` contains entries for `github` and `sentry`, produce:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_PERSONAL_ACCESS_TOKEN}"
      }
    },
    "sentry": {
      "type": "http",
      "url": "https://mcp.sentry.dev/mcp"
    }
  }
}
```

### Read-only constraint

User runs: `/adjust-claude-mcp-json github --read-only`

If the github server supports a read-only mode (e.g., via env var or arg), configure it:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_PERSONAL_ACCESS_TOKEN}",
        "GITHUB_TOOLSETS": "repos,issues,pull_requests"
      }
    }
  }
}
```

### All servers

User runs: `/adjust-claude-mcp-json all`

Transform every entry in `mcp-servers/mcp.json` into the Claude Code format.

### Merge into existing .mcp.json

If `.mcp.json` already exists with other servers, merge the new entries without removing existing ones. Warn the user if a server name conflicts and ask whether to overwrite.

## Validation checklist

Before writing the file, verify:

- [ ] Root object has a single `mcpServers` key
- [ ] `type` is omitted for stdio servers (or explicitly set to `"stdio"`)
- [ ] Remote servers use `"http"` (not `"streamable-http"`) or `"sse"` for `type`
- [ ] `title` and `description` fields are NOT present (Claude Code ignores them)
- [ ] Secret values use `${ENV_VAR_NAME}` interpolation with actual env var names
- [ ] No literal secrets or tokens in the file
- [ ] Server name keys are valid identifiers
- [ ] The file is valid JSON
- [ ] User constraints from `$ARGUMENTS` have been applied
