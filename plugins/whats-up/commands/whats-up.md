---
description: Summarize where things stand right now — progress, remaining work, the next step and suggested follow-ups, in a form you can read in five seconds.
---

# /whats-up — Where do things stand?

Produce a session status briefing for the user.

Follow the **`session-status` skill** shipped with this plugin. It holds the gathering
procedure, the evidence rules and the exact output template. Do not improvise a summary
from memory — the skill exists specifically to prevent that.

## Scope

`$ARGUMENTS`

- If `$ARGUMENTS` is empty, summarize the **whole session**.
- If `$ARGUMENTS` names a topic, subsystem, file or directory (e.g. `/whats-up auth`,
  `/whats-up plugins/obsidian`), narrow every section of the briefing to that scope, and
  say in the headline that the view is narrowed.

## Non-negotiables

- **Evidence before assertion.** Gather git and todo state *first*, then write. Anything
  you claim from memory alone gets tagged `(unverified)`.
- **Headline first.** The user must understand the situation from line one.
- **One concrete next step**, not a menu.
- **Stay inside the cap.** This is a briefing, not a report. If it doesn't fit, cut detail,
  never sections.
- **Read-only.** Never commit, push, stage, or modify files while answering this command.
