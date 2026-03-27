---
name: opt-planning
description: Use when you have an optimization design to break into parallel optimization points — creates structured plans with baseline records, per-point implementation steps, validation criteria, and combination strategy
---

# Optimization Planning

Write comprehensive optimization plans assuming the engineer has zero context. Document everything: which files to touch, what to measure, how to validate, what the expected gains are. Break into independent optimization points that can be explored in parallel.

**Announce at start:** "I'm using the opt-planning skill to create the optimization plan."

**Save plans to:** `docs/plans/YYYY-MM-DD-<optimization-name>.md`

## Key Design: Parallel Optimization Points

Unlike sequential task plans, optimization points are typically **independent and parallel**. Each point:
- Targets a specific bottleneck
- Has its own expected gain
- Can be implemented and validated independently
- Lives in its own git worktree during execution

```
Optimization Plan
├── Point A: SDPA attention        ← independent worktree
├── Point B: CUDA graph            ← independent worktree
├── Point C: KV cache optimization ← independent worktree
└── Baseline (untouched, for comparison)
```

## Plan Document Structure

Every plan MUST use this structure:

```markdown
# [Project] Performance Optimization Plan

> **For Claude:** Use sp-opt:opt-subagent-dev to execute this plan.

**Goal:** [one sentence, e.g. "Reduce PI0 inference latency from 50ms to 30ms"]
**Baseline:** [current performance — exact numbers from profiling]
**Hardware:** [GPU model x count, e.g. "1x H100 80GB"]
**Precision requirement:** [e.g. "max abs diff < 1e-5, cosine sim > 0.999"]

## Baseline Record

| Metric | Current Value | Measurement Method | Command |
|--------|--------------|-------------------|---------|
| throughput | [X] samples/s | [tool] | `[exact command]` |
| latency p50 | [X] ms | [tool] | `[exact command]` |
| latency p99 | [X] ms | [tool] | `[exact command]` |
| memory peak | [X] GB | torch.cuda.max_memory_allocated | `[exact command]` |

## Benchmark Script

Path: `[exact path]`
Exists: yes / no (if no, first optimization point must create it)

Requirements for benchmark script (if creating):
- Independently runnable
- Warmup 3 rounds + measure 10 rounds
- Reports mean ± stddev for each metric
- Supports --mode baseline / --mode optimized

## Optimization Points

### Point N: [Name]

**Bottleneck:** [profiling evidence with numbers]
**Approach:** [what to change]
**Expected gain:** [quantified, e.g. "attention 2x faster → overall ~30%"]
**Precision impact:** [expected, e.g. "numerically equivalent"]
**Files:** [exact paths to modify]
**Dependencies:** None / Point X must complete first

**Implementation steps:**
1. [Specific step with exact file and code description]
2. [Next step]
...

**Validation:**
- L0: [any point-specific static checks beyond the standard 10]
- L1: precision thresholds — [from design, or default]
- L2: target — [specific metric and threshold]
```

## Dependency Handling

Most optimization points should be independent. When dependencies exist:

- **Mark explicitly:** `Dependencies: Point 1 must complete first`
- **Reason:** e.g. "Point 2 builds on Point 1's CUDA graph by adding KV cache inside the graph"
