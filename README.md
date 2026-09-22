# Claude Code Marketplace

A collection of Claude Code plugins for integrating external tools and services.

## Available Plugins

### Herdr

Herdr workflow skill for a Jira ticket's workspace lifecycle, driving the Herdr terminal multiplexer.

**Requirements:** `open` must run inside a Herdr-managed pane (`HERDR_ENV=1`) with the `herdr` CLI on PATH; `close` runs from any worktree and needs `git` and `gh`. The Atlassian MCP plugin is used for Jira (`open` degrades to agent self-fetch without it)

**Features:**
- `/ticket open SE-1234` — create a tab labelled with the Jira ticket in the current workspace (Claude Code agent left, `console` shell top-right, Codex agent bottom-right), fetch the ticket via the Atlassian MCP into a brief file, and seed both agents with it
- `/ticket close [SE-1234] [PR#]` — close out a finished task once its PR is merged or abandoned: reconcile its docs, comment on the Jira ticket, and remove the worktree. Stops for confirmation before closing over uncommitted or unpushed work and before editing the ticket description. Ticket and PR are inferred from the current branch when omitted

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
