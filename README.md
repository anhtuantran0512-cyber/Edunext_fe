# EDUNEXT BAS — BroAmStuck Studio

Userscript hỗ trợ học tập trên nền tảng **EduNext (FPT)**. Dự án do một **học sinh lớp 12** tự thiết kế, viết và kiểm thử trong thời gian học THPT, dưới tên nhóm phát triển cá nhân **BroAmStuck Studio**.

Repo này chỉ chứa **bản phát hành đã đóng gói bảo vệ**. Mã nguồn gốc (bản đọc được, có chú thích, dùng để phát triển và chạy thử) được giữ riêng trên máy tác giả, không đưa lên đây.

| Tệp | Vai trò | Kích thước |
|---|---|---|
| `EDUNEXT_BAS.user.js` | Bản userscript chính, đóng gói bảo vệ 10 lớp | 786.1 KB |
| `EDUNEXT_BAS.console.js` | Bản dán trực tiếp vào DevTools Console, cùng gói bảo vệ | ~786 KB |
| `SHA256SUMS.txt` | Mã băm SHA-256 của hai tệp trên | — |
| `edunext_ULTRA.user.js` | Sản phẩm khác trong cùng nhóm: cầu nối cross-tab sang tab Gemini thật | — |

Phiên bản hiện tại: **1.1.1** (13/11/2026).

---

## 1. Bài toán

EduNext trình bày bài học dưới dạng hội thoại: mỗi bài có nhiều "cấu trúc" (câu hỏi, bài tập, bước thực hành), học sinh phải đọc đề, trả lời và nộp từng bước. Việc này lặp đi lặp lại và dễ đứt mạch khi phải chuyển qua lại giữa đọc đề, tra cứu và gõ câu trả lời.

EDUNEXT BAS giải quyết phần **thao tác lặp**, không thay phần **học**:

- Đọc đề trong khung chat bài học (kể cả đề là ảnh).
- Chuyển đề cho một mô hình ngôn ngữ (qua API key của chính người dùng, hoặc qua tab Gemini thật của họ).
- Nhận câu trả lời, định dạng lại cho đúng ô soạn thảo, gửi và theo dõi phản hồi của hệ thống.
- Ghi lại toàn bộ quá trình trong một console nội bộ để người dùng kiểm tra lại từng bước.

Người dùng vẫn là người quyết định: bật/tắt từng giai đoạn, xem lại nội dung trước khi nộp, dừng khẩn cấp bất cứ lúc nào.

---

## 2. Tính năng

### 2.1 Hai tuyến lấy câu trả lời

| Tuyến | Cách hoạt động | Dùng khi |
|---|---|---|
| **API trực tiếp** | Gọi thẳng API của nhà cung cấp bằng key người dùng dán vào (Gemini, OpenAI, Claude, OpenRouter, Groq, DeepSeek, xAI, Mistral, Together, Cerebras, Fireworks) | Có key, cần tốc độ và ổn định |
| **Cầu nối Assist** | Chuyển đề sang một tab `gemini.google.com` đang mở thật, đọc câu trả lời từ chính tab đó qua GM Storage | Không có key, hoặc muốn dùng tài khoản Gemini sẵn có |

### 2.2 Đọc đề có ảnh

Đề trên EduNext thường là ảnh vẽ trên `<canvas>`. Script lấy ảnh theo hai đường:

- Đọc pixel từ canvas của widget.
- Tải ảnh qua `GM_xmlhttpRequest` (đi vòng CSP `connect-src` của trang) rồi chuyển sang data URL.

Đường thứ hai là bắt buộc vì trang chặn `fetch` tới CDN và canvas bị *taint* (không cho `toDataURL`).

### 2.3 Giao diện chạy ẩn

