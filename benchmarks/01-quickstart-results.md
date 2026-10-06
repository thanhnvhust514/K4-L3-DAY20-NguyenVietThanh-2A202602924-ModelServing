# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2814 | 264 / 340 | 10.3 / 12.9 | 898 / 1101 / 1101 | 96.9 |
| UD-Q2_K_XL | 0.39 | 1903 | 244 / 276 | 9.2 / 9.3 | 824 / 860 / 860 | 109.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.13x faster** than `Q4_K_M` here, for 0.11 GB less on disk.

## Measured comparison and quality check

Bản 2-bit đạt 109.1 so với 96.9 token/s (+12.6%), nhỏ hơn 0.11 GB (22%). Đã thử cùng 3 prompt: hai bản đều đúng định nghĩa và JSON, nhưng đều sai phép tính 17 × 23 + 19 = 410. Mẫu này chưa đủ để khẳng định chất lượng tương đương; xem 01-quality-comparison.md.
