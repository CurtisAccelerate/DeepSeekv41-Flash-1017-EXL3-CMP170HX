# 4× CMP 170HX Broadwell versus Morrowmake

DeepSeek V4.1 Flash 1017. Both stacks were measured on the same host:
4× CMP 170HX, dual-socket Broadwell, PCIe Gen2 ×16, no NVLink, 140 W per card.
These figures are not a previously published table.

Broadwell is [DeepSeekv41-Flash-1017-EXL3-CMP170HX](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-1017-EXL3-CMP170HX),
the October 22 optimized stack (transpose, six-row host all-reduce, narrow pair route).
Morrowmake is [dsv41-flash-cmp170hx-recipe](https://github.com/Morrowmake/dsv41-flash-cmp170hx-recipe),
engine [vllm-cmp170hx @ a0c57b523](https://github.com/Morrowmake/vllm-cmp170hx/tree/a0c57b52376efd67c57735bc949d4bafa22ae29e).
The stacks differ in engine, draft depth, and KV cache. This is a system comparison on one board, not an isolated kernel result.

## Decode, one user

Decode tokens per second after the first token. Broadwell is the median of four rounds.
Morrowmake is the five-run median, peer access off.

| Task | CMP 170HX Broadwell | Morrowmake |
|---|---:|---:|
| Count | 187.2 | 122.8 |
| Code | 144.2 | 90.3 |
| Prose | 101.2 | 62.4 |

On this protocol, Broadwell is the highest-performing build tested on this board.

## Spark real-world tasks

Spark prompt set, counting excluded. Task rates are decode tokens per second per stream.
The concurrency table is aggregate tokens per second, the mean of those eight tasks.
Broadwell is the October 22 median of three passes. Morrowmake is one pass, peer access off.
Morrowmake one-user code is 30.3 tok/s in that pass; the same pass reached 98.7 tok/s at two users.

### One user, per stream

| Task | CMP 170HX Broadwell | Morrowmake |
|---|---:|---:|
| Code | 158.4 | 30.3 |
| JSON | 125.2 | 64.2 |
| Story | 65.3 | 41.9 |
| Prose | 83.6 | 63.2 |
| Math | 141.5 | 117.8 |
| Reason | 106.2 | 97.4 |
| Summary | 70.9 | 66.5 |
| Format | 160.4 | 143.4 |

### Eight users, per stream

| Task | CMP 170HX Broadwell | Morrowmake |
|---|---:|---:|
| Code | 66.9 | 51.5 |
| JSON | 48.9 | 41.1 |
| Story | 23.7 | 32.5 |
| Prose | 32.4 | 36.7 |
| Math | 59.5 | 46.1 |
| Reason | 42.3 | 40.3 |
| Summary | 27.0 | 35.0 |
| Format | 68.3 | 57.2 |

### Aggregate by concurrent users

| Users | CMP 170HX Broadwell | Morrowmake |
|---:|---:|---:|
| 1 | 91.4 | 67.2 |
| 2 | 119.4 | 131.9 |
| 3 | 152.7 | 173.1 |
| 4 | 185.6 | 184.9 |
| 5 | 204.2 | 179.9 |
| 6 | 216.9 | 205.6 |
| 8 | 271.6 | 168.7 |

## Cold prefill

Prompt tokens over time to the first token, cold. On Broadwell the pair route matches peers off: that sweep made no pair calls. Morrowmake was not prefilled with the pair route on.

| Prompt | Broadwell, peers off | Broadwell, pair route | Morrowmake, peers off |
|---|---:|---:|---:|
| 2K | 1564.8 | 1565.4 | 1633.6 |
| 8K | 1544.4 | 1544.5 | 1707.9 |
| 32K | 1539.3 | 1539.9 | 1732.5 |
| 64K | 1469.4 | 1471.3 | 1630.6 |
| 128K | 1329.0 | 1327.8 | 1558.7 |

## Pair route on versus off

One-user decode. Broadwell compares the same resident workers. Morrowmake compares two fresh boots. Those Morrowmake bars are not the five-run median in the first table. Count and code match. Morrowmake prose falls to 28.5 tok/s because two of the three runs dropped.

| Task | Broadwell, peers off | Broadwell, pair route | Morrowmake, peers off | Morrowmake, pair route |
|---|---:|---:|---:|---:|
| Count | 187.1 | 187.2 | 254.1 | 256.2 |
| Code | 141.0 | 144.2 | 196.2 | 197.8 |
| Prose | 101.6 | 101.2 | 139.7 | 28.5 |
