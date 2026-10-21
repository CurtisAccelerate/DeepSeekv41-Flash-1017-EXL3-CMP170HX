# 4× CMP 170HX versus 4× DGX Spark — October 21, 2026

The 4× CMP 170HX results are medians of three complete passes; brackets show the observed minimum and maximum, not confidence intervals. Each Spark column is a separate published run on four DGX Sparks. The later TensorFold run uses both network rails and patch0004.

The exact pinned `v41bench.py` prompts and timing formulas were used against native chat, with temperature 0 and thinking off. The run completed **804 requests**: 783 measured short requests, nine warmups and twelve prefill requests. A distinct native `cache_salt` per pass preserved prompt text while preventing cross-pass reuse; every request reported zero cached prompt tokens. Actual prompt token counts match the published comparator. Token counts come from server usage, not stream chunks.

The measured candidate uses the six-row EXL3 kernel where eligible; other shapes fall back. The controlled baseline was restored after benchmarking. This is a system comparison, not a measurement of the kernel change alone.

## Throughput by concurrency

Aggregate is all completion tokens divided by full batch wall time, including first-token latency and executor overhead. The headline is the unweighted mean of the eight category rates; the counting ceiling is excluded. Each reported median is taken across the three complete pass headlines.

### Aggregate tok/s

| Users | 4× CMP 170HX median [range] | 4× Spark native vLLM | 4× Spark TensorFold earlier two-rail | 4× Spark TensorFold later two-rail |
|---|---|---|---|---|
| 1 | 92.85 [88.11–93.28] | 48.48 | 93.57 | 98.55 |
| 2 | 122.60 [117.05–124.51] | 76.57 | 136.85 | 146.46 |
| 3 | 154.61 [141.83–158.61] | 102.58 | 167.33 | — |
| 4 | 186.69 [184.87–187.49] | 123.19 | 189.31 | 198.01 |
| 5 | 206.30 [202.23–209.76] | 144.61 | 202.12 | — |
| 6 | 219.07 [219.07–223.56] | 165.72 | 213.81 | — |
| 8 | 273.48 [269.98–278.88] | — | — | 221.66 |

### Per-stream decode tok/s

Per-request decode = (completion tokens − 1) / (response close after DONE − first token); category rates are means across streams.

| Users | 4× CMP 170HX median [range] | 4× Spark native vLLM | 4× Spark TensorFold earlier two-rail | 4× Spark TensorFold later two-rail |
|---|---|---|---|---|
| 1 | 113.29 [112.75–115.08] | 53.83 | 107.09 | 111.98 |
| 2 | 77.58 [76.71–78.96] | 44.89 | 81.26 | 86.36 |
| 3 | 66.07 [64.09–67.88] | 39.74 | 67.71 | — |
| 4 | 59.73 [59.49–60.51] | 36.55 | 58.47 | 61.13 |
| 5 | 52.40 [52.10–53.13] | 34.45 | 50.30 | — |
| 6 | 47.11 [47.01–47.58] | 32.28 | 44.60 | — |
| 8 | 44.83 [44.28–45.52] | — | — | 34.64 |

Against the later TensorFold run, C8 aggregate is **273.48 versus 221.66 tok/s (+23.4%)**. C1 aggregate is **92.85 versus 98.55 tok/s (−5.8%)**, while C1 decode is **113.29 versus 111.98 tok/s (+1.2%)**. Mean first-token latency at C1 is 265 ms versus 176 ms; at C8 it is 809 ms versus 807 ms.

The older one-rail TensorFold C8 result was 212.75 aggregate tok/s. It is not appended to either two-rail curve. No C8 native-vLLM result is available in the pinned runs.

## Per-category decode

These compare the median category rate with the later TensorFold two-rail run. All rates are tok/s; counting is excluded.

### 1 user

| Category | 4× CMP 170HX median [range] | 4× Spark TensorFold |
|---|---|---|
| coding | 155.94 [154.42–158.53] | 148.74 |
| json | 121.90 [121.82–122.77] | 117.78 |
| narrative | 67.57 [64.07–68.18] | 73.93 |
| prose | 80.14 [78.07–83.98] | 85.89 |
| math | 137.43 [134.54–141.22] | 132.79 |
| reasoning | 119.84 [106.90–121.03] | 108.37 |
| summary | 73.86 [70.78–74.01] | 76.81 |
| format | 157.23 [156.80–157.99] | 151.57 |

