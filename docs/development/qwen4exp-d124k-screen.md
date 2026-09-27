# Qwen4Exp depth-0 and depth-124000 kernel experiments

These experiments were measured on a Strix Halo gfx1151 (40 CU), using Qwen3.8-Flash-Next UD-Q4_K_XL. The original measurement revision was `c717021306e5e4e2c6df33be6bd505a1ea983f70`. The published experiment branches are replayed onto `exp/qwen4exp-optimizations-clean`, which is based on the current `master` and omits the local copies of the separate correctness-fix MRs. Measurements below belong to the original revision; do not treat the rebased branches as newly benchmarked.

All prefill runs used `-p 4096 -n 0 -b 4096 -ub 4096 -ngl 99 -fa on -ctk f16 -ctv f16 -lm dio -lzm on`, with `LLAMA_PLE_PRELOAD` unset. A depth-124000 trace includes the depth-fill work; only the final measured 4096-token pass is included in the kernel times below. Single profiler-run throughput is not an alternating end-to-end A/B result.

| Branch suffix | Workload | Changed kernel, stock -> experiment (ms/pass) | Correctness / decision |
| --- | --- | ---: | --- |
| `hc-stage-master` | depth 0 | HC gate-mix 101.3 -> 106.35 | VGPR 224 and scratch 144 B/thread unchanged; reject |
| `hc-cache-master` | depth 0 | HC combine+inject 152.35 -> 222.67 | Extra LDS and VGPR cost; reject |
| `qsa2-master` | depth 124000 hypothesis | Not timed after model correctness failure | 32/32 tolerant QSA tests passed, but all 248320 final logits differed (max abs 0.917); reject |
| `qsa-exact-master` | depth 124000 | QSA attention 332.78 -> 422.90 | 32/32 QSA tests and byte-identical complete final logits; reject |
| `qsa-maskskip-master` | depth 124000 | QSA attention 332.78 -> 325.48 | 32/32 QSA tests and byte-identical complete final logits; ~0.3% whole-pass ceiling, below the 1% acceptance threshold; reject |

The QSA correctness comparison used a deterministically repeated 5064-token prose prompt, context 8192, batch 8192 and ubatch 4096, with the same production model/KV/offload options. Two stock runs and both exact candidate runs had identical prompt token bytes (SHA-256 `f7bded96b9342841836d5dba3d144791ce27183f8b878870932c93c9b8caa959`) and full final logits (SHA-256 `9df02cdda7c02743378fb7f1ac69954cc92a6d3b3344e327ac8810eae35689ec`). The two-query union candidate had different logits (SHA-256 `94126843a8b138823f92a6c6244d9e400dc56bc2126e64ca72bcfbf8bd745e35`). It changed which selected keys share a 16-key streaming-softmax tile, and therefore changed the arithmetic order.

The complete original traces, counter CSVs and model-logit comparisons remain outside the repository at `strix-002:~/night_backup/d250k-20260927/`. In the stock depth-124000 trace, QSA attention consumed 332.78 ms across 12 launches, while the final 4096-token pass had 2833.57 ms of kernel time. The QSA L2 hit metric fell from 71.43% at depth 0 to 15.54% at depth 124000. ROCm `FETCH_SIZE` reported 11.21 GB and 119.65 GB for the respective 12-launch passes; do not treat these derived APU counter values as independently verified DRAM bandwidth.

None of the branches above is a validated speedup or intended to merge as-is. Production source and binaries were not replaced by these experiments.
