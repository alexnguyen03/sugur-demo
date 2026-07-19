# Roadmap: Xây dựng "Wails for Java"

> Thư viện desktop app cho Java theo triết lý Wails/Tauri: backend Java + frontend React chạy trên webview hệ thống, build ra một binary duy nhất ~20–30MB bằng GraalVM native-image.

**Tổng thời gian dự kiến:** 8–12 tháng (side project, ~10h/tuần)
**Nguyên tắc xuyên suốt:** App trước, lib sau · Scope tàn nhẫn · Mỗi milestone phải ra thứ nhìn thấy được

---

## Giai đoạn 0 — Nền móng kiến thức (2–3 tuần)

Hai điểm nghẽn kỹ thuật phải học thật, không vibe code được. Học qua bài tập nhỏ, không học chay.

### Việc cần làm

| Tuần | Nội dung | Bài tập kiểm chứng |
|---|---|---|
| 1 | FFM API (Project Panama, JDK 22+): Arena, MemorySegment, Linker, upcall stub | Bind `libsqlite3` của OS: mở db, chạy query, in kết quả |
| 2 | GraalVM native-image: closed-world analysis, reflection config, reachability metadata | Build bài tập tuần 1 thành binary native < 10MB, cố tình gây lỗi reflection để hiểu cách nó fail |
| 3 (đệm) | Đọc source webview_java (JNA) + webview_go để nắm trình tự gọi API C của libwebview | Ghi chú lại flow: create → bind → navigate → run loop → destroy |

### Tooling cài đặt

- GraalVM JDK 24+ (bản Oracle GraalVM, không phải Community, để có `-Os`)
- Maven hoặc Gradle (khuyên Gradle vì viết plugin CLI sau này dễ hơn)
- Visual Studio Build Tools (nếu dev trên Windows) / Xcode CLT (macOS) / gcc + webkit2gtk-dev (Linux)
- Node.js + pnpm cho phần frontend

### Điều kiện qua giai đoạn
✅ Binary native từ code FFM chạy được trên máy bạn, và bạn giải thích được vì sao nó chạy.

---

## Giai đoạn 1 — Proof of concept: App trước, lib sau (1–2 tháng)

Làm một app thật (gợi ý: SQL client mini cho SQLite) để chứng minh toàn bộ con đường kỹ thuật. Code bẩn được phép. Chỉ cần chạy trên máy của bạn.

### Milestone 1.1 — Cửa sổ đầu tiên (tuần 1–2)
- FFM binding tối thiểu cho libwebview: `webview_create`, `webview_set_title`, `webview_set_size`, `webview_navigate`, `webview_run`, `webview_destroy`
- Mở được cửa sổ hiển thị `https://example.com`
- Xử lý đúng thread model: macOS cần chạy trên main thread (`-XstartOnFirstThread` khi chạy JVM)
- 🎉 **Khoảnh khắc ăn mừng #1: cửa sổ mở lên**

### Milestone 1.2 — React render (tuần 3–4)
- Scaffold Vite + React + TS trong thư mục `frontend/`
- Dev mode: webview navigate tới `http://localhost:5173` → được HMR miễn phí
- Production mode: nhúng `dist/` vào resources, serve qua custom scheme hoặc mini HTTP server localhost (tạm chấp nhận, tối ưu sau)
- 🎉 **Khoảnh khắc #2: React app chạy trong cửa sổ native**

### Milestone 1.3 — Bridge thủ công (tuần 5–6)
- `webview_bind` để expose hàm Java cho JS: JS gọi `window.invoke("method", args)` → Java xử lý → trả JSON
- Viết tay 3–4 method cho app SQL client: `connect`, `query`, `listTables`
- Event từ Java về JS qua `webview_eval` (dispatch trên UI thread)
- Serialization: dùng records + serializer viết tay hoặc dsl-json (tránh Jackson databind)

### Milestone 1.4 — Binary native đầu tiên (tuần 7–8)
- Build native-image với flags: `-Os --gc=serial --no-fallback -march=compatibility`
- Viết reachability metadata cho phần FFM (downcall/upcall cần khai báo)
- Đo size, dùng `-H:BuildReport` xem class nào chiếm chỗ
- 🎉 **Khoảnh khắc #3: binary < 30MB, double-click chạy luôn, không cần JVM**

### Điều kiện qua giai đoạn
✅ App SQL client chạy từ binary native trên máy bạn. Bạn đã đụng và vượt qua cả hai điểm nghẽn.

