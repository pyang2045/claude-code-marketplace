# Claude Code Marketplace

A collection of Claude Code plugins for integrating external tools and services.

## Available Plugins

### Herdr

Herdr workflow commands for driving the Herdr terminal multiplexer.

**Requirements:** running inside a Herdr-managed pane (`HERDR_ENV=1`) with the `herdr` CLI on PATH; the Atlassian MCP plugin for ticket fetching (degrades to agent self-fetch without it)

**Features:**
- `/ticket SE-1234` — create a tab labelled with the Jira ticket in the current workspace (Claude Code agent left, `console` shell top-right, Codex agent bottom-right), fetch the ticket via the Atlassian MCP into a brief file, and seed both agents with it

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
