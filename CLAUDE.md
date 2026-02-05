# Project conventions

## Personal MCP server catalog

- After creating a `server.json` for any MCP server, append it to the array in `mcp-servers/servers.json` — do NOT create a separate file per server.
- `mcp-servers/servers.json` is the single source of truth for all server definitions in this project.
- After creating an `mcp.json` entry for any MCP server, add it to `mcp-servers/mcp.json` — do NOT create a separate file per server.
- `mcp-servers/mcp.json` is the single source of truth for all client-side MCP server configurations in this project.
- See `mcp-servers/CLAUDE.md` for conventions specific to that directory.