---

## Giai đoạn 2 — Rút xương thành lib (1–2 tháng)

Tách phần tái sử dụng khỏi app. App SQL client trở thành `examples/sql-client` — app mẫu đầu tiên.

### Cấu trúc repo đề xuất (monorepo)

```
yourlib/
├── core/              # FFM binding + Window/Webview API
├── bridge/            # IPC, serialization, event bus
├── processor/         # annotation processor (giai đoạn 3)
├── cli/               # CLI tool (giai đoạn 3)
├── runtime-js/        # package npm: JS runtime phía frontend
├── templates/         # project templates (giai đoạn 3)
├── examples/
│   └── sql-client/
├── docs/
└── .github/workflows/
```

### Việc cần làm

**Module `core`:**
- API công khai: `Application`, `Window` (builder pattern: title, size, resizable, min/max size)
- Quản lý lifecycle + thread model che giấu khỏi người dùng (tự dispatch về UI thread)
- Nạp native lib: bundle libwebview cho 3 OS trong resources, extract ra temp dir lúc chạy (hoặc link tĩnh nếu native-image)

**Module `bridge`:**
- Protocol JSON-RPC nhẹ: request id, method, params, result/error
- Promise phía JS ↔ CompletableFuture phía Java
- Event bus 2 chiều: `events.emit()` / `events.on()` cả hai phía

**Module `runtime-js`:**
- Package npm `@yourlib/runtime`: hàm `invoke()`, `events`, typing cơ bản
- Đây là thứ template React sẽ import

**Thiết kế API — viết README trước khi code:**
- Viết README với code mẫu "cách người ta sẽ dùng" TRƯỚC, code theo sau
- Mục tiêu trải nghiệm: người dùng viết < 15 dòng Java để mở app đầu tiên

### Điều kiện qua giai đoạn
✅ `examples/sql-client` chạy hoàn toàn trên API công khai của lib, không đụng internal.

---

## Giai đoạn 3 — Developer Experience (2–3 tháng)

Phần quyết định lib sống hay chết: biến "code chạy được" thành "người lạ dùng được trong 5 phút".

### Milestone 3.1 — Annotation processor (tuần 1–3)
- Annotation `@Bind` trên class/method
- Compile-time sinh ra:
  - Dispatcher Java (switch trên method name — không reflection, thân thiện native-image)
  - File `.d.ts` + JS stub: dev React gọi `Backend.getUser()` có autocomplete và type đầy đủ
  - Reachability metadata cho native-image tự động
- Hỗ trợ kiểu: primitives, String, records (nested), List/Map, CompletableFuture cho async

### Milestone 3.2 — CLI (tuần 4–7)

Viết CLI bằng chính Java + picocli, build native-image → chính CLI là demo cho lib.

| Lệnh | Chức năng |
|---|---|
| `ylib init <name>` | Scaffold project từ template (hỏi: React/Vue/Svelte/Vanilla, JS/TS) |
| `ylib dev` | Chạy Vite dev server + app Java song song, watch file Java → rebuild + restart, frontend có HMR |
| `ylib build` | npm build → nhúng assets → compile native-image → binary trong `build/bin/` |
| `ylib build --target=jlink` | Fallback: jlink runtime thay vì native-image (build nhanh hơn cho dev) |
| `ylib package` | Đóng gói: `.msi`/`.exe` (Windows), `.dmg`/`.app` (macOS), `.deb`/`.AppImage` (Linux) |
| `ylib doctor` | Kiểm tra môi trường: GraalVM, native toolchain, Node, WebView2 runtime |
| `ylib generate` | Chạy riêng codegen bindings (bình thường tự chạy khi compile) |

### Milestone 3.3 — Templates + CI (tuần 8–10)
- Template `react-ts` hoàn chỉnh: Vite config sẵn, `@yourlib/runtime` cài sẵn, ví dụ gọi backend
- GitHub Actions matrix 3 OS: build + test + build example app native trên cả Windows/macOS/Linux
- Đây là lúc lib lần đầu được kiểm chứng đa nền tảng — dự trù nhiều thời gian debug CI

### Milestone 3.4 — Docs (tuần 11–12)
- Docs site (Docusaurus/VitePress): Quickstart 5 phút, Guide (window, bind, events, build), API reference
- Trang "How it works" giải thích kiến trúc — người dùng lib nền tảng rất cần niềm tin này
- CONTRIBUTING.md + issue templates

