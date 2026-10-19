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

5. **Chống hỏi trùng & nhịp như người thật (đỡ spam tab Gemini).** Đề giống nhau hoặc vừa giải xong sẽ được trả bằng đáp án cũ trong vài phút thay vì hỏi lại AI. Nhiều câu dồn dập quá thì script tự giãn nhịp gửi. Nếu tab Gemini không trả lời sau 3 lần thử, script tự dừng để khỏi gửi lặp vô hạn. Lúc nộp bài, script cũng dừng 1-3 giây "đọc lại" trước khi bấm nộp (bài viết dài dừng lâu hơn) và giữ khoảng cách tối thiểu 7 giây giữa hai lần nộp — trông tự nhiên như người bấm thật. Bản STUDIO_DIRECT còn có thêm "Kho đáp án tự học" nhớ đáp án đúng lâu dài và "Kho chung" tự tra kho đáp án dùng chung trên GitHub để khỏi tốn token — xem mục "Bản STUDIO_DIRECT có gì riêng?" bên dưới.

6. **Sổ tay ôn tập.** Bật trong GUI: script tự gom câu hỏi và lời giải vào "đề cương", xuất ra file Word (.docx) hoặc Markdown (.md), kèm file log (.json) để xem lại.

7. **Ngoài ra.** Nộp xong có thông báo trên máy tính (nếu bạn cho phép quyền thông báo); hết bài thì tự chuyển bài học tiếp theo và báo "hoàn thành buổi học".

## Bản STUDIO_DIRECT có gì riêng? — Kho Đáp Án Tự Học & Kho Chung

Bản ULTRA và free quên đáp án sau vài ngày; bản STUDIO_DIRECT mở hẳn một cuốn kho riêng:

- **Tự học:** bài được chấm CHÍNH XÁC là câu hỏi + đáp án tự chui vào kho lưu trên máy bạn (không gửi đi đâu cả).
- **Trả ngay lần sau:** gặp lại đúng câu đó, script lấy đáp án từ kho mà trả — không tốn token, không cần AI.
- **Đề xáo phương án?** Kho tự dò nội dung phương án và chỉnh lại chữ cái cho khớp, không nộp nhầm.
- **Tự thanh lọc:** đáp án nào bị chấm sai thì bị xoá khỏi kho ngay, không bao giờ tái sử dụng lại; mỗi câu chỉ giữ tối đa 1 năm, kho quá 800 câu thì tự dọn câu cũ nhất.
- **Xuất / nhập để chia sẻ:** trong GUI, mở mục "📚 Kho Đáp Án Tự Học" có nút Xuất kho (ra file JSON), Nhập kho (gộp từ file của máy khác, tự giữ bản mới hơn) và Xoá kho. Có nút bật/tắt ON/OFF kèm thống kê: đã lưu bao nhiêu câu, trúng kho bao nhiêu lượt.
- **Kho chung (từ v3.1.0):** gặp câu chưa có trên máy, script tự tra **kho đáp án dùng chung** trên GitHub (`resourcepack/edunext_bank`) *trước khi* gọi AI — máy nào từng chấm đúng một lần là máy khác dùng lại được, không tốn token của ai. Kho được nuôi bằng nút "📡 Đóng góp": xuất file các câu đã chấm CHÍNH XÁC trên máy bạn để gửi gom — chỉ có khoá câu hỏi + đáp án + số lượt xác nhận, không tài khoản, không danh tính. Nút "🌐 Kho chung: ON/OFF" để tắt hẳn nếu không dùng.
- **Chống đáp án rác ở kho chung:** chỉ nhận đáp án từ file đóng góp của client (đã được Edunext chấm CHÍNH XÁC); đáp án lệch nội dung bị đánh dấu xung đột chứ không tự ghi đè; đáp án kho chung bị chấm sai sẽ gửi kèm phiếu phản hồi và tự bị loại khi số lần sai vượt số lần xác nhận.
- **Cứng hoá & bền mạng (v3.2.0):** mọi đáp án đọc từ kho chung đều bị **lọc 2 tầng** (quá dài / dính marker bản quyền / ký tự điều khiển đều bị chặn — kể cả shard cache cũ). Mạng chập 3 lần liên tiếp → script **tạm nghỉ tra kho 10 phút** để không làm chậm việc giải bài (bấm dòng "📡 Kho chung" để thử lại ngay). Bấm "📡 Đóng góp" giờ gửi **trọn kho máy** (kể cả câu học từ bản cũ), không cần chờ học câu mới.
- **Chặn đáp án sai lưu hành (v3.3.0):** đáp án nào từng bị chấm SAI trên máy bạn sẽ bị **chặn cục bộ 30 ngày** (chỉ đúng nội dung đó — kho sửa rồi thì dùng lại bình thường), không lặp lại lỗi cũ trong lúc chờ gom kho; hỏi lại câu đó sẽ nhường cho AI thay vì nộp lại đáp án sai. Đáp án còn nằm trong hộp gửi đóng góp mà bị chấm sai sẽ **tự rút lại** — không gieo rác lên kho chung. Nhập kho từ file lạ cũng bị lọc y như kho chung (quá dài / marker bản quyền / ký tự điều khiển). Đề xáo phương án: khớp nội dung đã chuẩn hoá khoảng trắng/unicode, và khi 2 phương án trùng nội dung (mơ hồ) thì giữ nguyên đáp án chứ không đoán liều.
- **Hội tụ sau mỗi lần gom kho (v3.4.0):** máy bạn tự phát hiện bản lưu tạm của kho chung đã **cũ hơn kho thật** (sau khi đáp án mới được gom lên hoặc đáp án sai bị kho xoá) và **tự tải bản mới** — ngay trong lúc tra câu nếu kịp, hoặc tự làm tươi ở chế độ nền khi bạn đang dùng bản cũ; shard nào kho đã dọn sạch thì **cache cũ trên máy cũng bị xoá theo**, không phục vụ đáp án đã bị gỡ. Nhờ vậy vòng "gom kho → máy mọi người nhận bản mới" khép kín dù ai cũng chỉ lưu bản tạm từ trước.

## Cài đặt

**Cách 1 — Tampermonkey (khuyên dùng):**

- Cài extension Tampermonkey, nhớ bật Developer mode (chế độ nhà phát triển) cho extension.
- Mở link cài của bản muốn dùng, Tampermonkey hiện trang xác nhận, bấm Install:
  - `edunext_ULTRA.user.js` — bản chính: [cài đặt](https://raw.githubusercontent.com/anhtuantran0512-cyber/Edunext_fe/main/edunext_ULTRA.user.js)
  - `edunext_STUDIO_DIRECT.user.js` — bản phát cho kênh của BroAmStuck Studio, kèm **Kho đáp án tự học & Kho chung đồng bộ qua GitHub resourcepack**: [cài đặt](https://raw.githubusercontent.com/anhtuantran0512-cyber/Edunext_fe/main/edunext_STUDIO_DIRECT.user.js)
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
