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
