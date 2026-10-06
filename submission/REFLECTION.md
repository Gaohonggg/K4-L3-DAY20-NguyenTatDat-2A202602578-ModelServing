# Reflection — Day 20 Lab (Personal Report)

**Họ Tên:** Nguyễn Tất Đạt

**MSSV:** 2A202602578

**Cohort:** K4 (theo tên repo bài tập)

**Ngày lập báo cáo:** 2026-10-06

## 1. Hardware & runtime

- **OS:** macOS, kernel Darwin 25.6.0, arm64.
- **CPU:** Apple M4 Pro; 12 physical / 12 logical cores theo hardware probe.
- **CPU extensions:** NEON.
- **RAM:** 24 GB unified memory.
- **Accelerator:** Apple Metal; runtime liệt kê MTL0: Apple M4 Pro.
- **Python:** 3.12.13 trong `.venv`, tạo và cài dependencies bằng uv 0.11.18.
- **Runtime:** llama.cpp b10488, commit 9d77fa172; asset `llama-b10488-bin-macos-arm64.tar.gz`.
- **Model:** Gemma 4 E2B, `LAB_MODEL=gemma4-e2b`, repo `unsloth/gemma-4-E2B-it-GGUF`.
- **Quantization:** UD-Q4_K_XL (primary), UD-Q2_K_XL (compare).
- **Nơi chạy:** laptop local của tôi.

**Setup story:** Tạo `.venv` bằng uv và cài dependencies từ requirements.txt. Runtime dùng prebuilt macOS arm64. Download qua Hugging Face Xet/CAS bị lỗi, sau đó tải GGUF bằng curl và tạo manifest từ file local. Runtime nhận Metal; benchmark load và phục vụ được cả hai quantization. Không dùng cloud fallback.

## 2. Đo lường

Nguồn: `benchmarks/01-quickstart-results.md` và `.json`. Baseline: 12 thread, ngl=99, tổng context 2048, parallel=4, max_tokens=64, temperature=0.7, reasoning off; bỏ request warm-up. Mỗi quantization hoàn thành 10/10 request.

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3075 | 83 / 220 | 12.5 / 13.0 | 864 / 996 / 996 | 80.0 |
| UD-Q2_K_XL | 2.24 | 2017 | 81 / 275 | 12.0 / 12.8 | 830 / 888 / 888 | 83.0 |

**Quan sát:** Q2 decode nhanh hơn 3,75%, nhỏ hơn 0,73 GB nhưng TTFT P95 cao hơn. Cùng prompt kiểm tra, Q4 giữ đủ “Nguyen An”, Q2 chỉ trả “Nguyen”; cả hai bọc JSON trong Markdown. Chọn Q4 vì đủ RAM và lợi ích tốc độ Q2 nhỏ. Đây chỉ là một kiểm tra chất lượng, chưa phải accuracy benchmark.

Kiểm tra chất lượng riêng dùng 6 thread, context 8192 / 4 slot, temperature=0, seed=42, max_tokens=256. Response đầy đủ ở `logs/15-quality-q4.json` và `logs/17-quality-q2.json`. Latency của kiểm tra này không thay thế baseline. Chỉ 10 mẫu/quantization nên percentile đuôi cần được đọc thận trọng.

## 3. Serving under load

Nguồn: hai Locust stats CSV, `02-server-results.md` và metrics CSV. Cấu hình: Q4, Metal ngl=99, 6 thread, tổng context 8192, 4 slot với 2048 token/slot, reasoning off. Workload mặc định 80% short / 20% long-RAG, max output tương ứng 48/96 token, temperature=0.5. Mỗi run đặt thời lượng 60 giây.

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|--:|
| 10 | 99 | 1.81 | 4400 | 7200 | 7700 | 8.1 | 0.0% |
| 50 | 113 | 1.91 | 21000 | 28000 | 47000 | 35.8 | 0.0% |

- **Số simulated users tăng:** 5×; đây là closed-loop, không đồng nghĩa arrival rate tăng 5×.
- **Throughput tăng:** 1,053×.
- **P95 tăng:** 3,89×.
- **Effective concurrency ở 50 users:** 35,8 so với 4 decode slots; gồm cả request chờ.
- **Peak gauge n_busy_slots_per_decode:** 3,94244 / 4; đây là gauge trung bình theo decode, không phải GPU utilization.
- **Peak requests_processing / requests_deferred:** 4 / 45.

