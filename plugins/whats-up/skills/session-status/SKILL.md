---
name: session-status
description: This skill should be used when the user asks "what's up", "where are we", "catch me up", "what's the status", "what's left", "what should I do next", "remind me what we were doing", or runs /whats-up. Also applies when resuming a session after a break or a context compaction, before handing work to someone else, or when the user sounds lost about the current state. Briefs the user on where work stands — progress, in-flight work, remaining tasks, the single next step, blockers and suggested follow-ups.
---

# Session status briefing

Turn the current state of the work into something the user can absorb in five seconds.

The failure mode this skill exists to prevent is **the confident summary from memory** — a
recap that says "done" about something that was only discussed, or lists next steps that
were superseded twenty messages ago. Gather evidence first. Write second.

## Step 1 — Gather evidence (before writing anything)

Run these read-only checks in **one batched block**. All are best-effort: a failure here is
a missing input, never a reason to stop.

```bash
git branch --show-current 2>/dev/null
git status --short 2>/dev/null
git log --oneline -8 2>/dev/null
git stash list 2>/dev/null
```

Then, still best-effort:

```bash
gh pr status 2>/dev/null | head -30
```

If the working directory is not a git repo, say so in one clause and continue — the
conversation and todo list still carry the session.

Then read the **todo/task list** if the session has one. It is the most direct statement of
remaining work that exists; prefer it over inference.

Only now revisit the **conversation**: decisions made, approaches abandoned and why,
files touched, questions the user asked that were never answered.

If the plugin's host session has an Obsidian session note or handoff in play *and it is
already in context*, use it. Do not go hunting for one — this skill must work in any repo.

## Step 2 — Classify, and mark whatever cannot be proven

Sort every item gathered into four buckets:

| Bucket | Test |
|---|---|
| **Done** | There is a commit, a merged PR, or a file on disk that proves it |
| **In flight** | Started and visible in `git status` / an in-progress todo, not finished |
| **Remaining** | Agreed on, not started |
| **Dropped** | Considered and rejected — include only if the user might otherwise re-raise it |

Two rules govern this step:

- **Evidence beats recall.** When something seems done but nothing in git, the todo list, or
  the filesystem shows it, keep the claim and append `(unverified)`. Never quietly upgrade a
  memory into a fact.
- **Uncommitted is not done.** Work sitting in `git status` is *in flight*, however finished
  it feels. Say what remains: review, tests, commit, push, PR.

## Step 3 — Write the briefing

Use this template exactly. Sections stay in this order; a section with nothing in it prints
`— none` and is never dropped or filled with a placeholder.

```markdown
🔵 **<one sentence: what we are doing and where it stands right now>**
<branch · working-tree state · PR link, if a repo>

**Done**
- ✅ <what> — <the commit sha / PR / file that proves it>

**In flight**
- 🔨 <what> — <where exactly it stopped, and what unblocks it>

**Remaining**
- ⬜ <concrete, actionable — not a theme>

**Next step**
→ <the single next action, specific enough to start without another decision>

**Blockers / open questions**
- ⚠️ <what is waiting on the user, or on something outside this session>

**Follow-ups worth considering**
1. <something nobody asked for but that this work makes obvious>
```

### Rules for the writing

- **The headline carries the answer.** If the user reads only the first line, they should
  know whether things are on track, blocked, or nearly finished. Never open with a restated
  question or a preamble.
- **Cap it.** Roughly 20 lines and at most 5 bullets per section. When there is more than
  that, keep the highest-stakes items and end the section with `- …and N more`. Cut detail,
  never sections.
- **Be specific.** `Fix auth` is not a task; `Add the 401 retry to api/client.ts:88` is.
  Every line should name a file, command, PR, or decision.
- **One next step.** When parallel options genuinely exist, put the recommendation in
  **Next step** and demote the rest to **Follow-ups**. Never hand the user a menu — that
  pushes the decision back onto them, which is the thing they asked to be spared.
- **Follow-ups are suggestions, not commitments.** Things the work makes obvious: a missing
  test, a version bump, a stale doc, a TODO left in the diff. Never start one unprompted.
- **Declare thin context.** After a compaction or a long gap, lead with what is verifiable
  and state plainly that earlier context is gone, rather than reconstructing it confidently.

### Scope narrowing

When invoked with a topic (`/whats-up auth`), filter every section to that scope, and make
the narrowing visible in the headline — e.g. *"Narrowed to `plugins/obsidian`."* Anything
excluded stays excluded; do not smuggle unrelated items into **Follow-ups**.

## Boundaries

- **Read-only.** This skill never commits, pushes, stages, edits files, or starts work. It
  reports. If the next step is obvious and cheap, still stop and offer it.
- **Not a handoff note.** This is an in-terminal briefing for the person already here.
  Writing a durable handoff into a session log is `/obs-close`'s job — mention it if the
  user sounds like they are wrapping up for the day.
