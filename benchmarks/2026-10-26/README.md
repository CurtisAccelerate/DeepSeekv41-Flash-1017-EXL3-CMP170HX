# 2–8 user attention, and when the drafter should sample — October 26, 2026

Same DeepSeek V4.1 Flash 1017 weights and the same 4× CMP 170HX host as the
[October 25 report](../2026-10-25/README.md). No new quantization. Draft length
stays 5. Prefill does not move. The numbers below are medians from that owner,
each optimized cell against the production path in the same run.

The interactive copy of these tables is the working canvas `r2-sweep-results`.
This file is the published report.

## Takeaways

1. **The new attention is faster at 2–8 users.** At temperature 0, count, code,
   and prose are ahead at every user count from 1 through 8. Seven users have no
   graph of their own; they run on the 48-row graph built for eight, and they
   still beat the old path at the same user count.
2. **Sample the drafter for one temperature-1 request. Leave it greedy for 2–8 users.**
   On Hermes, one request at a time, draft temperature 1 accepted 2.84 drafts per
   step against 2.59 for the greedy drafter, and the two code oracles stayed 3/3
   while model time fell from 294 to 170 seconds (dependencies) and 181 to 112
   (ledger). On Morrow at temperature 1, the greedy drafter held the gain through
   8 users, and sampling the drafter was slower than its own baseline at 7 and 8
   users on count.
3. **What is not in this result.** Draft length 7 was not loaded. Mia was not
   loaded. Prefill is flat. A temperature-0 request stays greedy at every batch
   size, because a greedy target does not sample the drafter.

