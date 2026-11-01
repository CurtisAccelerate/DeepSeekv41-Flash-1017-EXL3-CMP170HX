# BF16 cache and full prefill fusion — November 1, 2026

This update keeps our optimized serving stack, integrates selected GNmac target
and drafter paths through qualified runtime selections, and completes integer prefill
fusion across **all 40 target layers**. It is experimental and opt-in; the
normal public launcher is unchanged. The source and qualification are pinned
to [this kernel commit](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/tree/036f62398e32689e5cfa8bc6bd0255d602597dac/experiments/ds1017-bf16-full-fusion).

The best-supported common settings are **BF16 KV + full integer fusion + Marlin
dense prefill**. No arm proved best on every speed and quality measure. The original target paths lead
long-history and critical-field likelihood; the alternate target-only path
effectively ties them overall. The target-plus-drafter update is the faster
measured complete-stack option and leads short-history likelihood.
No extra dense/cuBLAS speed-only override is justified by the current evidence.

## Labels and what was actually tested

| Label | Meaning |
|---|---|
| A | Original optimized decode/target paths |
| B | Selected GN-derived target kernels, retaining our eligible expert and attention routes |
| C | B target plus FP8 draft head and fused Markov selection, with proposal/RNG guards |
| REFERENCE | Original prefill |
| MOE_TRY | Qualified MoE-only prefill, the isolated baseline |
| MOE_RAGGED | Full integer combine/pack + MoE prefill, with Marlin dense prefill |

All new comparisons below use **bfloat16 KV** on 4× CMP 170HX at **140 W**,
TP4, 262,144 context, eight maximum sequences, 2,048 batched tokens, DSpark
depth five. BF16 cache is not a full-BF16 model: the existing expert pack and
candidate short-row target math remain. The vocabulary head stays BF16.

The 30-request prefill sweep and 72-request prefill likelihood screen hold
C target/BF16 KV fixed. Full SPARK compares A+REFERENCE with C+MOE_RAGGED.
The 96-request A/B/C/A likelihood screen holds MOE_RAGGED/BF16 KV fixed.

## Prefill wall time versus previous baselines

Seconds, lower is better. Three paired frozen inputs per context. V5 stages
are previous boots; only the final two V6 columns are the same-boot ablation.

| Context | V5 partial fusion | V5 fusion + cuBLAS | V6 MoE baseline | V6 full fusion | V6 matched wall cut | Historical GNmac |
| --- | --- | --- | --- | --- | --- | --- |
| 2K | 1.234 | 1.210 | 1.279 | 1.194 | 6.72% | 1.160 |
| 8K | 4.818 | 4.719 | 4.926 | 4.671 | 5.17% | 4.433 |
| 32K | 19.933 | 19.552 | 20.394 | 19.280 | 5.46% | 18.130 |
| 64K | 41.415 | 40.631 | 43.001 | 40.297 | 6.29% | 38.486 |
| 128K | 89.700 | 88.234 | 91.585 | 87.615 | 4.34% | 86.333 |

The measured new kernel gain is **4.3–6.7% beyond MoE-only prefill**. At 128K,
91.585 → 87.615 seconds, about 1,496 actual input tokens/s. The older original
optimized BF16 C boot measured about 95.36s at 128K; that is additional
cross-boot context, not an isolated kernel comparison.

Each arm used the exact frozen body and actual prompt-token count, reset prefix
caching and a unique salt; actual cached tokens were zero. Wall is send-to-close;
cap one makes first-visible TTFT almost equal to wall. The CSV retains exact
TTFT and rate values. Cold means prefix-cache zero, not a flushed SSD/page cache.

The GNmac column is **our historical measurement of GNmac's code on this 140 W
machine**, not GNmac's published 180 W result. It used FP8 KV, slightly different
calibrated prompts and null cached-token fields. V6 remains about 1.5% slower
there at 128K. This is not a fully matched causal comparison and does not isolate
KV precision or prove one remaining kernel is responsible.

## Full matched SPARK temperature 1

All 1,680 requests passed independent source/body/content/path/cache checks:
three interleaved passes per arm, original eight categories plus an excluded
ceiling diagnostic, users 1/2/3/4/5/6/8, original filler prefill and seeded controls.
Full-wall tokens/s are higher-is-better; TTFT is seconds, lower-is-better.

