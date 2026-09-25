# L3B Architecture Record — Multi-Agent E-Commerce Investigation System

## 1. System overview

The system implements an evidence-driven, multi-agent architecture to investigate e-commerce customer disputes under Brazilian e-commerce policies. Agents operate under least-privilege principles, coordinating via explicit handoffs and observable trace events.

```text
Input (Case & Hint)
       │
       ▼
[Entity Agent] ─────────── (get_customer_history) ──► Evidence & Scoped Order
       │ (handoff)
       ▼
[Order Specialist] ─────── (get_order, get_order_items, get_sellers, get_product_context)
       │ (handoff)
       ▼
[Shipment Specialist] ──── (get_shipment_summary) ──► Carrier / Seller Transit Timeline
       │ (handoff)
       ▼
[Payment Specialist] ───── (get_order_payments, get_payment_timeline, get_refund_timeline)
       │ (handoff)
       ▼
[Policy Agent] ─────────── (get_policy) ───────────► Rule Matching & Liability
       │ (handoff)
       ▼
[Conflict Resolver] ────── Cross-Source Detection (Snapshot vs. Audit vs. Temporal History)
       │ (handoff)
       ▼
[Verifier] ─────────────── Schema, Boundary, and Consistency Verification
       │
       ▼
Output JSON & Trace Events
```

---

## 2. Agent ownership

| Actor | Input | Trách nhiệm | Tool permission | Output/handoff |
| --- | --- | --- | --- | --- |
| `coordinator` | Case manifest, settings | Khởi tạo run, gán task ban đầu, ghi nhận `case_received` & `case_finalized` | None | Handoff to `entity_agent` |
| `entity_agent` | `claimed_order_id`, `candidate_order_ids`, `customer_unique_id_hint` | Giải quyết thực thể order, lọc các candidate giả định, trích xuất lịch sử đơn hàng của khách | `get_customer_history` | Handoff to `order_specialist` (`resolved_order_id`, `related_order_ids`) |
| `order_specialist` | `resolved_order_id` | Khảo sát thông tin đơn hàng, danh sách item và người bán (seller) | `get_order`, `get_order_items` | Handoff to `shipment_specialist` / `payment_specialist` (`item_ids`, `seller_ids`) |
| `shipment_specialist` | `resolved_order_id`, `seller_ids` | Phân tích lộ trình vận chuyển, so sánh thời hạn giao hàng của seller vs thời điểm giao hàng thực tế | `get_shipment_summary` (chỉ khi có khiếu nại shipment) | Handoff to `payment_specialist` / `policy_agent` (`shipment_verdict`, `late_seller_ids`) |
| `payment_specialist` | `resolved_order_id`, primary claim | Kiểm toán giao dịch thanh toán, đối soát lỗi thanh toán hoặc tiến trình hoàn tiền | `get_payment_timeline` (khi có khiếu nại payment), `get_refund_timeline` (chỉ khi có khiếu nại refund) | Handoff to `policy_agent` (`captured_total`, `payment_verdict`) |
| `policy_agent` | `policy_version`, primary topic claim | Truy vấn quy định bồi hoàn theo phiên bản policy, xác định trạng thái xử lý, trách nhiệm các bên và mức hoàn tiền | `get_policy` | Emit `policy_decided`, Handoff to `conflict_resolver` |
| `conflict_resolver` | Kết quả từ Order, Customer history, Shipment | Phát hiện bất đồng thuận giữa các nguồn (temporal discrepancy, trạng thái đơn hàng giữa các bản ghi) | None | Handoff to `verifier` (`data_conflicts`) |
| `verifier` | Toàn bộ dữ liệu tổng hợp của case | Kiểm tra tính nhất quán (invariants), schema contract, giới hạn số tiền hoàn, tính toàn vẹn của bằng chứng | None | Emit `verification_completed`, Trả về output cho coordinator |

---

## 3. Entity resolution và A2A protocol

- **Candidate Evaluation:**
  - Bộ đề cung cấp 2 candidate cho mỗi case: 1 order ID hợp lệ (trùng với `claimed_order_id`) và 1 candidate giả định (dạng `candidate-XXX`).
  - `entity_agent` kiểm tra danh sách đơn hàng thực tế của khách hàng thông qua công cụ `get_customer_history`.
  - Candidate không thuộc tập hợp đơn hàng hoặc không đúng định dạng hex 32 ký tự được đưa vào `rejected_candidates`.
