# Controlled kernel sweeps — October 21, 2026

4× CMP 170HX, 140 W/card, TP4, DeepSeek V4.1 Flash 1017 with LibertAI
2.9 bpw main routed experts and native remaining components; DSpark depth 5,
262,144 context cap. Baseline and candidate are existing boot kernels in the
same resident owner/workers. The candidate serves only eligible six-row
SM80/K2–K3 calls; other shapes use the original extension. Actual concurrent
candidate coverage was not instrumented. Public baseline serving was restored.

The baseline is a controlled CUDA 12.8 rebuild, not proven identical to the
historical CUDA 13 stock extension. These are current paired measurements;
the README's October 17 numbers remain historical. Three repetitions do not
establish workload-wide statistical confidence.

## Short concurrency sweep

276 completed requests, including 6 warmups; 270 measured requests in 72 batches.
Every cell is the median of 3 paired batches, same three public historical
structured/code/prose prompts, temperature 0, explicit `top_p=1`, thinking off,
400-token cap. Aggregate counts every output token over full batch wall time,
including TTFT, executor overhead and DONE/stream-close. It is not per-user
decode speed. Spark-format per-request decode instead uses DONE/stream-close
after the first token; the context sweep below ends at last nonempty content.

| Workload | Users | Baseline aggregate tok/s | Candidate aggregate tok/s | Change |
|---|---:|---:|---:|---:|
| structured | 1 | 161.17 | 164.56 | +2.10% |
| structured | 2 | 240.65 | 240.59 | -0.02% |
| structured | 4 | 379.18 | 378.52 | -0.17% |
| structured | 8 | 569.67 | 569.78 | +0.02% |
| code | 1 | 99.46 | 101.48 | +2.04% |
| code | 2 | 129.59 | 130.67 | +0.83% |
| code | 4 | 212.47 | 211.81 | -0.31% |
| code | 8 | 328.19 | 329.52 | +0.41% |
| prose | 1 | 90.43 | 91.91 | +1.64% |
| prose | 2 | 126.06 | 125.39 | -0.53% |
| prose | 4 | 181.93 | 180.15 | -0.97% |
| prose | 8 | 261.26 | 271.85 | +4.05% |

Structured outputs and full speculative summaries matched every pair. Code
text matched every pair, while speculative summaries often differed. Prose
text and speculative work differed in every pair, so its gains are descriptive.
Output token IDs were not requested. C8 structured +0.02% and code +0.41% do not
support a large causal multi-user kernel gain. Historical omitted top_p and
used one batch/cell, so its differences are not isolated kernel effects.

## Context sweep through 128K

60 requests: 3 paired repetitions per arm at each of 5 exact input lengths,
each cold and immediately warm; 128 output tokens/request. Decode rate is
(128−1)/(last nonempty content SSE time−first nonempty content SSE time).
All 30 paired texts matched, as did verify-step and proposed-draft counts;
full speculative summaries matched 24/30. Matching summaries do not establish
identical ordered per-step work or tensor-level equivalence. Output IDs were
not requested because they would echo prompt IDs into the first timed frame.

| Input tokens | Baseline cold decode | Candidate cold decode | Change | Baseline warm decode | Candidate warm decode | Change |
|---:|---:|---:|---:|---:|---:|---:|
| 2,048 | 149.87 | 153.31 | +2.30% | 148.98 | 152.14 | +2.12% |
| 8,192 | 147.84 | 151.21 | +2.28% | 148.04 | 151.06 | +2.04% |
| 32,768 | 148.49 | 151.91 | +2.31% | 147.93 | 151.59 | +2.47% |
| 65,536 | 149.93 | 153.60 | +2.45% | 150.05 | 153.72 | +2.45% |
| 131,072 | 157.49 | 158.38 | +0.56% | 155.92 | 158.86 | +1.89% |

Rates are median tok/s; percentages use unrounded rates. Cold cached-token
counts were zero. Warm cached-token counts were input length minus 128 for
both arms. At 128K, cold TTFT was 98.280→98.313s and warm 0.556→0.549s.
Historical 128K decode was 163.89 tok/s, above both current arms; do not describe
this current comparison as a historical throughput improvement.

The separate Spark-format comparison uses 8 categories with different output
budgets, batch rules and timing. Its 273.48 tok/s C8 headline must not be compared
directly with the structured-only 569.78 tok/s cell above.

## Source identities and attribution

Baseline cubin SHA256: 506f7ccd0108693e8c9f97f866330c976d72fbfca8e093e414f2150962d7560f.
Candidate cubin SHA256: a4c2ed3e83ee8c9025962379fe1782b66c38b340f05cfa986b6d29c5ded63187.
These identify measured artifacts; binaries and quantized weight fixtures are
not distributed here. The source patch is experimental and the public serving
baseline remains selected. No general task-quality or full-model bitwise
equivalence claim follows. Measurements/experiment: CurtisAccelerate. ExLlamaV3 source: Turboderp (MIT).

The [canonical kernel source and qualification notes](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/tree/fb9786ec0ed6a94225ddc4b0aa2976b9f9a8bf69/experiments/exl3-cooperative-decode)
are maintained in the primary kernel repository at `fb9786ec0ed6a94225ddc4b0aa2976b9f9a8bf69`.
This repository retains the model recipe and these benchmark results.
