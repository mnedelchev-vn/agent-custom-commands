# Agent Custom Commands

A collection of custom [Claude Code](https://claude.com/claude-code) slash commands that automate common development workflows. Each command is a Markdown file with YAML frontmatter describing its arguments, allowed tools, and behavior — drop it into your `.claude/commands/` directory (or your project's equivalent) to make it available inside Claude Code.

## How these commands work

Claude Code slash commands are plain Markdown files. The frontmatter controls how the command is invoked and what it's allowed to do; the body is the instruction set Claude follows when the command runs.

```yaml
---
argument-hint: "..."          # shown to the user as a hint for expected arguments
description: "..."             # one-line summary shown in the command list
disable-model-invocation: true # prevents Claude from calling this command on its own
allowed-tools: Bash(git status:*), ...  # the exact tool/command allowlist the command can use
---
```

The body then documents context gathering (e.g. `!`command`` blocks that inject live shell output), argument parsing rules, and step-by-step task instructions.

## Commands

### `/push`

**File:** [push.md](push.md)

Adds, commits, and pushes pending changes — either as a single batched commit or as one commit per changed file — without ever force-pushing or rewriting history.

**Usage:**

```
/push
/push --commit-type=separate
/push --commit-message="fix: correct off-by-one in pagination"
/push --commit-type=separate --commit-message="chore: update deps"
```

**Arguments:**

| Flag | Values | Default | Description |
|---|---|---|---|
| `--commit-type` | `batch` \| `separate` | `batch` | `batch` stages and commits everything at once; `separate` commits and pushes each changed file individually. |
| `--commit-message` | any string | _(auto-generated)_ | Use this exact message for the commit(s). If omitted, Claude inspects the diff and writes a conventional-commit-style message. |

**Behavior at a glance:**
- Reads current branch, upstream status, and pending changes before doing anything.
- Stops immediately if the working tree is clean — nothing is committed or pushed.
- **Batch mode:** `git add -A`, one commit, one push.
- **Separate mode:** stages, commits, and pushes each file (or rename pair) individually, in sequence, reporting a per-file summary.
- Auto-detects whether an upstream branch exists and uses `git push -u origin <branch>` the first time if not.
- Never uses `--force` or `--force-with-lease`, never amends prior commits, and stops immediately if any push fails rather than attempting to resolve conflicts automatically.
- Restricted to a fixed set of read-only and additive `git` subcommands (`status`, `rev-parse`, `branch`, `add`, `commit`, `push`, `diff`, `log`) via `allowed-tools`.

## Adding a new command

1. Create a new `<name>.md` file in this repo following the frontmatter + task structure shown above.
2. Keep `allowed-tools` as narrow as possible — only grant the exact tool/command patterns the command needs.
3. Document safety notes explicitly (e.g. what the command must never do) so behavior stays predictable.
4. Add a section to this README describing its purpose, arguments, and behavior.
