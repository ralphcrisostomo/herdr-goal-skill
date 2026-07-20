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

### 2. Spawn one lead per workstream

Check every target repo is a Git repo before spawning — `worktree create` needs one. If a repo is uninitialized, `git init` it, confirm what belongs in `.gitignore` (generated output, scraped data, `node_modules`), and make an initial commit; that commit is the pre-work SHA for the review gate. Tell the user before initializing.

A worktree checkout excludes gitignored paths, so a lead gets no `node_modules` and no generated data. Put the install command and the absolute path to any generated fixtures in the main checkout into the brief.

Discover current syntax with `herdr worktree` (bare group prints usage; never probe `worktree create` with no args — it executes). If the printed usage differs from the commands below, the printed usage wins. Then:

```bash
herdr worktree create --cwd <repo> --branch herd/<workstream> --label "<workstream>" --no-focus --json
```

Parse the returned workspace and pane IDs from the JSON — never construct them.

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
- If the brief never registers (lead still `idle` 30s after submitting it) → `pane read` to see what happened, resend once, then escalate. Expect this on the FIRST brief: the idle event fires while the TUI is still initializing, and a brief sent immediately is usually swallowed — always confirm `working` before trusting a brief landed.
- A completed lead frees a concurrency slot for a queued workstream.

### 4. Complete a lead

```bash
herdr pane read <pane-id> --source recent-unwrapped --lines 200   # capture the summary
```

Confirm the branch is committed (ask the lead to `git status` if the summary doesn't say). An uncommitted worktree removed is lost work.

If a review gate follows (see the user's CLAUDE.md), keep the lead's pane alive until the reviewer returns clean. Review rounds are iterative, and a lead that still holds its own context fixes its own findings far more cheaply than a fresh one re-derives them — send each round's findings via `pane run`. Hand findings over as a file written into the worktree rather than as one giant `pane run` line; per-item file:line detail does not survive being flattened into a single argument.

When the work is genuinely finished:

```bash
herdr worktree remove --workspace <lead-workspace-id> --json
```

The branch survives worktree removal. Only remove workspaces/panes this orchestration created.

A lead spawned as a sibling pane (step 2's already-checked-out-branch case) has no linked workspace — `worktree remove` doesn't apply. Close it directly instead: `herdr pane close <pane-id>`. This is easy to forget precisely because it doesn't fit the worktree-remove muscle memory — after reading a sibling-pane lead's final summary, closing its pane is the explicit next action, not an implicit one.

To reattach a branch whose worktree was already removed, use `worktree create --branch <existing-branch>` — it checks the branch out again. `worktree open --branch` fails with `worktree_not_found`; it opens an existing worktree directory, not a bare branch.

### 5. Final report

One summary to the user: per workstream — outcome, branch name, verification status, open issues — plus a suggested merge order.

### 6. Learn — update this skill

After the final report, review the run for surprises: CLI behavior that differed from this document, waits that timed out, briefs that failed to land, recoveries that worked, anything you had to figure out live. For each one that would change how the NEXT run behaves, edit this SKILL.md immediately:

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
