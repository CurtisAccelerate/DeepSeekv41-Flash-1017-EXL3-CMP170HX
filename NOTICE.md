# Attribution

The 1017 weight recipe and the measurements in this repository are by
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
- Tensor-core EXL3 MoE kernels and the host-staged all-reduce are Morrowmake's,
  ported in
  [CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE),
  Apache-2.0. That repository's NOTICE names the kernel commit. The 1017
  weight selection in this repository is not that checkpoint's reader.
- EXL3 is the format from
  [turboderp-org/exllamav3](https://github.com/turboderp-org/exllamav3)
  (MIT). Copyright (c) 2025 Turboderp.
- The previous comparison serve is
  [Mia-AiLab/DeepSeek-V4.1-Flash-EXL3-3.0bpw](https://huggingface.co/Mia-AiLab/DeepSeek-V4.1-Flash-EXL3-3.0bpw)
  (MIT). Those weights are not in this repository.
