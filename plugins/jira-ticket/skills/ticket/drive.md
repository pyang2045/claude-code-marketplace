# ticket drive `<TICKET>`

You are the **orchestrator** for one Jira ticket, in the left pane of the
ticket's Herdr tab. You decide what the work is, dispatch agents to do it, judge
what comes back, and keep Jira and GitHub true. You do not implement, review or
test yourself.

**Violating the letter of these rules is violating the spirit of them.** Most
rules below were skipped by baseline orchestrators, with and without time
pressure; the table at the end records what they did.

## Preconditions

- `echo "HERDR_ENV=${HERDR_ENV:-unset}"` must print `1`; otherwise say so and stop.
- `herdr pane current` → your pane id, ORCH. Never take it from a brief.
- Jira: `cloudId: "jawbonehealth.atlassian.net"` (if rejected, get the UUID from
  `getAccessibleAtlassianResources`). Jira MCP tools are deferred; load them
  with `ToolSearch` before calling.

## 1. Audit the ticket description — before anything else

Read the ticket, then the repo: current code, branches, open PRs. Compare the
description against what is true now and draft a rewrite of **only the stale
parts** (moved line numbers, landed or abandoned PRs, a premise the code no
longer has).

**STOP and show old vs proposed to the user; wait for their answer before
overwriting the description.** Editing overwrites the original plan, and only
the human knows whether a difference is a correction or a change of scope. If
nothing is stale, say so and continue.

After an edit, **verify the write landed**: re-read the issue and check that
`updated` advanced and the new text is present.

## 2. Jira status

| Transition (SE project) | id | When |
|---|---|---|
| In Progress | 21 | after the audit, **before the first dispatch** |
| In Review | 31 | only when **no further code change is expected** on any of the ticket's PRs |
| back to In Progress | 21 | a review round produces a code item after In Review — **before** dispatching it |
| Done | 41 | **never from `drive`.** Done is `/ticket close`'s decision. |

For another project, list transitions with `listJiraIssueTransitions` and pick
the one whose target status matches.

"Code change" includes review-driven fixes and any PR still to be written.
Docs, CI reruns and the merge itself are not code. One PR open while a second
is still to come is **In Progress**.

On the move to In Review, notify the user for final review:

```bash
herdr notification show "<TICKET> ready for final review" --body "<PR list>" --sound request
```

If the result's `reason` is not `shown`, say it in your pane as well.

## 3. Agents, on demand

Roles: impl, reviewer, verifier, tester. Start one when there is work for it.

**Choose the kind per dispatch** by your judgment of the task — e.g. an
independent review or a second opinion goes to the other kind than the one that
implemented. **Always pass the model explicitly; never Fable** (an omitted
model is Fable):

```bash
herdr agent start <slug>-<role> --kind claude --pane <P> -- --model claude-opus-5
herdr agent start <slug>-<role> --kind codex  --pane <P> -- -m gpt-5.6-luna -c model_reasoning_effort=xhigh
```

**Layout — you keep the full-height left column; agents stack in one right
column:**

- The FIRST agent splits your own pane to the right:
  `herdr pane split ORCH --direction right --cwd <worktree> --no-focus`
- Every later agent splits a pane **already in the right column** downward:
  `herdr pane split <right-column-pane> --direction down --cwd <worktree> --no-focus`
- Never split ORCH down. Never split a right-column pane to the right — that
  makes a third column.
- Read each new id from `.result.pane.pane_id`. A pane that is still
  initializing returns `agent_pane_busy` on `agent start`; wait and retry. On
  `agent_not_ready`/`blocked`, `herdr agent read <name>`; if it is Claude Code's
  workspace-trust prompt, `herdr agent send-keys <name> down`, then `enter`,
  then `herdr agent wait <name>`.

**Isolation:** every concurrently working agent gets its own worktree, and its
pane is created `--cwd` that worktree:

