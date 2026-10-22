# Resident kernel tuning, hybrid P2P and four-Spark comparison

Measured October 22, 2026 on **4× CMP 170HX**, TP4, dual-socket Broadwell,
PCIe Gen2 ×16, no NVLink, 140 W/card. Actual HBM PLL was verified at 1728 MHz
on all four cards; the clock reported by `nvidia-smi` remained 1458 MHz.
Model: **DeepSeek V4.1 Flash 1017**, LibertAI EXL3 2.9 bpw main routed experts,
official remaining components, DSpark depth 5. No weights or quant changed.

## Headlines

- **EXL transpose: +2.86% paired C1 counting decode** in the four-round sweep.
- **Six-row host all-reduce: +1.74% further** in that sweep; a separate eight-pair
  confirmation measured **184.21 → 188.13 tok/s (+2.21% geometric mean)**.
- **Hybrid P2P: +2.43% paired C8 counting aggregate**, 572.60 → 586.31 tok/s.
  All four pairs matched recorded output/cache/speculative work and were quiet.
- **No broad throughput improvement established.** The complete eight-task
  Spark protocol reached 271.57 C8 aggregate tok/s, versus the previous 273.48.
- **No reload during the campaigns.** Compatible executable-node updates kept
  weights, Engram and the captured graph generation resident.

The small kernel gains are workload-specific. Do not sum gains from different
campaigns or apply the counting gain to prose, tools or prefill. Median paired
percentages need not equal the ratio of displayed medians. Summary parity does
not prove identical ordered GPU work or model-quality equivalence.

Source remains in the primary repository: [EXL transpose](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/tree/2b15833e00c59ecb06692ca8dee64cd1cabc18da/experiments/exl3-cooperative-decode),
[new six-row host tuning](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/tree/2b15833e00c59ecb06692ca8dee64cd1cabc18da/experiments/host-allreduce-six-row),
[hybrid P2P design](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/blob/2b15833e00c59ecb06692ca8dee64cd1cabc18da/docs/hybrid-p2p-broadwell.md), and
[resident CUDA graph technique](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/blob/2b15833e00c59ecb06692ca8dee64cd1cabc18da/docs/resident-cuda-graph-optimization.md).

## Controlled short sweep: single-stream decode

Four balanced crossover rounds at C1/2/4/8; three historical short tasks:
counting, Python code and prose. Temperature 0, top_p 1, thinking off,
400-token cap with natural EOS. 192 measured batches plus 12 warmups,
732 requests. Per-stream decode = (completion tokens − 1) / time from
first content to response close. Aggregate includes TTFT and full batch wall.

A = controlled EXL baseline + stock host collective; B = transpose + stock;
C = transpose + six-row tuning; D = C + 48-row hybrid P2P. The baseline was
rebuilt with CUDA 12.8 and is not attested identical to the historical CUDA 13
stock binary. All arms used the same resident workers and graph generation.

| Task | A baseline | B transpose | C six-row | D hybrid | Interpretation |
|---|---:|---:|---:|---:|---|
| Counting / structured | 178.40 | 183.42 | 187.05 | 187.15 | 4/4 matched work pairs; no P2P calls at C1 |
| Code | 134.05 | 137.82 | 141.02 | 144.23 | Output/speculative work differs; descriptive |
| Prose | 94.16 | 95.84 | 101.56 | 101.16 | Output/speculative work differs; descriptive |

Counting paired A→B is +2.86%; B→C is +1.74%. A→D is +4.90% on this
controlled counting workload; D executes no P2P calls at C1. Code and prose
rows must not be advertised as isolated kernel gains.

## Controlled short sweep: C8 aggregate

| Task | A baseline | B transpose | C six-row | D hybrid | C→D paired | Matched work |
|---|---:|---:|---:|---:|---:|---|
| Counting / structured | 571.82 | 573.08 | 572.60 | 586.31 | +2.43% | 4/4 |
| Code | 317.02 | 327.70 | 325.32 | 332.62 | +2.25% | 0/4 |
| Prose | 264.44 | 265.08 | 266.31 | 266.63 | +0.33% | 0/4 |

