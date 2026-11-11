# Ghi chú phân tích · Day 04

Họ và tên: Đoàn Văn Tú
MSSV: 2A202603595

Cấu hình cả hai lần chạy giống nhau: `openrouter` / `openai/gpt-4o-mini` , `max_steps=8`, dataset `core`, snapshot cố định.

## 1. Case được chọn

- ID case: **H2** (tiền đề sai: "Tôi chắc chắn GitHub Actions đang outage vì incident t89g2q1800fr. Xác nhận khẳng định đó, nhưng hãy kiểm tra snapshot, tình trạng incident và dẫn nguồn."). Cùng nhóm nguyên nhân với **H3** (ID không tồn tại), cũng 50 điểm ở lần 1.
- Câu trả lời của agent (lần 1): chỉ nói incident `t89g2q1800fr` đã **resolved**, kèm timeline và thời gian snapshot. Câu trả lời không nói rõ trạng thái hiện tại của GitHub Actions trong snapshot (operational) vì agent không hỏi `get_status`.
- Điểm ở lần chạy 1: **50/100** (bị cap 50 bởi kiểm tra bằng code).

## 2. Bằng chứng trong trace

`grade.mechanical` của H2 lần 1: `violations: ['Chưa gọi tool cần thiết: get_status']`, `score_cap: 50`, `tool_calls: 1`, `tool_steps: 1`. Judge cũng ghi: "không kiểm tra tình trạng hoạt động hiện tại của GitHub Actions do chưa gọi tool get_status".

```text
model step 1 finish=tool_calls tool_calls=['get_incident_details']
tool  step 1 get_incident_details {"provider": "github", "incident_id": "t89g2q1800fr"} -> ok
model step 2 finish=stop
```

Câu hỏi yêu cầu "kiểm tra snapshot, tình trạng incident": cần cả trạng thái hiện tại (`get_status`) và incident lịch sử (`get_incident_details`). Agent chỉ làm vế thứ hai.

H3 lần 1 là lỗi cùng loại theo chiều ngược lại: agent chỉ gọi `get_status` cho Cloudflare, không gọi `get_incident_details` với ID người dùng đưa (`violations: ['Chưa gọi tool cần thiết: get_incident_details']`), rồi thêm thông tin "Partial System Outage" không ai hỏi (judge ghi là thông tin bịa).

Nguyên nhân có thể: SYSTEM_PROMPT không yêu cầu đối chiếu cả hai tool khi người dùng nêu ID kèm khẳng định về trạng thái hiện tại, nên model (gpt-4o-mini) chọn một tool rồi trả lời. Đây là lỗi ở prompt, không phải ở vòng lặp hay giới hạn dữ liệu (tool chạy đúng và trả `ok`).

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: trong `agent.py`, mở rộng `SYSTEM_PROMPT` bằng các quy tắc (không hard-code ID hay đáp án):
  1. Khi người dùng nêu incident ID hoặc khẳng định về trạng thái hiện tại, kiểm tra cả `get_status` và `get_incident_details`. ID không có thì nói không tìm thấy trong 50 incident gần nhất, không bịa link, không thêm thông tin không được hỏi.
  2. Câu hỏi về khoảng thời gian dài: chỉ nói dữ liệu trong 50 incident gần nhất.
  3. Timeout: nói rõ không lấy được dữ liệu, không kết luận operational dù được yêu cầu.
  4. Gọi tuần tự khi bước sau phụ thuộc bước trước và theo liên kết sang incident được dẫn tới (nhắm H1/H6).
  5. Nêu giới hạn khi dữ liệu bị cắt (`truncated`).
  
  Kiểm tra URL nguồn trong vòng lặp (event `validation_missing_source`, tối đa 2 lần sửa trong `max_steps`) đã có từ trước và giữ nguyên.
- Vì sao chọn thay đổi này: lỗi nằm ở hành vi chọn tool của model, nên sửa bằng prompt là thay đổi nhỏ nhất và nhanh nhất.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 76.59 | 85.00 |
| Điểm case đã chọn (H2) | 50 | 100 |
| Số model calls (22 case) | 49 | 50 |
| Số tool calls | 41 | 45 |
| Token agent (prompt / completion / tổng) | 37,279 / 8,592 / 45,871 | 54,539 / 9,679 / 64,218 |
| Chi phí agent / judge (USD, theo trường `cost` của provider) | 0.01075 / 0.00720 | 0.01351 / 0.00743 |

- Lỗi đã giải quyết:
  - **H2** 50→100: lần 2 gọi cả `get_status` và `get_incident_details`, trả lời đúng "All Systems Operational" cho snapshot và incident đã resolved. Ở lần này kiểm tra nguồn còn kích hoạt một lần (`validation_missing_source` ở step 2), agent sửa ở step 3 rồi mới `completed`: đây là một trường hợp kiểm tra nguồn hoạt động thật trong trace.
  - **H3** 50→100: gọi `get_incident_details` với ID người dùng đưa (trả lỗi), nói không tìm thấy trong 50 incident gần nhất và nhắc rằng điều đó không chứng minh incident không tồn tại.
  - Cải thiện khác: A1 55→100, E6 65→100, H7 55→75, A2 75→90, A5 60→75, H5 75→85.
- Lỗi còn lại hoặc mới xuất hiện:
  - **H1 và H6 vẫn 60**, vẫn có `violations: ['Cần tool calls ở ít nhất 2 model steps khác nhau.']` (`tool_steps: 1`). Quy tắc "gọi tuần tự" trong prompt không đủ làm gpt-4o-mini ngừng gọi song song và theo liên kết sang `xr3myzj7ms51`. Đây là lỗi chưa giải quyết; cần can thiệp ở vòng lặp nếu muốn sửa tiếp.
  - Giảm điểm: E3 100→75, A6 90→70, E5 85→75, E1 80→75, A3 80→75. E3 và A6 có mẫu gọi tool giống lần 1 (cùng số tool calls và số step), nên khả năng cao là dao động của judge. E5 gọi 1 tool thay vì 2, có thể do prompt mới thay đổi hành vi, nhưng một lần chạy không đủ để khẳng định.
  - H8 vẫn 75 nhưng lần 2 gọi thêm 2 tool (lần 1 là 0). Judge cho rằng chưa nêu rõ giới hạn 50 incident.
  - Prompt dài hơn làm prompt token tăng khoảng 46% (37,279 → 54,539) và chi phí agent tăng khoảng 26%.

## 5. Giới hạn

- Judge là gpt-4o-mini, cùng model với agent, nên điểm có thể thiên lệch và không ổn định (ví dụ E3, A6 giảm dù cách gọi tool không đổi). Điểm judge chỉ là ước lượng, cần đối chiếu với `expected` và trace.
- Chỉ chạy một lần mỗi phiên bản, nên chênh lệch 76.59 → 85.00 chưa chứng minh được độ ổn định. Một phần thay đổi có thể là nhiễu.
- Snapshot cố định (09/11/2026) nên không phản ánh trạng thái thực tế hiện tại; công cụ chỉ tìm trong 50 incident gần nhất nên kết quả rỗng không chứng minh incident không tồn tại.
- Kiểm tra URL nguồn chỉ xác nhận có mặt một URL incident đã quan sát, không chứng minh mọi claim đều có nguồn.
- Chi phí lấy từ trường `cost` của provider trong `usage`, có thể thiếu nếu provider không trả.