# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert mẫu để tham khảo

Ví dụ dưới đây minh họa mức độ cụ thể cần có. Học viên không cần copy nguyên, nhưng ba alert trong bài nộp nên rõ ràng tương tự: điều kiện là gì, kéo dài bao lâu, ảnh hưởng tới user ra sao và người trực cần kiểm tra gì trước.

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: latency P95 của `response_sent.latency_ms`
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` trong 5 phút
- Ảnh hưởng tới người dùng: người dùng phải chờ lâu hơn trước khi nhận câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard latency để xác nhận P95/P99 và khoảng thời gian tăng.
  2. Lọc `data/logs.jsonl` trong khoảng đó, lấy một `correlation_id` có `latency_ms` cao.
  3. Mở trace cùng `correlation_id` trên Langfuse, so sánh các span chính để xác định bước nào bất thường.
- Mitigation tạm thời: dựa trên evidence thực tế để rollback prompt, khôi phục cấu hình liên quan, tắt practice scenario hoặc giảm tải khi demo.
- Owner: `student-2A202602719`

## Alert 1

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: latency P95 của `response_sent.latency_ms`
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` kéo dài 5 phút
- Ảnh hưởng tới người dùng: Người dùng phải chờ lâu hơn trước khi nhận câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard latency để xác nhận P95/P99 và khoảng thời gian tăng.
  2. Lọc `data/logs.jsonl` trong khoảng đó, lấy một `correlation_id` có `latency_ms` cao.
  3. Mở trace cùng `correlation_id` trên Langfuse, so sánh các span retrieval và generation để xác định bước gây chậm.
- Mitigation tạm thời: Rollback prompt sang version cũ nếu prompt dài; khởi động lại vector store nếu retrieval chậm; bật cache.
- Owner: `student-2A202602719`

## Alert 2

- Tên: `HighErrorRate`
- Severity: `critical`
- Duration: `3m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Tỷ lệ lỗi request (count `request_failed` / count `request_received`)
- Điều kiện và thời gian duy trì: `error_rate > 2%` kéo dài 3 phút
- Ảnh hưởng tới người dùng: Nhiều người dùng gặp lỗi 500 khi gửi tin nhắn chat
- Ba bước kiểm tra đầu tiên:
  1. Mở panel Errors trên dashboard để kiểm tra tỷ lệ lỗi và danh sách `error_type`.
  2. Lọc file `data/logs.jsonl` theo `event == "request_failed"` để xem chi tiết ngoại lệ và `correlation_id`.
  3. Mở trace tương ứng trên Langfuse để xác định span bị lỗi (retrieval lỗi kết nối, LLM provider timeout).
- Mitigation tạm thời: Kiểm tra hạ tầng backend; kích hoạt fallback response nếu retrieval lỗi; khởi động lại process API nếu có sự cố process.
- Owner: `student-2A202602719`

## Alert 3

- Tên: `RetrievalSuccessRateDrop`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Retrieval success rate (tỷ lệ span retrieval thành công)
- Điều kiện và thời gian duy trì: `retrieval_success_rate < 90%` kéo dài 5 phút
- Ảnh hưởng tới người dùng: Hệ thống không lấy được tài liệu hướng dẫn, câu trả lời suy giảm chất lượng
- Ba bước kiểm tra đầu tiên:
  1. Xem panel Errors trên dashboard để xem retrieval failure count và success rate.
  2. Lọc `data/logs.jsonl` tìm `tool_name == "retrieval"` và `tool_success == false`.
  3. Mở Langfuse trace kiểm tra span `retrieval`, xem query input và lỗi exception.
- Mitigation tạm thời: Tắt incident nếu đang kích hoạt practice scenario; chuyển sang fallback documents tĩnh; kiểm tra vector store service.
- Owner: `student-2A202602719`
