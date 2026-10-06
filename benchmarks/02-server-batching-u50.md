# 02 - Continuous batching under load (u50)

Host `Darwin-arm64` · `--parallel 4` · 5 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.94 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 11915 |

Highest sampled value was **3.94 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Nhận xét về batching và hàng đợi

Gauge `n_busy_slots_per_decode` đạt cao nhất 3,94244 trên 4 slot, đồng thời
`requests_processing` đạt 4 và `requests_deferred` đạt 45. Điều này chứng minh
các request được xử lý đồng thời trong các decode step và có request chờ khi
số slot không đủ. Giá trị 3,94 là mức cao nhất của gauge trung bình đã scrape,
không phải phép đo batch width tức thời hay phần trăm GPU utilization.

Effective concurrency khoảng 35,8 trong load report bao gồm request đang
compute và request đang chờ; busy slots chỉ phản ánh mức gộp request trong
decode. Hai đại lượng không cần bằng nhau và không mâu thuẫn. Dùng metrics
để xác nhận batching/queueing, và dùng RPS cùng latency để đánh giá trải
nghiệm dưới tải; không gọi tỷ số 35,8 / 4 là utilization.

Giới hạn: chỉ 5 sample thành công, 10 lần scrape failed. Sample đầu có
processing=0, deferred=0, nên giá trị busy slots lúc đó còn phản ánh lịch sử
decode trước tải 50 users. Bốn sample cuối ghi nhận processing=4 và
deferred=43–45, cung cấp bằng chứng trực tiếp của tải còn tồn tại tại các thời
điểm lấy mẫu. Gauge trung bình còn chịu ảnh hưởng các request trước đó trong
cùng vòng đời server. Không suy diễn rằng server duy trì cùng mức occupancy
trong toàn bộ 60 giây, và không xác định nguyên nhân scrape failed khi chưa
có kiểm tra riêng. `kv_cache_usage_ratio` không được runtime export, nên để
n/a thay vì ghi 0.
