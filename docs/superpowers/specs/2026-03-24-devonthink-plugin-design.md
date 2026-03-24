# DEVONthink Claude Code Plugin — Design Spec

## Overview

A Claude Code plugin that installs and configures the `mcp-server-devonthink` MCP server, and provides domain-grouped skills for working with DEVONthink Pro from Claude.

The plugin lives within the `claude-code-marketplace` repository, which is a collection of plugins. Each plugin is self-contained under `plugins/<name>/`.

## Repository Structure

```
claude-code-marketplace/
├── plugins/
│   └── devonthink/
│       ├── plugin.json
│       ├── commands/
│       │   └── install-devonthink.md
│       └── skills/
│           ├── devonthink-search.md
│           ├── devonthink-create.md
│           ├── devonthink-organize.md
│           └── devonthink-browse.md
```

Future plugins will follow the same pattern: `plugins/<name>/plugin.json` + component directories.

## Plugin Manifest (`plugin.json`)

- **name:** `devonthink`
- **description:** Install and use the DEVONthink MCP server for searching, creating, organizing, and browsing DEVONthink Pro databases.
- **Auto-discovers** commands from `commands/` and skills from `skills/`.

## Install Command (`/install-devonthink`)

A slash command that automates prerequisite checks and MCP server configuration.

### Flow

1. **Check platform** — Verify macOS. Fail with a clear message on other OSes.
2. **Check Node.js** — Verify `node` and `npx` are available on PATH.
3. **Check DEVONthink** — Use `osascript` to check if DEVONthink Pro is installed.
4. **Ask install scope** — Prompt the user to choose:
   - Project-level: `.mcp.json` in the current working directory
   - User-level: `~/.claude/.mcp.json` (available globally)
5. **Write MCP config** — Add the `devonthink` server entry to the chosen config file:
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
   If the file already exists, merge into existing `mcpServers` without overwriting other entries.
6. **Confirm success** — Report what was configured and suggest restarting Claude Code to activate.

### Error Handling

If any prerequisite fails, stop immediately with:
- What failed
- How to fix it (e.g., "Install Node.js from https://nodejs.org")

## Skills

### `devonthink-search` — Search & Retrieval

**Triggers on:** user wanting to find, search, look up, or retrieve records in DEVONthink.

**Tools covered:**
- `search` — Full-text search across databases
- `lookup_record` — Find by filename, path, URL, tags, or hash
- `get_record_properties` — Retrieve metadata for a record
- `get_record_content` — Extract the text content of a record

**Guidance includes:** tool parameters, usage examples, combining search with content retrieval, filtering strategies.

### `devonthink-create` — Content Creation

**Triggers on:** user wanting to create notes, bookmarks, groups, or save web pages into DEVONthink.

**Tools covered:**
- `create_record` — Create markdown notes, bookmarks, or groups
- `create_from_url` — Bookmark or save a web page

**Guidance includes:** record type options (markdown, bookmark, group), specifying target database/group, setting initial tags and metadata.

### `devonthink-organize` — Organization

**Triggers on:** user wanting to tag, classify, move, rename, compare, or delete records.

**Tools covered:**
- `add_tags` / `remove_tags` — Manage tags on records
- `classify` — AI-powered categorization into groups
- `compare` — Find similar records
- `move_record` — Move records between groups/databases
- `rename_record` — Rename records
- `delete_record` — Remove records

**Guidance includes:** bulk tagging patterns, classification workflows, using compare for deduplication, safe deletion practices.

### `devonthink-browse` — Database Browsing

**Triggers on:** user wanting to explore databases, list groups, or check DEVONthink status.

**Tools covered:**
- `is_running` — Check if DEVONthink is active
- `get_open_databases` — List all open databases
- `list_group_content` — Browse contents of a group/folder

**Guidance includes:** checking DEVONthink availability before operations, navigating the group hierarchy, understanding database structure.

## Platform Requirements

- **macOS only** (DEVONthink is macOS-exclusive)
- **Node.js** with `npx` available
- **DEVONthink Pro** installed and running

## MCP Server Details

- **Package:** `mcp-server-devonthink` (npm)
- **Latest version:** 1.7.1
- **License:** GPL-3.0
- **Source:** https://github.com/dvcrn/mcp-server-devonthink