**Saturation reading:** RPS gần plateau nhưng P95 tăng mạnh; metrics xác nhận 4 slot processing và 45 request chờ. Queueing góp phần tăng latency, chưa đo riêng queue time. Hai mức tải chưa xác định knee chính xác. SLO minh họa E2E P95 ≤ 10 giây: 10 users đạt, 50 không đạt. Knob thử tiếp là parallel=8, giữ context/slot, rồi đo lại; không mặc định throughput hay SLO sẽ tốt hơn.

Little's Law dùng RPS × latency trung bình: 1,81394 × 4,46438 ≈ 8,1 và 1,90996 × 18,75221 ≈ 35,8. Đây là ước lượng occupancy; run ngắn và request còn in-flight khi dừng giới hạn giả định steady state. Không tính được goodput@10s chính xác từ stats tổng hợp, và load test này chưa đo riêng TTFT/TPOT dưới concurrency.

**Giới hạn metrics:** 5 sample thành công và 10 lần scrape failed. Bốn sample cuối ghi nhận processing=4, deferred=43–45; gauge trung bình còn chịu ảnh hưởng lịch sử cùng server. Đủ quan sát batching tại các thời điểm lấy mẫu, chưa mô tả đầy đủ timeline. Runtime không export kv_cache_usage_ratio, nên báo n/a.

**Đối chiếu screenshot:** Bảng ở trên dùng stats CSV. Bảng cuối terminal trong ảnh 04 có 100 request thay vì 99; ảnh 05 có 115 request, RPS 1,93, P95 33000 ms và P99 53000 ms thay vì 113 / 1,91 / 28000 / 47000 của CSV. Locust ghi CSV định kỳ và in bảng cuối khi shutdown nên hai snapshot có thể khác nhau. Giữ nguyên dữ liệu, dùng CSV nhất quán cho phép tính và lưu logs/20-load-10.log cùng logs/22-load-50.log làm bằng chứng đối chiếu. Cả hai nguồn đều cho thấy saturation và 50 users không đạt SLO P95 ≤ 10 giây.

## 4. Integration

Nguồn: `benchmarks/03-integration-results.md` và `.json`, `logs/25-pipeline.log`.

| Day | Piece | Real hay stub? |
|:--|:--|:--|
| N16 Cloud/IaC | Localhost thay cho cluster/IaC | Stub |
| N17 Data pipeline | TOY_DOCS trong bộ nhớ, không có ingestion/orchestration | Stub |
| N18 Lakehouse | Python list/dict thay cho lakehouse | Stub |
| N19 Vector + features | Keyword overlap, không có vector index/feature store/embedding model | Stub |
| N20 Serving | Gemma Q4 qua llama-server b10488 / Metal | Real |

Pipeline hoàn thành cả ba query và in context IDs: `goodput, paged, radix`; `paged, radix, disagg`; `disagg, radix, batching`. Câu trả lời bám vào context phù hợp. Top-3 có cả document score=0 ở hai query đầu, cho thấy giới hạn keyword retrieval.

**Latency split trung bình của ba query:**

- embed: 0,0 ms (overhead nhánh không dùng embedding server, đã làm tròn).
- retrieve: 0,0 ms (làm tròn tới 0,1 ms, không phải thời gian thực bằng 0).
- llm: 489,1 ms.
- total: 489,1 ms.
- **Stage lớn nhất:** llm, xấp xỉ 100% sau làm tròn; gồm HTTP overhead, prefill và decode.

**Reflection:** LLM chiếm gần toàn bộ latency, phù hợp corpus sáu document và keyword retrieval. Muốn giảm tổng latency 2× cần thử giảm prompt/context budget, output budget hoặc tận dụng prefix reuse, rồi đo lại. Thread tuning khoảng 4% chưa đủ chứng minh mục tiêu 2×. Kết quả ba query tuần tự không đại diện RAG dưới tải.

## 5. The single change that mattered most

