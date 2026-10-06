# Bonus C9 — Embedding serving so với chat serving

## Thiết lập và nguồn số liệu

Lần chạy trên laptop local đã khai trong hardware.json. Lệnh client:
`.\lab.ps1 embed-demo`; endpoint thật `http://localhost:8081/v1/embeddings`,
không dùng `--offline`. Demo trả vector 1024 chiều cho corpus 8 tài liệu.
Runner `serve-embed` dùng primary chat GGUF Qwen3.5 0.8B Q4_K_M với
`--embedding --pooling mean`, không phải embedding model chuyên dụng.

Số liệu dưới đây được chép từ ảnh gốc
[`10-bonus-embedding.png`](../submission/screenshots/10-bonus-embedding.png).
Script `bonus/serving-regimes/embedding-serving.py` chỉ in ra console,
không tự lưu JSON hoặc vector. Không có log thô hoặc cấu hình environment
của lần chạy để kiểm tra thêm; không dựng dữ liệu JSON như thể script đã xuất.

## Kết quả thật trong ảnh

| Batch size | HTTP wall time (ms) | Throughput hiển thị (texts/s) |
|--:|--:|--:|
| 1 | 2461.9 | 0.4 |
| 2 | 2635.0 | 0.8 |
| 4 | 2493.5 | 1.6 |
| 8 | 2666.1 | 3.0 |
| 16 | 2795.0 | 5.7 |

Mỗi batch có một request chứa danh sách input. Theo source, batch 16 lặp lại
corpus 8 tài liệu hai lần; batch không được token-sort. Chỉ một lần đo mỗi
batch, sau khi đã embed corpus và query; chưa có percentile hay confidence interval.

Query: “Does embedding serving use a KV cache and a decode loop like chat serving?”

| Rank | Cosine similarity | Tài liệu |
|--:|--:|:--|
| 1 | 0.890 | Embedding serving is prefill-bound: one forward pass, no KV cache, no decode loop. |
| 2 | 0.842 | RadixAttention reuses a shared prompt prefix across requests via a radix tree. |
| 3 | 0.811 | Speculative decoding drafts several tokens and verifies them in one forward pass. |

## Phân tích regime và giới hạn

Batch 1 → 16 làm HTTP wall time tăng khoảng 13.5% (2461.9 → 2795.0 ms),
nhưng số văn bản tăng 16×. Tính từ timing chưa làm tròn throughput,
`(16 / 2.7950) / (1 / 2.4619) = 14.09×`. Không dùng 5.7/0.4 để suy ra
speedup vì hai giá trị đã làm tròn một chữ số thập phân.

Embedding trả pooled vector sau xử lý input, không cần vòng sinh token
tự hồi quy hoặc KV cache được giữ qua các bước decode như chat. Static
batch input có thể chia sẻ overhead và khai thác GPU, nhưng phải giới hạn
tổng input tokens và thời gian chờ gom batch để bảo vệ latency. Demo này
không đo bộ nhớ nên không khẳng định runtime hoàn toàn không cấp phát buffer KV.

Ở chat track 02, tăng concurrent users 10 → 50 làm RPS 2.58 → 2.36 và
P95 4.2 → 20 giây. Busy slots gần 4 và deferred tới 45 cho thấy queue.
Chat có output dài ngắn khác nhau và nhiều decode steps: continuous batching
cho request vào/ra theo step giúp dùng slot; chỉ tăng tải vẫn làm hàng đợi dài.
Đây là so sánh hình dạng hai regime, không so trực tiếp texts/s với RPS,
và không gọi static batch size 16 tương đương 16 concurrent chat users.

Nếu dùng chung autoscaler, một ngưỡng RPS có thể bỏ sót chi phí khác nhau
của hai endpoint. Nên theo dõi riêng queue, input tokens và latency embedding;
với chat theo dõi thêm output tokens, active slots và TTFT/TPOT. Tách worker
pool hoặc quy tắc admission giúp tránh một batch embedding dài chặn chat.

Timing là toàn bộ HTTP wall time. Source tạo một `httpx.post` mới mỗi lần;
chi phí kết nối/client, scheduling hoặc prefix reuse có thể ảnh hưởng.
Chưa có server timing để kết luận toàn bộ khoảng 2.5 giây là prefill compute,
hoặc tốc độ tăng chỉ do GPU static batching. Muốn xác nhận cần nhiều repetitions,
client connection reuse, kiểm soát prefix cache và ghi server timing.

Top-1 đúng với query đã thử nhưng cosine của hai tài liệu khác chủ đề cũng cao.
Một query không chứng minh retrieval quality. Chat GGUF với mean pooling
không được tối ưu như sentence encoder chuyên dụng. Chưa đo reranker,
chưa thay keyword retrieval trong pipeline base bằng embeddings. C9 ở đây
là bằng chứng cho B5; không khai B4 vì rubric loại C9 khỏi nhóm B4.
