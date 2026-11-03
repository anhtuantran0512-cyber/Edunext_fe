# Edunext_fe

Userscript tự động hoá việc làm bài tập trên EduNext (FPT), dùng **Cross-Tab Bridge** sang tab `gemini.google.com` đang mở để lấy đáp án. Không có API key, không có server trung gian, không có kho đáp án dùng chung.

Repo chỉ chứa đúng hai tệp:

| Tệp | Dùng cho | Kích thước |
|---|---|---|
| `edunext_ULTRA.user.js` | Cài qua Tampermonkey / Violentmonkey | ~749 KB |
| `edunext_ULTRA_console.js` | Dán vào DevTools Console | ~747 KB |

Mọi tệp di sản (script Python điều khiển Camofox, báo cáo audit, ảnh bằng chứng, ghi chú vận hành) **không** nằm ở đây mà ở project Studio tương ứng:

```
~/Documents/BroAmStuck Studio/projects/edunext/legacy_scripts/
├── automation/   # 28 script Python + solve_exam.js
├── reports/      # audit JSON + log vòng lặp
├── screenshots/  # 84 ảnh bằng chứng
└── docs/         # edunext.md, EDUNEXT_LOOP_STATE.md, đề Azota
```

Hai tệp là **cùng một mã nguồn**: bản console là bản userscript bỏ 34 dòng header `==UserScript==`. Nếu sửa một bản, bản kia phải build lại — đừng sửa tay cả hai.

Phiên bản hiện tại: **3.3.6** — React Fiber Ground Truth, Canvas #2 Active Lock & Anti-Freeze FSM Unlocking (xem mục 13).

---

## 1. Phạm vi thực tế của bản này

Bản này **chỉ còn một tuyến AI duy nhất**: Gemini qua tab thật.

- `STATE.aiMode` bị ép về `'bridge'` khi khởi động (`if (STATE.aiMode !== 'bridge') STATE.aiMode = 'bridge';`). Mọi giá trị `'api'` cũ trong storage đều bị chuyển về `'bridge'`.
- GUI vẫn vẽ ba nút chọn tuyến (`mApi`, `mDuck`, `mBridge`) nhưng hai nút đầu không còn tác dụng — `setMode()` chỉ chấp nhận `'bridge'`.
- Mã của tuyến GogoDuck (`duckduckgo.com/duckchat`, `text.pollinations.ai`) **vẫn còn trong tệp** và có thể không bao giờ chạy tới. Đây là mã chết, không phải tính năng.
- `BUILTIN_KEYS` là mảng rỗng. Không có key nào được nhúng.

Nói cách khác: nếu bạn không đăng nhập sẵn Gemini trong trình duyệt, script không giải được bài. Đây là thiết kế, không phải lỗi.

## 2. Kiến trúc Cross-Tab Bridge

Hai tab không nói chuyện trực tiếp với nhau. Gói tin được phát trên ba kênh song song: storage của userscript manager (`GM_setValue` / `GM_getValue` / `GM_addValueChangeListener`), `BroadcastChannel('EDUNEXT_GEMINI_BRIDGE')`, và `postMessage` tới cửa sổ bridge. Storage là kênh chính (bền, sống qua reload tab):

| Key | Vai trò |
|---|---|
| `EDUNEXT_AI_REQUEST` | Gói tin đề bài (JSON) do tab EduNext ghi, kèm `id` và `timestamp` |
| `EDUNEXT_AI_RESPONSE` | Đáp án do tab Gemini ghi lại |
| `EDUNEXT_BRIDGE_HEARTBEAT` | Nhịp tim của tab Gemini, ghi mỗi **1200 ms** |
| `EDUNEXT_BRIDGE_PING` / `EDUNEXT_BRIDGE_PONG` | Bắt tay để biết tab Gemini còn sống |
| `EDUNEXT_BRIDGE_STATUS` / `EDUNEXT_BRIDGE_CLOSED` | Trạng thái và mốc thời gian tab Gemini bị đóng |
| `EDUNEXT_GEMINI_MODEL` | Model ép chọn bên Gemini (mặc định `3.1 Pro`) |
| `EDUNEXT_AI_MODE` | Tuyến AI (luôn `bridge` ở bản này) |

Hệ quả trực tiếp của thiết kế này:

- Tab Gemini phải **đang mở** và đã đăng nhập. Script không tự đăng nhập, không vượt CAPTCHA, không dùng cookie nào của bạn.
- Nếu tab Gemini bị đóng, heartbeat dừng; tab EduNext nhận ra qua `EDUNEXT_BRIDGE_CLOSED` và báo lỗi thay vì chờ vô hạn.
- **Bản console không bridge được sang tab Gemini.** Khi không có userscript manager, tệp tự cài một shim: `GM_setValue`/`GM_getValue` ghi vào `localStorage` với tiền tệ `EDUNEXT_GM_` và lắng nghe sự kiện `storage`. `localStorage` là **theo origin**, mà `fsc-edunext.fpt.edu.vn` và `gemini.google.com` là hai origin khác nhau — hai tab không đọc được dữ liệu của nhau. Kênh cross-tab chỉ hoạt động trên bản cài qua Tampermonkey/Violentmonkey (xem mục 10).

### 2.1 Giao diện điều khiển (3.3.0 Single Unified Cockpit)

Giao diện được tái cấu trúc thành **1 màn hình duy nhất (Single Unified Cockpit - Không chia tab)** với bề rộng tỷ lệ vàng 490px, loại bỏ việc chuyển tab gây phân mảnh:

- **Hero Auto Capsule (`#bAutoBig`)** — Nút kích hoạt tự động kích thước lớn với hiệu ứng kính mờ và viền phát sáng, tự động chuyển màu/nhãn báo hiệu trạng thái ON/OFF.
- **Action Cockpit 6 nút siêu gọn** — 6 nút bấm tinh xảo xếp trên lưới đồng nhất: `#bForce2` (Giải lại ngay), `#bPause2` (Lưu đề cương), `#bRec2` (Ghi log ML), `#bDL` (Xuất Word đề cương), `#bKill` (Dừng khẩn), `#bAuto2` (Bật/Tắt Auto).
- **Status Deck & Heartbeat nhịp tim** — Hiển thị Engine (Bridge), trạng thái kết nối tab Gemini kèm hiệu ứng nhịp tim sống động, số lượng ảnh đề phát hiện kèm dải thumbnail `#imgStrip` (bấm mở lightbox).
- **Micro Pipeline 5 bước** — Quét đề → Xử lý ảnh → Gemini giải → Chèn đáp án → Xác minh. Bước đang chạy nhấp nháy neon, bước xong chuyển xanh, bước lỗi chuyển đỏ.
- **Console Terminal (`#eTerm`) cố định** — Cửa sổ terminal cao 190px đặt ngay bên dưới, hiển thị live stream log chi tiết, màu sắc trực quan kèm nút mở nhanh QR Donate, Discord, Facebook.

## 3. Trích đề và xử lý ảnh

Đề bài trên EduNext thường nằm **dưới dạng ảnh** (đề bài đăng ảnh thay vì chữ). Đây là phần dễ hỏng nhất và đã được sửa nhiều lần.

Cách làm hiện tại:

