---
name: process-record
description: Produce a structured record of an agent's command-execution process — the commands to be run, their purpose, results summaries, and full terminal output saved to log files. Triggers on explicit user requests like "过程记录", "做个过程记录", "记录这次执行过程", "process record", "document this run", "trace this workflow", or similar. Always use this skill when the user invokes it by name or asks for a recorded/documented run, even if the underlying task seems trivial. Plans are written before execution, the agent pauses for confirmation on risky/expensive steps, and results are filled in as commands run.
---

# Process Record

Produce a structured record of a multi-step command execution. The record captures the **plan** (which commands and why) up front, pauses for the user's go-ahead on anything risky, and then fills in the **results** (per-command summary plus saved log files) as work proceeds.

Use this skill only when the user explicitly invokes it. Don't volunteer it for ordinary Bash calls — only when asked.

## What you produce

Inside the current project, create `process-records/<task-slug>/<YYYY-MM-DD-HHMM>/`:

```
process-records/
└── cleanup-node-modules/
    └── 2026-05-08-1430/
        ├── record.md       # the structured record
        └── logs/
            ├── 01-du.log
            ├── 02-rm.log
            └── 03-npm-install.log
```

`<task-slug>` is a 2-5 word kebab-case description of the task. Grouping by slug lets multiple runs of the same task accumulate side-by-side under the same parent — useful when the user retries or revisits a workflow. Use the user's project working directory as the base, not `/tmp` or anywhere else — the record belongs with the project.

If a slug directory already exists from a prior run, just add a new timestamp subdirectory inside it; don't reuse or overwrite the previous run's files.

## Workflow

The four phases below run in order. Don't skip phase 2 (risk check) and don't fill in execution results before commands have actually run.

### Phase 1 — Write the plan (before running anything)

1. Create the record directory and a `logs/` subdirectory inside it.
2. Write `record.md` with the template below, leaving **Execution** and **Final summary** empty:

```markdown
# <Task title>

- **Started**: <YYYY-MM-DD HH:MM>
- **Goal**: <one-line description of what the user wants accomplished>

## Plan

| # | Command | Purpose |
|---|---------|---------|
| 1 | `<command>` | <why this command, what it accomplishes> |
| 2 | `<command>` | ... |

## Risk assessment

<Call out any irreversible or expensive steps and why they're necessary.
If nothing in the plan is risky, write "No risky steps — proceeding directly.">

## Execution

<filled in as each command runs>

## Final summary

<filled in at the end>
```

3. Tell the user the path to `record.md` so they know where to follow along.

The plan is the contract. Be specific — `git checkout main && git pull` is two steps, not one. If a command's exact form depends on what an earlier command returns (e.g., "rm whichever pid file step 1 finds"), say so in the Purpose column rather than guessing the literal command.

### Phase 2 — Risk check

After writing the plan, decide whether to pause for confirmation. The principle: **anything hard to undo, anything that affects state beyond this directory, or anything that burns significant time or tokens warrants a pause.** Use judgment informed by the examples below — they're not exhaustive.

**User override.** If the user has already said something like "直接跑 / just run it / 不用确认 / proceed without asking" when invoking the skill (or earlier in the conversation, scoped to this task), skip the pause entirely and go to Phase 3. They've taken responsibility for the cost. Don't relitigate. The one exception: if you discover something genuinely surprising mid-execution that wasn't visible from the plan (e.g., step 1 returns 80 GB of files step 2 was about to delete), surface it then — the override covered the plan you showed them, not new information.

**Pause and ask before continuing if the plan includes:**

- File deletion or overwriting moves: `rm`, `rm -rf`, `mv` onto an existing path, `> existing-file`
- Operations that change shared or remote state: `git push` (especially `--force`), `gh pr merge`, deploys, sending messages/emails, modifying CI
- Long-running or expensive work: estimated >30 s wall time, large downloads, package installs that fetch many MB, spawning subagents that will burn many tokens, large LLM calls
- Resource-heavy steps where parallelism is ambiguous: **two or more** commands that each saturate the GPU, CPU, RAM, or network (training runs, large builds, simulations, batch inference), **and** the user hasn't said whether to run them serially or in parallel. Running them in parallel can wedge the machine; running them serially can waste hours — either default can be wrong, so ask. Skip this check if the plan only has one such command, or the user has already stated their preference
- Destructive changes to the user's machine: editing `~/.<rc>` files, changing global git config, killing processes you didn't start, modifying system packages
- Steps where the right behavior is genuinely ambiguous and the wrong call would be hard to recover from