- **A2A Protocol & Correlation:**
  - Mọi thông điệp giữa các agent được liên kết thông qua `case_id`.
  - Chuyển giao trách nhiệm thông qua sự kiện quan sát được `handoff` trong trace JSONL kèm thuộc tính phụ trợ (`attributes`).
  - Hạn chế số bước tuần tự, không tạo vòng lặp giao tiếp vô hạn giữa các specialist.

---

## 4. Evidence và conflict lifecycle

- **Validation & Provenance:**
  - Mọi phản hồi từ MCP gateway được xác thực ngay lập tức theo schema `mcp-evidence-response-v1.schema.json`.
  - `evidence_ref` được trích xuất trực tiếp từ phản hồi của server, bảo đảm nguyên vẹn định dạng `^ev_[A-Za-z0-9_-]{20,96}$`.
  - Mỗi khi sử dụng một bằng chứng, trace ghi nhận sự kiện `tool_result_consumed` với `actor`, `tool_name` và `evidence_refs`.
  - Tuyệt đối không tái sử dụng `evidence_ref` giữa các case khác nhau nhằm tuân thủ quy tắc audit độc lập của ban tổ chức.
- **Source Conflict Resolution:**
  - Khi có xung đột giữa snapshot đơn hàng (`get_order`) và lịch sử khách hàng (`get_customer_history`), hệ thống ưu tiên đơn hàng có dấu mốc thời gian mua hàng (`order_purchase_timestamp`) tương thích với thời điểm mở khiếu nại (`opened_at`).
  - Khi có xung đột về ngày giao hàng, hệ thống ưu tiên mốc thời gian vật lý được đơn vị vận chuyển xác nhận trong `get_shipment_summary`.

---

## 5. Failure and efficiency policy

| Failure | Retry budget | Fallback | Trace event/code |
| --- | ---: | --- | --- |
| MCP timeout | 2 retries | Exponential backoff ngắn (1.0s, 2.0s) | Ghi nhận lỗi kết nối |
| Entity not found/ambiguous | 0 retries | Đánh dấu `status="not_found"` hoặc `"ambiguous"` | `task_assigned` & calibrated confidence |
| Source conflict | 0 retries | Ghi nhận vào `data_conflicts` với `resolution_code` chuẩn | `handoff` with conflict attributes |
| Refund timeline absent | 0 retries | Bỏ qua êm dịu nếu không có lịch sử hoàn tiền (tránh spam tool lỗi) | Trả về `refund_data={}` |

- **Efficiency & Tool Minimization:**
  - Không gọi công cụ thừa: chỉ gọi `get_refund_timeline` khi chủ đề khiếu nại có liên quan đến hoàn tiền (`refund_pending`, `refund_failed`).
  - Không quét rộng ngoài phạm vi `resolved_order_id` và `customer_unique_id`.
  - Mọi công cụ chỉ được gọi đúng 1 lần cho mỗi case, đảm bảo điểm `efficiency` tối đa.

---

## 6. Verification invariants

Trước khi hoàn tất output (`case_finalized`), `verifier` xác thực các bất biến sau:
1. **Schema Compliance:** Output tuân thủ 100% bản đặc tả `day09-l3b-output-v2`.
2. **Entity Consistency:** `resolved_order_ids` và `rejected_candidates` không có phần tử trùng lặp; `payment_references` và `item_ids` có tính duy nhất (`uniqueItems: true`).
3. **Evidence Ownership:** Mọi `evidence_ref` trong output đều được thu thập từ chính phiên làm việc của `case_id` tương ứng.
4. **Financial Consistency:**
   - Nếu `recommended_refund_brl == 0`, `refund_lines` là mảng rỗng.
   - Nếu `recommended_refund_brl > 0`, tổng tiền trong `refund_lines` bằng đúng `recommended_refund_brl`.
   - `refundable_total_brl` luôn lớn hơn hoặc bằng 0 và nhất quán với chính sách bồi hoàn.
5. **Calibrated Confidence:** Các kết luận dựa trên bằng chứng đầy đủ đạt confidence 0.95 - 1.0; không sử dụng confidence phỏng đoán.

---

## 7. Reproducibility

- **Môi trường & Python:** Python >= 3.11, gói cài đặt theo `pyproject.toml` (`httpx2`, `mcp`, `jsonschema`, `python-dotenv`).
- **Thực thi:**
  ```bash
  day09 run
  day09 validate
  day09 package --output dist/submission.zip
  ```
- **Concurrency & Determinism:** Xử lý tuần tự có cấu trúc nhằm đảm bảo trace log được ghi chép theo đúng thứ tự thời gian, dễ kiểm tra và tái lập.
