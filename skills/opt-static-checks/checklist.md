# Optimization Static Analysis Checklist

## Mandatory (Critical) — checks 1-6

| # | Check | What to verify | Common failures |
|---|-------|----------------|-----------------|
| 1 | dtype consistency | Input/output dtype unchanged before/after optimization. Check: function parameters, return values, intermediate tensors. | Implicit cast from fp32 to fp16 in fused kernel; losing double precision in accumulator |
| 2 | shape consistency | All tensor shapes unchanged. Check: output shape, intermediate shapes, batch dimension handling. | Transposed dimensions in optimized attention; squeezed batch dim in fused op |
| 3 | operator semantic equivalence | Replaced operator produces same mathematical result. Check: reduction order, broadcasting semantics, edge cases (empty input, single element). | softmax with different dim argument; mean vs sum in reduction; matmul vs bmm broadcasting difference |
| 4 | control flow preserved | No deleted conditional branches, exception handlers, or early returns. Check: if/else branches, try/except blocks, loop bounds. | Removed gradient clipping branch for "simplicity"; deleted NaN check in fast path |
| 5 | side effects preserved | In-place operations and global state modifications maintained. Check: in-place ops (.add_, .mul_), module state updates, buffer modifications. | Changed in-place to out-of-place breaking caller assumptions; removed running_mean update in fused BN |
| 6 | numerical stability | No removed stability guards. Check: clamp, eps additions, log(x+eps), softmax temperature, gradient scaling. | Removed eps in layer norm denominator; deleted clamp in log computation; removed gradient scaling |

## Advisory (Warning) — checks 7-10

| # | Check | What to verify | Common failures |
|---|-------|----------------|-----------------|
| 7 | memory lifecycle | New buffers/caches have correct allocation and release timing. Check: persistent buffers registered correctly, cache invalidation on shape change, no unbounded growth. | KV cache grows without bound; static buffer not freed on model.eval(); CUDA graph capture leaks replay buffers |
| 8 | concurrency safety | No race conditions in multi-GPU or multi-thread scenarios. Check: shared state access, NCCL collective ordering, stream synchronization. | Shared buffer written by multiple streams; all-reduce on different tensors across ranks; missing stream.synchronize() before cross-stream read |
| 9 | fallback path | Graceful degradation when hardware feature unavailable. Check: FlashAttention availability, tensor core support, specific CUDA compute capability. | Hard crash when FA not available; no fallback for SM < 8.0; missing check for bf16 support |
| 10 | API compatibility | External interface unchanged. Check: function signatures, return value types/shapes, keyword arguments, default values. | Changed return from tuple to single tensor; removed keyword argument; changed default value |