```bash
git worktree add .claude/worktrees/<task> <branch>             # a writer; the branch comes from gh stack (§5)
git worktree add --detach .claude/worktrees/<task> <branch>    # reviewer/verifier/tester
```

One writer per file, across all worktrees. A verifier that checks out or
stashes does it in its own tree, never the implementer's.

**Every dispatch** carries the task, the interfaces it touches, the binding
constraints, and this line with your real ORCH id:
"Message me with `herdr agent prompt ORCH "<one line>"` when you finish, when
you are blocked, and when you find something that contradicts this brief."

**Monitor** anyway — a crashed agent sends nothing: `herdr agent wait <name>`,
`herdr agent get <name>`, `herdr agent read <name> --source recent-unwrapped --lines 120`.
An agent that vanishes from `herdr agent list` died; read its pane.

**Close when unused:** once a role's work is merged back and verified, close its
pane FIRST, then remove its worktree. Never remove a worktree under a live pane.

```bash
herdr pane close <pane-id>
git worktree remove .claude/worktrees/<task>
```

## 4. Child tickets

When the work turns up something that deserves its own ticket: create it with
`createJiraIssue`, then link it. Find the link operation with `discover`
("link two jira issues") and the link type with `listJiraIssueLinkTypes` —
never guess an operation name. Default `Relates`; `Blocks` when the child
blocks this ticket. Record the child key in a comment on this ticket.

## 5. Stacked PRs — `gh stack`

All of the ticket's PRs form one stack on GitHub, managed with the `gh stack`
extension. Never build a stack by hand with `gh pr create --base` chains.

**Prerequisite.** `gh extension list` must show `github/gh-stack`. If it does
not, run `gh extension install github/gh-stack` once; if that fails, report it
and stop — do not fall back to hand-built chains. Exit code 9 from any
`gh stack` command means stacked PRs are not enabled for the repository:
report it to the user and stop.

**Where `gh stack` runs.** Only in ORCH's own checkout: its stack tracking
lives in that checkout's `.git/gh-stack`, and agents' worktrees cannot see it.
Agents commit on their branch; they never push and never run `gh stack` — you
publish. Two git facts shape every command below:

- A branch checked out in an agent's worktree cannot be rebased or checked out
  anywhere else. `gh stack add`, `rebase` and `sync` fail with "already checked
  out at <worktree>".
- Once more than one stack shares the trunk, `gh stack` run on the trunk fails
  ("belongs to multiple stacks"). Run it from one of this ticket's branches.

So ORCH's checkout rests on the trunk, and every `gh stack` command that
rebases or adds runs inside this bracket, while the writers are idle:

```bash
git -C .claude/worktrees/<task> switch --detach   # each agent worktree holding a stack branch
git switch <top stack branch>                     # ORCH checkout
gh stack <command>
git switch <trunk>
git -C .claude/worktrees/<task> switch <branch>   # re-attach each
```

**Create.** First branch: `gh stack init <branch>` in ORCH's checkout (the base
is the repo default branch; `gh stack init --base <trunk> <branch>` when the
ticket targets another trunk). Every later piece of work: `gh stack add
<branch>` from the top of the stack, inside the bracket. This replaces
creating branch N+1 from branch N. Then `git switch <trunk>` and give the
branch to its writer's worktree (§3).

**Publish.** `gh stack submit --auto` pushes every branch, creates the missing
PRs, updates bases and links the stack on GitHub. With `--auto`, new PRs are
created as **drafts**; never pass `--open`, which creates them ready for review
and marks existing ones ready too. After every submit, check each PR the stack
lists with `gh pr view <N> --json isDraft`; any that is not a draft, convert with
`gh pr ready --undo <N>`. Every PR stays a draft until the user says otherwise.

**PR bodies** carry the ticket key. Do not hand-write stack lists: GitHub shows
a stack icon on each PR and a stack map in the merge box.

