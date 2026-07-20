---
description: Close out the current task by writing a handoff summary into the Obsidian session log (updates today's session note or creates one).
---

# /obs-close — Close task & write handoff

Summarize the work done in this session as a **handoff document** and write it into the
user's default Obsidian vault via the `obsidian` CLI. Find the proper existing log and
update it; if none exists, create one. The goal is that the next session (human or Claude)
can pick up from the note alone.

Follow the user's global Obsidian conventions (`~/CLAUDE.md`): flat root, plural tags,
`YYYY-MM-DD` dates, wikilink first mentions, append-never-overwrite for logs.

## Steps

### 1. Verify the vault is reachable

```bash
obsidian --version 2>/dev/null && echo "OBSIDIAN_OK" || echo "OBSIDIAN_UNAVAILABLE"
```

- If `OBSIDIAN_UNAVAILABLE` (Obsidian not open / CLI missing), tell the user and offer to
  fall back to writing the note file directly with `cat`/`tee`. Do not silently skip.

### 2. Determine the project name

Infer the project from context, in this priority order:
1. The current working directory basename / git repo name (`basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)"`).
2. What the conversation was actually about.

Normalize to a short slug for tags (lowercase, hyphenated). Call it `<project>`.
If genuinely ambiguous, ask the user for the project name once — otherwise proceed.

### 3. Compose the handoff summary

From the conversation, write a concise handoff. Prefer specifics (file paths, commands,
decisions) over generic recap. Use this exact block, filling every field (write "none"
if empty — never leave a placeholder):

```markdown
## Handoff — <YYYY-MM-DD HHmm>
**Done:** <what was accomplished this session>
**Current state:** <where things stand right now — branch, build/test status, what works>
**Next steps:** <the concrete next actions to take>
**Open questions:** <unresolved decisions / blockers>
**Key files / links:** <paths, PR/branch, related [[notes]]>
```

Get the timestamp with `date +'%Y-%m-%d %H%M'`.

### 4. Find the proper session log to update

Session notes follow the convention `YYYY-MM-DD HHmm <project> session`. The SessionStart
workflow usually already created one today. Search for it:

```bash
obsidian search query="<project> session" limit=10
```

Choose the target note:
- Prefer a session note for **today** (`date +'%Y-%m-%d'`) matching this `<project>`.
- If several match today, pick the most recent (highest `HHmm`).
- If none match today, you will create a new one (step 5b).

### 5a. If an existing session note is found — append the handoff

Never overwrite. Append the handoff block and mark the note done:

```bash
obsidian append file="<existing session note name>" \
  content="\n$(cat <<'EOF'
## Handoff — <YYYY-MM-DD HHmm>
**Done:** ...
**Current state:** ...
**Next steps:** ...
**Open questions:** ...
**Key files / links:** ...
EOF
)"

obsidian property:set name="status" value="done" file="<existing session note name>"
```

Ensure the note carries the association tags (add `handoffs` if missing). If the CLI has no
tag-add command, note it in the appended body via an inline `#handoffs` reference.

### 5b. If no session note exists — create one

Create with the standard frontmatter so it is easy to associate later:

```bash
obsidian create \
  name="$(date +'%Y-%m-%d %H%M') <project> session" \
  content="---
title: <project> session
date: $(date +'%Y-%m-%d')
categories:
  - sessions
tags:
  - <project>
  - handoffs
  - sessions
status: done
---

## Handoff — $(date +'%Y-%m-%d %H%M')
**Done:** ...
**Current state:** ...
**Next steps:** ...
**Open questions:** ...
**Key files / links:** ...
" \
  silent
```

### 6. Link from the project evergreen note (best effort)

If a project note exists, add a wikilink to the session note under its sessions list:

```bash
obsidian search query="<project>" limit=5
# If an evergreen project note is found:
obsidian append file="<Project Note>" \
  content="\n- [[$(date +'%Y-%m-%d %H%M') <project> session]]"
```

Skip silently if no project note exists — an unresolved wikilink is acceptable, but don't
invent a project note here.

### 7. Ensure the handoffs browsing base exists (one-time)

This plugin ships a Bases view (`handoffs.base`) that lists every note tagged `#handoffs`.
Install it into the vault root the first time, so the user can browse all handoffs. Skip if
it already exists.

```bash
# Is it already in the vault?
obsidian search query="handoffs.base" limit=3
```

If it is **not** present, copy the template into the vault root. The template lives at
`${CLAUDE_PLUGIN_ROOT}/templates/handoffs.base`. Determine the vault root (from the
`obsidian` CLI config, or ask the user once if it cannot be resolved), then:

```bash
cp "${CLAUDE_PLUGIN_ROOT}/templates/handoffs.base" "<vault-root>/handoffs.base"
```

Do not overwrite an existing `handoffs.base`. Mention its location once in the report so the
user knows they can open it or embed it with `![[handoffs.base]]`.

### 8. Report

Tell the user, in one or two lines:
- Which note was **updated** or **created** (its exact name).
- That it is tagged `#handoffs` / `#sessions` for easy retrieval (e.g. `obsidian search query="handoffs"` or a `.base`).
