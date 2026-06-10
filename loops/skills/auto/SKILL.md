---
name: auto
description: "Autonomous improvement loop for any task with a shell-measurable metric. Use when the user wants to run an improvement loop, autonomous loop, or iteratively improve any measurable metric."
---

Try a change, measure it, keep what improves, revert what doesn't. Works for any file-based task where progress is measurable by a shell command.

Every run is five steps:
1. **Define** — `goal` + `metric` + `direction` + `target` = one verifiable stop condition.
2. **Scope** — `scope` may change, `frozen` never does.
3. **Act** — one change per iteration, committed to git.
4. **Verify** — metric says "better," `verify` says "still correct."
5. **Persist** — log and git survive between runs; the agent forgets, the repo doesn't.

## Parameters

Pass as `key: value` lines.

**Required:**
- `goal` — What to achieve
- `metric` — Shell command whose stdout contains a number
- `direction` — `up` or `down`
- `scope` — Files/dirs the loop may modify (comma-separated)
- `time_budget` — Per-iteration timeout in minutes

**Optional:**
- `verify` — Shell command that must exit 0 to keep a change
- `frozen` — Files never modified (default: everything outside scope)
- `target` — Stop when metric reaches this value
- `max_iters` — Iteration cap (default: unlimited)
- `parallel` — Max hypotheses per iteration (1–3, default 1). Treated as a hint; the loop decides per iteration whether parallel dispatch is actually warranted (see below).
- `log` — Results TSV path (default: `loop_results.tsv`)
- `strategy` — Hints to guide exploration

## Setup

1. Parse parameters. Halt if `time_budget` missing.
2. Read all files in scope and frozen.
3. Run metric, parse baseline.
4. **Resume or start:** if log exists, take its best `keep` row and `git reset --hard` to that commit. Otherwise init TSV with a baseline row.
5. If `verify` is set, it must pass on baseline — otherwise halt.
6. Summarize and begin.

## Loop (parallel=1)

1. **Ideate** — one focused, testable change based on goal, log history, and strategy.
2. **Apply** — edit scope only. `git add <scoped files>`, `git commit -m "experiment: <desc>"`.
3. **Measure** — run metric, parse result. Unparseable = crash.
4. **Evaluate** — if `verify` fails → discard. If improved (or tied + simpler) → keep.
5. **Keep:** log `keep`, amend commit to include log. **Discard:** log `discard`, `git reset --hard`.
6. Stop when target hit, `max_iters` reached, or 3 consecutive crashes.

## Parallel loop (parallel=2–3)

`parallel > 1` is a hint, not a guarantee. Before each iteration, check all four conditions:

1. **Metric is self-contained** — runs entirely within the worktree; no external server, database, or shared filesystem state required.
2. **Verify is self-contained** — same: runs in isolation without side effects on shared state.
3. **Hypotheses are orthogonal** — the approaches you'd dispatch are genuinely different strategies, not variations of the same idea.
4. **No active crash streak** — if the last iteration crashed, the environment may be broken; fix it before fanning out.

If all four hold → dispatch agents. If any fail → run sequentially this iteration and note why.

**Dispatch:** one `forge` agent per hypothesis, each in its own worktree (`isolation: worktree`). Use `scout` for reconnaissance before ideating, `critic` to verify the winner, `scribe` to update the log.

Collect results, eliminate any that fail verify, keep the best metric winner. If none improved, discard all and reset.

Teammate prompt:
```
Goal: {goal} | Hypothesis: {hypothesis} | Scope: {scope} | Frozen: {frozen}
Metric: {metric} | Direction: {direction} | Time budget: {time_budget} min
Apply the hypothesis, run the metric, run verify if set. Report: metric value, verify status, one-line description. Do NOT commit or touch files outside scope.
```

## Crashes

Recoverable (typo, bad path): fix inline, re-run, don't log. Fatal (OOM, broken toolchain): log `crash`, reset, try a different direction. Three consecutive crashes: stop and report.

## Log

TSV columns: `commit`, `metric`, `status`, `description`. Status: `baseline`, `keep`, `discard`, `crash`.

## Rules

- Metric must stay comparable across iterations — design it invariant to structural changes in scope.
- On ties prefer fewer lines and fewer moving parts.
- Redirect all command output to log files; never flood context.

## Examples

**Reduce reading difficulty:**
```
/loops:auto
goal: make the user guide readable at an 8th-grade level
metric: textstat grade_level docs/user-guide.md
direction: down
scope: docs/user-guide.md
time_budget: 2
target: 8.0
```

**Improve model accuracy:**
```
/loops:auto
goal: maximize validation accuracy on the sentiment classifier
metric: python train.py --eval-only | grep val_acc
direction: up
scope: configs/model.yaml, src/features.py
time_budget: 5
target: 0.92
frozen: data/, tests/
verify: pytest tests/ -q
```

## Scheduling

For recurring runs: schedule with finite `max_iters` per run. Each run resumes from the existing log and last kept commit.

## NEVER STOP

Once the loop begins, run until interrupted, target hit, or max_iters reached. Never ask "should I keep going?"
