---
name: update-project-docs
description: Maintain software-project documentation against code changes, including audits, synchronization, and validation. Use only when the user explicitly invokes `update-project-docs`.
---

# Documentation Updater

Guides you through maintaining project documentation against the codebase on the active branch. Works with any project regardless of language, framework, or documentation format.

## Activation boundary

Run this workflow only after the request has both of these anchors:

1. **Repository-doc scope** — the user explicitly wants an action on documentation files belonging to a software project (for example `docs/`, a repository `README`, `CHANGELOG`, or API reference).
2. **Project-change scope** — the action is tied to a concrete code diff or implemented/landed API, configuration, or behavior change, commit, pull request, or release, or the user explicitly asks for a first-time project-documentation audit or scaffold.

Both anchors must be stated in the current request; do not infer either one from earlier turns, surrounding discussion, or the mere presence of a `docs/` directory. Requests to capture or summarize a discussion, meeting, brainstorm, or other prose do not enter this workflow unless they also provide both anchors above. If either anchor is absent, use the ordinary writing/document workflow. An explicit invocation of `update-project-docs` is sufficient to pass this boundary.

> **Important**: For first-time runs or updates spanning a long period, it is strongly recommended to use the most capable model with thinking/extended thinking enabled and effort set to maximum. The first run performs a full-project audit that determines the baseline for all future incremental updates — any documentation gaps missed during this run will **not** be caught by subsequent incremental diffs, since incremental mode only reviews code changes after the recorded sync point. Investing in thoroughness up front pays off in every future run.

## Quick Start

1. **Pre-flight check**: Verify the working tree is clean and find the last sync point
2. **Discover project structure**: Find documentation directories, formats, and conventions
3. **Determine run mode**:
   - **First run** (no `.docs-sync` record): Perform a **full-project audit** — read the entire codebase, compare with existing docs, fill every gap. Do **not** rely on git diff.
   - **Incremental run** (sync record exists): Read the recorded commit hash, then resolve the effective diff base. If the commit immediately after the recorded hash updates the sync record, use that commit as the diff base; otherwise use the recorded hash itself.
4. **Map code to docs**: Identify which documentation is missing, outdated, or affected
5. **Scaffold missing doc infrastructure** (first run only, if needed): Create the doc site skeleton
6. **Review and update each doc**: Walk through updates with user confirmation
7. **Validate**: Run the project's lint/build checks
8. **Record sync point**: Save the current code-sync commit hash
9. **Commit**: Stage the documentation changes and sync record together

## Workflow: Pre-flight Check

Always run this before any other workflow step.

### Step 1: Check the working tree

```bash
git status --porcelain
```

If the output is non-empty, there are uncommitted changes (modified, staged, or untracked files). **Stop and ask the user how to proceed** before continuing. Present these options:

1. **Commit first, then continue** — The user commits or stashes their changes, then re-invokes the skill. This ensures uncommitted edits are included in the doc update.
2. **Ignore uncommitted changes** — Proceed using only committed history. Uncommitted edits will not be reflected in the documentation update.

Use the AskUserQuestion tool to present this choice. Do not silently include or exclude uncommitted changes — the user must decide explicitly.

### Step 2: Find the last sync point

The skill records the commit hash of the code state that the documentation is synced to in a tracking file so subsequent runs can do **incremental updates** instead of re-scanning the entire codebase.

Look for the sync record in this order:

1. **Dedicated tracking file** at the repository root or docs root:
   - `.docs-sync`
   - `docs/.docs-sync`
   - `.claude/docs-sync`
2. **Frontmatter field** on a docs index page (e.g., `last_synced_commit:` in `docs/index.md`)
3. **HTML comment** in a docs README (e.g., `<!-- docs-sync: <hash> -->`)

The file format records both the **end** and **start** commits of the most recent documentation sync, plus a timestamp:

```
end_commit: abc1234567890def...
start_commit: 7890fedcba0987...
synced_at: 2026-04-11T10:30:00Z
```

- **end_commit**: `HEAD` at the moment the documentation sync was committed. This is the primary sync point — the incremental run uses it as the diff base.
- **start_commit**: The diff base of the run that produced this record (i.e. the commit the docs were synced *from* in the previous round). It exists as a fallback for when `end_commit` is gone — most often after a squash-merge of a feature branch erases the `end_commit` that lived only on that branch, while `start_commit` (which sat on the long-lived base branch) survives. On the very first run there is no prior diff base, so `start_commit` may be omitted or set equal to `end_commit`.
- **synced_at**: ISO-8601 timestamp of when the sync was recorded (informational; not used for diff resolution).

