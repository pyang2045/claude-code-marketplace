---
name: ticket
description: Use when opening a Herdr workspace for a Jira ticket, when orchestrating a Jira ticket end to end as its orchestrator, or when closing out merged or abandoned work. Triggers on "/jira-ticket:ticket start", "/jira-ticket:ticket drive", "/jira-ticket:ticket close", "close the task", "close out", "wrap up this task".
argument-hint: start <ticket> | drive <ticket> | close [ticket] [pr]
allowed-tools: Bash(echo *), Bash(herdr tab create *), Bash(herdr pane split *), Bash(herdr pane rename *), Bash(herdr agent start *), Bash(herdr agent prompt *), Bash(herdr agent list), Bash(herdr agent get *), Bash(herdr agent read *), Bash(herdr agent wait *), Bash(herdr pane list *), Bash(herdr pane current), Bash(herdr pane read *), Bash(herdr pane layout *), Bash(git branch --show-current), Bash(git status *), Bash(git log *), Bash(git diff *), Bash(git -C * worktree list), Bash(git worktree list), Bash(gh pr view *), Bash(gh pr list *), Write, mcp__plugin_atlassian_atlassian__getJiraIssue, mcp__plugin_atlassian_atlassian__getAccessibleAtlassianResources, mcp__plugin_atlassian_atlassian__addOrEditJiraIssueComment
---

# Ticket

## Parse the arguments first

The whole argument string is: `$ARGUMENTS`

Split it on whitespace. The **first token** is the subcommand; the **remainder**
(everything after it, possibly empty) is that subcommand's arguments. Then open
the subcommand's file in this skill's directory and follow it:

| Subcommand | Who runs it | File |
|---|---|---|
| `start <TICKET>` | the session the user typed it in | `start.md` |
| `drive <TICKET>` | the orchestrator in the ticket tab's left pane | `drive.md` |
| `close [TICKET] [PR]` | anyone, from any worktree | `close.md` |

Missing or any other first token → show usage and stop:

```
/jira-ticket:ticket start SE-1234
/jira-ticket:ticket drive SE-1234
/jira-ticket:ticket close [SE-1234] [PR#]
```

Preconditions are per-subcommand and live in each file: `start` and `drive`
require Herdr (`HERDR_ENV=1`); `close` does not.

## Quick reference

| When | Do |
|---|---|
| New ticket, want a workspace | `/jira-ticket:ticket start SE-1234` — one new tab, orchestrator only |
| You are that orchestrator | `/jira-ticket:ticket drive SE-1234` — audit, status, agents, `gh stack` PRs |
| You have ticket work but sit in another tab | hand the ticket's orchestrator a brief (`herdr agent prompt <slug>-orch "Read <path> ..."`); it dispatches and drives the agents in its own tab — see `drive.md` § Layout |
| PRs merged or work abandoned | `/jira-ticket:ticket close` — docs, ticket comment, worktree |
