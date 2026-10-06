# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 20.8 | 38% |
| 3 | 43.6 | 80% |
| 6 | 54.3 | 100% |
| 12 | 37.1 | 68% |
| 24 | 4.0 | 7% |

**Best**: `-t 6` at 54.3 tok/s
**Slowest tested**: `-t 24` at 4.0 tok/s (13.75x spread)
**Against the physical-core default** (`-t 6`, 54.3 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Giải thích của em

Knee nằm ở **6 thread**, đúng bằng 6 core vật lý. Từ 1 lên 3 rồi 6 thread, throughput
tăng từ 20.8 lên 43.6 và 54.3 tok/s vì các core còn làm thêm được việc hữu ích. Khi tăng
lên 12 thread, throughput giảm còn 37.1 tok/s; 24 thread chỉ còn 4.0 tok/s. Decode phải
đọc weights lặp lại nên nhanh chóng bị giới hạn bởi memory bandwidth. Các logical thread
không tạo thêm memory channel mà còn tranh cache, bandwidth và tạo scheduling overhead,
vì vậy oversubscribe làm kết quả giảm mạnh.
