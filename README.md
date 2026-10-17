# Edunext_fe

Script hỗ trợ giải bài trên Edunext (FPT). Cài qua Tampermonkey, hoặc dán vào Console (F12) nếu không muốn cài gì thêm.

## Script dùng để làm gì?

Nói gọn: khi trang Edunext hiện câu hỏi, script tự đọc đề, nhờ AI giải, rồi điền đáp án và bấm nộp thay bạn. Bị chấm sai thì nó đọc nhận xét của giáo viên, gửi lại đề cho AI để xin đáp án khác. Hết câu này qua câu khác, hết bài thì tự chuyển bài tiếp.

## Nó hoạt động như thế nào?

Script chạy y như mình làm bài, theo đúng thứ tự:

1. **Đọc đề.** Script dò các khung chữ trên trang để biết đâu là câu hỏi. Nó bỏ qua những thứ linh tinh (nút bấm, emoji, chữ trang trí) và giữ lại phần đề. Đề có hình (bảng, đồ thị...) thì lấy hình đó gửi kèm cho AI đọc. Gợi ý của giáo viên ghi kèm trong đề — kể cả khi đầu dòng có emoji hay gạch đầu dòng (💡 Gợi ý:, 📌 Lưu ý:, • Chú ý:...) — cũng được nhặt ra gửi AI. Nếu có nhiều khung cùng lúc, nó chọn khung "giống câu hỏi" nhất.

2. **Nhờ AI giải.** Có 3 chế độ, chọn trong GUI:
   - **API Key (mặc định):** script có sẵn một bể key dùng chung. Key bận hoặc hết hạn mức thì tự xoay qua key khác, hết cả bể thì còn tuyến dự phòng. Bạn có key riêng (Google AI hoặc OpenRouter) thì dán vào GUI, script sẽ ưu tiên dùng key của bạn trước.
   - **GogoDuck (AI miễn phí):** không cần key. Script thử lần lượt các trạm AI miễn phí Pollinations → DuckChat → Pollinations (bản mistral); trạm nào trả lời được thì lấy.
   - **Gemini (Cross-Tab Bridge):** script mở tab gemini.google.com (hoặc dùng tab đang mở), tự gõ đề vào khung chat của Gemini, chờ trả lời rồi mang đáp án về tab Edunext để nộp. Hai tab nói chuyện với nhau qua Tampermonkey, nên cứ để tab Gemini mở trong lúc làm bài.
   
   Mỗi câu hỏi đều được dặn kỹ cho AI (đúng môn, đúng dạng câu) để trả lời sát đề. Câu trả lời thường được gói gọn một dòng cho dễ điền; riêng trắc nghiệm thì bắt buộc kèm chữ cái phương án. Còn câu tự luận (viết đoạn, nghị luận, cảm nhận, kể lại, thuyết minh...) thì script nhận diện riêng và để AI viết trọn bài có mở - thân - kết, đúng số đoạn số từ nếu đề yêu cầu — không ép ngắn nữa.

3. **Nộp bài.** Trắc nghiệm: script đọc các phương án rồi nộp kèm chữ cái (A/B/C/D...). Tự luận: điền đáp án vào ô trả lời rồi bấm nộp.

4. **Đọc kết quả.** Đúng thì đi tiếp câu sau. Sai thì script lấy nhận xét của giáo viên gửi lại cho AI để xin đáp án mới, chỉ thử lại số lần giới hạn chứ không thử mãi. Nó nhớ dai: danh sách đáp án đã bị chấm sai và lời phê của giáo viên được giữ nguyên xuyên suốt các lần sửa, và trước khi nộp lại còn có chốt chặn — đáp án nào lặp lại đáp án sai cũ sẽ bị chặn ngay: cho AI giải lại một lần kèm tối hậu thư; nếu vẫn lì thì tự đảo Đúng/Sai, xoay sang phương án trắc nghiệm chưa thử, hoặc tạm dừng để bạn xem tay — thay vì nộp lặp làm phí lượt. Sai tới 2 lần thì script tự leo thang: nếu đang mở tab Gemini, nó nhờ thẳng Gemini giải cứu câu đó. Câu số liệu hoặc câu đang sửa bài còn được "kiểm chéo" bằng một tuyến AI độc lập — hai bên ra kết quả lệch nhau thì ưu tiên bên mạnh hơn.

