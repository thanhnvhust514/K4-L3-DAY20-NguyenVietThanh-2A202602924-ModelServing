# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 150 | 2.58 | 2600 | 4200 | 6200 | 7.3 | 0.0% |
| 50 | 138 | 2.36 | 18000 | 20000 | 22000 | 38.9 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.92x** (18% of linear) |
| P95 latency | **4.76x** |
| Effective concurrency at 50 users | 38.9 vs `--parallel 4` slots (occupancy/slot ratio 9.73) |

**Saturated.** Throughput delivered only 0.92x for 5x the offered load, and effective concurrency (38.9) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.92x while P95 moved 4.76x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Phân tích saturation và SLO

Khi users tăng 10 → 50, RPS giảm 2.576 → 2.359 (~8.4%), trong khi P95 tăng 4.2 → 20 giây (4.76×). Peak `requests_processing=4`, `requests_deferred=45` và busy slots 3.90/4 cho thấy các slot hoạt động và có hàng đợi. Máy đã có queue ở 10 users (Little's Law cho 7.3 request trong hệ thống so với 4 slots), và 50 users vượt rõ vùng tải hữu ích. Hai điểm đo chưa xác định chính xác knee; phần latency tăng không thể phân tách hoàn toàn thành queue/compute chỉ từ percentile.

Dùng mục tiêu tham khảo **P95 E2E ≤ 5 giây** cho workload hỗn hợp này: 10 users đạt mục tiêu với khoảng 2.58 request/s và 0 lỗi; 50 users không đạt dù throughput vẫn khoảng 2.36 request/s. Đây là kiểm tra SLO ở cấp lượt chạy, **không phải phép đo goodput chính xác theo từng request**. CSV tổng hợp không đủ để tính số request dưới 5 giây hoặc SLO riêng TTFT/TPOT.

Knob thử trước là giới hạn concurrency được nhận/admission control để giảm request phải chờ, giữ cấu hình 4 slots và thử vài mức thấp hơn hoặc quanh 10 users. Sau đó mới thử tăng `--parallel`, đồng thời theo dõi P95 và KV/context vì thêm slot không tạo thêm GPU bandwidth. Với client được kiểm soát, giảm users là cách thử giới hạn này; admission control phía server chưa được triển khai trong lab.

### Ảnh và CSV là hai snapshot

Ảnh summary cuối có 153/141 request (10/50 users), CSV có 150/138. Báo cáo và REFLECTION dùng CSV không chỉnh sửa. Locust ghi CSV định kỳ và in summary sau khi runner shutdown, nên các request hoàn thành cuối có thể xuất hiện trong ảnh nhưng chưa có ở snapshot CSV. P95/P99 dùng trong báo cáo là 4200/6200 ms và 20000/22000 ms.
