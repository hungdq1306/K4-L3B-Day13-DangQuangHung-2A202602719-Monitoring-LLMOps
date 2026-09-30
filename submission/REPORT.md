# Báo cáo cá nhân — K4-L3B Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Đặng Quang Hưng
- **MSSV:** 2A202602719
- **Lớp:** K4-L3B
- **Repository URL:** https://github.com/hungdq1306/K4-L3B-Day13-DangQuangHung-2A202602719-Monitoring-LLMOps
- **Commit SHA cuối:** 61a34f827748393ced851ea7c9b412dd53dced23
- **Challenge ID:** `day13-k4-l3b-monitoring-llmops-v1`
- **Tên project Langfuse cá nhân:** `day13-k4-l3b-2A202602719`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 0/100 (chưa có log) | 100/100 | Đạt toàn bộ 4 tiêu chuẩn: JSON schema, Correlation ID, Log context enrichment, PII scrubbing (0 leaks). |
| `validate_dashboard.py` | HỢP LỆ: 6/6 panel | HỢP LỆ: 6/6 panel | Đủ 6 panel chuẩn theo contract `config/dashboard.yaml` (latency, traffic, errors, cost, tokens, quality). |
| `pytest` | 22 passed | 24 passed (100%) | Đã bổ sung 2 test case kiểm tra scrubbing CCCD 12 số và thẻ tín dụng Visa/MasterCard. |
| Số traces hợp lệ | 0 traces | 15+ traces | Các trace được tạo và xác thực trong project Langfuse cá nhân, có quan hệ cha-con đúng chuẩn. |
| Số PII leak | Chưa đo | 0 leak | Toàn bộ email, số điện thoại VN, CCCD, thẻ thanh toán được che bằng `[REDACTED_*]` trước khi ghi disk. |
| Latency P95 / TTFT P95 | 156ms / 50ms | 156ms / 50ms | Baseline ổn định tốt dưới ngưỡng SLO 3000ms; lúc kích hoạt incident đạt 2654ms / 50ms. |
| Retrieval success rate | 100% | 100% | Tỷ lệ tìm kiếm tài liệu thành công đạt 100% trong điều kiện bình thường. |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:**
  Trong `app/middleware.py`, `CorrelationIdMiddleware` chịu trách nhiệm quản lý ID vòng đời request:
  1. Gọi `clear_contextvars()` đầu mỗi request để đảm bảo không rò rỉ dữ liệu giữa các thread/coroutine.
  2. Trích xuất header `x-request-id` từ client nếu có; nếu không có thì tự động sinh ID duy nhất dạng `req-<8-hex>` thông qua `f"req-{uuid.uuid4().hex[:8]}"`.
  3. Gán `request.state.correlation_id = correlation_id` và bind vào contextvars qua `bind_contextvars(correlation_id=correlation_id)`.
  4. Sau khi `call_next(request)` kết thúc, gán ngược lại vào header response: `response.headers["x-request-id"]` và thời gian phản hồi `response.headers["x-response-time-ms"]`.
- **Các metadata được ghi vào structured log:**
  Tại endpoint `/chat` (`app/main.py`), log context được bổ sung:
  - `user_id_hash`: băm SHA-256 (12 ký tự) của `user_id` để ẩn danh.
  - `session_id`: mã phiên hội thoại của người dùng.
  - `feature`: phân hệ chức năng (ví dụ `qa`, `summary`).
  - `model`: tên mô hình LLM (`claude-sonnet-4-5`).
  - `env`: môi trường hoạt động (`APP_ENV=dev`).
  Khi response được trả về, các trường định lượng như `latency_ms`, `ttft_ms`, `tokens_in`, `tokens_out`, `cost_usd`, `quality_score`, `tool_name`, `tool_success` được ghi đầy đủ vào event `response_sent`.
- **Cách bảo đảm PII được scrub trước khi ghi:**
  Đăng ký processor `scrub_event` trong chuỗi xử lý của structlog (`app/logging_config.py`) nằm trước `JsonlFileProcessor` và `JSONRenderer`. Bộ regex `PII_PATTERNS` trong `app/pii.py` quét và thay thế:
  - Email: `[\w\.-]+@[\w\.-]+\.\w+` -> `[REDACTED_EMAIL]`
  - Điện thoại VN (+84/09x/dấu chấm/gạch): `(?<!\d)(?:\+84|0)(?:[ .-]?\d){9}(?!\d)` -> `[REDACTED_PHONE_VN]`
  - CCCD (12 chữ số): `\b\d{12}\b` -> `[REDACTED_CCCD]`
  - Thẻ thanh toán: `\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b` -> `[REDACTED_CREDIT_CARD]`
  Mọi giá trị chuỗi trong `event_dict` đều được duyệt đệ quy và làm sạch trước khi serialize thành JSON hoặc ghi xuống `data/logs.jsonl`.
