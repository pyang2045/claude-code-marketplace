# Claude Code Marketplace

A collection of Claude Code plugins for integrating external tools and services.

## Available Plugins

### DEVONthink

Search, create, organize, and browse DEVONthink Pro databases via MCP.

**Requirements:** macOS, Node.js, DEVONthink 3

**Features:**
- Auto-configured MCP server (`mcp-server-devonthink`)
- `/devonthink:setup` — prerequisite checker
- 4 domain-grouped skills: search, create, organize, browse

**Install:**

```bash
claude plugins add github:pyang2045/claude-code-marketplace --plugin devonthink
```

## Adding Plugins

Each plugin lives under `plugins/<name>/` with its own manifest, MCP config, commands, and skills. See [CLAUDE.md](CLAUDE.md) for conventions.
