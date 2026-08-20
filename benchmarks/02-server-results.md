# 02 - Serve: load test + saturation reading

Host `Darwin-arm64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=12` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 96 | 1.68 | 4700 | 7000 | 7900 | 8.2 | 0.0% |
| 50 | 104 | 1.76 | 27000 | 30000 | 32000 | 37.7 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.05x** (21% of linear) |
| P95 latency | **4.29x** |
| Effective concurrency at 50 users | 37.7 vs `--parallel 4` slots (occupancy/slot ratio 9.43) |

**Saturated.** Throughput delivered only 1.05x for 5x the offered load, and effective concurrency (37.7) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.05x while P95 moved 4.29x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**This server saturates below 10 users — the 50-user run is not the knee, it is well past it.**

**The number that convinced me: RPS did not move.** 1.68 -> 1.76 req/s while the offered
load went 5x. Throughput has a hard ceiling around **1.7 req/s** on this box, and both
runs are already sitting on it. Everything the extra 40 users bought was latency: P50
4700 -> 27000 ms (5.7x), P95 7000 -> 30000 ms (4.3x), P99 7900 -> 32000 ms (4.1x), with
**zero failures** in either run — nothing broke, it just queued.

**Saturation starts below 10 users, not at 50.** Effective concurrency at 10 users is
already **8.2 against `--parallel 4`** — roughly 4 requests decoding and 4 more waiting.
A server that is queueing at its lowest tested load was saturated before the experiment
started. The true knee is somewhere around 4-6 concurrent users; I did not measure it,
and I am not going to claim a number I did not run.

**The extra latency is queue time, and I can prove it server-side rather than infer it.**
Three independent lines of evidence agree:

1. **`llamacpp:requests_deferred` sat at 43-46 for the entire 60 s window**
   (`benchmarks/02-server-metrics-u50.csv`). That gauge counts requests that arrived and
   found no free slot. It is the queue, measured by the server itself, not derived.
2. **The two concurrency numbers reconcile exactly.** `n_busy_slots_per_decode` peaked at
   **3.95 of 4** while Little's Law gave **37.7**. Those are not in conflict:
   `3.95 running + ~34 deferred = ~37.9`, against a measured 37.7. The server was
   never idle *and* 90% of the requests in the system were doing nothing but waiting.
3. **The arithmetic closes.** At a 1.76 req/s ceiling, 34 queued requests take
   34 / 1.76 = **19.3 s** to drain. Add the ~7 s a request needed at 10 users when the
   queue was short, and you predict ~26 s. Measured P50 at 50 users: **27000 ms**. The
   whole 20 s of added latency is accounted for by waiting, with nothing left over to
   blame on compute.

**What I would change first — and it is not `--parallel`.** The tempting move is more
slots, and it is the wrong one: `n_busy_slots_per_decode` is already **3.95/4 (99%)**, so
the slots are not idle waiting for work — they are compute-saturated. Adding slots divides
the same fixed decode bandwidth across more concurrent streams. Aggregate throughput stays
pinned near 1.7 req/s (that ceiling is the GPU, not the scheduler), every individual
request gets slower, and TTFT improves only because requests start sooner and then crawl.
That trades a queue you can see for a slowdown you cannot.

**First knob: admission control / a bounded queue.** Fix an SLO — say **P95 <= 10 s**.
Little's Law then sets the budget directly: `L = 1.76 req/s x 10 s = ~17 requests` may be
in the system at once. We ran at 37.7, which is **2.2x over budget**, which is exactly why
P95 landed at 30 s instead of 10 s. Cap in-flight work at ~17 and shed or 429 the rest,
and goodput@SLO goes from **0 req/s** (at 50 users literally nothing met a 10 s P95) to
roughly the full 1.7 req/s. The throughput number does not improve at all — the number of
requests that are actually *useful* goes from none to all of them. That is the whole
goodput-vs-throughput argument, and this run is a clean instance of it.

**Second knob, if I wanted the ceiling itself to move:** cut work per request rather than
add capacity — cap `max_tokens` (the load generator asks for 48/96 tokens; decode is
~95% of service time), or enable prefix caching so the 200-token shared RAG context in the
`long-rag` task is not re-prefilled on every call. Only after those would I revisit slot
count.

**One caveat on sample size.** 97 and 106 completed requests over 60 s is a thin basis for
a P99, and locust averages only *completed* requests — with ~44 still queued at cutoff,
the 37.7 effective concurrency is an **under**-estimate, not an over-estimate. The
throughput-scaling row (1.05x for 5x load) is the robust finding; treat the P99 column as
indicative.
