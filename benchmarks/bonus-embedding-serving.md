# Bonus C9 / B5 — Embedding serving is a different regime

Host `Darwin-arm64` (Apple M2 Pro) · llama.cpp `b10488` · same Gemma 4 E2B
`UD-Q4_K_XL` weights as the chat server, started with `--embedding --pooling mean`
on `:8081` · embedding dim 1536 · corpus 8 docs.

Run logs: `submission/run-logs/14a-serve-embed.log` (server) ·
`14b-bonus-c9-embed-demo.log` (sweep).

## Batch-size sweep — pure prefill, no decode loop

| Batch | Latency (ms) | Throughput (texts/s) | vs batch 1 |
|--:|--:|--:|--:|
| 1 | 83.6 | 12.0 | 1.00x |
| 2 | 84.5 | 23.7 | 1.98x |
| 4 | 112.6 | **35.5** | **2.96x** |
| 8 | 245.6 | 32.6 | 2.72x |
| 16 | 477.0 | 33.5 | 2.79x |

**The curve saturates at batch 4 and then goes flat.** Batch 2 is nearly free — 83.6 →
84.5 ms for twice the work, so throughput doubles almost perfectly. That is the signature
of a device that was **idle inside a single request**: one 20-token text does not fill the
M2 Pro's GPU, so the second text rides along in the same kernel launch at no extra wall
time. By batch 8 the device is genuinely busy and latency scales linearly with batch size
(245.6 → 477.0 ms for 8 → 16 texts is 1.94x for 2x the work), so throughput stops
improving. Everything past batch 4 buys latency and returns nothing.

## Why this is the opposite discipline from track 02

| | Chat / decode serving (track 02) | Embedding serving (here) |
|:--|:--|:--|
| Work per request | 1 prefill + N sequential decode steps | **1 forward pass**, nothing else |
| KV cache | required, grows per token | **none** — no state survives the request |
| Where throughput comes from | **continuous** batching: requests join and leave the running batch every decode step | **static** batching: collect texts, launch once |
| What batching is worth | 3.95/4 slot occupancy, but throughput still ceilinged at ~1.7 req/s | 2.96x, then hard flat |
| Right knob | `--parallel`, admission control | batch size, token-length sorting |

The two regimes want **opposite** schedulers. Decode serving batches because requests are
long-lived and mostly waiting on memory, so a request that arrives mid-flight should be
allowed to join immediately — waiting to form a batch would waste decode steps. Embedding
serving batches because requests are short and compute-dense, so the scheduler should
*deliberately wait* to accumulate a full batch before launching. Continuous batching applied
to embeddings would launch tiny under-filled kernels and leave the device idle; static
batching applied to chat would stall live requests behind batch formation.

**The autoscaler consequence.** Put both behind one autoscaler on a single metric and it
will mis-scale one of them. Decode capacity is bounded by KV memory and slot count, and its
health signal is queue depth (`requests_deferred`, which hit 44 in track 02). Embedding
capacity is bounded by raw prefill FLOPs, and its health signal is batch fill rate —
queue depth for an embedding service is not a problem, it is the *input*, because a longer
queue means fuller batches and better throughput. A shared "scale on p95 latency" rule
would scale the embedding pool out exactly when it should have let the queue grow.

## Limitations, stated plainly

- **This is not an embedding model.** It is the Gemma 4 E2B chat decoder run with mean
  pooling, reused to avoid another download. A decoder trained for next-token prediction
  is a weak sentence encoder, and the ranking shows it: the top hit is correct (0.847 for
  the doc that literally answers the query), but the runner-up is RadixAttention at 0.784
  — a topically unrelated document scoring within 0.06 of the right answer. The similarity
  distribution is compressed and high, which is exactly the failure mode that makes a single
  cosine threshold unusable. Real retrieval needs Qwen3-Embedding, BGE-M3 or EmbeddingGemma.
- **Small corpus, single machine.** 8 documents, one query, no reranker stage. The batch
  curve is the finding here; the ranking quality is not.
- **Latency numbers include HTTP overhead** — this measures the serving path, not the kernel.