| Users | Previous C, BF16 KV | Current A | Current C | Current C/A change | A TTFT | C TTFT |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 111.28 | 88.66 | 112.02 | 26.35% | 0.319 | 0.225 |
| 2 | 134.49 | 118.46 | 133.09 | 12.35% | 0.499 | 0.351 |
| 3 | 172.91 | 154.62 | 166.40 | 7.61% | 0.575 | 0.412 |
| 4 | 197.28 | 182.96 | 196.00 | 7.13% | 0.634 | 0.451 |
| 5 | 219.38 | 203.98 | 214.34 | 5.08% | 0.681 | 0.486 |
| 6 | 235.21 | 214.60 | 232.99 | 8.57% | 0.824 | 0.538 |
| 8 | 280.25 | 245.44 | 281.48 | 14.68% | 0.916 | 0.611 |

The headline is the median of three pass-means of eight category full-wall token
rates, not total suite tokens divided by total suite duration. Streaming rates,
individual category walls, caps and seeded controls are supplied separately.

**Do not call the 26% one-user A/C gap a new V6 kernel gain.** Current C's
absolute rates are close to prior C (112.02 vs 111.28 at one user; 281.48 vs
280.25 at eight), while current A is slower than prior A (88.66 vs 100.90;
245.44 vs 261.50). Intermediate-user C rates also regress against prior C.
Reference/output/order drift is not isolated. The C-fixed prefill ablation
above is the direct evidence for the integer upgrade.

Primary requests are unseeded T1 with thinking disabled. Proposals sample at
actual active batch one and stay greedy above one; seed1017 one-user controls
retain greedy proposals. Sampled output lengths differ. Rates do not establish
task success or equal-work completion latency. Changing the seed also changes
target draws, so the seeded controls are not a proposal-only ablation.

The original SPARK filler targets labelled 2K/8K/32K/64K actually contain about
2.95K/11.59K/46.81K/93.34K tokens. They are distinct from the calibrated
Morrow 2K–128K prefill series.

## Perplexity and winner changes

These are **small synthetic expected-token likelihood screens**, not standard
corpus perplexity, HumanEval or full agent investigations. Each run scores 138
gold tokens across 0.4K/7.3K histories. Lower NLL/PPL is better.

Current A/B/C/A, all with full fusion and BF16 KV; A pools two reference passes:

| Arm | Overall NLL | Overall PPL | Long PPL | Short PPL | Critical-field NLL |
| --- | --- | --- | --- | --- | --- |
| A | 0.154141295 | 1.166655718 | 1.161538753 | 1.171795224 | 0.084583987 |
| B | 0.153980974 | 1.166468693 | 1.163144111 | 1.169802778 | 0.087384970 |
| C | 0.154837152 | 1.167467826 | 1.166629829 | 1.168306425 | 0.088415146 |

- A leads long-history and critical-field loss. C leads short-history loss.
- B's overall advantage over A is only about 0.016% PPL; its NLL difference is
  smaller than the repeat-A spread of 0.001129081 nats/token.
- C is about 0.070% higher overall PPL and 0.438% higher long-history PPL than
  pooled A. Those are likelihood changes, **not agent failure percentages**.
- B changed no eligible decoder winner. C and repeat A changed the exact same
  JSON-punctuation position away from the expected token: no adverse winner
  change unique to C. All 48 gold/wrong contrasts still preferred gold.

The separate C-fixed prefill experiment scored PPL **1.169096 → 1.165605**
(-0.299% overall; -0.688% long; +0.093% short). Critical-field NLL was slightly
higher. Its one toward-gold winner change also occurred in repeat reference.
All 36 gold/wrong contrasts preferred gold. The prior V5 dense/cuBLAS experiment
instead scored +0.723% overall / +1.854% long PPL against its own reference.
These are separate experiments; their PPL deltas must not be added together.

Teacher forcing bypasses proposal generation and the greedy-head shortcut.
It does **not** evaluate the drafter itself, a sampled gate's correction
distribution or 128K/whole-agent quality. B/C target-family sharing also means
small score differences cannot automatically be assigned to C's draft head.

## Morrow benchmark headline

