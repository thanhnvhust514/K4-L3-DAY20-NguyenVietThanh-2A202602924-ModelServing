# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over

60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.90 of 4 slots (98%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 19849 |

Highest sampled value was **3.90 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means

requests were served one at a time -- either the load was too light to overlap, or

they arrived too far apart. A peak approaching `--parallel` means the scheduler was

genuinely packing concurrent requests into shared decode steps.

`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.


## Đối chiếu batching và queue

CSV có 14 mẫu; peak busy slots = 3.90032 (report làm tròn 3.90) trên 4 slots, processing đạt 4 và deferred đạt 45. Những mẫu processing=4, deferred>0 trong thời gian load là bằng chứng server xử lý đồng thời nhiều request và có request chờ.

38.9 effective concurrency từ Little's Law là số request trung bình trong toàn hệ thống, gồm cả chờ và xử lý; 3.90 là số slot trung bình mỗi decode step, lấy giá trị cao nhất trong các lần scrape. Hai số không đo cùng đại lượng và không cần bằng nhau. Dùng processing/deferred để xác nhận queue trực tiếp; dùng RPS×mean latency như ước lượng, có sai lệch khi test ngắn và chỉ tính request hoàn thành.

`n_busy_slots_per_decode` vẫn gần 3.89 ở hai mẫu cuối khi processing/deferred đã bằng 0; đó là gauge trung bình tích lũy, không phải batch tức thời. Khoảng cách timestamp giữa mẫu khoảng 4.35 giây dù interval sleep cấu hình là 2 giây, vì scrape cũng tốn thời gian. Tổng thời gian từ mẫu đầu tới cuối là 56.5 giây. KV usage không được export nên không báo giá trị giả 0.
