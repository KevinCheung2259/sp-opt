# SP-Opt — Superpowers for Optimization

SP-Opt is an addon plugin for [Superpowers](https://github.com/obra/superpowers) that extends it with performance optimization workflows: profiling-guided bottleneck analysis, parallel optimization exploration, and three-layer validation.

Superpowers provides the foundation — TDD, code review, subagent architecture, git worktrees. SP-Opt adds the performance optimization domain knowledge on top.

## What makes performance optimization different

In regular software, "it works" means done. In performance optimization, "it works" is just the starting point — you need to prove it's **faster**, **numerically equivalent**, and **stable**.

SP-Opt addresses this with:
- **Profiling-guided brainstorming** — don't guess bottlenecks, measure them
- **Parallel optimization exploration** — try multiple approaches simultaneously, pick the winner
- **Three-layer validation** — static correctness → precision equivalence → performance benchmark
- **Combination verification** — optimizations that work alone may interact when combined

## Installation

### Prerequisites

Install [Superpowers](https://github.com/obra/superpowers) first.

### Claude Code

In Claude Code, run:

```
/plugin marketplace add KevinCheung2259/sp-opt
/plugin install sp-opt@KevinCheung2259
```

### Verify Installation

Start a new session and check that SP-Opt skills are available:

```
/sp-opt:opt-brainstorming    → start a new optimization task
/sp-opt:perf-diagnostics     → diagnose performance issues
```

## The Optimization Workflow

```
opt-brainstorming           "What's slow and why?"
    |                        Profile -> identify bottlenecks -> propose strategies
    |
opt-planning                "How to optimize, which points to try?"
    |                        Break into parallel optimization points
    |
opt-subagent-dev            "Execute in parallel"
    |  +-- Point A              Each point in its own worktree:
    |  +-- Point B              implement -> static check -> precision -> benchmark
    |  +-- Point C
    |
opt-verification            "Compare, combine, report"
                             Stack winners, verify combinations, final report
```

Side-channel: `perf-diagnostics` — triggered when optimization doesn't deliver expected gains.

## Three-Layer Validation

| Layer | Skill | What it catches | Duration |
|-------|-------|----------------|----------|
| L0 | opt-static-checks | Equivalence violations, stability regressions | Seconds (code review) |
| L1 | precision-validator | Numerical divergence beyond tolerance | Minutes |
| L2 | perf-benchmark | Performance gain with statistical significance | Minutes |

## Skills

| Skill | Type | Purpose |
|-------|------|---------|
| using-sp-opt | Entry router | Route to correct skill |
| opt-brainstorming | Process | Profiling guidance + bottleneck hypothesis |
| opt-planning | Process | Multi-point parallel optimization plan |
| opt-subagent-dev | Process | Parallel subagent execution engine |
| opt-static-checks | Validation (L0) | Static correctness check |
| precision-validator | Validation (L1) | Numerical equivalence validation |
| perf-benchmark | Validation (L2) | Performance comparison + significance |
| perf-diagnostics | Diagnosis | Bottleneck analysis + shift detection |
| opt-verification | Process | Combination verification + final report |

## From Superpowers (used via cross-plugin reference)

TDD, systematic-debugging, brainstorming, writing-plans, dispatching-parallel-agents, using-git-worktrees, requesting/receiving-code-review, finishing-a-development-branch — all provided by Superpowers.

## License

MIT License
