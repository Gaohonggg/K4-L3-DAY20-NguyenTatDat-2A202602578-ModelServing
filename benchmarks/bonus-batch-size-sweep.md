# Bonus - Batch-size sweep (chunked prefill)

Host `Darwin-arm64` · llama.cpp `b10488` ·
`threads=6` `ngl=99` · metric `pp512`

| -b (logical) | -ub (micro) | pp512 (tok/s) | vs best |
|:--|--:|--:|--:|
| 128 | 128 | 1128.8 | 93% |
| 256 | 256 | 1171.0 | 97% |
| 512 | 256 | 1167.0 | 96% |
| 512 | 512 | 1212.7 | 100% |
| 1024 | 512 | 1202.5 | 99% |
| 2048 | 512 | 1201.5 | 99% |

Best: `-b 512 -ub 512` at 1212.7 tok/s
(1.07x the slowest point tested).

This sweep only measures the throughput half of the trade. The cost it hides is
TTFT for queued requests: a larger micro-batch holds the device longer per step,
so anything waiting behind it waits longer. To see both halves, re-run
`make load-50` with your best and worst settings via
`.venv/bin/python labs/02-serve/serve.py -- -b N -ub M` and compare P95.

## Finding và before/after

Sweep dùng Gemma Q4, Metal ngl=99, 6 thread, prompt 512 token và ba repetition
mỗi cấu hình. `-b 512 -ub 512` đạt 1212,73 tok/s, tốt nhất trong các điểm đã
thử. So với điểm 128/128 (1128,75 tok/s), mức tăng là khoảng 7,44%, nhưng
so sánh đó thay cả logical batch và micro-batch.

Để tách một knob cho B3, giữ `-b 512` và chỉ tăng `-ub 256 → 512`:

| Cấu hình | Prefill pp512 (tok/s) |
|:--|--:|
| Before: -b 512 -ub 256 | 1166.99 |
| After: -b 512 -ub 512 | 1212.73 |

Speedup = 1212,73 / 1166,99 ≈ **1,039× (3,92%)**. Before là một điểm
thí nghiệm trong sweep, không phải tuyên bố về default của server. Với prompt
512 token, micro-batch lớn hơn cho phép xử lý prompt trong ít chunk hơn,
có thể giảm overhead điều phối/dispatch. Đây là cơ chế phù hợp với kết quả,
chưa có profiling để chứng minh nguyên nhân duy nhất.

Giữ ub=512 rồi tăng b từ 512 lên 1024/2048 không đem lại throughput cao hơn
trong lần đo này. Prompt chỉ dài 512 token, nên phép đo cũng chưa khai thác
khả năng logical batch lớn hơn trên prompt dài hoặc nhiều sequence.

Chọn 512/512 làm ứng viên cho workload prefill này. Chưa khẳng định đây là
cấu hình production tốt nhất: cần đo TTFT, P95 E2E, RPS và memory dưới workload
hỗn hợp ở cùng concurrency/context. Sweep chỉ đo prefill throughput, không
đo decode, queueing hay chat goodput. Mức tăng 3,92% từ một sweep ba repetition
không đủ để khẳng định độ ổn định qua các phiên đo; không có kết quả per-repeat
trong JSON tổng hợp để ước lượng khoảng tin cậy.
