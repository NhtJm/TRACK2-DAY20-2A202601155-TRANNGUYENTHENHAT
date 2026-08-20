# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.

**Họ Tên:** Trần Nguyễn Thế Nhật (MSSV 2A202601155)
**Cohort:** K3
**Ngày submit:** 2026-08-20

> Mọi con số trong file này đều lấy từ `benchmarks/*.md` do lab sinh ra, và mọi lệnh đã
> chạy đều có log đầy đủ kèm exit code trong [`run-logs/`](run-logs/). Cách đọc và tái
> chạy: [`RUNBOOK.md`](RUNBOOK.md).

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

- **OS:** macOS 14.6.1 (Sonoma, build 23G93) · Darwin 23.6.0 arm64
- **CPU:** Apple M2 Pro
- **Cores:** 12 physical / 12 logical — **nhưng thực tế là 8 P-core + 4 E-core** (xem bên dưới)
- **CPU extensions:** NEON
- **RAM:** 16.0 GB (unified memory)
- **Accelerator:** Apple Metal — `MTL0: Apple M2 Pro (10922 MiB, 10922 MiB free)`
- **llama.cpp asset đã tải:** `llama-b10488-bin-macos-arm64.tar.gz` (prebuilt, không compile)
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary, 2.97 GB) + UD-Q2_K_XL (compare, 2.24 GB)

**Chạy ở đâu:** laptop của tôi — không dùng Colab/Kaggle. Toàn bộ base track và bonus
track chạy local.

**Setup story:**

Ba việc phải xử lý, cả ba đều có log làm bằng chứng.

**1. Tải model bị stall, phải chuyển sang curl.** `make setup` chạy được ~162 MB của file
compare rồi đứng hẳn: mọi TCP connection tới HF CDN chuyển sang `CLOSED` và file
`.incomplete` không tăng byte nào trong hơn 4 phút. Nguyên nhân là `hf_xet` gửi request
unauthenticated (server đã cảnh báo `You are sending unauthenticated requests to the HF
Hub`). Workaround theo đúng `labs/00-setup/MANUAL-DOWNLOAD.md` Cách 1 — `curl -L -C -`,
tải lại ổn định ~5.5 MB/s. Sau đó `make setup` chạy lại, phát hiện cả hai file đã có
(`already present`) và ghi `models/active.json` bình thường.

**2. `GPU offload` đổi từ OFF sang ACTIVE giữa hai lần probe.** Lần `make probe` đầu tiên
(chạy trước khi runtime binary được giải nén) báo không thấy accelerator. Sau khi
`fetch-runtime.py` xong, probe enumerate được `MTL0: Apple M2 Pro` và lab tự chuyển
`ngl` từ 0 lên 99. Đây không phải bug: `labkit.n_gpu_layers()` cố ý **hỏi binary** thay
vì tin `hardware.json`, để report không ghi khống một lần chạy GPU chưa từng xảy ra. Hệ
quả là toàn bộ số liệu dưới đây là số **Metal**, không phải CPU.

**3. `hardware.json` mô tả sai cấu trúc CPU — và đây là nguồn gốc của §5.** File ghi
"12 physical · 12 logical", nhưng M2 Pro là chip **dị thể**. `sysctl` xác nhận
(`run-logs/03b-cpu-topology.log`): `hw.nperflevels = 2`, `perflevel0 = 8 cores` (L2 16 MB),
`perflevel1 = 4 cores` (L2 4 MB). Lab lấy `cores_physical = 12` làm thread mặc định, và
đó lại đúng là thread count **tệ nhất** trên máy này.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Nguồn: `benchmarks/01-quickstart-results.md` (`make bench`, log `run-logs/02-bench.log`).
> `threads=12` `ngl=99` `ctx=2048` `max_tokens=64`, warm-up bị loại, 10/10 request thành
> công ở cả hai quantization.

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 6195 | 114 / 337 | 18.6 / 19.9 | 1297 / 1512 / 1512 | 53.6 |
| UD-Q2_K_XL | 2.24 | 5114 | 109 / 452 | 17.8 / 19.7 | 1227 / 1544 / 1544 | 56.3 |

**Quan sát:**

2-bit nhanh hơn **1.05×** (56.3 vs 53.6 tok/s) và nhỏ hơn 0.73 GB. **Không đáng đổi.**
Điểm đáng chú ý là weight nhỏ đi 25% nhưng decode chỉ nhanh hơn 5% — nếu decode ở đây
thực sự bị chặn bởi memory bandwidth thì con số phải gần 1.25× hơn. Nó không, nghĩa là
bytes không phải ràng buộc: với `ngl=99` mọi layer nằm trong unified memory và được
dequantize thành fp16 ngay trong Metal kernel, và block layout phức tạp hơn của
UD-Q2_K_XL trả lại gần hết phần bandwidth tiết kiệm được dưới dạng công ALU. Thêm nữa,
**tail latency còn tệ hơn**: TTFT P95 tăng 337 → 452 ms (+34%).

