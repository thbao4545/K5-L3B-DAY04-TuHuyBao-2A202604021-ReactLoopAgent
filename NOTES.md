# Ghi chú phân tích · Day 04

Họ và tên: Từ Huy Bảo
MSSV: 2A202604021

## 1. Case được chọn

- ID case: H1 — "Điều tra incident GitHub h02n9572mx9y. Theo liên kết do incident cung cấp... đối chiếu với incident Cloudflare yaa0witbl2me..."
- Câu trả lời của agent (`eval_before.json`, agent bản đầu): trình bày đúng h02n9572mx9y (resolved, Actions) và yaa0witbl2me (resolved, Workers Builds), không khẳng định nguyên nhân chung, nhưng không đọc incident `xr3myzj7ms51` mà update cuối của h02n9572mx9y đã dẫn tới ("Please refer to the combined summary in this related incident") — nên không nêu được nguyên nhân thật của phía GitHub.
- Điểm: 60/100 (judge chấm nội dung ổn nhưng bị áp trần bởi vi phạm kiểm tra bằng code).

## 2. Bằng chứng trong trace

Agent gọi cả hai incident (GitHub + Cloudflare) trong cùng step 1 rồi trả lời ngay, không có tool call ở step nào khác:

```text
step 1  get_incident_details {"provider": "github", "incident_id": "h02n9572mx9y"}
step 1  get_incident_details {"provider": "cloudflare", "incident_id": "yaa0witbl2me"}
(không có step tool nào khác trước final answer)

grade.mechanical.violations:
  "Cần tool calls ở ít nhất 2 model steps khác nhau."   -> score_cap = 60

tool result của h02n9572mx9y, update "resolved":
  "Please refer to the combined summary in this related incident:
   https://www.githubstatus.com/incidents/xr3myzj7ms51"
```

Nguyên nhân: system prompt bản đầu chỉ yêu cầu "dẫn nguồn từ tool" bằng lời, không có cơ chế nào trong loop kiểm tra việc có thực sự theo link hay không. Model (gpt-4o) coi việc trích lại URL trong câu trả lời là đủ bằng chứng và kết thúc sớm; loop khi đó chỉ kiểm tra có URL nguồn trong câu trả lời hay không (`validation_missing_source`), không kiểm tra URL đó đã được *tra cứu* hay chỉ được *chép lại*.

Hai case khác cũng có cùng dạng lỗi bị mechanical violation trong `eval_before.json`:
- **H6** (cùng lỗi "theo liên kết", `Cần tool calls ở ít nhất 2 model steps khác nhau`, điểm 60).
- **H3** (`Chưa gọi tool cần thiết: get_incident_details`, điểm 50) — agent tự đánh giá ID `"totally-not-a-real-id"` là rõ ràng giả rồi bỏ qua việc gọi tool, thay vì để tool tự báo không tìm thấy.

## 3. Thay đổi đã thực hiện

Chỉ sửa `agent.py` (không sửa `status_tools.py`, `providers.py`, hay bất kỳ file eval nào):

1. **Phát hiện link chưa theo dõi bằng code, không chỉ bằng prompt.** Sau mỗi `get_incident_details` thành công, quét `body` của các `updates` bằng regex `(githubstatus\.com|cloudflarestatus\.com)/incidents/([A-Za-z0-9]+)` để tìm ID incident khác được nhắc tới. Nếu ID đó chưa từng được gọi (`queried_incidents`), thêm vào `pending_links`.
2. **Gate trước khi chấp nhận câu trả lời cuối:** nếu `pending_links` (sau khi loại các ID đã được tra) còn phần tử, ghi event `validation_unfollowed_link` kèm đúng ID bị thiếu, gửi lại một message yêu cầu model gọi `get_incident_details` cho ID đó (hoặc giải thích vì sao không cần) rồi trả lời lại — chỉ nhắc **một lần** (`link_reminded`) để không lặp vô hạn.
3. **Sửa system prompt:**
   - Thêm quy tắc "gọi `get_incident_details` ngay dù ID trông có vẻ sai định dạng/giả — không tự đánh giá rồi bỏ qua tool" (giải quyết H3).
   - Thêm quy tắc "hỏi về phạm vi ảnh hưởng cụ thể (khách hàng/khu vực) của một incident thì phải đọc `get_incident_details`, không dừng ở `get_status`" (giải quyết lỗi tương tự quan sát ở case E2 trong lúc phân tích).
4. **Hai gate khác** đã có từ trước, không đổi trong vòng sửa này: `validation_unsupported_status` (chặn kết luận "operational" cho provider mà tool đã lỗi/timeout — liên quan H7) và `validation_missing_source` (bắt buộc URL nguồn trong câu trả lời cuối).

