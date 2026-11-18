# EDUNEXT BAS

![version](https://img.shields.io/badge/version-1.1.3-informational)
![license](https://img.shields.io/badge/license-proprietary-lightgrey)
![platform](https://img.shields.io/badge/platform-Tampermonkey-black)

Userscript chạy trên EduNext (FPT). Đọc đề trong bài học, lấy câu trả lời từ mô hình ngôn ngữ, điền vào ô soạn thảo và theo dõi kết quả nộp bài.

## Mục lục

- [Cách hoạt động](#cách-hoạt-động)
- [Cài đặt](#cài-đặt)
- [Sử dụng](#sử-dụng)
- [Cấu hình](#cấu-hình)
- [Tệp trong repo](#tệp-trong-repo)
- [Giới hạn](#giới-hạn)
- [Xử lý sự cố](#xử-lý-sự-cố)
- [Giấy phép](#giấy-phép)

## Cách hoạt động

EduNext trình bày bài học dưới dạng hội thoại. Mỗi bài gồm nhiều structure (câu hỏi, bài tập, bước thực hành), mỗi structure cần đọc đề, trả lời và nộp riêng. Script chạy một vòng lặp trên các structure chưa hoàn thành:

```text
đọc đề --> chuẩn hoá --> gọi model --> định dạng --> nộp --> đọc phản hồi
   ^                                                              |
   +----------------------- structure kế tiếp -------------------+
```

**1. Xác định vị trí bài**

Script đọc `completed_structure_count` / `total_structure_count` từ endpoint `structure-progress` của EduNext. Khi endpoint không trả về, nó đọc chuỗi "x/y cấu trúc" trong widget bài học. Vòng lặp dừng khi hai số bằng nhau.

**2. Trích đề**

Đề nằm trong khung chat của widget. Script lấy theo hai đường:

- Text: đọc node con của khung chat, bỏ phần header/footer, gộp thành một khối.
- Ảnh: đề thường là hình vẽ trên `<canvas>`. Gọi `toDataURL()` trực tiếp sẽ ném `SecurityError` vì canvas bị taint bởi ảnh tải từ CDN khác origin. Script lấy `src` của ảnh nguồn, tải lại bằng `GM_xmlhttpRequest` (đi vòng qua `connect-src` của CSP trang), rồi chuyển sang data URL và gửi kèm prompt.

**3. Gọi model**

Hai tuyến, chọn trong bảng điều khiển:

| Tuyến | Cơ chế |
| --- | --- |
| API trực tiếp | `fetch` tới endpoint của nhà cung cấp, header `Authorization` lấy từ key người dùng dán vào. Có timeout và retry theo mã lỗi. |
| Cầu nối Gemini | Ghi đề vào `GM_setValue`, tab `gemini.google.com` đang mở đọc ra, chèn vào ô chat, chờ câu trả lời, ghi kết quả lại storage. Script gốc poll storage để nhận. |

**4. Định dạng câu trả lời**

Prompt được chọn theo dạng đề (trắc nghiệm, điền khuyết, tự luận ngắn). Phản hồi được cắt phần dẫn nhập, chuẩn hoá xuống dòng và ký tự đặc biệt, rồi ghi vào ô soạn thảo bằng cách set giá trị DOM và phát sự kiện `input` để framework của trang nhận giá trị mới.

**5. Nộp và đọc kết quả**

Script bấm nút nộp, chờ phản hồi của hệ thống, đọc trạng thái đúng/sai và ghi vào console nội bộ. Mỗi bước là một trạng thái có timeout riêng và điều kiện thoát riêng; không có `sleep` cố định chờ may rủi.

Toàn bộ tiến trình chạy tuần tự, một structure tại một thời điểm. `Ctrl + Shift + X` dừng ngay vòng lặp và không nộp thêm.

## Cài đặt

Yêu cầu Tampermonkey hoặc Violentmonkey, Developer Mode bật trong trang quản lý tiện ích. Đã thử trên Brave và Chrome; Firefox cần bật `extensions.tampermonkey.alwaysAllowScripts` nếu CSP của trang chặn.

1. Mở `EDUNEXT_BAS.user.js`, bấm **Raw**.
2. Tampermonkey hiện trang cài đặt, bấm **Install**.
3. Mở EduNext, đăng nhập, vào một bài học.
4. Bấm `Ctrl + E` để mở bảng điều khiển.

Bản console dùng khi không cài được userscript: `F12` → **Console** → dán toàn bộ `EDUNEXT_BAS.console.js` → Enter. Bản này lưu cấu hình trong `localStorage` của trang nên không đồng bộ giữa các máy, và không dùng được tuyến cầu nối Gemini vì thiếu GM Storage.

## Sử dụng

| Tổ hợp | Chức năng |
| --- | --- |
| `Ctrl + E` | Mở hoặc đóng bảng điều khiển |
| `Ctrl + Shift + E` | Dự phòng khi trang giữ `Ctrl + E` |
| `Ctrl + Alt + E` | Dự phòng |
| `Ctrl + Shift + X` | Dừng khẩn cấp |
| `Esc` | Đóng bảng đang mở |

Bảng điều khiển có ba dạng hiển thị (bảng đầy đủ, thanh trạng thái, thanh trạng thái kèm console), đổi trong mục Giao diện. Toàn bộ thông báo đi vào console nội bộ, không bật toast đè lên trang.

## Cấu hình

Key dán vào ô trong bảng điều khiển, lưu trong storage của trình duyệt và chỉ gửi tới nhà cung cấp tương ứng. Đã hỗ trợ: Gemini, OpenAI, Anthropic, OpenRouter, Groq, DeepSeek, xAI, Mistral, Together, Cerebras, Fireworks.

Hạn mức tính theo tài khoản nhà cung cấp, không theo số key. Nhiều key tạo từ cùng một tài khoản dùng chung hạn mức.

## Tệp trong repo

| Tệp | Nội dung |
| --- | --- |
| `EDUNEXT_BAS.user.js` | Bản userscript, cài qua Tampermonkey |
| `EDUNEXT_BAS.console.js` | Bản chạy trong DevTools Console |
| `SHA256SUMS.txt` | Mã SHA-256 của hai tệp trên |
| `edunext_ULTRA.user.js` | Bản dùng cầu nối sang tab Gemini |

```bash
sha256sum -c SHA256SUMS.txt
```

Tệp trong repo là bản đã nén và làm rối, không phải mã nguồn gốc. Mã nguồn phát triển không công khai.

## Giới hạn

- Tuyến cầu nối Gemini phụ thuộc DOM của `gemini.google.com`. Google đổi cấu trúc trang là hỏng, phải sửa selector.
- Bản phát hành kiểm tra tên miền khi khởi động và chỉ chạy trên `*.fpt.edu.vn` và `gemini.google.com`.
- Đề là video, đề yêu cầu thao tác ngoài khung chat, hoặc structure chưa có nội dung: script dừng và ghi lý do, không tự suy đoán.
- Ô soạn thảo của EduNext là `contenteditable`; một số trình duyệt chặn `execCommand` khi trang chưa focus, cần bấm vào trang một lần.

## Xử lý sự cố

**Script dừng giữa bài.** Đọc dòng cuối trong console nội bộ. Thường gặp: hết hạn mức (`429`), key hết hiệu lực (`401`), hoặc trang đang chặn do nộp quá nhanh.

**Không thấy bảng điều khiển.** Kiểm tra Developer Mode đã bật, userscript đã bật trong Tampermonkey, và trang khớp `@match`. Thử `Ctrl + Shift + E` nếu trang giữ phím.

**Nộp xong nhưng điểm không đổi.** EduNext chấm bất đồng bộ; script đọc lại tiến độ sau một khoảng chờ. Nếu vẫn không đổi, tải lại trang rồi chạy lại bài đó.

## Giấy phép

Xem [LICENSE](LICENSE).