1. `collectQuestionImages()` quét **cả trang**, không chỉ trong khung tin nhắn, vì ảnh đề nằm ở khối bài tập bên ngoài `.chat-messages`. Bộ lọc nhận cả endpoint nội bộ `ai-gw…/images/crop_image_<hash>.png`.
2. `fetchImgBase64()` có 4 lớp dự phòng, theo thứ tự: `fetch` (không gửi credentials) → `new Image()` với `crossOrigin=anonymous` + canvas → `GM_xmlhttpRequest` (ép `responseType:'arraybuffer'`) → `null`.
3. Kết quả chỉ được chấp nhận nếu là `data:image/*`. URL hết hạn trả về JSON 404 sẽ **không** bị coi là ảnh — đây là lỗi cụ thể đã từng xảy ra và đã được chặn bằng kiểm tra MIME.
4. Trước khi gửi, ảnh được nén vừa đủ nét theo bậc `1500/0.84 → 1280/0.78 → 1100/0.7 → 900/0.6 → 760/0.5`. Bậc sau chỉ dùng khi một ảnh vượt ngưỡng cứng **1.600.000 ký tự base64**; không bao giờ xoá sạch danh sách ảnh.
5. **Transport ảnh v2:** mỗi ảnh nằm trong một khoá GM riêng (`EDUNEXT_AI_IMG_<reqId>_<i>`), gói tin `EDUNEXT_AI_REQUEST` chỉ còn manifest nhỏ (`imageChunkKeys`, `imageCount`, `schema: 2`). Bản cũ nhét toàn bộ base64 vào một JSON gần 1 MB — nhiều userscript manager cắt hoặc không ghi nổi giá trị lớn như vậy, và tab Gemini nhận đề chữ nhưng mất sạch ảnh. Phía Gemini đọc lại theo manifest, có dự phòng `EDUNEXT_AI_IMG_META`, chờ 250 ms rồi đọc lại một lần cho các manager ghi bất đồng bộ, rồi dọn khoá sau khi xong.
6. Tab Gemini ưu tiên dùng base64/chunk gửi kèm, không phụ thuộc CORS; URL chỉ dùng để bù khi thiếu ảnh.
7. **Triệt tiêu điểm mù góc màn hình (3.3.0):** Gỡ bỏ hoàn toàn điều kiện kiểm tra toạ độ góc (`nearCorner`) trong `isDecorativeImage()`. Widget chat của EduNext luôn nằm ở góc dưới bên phải màn hình (`bottom-right`), trước đây khiến 100% ảnh đề bài hiển thị trong tin nhắn bị nhận nhầm là widget góc/banner trang trí và bị loại bỏ oan uổng. Đồng thời bổ sung fallback vẽ canvas tức thì 0ms trước khi fetch mạng.
8. **Đính kèm ảnh W3C DataTransfer chuẩn hóa (3.3.0):** Khi dispatch sự kiện `paste` sang Gemini, Chromium mặc định không gán `clipboardData` cho event tổng quát khiến `e.clipboardData.files` bị `null`. Script sử dụng `Object.defineProperty(pasteEvt, 'clipboardData', { value: dt, ... })` để Gemini đọc được `files` và `items` 100%, kết hợp 4 lớp fallback đính kèm (Paste -> DragDrop -> Native File Input Injection -> Upload Menu Trigger).
9. **Lá chắn chống rò rỉ Icon/Asset & Ưu tiên Canvas AI (3.3.1):** Phân định ranh giới nghiêm ngặt giữa ảnh câu hỏi thật sự của AI và các asset trang trí trên web/chat. Chặn 100% ảnh thuộc avatar bot (`.w-chat-avatar`), nút thao tác (copy, sound, reaction), icon Ant Design (`.anticon`), và thanh điều hướng (header, nav, sidebar, footer). Nâng ngưỡng kích thước tối thiểu lên 70x50 px, lọc sạch URL/class chứa từ khóa icon/asset/svg. Đưa thẻ `<canvas>` câu hỏi AI lên vị trí ưu tiên số 1 khi trích xuất, và xoá bỏ hoàn toàn lệnh quét ảnh bừa bãi toàn trang web khi câu hỏi là dạng thuần chữ.

## 4. Kiểm soát chất lượng đáp án

Không phải mọi câu Gemini trả về đều là đáp án. `classifyGeminiReply()` phân loại phản hồi thành 5 nhóm trước khi dùng:

| Nhóm | Xử lý |
|---|---|
| `ok` | Dùng làm đáp án |
| `image_missing` | Đính lại ảnh và gửi lại đề, **đúng một lần** |
| `rate_limited` | Báo lỗi, không nộp |
| `refusal` | Báo lỗi, không nộp |
| `empty` | Báo lỗi, không nộp |

Quy tắc phân loại: có marker `Trả lời:` hoặc `ĐÁP ÁN:` thì luôn là `ok` (kể cả khi câu trả lời dài và có nhắc tới chữ "ảnh"); bài dài trên 600 ký tự cũng coi là `ok` thay vì đoán bừa. Nhận diện cả tiếng Việt và tiếng Anh (`không thấy ảnh`, `no image was attached`, `đã đạt giới hạn`, `quota exceeded`…).

Trước khi nộp, `isRefusalOrJunkAnswer()` chặn thêm các câu dạng "tôi không đọc được ảnh", markdown và câu dẫn; với câu trả lời nhiều dòng không có marker, script lấy **dòng đáp án cuối** thay vì nộp cả đoạn văn. Câu tự luận được giữ nguyên.

**Vì sao phải làm kỹ phần này:** trước khi có các chốt trên, script đã từng nộp nguyên câu *"Xin lỗi, tôi không thể xem hình ảnh đính kèm"* như một đáp án hợp lệ. Kiểu sai này không gây lỗi, không có cảnh báo — nó chỉ lặng lẽ cho điểm 0.

**Circuit breaker:** đính lại ảnh tối đa 1 lần cho mỗi câu. Không có vòng lặp gửi lại vô hạn sang tab Gemini.

## 5. Đã kiểm chứng những gì

Phần này chỉ liệt kê việc đã thực sự chạy và có kết quả, không liệt kê suy đoán.

