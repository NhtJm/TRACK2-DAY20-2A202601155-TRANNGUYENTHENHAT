# Bonus C2 — KV cache quantization: `f16` vs `q8_0`

Host `Darwin-arm64` (Apple M2 Pro, 16 GB) · llama.cpp `b10488` · Gemma 4 E2B
`UD-Q4_K_XL` · `ngl=99` (Metal) · the **only** thing that changes between the two
columns is `--cache-type-k` / `--cache-type-v`.

Run logs: `submission/run-logs/13-bonus-c2-kv-quant.log` (eval + serving latency) ·
`13b-bonus-c2-kv-memory.log` (memory) · `13c-bonus-c2-kv-bench.log` (llama-bench).

## 1. Memory — the saving is real and it is exactly 2x

Measured with `vmmap -summary` on the live `llama-server` process, `--ctx-size 32768`
split across `--parallel 4` slots. Physical footprint counts the Metal allocations;
the mmap'd weights do not appear in it, so the delta is the KV cache and nothing else.

| KV type | Physical footprint | RSS | Delta |
|:--|--:|--:|--:|
| `f16` (default) | **372.9 MB** | 338 MB | — |
| `q8_0` | **265.9 MB** | 187 MB | **−107.0 MB (−28.7%)** |

107 MB saved out of a ~214 MB f16 KV cache is a clean **2.0x reduction** — exactly what
halving the per-element width predicts, with no hidden overhead. The knob does what it
says.

## 2. Latency — it costs a quarter of decode throughput

`llama-bench`, `-r 3`, `-t 6` (the tuned thread count), same model and backend:

| KV type | pp512 — prefill (tok/s) | tg128 — decode (tok/s) |
|:--|--:|--:|
| `f16` | 1022.05 ± 25.50 | **66.60 ± 1.92** |
| `q8_0` | 997.46 ± 8.29 | **50.03 ± 1.71** |
| change | −2.4% (error bars overlap — call it flat) | **−24.9%** |

The serving-path measurement agrees independently: over the 10-prompt eval through
`/v1/chat/completions` at `ctx=8192`, decode fell **57.1 → 45.3 tok/s (−20.7%)** and mean
prefill rose 105.2 → 123.6 ms. Two different harnesses, two different thread counts, same
direction and roughly the same magnitude.

## 3. Quality — no measurable change

10 auto-gradable prompts (5 arithmetic, 5 JSON extraction), `temperature=0`, `seed=7`,
graded by exact numeric match and parsed-JSON equality:

| KV type | Score |
|:--|--:|
| `f16` | **9 / 10** |
| `q8_0` | **9 / 10** |

Both configurations failed **the same** item (`18 * 15 + 30`, answered `330` instead of
`300`) and passed all five JSON extractions byte-identically. That shared failure is the
useful part: it is a model arithmetic error, not a cache-precision error, and its presence
in both columns is the control that says the eval is measuring the model rather than noise.

## Verdict

**Do not enable `q8_0` KV on this machine.** The trade is 107 MB of memory for 25% of
decode throughput, on a 16 GB box where the Metal device reports 10.9 GB free and the KV
cache is under 3% of what is available. Paying a quarter of the serving rate to reclaim
memory that was never scarce is a straight loss.

**Why it loses here, mechanically.** A decode step reads the whole 2.95 GB weight set plus
the KV cache for the tokens so far. At the context lengths in this test the KV cache is a
rounding error next to the weights, so halving it removes almost no bytes from the critical
path — while the dequantization of every cached K and V block, on every attention step, adds
ALU work that was not there before. Cost with no matching benefit.

**This is the same result as the weight-quantization experiment, and that is the finding.**
`benchmarks/01-quickstart-results.md` found `UD-Q2_K_XL` gave only 1.05x for a 25% smaller
model. Here `q8_0` KV gives −25% for a 50% smaller cache. Two independent knobs, one
conclusion: **on this hardware, fewer bits does not buy speed, because decode is not
starved for bytes — it is paying for dequantization.** The deck's "quantize to go faster"
intuition is a statement about bandwidth-bound systems, and an M2 Pro running a 4.65 B
model on Metal is not one at these sizes.

**Where it would flip.** The trade turns positive the moment KV genuinely competes for
memory: a model large enough that weights + KV exceed the device, contexts in the 100k+
range, or many more concurrent slots. The knob is not bad — this machine is just not the
place where it pays.
