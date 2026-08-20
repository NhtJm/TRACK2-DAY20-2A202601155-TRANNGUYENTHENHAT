# RUNBOOK — cách chạy lại toàn bộ lab và tìm bằng chứng

Mọi lệnh dưới đây **đã được chạy thật** trên máy nộp bài, và output đầy đủ của từng
lệnh nằm trong [`run-logs/`](run-logs/). Mỗi log bắt đầu bằng header ghi lệnh chính xác,
thư mục, máy, thời điểm bắt đầu, và kết thúc bằng footer ghi thời gian chạy + **exit code**:

```
===============================================================================
COMMAND : make bench
CWD     : /Users/nhatnguyen/Desktop/AIA/labs/phase2-track2/TRACK2-DAY20-...
HOST    : Nhats-MBP-2 | Darwin 23.6.0 arm64
STARTED : 2026-08-20 22:55:09 +07
===============================================================================
... toàn bộ output ...
===============================================================================
FINISHED: 2026-08-20 22:55:46 +07   ELAPSED: 37s   EXIT CODE: 0
===============================================================================
```

Máy: **Apple M2 Pro, 16 GB, macOS 14.6.1**, Metal active. Model: **Gemma 4 E2B**
(`UD-Q4_K_XL` + `UD-Q2_K_XL`). Runtime: **llama.cpp b10488 prebuilt**, không compile.

---

## Base track — 100 điểm

| # | Lệnh | Run log | File sinh ra | Rubric |
|--:|:--|:--|:--|:--|
| 1 | `make probe` | [`00-probe.log`](run-logs/00-probe.log) | `hardware.json` | 1 |
| 2 | `make setup` | [`01-setup.log`](run-logs/01-setup.log) | `models/active.json`, `runtime/`, `models/*.gguf` | 2 |
| 3 | `make bench` | [`02-bench.log`](run-logs/02-bench.log) | `benchmarks/01-quickstart-results.md` | 3, 4, 5 |
| 4 | `make tune` | [`03-tune.log`](run-logs/03-tune.log) | `benchmarks/01-tuning-tg128.md` | 11 |
| 4b | `sysctl hw.perflevel*` | [`03b-cpu-topology.log`](run-logs/03b-cpu-topology.log) | — (bằng chứng 8 P-core + 4 E-core cho §5) | 11 |
| 5 | `make serve` | [`04-serve.log`](run-logs/04-serve.log) | server :8080 | 6 |
| 6 | `make smoke` | [`05-smoke.log`](run-logs/05-smoke.log) | — | 6, 7 |
| 7 | `make load-10` | [`06-load-10.log`](run-logs/06-load-10.log) | `benchmarks/locust-10_*.csv` | 8 |
| 8 | `make load-50` | [`07-load-50.log`](run-logs/07-load-50.log) | `benchmarks/locust-50_*.csv` | 8 |
| 9 | `make metrics` **(chạy chồng với bước 8)** | [`08-metrics-u50.log`](run-logs/08-metrics-u50.log) | `benchmarks/02-server-batching-u50.md` + `.csv` | 9 |
| 10 | `make load-report` | [`09-load-report.log`](run-logs/09-load-report.log) | `benchmarks/02-server-results.md` | 10 |
| 11 | `make pipeline` | [`10-pipeline.log`](run-logs/10-pipeline.log) | `benchmarks/03-integration-results.md` | 12, 13 |
| 12 | so sánh chất lượng Q4 vs Q2 | [`11a-quality-q4.log`](run-logs/11a-quality-q4.log), [`11b-quality-q2.log`](run-logs/11b-quality-q2.log), [`11c-serve-compare.log`](run-logs/11c-serve-compare.log) | — | 5 |
| 13 | `make verify` | [`15-verify.log`](run-logs/15-verify.log) | — | 14 |

### Bước 9 phải chồng thời gian với bước 8 — đây là lỗi phổ biến nhất của lab

`make metrics` chạy khi server rảnh sẽ cho `n_busy_slots_per_decode ≈ 1` và **không**
chứng minh được continuous batching. Bằng chứng hai lệnh chạy chồng nhau nằm ngay trong
timestamp của log:

```
07-load-50.log      STARTED 22:58:58  →  FINISHED 22:59:58   (60s)
08-metrics-u50.log  STARTED 22:59:00  →  FINISHED 23:00:02   (62s)
```

Kết quả: peak `n_busy_slots_per_decode` = **3.95 / 4 slots**, `requests_deferred` = 43–46.

### Chạy lại từ đầu

```bash
make probe                       # 1
make setup                       # 2  (~5.2 GB model, 10-20 phút)
make bench                       # 3  (~40s)
make tune                        # 4  (~3 phút)

make serve                       # terminal 1 — để chạy suốt
make smoke                       # terminal 2
make load-10                     # terminal 2  (60s)
make load-50                     # terminal 2  (60s)  ┐ hai lệnh này
make metrics                     # terminal 3  (60s)  ┘ PHẢI chạy cùng lúc
make load-report                 # terminal 2
make pipeline                    # terminal 2
make verify                      # phải exit 0
```

