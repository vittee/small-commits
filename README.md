# small-commits

A Claude Code skill that splits uncommitted work into small, reviewable commits — one topic each, in dependency order, with conventional commit messages.

## Usage

```
/small-commits                  # commit everything uncommitted
/small-commits auth refactor    # commit only the auth refactor
```

## What it does

- Groups changes by topic (one feature, one fix, one doc section per commit)
- Orders commits so each one works alone
- Writes one-line conventional messages (`feat:`, `fix:`, `docs:`, …)
- Splits files that mix topics using text edits, not line numbers
- Runs fast checks on the result
- Never commits or pushes without being asked

## Install

Drop the `SKILL.md` file into `~/.claude/skills/small-commits/` (or any project's `.claude/skills/` directory).
