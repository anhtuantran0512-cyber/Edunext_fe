# Edunext_fe — BroAmStuck Studio

Userscript hỗ trợ học tập trên **EduNext (FPT)**. Repo chỉ giữ **hai tệp phát hành** và tệp README này.

## Tệp trong repo

| Tệp | Bản | Vai trò |
|---|---|---|
| `EDUNEXT_BAS.user.js` | BAS **1.0.9** | Gia sư tự động, gói mã hoá lồng 2 tầng, ẩn/hiện bằng **Ctrl + E** |
| `edunext_ULTRA.user.js` | ULTRA **3.9.0** | Cầu nối cross-tab sang tab Gemini thật, tự giám sát 3 nguồn |

Đã xoá khỏi repo: bản console (trùng nội dung userscript, chỉ khác header), prompt tái cấu trúc cũ, và các bản `STUDIO_DIRECT` / `free` / `freetrial` trước đây.

## Cài đặt

1. Cài Tampermonkey hoặc Violentmonkey, bật Developer Mode.
2. Mở tệp → **Raw**:
   - `https://raw.githubusercontent.com/anhtuantran0512-cyber/Edunext_fe/main/EDUNEXT_BAS.user.js`
   - `https://raw.githubusercontent.com/anhtuantran0512-cyber/Edunext_fe/main/edunext_ULTRA.user.js`
3. Manager hiện trang xác nhận → **Install**.
4. Đăng nhập sẵn `gemini.google.com` trong cùng trình duyệt và để tab đó mở (ULTRA cần tab này làm cầu nối).
5. Vào `fsc-edunext.fpt.edu.vn` hoặc `edunext.fpt.edu.vn`, mở khung chat bài học.

## Phím tắt

| Phím | Tác dụng |
|---|---|
| `Ctrl + E` | Ẩn / hiện toàn bộ giao diện (chế độ chạy ngầm) |
| `Ctrl + Shift + E`, `Ctrl + Alt + E` | Dự phòng khi `Ctrl + E` bị trang chiếm |
| `Ctrl + Shift + X` | Dừng khẩn cấp mọi tiến trình |
| `Esc` | Đóng thiệp / hộp thoại phụ |

Dải phím tắt cũng nằm trong sổ: **Phụ lục → tuỳ chọn → dải Phím tắt**.

## BAS 1.0.9 — bảo vệ gói phát hành

- **Mã hoá lồng 2 tầng**: gói ngoài giải ra một bộ giải mã khác (seed, khoá 64 byte, độ xoay bit, salt riêng) rồi mới giải ra mã thật.
- **Hoán vị chunk** theo LCG: ciphertext không nằm liền mạch, thứ tự dựng lại từ seed.
- **Băm toàn vẹn FNV-1a** ở cả hai tầng — sửa một ký tự là gói dừng nạp.
- Watermark bản quyền mỗi 250 dòng (35 mốc, 3 kiểu xoay vòng), canary chống AI ở đầu / giữa / rải trong tệp.
- Giới hạn thật: đây là nhiều lớp mã hoá + làm rối, **không phải** "không thể dịch ngược". Mục tiêu là nâng chi phí đọc và viết lại.

## Đã kiểm và chưa kiểm

Đã kiểm: `node --check` cả hai tệp; 43 test logic của BAS; sửa 1 ký tự trong gói BAS thì script báo "Toàn vẹn gói bị thay đổi" và dừng nạp; grep bản phát hành không lộ tên hàm nội bộ.

Chưa kiểm: chưa chạy trọn một chuỗi thật "cài trên EduNext → giải → nộp → đọc nhận xét" bằng tài khoản thật. Không có số liệu độ chính xác trên N câu — mọi con số "95% đúng" bạn thấy ở nơi khác đều không đến từ repo này.

## Điều cần biết trước khi dùng

- Script **tự động hoá việc làm bài tập được giao để đánh giá năng lực**. Dùng nó có thể vi phạm quy chế đào tạo của trường và điều khoản của EduNext; trách nhiệm thuộc người dùng.
- Nội dung đề và ảnh được gửi cho **Google Gemini** qua tab bạn mở, chịu chính sách dữ liệu của Google.
- Script không vượt CAPTCHA, không đăng nhập hộ, không dùng cookie của bạn.
- Bản chất là bám vào DOM của trang do người khác kiểm soát, nên **sẽ hỏng khi EduNext hoặc Gemini đổi giao diện**. Đó là đặc tính, không phải sự cố tạm thời.
- Có cơ chế chống sửa: đổi các trường bản quyền trong `CONFIG` sẽ khiến giao diện bị khoá.

## Tự kiểm nhanh

```bash
node --check EDUNEXT_BAS.user.js
node --check edunext_ULTRA.user.js
grep -nE "sk-[A-Za-z0-9]{8}|AIza[A-Za-z0-9_-]{20}|ghp_[A-Za-z0-9]{20}" *.js   # không có key nhúng
```

Lịch sử thay đổi chi tiết nằm trong `git log`, không lặp lại ở đây.

Mã thuộc BroAmStuck Studio.