For backwards compatibility, older sync records may contain only a bare commit hash (and optional timestamp) on its own line — in that case, treat the hash as `end_commit` with no `start_commit`.

**Decide the run mode based on what you find:**

#### Incremental run (sync record exists)

Resolve a usable diff base by trying candidates in order: `end_commit` first, then `start_commit`, and only if both fail prompt the user.

For each candidate hash, run **both** validation checks:

1. **Existence**: `git cat-file -e <hash>` — the commit object is in the local repo
2. **Reachability**: `git merge-base --is-ancestor <hash> HEAD` — the commit is on the ancestry of the current branch. The existence check alone passes for any commit in the object database (including ones on unrelated branches or detached from the current branch's ancestry), which would silently produce a wrong diff range, so the reachability check is required.

Resolution flow:

- **If `end_commit` passes both checks**, use it as the diff base. Then refine: find the commit immediately after `end_commit` on the current branch with `git rev-list --ancestry-path --first-parent --reverse <end_commit>..HEAD | head -n 1`.
  - If that immediate next commit updates the sync record, treat that commit as the effective diff base. This skips the previous documentation-sync commit instead of reprocessing it on every run.
  - If that immediate next commit does **not** update the sync record, use `end_commit` itself as the diff base and include that commit in the review.
  - Use the effective base for review: `git diff <effective-base>...HEAD`
- **If `end_commit` fails but `start_commit` exists in the record and passes both checks**, fall back to `start_commit` as the diff base. **Inform the user this fallback is being used** (so they understand why already-reviewed code may reappear in the diff) — typically this happens after a squash-merge erased `end_commit`, and `start_commit` is the still-reachable ancestor on the base branch.

  Then apply a **squash-merge skip** when it can be proven safe:

  - Find the commit immediately after `start_commit` on the current branch: `git rev-list --ancestry-path --first-parent --reverse <start_commit>..HEAD | head -n 1`.
  - That commit qualifies as the squash-merge of the previous round if **all** of the following hold:
    1. It updated the sync record file (`git diff-tree` on that commit shows the sync record path), and
    2. The `start_commit` field inside the *updated* sync record (read it from that commit's tree, e.g. `git show <next>:<sync-record-path>`) resolves to the same commit as our current `start_commit`.
  - When both conditions hold, treat that immediate next commit as the effective diff base (skip it). It is the squashed result of the previous doc-sync round, and the assumption is that the pre-merge code review already verified the docs were updated comprehensively for that round, so re-reviewing the squashed code changes here would add no value.
  - When either condition fails, do **not** skip. Use `start_commit` itself as the diff base and include the immediate next commit in the review — the equality check on the recorded `start_commit` is what distinguishes "this is the squash of the round that started from us" from "this is some unrelated later doc-sync commit that just happens to be next".

  Whether or not the skip applies, the diff range may still redundantly include code already reviewed in the previous round (e.g., commits that landed *after* the squash-merge but were not part of it) — accept that as the price of avoiding a full audit.
- **If both `end_commit` and `start_commit` are unusable** (missing, unreachable, or the record predates the start/end format and only had a single hash that itself failed), **stop and ask the user how to proceed** via AskUserQuestion. Present these options:
  1. **Specify a different commit hash** — the user provides a replacement commit (e.g., a known-good earlier sync point, the last release tag, or a commit they remember the docs were accurate at). Re-validate the new hash with both `cat-file -e` and `merge-base --is-ancestor`, then continue with the incremental flow using that hash as the diff base. After the run completes, the sync record is updated to the new `HEAD` as usual.
  2. **Fall back to first-run / full-project audit** — perform a complete audit, then record `HEAD` as the new sync point.

  Falling back to a full audit is significantly more expensive than incremental review, so giving the user a chance to point at a still-reachable earlier commit is often the right escape hatch. If a project's `.docs-sync` is frequently invalidated all the way through `start_commit`, surface that to the user (likely caused by force-push / rebase habits, repeated history rewrites, or shallow clones) rather than silently re-scanning the entire codebase on every run.

#### First run (no sync record)

**Do not use git diff as the entry point.** A first run means the documentation has never been audited against the current codebase — there may be missing, outdated, or stale documentation regardless of recent git history. Even if `git diff origin/main...HEAD` is empty, the docs may still need substantial work.

Instead, perform a **full-project audit** (see the "First-Run Full Audit" workflow below). After the audit is complete and documentation is updated, record the current `HEAD` as the code-sync point so subsequent runs can switch to incremental mode.

## Workflow: Discover Project Structure

Before analyzing changes, understand the project's documentation setup.

### Step 1: Find the base branch

```bash
# Detect the default branch
git remote show origin | grep 'HEAD branch'

# Or check common names
git branch -a | grep -E 'main|master|develop'
```

### Step 2: Find documentation directories

Use the Glob tool to search for common documentation locations:

- `docs/`, `documentation/`, `doc/`
- `site/`, `website/`, `content/`
- `wiki/`, `guides/`, `manual/`
- Co-located `README.md` files alongside source code
- `api-docs/`, `api-reference/`

Also check for documentation build configuration files that reveal the doc root:

- `mkdocs.yml` (MkDocs)
- `docusaurus.config.js` / `docusaurus.config.ts` (Docusaurus)
- `conf.py` (Sphinx)
- `book.toml` (mdBook)
- `antora.yml` (Antora)
- `.vitepress/` (VitePress)
- `_config.yml` with docs theme (Jekyll)

### Step 3: Identify documentation format

| Format      | Extensions           | Common In                       |
| ----------- | -------------------- | ------------------------------- |
| Markdown    | `.md`                | Most projects                   |
| MDX         | `.mdx`               | React-based doc sites           |
| reStructuredText | `.rst`          | Python projects (Sphinx)        |
| AsciiDoc    | `.adoc`, `.asciidoc` | Java/enterprise projects        |
| HTML        | `.html`              | Legacy or generated docs        |

### Step 4: Discover sidebar / navigation structure

Many documentation systems use a sidebar or navigation config that defines the canonical hierarchy and ordering of pages. If one exists, it is the **single source of truth** for how documentation files should be organized on disk. Check for:

| Config File                  | System          |
| ---------------------------- | --------------- |
| `sidebars.js` / `sidebars.ts` | Docusaurus      |
| `mkdocs.yml` → `nav:` section | MkDocs          |
| `SUMMARY.md`                 | mdBook          |
| `_sidebar.md`                | Docsify         |
| `_toc.yml`                   | Jupyter Book    |
| `.vitepress/config.*` → `sidebar` | VitePress  |
| `antora.yml` → `nav:`       | Antora          |
| `_data/navigation.yml`      | Jekyll          |
| `book.json` / `book.js`     | GitBook         |

When a sidebar config is found:

1. **Parse its hierarchy** — understand the tree structure (sections, groups, ordering)
2. **Map it to the file system** — note how sidebar entries map to directories and file paths
3. **Use it as the default organization rule** — when creating or moving documentation files, place them according to the sidebar hierarchy unless the user explicitly provides different instructions
4. **Keep sidebar and directories in sync** — if the sidebar groups topics into sections like `Getting Started > Installation`, the corresponding file should live in a directory path that reflects that grouping (e.g., `docs/getting-started/installation.md`)

If no sidebar config is found, fall back to the existing directory structure as the organizational guide.

### Step 5: Discover validation commands

Check for lint/build commands in:

- `package.json` (scripts section) — look for `lint`, `docs:build`, `docs:lint`
- `Makefile` / `justfile` — look for `docs`, `lint-docs`, `build-docs` targets
- `tox.ini` / `noxfile.py` — look for docs environments
- CI config (`.github/workflows/`, `.gitlab-ci.yml`) — look for doc validation steps

## Workflow: First-Run Full Audit

Use this workflow when **no sync record exists**. The goal is to bring documentation up to parity with the current state of the codebase, not to review recent changes.

### Step 1: Enumerate the codebase

Build a picture of what the project actually contains. Use Glob and Read (or delegate to the Explore subagent for larger codebases) to identify:

- **Public APIs**: exported functions, classes, modules, CLI commands, HTTP endpoints
- **Configuration surface**: config files, environment variables, CLI flags
- **User-facing features**: anything a consumer of this project would need to know about
- **Entry points**: main modules, `__init__.py`, `index.*`, `main.*`, `cli.*`
- **Project metadata**: `pyproject.toml`, `package.json`, `Cargo.toml`, `go.mod`, etc. for project purpose and dependencies

Ignore internal-only utilities, test fixtures, and build artifacts.

### Step 2: Enumerate existing documentation

List every existing documentation file and note what each covers:

- All files under the doc root (`docs/`, etc.)
- `README.md` at the project root
- Co-located README files in source directories
- Any sidebar/navigation config (it tells you what docs the project *intends* to have)

### Step 3: Build a gap analysis

Compare the codebase to the docs and categorize. Documentation maintenance is not just about adding and updating — **deleting obsolete docs and merging redundant ones are equally important** to keep the documentation concise, accurate, and maintainable.

| Status        | Meaning                                               | Action                          |
| ------------- | ----------------------------------------------------- | ------------------------------- |
| **Missing**   | Feature/API exists in code but has no documentation    | Create a new doc                |
| **Outdated**  | Doc exists but references removed/changed code        | Update the doc                  |
| **Obsolete**  | Doc describes a feature/workflow that no longer exists or is no longer relevant to the project | **Delete** — remove the file, remove its sidebar entry, remove links pointing to it |
| **Redundant** | Multiple docs cover the same topic with overlapping content, or a single topic is fragmented across files unnecessarily | **Merge** — consolidate into one doc, delete the duplicates, update all links |
| **Orphaned**  | Doc exists on disk but is not referenced by sidebar or any other doc | Evaluate: add to sidebar, merge into another doc, or delete |
| **Accurate**  | Doc matches current code                              | Leave alone                     |

Also check whether the doc site itself is complete:
- Is there an index/home page?
- Is there a sidebar/navigation config?
- Does the sidebar reference files that don't exist?
- Are there essential infrastructure files missing (e.g., `index.html` for Docsify)?

### Step 4: Audit directory structure

If the project's documentation is already organized into subdirectories, **do not assume the existing structure is correct**. Evaluate it against the current state of the project:

1. **Compare directory groupings to project topology** — do the doc categories still make sense? For example, if the project has grown a new major subsystem (e.g., a CLI was added, or an API layer was split into public/internal), the doc directories should reflect that.
2. **Check for misplaced files** — are there docs in a directory that no longer matches their topic? (e.g., a "deployment" guide sitting under `api/`)
3. **Check for overly flat or overly deep nesting** — a single directory with 20+ files may need splitting; a 4-level deep hierarchy with 1 file per level may need flattening.
4. **Cross-check against sidebar** — if a sidebar/navigation config exists, the file system directories must mirror the sidebar hierarchy. Specifically:
   - Every section/group in the sidebar should correspond to a directory on disk
   - Every file referenced in the sidebar should exist at the path the sidebar points to
   - Files that exist on disk but are not in the sidebar should be either added to the sidebar or flagged as orphaned
5. **Propose restructuring if needed** — if the directory layout is inconsistent with the sidebar or with the project's actual structure, include specific move/rename operations in the gap analysis. Do not silently leave a mismatched directory structure in place.

### Step 5: Present the gap analysis to the user

Before making changes, show the user the categorized list. Include any directory restructuring proposals:

```
First-run audit results:

Missing docs (5):
  - CLI commands: `mytool run`, `mytool init`
  - Public function: `parse_config()`
  - Configuration: `MYTOOL_CACHE_DIR` env var
  - ...

Outdated docs (2):
  - docs/api/client.md — references removed `Client.legacy_connect()`
  - docs/config.md — missing new `timeout` option

Obsolete docs to delete (2):
  - docs/guides/legacy-auth.md — legacy auth system was removed in v3.0
  - docs/api/xml-export.md — XML export feature no longer exists

Redundant docs to merge (1):
  - docs/guides/setup.md + docs/getting-started/installation.md — both cover
    installation steps with overlapping content → merge into docs/getting-started/installation.md

Orphaned docs (1):
  - docs/notes/roadmap.md — not in sidebar, not linked from any doc → delete or add to sidebar

Missing infrastructure (3):
  - docs/index.html (Docsify entry point)
  - docs/_coverpage.md
  - Sidebar references docs/guides/advanced.md which does not exist

Directory restructuring (2):
  - Move docs/deployment.md → docs/guides/deployment.md (matches sidebar section "Guides")
  - Create docs/cli/ directory (sidebar has "CLI Reference" section but files are in docs root)
```

Confirm the plan before making any edits.

### Step 6: Scaffold missing infrastructure

If the doc site is incomplete, fill in the missing infrastructure files. See "Workflow: Scaffold Documentation Site Infrastructure" below.

### Step 7: Restructure directories

If directory restructuring was proposed and the user confirmed:

1. Move/rename files to their new locations
2. Update all internal cross-references and links in affected docs
3. Update the sidebar config to reflect the new paths (or verify it already matches, since the restructuring was driven by the sidebar)
4. Verify no broken links remain

### Step 8: Apply updates

Walk through each action with user confirmation. The full set of operations includes:

1. **Delete obsolete docs** — remove files that describe features/workflows no longer present in the project. For each deletion:
   - Remove the file from disk
   - Remove its entry from the sidebar/navigation config
   - Search for and remove or update any links pointing to the deleted doc from other docs
2. **Merge redundant docs** — when multiple docs cover the same topic, consolidate them into a single authoritative doc. For each merge:
   - Choose the best target file (the more complete or better-located one)
   - Incorporate unique content from the other file(s) into the target
   - Delete the source file(s)
   - Update the sidebar to remove duplicates and keep only the merged doc
   - Redirect or update all links that pointed to the deleted source files
3. **Update outdated docs** — following the "Update Existing Documentation" workflow
4. **Create missing docs** — following the "Scaffold New Feature Documentation" workflow

Execute deletions and merges **before** creating new docs, to avoid writing content that would overlap with existing docs about to be merged.

## Workflow: Analyze Code Changes (Incremental Run)

Use this workflow when **a sync record exists**. It reviews only the changes since the last documentation sync.

### Step 1: Get the diff

Resolve the effective diff base from the sync record. Try `end_commit` first, fall back to `start_commit` if `end_commit` is missing or unreachable, and only prompt the user if both fail (see "Incremental run" above for the full fallback contract):

```bash
END_COMMIT=<end_commit-from-sync-record>
START_COMMIT=<start_commit-from-sync-record-or-empty>
SYNC_RECORD_PATH=<path-to-sync-record-file-or-doc-page>

# Helper: a commit is usable iff it both exists locally and is an ancestor of HEAD.
is_usable() {
  git cat-file -e "$1" 2>/dev/null && git merge-base --is-ancestor "$1" HEAD
}

if is_usable "$END_COMMIT"; then
  RECORDED="$END_COMMIT"
  # If the immediate next commit updated the sync record, it is the previous
  # documentation-sync commit and should be skipped on the next run.
  NEXT=$(git rev-list --ancestry-path --first-parent --reverse "$RECORDED"..HEAD | head -n 1)
  BASE="$RECORDED"
  if [ -n "$NEXT" ] && git diff-tree --no-commit-id --name-only -r "$NEXT" -- "$SYNC_RECORD_PATH" | grep -q .
  then
    BASE="$NEXT"
  fi
elif [ -n "$START_COMMIT" ] && is_usable "$START_COMMIT"; then
  # end_commit is gone (typical after a squash-merge erased the feature
  # branch). Fall back to start_commit.
  echo "WARN: end_commit unreachable; falling back to start_commit. Diff may redundantly include code already reviewed in the previous round." >&2
  BASE="$START_COMMIT"

  # Squash-merge skip: if the immediate next commit after start_commit is
  # the squash-merge of the previous doc-sync round, we can skip it. The
  # proof is that the next commit (a) updated the sync record and (b) the
  # start_commit recorded inside that updated record equals our current
  # start_commit — i.e., it was produced by a run that started from the
  # same base we are now falling back to. Pre-merge code review is assumed
  # to have verified docs completeness for that squashed round.
  NEXT=$(git rev-list --ancestry-path --first-parent --reverse "$START_COMMIT"..HEAD | head -n 1)
  if [ -n "$NEXT" ] && git diff-tree --no-commit-id --name-only -r "$NEXT" -- "$SYNC_RECORD_PATH" | grep -q .; then
    NEXT_RECORDED_START=$(git show "$NEXT:$SYNC_RECORD_PATH" 2>/dev/null \
      | sed -n 's/^start_commit:[[:space:]]*//p' | head -n 1)
    if [ -n "$NEXT_RECORDED_START" ]; then
      START_RESOLVED=$(git rev-parse --verify "$START_COMMIT^{commit}" 2>/dev/null)
      NEXT_START_RESOLVED=$(git rev-parse --verify "$NEXT_RECORDED_START^{commit}" 2>/dev/null)
      if [ -n "$START_RESOLVED" ] && [ "$START_RESOLVED" = "$NEXT_START_RESOLVED" ]; then
        BASE="$NEXT"
        echo "INFO: skipping $NEXT (squash-merge of previous doc-sync round, same start_commit)." >&2
      fi
    fi
  fi
else
  # Both candidates failed — caller must invoke the AskUserQuestion fallback
  # described in "Incremental run (sync record exists)".
  echo "ERROR: neither end_commit nor start_commit is reachable; ask the user to specify a commit or fall back to a full audit." >&2
  exit 1
fi

# See all changed files since the chosen base
git diff $BASE...HEAD --stat

# See detailed changes in source directories
git diff $BASE...HEAD -- src/ lib/ packages/

# See the commit log for context
git log --oneline $BASE..HEAD
```

Only changes since the effective diff base are reviewed, instead of re-scanning the full codebase every time. In the normal case (`end_commit` reachable), this skips the immediately preceding documentation-sync commit without skipping unrelated code commits. In the fallback case (`start_commit`), the same skip can apply if and only if the immediate next commit is provably the squash-merge of the previous round (it updated the sync record AND the new record's `start_commit` equals our `start_commit`); otherwise the diff includes the previous round's code changes too — accept the redundancy as the price of avoiding a full audit.

### Step 2: Identify documentation-relevant changes

Look for changes that affect public-facing behavior:

| Change Type                  | Likely Doc Impact                     |
| ---------------------------- | ------------------------------------- |
| New exported function/class  | New API reference page or section     |
| Changed function signature   | Update parameter docs and examples    |
| New configuration option     | Update configuration reference        |
| Changed default behavior     | Update descriptions and examples      |
| Deprecated feature           | Add deprecation notice and migration  |
| New CLI command/flag         | Update CLI reference                  |
| Bug fix with workaround docs | Remove or update workaround guidance  |
| **Removed feature/API**      | **Delete** the corresponding doc      |
| **Merged/consolidated modules** | **Merge** the corresponding docs into one |

Internal-only changes (private utilities, refactors without behavior change) typically don't need doc updates.

### Step 3: Map changes to documentation files

See `references/CODE-TO-DOCS-MAPPING.md` for detailed discovery strategies. Key techniques:

1. **Search by symbol name**: Grep for changed function/class/config names in documentation files
2. **Search by file path**: Grep for references to the changed source file path in docs
3. **Co-located docs**: Check for README files in the same directory as changed code
4. **Directory conventions**: Map source directories to corresponding doc directories

## Workflow: Scaffold Documentation Site Infrastructure

Use this when the project needs a documentation site but the infrastructure files are missing or incomplete. Most commonly triggered during a first-run audit.

### Choosing a doc system

| Situation                                                     | What to do                                    |
| ------------------------------------------------------------- | --------------------------------------------- |
| Project already uses a specific doc system (MkDocs, Docusaurus, Sphinx, VitePress, mdBook, etc.) | Respect it — fill in missing files using that system's conventions |
| Project has partial Docsify setup (e.g., `_sidebar.md` exists but no `index.html`) | Complete the Docsify setup                    |
| Project has no doc site infrastructure at all                  | **Default to Docsify** — lightweight, no build step, plain Markdown |
| User explicitly requested a different system                  | Follow the user's instructions                |

### Docsify scaffolding (default)

Docsify is a zero-build documentation site generator — it serves Markdown files directly via a single `index.html`. When scaffolding a new doc site or completing a partial Docsify setup, create these files under `docs/`:

**`docs/index.html`** — the Docsify entry point:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Project Name</title>
  <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1">
  <meta name="viewport" content="width=device-width,initial-scale=1.0,minimum-scale=1.0">
  <meta name="description" content="Project description">
  <link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/vue.css">
</head>
<body>
  <div id="app">Loading...</div>
  <script>
    window.$docsify = {
      name: 'Project Name',
      repo: '',
      loadSidebar: true,
      loadNavbar: false,
      coverpage: true,
      auto2top: true,
      search: 'auto',
      subMaxLevel: 3,
    }
  </script>
  <script src="//cdn.jsdelivr.net/npm/docsify@4"></script>
  <script src="//cdn.jsdelivr.net/npm/docsify/lib/plugins/search.min.js"></script>
  <script src="//cdn.jsdelivr.net/npm/prismjs@1/components/prism-bash.min.js"></script>
</body>
</html>
```

Replace `Project Name` and the description based on the project's metadata (`pyproject.toml`, `package.json`, etc.).

**`docs/_sidebar.md`** — navigation sidebar:

```markdown
- [Home](/)
- Getting Started
  - [Installation](/getting-started/installation.md)
  - [Quick Start](/getting-started/quick-start.md)
- Guides
  - [Basic Usage](/guides/basic-usage.md)
- API Reference
  - [Overview](/api/index.md)
```

Populate the sidebar based on the actual topics the project needs documented. The file-system structure under `docs/` should mirror the sidebar hierarchy.

> **Critical**: Every link in `_sidebar.md` **must** use an absolute path starting with `/` (resolved from the docs root), e.g. `/guides/basic-usage.md` — never the relative form `guides/basic-usage.md`. Docsify resolves relative sidebar links against the **current page's URL**, not the docs root, so when a user is already on `/task-configuration/foo.md` and clicks a relative link `task-configuration/bar.md`, the browser requests `/task-configuration/task-configuration/bar.md` and 404s. Absolute paths always resolve from the docs root regardless of which page the user is currently viewing. This rule applies to every sidebar entry, including nested groups and the home link.

**`docs/_coverpage.md`** — landing page:

```markdown
![logo](_media/logo.png ':size=200')

# Project Name <small>v1.0</small>

> Brief tagline describing what the project does.

- Key feature 1
- Key feature 2
- Key feature 3

[GitHub](https://github.com/owner/repo)
[Get Started](#project-name)
```

Fill in real content from the project metadata — remove the logo line if there's no existing logo.

**`docs/README.md`** — home page content (Docsify uses this as the default landing page):

```markdown
# Project Name

Introduction to the project.

## Features

- ...

## Installation

...

## Quick Example

...
```

**`docs/.nojekyll`** — empty file to disable Jekyll processing on GitHub Pages:

```
```

### Confirm with the user before creating infrastructure files

Before creating these files, tell the user what will be created and confirm. They may prefer a different doc system, or want specific branding/metadata in the landing page.

## Workflow: Update Existing Documentation

### Step 1: Read the current documentation

Before making changes, read the existing doc to understand:

- Current structure and sections
- Frontmatter fields in use
- Formatting conventions and component usage
- Whether the doc is auto-generated (if so, update the source, not the doc)

### Step 2: Identify what needs updating

Common updates include:

- **New parameters/options**: Add to reference tables and create sections explaining usage
- **Changed behavior**: Update descriptions and examples
- **Deprecated features**: Add deprecation notices with migration guidance
- **New examples**: Add code blocks following the project's existing conventions
- **Version notes**: Add version badges or callouts for new/changed features

### Step 3: Apply updates with confirmation

For each change:

1. Show the user what you plan to change
2. Wait for confirmation before editing
3. Apply the edit
4. Move to the next change

### Step 4: Follow existing conventions

See `references/DOC-CONVENTIONS.md` for how to discover and follow the project's documentation conventions. Always match:

- Heading style and hierarchy
- Code block formatting (language tags, filename annotations)
- Callout/admonition style
- Frontmatter schema
- Cross-reference format

### Step 5: Validate changes

Run any documentation validation commands discovered in the project setup step:

```bash
# Examples — use whichever applies to the project
npm run lint          # or yarn/pnpm equivalent
make docs-lint
sphinx-build -W ...   # Warnings as errors
mkdocs build --strict
```

## Workflow: Scaffold New Feature Documentation

Use this when adding documentation for entirely new features.

### Step 1: Determine the doc type and location

**If a sidebar/navigation config was discovered**: Use the sidebar hierarchy as the primary guide for placement. Find the section in the sidebar where the new doc logically belongs, and place the file in the directory path that mirrors that sidebar position. Then update the sidebar config to include the new entry.

**If no sidebar exists**: Examine the existing documentation directory structure to find the right location:

| Doc Type            | Where to Look                            |
| ------------------- | ---------------------------------------- |
| API reference       | `api/`, `api-reference/`, `reference/`   |
| Guide / How-to      | `guides/`, `tutorials/`, `how-to/`       |
| Configuration       | `configuration/`, `config/`, `reference/`|
| CLI reference       | `cli/`, `commands/`                      |
| Conceptual / Explanation | `concepts/`, `architecture/`, `explanation/` |

In either case, match the project's existing structure rather than inventing new locations.

### Step 2: Create the file with proper naming

Follow the naming conventions used by existing docs:

- Check for kebab-case (`my-feature.md`) vs snake_case (`my_feature.md`)
- Check for numeric prefixes (`01-my-feature.md`) for ordering
- Check for index files (`index.md`, `_index.md`, `README.md`)

### Step 3: Use the project's existing patterns as a template

Instead of applying a fixed template, read 2-3 similar existing docs and replicate their structure:

1. Copy the frontmatter schema from an existing doc of the same type
2. Follow the same heading hierarchy
3. Match the code example style (language tags, annotations)
4. Include the same structural elements (prerequisites, examples, related links, etc.)

**Minimal fallback template** (if no existing docs to reference):

```markdown
---
title: Feature Name
description: Brief description of what this feature does.
---

# Feature Name

Brief introduction explaining what this feature does and why it's useful.

## Usage

Basic usage example with code.

## API Reference

Detailed reference for parameters, options, and return values.

## Examples

Additional examples for common use cases.

## Related

- Links to related documentation
```

### Step 4: Update navigation/sidebar config

If the project has a sidebar/navigation config (discovered in the project structure step), you **must** update it when adding a new page:

1. **Determine the insertion point** — find the sidebar section that matches the new doc's topic
2. **Add the entry** — use the same format as existing entries (path, label, ordering)
3. **Verify directory matches sidebar** — the file's directory path on disk should mirror its position in the sidebar hierarchy
4. **Preserve ordering** — respect numeric prefixes or explicit ordering if the project uses them

Common sidebar config files:

- `_sidebar.md`, `SUMMARY.md` (Docsify, mdBook)
- `sidebars.js` (Docusaurus)
- `mkdocs.yml` nav section (MkDocs)
- `_toc.yml` (Jupyter Book)
- `antora.yml` nav (Antora)

If no sidebar config exists, skip this step.

## Workflow: Record Sync Point

After all documentation updates are applied and validated, record both the start and end of this sync round so the next run can resolve its incremental diff base correctly — and so a fallback target survives if `end_commit` is later squashed away.

### Step 1: Determine the start and end hashes

- **end_commit**: the new `HEAD` after the documentation changes are committed (`git rev-parse HEAD`).
- **start_commit**: the diff base actually used during this run — i.e. the value of `BASE` resolved in "Analyze Code Changes → Step 1: Get the diff". Concretely:
  - If this was an incremental run with `end_commit` reachable, `start_commit` is the effective base after the "skip immediate next commit" refinement.
  - If this was an incremental run that fell back to the previous record's `start_commit`, the new `start_commit` is that fallback value (carry it forward).
  - If this was a first-run / full-project audit, set `start_commit` equal to `end_commit` (or omit it). On a clean first run there is no earlier point that the docs are partially synced from.

### Step 2: Write the sync record

Update (or create) the sync tracking file. Prefer the location that already exists; otherwise ask the user where to create it. Default: `.docs-sync` at the repository root.

```
end_commit: <new-HEAD-hash>
start_commit: <diff-base-used-this-run>
synced_at: <ISO-8601 timestamp>
```

Example `.docs-sync` content:

```
end_commit: abc1234567890abcdef1234567890abcdef123456
start_commit: 7890fedcba0987654321fedcba0987654321fedc
synced_at: 2026-04-11T14:22:00Z
```

Notes:
- Commit the sync file alongside the documentation changes — the next run reads it from the committed history
- If the user chose to ignore uncommitted changes in the pre-flight step, still record `HEAD` as `end_commit` (the recorded hashes represent what the docs are synced *to* and *from*, not what was reviewed)
- The next run may use the commit immediately after `end_commit` as the effective diff base, but only if that immediate next commit updated the sync record
- `start_commit` is the squash-merge safety net: when a feature branch carrying `end_commit` is squash-merged into the base branch, the original `end_commit` disappears, but `start_commit` (which sat on the base branch) is still reachable and serves as the fallback diff base
- Add `.docs-sync` to `.gitignore`? **No** — the file must be committed so other contributors and future runs can read it
- If the project already has a sync record in a different location (e.g., frontmatter on a docs index page), update that instead of creating a new file. When migrating an old single-hash record to the new format, set both `end_commit` and `start_commit` to the existing hash (the old record contained no separate start) and add the timestamp.

### Step 3: Stage the sync file with the doc changes

When committing documentation updates, include the sync file in the same commit so the sync state and the docs it represents stay in lockstep.

## Validation Checklist

Before committing documentation changes:

- [ ] Working tree was clean (or user explicitly chose to ignore uncommitted changes)
- [ ] Effective diff base was resolved from the recorded sync point (or a full audit was used on the first run)
- [ ] Content accurately reflects the code changes
- [ ] Frontmatter matches the project's schema
- [ ] Code examples are correct and runnable
- [ ] Internal links point to valid paths
- [ ] Formatting matches existing documentation conventions
- [ ] Navigation/sidebar updated if a new page was added
- [ ] **Sidebar links use absolute paths** — every link in `_sidebar.md` (or equivalent) starts with `/` and resolves from the docs root; no relative paths that would cause duplicate-prefix 404s when navigating from a nested page
- [ ] **Directory structure matches sidebar hierarchy** — every sidebar section maps to a directory, every sidebar entry points to a file that exists at the expected path, no file is in a directory that contradicts its sidebar position
- [ ] Project's doc lint/build commands pass (if available)
- [ ] Sync record updated with the current code-sync `HEAD` hash
- [ ] Changes reviewed with the user before committing

## References

- `references/DOC-CONVENTIONS.md` - How to discover and follow project documentation conventions
- `references/CODE-TO-DOCS-MAPPING.md` - Strategies for mapping source code changes to documentation files
