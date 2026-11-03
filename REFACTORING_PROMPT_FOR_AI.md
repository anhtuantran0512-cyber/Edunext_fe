# 🚀 SIÊU PROMPT TÁI CẤU TRÚC & NÂNG CẤP TOÀN DIỆN CODEBASE: EDUNEXT ULTRA SUITE
*(Dành cho các AI Engine Cấp cao: Claude 3.7 Sonnet, GPT-4.5/o3, Gemini 2.0 Flash/Pro Thinking thẩm định, cắt tỉa và nâng cấp)*

---

## 🎯 BỐI CẢNH & MỤC TIÊU DỰ ÁN (PROJECT OVERVIEW)

Bạn là một **Chuyên gia Kiến trúc Mã nguồn Cấp cao (Principal Software Architect & Lead Security Engineer)**. Bạn đang tiếp nhận một dự án Userscript/Automation phức tạp mang tên **EduNext ULTRA Suite (v3.3.2)** dành cho nền tảng học tập **EduNext (FPT Education)**.

- **Vị trí tệp chính:** `edunext_ULTRA.user.js` (Bản chạy qua Tampermonkey/Violentmonkey) và `edunext_ULTRA_console.js` (Bản Bookmarklet / F12 DevTools Console).
- **Hiện trạng:** Tệp mã nguồn đang ở dạng **Monolithic khổng lồ (~12.500 dòng code, ~765 KB)**. Mặc dù hệ thống đang hoạt động đầy đủ chức năng, mã nguồn đang gặp vấn đề **"Phồng code" (Code Bloat)** nghiêm trọng do tích tụ nhiều phiên bản cũ, chứa nhiều khối mã chết (Dead Code), logic lặp lại và CSS inline dư thừa.
- **Nhiệm vụ của bạn:** Thẩm định toàn diện kiến trúc, cắt tỉa ít nhất 50-65% dung lượng (đưa về ~3.500 – 4.500 dòng sạch sẽ hoặc đề xuất kiến trúc module hóa qua Vite/Rollup), khắc phục các rủi ro tiềm ẩn (race condition, DOM fragility) nhưng **BẢO TOÀN 100% CÁC TÍNH NĂNG CỐT LÕI**.

---

## 🏗️ BẢN ĐỒ KIẾN TRÚC HIỆN TẠI (CURRENT SYSTEM TOPOLOGY - 14 MODULES)

File monolithic hiện tại được cấu thành từ 14 khối chức năng chính:

1. **Tampermonkey Metadata Header:** Cấu hình `@name`, `@match` (`*://*.edunext.vn/*`, `*://gemini.google.com/*`), `@grant` (`GM_setValue`, `GM_getValue`, `GM_xmlhttpRequest`, `unsafeWindow`).
2. **Global Configuration & STATE:** Reactive Store lưu trữ trạng thái (`aiMode: 'bridge'`, `autoMode: false`, `currentLesson`, `hasSentInitialGreeting`, `hasSentInitialConfirm`, v.v.).
3. **Sound Synthesizer (AudioContext):** Hiệu ứng âm thanh cơ bản bằng Web Audio API (không dùng asset ngoài).
4. **Log & UI Notification Engine:** Ghi log nội bộ và đẩy vào Console Mini HUD.
5. **DOM Scraper & Observation Engine (`domScraper`):**
   - MutationObserver quét hội thoại EduNext.
   - Bộ nhận diện tin nhắn Bot vs Tin nhắn User.
   - Bộ đếm tin nhắn để xử lý **Handshake 2 bước mở đầu (Chào -> Nhận ảnh tên lửa -> Xác nhận Sẵn sàng)**.
6. **Smart Question Classifier & Image Detector (`questionParser`):**
   - Phân loại: Trắc nghiệm (`single_choice`, `multiple_choice`), Tự luận (`text`), Điền từ (`fill_blank`), Câu hỏi mở đầu (`ready_prompt`).
   - Bộ lọc ảnh thông minh (`extractContentWithImages`): Phân biệt chính xác **Ảnh bài tập của Bot** với ảnh đại diện (avatar), icon hệ thống, SVG trang trí và **ảnh Rocket 🚀 mở đầu**.
   - Chuyển đổi ảnh sang Base64/Blob qua `GM_xmlhttpRequest` hoặc Canvas để vượt rào cản CORS.
7. **Cross-Tab Bridge Communication (`bridgeService`):**
   - Giao tiếp 2 chiều giữa Tab EduNext và Tab Gemini Web (`https://gemini.google.com/app`) thông qua `BroadcastChannel('edunext_bridge_channel')` và `GM_setValue`/`GM_getValue` fallback.
   - Gửi payload: `{ id, prompt, images: [{ base64, mimeType }], options }`.
   - Nhận payload phản hồi từ Gemini: `{ id, status, text, error }`.
