# Our build vs GNmac: key architectural decisions

Recorded one-user prose decode: our speed configuration 143.06, our BF16-projection option 135.18, historical GNmac 141.55 tok/s.

| Decision | Our build | GNmac recipe | Key tradeoff / choice |
| --- | --- | --- | --- |
| Model / layout | DeepSeek EXL3 serving integration; TP4 shards across 384 routed experts. | Same EXL3 pack family and TP4 layout. | Serving and kernel changes, not a new model or expert quantization. |
| KV cache | BF16 latent/SWA cache in all current lane options. | FP8 MLA cache (fp8_ds_mla). | BF16 retains more cache precision; FP8 uses fewer cache bytes and provides more capacity. |
| Output / dense math | Speed: short-row FP8 dense + output projection. BF16 projection is configurable. | Short-row FP8 dense + FP8 output projection. | Projection precision is separate from KV precision. Target vocabulary head remains BF16. |
| Expert dispatch | Our specialized 2–6 user paths; selected alternate paths at 1/7/8 users. | Alternate TP4 expert/padding recipe. | Retain our shape-specific kernels where useful; integrate selected alternate routes. |
| Attention / collectives | Retained split-attention integration and Broadwell host/pair tuning. | Historical attention dispatch and NCCL/PyNccl collective configuration. | Our integration preserves tuning for this Broadwell / PCIe topology. |
| Prefill | Our integer preparation/packing fusion across all 40 target layers, plus MoE prefill and Marlin dense. | GNmac expert-prefill and attention recipe. | Our fusion reduces integer preparation/launch work while retaining the floating attention kernel. |
| Draft head / sampler | FP8 draft head and fused sampler integrated with our target graph banks. | FP8 draft head and fused sampler. | Selected GNmac components adapted into our existing stack; target verification retained. |
| Verification policy | Fixed 5 verification; confidence gate off. | Adaptive 3/5 verification and 0.10 gate in the recorded recipe. | Our local policy tests favored fixed 5 overall; adaptive/gated variants remain optional source. |
| Temperature 1 proposals | Single-user unseeded proposals sampled by default; greedy opt-in. | Greedy proposals including temperature 1. | A separate proposal-policy choice; we measured mixed workload gains from changing it. |

Headline rates are recorded one-user prose decode at 140 W per GPU, from separate boots/configurations; they are not a fresh matched A/B. KV labels refer to latent/SWA cache; the indexer remains quantized. November 2, 2026.
