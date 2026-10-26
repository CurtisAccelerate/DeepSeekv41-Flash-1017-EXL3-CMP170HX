# Native mixed experts and experimental combined decode — October 25, 2026

Same DeepSeek V4.1 Flash 1017 weight selection: LibertAI routed experts at
approximately 2.9 bpw, with the other components from the official checkpoint.
Hardware: **4x CMP 170HX**. No new quantization, SM unlock or P2P/prefill
optimization is represented by these results.

The [canonical component source and integration guide](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/tree/8a59fd5a0bfbe61c43e24b01737adff18e0025d9/experiments/ds1017-native-mixed-and-split)
are maintained in the base/kernel repository. That source bundle includes
native kernels and a guarded adapter, split-attention/captured-drafter code,
retained dependencies, generated fixtures, CPU checks and independent opt-in
flags. It requires a compatible server's startup/capture coordinator; it is
not a complete image or an automatic installer for the historical launch.

**Experimental, opt-in.** The integrated attention/drafter path passed initial
numerical/runtime checks and was evaluated in limited matched end-to-end tests;
task-quality results were mixed. Split attention
changes floating-point reduction/rounding and is not bit-exact with the
reference; broader verification of numerical accuracy and task quality is still
warranted. Results do not establish universal quality equivalence. The quality
screen below did not pass every acceptance criterion.

## Two successive changes

**Part 1, native mixed experts:** K2 retains guarded behavior and writer-fence
handling; K3 specializes the eligible six-row native MoE contract. The previous
transpose and six-row host-allreduce improvements were already published.
Controlled counting measured **187.723 → 195.423 tok/s (+4.10%)**, 24 requests
in six balanced blocks, with matched prompt/output IDs, cached tokens and
speculative work. This is not a +4.10% claim for every code or prose request.

**Part 2, combined C1 path:** split attention uses **8 splits / 8 core warps /
4 reducer warps / output copy**. The captured vocabulary-sharded drafter keeps
the existing norm/local head, projects 32,320 of 129,280 vocabulary rows per
rank and exchanges local maximum value/token-ID pairs for the five-position
greedy chain. Target verification and target sampling remain unchanged; **K5
is unchanged**. The measured combined bank adds these to native mixed on the
eligible FULL T6/B1 descriptor. These are separate mechanisms and have separate
component preparation flags.

## Cumulative single-user performance

Single-user streamed decode, tokens/second. Starting point is the October 22 published optimized configuration. Part 1 is the later native-mixed configuration; Part 2 adds experimental split attention and captured vocabulary-sharded drafting. Historical comparisons describe observed configuration results, not isolated component effects.

### Short prompts

| Task | Published baseline | Part 1: native mixed | Part 1 change | Part 2: combined | Part 2 change | Final total |
|---|---:|---:|---:|---:|---:|---:|
| Short code | 144.23 | 146.28 | +1.42% | 155.54 | +6.33% | **+7.84%** |
| Short prose | 101.16 | 102.82 | +1.64% | 116.05 | +12.87% | **+14.72%** |

### Spark-matched prompts

| Task | Published baseline | Part 1: native mixed | Part 1 change | Part 2: combined | Part 2 change | Final total |
|---|---:|---:|---:|---:|---:|---:|
| Spark-matched code | 158.36 | 169.44 | +7.00% | 175.73 | +3.71% | **+10.97%** |
| Spark-matched prose | 83.64 | 88.46 | +5.76% | 94.65 | +7.00% | **+13.16%** |

All percentages above are ratios of the displayed rates so the stages compound consistently. In the same-load three-pair comparison, median paired Part 2 gains were short code +6.33%, short prose +12.87%, Spark code +3.71%, Spark prose +6.25%. Those are a different statistic and are not added to historical percentages.

Native mixed independently measured 187.72 → 195.42 tok/s (+4.10%) on controlled counting with matched work. This is not a +4.10% claim for every code/prose request.


## Concurrency and prefill

The full Spark-matched suite completed 1,608 requests across both configurations.
The C8 eight-category aggregate was **275.11 → 276.55 tok/s (+0.52%, ratio of
medians)**. At 93,335 prompt tokens, cold prefill was **1,389.0 → 1,391.1 prompt
tok/s**. No consistent C4/C8 or cold-prefill improvement is established.

Unsupported combined shapes used the older transpose/boot + six-row fallback,
whereas the optimized reference used global native mixed + six. Concurrency
comparisons therefore measure different whole configurations, not an isolated
attention/drafter effect. The historical published baseline is consistently
the October 22 configuration; a later per-task best from another configuration
is not substituted into individual cells.

## Limited matched task-quality screen

Ten matched pairs / 20 complete tool-using tasks, 40 stages: eight independent
templates plus two repeats. All completed; there were no case exclusions or
retries in the primary result. Stages and repeats are not independent samples.

| Outcome | Optimized reference | Combined candidate |
|---|---:|---:|
| Code tasks passing functional/quality review | 5/5 | 4/5 |
| Prose tasks passing strict source-grounding rubric | 0/5 | 1/5 |
| Prose required-fact checklist coverage and format | 5/5 | 5/5 |

The candidate's one code failure used SKU where order ID was required; the
initial stock-task wording was ambiguous. Excluding both stock-task pairs is
a sensitivity analysis only: code becomes 3/3 versus 3/3. Under lenient prose
attribution, results become 3/5 versus 4/5. Both arms had source-grounding
failures. Checklist coverage does **not** mean all claims were accurate.
The primary comparative code criterion failed; these results neither establish
universal quality equivalence nor prove an attention-induced regression.

| Measured whole-task time | Reference seconds | Candidate seconds | Change |
|---|---:|---:|---:|
| All 20 tasks, including quality failures | 2101.456 | 1792.038 | -14.72% |
| All code tasks | 931.757 | 949.407 | +1.89% |
| All prose tasks | 1169.698 | 842.631 | -27.96% |
| Four code pairs passing in both arms | 774.838 | 764.456 | -1.34% |

Generated work, tool choices and output lengths differed. These timings are not
isolated kernel gains. There were zero strict both-pass prose pairs, so the
27.96% prose time reduction is **not** a quality-qualified prose speedup. The
sample is small and does not support a broad agent-speed or non-inferiority
claim. No additional GPU tests were performed for source publication.

## Use and verification

The base repository's [integration guide](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/blob/8a59fd5a0bfbe61c43e24b01737adff18e0025d9/experiments/ds1017-native-mixed-and-split/INTEGRATION.md)
documents `DS1017_NATIVE_MIXED`, `W26_ATTENTION_SPLIT`, and `W26_SAMPLER_BANK`.
All default off. The sampler flag gates collective startup preparation; an
explicit prepared-state selection activates its callsite. The attention flag
prepares a dedicated target capture; it does not replace arbitrary attention
calls. Exporting flags alone does not activate ordinary serving.

Public adapter checks cover source/dependency signatures, ABI, default-off and
metadata rejection behavior. The native source was rebuilt without GPU
execution and its five machine-code sections matched the measured binary.
The public extraction still needs GPU qualification within each consumer's
complete integration. [Sanitized numerical records](results.json) preserve the
reported rates, methods and quality counts without operator logs or model data.
