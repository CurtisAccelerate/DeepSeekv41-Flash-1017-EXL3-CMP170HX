# DeepSeek V4.1 Flash 1017 — EXL3 routed experts on CMP 170HX

[CurtisAccelerate](https://github.com/CurtisAccelerate).

Optimized for dual-socket Broadwell, 4× CMP 170HX, PCIe Gen2 ×16.
The October 22 published configuration measured **187.2 / 144.2 / 101.2 tok/s**
(count, code, prose). The October 25 experimental update is described below.
The comparison measures Morrowmake's repo run on this Broadwell. Those figures are our measurements, not numbers published by Morrowmake.
[This Broadwell serve versus Morrowmake's repo run here](benchmarks/2026-10-24/broadwell-vs-morrowmake.md).
The same charts: [broadwell-vs-morrowmake.html](https://curtisaccelerate.github.io/DeepSeekv41-Flash-1017-EXL3-CMP170HX/benchmarks/2026-10-24/broadwell-vs-morrowmake.html).

The October 17 measurements were 180.4 tok/s structured, 137.1 code,
96.6 prose; eight users aggregate 621, 376, and 283. The latest controlled kernel, P2P and four-Spark measurements are linked
under [Kernel improvements](#kernel-improvements--october-22-2026).

Routed experts only, layers 0–39, are LibertAI EXL3 2.9. Attention, shared
experts, the head, the draft, and Engram stay the official FP8 checkpoint.
Measured on 4× CMP 170HX at 140 W.

This repository is that weight selection and these measurements. It does
not contain the weight files. Licenses are in [NOTICE.md](NOTICE.md).

## Expert gates for 2–6 users — October 28, 2026

The rebuilt gate kernel is faster at 2–6 users and matches the previous expert
kernel bit for bit. Counting decode is +9% to +14%, code +6% to +14%, prose
+4% to +15%. One user stays on the October 26 attention kernel. Seven and
eight users stay on the tensor-core path.
[Headlines and tables](benchmarks/2026-10-28/README.md).

## 2–8 user attention — October 26, 2026

The new attention is faster at every user count from 1 through 8 on temperature-0
count, code, and prose. One temperature-1 request should sample the drafter.
Batches of 2–8 should keep the new attention and leave the drafter greedy.
Prefill is flat. Draft length stays 5.
[Headlines and tables](benchmarks/2026-10-26/README.md).

## Native mixed and experimental combined decode — October 25, 2026

Two successive additions now have [published source and explicit opt-in component flags](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/tree/8a59fd5a0bfbe61c43e24b01737adff18e0025d9/experiments/ds1017-native-mixed-and-split):
native mixed K2/K3 experts, then split8/core8 attention plus a captured
vocabulary-sharded K5 drafter. Same weights and quantization; target verification
is unchanged. The source bundle requires startup/capture integration and does
not activate the historical launch by exporting flags alone.

From the consistently published October 22 baseline to the combined candidate,
single-user short code/prose measured **144.23 → 155.54 / 101.16 → 116.05 tok/s**
(**+7.84% / +14.72%**); Spark-matched code/prose measured
**158.36 → 175.73 / 83.64 → 94.65 tok/s** (**+10.97% / +13.16%**).
These cumulative comparisons are descriptive across runs. The latest paired
increment over native mixed and the separate controlled **+4.10% counting**
result are distinguished in the [full tables and quality report](benchmarks/2026-10-25/README.md).
No consistent C4/C8 or cold-prefill gain is established.

**Experimental, opt-in.** Initial numerical/runtime checks passed and the path was evaluated in
limited matched end-to-end tests; task-quality results were mixed. Split attention changes floating-point reduction and
is not bit-exact; broader numerical accuracy and task-quality verification is
still warranted. The matched quality screen did not pass every criterion and
does not establish universal quality equivalence. All public component flags
default off.

Attention builds on vLLM's ROCm split-K kernels and PixelML's SM80 adaptation.
CurtisAccelerate developed the DS1017-specific tuning, dispatch/capture
integration and captured drafter path, and performed these measurements.
[Prior work, named contributors and source lineage](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/blob/main/experiments/ds1017-native-mixed-and-split/PRIOR-WORK.md).

## What you need

Historical recipe: 4× CMP 170HX, 64 GB each, sm_80, tensor parallel 4,
140 W, peer access off. The separate October 22 experiment enables only
qualified socket-local peers; it does not change the launch template below.
This run filled about 62.5 GB of each card.

The host this was measured on has 378 GB of RAM visible to the OS. That is
the machine, not a minimum. While this serve is up it is using about 289 GB,
and 89 GB is still free. Shared memory is 265 GB, which is the Engram pin.
Engram does not go on the GPUs. A 256 GB host does not fit that pin.

Two packs:

- [deepseek-ai/DeepSeek-V4.1-Flash-1017](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017).
  FP8 weights, FP4 experts. Take attention, shared experts, the head, the
  draft (including its FP4 experts), and the Engram weight and scale.
  Engram is shards 47 and 48. The 1017 release replaced those tables.
- [LibertAIDAI/DeepSeek-V4.1-Flash-1017-EXL3-2.9bpw](https://huggingface.co/LibertAIDAI/DeepSeek-V4.1-Flash-1017-EXL3-2.9bpw).
  Take only the main routed experts, layers 0–39. Leave its attention,
  head, and draft behind. The pack does not contain the 1017 Engram tables.

Put [quantization_config.json](quantization_config.json) on the hybrid
checkpoint. Do not copy the previous pack's config onto it.

## Run

No new container image, and no extra Python. 1017 uses the same serving
image as the kernel recipe, and the same EXL3 plugin. The only build is
the tensor-core `.so` from
[DeepSeekv41-Flash-EXL3-TC-MoE](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE).

The measured run used local image
`sha256:55d2bd9a6664aea2540e3f8e373d784b88420d35c3cedefc9f03923cd687b6f5`.
Host all-reduce and the sm80 package mounted beside it are
[vllm-cmp170hx](https://github.com/Morrowmake/vllm-cmp170hx) at
`18305948acba3f4c4d912cdd56f2202d2ebd94e8`.

`819146023f` is the GLM 5.3 recipe pin. That tree rejects
`VLLM_USE_V2_MODEL_RUNNER=0` with DSpark. This serve sets
`DSV41_DSPARK_V1=1` and runs DSpark on the V1 runner.

SM80 attention for this model is
[sm80vllm](https://github.com/PixelML/sm80vllm) branch `sm80`
(`DeepseekV4AmpereMLAAttention`).

1. Build `out/_dsv41_tc_moe_C.abi3.so` with that repo's
   `vllm_exl3/tc_moe/build_standalone.py`, against the PyTorch in the image
   you already serve this model with.
2. Make one checkpoint directory. Official 1017 tensors for attention,
   shared experts, the head, the draft, and Engram. LibertAI tensors for
   layers 0–39 routed experts only. Copy `quantization_config.json` into
   that directory.
3. The compressor projection is BF16. The image must build that linear
   with no quant config. Passing this config into it packs the weight into
   the wrong shape.
4. Launch with [scripts/run-tp4-1017.sh](scripts/run-tp4-1017.sh). Set
   `DSV41_IMAGE` to that existing image, `DSV41_MODEL_DIR` to the hybrid
   directory, and the same plugin, hook, `.so`, and Morrowmake host-allreduce
   files the kernel script takes. Set `DSV41_ARENA_CACHE_DIR` to a directory
   on fast disk.

The script turns on `VLLM_EXL3_ARENA_CACHE`. The first start writes the
arenas. A later start reads them. The loader skips the write if less than
4 GiB would remain free. One layer file is about 1.1 GiB. Decode does not
use the disk cache. It only shortens the next load.

`--max-model-len 262144` is the cap for these numbers. The weights address
1,048,576. The kernel repo's `scripts/run-tp4-tcmoe.sh` is the same launch
for the previous checkpoint. It did not document the arena cache. It does
now, when `DSV41_ARENA_CACHE_DIR` is set.

Serve flags for this measurement: `--tensor-parallel-size 4`,
`--max-model-len 262144`, `--max-num-seqs 8`, `--max-num-batched-tokens 2048`,
`--gpu-memory-utilization 0.97`, `--kv-cache-dtype fp8_ds_mla`, Engram
`cpu_offload`, prefix caching, `--async-scheduling`, DSpark depth 5, CUDA
graph mode `FULL_AND_PIECEWISE` with capture sizes
`1,2,3,4,6,8,12,18,24,30,36,42,48`.

```
VLLM_USE_V2_MODEL_RUNNER=0
DSV41_DSPARK_V1=1
VLLM_EXL3_COOP=1
VLLM_HOST_ALLREDUCE=1
NCCL_P2P_DISABLE=1
VLLM_EXL3_TC_MOE_V2=1
VLLM_EXL3_TC_MOE_PREFILL=1
VLLM_EXL3_TC_PREFILL_ROWS=2048
VLLM_EXL3_MOE_TP_ALIGN=128
VLLM_EXL3_MOE_TP_ROTATE=1
TORCH_CUDA_ARCH_LIST=8.0
```

The MoE kernels and the row rule are in
[DeepSeekv41-Flash-EXL3-TC-MoE](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE).
They run the experts. They do not choose which file an attention weight
comes from.

## Weight split

LibertAI supplies `layers.0`–`layers.39` routed experts only. Expert width
is on the trellis: most of those layers are 3-bit, and layers 18–22 are
2-bit. The `bits` field in the config is not that width.

Attention, shared experts, the head, the draft, and Engram weight plus
scale come from the official checkpoint. There is no second conversion and
no Engram table from the previous model. Using the published 1017 table
means the lookup keys match the checkpoint they were trained with.
LibertAI's own check with Engram switched off jumps code perplexity from
1.91 to 4.16 and prose from 6.45 to 9.06.

## Reader

[quantization_config.json](quantization_config.json) is the switch. The
EXL3 reader already branches on these keys. Two of them differ from its
defaults:

| Key | 1017 | If you leave the default |
|---|---|---|
| `head_bits` | omitted | when it is set, attention and the draft are built as EXL3 |
| `mtp_experts` | `source` | `exl3`, which quantizes the draft experts |

`mtp_experts_start_layer` is 40, so draft experts stay official FP4.
`scope` is `dsv41_main_routed_experts_only`. Do not set `non_routed_exl3`.
Attention, shared experts, the head, and Engram then stay on
`deepseek_v4_fp8`.

## Kernel improvements — October 22, 2026

**Controlled counting: small reproducible kernel gains. Resident A/B testing:
no weight reload.** On 4× CMP 170HX, the four-round sweep confirmed **+2.86%**
from the EXL transpose optimization and **+1.74% further** from six-row host
all-reduce tuning. A separate eight-pair confirmation measured
**184.21 → 188.13 tok/s (+2.21%)** for host tuning.

**Hybrid P2P:** socket-local peers plus host communication across sockets,
restricted to 48-row collectives. C8 counting improved
**572.60 → 586.31 aggregate tok/s (+2.43% paired)** with matched work.
Large-prefill paths did not win and remain unchanged. General prose benefit
is unproven; this is an opt-in topology-specific experiment.

**Full standardized suite:** C8 aggregate **271.57 tok/s**, versus previous
**273.48** and the published **four-DGX-Spark 221.66**. C1 aggregate was
**91.44 versus Spark 98.55**; 93,335-token prefill took **68.003 versus
42.180 s**. Different quants/engines/hardware prevent causal or quality claims.

Read the [controlled tables, 128K sweep, Spark comparison and resident P2P A/B](benchmarks/2026-10-22/README.md),
with [sanitized numerical records](benchmarks/2026-10-22/results.json).
The primary repo maintains the [new host-tuning patch](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/tree/2b15833e00c59ecb06692ca8dee64cd1cabc18da/experiments/host-allreduce-six-row),
[P2P design](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/blob/2b15833e00c59ecb06692ca8dee64cd1cabc18da/docs/hybrid-p2p-broadwell.md), and
[online CUDA graph technique](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/blob/2b15833e00c59ecb06692ca8dee64cd1cabc18da/docs/resident-cuda-graph-optimization.md).
Compatible late registration took about **0.30 s across four ranks**; updates
to 600 nodes/rank took **2.7–3.9 ms**, without reloading model/Engram or recapture.
Requests are drained during changes; this is not arbitrary live engine replacement.

The selected measured runtime keeps transpose + six-row tuning + narrow P2P.
Published launch templates retain their existing defaults. The October 21
section below describes its earlier experiment state, not current selection.

## Kernel improvements — October 21, 2026

A six-row ExLlamaV3 cooperative MoE experiment on **4× CMP 170HX**
preserves the EXL3 weight format and transposes the MMA operands. It passed
72/72 fixture/card bitwise comparisons and reduced tested expert latency
by about 4.8–6.7%. Controlled serving showed about 2–3% short-context decode improvement;
the public serving baseline remained selected at the end of that experiment. The baseline is a controlled
CUDA 12.8 rebuild, not proven identical to the historical stock binary.

[Two controlled sweep tables](benchmarks/2026-10-21/kernel-sweeps.md) cover
1/2/4/8 users and 2K–128K context. The separate
[4× CMP 170HX versus 4× DGX Spark comparison](benchmarks/2026-10-21/spark-comparison.md)
reports three complete passes: C8 aggregate median **273.48 tok/s**
[269.98–278.88], versus published TensorFold stock both-rails + patch0004
**221.66 tok/s**. Single-user aggregate was **92.85 versus 98.55 tok/s**;
93,335-token cold prefill took **67.975 versus 42.180 seconds**. This
descriptive comparison uses different systems,
quantizations and speculative work; it is not a causal kernel gain or quality
equivalence result. C8 in the controlled sweep showed +0.02% structured and
+0.41% code; prose differed in output/work and cannot isolate the kernel.

The [kernel source, CPU check and qualification notes](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/tree/fb9786ec0ed6a94225ddc4b0aa2976b9f9a8bf69/experiments/exl3-cooperative-decode)
are maintained in the primary kernel repository, pinned here to commit
`fb9786ec0ed6a94225ddc4b0aa2976b9f9a8bf69`. The patch applies to ExLlamaV3
at `da68d57de61781e5b74ad68698ac4caf36408d7a`. It remains an experimental
artifact requiring a guard matching the tested scope before general
integration. This 1017 repository maintains the model recipe and benchmark
evidence. No upstream PR or production default change is represented by this record.

## Historical benchmarks — October 17, 2026

140 W, thinking off, temperature 0, same prompts as the previous 140 W
sweep. Median of three warmed runs versus the previous Mia EXL3 3.0 bpw
serve, median of five. Decode is per stream after the first token.

| | 1017 | Previous 140 W |
|---|---:|---:|
| Structured tok/s | 180.4 | 178.6 |
| Code tok/s | 137.1 | 138.7 |
| Prose tok/s | 96.6 | 84.9 |

The cycle is about 34 ms. Structured keeps 5.97 tokens per step. Prose
keeps 3.31, versus about 3.12 on the previous shipped prose cell
(91 tok/s). The prose gain is more accepted draft tokens, not a faster
step.

Aggregate is every completion token divided by the wall time of the batch.
Decode is the per-stream rate beside it. Code stopped at 95 tokens on
every stream, so its one-user aggregate includes startup. Structured and
prose ran to 400.

| Users | Structured | Code | Prose |
|---:|---:|---:|---:|
| 1 | 156.5 / 180.8 | 93.4 / 137.2 | 89.5 / 95.9 |
| 2 | 248.0 / 155.3 | 127.6 / 112.7 | 133.6 / 71.3 |
| 4 | 419.2 / 114.2 | 173.2 / 83.0 | 187.8 / 50.0 |
| 8 | 621.1 / 85.1 | 376.1 / 62.8 | 282.5 / 37.2 |

Each cell is aggregate / decode. The previous structured eight-user
aggregate on the 140 W sweep was 564.7.

Long context on `/v1/completions`, 128 output tokens, median of three,
versus the previous 140 W curve. Each cell is prefill tok/s, decode tok/s,
and wall time.

| Prompt | 1017 | Previous 140 W |
|---:|---|---|
| 2,048 | 1,523 / 150.3 / 2.19 s | 1,570 / 149.2 / 2.16 s |
| 8,192 | 1,531 / 148.2 / 6.21 s | 1,528 / 146.8 / 6.22 s |
| 32,768 | 1,526 / 148.4 / 22.33 s | 1,519 / 148.5 / 22.42 s |
| 65,536 | 1,460 / 150.5 / 45.73 s | 1,453 / 149.9 / 45.94 s |
| 131,072 | 1,330 / 163.9 / 99.37 s | 1,329 / 156.5 / 99.48 s |

## Context

262,144 is the cap this serve was launched with, the same cap as the
previous measurement. It is not a property of the weights. The checkpoint
addresses 1,048,576 tokens, YaRN ×16 from 65,536.

The KV pool on this run is 1,534,065 tokens, 5.85 windows of 262,144,
versus 616,916 and about 2.4 windows on the previous quant. Eight chats
fit at about 192k each. One chat cannot spend the whole pool. Vision is on
(32-layer tower, hidden 1024). The drafter stays text-only.

## Needles

9/9. The needle code was `cobalt-heron-4417`, depths 10 / 50 / 90. Prompt
lengths on this run were 30,225, 120,967, and 241,425. The previous grid
was 9/9 at 32,706, 131,010, and 261,505.

## Accuracy

LibertAI measured their full-trunk 2.9 bpw pack, which also quantizes
attention and a 6-bit head, on unpublished held-out text (32×2048). Code
was 1.9094 against 1.8749 unquantized (+1.8%). Prose was 6.4520 against
6.4277 (+0.4%).

This serve quantizes only the routed experts, so it should sit closer to
the unquantized column than that 2.9 bpw row. That is an expectation, not
a new measurement. Their corpus was not rerun here.

## Credits

CurtisAccelerate implemented and published the DeepSeek V4.1 Flash EXL3 MoE
port and serving build, the 1017 weight recipe, six-row transpose/native-mixed
optimizations, Engram retention/reattachment, and resident A/B workflow.
The [contribution breakdown](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/blob/main/experiments/ds1017-native-mixed-and-split/PRIOR-WORK.md#deepseek-build-expert-kernels-and-loading-separate-contributions)
distinguishes our implementations from their upstream foundations.
The row rule on this hardware is
[DeepSeekv41-Flash-EXL3-TC-MoE](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE).

## Attribution -- Special Thanks!

DeepSeek, for the 1017 checkpoint.

LibertAI, for the EXL3 2.9 expert pack. The quant is theirs.

Morrowmake, for the original host-staged all-reduce and the earlier tensor-core
operators in [vllm-cmp170hx](https://github.com/Morrowmake/vllm-cmp170hx) that
CurtisAccelerate ported to DeepSeek. Our newer six-row transpose/native-mixed
expert changes build on Turboderp's cooperative kernels; they are a separate
source lineage. The six-row host-collective launch tuning is also our change
to Morrowmake's original collective.

Mia-AiLab, for the EXL3 3.0 bpw checkpoint this serve is compared with.

Turboderp, for the EXL3 format in
[exllamav3](https://github.com/turboderp-org/exllamav3).

The vLLM ROCm contributors, including Hexiang Wang, Fangzhou Ai and Tuukka
Sarvi, for the sparse-attention and split-decode lineage; haosdent and
thomaslwang in [PixelML/sm80vllm](https://github.com/PixelML/sm80vllm), for the
SM80 adaptation and FP8 helpers. The October 25 attention integration builds
on that work; [specific commits and our contributions are documented here](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/blob/main/experiments/ds1017-native-mixed-and-split/PRIOR-WORK.md).
