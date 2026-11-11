# Ghi chú phân tích · Day 04

Họ và tên: Hồ Nguyễn Minh Sang
MSSV: 2A202603905

## 1. Tình huống được chọn

E2 hỏi về incident Cloudflare cache purge và phạm vi ảnh hưởng đến khách hàng không thuộc FedRAMP. Điểm trước cải tiến: 85/100. Câu trả lời trước cải tiến dùng ký hiệu trích dẫn nội bộ thay vì URL incident đã quan sát. Xem đầy đủ câu trả lời trong `eval_before.json`, case E2.

## 2. Bằng chứng trong trace

Ở step 2, `get_incident_details` được gọi với `provider=cloudflare`, `incident_id=wp49bp8bqo6c`. Kết quả trả `ok=true` và URL `https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c`. Câu trả lời cuối không chứa URL incident đã đọc. Nguyên nhân có thể là prompt chưa yêu cầu URL thật rõ ràng và agent chấp nhận câu trả lời khi model dừng.

## 3. Thay đổi đã thực hiện

Trong `agent.py`, lưu URL từ `get_incident_details` thành công. Nếu câu trả lời thiếu mọi URL đã quan sát, ghi `validation_missing_source` và yêu cầu model sửa trong giới hạn `max_steps`. Không yêu cầu URL khi tool thất bại. Prompt cũng yêu cầu URL thật, kiểm tra status hiện tại và không suy ra trạng thái khi timeout.

## 4. So sánh hai lần chạy

| Chỉ số | Trước cải tiến | Sau cải tiến |
|---|---:|---:|
| Điểm tổng | 91.05 | 94.91 |
| Điểm E2 | 85 | 100 |
| Số model calls của agent | 64 | 62 |
| Token agent | 74763 | 77374 |
| Token judge | 54894 | 51537 |

Cùng Groq agent `openai/gpt-oss-120b`, judge `openai/gpt-oss-20b`, snapshot, dataset và `max_steps=8`. Provider không báo chi phí. E2 sau cải tiến có URL thật; trace không ghi `validation_missing_source`, nên không chứng minh nhánh sửa đã kích hoạt trong lần đó. H7 tăng 45 lên 85; A3 giảm 100 xuống 83. Xem answer và trace của từng case để đối chiếu.

## 5. Giới hạn

Judge có thể sai; một lần chạy mỗi phiên bản không chứng minh độ ổn định hay tác động riêng của từng thay đổi. Snapshot không đại diện trạng thái live và chỉ tìm trong 50 incident gần nhất. Kiểm tra nguồn chỉ yêu cầu một URL đã thấy, chưa xác minh mọi claim hay nguồn của từng provider.


## Bonus: kết quả và bằng chứng

Bộ bonus dùng Groq `qwen/qwen3.8-27b` cho cả agent và mô hình chấm, `max_steps=16`, `reasoning_effort=none`, `max_completion_tokens=2048`. Đủ 10/10 tình huống được chấm: **98.9/100**, quy đổi thành **14.84/15**. B1 đạt 95, B2 đạt 94; B3–B10 đạt 100. Không còn vi phạm kiểm tra bằng mã.

Cấu hình này khác lần bonus trước, nên mức tăng 84.9 lên 98.9 không chứng minh riêng hiệu quả của thay đổi mã nguồn. Dùng cùng mô hình để trả lời và chấm có thể gây thiên lệch.

| Tình huống | Bằng chứng trong nhật ký |
|---|---|
| B3 | Thử lại một lần sau `RATE_LIMITED` giả lập; báo trạng thái, thời điểm dữ liệu và nguồn. |
| B4 | Dừng sau ba lần Vercel bị giới hạn; vẫn báo kết quả GitHub và phần chưa biết. |
| B5 | Sửa tham số sau `INVALID_ARGUMENTS`, đọc cập nhật và dẫn nguồn. |
| B6 | Đọc chi tiết hai incident, nêu phạm vi, thời điểm và URL; `context_compacted` ghi nhận bỏ 160 mục nhiễu. |
| B10 | Kiểm tra cả năm nhà cung cấp, thử lại Vercel một lần, không suy luận chung nguyên nhân. |

Chỉ trường nhiễu `distractors` được lược bỏ khỏi ngữ cảnh và nhật ký, có ghi số mục bị bỏ. Dữ liệu trạng thái, incident, cập nhật, phân trang, nguồn và lỗi vẫn được giữ; fixtures không thay đổi. Agent kiểm tra đủ URL của incident đã đọc. Phân trang dựa trên `next_offset` khi cần; không tuyên bố đã đọc hết nếu chưa đủ bằng chứng.

Các lần lỗi tạo lời gọi công cụ từ nhà cung cấp được lưu tại `run_metadata.failed_attempts`. Chỉ phục hồi tình huống chưa chấm được; các tình huống đã có điểm được giữ nguyên.

Phản hồi còn lại của mô hình chấm: B1 chưa phân tích sâu incident Cloudflare; B2 trích thông báo chưa thật chính xác và có thể làm rõ thời điểm bản chụp hơn. Một lần chạy chưa chứng minh độ ổn định. Kiểm tra URL chưa xác minh mọi khẳng định; dữ liệu nhiễu bị bỏ được biểu diễn bằng số lượng.

### Kiểm chứng bổ sung và giới hạn của kết quả

Chỉ dẫn bài chính được bổ sung cách phân bổ lượt: kiểm tra đủ các dịch vụ trước, chọn incident tiêu biểu và dành một lượt cho câu trả lời cuối. Chạy riêng A1 với cấu hình Groq gốc hoàn thành và đạt 100. Cơ chế kiểm tra nguồn yêu cầu đủ mọi URL còn thiếu trong bonus; việc thử lại dừng ngay khi `retriable=false`, kể cả khi thời gian chờ bằng 0.

Kết quả so sánh thuộc hai lần chạy cùng cấu hình; các điều chỉnh bổ sung được kiểm tra bằng unit tests và chạy riêng A1, chưa đánh giá lại trên toàn bộ bộ tình huống.
Các kiểm tra ngoại tuyến trên phiên bản hiện tại đều đạt: 40 kiểm thử bài chính, 3 kiểm thử bonus có sẵn và 3 kiểm thử hồi quy cục bộ. Kiểm thử cục bộ bao phủ đủ nguồn incident, lược bỏ dữ liệu nhiễu và dừng thử lại khi `retriable=false`. Các kiểm thử này chưa thay thế đánh giá bằng mô hình thật.

Việc chờ HTTP 429 theo `Retry-After` và lưu tiến độ từng tình huống thuộc trình hỗ trợ đánh giá cục bộ. Bộ kết nối chuẩn hỗ trợ tùy chọn `GROQ_MAX_COMPLETION_TOKENS=2048` và `GROQ_REASONING_EFFORT=none`, nhưng không tự thử lại lỗi HTTP của nhà cung cấp; lỗi được báo về theo contract. Việc thử lại lỗi công cụ bên trong agent là cơ chế riêng.
