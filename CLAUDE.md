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
- **Version bump is mandatory on every plugin change**: any edit under `plugins/<name>/` must bump that plugin's semver in BOTH `plugins/<name>/.claude-plugin/plugin.json` and its entry in root `.claude-plugin/marketplace.json` (keep the two in sync). Patch for fixes, minor for new commands/skills/features, major for breaking changes. Do this in the same commit — no separate "bump version" ask needed

## MCP Tool Naming
- A **plugin-shipped** MCP server (declared in the plugin's `.mcp.json`) is namespaced `mcp__plugin_<plugin>_<server>__<tool_name>` — e.g. a server named `foo` inside plugin `foo` resolves as `mcp__plugin_foo_foo__some_tool`
- A **user-configured** server (registered outside a plugin) is namespaced `mcp__<server>__<tool_name>`
- Skills that ship alongside their own MCP server must use the plugin-namespaced form; the bare form does not resolve for plugin-shipped servers

## Adding a New Plugin
1. Create `plugins/<name>/.claude-plugin/plugin.json` (add `.mcp.json` only if the plugin ships an MCP server)
2. Add commands in `commands/` and/or skills in `skills/<skill-name>/SKILL.md`
3. Register in root `.claude-plugin/marketplace.json`