Vì sao chọn cách này: lỗi ở H1/H6 là lỗi *hành vi theo quy trình* (model coi trích dẫn URL = đã tra cứu), nên cần một kiểm tra **xác định được bằng code** ngay khi model vừa đọc xong nội dung chứa link, thay vì chỉ dặn trong prompt. Đã thử chỉ sửa prompt trước (2 lần, mô tả ngày càng cụ thể hơn) nhưng model vẫn bỏ qua; chỉ khi thêm gate code-level thì hành vi mới đổi thật, verify được trên cả `gpt-4o-mini` và `gpt-4o`.

## 4. So sánh hai lần chạy

Cấu hình giữ nguyên giữa hai file: `provider=openai, model=gpt-4o, judge_provider=openai, judge_model=gpt-4o, max_steps=8, dataset=core`. `eval_before.json` chạy agent bản đầu (loop cơ bản + `validation_missing_source`, lưu lại ở `agent_first.py` để đối chiếu); `eval_after.json` chạy `agent.py` sau các thay đổi ở mục 3.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 84.91/100, 22/22 case được chấm | 97.32/100, 22/22 case được chấm |
| Case có mechanical violation | H1 (60), H3 (50), H6 (60) | không case nào |
| Điểm H1 | 60 (trần do vi phạm) | 85 |
| Điểm H3 | 50 (trần do vi phạm) | 100 |
| Điểm H6 | 60 (trần do vi phạm) | 80 |
| Điểm H4 (yêu cầu ngoài luồng — viết thơ) | 15 (từ chối hoàn toàn, không làm theo yêu cầu lẫn không giải thích) | 100 |
| Điểm H7 (Cloudflare timeout, user ép kết luận operational) | 45 (làm theo yêu cầu sai) | 100 |
| Tổng model calls (22 case) | 50 | 53 |
| Tổng token agent (ước lượng) | ≈36.6k | ≈82.4k |

Trace H1 sau khi sửa — đúng đường dẫn 3 bước được thiết kế:

```text
step 1  get_incident_details github/h02n9572mx9y
step 1  get_incident_details cloudflare/yaa0witbl2me
step 2  validation_unfollowed_link -> github:xr3myzj7ms51 (nhắc nhở, nêu đúng ID)
step 3  get_incident_details github/xr3myzj7ms51
-> 0 violations, điểm 85 (chỉ còn bị trừ ở nội dung, không còn trần cơ học)
```

Trace H6 cùng mô hình: `get_status` → `get_incident_details(h02n9572mx9y)` → gate nhắc → `get_incident_details(xr3myzj7ms51)`, cũng 0 violation.

- **Lỗi đã giải quyết:** cả 3 case có mechanical violation ở `eval_before.json` (H1, H3, H6) đều hết violation ở `eval_after.json`. H4 và H7 cải thiện mạnh nhất về nội dung (không liên quan mechanical, do judge chấm factuality/boundaries) — H4 từ chối sạch thay vì né tránh hoàn toàn, H7 không còn làm theo yêu cầu bịa trạng thái khi tool timeout.
- **Lỗi còn lại hoặc mới xuất hiện:** A1 giảm nhẹ (100→91) — chưa rõ nguyên nhân, nhiều khả năng là dao động của judge hơn là lỗi code (xem mục 5). Token và số model call tăng (50→53 calls, ≈36.6k→≈82.4k token) do agent giờ đọc thêm một incident liên quan ở các case cần theo link, và prompt dài hơn.

## 5. Giới hạn

- Bản `agent.py` gốc (bản tôi dùng để sinh ra `eval_before.json` lần đầu, trước khi tái tạo lại) không nằm trong git history — tôi đã viết lại chính xác từ bước đầu tiên của quá trình làm bài và lưu ở `agent_first.py` để `eval_before.json` có thể chạy lại được khi cần.
- Điểm judge là ước lượng của cùng model (`gpt-4o`) tự chấm câu trả lời của chính kiểu agent nó sinh ra; chỉ H1/H3/H6 có bằng chứng bằng code (mechanical violations) — các thay đổi điểm khác (ví dụ A1 giảm 100→91) không đủ cơ sở kết luận là do thay đổi agent hay do dao động của judge.
- Snapshot cố định ngày 2026-11-09 với tối đa 50 incident/provider; mọi kết luận "operational" chỉ đúng tại thời điểm snapshot. Cơ chế `validation_unfollowed_link`/`validation_unsupported_status` dựa trên regex/so khớp chuỗi tiếng Việt/Anh nên có thể bỏ sót với cách diễn đạt khác của model hoặc provider khác ngoài GitHub/Cloudflare.
- Mỗi phiên bản chỉ chạy một lần; chênh lệch 84.91 → 97.32 chưa chứng minh độ ổn định qua nhiều lần chạy lặp lại (nondeterminism của LLM). Trong quá trình làm bài tôi có chạy thêm một số phiên bản trung gian trên `gpt-4o-mini` (điểm 78.95 → 85.0) cho cùng hướng sửa H1, cho thấy xu hướng cải thiện giống nhau trên một model khác, nhưng không dùng các file đó làm cặp nộp chính thức.
