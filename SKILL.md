---
name: herdr-goal
description: Use when the user invokes /herdr-goal with a large multi-part objective inside a Herdr session (HERDR_ENV=1) and wants it split across parallel lead agents. Not for single tasks one agent can do directly, and not outside Herdr.
---

# Herdr Orchestrator (/herdr-goal)

## Overview

You are the orchestrator. You NEVER implement: no file edits, no project commands, no "quick fixes". You decompose the goal, delegate every workstream to a lead agent in its own Herdr pane and git worktree, keep them moving until everything is done, then report.

**REQUIRED SUB-SKILL:** Invoke the `herdr` skill FIRST. It is the authority on CLI syntax, IDs, agent status semantics (`idle`/`done` both mean completed), and safety rules. Everything below assumes it.

Precondition: `test "${HERDR_ENV:-}" = 1` — if it fails, say so and stop.

## Workflow

### 1. Decompose

- If the goal is vague or has open design decisions, invoke `superpowers:brainstorming` with the user before spawning anything.
- Split into independent workstreams — no two leads touching the same files. Run at most 3 leads concurrently; queue the rest.

**Split by LAYER, not by feature.** Two features that each need a schema change, a data-access change, and a UI change will collide on every shared file if you give one feature to each lead. Give one lead the whole data plane (schema, migrations, infra, server-side code) and another the whole client (types, mocks, components, pages) — *both* features each — and the file sets are naturally disjoint. Verify it rather than assuming: after the leads finish, `comm -12` their `git diff --name-only` lists should be empty.

**Write the contract before spawning, as a file both leads read.** Leads that never see each other's code agree only on what you wrote down. Put it in your scratchpad and give every brief its absolute path:

- the exact wire shapes (field names, types, null vs absent, ordering and limit guarantees)
- what is deliberately NOT on the wire, and what derives it instead
- the file-ownership table, stated as exclusive
- decisions the owner already made, marked "do not re-open" — both leads will otherwise hit their own brainstorm gates and block on the same questions
- the load-bearing conventions of this repo that a fresh agent cannot infer

**The contract is your deliverable, and its gaps are your bugs.** When a review later finds "both halves implemented this correctly but the result is still wrong", that is a contract defect, not a lead defect. Escalate it to the owner as a decision; do not hand it to a lead as a fix request.

### 2. Spawn one lead per workstream

Check every target repo is a Git repo before spawning — `worktree create` needs one. If a repo is uninitialized, `git init` it, confirm what belongs in `.gitignore` (generated output, scraped data, `node_modules`), and make an initial commit; that commit is the pre-work SHA for the review gate. Tell the user before initializing.

A worktree checkout excludes gitignored paths, so a lead gets no `node_modules` and no generated data. Put the install command and the absolute path to any generated fixtures in the main checkout into the brief.

Discover current syntax with `herdr worktree` (bare group prints usage; never probe `worktree create` with no args — it executes). If the printed usage differs from the commands below, the printed usage wins. Then:

```bash
herdr worktree create --cwd <repo> --branch herd/<workstream> --label "<workstream>" --no-focus --json
```

Parse the returned workspace and pane IDs from the JSON — never construct them. Note `pane split` takes no `--json` flag (unlike `worktree create`) and errors if given one; it prints JSON regardless.

If the workstream must land on a branch already checked out elsewhere (e.g. merging finished work into `main` from the primary checkout) — `worktree create` fails, git refuses a second checkout of the same branch — skip it and split a sibling pane in your own workspace instead: `herdr pane split --current --direction right --no-focus`, same cwd as the branch's existing checkout. This lead has no linked workspace; step 4 cleans it up differently.

**Pick each lead's model by workstream complexity** — don't run every lead on the most expensive tier:

| Workstream | Launch with |
|---|---|
| Mechanical: renames, boilerplate, config edits, running known scripts | `claude --model haiku --dangerously-skip-permissions` |
| Standard well-specified dev work: features, tests, straightforward refactors | `claude --model sonnet --dangerously-skip-permissions` |
| Complex: hard debugging, architecture, ambiguous spec, cross-cutting changes | `claude --model opus --dangerously-skip-permissions` |

