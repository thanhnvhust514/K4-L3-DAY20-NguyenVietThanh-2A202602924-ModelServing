# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 5547.0 | 5547.4 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 3383.4 | 3383.5 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 4536.9 | 4536.9 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **4489.1** · total **4489.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput is more useful than raw throughput because it focuses exclusively on the **throughput of requests that met the TTF (Total Time to Function) and TPOT (Total Time to Process) targets**.

While raw throughput measures the total number of requests processed per second, it often ignores the critical SLOs (Service Level Objectives) that define the quality of the service. By excluding requests th

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory**, specifically by storing the Key-Value (KV) cache in non-contiguous pages.

This design allows the engine to avoid wasting most of the available GPU memory that would otherwise be consumed by the contiguous memory needed to store the cache.

**When does splitting prefill and decode help?**

> Based on the provided context, splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bandwidth-bound**.

The context explicitly states: "Disaggregated serving splits prefill and decode onto separate pools because prefill is compute-bound and decode is memory-bandwidth-bound."

This implies that the system uses these splits to optimize performance by leveraging the


## Real/stub và latency

| Day | Thành phần trong lần chạy | Trạng thái |
|---|---|---|
| N16 | HTTP localhost; không có cloud/IaC | stub |
| N17 | TOY_DOCS in-memory | stub |
| N18 | Danh sách 6 tài liệu mẫu; không có lakehouse | stub |
| N19 | Keyword overlap; không gọi embedding/vector index/feature store | stub |
| N20 | llama-server thật, Qwen Q4_K_M | real |

Ba query hoàn tất; mean LLM wall time là 4489.1 ms trên total 4489.3 ms (~100%). Embed không được gọi; retrieve được làm tròn còn 0.0 ms, không có nghĩa là không tốn thời gian. Kết quả phù hợp với retrieval trên chỉ 6 toy docs rất nhỏ, còn LLM phải xử lý prompt và sinh token.

Để giảm latency 2×, nên đo lại LLM stage và thử giới hạn độ dài đáp án, giảm context không liên quan hoặc tái sử dụng prefix. Chưa chứng minh một thay đổi nào đạt 2×. Wall time phía client lớn hơn tổng `prompt_ms+predicted_ms` phía server; còn HTTP/client initialization/scheduling/queue và chi phí khác, nên không gọi cả 4489.1 ms là decode compute.

Đáp án được giữ nguyên làm bằng chứng. Query đầu diễn giải sai TTFT/TPOT; query cuối gộp prefix reuse với disaggregated serving không chính xác. Pipeline chạy end-to-end không đồng nghĩa đáp án đều đúng. Dùng kết quả này làm diagnostic, không làm định nghĩa học thuật.