**Đã thử cùng câu hỏi trên cả hai** (`make serve` :8080 vs `serve.py --compare` :8090,
`temperature=0`, `seed=42` — log `run-logs/11a-quality-q4.log` và `11b-quality-q2.log`).
Ba probe: giải thích cơ chế, tính Little's Law, và trả JSON đúng format. **Cả hai đều
đúng cả ba** — UD giữ layer nhạy cảm ở precision cao nên 2-bit không vỡ. Khác biệt duy
nhất quan sát được: Q2 **dài dòng hơn** (74 token so với 55 cho cùng câu trả lời), nên
với `max_tokens` cố định nó trả lại một phần lợi thế TPOT. Ba prompt là mẫu mỏng, chỉ đủ
kết luận "chưa hỏng rõ", không đủ để nói "tương đương".

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Nguồn: `benchmarks/02-server-results.md` (`make load-report`) và
> `benchmarks/02-server-batching-u50.md` (`make metrics` chạy **chồng thời gian** với
> `make load-50` — log `07-load-50.log` kết thúc 22:59:58, `08-metrics-u50.log` bao trùm
> cửa sổ đó). Server: `--parallel 4 --cont-batching`, `ctx=2048`, `ngl=99`.

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.68 | 4700 | 7000 | 7900 | 8.2 | 0 |
| 50 | 1.76 | 27000 | 30000 | 32000 | 37.7 | 0 |

- **Offered load tăng 5×, throughput thực tăng:** 1.05×
- **P95 tăng:** 4.29×
- **Effective concurrency ở 50 users:** 37.7 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.95 / 4 slots (99%)

**Saturation reading:**

Server bão hoà **dưới 10 users**, không phải ở 50. Bằng chứng thuyết phục nhất là **RPS
không nhúc nhích**: 1.68 → 1.76 req/s khi tải tăng 5×. Trần throughput ~1.7 req/s, và cả
hai lần chạy đều đã ngồi trên trần đó. Ngay ở 10 users, effective concurrency đã là
**8.2 so với 4 slot** — tức đã có hàng đợi từ mức tải thấp nhất tôi đo.

Phần latency tăng thêm là **queue time**, và tôi chứng minh được từ phía server chứ không
phải suy diễn. Ba dòng bằng chứng khớp nhau: (1) `requests_deferred` giữ ở 43–46 suốt 60 s
— chính server đếm hàng đợi; (2) hai con số concurrency hoà giải chính xác —
`3.95 đang chạy + ~34 đang đợi = ~37.9` so với 37.7 tính bằng Little's Law; (3) số học
khớp — 34 request xếp hàng ở trần 1.76 req/s cần 19.3 s để rút, cộng ~7 s compute ra ~26 s,
đo được P50 = 27 s. Không còn phần dư nào để đổ cho compute.

**Knob tôi đổi trước tiên: admission control, không phải `--parallel`.** Thêm slot là
nước đi sai vì `n_busy_slots_per_decode` đã 3.95/4 — slot không rảnh chờ việc, chúng đang
bão hoà compute. Thêm slot chỉ chia cùng một băng thông decode cho nhiều luồng hơn:
throughput vẫn ~1.7 req/s còn mỗi request chậm đi. Với SLO **P95 ≤ 10 s**, Little's Law
cho ngay ngân sách: `L = 1.76 × 10 = ~17` request được phép nằm trong hệ thống. Chúng tôi
chạy ở 37.7, tức **gấp 2.2× ngân sách** — đúng lý do P95 rơi vào 30 s. Chặn ở ~17 và
429 phần còn lại thì goodput@SLO đi từ **0 req/s** (ở 50 users không request nào đạt
P95 10 s) lên gần trọn 1.7 req/s. Throughput không tăng một chút nào; số request **có
ích** đi từ không có gì lên toàn bộ.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Nguồn: `benchmarks/03-integration-results.md` (`make pipeline`, log `10-pipeline.log`).
> 3/3 query chạy xong và in ra context đã retrieve.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Không có cloud, không có IaC — chạy trên 1 laptop local | **stub** |
| N17 Data pipeline | Không ingest gì; corpus là literal `TOY_DOCS` 6 document trong `pipeline.py` | **stub** |
| N18 Lakehouse | Không có table format, không object store — "store" là một Python list trong RAM | **stub** |
| N19 Vector + features | Không có embedding model, không có vector index; `retrieve()` rơi về **keyword overlap** | **stub** |
| N20 Serving | `llama-server` b10488, Gemma 4 E2B UD-Q4_K_XL, Metal, OpenAI-compat | real |