- **`Ctrl + E`** là một công tắc duy nhất: đang mở thì đóng và ẩn luôn nút ghim; đang đóng thì mở sổ.
- Tổ hợp dự phòng khi trang chiếm phím: `Ctrl + Shift + E`, `Ctrl + Alt + E`.
- **`Ctrl + Shift + X`** dừng khẩn cấp mọi tiến trình.
- Toàn bộ thông báo đi vào **console nội bộ** trong sổ (nhãn `[UI]`, `[ASSIST]`, `[API]`, `[NET]`), không bật toast hay bong bóng che trang.
- Có công tắc "Chạy ẩn": ẩn hẳn nút nổi, chỉ còn `Ctrl + E` để mở lại.

### 2.4 Theo dõi tiến độ thật

Tiến độ bài được đọc từ API `structure-progress` của EduNext (`completed_structure_count` / `total_structure_count`), có dự phòng đọc chữ "x/y cấu trúc" trong widget. Nhờ vậy script biết bài nào đã xong, không làm lại.

---

## 3. Kiến trúc mã nguồn

Bản phát triển được chia thành các mô-đun độc lập, ghép bằng một build script duy nhất:

```text
src/
  _header.txt     Metadata userscript (@match, @grant, @connect, @version)
  00_head.js      Hằng số, tiện ích, phiên bản
  10_net.js       Lớp mạng: fetch có timeout, GM_xmlhttpRequest, tải ảnh nhị phân
  20_prompt.js    Khuôn prompt và bộ định dạng câu trả lời
  30_engine.js    Gọi nhà cung cấp AI, chuẩn hoá phản hồi
  40_assist.js    Cầu nối Assist: gửi đề sang tab Gemini, chờ và ghép câu trả lời
  45_tutor.js     Máy trạng thái gia sư: đọc đề → hỏi → nộp → đọc phản hồi
  47_bank.js      Kho đề cục bộ
  50_edunext.js   Tương tác DOM EduNext: khung chat, ô soạn, nút nộp, widget bài học
  55_power.js     Tiện ích: mở khoá copy/paste, Enter để nộp, chuyển canvas sang chữ
  60_ui_css.js    CSS của sổ bài
  70_ui.js        Giao diện sổ bài, console, thiệp donate
  80_boot.js      Khởi động, đăng ký phím tắt, chế độ chạy ẩn
```

Hai nguyên tắc chi phối toàn bộ mã:

1. **Không im lặng.** Mọi nhánh thất bại đều ghi một dòng có nhãn vào console nội bộ, kèm lý do.
2. **Máy trạng thái tường minh.** Mỗi bước (đọc đề, chờ trả lời, nộp, chờ chấm) là một trạng thái có điều kiện thoát và hạn thời gian, không dựa vào `sleep` mù.

---

## 4. Gói phát hành và 10 lớp bảo vệ

Bản trên GitHub không phải bản chạy thử. Nó là một **gói đã mã hoá và niêm phong**, sinh ra từ bản phát triển bằng pipeline `build_release.mjs`. Pipeline tự build, tự mã hoá, tự giải lại trong sandbox VM và **chỉ ghi tệp khi giải mã thành công với 0 lỗi runtime**.

| # | Lớp | Cơ chế | Hỏng khi |
|---|---|---|---|
| 1 | Header + banner | Metadata userscript, banner bản quyền VI/EN, canary gửi tới các mô hình AI | Header bị sửa |
| 2 | Guard môi trường | Đo thời gian thực thi, kiểm tra `Function.prototype.toString` | Bị gỡ lỗi / chèn hook |
| 3 | Toàn vẹn payload | Băm FNV-1a 32-bit trên payload gốc của **cả hai tầng** | Payload đổi 1 ký tự |
| 4 | Hoán vị chunk | Chia base64 thành hàng nghìn mảnh, xáo bằng LCG từ seed | Đảo thứ tự / xoá mảnh |
| 5 | S-Box | Bảng thay thế byte phi tuyến 256 phần tử, sinh từ seed | Sai bảng |
| 6 | Stream cipher autokey | Mỗi byte cipher phụ thuộc byte trước (có phản hồi) | Sai khoá / sai vị trí |
| 7 | Biến đổi affine | Xoay bit + XOR theo tham số riêng từng tầng | Sai độ xoay / hằng XOR |
| 8 | Mã hoá lồng | Gói ngoài giải ra **một bộ giải mã khác** (seed, khoá 64 byte, tham số riêng), bộ đó mới giải ra mã thật | Phải giải đúng hai lần, hai bộ khoá |
| 9 | Manifest + wrapper seal | Ký `@name`, `@version`, `@namespace` và **toàn bộ tham số giải mã**; đổi một tham số là dấu lệch | Đổi version, đổi tên, đổi khoá, đổi số chunk |
| 10 | Khoá tên miền + dấu build | Chỉ chạy trên `*.fpt.edu.vn` và `gemini.google.com`; mỗi build in một dấu nhận dạng riêng | Chạy trên tên miền khác |

