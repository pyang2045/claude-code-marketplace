# DEVONthink Claude Code Plugin — Design Spec

## Overview

A Claude Code plugin that provides the `mcp-server-devonthink` MCP server and domain-grouped skills for working with DEVONthink Pro from Claude.

The plugin lives within the `claude-code-marketplace` repository, which is a collection of plugins. Each plugin is self-contained under `plugins/<name>/`.

## Repository Structure

```
claude-code-marketplace/
├── .claude-plugin/
│   └── marketplace.json           # Marketplace registry for all plugins
├── plugins/
│   └── devonthink/
│       ├── .claude-plugin/
│       │   └── plugin.json        # Plugin manifest
│       ├── .mcp.json              # MCP server config (auto-registered)
│       ├── commands/
│       │   └── setup.md           # /devonthink:setup prerequisite checker
│       └── skills/
│           ├── devonthink-search/
│           │   └── SKILL.md
│           ├── devonthink-create/
│           │   └── SKILL.md
│           ├── devonthink-organize/
│           │   └── SKILL.md
│           └── devonthink-browse/
│               └── SKILL.md
```

Future plugins follow the same pattern: `plugins/<name>/.claude-plugin/plugin.json` + component directories.

## Marketplace Manifest (`.claude-plugin/marketplace.json`)

Located at the repo root. The Claude Code plugin system reads this file when the user installs the marketplace (e.g., `claude plugins add github:pyang2045/claude-code-marketplace`). It tells Claude Code which plugins are available and where to find them within the repo. Each plugin is independently installable via its `path`.

```json
{
  "name": "claude-code-marketplace",
  "plugins": [
    {
      "name": "devonthink",
      "path": "plugins/devonthink"
    }
  ]
}
```

## Plugin Manifest (`plugins/devonthink/.claude-plugin/plugin.json`)

```json
{
  "name": "devonthink",
  "version": "1.0.0",
  "description": "Search, create, organize, and browse DEVONthink Pro databases via MCP.",
  "author": { "name": "pyang2045" },
  "license": "MIT",
  "keywords": ["devonthink", "mcp", "macos", "knowledge-management"],
  "repository": "https://github.com/pyang2045/claude-code-marketplace"
}
```

Auto-discovers commands from `commands/` and skills from `skills/*/SKILL.md`.

## MCP Server Config (`plugins/devonthink/.mcp.json`)

Shipped with the plugin. The plugin system automatically registers and starts this MCP server when the plugin is installed — no manual configuration needed.

```json
{
  "mcpServers": {
    "devonthink": {
      "command": "npx",
      "args": ["-y", "mcp-server-devonthink"]
    }
  }
}
```

## Setup Command (`/devonthink:setup`)

