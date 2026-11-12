# Ghi chú phân tích · Day 04

Họ và tên: Từ Huy Bảo
MSSV: 2A202604021

## 1. Case được chọn

- ID case: H1 — "Điều tra incident GitHub h02n9572mx9y. Theo liên kết do incident cung cấp... đối chiếu với incident Cloudflare yaa0witbl2me..."
- Câu trả lời của agent (lần chạy 1, `eval_after.json`): trình bày đúng h02n9572mx9y (resolved, Actions) và yaa0witbl2me (resolved, Workers Builds), kết luận không có bằng chứng chung nguyên nhân, nhưng chỉ *trích lại* URL của incident xr3myzj7ms51 trong update cuối ("Please refer to the combined summary in this related incident") mà không đọc incident đó, nên không nêu được nguyên nhân GitHub.
- Điểm ở lần chạy 1: 60 (judge raw 75, bị áp trần 60 do vi phạm kiểm tra bằng code).

## 2. Bằng chứng trong trace

Agent gọi cả hai incident trong cùng step 1 rồi trả lời ngay ở step 2; không có tool call nào ở step thứ hai.

```text
step 1  get_incident_details {"provider": "github", "incident_id": "h02n9572mx9y"}
step 1  get_incident_details {"provider": "cloudflare", "incident_id": "yaa0witbl2me"}
(step 2: final answer, không gọi tool)

grade.mechanical.violations:
  "Cần tool calls ở ít nhất 2 model steps khác nhau."   -> score_cap = 60

tool result của h02n9572mx9y, update "resolved":
  "Please refer to the combined summary in this related incident:
   https://www.githubstatus.com/incidents/xr3myzj7ms51"
```

Nguyên nhân có thể xảy ra: system prompt cũ đã yêu cầu "theo liên kết" bằng lời, nhưng không có cơ chế nào trong loop kiểm tra việc đó. Model (gpt-4o-mini) coi việc dẫn lại URL là đủ và kết thúc sớm; loop chấp nhận câu trả lời vì chỉ kiểm tra có URL nguồn hay không.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: chỉ sửa `agent.py`.
  1. Trong vòng xử lý tool result: quét các update của `get_incident_details` thành công bằng regex `.../incidents/<id>`; ID nào chưa được gọi sẽ được đưa vào `pending_links` và chèn thêm trường `agent_note` vào tool result gửi cho model ("update này dẫn tới incident X mà bạn CHƯA đọc, hãy gọi get_incident_details ở bước tiếp theo"). Trace ghi `notes` riêng, `result` giữ nguyên.
  2. Trước khi chấp nhận final answer: nếu `pending_links` còn phần tử, ghi event `validation_unfollowed_link` và yêu cầu model gọi tool đó rồi trả lời lại (tối đa 2 lần, `MAX_REVISIONS`).
  3. System prompt viết lại thành quy tắc đánh số; quy tắc 3 nêu rõ phải theo liên kết ở một bước riêng sau khi đã đọc nội dung chứa ID.
  4. Cùng lúc thêm hai gate khác cho các case lỗi còn lại: `validation_unverified_status` (tool timeout thì không được gán trạng thái, H7/A3) và `validation_current_state_without_status` (khẳng định trạng thái hiện tại thì phải có get_status, H2).
- Vì sao chọn thay đổi này: lỗi là lỗi *hành vi theo quy trình* (bỏ qua bước tra cứu) nên cần kiểm tra bằng code ngay tại thời điểm model nhìn thấy link, thay vì chỉ dặn trong prompt. Ghi chú trong tool result rẻ hơn gate cuối (không tốn một lượt trả lời bị loại), gate cuối là lưới an toàn.

## 4. So sánh hai lần chạy

Cấu hình giữ nguyên: provider openai, agent và judge đều gpt-4o-mini, max_steps 8, snapshot và dataset không đổi.

| Chỉ số | `eval_after.json` (trước khi sửa) | `eval_after_v2.json` (sau khi sửa) |
|---|---|---|
| Điểm tổng | 78.95 | 85.0 |
| Điểm case H1 | 60 (raw 75, trần 60) | 95 (không vi phạm) |
| Số model calls toàn bộ | 49 | 56 |
| Số model calls của H1 | 2 | 3 |
| Token agent (tổng 22 case) | ≈45.5k | ≈115k |

Trace H1 sau khi sửa:

```text
step 1  get_incident_details github/h02n9572mx9y     (notes: link tới xr3myzj7ms51 chưa đọc)
step 1  get_incident_details cloudflare/yaa0witbl2me
step 2  get_incident_details github/xr3myzj7ms51
step 3  final answer -> 2 tool steps, không còn vi phạm
```

- Lỗi đã giải quyết: H1 60→95 (theo liên kết), H2 50→100 (gọi thêm get_status), H7 35→75 (không còn kết luận Cloudflare operational khi timeout), E4 77→100, E5 75→95, H8 75→95.
- Lỗi còn lại hoặc mới xuất hiện: H5 75→45 với câu trả lời đúng nội dung (judge cho factuality 0, nhiều khả năng do judge bị nhiễu bởi notice injection trong trace); A7 75→65, H4 100→80 (judge trừ evidence cho bài thơ dù rubric nói không trừ); A3 bị gate `validation_unverified_status` bắn nhầm hai lần vì một câu gộp "GitHub bình thường, trong khi không có dữ liệu Cloudflare" — đã sửa sang kiểm tra theo mệnh đề nhưng chưa chạy lại được toàn bộ do lỗi mạng. Token tăng gấp đôi do prompt dài hơn và model đọc nhiều incident hơn.

## 5. Giới hạn

- Điểm judge là ước lượng của cùng model gpt-4o-mini tự chấm; cùng một câu trả lời đúng có thể nhận 75 ở lần này và 45 ở lần khác (H5). Chỉ H1/H2 có bằng chứng bằng code (mechanical cap), các dao động ±10 ở case khác không kết luận được là do agent.
- Snapshot cố định ngày 2026-11-09 với tối đa 50 incident/provider; mọi kết luận "operational" chỉ đúng tại thời điểm snapshot, và các gate của agent dựa trên so khớp chuỗi tiếng Việt/Anh nên có thể bắn nhầm hoặc bỏ sót với cách diễn đạt khác.
- Mỗi phiên bản chỉ chạy một lần; chênh lệch 78.95 → 85.0 chưa chứng minh độ ổn định. Cần chạy lặp vài lần hoặc đổi judge để kiểm chứng.
