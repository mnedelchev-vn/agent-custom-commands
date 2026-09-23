---
argument-hint: "[--commit-type=batch|separate] [--commit-message=<msg>]"
description: Add, commit, and push pending changes in one commit (batch) or one commit per file (separate)
disable-model-invocation: true
allowed-tools: Bash(git status:*), Bash(git rev-parse:*), Bash(git symbolic-ref:*), Bash(git branch:*), Bash(git add:*), Bash(git commit:*), Bash(git push:*), Bash(git diff:*), Bash(git log:*)
---

## Context
- Current branch: !`git symbolic-ref --short HEAD 2>/dev/null || git rev-parse --abbrev-ref HEAD`
- Upstream: !`git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null || echo "NONE"`
- Has any commits: !`git rev-parse --verify HEAD >/dev/null 2>&1 && echo "YES" || echo "NO"`
- Pending changes (porcelain):
!`git status --porcelain`

## Task
Push the pending changes shown above. Behavior is controlled by the argument string: `$ARGUMENTS`

### Argument parsing
Parse two recognized flags out of `$ARGUMENTS`. Flags may appear in any order.
1. **`--commit-type`**: if `$ARGUMENTS` contains `--commit-type=separate`, use **SEPARATE** mode. Otherwise (including `--commit-type=batch`, an unrecognized value, or the flag being absent), use **BATCH** mode. Batch is the default.
2. **`--commit-message`**: locate `--commit-message=` in `$ARGUMENTS`. Its value is everything after the `=`, up to either the start of the other recognized flag (`--commit-type=`) if that flag appears later in the string, or the end of the string otherwise. Trim surrounding whitespace and a single matching pair of surrounding quotes (`"..."` or `'...'`).
    - If a non-empty value is found, that is the **provided message**. Do NOT generate a message; use the provided message exactly as given.
    - If the flag is absent or its value is empty, there is no provided message; you must **generate** one (see each mode below).

### Preconditions (both modes)
1. If there are no pending changes in the porcelain output, stop and report "Nothing to commit — working tree clean." Do not commit or push.
2. If "Has any commits" above is `NO`, this is a brand-new repository with an unborn `HEAD` (no commits yet). This is expected and not an error:
    - Commands like `git commit` work normally and create the initial commit.
    - Do not attempt anything HEAD-relative (e.g. no `git diff` against `HEAD`, no `git log`) before the first commit exists — `git diff --cached` and `git diff --cached --stat` still work fine against an empty tree for staged changes.
    - Treat Upstream as `NONE` (it always will be) and proceed as described below.
3. Determine the push command once:
    - If an upstream exists (Upstream above is not `NONE`), push with `git push`.
    - If there is no upstream, push with `git push -u origin <current-branch>`. This is also how the very first commit gets pushed and sets up tracking.

### BATCH mode (default)
1. Stage everything: `git add -A`
2. Determine the commit message:
    - If a provided message exists, use it verbatim.
    - Otherwise, inspect the staged changes with `git diff --cached --stat` and `git diff --cached`, then write ONE concise message in conventional-commit style (`feat:`, `fix:`, `chore:`, `refactor:`, `docs:`, `test:`), subject line ≤ 72 chars. Summarize at a higher level if the changes are varied.
3. Commit: `git commit -m "<message>"`
4. Push using the push command determined above.
5. Report the commit hash, the message used, whether it was provided or generated, and the branch pushed to.

### SEPARATE mode
For EACH pending path in the porcelain output, perform an independent add → commit → push cycle, in order:
1. Parse each path from the porcelain output. Handle every change type:
    - Modified, added, untracked (`??`), and deleted files: stage with `git add -A -- "<path>"`.
    - Renames (`R  old -> new`): stage both sides with `git add -A -- "<old>" "<new>"` and treat them as a single unit.
    - Always quote paths to handle spaces.
2. After staging a single file, confirm only that path is staged with `git diff --cached --stat`.
3. Determine the commit message for THAT file:
    - If a provided message exists, use it verbatim for this file's commit. (The same provided message is reused for every file in this mode.)
    - Otherwise, inspect that file's change with `git diff --cached -- "<path>"` and write a message specific to that file, conventional-commit style, subject ≤ 72 chars (e.g. `fix(QueueVault): guard withdrawal reentrancy`).
4. Commit only that file: `git commit -m "<message>"` (only the single path is staged, so this commits just that file).
5. Push immediately using the push command determined above. After the first push sets the upstream, subsequent pushes can use plain `git push`.
6. Move to the next file and repeat until all pending paths are processed.
7. Report a summary table: each file, its commit hash, and its message. Note whether messages were provided or generated.

### Safety notes
- Never use `git push --force` or `--force-with-lease` in either mode.
- If any `git push` fails (e.g. rejected non-fast-forward, no network, auth error), STOP immediately, report the exact error, and do not continue to the next file in separate mode. Do not attempt to resolve conflicts automatically.
- Do not modify file contents, amend prior commits, or run any git command not listed in allowed-tools.