Original means our optimized target/decode paths; updated means the selected
GNmac-derived short-row target and draft-head/selection paths integrated with
our retained expert, attention and collective optimizations. Both use BF16 KV
on the same loaded model at 140 W. These figures compare the complete selected
stacks, not the prefill fusion in isolation.

Morrow prompts, temperature 0, thinking off, output cap 400; one measured batch
per cell on this load, at one and eight concurrent users. Decode tok/s is per
user after the first token, `(completion_tokens - 1) / (end - first_token)`;
request wall includes the first-token wait. Positive wall reduction means
faster completion; a negative value means a longer wait. This is a limited
control, not a new three-repeat 1–8-user sweep.

| Users | Prompt | Original decode tok/s | Updated decode tok/s | Decode change | Request wall reduction |
| ---: | --- | ---: | ---: | ---: | ---: |
| 1 | Count | 221.93 | 271.77 | +22.46% | +15.39% |
| 1 | Code | 196.99 | 245.96 | +24.86% | +17.84% |
| 1 | Prose | 116.68 | 149.54 | +28.17% | +20.44% |
| 8 | Count | 88.10 | 81.12 | -7.92% | -40.35% |
| 8 | Code | 76.90 | 81.53 | +6.02% | +1.60% |
| 8 | Prose | 39.10 | 43.89 | +12.24% | +11.74% |

The fresh comparison measured one batch per cell at one/eight users, using
the retained `results-decode-controls` rows. It excludes the separate first-use
probe; those raw first-use results remain in
[MORROW-CURRENT-WALLS.csv](MORROW-CURRENT-WALLS.csv).

| Users | Prompt | Original wall, s | Updated wall, s | Original TTFT, s | Updated TTFT, s |
| ---: | --- | ---: | ---: | ---: | ---: |
| 1 | Count | 2.060 | 1.743 | 0.261 | 0.274 |
| 1 | Code | 2.271 | 1.866 | 0.245 | 0.243 |
| 1 | Prose | 3.683 | 2.930 | 0.263 | 0.262 |
| 8 | Count | 5.166 | 7.250 | 0.659 | 2.514 |
| 8 | Code | 6.056 | 5.960 | 0.756 | 0.845 |
| 8 | Prose | 11.187 | 9.873 | 0.682 | 0.682 |

The earlier full 1–8-user, three-repeat results remain in
[MORROW-PREVIOUS-FULL.csv](MORROW-PREVIOUS-FULL.csv); they are from an earlier
boot and are not fresh measurements of this prefill build. Machine-readable
headline values and exact percentage formulas are in
[MORROW-HEADLINE.csv](MORROW-HEADLINE.csv). The original/update mapping is the
original optimized decode versus the target-plus-drafter candidate described
in the technical labels above; no arm is being silently redefined.

### Real latency exceptions

Eight-user counting regressed: **updated 7.250s / TTFT 2.514s versus original
5.166s / 0.659s**. Code/prose at eight users were faster. The first seeded
short coding delivery probe also paused for **0.399s**, despite lower TTFT;
seven other delivery probes had no such gap. These observations are retained
alongside the streaming gains.

Verbose logs show shared Engram/gather/draft compilation in the slow count
window and rejection/recovery compilation during the seeded pause. This supports
a first-use explanation, without quantifying the entire delay or proving Hermes
buffering. Broader representative engine warmup before admission is the next
precision-preserving latency direction; it was not integrated or qualified here.

## Mechanism, qualification and provenance

The previous fusion admitted only the donor-shaped ratio domain, leaving our
18 ratio-2 and 2 sliding-window target layers on the original fallback. V6
directly builds the same int32 ragged indices/row pointers for 20/18/2 layers.
Invalid slots, order, wrapping and physical bounds are preserved; floating
attention is unchanged. All four GPUs showed all 40 target fusion calls.

Per GPU: 85 integer and 9 same-attention bitwise/finite fixtures, 144 actual-weight
MoE cases, retained component checks and A/B/C users 1–8 qualified. The cleaned
public source also passed 96 exact-image CPU checks with no skips and CUDA
initialization disabled. Profiling was off for headline timing. Fifteen bounded
traces confirm 40 unchanged floating attention calls and removal of 40 old
packing calls per target step, plus fewer framework index/scalar operations.
Profiled spans are not unprofiled wall percentages or global critical-path proof.

