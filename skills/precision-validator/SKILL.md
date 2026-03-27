---
name: precision-validator
description: Use when validating that an optimization preserves numerical correctness — runs identical inputs through original and optimized code, compares output tensors against configurable tolerance thresholds. L1 in the three-layer validation.
---

# L1: Precision Validator

Run identical inputs through original and optimized code paths, compare every output tensor against tolerance thresholds. Separates "optimization broke correctness" from "optimization is correct but slow."

**This is a RIGID skill.** Follow exactly. Do not skip steps.

## When to Use

- Called by sp-opt:opt-subagent-dev after L0 passes
- Can be invoked standalone to validate any code change that should be numerically equivalent

## Prerequisites

- L0 (opt-static-checks) must have passed
- Both original and optimized code paths must be runnable
- A way to produce identical inputs for both paths

## Flow

```
1. Prepare test inputs
2. Run original code → record output tensors
3. Run optimized code → record output tensors
4. Per-tensor comparison against thresholds
5. Report results
```

## Step 1: Prepare Test Inputs

Generate or sample inputs that cover three categories:

| Category | Purpose | Example |
|----------|---------|---------|
| Typical | Normal operating range | Average-length sequence, typical batch size |
| Boundary | Edge cases for shape handling | Very long/short sequence, batch_size=1, empty input |
| Extreme values | Numerical edge cases | Very large/small values, zeros, near-overflow |

**Rules:**
- Use **identical inputs** for both code paths (not independent random generation)
- Fix all random seeds: `torch.manual_seed(42)`, `np.random.seed(42)`, `random.seed(42)`
- If optimization changes batching behavior, test at least 3 different batch sizes
- Store inputs so they can be reused if re-validation is needed

## Step 2-3: Run Both Code Paths

- Run original code, save all output tensors (detached, on CPU)
- Run optimized code with same inputs, save all output tensors
- Ensure both runs use the same device, same dtype, same random state

## Step 4: Compare

**Default thresholds:**

| Metric | Default | Description |
|--------|---------|-------------|
| max abs diff | < 1e-5 | Largest absolute difference across all elements |
| mean abs diff | < 1e-6 | Average absolute difference |
| cosine similarity | > 0.99999 | Directional consistency of flattened tensors |
| max relative diff | < 1e-4 | Largest relative difference (|a-b|/max(|a|,|b|,1e-8)) |

**Threshold override:** Thresholds are set during opt-brainstorming and recorded in the plan. Common overrides:
- fp16 optimization: relax max_abs_diff to 1e-3, cosine to 0.999
- Approximate algorithms (e.g., approximate softmax): relax per user specification
- Quantization: custom thresholds per use case

## Step 5: Report

**Output format:**

```
## L1 Precision Validation Results

### Test Inputs
| Category | Description | Shape |
|----------|-------------|-------|
| Typical | [description] | [shape] |
| Boundary | [description] | [shape] |
| Extreme | [description] | [shape] |

### Per-Tensor Results
| Tensor | max_abs_diff | mean_abs_diff | cosine_sim | max_rel_diff | Status |
|--------|-------------|---------------|------------|-------------|--------|
| output[0] | 2.3e-6 | 1.1e-7 | 0.999999 | 8.5e-5 | PASS |
| output[1] | 1.2e-3 | 5.4e-4 | 0.998 | 2.1e-2 | FAIL |

### Failure Details (if any)
- output[1]: max_abs_diff 1.2e-3 exceeds threshold 1e-5
  - Max divergence at index [32, 128]: original=0.5123, optimized=0.5135
  - Input category: extreme values

### Verdict: PASS / FAIL
[If FAIL: which tensors failed, which metrics exceeded, which input categories triggered failure]
```

## Failure Handling

- **Single metric fails on one input category** → Report specifics, likely a numerical edge case. Implementer should investigate that code path.
- **Multiple metrics fail across categories** → Likely an implementation bug. Recommend falling back to L0 static checks to find the root cause.
- **Only extreme-value inputs fail** → May be acceptable depending on the use case. Report to user for decision.

## Fix Loop

```
L1 fails → Implementer fixes → re-run L1 (all inputs, not just failing ones)
    → 3 failures → likely fundamental issue, trigger perf-diagnostics or escalate
    → 5 failures → stop, report to user
```
