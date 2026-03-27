---
name: opt-verification
description: Use when all optimization points are complete — aggregates results, detects conflicts, verifies optimization combinations with incremental stacking, and produces a final report with recommendations
---

# Optimization Verification

Final gate after all optimization points complete. Compare gains, detect conflicts, verify combinations through incremental stacking, and produce the final optimization report.

**This is a RIGID skill.** Follow the full checklist. Do not skip combination verification.

<HARD-GATE>
Do NOT claim optimization is complete without:
1. All optimization point reports collected
2. Conflict detection completed
3. Combination verification passed (if multiple points succeeded)
4. Final report written
</HARD-GATE>

## When to Use

- All optimization point subagents have completed (from opt-subagent-dev)
- Ready to decide which optimizations to keep and how to combine them

## Checklist

### 1. Single-Point Summary

Collect results from all optimization points into a comparison table:

```
| Point | L0 | L1 max_diff | L2 throughput | L2 latency | L2 memory | Target | Status |
|-------|----|-----------|--------------|-----------|---------|---------|---------|
| SDPA | PASS | 2.3e-6 | +35% | -26% | -5% | +20% | MET |
| CUDA graph | PASS | 0 | +18% | -15% | +2% | +20% | PARTIAL |
| Double buffer | PASS | 0 | +8% | -7% | +12% | +10% | NOT MET |
```

Categorize each point:
- **MET** — all validation layers passed AND performance target reached
- **PARTIALLY MET** — all validation layers passed but performance below target
- **NOT MET** — validation failed or no meaningful improvement
- **FAILED** — could not complete (errors, crashes)

### 2. Conflict Detection

Check whether any two optimization points modified the same files:

```bash
# For each pair of points, compare their git diffs
git diff worktree-a...main --name-only
git diff worktree-b...main --name-only
# Intersection = conflict
```

- **No conflict** → Points can be combined freely
- **Conflict detected** → List the conflicting files and which points touch them. User must choose:
  - Keep one, discard the other
  - Manually merge (with re-validation after merge)

### 3. Combination Verification

**Only for points with status MET or PARTIALLY MET.**

Stack optimizations one at a time, ordered by gain (largest first). After each addition, re-run L1 (precision) and L2 (performance):

```
Step 1: baseline
Step 2: + SDPA → run L1 + L2 → record incremental gain
Step 3: + CUDA graph → run L1 + L2 → record incremental gain
Step 4: + next point → ...
```

**Rules:**
- If adding a point causes L1 failure → that point interacts badly with previous optimizations. Remove it and report.
- If adding a point shows no incremental gain (or regression) → likely the gains don't stack. Report the interaction.
- Record the incremental contribution of each point in the combination.

### 4. Final Report

```markdown
# [Project] Performance Optimization Report

## Baseline
| Metric | Value |
|--------|-------|
| throughput | [X] samples/s |
| latency p50 | [X] ms |
| latency p99 | [X] ms |
| memory peak | [X] GB |

## Final Result (after combination)
| Metric | Value | vs Baseline |
|--------|-------|-------------|
| throughput | [X] samples/s | +[X]% |
| latency p50 | [X] ms | -[X]% |
| latency p99 | [X] ms | -[X]% |
| memory peak | [X] GB | [+/-X]% |

## Per-Point Contribution (stacking order)
| Step | Added | Incremental throughput | Cumulative throughput | L1 status |
|------|-------|----------------------|----------------------|-----------|
| 0 | baseline | — | 100 samples/s | — |
| 1 | + SDPA | +35 | 135 samples/s | PASS |
| 2 | + CUDA graph | +15 | 150 samples/s | PASS |

## Precision Impact (combined)
| Metric | Value | Threshold | Status |
|--------|-------|-----------|--------|
| max abs diff | [X] | [threshold] | PASS/FAIL |
| cosine similarity | [X] | [threshold] | PASS/FAIL |

## Points Not Included
| Point | Reason |
|-------|--------|
| Double buffer | L2 not met (only +8%, target was +10%) |

## Recommendations
[Next optimization directions, remaining bottlenecks, or "optimization target fully met"]
```

### 5. Decision Handoff

Present options to the user:

| Option | When | Action |
|--------|------|--------|
| **Merge** | Happy with results | → `superpowers:finishing-a-development-branch` |
| **Code review** | Want team review before merge | → `superpowers:requesting-code-review` |
| **Continue optimizing** | Want more gains | → back to `sp-opt:opt-brainstorming` with updated profiling data |
| **Discard some points** | Don't want all optimizations | → clean up worktrees, re-run combination verification |
