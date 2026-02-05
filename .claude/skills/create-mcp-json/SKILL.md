---
name: create-mcp-json
description: Create an mcp.json client configuration from a server.json. Use when the user wants to generate a ready-to-use MCP client config for connecting to an MCP server.
user-invocable: true
argument-hint: [constraints e.g. "read only tools", "no env vars"]
---

# create-mcp-json

Generate a ready-to-use `mcp.json` client configuration from an MCP server's `server.json`.

The user provided this context about the configuration they want:

```
$ARGUMENTS
```

## Prerequisites

A `server.json` for the target server MUST already be in context before running this skill. If one is not present:

1. Ask the user to provide the `server.json`, OR
2. Suggest running `/create-server-json` first to generate one

Do NOT attempt to create an mcp.json without a server.json to derive it from.

## What is mcp.json?

An `mcp.json` is a **client-side configuration** that tells an MCP client (Claude Code, Claude Desktop, VS Code, Cursor, etc.) exactly how to connect to and run MCP servers. It is the fully-resolved, concrete form of what a `server.json` describes abstractly.

Think of it this way:
- `server.json` = "Here's how this server *can* be configured"
- `mcp.json` = "Here's exactly how this client *will* connect to its servers"

Reference the full mcp.json specification at: https://github.com/pulsemcp/mcp-servers/blob/main/docs/MCP_JSON.md

## Instructions

