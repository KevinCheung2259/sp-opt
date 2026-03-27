---
name: opt-static-checks
description: Use when reviewing optimization code for static correctness — verifies the optimization doesn't introduce dtype mismatches, shape changes, semantic differences, or numerical instability. Dispatched as L0 in the three-layer validation.
---

# L0: Optimization Static Checks

Specialized static-analysis subagent checking that an optimization preserves correctness. Catches equivalence violations, numerical instability introductions, and API breakage before any code is executed.

**This is a RIGID skill.** Follow exactly. Do not skip checks. Do not adapt away discipline.

## When to Use

- Called by sp-opt:opt-subagent-dev after implementation, before precision-validator
- Can also be invoked standalone to review an optimization change

## Input

The subagent receives:
- The optimization point description (from the plan)
- The git diff of changes
- The original code context

## Checklist

Refer to `checklist.md` for the full checklist. Summary:

| # | Check | Level | What to verify |
|---|-------|-------|----------------|
| 1 | dtype consistency | Mandatory | Input/output dtype unchanged |
| 2 | shape consistency | Mandatory | Tensor shapes unchanged |
| 3 | operator semantic equivalence | Mandatory | Replaced operators have same semantics |
| 4 | control flow preserved | Mandatory | No deleted branches or exception handling |
| 5 | side effects preserved | Mandatory | In-place ops, global state maintained |
| 6 | numerical stability | Mandatory | No removed clamp/eps/stability guards |
| 7 | memory lifecycle | Advisory | New buffers/caches have correct release |
| 8 | concurrency safety | Advisory | No race conditions in multi-GPU/thread |
| 9 | fallback path | Advisory | Hardware fallback when optimization unavailable |
| 10 | API compatibility | Advisory | Function signatures, return values unchanged |

## Severity Tiers

- **Mandatory checks (1-6):** Critical — blocks progress. Must pass before proceeding to L1.
- **Advisory checks (7-10):** Warning — does not block, but reported in the review output.

## Execution

This runs as a **code review subagent** — read-only, no code execution.

1. Read the git diff of the optimization changes
2. Read surrounding context (original functions, callers, tests)
3. Walk through each checklist item
4. For each mandatory check: PASS or FAIL with specific evidence (file, line, reason)
5. For each advisory check: OK or WARNING with explanation

## Output Format

```
## L0 Static Check Results

### Mandatory Checks
| # | Check | Result | Evidence |
|---|-------|--------|----------|
| 1 | dtype consistency | PASS/FAIL | [specific finding] |
| ... | ... | ... | ... |

### Advisory Checks
| # | Check | Result | Evidence |
|---|-------|--------|----------|
| 7 | memory lifecycle | OK/WARNING | [specific finding] |
| ... | ... | ... | ... |

### Verdict: PASS / FAIL
[If FAIL: list which mandatory checks failed and what needs to be fixed]
```

## Fix Loop

```
Mandatory check fails → Implementer fixes → re-run L0 → pass → proceed to L1
    → 5 failures on same check → stop, report to user
```
