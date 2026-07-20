# herdr-goal-skill

A Claude Code skill that orchestrates a large multi-part objective across parallel lead agents inside a [Herdr](https://herdr.dev) session.

`/herdr-goal <objective>` decomposes the goal into independent workstreams, spawns one Claude Code lead per workstream (each in its own git worktree + Herdr pane, model picked by task complexity), monitors them until done, cleans up the worktrees, and reports back. It also updates its own SKILL.md with lessons learned after each run.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- Herdr with `HERDR_ENV=1` (must be invoked from inside a Herdr-managed pane)
- The `herdr` skill installed alongside (the CLI authority this skill builds on)

## Install

```sh
git clone git@github.com:ralphcrisostomo/herdr-goal-skill.git ~/.claude/skills/herdr-goal
```

Then invoke with `/herdr-goal <objective>` in any Herdr session.

## How it works

See [SKILL.md](SKILL.md) — that file is the skill.