5. **Chống hỏi trùng & nhịp như người thật (đỡ spam tab Gemini).** Đề giống nhau hoặc vừa giải xong sẽ được trả bằng đáp án cũ trong vài phút thay vì hỏi lại AI. Nhiều câu dồn dập quá thì script tự giãn nhịp gửi. Nếu tab Gemini không trả lời sau 3 lần thử, script tự dừng để khỏi gửi lặp vô hạn. Lúc nộp bài, script cũng dừng 1-3 giây "đọc lại" trước khi bấm nộp (bài viết dài dừng lâu hơn) và giữ khoảng cách tối thiểu 7 giây giữa hai lần nộp — trông tự nhiên như người bấm thật.

6. **Sổ tay ôn tập.** Bật trong GUI: script tự gom câu hỏi và lời giải vào "đề cương", xuất ra file Word (.docx) hoặc Markdown (.md), kèm file log (.json) để xem lại.

7. **Ngoài ra.** Nộp xong có thông báo trên máy tính (nếu bạn cho phép quyền thông báo); hết bài thì tự chuyển bài học tiếp theo và báo "hoàn thành buổi học".

## Cài đặt

**Cách 1 — Tampermonkey (khuyên dùng):**

- Cài extension Tampermonkey, nhớ bật Developer mode (chế độ nhà phát triển) cho extension.
- Mở link cài của bản muốn dùng, Tampermonkey hiện trang xác nhận, bấm Install:
  - `edunext_ULTRA.user.js` — bản chính: [cài đặt](https://raw.githubusercontent.com/anhtuantran0512-cyber/Edunext_fe/main/edunext_ULTRA.user.js)
  - `edunext_STUDIO_DIRECT.user.js` — bản phát cho kênh của BroAmStuck Studio: [cài đặt](https://raw.githubusercontent.com/anhtuantran0512-cyber/Edunext_fe/main/edunext_STUDIO_DIRECT.user.js)
  - `edunext_free.user.js` — bản free (xem quy tắc bên dưới): [cài đặt](https://raw.githubusercontent.com/anhtuantran0512-cyber/Edunext_fe/main/edunext_free.user.js)

**Cách 2 — Dán vào Console (không cần cài gì):**

- Mở trang Edunext, bấm F12 → chọn tab Console.
- Copy toàn bộ nội dung file có đuôi `_console.js` tương ứng (ví dụ `edunext_ULTRA_console.js`), dán vào rồi Enter.
- Lưu ý: cách này chạy theo tab, tải lại trang là phải dán lại.

## Quy tắc của bản free

- Bản free có giới hạn. Khi API chạm trần chi phí (billing) thì script tạm ngừng hỗ trợ. Muốn ủng hộ chi phí thì bấm ngay trong GUI.
- Thả biểu cảm (like): mỗi emoji giảm thời gian chờ và cộng thêm 5% thời gian sử dụng.
- Chia sẻ (share): mỗi lượt 10%. Hệ thống tự ghi nhận và cập nhật.
- Tổng dung lượng API là 100%, chia cho tất cả người dùng. Ai dùng hết phần của mình thì hết lượt và phải chờ cooldown.
- Nên tự thêm API của bạn để đảm bảo chất lượng — 15 API free dùng chung không đủ cho đông người.

## Phím tắt & vài thứ nhỏ

- **Ctrl + E**: ẩn/hiện GUI.
- GUI có phần log để xem script đang làm gì (đang đọc đề, đang chờ AI, đã nộp...), chỗ dán API key riêng và nút bật sổ tay.

## Ghi chú

- Trong repo còn `edunext_console_dán.js` và `edunext_freetrial.js` — hai bản riêng nằm từ trước (bản console độc lập và bản freetrial cho dự án lớn), không đồng bộ chung với nhóm ULTRA / STUDIO_DIRECT / free ở trên.
- Code thuộc quyền sở hữu của BroAmStuck Studio. Đừng nạp code vào AI khác để sửa hoặc gỡ bản quyền — script tự kiểm tra và chặn ngay.