> **Nếu `make setup` bị treo khi tải model:** trên máy này `hf_xet` đứng hẳn ở ~162 MB
> (mọi TCP connection `CLOSED`, file `.incomplete` không tăng trong 4 phút) do request
> unauthenticated tới HF Hub. Cách chữa theo `labs/00-setup/MANUAL-DOWNLOAD.md` Cách 1:
> ```bash
> curl -L -C - -o models/gemma-4-E2B-it-UD-Q2_K_XL.gguf \
>   https://huggingface.co/unsloth/gemma-4-E2B-it-GGUF/resolve/main/gemma-4-E2B-it-UD-Q2_K_XL.gguf
> make setup     # phát hiện file đã có, chỉ ghi models/active.json
> ```

---

## Bonus track — 16/20 điểm

| # | Nội dung | Lệnh | Run log | Report |
|:--|:--|:--|:--|:--|
| **B2** | GPU offload sweep | `make sweep-gpu` | [`12-sweep-gpu.log`](run-logs/12-sweep-gpu.log) | `benchmarks/bonus-gpu-offload-sweep.md` |
| **B3** | before/after của bonus | — | (như trên) | REFLECTION §6 |
| **B4** | Challenge **C2** — KV cache quantization | eval 10 prompt + đo memory + `llama-bench` | [`13-bonus-c2-kv-quant.log`](run-logs/13-bonus-c2-kv-quant.log), [`13b-bonus-c2-kv-memory.log`](run-logs/13b-bonus-c2-kv-memory.log), [`13c-bonus-c2-kv-bench.log`](run-logs/13c-bonus-c2-kv-bench.log) | `benchmarks/bonus-kv-cache-quant.md` |
| **B5** | Challenge **C9** — embedding serving | `make serve-embed` + `make embed-demo` | [`14a-serve-embed.log`](run-logs/14a-serve-embed.log), [`14b-bonus-c9-embed-demo.log`](run-logs/14b-bonus-c9-embed-demo.log) | `benchmarks/bonus-embedding-serving.md` |
| **B1** | build từ source | **chưa làm** — `cmake` không có trên máy | — | — |

Lệnh tái tạo phần C2 (không có trong Makefile — script nằm ngoài repo, các flag đã ghi
đầy đủ trong log):

```bash
# memory: so sánh physical footprint ở ctx lớn
llama-server -m models/gemma-4-E2B-it-UD-Q4_K_XL.gguf --ctx-size 32768 --parallel 4 -ngl 99 \
  [--cache-type-k q8_0 --cache-type-v q8_0]      # rồi: vmmap -summary <pid>

# latency: llama-bench, 3 repetition
llama-bench -m models/gemma-4-E2B-it-UD-Q4_K_XL.gguf -t 6 -ngl 99 -p 512 -n 128 -r 3
llama-bench -m models/gemma-4-E2B-it-UD-Q4_K_XL.gguf -t 6 -ngl 99 -ctk q8_0 -ctv q8_0 -p 512 -n 128 -r 3
```

---

## 5 screenshots

Tất cả đều là ảnh chụp **cửa sổ Terminal.app thật** trên chính máy này.

| # | File | Nội dung | Rubric |
|--:|:--|:--|:--|
| 1 | `screenshots/01-hardware-probe.png` | `make probe` chạy trực tiếp — CPU, cores, RAM, `GPU offload: ACTIVE`, llama.cpp build | 1 |
| 2 | `screenshots/02-bench.png` | output của `make bench` (từ `02-bench.log`) — bảng đủ 2 quantization, cột TTFT và TPOT tách riêng, `EXIT CODE: 0` | 3, 4 |
| 3 | `screenshots/03-serve-and-smoke.png` | socket `127.0.0.1:8080 (LISTEN)`, `/health` ok, một completion thật, và `tokens_predicted_total = 48 (non-zero)` — chạy trực tiếp | 6, 7 |
| 4 | `screenshots/04-locust-10.png` | summary locust 10 users (từ `06-load-10.log`) — `# reqs · Median · 95% · 99%` | 8 |
| 5 | `screenshots/05-locust-50.png` | summary locust 50 users (từ `07-load-50.log`) | 8 |

Ảnh 2, 4, 5 hiển thị **run log đã lưu** thay vì chạy lại lệnh, vì chạy lại `make bench` /
`make load-10` / `make load-50` sẽ ghi đè chính những artifact đang được chấm và làm số
liệu lệch khỏi REFLECTION. Nội dung trong ảnh là output nguyên văn của lần chạy thật,
kèm header lệnh và exit code.
