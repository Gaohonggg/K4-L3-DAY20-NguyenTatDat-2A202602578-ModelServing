# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **12 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 83.2 | 98% |
| 6 | 85.3 | 100% |
| 12 | 81.9 | 96% |
| 24 | 74.4 | 87% |

**Best**: `-t 6` at 85.3 tok/s
**Slowest tested**: `-t 24` at 74.4 tok/s (1.15x spread)
**Against the physical-core default** (`-t 12`, 81.9 tok/s): 1.04x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Phân tích kết quả

Trong lần sweep này, đỉnh nằm ở 6 thread: 85,27 tok/s, so với 81,87 tok/s
ở mặc định 12 thread, tương đương speedup 1,042× (khoảng 4,15%). Tăng lên
24 thread làm throughput giảm còn 74,36 tok/s. Chọn 6 thread cho bước serving;
đây là cấu hình tốt nhất trong các điểm đã thử, chưa phải tối ưu trên mọi workload.

Phép đo dùng Metal với `ngl=99`, nên không nên kỳ vọng số CPU thread tối ưu
phải bằng 12 core vật lý: phần tính toán của model được offload sang GPU,
còn CPU vẫn tham gia điều phối và xử lý phía host. Kết quả 1 và 6 thread
khá gần nhau, trong khi 24 thread vượt số core và chậm hơn rõ rệt. Hình dạng
này phù hợp với giả thuyết tăng thread không giúp phần GPU và có thể tăng
chi phí lập lịch hoặc tranh chấp tài nguyên phía CPU. Sweep chưa đo bandwidth,
CPU utilization hay độ biến thiên từng repetition, nên chưa xác định được
nguyên nhân duy nhất; đặc biệt mức cải thiện khoảng 4% cần được hiểu trong
giới hạn của một lần sweep với ba repetition mỗi cấu hình.

`tg128` đo decode bằng llama-bench, khác với TTFT/TPOT qua HTTP và throughput
dưới concurrency. Không dùng trực tiếp speedup này để khẳng định serving
nhanh hơn 4,15%; các phép đo load ở bước sau sẽ đánh giá hành vi serving.
