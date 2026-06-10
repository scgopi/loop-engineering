---
name: optimize
description: "Autonomous improvement loop for any task with a shell-measurable metric. Use when the user asks to 'optimize', 'autoloop', 'autoresearch', 'run improvement loop', 'optimize metric', 'autonomous loop', or wants to iteratively improve any measurable metric."
disable-model-invocation: true
---

Autonomous improvement loop: try a change, measure it against a metric, keep what improves, revert what doesn't. Works for any file-based task where progress is measurable by a shell command — writing, data analysis, config tuning, prompt engineering, infrastructure, software. Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch): one mutable surface, one metric, fixed time budget, git as the keep/revert mechanism. Autonomy comes from constraints, not freedom.

Every run is five steps:

1. **Define a measurable goal** — `goal` + `metric` + `direction` + `target` form one verifiable stop condition.
2. **Scope it** — `scope` may change, `frozen` never does.
3. **Trigger it** — run-until-done in session, or on a schedule (see Scheduling).
4. **Act, then verify** — one change per iteration, measured by the metric, checked by a separate verifier if `verify` is set. The maker never gets the final word on its own work.
5. **Persist state** — the log and git history survive between runs. The agent forgets; the repo doesn't.

## Parameters ($ARGUMENTS)

Pass as `key: value` lines.

**Required:**
- `goal` — What to achieve (natural language)
- `metric` — Shell command whose stdout contains a parseable number
- `direction` — `up` or `down`
- `scope` — Files/dirs the loop may modify (comma-separated)
- `time_budget` — Per-iteration timeout in minutes (constraints enable autonomy)

**Optional:**
- `verify` — Shell command that must exit 0 for a keep (tests, linter). The maker-checker split: metric says "better," verify says "still correct." Failing verify forces discard even if the metric improved.
- `frozen` — Files never modified (default: everything outside scope)
- `target` — Stop when metric reaches this value
- `max_iters` — Iteration cap (default: unlimited)
- `parallel` — Hypotheses per iteration (1–3, default 1). >1 dispatches a team of agents in isolated worktrees; best result wins (beam search instead of greedy)
- `log` — Results TSV path (default: `loop_results.tsv`)
- `strategy` — Hints to guide exploration
- `simplicity` — Prefer simpler on metric ties (default: true)

## Setup (once, before the loop)

1. Parse parameters. Halt if `time_budget` missing.
2. Read every file in scope and frozen.
3. Run `<metric_cmd> > run.log 2>&1`; parse the baseline.
4. **Resume or baseline:** if the log already exists, take its best `keep` row as current best and `git reset --hard` to the last kept commit. Otherwise initialize the TSV: header `commit	metric	status	description` plus a baseline row.
5. If `verify` is set, it must pass on the baseline — otherwise halt and report.
6. Summarize (goal, baseline, scope, fresh vs resume) and begin immediately.

## Sequential loop (parallel=1)

1. **Ideate** — one focused, testable change based on goal, log history, current files, and strategy. If stuck: go more radical, revisit near-misses, combine approaches.
2. **Apply** — edit scope only. `git add <scoped files>` (never `-A`), `git commit -m "experiment: <desc>"`.
3. **Measure** — `<metric_cmd> > run.log 2>&1`; parse. Unparseable = crash.
4. **Verify & evaluate** — if `verify` is set and fails → discard regardless of metric. Improved = better than best so far per `direction`; tie + simpler counts as improved.
5. **Keep:** log `keep`, `git add <log> && git commit --amend --no-edit`. **Discard:** log `discard`, `git reset --hard <last kept commit>`.
6. **Stop** when `target` hit, `max_iters` reached, or 3 consecutive crashes.
7. Loop.

## Parallel loop (parallel=2–3): team of agents

Beam search: a team of agents tests N hypotheses per iteration; only the best survives.

Use the plugin's agents by role when dispatching: `scout` for read-only reconnaissance during ideation, one `forge` teammate per hypothesis, `critic` for verification of the winner, `scribe` to update the log. If the agents are unavailable, fall back to general-purpose subagents with the prompt below.

1. **Ideate N orthogonal hypotheses** — genuinely different approaches, not variations of one idea. Dispatch `scout` first if you need fresh reconnaissance.
2. **Dispatch the team** simultaneously, one `forge` teammate per hypothesis, each in its own worktree (`isolation: "worktree"`), using the prompt below.
3. **Collect** all results.
4. **Select** — eliminate teammates whose verify failed, then pick the best metric. If it beats the current best: apply the winner's changes to the main branch, commit, log `keep` for the winner and `discard` for the rest. If none improved: log discards, `git reset --hard <last kept commit>`.
5. Same stop conditions as sequential. Loop.

Teammate prompt:

```
You are one member of an optimize-loop agent team, testing a single hypothesis.
Goal: {goal} | Hypothesis: {hypothesis} | Scope: {scope} | Frozen: {frozen}
Metric: {metric} | Direction: {direction} | Time budget: {time_budget} min

1. Apply the hypothesis by editing files in scope only.
2. Run {metric} > run.log 2>&1; parse the number.
3. If a verify command is provided, run {verify} > verify.log 2>&1; note exit status.
4. Report: metric value, verify status, one-line description of the change.

Do NOT commit, touch files outside scope, or run other experiments.
If the metric command crashes, report status: crash with the error.
```

## Crashes

Easy fix (typo, bad path, missing dep): fix inline and re-run, don't log. Fundamental failure (OOM, broken toolchain): log `crash` (metric 0), `git reset --hard`, try a different direction. Timeout: kill at `time_budget` and treat as crash; force-kill at 3×. Three consecutive crashes: stop and report the blocker.

## Log

Tab-separated, columns `commit`, `metric`, `status`, `description`; status is `baseline`, `keep`, `discard`, or `crash`.

## Rules

- **Metric comparability** — the metric must stay comparable across iterations; design it invariant to structural changes in scope (Karpathy uses bits-per-byte, not loss-per-token, for exactly this reason).
- **Simplicity** — on ties prefer fewer lines and fewer moving parts; a tiny gain that adds real complexity may not be worth keeping.
- **Output hygiene** — always redirect command output to a log file and read it with targeted greps; never flood context.

## Examples

**Writing — reduce reading difficulty:**
```
/loop-engineering:optimize
goal: make the user guide readable at an 8th-grade level
metric: textstat grade_level docs/user-guide.md
direction: down
scope: docs/user-guide.md
time_budget: 2
target: 8.0
strategy: shorten sentences, replace jargon, break up long paragraphs
```

**Data science — improve model accuracy (with verifier):**
```
/loop-engineering:optimize
goal: maximize validation accuracy on the sentiment classifier
metric: python train.py --eval-only | grep val_acc
direction: up
scope: configs/model.yaml, src/features.py
time_budget: 5
target: 0.92
frozen: data/, tests/
verify: pytest tests/ -q
strategy: feature engineering first, then hyperparameter tuning
```

The same loop fits any domain — only the scope and the metric command change (Dockerfile size, CI wall-clock time, prompt eval accuracy, etc.).

## Scheduling (the trigger)

Default trigger is run-until-done in session. For a recurring trigger, schedule the same invocation (cron, scheduled task, CI) with a finite `max_iters` per run. Setup resumes from the existing log and last kept commit, so each run picks up where the previous one stopped — the log is the loop's memory, the schedule is its heartbeat.

## NEVER STOP

Once the loop begins, you are fully autonomous. Never ask "should I keep going?" — the human may be away. Run until interrupted, target hit, or max_iters reached.