8. **Gemini Web Client Injector (`geminiInjector`):**
   - Chạy trên ngữ cảnh trang `https://gemini.google.com/*`.
   - Tìm thẻ input văn bản (RichText editor `div.ql-editor` hoặc `rich-textarea`).
   - Cơ chế giả lập kéo thả/dán tệp (File Drop / Clipboard DataTransfer) để đính kèm ảnh bài tập vào khung chat Gemini.
   - Giả lập bấm nút gửi (`button[aria-label*="Send"]`).
   - Lắng nghe phản hồi Stream và gửi trả text hoàn chỉnh về tab EduNext.
9. **Multi-tier AI Solver Hub (`aiProvider`):**
   - Tuyến 1: Bridge Gemini Web (Chế độ ưu tiên số 1, miễn phí, hỗ trợ đa phương tiện hình ảnh mạnh nhất).
   - Tuyến 2: Direct Gemini API (Dành cho người dùng tự cấu hình API Key cá nhân trong Settings).
   - Tuyến 3 (Legacy): DuckDuckGo Chat VQD Scraper, Pollinations.ai, Multi-Key Pool *(CẦN ĐÁNH GIÁ CẮT TỈA)*.
10. **Prompt Builder & Static Subject Injection:**
    - Khối quy định format câu trả lời trắc nghiệm/tự luận ngắn gọn, không giải thích dài dòng.
    - Khối phân loại môn học (Toán, Lý, Hóa, Sinh, Sử, Địa, Ngoại ngữ...) chiếm tới 1.600 dòng prompt tĩnh.
11. **Shadow DOM HUD & Console GUI (`guiController`):**
    - Toàn bộ giao diện Bento thu gọn (Mini HUD 1 màn hình) được cô lập bên trong `#edunext-hud-root` qua Shadow DOM.
    - Gồm: Trạng thái kết nối Bridge, Bộ đếm câu hỏi, Nút Bật/Tắt Auto, Console terminal log trực tiếp, Cửa sổ Settings ẩn.
12. **Auto-Runner & Session Dispatcher (`autoRunner`):**
    - Điều phối chu trình: Đọc đề -> Kiểm tra Handshake -> Giải -> Điền câu trả lời -> Bấm Gửi -> Chờ xác nhận -> Chuyển bài kế tiếp.
13. **Auto Lesson Switcher (`lessonSwitcher`):**
    - Tự động quét danh sách bài học/session trong sidebar.
    - Tìm bài học tiếp theo, kích hoạt click chuyển trang, xử lý dialog xác nhận rời trang.
14. **Anti-Detection & Human Simulation:**
    - Giả lập trễ ngẫu nhiên (Jitter Delay 2 - 5s).
    - Giả lập sự kiện gõ phím (`keydown`, `input`, `keyup`) và cuộn chuột mượt mà.

---

## ⚠️ DANH SÁCH CÁC THIẾU SÓT & ĐIỂM NGHẼN CẦN BẠN CẮT TỈA & TỐI ƯU (PAIN POINTS & DEFECTS)

### 1. Phồng Code & Mã Chết (Severe Dead Code & Bloat ~3.500 dòng)
- **Tuyến DuckDuckGo / Pollinations / Legacy Key Pool:** Toàn bộ hệ thống hiện tại đã chuyển hướng sang **Bridge Gemini Web** và **Custom API Key của người dùng**. Các hàm `solveViaGogoDuck()`, `getDuckChatVQD()`, `pollinations.ai`, `solveWithMultiKeyPool()`, `rotateKey()`, `BUILTIN_KEYS = [...]` hoàn toàn là mã chết, làm tăng nguy cơ lỗi mạng và phình to tệp.
- **Prompt Môn Học Tĩnh Hardcoded (~1.600 dòng):** Hàng chục hàm bổ sung prompt theo môn học chiếm diện tích lớn nhưng thực tế Gemini Web chỉ cần 1 System Prompt duy nhất định hướng cô đọng: *"Đóng vai trợ lý giải đề EduNext, đưa ra đáp án chính xác nhất, trắc nghiệm ghi rõ A/B/C/D, tự luận trả lời trọng tâm dưới 100 từ"*.
- **CSS Shadow DOM dư thừa (~1.200 dòng):** Chứa nhiều class mồ côi từ layout Bento 4 tabs cũ (Tab API, Tab Config, Tab Stats...). Chỉ cần giữ lại giao diện Console thu gọn hợp nhất.

