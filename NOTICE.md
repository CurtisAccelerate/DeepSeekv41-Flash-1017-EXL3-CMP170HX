# Attribution

The published DeepSeek serving build and MoE port, 1017 weight recipe,
six-row transpose/native-mixed changes, Engram retention integration and
measurements are by
CurtisAccelerate (https://github.com/CurtisAccelerate).

This repository does not contain weights. It did not author the DeepSeek
checkpoint or the LibertAI quantization.

- [deepseek-ai/DeepSeek-V4.1-Flash-1017](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017)
  is the official checkpoint. The model card licenses the repository and the
  weights under MIT. Attention, shared experts, the head, the draft
  (including its FP4 experts), and the Engram weight and scale, shards 47
  and 48, come from that release.
- [LibertAIDAI/DeepSeek-V4.1-Flash-1017-EXL3-2.9bpw](https://huggingface.co/LibertAIDAI/DeepSeek-V4.1-Flash-1017-EXL3-2.9bpw)
  supplies the main routed experts, layers 0–39. LibertAI licenses that
  pack under MIT, the same license as the base model. The quant is theirs.
- The original host-staged all-reduce is Morrowmake's; the six-row launch-policy
  optimization is CurtisAccelerate's. The earlier tensor-core/prefill MoE path
  uses Morrowmake's operator foundations, with the DeepSeek port, K2/K3 support,
  dispatch, workspace integration and spill fixes implemented by CurtisAccelerate in
  [CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE),
  Apache-2.0. Our newer six-row transpose and native-mixed changes instead
  derive from Turboderp's cooperative expert kernels. That repository's NOTICE
  separates these lineages and names their source commits. The 1017
  weight selection in this repository is not that checkpoint's reader.
- EXL3 is the format from
  [turboderp-org/exllamav3](https://github.com/turboderp-org/exllamav3)
  (MIT). Copyright (c) 2025 Turboderp.
- The previous comparison serve is
  [Mia-AiLab/DeepSeek-V4.1-Flash-EXL3-3.0bpw](https://huggingface.co/Mia-AiLab/DeepSeek-V4.1-Flash-EXL3-3.0bpw)
  (MIT). Those weights are not in this repository.

## October 25 attention and drafter update

The attention integration directly uses retained vLLM ROCm sparse-attention
and split-K primitives, including work by Hexiang Wang, Fangzhou Ai and Tuukka
Sarvi, with the SM80/FP8 adaptation by haosdent, thomaslwang and PixelML/sm80vllm
contributors. The original Apache-2.0 notices and license are preserved in the
canonical source repository.

CurtisAccelerate developed the DS1017-specific dispatch, launch tuning,
capture/fallback integration and captured vocabulary-sharded drafter path,
and performed the measurements and validation reported here. See the
[detailed prior-work record](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/blob/main/experiments/ds1017-native-mixed-and-split/PRIOR-WORK.md)
for the upstream commits, dependency hashes and contribution boundaries.