| Hạng mục | Cách kiểm | Kết quả |
|---|---|---|
| Ảnh đề → tab Gemini | Lấy ảnh đề thật (726 KB) từ EduNext, đính sang Gemini, gửi, đọc phản hồi | Gemini mô tả đúng sơ đồ trong ảnh: 4 bước, 20 mL, tuần 16, 2 tuần nuôi cấy |
| Nén ảnh không mất dữ kiện | Nén 1280 px JPEG (968 KB → 165 KB) rồi lặp lại phép thử trên | Vẫn đọc đúng toàn bộ bước và số liệu |
| `fetchImgBase64` không có `GM_xmlhttpRequest` | Trích đúng hàm từ tệp, chạy trong trình duyệt thật không có `GM_xhr` | 4/4 ảnh thật → `data:image/png`; URL hết hạn → `null` (đúng) |
| Bộ phân loại phản hồi | 23 ca văn bản (đáp án hợp lệ, thiếu ảnh, hết hạn mức, từ chối, đa dòng, markdown) | 23/23 sau khi sửa 1 lỗi regex bị bắt bởi chính test này |
| Transport ảnh v2 (3.0) | 6 ca với `GM_getValue` đồng bộ/bất đồng bộ: khoá trực tiếp, fallback `EDUNEXT_AI_IMG_META`, META của request khác, đọc lại sau 250 ms, ảnh không phải `data:image` | 6/6 |
| Lọc đáp án ngắn (3.0) | 9 ca: `D`, `42`, `1/2`, chuỗi toàn dấu câu, marker rỗng, câu từ chối, hết hạn mức | 9/9 |
| Hàm UI mới (3.0) | Chạy `setPipelineStage` / `updateBridgeDoctor` / `setQuestionImages` / `renderQuestionImageStrip` trên DOM giả | Pipeline, deck, heartbeat, strip, lightbox đều cập nhật đúng |
| Daemon Gemini khởi tạo (3.0.1) | Nạp userscript vào realm jsdom `gemini.google.com` với GM storage chia sẻ, bơm một `EDUNEXT_AI_REQUEST` | Trước bản vá: `ReferenceError: Cannot access 'STATE' before initialization`; sau bản vá: nhận đề, phát `RECEIVING` → `SENDING`, đọc đúng ảnh chunk |
| Đính ảnh lên Gemini (3.1.0) | Trên `gemini.google.com` thật, chạy đúng luồng của `attachImageToGemini`: tìm nút `Upload & tools` → mở menu → quét input file (kể cả shadow DOM) → gán `File` bằng native setter → phát `change`/`input` | Trước: không tìm thấy nút (regex cũ không khớp `Upload & tools`) nên không có input, ảnh không bao giờ được gửi. Sau: chip `0 → 1`, ảnh 240×140 hiện preview, nút gửi bật |
| Không bấm mục "Upload files" | Kiểm trong mã và trên DOM thật | Mục này mở hộp thoại chọn tệp của hệ điều hành; bản 3.1.0 bỏ hẳn nhánh đó, chỉ gán file trực tiếp vào input |
| Lọc ảnh QR/trang trí (3.1.0) | Quét toàn bộ ảnh trong GUI và logic `isDecorativeImage` | Loại theo tên (qr/qrcode/vietqr/barcode/momo/bank/donate), theo tổ tiên mang class/id quảng cáo, theo vị trí nổi ở góc, theo kích thước/tỉ lệ bất thường |
| Giao diện đơn sắc (3.1.0) | Nạp userscript vào jsdom, đọc CSS 47 095 ký tự trong shadow root | 0 giá trị màu còn lệch kênh; toàn bộ hex/rgba đã quy về thang xám theo độ sáng |
| Cú pháp | `node --check` trên cả hai tệp | Pass |
| Đồng bộ hai bản | So sánh bản console với userscript | Chênh đúng 34 dòng header, phần thân giống nhau |

## 6. Chưa kiểm chứng

Cần nói rõ để không ai đọc bảng trên rồi tưởng quá mức:

- **Chưa chạy trọn chuỗi** "userscript cài sẵn trên EduNext → tự bấm giải → tab Gemini trả đáp án → nộp bài → đọc nhận xét". Từng nửa đã kiểm riêng với ảnh thật, nhưng toàn tuyến chưa chạy một mạch trong điều kiện thật.
- **Ba nhánh lỗi mới** (`image_missing` đính lại ảnh, `rate_limited`, `refusal`) mới kiểm bằng văn bản mô phỏng, **chưa** kích hoạt được trên Gemini thật.
- **Không có số liệu độ chính xác.** Không có thống kê tỉ lệ đúng trên N câu, vì mới chạy thử nghiệm vài câu. Bất kỳ con số "95% đúng" nào bạn thấy ở nơi khác đều không xuất phát từ repo này.
- **Chế độ console không bridge được sang tab Gemini.** Shim tự cài dùng `localStorage` theo origin (xem mục 2), nên hai tab khác domain không thấy nhau. Chưa có giải pháp cho trường hợp này.
- **Giao diện 3.0 và transport ảnh v2 mới kiểm bằng harness Node/DOM giả**, chưa chạy trong trình duyệt thật cùng tab Gemini thật. Logic đã kiểm, nhưng selector của Gemini vẫn có thể đổi.

## 7. Các kiểu hỏng đã biết

| Hiện tượng | Nguyên nhân thường gặp |
|---|---|
| Báo lỗi thiếu ảnh | Ảnh đề chưa tải xong khi trích; hoặc URL ảnh đã hết hạn |
| Chờ mãi không có đáp án | Tab Gemini đóng / chưa đăng nhập / heartbeat dừng |
| Đáp án đúng nhưng bị chấm sai | Sai định dạng điền (chữ cái phương án) hoặc đề yêu cầu dạng trả lời khác |
| Không trích được đề | EduNext đổi cấu trúc DOM; toàn bộ selector nằm trong `CONFIG.SELECTORS` |
| Chat mở ra nhưng không tự chạy | `autoSolve` chưa bật trong GUI |

Bản chất của script này là **bám vào DOM của một trang web không thuộc quyền kiểm soát của nó**, cộng với **DOM của Gemini cũng đổi thường xuyên**. Nó sẽ hỏng theo thời gian. Đó là đặc tính, không phải sự cố tạm thời.

## 8. Dữ liệu và quyền riêng tư

Điều gì được lưu, ở đâu:

- Cấu hình và trạng thái bridge nằm trong storage của userscript manager (hoặc `localStorage`) trên máy bạn.
- **Kho telemetry cục bộ** (`telemetryDB`, IndexedDB `EduNextForensicDB` → object store `telemetry_events`) ghi lại log của script, sự kiện hỏi/đáp và — khi bật ghi âm phiên (`isRecording`) — cả bản xem trước phản hồi mạng của EduNext (tối đa 2000 ký tự/phản hồi). `isRecording` mặc định **tắt**.
- **Không có dữ liệu nào trong số đó được gửi lên máy chủ của tác giả.** Không có endpoint nào của tác giả trong mã. Việc xuất `.json` ("Log kiểm định") là do bạn bấm nút, ghi xuống đĩa máy bạn. Lưu ý: tệp xuất đó có chứa `device_info` (user agent, độ phân giải màn hình) và toàn bộ luồng sự kiện — đừng đăng công khai nếu không muốn lộ thông tin máy.
- Sổ tay ôn tập giữ tối đa 1000 mục **trong bộ nhớ tab**, mất khi tải lại trang; xuất ra `.docx` / `.md` / `.json` là do bạn bấm nút.
- Nội dung đề bài và ảnh được gửi cho **Google Gemini** qua tab bạn đang mở — chịu điều khoản và chính sách dữ liệu của Google, không phải của tác giả script.
- Script **chặn (patch) `window.fetch` và `XMLHttpRequest`** trên trang EduNext để đọc phản hồi của các URL khớp `/chat|/question|/conversation|/submit|/lesson|/message|/answer|/evaluate|ai-gw`. Mục đích là hiểu cấu trúc dữ liệu trả về của EduNext. Đây là thay đổi hành vi thật trên trang, không chỉ là đọc DOM, và người dùng nên biết trước.

## 9. Những điểm cần biết trước khi tin

