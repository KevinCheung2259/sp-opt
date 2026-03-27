---
name: opt-brainstorming
description: Use before any performance optimization work — guides profiling, identifies bottlenecks, collects hardware/software context, establishes optimization hypotheses and validation criteria before implementation
---

# Optimization Brainstorming: Bottlenecks Into Optimization Plans

Help turn performance problems into structured optimization strategies through profiling-guided dialogue.

Start by understanding the current project and performance context, then ask questions one at a time to identify bottlenecks and define optimization approaches. Once you understand what to optimize, present the strategy and get user approval.

**Core principle:** Don't guess bottlenecks — measure them. An optimization targeting the wrong bottleneck wastes engineering time and may make things worse.

<HARD-GATE>
Do NOT invoke any implementation skill, write any code, or take any optimization action until you have presented a strategy and the user has approved it.
</HARD-GATE>

## Anti-Pattern: "This Optimization Is Obvious"

Every optimization goes through this process. "Just add CUDA graph" or "obviously memory-bound" — these assumptions are wrong often enough that skipping profiling wastes more time than it saves. The strategy can be short for truly simple cases, but you MUST present it and get approval.

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Explore project context** — check code, docs, recent commits, existing benchmarks
2. **Collect optimization context** — hardware, current performance, existing profiling
3. **Guide profiling** — if no performance data exists, help user profile first
4. **Identify bottlenecks** — from profiling data, identify what's slow and why
5. **Propose optimization hypotheses** — 2-3 approaches with expected gains and trade-offs
6. **Confirm validation scope** — which layers, precision tolerance, performance target
7. **Present strategy** — get user approval
8. **Write design doc** — save to `docs/plans/YYYY-MM-DD-<topic>-design.md` and commit
9. **Transition to implementation** — invoke `sp-opt:opt-planning`

## Process Flow

```
Explore project context
    │
Collect optimization context (one question at a time)
    │
Has profiling data? ─── No ──→ Guide profiling → collect results
    │ Yes                                │
    ▼                                    ▼
Identify bottlenecks from profiling data
    │
Propose 2-3 optimization strategies with trade-offs
    │
Confirm validation scope (L0/L1/L2 settings)
    │
Present strategy → user approves? ─── No ──→ revise
    │ Yes
    ▼
Write design doc → invoke sp-opt:opt-planning
```

**Terminal state:** Invoke `sp-opt:opt-planning`. Do NOT invoke any other skill.

## Collecting Optimization Context

Ask these one at a time. Skip what's already clear from project context:

- **Optimization target:** What metric matters most?
  - Training throughput (samples/s, tokens/s)
  - Inference latency (p50, p99, TTFT)
  - Inference throughput (requests/s, tokens/s)
  - Memory footprint (peak, resident)
  - Other?
- **Hardware environment:** GPU model, count, interconnect (NVLink/PCIe), CPU cores, RAM, storage type (SSD/NVMe)
- **Current performance baseline:** Do you have numbers? (throughput, latency, memory usage)
- **Existing profiling:** Have you profiled this? What tool? What did it show?
- **Known bottlenecks:** Do you already suspect what's slow?
- **Precision requirements:** Acceptable precision loss?
  - Zero loss (bitwise identical)
  - fp32-level tolerance (max_diff < 1e-5)
  - fp16-level tolerance (max_diff < 1e-3)
  - Custom threshold
- **Constraints:** What can't be changed?
  - Model architecture frozen?
  - Must be compatible with specific framework/version?
  - Must work on specific hardware?
  - Latency budget? Memory budget?

## Profiling Guidance

If the user has no performance data, guide them through profiling before proposing optimizations:

### Step 1: Coarse-grained timing

```python
# Wrap each major stage and measure wall time
import time
torch.cuda.synchronize()
start = time.perf_counter()
# ... stage code ...
torch.cuda.synchronize()
elapsed = time.perf_counter() - start
```

Measure: data loading, forward pass, backward pass, optimizer step, communication (if multi-GPU).

### Step 2: Identify bottleneck direction

| If... | Then likely... |
|-------|---------------|
| GPU utilization consistently >90% | Compute-bound |
| GPU utilization low, but GPU memory bandwidth high | Memory-bound |
| GPU utilization has frequent gaps | IO-bound or launch-bound |
| Time spikes at sync points | Communication-bound |

### Step 3: Fine-grained profiling

Based on direction, recommend specific tools:
- **Compute-bound:** `torch.profiler` with `record_shapes=True`, sorted by CUDA time
- **Memory-bound:** `torch.profiler` memory view, `torch.cuda.memory_stats()`
- **IO-bound:** Profile DataLoader separately, check `num_workers`, `pin_memory`
- **Communication-bound:** `torch.profiler` with distributed annotations
- **Launch-bound:** `nsight systems` timeline for kernel launch patterns

### Step 4: Establish baseline

Record concrete numbers that become the baseline for all future comparisons.

## Optimization Hypothesis Format

For each proposed optimization:

```
Bottleneck: [specific bottleneck with profiling evidence, e.g. "self-attention
  forward takes 45ms/step (60% of forward pass), profiler shows non-fused
  scaled_dot_product kernel"]
Optimization: [specific approach, e.g. "Replace manual attention with
  torch.nn.functional.scaled_dot_product_attention (SDPA) which uses
  FlashAttention-2 on A100/H100"]
Expected gain: [quantified with reasoning, e.g. "FA2 typically 2-3x faster
  for attention with seq_len>512. With attention at 60% of forward, expect
  30-40% forward speedup, ~20% overall"]
Precision impact: [e.g. "Numerically equivalent for fp16/bf16. For fp32,
  max_abs_diff typically <1e-6"]
Risk: [e.g. "Requires PyTorch >=2.0. Custom attention masks may need adaptation"]
```

## Validation Scope Confirmation

For each optimization point, confirm:

| Layer | Default | What to confirm |
|-------|---------|----------------|
| L0: Static checks | Always enabled | Any project-specific checks? Custom operators? |
| L1: Precision validation | Always enabled | Tolerance thresholds? (default: max_diff<1e-5, cosine>0.999) |
| L2: Performance benchmark | Always enabled | Target metric and threshold? (e.g. "throughput +20%") |

Record these decisions — they become the validation criteria in the plan.

## Presenting the Strategy

- Scale each section to its complexity
- Ask after each section whether it looks right
- Cover: bottleneck analysis, proposed optimizations (ranked by expected gain), validation criteria, risks
- Be ready to go back and clarify

## After Approval

**Write design doc:**
- Save to `docs/plans/YYYY-MM-DD-<topic>-design.md`
- Include: bottleneck analysis, optimization hypotheses, validation scope, constraints
- Commit to git

**Transition:**
- Invoke `sp-opt:opt-planning` to create the implementation plan
- Do NOT invoke any other skill