- **Cách kiểm chứng kết quả:**
  Chạy `python scripts/validate_logs.py` kiểm tra toàn bộ bản ghi trong `data/logs.jsonl`: không có dòng nào thiếu trường bắt buộc, không thiếu context enrichment, và 0 rò rỉ PII (Điểm: 100/100). Kiểm chứng thêm qua `pytest tests/test_pii.py` và `tests/test_validate_logs.py`.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:**
  Các trace được đẩy trực tiếp lên Langfuse Cloud project `day13-k4-l3b-2A202602719` bằng `LANGFUSE_PUBLIC_KEY` và `LANGFUSE_SECRET_KEY` cá nhân. Trace có tags `["lab", feature, self.model]`, chứa `user_id_hash` và `correlation_id` khớp từng ký tự với log trong `data/logs.jsonl`.
- **Cấu trúc root/retrieval/generation observations:**
  Sử dụng Langfuse SDK v4 Observation API:
  - Root observation: `@observe(name="lab-agent-run", as_type="agent")`.
  - Child observation 1 (Retrieval): `langfuse_client.start_as_current_observation(name="retrieval", as_type="retriever")` đo thời gian truy xuất tài liệu từ vector store.
  - Child observation 2 (Generation): `langfuse_client.start_as_current_observation(name="generation", as_type="generation")` ghi nhận model, prompt, `usage_details` (input/output tokens) và `cost_details` (chi phí USD).
- **Cách nối trace với log:**
  Trường `correlation_id` từ middleware được đưa vào metadata của trace qua `propagate_attributes(metadata={"correlation_id": correlation_id})`. Khi có sự cố, từ `correlation_id` trong log có thể tìm thấy ngay trace tương ứng trên Langfuse và ngược lại.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1 gắn nhãn `['baseline', 'production']`.
- **Version/label candidate:** Version 2 gắn nhãn `['candidate']` (bổ sung ràng buộc trả lời ngắn gọn trong 1-2 câu).
- **Trace ID của mỗi version:**
  - Version 1: `tr-27e3dede-01-day13-req` (correlation_id: `req-27e3dede`)
  - Version 2: `tr-candidate-v2-test` (correlation_id: `req-b68ec36d`)
- **Cách promote và rollback `production`:**
  - Promote: Cập nhật nhãn trên Langfuse: chuyển label `production` từ Version 1 sang Version 2 qua `lf.update_prompt(name='day13-chat', version=2, new_labels=['candidate', 'production'])`.
  - Rollback: Khi candidate gây lỗi hoặc tăng token/chi phí bất thường, đưa label `production` quay lại Version 1: `lf.update_prompt(name='day13-chat', version=1, new_labels=['baseline', 'production'])`. Ứng dụng tự động nhận diện version mới theo label mà không cần sửa một dòng code nào.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:**
  Đã dựng đủ 6 panel trực quan từ nguồn dữ liệu `data/logs.jsonl`:
  1. Latency: P50 (152ms), P95 (2653ms), P99 (2654ms), TTFT P95 (50ms).
  2. Traffic: tổng số request và tốc độ request/phút.
  3. Errors & Retrieval: error rate (0%), retrieval success rate (100%).
  4. Cost: tổng chi phí tiêu thụ ($0.0304 USD) theo dõi theo phút.
  5. Tokens: tổng số input tokens (450) và output tokens (1,850).
  6. Quality: điểm chất lượng proxy trung bình (0.86 / 1.00).
- **SLO và lý do chọn:**
  SLO chính: `fast_successful_requests` với cửa sổ 28 ngày: `event == "response_sent" and latency_ms <= 3000` đạt mục tiêu 99.5%.
  Lý do: Ứng dụng chat hội thoại trực tiếp cần đảm bảo độ trễ dưới 3 giây để người dùng không cảm thấy lag, đồng thời 99.5% là mục tiêu thực tế cho phép xử lý bảo trì hoặc sự cố đột xuất.
- **Cách tính error budget:**
  Với SLO 99.5%, Error Budget cho phép là 0.5%. Nếu hệ thống phục vụ 10,000 requests trong 28 ngày, ngân sách lỗi tối đa là:
  $$10,000 \times 0.5\% = 50\text{ requests}$$
  Nếu có quá 50 requests bị lỗi HTTP 500 hoặc latency > 3000ms, Error Budget bị cạn kiệt (burn rate > 100%), kích hoạt đóng băng triển khai mới để ưu tiên khắc phục độ tin cậy.
- **Ba alert và runbook tương ứng:**
  1. `HighLatencyP95` (warning, `p95(latency_ms) > 3000ms` trong 5m): Runbook xem dashboard latency, lấy correlation_id từ log, mở Langfuse waterfall để phân biệt chậm do RAG hay LLM; rollback prompt nếu do prompt dài.
  2. `HighErrorRate` (critical, `error_rate > 2%` trong 3m): Runbook lọc log `request_failed`, kiểm tra ngoại lệ backend (database, LLM API timeout), bật fallback response tĩnh.
  3. `RetrievalSuccessRateDrop` (warning, `retrieval_success_rate < 90%` trong 5m): Runbook kiểm tra vector store connectivity, restart service vector store nếu rớt kết nối.

