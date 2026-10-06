# Reflection — Day 20 Lab (Personal Report)

**Họ tên:** [CẦN BẠN ĐIỀN]
**MSSV:** [CẦN BẠN ĐIỀN]
**Cohort:** [CẦN BẠN ĐIỀN]
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime

- **OS:** Ubuntu 26.04.1 LTS trên WSL2
- **CPU:** Intel Core i5-11400H @ 2.70 GHz
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2, AVX-512
- **RAM:** 7.6 GB
- **Accelerator:** NVIDIA GeForce GTX 1650 4 GB; lần đo này dùng `ngl=0` nên inference chạy CPU
- **llama.cpp asset:** `llama-b10488-bin-ubuntu-vulkan-x64.tar.gz`
- **Model:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) và UD-Q2_K_XL (compare)
- **Nơi chạy:** laptop cá nhân, không dùng Colab/Kaggle

Do máy có 7.6 GB RAM nên em chọn Qwen3.5 0.8B thay cho Gemma. Setup đã tải đủ hai
weights và runtime. Khi bắt đầu benchmark, runtime thiếu `libgomp.so.1`; em tải gói
`libgomp1` chính thức của Ubuntu, giải nén cục bộ và thêm đường dẫn thư viện lúc chạy,
không phải cài lại model.

---

## 2. Đo lường

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 10009 | 195 / 220 | 19.1 / 19.5 | 1403 / 1416 / 1416 | 52.3 |
| UD-Q2_K_XL | 0.39 | 9737 | 262 / 274 | 18.9 / 19.4 | 1449 / 1484 / 1484 | 53.0 |

Q2 chỉ tăng decode khoảng 1.3% và tiết kiệm 0.11 GB, nhưng TTFT P50 tăng 67 ms. Em đã
hỏi cùng một câu trên cả hai bản: Q4 vẫn hiểu sai thuật ngữ, còn Q2 trả lời lạc sang máy
in 3D. Vì vậy với máy này em thấy Q2 không đáng dùng; Q4 hợp lý hơn về chất lượng trong
khi tốc độ gần như tương đương.

---

## 3. Serving under load

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.29 | 6100 | 11000 | 12000 | 8.5 | 0.0% |
| 50 | 1.48 | 27000 | 35000 | 36000 | 33.8 | 0.0% |

- **Offered load tăng:** 5×
- **Throughput thực tăng:** 1.15×
- **P95 tăng:** 3.18×
- **Effective concurrency ở 50 users:** 33.8 so với `--parallel=4`
- **Peak `llamacpp:n_busy_slots_per_decode`:** 3.96/4 slots
- **Peak deferred:** 46 requests

Server đã bão hòa từ khoảng 10 users hoặc thấp hơn vì effective concurrency tại 10 users
đã là 8.5, vượt 4 slot. Lên 50 users, throughput chỉ tăng 15% nhưng P95 tăng từ 11 lên
35 giây; `busy_slots=3.96/4` và 46 request deferred xác nhận phần tăng thêm chủ yếu là
queue time. Với SLO P95 dưới 12 giây, lượt 10 users còn đạt còn lượt 50 users không đạt.
Em sẽ giảm output-token budget và tách request dài trước, vì tăng slot trên cùng memory
bandwidth có thể làm từng request chậm thêm.

---

## 4. Integration

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Không nối hệ thống ngoài | stub |
| N17 Data pipeline | Không nối pipeline dữ liệu ngoài | stub |
| N18 Lakehouse | Không dùng lakehouse | stub |
| N19 Vector + features | Keyword overlap trên `TOY_DOCS` | stub |
| N20 Serving | `llama-server` OpenAI-compatible | real |

Latency trung bình của 3 query:

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 2527.6 ms
- total: 2527.7 ms
- stage chiếm nhiều nhất: LLM, gần 100% total

Bottleneck đúng như em dự đoán là LLM vì pipeline đang dùng keyword retrieval trên
corpus nhỏ và không có embedding server. Nếu cần giảm latency 2×, em sẽ giảm output
token và độ dài context/prompt ở stage LLM; tối ưu embed hoặc retrieve đang gần 0 ms sẽ
không tạo khác biệt đáng kể.

---

## 5. The single change that mattered most

**Change:** giảm thread decode từ `-t 24` (oversubscribe) xuống `-t 6` (bằng số core vật lý).

```text
before:  4.0 tok/s với 24 threads
after:   54.3 tok/s với 6 threads
speedup: 13.75× theo kết quả sweep (các giá trị trong bảng đã làm tròn)
```

Điểm knee nằm đúng tại 6 physical cores. Từ 1 lên 3 rồi 6 thread, throughput tăng 20.8
→ 43.6 → 54.3 tok/s vì vẫn còn core vật lý để chia việc. Khi lên 12 logical threads,
throughput giảm còn 37.1 tok/s; 24 threads chỉ còn 4.0 tok/s.

Decode phải đọc lại weights cho từng token nên sớm chạm giới hạn memory bandwidth. SMT
không tạo thêm memory channel; các thread dư còn tranh cache/bandwidth và làm scheduler
tốn công chuyển ngữ cảnh. Vì vậy “nhiều thread hơn” không đồng nghĩa nhanh hơn. Trên máy
này, đặt thread bằng số core vật lý là lựa chọn hợp lý nhất.

---

## 6. Bonus

Em không làm bonus để ưu tiên hoàn thiện và giải thích đầy đủ base track.

---

## 7. Điều làm em ngạc nhiên nhất

Điều em bất ngờ nhất là 24 thread chậm hơn 6 thread rất nhiều, và Q2 nhỏ hơn nhưng gần
như không nhanh hơn Q4. Hai kết quả này cho thấy cấu hình serving phải đo trên đúng máy,
không thể chỉ suy ra từ số thread hoặc số bit.

---

## 8. Self-check trước khi push

- [x] Có `hardware.json` và `models/active.json`
- [x] Có benchmark hai quantization và thread sweep
- [x] Có load test 10/50 users, saturation report và metrics batching
- [x] Pipeline chạy đủ 3 query, khai báo real/stub rõ ràng
- [x] Không còn phần nhận xét bắt buộc chưa điền trong `benchmarks/*.md`
- [ ] Điền Họ tên, MSSV, cohort ở đầu file
- [ ] Tự chụp và thêm đủ 5 ảnh vào `submission/screenshots/`
- [ ] Commit các file, chạy `make verify`, push repo public và nộp URL lên LMS

---

## 9. Khai báo sử dụng AI

Em dùng Codex để đọc hướng dẫn, chạy các lệnh lab, hỗ trợ xử lý lỗi thiếu `libgomp`, đối
chiếu số liệu giữa các report và gợi ý cách diễn đạt. Các số trong bài đều được sinh từ
chính máy đã khai báo; không dùng AI để tạo số liệu hoặc screenshot giả.
