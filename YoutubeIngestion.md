# 🎯 YÊU CẦU: Tích hợp tính năng Native YouTube Transcript Ingestion vào WeKnora

## 1. Mục tiêu
Xây dựng một tính năng mới hoàn chỉnh (cả UI và Backend) trong mã nguồn WeKnora cho phép người dùng dán danh sách URL YouTube (video lẻ hoặc playlist), tự động bóc tách nội dung text không chứa timestamp, và nạp trực tiếp vào Knowledge Base của WeKnora. Sau khi chạy thành công, giao diện tự động xóa sạch các link đã dán.

---

## 2. Nhiệm vụ Backend

### Bước 1: Cài đặt thư viện phụ thuộc
- Bổ sung vào requirements/dependencies của backend:
  - `yt-dlp`
  - `youtube-transcript-api`

### Bước 2: Tạo service xử lý YouTube (`youtube_service`)
Tạo module xử lý logic gồm các bước:
1. Nhận danh sách URL từ request.
2. Dùng `yt-dlp` với tùy chọn `extract_flat: in_playlist` để bóc tách tất cả Video ID và Video Title (hỗ trợ cả link video lẻ và link playlist).
3. Dùng `youtube-transcript-api` để lấy phụ đề sạch:
   - Ưu tiên ngôn ngữ: `['vi', 'en']`.
   - Nối toàn bộ `item['text']` thành văn bản thuần túy (không lấy timestamp để tối ưu token).
4. Gọi trực tiếp hàm nội bộ của WeKnora dùng để tạo tài liệu/knowledge item (internal ingest function), truyền vào:
   - `title`: Tiêu đề video.
   - `content`: Nội dung transcript kèm dòng đầu ghi nguồn URL.
   - `source_type`: 'youtube'.

### Bước 3: Tạo REST API Endpoint
- Thêm route: `POST /api/v1/ingest/youtube`
- Input Body: `{ "urls": ["https://...", "https://..."], "kb_id": "<knowledge_base_id>" }`
- Output: Trả về trạng thái thành công, số lượng video đã nạp, và danh sách các video bị lỗi (nếu video không có phụ đề).

---

## 3. Nhiệm vụ Frontend (Giao diện)

### Bước 1: Thêm trang / Tab mới
- Thêm một mục vào sidebar hoặc menu cấu hình dữ liệu: **YouTube Ingestion**.
- Nếu WeKnora tổ chức theo từng Knowledge Base, hãy đặt nó trong phần **Add Document / Data Source** của Knowledge Base đó.

### Bước 2: Thiết kế giao diện
- **Ô nhập liệu:** Một `textarea` lớn cho phép người dùng dán nhiều URL (mỗi URL một dòng).
- **Nút hành động:** Nút bấm **"Bắt đầu nạp dữ liệu"** (có trạng thái loading khi đang xử lý).
- **Hiển thị kết quả:** Thông báo toast hoặc bảng trạng thái báo cáo số video nạp thành công / thất bại.
- **Tự động xóa link:** Khi API trả về response thành công (200 OK), lập tức đặt lại state của ô `textarea` thành chuỗi rỗng `""`.

---

## 4. Ràng buộc kỹ thuật
- Tự động bắt lỗi (try-catch) đối với từng video riêng lẻ: nếu một video trong playlist không có phụ đề thì ghi nhận log lỗi và tiếp tục xử lý các video còn lại, không làm crash toàn bộ tiến trình.
- Không lưu file tạm ra ổ đĩa nếu không cần thiết; ưu tiên truyền text trực tiếp vào pipeline xử lý document của WeKnora.