# 02 - Serve: load test + saturation reading

Host `Darwin-arm64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=8192` · `threads=6` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 99 | 1.81 | 4400 | 7200 | 7700 | 8.1 | 0.0% |
| 50 | 113 | 1.91 | 21000 | 28000 | 47000 | 35.8 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Số simulated users (closed-loop) | 5x |
| Throughput actually delivered | **1.05x** (21% of linear) |
| P95 latency | **3.89x** |
| Effective concurrency at 50 users | 35.8 vs `--parallel 4` slots (occupancy/slot ratio 8.95) |

## Phân tích saturation và SLO

Tăng từ 10 lên 50 users chỉ tăng RPS từ 1,8139 lên 1,9100 (1,053×), trong khi
P95 tăng từ 7200 lên 28000 ms (3,89×) và P99 từ 7700 lên 47000 ms. Không có
failure được ghi nhận trong 99 và 113 request hoàn thành. Đây là bằng chứng
throughput gần plateau ở hai mức tải đã thử, còn latency tăng mạnh.

Effective concurrency ước lượng bằng Little's Law là 8,1 ở 10 users và 35,8
ở 50 users, đều lớn hơn 4 slot. Metrics ở tải 50 users xác nhận tối đa
4 request processing, 45 deferred, và gauge busy slots đạt 3,94244 / 4.
Các bằng chứng này phù hợp với việc hàng đợi là thành phần đáng kể của latency
ở tải cao. Không thể lấy riêng Little's Law hoặc chênh lệch P95 để tính queue
time chính xác: batch compute cũng có thể thay đổi theo concurrency.

Server đã có dấu hiệu queueing ở 10 users và bị giới hạn năng lực rõ ở 50
users. Hai điểm tải chưa xác định được số user tại knee chính xác; muốn tìm
ngưỡng cần đo thêm các mức thấp hơn và trung gian. Locust là closed-loop với
think time: tăng số user 5× không đồng nghĩa arrival rate tăng 5×. Các run
60 giây và request chưa hoàn thành khi dừng cũng giới hạn diễn giải percentile
và giả định trạng thái ổn định của Little's Law.

Chọn SLO minh họa **E2E P95 ≤ 10 giây** cho workload hỗn hợp này: 10 users đạt
SLO (7,2 giây), 50 users không đạt (28 giây). Đây là SLO E2E; load test hiện
không đo riêng TTFT/TPOT dưới tải. Không báo một con số goodput@10s chính xác
vì CSV tổng hợp không cung cấp số request hoàn thành trong 10 giây. Dữ liệu
cho thấy RPS cao hơn chưa đảm bảo chất lượng trải nghiệm tốt hơn.

Knob thử tiếp theo để tăng năng lực là `--parallel 4 → 8`, với tổng context
16384 để giữ 2048 token/slot. Đây là giả thuyết cần đo lại RPS, P95 và memory:
batch lớn hơn có thể tận dụng GPU tốt hơn nhưng cũng tăng chi phí mỗi decode
step và không bảo đảm cải thiện SLO. Giới hạn concurrency/queue là cách cần
đánh giá riêng nếu mục tiêu là bảo vệ latency khi tải vượt năng lực.

Metrics chỉ có 5 sample thành công và 10 lần scrape failed; phần này có bằng
chứng hoạt động batching nhưng không mô tả đầy đủ timeline 60 giây.

## Đối chiếu CSV và bảng cuối terminal

Bảng chính và các phép tính của report lấy từ `locust-10_stats.csv` và
`locust-50_stats.csv`. Bảng cuối terminal trong logs và screenshots là một
snapshot muộn hơn:

| Users | Nguồn | Request count | RPS | P95 (ms) | P99 (ms) |
|--:|:--|--:|--:|--:|--:|
| 10 | Stats CSV | 99 | 1.81394 | 7200 | 7700 |
| 10 | Terminal cuối / screenshot 04 | 100 | 1.81 | 7200 | 7700 |
| 50 | Stats CSV | 113 | 1.90996 | 28000 | 47000 |
| 50 | Terminal cuối / screenshot 05 | 115 | 1.93 | 33000 | 53000 |

Locust ghi stats CSV định kỳ trong background, còn bảng cuối được in lúc
shutdown; mã Locust hiện cài không ghi lại một stats snapshot cuối đồng bộ
trước khi đóng file. Vì vậy hai nguồn có thể lệch vài request hoàn thành sát
thời điểm dừng, làm percentile đuôi thay đổi. Giữ nguyên cả dữ liệu CSV và
log terminal; không sửa tay số liệu để làm chúng khớp. Kết luận throughput
plateau và SLO P95 ≤ 10 giây không đạt ở 50 users giữ nguyên ở cả hai nguồn.
