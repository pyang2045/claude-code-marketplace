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

## Instructions
1. Create the file with exact content above (everything between the first --- and the end)
2. Verify frontmatter is valid (starts with ---, has name and description, closes with ---)
3. Commit with message: `feat: add devonthink-search skill with tool reference and workflows`
4. Include `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>` in commit
