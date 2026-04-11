---
name: update-project-docs
description: This skill should be used when the user asks to "update documentation for my changes", "check docs for this PR", "what docs need updating", "sync docs with code", "scaffold docs for this feature", "document this feature", "review docs completeness", "add docs for this change", "what documentation is affected", "docs impact", or mentions documentation updates in any project. Provides a guided workflow for updating project documentation based on code changes.
---

# Documentation Updater

Guides you through updating project documentation based on code changes on the active branch. Works with any project regardless of language, framework, or documentation format.

## Quick Start

1. **Pre-flight check**: Verify the working tree is clean and find the last sync point
2. **Discover project structure**: Find documentation directories, formats, and conventions
3. **Analyze changes**: Diff from the last sync point (or base branch) to see what files changed
4. **Map code to docs**: Identify which documentation is affected by the changes
5. **Review and update each doc**: Walk through updates with user confirmation
6. **Validate**: Run the project's lint/build checks
7. **Record sync point**: Save the current commit hash so the next run can do an incremental update
8. **Commit**: Stage documentation changes

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

The skill records the commit hash of the last documentation sync in a tracking file so subsequent runs can do **incremental updates** instead of re-scanning the entire history.

Look for the sync record in this order:

1. **Dedicated tracking file** at the repository root or docs root:
   - `.docs-sync`
   - `docs/.docs-sync`
   - `.claude/docs-sync`
2. **Frontmatter field** on a docs index page (e.g., `last_synced_commit:` in `docs/index.md`)
3. **HTML comment** in a docs README (e.g., `<!-- docs-sync: <hash> -->`)

The file format is a single line with the commit hash, optionally with a timestamp:

```
abc1234567890def...
2026-04-11T10:30:00Z
```

If a sync record exists:
- Verify the recorded commit still exists in history (`git cat-file -e <hash>`)
- Use it as the diff base: `git diff <recorded-hash>...HEAD`
- If the commit no longer exists (force-pushed, rebased away), warn the user and fall back to the base branch

If no sync record exists, this is the first run — fall back to diffing against the base branch and offer to create a sync record at the end.

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

## Workflow: Analyze Code Changes

### Step 1: Get the diff

Use the diff base determined in the pre-flight step:

- **If a sync record was found**: diff from the recorded commit hash
  ```bash
  BASE=<recorded-hash-from-sync-file>
  ```
- **If no sync record exists** (first run): diff from the base branch
  ```bash
  BASE=$(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD origin/master 2>/dev/null)
  ```

Then run the diff:

```bash
# See all changed files since the diff base
git diff $BASE...HEAD --stat

# See detailed changes in source directories
git diff $BASE...HEAD -- src/ lib/ packages/

# See the commit log for context
git log --oneline $BASE..HEAD
```

Using the recorded sync point makes runs **incremental** — only changes since the last documentation update are reviewed, instead of re-scanning the full branch history every time.

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

Internal-only changes (private utilities, refactors without behavior change) typically don't need doc updates.

### Step 3: Map changes to documentation files

See `references/CODE-TO-DOCS-MAPPING.md` for detailed discovery strategies. Key techniques:

1. **Search by symbol name**: Grep for changed function/class/config names in documentation files
2. **Search by file path**: Grep for references to the changed source file path in docs
3. **Co-located docs**: Check for README files in the same directory as changed code
4. **Directory conventions**: Map source directories to corresponding doc directories

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

After all documentation updates are applied and validated, record the current commit hash so the next run can do an incremental update.

### Step 1: Get the current commit hash

```bash
git rev-parse HEAD
```

### Step 2: Write the sync record

Update (or create) the sync tracking file. Prefer the location that already exists; otherwise ask the user where to create it. Default: `.docs-sync` at the repository root.

```
<commit-hash>
<ISO-8601 timestamp>
```

Example `.docs-sync` content:

```
abc1234567890abcdef1234567890abcdef123456
2026-04-11T14:22:00Z
```

Notes:
- Commit the sync file alongside the documentation changes — the next run reads it from the committed history
- If the user chose to ignore uncommitted changes in the pre-flight step, still record `HEAD` (the recorded hash represents what the docs are synced *to*, not what was reviewed)
- Add `.docs-sync` to `.gitignore`? **No** — the file must be committed so other contributors and future runs can read it
- If the project already has a sync record in a different location (e.g., frontmatter on a docs index page), update that instead of creating a new file

### Step 3: Stage the sync file with the doc changes

When committing documentation updates, include the sync file in the same commit so the sync state and the docs it represents stay in lockstep.

## Validation Checklist

Before committing documentation changes:

- [ ] Working tree was clean (or user explicitly chose to ignore uncommitted changes)
- [ ] Diff base was the recorded sync point (or base branch on first run)
- [ ] Content accurately reflects the code changes
- [ ] Frontmatter matches the project's schema
- [ ] Code examples are correct and runnable
- [ ] Internal links point to valid paths
- [ ] Formatting matches existing documentation conventions
- [ ] Navigation/sidebar updated if a new page was added
- [ ] Project's doc lint/build commands pass (if available)
- [ ] Sync record updated with the current `HEAD` hash
- [ ] Changes reviewed with the user before committing

## References

- `references/DOC-CONVENTIONS.md` - How to discover and follow project documentation conventions
- `references/CODE-TO-DOCS-MAPPING.md` - Strategies for mapping source code changes to documentation files