The initial 2K screen stopped on a first-shape latency regression. Shared-kernel
JIT logs correlated with it. A separately labelled warm confirmation preceded
quality and the full run; the failure was not discarded or called a 58% gain.
All measured paths were checked on every rank; fallback was never counted as a gain.

Snapshot and artifact hashes are in [SNAPSHOT.json](SNAPSHOT.json).
The effective source/native snapshot rechecked 180 files against their pins.
The full raw evidence bundle (hash below) is retained separately; it is not a
public download hosted by this commit. No weights, binary modules, private
container inspection, launch/state records or complete server logs are committed.

## Attribution and drafter scope

GNmac's MIT-licensed recipe supplied the selected short-row FP8 target paths,
FP8 draft head and fused Markov implementation. Our integration retains the
earlier optimized expert/attention/collective paths, adds qualification and
sampling/graph guards, and completes full integer prefill fusion. Morrowmake's
MoE foundations and the vLLM/ExLlamaV3 lineages remain separately credited in
[the component notice](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/tree/036f62398e32689e5cfa8bc6bd0255d602597dac/experiments/ds1017-bf16-full-fusion/NOTICE.md).

The main measured arm used five draft tokens with the confidence gate off.
It did **not** test GNmac's full adaptive 3/5 verification policy or 0.10 gate.
The loaded bank cannot enable uncaptured arms with an environment toggle.
[The next drafter ablation](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/tree/036f62398e32689e5cfa8bc6bd0255d602597dac/experiments/ds1017-bf16-full-fusion/DRAFTING-ABLATION.md) isolates widths,
gate and proposal policy, measures local costs without donor interpolation,
and checks greedy IDs plus sampled rejection/RNG correctness before promotion.

## Follow-up: already-integrated draft components

After the immutable main report was frozen, a new user-requested B → C → B
comparison ran on the same loaded image: one user, temperature zero, three
repeats per prompt, fixed five drafts, gate off, REFERENCE prefill and BF16 KV.
It holds the B/C target family fixed and compares the integrated FP8 draft-head
and fused Markov components. All 27 measured requests passed body/content/hash,
cache-zero and all-rank target-path checks. Warmups are separate.

| Prompt | B first wall | C wall | B repeat wall | C cut vs bracketing B |
|---|---:|---:|---:|---:|
| count | 1.898s | 1.875s | 1.895s | 1.13% |
| code | 2.012s | 2.002s | 2.030s | 0.94% |
| prose | 3.331s | 3.242s | 3.319s | 2.48% |

The current component benefit is modest; this does not test the complete GNmac
adaptive policy or its gate, and cannot assign the historical GNmac speed gap
to them. Count/code response text matched all three times. Prose differed both
between C and first B and between the two B passes. This is a timing follow-up,
not a new functional-quality result or proof of a unique C accuracy change.
Qualified target-graph hit counts are recorded, but are not a direct
per-position acceptance-rate measurement. B still has no new full SPARK sweep.

`DRAFT-COMPONENT-FOLLOWUP.json` preserves medians, text-match counts, graph hits,
fallbacks and the raw follow-up digest. The original main-run evidence ZIP and
the pre-follow-up effective-code snapshot remain unchanged.

## Data files

- `measurements.json`: current and historical aggregate cells and likelihood results.
- `request-timings.jsonl`: compact full-SPARK request timings and body/content hashes.
- `quality-positions.json`: raw per-token score/winner fields for the two new screens.
- CSV files: prefill, Morrow, SPARK categories, streaming/TTFT and seeded controls.
- `SNAPSHOT.json`: immutable image/source/artifact pins, with explicit scope.

No single decode arm proved globally optimal. Keep BF16 KV/full fusion/Marlin as
the supported common kernel settings; choose A for accuracy-first use or C for
the measured speed option. B needs a fresh full SPARK comparison before being called
the best balance. No default rollout or live model reload accompanies this update.

## Publication privacy

The span metadata endpoint is a sanitized loopback example. No private LAN
endpoint, personal home path or site-specific Docker socket is required to
interpret the measurements. All scores, token IDs and measured timings remain
unchanged. The source manifest distinguishes the six privacy-edited offline
helpers from the unchanged tested inference runtime/native code.