**Change:** Giảm CPU thread từ mặc định 12 xuống 6, giữ Q4 và Metal ngl=99 trong sweep tg128, ba repetition mỗi cấu hình.

```text
before:  81.87 tok/s (-t 12)
after:   85.27 tok/s (-t 6)
speedup: 1.042×, khoảng 4.15%
```

Nguồn: `benchmarks/01-tuning-tg128.json`. Chọn thay đổi này làm before/after của base vì nó giữ nguyên model, quantization, backend và workload; mức tăng decode của Q2 khoảng 3,75% còn đi kèm lỗi mất một phần tên trong kiểm tra chất lượng. Trong các điểm sweep 1/6/12/24 thread, 6 tốt nhất; 24 giảm còn 74,36 tok/s.

Phép đo offload model sang Metal, nên số thread CPU tối ưu không bắt buộc bằng số core vật lý. Thêm CPU thread không tăng số tài nguyên GPU; có thể tăng overhead lập lịch và tranh chấp phía host. Dạng curve gần phẳng ở 1–6 rồi giảm ở 12–24 phù hợp giả thuyết đó, nhưng chưa có profiling để xác định nguyên nhân duy nhất. Một sweep với chênh lệch khoảng 4% chưa chứng minh độ ổn định qua nhiều phiên đo. Đây là speedup tg128 của llama-bench, chưa phải speedup HTTP hoặc serving dưới load.

## 6. Bonus

**Đã thực hiện:** B2 batch-size sweep, before/after của B3 và B5/C9 embedding
serving thực qua HTTP. B1 và B4 chưa thực hiện; gói này cung cấp bằng chứng
cho ba tiêu chí bonus, mỗi tiêu chí 2 điểm theo rubric, điểm thực tế do grader chấm.

### B2/B3 — Micro-batch và prefill throughput

Nguồn: `benchmarks/bonus-batch-size-sweep.md`, `.json`,
`logs/27-bonus-batch-sweep.log`. Cùng Q4, Metal ngl=99, 6 thread, pp512 và
ba repetition/điểm. Giữ logical batch=512, thay micro-batch 256 → 512:

```text
before:  1166.99 tok/s (-b 512 -ub 256)
after:   1212.73 tok/s (-b 512 -ub 512)
speedup: 1.039×, khoảng 3.92%
```

Micro-batch lớn hơn có thể giảm số chunk/dispatch cho prompt 512 token.
512/512 là điểm tốt nhất đã thử; tăng logical batch lên 1024/2048 với ub=512
không tăng throughput. Điều này cho thấy knob phải khớp kích thước workload:
batch lớn hơn không tự động tốt hơn. Before là điểm sweep, không phải server
default. Chưa đo độ ổn định qua nhiều phiên, profiling, hoặc ảnh hưởng tới
TTFT/P95 dưới contention, nên chỉ kết luận về prefill throughput của sweep.

### B5/C9 — Embedding serving so với chat serving

Nguồn: `logs/28-embedding-server.log` và ba log
`logs/29-embedding-demo-1.log`, `logs/29-embedding-demo-2.log`,
`logs/29-embedding-demo-3.log`. Dùng cùng Gemma Q4 và Metal, 6 thread,
`-b 512 -ub 512`, context 8192, `--embedding --pooling mean` trên port 8081.
Startup log ghi 4 slot, n_ctx_slot=8192, kv_unified=true; cấu hình phân bổ
context này khác chat server dùng kv_unified=false / 2048 token mỗi slot.
Không suy diễn embedding mode không cấp phát KV memory chỉ từ việc không có
vòng autoregressive decode.

Endpoint trả vector 1536 chiều, corpus 8 document. Cả ba lần chạy đều xếp
document về embedding serving đứng đầu với cosine 0,847; các vị trí tiếp theo
là RadixAttention 0,784 và speculative decoding 0,751. Chỉ một query nên không
đánh giá được retrieval accuracy tổng quát. Đây là mean-pooled chat model,
không phải model embedding được train chuyên dụng; không có reranker thực
trong phép đo này.

Mỗi batch có ba phép đo tuần tự. Bảng dưới lấy median riêng của latency và
throughput được in trong ba log, không tính lại dữ liệu độ chính xác cao hơn
từ các số đã làm tròn:

