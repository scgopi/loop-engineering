# loop-engineering

A Claude Code plugin for loop engineering: stop prompting your agent by hand and design the system that prompts it. One optimization loop, one design skill, and a four-agent team with the maker–checker split built in.

## What's inside

| Component | What it does |
| --- | --- |
| `skills/autoloop` | Autonomous improvement loop: try a change, measure it with a shell command, keep what improves, `git reset` what doesn't. Resumable via a TSV log; schedulable; parallel beam search via worktrees. |
| `skills/loop-design` | Turns a vague goal ("make this faster") into a ready-to-run autoloop invocation: picks a non-gameable metric, draws scope, chooses a verifier. |
| `agents/scout` | Read-only explorer (haiku). Maps the repo, finds candidate work. |
| `agents/forge` | Maker. Implements one hypothesis in an isolated git worktree (`isolation: worktree`). Never commits, never widens the job. |
| `agents/critic` | Adversarial verifier (opus). Runs the tests itself, reads the real diff, rejects anything unverifiable. The maker never gets the final word. |
| `agents/scribe` | Memory keeper (haiku). Maintains the experiment log and state files so runs resume instead of restarting. |

## Install

From this repo as a marketplace:

```
/plugin marketplace add <github-user>/looping
/plugin install loop-engineering@looping
```

Or test locally:

```
claude --plugin-dir ./loop-engineering
```

## Use

Design a loop from a vague goal:

```
/loop-engineering:loop-design make the user guide easier to read
```

Run a loop directly:

```
/loop-engineering:autoloop
goal: minimize production Docker image size in MB
metric: docker image inspect app --format '{{.Size}}' | awk '{print $1/1048576}'
direction: down
scope: Dockerfile, .dockerignore
time_budget: 10
target: 100
```

Schedule it: invoke the same command from cron/CI with a finite `max_iters` per run — the TSV log makes every run resume from the best kept commit.

## The design

Every loop here follows five steps: define a measurable goal (metric + direction + target = one stop condition), scope it (scope/frozen), trigger it (in-session or scheduled), act then verify (forge makes, critic checks — never the same agent), persist state (log + git survive between runs). Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) and the loop-engineering pattern described by [Addy Osmani](https://addyosmani.com/blog/loop-engineering/).
