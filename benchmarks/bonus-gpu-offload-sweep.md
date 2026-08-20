# Bonus - GPU offload sweep

Host `Darwin-arm64` · backend(s) `apple_metal` ·
llama.cpp `b10488` · `threads=12` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 8.5 | 1.00x | 14% |
| 8 | 5.6 | 0.66x | 9% |
| 16 | 25.9 | 3.05x | 42% |
| 24 | 22.7 | 2.68x | 37% |
| 32 | 41.0 | 4.82x | 66% |
| 99 | 62.1 | 7.31x | 100% |

Best: `-ngl 99` at 62.1 tok/s
-- 7.31x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

**Yes — full offload wins outright, and by the largest margin of anything measured in this
lab: `-ngl 0` -> `-ngl 99` is 8.5 -> 62.1 tok/s = 7.31x.** Nothing ran out. The curve does
not peak at a partial value, so neither VRAM nor host-device bandwidth was the limit:
Gemma 4 E2B `UD-Q4_K_XL` is 2.97 GB and the Metal device reports 10922 MiB available
(`submission/run-logs/00-probe.log`), so all 35 layers fit with ~7.5 GB to spare.

**But the curve is not monotonic, and that is the interesting part.** `-ngl 8` (5.6 tok/s)
is *slower than CPU-only* (8.5 tok/s) — offloading 8 layers made things **34% worse** than
offloading none. `-ngl 24` (22.7) is likewise below `-ngl 16` (25.9). Partial offload is
not a smooth interpolation between the two endpoints; over part of the range it is worse
than either.

**Why partial offload can lose to CPU-only.** With a split model, every token's forward
pass crosses the CPU/GPU boundary at each transition between a CPU-resident block and a
GPU-resident block. Apple Silicon's unified memory means those crossings are not PCIe
copies — there is no data movement to pay for — but each one is still a **kernel dispatch
and a synchronization point**: the CPU side must finish and signal, the GPU side must be
scheduled, and neither can overlap with the other because a decode step is a strict
dependency chain. At `-ngl 8` you have paid for a full set of boundary crossings per token
and bought only 8 layers of GPU compute in return, so the sync overhead dominates. By
`-ngl 32` the GPU is doing enough work per crossing to be ahead, and at `-ngl 99` there
are no crossings left at all — one dispatch, whole graph, no CPU in the decode loop.

**A confound I should name.** This sweep ran at `threads=12`, which
`benchmarks/01-tuning-tg128.md` shows is the *worst* thread count on this chip (four ggml
workers land on E-cores and the barrier waits for them). That penalty falls hardest on the
low-`ngl` points, where the CPU is doing most of the work, so the true CPU-only baseline is
probably somewhat better than 8.5 tok/s and the real speedup somewhat below 7.31x. The
direction and the order of magnitude are not in question; the exact multiple is.

**And a sample-size caveat.** `-r 2`: two repetitions per point. The 16-vs-24 inversion is
within the range I would expect from run-to-run variance and thermal state on a laptop, so
I would not defend "16 beats 24" as a real effect. The endpoints (0 vs 99) differ by 7.3x,
which no amount of variance at this scale explains.

**What it means beyond the number.** The deck treats GPU offload as a capacity decision —
does the model fit. This says it is also a *granularity* decision: a model that half-fits
can serve worse than one that does not fit at all, so "offload as many layers as VRAM
allows" is bad advice in the region where it matters most. Either the whole model goes on
the device, or you should seriously consider leaving it entirely on the CPU and spending
the effort on thread placement instead.