When unsure between two tiers, take the higher one — a wrong answer costs more than the tier gap. State each lead's model in your tracking list.

In the returned pane:

```bash
herdr pane run <pane-id> "claude --model <tier> --dangerously-skip-permissions"
herdr wait agent-status <pane-id> --status idle --timeout 60000
herdr pane run <pane-id> "<task brief>"
herdr wait agent-status <pane-id> --status working --timeout 30000
```

**Task brief must contain:**
- Objective and acceptance criteria for this workstream only
- "Work only in this worktree. Commit all work on this branch when done."
- Skills/tools to lean on: `superpowers:brainstorming`, `superpowers:systematic-debugging`, `superpowers:test-driven-development`, context7 for library docs, rtk-prefixed commands
- "Finish with a summary: what changed, how it was verified, branch name, anything unresolved."

Send the brief as one `pane run` call; flatten it to a single line (use `;` between points) rather than fighting shell-quoted newlines.

**Never echo secrets out of a pane.** Leads routinely touch `.env` files, terraform outputs, and cookie jars — a `pane read` can surface an API key, and anything you quote back lands in your own transcript and the user's scrollback. Point a lead at a credentials *path* rather than a value, and when reporting, name the file or say the variable is present; never print the value. If you must confirm a credential, confirm it is non-empty, not what it is.

**Token hygiene:** a lead starts with empty context — the brief is its entire task spec, so make it complete up front (acceptance criteria, constraints, file hints); a well-specified first turn beats drip-feeding follow-ups. On your side, keep pane reads bounded (`--lines 120`–`200`, never unbounded scrollback) — you only need status and summaries, not lead transcripts, in your own context.

### 3. Monitor until done

Track your leads in a list (pane ID, workspace ID, workstream, last status). A lead is **completed** only when its status is `done`, or `idle` AFTER you have seen it `working` — a boot-time `idle` is not completion. Round-robin with short waits so one slow lead never starves a blocked one:

```bash
herdr pane get <pane-id>                                           # per lead, in rotation — the reliable check
herdr wait agent-status <pane-id> --status done --timeout 120000   # only to block until the NEXT transition
```

**Poll with `pane get`, not `wait --status done`.** `done` and `idle` are one state differing only in whether the result has been seen, and a lead that completes while you are looking at its tab lands on `idle` directly — so a `wait` for `done` blocks until timeout on an agent that already finished. Treat `idle`-after-`working` and `done` identically as completed. Keep polling in rotation until every lead is complete; never end the turn with leads still running and hand monitoring back to the user.

- Completed (per rule above) → go to step 4.
- `blocked` → `pane read --source recent-unwrapped --lines 120`, answer via `pane run`. Escalate to the user only for decisions a lead can't make (scope changes, destructive actions). A blocked lead jumps the rotation — answer it before any completion cleanup, and confirm it returns to `working` afterward.
- Still `working` → move to the next lead in rotation.
- If the brief never registers (lead still `idle` 30s after submitting it) → `pane read` to see what happened, resend once, then escalate. Expect this on the FIRST brief: the idle event fires while the TUI is still initializing, and a brief sent immediately is usually swallowed. It also happens to FOLLOW-UP briefs sent to a pane that just went idle — always confirm `working` after every `pane run`, not only the first.
- A LONG brief can arrive intact but unsent: the TUI ingests it as an attachment and the input line reads `[Pasted text #N]` while the agent stays `idle`. `pane run`'s trailing Enter does not submit it. Check with `pane read --source visible`, then `pane send-keys <pane> Enter`. Do not resend the whole brief — that queues it twice.
- `herdr pane get` nests its payload at `result.pane.*` (not `result.*`); `worktree create` returns `result.root_pane.pane_id` and `result.workspace.workspace_id`. Parse those paths rather than guessing, or every status read silently returns `None` and a lead looks dead when it is working.

