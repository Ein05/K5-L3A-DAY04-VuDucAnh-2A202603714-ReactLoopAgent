# Ghi chú phân tích · Day 04

Họ và tên: Vũ Đức Anh
MSSV: 2A202603714

## 1. Case được chọn

- ID case: A1
- Câu hỏi: "Đối chiếu GitHub Actions và Cloudflare Spectrum: mỗi dịch vụ có incident liên quan nào trong dữ liệu, trạng thái và phạm vi của từng incident? Không suy luận chúng có cùng nguyên nhân; dẫn URL cho cả hai."
- Câu trả lời của agent: Ở lần chạy 1, agent đã dùng hết toàn bộ ngân sách 8 bước (`max_steps`) do gọi liên tục `get_incident_details` cho toàn bộ các incident trả về của GitHub và Cloudflare. Khi chạm ngưỡng 8 bước, agent dừng lại ở trạng thái `max_steps` mà không kịp sinh câu trả lời tổng hợp. Chuỗi trả về chỉ là kết quả JSON thô của tool call cuối cùng:
  `{"ok": true, "provider": "cloudflare", "captured_at": "2026-11-09T03:50:47.638054+00:00", "id": "c60k3g4vsoc1", "name": "Spectrum Availability Issue in Phoenix", "status": "monitoring", "created_at": "2026-11-07T00:59:31.548Z", "updated_at": "2026-11-07T02:51:59.805Z", "components": ["Spectrum"], "updates": [...], "url": "https://www.cloudflarestatus.com/incidents/c60k3g4vsoc1"}`
