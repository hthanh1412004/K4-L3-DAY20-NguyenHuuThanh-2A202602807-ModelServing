# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 29 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.96 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 9321 |

Highest sampled value was **3.96 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Nhận xét của em

Peak `n_busy_slots_per_decode` là **3.96/4**, tức gần như cả bốn slot cùng tham gia mỗi
bước decode. `requests_processing=4` và `requests_deferred=46` cũng cho thấy server đã
kín slot và có hàng đợi. Effective concurrency 33.8 lớn hơn 4 không mâu thuẫn với gauge:
con số 33.8 tính cả request đang chờ theo Little's Law, còn 3.96 là mức sử dụng thực của
các decode slot. Vì vậy em dùng gauge để kết luận batching và dùng effective concurrency
để đọc độ dài hàng đợi.
