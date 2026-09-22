# ticket start `<TICKET>`

You are the **invoking** session, in the user's original tab. Your job ends when
the orchestrator in the new tab has been handed the brief. You do not run the
ticket, and you do not create any other pane: the tab gets exactly one pane,
LEFT, full height, holding the orchestrator. The orchestrator builds its own
right column later (`/ticket drive`).

Follow these steps exactly. Parse every ID from the JSON the CLI returns — never
guess IDs, and never use `--current` or omit a pane target (this session's pane
is in the *original* tab; `--current` would act on the user's own pane instead
of the new tab's).

1. Check preconditions (the ticket argument is the `start` remainder):
   - If the ticket argument is missing or does not match `^[A-Za-z]+-[0-9]+$`,
     show usage `/ticket start SE-1234` and stop.
   - Run `echo "HERDR_ENV=${HERDR_ENV:-unset} WORKSPACE=${HERDR_WORKSPACE_ID:-unset}"`.
     If HERDR_ENV is not `1`, say this session is not running inside Herdr and stop.
   Derive TICKET = the argument uppercased (e.g. `SE-1234`) and SLUG = it
   lowercased (e.g. `se-1234`).

2. Check for name collisions: run `herdr agent list`. If `<SLUG>-orch` is
   already a live agent name, append `-2` (then `-3`, …) until unique. Names
   must match `[a-z][a-z0-9_-]{0,31}`.

3. Fetch the ticket content via the Atlassian MCP. The tool is deferred — load
   it first with `ToolSearch("select:mcp__plugin_atlassian_atlassian__getJiraIssue")`,
   then call it with `cloudId: "jawbonehealth.atlassian.net"`,
   `issueIdOrKey: <TICKET>`, `responseContentFormat: "markdown"` (if the
   hostname is rejected as cloudId, get the UUID from
   `getAccessibleAtlassianResources`). Read `key`, `fields.summary`,
   `fields.status.name`, `fields.assignee.displayName`, `fields.priority.name`,
   `fields.description`, `webUrl`.
   - **If the ticket does not exist, report that and stop — build nothing.** No
     tab, no pane, no agent. A tab for a key that does not resolve is a
     workspace nobody can drive.
   - If the fetch fails for auth/plugin reasons, note it and continue; step 6
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

   Read `.result.root_pane.pane_id` → LEFT and `.result.tab.tab_id` → TAB.
   Do not split it.

5. Start the orchestrator in LEFT. The orchestrator runs on Fable; if Fable is
   unavailable, use `claude-opus-5` instead and say so in the report.

   ```bash
   herdr agent start <SLUG>-orch --kind claude --pane <LEFT> -- --model claude-fable-5-1
   ```

   If the start returns `agent_not_ready`, or the agent shows `blocked`, read
   the pane:

   ```bash
   herdr agent read <SLUG>-orch --source recent-unwrapped --lines 40
   ```

   If it is Claude Code's workspace-trust prompt, accept it and wait for the
   agent to settle:

   ```bash
   herdr agent send-keys <SLUG>-orch down
   herdr agent send-keys <SLUG>-orch enter
   herdr agent wait <SLUG>-orch --timeout 60000
   ```

   For anything else, or a timeout, report what the pane shows instead of
   retrying blindly.

6. Seed the orchestrator — one prompt, one sentence:

   ```bash
   herdr agent prompt <SLUG>-orch "Read <brief-file-path>, then invoke /ticket drive <TICKET>." --wait --timeout 120000
   ```

   Never inline the ticket body into the prompt text. If step 3 failed and
   there is no brief file, send instead
   `"Fetch Jira ticket <TICKET> with your available Jira tools, then invoke /ticket drive <TICKET>."`
   On a timeout or `agent_prompt_stalled`, read the pane and report what is
   blocking instead of retrying blindly.

7. Do not send the orchestrator anything further and do not focus the new tab.
   Report to the user: the tab label and TAB id, LEFT's pane id and the agent
   name `<SLUG>-orch`, the model it runs on, the brief file path, and that they
   can switch to it when ready.