| Texts/batch | Median batch latency (ms) | Median throughput (texts/s) | Throughput range (texts/s) |
|--:|--:|--:|:--|
| 1 | 57.3 | 17.5 | 17.2–18.3 |
| 2 | 61.3 | 32.6 | 32.2–33.3 |
| 4 | 83.0 | 48.2 | 47.4–49.2 |
| 8 | 193.1 | 41.4 | 40.1–41.5 |
| 16 | 376.4 | 42.5 | 41.6–42.6 |

Trong cả ba run, throughput cao nhất ở batch=4. So median batch=1 → 4,
throughput tăng 48,2 / 17,5 ≈ 2,75× trong khi batch latency tăng
83,0 / 57,3 ≈ 1,45×. Batch 8/16 chậm hơn về texts/s dù gom nhiều văn bản hơn.
Đây là knee của các điểm đã đo, không phải quy tắc batch tối ưu cho mọi corpus.
Runtime có 4 slot nên giới hạn song song phía server là một giả thuyết cần
thử thay slot count để kiểm chứng; hiện chưa tách được tác động của scheduling,
độ dài input và overhead phía client.

Embedding không sinh chuỗi output theo từng decode step như chat: gom nhiều
text trong một API call có thể amortize overhead và tăng throughput. Chat cần
giữ KV theo sequence, phục vụ prefill/decode và liên tục nhận/rút request khỏi
batch; tăng users ở base chỉ tăng RPS khoảng 1,05× nhưng P95 tăng 3,89× theo
CSV. Hai thí nghiệm minh họa mục tiêu batching khác nhau; không so trực tiếp
texts/s với tok/s hoặc coi batch latency embedding là chat TTFT.

Nếu chung autoscaler, cần theo dõi riêng queue, độ dài input, embedding batch
latency/texts throughput và chat TTFT/TPOT/SLO; một ngưỡng chung theo RPS có
thể không phản ánh hai workload. Chưa triển khai autoscaler trong lab.

Giới hạn: script thay thành phần văn bản theo batch size; batch=16 lặp lại
corpus 8 document. Các run có thứ tự cố định 1→2→4→8→16, chạy trên cùng server
và corpus được xử lý trước sweep, nên cache/warm-up có thể ảnh hưởng. Cần corpus
cân bằng độ dài, thứ tự random và đo token throughput nếu muốn tách riêng
tác động batching. Không đo FP8 hay so runtime khác; các teaching notes FP8
trong script không phải kết quả thực nghiệm của bài này.

## 7. Điều làm tôi ngạc nhiên nhất

Q2 nhỏ hơn khoảng 24,6% nhưng chỉ tăng decode khoảng 3,75%. Với server bốn slot, tăng từ 10 lên 50 users chủ yếu làm P95 tăng, trong khi RPS gần như không đổi.

## 8. Self-check trước khi push

- [ ] Hardware, manifest, report và CSV đã commit.
- [ ] Các ảnh chụp thật đã lưu trong submission/screenshots và commit.
- [x] Các report base không còn section required -- replace this line.
- [x] REFLECTION khai báo rõ real/stub và số liệu nguồn.
- [ ] Đã đọc lại và xác nhận hiểu các lập luận trong báo cáo.
- [ ] Xác nhận cohort K4 theo thông tin lớp thực tế.
- [x] Base và verify sau bonus đều đạt All checks passed (logs/26-verify-base.log, logs/30-verify-final.log).
- [ ] Repo GitHub đúng tên, public, commit cuối đã push.
- [ ] URL repo đã nộp LMS trước deadline thực tế do coach quy định.
- [x] Git không track GGUF, runtime hay .venv; các artifact đã stage đúng phạm vi bài tập.

## 9. Khai báo sử dụng AI

Dùng Codex để đọc yêu cầu, lập kế hoạch, hướng dẫn lệnh chạy, đọc log. Tôi tự chạy các phép đo trên máy local. Code lab hiện có được giữ nguyên. Các diễn giải cơ chế chưa có profiling được ghi là giả thuyết, và tôi cần đọc lại để xác nhận hiểu trước khi nộp.