**Latency split** (mean của 3 query):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 1646.6 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection:**

Đúng hướng nhưng sai mức độ — tôi đoán LLM chiếm ưu thế, không đoán hai stage kia bằng
**0.0 ms**. `embed = 0.0` vì không có lệnh embed nào được gọi (`embed_backend` ghi rõ
`keyword overlap`); `retrieve = 0.0` vì đó là phép giao tập hợp trên 6 chuỗi ngắn, xong
trong độ phân giải của đồng hồ. Đây là **hệ quả của stub**, không phải kết luận về RAG.

Mean còn bị lệch bởi một request lạnh: query 1 tốn 3513 ms (prefill 149 tok trong
**2812 ms**), query 2–3 chỉ 705/721 ms (prefill 113–114 tok trong **203–204 ms**). Đó là
Metal graph warm-up sau khi server vừa bị load test 50 users vùi dập — **median thực tế
là ~710 ms, không phải 1647 ms**.

Muốn giảm 2× thì phải đánh vào decode, không phải retrieval: retrieval đã 0.0 ms nên tối
ưu nó mua được 0%. Trong 705 ms của một query ấm, tỉ lệ là ~203 ms prefill + ~500 ms
decode (23–30 token ở ~48 tok/s). Hai đòn theo thứ tự: siết `max_tokens` (xin 200, dùng
23–30 — decode tỉ lệ thuận trực tiếp với số token phát ra), rồi prefix caching cho system
prompt + context dùng chung (bỏ được phần lớn 203 ms, và sẽ quan trọng hơn nhiều khi
corpus thật và context dài ra).

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

**Change:** hạ `-t` từ **12** (mặc định lab lấy từ `cores_physical` trong `hardware.json`)
xuống **6**. Một số nguyên, không compiler, không GPU, không đổi model.

```
before:  47.6 tok/s   (-t 12, tg128, llama-bench)
after:   65.9 tok/s   (-t 6,  tg128, llama-bench)
speedup: 1.38×
```

Nguồn: `benchmarks/01-tuning-tg128.md` (`make tune`, log `03-tune.log`). Toàn bộ grid:
`-t 1` = 65.5 · `-t 6` = **65.9** · `-t 12` = 47.6 · `-t 24` = 57.5 tok/s.

**Tại sao nó work:**

Curve này **không** có hình dạng deck dự đoán. Kỳ vọng: leo tới physical core count rồi
phẳng. Thực tế: phẳng từ 1 đến 6 thread (65.5 → 65.9, +0.6%), **rơi 28%** ở `-t 12`, rồi
hồi một phần ở `-t 24`. Chạy đúng physical core count — giá trị lab chọn mặc định — là
setting **tệ nhất** trong grid.

Cơ chế nằm ở chỗ `hardware.json` mô hình hoá sai con chip. Nó ghi "12 physical", nhưng
`sysctl` (log `03b-cpu-topology.log`) cho thấy M2 Pro có `hw.nperflevels = 2`: **8 P-core
với L2 16 MB, và 4 E-core với L2 4 MB**. Không phải 12 core ngang nhau. Thread pool của
ggml đồng bộ bằng **barrier giữa các graph node** — mọi worker phải tới đích trước khi
node kế tiếp chạy, nên mỗi bước chạy với tốc độ của thằng **chậm nhất**. Ở `-t 6` cả sáu
worker nằm trên P-core. Ở `-t 12`, bốn worker bị đẩy xuống E-core clock thấp hơn và chỉ
có 4 MB L2 — và vì có barrier, bốn thằng đó quyết định nhịp cho cả mười hai. Tám thread
còn lại đốt thời gian spin ở barrier. 28% biến mất ở đó: không phải vào memory bandwidth,
mà vào **chờ đợi**.

Còn `-t 24` hồi lên 57.5 vì ở mức oversubscribe 2×, số thread runnable nhiều hơn số core
nên macOS scheduler timeslice thay vì để cố định bốn worker nằm lì trên E-core. Không
thread nào bị kẹt trên silicon chậm suốt cả run, nên hình phạt "chờ thằng chậm nhất" được
trung bình hoá — nhưng vẫn thấp hơn `-t 6` 13% vì giờ trả giá bằng context switching. Hai
hình phạt khác nhau, cùng một barrier.

Điều này **mâu thuẫn với deck** ở chỗ: câu chuyện "quá physical core count thì thread
tranh nhau memory channel" giả định core đồng nhất và weight nằm ở CPU. Ở đây không cái
nào đúng — với `ngl=99` các matmul chạy trên Metal, `-t` chỉ điều khiển phần CPU-side và
đồng bộ quanh mỗi dispatch (đó cũng là lý do `-t 1` chỉ kém best 1%). Ràng buộc thật là
**barrier synchronization giữa các core không cùng tốc độ**. Quy tắc đúng trên máy này
không phải "threads = physical cores" mà là **"threads ≤ performance cores"**, và khi đã
offload GPU thì kể cả thế vẫn còn rộng tay.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

