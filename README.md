# loop-engineering

A Claude Code plugin for loop engineering: stop prompting your agent by hand and design the system that prompts it. One optimization loop, one design skill, and a four-agent team with the maker–checker split built in.

## Install

```
/plugin marketplace add scgopi/loop-engineering
/plugin install loops@loop-engineering
```

## What's inside

| Component | What it does |
| --- | --- |
| `skills/optimize` | Autonomous improvement loop: try a change, measure it with a shell command, keep what improves, `git reset` what doesn't. Resumable via a TSV log; schedulable; parallel beam search via worktrees. |
| `skills/design` | Turns a vague goal ("make this faster") into a ready-to-run optimize invocation: picks a non-gameable metric, draws scope, chooses a verifier. |
| `agents/scout` | Read-only explorer. Maps the repo, finds candidate work. |
| `agents/forge` | Maker. Implements one hypothesis in an isolated git worktree (`isolation: worktree`). Never commits, never widens the job. |
| `agents/critic` | Adversarial verifier. Runs the tests itself, reads the real diff, rejects anything unverifiable. The maker never gets the final word. |
| `agents/scribe` | Memory keeper. Maintains the experiment log and state files so runs resume instead of restarting. |

## Use

Design a loop from a vague goal:

```
/loops:design make the user guide easier to read
```

Run a loop directly:

```
/loops:optimize
goal: minimize production Docker image size in MB
metric: docker image inspect app --format '{{.Size}}' | awk '{print $1/1048576}'
direction: down
scope: Dockerfile, .dockerignore
time_budget: 10
target: 100
```

Or test locally without installing:

```
claude --plugin-dir ./loops
```

## Design

Every loop follows five steps: define a measurable goal (metric + direction + target = one stop condition), scope it (scope/frozen), trigger it (in-session or scheduled), act then verify (forge makes, critic checks — never the same agent), persist state (log + git survive between runs). Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) and the loop-engineering pattern described by [Addy Osmani](https://addyosmani.com/blog/loop-engineering/).
