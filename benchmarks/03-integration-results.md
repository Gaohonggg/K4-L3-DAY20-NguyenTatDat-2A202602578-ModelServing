# 03 - Integrate: RAG pipeline run

Host `Darwin-arm64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 610.4 | 610.4 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 419.8 | 419.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 437.0 | 437.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **489.1** · total **489.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Thành phần real và stub

| Day | Thành phần trong lần chạy này | Trạng thái |
|:--|:--|:--|
| N16 Cloud/IaC | HTTP server trên localhost, không triển khai cluster/IaC | Stub cho hạ tầng N16 |
| N17 Data pipeline | Danh sách TOY_DOCS trong bộ nhớ, không có orchestration/ingestion job | Stub |
| N18 Lakehouse | Python dict/list thay cho lakehouse/table | Stub |
| N19 Vector + features | Keyword overlap, không có vector index/feature store hay embedding model | Stub |
| N20 Serving | Gemma 4 E2B Q4 qua llama-server b10488, Metal | Real |

Server dùng 6 thread, tổng context 8192, 4 slot (2048 token/slot), reasoning
off. Ba query trả lời đúng ý của context tương ứng: goodput, PagedAttention
và phân tách prefill/decode. Retrieval trả top-3, có cả document score=0 trong
hai query đầu; đây là giới hạn của keyword fallback, không phải bằng chứng
chất lượng semantic retrieval.

LLM latency trung bình là 489,1 ms, gần bằng total 489,1 ms sau làm tròn,
phù hợp với corpus chỉ có 6 document và retrieval đơn giản. `embed=0,0 ms`
là overhead nhánh không dùng embedding server; `retrieve=0,0 ms` là kết quả
làm tròn tới 0,1 ms, không khẳng định thời gian thực bằng 0. Stage `llm`
được đo ở client nên gồm HTTP overhead, prefill và decode, không chỉ compute.

Muốn giảm latency tổng khoảng 2×, cần tập trung vào stage LLM: đo ảnh hưởng
của context/prompt budget, độ dài câu trả lời và tái sử dụng prefix trước khi
chọn thay đổi. Riêng thread tuning chỉ cải thiện tg128 khoảng 4%, chưa đủ
chứng minh giảm pipeline latency 2×. Số đo này chỉ áp dụng cho ba query ngắn
chạy tuần tự, không đại diện RAG corpus lớn hoặc pipeline dưới load.
