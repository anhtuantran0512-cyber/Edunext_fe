# EDUNEXT BAS

Userscript hỗ trợ làm bài tập trên EduNext (FPT). Tác giả: Trần Anh Tuấn.

## Chức năng

- Đọc đề trong khung chat bài học, gồm cả đề là ảnh vẽ trên canvas.
- Lấy câu trả lời từ API key của người dùng hoặc từ tab `gemini.google.com` đang mở.
- Điền câu trả lời vào ô soạn thảo, nộp bài, đọc phản hồi và tiến độ bài.
- Ghi từng bước vào console nội bộ, dừng được bất cứ lúc nào.

## Tệp

| Tệp | Nội dung |
| --- | --- |
| `EDUNEXT_BAS.user.js` | Bản userscript |
| `EDUNEXT_BAS.console.js` | Bản chạy trong DevTools Console |
| `SHA256SUMS.txt` | Mã SHA-256 của hai tệp trên |
| `edunext_ULTRA.user.js` | Bản dùng cầu nối sang tab Gemini |

Phiên bản: 1.1.3. Xem [CHANGELOG.md](CHANGELOG.md).

## Cài đặt

1. Cài Tampermonkey hoặc Violentmonkey, bật Developer Mode trong trang quản lý tiện ích.
2. Mở `EDUNEXT_BAS.user.js`, bấm **Raw**, bấm **Install**.
3. Vào EduNext, mở một bài học.
4. Bấm `Ctrl + E`, dán API key hoặc mở tab Gemini.

Bản console: `F12` → Console → dán `EDUNEXT_BAS.console.js`. Cấu hình lưu trong `localStorage`, không dùng được cầu nối Gemini.

## Phím tắt

| Tổ hợp | Chức năng |
| --- | --- |
| `Ctrl + E` | Mở/đóng bảng điều khiển |
| `Ctrl + Shift + E` | Dự phòng khi trang giữ `Ctrl + E` |
| `Ctrl + Alt + E` | Dự phòng |
| `Ctrl + Shift + X` | Dừng khẩn cấp |
| `Esc` | Đóng bảng |

## Cấu hình

Paste key vào ô trong bảng điều khiển. Hỗ trợ Gemini, OpenAI, Anthropic, OpenRouter, Groq, DeepSeek, xAI, Mistral, Together, Cerebras, Fireworks.

Hạn mức tính theo tài khoản, không theo số key. Hết hạn mức thì script dừng và ghi lý do vào console.

## Ghi chú

- Cầu nối Gemini phụ thuộc DOM của `gemini.google.com`.
- Bản phát hành chỉ chạy trên `*.fpt.edu.vn` và `gemini.google.com`.
- Đề là video hoặc cần thao tác ngoài khung chat: script dừng, không xử lý.
- Đã thử trên Brave và Chrome với Tampermonkey.
- Tệp trong repo đã được nén và làm rối. Mã nguồn gốc không công khai.

## Giấy phép

Xem [LICENSE](LICENSE).

Liên hệ: Trần Anh Tuấn, facebook.com/broamstuck-cybr
