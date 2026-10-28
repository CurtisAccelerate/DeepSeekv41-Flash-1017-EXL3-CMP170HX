# Expert gates for 2–6 users — October 28, 2026

Same DeepSeek V4.1 Flash 1017 weights and the same 4× CMP 170HX host as the
[October 26 report](../2026-10-26/README.md). Power cap 140 W. Draft length
stays 5. Temperature 0, thinking off. Decode is
`(completion_tokens - 1) / (end - first_token)`, median of 3 runs, 400 output
tokens.

The source is
[experiments/ds1017-decode-gates](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/blob/main/experiments/ds1017-decode-gates/README.md).
Cloning either repository does not load it into a server.

The change is a rebuilt expert gate, up, and down kernel for the decode row
counts that 2–6 users actually hit: 8, 12, 18, 24, 30, and 36 rows. One user
is 6 rows. The new kernel matches the previous expert kernel bit for bit, so
draft acceptance does not move and the token rate moves with the step rate.

One user stays on the October 26 mixed-attention kernel. A fused copy of the
new body inside that kernel was flat on counting, 0.8% slower on code, and
4.2% slower on prose, so it is not what that graph runs.

Seven and eight users do not enter this kernel. Seven is padded onto the
eight-user graph. That graph is 48 rows, and 48-row experts take the
tensor-core path. The small differences at 7 and 8 users are run noise.

## Takeaways

1. **2–6 users are faster.** Counting decode is +9% to +14%. Code is +6% to
   +14%. Prose is +4% to +15%.
2. **One user stays on the previous attention kernel.** The fused trial was
   not faster.
3. **Seven and eight users are unchanged.** They are the tensor-core path.

Serving choice: previous attention at 1 user, the new gates at 2–6 users,
tensor-core at 7–8. The October 26 attention and drafter rules are otherwise
unchanged.

## Morrow decode, tokens per second

Prior is the previous expert kernel. New is the rebuilt kernel. The fused
1-user trial is shown so the choice is visible. It is not the serving path.

| Users | Count prior | Count new | Count | Code prior | Code new | Code | Prose prior | Prose new | Prose |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | 205.7 | 206.0 | +0.1% | 150.5 | 149.3 | −0.8% | 112.1 | 107.4 | −4.2% |
| 2 | 140.8 | 153.9 | +9.3% | 94.7 | 100.4 | +6.0% | 74.4 | 77.5 | +4.2% |
| 3 | 124.0 | 138.7 | +11.9% | 85.5 | 97.8 | +14.4% | 59.4 | 62.8 | +5.8% |
| 4 | 108.4 | 120.2 | +10.9% | 75.8 | 84.3 | +11.2% | 49.2 | 54.2 | +10.3% |
| 5 | 95.9 | 108.2 | +12.9% | 70.7 | 76.8 | +8.6% | 42.7 | 48.9 | +14.5% |
| 6 | 86.8 | 98.5 | +13.5% | 63.4 | 70.6 | +11.3% | 38.9 | 44.8 | +15.0% |
| 7 | 73.5 | 74.0 | +0.7% | 54.1 | 54.5 | +0.8% | 34.7 | 34.5 | −0.7% |
| 8 | 81.7 | 81.6 | −0.2% | 59.2 | 60.4 | +2.1% | 36.8 | 36.6 | −0.5% |

What is serving, decode tokens per second:

| Users | Path | Count | Code | Prose |
|---:|---|---:|---:|---:|
| 1 | Previous attention | 205.7 | 150.5 | 112.1 |
| 2 | New gates | 153.9 | 100.4 | 77.5 |
| 3 | New gates | 138.7 | 97.8 | 62.8 |
| 4 | New gates | 120.2 | 84.3 | 54.2 |
| 5 | New gates | 108.2 | 76.8 | 48.9 |
| 6 | New gates | 98.5 | 70.6 | 44.8 |
| 7 | Tensor core | 74.0 | 54.5 | 34.5 |
| 8 | Tensor core | 81.6 | 60.4 | 36.6 |

## Spark prompt suite

These rows are not a DGX Spark. The prompt text and timing rules are the
pinned bench used on October 26, run on this 4× CMP 170HX. Median of 3
passes. Aggregate includes time to first token. The ceiling category is
excluded. There is no 7-user row.

The 1-user pass was measured on the fused trial, which is not the serving
path. Users 2–8 match what is serving.

| Users | Aggregate | Per stream |
|---:|---:|---:|
| 1 | 96.2 | 120.8 |
| 2 | 154.4 | 96.9 |
| 3 | 197.0 | 81.3 |
| 4 | 209.3 | 66.1 |
| 5 | 231.2 | 59.7 |
| 6 | 243.8 | 53.6 |
| 8 | 275.7 | 45.3 |

October 26 is the new-attention suite from that report. October 28 is the
table above. Median of 3 passes, same headline rule. The runs are two days
apart.

| Users | Aggregate Oct 26 | Aggregate Oct 28 | Aggregate | Per stream Oct 26 | Per stream Oct 28 | Per stream |
|---:|---:|---:|---:|---:|---:|---:|
| 1 | 99.0 | 96.2 | −2.9% | 124.3 | 120.8 | −2.8% |
| 2 | 122.6 | 154.4 | +25.9% | 78.6 | 96.9 | +23.3% |
| 3 | 157.5 | 197.0 | +25.1% | 67.9 | 81.3 | +19.8% |
| 4 | 185.4 | 209.3 | +12.9% | 59.9 | 66.1 | +10.5% |
| 5 | 207.2 | 231.2 | +11.6% | 53.3 | 59.7 | +11.9% |
| 6 | 220.5 | 243.8 | +10.6% | 48.1 | 53.6 | +11.4% |
| 8 | 270.0 | 275.7 | +2.1% | 45.0 | 45.3 | +0.6% |

The October 22 published CMP suite, a different stack and not a same-hour
rerun, was 91.4 aggregate at 1 user, 185.6 at 4, and 271.6 at 8. Published
per-stream was 114.7 at 1 user and 46.0 at 8.

## Prefill

Prefill was not the change. Cold prefill at 93,335 prompt tokens was 1,376
tokens per second, median of three passes. The first cold pass was 1,415 at
2,950 prompt tokens, 1,530 at 11,592, and 1,472 at 46,810. Later passes of
the shorter prompts hit cache and are not a prefill rate.
