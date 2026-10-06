# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Việt Thanh
**MSSV:** 2A202602924
**Cohort:** K4-L3 Track 2
**Ngày báo cáo:** 2026-10-06 (ngày hoàn thiện báo cáo; trạng thái nộp LMS xem §8)

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11
- **CPU:** 12th Gen Intel(R) Core(TM) i5-12450HX
- **Cores:** 8 physical / 12 logical
- **CPU extensions:** Chưa xác minh; probe trên Windows không thu thập CPU extensions
- **RAM:** 11.7 GB
- **Accelerator:** NVIDIA GeForce RTX 2050, 4096 MiB VRAM; runtime CUDA
- **llama.cpp asset đã tải:** llama-b10488-bin-win-cuda-12.4-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** `Q4_K_M` + `UD-Q2_K_XL` (từ `models/active.json`)

**Chạy ở đâu:** Laptop local
Không dùng cloud fallback.

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Setup local với Qwen3.5 0.8B. Windows PowerShell 5.1 đọc sai UTF-8 nên thêm BOM cho script. Pip gặp lỗi DNS; runtime ZIP tải thiếu nên thêm kiểm tra và retry. Server gặp lỗi đường dẫn có dấu cách nên dùng subprocess trên Windows, hoặc gọi binary bằng đường dẫn tương đối.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2814 | 264 / 340 | 10.3 / 12.9 | 898 / 1101 / 1101 | 96.9 |
| UD-Q2_K_XL | 0.39 | 1903 | 244 / 276 | 9.2 / 9.3 | 824 / 860 / 860 | 109.1 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Bản 2-bit đạt 109.1 so với 96.9 token/s (+12.6%), nhỏ hơn 0.11 GB (22%). Đã thử cùng 3 prompt: hai bản đều đúng định nghĩa và JSON, nhưng đều sai phép tính 17 × 23 + 19 = 410. Mẫu này chưa đủ để khẳng định chất lượng tương đương; xem 01-quality-comparison.md.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 2.58 | 2600 | 4200 | 6200 | 7.3 | 0.0% |
| 50 | 2.36 | 18000 | 20000 | 22000 | 38.9 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.92× (RPS giảm khoảng 8.4%)
- **P95 tăng:** 4.76×
- **Effective concurrency ở 50 users:** 38.9 so với `--parallel` = 4 slots

Ảnh probe chụp trước khi chọn Qwen nên còn recommendation Gemma; các lần chạy/report dùng Qwen. Ảnh Locust là summary cuối 153/141 requests, CSV là snapshot 150/138; bảng này lấy CSV không sửa tay.

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.90 / 4 slots; peak requests_deferred = 45

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

RPS giảm 2.58 → 2.36 khi users tăng 5×; P95 tăng 4.2 → 20 giây. Processing đạt 4, deferred tới 45 xác nhận queue. Hai mức tải chưa tìm chính xác knee. Với mục tiêu P95 E2E ≤ 5 giây, lượt 10 users đạt, 50 users không đạt. Thử giới hạn concurrency trước để giảm chờ; chưa đo goodput từng request.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost, không có cloud/IaC trong lần chạy này | stub |
| N17 Data pipeline | TOY_DOCS in-memory | stub |
| N18 Lakehouse | Danh sách tài liệu mẫu, không có lakehouse | stub |
| N19 Vector + features | Keyword overlap, không gọi embedding endpoint | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms (không gọi embedding endpoint)
- retrieve: 0.0 ms (giá trị trung bình được làm tròn tới 0.1 ms)
- llm: 4489.1 ms
- **stage chiếm nhiều nhất:** llm (xấp xỉ 100% của total 4489.3 ms)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM wall time chiếm gần 100% của total 4489.3 ms; retrieval trên 6 toy docs rất nhỏ. Ưu tiên giảm/đo lại LLM stage: giới hạn độ dài đáp án, bỏ context không liên quan và kiểm tra prefix reuse. Chưa chứng minh giảm 2×; wall time còn HTTP/scheduling, không chỉ decode.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** So sánh Q4_K_M → UD-Q2_K_XL, giữ threads=8, ngl=99, ctx=2048, max_tokens=64 như benchmark.

```
before:  96.9 token/s (Q4_K_M)
after:   109.1 token/s (UD-Q2_K_XL)
speedup: 1.13×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):



Đổi Q4_K_M sang UD-Q2_K_XL làm decode đo qua HTTP tăng từ 96.9 lên 109.1 token/s, khoảng 1.13×, và giảm dung lượng 0.11 GB. Ít byte weights hơn có thể giảm lưu lượng đọc bộ nhớ và cải thiện cache residency khi sinh token. Đây là giả thuyết phù hợp kết quả; chưa có profiler để kết luận bandwidth là nguyên nhân duy nhất. Kernel/dequantization và phần cứng GPU cũng ảnh hưởng.

Thread sweep có GPU offload cho plateau từ khoảng 4 threads; 8 threads chỉ hơn 4 khoảng 0.16% và không cải thiện so với default. Vì vậy chọn quantization làm before/after chính, không gọi spread 1.18× của tuning là gain so với default. Benchmark chỉ có 10 prompt mỗi bản, temperature 0.7, output có thể khác độ dài và page cache có thể ảnh hưởng lần chạy sau. Hai bản đều sai một phép tính trong diagnostic 3 prompt, nên cần kiểm tra chất lượng trên tác vụ thực trước khi chọn 2-bit để triển khai.

---

## 6. Bonus *(optional)*

Không làm bonus trong bài nộp này. Diagnostic 3 prompt ở 01-quality-comparison là phần kiểm tra chất lượng của base, không khai là bonus.

---

## 7. Điều làm ngạc nhiên

24 CPU threads gần ngang 8 threads trong workload đã offload lên GPU. Throughput của 50 users thấp hơn 10 users nhưng P95 cao hơn nhiều; tăng tải không đảm bảo phục vụ hữu ích hơn.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 8 screenshots trong `submission/screenshots/` (đủ 5 nhóm bắt buộc)
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [ ] Paste URL repo vào VinUni LMS và kiểm tra deadline của coach. Trạng thái push xem READINESS-CHECK.md; chưa thực hiện nộp LMS.
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Dùng Codex để đọc tài liệu, hướng dẫn chạy lab, debug lỗi encoding PowerShell, tải runtime ZIP và đường dẫn có dấu cách; tổng hợp thông tin máy và số liệu từ report. Các kết quả được đo trên laptop local. Codex hỗ trợ chạy diagnostic chất lượng, kiểm tra số liệu và soạn phần giải thích dựa trên bằng chứng, có ghi rõ giới hạn và giả thuyết. Người làm bài cần hiểu phần lập luận để trình bày với coach.