**Just proceed (no need to ask) when the plan is:**

- Read-only inspection: `ls`, `cat`, `find`, `grep`, `git status`, `git log`, `du -sh`, `wc -l`
- Creating new files anywhere (a path that doesn't exist yet) — overwriting an existing file is a different matter and stays in the pause list above
- Quick local commands with no side effects: version checks, formatters/linters in check mode, dry-run flags
- Anything fully reversible by `git checkout`, `rm -rf <new-dir>`, or equivalent local rollback

When in doubt, pause. A 10-second confirmation is trivially cheap compared to an unwanted destructive action.

When pausing, be specific about the actual risk, what you'd do next, and one or two alternatives. For example:

> Step 3 is `rm -rf node_modules` (~600 MB) before reinstall. It's reversible via `npm install`, but redownloading takes ~2 min. Continue, or try `npm ci` instead first?

Don't ask vague yes/no questions ("This might be destructive — proceed?"); the user can't make a decision from that.

### Phase 3 — Execute and record

For each command, save the full terminal output to a log file. The simplest pattern is `tee`:

```bash
<command> 2>&1 | tee process-records/<task-slug>/<timestamp>/logs/<NN>-<short>.log
```

- `<NN>` is the zero-padded step number (`01`, `02`, ...).
- `<short>` is a 1-3 word slug for the command.
- `2>&1` matters — without it, stderr won't end up in the log.

The `2>&1 | tee ...` wrapping is a logging mechanism, not part of the workflow itself. The **Plan** table and the **Execution** section's step headings should always record the *original* command (`du -sh node_modules`), never the tee-wrapped form. The user reading the record cares what work was done, not how it got logged.

Note that piping through `tee` causes the pipeline's exit code to come from `tee`, not the original command. If the exit code matters (e.g., to detect failure), use `set -o pipefail` or check `${PIPESTATUS[0]}`, or run the command and redirect: `<command> >logfile 2>&1`.

After each command finishes, append an entry to the **Execution** section of `record.md`:

```markdown
### Step 1 — `du -sh node_modules`

- **Status**: ✅ success (or ❌ failed, exit code N)
- **Log**: [logs/01-du.log](logs/01-du.log)
- **Summary**: node_modules is 612 MB across 47k files.
```

The summary should be 1-3 sentences focused on what the user actually needs to know — the answer the command produced, not a paraphrase of the raw output. For failures, lead with the error message and a hypothesis about cause if you have one.

If the plan changes mid-execution — step 2 reveals step 3 is unnecessary, or you discover a new step is needed — update the **Plan** table to reflect what actually happened, and note the change in the affected step's summary ("skipped because step 1 showed X", "added in response to Y"). The plan should always match reality after the fact.

### Phase 4 — Final summary

When all commands are done, fill in the **Final summary** section:

- One sentence on whether the goal was accomplished.
- Anything notable: surprises, deviations from the original plan, useful follow-ups for the user.
- If anything is left in a weird state (partially-applied changes, stopped halfway), say so explicitly so the user knows what they're walking into.

Then tell the user where to find the record.

## A few things to keep in mind

**The record is for humans, not for you.** The user should be able to come back weeks later and reconstruct what happened. Be specific in summaries — "build succeeded" is less useful than "build succeeded, output 4.2 MB to `dist/`, took 38 s."

**Logs are the source of truth.** If a summary and log disagree, the log wins. Don't rewrite, trim, or "clean up" log files — they're raw output from `tee`, kept as-is so the user can grep them later.

**Match document depth to task depth.** A two-step `ls` + `wc -l` workflow doesn't need a paragraph of risk analysis — "No risky steps — proceeding directly." is fine. Don't pad with ceremony.

**One record per logical task.** If the user kicks off something unrelated, start a new record directory. Cramming multiple workflows into one record makes both harder to find later.