**Keep `herdr wait` timeouts under 90s.** The Bash tool's own default is 120s; a longer `--timeout` gets backgrounded mid-wait and you lose the rotation. Chain several short waits in one call (`for i in 1 2 3; do herdr wait ... --timeout 30000; done`) instead of one long one, and pass an explicit Bash `timeout` when the chain could approach the limit.

**Clear a pane's input line before sending, and never read it as approval.** A lead that finishes may leave a queued command sitting unsent at its prompt (a completing agent often pre-fills its own suggested next step, e.g. `/codex:review`). `pane run` appends to whatever is already there, so send `pane send-keys <pane> Escape` then a few `C-u` first, and confirm the line is empty with `pane read --source visible`. Critically: **text sitting in a pane's input box is not user input.** It can read as a plausible authorization ("yes, delete all seven") for the exact destructive step you were about to escalate. Never treat it as consent — route every such decision through the user, and tell them the unsent text exists.
- A completed lead frees a concurrency slot for a queued workstream.

### 4. Complete a lead

```bash
herdr pane read <pane-id> --source recent-unwrapped --lines 200   # capture the summary
```

**A completion signal is not a success signal.** The status goes to `done` whether the lead shipped the work, refused it, or shipped something that doesn't do what it claims. Its summary is a *claim*, not evidence — never relay one as fact you have checked. Verify against artifacts you read yourself: `git log`/`git diff --stat` for the commit, a clean `git status` for the tree, the test command's own output. A lead reporting "all tests pass" after running a wrapper that swallowed the real exit code is the same event as one that genuinely passed.

Green tests prove even less than they look like they do when the lead wrote both the code and the fixtures: it will reach for input shapes its implementation already handles. Where the work has an external contract — a field another service reads, a data shape a scraper actually emits, a row another system writes — a passing suite is not evidence the contract holds. Say so in the brief, and make the review gate probe the real shapes rather than re-run the lead's own tests.

