# Claude Code Marketplace

## Project Structure
- Multi-plugin marketplace repo. Each plugin lives under `plugins/<name>/`
- Each plugin is self-contained. Only `.claude-plugin/plugin.json` is required; `.mcp.json`, `commands/`, and `skills/` are optional (a plugin can be command-only, e.g. `obsidian`)
- Plugins may ship non-Claude assets consumed at runtime (e.g. `obsidian/templates/*.base`)
- Root `.claude-plugin/marketplace.json` registers all plugins

## Plugin Conventions
- Plugin manifest at `plugins/<name>/.claude-plugin/plugin.json` (NOT bare `plugin.json`)
- Skills use `skills/<skill-name>/SKILL.md` directory pattern (NOT flat .md files)
- MCP config uses `mcpServers` wrapper key in `.mcp.json`
- SKILL.md requires YAML frontmatter with `name` and `description` fields
- Commands require YAML frontmatter with `description` field
- Author field uses object format: `{ "name": "..." }`

## MCP Tool Naming
- MCP tools are namespaced as `mcp__<server>__<tool_name>`
- mcp-server-devonthink uses snake_case tool names (e.g., `is_running`, `get_record_content`)

## Adding a New Plugin
1. Create `plugins/<name>/.claude-plugin/plugin.json` (add `.mcp.json` only if the plugin ships an MCP server)
2. Add commands in `commands/` and/or skills in `skills/<skill-name>/SKILL.md`
3. Register in root `.claude-plugin/marketplace.json`
