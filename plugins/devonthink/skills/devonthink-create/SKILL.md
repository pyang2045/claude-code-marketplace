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
4. After creation, use `add_tags` to tag the new record (create_record does not accept tags)

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

## Instructions
1. Create the file with exact content above
2. Verify frontmatter is valid
3. Commit: `feat: add devonthink-create skill with tool reference and workflows`
4. Include `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>`

Report: DONE, DONE_WITH_CONCERNS, NEEDS_CONTEXT, or BLOCKED.