Serving choice from these measurements: new attention for every batch size,
sampled drafter only when a single request is at temperature 1, greedy drafter
for batches of 2–8. That selection is not yet the public boot default. The
kernel repository records the image gap:
[October 26 serving note](https://github.com/CurtisAccelerate/DeepSeekv41-Flash-EXL3-TC-MoE/blob/main/experiments/ds1017-native-mixed-and-split/SERVING-2026-10-26.md).

## Lead tables

### Temperature 0, new attention versus production

Decode tokens per second. Both arms leave the drafter greedy. Every cell is ahead.

| Users | Count base | Count new | Count | Code base | Code new | Code | Prose base | Prose new | Prose |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | 182.6 | 204.8 | +12.2% | 136.5 | 147.7 | +8.2% | 96.6 | 110.4 | +14.3% |
| 2 | 129.2 | 140.5 | +8.8% | 89.1 | 95.0 | +6.7% | 67.1 | 73.9 | +10.1% |
| 4 | 99.9 | 108.7 | +8.8% | 75.0 | 76.7 | +2.4% | 47.9 | 51.4 | +7.3% |
| 7 | 71.3 | 73.9 | +3.6% | 52.5 | 54.8 | +4.4% | 33.3 | 34.3 | +2.8% |
| 8 | 78.6 | 81.4 | +3.6% | 60.3 | 60.9 | +0.9% | 35.7 | 36.7 | +2.7% |

Seven is slower than eight in absolute tokens per second because it is padded
onto the eight-user graph. The comparison that matters is seven against the old
path at seven users. That comparison is positive.

### Hermes, one temperature-1 request, greedy drafter versus sampled drafter

Both columns are the new attention. A Hermes trajectory is one request, so the
sampled steps use the eager sampled drafter, not the captured 2–8 user sampler.
The round log shows that path on every sampled verify step, and the captured
greedy sampler on every greedy step.

| Task | What the grade is | Greedy | Sampled | Model seconds |
|---|---|---:|---:|---:|
| dependencies | Sandbox oracle | 3/3 | 3/3 | 294.4 / 170.0 |
| ledger | Sandbox oracle | 3/3 | 3/3 | 181.1 / 112.2 |
| cleanup-runbook | Word and citation screen | 2/3 | 3/3 | 82.6 / 119.2 |
| testing-review | Word and citation screen | 0/3 | 0/3 | 139.9 / 131.7 |

Accepted draft tokens per verify step, after the canaries: **2.59** greedy
(17,349 steps) and **2.84** sampled (12,066 steps), out of 5 proposals.
Cleanup is a prose runbook, not a code oracle. Its model time went up. The
testing-review screen failed on both drafters. Those screens check words and
citations. They are not a semantic grade.

### Temperature 1, greedy drafter, new attention versus production

This is the 2–8 user choice. The drafter is greedy on both arms. Count and prose
are ahead at every user count. Code is ahead except 7 users.

| Users | Count | Code | Prose |
|---:|---:|---:|---:|
| 1 | +13.2% | +13.8% | +15.1% |
| 2 | +7.8% | +9.8% | +6.4% |
| 4 | +7.4% | +4.9% | +5.7% |
| 7 | +3.6% | −2.9% | +3.3% |
| 8 | +3.7% | +12.3% | +8.8% |

Sampling the drafter on the same Morrow prompts was the weaker speedup, and it
was behind its own baseline at 7 users (−0.7%) and 8 users (−2.2%) on count.
That is why 2–8 stays greedy.

## Full bench tables

Users 3, 5, and 6 at temperature 0, decode tokens per second.

| Users | Count base | Count new | Count | Code base | Code new | Code | Prose base | Prose new | Prose |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 3 | 113.4 | 124.6 | +9.9% | 81.6 | 89.5 | +9.7% | 54.7 | 60.0 | +9.7% |
| 5 | 91.5 | 95.4 | +4.3% | 66.9 | 68.8 | +2.9% | 41.6 | 42.8 | +3.1% |
| 6 | 82.5 | 86.4 | +4.7% | 60.9 | 64.7 | +6.2% | 38.7 | 39.6 | +2.3% |

Temperature 1, percent versus each run's own production baseline. Greedy is the
later run. Sampled sets draft temperature to 1. These are separate runs.

| Users | Count greedy | Count sampled | Code greedy | Code sampled | Prose greedy | Prose sampled |
|---:|---:|---:|---:|---:|---:|---:|
| 1 | +13.2% | +9.3% | +13.8% | +4.6% | +15.1% | +17.7% |
| 2 | +7.8% | +4.2% | +9.8% | −2.4% | +6.4% | +9.4% |
| 3 | +9.5% | +4.5% | +0.7% | +4.1% | +10.1% | +7.6% |
| 4 | +7.4% | +2.1% | +4.9% | +1.4% | +5.7% | +12.0% |
| 5 | +4.1% | +0.5% | +10.9% | +10.1% | +3.5% | +5.7% |
| 6 | +4.3% | +0.6% | +1.2% | +4.3% | +5.4% | +8.2% |
| 7 | +3.6% | −0.7% | −2.9% | −3.9% | +3.3% | +2.7% |
| 8 | +3.7% | −2.2% | +12.3% | +4.9% | +8.8% | +2.1% |

Spark, per-stream median. Temperature 0 is ahead at every listed user count.
Temperature 1 is ahead except 2 users and 8 users. Prefill at about 93K prompt
tokens is −0.2% at temperature 0 and −0.2% at temperature 1.

| Users | Spark T=0 | Spark T=1, greedy drafter |
|---:|---:|---:|
| 1 | +12.3% | +7.5% |
| 2 | +3.9% | −3.4% |
| 3 | +5.0% | +3.8% |
| 4 | +6.6% | +6.2% |
| 5 | +3.4% | +7.0% |
| 6 | +3.9% | +3.0% |
| 8 | +2.9% | −3.3% |

Cached-prefix decode from 2K through 128K, new attention versus production.
All 20 cells are ahead. This is not a prefill measurement.

| Context | 1 user | 2 users | 4 users | 8 users |
|---:|---:|---:|---:|---:|
| 2K | +25.4% | +21.5% | +18.0% | +3.3% |
| 8K | +8.0% | +23.1% | +5.0% | +4.4% |
| 32K | +22.1% | +17.3% | +5.1% | +14.6% |
| 64K | +12.3% | +15.9% | +7.7% | +6.9% |
| 128K | +14.7% | +12.5% | +4.3% | +2.1% |

Code quality on the matched screen was 10 of 10 fully correct on both arms.
Prose quality completed 10 of 10 with a valid infrastructure receipt and no
semantic grade. Earlier Hermes passes, with the drafter left greedy, kept the
dependencies and ledger oracles at 3/3 on both arms. The word screens moved
around and are not used as the ship decision.
