# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `nvidia_cuda, vulkan` ·
llama.cpp `b10488` · `threads=8` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 34.5 | 1.00x | 25% |
| 8 | 59.6 | 1.72x | 44% |
| 16 | 79.5 | 2.30x | 58% |
| 24 | 125.0 | 3.62x | 92% |
| 32 | 136.3 | 3.95x | 100% |
| 99 | 136.2 | 3.94x | 100% |

Best: `-ngl 32` at 136.3 tok/s
-- 3.95x faster than CPU-only.

## Thiết lập và bằng chứng

Lệnh: `.\lab.ps1 sweep-gpu`, dùng primary model Qwen3.5 0.8B Q4_K_M,
threads=8, runtime prebuilt b10488 CUDA trên RTX 2050 4 GB. Script mặc định
đo `tg128` bằng `llama-bench -p 0 -n 128 -r 2`, thay đổi duy nhất `-ngl`
qua 0, 8, 16, 24, 32, 99. Dữ liệu gốc: `bonus-gpu-offload-sweep.json`;
ảnh lần chạy: `../submission/screenshots/09-bonus-gpu.png`.
Header `nvidia_cuda, vulkan` là danh sách backend phần cứng phát hiện được,
không phải phép so sánh hai runtime CUDA/Vulkan.

## Your finding

CPU-only (`ngl=0`) đạt 34.54 token/s; `ngl=32` đạt 136.33 token/s,
tương đương 3.95×. Full-offload request (`ngl=99`) đạt 136.16 token/s,
tương đương 3.94× CPU-only. Offload nhiều lớp chuyển phần tính toán và
đọc weights sang GPU, phù hợp với mức tăng lớn ở lần đo này; chưa có
profiler để tách riêng đóng góp compute và bandwidth.

32 và 99 chỉ chênh 0.17 token/s (khoảng 0.12%). Với hai repetitions mỗi
cấu hình và không lưu độ lệch chuẩn, chưa đủ bằng chứng 32 tốt hơn 99.
Đường cong gần như phẳng ở hai mức cuối phù hợp với việc đã offload gần
hết công việc khả dụng. Không có số đo VRAM hay host-device traffic để
kết luận hết VRAM hoặc spill weights. Không gọi 32 là một tối ưu partial
offload đã được chứng minh. Có thể giữ ngl=99 cho cấu hình serving hiện tại.

Đây là native decode benchmark tg128, không phải throughput HTTP hay RPS
under load. Không so trực tiếp 136.33 với 96.9 token/s của baseline HTTP
hoặc 109.26 token/s của thread sweep ở một lần chạy khác. B2 có sweep
thật; B3 dùng before/after CPU-only → GPU trong chính lần sweep này.
