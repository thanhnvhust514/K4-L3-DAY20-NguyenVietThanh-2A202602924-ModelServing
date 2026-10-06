# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 92.6 | 85% |
| 4 | 109.1 | 100% |
| 8 | 109.3 | 100% |
| 12 | 107.3 | 98% |
| 24 | 109.1 | 100% |

**Best**: `-t 8` at 109.3 tok/s
**Slowest tested**: `-t 1` at 92.6 tok/s (1.18x spread)
**Against the physical-core default** (`-t 8`, 109.3 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Giải thích thread sweep

Từ 1 lên 4 threads, decode tăng 92.64 → 109.09 token/s; sau đó gần như đi ngang. Mức 8 đạt 109.26, chỉ cao hơn 4 khoảng 0.16%. Mức 12 thấp hơn 8 khoảng 1.81%, còn 24 gần ngang 8. Điểm chuyển sang plateau nằm quanh 4 threads trong grid đã thử; chọn 8 để giữ default và cùng cấu hình benchmark.

Đây là lần chạy có GPU offload (`ngl=99`), nên tăng CPU threads không nhất thiết tăng tốc phần tính toán chính trên GPU. Plateau phù hợp với việc CPU parallelism không còn là giới hạn chính; overhead CPU, GPU bandwidth hoặc kernel cũng có thể giới hạn tốc độ. Chưa có profiler/repeated trials đủ để phân biệt các cơ chế. Không có bằng chứng cho một cú giảm mạnh do oversubscription ở 24 threads.

**Không có speedup so với default 8 threads: 1.00×.** Spread 1.18× là best so với 1 thread, không phải gain so với default. Tốc độ `tg128` của llama-bench cũng không đồng nhất với decode đo qua HTTP; không trộn hai phương pháp vào một before/after.