### 8 users

| Category | 4× CMP 170HX median [range] | 4× Spark TensorFold |
|---|---|---|
| coding | 64.99 [63.16–67.75] | 41.65 |
| json | 48.15 [47.56–48.47] | 34.61 |
| narrative | 23.02 [22.42–23.78] | 29.06 |
| prose | 31.52 [31.47–32.33] | 27.19 |
| math | 56.39 [55.57–56.48] | 44.24 |
| reasoning | 43.34 [41.29–43.48] | 31.37 |
| summary | 25.45 [25.42–26.97] | 23.84 |
| format | 65.81 [65.37–66.80] | 45.16 |

## Cold prefill

This metric is actual prompt tokens divided by total request duration for a one-token reply. It includes frontend/scheduling/first-token work; the upstream JSON names that total duration `ttft_s`.

| Actual prompt tokens | 4× CMP 170HX seconds | 4× CMP 170HX tok/s median [range] | 4× Spark seconds | 4× Spark tok/s |
|---|---|---|---|---|
| 2950 | 1.970 | 1497.40 [1496.00–1502.60] | 1.376 | 2143.7 |
| 11592 | 7.485 | 1548.60 [1545.90–1550.30] | 5.115 | 2266.1 |
| 46810 | 31.804 | 1471.80 [1470.50–1472.60] | 20.685 | 2263.0 |
| 93335 | 67.975 | 1373.10 [1373.00–1374.10] | 42.180 | 2212.8 |

At 93,335 input tokens, the later Spark run completes in 42.180 seconds versus 67.975 seconds: about 1.61× faster for this cold-prefill request.

## Scope and reproducibility

- 4× CMP 170HX: 140 W/card, SM80, TP4, LibertAI 2.9 bpw main routed experts with native remaining components, DSpark depth 5 and a 262,144 context cap. TensorFold uses a 3.5 bpw all-matrix recipe and a different engine; native Spark uses the release FP8/MXFP4 mix. Quality equivalence was not measured.
- Natural output lengths and speculative acceptance vary. Matching prompts and input counts do not establish identical generated work.
- Spark numbers are individual published batches. The later TensorFold notes allow other agents on the endpoint; their background load and cache hits were not attested. No measured Spark power receipt was available, so no performance-per-watt claim is made.
- Native `v41bench.py` uses chat with `thinking: false`. The TensorFold variant sends DeepSeek-encoded text through completions. The benchmark source and prompt JSON were unchanged; only private admission metadata and per-pass cache salt were added to the local requests.
- All 804 local requests passed stream/usage/cache checks and exact request/token counter reconciliation. Numerical formulas and public comparisons were independently recalculated. Raw streams and operational receipts are retained privately; [compact results](results.json) are published here.

## Pinned primary sources

- [Benchmark source](https://github.com/tonyd2wild/DeepSeek-V4.1-Flash-vLLM-DGX-Spark/blob/9fa230d3eb119f588e690bf9b87ce41ed902db60/bench/v41bench.py)
- [Prompt set](https://github.com/tonyd2wild/DeepSeek-V4.1-Flash-vLLM-DGX-Spark/blob/9fa230d3eb119f588e690bf9b87ce41ed902db60/bench/prompts-v1.json)
- [Native vLLM results](https://github.com/tonyd2wild/DeepSeek-V4.1-Flash-vLLM-DGX-Spark/blob/9fa230d3eb119f588e690bf9b87ce41ed902db60/runs/2026-10-17-1017/results/boot1017g/bench-boot1017g.json)
- [Earlier TensorFold two-rail results](https://github.com/tonyd2wild/DeepSeek-V4.1-Flash-vLLM-DGX-Spark/blob/9fa230d3eb119f588e690bf9b87ce41ed902db60/runs/2026-10-17-1017-tensorfold/results/tf1017b-rail2/bench-tf1017b-rail2.json)
- [Later TensorFold two-rail results](https://github.com/tonyd2wild/DeepSeek-V4.1-Flash-vLLM-DGX-Spark/blob/9fa230d3eb119f588e690bf9b87ce41ed902db60/runs/2026-10-17-1017-tensorfold/results/tf1017-stock-c1c32/bench-tf1017-stock-c1c32.json)
