# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Darwin-arm64` · llama.cpp `b10488`
Settings: `threads=12` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 6195 | 114 / 337 | 18.6 / 19.9 | 1297 / 1512 / 1512 | 53.6 |
| UD-Q2_K_XL | 2.24 | 5114 | 109 / 452 | 17.8 / 19.7 | 1227 / 1544 / 1544 | 56.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.05x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

**Verdict: keep `UD-Q4_K_XL`. The 2-bit build is not worth it on this machine.**

What the smaller quant actually buys here: **1.05x decode** (56.3 vs 53.6 tok/s, i.e.
TPOT P50 17.8 vs 18.6 ms) and **0.73 GB less on disk** (2.24 vs 2.97 GB). It also loads
1.1 s faster (5114 vs 6195 ms), which is a one-off cost, not a serving cost.

**The speedup is far smaller than the size cut, and that is the interesting part.**
The weights shrank 25%, but decode only got 5% faster. If decode on this box were purely
memory-bandwidth-bound, moving 25% fewer bytes per token should have moved the number a
lot closer to 1.25x. It did not, which says the bytes were not the binding constraint:
with `ngl=99` every layer sits in the M2 Pro's unified memory and is dequantized to fp16
inside the Metal kernel before the matmul. `UD-Q2_K_XL` has a more complex block layout
than `UD-Q4_K_XL`, so it hands back most of the bandwidth saving as extra dequantization
ALU work. Fewer bits only converts into speed when you are actually starved for bytes.

**Tail latency got worse, not better.** TTFT P95 went 337 -> 452 ms (+34%) on the 2-bit
build even though TTFT P50 barely moved (114 -> 109 ms). Same for E2E P95 (1512 -> 1544
ms). So the quantization that looks faster at the median is the one with the wider tail.

**Quality check (same prompts, both quants, `temperature=0`, `seed=42`).** Run logs:
`submission/run-logs/11a-quality-q4.log` and `11b-quality-q2.log`. Three probes —
a mechanism question, a Little's Law calculation, and a strict-JSON formatting task:

| Probe | UD-Q4_K_XL | UD-Q2_K_XL |
|:--|:--|:--|
| Why is decode bandwidth-bound? | correct, 55 tok | correct, **74 tok** |
| Little's Law, L = 1.8 x 21.5 | **38.7**, correct | **38.7**, correct |
| Return only JSON, values swapped | exact match | exact match |

At this difficulty the 2-bit build did **not** visibly degrade — Unsloth Dynamic keeps
the sensitive layers at higher precision, which is the whole point of the UD quants. But
it was consistently **more verbose**: 74 tokens where the 4-bit build used 55 for the same
answer. Under a fixed `max_tokens` budget, a chattier model finishes fewer answers, so its
5% TPOT advantage is partly given straight back in end-to-end time.

**Caveat, stated honestly:** three prompts is a thin sample and cannot detect the kind of
degradation 2-bit is known for (long-horizon reasoning, rare facts, code). It is enough to
say "not obviously broken", not enough to say "equivalent".

**When I would switch:** only under RAM pressure. On a 16 GB box with a 3 GB model there
is nothing to buy — 0.73 GB saved changes no decision. On an 8 GB machine, or if I wanted
a second model resident at the same time, 0.73 GB is the difference between fitting and
swapping, and then the 2-bit build wins on the only axis that matters.
