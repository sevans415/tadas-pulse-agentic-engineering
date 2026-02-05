# mcp-servers conventions

## Files

- `servers.json` — array of server.json definitions (single file, not one per server)
- `mcp.json` — flat map of client-side MCP server configurations (single file, not one per server)
- `SETUP.md` — external setup instructions per server (credentials, IAM, CLI installs, API keys, etc.)
- `../.env.template` — template of all env vars across all servers, grouped by server name

## Keeping files in sync

- When adding a new server to `servers.json`, also add a corresponding entry to `SETUP.md` and a section to `../.env.template` with all its env vars.
- When updating a server in `servers.json` (e.g., new env vars, changed args, version bump), update the matching `mcp.json` entry, `SETUP.md` section, and `../.env.template` section to reflect the changes.
- When removing a server from `servers.json`, remove it from `mcp.json`, `SETUP.md`, and `../.env.template` as well.
