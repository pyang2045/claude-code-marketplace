# ticket close `[TICKET] [PR]`

Close out finished work in three steps, then report. Run the steps in order — each
depends on facts established by the previous one. Do not skip a step silently: if
one does not apply, say so in the final summary and why.

The `close` remainder may name a ticket key (SE-1234) and/or a PR number. If
absent, infer both from the current branch name and
`gh pr list --head "$(git branch --show-current)"`.

## Step 0 — Establish the facts first

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

**If the ticket has a PR stack, confirm EVERY PR in the stack is terminal
(merged or closed) before Step 3 removes any worktree.** A worktree removed
while a later PR in the stack is still open strands that PR's branch. List the
stack from the PR bodies (each carries the ordered stack list) and check each:

```bash
gh pr view <each-N> --json number,state,mergedAt,baseRefName
```

## Step 1 — Reconcile the docs

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

## Step 2 — Update the ticket

Comment on the ticket with:

- The merge commit SHA and PR number, or the reason the work was abandoned.
- What shipped, in the reader's terms rather than the diff's.
- **Any behavioural or data-shape change, called out separately and prominently**
  — anything QA or the phone/app side would notice. Name the commit that isolates
  it so it can be reverted alone.
- Known gaps carried forward.

A comment is **additive and safe** — write it without asking.

### Audit the description against what actually shipped

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

### Status

**Do not transition the ticket to Done reflexively.** Ask whether the ticket's
premise is actually satisfied:

- Work that *enables future diagnosis* (instrumentation, logging) does not close
  the bug it was written for — the root cause is still unknown. Say so and leave
  it open.
- Work fully verified on hardware, or a self-contained fix, can close.

When in doubt, comment and leave the status alone; say in the summary that you
did, so the human can decide. Also flag an auto-assigned priority that looks
wrong rather than silently accepting it.

## Step 3 — Close the worktree

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

## Step 4 — Report the summary

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
- **PR stack** — every PR in stack order with its final state (only when the
  ticket had more than one PR).

Do not claim a step succeeded without having seen it succeed. If a command
failed or a check was skipped, say so in the row.