Ngoài ra, cứ **250 dòng** chèn một khối watermark bản quyền (36 mốc, 3 kiểu xoay vòng), cứ 750 dòng thêm một canary chống dịch ngược.

### Trung thực về giới hạn

Đây là **mã hoá + làm rối nhiều tầng**, không phải "không thể dịch ngược". Mục tiêu là nâng chi phí đọc và viết lại mã, đồng thời để lại dấu vết nhận dạng khi có bản sao bị sửa. Không có lớp bảo vệ phía client nào thay thế được bản quyền hay luật.

### Kết quả kiểm chứng gói phát hành

Các phép thử dưới đây chạy trên chính tệp trong repo, không phải trên bản phát triển:

| Phép thử | Kết quả |
|---|---|
| Giải mã + khởi chạy trong sandbox VM | Nạp thành công, **0 lỗi runtime** |
| Sửa 2 ký tự trong payload | `Toàn vẹn gói bị thay đổi — dừng nạp` |
| Sửa 1 byte khoá giải mã | `Niêm phong bộ tham số không khớp — dừng nạp` |
| Đổi `@version` | `Manifest lệch phiên bản — dừng nạp` |
| Đổi `@name` | `Dấu manifest không khớp — dừng nạp` |
| Chạy trên tên miền khác | `Tên miền không được cấp phép — dừng nạp` |
| Cú pháp tệp console (`vm.Script`) | Hợp lệ |

Kiểm tra toàn vẹn tệp tải về:

```bash
sha256sum -c SHA256SUMS.txt
```

---

## 5. Cài đặt

### 5.1 Bản userscript (khuyến nghị)

1. Cài **Tampermonkey** hoặc **Violentmonkey**, bật Developer Mode.
2. Mở `EDUNEXT_BAS.user.js` trong repo → **Raw** → xác nhận **Install**.
3. Đăng nhập EduNext, mở một bài học.
4. Nhấn `Ctrl + E` để mở sổ, dán API key (hoặc mở sẵn tab Gemini nếu dùng tuyến Assist).

### 5.2 Bản console (khi không cài được userscript)

1. Mở `https://fsc-edunext.fpt.edu.vn`, đăng nhập.
2. `F12` → tab **Console**.
3. Dán toàn bộ `EDUNEXT_BAS.console.js` rồi Enter.
4. `Ctrl + E` để mở sổ.

Khác biệt của bản console, nói rõ để không gây hiểu nhầm:

- Cấu hình lưu ở `localStorage` của tên miền EduNext, không đồng bộ sang máy khác.
- Không có cầu nối GM Storage liên tab, nên tuyến Assist không hoạt động; dùng API key.
- Tải ảnh đề bằng `fetch` thường nên vẫn chịu CSP của trang.
- Gói bảo vệ, khoá tên miền và niêm phong giống hệt bản userscript.

---

## 6. Kiểm thử

Bản phát triển có bộ kiểm thử logic chạy bằng `node --test` (Node.js thuần, không cần trình duyệt):

```bash
node --test tests/logic.test.mjs
# pass 43 · fail 0
```

