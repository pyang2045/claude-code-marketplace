# Claude Code Marketplace

A collection of Claude Code plugins for integrating external tools and services.

## Available Plugins

### Herdr

Herdr workflow skill for a Jira ticket's workspace lifecycle, driving the Herdr terminal multiplexer.

**Requirements:** `start` and `drive` must run inside a Herdr-managed pane (`HERDR_ENV=1`) with the `herdr` CLI on PATH; `drive` also uses `git`, `gh`, and Claude Code and Codex agents. `close` runs from any worktree and needs `git` and `gh`. The Atlassian MCP plugin is used for Jira (`start` degrades to agent self-fetch without it)

**Features:**
- `/ticket start SE-1234` — create a tab labelled with the Jira ticket in the current workspace holding a single orchestrator agent (full height), fetch the ticket via the Atlassian MCP into a brief file, and seed the orchestrator to run `/ticket drive`. Stops without building anything if the ticket does not exist
- `/ticket drive SE-1234` — run by that orchestrator: audit the ticket description against the code (confirming with you before overwriting it), keep the Jira status in step with the work, start implementer / reviewer / verifier / tester agents on demand in their own git worktrees, file linked child tickets, and build the ticket's PRs as a stack of draft PRs. Hands off to `/ticket close` when everything is merged
- `/ticket close [SE-1234] [PR#]` — close out a finished task once its PRs are merged or abandoned: reconcile its docs, comment on the Jira ticket, and remove the worktree. Stops for confirmation before closing over uncommitted or unpushed work and before editing the ticket description. Ticket and PR are inferred from the current branch when omitted

**Install:**

```bash
claude plugins add github:pyang2045/claude-code-marketplace --plugin herdr
```

### Obsidian

Obsidian vault workflow helpers driven by the shell `obsidian` CLI.

**Requirements:** the `obsidian` CLI on PATH, with Obsidian running (targets the open default vault)

**Features:**
- `/obs-close` — close out a task by writing a handoff summary (Done / Current state / Next steps / Open questions / Key files) into today's session note, or creating one; tags it `#handoffs` for retrieval
- `templates/handoffs.base` — a Bases view listing every `#handoffs` note, installed into the vault root on first `/obs-close`

**Install:**

```bash
claude plugins add github:pyang2045/claude-code-marketplace --plugin obsidian
```

## Adding Plugins

Each plugin lives under `plugins/<name>/` with its own manifest, MCP config, commands, and skills. See [CLAUDE.md](CLAUDE.md) for conventions.
