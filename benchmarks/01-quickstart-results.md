# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Darwin-arm64` · llama.cpp `b10488`
Settings: `threads=12` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3075 | 83 / 220 | 12.5 / 13.0 | 864 / 996 / 996 | 80.0 |
| UD-Q2_K_XL | 2.24 | 2017 | 81 / 275 | 12.0 / 12.8 | 830 / 888 / 888 | 83.0 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.04x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Nhận xét về tốc độ và chất lượng

Trên Apple M4 Pro 24 GB, Q2 đạt 83,0 tok/s so với 80,0 tok/s của Q4:
nhanh hơn 3,75%, đồng thời nhỏ hơn 0,73 GB (khoảng 24,6%). TPOT P50
giảm từ 12,50 xuống 12,04 ms. Tuy nhiên TTFT P95 của Q2 là 275,1 ms,
cao hơn 219,7 ms của Q4 trong lần đo này; chưa thể kết luận Q2 nhanh hơn
ở mọi khía cạnh. Mỗi bản chỉ có 10 request, nên percentile đuôi còn hạn chế.

Để kiểm tra chất lượng, gửi cùng một prompt gồm phép tính, trích xuất dữ liệu
và giải thích latency tới từng server, với `temperature=0`, `seed=42`,
`max_tokens=256`, reasoning off, 6 thread và tổng context 8192 / 4 slot.
Đây là kiểm tra chất lượng riêng, khác cấu hình baseline 12 thread / context
2048 ở bảng trên. Câu trả lời đầy đủ được lưu trong
`logs/15-quality-q4.json` và `logs/17-quality-q2.json`.

| Tiêu chí quan sát | Q4 | Q2 |
|:--|:--|:--|
| Phép tính 17 × 23 | Đúng: 391 | Đúng: 391 |
| Tên khách hàng trong dữ liệu nguồn | Giữ đủ "Nguyen An" | Chỉ trả "Nguyen", làm mất một phần tên |
| Số lượng, đơn giá và tổng tiền | Đúng: 3, 25000 và 75000, kiểu số | Giá trị đúng nhưng biểu diễn bằng chuỗi |
| Giải thích P95 khi concurrency tăng | Nêu tranh chấp tài nguyên, ví dụ CPU/database mang tính tổng quát | Có nhắc queuing delay và tranh chấp tài nguyên |
| Yêu cầu chỉ trả JSON object | Vẫn bọc trong Markdown code fence | Vẫn bọc trong Markdown code fence |

Chọn Q4 cho serving tiếp theo: máy đủ RAM, lợi ích decode của Q2 trong baseline
khá nhỏ, còn kiểm tra này quan sát được Q2 làm mất thông tin tên khách hàng.
Không xem đây là benchmark accuracy tổng quát: chỉ có một prompt tổng hợp,
và Q4 cũng chưa tuân thủ hoàn toàn định dạng output. Các số latency trong
hai response kiểm tra chất lượng không thay thế bảng latency baseline.