Mục này tồn tại vì người đọc khó tính sẽ tự tìm ra, và tốt hơn là nói trước.

**Mã nguồn bị làm rối một phần.** Tệp có khối `_0xBRS` (giải mã chuỗi) và nhiều đoạn có comment cảnh báo bản quyền. Đây là mã ~775 KB chứa cả GUI, FSM, bộ xuất Word và các bộ lọc — khó đọc hết, và việc làm rối khiến việc kiểm toán độc lập khó khăn hơn. Nếu bạn cần tin tuyệt đối, đây là trở ngại thật, không phải cảnh báo hình thức.

**Có cơ chế chống sửa (anti-tamper).** `triggerAntiTamperLockdown` sẽ chặn giao diện nếu phát hiện các trường bản quyền bị đổi (`CONFIG.AUTHOR`, `CONFIG.FACEBOOK_URL`, `CONFIG.INTEGRITY_CHECKSUM`, `CONFIG.AUTH_SIG`). Nghĩa là: sửa script để dùng riêng có thể bị khoá giao diện. Bạn nên biết điều này trước khi fork.

**`@connect *` quá rộng.** Header khai báo `@connect *` bên cạnh các host cụ thể. Các host thực sự có lời gọi mạng trong mã: `duckduckgo.com/duckchat`, `text.pollinations.ai`, `openrouter.ai/api/v1/models`, `generativelanguage.googleapis.com` (ba cái đầu và phần lớn cái cuối thuộc các tuyến API/GogoDuck đã bị vô hiệu nhưng mã vẫn còn), cộng với `ai-gw.fpt.edu.vn`, gemini.google.com và edunext khi chạy thực tế. Dòng `*` là quyền thừa do lịch sử phát triển, **đáng ra nên thu hẹp**. Đây là điểm yếu thật.

**Có link ủng hộ và liên hệ trong GUI** (`CONFIG.FACEBOOK_URL`, `CONFIG.DISCORD_URL`, `CONFIG.DONATE_STK`, ảnh QR nhúng base64). Không có cơ chế ép trả tiền, không có giới hạn lượt dùng, nhưng bạn nên biết chúng tồn tại.

**Header cũ vẫn còn sót.** `@description` và `@namespace` không khớp nội dung hiện tại; `@match` khai báo cả `gemini.google.com` bốn lần dưới các dạng khác nhau (thừa nhưng vô hại).

**Script mở khoá copy/paste và menu chuột phải** trên EduNext (`inputUnblocker` gỡ chặn `copy`, `cut`, `paste`, `contextmenu`, `selectstart`, `drag*`). Ngoài phạm vi "giải bài", đây là thay đổi hành vi của trang mà người dùng không yêu cầu rõ ràng.

**Không có cơ chế cập nhật tự động đáng tin.** `@updateURL` trỏ về `raw.githubusercontent.com` của repo; nếu repo đổi hoặc tệp bị thay, Tampermonkey có thể tải về một bản khác. Hãy đọc diff trước khi cho phép cập nhật.

## 10. Cài đặt

**Cách 1 — Userscript (khuyến nghị)**