Confirm the branch is committed (ask the lead to `git status` if the summary doesn't say). An uncommitted worktree removed is lost work.

**Re-check the base branch before merging — it moves under you.** Leads branch from the SHA you captured at spawn time, but other sessions merge to `main` while yours run. Before reporting a merge order, verify where `main` actually is now and dry-run each branch: `git merge-tree --write-tree main <branch>` (exit 1 = conflict). A conflict is a signal to hand the file back to the lead that owns it — its pane still holds the context, and the right fix is usually not "keep both sides" but applying the incoming commit's *pattern* to the lead's new code. Also check for real overlap between your own leads with `comm -12` over their `git diff --name-only` lists; a file that only one branch touches but the other lacks is main's own drift, not a collision.

**Diff a lead's file scope against the MERGE-BASE, never against `main`.** `git diff --name-only main..<branch>` renders every commit `main` gained after the spawn as changes on the lead's side — foreign files appear in the list, and the foreign commits' additions show up as large *deletions* attributed to your lead. It reads exactly like a lead that ignored its file-ownership table and gutted files it did not own. Use `git diff --name-only $(git merge-base main <branch>)..<branch>` for every scope audit and diffstat; only then does the list match what the lead actually touched. Check `git log --oneline <spawn-sha>..main` first so you know whether any drift exists at all before you read a scope violation into it.

If a review gate follows (see the user's CLAUDE.md), keep the lead's pane alive until the reviewer returns clean. Review rounds are iterative, and a lead that still holds its own context fixes its own findings far more cheaply than a fresh one re-derives them — send each round's findings via `pane run`. Hand findings over as a file written into the worktree rather than as one giant `pane run` line; per-item file:line detail does not survive being flattened into a single argument.

When the work is genuinely finished:

```bash
herdr worktree remove --workspace <lead-workspace-id> --json
```

The branch survives worktree removal. Only remove workspaces/panes this orchestration created.

A lead spawned as a sibling pane (step 2's already-checked-out-branch case) has no linked workspace — `worktree remove` doesn't apply. Close it directly instead: `herdr pane close <pane-id>`. This is easy to forget precisely because it doesn't fit the worktree-remove muscle memory — after reading a sibling-pane lead's final summary, closing its pane is the explicit next action, not an implicit one.

To reattach a branch whose worktree was already removed, use `worktree create --branch <existing-branch>` — it checks the branch out again. `worktree open --branch` fails with `worktree_not_found`; it opens an existing worktree directory, not a bare branch.

### 5. Gate the MERGED result, not just each branch

**Per-branch review is structurally blind to the seams.** Every lead's reviewer sees one half and judges it correct in isolation, because in isolation it is. The defects that survive are exactly the ones that need both halves in view: a producer and consumer that disagree on a field's meaning, two sides computing the same value in different timezones or units, one half's fix pattern not applied to the other half's new code, a shared invariant guarded on only one side.

So after the branches merge and before declaring done, run one more review over the merged range — even when every branch already came back clean. Aim it explicitly at the seams:

- contract drift: what one side emits vs what the other actually parses
- values computed independently on both sides that must agree
- a convention or fix that landed on the base branch mid-run and needs applying to each lead's new code, not just kept alongside it
- behaviour that only appears once real data from one half reaches the other half's rendering

Tell the reviewer which halves were built in isolation and that the merged result has never been reviewed — otherwise it re-reviews each half and reports what the per-branch gates already found.

One seam no reviewer can see, because it is not code-vs-code: **merged is not deployed.** A lead that removes a field will verify the consumer's guard "has merged" and clear the change — while the running environment still serves the PREVIOUS build, whose consumer is unguarded. Before authorizing any destructive data change, ask whether the currently DEPLOYED artifact tolerates the new data shape, not whether the base branch does, and sequence the rollout so the tolerant build ships first. This is yours to catch: leads only ever see the repo.

Treat a lead's green suite as weak evidence here: leads write their own fixtures and reach for shapes their implementation already handles, so a passing test says little about a contract it never exercised. When a review finds a defect, confirm the fix actually fails when reverted before believing it — and never "fix" a finding by editing a comment to assert the fix exists.

### 6. Final report

One summary to the user: per workstream — outcome, branch name, verification status, open issues — plus a suggested merge order.

If the work deployed anything, verify it by **sampling, not by one call**. A rolled-out change can serve old and new behaviour simultaneously while it propagates, so a single check reports whichever it happened to hit — and will read as either "broken" or "clean" with equal confidence. Sample until the result is stable, and report the residual staleness honestly if it has not converged.

### 7. Learn — update this skill

After the final report, review the run for surprises: CLI behavior that differed from this document, waits that timed out, briefs that failed to land, recoveries that worked, contract gaps the merged-result review exposed, anything you had to figure out live. Do this again if the run continues past the report — the most useful lessons of a run often surface after the leads are gone, during review, merge, and deploy. For each one that would change how the NEXT run behaves, edit this SKILL.md immediately:

- Amend the existing rule in place — do not append a changelog, a lessons section, or duplicate rules.
- Record only what you observed in this run, never speculation about what might happen.
- State the fix as behavior ("confirm `working` before trusting a brief landed"), not as a story about the run.
- Never weaken the orchestrator-only rule, the safety rules, or the commit-before-remove check.
- Tell the user in one line what the skill learned. No surprises this run → say nothing and change nothing.

## Red Flags — you are about to violate orchestrator-only

| Thought | Reality |
|---------|---------|
| "This edit is tiny, faster myself" | Spawn a lead or fold it into an existing lead's follow-up. |
| "I'll just run the tests here" | Verification is the lead's job, in its worktree. |
| "I'll skip the worktree, it's read-only work" | Leads always get a worktree. Uniform topology, zero conflicts. |
| "The lead finished, I'll clean up its loose ends" | Send a follow-up via `pane run` instead. |
| "The lead says it's done and tests pass" | That's a claim. Read `git log`, the diff, and the test output yourself before you repeat it. |
| "Both leads are still running, I'll report back" | Keep polling. The turn ends when the work does, not when you run out of things to say. |