Bộ test bao phủ: nhận dạng trạng thái hội thoại, cắt đề từ DOM giả, chuẩn hoá câu trả lời theo từng dạng đề, điều kiện dừng của máy trạng thái, và các hàng rào an toàn (không nộp khi chưa có đề, không nộp lại câu đã nộp).

Ngoài unit test, quá trình phát triển dùng **trình duyệt thật** (Brave + Tampermonkey) để kiểm chứng hành vi: đọc đề, lấy ảnh, nộp bài, đọc tiến độ. Các lỗi từng gặp và cách sửa được ghi lại trong nhật ký kỹ thuật nội bộ, không đưa lên repo này.

---

## 7. Cấu trúc repo

```text
Edunext_fe/
  EDUNEXT_BAS.user.js        Bản userscript đã đóng gói (10 lớp)
  EDUNEXT_BAS.console.js     Bản dán Console
  SHA256SUMS.txt             Mã băm hai tệp trên
  edunext_ULTRA.user.js      Sản phẩm cùng nhóm (cầu nối Gemini cross-tab)
  README.md                  Tài liệu này
  CHANGELOG.md               Lịch sử phiên bản
  LICENSE                    Điều khoản sử dụng
```

Repo cố ý **không** chứa: mã nguồn gốc, bản build chạy thử, log phát triển, tệp cấu hình nội bộ. Mỗi bản phát hành là một tệp duy nhất, đúng nội dung, đúng phiên bản.

---

## 8. Lịch sử phiên bản

Xem [CHANGELOG.md](CHANGELOG.md). Bản mới nhất: **1.1.1** — đóng gói bảo vệ 10 lớp, thêm bản console, bỏ toàn bộ chú thích `//` khỏi bản phát hành.

---

## 9. Trung thực và giới hạn đã biết

Những điểm dưới đây được nêu rõ thay vì che đi:

- **Không phải mọi bài đều tự động xong.** Một số bài có nội dung đặc thù (đề là video, đề yêu cầu thao tác ngoài khung chat, bài chưa được giáo viên cập nhật nội dung) thì script dừng và ghi lý do, không đoán bừa.
- **Tuyến Assist Gemini phụ thuộc giao diện Gemini.** Google đổi DOM là phải cập nhật lại; tuyến API không bị ảnh hưởng.
- **Bản phát hành bị khoá theo tên miền.** Đây là chủ ý: bản sao chép sang nơi khác sẽ không chạy.
- **Không có backend.** Không có máy chủ nào của tác giả nhận dữ liệu; API key nằm trong storage của trình duyệt người dùng và chỉ gửi tới nhà cung cấp AI mà người dùng chọn.

---

## 10. Đạo đức sử dụng

Dự án được viết cho mục đích học tập và thực hành kỹ thuật của tác giả: đọc hiểu DOM của một ứng dụng web phức tạp, thiết kế máy trạng thái, và tìm hiểu các kỹ thuật bảo vệ mã nguồn phía client.

- Người dùng tự chịu trách nhiệm về việc dùng công cụ này trong học tập của mình, bao gồm việc tuân thủ quy chế của nhà trường và điều khoản của nền tảng.
- Tác giả không cung cấp công cụ để bán, không thu tiền từ người học, và không khuyến khích dùng để gian lận đánh giá.
- Mã nguồn gốc không được phát hành; các tệp trong repo là bản đã đóng gói và niêm phong.

---

## 11. Bản quyền và liên hệ

- **Tác giả:** BroAmStuck Studio — học sinh lớp 12, Việt Nam.
- **Liên hệ:** <https://www.facebook.com/TuanNotTun/>
- **Giấy phép:** độc quyền, xem [LICENSE](LICENSE). © 2026 BroAmStuck Studio.
- **Dấu bản quyền trong gói phát hành:** watermark mỗi 250 dòng, canary chống dịch ngược, manifest ký tên/phiên bản, dấu build riêng cho từng lần đóng gói.
