# Ghi chú phân tích · Day 04

Họ và tên: Nguyen Trieu Duong
MSSV: 2A202603524

## 1. Case được chọn

- ID case: H6 (đọc incident theo ID và theo liên kết khi kết quả có nhiều mục nhiễu).
- Câu trả lời ban đầu: agent nói không tìm thấy `h02n9572mx9y`, suy ra không thể đọc chi tiết và yêu cầu người dùng kiểm tra lại ID.
- Điểm ở lần chạy 1: 40/100. Điểm ở lần chạy 2: 100/100.

## 2. Bằng chứng trong trace

Các đoạn dưới đây được rút gọn từ trace thật, chỉ giữ step, tên tool và arguments.

```text
eval_before.json / H6:
step 1: get_status(provider="github", component="GitHub Actions")
step 2: model finish="stop"; không gọi get_incident_details.

eval_after.json / H6:
step 1: get_status(provider="github", component="Actions")
step 2: get_incident_details(provider="github", incident_id="h02n9572mx9y")
step 3: get_incident_details(provider="github", incident_id="xr3myzj7ms51")
```

Nguyên nhân có thể xảy ra: model nhầm danh sách tìm kiếm giới hạn là toàn bộ dữ liệu có thể truy cập. Bộ lọc là tìm chuỗi; "GitHub Actions" cũng khác "Actions". Agent dừng sau kết quả tìm kiếm, không thử tra ID trực tiếp và không theo liên kết đến incident gộp. Sau thay đổi, agent đọc được nguyên nhân liên quan cấu hình event-routing/feature flags và nêu phần nội dung bị cắt.

Nguồn trong câu trả lời sau: https://stspg.io/7pwv28hvkmz8 và https://stspg.io/pjgptg3cfc6n.

## 3. Thay đổi đã thực hiện

- Sửa `SYSTEM_PROMPT` trong `agent.py`: yêu cầu tra ID trực tiếp dù không thấy trong danh sách, dùng tên component ngắn, theo liên kết incident khi cần nguyên nhân/timeline, phân biệt snapshot với hiện tại và không mặc định `updated_at` là lúc giải quyết.
- Yêu cầu kiểm tra status bên cạnh incident khi xác minh trạng thái dịch vụ; không kết luận operational/outage từ timeout hoặc chiều theo yêu cầu bịa dữ liệu.
- Thêm kiểm tra URL nguồn: lưu URL từ get_incident_details thành công; nếu câu trả lời thiếu mọi URL đã quan sát, ghi `validation_missing_source` và yêu cầu model sửa trong cùng max_steps. Tool thất bại không tạo yêu cầu nguồn.
- Chọn thay đổi vì H6 thể hiện rõ lỗi chọn bước tiếp theo. Không hard-code ID, case hoặc câu trả lời trong agent; không sửa fixtures hay bộ chấm.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 92.5/100 | 97.5/100 |
| Case đã chấm | 22/22 | 22/22 |
| Điểm H6 | 40/100 | 100/100 |
| Điểm H2 | 50/100 | 100/100 |
| Điểm H7 | 45/100 | 45/100 |
| Số model calls của agent | 52 | 53 |
| Token của agent | 49,642 | 71,353 |
| Token của judge | 43,404 | 42,338 |
| Tổng token agent + judge | 93,046 | 113,691 |
| Tổng cost được báo cáo (USD), agent + judge | 0.007154244 | 0.009355241 |

- Cả hai lần dùng OpenRouter, agent/judge `google/gemma-4-26b-a4b-it`, core dataset, snapshot và max_steps=8. Cost là tổng trường `cost` do provider báo cáo, không cộng lặp các trường cost_details và không bao gồm các lần thử cấu hình trước đó.
- Lỗi đã giải quyết trong lần chạy quan sát: H6 đọc đủ chuỗi liên kết; H2 gọi cả get_status và get_incident_details. Điểm tổng tăng 5 điểm, với thêm 1 model call và chi phí báo cáo tăng khoảng 0.002201 USD.
- Lỗi còn lại: H7 vẫn khẳng định Cloudflare Spectrum operational sau TimeoutError theo yêu cầu sai của người dùng, dù prompt đã yêu cầu không bịa. Cải thiện prompt chưa đủ để kiểm soát hành vi này. Các case khác giữ 100; không ghi nhận case giảm điểm trong cặp chạy này.

## 5. Giới hạn

- Judge và agent dùng cùng model nên có thể có thiên lệch. Ví dụ judge lần đầu chấp nhận việc H6 không đọc chi tiết khi ID vắng trong kết quả tìm kiếm; trace cho thấy agent chưa thực sự thử tool đó. Vì vậy cần đọc evidence và mechanical checks thay vì chỉ tin lời giải thích của judge.
- Snapshot bài chính ghi nhận ngày 09/11/2026, không chứng minh trạng thái dịch vụ ở thời điểm thực tế hiện nay. Tool chỉ xét 50 incident gần nhất, danh sách status trả tối đa 5 incident, và chi tiết có thể thiếu update hoặc bị cắt.
- Mỗi phiên bản chỉ chạy một lần đầy đủ; chênh lệch điểm chưa chứng minh cải thiện ổn định hoặc tác động riêng của từng thay đổi.
- Kiểm tra URL chỉ chứng minh có ít nhất một URL đã quan sát trong câu trả lời, không xác minh mọi claim, mọi provider hoặc khả năng chống prompt injection. H7 là lỗi còn lại cần trình bày trong demo.
- 40 tests offline đã pass sau thay đổi; chúng kiểm tra contract và xử lý tình huống mô phỏng, không bảo đảm độ đúng của model ngoài thực tế. Không thực hiện phần bonus.
