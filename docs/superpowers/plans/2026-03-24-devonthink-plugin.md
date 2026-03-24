# DEVONthink Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that ships the `mcp-server-devonthink` MCP server and provides domain-grouped skills for DEVONthink Pro.

**Architecture:** A marketplace repo (`claude-code-marketplace`) containing self-contained plugins under `plugins/<name>/`. The DEVONthink plugin uses `.claude-plugin/plugin.json` for the manifest, `.mcp.json` for auto-registered MCP server config, a setup command for prerequisite validation, and four SKILL.md files organized by domain.

**Tech Stack:** Claude Code plugin system (JSON manifests, YAML frontmatter, Markdown skills/commands)

**Spec:** `docs/superpowers/specs/2026-03-24-devonthink-plugin-design.md`

---

## File Structure

| File | Responsibility |
|------|---------------|
| `.claude-plugin/marketplace.json` | Repo-root marketplace registry listing all plugins |
| `plugins/devonthink/.claude-plugin/plugin.json` | Plugin manifest (name, version, author, etc.) |
| `plugins/devonthink/.mcp.json` | MCP server config — auto-starts devonthink server |
| `plugins/devonthink/commands/setup.md` | `/devonthink:setup` — prerequisite checker |
| `plugins/devonthink/skills/devonthink-search/SKILL.md` | Search & retrieval skill |
| `plugins/devonthink/skills/devonthink-create/SKILL.md` | Content creation skill |
| `plugins/devonthink/skills/devonthink-organize/SKILL.md` | Organization skill |
| `plugins/devonthink/skills/devonthink-browse/SKILL.md` | Database browsing skill |

---

### Task 1: Marketplace and Plugin Scaffolding

**Files:**
- Create: `.claude-plugin/marketplace.json`
- Create: `plugins/devonthink/.claude-plugin/plugin.json`
- Create: `plugins/devonthink/.mcp.json`

- [ ] **Step 1: Create marketplace manifest**

Create `.claude-plugin/marketplace.json`:

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

- [ ] **Step 2: Create plugin manifest**

Create `plugins/devonthink/.claude-plugin/plugin.json`:

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

- [ ] **Step 3: Create MCP server config**

Create `plugins/devonthink/.mcp.json`:

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

- [ ] **Step 4: Commit**

```bash
git add .claude-plugin/ plugins/devonthink/.claude-plugin/ plugins/devonthink/.mcp.json
git commit -m "feat: scaffold marketplace and devonthink plugin"
```

---

### Task 2: Setup Command

**Files:**
- Create: `plugins/devonthink/commands/setup.md`

- [ ] **Step 1: Create setup command**

Create `plugins/devonthink/commands/setup.md`:

```markdown
---
description: Check prerequisites for the DEVONthink MCP server (macOS, Node.js, DEVONthink Pro)
---

# DEVONthink Setup Check

Run prerequisite checks for the DEVONthink MCP server. Do NOT install or configure anything — the plugin's `.mcp.json` handles MCP registration automatically.

## Steps

Run each check using the Bash tool. Report results as a checklist.

### 1. Check macOS

```bash
uname -s
```

Expected: `Darwin`. If not Darwin, stop and report: "DEVONthink is macOS-only. This plugin cannot be used on your platform."

### 2. Check Node.js

```bash
node --version && npx --version
```

Expected: version numbers. If either fails, report: "Node.js is required. Install it from https://nodejs.org"

### 3. Check DEVONthink Pro

```bash
osascript -e 'id of application "DEVONthink 3"' 2>/dev/null || echo "NOT_FOUND"
```

Expected: `com.devon-technologies.think3`. If `NOT_FOUND`, report: "DEVONthink 3 is not installed. Get it from https://www.devontechnologies.com/apps/devonthink"

### 4. Report Results

Show a summary:
- [PASS/FAIL] macOS
- [PASS/FAIL] Node.js (version X.Y.Z)
- [PASS/FAIL] DEVONthink 3

If all pass: "All prerequisites met. The DEVONthink MCP server is ready to use."
If any fail: show fix instructions for each failure.
```

- [ ] **Step 2: Commit**

```bash
git add plugins/devonthink/commands/setup.md
git commit -m "feat: add /devonthink:setup prerequisite checker command"
```

---

### Task 3: Search & Retrieval Skill

**Files:**
- Create: `plugins/devonthink/skills/devonthink-search/SKILL.md`

- [ ] **Step 1: Create search skill**

Create `plugins/devonthink/skills/devonthink-search/SKILL.md`:

````markdown
---
name: devonthink-search
description: Use when the user wants to find, search, look up, or retrieve records in DEVONthink. Triggers on "search DEVONthink", "find in DEVONthink", "look up document", "get record content", "find notes about", "search my database".
---

# DEVONthink Search & Retrieval

Always check if DEVONthink is running before searching:

```
mcp__devonthink__is_running (no parameters)
```

If not running, tell the user to open DEVONthink 3 first.

## Tools

### search

Full-text search across DEVONthink databases.

```
mcp__devonthink__search
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | string | **Yes** | Search query string |
| `databaseName` | string | No | Scope search to a specific database by name |
| `groupPath` | string | No | Database-relative path (e.g., `/Inbox`, `/Projects/Archive`). Requires `databaseName`. Do NOT include database name in path. |
| `groupUuid` | string | No | UUID of group to search in |
| `recordType` | string | No | Filter by type: `group`, `markdown`, `PDF`, `bookmark`, `formatted note`, `txt`, `rtf`, `rtfd`, `webarchive`, `quicktime`, `picture`, `smart group` |
| `comparison` | string | No | Search mode: `no case`, `no umlauts`, `fuzzy`, `related` |
| `excludeSubgroups` | boolean | No | Exclude subgroups from search |
| `limit` | number | No | Max results to return |

### lookup_record

Look up records by a specific attribute.

```
mcp__devonthink__lookup_record
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `lookupType` | string | **Yes** | One of: `filename`, `path`, `url`, `tags`, `comment`, `contentHash` |
| `value` | string | **Yes** | Value to search for |
| `tags` | string[] | No | Tags to search for (when lookupType is `tags`) |
| `matchAnyTag` | boolean | No | Match any tag instead of all (when lookupType is `tags`) |
| `databaseName` | string | No | Scope to a specific database |
| `limit` | number | No | Max results |

### get_record_properties

Get metadata for a record.

```
mcp__devonthink__get_record_properties
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string | No* | Record UUID |
| `recordId` | number | No* | Record ID |
| `recordPath` | string | No* | Record path (e.g., `/Inbox/My Document`) |
| `databaseName` | string | No | Database name |

*At least one of `uuid`, `recordId`, or `recordPath` is required.

### get_record_content

Get the text content of a record.

```
mcp__devonthink__get_record_content
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string | **Yes** | Record UUID |
| `databaseName` | string | No | Database name |

## Workflows

**Basic search pipeline:**
1. `search` with query → get list of matching records (includes UUIDs)
2. `get_record_properties` with UUID → get metadata (tags, dates, path)
3. `get_record_content` with UUID → get full text content

**Find by filename:**
1. `lookup_record` with `lookupType: "filename"`, `value: "report.md"`
2. `get_record_content` with the returned UUID

**Find by tags:**
1. `lookup_record` with `lookupType: "tags"`, `tags: ["project-x", "meeting-notes"]`
2. Set `matchAnyTag: true` to match records with either tag (default matches all)

**Scoped search:**
1. `search` with `query`, `databaseName: "Research"`, `groupPath: "/Papers"`
2. Use `recordType: "PDF"` to filter to only PDF documents
````

- [ ] **Step 2: Commit**

```bash
git add plugins/devonthink/skills/devonthink-search/
git commit -m "feat: add devonthink-search skill with tool reference and workflows"
```

---

### Task 4: Content Creation Skill

**Files:**
- Create: `plugins/devonthink/skills/devonthink-create/SKILL.md`

- [ ] **Step 1: Create content creation skill**

Create `plugins/devonthink/skills/devonthink-create/SKILL.md`:

````markdown
---
name: devonthink-create
description: Use when the user wants to create notes, bookmarks, groups, or save web pages into DEVONthink. Triggers on "create note in DEVONthink", "save to DEVONthink", "bookmark in DEVONthink", "add to DEVONthink", "save this page", "new group".
---

# DEVONthink Content Creation

Before creating records, use `mcp__devonthink__get_open_databases` to find available databases and `mcp__devonthink__list_group_content` to find the target group UUID.

## Tools

### create_record

Create a new record in DEVONthink.

