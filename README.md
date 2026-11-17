# EDUNEXT BAS

Userscript hỗ trợ học tập trên nền tảng EduNext (FPT).

Tôi viết script này để bớt phần thao tác lặp lại khi làm bài: đọc đề, gõ câu trả lời, bấm nộp. Câu trả lời lấy từ một mô hình ngôn ngữ bằng key của chính bạn, và bạn vẫn là người bấm nộp.

Repo này chỉ có tệp đã đóng gói để cài. Bản đọc được tôi giữ ở máy, không đưa lên đây.

## Tệp trong repo

| Tệp | Dùng để |
|---|---|
| `EDUNEXT_BAS.user.js` | Bản chính, cài bằng Tampermonkey hoặc Violentmonkey |
| `EDUNEXT_BAS.console.js` | Bản dán vào DevTools Console, dùng khi không cài được userscript |
| `SHA256SUMS.txt` | Mã SHA-256 của hai tệp trên, để kiểm tra bản tải về có đúng không |
| `edunext_ULTRA.user.js` | Bản khác của tôi: chuyển đề sang một tab Gemini đang mở để lấy câu trả lời |

Phiên bản hiện tại: 1.1.3. Lịch sử sửa đổi ở [CHANGELOG.md](CHANGELOG.md).

## Script làm gì

- Đọc đề trong khung chat của bài học, kể cả đề là ảnh vẽ trên canvas.
- Gửi đề cho mô hình ngôn ngữ theo một trong hai cách: dán API key của bạn, hoặc mở sẵn tab `gemini.google.com` để script chuyển đề sang đó.
- Nhận câu trả lời, gõ vào ô soạn thảo, bấm nộp, rồi đọc phản hồi và tiến độ của bài.
- Ghi từng bước vào một console nhỏ trong script để bạn xem lại và dừng khi cần.

## Script không làm gì

- Không có máy chủ nào của tôi. API key nằm trong storage trình duyệt của bạn và chỉ đi tới nhà cung cấp AI mà bạn chọn.
- Không đảm bảo mọi bài đều xong. Đề là video, đề cần thao tác ngoài khung chat, hay bài chưa có nội dung thì script dừng và ghi lý do, không đoán bừa.
- Không sửa điểm, không gửi dữ liệu của bạn cho ai khác.

## Cài đặt

1. Cài Tampermonkey (hoặc Violentmonkey), bật Developer Mode trong trang quản lý tiện ích.
2. Mở `EDUNEXT_BAS.user.js` trong repo, bấm **Raw**, bấm **Install**.
3. Vào EduNext, mở một bài học.
4. Bấm `Ctrl + E` để mở bảng điều khiển, dán API key (hoặc mở tab Gemini nếu dùng cách thứ hai).

Không cài được userscript thì dùng bản console: mở `https://fsc-edunext.fpt.edu.vn`, đăng nhập, `F12` → **Console**, dán toàn bộ `EDUNEXT_BAS.console.js`, Enter. Bản này lưu cấu hình trong `localStorage` của trang nên không đồng bộ sang máy khác, và không nối được sang tab Gemini.

## Phím tắt

| Phím | Việc |
|---|---|
| `Ctrl + E` | Mở hoặc đóng bảng điều khiển |
| `Ctrl + Shift + E` / `Ctrl + Alt + E` | Dự phòng khi trình duyệt giữ `Ctrl + E` |
| `Ctrl + Shift + X` | Dừng ngay mọi tiến trình đang chạy |
| `Esc` | Đóng bảng đang mở |

## API key

Script không kèm key. Bạn tự tạo key ở nhà cung cấp bạn dùng: Google AI Studio, OpenAI, Anthropic, OpenRouter, Groq, DeepSeek... Tài khoản miễn phí đều có hạn mức riêng, hết hạn mức thì phải chờ, script sẽ ghi lý do vào console.

## Giới hạn đã biết

- Cách nối sang tab Gemini phụ thuộc giao diện của Gemini. Google đổi giao diện là phải sửa lại phần đó.
- Tệp phát hành chỉ chạy trên tên miền của trường và `gemini.google.com`. Cài ở nơi khác thì script không chạy, đây là chủ ý.
- Tôi mới thử trên Brave và Chrome với Tampermonkey. Trình duyệt khác có thể lệch giao diện hoặc lỗi.
- Bản console không dùng được cách nối sang tab Gemini, chỉ dùng API key.

## Dùng thế nào là việc của bạn

Tôi viết script này để thực hành: đọc hiểu DOM của một web app thật, viết vòng chờ, xử lý lỗi mạng và lỗi dữ liệu. Tôi không bán script, không thu tiền, không nhận làm bài hộ.

Nhà trường có quy chế riêng về việc dùng công cụ khi làm bài. Bạn tự kiểm tra quy chế đó và tự chịu trách nhiệm. Nếu quy chế không cho phép thì đừng dùng.

## Về tệp phát hành

Tệp trong repo không phải bản đọc được. Tôi nén và làm rối trước khi đẩy lên để tránh bị copy nguyên xi, nên mở ra sẽ không đọc được. Muốn biết script làm gì thì đọc các mục ở trên hoặc hỏi tôi. Cách làm này không phải bảo mật tuyệt đối, chỉ làm việc copy lại tốn công hơn.

## Liên hệ

- Tác giả: Trần Anh Tuấn
- Facebook: <https://github.com/broamstuck-cybr/Edunext_fe>
- Lỗi hoặc góp ý: mở issue trong repo này, hoặc nhắn Facebook.

Giấy phép: xem [LICENSE](LICENSE).
