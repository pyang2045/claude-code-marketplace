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

## Instructions
1. Create the file with exact content above
2. Verify frontmatter
3. Commit: `feat: add devonthink-organize skill with tool reference and workflows`
4. Include `Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>`
