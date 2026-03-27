---
name: using-sp-opt
description: Use when starting any conversation that may involve performance optimization — routes to the correct SP-Opt skill based on task type, requiring skill invocation before any response
---

<SUBAGENT-STOP>
If you were dispatched as a subagent to execute a specific task, skip this skill.
</SUBAGENT-STOP>

<EXTREMELY-IMPORTANT>
If you think there is even a 1% chance an SP-Opt skill might apply to what you are doing, you ABSOLUTELY MUST invoke the skill.
</EXTREMELY-IMPORTANT>

# SP-Opt: Superpowers for Optimization

SP-Opt extends [Superpowers](https://github.com/obra/superpowers) with performance optimization workflows. Use SP-Opt for training/inference efficiency work. Use Superpowers for general software development.

## Scope Gate

Before considering any sp-opt:* skill, determine: **Is this a performance optimization task?**

**SP-Opt applies when the goal is:**
- Making code faster (throughput, latency)
- Reducing memory usage
- Profiling / benchmarking
- Diagnosing performance problems
- Comparing optimization approaches

**Superpowers applies when the goal is (even if ML-related):**
- Writing new features
- Fixing bugs (non-performance)
- Refactoring
- CI/CD, docs, tests

## Routing Rules

| Signal | Route to |
|--------|----------|
| New optimization task, need to analyze bottlenecks | `sp-opt:opt-brainstorming` |
| Have a design, need to create an optimization plan | `sp-opt:opt-planning` |
| Have a plan, ready to execute | `sp-opt:opt-subagent-dev` |
| "Why is it slow?" / "Why didn't optimization work?" | `sp-opt:perf-diagnostics` |
| All optimization points done, need to compare and combine | `sp-opt:opt-verification` |
| Not performance optimization | Superpowers original skills |

## Keywords

These indicate SP-Opt scope: accelerate, optimize, speed up, throughput, latency, memory, profiling, benchmark, CUDA graph, kernel fusion, memory optimization, IO optimization, batch optimization, pipeline optimization, inference optimization, training speed, tokens/s, samples/s, MFU, FLOPS.

## Skill Priority

1. **perf-diagnostics** — if user arrives with "why slow" or "optimization didn't work," diagnose first
2. **opt-brainstorming** — new optimization tasks
3. **opt-planning** — have a design, need a plan
4. **opt-subagent-dev** — have a plan, ready to execute
5. **opt-verification** — completion stage

## Skill Types

- **Rigid** (follow exactly): opt-subagent-dev, opt-static-checks, precision-validator, perf-benchmark, perf-diagnostics, opt-verification
- **Flexible** (adapt to context): opt-brainstorming, opt-planning

## Three-Layer Validation

SP-Opt uses a three-layer validation instead of Superpowers' standard code review:

| Layer | Skill | What it checks | When |
|-------|-------|---------------|------|
| L0 | sp-opt:opt-static-checks | Optimization doesn't break correctness (static) | After implementation |
| L1 | sp-opt:precision-validator | Output numerically equivalent within tolerance | After L0 passes |
| L2 | sp-opt:perf-benchmark | Actual performance gain with significance | After L1 passes |

## Integration with Superpowers

SP-Opt uses these Superpowers skills directly:
- `superpowers:using-git-worktrees` — isolate each optimization point
- `superpowers:dispatching-parallel-agents` — run optimization points in parallel
- `superpowers:requesting-code-review` — optional code review at verification stage
- `superpowers:finishing-a-development-branch` — merge after verification
- `superpowers:test-driven-development` — when writing benchmark scripts