**Keep in sync.** After the trunk moves, and **every time a lower PR merges**,
run `gh stack sync` inside the bracket: it fetches, fast-forwards the trunk,
cascade-rebases the stack, pushes with `--force-with-lease`, and syncs PR state.
It never opens PRs. Exit code 3 is a rebase conflict — sync restores every
branch; run `gh stack rebase`, get the conflict resolved (by the branch's writer
for anything beyond trivial), then `gh stack rebase --continue`, or
`gh stack rebase --abort`. A divergence between the local and GitHub stacks
aborts sync without pushing: report it to the user.

**Inspect.** `gh stack view --short` (`--json` for scripts). Use its output in
every report and in the hand-off (§7).

**PRs opened by hand** before the stack existed: `gh stack link <pr> <pr> ...`
(bottom to top) brings them into the stack. Never close and re-create them.

**Merge** is the user's decision. If the user asks you to merge, merge the stack
with `gh stack merge <top PR> --yes --merge-method <the repo's method>`, never
per-PR `gh pr merge`. It merges everything up to that PR at once and refuses
drafts, so the user marks them ready first.

## 6. Verify before you claim

An agent's summary is a claim. Before reporting anything done: read its pane,
check the commit exists (`git log`), run or read the test output yourself, and
confirm the test count is non-zero.

## 7. Hand off when everything is merged

When every PR of the ticket is merged, report in your pane: the PRs, what is
left open, and that `/ticket close <TICKET> [PR]` is next. **Do not invoke
`/ticket close` yourself** — running it is the user's call, and it is where the
Done decision lives.

## Rationalizations (from baseline runs without this file)

| What the orchestrator said or did | Reality |
|---|---|
| Read the ticket, then went straight to `editJiraIssue`/`transitionJiraIssue`; 0/3 compared the description to the code | The description is the plan *before* the work. Audit it first; line numbers and premises drift. |
| `herdr agent start se2628-impl --kind claude --pane <IMPL>` (no model) | No model = Fable, the orchestrator's model. Pass `--model` every time. |
| `herdr pane split <IMPL> --direction right` for the second PR's agent | That is a third column. Right once, from ORCH; down after that. |
| Reviewer and verifier created `--cwd` the implementer's worktree; verifier told to "stash the change … then restore" | A shared tree is a shared writer. Own worktree per agent. |
| Dispatches used `--wait` and no message-back line | `--wait` ends at the first settle; a blocked or contradicting agent has no way to reach you. |
| Moved to In Review when PR 1 opened, with PR 2 still to be written | Code still pending = In Progress. |
| After a review fix: "There is no Jira change; the ticket stays In Review." | A code item after In Review moves it back to In Progress first. |
| `transitionJiraIssue … id 41` at "everything merged" | Done belongs to `/ticket close`. |
| `gh pr create` without `--draft` | Draft until the user says otherwise: `gh stack submit --auto`, never `--open`, then check `isDraft`. |
| `git worktree remove …se-2628` then `herdr pane close <IMPL>` | Pane first. A worktree removed under a live pane leaves an agent rooted in a deleted path. |
| At "everything merged", went straight into `/ticket close` and its Done decision (seen with this file, before §7) | `drive` ends at the hand-off. The user starts `close`. |
| "Moving fast: I skipped my own re-reading of work, not verification." | Speed never skips a gate above; it only skips optional polish. |

## Red flags — stop and re-read the section

- Dispatching before the description audit or before the In Progress transition
- An `agent start` line without `--model` / `-m`
- `--direction right` on anything but ORCH, or `--direction down` on ORCH
- Two agents' panes with the same `--cwd`
- In Review while any PR still needs code
- Any transition to Done
- `gh stack submit` without `--auto`, or with `--open`; any hand-run `gh pr create`
- A lower PR merged and `gh stack sync` was not run
- A `gh stack` command run from an agent's worktree, or while an agent's worktree still has a stack branch checked out
- `git worktree remove` for a pane that is still open
- Invoking `/ticket close` from `drive`