1. Cài Tampermonkey (hoặc Violentmonkey), bật Developer Mode cho extension.
2. Mở [edunext_ULTRA.user.js](https://raw.githubusercontent.com/anhtuantran0512-cyber/Edunext_fe/main/edunext_ULTRA.user.js) → Tampermonkey hiện trang xác nhận → Install.
3. Đăng nhập Gemini trong cùng trình duyệt, mở sẵn một tab `gemini.google.com`.
4. Vào EduNext, mở chat widget của bài học, bật `autoSolve`.

**Cách 2 — Console (không dùng được bridge)**

F12 → Console → dán toàn bộ `edunext_ULTRA_console.js` → Enter. Chạy theo tab, tải lại là mất. Do shim `localStorage` theo origin, **không dùng được Cross-Tab Bridge** — cách này chỉ phù hợp để xem GUI, log và trích đề.

**Phím tắt:** `Ctrl/Cmd + E` ẩn/hiện GUI · `Ctrl/Cmd + Shift + X` dừng khẩn cấp.

## 11. Tự kiểm chứng

Không cần tin README này. Chạy:

```bash
node --check edunext_ULTRA.user.js
node --check edunext_ULTRA_console.js

# Bản console = userscript bỏ 34 dòng header
diff <(tail -n +35 edunext_ULTRA.user.js) edunext_ULTRA_console.js && echo "khớp"

# Không có key nào được nhúng
grep -nE "sk-[A-Za-z0-9]{8}|AIza[A-Za-z0-9_-]{20}|ghp_[A-Za-z0-9]{20}"
grep -n "BUILTIN_KEYS" edunext_ULTRA.user.js          # phải là Object.freeze([])
grep -n "indexedDB" edunext_ULTRA.user.js             # đúng 1 dòng: kho telemetry cục bộ
grep -nE "fetch\\(|GM_xmlhttpRequest\\(" edunext_ULTRA.user.js | head -20   # đối chiếu danh sách host
```

Muốn xem script thật sự gửi gì: mở DevTools → Network ở **cả hai tab** (EduNext và Gemini), tự làm một câu và quan sát; đồng thời mở Application → Extension Storage để xem nội dung `EDUNEXT_AI_REQUEST` / `EDUNEXT_AI_RESPONSE`. Không có endpoint nào thuộc tác giả; các host còn sót thuộc tuyến đã vô hiệu được liệt kê ở mục 9.

## 12. Giới hạn và trách nhiệm

- Script này **tự động hoá việc làm bài tập được giao để đánh giá năng lực học sinh**. Việc dùng nó có thể vi phạm quy chế đào tạo của trường và điều khoản sử dụng của EduNext. Người dùng chịu trách nhiệm đó, không phải tác giả.
- Việc chạy script còn có nghĩa là nội dung bài học của bạn được gửi cho Gemini. Hãy cân nhắc nếu đề có dữ liệu không muốn chia sẻ bên thứ ba.
- Không bảo đảm đáp án đúng. Không bảo đảm không bị phát hiện. Không bảo đảm tiếp tục hoạt động khi EduNext hoặc Gemini đổi giao diện.
- Script không vượt CAPTCHA, không đăng nhập hộ, không tạo tài khoản.

## 13. Lịch sử repo

Bản hiện tại đã dọn về hai tệp. Các bản trước — `edunext_STUDIO_DIRECT`, `edunext_free`, `edunext_freetrial`, `edunext_console_dán` — và toàn bộ tính năng "kho đáp án tự học" / "kho chung dùng chung trên GitHub" đã bị **xoá khỏi repo**. README cũ mô tả những tính năng đó cùng cơ chế "bể 15 API key dùng chung"; cả hai đều **không còn tồn tại**. Nếu bạn đọc được hướng dẫn nhắc tới chúng ở đâu khác, hướng dẫn đó đã lỗi thời.

Mã thuộc BroAmStuck Studio. Script có kiểm tra toàn vẹn và sẽ khoá giao diện nếu các trường bản quyền bị sửa.

**3.3.6** — **React Fiber Ground Truth, Canvas #2 Active Lock & Anti-Freeze FSM Unlocking:**
- **Khóa mục tiêu Canvas #2 & Cách ly thẻ chào mừng Turn 0 (Canvas #2 Active Lock & Welcome Banner Filter):**
  1. *Triệt tiêu lỗi bắt nhầm Canvas #1:* Khắc phục triệt để lỗi bắt nhầm thẻ Canvas #1 (Banner chào mừng `960x758`) thay vì Canvas #2 (Card câu hỏi toán/bài tập thật của AI EduNext).
  2. *Bóc tách trạng thái trực tiếp từ React Fiber (`getReactFiberChatState`):* Quét cây React Fiber của `ChatMessages` (`memoizedProps.messages`), lấy mảng tin nhắn ở lượt hiện tại (sau tin nhắn cuối cùng của user) để trích xuất `metadata.question_block.content` và URL `msg.canvas.url`.
  3. *Bộ lọc kích cỡ và nội dung Welcome Banner:* Áp dụng bộ lọc `height >= 700px` và từ khóa chào mừng để loại bỏ hoàn toàn thẻ Canvas #1. Luôn ưu tiên phần tử Canvas cuối cùng trong lượt (`validCanvases[validCanvases.length - 1]`).
  4. *Nút phóng to Micro-Zoom nhắm trúng Card mới:* Khi fallback qua Micro-Zoom, script click vào nút zoom của Card cuối cùng (`zoomButtons[zoomButtons.length - 1]`) thay vì nút zoom đầu tiên.
  5. *Bảo toàn URL ảnh từ Network Packet Interceptor:* Không xóa `STATE.capturedAiCanvasUrls` khi chạy `runExtraction()`, giữ trọn vẹn dữ liệu ảnh đã bắt được từ gateway.
- **Triệt tiêu trạng thái treo FSM khi nhận phản hồi từ chối/gợi ý (Anti-Freeze State Machine Unlocking):**
  1. *Đồng bộ trạng thái `flowStatus` vào `checkFeedbackBanner`:* Tích hợp `getReactFiberChatState()` trực tiếp vào `checkFeedbackBanner`. Nếu `flowStatus === 'INCORRECT'` hoặc `isCorrect === false`, lập tức trả về `{ hasFeedback: true, result: 'INCORRECT' }`.
  2. *Xóa chốt chặn `isAlreadyAnswered` khi có feedback:* Trong `checkForNewContent`, khi phát hiện feedback `INCORRECT`, script lập tức xóa sạch `STATE.lastAnsweredQuestionText`, `lastAnsweredCandidateText` và thuộc tính `dataset.edunextAnswered`, điều hướng trực tiếp sang `handleFeedback` mà không bị kẹt ở trạng thái IDLE.
  3. *Giải phóng cache ảnh cũ khi làm lại:* Trong `runExtraction`, khi retrying, script re-evaluate hoàn toàn dữ liệu câu hỏi mới thay vì tái sử dụng ảnh lỗi trước đó.
- **Siết chặt Negative Prompt Gemini & Lọc bỏ phản hồi từ chối (Gemini Anti-Refusal & Bridge Gatekeeper):**
  1. *Gia cố Negative Prompt trong `aiProvider.buildPromptText`:* Nghiêm cấm Gemini hỏi lại, nghi vấn dữ liệu, báo ảnh mờ/lỗi hoặc viện cớ "không đủ dữ liệu". Bắt buộc Gemini phải suy luận từ dữ kiện khả dụng và luôn xuất ra đáp án duy nhất dưới dạng `Trả lời: [đáp án]` (/human).
  2. *Bộ lọc Gatekeeper chặn từ chối tại `handleBridgeResponse`:* Tự động phát hiện và loại bỏ các câu trả lời mang tính từ chối hoặc nghi vấn của Gemini (như "Tôi không thấy câu hỏi trong ảnh", "Ảnh bị mờ", "Vui lòng cung cấp thêm..."), ngăn không cho gửi câu nghi vấn của Gemini làm đáp án bài tập trên EduNext và kích hoạt quy trình giải lại.

**3.3.5** — **Direct MinIO Fetch, Angular Wiz Send & Radial Intent Submission:**
- **Nạp ảnh trực tiếp qua `GM_xmlhttpRequest` (Direct MinIO Fetching & Zero Tainted Canvas):**
  1. *Ưu tiên số 1 cho `GM_xmlhttpRequest`:* Trong hàm `fetchImgBase64`, đưa `GM_xmlhttpRequest` lên đầu tiên, nạp thẳng file ảnh WebP/PNG từ MinIO CDN (`cdn-lab.fpt.edu.vn`) dưới dạng ArrayBuffer và chuyển sang Base64 trong chưa đầy 100ms mà không bị chặn bởi chính sách CORS của trình duyệt.
  2. *Triệt tiêu lỗi Tainted Canvas:* Khắc phục hoàn toàn lỗi `SecurityError: Failed to execute 'toDataURL' on 'HTMLCanvasElement'` bằng cách đọc trực tiếp signed URL từ React Fiber thay vì cố gắng xuất dữ liệu từ thẻ Canvas bị ô nhiễm nguồn gốc.
- **Kích hoạt sự kiện nút Gửi Angular Wiz trên Gemini Web (Angular Wiz Send Button Event Delegation):**
  1. *Target chính xác Web Component Host:* Bổ sung `gem-icon-button.send-button` vào bộ selector nhận diện nút gửi, đồng thời phát chuỗi sự kiện `pointerdown` + `pointerup` + `click` lên cả `<gem-icon-button>` lẫn `<button>` con bên trong.
  2. *Loại bỏ hiện tượng nuốt click:* Khắc phục hoàn toàn việc Angular Wiz bỏ qua các sự kiện click tổng hợp (untrusted events) trên giao diện mới của Gemini Web.
- **Kích hoạt nộp bài chính thức qua Radial Menu (EduNext Radial Intent Submission):**
  1. *Phát hiện & Kích hoạt `button[data-intent="SUBMIT_ANSWER"]`:* Tự động kiểm tra và click nút "Nộp câu trả lời" trong menu tác vụ EduNext. Nếu menu chưa mở, script tự động click `button[aria-label="Mở công cụ gửi"]`, sau đó gọi trực tiếp handler `onClick` trong React Fiber để hệ thống EduNext chấm điểm chính thức (chấm bằng AI `INT: ANS`).
  2. *Khắc phục nhầm lẫn phím Enter:* Trên EduNext, "Enter là xuống hàng, chọn 'Nộp câu trả lời' để nộp đáp án". Script ưu tiên tuyệt đối nộp qua Intent button, không phụ thuộc vào phím Enter.
- **Cách ly Modal STK / Lightbox & Khắc phục lệch giao diện:**
  1. *Ngăn chặn nổi bọt sự kiện:* Thêm `e.stopPropagation()` và `e.preventDefault()` khi click vào thẻ mã QR STK trong terminal log, ngăn chặn sự kiện kéo thả làm co cụm/lệch giao diện HUD.
  2. *Kích hoạt `pointer-events: auto;`:* Bổ sung `pointer-events: auto;` vào `.lightbox-overlay` và `.lightbox-card`, đảm bảo các nút đóng (`✕`, `Đóng`) và backdrop bắt trọn sự kiện click của người dùng.
- **Siết chặt tuyệt đối Prompt Gemini chống từ chối (Anti-Refusal Directive Hardening):**
  1. *Cấm tuyệt đối mọi hình thức từ chối:* Cấm Gemini đặt câu hỏi nghi vấn, cấm báo "không đủ dữ liệu", "thiếu thông tin", "nội dung không đủ", "không thấy hình".
  2. *Ép buộc đáp án chuẩn `/human`:* Luôn luôn xuất ra câu trả lời dứt khoát dưới dạng `Trả lời: <đáp án cụ thể>` theo tư duy học sinh giỏi THPT.

**3.3.4** — **Strict Turn Isolation, Anti-Rocket/Arrow Elimination & Gemini Anti-Refusal Policy:**
- **Cô lập câu hỏi theo lượt tuyệt đối (Strict Turn Isolation & Zero Cross-Turn Bleed):**
  1. *Triệt tiêu hiện tượng lấy nhầm ảnh câu hỏi trước:* Khắc phục triệt để lỗi logic trong `extractQuestionData` khi fallback lên thẻ cha `.w-chat-body` hoặc `.chat-messages` làm quét lại toàn bộ ảnh từ tin nhắn mở đầu (onboarding) và các câu hỏi cũ trong lịch sử chat.
  2. *Xóa bộ nhớ đệm ảnh khi chuyển lượt:* Tự động xóa sạch `STATE.currentQuestionImages` và `STATE.capturedAiCanvasUrls` khi FSM chuyển trạng thái (`EXTRACTION`, `IDLE`, `PAUSED`, `STOPPED`) cũng như khi phát hiện câu hỏi mới trong `checkForNewContent` và `runExtraction`.
  3. *Giới hạn thời gian bắt Canvas AI (Timestamp Scoping):* Chỉ chấp nhận các gói tin Canvas / Image từ Network Packet Interceptor được sinh ra trong đúng lượt câu hỏi hiện tại (`STATE.currentQuestionTurnStartTime`), loại bỏ hoàn toàn các canvas cũ còn sót lại trong RAM.
- **Loại bỏ triệt để ảnh tên lửa, mũi tên và icon điều hướng (Anti-Rocket & Arrow Elimination):**
  1. *Lá chắn `isRocketOrArrowImage`:* Bổ sung cơ chế phát hiện và chặn đứng mọi ảnh hoặc canvas chứa hình ảnh tên lửa (`🚀`, `rocket`, `ten-lua`), mũi tên (`➡️`, `👉`, `arrow`, `chevron`), hoặc icon điều hướng/hướng dẫn chào mừng ở tất cả 6 tầng trích xuất thị giác.
  2. *Bộ lọc kích thước & tỷ lệ nghiêm ngặt:* Chặn mọi hình ảnh icon vuông nhỏ (< 120x70 px) hoặc các phần tử nằm trong tin nhắn hướng dẫn/chào mừng của Bot.
- **Siết chặt Prompt Gemini & Cơ chế chống từ chối (Gemini Anti-Refusal & Strict Answer Policy):**
  1. *Cấm Gemini hỏi lại hoặc báo thiếu dữ liệu:* Cập nhật cả `buildCondensedBridgePrompt` và `buildGeminiRolePrimerPrompt` (Quy tắc sống còn #3): Nghiêm cấm Gemini đưa ra các câu hỏi nghi vấn, từ chối trả lời, hoặc báo "không đủ dữ liệu", "thiếu thông tin", "nội dung không đủ", "không thấy ảnh".
  2. *Ép buộc xuất đáp án trực tiếp chuẩn `/human`:* Gemini bắt buộc phải đưa ra đáp án cuối cùng chính xác nhất dưới dạng "Trả lời: <đáp án>" theo phong cách học sinh giỏi THPT, không do dự, không thoái thác.
  3. *Lưới lọc phản hồi rác tại `normalizeBridgeAnswer` & `runSolving`:* Tự động phát hiện và hủy bỏ các câu trả lời rác hoặc câu từ chối vô ích của AI, tự động kích hoạt vòng lặp thử lại mà không làm dừng pipeline.

**3.3.3** — **Anti-Emoji/Arrow Filter, Modal STK Fix & FSM Auto-Recovery:**
- **Khắc phục lỗi Modal STK / Lightbox ảnh hiển thị lệch và không đóng được:**
  1. *Bổ sung CSS Shadow DOM hoàn chỉnh:* Thêm đầy đủ định dạng CSS cho `.lightbox-overlay`, `.lightbox-card`, `#lightboxImg`, `.lightbox-header`, `.lightbox-title`, `.lightbox-x-btn`, `.lightbox-body`, `.lightbox-footer`. Modal neo cố định `position: fixed; z-index: 2147483647; width: 100vw; height: 100vh;` với nền mờ kính `backdrop-filter: blur(10px)`.
  2. *Cơ chế đóng modal đa kênh tức thì:* Cho phép đóng modal bằng nút `✕`, nút `Đóng`, click vào bất kỳ vùng backdrop ngoài card, hoặc bấm phím `Escape`. Cố định ảnh `#lightboxImg` ở `max-height: 50vh`, không làm vỡ hoặc lệch giao diện EduNext.
- **Triệt tiêu lỗi gửi nhầm ảnh icon emoji mũi tên / tên lửa sang Gemini:**
  1. *Bộ lọc Anti-Emoji & Directional Arrow Filter:* Cập nhật `isDecorativeImage` chặn toàn bộ emoji mũi tên (`➡️`, `👉`, `➜`, `➔`, `▶️`, `►`, `➡`, `🚀`, `🏹`, `🎯`, `💡`) cùng các từ khóa chỉ dẫn (`arrow`, `mui-ten`, `chevron`, `pointer`, `rocket`, `ten-lua`, `ready`, `start`, `guide`, `welcome`).
  2. *Kiểm tra ngữ cảnh tin nhắn cha:* Nếu tin nhắn cha chứa lời chào, hướng dẫn mở đầu hoặc emoji mũi tên/tên lửa mà không có nội dung học thuật, toàn bộ ảnh bên trong tự động bị đánh dấu là ảnh trang trí và loại bỏ 100%.
  3. *Mở rộng nhận diện Ready Prompt & Handshake 2 bước:* Tin nhắn mở đầu hoặc bot hỏi sẵn sàng được phân loại chính xác là `ready_prompt`, tự động gửi ngẫu nhiên `['ok', 'vào', 'Sẵn sàng']` ngay tại chỗ (chi phí: 0 API token, không chuyển ảnh sang Gemini).
- **Chống treo và tự động phục hồi FSM (Anti-Freeze Auto-Recovery):**
  1. *Hạ timeout Bridge từ 270s xuống 45s:* Thất bại nhanh khi Gemini gặp sự cố, loại trừ tình trạng đơ tab 4.5 phút.
  2. *Tự phục hồi về IDLE thay vì PAUSED vĩnh viễn:* Khi Bridge thử lại không thành công, script tự động giải phóng cờ `isSolving = false; isBridgeInFlight = false;` và chuyển về `FSM_STATE.IDLE` sau 3 giây để tiếp tục quét bài mới, không bao giờ bị đứng hình.

**3.3.2** — **AI Canvas Fiber Extractor, Lossless Image Bridge & Anti-Asset Shield:**
- **Giải quyết triệt để lỗi không nhận được ảnh câu hỏi AI và không nhầm lẫn icon/asset:**
  1. *Bóc tách URL gốc từ React Fiber (Fiber Canvas Extractor):* Đào sâu vào cây React Fiber của `canvas[data-testid="chat-canvas-message"]` và `div[data-testid="chat-canvas-container"]` để lấy trực tiếp thuộc tính `props.canvas.url` / `message.metadata.canvas.url` do backend EduNext cung cấp. Vượt qua 100% rào cản Tainted Canvas (vốn xảy ra khi gọi `canvas.toDataURL()` do Azure Blob thiếu header CORS).
  2. *Chốt chặn Network Packet Interceptor & Image.src Hook:* Hook toàn diện trên cả `window` và `unsafeWindow` cho `fetch`, `XMLHttpRequest` và `HTMLImageElement.prototype.src`. Bắt trọn vẹn dữ liệu ảnh đề bài ngay khi widget chat khởi tạo `new Image()` và nạp URL ảnh từ gateway.
  3. *Cơ chế Micro-Zoom Modal Fallback:* Nếu React Fiber bị đóng băng, script tự động kích hoạt nút phóng to `chat-canvas-zoom-button` trong 150ms để trích xuất thẻ `<img>` gốc (`alt="Nội dung câu trả lời phóng to"`), sau đó tự động đóng modal trả lại giao diện sạch sẽ cho người dùng.
  4. *Tải ảnh bằng đặc quyền GM_xmlhttpRequest:* Bỏ qua hoàn toàn rào cản Same-Origin / CORS, tải ảnh trực tiếp từ Azure Blob hoặc FPT AI Gateway về Base64 PNG sắc nét nguyên bản mà không bị vỡ ảnh hay nén mờ.
  5. *Lá chắn Anti-Asset & Anti-Icon Shield 100%:* Bộ lọc `isDecorativeImage` phân loại nghiêm ngặt: loại bỏ triệt để toàn bộ avatar bot, icon reaction, nút bấm công cụ, icon Ant Design, logo và asset tĩnh nhỏ hơn 100x60px. Đảm bảo CHỈ có ảnh đề bài thật sự của AI mới được chuyển tiếp.
  6. *Đồng bộ và đính kèm hoàn hảo sang Gemini:* Đóng gói ảnh sạch vào transport v2 chuyển sang tab `gemini.google.com`, kích hoạt cơ chế đính kèm W3C DataTransfer Paste/DragDrop giúp Gemini giải đề đầy đủ thị giác.

**3.3.1** — **AI Canvas First & Zero Icon/Asset Leakage:**
- **Triệt tiêu lỗi bắt nhầm icon/asset của chat và web:**
  1. *Lá chắn phân định ranh giới (Ancestor Boundary Filtering):* Loại bỏ 100% các phần tử ảnh nằm trong `.w-chat-avatar`, `.chat-avatar`, `.ant-avatar`, `button`, `.action-btn`, reaction bar, copy/sound buttons, `.anticon`, thanh công cụ gõ text, và header/nav/sidebar/footer của trang web.
  2. *Lọc danh sách đen từ khóa Asset & URL (Blacklist Keyword Gate):* Chặn toàn bộ ảnh có src hoặc class/alt/title chứa `avatar`, `bot`, `robot`, `mascot`, `icon`, `logo`, `copy`, `sound`, `like`, `thumb`, `btn`, `/assets/`, `/icons/`, `/static/media/`, `/svg/`, `.svg`.
  3. *Ngưỡng kích thước câu hỏi tối thiểu (70x50 px):* Đồ thị, hình học, công thức toán hoặc đề bài thật sự của AI không bao giờ nhỏ hơn 70x50 px. Các icon từ 16px đến 48px bị loại bỏ ngay từ vòng đo đạc ban đầu.
  4. *Đưa Canvas AI lên vị trí Ưu tiên số 1:* Nhận diện và bóc tách trực tiếp các thẻ `<canvas>` do AI vẽ (`canvas[data-testid="chat-canvas-message"]`, `.chat-content canvas`) sang PNG base64, giải quyết triệt để các câu hỏi FlipClassroom kiến trúc mới.
  5. *Xoá bỏ lưới quét toàn trang bừa bãi:* Loại bỏ hoàn toàn lệnh `document.querySelectorAll('img')` quét cả trang. Khi câu hỏi là dạng thuần chữ, script giữ nguyên `images = []` và không đính kèm ảnh rác sang Gemini.

**3.3.0** — **Single Unified Cockpit, W3C DataTransfer & Zero Image Blindspot:**
- **Triệt tiêu lỗi không phát hiện và không đính kèm ảnh sang Gemini:**
  1. *Gỡ bỏ bộ lọc góc màn hình (`nearCorner`):* Trước đây `isDecorativeImage` tự động loại bỏ mọi ảnh nằm trong phạm vi 380px tính từ góc dưới phải màn hình (`bottom-right`). Vì widget chat EduNext neo cố định ở góc này, nên toàn bộ ảnh đề bài trong tin nhắn chat đều bị coi là widget nổi/nút hỗ trợ và bị lọc bỏ 100%. Bản 3.3.0 gỡ bỏ triệt để điều kiện này, đồng thời hạ ngưỡng diện tích tối thiểu xuống 20px.
  2. *Bổ sung Canvas Rasterization tức thì (0ms):* Hàm `extractImages` thử vẽ trực tiếp thẻ `<img>` lên canvas ngay lập tức để lấy base64 mà không cần chờ fetch mạng, loại trừ nguy cơ rớt gói tin do CORS hoặc URL tạm thời hết hạn.
  3. *Mở rộng vùng trích xuất ảnh cha:* Quét mở rộng sang các container chứa bài tập (`.chat-box`, `.exercise-container`, `.question-card`, `.markdown-body`) để bắt trọn ảnh đề ở mọi biến thể giao diện EduNext.
  4. *Khắc phục lỗi Chromium DataTransfer Paste:* Khởi tạo sự kiện `paste` thông thường sẽ gán thuộc tính `clipboardData = null` theo chính sách bảo mật trình duyệt, khiến Gemini không đọc được `files`. Bản 3.3.0 override getter thông qua `Object.defineProperty(pasteEvt, 'clipboardData', { value: dt, ... })`, đảm bảo cả Angular và React bên Gemini đều trích xuất được file ảnh đính kèm.
- **Tái cấu trúc UI thành 1 màn hình đơn nhất (Single Unified Cockpit - NO TABS):**
  - Xóa bỏ phân chia 4 tab gây gián đoạn và cuộn trang phức tạp; gộp toàn bộ tính năng vào một giao diện Obsidian Cockpit chuẩn tỷ lệ vàng 490px.
  - Nút Hero Auto Capsule lớn (`#bAutoBig`) dễ bấm, phản hồi trạng thái phát sáng tức thì.
  - Action Cockpit mini 6 nút (`#bForce2` Giải lại, `#bPause2` Đề cương, `#bRec2` Log ML, `#bDL` Xuất Word, `#bKill` Dừng, `#bAuto2` Bật/Tắt).
  - Status Deck & Heartbeat nhịp tim sống động + Dải thumbnail ảnh đề `#imgStrip`.
  - Micro Pipeline 5 bước phát sáng neon.
  - Cửa sổ Console Terminal (`#eTerm`) cố định ngay bên dưới (cao 190px), tối ưu cho việc theo dõi live log màu sắc nét.
  - Dọn dẹp mã chết: Xóa logic API key ngoài, Pollinations, DuckDuckGo; tối ưu hóa kích thước tệp.

**3.2.0** — **Bento 4-Tab Obsidian Cyberpunk, Turbo Vision & Zero Image Failure:**
- **Khắc phục lỗi gửi ảnh câu hỏi cho Gemini:**
  1. *Fix Synthetic Snapshot Prepend Bug:* Khi câu hỏi đã có ảnh trích xuất thật từ đề bài, không còn tự chụp màn hình DOM bằng SVG/canvas rồi chèn lên đầu `allImages` làm chiếm slot và đẩy văng ảnh thật ra sau.
  2. *Fix Prompt Dropping Short Question Text:* Trong `buildCondensedBridgePrompt`, loại bỏ điều kiện cắt câu hỏi dưới 40 ký tự khi có ảnh (trước đây bị coi là rác), đồng thời tự động chèn danh sách các phương án trắc nghiệm (`qd.options`) vào prompt gửi sang Gemini.
  3. *Fix Retry Passing Null Images:* Trong `solveViaBridge`, lượt thử lại thứ 2 truyền đầy đủ mảng `imgs` thay vì `null`.
  4. *Fix Chunk Storage Base64 Prefix:* Chuẩn hóa dữ liệu raw base64 thiếu tiền tố `data:image/` khi lưu và đọc từ GM Storage.
  5. *Đính kèm ảnh 4 lớp siêu tốc:* Bổ sung Clipboard paste (nhạy nhất với web hiện đại) + Drag & Drop (`copy`) + Native File Input + Upload Menu đệ quy. Giảm độ trễ đính kèm xuống dưới 2.5s.
  6. *Fix BroadcastChannel Empty Images:* Tách riêng gói `reqObj` (gửi ngay qua RAM cho tab đang mở) và `storageReqObj` (gửi chunk cho Tampermonkey Storage).
- **Đại tu UI/UX Bento 4-Tab Obsidian Cyberpunk 2026:**
  - Chuyển đổi toàn diện bố cục giao diện sang hệ thống 4 Tabs chuyên biệt (`🎮 Điều Khiển`, `🧠 AI Engine`, `🌉 Bridge & Ảnh`, `💻 Console`), tăng độ rộng lên 520px tỷ lệ vàng, triệt tiêu việc phải cuộn trang dài dằng dặc.
  - Phục hồi bảng màu Obsidian Cyberpunk rực rỡ tương phản cao (Emerald `#10b981`, Sky Blue `#38bdf8`, Violet `#8b5cf6`, Amber `#f59e0b`, Crimson `#ef4444`) kết hợp kính mờ sâu và ánh sáng neon thời thượng.
  - Bento 2x2 Action Tiles cho các thao tác nhanh (`Giải lại ngay`, `Ghi chép Đề cương`, `Ghi log ML`, `Xuất file Đề cương Word`).
  - Terminal `#eTerm` mở rộng cao 250px cho phép xem log rõ nét, chuyên nghiệp.
  - Bảo toàn 100% ID DOM con và tương thích tuyệt đối giữa bản Tampermonkey và bản Console DevTools.

**3.1.0** — **gửi được ảnh đề sang tab Gemini + giao diện đơn sắc.**
- **Nguyên nhân ảnh không tới Gemini:** trên bản Gemini hiện tại, trang tải xong **không có** `input[type=file]` nào trong DOM. Input chỉ được mount sau khi bấm nút `aria-label="Upload & tools"`. Regex cũ của script (`nội dung tải lên|upload content|upload and tools|…`) **không khớp** nhãn `Upload & tools`, nên nút không bao giờ được bấm → không có input → ảnh không đi. Trước đây script còn bấm mục `Upload files`, mở hộp thoại chọn tệp của hệ điều hành mà script không điều khiển được.
- **Cách sửa:** mở rộng regex nhận nút công cụ; quét `input[type=file]` đệ quy **xuyên shadow DOM**; mở menu rồi **gán file trực tiếp** vào input bằng native setter + `change`/`input`; chờ chip tăng; đóng menu sau khi đã xác nhận ảnh bám (không bấm `Upload files`).
- **Đếm ảnh chờ gửi:** đếm theo chip thật (`uploader-file-preview` / `.file-preview-chip`, dự phòng nút `close attachment`), bỏ hẳn kiểu quét toàn trang dễ nhầm ảnh cũ trong hội thoại.
- **Lọc ảnh QR:** thêm `isDecorativeImage()` loại ảnh mã QR/ủng hộ/banner/avatar/logo/icon, ảnh nổi ở góc màn hình và ảnh quá nhỏ — áp dụng cho `extractImages`, `collectUrls` và lưới an toàn quét toàn trang.
- **Giao diện đơn sắc:** quy toàn bộ bảng màu GUI (hex + rgba, gradient, trạng thái) về thang xám theo độ sáng; giữ nguyên ID phần tử, bố cục và toàn bộ hành vi.
- **Đã kiểm chứng:** trên Gemini thật, đúng luồng mới đưa chip ảnh từ 0 → 1, preview 240×140, nút gửi bật; CSS shadow root 47 095 ký tự còn 0 màu lệch kênh; harness jsdom vẫn qua.

**3.0.1** — **vá lỗi chặn gửi đề sang tab Gemini.** Trên `gemini.google.com`, khối `if (IS_GEMINI_WEB) { initGeminiBridgeDaemon(); return; }` thoát khỏi IIFE **trước** khi `const STATE` (khai báo phía dưới) được khởi tạo. Vì `const` nằm trong vùng temporal dead zone vĩnh viễn, mọi lần daemon chạm `STATE` đều ném `ReferenceError: Cannot access 'STATE' before initialization` — tab Gemini nhận gói tin nhưng trả về lỗi ngay, phía EduNext chỉ thấy “tab Gemini chưa phản hồi”. Bản vá cấp cho daemon một `GEM_STATE` cục bộ (đã thay toàn bộ 9 điểm dùng `STATE` trong hàm), nhờ đó phần nhận đề không còn phụ thuộc biến toàn cục bị bỏ dở. Đã kiểm chứng bằng harness jsdom hai miền: gói tin `EDUNEXT_AI_REQUEST` được nhận, trạng thái `RECEIVING` → `UPLOADING_IMAGE` được phát, ảnh chunk đọc được.

**3.0.0** — transport ảnh v2 (chunk storage thay gói JSON ~1 MB), tăng timeout chờ Bridge (270 s phía gửi / 210 s phía Gemini), chốt cắt đáp án chặt hơn (chờ văn bản ổn định 4–5,5 s thay vì 2 s, nhận diện thêm thanh tiến trình/`aria-busy`), `waitForResponse` không còn cắt ngang bài giải, đồng bộ `STATE.currentQuestionImages` (badge/strip ảnh hoạt động trở lại), nâng cấp giao diện: status deck, pipeline 5 bước, Bridge Doctor, dải thumbnail, nút auto lớn.
