---
name: ticket
description: Use for a Jira ticket's workspace lifecycle. `/ticket open SE-1234` opens a Herdr tab for the ticket — Claude, console, and Codex panes, agents seeded with the ticket content. `/ticket close [SE-1234] [PR#]` closes out a finished task when work is merged or abandoned and the workspace needs closing out. Triggers on "close the task", "close out", "wrap up this task", "/ticket close".
argument-hint: open <ticket> | close [ticket] [pr]
allowed-tools: Bash(echo *), Bash(herdr tab create *), Bash(herdr pane split *), Bash(herdr pane rename *), Bash(herdr pane read *), Bash(herdr pane layout *), Bash(herdr pane list *), Bash(herdr agent start *), Bash(herdr agent list), Bash(herdr agent prompt *), Write, mcp__plugin_atlassian_atlassian__getJiraIssue, mcp__plugin_atlassian_atlassian__getAccessibleAtlassianResources, Bash(git branch --show-current), Bash(git status *), Bash(git log *), Bash(git diff *), Bash(git -C * worktree list), Bash(gh pr view *), Bash(gh pr list *), mcp__plugin_atlassian_atlassian__addOrEditJiraIssueComment
---

# Ticket

Two subcommands: `open` sets up a Herdr tab for a ticket; `close` closes out the
finished task.

## Parse the arguments first

The whole argument string is: `$ARGUMENTS`

Split it on whitespace. The **first token** is the subcommand; the **remainder**
(everything after it, possibly empty) is that subcommand's arguments.

- `open` → follow **open** below with the remainder.
- `close` → follow **close** below with the remainder.
- Missing or anything else → show usage and stop:

  ```
  /ticket open SE-1234
  /ticket close [SE-1234] [PR#]
  ```

Preconditions are per-subcommand: `open` requires Herdr; `close` does not and
runs from any worktree.

## open `<TICKET>`

<!--
Usage: /ticket open SE-1234
Creates a Herdr tab named after the ticket in the current workspace with:
  left (full height)  = Claude Code agent
  top-right           = plain console shell
  bottom-right        = Codex agent
Both agents are seeded with the Jira ticket content (fetched via Atlassian MCP).
Requires: running inside a Herdr-managed pane (HERDR_ENV=1).
-->

Set up a Herdr tab for the Jira ticket named by the `open` remainder in the
current workspace, laid out as: Claude Code agent on the left (full height), a
plain console shell top-right, and a Codex agent bottom-right. Seed both agents
with the ticket's content.

