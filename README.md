# Edunext_fe

Userscript tự động hoá việc làm bài tập trên EduNext (FPT), dùng **Cross-Tab Bridge** sang tab `gemini.google.com` đang mở để lấy đáp án. Không có API key, không có server trung gian, không có kho đáp án dùng chung.

Repo chỉ chứa đúng hai tệp:

| Tệp | Dùng cho | Kích thước |
|---|---|---|
| `edunext_ULTRA.user.js` | Cài qua Tampermonkey / Violentmonkey | ~750 KB |
| `edunext_ULTRA_console.js` | Dán vào DevTools Console | ~748 KB |

Hai tệp là **cùng một mã nguồn**: bản console là bản userscript bỏ 34 dòng header `==UserScript==`. Nếu sửa một bản, bản kia phải build lại — đừng sửa tay cả hai.

Phiên bản hiện tại: **2.2.7**. Trường `@description` trong header vẫn là câu cũ từ bản trước (`AUTO Giải edunext bản prenium`), chưa được cập nhật; đây là sai sót văn bản, không phải mô tả tính năng.

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
| `EDUNEXT_BRIDGE_HEARTBEAT` | Nhịp tim của tab Gemini, ghi mỗi **800 ms** |
| `EDUNEXT_BRIDGE_PING` / `EDUNEXT_BRIDGE_PONG` | Bắt tay để biết tab Gemini còn sống |
| `EDUNEXT_BRIDGE_STATUS` / `EDUNEXT_BRIDGE_CLOSED` | Trạng thái và mốc thời gian tab Gemini bị đóng |
| `EDUNEXT_GEMINI_MODEL` | Model ép chọn bên Gemini (mặc định `3.1 Pro`) |
| `EDUNEXT_AI_MODE` | Tuyến AI (luôn `bridge` ở bản này) |

Hệ quả trực tiếp của thiết kế này:

- Tab Gemini phải **đang mở** và đã đăng nhập. Script không tự đăng nhập, không vượt CAPTCHA, không dùng cookie nào của bạn.
- Nếu tab Gemini bị đóng, heartbeat dừng; tab EduNext nhận ra qua `EDUNEXT_BRIDGE_CLOSED` và báo lỗi thay vì chờ vô hạn.
- **Bản console không bridge được sang tab Gemini.** Khi không có userscript manager, tệp tự cài một shim: `GM_setValue`/`GM_getValue` ghi vào `localStorage` với tiền tệ `EDUNEXT_GM_` và lắng nghe sự kiện `storage`. `localStorage` là **theo origin**, mà `fsc-edunext.fpt.edu.vn` và `gemini.google.com` là hai origin khác nhau — hai tab không đọc được dữ liệu của nhau. Kênh cross-tab chỉ hoạt động trên bản cài qua Tampermonkey/Violentmonkey (xem mục 10).

## 3. Trích đề và xử lý ảnh

Đề bài trên EduNext thường nằm **dưới dạng ảnh** (đề bài đăng ảnh thay vì chữ). Đây là phần dễ hỏng nhất và đã được sửa nhiều lần.

Cách làm hiện tại:

1. `collectQuestionImages()` quét **cả trang**, không chỉ trong khung tin nhắn, vì ảnh đề nằm ở khối bài tập bên ngoài `.chat-messages`. Bộ lọc nhận cả endpoint nội bộ `ai-gw…/images/crop_image_<hash>.png`.
2. `fetchImgBase64()` có 4 lớp dự phòng, theo thứ tự: `fetch` (không gửi credentials) → `new Image()` với `crossOrigin=anonymous` + canvas → `GM_xmlhttpRequest` (ép `responseType:'arraybuffer'`) → `null`.
3. Kết quả chỉ được chấp nhận nếu là `data:image/*`. URL hết hạn trả về JSON 404 sẽ **không** bị coi là ảnh — đây là lỗi cụ thể đã từng xảy ra và đã được chặn bằng kiểm tra MIME.
4. Trước khi gửi, ảnh được nén theo bậc `1280/0.8 → 1100/0.7 → 900/0.6 → 760/0.5`. Ngưỡng là **900.000 ký tự JSON** (`LIMIT`, tính trên chuỗi sau stringify, không phải byte ảnh); chỉ khi vẫn vượt **1.800.000** ký tự mới cắt bớt **số lượng** ảnh, và luôn giữ ảnh đầu tiên. Bản cũ xoá sạch `images` khi quá nặng, khiến Gemini nhận đề không có hình — lỗi đó đã bị loại bỏ.
5. Tab Gemini ưu tiên dùng base64 gửi kèm, không phụ thuộc CORS; URL chỉ dùng để bù khi thiếu base64.

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
| Cú pháp | `node --check` trên cả hai tệp | Pass |
| Đồng bộ hai bản | So sánh bản console với userscript | Chênh đúng 34 dòng header, phần thân giống nhau |

## 6. Chưa kiểm chứng

Cần nói rõ để không ai đọc bảng trên rồi tưởng quá mức:

- **Chưa chạy trọn chuỗi** "userscript cài sẵn trên EduNext → tự bấm giải → tab Gemini trả đáp án → nộp bài → đọc nhận xét". Từng nửa đã kiểm riêng với ảnh thật, nhưng toàn tuyến chưa chạy một mạch trong điều kiện thật.
- **Ba nhánh lỗi mới** (`image_missing` đính lại ảnh, `rate_limited`, `refusal`) mới kiểm bằng văn bản mô phỏng, **chưa** kích hoạt được trên Gemini thật.
- **Không có số liệu độ chính xác.** Không có thống kê tỉ lệ đúng trên N câu, vì mới chạy thử nghiệm vài câu. Bất kỳ con số "95% đúng" nào bạn thấy ở nơi khác đều không xuất phát từ repo này.
- **Chế độ console không bridge được sang tab Gemini.** Shim tự cài dùng `localStorage` theo origin (xem mục 2), nên hai tab khác domain không thấy nhau. Chưa có giải pháp cho trường hợp này.

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

**Mã nguồn bị làm rối một phần.** Tệp có khối `_0xBRS` (giải mã chuỗi) và nhiều đoạn có comment cảnh báo bản quyền. Đây là mã 750 KB chứa cả GUI, FSM, bộ xuất Word và các bộ lọc — khó đọc hết, và việc làm rối khiến việc kiểm toán độc lập khó khăn hơn. Nếu bạn cần tin tuyệt đối, đây là trở ngại thật, không phải cảnh báo hình thức.

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