### Điều kiện qua giai đoạn
✅ Một người lạ (nhờ bạn bè test) đi từ `ylib init` đến app chạy được trong < 10 phút không cần hỏi bạn.

---

## Giai đoạn 4 — Hardening + Ra mắt v0.1 (1 tháng)

### Scope khóa cứng cho v0.1
**Có:** một cửa sổ · webview · `@Bind` + sinh TS · events · CLI init/dev/build/package · template React · 3 OS desktop
**Không (để sau, ghi rõ trong README):** multi-window, system tray, native menu, dialogs, auto-update, mobile

### Việc cần làm
- Test trên máy thật cả 3 OS (mượn máy/VM nếu cần), đặc biệt Windows 10 không có WebView2 sẵn → CLI package cần kèm bootstrapper cài WebView2 runtime
- Error message tử tế cho 10 lỗi phổ biến nhất (thiếu GraalVM, thiếu toolchain, port bận...)
- Versioning: SemVer, đánh dấu API `@Experimental` nếu chưa chắc
- License MIT hoặc Apache-2.0 (đã chọn từ commit đầu)
- Publish: Maven Central (core/bridge/processor), npm (`@yourlib/runtime`), Homebrew/Scoop cho CLI (có thể để v0.2)

### Ra mắt
- Bài viết "I built Wails for Java" — kể chuyện kỹ thuật thật (FFM, native-image, những chỗ suýt bỏ cuộc)
- Đăng: r/java, Hacker News (Show HN), Twitter/X, viblo + group Java Việt Nam
- Chuẩn bị tinh thần trả lời issue dồn dập 2 tuần đầu — đó là tín hiệu tốt

---

## Giai đoạn 5 — Hậu ra mắt → v1.0 (3–6 tháng, theo nhu cầu người dùng)

Thứ tự ưu tiên gợi ý (điều chỉnh theo issue thực tế):

1. **v0.2** — Native dialogs (open/save file, message box) + clipboard: app thật nào cũng cần ngay
2. **v0.3** — Multi-window + window events (close, focus, resize)
3. **v0.4** — System tray + native menus
4. **v0.5** — Custom scheme asset serving hoàn chỉnh (bỏ hẳn localhost server), CSP mặc định an toàn
5. **v0.6** — Auto-updater (khó, cân nhắc tích hợp bên thứ ba)
6. **v1.0** — Khi: API ổn định 3+ tháng không breaking change, ≥ 2 app thật ngoài đời dùng production, docs đầy đủ, CI xanh ổn định

### Việc vận hành song song
- Trả lời issue trong 48h giai đoạn đầu (không cần fix ngay, chỉ cần phản hồi)
- Gắn nhãn `good-first-issue` để kéo contributor
- Changelog mỗi release, blog ngắn cho mốc lớn

---

## Rủi ro chính và đối sách

| Rủi ro | Đối sách |
|---|---|
| Kẹt ở FFM/segfault không debug nổi | Luôn có bản đối chiếu webview_go/webview_java; test từng hàm C một; giữ binding tối thiểu |
| native-image build được máy mình, fail máy khác | CI 3 OS từ sớm (giai đoạn 3.3, đừng để cuối); test Windows 10 sạch |
| WebKitGTK trên Linux phân mảnh phiên bản | Ghi rõ distro hỗ trợ (Ubuntu 22.04+); AppImage bundle dependency |
| "Tuần thứ 6 trầm cảm" — mất động lực | Milestone nhỏ có kết quả nhìn thấy; đăng tiến độ công khai (build in public) để có áp lực dương |
| Scope creep — muốn thêm mãi tính năng | Danh sách "Không làm ở v0.1" dán ngay đầu README; mọi ý tưởng mới → ghi vào issue, không code |
| Ra mắt xong không ai dùng | Vẫn thắng: kỹ năng FFM + native-image + APT + OSS ops là tài sản nghề nghiệp thật |

---

## Tóm tắt timeline

| Giai đoạn | Thời gian | Kết quả |
|---|---|---|
| 0 — Nền móng | 2–3 tuần | Hiểu FFM + native-image qua bài tập |
| 1 — PoC app | 1–2 tháng | SQL client native binary < 30MB |
| 2 — Rút lib | 1–2 tháng | Core + Bridge + API công khai |
| 3 — DX | 2–3 tháng | @Bind codegen, CLI, template, CI, docs |
| 4 — v0.1 | 1 tháng | Ra mắt công khai |
| 5 — v1.0 | 3–6 tháng | Ổn định theo nhu cầu thật |