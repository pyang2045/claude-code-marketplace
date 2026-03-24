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