## 7. Điều tra challenge

- **Challenge ID:** `day13-k4-l3b-monitoring-llmops-v1`
- **Khoảng thời gian điều tra:** `2026-09-30 04:03:38 UTC` đến `04:03:52 UTC`
- **Triệu chứng từ metrics:**
  Panel Latency ghi nhận P95 tăng vọt từ 156ms lên 13,298ms trên phân hệ `monitoring` khi chạy tải 5 luồng đồng thời. Tuy nhiên TTFT P95 vẫn ở mức 50ms, Error rate là 0%, Token và Cost không thay đổi bất thường. Điều này chứng minh LLM sinh token bình thường, sự chậm trễ nằm ở giai đoạn tiền sinh (pre-generation retrieval).
- **Log line và correlation ID liên quan:**
  Lọc log với điều kiện `latency_ms > 2000` và `feature == "monitoring"`, phát hiện request tiêu biểu có `correlation_id: req-ec763c7c`:
  `{"event": "response_sent", "correlation_id": "req-ec763c7c", "latency_ms": 7981, "ttft_ms": 50, "feature": "monitoring", "tool_name": "retrieval", "tool_success": true}`
- **Trace ID và span gây ảnh hưởng:**
  Tra cứu `req-ec763c7c` trên Langfuse trace waterfall:
  - Root span `day13-agent-request`: tổng thời gian 7981.5ms
  - Child span `lab-agent-run`: 7981.0ms
  - Child span `retrieval` (retriever): mất **7830.2ms** (chiếm 98.1% tổng thời gian xử lý!)
  - Child span `generation` (generation): chỉ mất **148.5ms** (bình thường, chiếm 1.9%)
- **Root cause:**
  Sự cố nghẽn mạng và concurrency lock trên Vector Database / tầng RAG retrieval (kịch bản sự cố `rag_slow`), gây chậm trễ nghiêm trọng khi có nhiều truy vấn đồng thời truy cập vector index trước khi gọi LLM.
- **Fix action:**
  Tắt kịch bản incident qua `python scripts/inject_incident.py --disable`. Trong production: cấu hình lại connection pooling, mở rộng cụm Vector DB replicas, và áp dụng semantic cache cho các truy vấn lặp lại.
- **Preventive measure:**
  Đặt timeout cứng 2.0s cho bước retrieval, nếu vượt quá hạn thì fallback về domain document tĩnh; thiết lập alert `HighLatencyP95` (với threshold 3000ms / duration 5m) để cảnh báo on-call engineer kịp thời.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:**
  Tách bạch hai child observations độc lập (`retrieval` dạng `retriever` và `generation` dạng `generation`) bên trong agent observation. Lý do: nếu gom chung thành một observation duy nhất, khi độ trễ tăng vọt ta không thể biết hệ thống chậm do vector store (RAG) hay do LLM provider, khiến quá trình điều tra kéo dài và không chính xác.
- **Một lỗi/blocker đã gặp:**
  Xung đột cổng mạng 8000 do container docker chạy nền từ repo khác chiếm dụng, khiến endpoint trả về định dạng lạ; và sự thay đổi của Langfuse Cloud khi deprecate endpoint v3 sang OpenTelemetry v4 observation API.
- **Cách tìm nguyên nhân và xử lý:**
  Sử dụng `netstat -ano` và `Get-Process` trên PowerShell để truy vết PID tiến trình xung đột và dùng `docker stop` giải phóng cổng. Với Langfuse v4, thiết kế adapter `_child_observation` bọc `start_as_current_observation` và `update_current_generation` đảm bảo tương thích hoàn hảo.
- **Cách hiểu luồng Metrics → Logs → Traces:**
  - Metrics cho biết **CÁI GÌ** đang xảy ra và **KHI NÀO** (triệu chứng tổng quan).
  - Logs cho biết **REQUEST NÀO** bị ảnh hưởng (định danh bằng `correlation_id`).
  - Traces cho biết **BƯỚC NÀO / SPAN NÀO** là nguyên nhân gốc rễ (phân rã chi tiết thời gian và lỗi).
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:**
  Khác với phần mềm truyền thống, ứng dụng LLM có thể gặp sự cố phi chức năng: câu trả lời dài dòng làm tăng vọt chi phí token và độ trễ dù không có lỗi code. Quản lý prompt version tập trung và khả năng rollback tức thời qua nhãn `production` giúp cô lập rủi ro và khôi phục hệ thống trong vài giây.
- **Điều quan trọng nhất đã học:**
  Kỹ năng vận hành LLMOps chuẩn mực: luôn xây dựng hệ thống quan sát đa tầng (Metrics, Logs, Traces) và bảo vệ dữ liệu nhạy cảm (PII Redaction) ngay từ đầu trước khi đưa AI ra production.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:**
  Bài lab đã hoàn thiện đầy đủ 100% các yêu cầu từ CP0 đến CP4, vượt qua mọi bài kiểm tra tự động và chuẩn bị đầy đủ bằng chứng kiểm tra.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [x] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
