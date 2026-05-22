---
name: git-upstream-rebase
description: "Rebase a local branch onto upstream/main and handle conflicts safely. Use when user says \"upstream 更新了\", \"git pull --rebase upstream main\", \"rebase upstream/main\", \"同步上游 main\", \"处理 rebase 冲突\", or wants to update a fork from upstream with automatic conflict resolution."
argument-hint: "[branch] [--upstream upstream/main] [--push]"
allowed-tools: Bash(*), Read, Edit, Grep
---

# Git Upstream Rebase

Rebase the current work onto an upstream branch, typically `upstream/main`, and resolve conflicts when it is safe to do so.

This skill is for fork workflows where:

- `origin` points to the user's fork.
- `upstream` points to the original repository.
- The local branch should replay its commits on top of upstream's latest `main`.

## When to Use

Use this skill for requests like:

- "上游 main 更新了，帮我 rebase 一下"
- "基于 upstream/main 更新我的 main"
- "git pull --rebase upstream main 可以吗？帮我处理冲突"
- "把我的 fork 同步到 upstream，并自动解决冲突"

Do not use this skill for merge-based synchronization unless the user explicitly asks for merge commits.

## Inputs

Default values:

- Local branch: current branch.
- Upstream target: `upstream/main`.
- Push target: none by default. Only discuss or run push when the user explicitly says `push`, `推送`, or passes `--push`.

Optional user inputs:

- A different upstream target, such as `upstream/master` or `upstream/dev`.
- A specific local branch to rebase.
- Explicit permission to push after a successful rebase, expressed as `push`, `推送`, or `--push`.

## Procedure

### 1. Preflight Checks

Run these checks before changing history:

```bash
git status --short
git branch --show-current
git remote -v
git rev-parse --verify upstream/main
```

If `upstream/main` is missing or stale, run:

```bash
git fetch upstream
```

Check whether the working tree is clean:

```bash
git diff --stat
git diff --cached --stat
```

Decision rules:

| State | Action |
|---|---|
| Clean working tree | Continue |
| Uncommitted user changes exist | Ask whether to commit or stash before rebase |
| Existing rebase in progress | Inspect status; continue, abort, or ask user based on state |
| `upstream` remote missing | Stop and ask to configure upstream first |
| Branch is not the intended branch | Ask before switching branches |

Never silently discard user changes.

### 2. Start the Rebase

Preferred explicit commands:

```bash
git fetch upstream
git rebase upstream/main
```

Equivalent one-shot command if the user only wants a simple update:

```bash
git pull --rebase upstream main
```

Prefer `fetch` + `rebase` when you expect conflicts because it gives a clearer recovery path.

### 3. Detect Conflict Type

If rebase stops, inspect:

```bash
git status --short
git diff --name-only --diff-filter=U
git diff --check
```

Classify conflicts:

| Conflict Type | Auto-resolution Policy |
|---|---|
| Add/add in docs, examples, catalog tables | Attempt manual merge preserving both sides |
| Same-line Markdown prose conflict | Attempt semantic merge only if both edits are clearly compatible |
| Generated files, lockfiles, snapshots | Prefer regenerating from source if the repo has a known command; otherwise ask |
| Code logic conflict | Resolve only if behavior is obvious and tests can verify it; otherwise ask |
| Delete/modify conflict | Ask unless the deleted file is clearly generated or obsolete |
| Binary conflict | Ask; do not guess |

### 4. Automatic Conflict Resolution Loop

For each conflicted file:

1. Open the conflict hunks.
2. Remove conflict markers: `<<<<<<<`, `=======`, `>>>>>>>`.
3. Preserve upstream changes and local changes when compatible.
4. Keep existing file style, ordering, and formatting.
5. Run a targeted validation command when available.
6. Stage only the resolved files.

Useful commands:

```bash
git diff -- <file>
git add <file>
git rebase --continue
```

Repeat until the rebase completes or a conflict is ambiguous.

If a command opens an editor for a commit message, keep the existing message unless the user asked to edit it.

### 5. Validate After Rebase

Always run:

```bash
git status --short
git log --oneline --decorate --max-count=8
git diff --check
```

If the project has obvious lightweight checks, run the narrowest relevant one. Examples:

```bash
npm test -- --runInBand
pytest
make test
```

Do not run expensive full test suites without user approval if they are known to be slow.

Completion criteria:

- No rebase is in progress.
- `git status --short` has no unexpected changes.
- `HEAD` is based on the requested upstream target.
- Conflict markers are absent from resolved files.
- `git diff --check` passes.
- Targeted tests or format checks pass when available.

### 6. Push Only When Explicitly Requested

A rebase rewrites local commit hashes. Do not ask about push, suggest push as the next action, or run push unless the user explicitly includes `push`, `推送`, or `--push` in the request.

If the user did not explicitly ask for push, report that push was not run and provide the command only as an optional manual follow-up.

If pushing after a rebase, prefer:

```bash
git push --force-with-lease origin <branch>
```

Use normal push only when the remote branch has not been published before:

```bash
git push -u origin <branch>
```

Explain that `--force-with-lease` is safer than `--force` because it refuses to overwrite remote work that appeared after the last fetch.

## Recovery Commands

If the rebase becomes unsafe or the user wants to stop:

```bash
git rebase --abort
```

If a file was resolved incorrectly before staging, restore the conflicted state from the index when possible:

```bash
git checkout --conflict=merge -- <file>
```

If the rebase completed but must be undone, inspect reflog first:

```bash
git reflog --date=local --max-count=20
```

Then ask before resetting to an earlier commit.

## Reporting Format

When finished, report:

```text
Rebased <branch> onto <upstream-target>.
Conflicts resolved: <count/files or none>.
Validation: <commands and pass/fail>.
Push: not run / pushed to origin with --force-with-lease.
```

If paused for ambiguity, report:

```text
Rebase paused at <file>.
Reason: <why automatic resolution is unsafe>.
Options: keep upstream / keep local / manually combine / abort rebase.
```

## Safety Rules

- Never run `git reset --hard`, `git clean`, or destructive checkout commands without explicit user approval.
- Never ask about push or run push unless the user explicitly says `push`, `推送`, or `--push`.
- Never use `git push --force`; use `git push --force-with-lease` only after explicit push permission.
- Never resolve code conflicts by blindly choosing `--ours` or `--theirs` across the whole repo.
- Never delete conflict markers without reviewing the full hunk.
- Never collect Git credentials or tokens in chat; ask the user to type secrets directly into the terminal.
- Prefer explicit `git fetch upstream && git rebase upstream/main` over ambiguous `git pull --rebase`.