Follow these steps exactly. Parse every ID from the JSON the CLI returns — never
guess IDs, and never use `--current` or omit a pane target (this session's pane
is in the *original* tab; `--current` would split the user's own pane instead of
the new tab's).

1. Check preconditions (the ticket argument is the `open` remainder):
   - If the ticket argument is missing or does not match `^[A-Za-z]+-[0-9]+$`,
     show usage `/ticket open SE-1234` and stop.
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

## close `[TICKET] [PR]`

Close out finished work in three steps, then report. Run the steps in order — each
depends on facts established by the previous one. Do not skip a step silently: if
one does not apply, say so in the final summary and why.

The `close` remainder may name a ticket key (SE-1234) and/or a PR number. If
absent, infer both from the current branch name and
`gh pr list --head "$(git branch --show-current)"`.

### Step 0 — Establish the facts first

Never act on assumed state. Gather, in the worktree being closed:

```bash
git branch --show-current
git status --short                    # uncommitted work?
git log --oneline @{upstream}..HEAD   # unpushed commits?
gh pr view <N> --json state,mergedAt,mergeCommit,headRefName
```

**If there is uncommitted or unpushed work, STOP and ask.** Closing out over the
top of unsaved work destroys it. This is the one hard gate in this workflow.

Record whether the task was **merged**, **abandoned**, or **still open** — every
later step branches on it.

### Step 1 — Reconcile the docs

Any design or plan doc the work carried describes the code as *proposed*, not as
*shipped*. A stale doc reaching the default branch is worse than no doc, because
the next reader trusts it.

```bash
git diff --name-only <base>..HEAD | grep -iE '\.md$|^doc/'
```

If that returns nothing, there is nothing to reconcile — say so and move on.

Otherwise check the three ways these actually drift:

- **Renames and moved responsibilities** — a doc naming a module, function or call
  site that review relocated.
- **Changes made mid-implementation but never written down** — anything added to
  unblock the work is invisible to a doc written before it existed.
- **Procedures that turned out not to work** — the most damaging kind, because
  they send the next person down a dead end already paid for once.

Then **consolidate**. A design doc plus a plan doc plus their overlap is three
copies to keep true. Merge into **one** doc rewritten against the shipped code —
not concatenated. Drop what the merge makes dead (per-task checklists,
step-by-step instructions, self-reviews). Keep the constraint that forced the
design, why rejected alternatives were rejected, and the traps found on the way.

Name the survivor for the feature, not the date (`doc/<feature>.md`), `git rm` the
originals, and update every reference — the PR body and the ticket both point at
the old paths.

Follow the project's own doc rules where they exist; a repo `CLAUDE.md` wins over
this file.

**The ticket description is a document too** — and the one most likely to be
stale, because it is written before the work and rarely revisited after. It is
handled in Step 2, where the ticket is already open.

### Step 2 — Update the ticket

Comment on the ticket with:

- The merge commit SHA and PR number, or the reason the work was abandoned.
- What shipped, in the reader's terms rather than the diff's.
- **Any behavioural or data-shape change, called out separately and prominently**
  — anything QA or the phone/app side would notice. Name the commit that isolates
  it so it can be reverted alone.
- Known gaps carried forward.

A comment is **additive and safe** — write it without asking.

#### Audit the description against what actually shipped

The description states the plan as it was understood *before* the work. Review it
against the merged diff and sort the differences into two lists:

- **In the diff, not in the description** — work that landed during review. These
  are the ones a future reader most needs, because they change *where you go to
  edit things*: a new shared header, a renamed or deleted symbol, an API that
  became an accessor, a block of code removed entirely.
- **In the description, not in the diff** — claimed but not shipped.

**STOP and show both lists to the user before editing the description.** Do not
rewrite it unattended. Unlike the comment, editing the description **overwrites**
the record of what was originally planned, and an item that never shipped can be
any of: deliberately descoped, deferred to a follow-up ticket, delivered
somewhere else, or genuinely forgotten. Those need different handling and only
the human knows which applies. Guessing silently either erases a real commitment
or claims something shipped that did not.

Never delete an undelivered item to make the description match the code. Move it
to a "Not implemented" or "Deferred" heading with the reason, once the user has
said which it is.

When you do edit, **verify the write landed** — re-read the issue and check that
`updated` advanced and the new text is present. A timed-out or failed write is
ambiguous, and reporting success on one leaves the ticket stale while telling the
user it is fixed.

#### Status

**Do not transition the ticket to Done reflexively.** Ask whether the ticket's
premise is actually satisfied:

- Work that *enables future diagnosis* (instrumentation, logging) does not close
  the bug it was written for — the root cause is still unknown. Say so and leave
  it open.
- Work fully verified on hardware, or a self-contained fix, can close.

When in doubt, comment and leave the status alone; say in the summary that you
did, so the human can decide. Also flag an auto-assigned priority that looks
wrong rather than silently accepting it.

### Step 3 — Close the worktree

Only after steps 0-3, and only if the PR is **merged or explicitly abandoned**.

```bash
gh pr view <N> --json state,mergedAt          # confirm terminal state
git -C <main-checkout> worktree list          # confirm which one
```

Then remove the worktree:

```bash
git worktree unlock <path>          # if locked
git worktree remove <path>
git worktree remove --force <path>  # if it refuses over submodules
```

Plain `remove` **fails with "cannot remove a worktree that contains submodules"**
on any worktree where submodules were initialized (i.e. any one you built in).
`--force` is the right escalation here and is safe once `git status` is clean —
you already confirmed that in Step 0, which is why that step comes first.

**Deleting the branch is a separate decision from removing the worktree.** After
a *squash* merge the branch is not an ancestor of the default branch — `git
branch --merged` will not list it — and its individual commits exist only on that
ref. Deleting it discards the per-commit reasoning; only the squashed message
survives on the default branch. Keep the branch unless the human says otherwise,
and say so in the summary. `/clean_gone` only cleans branches already gone on the
remote, so it is a no-op for a branch you kept.

**Keep the worktree instead when it holds something not reproducible from the
repo** — a built artifact that verified the change, captured device logs. Say
which and why in the summary rather than deleting to be tidy.

### Step 4 — Report the summary

End with a table, one row per step, plus what was deliberately left undone:

```
| Step        | Outcome                                                    |
|-------------|------------------------------------------------------------|
| Docs        | none carried / consolidated N -> doc/<feature>.md          |
| Jira        | <KEY> commented; description <edited after confirm/left>   |
|             | status left <state> because <reason>                       |
| Worktree    | removed <path> + branch / kept because <reason>            |
```

Then list explicitly:

- **Left open on purpose** — with the reason for each.
- **Needs a human** — anything you declined to decide.

Do not claim a step succeeded without having seen it succeed. If a command
failed or a check was skipped, say so in the row.
