# Obsidian Plugin

Command-only Claude Code plugin for Obsidian vault workflows, driven by the shell `obsidian` CLI.

## Structure
- `.claude-plugin/plugin.json` — manifest
- `commands/obs-close.md` — the `/obs-close` slash command
- `templates/handoffs.base` — Bases view shipped to the vault, not a Claude Code component
- No `.mcp.json`, no `skills/` — this plugin has no MCP server; it invokes the `obsidian` CLI via Bash

## Dependencies
- Requires the `obsidian` CLI on PATH and Obsidian running (the CLI targets the open default vault)
- Commands must fall back to direct file writes (`cat`/`tee`/`cp`) when the CLI is unavailable

## Conventions
- Follow the user's global Obsidian rules (`~/CLAUDE.md`): flat vault root, **plural tags**, `YYYY-MM-DD` dates, wikilink first mentions, **append never overwrite** for logs/session notes
- `/obs-close` writes a handoff into **this conversation's** session note (`YYYY-MM-DD HHmm <project> session`) — sessions may cross midnight, so the target is resolved by conversation context first, then by the newest non-`done` session note for the project within a 3-day lookback (never "today's date"); creates one only if neither exists. Standard association meta is `categories: [sessions]`, `tags: [<project>, handoffs, sessions]`, `status: done`
- On close, `/obs-close` sets `end: YYYY-MM-DD`; the frontmatter `date` is the session's start day, `end` is the close day (they differ for cross-midnight sessions — one note per session, never one per day)
- The `## Handoff — <YYYY-MM-DD HHmm>` heading and `#handoffs` tag are the stable anchors the `handoffs.base` view filters on — do not rename them without updating the base
- Reference plugin-bundled files via `${CLAUDE_PLUGIN_ROOT}` (e.g. `${CLAUDE_PLUGIN_ROOT}/templates/handoffs.base`)

## Editing `templates/handoffs.base`
- It is Obsidian Bases YAML (see the `obsidian-bases` skill). Keep it valid YAML and keep the `file.hasTag("handoffs")` filter aligned with the tag `/obs-close` writes.
