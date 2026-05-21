---
name: git-fork-remote
description: "Configure Git remotes for a fork workflow. Use when user says \"add my fork\", \"set origin upstream\", \"fork remote\", \"git remote\", \"upstream remote\", \"像 origin/upstream 这样\", or wants origin to point to their fork and upstream to the original repository."
argument-hint: "<fork-url> [--upstream <url>] [--remote-name origin]"
allowed-tools: Bash(*), Read
---

# Git Fork Remote Setup

Configure a repository to use the standard fork layout:

- `origin` points to the user's fork and is the default push target.
- `upstream` points to the original repository and is used for syncing changes.
- Redundant temporary remotes such as `fork` are removed after migration.

## When to Use

Use this skill when the user wants a remote layout like:

```bash
origin    https://github.com/<user>/<fork>.git
upstream  https://github.com/<owner>/<original>.git
```

Typical requests:

- "帮我添加我 fork 的远程链接"
- "我希望 remote 是 origin 指向我的 fork，upstream 指向原仓库"
- "把 fork 改成 origin，把原来的 origin 改成 upstream"
- "set up git remotes for my fork"

## Inputs

Required:

- Fork URL, usually a GitHub HTTPS or SSH URL.

Optional:

- Upstream URL. If omitted, preserve the current `origin` URL as `upstream`.
- Preferred fork remote name. Default is `origin`; only use another name if the user explicitly asks.

## Procedure

### 1. Inspect Current Remotes

Run from the repository root:

```bash
git remote -v
```

Identify:

- Current `origin` URL.
- Whether `upstream` already exists.
- Whether a temporary `fork` remote already exists.
- Whether the current directory is actually inside a Git repository.

If not inside a Git repository, stop and tell the user to run the skill from the repository root or provide the correct path.

### 2. Decide Target Mapping

Use this decision table:

| Current State | Action |
|---|---|
| `origin` points to original repo, fork URL supplied | Save current `origin` URL as `upstream`, then set `origin` to fork URL |
| `fork` remote points to fork URL, `origin` points to original repo | Set `origin` to fork URL, set `upstream` to old `origin`, remove `fork` |
| `upstream` exists and matches original repo | Keep it and set `origin` to fork URL |
| `upstream` exists but conflicts with the old `origin` or supplied upstream URL | Ask before overwriting unless the user explicitly requested replacement |
| No `origin` exists | Add `origin` as fork URL; ask for upstream URL if not supplied |
| Fork URL equals current `origin` | Do not change `origin`; ensure `upstream` is configured if possible |

Never delete or overwrite a remote URL without first preserving it or confirming that it is redundant.

### 3. Apply Safe Remote Changes

Prefer explicit `git remote` commands over complex shell conditionals. Avoid multi-line `if` commands in terminals that may be sensitive to quoting or continuation prompts.

Recommended command sequence when current `origin` is the original repository:

```bash
original_url=$(git remote get-url origin)
git remote set-url origin <fork-url>
git remote add upstream "$original_url" 2>/dev/null || git remote set-url upstream "$original_url"
git remote remove fork 2>/dev/null || true
git remote -v
```

If `origin` does not exist:

```bash
git remote add origin <fork-url>
git remote add upstream <upstream-url>
git remote -v
```

If `origin` already points to the fork and only `upstream` is missing:

```bash
git remote add upstream <upstream-url>
git remote -v
```

Use `git -C <repo-path>` if the active terminal may not be in the repository root.

### 4. Verify Final Layout

Always finish with:

```bash
git remote -v
```

Completion criteria:

- `origin` fetch and push URLs both equal the user's fork URL.
- `upstream` fetch and push URLs both equal the original repository URL.
- Temporary `fork` remote is absent unless the user explicitly asked to keep it.
- No command is left waiting at a shell continuation prompt such as `then>`.

Report the final remote mapping concisely:

```text
origin    <fork-url>
upstream  <original-url>
```

Then suggest common follow-up commands:

```bash
git push origin main
git fetch upstream
```

## Safety Rules

- Do not run `git push`, `git pull`, `git fetch`, or network-mutating commands unless the user asks.
- Do not change branches, reset, rebase, or edit files as part of this skill.
- If Git asks for credentials or a token, do not collect secrets in chat; ask the user to type them directly into the terminal.
- If a command enters a continuation prompt, cancel or kill that terminal and retry with simpler one-line commands.
- Treat remote URL changes as reversible but still important: show the before/after mapping when the user needs auditability.

## Example

User request:

```text
帮我添加我 fork 的远程链接：https://github.com/OpenGHz/project.git，我希望 origin 是我的 fork，upstream 是原仓库。
```

Expected outcome:

```text
origin    https://github.com/OpenGHz/project.git
upstream  https://github.com/original-owner/project.git
```