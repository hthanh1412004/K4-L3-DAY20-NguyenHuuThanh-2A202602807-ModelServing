# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 73 | 1.29 | 6100 | 11000 | 12000 | 8.5 | 0.0% |
| 50 | 86 | 1.48 | 27000 | 35000 | 36000 | 33.8 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.15x** (23% of linear) |
| P95 latency | **3.18x** |
| Effective concurrency at 50 users | 33.8 vs `--parallel 4` slots (occupancy/slot ratio 8.46) |

**Saturated.** Throughput delivered only 1.15x for 5x the offered load, and effective concurrency (33.8) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.15x while P95 moved 3.18x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Cách em đọc kết quả

Server đã bắt đầu bão hòa ở khoảng **10 users hoặc thấp hơn**: ngay tại 10 users,
effective concurrency đã là 8.5, vượt 4 slot. Khi tăng offered load 5 lần, throughput
chỉ tăng 1.15 lần nhưng P95 tăng 3.18 lần lên 35 giây; ở 50 users có 33.8 request trong
hệ thống và metrics ghi 46 request deferred. Nếu chọn SLO P95 dưới 12 giây thì lượt 10
users còn đạt, còn lượt 50 users không đạt nên goodput không tăng theo throughput. Em sẽ
giảm output-token budget/bucket request dài trước để rút thời gian giữ slot; chỉ tăng
`--parallel` có thể làm từng request chậm thêm vì các slot vẫn chia sẻ cùng bandwidth.