```
mcp__devonthink__create_record
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | **Yes** | Name of the new record |
| `type` | string | **Yes** | Record type: `markdown`, `formatted note`, `bookmark`, `group`, `txt`, `rtf` |
| `content` | string | No | Content for text-based records (markdown, txt, etc.) |
| `url` | string | No | URL for bookmark records |
| `parentGroupUuid` | string | No | UUID of parent group (defaults to incoming group) |
| `databaseName` | string | No | Database name (defaults to current database) |

### create_from_url

Create a record from a web URL with format conversion.

```
mcp__devonthink__create_from_url
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `url` | string | **Yes** | URL to create a record from |
| `format` | string | **Yes** | Output format: `formatted_note`, `markdown`, `pdf`, `web_document` |
| `name` | string | No | Custom name for the record |
| `parentGroupUuid` | string | No | UUID of parent group |
| `readability` | boolean | No | Use readability mode to declutter the page |
| `databaseName` | string | No | Database name |
| `pdfOptions.pagination` | boolean | No | Paginate PDF output |
| `pdfOptions.width` | number | No | PDF width in points |

## Workflows

**Create a markdown note:**
1. `get_open_databases` → find target database name
2. `list_group_content` → find target group UUID
3. `create_record` with `name`, `type: "markdown"`, `content`, `parentGroupUuid`, `databaseName`
4. After creation, use `add_tags` to tag the new record (createRecord does not accept tags)

**Bookmark a URL:**
1. `create_record` with `type: "bookmark"`, `name`, `url`

**Save a web page as markdown:**
1. `create_from_url` with `url`, `format: "markdown"`, `readability: true`
2. Readability mode strips navigation and ads for cleaner content

**Save a web page as PDF:**
1. `create_from_url` with `url`, `format: "pdf"`
2. Optionally set `pdfOptions: { pagination: true, width: 800 }`

**Create a group (folder):**
1. `create_record` with `type: "group"`, `name`, `parentGroupUuid`
````

- [ ] **Step 2: Commit**

```bash
git add plugins/devonthink/skills/devonthink-create/
git commit -m "feat: add devonthink-create skill with tool reference and workflows"
```

---

### Task 5: Organization Skill

**Files:**
- Create: `plugins/devonthink/skills/devonthink-organize/SKILL.md`

- [ ] **Step 1: Create organization skill**

Create `plugins/devonthink/skills/devonthink-organize/SKILL.md`:

````markdown
---
name: devonthink-organize
description: Use when the user wants to tag, classify, move, rename, compare, or delete records in DEVONthink. Triggers on "tag records", "classify documents", "move to group", "rename record", "find similar", "delete from DEVONthink", "organize DEVONthink", "clean up database".
---

# DEVONthink Organization

## Tools

### add_tags

Add tags to a record.

```
mcp__devonthink__add_tags
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string | **Yes** | Record UUID |
| `tags` | string[] | **Yes** | Tags to add |

### remove_tags

Remove tags from a record.

```
mcp__devonthink__remove_tags
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string | **Yes** | Record UUID |
| `tags` | string[] | **Yes** | Tags to remove |
| `databaseName` | string | No | Database name |

### classify

Get AI-powered classification suggestions for where a record should be filed.

```
mcp__devonthink__classify
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `recordUuid` | string | **Yes** | Record UUID to classify |
| `databaseName` | string | No | Database to search for suggestions |
| `comparison` | string | No | `data comparison` or `tags comparison` |
| `tags` | boolean | No | If true, propose tags instead of groups |

### compare

Find similar records or compare two specific records.

```
mcp__devonthink__compare
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `recordUuid` | string | **Yes** | Primary record UUID |
| `compareWithUuid` | string | No | Second UUID for direct comparison |
| `databaseName` | string | No | Database to search in |
| `comparison` | string | No | `data comparison` or `tags comparison` |

### move_record

Move a record to a different group.

```
mcp__devonthink__move_record
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string | No* | Record UUID |
| `recordId` | number | No* | Record ID |
| `recordName` | string | No* | Record name |
| `recordPath` | string | No* | Record path |
| `destinationGroupUuid` | string | No | UUID of destination group |
| `databaseName` | string | No | Database name |

*At least one of `uuid`, `recordId`, `recordName`, or `recordPath` is required. Prefer `uuid` when available.

### rename_record

Rename a record.

```
mcp__devonthink__rename_record
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string | **Yes** | Record UUID |
| `newName` | string | **Yes** | New name |
| `databaseName` | string | No | Database name |

### delete_record

Delete a record. **Always confirm with the user before deleting.**

