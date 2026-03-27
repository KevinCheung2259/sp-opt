---
name: perf-diagnostics
description: Use when performance optimization doesn't deliver expected gains, performance regresses, or you need to identify bottlenecks — systematic 4-phase diagnosis covering compute, memory, IO, and communication bottlenecks with bottleneck-shift detection
---

# Performance Diagnostics

Systematic diagnosis for performance optimization problems. Triggered when an optimization doesn't work as expected, performance regresses, or bottlenecks need identification.

**This is a RIGID skill.** Follow the four phases in order. Do not skip Phase 1.

## When to Use

- perf-benchmark shows optimization ineffective or below expectation
- Performance regressed after optimization
- Need to identify bottlenecks before opt-brainstorming (no profiling data yet)
- Bottleneck shifted after an optimization (optimized A, but B became slow)

## Three Core Questions

| Question | Trigger | Key evidence |
|----------|---------|-------------|
| Q1: Where's the bottleneck? | No profiling data, or bottleneck shifted | Per-stage timing, GPU utilization |
| Q2: Why didn't it speed up? | perf-benchmark shows no gain or below target | Fine-grained timing around optimization point |
| Q3: Where did the memory go? | Memory peak increased, or OOM | Memory snapshot, allocation list |

Identify which question you're answering before starting. Multiple questions can apply simultaneously.

## Phase 1: Evidence Collection

<HARD-GATE>
Do NOT propose fixes, hypotheses, or root causes until Phase 1 is complete.
Collect evidence first. Always.
</HARD-GATE>

### Evidence by question

**Q1 — Where's the bottleneck?**

| Evidence | Tool/Command | What to look for |
|----------|-------------|-----------------|
| Per-stage timing | `torch.profiler` with `record_shapes=True` | Which stage dominates wall time |
| GPU utilization timeline | `nvidia-smi dmon -s u -d 1` | Gaps in GPU utilization = CPU bottleneck |
| Kernel-level breakdown | `torch.profiler` table sorted by CUDA time | Which kernels dominate |
| IO timing | Manual timing around DataLoader | Data loading vs compute ratio |
| Communication timing | `torch.profiler` with `profile_memory=True` | NCCL collective time |

**Q2 — Why didn't it speed up?**

| Evidence | Tool/Command | What to look for |
|----------|-------------|-----------------|
| Fine-grained timing | `torch.cuda.Event` around optimized section | Actual time saved in the optimized section |
| Kernel comparison | `torch.profiler` before vs after | Did kernel count/duration actually change? |
| Launch overhead | `nsight systems` timeline | Many small kernels = launch-bound |
| Memory bandwidth | `torch.profiler` memory view | Memory-bound kernels won't speed up from compute optimization |
| Sync points | Search for `.item()`, `.cpu()`, `synchronize()` | Hidden synchronization killing parallelism |

**Q3 — Where did the memory go?**

| Evidence | Tool/Command | What to look for |
|----------|-------------|-----------------|
| Memory snapshot | `torch.cuda.memory_snapshot()` | Which allocations are largest |
| Peak moment | `torch.cuda.max_memory_allocated()` reset between steps | When peak occurs |
| Fragmentation | `torch.cuda.memory_stats()['active_bytes.all.peak']` vs `allocated_bytes` | High fragmentation = frequent alloc/free |
| Tensor lifecycle | Manual inspection | Tensors held longer than needed |
| Gradient accumulation | Check grad storage | Gradients not freed between micro-batches |

## Phase 2: Pattern Analysis

Match collected evidence against common performance anti-patterns:

| Category | Symptoms | Common Causes |
|----------|----------|--------------|
| Compute-bound | High GPU utilization, kernel time dominates | Low operator efficiency, not using tensor cores, suboptimal tile size |
| Memory-bound | Low compute utilization, high bandwidth usage | Bandwidth saturated, unnecessary copies, poor access patterns |
| IO-bound | Low GPU utilization with gaps, DataLoader dominates | Slow disk, insufficient num_workers, no prefetch |
| Communication-bound | Spikes at sync points, idle between collectives | Large all-reduce, gradient bucketing too small, pipeline bubbles |
| Launch-bound | Many short kernels, high CPU overhead | Too many small ops, Python-level loops, excessive eager sync |
| Bottleneck shift | Optimized section faster, but total time unchanged | Different stage became the new bottleneck |

### Bottleneck Shift Detection

**This check is mandatory when Q2 applies.** After optimization, compare the time distribution across stages:

```
Before: forward 60% | backward 30% | dataloader 10%
After:  forward 30% | backward 30% | dataloader 40%  ← dataloader is new bottleneck
Total time only improved 30%, not the expected 50%
```

If time distribution shifted significantly, report the new bottleneck and recommend it as a candidate for the next optimization round.

## Phase 3: Hypothesis Testing

- **One hypothesis at a time.** Do not propose multiple fixes simultaneously.
- **Minimal change to validate.** E.g., disable one optimization to see if behavior recovers; add a single `synchronize()` to test if async is the issue.
- **Record each hypothesis:** what you tested, what you expected, what actually happened.

**3-strike rule:** After 3 failed hypotheses → STOP. Do not continue guessing. Step back and:
1. Question your understanding of the system
2. Look for evidence you missed in Phase 1
3. Discuss with the user — the problem may require domain knowledge you don't have

## Phase 4: Fix and Verify

1. Implement the fix based on the validated hypothesis
2. Re-run precision-validator (ensure fix doesn't break correctness)
3. Re-run perf-benchmark (verify the fix actually improves performance)
4. Compare against the original baseline (not just the broken state)

## Output Format

```
## Performance Diagnostics Report

### Question(s): Q1 / Q2 / Q3

### Phase 1: Evidence
[Collected evidence with actual numbers]

### Phase 2: Pattern Match
- Primary pattern: [category]
- Evidence: [specific metrics pointing to this pattern]
- Bottleneck shift: [detected / not detected]

### Phase 3: Hypotheses Tested
| # | Hypothesis | Test | Expected | Actual | Result |
|---|-----------|------|----------|--------|--------|
| 1 | [hypothesis] | [what you did] | [expected] | [actual] | confirmed/rejected |

### Phase 4: Fix
- Fix: [what was changed]
- Precision impact: [L1 results after fix]
- Performance impact: [L2 results after fix]

### Recommendation
[Next steps: continue optimizing, new optimization point, or accept current result]
```

## Your Job

1. Create the file with EXACTLY the content above (including the frontmatter between --- markers) using the Write tool
2. Do NOT commit
