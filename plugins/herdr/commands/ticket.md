---
description: Open a Herdr tab for a Jira ticket — Claude, console, and Codex panes, agents seeded with the ticket content
argument-hint: [jira-ticket]
allowed-tools: Bash(echo *), Bash(herdr tab create *), Bash(herdr pane split *), Bash(herdr pane rename *), Bash(herdr pane read *), Bash(herdr pane layout *), Bash(herdr pane list *), Bash(herdr agent start *), Bash(herdr agent list), Bash(herdr agent prompt *), Write, mcp__plugin_atlassian_atlassian__getJiraIssue, mcp__plugin_atlassian_atlassian__getAccessibleAtlassianResources
---

<!--
Usage: /ticket SE-1234
Creates a Herdr tab named after the ticket in the current workspace with:
  left (full height)  = Claude Code agent
  top-right           = plain console shell
  bottom-right        = Codex agent
Both agents are seeded with the Jira ticket content (fetched via Atlassian MCP).
Requires: running inside a Herdr-managed pane (HERDR_ENV=1).
-->

Set up a Herdr tab for Jira ticket `$1` in the current workspace, laid out as:
Claude Code agent on the left (full height), a plain console shell top-right,
and a Codex agent bottom-right. Seed both agents with the ticket's content.

Follow these steps exactly. Parse every ID from the JSON the CLI returns — never
guess IDs, and never use `--current` or omit a pane target (this session's pane
is in the *original* tab; `--current` would split the user's own pane instead of
the new tab's).

1. Check preconditions (the ticket argument is `$1` above, or the `ARGUMENTS:`
   line if `$1` was not substituted):
   - If the ticket argument is missing or does not match `^[A-Za-z]+-[0-9]+$`,
     show usage `/ticket SE-1234` and stop.
   - Run `echo "HERDR_ENV=${HERDR_ENV:-unset} WORKSPACE=${HERDR_WORKSPACE_ID:-unset}"`.
     If HERDR_ENV is not `1`, say this session is not running inside Herdr and stop.
   Derive TICKET = the argument uppercased (e.g. `SE-1234`) and SLUG = it
   lowercased (e.g. `se-1234`).

2. Check for name collisions: run `herdr agent list`. If `<SLUG>-claude` or
   `<SLUG>-codex` is already a live agent name, append `-2` (then `-3`, …) to
   both names until unique. Names must match `[a-z][a-z0-9_-]{0,31}`.

3. Fetch the ticket content via the Atlassian MCP. The tool is deferred — load
   it first with `ToolSearch("select:mcp__plugin_atlassian_atlassian__getJiraIssue")`,
   then call it with `cloudId: "jawbonehealth.atlassian.net"`,
   `issueIdOrKey: <TICKET>`, `responseContentFormat: "markdown"` (if the
   hostname is rejected as cloudId, get the UUID from
   `getAccessibleAtlassianResources`). Read from `issues.nodes[0]`: `key`,
   `fields.summary`, `fields.status.name`, `fields.assignee.displayName`,
   `fields.priority.name`, `fields.description`, `webUrl`.
   - If the ticket does not exist, report that and stop — build nothing.
   - If the fetch fails for auth/plugin reasons, note it and continue; step 8
     falls back to self-fetch.
   - On success, Write a brief to `${TMPDIR:-/tmp}/<SLUG>-ticket.md`: a
     frontmatter-free markdown file with the key + summary as the title, then
     status / priority / assignee / URL on one line each, then the full
     description verbatim. Skip comments unless the description is trivially
     short.

4. Create the tab in the current workspace, keeping the user's focus where it is:

   ```bash
   herdr tab create --workspace "$HERDR_WORKSPACE_ID" --label "<TICKET>" --cwd "$PWD" --no-focus
   ```

   Read `.result.root_pane.pane_id` → this is LEFT (it stays full-height on the
   left) and `.result.tab.tab_id` → TAB.

5. Split LEFT to the right:

   ```bash
   herdr pane split <LEFT> --direction right --cwd "$PWD" --no-focus
   ```

   Read `.result.pane.pane_id` → RIGHT.

6. Split RIGHT down. The original pane stays on top (console), the new pane is
   the bottom:

   ```bash
   herdr pane split <RIGHT> --direction down --cwd "$PWD" --no-focus
   ```

   Read `.result.pane.pane_id` → BOTTOM_RIGHT. RIGHT is now the top-right
   console pane; rename it: `herdr pane rename <RIGHT> console`.

7. Start the agents (each pane must be at its interactive shell prompt, which a
   freshly created pane is):

   ```bash
   herdr agent start <SLUG>-claude --kind claude --pane <LEFT>
   herdr agent start <SLUG>-codex --kind codex --pane <BOTTOM_RIGHT>
   ```

   If a start times out, read the pane with
   `herdr pane read <pane_id> --source recent-unwrapped --lines 40` and report
   what is blocking instead of retrying blindly.

8. Seed both agents with the ticket content — two `herdr agent prompt` calls,
   issued in parallel:

   ```bash
   herdr agent prompt <SLUG>-claude "Load context for Jira ticket <TICKET>: read <brief-file-path> — it contains the full ticket. Do not start work; wait for instructions." --wait --timeout 120000
   herdr agent prompt <SLUG>-codex "Load context for Jira ticket <TICKET>: read <brief-file-path> — it contains the full ticket. Do not start work; wait for instructions." --wait --timeout 120000
   ```

   Keep each prompt a single short sentence pointing at the brief file — never
   inline the ticket body into the prompt text. If step 3 failed and there is
   no brief file, fall back to
   `"Load context for Jira ticket <TICKET>: fetch it with your available Jira tools and summarize it. Do not start work; wait for instructions."`
   (the Codex agent may lack Jira access — if so it will at least hold the
   key). On a timeout or `agent_prompt_stalled`, read the pane and report what
   is blocking instead of retrying blindly.

9. Do not send the agents anything further and do not focus the new tab.
   Report to the user: the tab label and TAB id, the layout (left =
   `<SLUG>-claude`, top-right = console, bottom-right = `<SLUG>-codex`), the
   brief file path, and that they can switch to it when ready.
