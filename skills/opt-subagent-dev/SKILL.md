---
name: opt-subagent-dev
description: Use when executing optimization plans with parallel subagents — dispatches one subagent per optimization point in isolated git worktrees, each runs implement + three-layer validation, then aggregates and compares results
---

# Optimization Subagent-Driven Development

Execute optimization plans by dispatching one subagent per optimization point in parallel. Each subagent implements the optimization and runs three-layer validation (L0 → L1 → L2) independently. The orchestrator aggregates results and hands off to opt-verification.

**This is a RIGID skill.** Follow the execution model exactly.

**Adapted from:** `superpowers:subagent-driven-development`. Key changes:
- **Parallel dispatch** (not sequential) — optimization points are independent
- **Git worktree isolation** — each subagent works in its own worktree
- **No separate code review** — three-layer validation IS the review
- **Aggregation phase** — compare results across optimization points

## When to Use

- You have an optimization plan (from sp-opt:opt-planning)
- Optimization points are independent (no dependencies between them)
- You want to stay in this session

## Pre-Flight Checks

Before dispatching any subagent, verify:

1. **Plan exists and is complete** — read the plan file, confirm all optimization points have: bottleneck evidence, approach, files, expected gain, and target
2. **Baseline is recorded** — plan has a baseline record section with actual numbers
3. **Benchmark script exists** — verify the script path in the plan is valid and runnable
4. **Independent points** — confirm optimization points don't have undeclared dependencies (modifying same files without marking it)

If any check fails, stop and fix before proceeding.

## Execution Model

```
orchestrator
    │
    ├─ Pre-flight checks
    │
    ├─ Create worktrees (one per independent point)
    │   Uses: superpowers:using-git-worktrees
    │
    ├─ Parallel dispatch ──────────────────────────────
    │   │                    │                    │
    │   ▼                    ▼                    ▼
    │  subagent A           subagent B           subagent C
    │  (worktree-a)         (worktree-b)         (worktree-c)
    │   │                    │                    │
    │   ├ Confirm baseline   ├ Confirm baseline   ├ Confirm baseline
    │   ├ Implement          ├ Implement          ├ Implement
    │   ├ L0: static-checks  ├ L0: static-checks  ├ L0: static-checks
    │   ├ L1: precision      ├ L1: precision       ├ L1: precision
    │   ├ L2: benchmark      ├ L2: benchmark       ├ L2: benchmark
    │   └ Output report      └ Output report       └ Output report
    │
    ├─ Collect all reports ────────────────────────────
    │
    ├─ Rank by gain
    │
    └─ Transition to sp-opt:opt-verification
```

**Dependent points:** If the plan marks dependencies between points, execute them sequentially (dependent point starts after its dependency completes and passes validation).

## Implementer Subagent Prompt

When dispatching each subagent, use this prompt template:

```markdown
# Optimization Task: [Point Name]

## Context
- Project: [from plan]
- Optimization goal: [from plan]
- Baseline performance: [from plan baseline record]
- Precision requirement: [from plan]

## Your Task
[Copy the full optimization point section from the plan, including:
 bottleneck, approach, expected gain, files involved, implementation steps]

## Benchmark Script
- Path: [from plan]
- Baseline command: [from plan]
- Optimized command: [from plan]

## Work Steps

### 1. Confirm baseline
Run the benchmark script, verify your baseline matches the plan's recorded baseline (within 5% tolerance). If it doesn't match, STOP and report — the environment may have changed.

### 2. Implement the optimization
Follow the implementation steps from the plan. Modify only the files listed.

### 3. L0: Static checks
Review your changes against the opt-static-checks checklist:
- dtype consistency, shape consistency, operator equivalence
- control flow, side effects, numerical stability preserved
- Report PASS/FAIL for each mandatory check

### 4. L1: Precision validation
- Prepare test inputs (typical, boundary, extreme)
- Run original code path → save outputs
- Run optimized code path → save outputs
- Compare: max_abs_diff, mean_abs_diff, cosine_sim, max_rel_diff
- Thresholds: [from plan precision requirement]

### 5. L2: Performance benchmark
- Run benchmark: warmup 3 + measure 10 rounds
- Record: throughput, latency p50/p99, memory peak/resident
- Compute speedup ratio and statistical significance
- Compare against target: [from plan expected gain]

## Report Format

Submit this exact format:

### Optimization Point: [Name]

#### Implementation Summary
[What was changed, which files, key design decisions]

#### L0: Static Checks
| # | Check | Result |
|---|-------|--------|
[All 10 checks]

#### L1: Precision Validation
| Metric | Value | Threshold | Status |
|--------|-------|-----------|--------|
[All 4 metrics]

#### L2: Performance Benchmark
| Metric | baseline | optimized | speedup | significant? |
|--------|----------|-----------|---------|-------------|
[All required metrics]

#### Conclusion
- Target: [from plan]
- Result: MET / NOT MET / PARTIALLY MET
- If not met: [reason]
```

## Fix Loop

When a validation layer fails:

```
L0 fails → subagent fixes implementation → re-run L0
L1 fails → subagent investigates (likely bug) → fix → re-run L0 + L1
L2 fails (no speedup) → trigger perf-diagnostics within subagent
    → 3 failures at any layer → escalate to orchestrator
    → 5 failures total → stop subagent, report to user
```

**Precision failure vs performance failure — different paths:**
- **L1 (precision) fails:** Implementation bug. Review the diff, find where numerical behavior changed. May need to revert and try a different approach.
- **L2 (performance) not met:** Not necessarily a bug. Run perf-diagnostics to understand why. Could be: bottleneck shift, measurement noise, wrong optimization target, theoretical limit.

## Aggregation Phase

After all subagents complete (or fail):

1. **Collect reports** — gather the report from each subagent
2. **Rank by gain** — sort optimization points by their primary metric speedup
3. **Flag conflicts** — check if any two points modified the same files
4. **Summarize** — present a comparison table to the orchestrator

```
## Aggregation Summary

| Point | L0 | L1 | L2 speedup | Target | Status |
|-------|----|----|-----------|--------|--------|
| SDPA | PASS | PASS | 1.35x | 1.20x | MET |
| CUDA graph | PASS | PASS | 1.18x | 1.20x | PARTIAL |
| Double buffer | PASS | FAIL | — | 1.10x | FAILED |

Conflicts: [none / list of conflicting points]
```

5. **Transition** — hand off to `sp-opt:opt-verification` for combination testing and final report

## Key Rules

- **Never skip validation layers.** L0 before L1 before L2. Always.
- **Each subagent is fresh.** No shared state between subagents. Each gets its own worktree and full context from the plan.
- **Baseline must match.** If a subagent's baseline doesn't match the plan's baseline (>5% deviation), something is wrong. Stop and investigate.
- **Independence is real.** If two "independent" points turn out to have hidden dependencies (e.g., both import the same utility), flag as a conflict.