### 2. Quá Nhiều Hàm Lồng Nhau & Timer Phụ Thuộc (Timeout-based Fragility)
- Nhiều tác vụ quan trọng đang dùng `setTimeout(..., 1000)` hoặc `setTimeout(..., 3000)` thay vì cơ chế hướng sự kiện (Event-driven / MutationObserver / Promise-based wait).
- Cần chuẩn hóa thành hàm `waitForElement(selector, timeout)` và State Machine hữu hạn (FSM) để quản lý luồng:
  `IDLE -> HANDSHAKE_GREET -> HANDSHAKE_CONFIRM -> WAITING_QUESTION -> SOLVING -> SUBMITTING -> NEXT_LESSON`.

### 3. DOM Selector Khô Cứng (DOM Selector Drift Risk)
- EduNext sử dụng giao diện React SPA / Ant Design. Việc dùng hardcoded selector như `div.chat-content-container` có thể bị gãy khi EduNext cập nhật class name.
- Cần cơ chế **Resilient Fallback Selector**: Thử class chính -> Thử thuộc tính `[role="log"]`, `[aria-live="polite"]`, cấu trúc cây DOM cha/con.

### 4. Truyền Tải Ảnh Đa Luồng Qua BroadcastChannel
- Đôi khi ảnh chụp bài thi từ EduNext có kích thước lớn (base64 > 4MB), việc bắn qua `BroadcastChannel` có thể gây đầy bộ nhớ tạm của trình duyệt hoặc lag tab.
- Cần tối ưu: Nén ảnh nhẹ qua Canvas (resize max width/height 1600px, JPEG chất lượng 0.85) trước khi đẩy qua Bridge.

---

## 📋 TÀI NGUYÊN & MÔI TRƯỜNG THỰC TẾ (RESOURCES & SPECIFICATIONS)

Khi tiến hành thẩm định và nâng cấp, bạn cần nắm chắc các thông số kỹ thuật sau:

### 1. Quy Trình Handshake 2 Bước Mở Đầu (BẮT BUỘC BẢO TOÀN):
- Khi vừa vào một bài học mới (chưa có tin nhắn nào từ người dùng `userMsgs.length === 0`):
  - **Lần 1:** Script tự động nhắn 1 câu chào ngẫu nhiên: `['hello', 'xin chào', 'alo', 'hi', 'hiii']`.
  - **Lần 2:** Bot EduNext sẽ phản hồi bằng một tin nhắn ảnh (có icon tên lửa 🚀 hỏi *"Bạn đã sẵn sàng..."*). Script **TUYỆT ĐỐI KHÔNG GỬI ẢNH NÀY SANG GEMINI** vì sẽ làm Gemini hiểu nhầm là bài tập. Script nhận diện cờ hoặc nội dung này và tự động nhắn xác nhận ngẫu nhiên: `['ok', 'vào', 'Sẵn sàng']` (Chi phí: 0 API token, trễ 1-2s giả lập người dùng).
  - **Lần 3 trở đi:** Bot gửi bài tập thực sự -> Script mới kích hoạt Image Scraper và gửi sang Gemini Bridge.

### 2. Bộ Nhận Diện Ảnh Thật Của Bài Tập (Anti-Icon / Anti-Rocket Algorithm):
Một thẻ `<img>` trong tin nhắn Bot chỉ được coi là ảnh bài tập nếu thỏa mãn:
- `width >= 80` và `height >= 80` (Loại bỏ toàn bộ icon 16x16, 24x24, 32x32, 48x48).
- Không chứa URL của avatar người dùng/bot (không chứa `avatar`, `profile`, `logo`, `icon`).
- Không chứa icon tên lửa mở đầu (không chứa từ khóa `rocket`, `ready`, hoặc nếu là base64 thì kiểm tra xem có rơi vào giai đoạn Handshake hay không).
- Được fetch qua `GM_xmlhttpRequest({ responseType: 'blob' })` để lấy `image/png` hoặc `image/jpeg` sạch sẽ.

### 3. Cơ Chế Gemini Web Injector (`https://gemini.google.com/app`):
- **Khung nhập liệu:** `div.ql-editor[contenteditable="true"]` hoặc `div[contenteditable="true"]`.
- **Đính kèm tệp:** Mô phỏng sự kiện `paste` với `ClipboardEvent` chứa `new DataTransfer().items.add(file)` HOẶC tìm thẻ `<input type="file">` ẩn và gán `input.files = dataTransfer.files`.
- **Nút gửi:** `button[aria-label*="Send"]`, `button.send-button`, hoặc phím tắt `Enter`.
- **Đọc kết quả:** Quét phần tử phản hồi gần nhất của model (`div.model-response-text`, `message-content`), kiểm tra cờ loading (khi nào icon pause/stop biến mất và nút Send sáng trở lại thì bài giải đã xong).