A slash command that validates prerequisites. It does **not** write MCP config (the plugin's `.mcp.json` handles that automatically).

### Frontmatter

```yaml
---
description: Check prerequisites for the DEVONthink MCP server
---
```

### Flow

1. **Check platform** — Verify macOS. Fail with a clear message on other OSes.
2. **Check Node.js** — Verify `node` and `npx` are available on PATH.
3. **Check DEVONthink** — Use `osascript` to check if DEVONthink Pro is installed.
4. **Report results** — Show pass/fail for each check with fix instructions for failures.

### Error Handling

If any prerequisite fails, report:
- What failed
- How to fix it (e.g., "Install Node.js from https://nodejs.org")

## Skills

All skills use `skills/<name>/SKILL.md` directory convention with YAML frontmatter. MCP tools are namespaced as `mcp__devonthink__<tool_name>`.

### `devonthink-search` — Search & Retrieval

```yaml
---
name: devonthink-search
description: Use when the user wants to find, search, look up, or retrieve records in DEVONthink. Triggers on requests like "search DEVONthink", "find in DEVONthink", "look up document", "get record content".
---
```

**Tools covered:**
- `mcp__devonthink__search` — Full-text search across databases
- `mcp__devonthink__lookup_record` — Find by filename, path, URL, tags, or hash
- `mcp__devonthink__get_record_properties` — Retrieve metadata for a record
- `mcp__devonthink__get_record_content` — Extract the text content of a record

**Body outline:**
- Workflow: always check `is_running` before searching
- `search` parameters: `query` (string), `database` (optional UUID to scope search)
- `lookup_record` parameters: `filename`, `path`, `url`, `tags`, `hash` — use whichever is known
- `get_record_properties` / `get_record_content`: require a record UUID from a prior search/lookup
- Example: search → get properties → get content pipeline
- Tip: combine `search` with `get_record_content` to retrieve full text of matches

### `devonthink-create` — Content Creation

```yaml
---
name: devonthink-create
description: Use when the user wants to create notes, bookmarks, groups, or save web pages into DEVONthink. Triggers on "create note in DEVONthink", "save to DEVONthink", "bookmark in DEVONthink", "add to DEVONthink".
---
```

**Tools covered:**
- `mcp__devonthink__create_record` — Create markdown notes, bookmarks, or groups
- `mcp__devonthink__create_from_url` — Bookmark or save a web page

**Body outline:**
- `create_record` parameters: `name`, `type` (markdown|bookmark|group), `content`, `database` (UUID), `group` (optional UUID for target location), `tags` (optional array)
- `create_from_url` parameters: `url`, `title` (optional), `database`, `group` (optional)
- Example: creating a markdown note in a specific group with tags
- Example: bookmarking a URL into a research database
- Tip: use `get_open_databases` first to find the target database UUID

### `devonthink-organize` — Organization

```yaml
---
name: devonthink-organize
description: Use when the user wants to tag, classify, move, rename, compare, or delete records in DEVONthink. Triggers on "tag records", "classify documents", "move to group", "rename record", "find similar", "delete from DEVONthink".
---
```

**Tools covered:**
- `mcp__devonthink__add_tags` / `mcp__devonthink__remove_tags` — Manage tags on records
- `mcp__devonthink__classify` — AI-powered categorization into groups
- `mcp__devonthink__compare` — Find similar records
- `mcp__devonthink__move_record` — Move records between groups/databases
- `mcp__devonthink__rename_record` — Rename records
- `mcp__devonthink__delete_record` — Remove records

**Body outline:**
- `add_tags` / `remove_tags` parameters: `uuid` (record), `tags` (array of strings)
- `classify` parameters: `uuid` (record) — returns suggested group placements
- `compare` parameters: `uuid` (record) — returns similar records with similarity scores
- `move_record` parameters: `uuid`, `target_group` (UUID), `target_database` (optional UUID)
- `rename_record` parameters: `uuid`, `name` (new name)
- `delete_record` parameters: `uuid` — warn user before deletion, this is irreversible
- Example: bulk tagging workflow — search → iterate results → add_tags to each
- Example: classify a record and move it to the suggested group
- Tip: use `compare` before deleting to check for duplicates

### `devonthink-browse` — Database Browsing

```yaml
---
name: devonthink-browse
description: Use when the user wants to explore databases, list groups, or check DEVONthink status. Triggers on "list databases", "browse DEVONthink", "show groups", "is DEVONthink running", "what's in my database".
---
```

**Tools covered:**
- `mcp__devonthink__is_running` — Check if DEVONthink is active
- `mcp__devonthink__get_open_databases` — List all open databases
- `mcp__devonthink__list_group_content` — Browse contents of a group/folder

**Body outline:**
- `is_running` — no parameters, returns boolean; call this before any other DEVONthink operation
- `get_open_databases` — no parameters, returns list of database names and UUIDs
- `list_group_content` parameters: `database` (UUID), `group` (optional UUID, defaults to root)
- Example: check running → list databases → browse root of first database → drill into a group
- Tip: build a mental map by recursively listing group contents

## Platform Requirements

- **macOS only** (DEVONthink is macOS-exclusive)
- **Node.js** with `npx` available
- **DEVONthink Pro** installed and running

## MCP Server Details

- **Package:** `mcp-server-devonthink` (npm)
- **License:** GPL-3.0
- **Source:** https://github.com/dvcrn/mcp-server-devonthink
