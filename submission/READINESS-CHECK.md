# Kiểm tra bài nộp — 2026-10-06

## Ảnh

Đã xem trực tiếp cả 8 ảnh trong `submission/screenshots/`. Không xóa hoặc thay thế ảnh gốc.
Các ảnh 03, 04, 05, 07, 08 được copy từ `C:/Users/ADMIN/Pictures/Screenshots/` vào repo.

| File | Bằng chứng quan sát được |
|---|---|
| 01-hardware-probe.png | Windows 11, i5-12450HX, 8 physical/12 logical cores, RAM 11.7 GB, RTX 2050 4096 MiB |
| 02-bench.png | Hai quantization Qwen, TTFT/TPOT/E2E/decode, đúng bảng report |
| 03-serve-and-smoke.png | Server xử lý request; completion thật; tokens_predicted_total = 86 và dòng OK |
| 04-locust-10.png | Summary cuối của load-10, request count/RPS/P50/P95/P99 |
| 05-locust-50.png | Summary cuối của load-50, request count/RPS/P50/P95/P99; bảng bị wrap nhưng vẫn đọc được |
| 06-tune.png | Thread sweep, best = 8, 109.3 token/s, 1.00× so với default |
| 07-batching.png | Peak busy slots = 3.90/4, processing = 4, deferred tới 45 |
| 08-pipeline.png | 3 query, context, đáp án, latency từng stage, mean LLM = 4489.1 ms |

Đủ cả 5 nhóm ảnh bắt buộc. Không cần chạy lại chỉ để bổ sung ảnh.

## Đối chiếu cần giữ rõ

- Ảnh probe được chụp trước khi chọn Qwen nên hiển thị recommendation Gemma.
  Model thực tế trong `models/active.json`, benchmark, tuning và integration là Qwen3.5 0.8B.
- Summary cuối trong ảnh load-10 có 153 requests, CSV có 150. Ảnh load-50 có 141 requests,
  CSV có 138. Report dùng CSV: RPS 2.58/2.36, P95 4200/20000 ms, P99 6200/22000 ms.
  Trong bản Locust cài local, CSV được ghi định kỳ; lúc shutdown, runner.quit() chạy trước
  khi in summary cuối, và close_files() chỉ đóng file CSV. Cơ chế này có thể giải thích
  vài request hoàn thành cuối chưa có trong snapshot CSV. Giữ nguyên số liệu; không sửa
  CSV cho khớp ảnh. Khi trình bày, nói rõ report lấy từ snapshot CSV còn ảnh là summary cuối.
- Completion của smoke có định nghĩa goodput sai. Smoke chứng minh endpoint và metrics
  hoạt động, không chứng minh tính đúng của nội dung. Trong pipeline, đáp án đầu cũng
  diễn giải sai tên viết tắt TTFT/TPOT. Không dùng các câu trả lời này làm định nghĩa học thuật.
- Thử chất lượng bổ sung lưu ở `benchmarks/01-quality-comparison.md` và `.json`:
  hai bản đều đúng ý về timing và xuất JSON đúng trong các prompt đã thử; đều sai phép tính.
  Đây là mẫu nhỏ, không chứng minh chất lượng tương đương.
- Thread tuning không cải thiện so với default 8 threads; không báo 1.18× là speedup
  so với default. 1.18× là best so với slowest tested (1 thread).

## Kết quả kiểm tra cuối

- Nội dung base hoàn chỉnh: REFLECTION, các report và 8 ảnh đã có; không còn placeholder ở phần bắt buộc.
- `powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\lab.ps1 verify`: exit 0 sau khi stage, và exit 0 sau commit `fc2f9ee`.
- Đã kiểm tra JSON/CSV load khớp nhau, model manifest khớp hai file benchmark, 20/20 baseline requests, thread best, các peak metrics và mean của 3 query RAG.
- Đã kiểm tra CRC của 8 PNG và đối chiếu hash các ảnh copy với ảnh gốc.
- Đã đối chiếu hash dữ liệu JSON/CSV và ảnh với backup: bản local không đổi. `.gitattributes` giữ nguyên byte dữ liệu đo khi commit, gồm cả newline gốc Windows; không sửa số liệu để chữa cảnh báo whitespace của Git.
- File `.env`, model weights, runtime và virtualenv không được track. Raw history/failures/exceptions CSV vẫn giữ trên máy nhưng được gitignore theo repo gốc.
- Repo tên đúng mẫu, visibility PUBLIC đã được xác minh bằng GitHub CLI.
- Đã commit kết quả base, checklist và quy tắc giữ nguyên byte dữ liệu. Bản clone sạch của commit `a4f820c`, không có model/runtime/venv, chạy `lab.ps1 verify` exit 0.
- Đang xuất bản các commit hoàn thiện lên origin/main; commit được xuất bản xem lịch sử Git. Chưa nộp URL vào LMS.
- Không cần chạy lại phép đo hoặc làm bonus để đủ artifact base. Backup report trước khi hoàn thiện giữ trong `runtime/submission-backups/20261006-finalize/`.

Các giới hạn bên trên vẫn cần giữ rõ trong báo cáo. `verify` pass xác nhận file và cấu trúc; không bảo đảm đáp án LLM đúng hoặc tự động bảo đảm điểm số.

## Giải thích để người làm bài tự hoàn thiện

- Saturation: RPS giảm khi users tăng 5×, P95 tăng 4.76×; deferred tới 45 và
  processing = 4 là bằng chứng có hàng đợi. Hai mức tải chưa tìm chính xác điểm knee.
- Batching: busy slots là số trung bình trên mỗi decode step; effective concurrency
  từ Little's Law gồm cả request chờ, nên 38.9 không phải 38.9 slot đang decode.
- Integration: LLM stage là toàn bộ HTTP wall time, không chỉ decode compute;
  keyword retrieval trên 6 toy docs rất nhỏ. N16–N19 stub, N20 real.
- Quantization: 2-bit giảm lượng weights cần đọc; speedup 1.13× phù hợp với giả thuyết
  bandwidth/cache, nhưng chưa có profiler để xác nhận. Chỉ 10 prompt mỗi quantization,
  có GPU offload và độ dài output khác nhau; không suy ra mọi workload đều nhanh hơn.

Các luận điểm trên là ghi chú kiểm tra dựa trên dữ liệu; người làm bài cần hiểu và
hoàn thiện phần diễn giải cá nhân theo docs/RULES.md.