**Đã làm:** **B2** `make sweep-gpu` · **B3** before/after dưới đây · **B4** challenge
**C2** (KV cache quantization) · **B5** challenge **C9** (embedding serving regime).
**B1 không làm** — `cmake` không có trên máy và tôi không cài thêm toolchain cho lab này.

**Numbers (B3 — từ B2 `sweep-gpu`, không phải kết quả `make tune` của base):**

```
before:  8.5 tok/s    (-ngl 0,  CPU-only,     tg128)
after:   62.1 tok/s   (-ngl 99, full Metal,   tg128)
speedup: 7.31×
```

Nguồn: `benchmarks/bonus-gpu-offload-sweep.md`, log `12-sweep-gpu.log`.

**Điều này nói lên gì mà deck chưa nói:**

**GPU offload không phải quyết định nhị phân "có vừa hay không" — nó là quyết định về
granularity, và vùng ở giữa có thể tệ hơn cả hai đầu.** Curve không đơn điệu: `-ngl 8`
đạt **5.6 tok/s, chậm hơn cả CPU-only 8.5 tok/s**. Offload 8 layer làm mọi thứ **tệ đi
34%** so với không offload gì. `-ngl 24` (22.7) cũng thấp hơn `-ngl 16` (25.9).

Lý do: mỗi token phải vượt biên CPU/GPU tại mỗi chỗ chuyển giữa block ở CPU và block ở
GPU. Trên Apple Silicon những lần vượt biên đó **không** phải copy qua PCIe — unified
memory, không có data movement để trả giá — nhưng mỗi lần vẫn là một **kernel dispatch và
một điểm đồng bộ**, và một bước decode là chuỗi phụ thuộc chặt nên hai phía không chồng
lấn được. Ở `-ngl 8` bạn trả trọn bộ chi phí vượt biên mỗi token mà chỉ mua về 8 layer
compute. Tới `-ngl 99` thì không còn lần vượt biên nào: một dispatch, cả graph, CPU ra
khỏi vòng decode.

Nghĩa là lời khuyên "offload nhiều layer nhất mà VRAM cho phép" là **lời khuyên tồi đúng
ở vùng nó quan trọng nhất**. Hoặc đưa cả model lên device, hoặc để nguyên trên CPU và đi
tối ưu thread placement — nửa vời là lựa chọn tệ nhất.

**C2 — KV cache quantization (`f16` → `q8_0`):** tiết kiệm **107 MB (−28.7%)** physical
footprint ở `ctx=32768`, đúng bằng 2× như lý thuyết. Chất lượng **không đổi**: 9/10 ở cả
hai, và **cùng fail một câu** (`18*15+30` → `330`) nên đó là lỗi số học của model chứ
không phải lỗi precision. Nhưng decode **mất 24.9%** (66.60 ± 1.92 → 50.03 ± 1.71 tok/s,
`llama-bench -r 3`), prefill gần như không đổi. **Không đáng bật trên máy này** — đổi 25%
throughput lấy 107 MB trên hộp 16 GB còn trống 10.9 GB là lỗ thuần.
Chi tiết: `benchmarks/bonus-kv-cache-quant.md`.

**C9 — embedding serving:** throughput bão hoà ở **batch 4 (35.5 texts/s, 2.96×)** rồi
phẳng hẳn — batch 8 và 16 chỉ mua thêm latency. Đây là **regime ngược** với track 02:
prefill-bound, không KV cache, không decode loop, và throughput đến từ **static batching**
chứ không phải continuous batching. Hai regime cần scheduler trái dấu nhau: chat phải cho
request nhập batch ngay lập tức, embedding phải **cố tình chờ** để gom đủ batch. Đặt cả
hai sau một autoscaler dùng chung một metric thì chắc chắn scale sai một cái.
Chi tiết: `benchmarks/bonus-embedding-serving.md`.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Ba thí nghiệm độc lập trên máy này đều cho **cùng một kết luận, ngược với trực giác của
deck**: bớt bit không mua được tốc độ. Weight 25% nhỏ hơn chỉ nhanh hơn 1.05×; KV cache
nhỏ đi 2× lại **chậm đi** 25%; và thêm thread đúng bằng số core lại chậm hơn dùng nửa số
đó 1.38×. Điểm chung là các quy tắc trong deck đều là phát biểu về hệ **bandwidth-bound
với core đồng nhất**, còn một M2 Pro chạy model 4.65 B trên Metal thì không phải hệ đó.
Thứ đắt nhất ở đây không phải bytes — mà là **dequantization và đồng bộ**.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` + `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của tôi
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
