# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **12 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 65.5 | 99% |
| 6 | 65.9 | 100% |
| 12 | 47.6 | 72% |
| 24 | 57.5 | 87% |

**Best**: `-t 6` at 65.9 tok/s
**Slowest tested**: `-t 12` at 47.6 tok/s (1.38x spread)
**Against the physical-core default** (`-t 12`, 47.6 tok/s): 1.38x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Your explanation

**The knee is at `-t 6`, and the collapse at `-t 12` is the real finding.**

The curve does *not* have the shape the deck predicts. Expected: climb to the physical
core count, then flatten. Measured: essentially flat from 1 to 6 threads (65.5 -> 65.9
tok/s, +0.6%), then a **28% drop** at `-t 12` (47.6 tok/s), then a partial recovery at
`-t 24` (57.5 tok/s). Running at the physical core count — the value the lab picks by
default — is the **worst** setting tested.

**Why the curve is flat from 1 to 6.** With `ngl=99` every one of the model's layers is
resident on the Metal device, so the matmuls that dominate decode do not run on the CPU
at all. The `-t` workers only handle CPU-side graph nodes and the synchronization around
each Metal dispatch. One thread is already enough for that, which is why `-t 1` is within
1% of the best result. Thread count is not buying parallel compute here; it is only
changing how much CPU-side coordination overhead the run carries.

**Why `-t 12` collapses.** `hardware.json` reports "12 physical / 12 logical" cores, and
that is where the default comes from — but it is the wrong model of this chip. `sysctl`
(logged in `submission/run-logs/03b-cpu-topology.log`) shows the M2 Pro is **heterogeneous**:

```
hw.nperflevels       = 2
perflevel0 (P-cores) = 8 cores,  L2 = 16 MB
perflevel1 (E-cores) = 4 cores,  L2 =  4 MB
```

Eight performance cores and four efficiency cores, not twelve equal ones. ggml's thread
pool synchronizes on a **barrier between graph nodes**: every worker must arrive before
the next node starts, so each step runs at the speed of the *slowest* participant. At
`-t 6` all six workers land on P-cores. At `-t 12` four of the twelve workers are pinned
to E-cores that clock lower and share a 4 MB L2 instead of 16 MB — and because of the
barrier, those four set the pace for all twelve. The other eight threads spend the
difference spinning at the barrier. That is where the 28% goes: not to memory bandwidth,
to waiting.

**Why `-t 24` partially recovers.** At 2x oversubscription there are more runnable
threads than cores, so the macOS scheduler timeslices them instead of leaving four
workers permanently parked on E-cores. No single thread is stuck on slow silicon for the
whole run, so the slowest-arrival penalty averages out across the barrier — 57.5 tok/s,
better than `-t 12` but still 13% below `-t 6`, because now the cost is context switching
instead. Two different penalties, same barrier.

**What this contradicts, and what it means.** The deck's story — "past the physical core
count the threads fight over memory channels" — assumes homogeneous cores and CPU-resident
weights. Neither holds here. On this machine the binding constraint is barrier
synchronization across cores of *unequal speed*, so the right rule is not "threads =
physical cores" but "threads <= performance cores", and with GPU offload even that is
generous: `-t 1` is within 1% of best.

**Before / after actually applied:** `-t 12` (the lab default from `hardware.json`) ->
`-t 6`, **47.6 -> 65.9 tok/s = 1.38x**, from changing one integer. Full spread across the
grid is 1.38x (65.9 / 47.6).
