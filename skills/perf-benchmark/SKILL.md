---
name: perf-benchmark
description: Use when measuring the actual performance impact of an optimization — runs warmup + measurement rounds for both baseline and optimized code, compares throughput/latency/memory with statistical significance checks. L2 in the three-layer validation.
---

# L2: Performance Benchmark

Quantify the actual performance gain of an optimization. Runs structured benchmarks with warmup, multiple measurement rounds, and statistical significance testing.

**This is a RIGID skill.** Follow exactly. Do not skip warmup. Do not skip significance checks.

## When to Use

- Called by sp-opt:opt-subagent-dev after L1 passes
- Can be invoked standalone to benchmark any code change
- Also called during opt-verification combination stacking

## Prerequisites

- L1 (precision-validator) must have passed
- A benchmark script must exist (required by opt-planning; if missing, create one first)

## Benchmark Script Requirements

If no benchmark script exists, create one before proceeding. Requirements:

- Independently runnable (no dependency on training loop or other infrastructure)
- Outputs formatted performance data (throughput, latency, memory)
- Supports warmup rounds (default 3) and measurement rounds (default 10)
- Reports mean and standard deviation for each metric
- Supports switching between baseline and optimized code paths (e.g., `--mode baseline` / `--mode optimized`, or environment variable, or git branch)
- Uses `torch.cuda.synchronize()` before timing measurements
- Clears CUDA cache between runs: `torch.cuda.empty_cache()`

## Flow

```
1. Confirm benchmark script exists and runs
2. Run baseline: warmup 3 rounds + measure 10 rounds
3. Run optimized: warmup 3 rounds + measure 10 rounds
4. Compute statistics and significance
5. Report results
```

## Required Metrics

| Metric | Unit | How to measure |
|--------|------|----------------|
| throughput | samples/s or tokens/s | total_items / elapsed_time per round |
| latency p50 | ms | median of per-iteration times |
| latency p99 | ms | 99th percentile of per-iteration times |
| memory peak | GB | torch.cuda.max_memory_allocated() |
| memory resident | GB | torch.cuda.memory_allocated() after forward pass |

## Optional Metrics (record if applicable)

- **MFU** — for compute-bound scenarios, compute Model FLOPs Utilization
- **TTFT** — for inference scenarios, Time To First Token
- **inter-batch variance** — stddev of per-batch times, indicates stability

## Measurement Protocol

1. **Warmup** (3 rounds, not counted): Ensures JIT compilation, CUDA context, memory allocation are settled
2. **Measurement** (10 rounds): Each round records all metrics independently
3. **Between rounds**: Call `torch.cuda.synchronize()` before start/stop timing
4. **Between baseline/optimized**: Call `torch.cuda.empty_cache()` and reset `max_memory_allocated`

## Statistical Significance

- Compute mean and stddev for each metric across 10 rounds
- Speedup ratio = baseline_mean / optimized_mean
- Compute 95% confidence interval using t-distribution (df=9 for 10 samples)
- If the confidence interval of the speedup ratio includes 1.0 → mark as **"not significant"**
- Report significance alongside each metric

## Output Format

```
## L2 Performance Benchmark Results

### Configuration
- Benchmark script: [path]
- Warmup rounds: 3
- Measurement rounds: 10
- Hardware: [GPU model]
- Input: [description of benchmark input]

### Results
| Metric | baseline (mean±std) | optimized (mean±std) | speedup | 95% CI | significant? |
|--------|--------------------|--------------------|---------|--------|-------------|
| throughput | 100±3 samples/s | 135±2 samples/s | 1.35x | [1.31, 1.39] | yes |
| latency p50 | 50±1ms | 37±1ms | 1.35x | [1.32, 1.38] | yes |
| latency p99 | 55±2ms | 40±2ms | 1.38x | [1.30, 1.45] | yes |
| memory peak | 8.2±0.0GB | 7.1±0.0GB | -13% | — | yes |
| memory resident | 6.1±0.0GB | 5.5±0.0GB | -10% | — | yes |

### Verdict: MET TARGET / NOT MET / PARTIALLY MET
- Target: [from plan, e.g. "throughput +20%"]
- Actual: [actual speedup]
- [If not met: which metrics fell short]
```

## Failure Criteria

| Condition | Action |
|-----------|--------|
| No improvement or regression | Trigger sp-opt:perf-diagnostics |
| Gain below target | Mark "partially met," continue to opt-verification |
| Gain not statistically significant | Mark "not significant," recommend more measurement rounds or larger input |
| Memory increase > 10% | Warn (unless explicitly accepted in opt-brainstorming) |
| High variance (stddev > 10% of mean) | Warn, recommend investigating variability sources |

## Your Job

1. Create the file with EXACTLY the content above (including the frontmatter between --- markers) using the Write tool
2. Do NOT commit