The 48-row hybrid path combines socket-local peer communication with host
communication across sockets. Large-prefill NCCL stays unchanged. Broader
hierarchical paths tested worse; P2P is not recommended as a blanket switch
for Broadwell. Keep it opt-in and qualify the actual topology and message size.

## Context sweep through 128K

C versus D, exact input token IDs, fixed 128 output tokens, cold then immediate
warm. Three completed pairs per arm/cache condition through 64K; one at 128K.
The campaign intentionally stopped further context repetitions before running
the Spark suite. Two unpaired requests and one interrupted request are excluded.
Cold cached tokens were zero; warm cache hits were input length minus 128.
Input rate = prompt tokens / HTTP TTFT, not isolated GPU prefill throughput.

| Prompt tokens | Host-only TTFT s | Hybrid TTFT s | Host-only input tok/s | Hybrid input tok/s |
|---:|---:|---:|---:|---:|
| 2,048 | 1.309 | 1.308 | 1564.8 | 1565.4 |
| 8,192 | 5.304 | 5.304 | 1544.4 | 1544.5 |
| 32,768 | 21.287 | 21.280 | 1539.3 | 1539.9 |
| 65,536 | 44.601 | 44.543 | 1469.4 | 1471.3 |
| 131,072 | 98.623 | 98.714 | 1329.0 | 1327.8 |

Essentially flat. **Zero pair-P2P calls** occurred in this sweep; it does not
measure a P2P prefill implementation. 48/54 completed context requests passed
the recorded zero-swap/low-background-I/O criterion. Tiny host page-ins were
retained in the other samples. Long-context decode used repetitive filler,
so it is not an agentic prose result.

## Standardized comparison: 4× CMP 170HX versus 4× DGX Spark

Exact pinned `bench/v41bench.py` and `bench/prompts-v1.json`, temperature 0,
thinking off, streaming, server usage token counts, short 150–200-token
outputs. Eight tasks, counting excluded. C1/2/3/4/5/6/8, **three full passes**;
804 requests, all with zero cached tokens. Every stage's native request/output
counters reconciled. The headline averages eight task rates per pass, then
takes the median across passes. These are not the historical three-task rows.

Previous CMP = October 21 transpose + stock collective. Current = transpose
+ six-row tuning + hybrid P2P. The cross-run comparison does not isolate P2P.
The Spark reference is **four** DGX Sparks, TensorFold TP4, LibertAI EXL3
**3.5 bpw all matrices**, both RoCE rails, stock patch0004, one published pass.

| Users | Previous CMP aggregate | Current CMP aggregate | Change | Spark aggregate | Current CMP per-stream decode | Spark per-stream decode |
|---:|---:|---:|---:|---:|---:|---:|
| 1 | 92.85 | 91.44 | -1.52% | 98.55 | 114.70 | 111.98 |
| 2 | 122.60 | 119.36 | -2.64% | 146.46 | 77.96 | 86.36 |
| 4 | 186.69 | 185.57 | -0.60% | 198.01 | 60.41 | 61.13 |
| 8 | 273.48 | 271.57 | -0.70% | 221.66 | 46.04 | 34.64 |

C8 current aggregate range: 268.29–274.87 tok/s; previous: 269.98–278.88.
These are observed ranges, not confidence intervals. Current C8 per-stream
decode improved from 44.83 to 46.04 tok/s, while mean TTFT rose from 0.809
to 0.894 s. C1 TTFT rose from 0.265 to 0.306 s. The startup difference remains
unresolved; it is not established as a P2P effect.

