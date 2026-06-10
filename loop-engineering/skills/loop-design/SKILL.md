---
name: loop-design
description: "Design an autonomous improvement loop from a vague goal. Use when the user wants to 'design a loop', 'set up a loop', 'loop engineer' a task, or has an improvement goal but no metric, scope, or verifier yet. Produces a ready-to-run autoloop invocation."
disable-model-invocation: true
---

Turn a vague improvement goal into a ready-to-run `/loop-engineering:autoloop` invocation. The hard part of loop engineering is not running the loop — it is choosing a metric that can't be gamed, a scope that can't wander, and a verifier that makes "done" mean something. This skill does that design work.

User's goal: $ARGUMENTS

## Process

1. **Pin the goal to a number.** Ask (or infer from the repo): what single number, measurable by a shell command, captures "better"? If the user names a quality ("readable", "fast", "small"), propose 2–3 candidate metrics with their shell commands and trade-offs, and let them pick. Dispatch `scout` to find where that number can be measured in the repo.

2. **Check metric integrity.** Before accepting a metric, ask: could the loop improve this number while making the real thing worse? (Deleting content shrinks files; skipping tests speeds CI.) If gameable, either fix the metric or add a `verify` command that blocks the exploit.

3. **Draw the scope.** Smallest set of files whose change could plausibly move the metric. Everything that defines correctness (tests, eval data, the metric script itself) goes in `frozen`.

4. **Pick the verifier.** A shell command that must exit 0 for a keep — test suite, linter, schema check. If no such command exists yet, say so plainly and offer to create one first; a loop without a checker is a loop you can't walk away from.

5. **Set the budget.** `time_budget` per iteration (start small), `target` if one exists, `max_iters` for scheduled runs, `parallel` only if hypotheses are genuinely independent.

6. **Emit the invocation.** Output a complete, copy-pasteable block:

```
/loop-engineering:autoloop
goal: ...
metric: ...
direction: ...
scope: ...
time_budget: ...
verify: ...
frozen: ...
target: ...
strategy: ...
```

7. **Offer the trigger.** Ask whether this is a one-shot run-until-done or a recurring loop; if recurring, recommend a schedule with finite `max_iters` per run and remind that the log makes runs resumable.

Keep the whole exchange short. One round of questions maximum — propose defaults and let the user correct, rather than interviewing them parameter by parameter.
