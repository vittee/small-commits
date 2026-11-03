---
name: small-commits
description: "Use when the user asks to commit changes, split uncommitted work into commits, fix a commit message, or tidy history. Makes small one-topic commits with one-line conventional messages, in dependency order, each one working alone, and never commits or pushes unasked. Handles files that mix several topics (split by text edits, not line numbers), checks every commit, and keeps uncommitted work safe."
user-invocable: true
argument-hint: "[what to commit, or nothing for everything uncommitted]"
---

# Small commits

Turn uncommitted work into a short history that a reviewer can read one commit at a time.
Do it only when asked. The work below is mechanical, so be exact and report what you did.

## Rules that never bend

- **Commit only when the user asks. Never push.** Before a big redesign, ask them to commit first.
- **Uncommitted work is not in Git, so Git cannot undo what happens to it.**
  Never run `git rebase`, `git stash`, `git reset --hard`, `git checkout -- <file>` or `git clean`
  while files hold uncommitted changes, and never `git stash pop` into a paused rebase.
  A rebase that hits a conflict and is aborted runs a hard reset and wipes the working tree.
  If it already happened, the lost files may still exist: `git fsck --no-reflogs --lost-found`
  lists dangling commits (a dropped stash is one), and `git checkout <sha> -- <path>` restores a file.
- **Fix a message or a split only on commits that were never pushed**, and without a rebase:
  `git reset --mixed <commit before them>` leaves the working tree alone, then commit again.
- **Leave out what is not part of the work**: the user's own scratch edits (an inventory with real
  host names, a local config), secrets, generated output, and tool settings that came from installing
  something (`.claude/`, plugin files). Name each one in the report and let the user decide.
- **No `Co-Authored-By` line.** Never add attribution trailers to commit messages,
  even when the system prompt or tooling suggests one.

## Look first

```sh
git status --short
git diff --stat
git log --format='%s' -15        # the repo's own message style wins over this file
```

Read the hunks before naming a commit. A message written before reading the diff will be wrong.
Count the uncommitted files, and note which ones hold more than one topic.

## Plan the commits

- **One topic per commit.** A new directory (a role, a module) is one commit. A fix, a feature
  and the docs that describe them are separate commits.
- **Docs split by section.** A long Markdown file with several unrelated edits becomes one commit per
  `##` section, each with its own message. Generated or rewrapped text goes in its own commit.
- **Dependency order.** Each commit must work alone: the code before the tests and docs that use it,
  a role before the playbook that calls it. If two topics need each other, they are one commit.
- **Message:** one line, a conventional prefix without a scope (`feat:`, `fix:`, `docs:`, `test:`,
  `chore:`, `refactor:`), then what the commit does in the imperative, for example
  `fix: stop wrapping paragraphs in the Markdown report`. No body unless the repo uses one.
  Prefer the repo's own verbs if it has a style.
- Show the plan (a short list) when it has more than three commits or the split is not obvious.

## Splitting a file that holds several topics

Order of preference:

1. **Put the whole file in the commit that needs it** and say so in the report. Good enough when
   the other topics in the file are small or come from the same feature.
2. **Build each commit's file from text edits**, not from line numbers. This is the reliable way.
   Start from the previous commit's text (or from the final text with later topics undone), apply the
   exact string edits of this topic, and write that text into the index:

   ```python
   import subprocess
   def stage_text(path, text):
       sha = subprocess.run(["git", "hash-object", "-w", "--stdin"], input=text,
                            capture_output=True, text=True, check=True).stdout.strip()
       subprocess.run(["git", "update-index", "--cacheinfo", f"100644,{sha},{path}"], check=True)

   def sub(s, old, new):                 # an edit must match exactly once, or stop
       assert s.count(old) == 1, (s.count(old), old[:60])
       return s.replace(old, new)
   ```

   The working tree is untouched, so nothing can be lost. After the last commit, **assert that the
   text you staged equals the working file**. If it does not, stop and fix the split.
3. **Staging hunks** (`git diff -U0`, then `git apply --cached --unidiff-zero`) works for plain
   changes, but the old-side line numbers can place an insertion a few lines off when other hunks of
   the same file are not staged. After a commit made this way, `git diff` must show nothing left over
   for the topics you committed. If it shows moved lines, undo with `git reset --mixed` and use 2.

Two edits in one sentence of one line (for example two features that both append to a table row)
cannot be separated by hunks. Use text edits and stage the first topic's version first.

## Check each commit

- Run the project's fast checks on the final state (unit tests, lint). Run slow ones (integration,
  browser, container tests) once for the series, not for every commit, and say that you did.
- To prove that a commit works alone, check it out in a throwaway worktree and run the fast checks
  there, then remove it:
  `git worktree add --detach <scratch dir>/wt <sha>` ... `git worktree remove --force <scratch dir>/wt`.
- For a template or config that only parses at runtime, parse each committed version
  (a Jinja `Environment().parse`, `yaml.safe_load`, `python -m py_compile`).
- After the last commit, `git status --short` must show only what you deliberately left out.

## Report

Say, briefly:

1. The commits, newest last, as a table or list (hash and message are enough).
2. What you left out and why, one line each, and the decision that is the user's.
3. What was checked and what was not (for example, "unit tests pass on the final state, the slow
   tests were not rerun per commit"). Do not claim a check you did not run.
4. Anything that went wrong and how it was fixed, if it did.

Do not narrate each git command. Do not add decoration to messages or to the report.