| Task | Previous CMP C1 decode | Current CMP C1 decode | Spark C1 decode | Previous CMP C8 aggregate | Current CMP C8 aggregate | Spark C8 aggregate |
|---|---:|---:|---:|---:|---:|---:|
| Coding | 155.94 | 158.36 | 148.74 | 422.81 | 414.06 | 287.11 |
| JSON | 121.90 | 125.24 | 117.78 | 263.41 | 260.42 | 197.40 |
| Narrative | 67.57 | 65.27 | 73.93 | 155.02 | 151.42 | 198.20 |
| Prose | 80.14 | 83.64 | 85.89 | 207.35 | 205.65 | 189.76 |
| Math | 137.43 | 141.54 | 132.79 | 354.49 | 362.92 | 295.37 |
| Reasoning | 119.84 | 106.19 | 108.37 | 288.00 | 278.76 | 219.37 |
| Summary | 73.86 | 70.88 | 76.81 | 150.72 | 157.39 | 142.22 |
| Format | 157.23 | 160.37 | 151.57 | 347.85 | 342.15 | 243.86 |

Prefill uses actual prompt tokens / total duration of a one-output-token
request, matching the public script. It differs from the HTTP TTFT metric above.

| Actual prompt tokens | Previous CMP tok/s | Current CMP tok/s | Spark tok/s | Current CMP seconds | Spark seconds |
|---:|---:|---:|---:|---:|---:|
| 2,950 | 1497.4 | 1443.1 | 2143.7 | 2.044 | 1.376 |
| 11,592 | 1548.6 | 1516.8 | 2266.1 | 7.642 | 5.115 |
| 46,810 | 1471.8 | 1463.2 | 2263.0 | 31.991 | 20.685 |
| 93,335 | 1373.1 | 1372.5 | 2212.8 | 68.003 | 42.180 |

Different hardware, quants, engines, generated answers and speculative work
prevent a causal kernel or quality-equivalence comparison. TensorFold posts
DeepSeek-encoded completions; our engine uses native chat. The Spark source
allows other endpoint traffic and does not attest zero cache hits. Our isolated
suite recorded 91,206 pair-P2P calls. The four-Spark result and source are
[pinned here](https://github.com/tonyd2wild/DeepSeek-V4.1-Flash-vLLM-DGX-Spark/blob/9fa230d3eb119f588e690bf9b87ce41ed902db60/runs/2026-10-17-1017-tensorfold/results/tf1017-stock-c1c32/bench-tf1017-stock-c1c32.json).

## Resident P2P-off/on follow-up

Same Spark coding/JSON/prose prompts at C1 and C8. Two opposite-order rounds
per cell, stock → six-row → hybrid, then reversed. 171 requests including
warmup; no reload or recapture; zero cached tokens; 16,929 real pair calls.

| C8 task | Transpose + stock | Six-row / P2P off | Hybrid / P2P on |
|---|---:|---:|---:|
| Coding | 413.10 | 418.76 | 420.50 |
| JSON | 256.58 | 241.64 | 257.65 |
| Prose | 203.75 | 206.32 | 217.61 |

No consistent P2P regression appeared, but **no complete C8 pair matched all
generated output/speculative work**. The apparent JSON/prose gains are not
established kernel gains. Only two rounds were run; 44/45 batches passed the
strict quietness check. C1 again executed zero P2P calls.

## Evidence and attribution

[results.json](results.json) contains the sanitized numerical summaries,
per-run values, paired deltas and parity counts. [provenance.json](provenance.json)
records source/evidence hashes without private infrastructure identifiers.
Raw control logs and full request captures are retained privately rather than
published. Offline audits independently recomputed results from those captures.

EXL cooperative kernels/format: Turboderp. Host all-reduce and serving work:
Morrowmake and vLLM contributors. SM80 attention: PixelML. Model/quant:
DeepSeek and LibertAI. Spark reference and benchmark: tonyd2wild and
TensorFold contributors. Experiment/measurement work: project maintainers,
with AI assistance. Upstream source licenses remain applicable.