---

## 🛠️ YÊU CẦU ĐẦU RA DÀNH CHO AI THỰC THI (ACTIONABLE DELIVERABLES)

Hãy đóng vai trò một AI Architect thực thụ và trả lời chi tiết theo 4 phần sau:

### PHẦN 1: BÁO CÁO THẨM ĐỊNH MÃ NGUỒN & DANH SÁCH CẮT TỈA (CODE AUDIT & PRUNING MANIFEST)
- Chỉ rõ chính xác các hàm, biến và khối code cụ thể cần **XÓA BỎ VĨNH VIỄN** (kèm số dòng ước tính và lý do).
- Đánh giá khả năng tối ưu hóa CSS Shadow DOM (giảm từ 1.200 dòng xuống < 300 dòng bằng CSS variables và utility classes).
- Đánh giá việc thay thế toàn bộ prompt môn học 1.600 dòng bằng 1 Unified Meta-Prompt siêu thông minh dài dưới 50 dòng.

### PHẦN 2: THIẾT KẾ KIẾN TRÚC MỚI (MODULAR ARCHITECTURE / ESM)
- Đề xuất cấu trúc thư mục sạch nếu chuyển dự án sang mô hình phát triển hiện đại (Rollup/Vite/ESBuild) để đóng gói ra 1 file duy nhất cho Tampermonkey:
  ```text
  edunext-suite/
  ├── src/
  │   ├── config/          # Cấu hình, selector registry, hằng số
  │   ├── core/            # State Store phản ứng (Reactive Store), FSM Runner
  │   ├── dom/             # Bộ scraper EduNext, resilient querySelector, Handshake guard
  │   ├── bridge/          # Kênh truyền thông BroadcastChannel, Image compress, Gemini Injector
  │   ├── ai/              # Bộ định tuyến AI (Bridge, Direct API, Unified Prompt)
  │   ├── ui/              # Shadow DOM Mini Console HUD (1 màn hình duy nhất, siêu nhẹ)
  │   └── index.js         # Entrypoint
  ├── package.json
  └── vite.config.js
  ```
- Trình bày sơ đồ luồng hoạt động chuẩn dạng Mermaid Diagram giữa: `EduNext Tab` <---> `BroadcastChannel` <---> `Gemini Web Tab`.

### PHẦN 3: BẢN CODE NÂNG CẤP & TINH GỌN HOÀN CHỈNH (REFACTORED CORE)
- Viết lại toàn bộ lõi của Script bằng mã nguồn JavaScript ES6+ hoàn chỉnh, chuẩn mực, **KHÔNG CẮT CỤT**, không dùng code giả lập hoặc placeholder `// ... existing code ...`.
- Áp dụng các nguyên tắc:
  - **Defensive Programming:** Không dùng `try/catch` rỗng nuốt lỗi, luôn có log rõ ràng kèm Fallback.
  - **Zero External Dependencies:** Tự chủ 100% native Web APIs.
  - **Robust Handshake:** Xử lý triệt để 2 lượt mở đầu (Chào -> Tên lửa -> Vào đề).
  - **Zero AI Slop:** Giao diện Shadow DOM đen/tím cyberpunk hoặc obsidian tối giản, không layout rối rắm, kích thước phím bấm khoa học, console log hiển thị thời gian thực mượt mà.

### PHẦN 4: CHECKLIST NGHIỆM THU & BẢO VỆ CHỐNG HỒI QUY (ZERO REGRESSION TEST SUITE)
- Cung cấp danh sách 10 kịch bản kiểm thử biên (Edge Cases):
  1. EduNext đổi selector chat.
  2. Mạng giật lag khiến Gemini Web phản hồi chậm hơn 60 giây.
  3. Tin nhắn bot là ảnh phương trình toán học mờ/độ phân giải thấp.
  4. Tab Gemini Web bị người dùng vô tình đóng.
  5. Bài học không có câu hỏi mà chỉ có video xem xong tự chuyển.
  6. Handshake bị trôi tin nhắn do mạng.
  7. Bài tập dạng trắc nghiệm chọn nhiều đáp án (checkbox).
  8. Trình duyệt chặn quyền truy cập Clipboard khi paste ảnh.
  9. Chuyển session khi chưa kết thúc chu trình trả lời.
  10. Nút Auto bật/tắt liên tục trong lúc đang giải.
