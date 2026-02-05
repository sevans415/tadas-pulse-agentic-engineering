---
name: create-server-json
description: Create a valid MCP Registry server.json file for an MCP server. Use when the user wants to generate, scaffold, or write a server.json for any MCP server project.
user-invocable: true
argument-hint: [server-name-or-url]
---

# create-server-json

Create a valid MCP Registry `server.json` file for an MCP server project. The user provided this context about the server:

```
$ARGUMENTS
```

## What is server.json?

A `server.json` is a standardized self-description file for MCP (Model Context Protocol) servers. It follows the schema defined by the MCP Registry working group and enables AI agents and MCP clients to automatically discover, install, and configure servers.

Before generating the file:

1. Check the latest schema version at https://github.com/modelcontextprotocol/static/tree/main/schemas to find the most recent dated directory (e.g., `2025-12-11`).
2. Download the schema to a temp file so it can be used for validation later:
   ```bash
   curl -sL "https://raw.githubusercontent.com/modelcontextprotocol/static/main/schemas/<latest-date>/server.schema.json" -o /tmp/server.schema.json
   ```
3. Read `/tmp/server.schema.json` to understand the exact fields, types, and constraints — use this as the authoritative reference for what fields are valid. Do NOT invent fields that are not in the schema.
4. Use that schema URL as the `$schema` value in the generated file.
5. Pull https://github.com/modelcontextprotocol/registry/blob/main/docs/reference/server-json/generic-server-json.md into context for examples of what a variety of server.json files might look like.

## Instructions

When the user asks you to create a `server.json`, start by doing the following:

1) Check for a server.json file corresponding to the server at hand on registry.modelcontextprotocol.io
2) Check for a server.json file on pulsemcp.com/servers
3) Fetch any context provided by the user that could be useful for putting together the ensuing information

In some cases, you might be able to just call a finding from (1) or (2) the final version. In other cases, you might only get a partial server.json that either (1) isn't up to date, or (2) doesn't actually contain comprehensive `remotes` and/or `packages` entries for how we can actually run the server.

Using the context you collected, you should be able to construct a comprehensive `server.json` that describes a meaningful amount of top level metadata + structured, comprehensive metadata for how to run/connect to a server embedded into `packages` and/or `remotes`.

## Validation

After writing the `server.json` file, validate it against the downloaded schema using a JSON Schema validator. First ensure a validator is available, then run it:

```bash
# Install validator if needed (check first)
which ajv || npm install -g ajv-cli

# Validate the generated file against the schema
# The schema uses draft-07 and non-standard keywords like "example", so use --spec=draft7 --strict=false
ajv validate -s /tmp/server.schema.json -d <path-to-generated-server.json> --spec=draft7 --strict=false
```

If validation fails, read the errors, fix the generated file, and re-validate until it passes. Do not consider the task complete until validation succeeds.