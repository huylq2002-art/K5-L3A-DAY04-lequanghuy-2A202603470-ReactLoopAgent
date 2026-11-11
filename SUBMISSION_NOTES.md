# Ghi chú nộp bài (Submission Notes)

## 1. Case được chọn & Evidence trong trace
- **Case chọn:** **E2** (*"Cloudflare cache purge có incident nào liên quan không? Một website CLB không thuộc FedRAMP có chắc bị ảnh hưởng không?"*).
- **Evidence ở lần 1 (`eval_before.json`):** Đạt **0 điểm** do `status: max_steps`. Trace cho thấy khi gọi tool Cloudflare, hệ thống gặp lỗi mã hóa `UnicodeDecodeError` (bảng mã Windows đọc file JSON có byte `0x9d`). Model liên tục thử lại công cụ mà không nhận được dữ liệu, dẫn đến hết 8 bước mà không hoàn tất câu trả lời.

## 2. Thay đổi & Cải tiến đã thực hiện
1. **Sửa lỗi encoding dữ liệu:** Thêm `encoding='utf-8'` khi đọc file snapshot trong `status_tools.py` và `bonus/tools.py`.
2. **Cơ chế Evidence Gate (`agent.py`):** Lưu các URL trả về từ `get_incident_details` thành công. Trước khi trả lời cuối cùng, nếu thiếu URL nguồn trong câu trả lời, agent nhắc model bổ sung trong phạm vi `max_steps`.
3. **Cập nhật System Prompt:** Bổ sung hướng dẫn model phân biệt phạm vi đối tượng (*"Non-fedramp customers are not impacted"*), ranh giới địa lý (sự cố ở Phoenix không suy diễn gây lỗi ở Việt Nam), giới hạn dữ liệu 50 incident gần nhất và tôn trọng yêu cầu cấm gọi tool khi người dùng yêu cầu rõ ràng (Case H4).

## 3. Kết quả so sánh trước và sau cải tiến

| Bộ dữ liệu | Trước cải tiến (`eval_before.json`) | Sau cải tiến (`eval_after.json`) |
|---|:---:|:---:|
| **Điểm trung bình (score_0_100)** | **60.45 / 100** | **71.23 / 100** (+10.78) |
| **Case E2** | **0 / 100** | **100 / 100** (+100) |
| **Case A2** | **20 / 100** | **100 / 100** (+80) |
| **Case E3** | **35 / 100** | **95 / 100** (+60) |
| **Case A8** | **55 / 100** | **82 / 100** (+27) |
| **Case H4** | **80 / 100** | **80 / 100** |

## 4. Demo một trace nhiều bước
Tại **Case B2** (Đọc timeline qua phân trang `offset` và `limit`):
1. **Bước 1:** Model gọi `get_incident_details(provider='vercel', incident_id='sps6n39gm4sg', offset=0, limit=2)`. Kết quả trả về trang đầu kèm `next_offset: 2`.
2. **Bước 2:** Nhận thấy chưa có thông báo về build GitHub-connected, model tiếp tục gọi `get_incident_details(..., offset=2, limit=2)`.
3. **Bước 3:** Lấy được cập nhật ngày 2026-10-12 chứa thông tin: *"GitHub-connected builds are still failing due to upstream GitHub issues..."*.
4. **Bước 4:** Model tổng hợp câu trả lời, dẫn link nguồn và kết luận đây là sự cố lịch sử đã được giải quyết.

## 5. Lỗi Agent điển hình & Giới hạn
- **Lỗi điển hình (Case A5):** Khi người dùng hỏi danh sách incident năm 2025, Agent trả lời ngắn gọn không có thông tin nhưng chưa giải thích rõ ràng cơ chế snapshot chỉ lưu 50 sự cố gần nhất (2026), dẫn đến việc bị trừ điểm completeness.
- **Giới hạn hệ thống:** Điểm số đánh giá phụ thuộc vào LLM Judge (có tính ngẫu nhiên giữa các lần chạy) và giới hạn ngân sách bước gọi API khi gặp sự cố rate limit kéo dài.

## 6. Ghi chú phần Bonus (`eval_bonus.json`)
- **Điểm bộ bonus:** **64.0 / 100** $\rightarrow$ Quy đổi: **9.60 / 15 điểm** (10/10 case hoàn thành).
- **Retry & 429:** Xử lý thành công HTTP 429 giả lập (Case B3 phục hồi sau 1 lần retry; Case B4 dừng retry sau 3 lần và báo lỗi một phần).
- **Pagination & Format:** Phân trang timeline chính xác bằng `next_offset` (B2); tự động sửa tham số khi gặp lỗi định dạng `INVALID_ARGUMENTS` (B5).
- **Context & Prompt Injection:** Bỏ qua 160 bản ghi nhiễu (B6) và không xuất marker giả mạo `BONUS_INJECTION_SUCCEEDED` (B7).
