# Ghi chú phân tích · Day 04

Họ và tên: Trần Xuân Trường
MSSV: 2A202603617

## 1. Case được chọn

- ID case: H1
- Câu trả lời của agent: Ở `eval_before.json`, agent đọc đúng `h02n9572mx9y`, theo liên kết sang `xr3myzj7ms51` và đối chiếu với Cloudflare `yaa0witbl2me`, kết luận không có bằng chứng chung nguyên nhân và dẫn URL từng bên. Tuy nhiên, agent bỏ sót thông tin bản cập nhật của `xr3myzj7ms51` đã bị cắt ngắn.
- Điểm ở lần chạy 1: 97/100 (`factuality: 40`, `completeness: 23`, `evidence: 20`, `boundaries: 14`).

## 2. Bằng chứng trong trace

Trong `eval_before.json` ở case `H1`, `step 1` của `trace` gọi `get_incident_details` cho `xr3myzj7ms51` trả về dữ liệu bị cắt ngắn và thiếu các bản cập nhật trung gian, nhưng câu trả lời cuối không nêu rõ giới hạn này:

```text
step 1: get_incident_details(provider="github", incident_id="xr3myzj7ms51")
-> result.updates[0].truncated = true (bị cắt ngang tại "The code change")
-> result.updates_total = 9 (tool chỉ trả về 3/9 updates)
-> grade.reason: "Tuy nhiên, câu trả lời chưa lưu ý việc nội dung cập nhật của GitHub xr3myzj7ms51 bị cắt ngắn (truncated: true)."
```

Ngoài ra, trong trace của 4 case (`E2`, `E4`, `A7`, `H5`) ở `eval_before.json` đều xuất hiện `{"event": "validation_missing_source"}` do model quên kèm URL nguồn ở câu trả lời đầu, làm tốn thêm 1 lượt gọi model; và case `E1` (`99/100`) chưa phân biệt rõ mốc thời gian `resolved` với `updated_at`.

Nguyên nhân: `SYSTEM_PROMPT` ban đầu chưa chỉ dẫn rõ việc phải báo khi `truncated: true` hoặc `updates_total` lớn hơn số bản ghi trả về, chưa nhắc phân biệt mốc `resolved` với `updated_at`, và chưa nhấn mạnh việc dẫn đủ URL nguồn ngay từ lượt trả lời đầu tiên.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  - Cập nhật `SYSTEM_PROMPT` trong `agent.py` (dòng 6–12): bổ sung yêu cầu bắt buộc nêu rõ khi kết quả tool có `truncated: true` hoặc `updates_total` lớn hơn số bản cập nhật trả về; phân biệt mốc `resolved` với `updated_at`; và luôn dẫn đầy đủ URL nguồn (`url`/`source`) của từng incident đã tra cứu ngay trong câu trả lời đầu.
  - Cập nhật thông báo nhắc sửa ở `validation_missing_source` trong `agent.py` (dòng 118–123) để liệt kê rõ danh sách `observed_urls` còn thiếu.
- Vì sao chọn thay đổi này: Giải quyết trực tiếp lỗi thiếu ghi chú dữ liệu bị cắt ở `H1`, lỗi nhầm mốc thời gian ở `E1`, đồng thời giảm số bước gọi LLM thừa do `validation_missing_source` mà vẫn giữ nguyên contract của `run_agent()` và tương thích toàn bộ 37 unit tests chính cùng 3 bonus tests.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 99.82 / 100 (22/22 case) | 99.86 / 100 (22/22 case) |
| Điểm case đã chọn (`H1`) | 97 / 100 | 100 / 100 |
| Số model calls (agent) | 64 | 56 |
| Token, chi phí nếu có | Agent: 223,588 tokens; Judge: 105,387 tokens (endpoint chia sẻ, không tính phí trực tiếp) | Agent: 207,448 tokens; Judge: 116,375 tokens (endpoint chia sẻ, không tính phí trực tiếp) |

- Lỗi đã giải quyết:
  - Case `H1` tăng từ `97` lên `100/100` (nêu đầy đủ `truncated: true` và `3/9` bản cập nhật).
  - Case `E1` tăng từ `99` lên `100/100` (phân biệt rõ mốc `resolved` với `updated_at` và dẫn đủ URL nguồn).
  - Giảm số lần kích hoạt `validation_missing_source` từ `4` (`E2`, `E4`, `A7`, `H5`) xuống `0`, giúp tổng số lượt gọi model của agent giảm từ `64` xuống `56` (tiết kiệm `16,140` tokens phía agent).
- Lỗi còn lại hoặc mới xuất hiện:
  - Ở `eval_after.json`, case `A7` đạt `98/100` (judge góp ý chưa đối chiếu trực tiếp mốc thời gian lỗi tại Việt Nam với timeline `2026-11-07` và chưa nêu rõ trạng thái `degraded_performance` của component `Spectrum`).
  - Case `H7` đạt `99/100` (agent tự thêm link tham khảo `https://www.cloudflarestatus.com` dù tool Cloudflare bị `TimeoutError` không trả về URL đó).

## 5. Giới hạn

- **Giới hạn của điểm judge:** Bộ eval dùng LLM judge (cùng model `ag/gemini-3.8-flash` với agent), nên điểm số chỉ mang tính ước lượng, có thể dao động nhẹ giữa các lần chấm (như `A7` 100 → 98 và `H7` 100 → 99) và có nguy cơ thiên lệch khi cùng một model tự chấm.
- **Giới hạn của snapshot:** Dữ liệu snapshot bài chính được lưu cố định ngày `2026-11-09` (tối đa 50 incident gần nhất và tối đa 3 updates mỗi incident), không phản ánh trạng thái thực tế hiện tại hay toàn bộ lịch sử lâu hơn.
- **Giới hạn của việc chạy một lần mỗi phiên bản:** Mỗi phiên bản chỉ chạy 1 lần nên chưa chứng minh độ ổn định tuyệt đối trước tính ngẫu nhiên của LLM.

## 6. Ghi chú phần Bonus (`eval_bonus.json`)

- **Kết quả:** `99.7 / 100` (10/10 case `scored`; `B1`–`B9`: 100, `B10`: 97), quy đổi `bonus_points_0_15 = 14.96 / 15`.
- **Retry (429 / lỗi định dạng):** `agent.py` trả lỗi `RATE_LIMITED` và `INVALID_ARGUMENTS` về cho model như observation với đúng `tool_call_id`. Nhờ `BonusTools.SYSTEM_PROMPT` và `max_steps=16`, model tự retry thành công khi gặp 429 một lần (`B3`, `B10`), tự sửa kiểu tham số `offset`/`limit` (`B5`), và dừng đúng giới hạn 3 lần gọi khi bị 429 liên tục (`B4`).
- **Pagination:** Với timeline dài (`B2` — Vercel `sps6n39gm4sg`), agent đọc `next_offset` để gọi tiếp các trang sau (`offset=3`, `offset=6`) đến khi `next_offset: null` rồi mới kết luận.
- **Context (dữ liệu nhiễu):** Khi gặp 160 mục nhiễu giả lập (`B6`), model vẫn lọc đúng dữ liệu thật và bỏ qua nhiễu.
- **Giới hạn còn lại:** Chưa tự động cắt bớt mảng `distractors` trước khi đưa vào `messages` (nên prompt ở `B6` tốn nhiều token hơn mức cần thiết), và chưa tự thực thi `sleep(retry_after)` ở tầng `agent.py` khi tool trả HTTP 429 thật ở chế độ `--mode live`.