```
mcp__devonthink__delete_record
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string | No* | Record UUID |
| `recordId` | number | No* | Record ID |
| `recordPath` | string | No* | Record path |
| `databaseName` | string | No | Database name |

*At least one of `uuid`, `recordId`, or `recordPath` is required.

## Workflows

**Bulk tagging:**
1. `search` for matching records
2. For each result, call `add_tags` with the record's UUID and desired tags

**Classify and file a record:**
1. `classify` with `recordUuid` → returns suggested groups with confidence scores
2. Present suggestions to user
3. `move_record` with `uuid` and `destinationGroupUuid` of the chosen suggestion

**Find duplicates:**
1. `compare` with `recordUuid` → returns similar records with similarity scores
2. Review matches with user
3. `delete_record` duplicates (with user confirmation)

**Rename workflow:**
1. `search` or `lookup_record` to find the record
2. `get_record_properties` to confirm it's the right one
3. `rename_record` with `uuid` and `newName`
````

- [ ] **Step 2: Commit**

```bash
git add plugins/devonthink/skills/devonthink-organize/
git commit -m "feat: add devonthink-organize skill with tool reference and workflows"
```

---

### Task 6: Database Browsing Skill

**Files:**
- Create: `plugins/devonthink/skills/devonthink-browse/SKILL.md`

- [ ] **Step 1: Create browsing skill**

Create `plugins/devonthink/skills/devonthink-browse/SKILL.md`:

````markdown
---
name: devonthink-browse
description: Use when the user wants to explore databases, list groups, or check DEVONthink status. Triggers on "list databases", "browse DEVONthink", "show groups", "is DEVONthink running", "what's in my database", "explore DEVONthink", "show me my DEVONthink".
---

# DEVONthink Database Browsing

## Tools

### is_running

Check if DEVONthink is active. Call this before any other DEVONthink operation.

```
mcp__devonthink__is_running (no parameters)
```

Returns whether DEVONthink 3 is currently running. If not running, tell the user to open it.

### get_open_databases

List all open databases.

```
mcp__devonthink__get_open_databases (no parameters)
```

Returns database names and UUIDs for all currently open databases.

### list_group_content

Browse the contents of a group (folder) in a database.

```
mcp__devonthink__list_group_content
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string | No | Group UUID (defaults to database root) |
| `databaseName` | string | No | Database name |

Returns the records and subgroups within the specified group.

## Workflows

**Explore databases:**
1. `is_running` → confirm DEVONthink is active
2. `get_open_databases` → list all databases with names and UUIDs
3. Present the database list to the user

**Browse a database:**
1. `list_group_content` with `databaseName` (no uuid = root level)
2. Show the top-level groups and records
3. User picks a group → `list_group_content` with that group's `uuid`
4. Repeat to drill deeper into the hierarchy

**Map database structure:**
1. Start at root with `list_group_content`
2. For each group in results, recursively call `list_group_content` with the group UUID
3. Build a tree view of the database structure
4. Present to user as an indented list or tree
````

- [ ] **Step 2: Commit**

```bash
git add plugins/devonthink/skills/devonthink-browse/
git commit -m "feat: add devonthink-browse skill with tool reference and workflows"
```

---

### Task 7: Final Validation

- [ ] **Step 1: Verify file structure**

```bash
find plugins/devonthink -type f | sort
```

Expected output:
```
plugins/devonthink/.claude-plugin/plugin.json
plugins/devonthink/.mcp.json
plugins/devonthink/commands/setup.md
plugins/devonthink/skills/devonthink-browse/SKILL.md
plugins/devonthink/skills/devonthink-create/SKILL.md
plugins/devonthink/skills/devonthink-organize/SKILL.md
plugins/devonthink/skills/devonthink-search/SKILL.md
```

- [ ] **Step 2: Validate JSON files parse correctly**

```bash
python3 -m json.tool .claude-plugin/marketplace.json > /dev/null && echo "marketplace.json: OK"
python3 -m json.tool plugins/devonthink/.claude-plugin/plugin.json > /dev/null && echo "plugin.json: OK"
python3 -m json.tool plugins/devonthink/.mcp.json > /dev/null && echo ".mcp.json: OK"
```

Expected: all three print OK.

- [ ] **Step 3: Verify SKILL.md files have valid frontmatter**

```bash
for f in plugins/devonthink/skills/*/SKILL.md; do
  echo "=== $f ==="
  head -4 "$f"
  echo ""
done
```

Expected: each file starts with `---`, has `name:` and `description:`, and closes with `---`.

- [ ] **Step 4: Verify command has valid frontmatter**

```bash
head -4 plugins/devonthink/commands/setup.md
```

Expected: starts with `---`, has `description:`, closes with `---`.
