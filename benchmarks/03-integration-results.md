# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 2726.8 | 2726.8 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 1727.1 | 1727.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 3129.0 | 3129.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **2527.6** · total **2527.7**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it specifically accounts for **SLOs** (Service Level Objects) by counting only requests per second that met their targets.

Raw throughput ignores SLOs, whereas Goodput counts only the requests per second that met the targets, making it more useful for monitoring and managing service performance relative to the S

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing key-value (KV) cache in non-contiguous pages.

By storing KV in non-contiguous pages, the model removes this wasted internal fragmentation, allowing for more efficient memory usage on GPUs.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill and decode are bound to different memory bandwidth or compute resources**, allowing the system to optimize performance by utilizing parallel processing where one component is memory-bound and the other is compute-bound.

Based on the context provided:
*   **Prefill** is described as **compute-bound**.
*   **Decode** is described as **memory-bandwid


## Các thành phần N16-N19 em đã dùng

- N16 Cloud/IaC: **stub**.
- N17 Data pipeline: **stub**.
- N18 Lakehouse: **stub**.
- N19 Vector + features: **stub**; pipeline dùng keyword overlap trên `TOY_DOCS`.
- N20 Serving: **real**, gọi `llama-server` qua endpoint OpenAI-compatible.

LLM là stage chiếm nhiều nhất với trung bình 2527.6 ms, gần 100% tổng thời gian. Kết
quả này đúng với kỳ vọng vì embed đang tắt và corpus đồ chơi làm retrieval gần như tức
thời. Nếu cần giảm latency pipeline xuống một nửa, em sẽ ưu tiên giảm số output token và
độ dài context/prompt ở stage LLM, vì tối ưu hai stage gần 0 ms không tạo ra khác biệt
đáng kể.
