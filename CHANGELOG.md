# Changelog

Mọi thay đổi đáng chú ý của bản phát hành **EDUNEXT BAS** trên repo này.
Định dạng: [Keep a Changelog](https://keepachangelog.com/vi/1.1.0/), phiên bản theo [SemVer](https://semver.org/lang/vi/).

Bản phát hành trên repo là **gói đã đóng gói bảo vệ**. Bản phát triển (mã nguồn đọc được) không nằm trong repo này.

---

## [1.1.2] — 2026-11-13

### Sửa

- **Icon mạng xã hội luôn mở tab mới.** Trước đây bấm icon Facebook/Discord có thể
  điều hướng ngay tab đang làm bài. Nguyên nhân: EduNext gắn handler riêng cho thẻ `<a>`
  nên `target="_blank"` bị vô hiệu. Nay sự kiện được chặn ở pha capture và tab mới được
  mở bằng `window.open(url, '_blank', 'noopener,noreferrer')`; `opener` bị đặt về `null`
  để trang mới không đụng được tab gốc. Nếu trình duyệt chặn popup, thẻ `<a>` giữ
  nguyên `target="_blank"` làm đường dự phòng.

### Ghi chú kiểm chứng

- 43/43 unit test pass, `node --check` hợp lệ, self-test gói phát hành 0 lỗi runtime.
- Chưa kiểm chứng bằng trình duyệt thật trong lần này (phiên làm việc không có trình
  duyệt mở); phần sửa là thay đổi cách bắt sự kiện click, không đổi luồng dữ liệu.

---

## [1.1.1] — 2026-11-13

### Thêm

- **Lớp bảo vệ 9 — manifest binding + wrapper seal:** ký `@name`, `@version`, `@namespace` và toàn bộ tham số giải mã (seed, khoá 64 byte, độ xoay bit, hằng XOR, hệ số, salt, số chunk). Lệch bất kỳ tham số nào thì gói dừng nạp.
- **Lớp bảo vệ 10 — khoá tên miền + dấu build:** bản phát hành chỉ chạy trên `*.fpt.edu.vn` và `gemini.google.com`; mỗi lần đóng gói sinh một dấu build riêng, in ra console khi nạp thành công.
- **`EDUNEXT_BAS.console.js`:** bản dán trực tiếp vào DevTools Console cho trường hợp không cài được trình quản lý userscript. Có shim `GM_*` dựa trên `localStorage`, cùng gói bảo vệ và cùng niêm phong.
- **`SHA256SUMS.txt`:** mã băm SHA-256 của hai tệp phát hành.

### Thay đổi

- Bản phát hành **không còn chú thích `//`** nào ngoài header userscript bắt buộc. Việc bỏ chú thích dùng bộ quét có nhận diện chuỗi, template literal và regex literal, nên không cắt nhầm `/^data:image\//` hay URL.
- Self-test của pipeline giờ **thất bại thật** nếu payload không nạp được (trước đây lỗi bị `try/catch` của runtime nuốt và vẫn báo thành công).
- Pipeline đổi tên phiên bản nội bộ thành v3, mô tả đúng 10 lớp.

### Kiểm chứng

- `node --test tests/logic.test.mjs` → **43/43 pass, 0 fail**.
- Giải mã và khởi chạy trong sandbox VM → **0 lỗi runtime**.
- Sửa payload, sửa khoá, đổi `@version`, đổi `@name`, chạy sai tên miền → **cả 5 trường hợp đều bị từ chối nạp**.

---

## [1.0.9] — 2026-11-12

### Thay đổi

- Mã hoá lồng **hai tầng**: gói ngoài giải ra một bộ giải mã khác (seed, khoá, độ xoay, salt riêng), bộ đó mới giải ra mã thật.
- Hoán vị chunk bằng LCG để ciphertext không nằm liền mạch.
- Băm toàn vẹn FNV-1a ở cả hai tầng.
- Watermark bản quyền mỗi 250 dòng (35 mốc, 3 kiểu xoay vòng) và canary chống dịch ngược.
- Cập nhật `edunext_ULTRA.user.js` cùng đợt.

---

## [1.0.1] — 2026-11-11

### Thêm

- Chế độ **chạy ẩn** với `Ctrl + E` làm công tắc một phím (ẩn/hiện cả sổ và nút ghim).
- Gói bảo vệ nhiều lớp đầu tiên và watermark bản quyền trong tệp phát hành.

---

## [1.0.0] — 2026-11-11

### Thêm

- Bản phát hành đầu tiên của **EDUNEXT BAS**: đọc đề trong khung chat bài học, gọi mô hình qua API key người dùng, nộp bài và theo dõi tiến độ cấu trúc.
- Giao diện sổ bài, console nội bộ, phím tắt dừng khẩn cấp.

---

## Sản phẩm cùng nhóm

`edunext_ULTRA.user.js` có lịch sử phiên bản riêng (3.x), không gộp vào tệp này.