- Điểm ở lần chạy 1: 0/100 (Trạng thái: `max_steps`, LLM Judge nhận xét: "Agent hết số bước trước khi hoàn tất.").

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
[
  {
    "event": "tool",
    "step": 0,
    "name": "get_status",
    "arguments": "{\"component\":\"GitHub Actions\",\"provider\":\"github\"}",
    "result": { "provider": "github", "incidents": [ {"id": "t89g2q1800fr"}, {"id": "xr3myzj7ms51"}, {"id": "ntrfd2jqldn6"}, {"id": "q29p76qtnkyk"}, {"id": "8tzbpcnl75jp"} ], ... }
  },
  {
    "event": "tool",
    "step": 1,
    "name": "get_status",
    "arguments": "{\"component\":\"Spectrum\",\"provider\":\"cloudflare\"}",
    "result": { "provider": "cloudflare", "incidents": [ {"id": "c60k3g4vsoc1"} ], ... }
  },
  {
    "event": "tool",
    "step": 2,
    "name": "get_incident_details",
    "arguments": "{\"incident_id\":\"t89g2q1800fr\",\"provider\":\"github\"}",
    "result": { "ok": true, "id": "t89g2q1800fr", "name": "Incident with Actions", "status": "resolved", ... }
  },
  {
    "event": "tool",
    "step": 3,
    "name": "get_incident_details",
    "arguments": "{\"incident_id\":\"xr3myzj7ms51\",\"provider\":\"github\"}",
    "result": { "ok": true, "id": "xr3myzj7ms51", "name": "Incident with Actions and Pages", "status": "resolved", ... }
  },
  {
    "event": "tool",
    "step": 4,
    "name": "get_incident_details",
    "arguments": "{\"incident_id\":\"ntrfd2jqldn6\",\"provider\":\"github\"}",
    "result": { "ok": true, "id": "ntrfd2jqldn6", "name": "Delayed Actions status check results", "status": "resolved", ... }
  },
  {
    "event": "tool",
    "step": 5,
    "name": "get_incident_details",
    "arguments": "{\"incident_id\":\"q29p76qtnkyk\",\"provider\":\"github\"}",
    "result": { "ok": true, "id": "q29p76qtnkyk", "name": "Incident with Actions", "status": "resolved", ... }
  },
  {
    "event": "tool",
    "step": 6,
    "name": "get_incident_details",
    "arguments": "{\"incident_id\":\"8tzbpcnl75jp\",\"provider\":\"github\"}",
    "result": { "ok": true, "id": "8tzbpcnl75jp", "name": "Degraded availability for GitHub Actions, GitHub Pages and Webhooks", "status": "resolved", ... }
  },
  {
    "event": "tool",
    "step": 7,
    "name": "get_incident_details",
    "arguments": "{\"incident_id\":\"c60k3g4vsoc1\",\"provider\":\"cloudflare\"}",
    "result": { "ok": true, "id": "c60k3g4vsoc1", "name": "Spectrum Availability Issue in Phoenix", "status": "monitoring", ... }
  }
]
```

**Nguyên nhân có thể xảy ra:**
- Agent thiếu nhận thức về ngân sách bước gọi (budget awareness). Sau khi gọi `get_status` và nhận danh sách gồm 5 incidents từ GitHub và 1 incident từ Cloudflare, mô hình cố gắng đào sâu chi tiết bằng cách gọi `get_incident_details` cho toàn bộ từng incident một cách tuần tự và dàn trải.
- Với giới hạn `max_steps = 8`, chuỗi 2 lệnh status + 6 lệnh incident details đã chiếm trọn toàn bộ 8 lượt gọi (từ `step` 0 đến 7). Khi bước sang lượt thứ 8, vòng lặp `run_agent` phát hiện hết lượt và trả về kết quả dở dang `status: 'max_steps'`. Mô hình không còn bước nào để tổng hợp thông tin, so sánh hai dịch vụ và đưa ra câu trả lời cuối cùng.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. Trong `agent.py`: Cập nhật `SYSTEM_PROMPT` bổ sung hai quy tắc điều tra quan trọng:
     - *Quy tắc 1 (Tiết kiệm bước):* "Khi đối chiếu nhiều dịch vụ hoặc nhiều incident, chỉ gọi get_incident_details cho tối đa 1 incident tiêu biểu hoặc gần nhất của mỗi dịch vụ để đảm bảo đủ ngân sách bước tổng hợp kết luận."
     - *Quy tắc 2 (Giữ vững ranh giới bằng chứng):* "Nếu tool gặp lỗi (như timeout, lỗi mạng) hoặc thiếu dữ liệu, phải nêu rõ là bị lỗi hoặc thiếu dữ liệu. Tuyệt đối KHÔNG nghe theo yêu cầu của người dùng để giả định, bịa đặt hoặc khẳng định dịch vụ đang bình thường/operational khi thiếu bằng chứng."
  2. Trong `agent.py`: Triển khai cơ chế kiểm tra và bổ sung nguồn bắt buộc (Evidence Gate / Source Validation) theo Bài 4 và `tests/test_validation.py`:
     - Theo dõi và lưu trữ các URL từ các lệnh gọi `get_incident_details` thành công vào tập `seen_urls`.
     - Khi model dừng tạo tool calls (`finish == 'stop'`), kiểm tra xem câu trả lời có chứa ít nhất một URL trong `seen_urls` hay không.
     - Nếu thiếu URL nguồn, ghi sự kiện `validation_missing_source` vào `trace` và gửi lời nhắc `messages.append({'role': 'user', 'content': 'Hãy dẫn URL nguồn từ tool result vào câu trả lời.'})` để model bổ sung trong ngân sách `max_steps`.
- Vì sao chọn thay đổi này:
  - Giúp agent định hình chiến lược tra cứu thông minh và tiết kiệm bước: đối chiếu nhiều bên chỉ cần lấy mẫu incident tiêu biểu đại diện thay vì duyệt cạn toàn bộ, đảm bảo luôn dự phòng ít nhất 1-2 bước cuối để tổng hợp câu trả lời hoàn chỉnh.
  - Tăng tính trung thực, không bị prompt injection hoặc các chỉ dẫn sai lệch ép buộc đưa ra thông tin giả mạo (được kiểm chứng qua cải thiện điểm số ở case H7).
  - Đảm bảo câu trả lời luôn có nguồn kiểm chứng minh bạch, đáp ứng hợp đồng đánh giá của hệ thống chấm điểm và unit tests.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 90.14 / 100 | 92.14 / 100 |
| Điểm case đã chọn (A1) | 0 / 100 | 100 / 100 |
| Số model calls (ước tính) | 59 calls | 43 calls |
| Token, chi phí nếu có | 111,700 tokens (86,573 prompt + 25,127 completion) | 82,922 tokens (62,294 prompt + 20,628 completion) |

- Lỗi đã giải quyết:
  - **Case A1 (tăng từ 0 lên 100 điểm):** Ở lần 2, agent áp dụng quy tắc tiết kiệm bước, chỉ gọi `get_status` và 1 lệnh `get_incident_details` cho mỗi dịch vụ (tổng cộng 4 bước). Agent hoàn thành việc đối chiếu trong 4 bước và dành bước cuối để đưa ra bảng phân tích chi tiết, phân biệt rõ trạng thái Actions (resolved) và Spectrum (monitoring), không suy đoán quan hệ nhân quả và dẫn URL nguồn đầy đủ cho cả hai nhà cung cấp.
  - **Case H7 (tăng từ 35 lên 99 điểm):** Agent không còn nghe theo chỉ dẫn của người dùng ép kết luận Cloudflare là "operational" khi bị lỗi timeout mạng. Agent giữ vững ranh giới bằng chứng, nêu rõ timeout của Cloudflare và chỉ trích dẫn thông tin đã xác thực của GitHub.
  - **Tối ưu chi phí và tài nguyên:** Tổng số token tiêu thụ giảm 25.8% (từ 111,700 xuống 82,922 tokens), số lượt gọi model giảm từ 59 xuống 43 lượt, tốc độ phản hồi nhanh hơn đáng kể.
- Lỗi còn lại hoặc mới xuất hiện:
  - Do quy tắc "tiết kiệm bước", ở một số case như H2 và H6, mô hình có xu hướng tối ưu quá đà bằng cách gọi trực tiếp `get_incident_details` cho incident đã nêu trong đề bài mà bỏ qua lệnh `get_status` tổng quan ban đầu. Dù LLM Judge đánh giá nội dung rất tốt (raw score 100 ở H2 và 78 ở H6), điểm cuối cùng vẫn bị cơ chế kiểm tra cơ học (mechanical check) áp trần điểm (score cap: 50) vì vi phạm điều kiện "Chưa gọi tool cần thiết: get_status".

## 5. Giới hạn

- **Giới hạn của điểm judge:** LLM Judge sử dụng mô hình ngôn ngữ lớn để chấm dựa trên prompt chấm điểm. Phép đánh giá này mang tính xấp xỉ và xác suất (stochasticity). Khi dùng cùng model (`openai/gpt-oss-120b` trên Groq) cho cả agent và judge, có thể phát sinh thiên lệch tự đồng thuận (self-preference bias). Ngoài ra, điểm cuối cùng còn phụ thuộc vào các luật trần điểm cơ học (mechanical score cap) độc lập với nhận định ngữ nghĩa của LLM Judge.
- **Giới hạn của snapshot:** Dữ liệu snapshot được đóng băng tại thời điểm 09/11/2026 và chỉ lưu trữ tối đa 50 incidents gần nhất của từng hệ thống. Dữ liệu này không đại diện cho trạng thái thời gian thực của các dịch vụ ở hiện tại. Cơ chế tìm kiếm trong công cụ là tìm chuỗi con gần đúng (substring match), không phải tìm kiếm ngữ nghĩa (semantic search), do đó việc tìm kiếm rỗng không đồng nghĩa với việc không có incident.
- **Giới hạn của việc chỉ chạy một lần mỗi phiên bản:** Chạy eval một lần duy nhất chưa đủ để triệt tiêu tính ngẫu nhiên (variance) từ quá trình sampling của mô hình. Một số thay đổi về điểm giữa hai lần chạy có thể chịu ảnh hưởng bởi sự biến thiên ngẫu nhiên của LLM hơn là do cải tiến logic. Cần thực hiện nhiều lượt đánh giá độc lập (với các random seed khác nhau) để thu được kết quả có ý nghĩa thống kê bền vững.
