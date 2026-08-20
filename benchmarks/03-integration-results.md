# 03 - Integrate: RAG pipeline run

Host `Darwin-arm64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 3513.2 | 3513.2 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 705.5 | 705.6 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 721.2 | 721.2 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **1646.6** · total **1646.7**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

**Only N20 is real. N16 through N19 are all stubs, and none of them were replaced.**

| Day | Piece | Real or stub | What is actually running |
|:--|:--|:--|:--|
| N16 | Cloud / IaC | **stub** | No cloud, no IaC. Everything runs on one local laptop (Apple M2 Pro, macOS 14.6.1). No provisioning code was written. |
| N17 | Data pipeline | **stub** | No ingestion. The corpus is the 6-document `TOY_DOCS` literal hard-coded in `labs/03-integrate/pipeline.py`. |
| N18 | Lakehouse | **stub** | No lakehouse, no table format, no object store. The "store" is a Python list in process memory. |
| N19 | Vector + features | **stub** | No embedding model and no vector index. `retrieve()` fell back to **keyword overlap** — set intersection of words longer than 3 characters — which is why `embed` reads 0.0 ms in every row: no embedding call was ever made. `embed_backend` in the JSON confirms `keyword overlap`. |
| N20 | Serving | **real** | `llama-server` (llama.cpp `b10488`, prebuilt macOS arm64), Gemma 4 E2B `UD-Q4_K_XL`, `ngl=99` on Metal, OpenAI-compatible `/v1/chat/completions`, `--parallel 4 --cont-batching --metrics`. |

**Is the dominant stage what I expected? Yes on direction, no on degree.** I expected the
LLM to dominate; I did not expect the other two stages to be literally unmeasurable. The
split is `embed 0.0 ms · retrieve 0.0 ms · llm 1646.6 ms` — **llm is 100.0% of total**,
and that is not rounding: retrieval is a set intersection over 6 short strings, so it
finishes inside the timer's resolution. This result is an artifact of the stub, not a
finding about RAG. A real N19 (embedding model + ANN index over a real corpus) would put
a measurable embed call and a network round-trip into that budget; the honest reading of
this table is "the toy retriever costs nothing", not "retrieval is free in RAG".

**The mean is skewed by one cold request.** Query 1 cost 3513 ms; queries 2 and 3 cost
705 ms and 721 ms. The server timings show why — query 1 prefilled 149 tokens in
**2812 ms** (53 tok/s), while queries 2 and 3 prefilled 113-114 tokens in **203-204 ms**
(~560 tok/s). This was the first request after the server had been hammered by the
50-user load test, so it paid Metal graph/pipeline warm-up that the later two inherited
for free. A 13x swing in prefill rate between the first and second request is worth more
than the mean: **the median query here is ~710 ms, not 1647 ms.**

**If I had to halve this pipeline's latency, I would attack decode, not retrieval.**
There is nothing to win in embed or retrieve — they are already 0.0 ms, so even making
them infinitely fast buys 0%. Inside the 705-721 ms of a warm query the split is roughly
**203 ms prefill + ~500 ms decode** (23-30 tokens at ~48 tok/s). So:

1. **Cut output tokens.** `max_tokens=200` is requested but the answers used 23-30. Decode
   is ~70% of a warm query and is strictly proportional to tokens emitted; a tighter
   instruction ("answer in one sentence") is the cheapest real 2x available.
2. **Prefix caching for the shared prompt.** The system prompt plus the retrieved context
   block is re-prefilled on every call. Caching that shared prefix removes most of the
   203 ms — worth ~29% of a warm query, and it is the change that would matter most once
   the corpus is real and contexts get long.
3. **Not more slots.** This is a 3-query sequential run at concurrency 1; `--parallel`
   changes nothing here.
