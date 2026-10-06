# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=6` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 10009 | 195 / 220 | 19.1 / 19.5 | 1403 / 1416 / 1416 | 52.3 |
| UD-Q2_K_XL | 0.39 | 9737 | 262 / 274 | 18.9 / 19.4 | 1449 / 1484 / 1484 | 53.0 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `Q4_K_M` decode within 2% of each other here, for 0.11 GB difference on disk.

## Nhận xét của em

Q2 chỉ nhanh hơn khoảng **1.3%** về decode (53.0 so với 52.3 tok/s), trong khi dung
lượng giảm 0.11 GB nhưng TTFT P50 lại tăng từ 195 ms lên 262 ms. Khi hỏi cùng một câu
về TTFT/TPOT, Q4 còn hiểu sai thuật ngữ nhưng Q2 trả lời lạc hẳn sang máy in 3D. Vì vậy
trên máy này em chọn Q4: phần dung lượng tiết kiệm được không bù cho chất lượng giảm và
tốc độ gần như không đổi.
