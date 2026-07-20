---
description: Close out the current task by writing a handoff summary into the Obsidian session log (updates this conversation's session note — even across midnight — or creates one).
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

Session notes follow the convention `YYYY-MM-DD HHmm <project> session`. A session may
**cross midnight or span multiple days**, so never assume the note is dated today — the
note's identity comes from the session, not the calendar. Resolve the target in this
priority order:

**4.1 — The note this conversation already owns (preferred, date-independent).**
If this conversation created or appended to a session note earlier (the SessionStart
workflow usually creates one), that exact note is the target — even if its date prefix is
yesterday or older. Verify it still exists before using it:

```bash
obsidian read file="<session note name from this conversation>" >/dev/null && echo FOUND
```

**4.2 — Newest still-open session note for the project (lookback, not "today").**
Only if the conversation has no known session note (e.g. context was compacted), search:

```bash
obsidian search query="<project> session" limit=10
```

Parse candidate names by their `YYYY-MM-DD HHmm` prefix and check each candidate's status:

```bash
obsidian property:read name="status" file="<candidate note name>"
```

Pick the **most recent** candidate whose `status` is not `done`, dated within the **last
3 days** — this is what catches a session that started before midnight. Ignore notes
already marked `done` (they were closed by a previous `/obs-close`).

**4.3 — Stale or missing.**
- If the only open candidates are **older than 3 days**, treat them as stale: create a new
  note (step 5b) and mention the stale open note(s) in the report so the user can close or
  clean them up.
- If nothing matches at all, create a new one (step 5b).

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
obsidian property:set name="end" value="$(date +'%Y-%m-%d')" type=date file="<existing session note name>"
```

Setting `end` records the session span: the note's `date` property is the start day, `end`
is the close day. For a same-day session they are equal; for a session that crossed
midnight they differ — that is expected and is how multi-day sessions stay in one note.

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
end: $(date +'%Y-%m-%d')
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

If a project note exists, add a wikilink to the session note under its sessions list.
Use the **exact name of the target note from step 4/5** — never rebuild the name from the
current timestamp (the target note's `HHmm` is its creation time, not the close time, so a
rebuilt name would be a broken wikilink):

```bash
obsidian search query="<project>" limit=5
# If an evergreen project note is found:
obsidian backlinks file="<session note name>"   # skip the append if the project note already links it
obsidian append file="<Project Note>" \
  content="\n- [[<session note name>]]"
```

Skip silently if no project note exists — an unresolved wikilink is acceptable, but don't
invent a project note here.

### 7. Ensure the handoffs browsing base exists (one-time, best effort)

This plugin ships a Bases view (`handoffs.base`) that lists every note tagged `#handoffs`.
Install it into the vault root the first time, so the user can browse all handoffs.

This step is **best effort**: it must never block or fail the handoff. On any problem
(vault root unresolvable, template missing, copy fails), skip it and mention the reason
in the report instead of erroring out.

Check and install with a **filesystem test**, in one guarded block:

```bash
VAULT="$(obsidian vault info=path)"
if [ -z "$VAULT" ] || [ ! -d "$VAULT" ]; then
  echo "BASE_SKIP: vault root not resolvable"
elif [ -f "$VAULT/handoffs.base" ]; then
  echo "BASE_EXISTS: $VAULT/handoffs.base"
elif [ ! -f "${CLAUDE_PLUGIN_ROOT}/templates/handoffs.base" ]; then
  echo "BASE_SKIP: plugin template missing"
else
  cp "${CLAUDE_PLUGIN_ROOT}/templates/handoffs.base" "$VAULT/handoffs.base" \
    && echo "BASE_INSTALLED: $VAULT/handoffs.base" \
    || echo "BASE_SKIP: copy failed"
fi
```

Pitfalls this layout avoids — do not "simplify" back into them:
- **Do not use `obsidian search` to test for the base.** Search only indexes notes, not
  `.base` files, so it reports "No matches" even when `handoffs.base` is already installed,
  which leads to a pointless install attempt.
- **Do not use `cp -n` inside an `&&` chain.** When the destination already exists,
  `cp -n` exits non-zero, which aborts the rest of the chain and reads as a failure even
  though nothing is wrong. Test existence with `[ -f ... ]` first, then plain `cp`.
- **Always quote `"$VAULT"`** — vault paths commonly contain spaces
  (e.g. `~/Documents/Obsidian Vault`).

Do not overwrite an existing `handoffs.base`. On `BASE_INSTALLED`, mention its location once
in the report so the user knows they can open it or embed it with `![[handoffs.base]]`.

### 8. Report

Tell the user, in one or two lines:
- Which note was **updated** or **created** (its exact name). If the session crossed
  midnight, note the span (e.g. "started 2026-07-19, closed 2026-07-20").
- That it is tagged `#handoffs` / `#sessions` for easy retrieval (e.g. `obsidian search query="handoffs"` or a `.base`).
- Any stale open session notes found in step 4.3, so the user can close or delete them.