1. Read the `server.json` already in context
2. If `$ARGUMENTS` includes constraints (e.g., "read only tools", "no write access", "minimal env vars"), factor those into the configuration — for example by selecting specific `packageArguments` or omitting optional environment variables
3. Generate one mcp.json entry per `packages` item AND one per `remotes` item in the server.json
4. Present the result and ask the user which output format they want (see [Output Formats](#output-formats))

## Transformation Rules

### From `packages[]` (stdio servers)

| server.json field | mcp.json field | Notes |
|---|---|---|
| `title` | `title` | Direct copy |
| `description` | `description` | Direct copy |
| `packages[N].transport.type` | `type` | Usually `"stdio"` |
| `packages[N].registryType` / `runtimeHint` | `command` | See command mapping below |
| `packages[N].identifier` + `packageArguments` | `args` | See args construction below |
| `packages[N].environmentVariables` | `env` | Only `isRequired: true` vars; secrets use `${slug}` interpolation |

#### Command mapping

| `registryType` | `runtimeHint` | `command` |
|---|---|---|
| `npm` | `npx` (default) | `"npx"` |
| `pypi` | `uvx` (default) | `"uvx"` |
| `oci` | — | `"docker"` |

#### Args construction

- **npm**: `["-y", "<identifier>", ...packageArguments]`
- **pypi**: `["<identifier>", ...packageArguments]`
- **oci**: `["run", "-i", "--rm", ...runtimeArguments, "<identifier>:<version>", ...packageArguments]`

For `packageArguments`, resolve them as follows:
- `"positional"` args: append `value` directly
- `"named"` args: append `name` then `value` (e.g., `"--host", "localhost"`)
- If the argument has `isRequired: true` and no `value`, use `valueHint` as a placeholder comment or ask the user

### From `remotes[]` (HTTP/SSE servers)

| server.json field | mcp.json field | Notes |
|---|---|---|
| `title` | `title` | From top-level server.json |
| `description` | `description` | From top-level server.json |
| `remotes[N].type` | `type` | `"streamable-http"` or `"sse"` |
| `remotes[N].url` | `url` | Resolve any `{variable}` templates — ask user for values |
| `remotes[N].headers` | `headers` | Secret header values use `${slug}` interpolation |

### Secret interpolation

- Secrets (env vars or headers with `isSecret: true`) use `${product-slug}` format
- Slugs should be concise, lowercase, hyphenated, derived from the product/service name
- Examples: `${github-token}`, `${sentry-auth-token}`, `${openai-api-key}`
- Non-secret environment variables with a `default` value should use that default directly
- Non-secret environment variables without a default should use a reasonable concrete value or ask the user

### Applying user constraints from `$ARGUMENTS`

If the user specifies constraints, apply them:

- **"read only tools"** / **"read only"**: If the server supports tool filtering via `packageArguments` or env vars (e.g., `ENABLED_TOOLS`, `--read-only`), configure accordingly. Note this in the output.
- **"no env vars"**: Omit optional env vars, only include `isRequired: true` vars
- **"minimal"**: Only include required fields, skip `title` and `description`
- **Custom constraints**: Interpret and apply as best as possible, noting what was done

## Output Formats

### Flat mcp.json format (default)

The root object is a map of server names to configurations. This is the format defined in the mcp.json spec:

```json
{
  "server-name": {
    "title": "Human-Readable Name",
    "type": "stdio",
    "command": "npx",
    "args": ["-y", "@scope/package-name"],
    "env": {
      "API_KEY": "${service-api-key}"
    }
  }
}
```

### Claude Code `.mcp.json` format

Wraps servers under an `mcpServers` key. Use this when the user is configuring Claude Code specifically:

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@scope/package-name"],
      "env": {
        "API_KEY": "${service-api-key}"
      }
    }
  }
}
```

Note: Claude Code defaults `type` to `"stdio"`, so the `type` field can be omitted for stdio servers. For `sse` or `streamable-http` servers it must be included.

## Examples

### npm package → mcp.json

Given this server.json:
```json
{
  "name": "io.github.modelcontextprotocol/server-github",
  "title": "GitHub",
  "description": "GitHub API integration",
  "packages": [{
    "registryType": "npm",
    "identifier": "@modelcontextprotocol/server-github",
    "version": "0.6.2",
    "transport": { "type": "stdio" },
    "environmentVariables": [{
      "name": "GITHUB_PERSONAL_ACCESS_TOKEN",
      "description": "GitHub PAT",
      "isRequired": true,
      "isSecret": true
    }]
  }]
}
```

Produces:
```json
{
  "github": {
    "title": "GitHub",
    "description": "GitHub API integration",
    "type": "stdio",
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-github"],
    "env": {
      "GITHUB_PERSONAL_ACCESS_TOKEN": "${github-token}"
    }
  }
}
```

### pypi package → mcp.json

Given a PyPI server.json:
```json
{
  "name": "io.github.example/weather",
  "title": "Weather",
  "packages": [{
    "registryType": "pypi",
    "identifier": "weather-mcp-server",
    "version": "0.5.0",
    "runtimeHint": "uvx",
    "transport": { "type": "stdio" },
    "environmentVariables": [
      { "name": "WEATHER_API_KEY", "isRequired": true, "isSecret": true },
      { "name": "WEATHER_UNITS", "default": "celsius" }
    ]
  }]
}
```

Produces:
```json
{
  "weather": {
    "title": "Weather",
    "type": "stdio",
    "command": "uvx",
    "args": ["weather-mcp-server"],
    "env": {
      "WEATHER_API_KEY": "${weather-api-key}",
      "WEATHER_UNITS": "celsius"
    }
  }
}
```

### Remote server → mcp.json

Given a remote server.json:
```json
{
  "name": "io.github.example/analytics",
  "title": "Analytics",
  "remotes": [{
    "type": "streamable-http",
    "url": "https://mcp.analytics.example.com/mcp",
    "headers": [{
      "name": "Authorization",
      "isRequired": true,
      "isSecret": true
    }]
  }]
}
```

Produces:
```json
{
  "analytics": {
    "title": "Analytics",
    "type": "streamable-http",
    "url": "https://mcp.analytics.example.com/mcp",
    "headers": {
      "Authorization": "Bearer ${analytics-token}"
    }
  }
}
```

### OAuth remote → mcp.json (no auth config needed)

```json
{
  "linear": {
    "title": "Linear",
    "description": "Linear project management integration",
    "type": "streamable-http",
    "url": "https://mcp.linear.app/mcp"
  }
}
```

OAuth is handled out-of-band by the MCP client via well-known metadata endpoints. No explicit auth config is needed in the mcp.json.

## Validation Checklist

Before writing the file, verify:

- [ ] Every entry has a `type` field (`stdio`, `sse`, or `streamable-http`)
- [ ] `stdio` entries have `command` and do NOT have `url` or `headers`
- [ ] `sse` / `streamable-http` entries have `url` and do NOT have `command` or `args`
- [ ] Secret values use `${slug}` interpolation, not literal values
- [ ] Server name keys match `^[a-zA-Z0-9_[\]-]+$` (alphanumeric, hyphens, underscores, brackets)
- [ ] Non-secret env vars with defaults use the default value directly
- [ ] User constraints from `$ARGUMENTS` have been applied and noted
- [ ] The file is valid JSON
