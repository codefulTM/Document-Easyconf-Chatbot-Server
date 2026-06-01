# Kế hoạch triển khai Live Multi-Agent cho Easyconf Chatbot

## 1. Mục tiêu

Tài liệu này mô tả kế hoạch xây dựng hệ thống **multi-agent ở chế độ live** cho Easyconf Chatbot. Hệ thống live này chạy **song song** với text chatbot hiện tại, không thay thế và không loại bỏ text chatbot. Text chatbot vẫn giữ nguyên luồng xử lý hiện có; live chatbot chỉ xây thêm một kênh tương tác mới dùng **giọng nói** và **chia sẻ màn hình**.

Nguồn tham khảo chính:

- `REPORT.md` trong thư mục này: mô tả luồng microphone, screen sharing, WebSocket bridge và Gemini Live.
- `https://github.com/googleapis/js-genai`: SDK `@google/genai`, có `ai.live`, `session.sendRealtimeInput()`, function calling, MCP support.
- `https://github.com/google/adk-js`: ADK TypeScript, hỗ trợ định nghĩa `LlmAgent`, tool bằng code, multi-agent workflow, routed agent, sequential/parallel orchestration.
- Các repo Easyconf hiện tại: `Easyconf-Chatbot-Server`, `Easyconf-FE-Client`, `Easyconf-BE`, `Easyconf-FE-Admin`, `Easyconf-Tracking-System`.

Kết quả mong muốn:

- Người dùng nói chuyện với chatbot bằng micro.
- Người dùng có thể bật chia sẻ màn hình để chatbot hiểu đang nhìn trang nào, nút nào, form nào, danh sách hội nghị nào.
- Server giữ Gemini API key, FE không gọi trực tiếp Gemini bằng API key.
- Live agent vẫn dùng được các năng lực cũ: tìm hội nghị, follow/unfollow, calendar, blacklist, gửi email admin, điều hướng, mở map, trả lời hướng dẫn website.
- Text chatbot hiện tại vẫn hoạt động bình thường qua Socket.IO/REST flow cũ.
- Live chatbot có session, transport, prompt, agent orchestration và logging riêng; chỉ tái sử dụng tool/handler nghiệp vụ khi phù hợp.
- Hệ thống có log/thought/status rõ ràng để dev mới debug được.

## 1.1 Nguyên tắc quan trọng: chạy song song, không thay thế

Live multi-agent là một **runtime mới** bên cạnh runtime text chatbot.

| Phần | Text chatbot hiện tại | Live chatbot mới |
| --- | --- | --- |
| Input chính | Text message, file upload, page context | Audio realtime, screen frame, transcript, optional text fallback |
| Output chính | Text streaming/result, frontend action | Audio PCM, transcript, frontend action, live status |
| Transport | Socket.IO/API hiện có | WebSocket live endpoint riêng |
| Agent orchestration | HostAgent + sub-agent flow hiện tại | LiveHostAgent + live sub-agent flow riêng |
| Tool nghiệp vụ | Handler hiện có | Tái sử dụng qua adapter, không sửa phá handler cũ |
| History | Text conversation history hiện tại | Live transcript/action summary, có thể sync vào conversation sau |

Các việc **không làm** trong plan này:

- Không xóa hoặc thay route/socket của text chatbot.
- Không đổi contract hiện tại của regular chat nếu không thật sự cần.
- Không ép text chatbot dùng ADK.
- Không chuyển toàn bộ tool handler sang live-only.
- Không để live refactor làm regression các flow text như search, follow, calendar, blacklist, email admin.

Các việc **được phép làm**:

- Thêm module live mới trong Chatbot Server.
- Thêm hook/UI live mới hoặc refactor phần live chat hiện có trong FE.
- Tạo adapter để live gọi lại handler cũ.
- Chia sẻ type/action chung nếu tương thích ngược.

## 2. Hiện trạng cần kế thừa

### 2.1 Chatbot text hiện tại

Repo `Easyconf-Chatbot-Server` đang có mô hình multi-agent tự xây:

```text
User
  -> HostAgent
      -> ConferenceAgent
      -> AdminContactAgent
      -> NavigationAgent
      -> WebsiteInfoAgent
      -> HostAgent tổng hợp phản hồi cuối
```

Các agent chính:

| Agent | Vai trò | Tool chính |
| --- | --- | --- |
| `HostAgent` | Điều phối, lập kế hoạch, định tuyến task | `routeToAgent`, `saveResultSet`, `finishWorkflow` hiện được xử lý trong host flow |
| `ConferenceAgent` | Xử lý dữ liệu hội nghị | `retrieveKnowledge`, `manageFollow`, `manageCalendar`, `manageBlacklist`, recommendation, feedback |
| `AdminContactAgent` | Gửi email/liên hệ admin | `sendEmailToAdmin` |
| `NavigationAgent` | Điều hướng FE, mở map | `navigation`, `openGoogleMap` |
| `WebsiteInfoAgent` | Hướng dẫn dùng website | `getWebsiteInfo` |

Các type quan trọng đã có trong `src/chatbot/shared/types.ts`:

| Type | Ý nghĩa |
| --- | --- |
| `AgentId` | ID agent như `HostAgent`, `ConferenceAgent`, `NavigationAgent` |
| `StatusUpdate` | Gửi trạng thái xử lý realtime về FE |
| `ThoughtStep` | Ghi lại bước suy nghĩ/thao tác của agent |
| `AgentCardRequest` | Request nội bộ từ HostAgent sang sub-agent |
| `AgentCardResponse` | Response nội bộ từ sub-agent về HostAgent |
| `FrontendAction` | Action để FE thực thi như navigate, open map, display list |

### 2.2 Live chat hiện tại ở FE

Repo `Easyconf-FE-Client` đã có thư mục `src/app/[locale]/chatbot/livechat` với live chat **gần hoàn chỉnh** (~40 files):

| File/nhóm file | Hiện trạng |
| --- | --- |
| `hooks/useLiveApi.ts` | FE đang tạo `GoogleGenAI` và gọi `genAI.live.connect()` trực tiếp từ browser |
| `hooks/useConnection.ts` | Quản lý mic permission, kết nối/ngắt kết nối, auto-reconnect |
| `hooks/useAudioRecorder.ts` | Thu âm PCM 16kHz, gửi vào SDK `sendRealtimeInput` |
| `hooks/useModelAudioResponse.ts` | Xử lý audio output từ model |
| `hooks/useInteractionHandlers.ts` | Gửi text message, trigger voice start |
| `hooks/useVolumeControl.ts` | Điều khiển volume |
| `lib/audio-recorder.ts` | Thu âm bằng `AudioWorklet`, PCM 16kHz mono, có thể chuyển sang gửi binary frame |
| `lib/audio-streamer.ts` | Phát PCM 24kHz output với queue/buffer management |
| ~~`services/tool.handlers.ts`~~ | ~~FE tự xử lý tool handlers~~ → Đã bỏ, server execute tool trực tiếp |
| `logger/*` | Hệ thống log message/tool/audio/transcription đầy đủ |
| `LiveChat.tsx` + `layout/*` | UI live chat hoàn chỉnh: mic button, connection status, transcript, logger |
| `contexts/*` | React context cho live API và settings |

Điểm cần thay đổi quan trọng trong **nhánh live chat**, không áp dụng cho text chatbot:

- Không để FE giữ Gemini API key.
- Không để FE tự gọi `genAI.live.connect()` trong production.
- **Không để FE xử lý tool nghiệp vụ** — server execute toàn bộ tool bằng `liveToolAdapter` (tái sử dụng handler text chatbot hiện có).
- FE nên trở thành client media/UI: thu âm, chụp màn hình, phát audio, hiển thị transcript/status/frontend-action.
- Chatbot Server nên có thêm Live Gateway riêng: mở Gemini Live session, điều phối live multi-agent, gọi tool backend.

### 2.3 Tổng quan hiện trạng thực tế so với plan

| Thành phần | Trạng thái | Ghi chú |
|-----------|-----------|---------|
| Server Live Gateway (`src/live/`) | ✅ **Đã có Phase 1** | WebSocket endpoint, session manager, Gemini bridge (parse `LiveServerMessage` + dispatch), auth (JWT + `ConfigService`), `LiveConfig`, idle timeout 10 phút, binary BE→FE audio, frame size guard + warning |
| FE Live Chat (voice, tool, UI) | 🔄 **Đang refactor (Phase 2)** | Chuyển từ direct SDK sang server bridge + thêm AEC (WebRTC loopback) |
| Screen Sharing | 🔄 **Trong Phase 2** | Server đã hỗ trợ video frame. FE sẽ implement `useScreenShare.ts` + `screenFrameCapture.ts` |
| Acoustic Echo Cancellation | 🔄 **Trong Phase 2** | WebRTC loopback technique. Gộp 2 AudioContext → 1 shared context. Route AI audio qua RTCPeerConnection local để browser AEC3 cancel |
| Tool Adapter (`src/live/tools/liveToolAdapter.ts`) | ✅ **Đã merge vào Phase 2** | Server execute tool trực tiếp, không forward FE |
| Multi-Agent Orchestration | ❌ **Chưa có** | Greenfield. Phase 4 của plan |
| ADK Integration | ❌ **Chưa có** | Greenfield. Phase 4 mức B |
| Database / History | ⏸️ **Tạm hoãn** | Phase 6 (deferred) |
| Tracking / Recommendation | 🗑️ **Đã loại bỏ** | Không nằm trong plan |

### 2.4 Live bridge trong `REPORT.md`

`REPORT.md` đã mô tả luồng live cơ bản:

```text
Browser
  -> mic PCM 16kHz qua WebSocket
  -> screen JPEG frame qua WebSocket
  -> Chatbot Server
  -> Gemini Live API
  -> audio/transcript/tool/status về Browser
```

Các format nên giữ:

| Dữ liệu | Format |
| --- | --- |
| Audio input | `audio/pcm;rate=16000`, 16-bit signed PCM, mono |
| Audio output | `audio/pcm;rate=24000`, 16-bit signed PCM, mono |
| Screen frame | `image/jpeg`, khuyến nghị 1 FPS hoặc thấp hơn khi không cần realtime cao |
| Transport FE -> Server | WebSocket, **binary frame** với 1-byte header (`0x01`=audio, `0x02`=video). Text frame dùng JSON cho control message |
| Transport Server -> FE | Audio: **binary frame `0x01`** (raw PCM 24kHz, không base64). Transcript/tool/status: JSON text frame |
| Transport Server -> Gemini | `session.sendRealtimeInput({ audio/video/text })` |

## 3. Kiến trúc đề xuất

### 3.1 Tổng quan

```text
Easyconf-FE-Client
  Microphone + Screen Share + UI actions
        |
        | WebSocket /api/live-agent
        v
Easyconf-Chatbot-Server
  Live Gateway
  Live Session Manager
  Live Host Agent / Agent Router
  Tool Adapter Layer
        |
        | Function calls / HTTP / internal services
        v
Easyconf-BE + RAG + Tracking System
        |
        v
Gemini Live API via @google/genai
```

Vai trò từng lớp:

| Lớp | Trách nhiệm |
| --- | --- |
| FE Live Client | Xin quyền mic/screen, encode audio/video, gửi stream, phát audio, hiển thị transcript/status/action |
| Live Gateway | Quản lý WebSocket live client, auth, rate limit, decode/encode message, không thay Socket.IO text chat |
| Live Session Manager | Mở/đóng Gemini Live session theo user/session, giữ state, cleanup khi disconnect |
| Live Agent Orchestrator | HostAgent live, định tuyến sub-agent, tổng hợp kết quả nói lại cho user |
| Tool Adapter Layer | Bọc handler cũ thành tool/function declaration dùng được trong live |
| Existing Backend/RAG | Xử lý dữ liệu hội nghị, user actions, recommendation, tracking |

### 3.2 Lựa chọn công nghệ

| Nhu cầu | Công nghệ đề xuất | Lý do |
| --- | --- | --- |
| Realtime audio/video với Gemini | `@google/genai` `ai.live` | SDK chính thức hỗ trợ Live API, `sendRealtimeInput`, audio output, function calling |
| Multi-agent code-first | `@google/adk` | Dễ định nghĩa agent/tool/workflow bằng TypeScript, dễ test bằng `adk run`/`adk web` |
| WebSocket FE <-> Server | `ws` hoặc Socket.IO hiện có | Live media stream cần low latency, server hiện đã dùng Socket.IO và có `ws` dependency |
| Tool validation | `zod` | ADK hỗ trợ Zod schema, repo đã dùng Zod |
| Auth/token | Cơ chế hiện tại của FE/BE | Tool follow/calendar/email cần user token |
| Log/trace | `StatusUpdate`, `ThoughtStep`, existing logger | FE đã có logger cho live chat |

Khuyến nghị triển khai theo hướng **incremental và song song**:

- Giai đoạn đầu dùng `@google/genai` để mở Gemini Live session trên server và tái sử dụng function declarations/handlers hiện có thông qua adapter, không sửa trực tiếp flow text chatbot.
- Song song định nghĩa cấu trúc agent/tool theo style của ADK để dễ migrate.
- Khi live ổn định, bọc các specialist agent bằng `@google/adk` `LlmAgent`/`RoutedAgent` để thống nhất orchestration và test agent độc lập.

Lý do không nên thay toàn bộ text chatbot bằng ADK:

- Text chatbot hiện tại đã có nhiều guard, history compression, routing fallback, payload tiering, tool handler và log nghiệp vụ.
- Live có rủi ro riêng về media stream, latency, interruption, auth, cleanup.
- Nên tách bài toán “xây live runtime song song” khỏi bài toán “migration framework”. Trong phạm vi này không migration text chatbot.

## 4. Thiết kế agent cho chế độ live

### 4.1 Danh sách agent

| Agent live | Kế thừa từ agent cũ | Vai trò trong live |
| --- | --- | --- |
| `LiveHostAgent` | `HostAgent` | Nghe user, đọc screen context, quyết định trả lời trực tiếp hay gọi sub-agent |
| `LiveConferenceAgent` | `ConferenceAgent` | Tìm hội nghị, lọc, follow, calendar, blacklist, recommendation |
| `LiveNavigationAgent` | `NavigationAgent` | Điều hướng trang theo lời nói hoặc theo màn hình đang share |
| `LiveWebsiteInfoAgent` | `WebsiteInfoAgent` | Hướng dẫn sử dụng website khi user hỏi “nút này làm gì”, “tôi đang ở đâu” |
| `LiveAdminContactAgent` | `AdminContactAgent` | Soạn email/report admin, yêu cầu xác nhận bằng giọng nói hoặc UI confirm |

### 4.2 Luồng một lượt hội thoại live

```text
1. FE gửi audio liên tục và screen frame định kỳ.
2. Gemini Live nhận audio/video và tạo input transcription.
3. LiveHostAgent hiểu ý định từ lời nói + screen context.
4. Nếu cần dữ liệu/action, LiveHostAgent gọi tool `routeToAgent` hoặc gọi tool chuyên biệt.
5. Server thực thi tool bằng adapter tái sử dụng handler cũ.
6. Kết quả tool được trả lại Live session qua function response.
7. Gemini nói câu trả lời cuối bằng audio.
8. Server gửi transcript/status/action về FE để hiển thị và thực thi UI.
```

### 4.3 Khi nào dùng agent nào

| Tình huống user nói | Agent xử lý chính | Ghi chú |
| --- | --- | --- |
| “Tìm hội nghị AI rank A năm 2026” | `LiveConferenceAgent` | Dùng RAG/API giống text chatbot |
| “Follow hội nghị này” khi đang share detail page | `LiveHostAgent` + `LiveConferenceAgent` | Screen context được Gemini tự hiểu, `LiveConferenceAgent` xử lý follow |
| “Mở trang calendar của tôi” | `LiveNavigationAgent` | Trả `FrontendAction` navigate |
| “Nút này dùng làm gì?” | `LiveWebsiteInfoAgent` | Gemini tự đọc UI từ screen frame, `LiveWebsiteInfoAgent` so sánh với knowledge base nếu cần |
| “Gửi admin báo lỗi trang này” | `LiveAdminContactAgent` | Cần confirmation trước khi gửi |
| “Mở map địa điểm hội nghị đầu tiên” | `LiveConferenceAgent` + `LiveNavigationAgent` | Conference lấy location, Navigation mở map |

## 5. Thiết kế WebSocket contract

### 5.1 Endpoint đề xuất

```text
ws(s)://<chatbot-server>/api/live-agent
```

**Auth:** Không dùng query param hay header. Dùng auth message (method #4) để tránh token bị lộ trong lịch sử web, server logs:

1. Client mở WebSocket — không kèm auth token.
2. Server gửi `{ type: "session.auth_required" }`.
3. Client gửi `{ type: "auth", payload: { token: "<JWT>", locale?: "vi" } }`.
4. Server verify JWT, nếu OK → gửi `session.ready`, nếu sai → đóng kết nối.
5. Auth timeout 30 giây — client không gửi auth kịp sẽ bị đóng.

| Trường | Mục đích |
| --- | --- |
| `mode=voice-screen` | Phân biệt live mode với text mode |

### 5.2 Message FE gửi lên server

**Binary frame** (dùng cho luồng realtime liên tục — audio, video):

| Byte 0 | Loại | Phần còn lại | Tần suất |
|--------|------|-------------|----------|
| `0x01` | Audio PCM | PCM 16kHz 16-bit signed mono, raw binary | Mỗi ~20-100ms |
| `0x02` | Video JPEG | JPEG bytes, đã resize/compress | 1 FPS hoặc thấp hơn |

Không cần JSON, không base64. Server đọc byte 0, switch, xử lý phần còn lại.

**Thứ tự gửi:** Client phải gửi `auth` message trước. Binary frame (audio/video) chỉ được gửi sau khi nhận được `session.ready` từ server. Mọi binary frame trước khi auth sẽ bị server bỏ qua.

**Text frame** (dùng cho control message, vài lần trong cả session):

| Type | Payload | Ghi chú |
| --- | --- | --- |
| `auth` | `{ token, locale? }` | Xác thực sau connect (method #4) |
| `session.start` | `{ locale, model, voice, responseModalities }` | Khởi tạo Live session |
| `session.stop` | `{ reason? }` | Kết thúc chủ động |
| `text` | `{ text }` | Fallback text hoặc debug |

Không có `tool.confirmation` — tool execution hoàn toàn server-side.

### 5.3 Message server gửi về FE

| Type | Payload | Ghi chú |
| --- | --- | --- |
| `session.auth_required` | `{}` | Server yêu cầu client gửi auth message |
| `session.ready` | `{ userId, locale, screenMaxBytes }` | Server đã mở Gemini Live session, kèm thông tin xác thực |
| `audio` | **Binary frame** `0x01` + raw PCM 24kHz | Dùng header `0x01` như FE→BE, FE đọc `ArrayBuffer` và đưa vào `AudioStreamer.addPCM16()` |
| `transcript` | `{ source: "user" \| "model", text, final }` | Hiển thị phụ đề/log |
| `status` | `StatusUpdate` | Cho UI biết agent đang làm gì |
| `frontend-action` | `FrontendAction[]` | Navigate, open map, display list, confirm dialog — do server gửi sau khi execute tool |
| `tool-call` | `FunctionCall[]` | (Debug) tool call từ Gemini |
| `tool-result` | `{ name, result }` | (Debug) kết quả tool |
| `error` | `{ code, message, recoverable }` | Lỗi có thể retry hoặc phải đóng session |
| `session.closed` | `{ reason }` | Cleanup FE |

## 6. Kế hoạch triển khai theo phase

### Phase 0: Chốt phạm vi MVP

Mục tiêu: xác định MVP đủ nhỏ để làm được và test được.

MVP nên gồm:

- Voice hỏi đáp cơ bản bằng Gemini Live.
- Screen sharing gửi JPEG frame lên server.
- Server giữ API key và bridge Gemini Live.
- Dùng được `retrieveKnowledge`, `navigation`, `getWebsiteInfo`.
- Chưa bắt buộc follow/calendar/blacklist/email trong MVP nếu confirmation/action flow chưa ổn.

Definition of Done:

- User nói “tìm hội nghị AI năm 2026” và nhận audio trả lời.
- User share màn hình trang conference list, nói “mở chi tiết hội nghị đầu tiên”, FE nhận action navigate.
- API key không xuất hiện trong bundle FE.

### Phase 1: Server Live Gateway

Thêm module mới trong `Easyconf-Chatbot-Server`:

```text
src/live/
  live.routes.ts
  liveSessionManager.ts
  liveGeminiBridge.ts
  liveMessage.types.ts
  liveAuth.ts
```

Nhiệm vụ:

- Tạo WebSocket endpoint `/api/live-agent`.
- Xác thực bằng auth message (method #4): gửi `session.auth_required`, nhận `{ type: "auth", payload: { token } }`, verify JWT → `{ userId, locale }`.
- Khởi tạo `GoogleGenAI` từ server env, không nhận API key từ FE.
- Mở `ai.live.connect({ model, config, callbacks })`.
- Forward audio/video/text từ FE vào `session.sendRealtimeInput()`.
- Parse Gemini `LiveServerMessage`: extract audio data (base64 → binary) + transcript + tool call, dispatch đúng callback.
- Forward audio về FE dùng binary frame `0x01` (raw PCM, không base64), transcript/tool/status dùng JSON text frame.
- Cleanup session khi FE disconnect, user stop, lỗi Gemini, timeout idle.

Config: dùng `LiveConfig` class (`src/config/live.config.ts`) đọc từ schema, wire vào `ConfigService`:

| Biến env | Mục đích | Default gợi ý |
| --- | --- | --- |
| `LIVE_AGENT_ENABLED` | Bật/tắt live endpoint | `false` |
| `LIVE_AGENT_MODEL` | Model live | `gemini-live` |
| `LIVE_AGENT_MAX_SESSION_MS` | Giới hạn thời lượng session | `900000` |
| `LIVE_AGENT_IDLE_TIMEOUT_MS` | Đóng nếu không có input | `600000` (10 phút) |
| `LIVE_AGENT_AUDIO_BINARY` | Cho phép binary audio | `true` |
| `LIVE_AGENT_SCREEN_FPS` | FPS server chấp nhận | `1` |
| `LIVE_AGENT_SCREEN_MAX_BYTES` | Giới hạn frame JPEG | `250000` |

Lưu ý kỹ thuật từ `@google/genai`:

- Dùng `GoogleGenAI` ở server.
- Dùng `ai.live` để mở live session.
- Gửi audio bằng `session.sendRealtimeInput({ audio: { data, mimeType: "audio/pcm;rate=16000" } })`.
- Gửi screen frame bằng `session.sendRealtimeInput({ video: { data, mimeType: "image/jpeg" } })`.
- Với tool call, server phải execute tool rồi trả `session.sendToolResponse()`.

### Phase 2: FE chuyển từ direct Gemini sang server bridge

Refactor `Easyconf-FE-Client/src/app/[locale]/chatbot/livechat`.

#### Chiến lược

Giữ nguyên interface `UseLiveAPIResults` để mọi consumer không cần sửa. Swap implementation bên trong.

Ba mục tiêu chính:
1. **Thay direct Gemini SDK bằng WebSocket bridge** — FE không còn giữ API key, không gọi Gemini trực tiếp.
2. **Thêm screen sharing** — `getDisplayMedia()` + gửi JPEG frame 1 FPS qua binary frame `0x02`.
3. **Thêm Acoustic Echo Cancellation (AEC)** — chống feedback loop khi micro thu lại giọng AI từ loa.

Quyết định thiết kế:
- **Tool execution server-side ngay từ Phase 2** (bỏ hẳn FE tool handler). Server execute tool bằng `liveToolAdapter` (tái sử dụng handler text chatbot), gửi `frontend-action` về FE, gọi `sendToolResponse()` lên Gemini. FE chỉ nhận kết quả, không execute hay confirm tool.
- **AEC** dùng WebRTC loopback technique: route audio output AI qua một cặp `RTCPeerConnection` local, khiến browser AEC3 (cùng engine Google Meet) nhận diện giọng AI như "remote participant audio" và tự động cancel khỏi mic input.

#### Thứ tự implement

| Bước | File mới/sửa | Hành động | Ý tưởng |
|------|-------------|-----------|---------|
| 1 | `liveProtocol.ts` | NEW | Binary frame helpers (header `0x01` audio, `0x02` video) + JSON message type cho text frame giữa FE và Server. Thêm `LiveAuthPayload`, `"auth"` message type, `"session.auth_required"` server event |
| 2 | `useLiveAgentSocket.ts` | NEW | WebSocket lifecycle: connect/disconnect, send binary + JSON, nhận binary frame audio/video + JSON event từ server. Auto-reconnect. Auth bằng auth message (method #4) — gửi `{ type: "auth", payload: { token, locale } }` ngay sau `onopen`, không đặt token vào URL query param |
| 3 | `useLiveApi.ts` | REFACTOR | Thay `GoogleGenAI` SDK bằng socket bridge. Giữ nguyên `on/off` event emitter. Map server JSON events (`session.auth_required`, `session.ready`, `transcript`, `frontend-action`, `status`) → emitter events. Bỏ `session.start` (không còn cần thiết vì model/voice/server-configured) |
| 4 | `useAudioRecorder.ts` | REFACTOR | Gửi binary frame `0x01` + PCM 16kHz qua socket thay vì `SDKBlob` → `sendRealtimeInput()` |
| 5 | `LiveChatAPIConfig.tsx` | REFACTOR | Bỏ SDK config (`setConfig`, function declarations). Tool call server-side, FE chỉ nhận `frontend-action` event |
| 6 | `LiveAPIContext.tsx` | REFACTOR | Bỏ prop `apiKey`, thay bằng `serverUrl` + `token` |
| 7 | `useScreenShare.ts` | NEW | `getDisplayMedia()` → loop RAF 1 FPS → gửi binary frame `0x02` + JPEG |
| 8 | `screenFrameCapture.ts` | NEW | Canvas resize frame về 768x768, nén JPEG quality 0.7 |
| 9 | `useAudioAEC.ts` | NEW | **WebRTC loopback AEC**: route AI audio output qua RTCPeerConnection local, phát remote stream ra loa thật → browser AEC3 tự cancel khỏi mic. Gộp 2 AudioContext (recording 16kHz + playback 24kHz) thành 1 shared context. Không cần AudioWorklet — AEC do browser engine xử lý |
| 10 | `audio-recorder.ts` | UPDATE | Thêm constraint `echoCancellation: true, noiseSuppression: true, autoGainControl: true` vào getUserMedia. AEC tự động lấy reference signal từ remote audio (pc2) đang phát ra loa để cancel, không cần merge stream thủ công |
| 11 | `audio-streamer.ts` | UPDATE | Playback qua shared AudioContext, thêm `MediaStreamDestination` để feed reference signal vào AEC |
| 12 | `LiveChat.tsx` | UPDATE | Wire screen share button, AEC status indicator, pass token từ auth store |
| 13 | Server: `liveSessionManager.ts` + `liveGeminiBridge.ts` | UPDATE | Server execute tool khi nhận `toolCall` từ Gemini thay vì forward FE; thêm `liveToolAdapter` |

#### AEC — WebRTC Loopback (chi tiết)

Vấn đề: Browser AEC (`getUserMedia` echoCancellation) chỉ cancel audio từ WebRTC peer connection, không cancel audio phát từ Web Audio API.

Giải pháp — WebRTC loopback:
- Tạo 1 cặp `RTCPeerConnection` local (pc1 ↔ pc2) trên cùng trang.
- AI audio output được route qua `MediaStreamDestination` → addTrack vào pc1.
- pc2 nhận stream → coi như "remote participant audio".
- Gắn remote stream từ pc2 vào `<audio autoplay>` để phát ra loa thật — AEC cần acoustic reference signal là audio đang phát ra loa để so sánh với mic.
- `getUserMedia({ echoCancellation: true })` thu mic — browser AEC3 tự động lấy reference signal từ remote audio của pc2 và trừ nó khỏi tín hiệu mic, đầu ra chỉ còn giọng user.

Yêu cầu hạ tầng:
- Gộp 2 `AudioContext` riêng (hiện tại recording 16kHz + playback 24kHz) thành 1 shared context.
- `AudioStreamer` playback qua shared context, đồng thời feed AI audio vào `MediaStreamDestination` (không cần AudioWorklet cho việc routing reference — AEC hoàn toàn do browser engine xử lý).
- Không cần thêm thư viện — WebRTC API có sẵn trong browser.

#### Tổng quan files thay đổi

| File | Trạng thái | Ý tưởng thay đổi |
|------|-----------|-----------------|
| `useLiveApi.ts` | REFACTOR | Socket bridge thay SDK, giữ event emitter |
| `useAudioRecorder.ts` | REFACTOR | Binary frame thay SDK blob |
| `LiveChatAPIConfig.tsx` | REFACTOR | Bỏ SDK config, giữ toolcall routing |
| `LiveAPIContext.tsx` | REFACTOR | `serverUrl`+`token` thay `apiKey` |
| ~~`tool.handlers.ts`~~ | ~~Giữ nguyên~~ → **Xóa bỏ** | Server execute tool bằng `liveToolAdapter` |
| `audio-recorder.ts` | UPDATE | AEC constraints + loopback stream |
| `audio-streamer.ts` | UPDATE | Shared AudioContext + AEC reference |
| `LiveChat.tsx` | UPDATE | Screen share + AEC status + token |

| File mới | Ý tưởng |
|----------|---------|
| `liveProtocol.ts` | Binary encode/decode, JSON message types |
| `useLiveAgentSocket.ts` | WebSocket lifecycle |
| `useScreenShare.ts` | Display media capture loop |
| `screenFrameCapture.ts` | Frame resize/compress |
| `useAudioAEC.ts` | WebRTC loopback AEC setup |
| ~~`aecWorklet.ts`~~ | **Không cần** — AEC do browser engine xử lý, không cần AudioWorklet để route reference |

#### Protocol flow

```text
Auth flow (method #4):
  Client connect → Server gửi session.auth_required
                  → Client gửi { type: "auth", payload: { token, locale? } }
                  → Server verify JWT → tạo Gemini bridge → session.ready
                  → Bắt đầu trao đổi audio/video/text
                  → Auth timeout 30s nếu client không gửi auth

Tool execution (server-side):
  Gemini → toolCall → Server → liveToolAdapter.executeTool()
    → handler.execute() → { modelResponseContent, frontendActions }
    → Server → session.sendToolResponse({ functionResponses }) → Gemini
    → Server → JSON "frontend-action" → FE (nếu có UI action)

AEC flow:
  AI Audio → AudioStreamer → MediaStreamDestination
    → pc1.addTrack() → pc2.ontrack → remoteStream
    → &lt;audio autoplay srcObject={remoteStream}&gt; → LOA THẬT
    → Browser AEC3 so reference (loa) vs capture (mic)
    → getUserMedia({echoCancellation:true}) → mic không còn echo AI
```

### Phase 3: Tool Adapter Layer — Nâng cấp

Mục tiêu: Hoàn thiện tool adapter (đã triển khai sơ bộ ở Phase 2), thêm schemas validation, agent context injection, logging chuẩn.

Thêm module:

```text
src/live/tools/
  liveToolSchemas.ts       ← MỚI: Zod schema cho từng tool
```

Cải tiến `liveToolAdapter.ts` (đã có từ Phase 2):

- `FrontendAction` đã chuẩn (dùng chung type với text chatbot). FE live nhận `"frontend-action"` event và handle trực tiếp, không cần mapper riêng.

- Inject `userToken` từ auth context (hiện tại đang `null`).
- Inject `screenContext` khi screen sharing hoạt động.
- Gửi `ThoughtStep` về FE song song với execution.
- Log tool execution vào `ChatbotFlowLoggerService`.
- Bắt lỗi và trả lỗi có cấu trúc, tránh model hallucinate.

Tool đã hoạt động ngay từ Phase 2 (qua `functionRegistry` tái sử dụng):

| Tool live | Handler cũ | Agent sở hữu |
| --- | --- | --- |
| `retrieveKnowledge` | `RetrieveKnowledgeHandler` | `LiveConferenceAgent` |
| `navigation` | `NavigationHandler` | `LiveNavigationAgent` |
| `openGoogleMap` | `OpenGoogleMapHandler` | `LiveNavigationAgent` |
| `getWebsiteInfo` | `GetWebsiteInfoHandler` | `LiveWebsiteInfoAgent` |
| `manageFollow` | `ManageFollowHandler` | `LiveConferenceAgent` |
| `manageCalendar` | `ManageCalendarHandler` | `LiveConferenceAgent` |
| `manageBlacklist` | `ManageBlacklistHandler` | `LiveConferenceAgent` |
| `sendEmailToAdmin` | `SendEmailToAdminHandler` | `LiveAdminContactAgent` |
| (và tất cả handler còn lại) | ... | ... |

Điều kiện để bật tool nhạy cảm:

| Tool | Điều kiện |
| --- | --- |
| `manageFollow` / `manageCalendar` / `manageBlacklist` | Auth + guard screen context rõ ràng |
| `sendEmailToAdmin` | Confirmation qua voice hoặc UI |

### Phase 4: Live multi-agent orchestration

Có hai mức triển khai.

Mức A, khuyến nghị cho MVP live:

- Dùng current custom HostAgent pattern làm tham chiếu, nhưng tạo `LiveHostAgent`/live orchestration riêng.
- `LiveHostAgent` là system instruction + function declarations trong Gemini Live session, không dùng chung runtime instance với text `HostAgent`.
- Tool `routeToAgent` gọi qua adapter/service tương đương để chạy specialist task, tránh sửa trực tiếp `hostAgent.nonStreaming.handler.ts` hoặc `hostAgent.streaming.handler.ts` nếu không cần.
- Giữ `AgentCardRequest`/`AgentCardResponse` để tái sử dụng protocol nội bộ.

Mức B, sau khi MVP ổn:

- Cài `@google/adk` trong `Easyconf-Chatbot-Server`.
- Định nghĩa `LlmAgent` cho từng specialist.
- Dùng `RoutedAgent` hoặc router function để chọn agent theo intent/context.
- Dùng `SequentialAgent` cho luồng nhiều bước bắt buộc, ví dụ lấy location rồi mở map.
- Dùng `ParallelAgent` cho tác vụ độc lập, ví dụ vừa tóm tắt screen vừa tìm knowledge base nếu không phụ thuộc nhau.

Ví dụ mapping ADK:

```text
LiveRoutedAgent
  router(context)
    -> hỏi hội nghị: LiveConferenceAgent
    -> yêu cầu mở trang/map: LiveNavigationAgent
    -> hỏi cách dùng web: LiveWebsiteInfoAgent
    -> liên hệ admin: LiveAdminContactAgent
```

Lưu ý khi dùng ADK:

- `SequentialAgent` phù hợp khi thứ tự cố định, ví dụ `ConferenceAgent -> NavigationAgent`.
- `ParallelAgent` phù hợp khi task độc lập, nhưng phải cẩn thận shared state/race condition.
- `RoutedAgent` phù hợp chọn đúng một agent tại runtime và có fallback nếu agent lỗi trước khi emit event.
- Tool schema nên dùng Zod để type-safe và dễ validate.

### (Đã lược bỏ) Phase 5: Screen context và grounding

~Đã lược bỏ — Gemini Live xử lý screen context tự nhiên qua frame ảnh. Không cần `LiveScreenState` riêng.~

### ⏸️ Phase 6: Conversation history và memory (tạm hoãn)

Live session cần lưu cả text transcript và action summary, không lưu raw audio/video mặc định.

Lưu vào conversation history:

- Final user transcript.
- Final model transcript.
- Tool calls và tool results đã rút gọn.
- Frontend actions đã thực thi.
- Screen state summary, không lưu ảnh nếu chưa có chính sách lưu trữ.

Không nên lưu mặc định:

- Raw PCM audio.
- Raw screen JPEG frame.
- API key/token.
- Dữ liệu nhạy cảm xuất hiện trên màn hình.

Nếu cần debug media:

- Chỉ bật bằng env `LIVE_AGENT_DEBUG_MEDIA=true` ở môi trường dev.
- Có TTL ngắn và cảnh báo rõ.

### (Đã lược bỏ) Phase 7: Tracking và recommendation

~Đã lược bỏ — không cần thiết cho MVP. Tracking/recommendation đã có sẵn qua text chatbot.~

## 7. Thay đổi cụ thể theo repo

### 7.1 `Easyconf-Chatbot-Server`

Thêm dependency:

- Nâng `@google/genai` lên version mới tương thích Live API nếu version hiện tại chưa đủ.
- Thêm `@google/adk` ở phase ADK.

Thêm module server live riêng, đặt cạnh chatbot hiện tại nhưng không thay thế các module chatbot text:

```text
src/live/live.routes.ts
src/live/liveSessionManager.ts
src/live/liveGeminiBridge.ts
src/live/liveMessage.types.ts
src/live/liveAuth.ts
src/live/liveAgentOrchestrator.ts
src/live/tools/liveToolRegistry.ts
src/live/tools/liveToolAdapter.ts
```

Tích hợp vào bootstrap:

- Mount WebSocket cùng HTTP server hiện có: `const liveWss = initLiveGateway(httpServer)`.
- `liveWss` được trả về trong `LoadersResult` cùng với `app`, `httpServer`, `io`.
- Không thay đổi route `/api/v1/chatbot`, Socket.IO regular chat và các handler text hiện tại.
- Validate origin/CORS.
- Log session lifecycle.
- Shutdown graceful: `closeAllLiveSessions("server_shutdown")` được gọi từ `gracefulShutdown` trong `server.ts` (Bước 4), sau khi đóng Playwright và trước khi flush logs.

Tái sử dụng code hiện có qua adapter, giữ tương thích ngược:

- `languageConfig.ts` để lấy system instructions/function declarations theo agent/language.
- `functionRegistry.ts` hoặc handler classes để execute tools.
- `AgentCardRequest`/`AgentCardResponse` cho delegation.
- `StatusUpdate`/`ThoughtStep` để stream trạng thái về FE.

### 7.2 `Easyconf-FE-Client`

Refactor/thêm mới trong thư mục live chat, không ảnh hưởng regular text chat:

```text
src/app/[locale]/chatbot/livechat/hooks/useLiveAgentSocket.ts
src/app/[locale]/chatbot/livechat/hooks/useScreenShare.ts
src/app/[locale]/chatbot/livechat/hooks/useAudioAEC.ts
src/app/[locale]/chatbot/livechat/lib/liveProtocol.ts
src/app/[locale]/chatbot/livechat/lib/screenFrameCapture.ts
src/app/[locale]/chatbot/livechat/lib/aecWorklet.ts
```

Điều chỉnh trong live chat:

- `useLiveApi.ts` không còn gọi `GoogleGenAI` trực tiếp trong production.
- `AudioRecorder` gửi audio tới socket bridge.
- `AudioStreamer` nhận audio từ server và phát.
- ~~`tool.handlers.ts`~~ **Xóa bỏ** — server execute toàn bộ tool. FE chỉ nhận `frontend-action` event.
- Live UI thêm nút share screen, trạng thái “đang chia sẻ”, “đang nghe”, “agent đang xử lý”.
- Hiển thị transcript user/model để người dùng kiểm tra chatbot nghe đúng chưa.
- Giữ nguyên `/chatbot/regularchat`, các store/hook regular chat và Socket.IO text flow hiện tại.

### 7.3 `Easyconf-BE`

Không cần sửa lớn trong MVP.

Cần kiểm tra API hiện có cho:

- Conference search/detail.
- Follow/unfollow.
- Calendar add/remove/list.
- Blacklist add/remove/list.
- User profile/personalization.

Nếu live cần action mới, thêm API nhỏ và vẫn để Chatbot Server gọi qua backend, không để FE gọi trực tiếp nếu action do agent quyết định.

### 7.4 `Easyconf-FE-Admin`

Không nằm trong MVP.

Có thể thêm sau:

- Dashboard xem live chatbot logs.
- Thống kê lỗi live session.
- Bật/tắt feature flag live agent.

### 7.5 `Easyconf-Tracking-System`

~Tracking/recommendation tích hợp đã được lược bỏ khỏi plan. Text chatbot tracking hiện tại không bị ảnh hưởng.~

## 8. Prompt và instruction cho LiveHostAgent

Live prompt cần khác text prompt vì user đang nói và có screen context.

Nguyên tắc:

- Trả lời ngắn hơn text chat, vì output là giọng nói.
- Nếu đang xử lý lâu, nói một câu status ngắn hoặc gửi `agent-status` cho FE.
- Không đọc danh sách quá dài bằng giọng nói. Với list dài, gửi `displayList` action và nói tóm tắt.
- Khi không chắc screen đang hiển thị gì, hỏi lại thay vì đoán.
- Với action ghi dữ liệu, phải xác nhận rõ item và action.

Ví dụ instruction bổ sung:

```text
Bạn là LiveHostAgent của Easyconf. Người dùng tương tác bằng giọng nói và có thể chia sẻ màn hình.
Luôn dùng tiếng Việt nếu locale là vi.
Hãy trả lời ngắn gọn, tự nhiên như đang nói chuyện.
Nếu cần dữ liệu hội nghị, hãy gọi ConferenceAgent/tool phù hợp.
Nếu cần điều hướng, hãy gọi NavigationAgent/tool phù hợp.
Nếu user nói "cái này", "hội nghị này", "nút này", hãy dùng screen context. Nếu screen context không đủ chắc chắn, hỏi lại.
Không tự ý follow, thêm calendar, blacklist hoặc gửi email nếu chưa xác nhận rõ với user.
```

## 9. Test plan

### 9.1 Unit test

| Module | Test |
| --- | --- |
| `liveProtocol` | parse/validate message đúng, reject message quá lớn |
| `liveToolAdapter` | map function call sang handler đúng, trả function response đúng |
| `liveScreenState` | update frame/screen state, expire context cũ |
| `liveSessionManager` | cleanup khi disconnect/timeout/error |
| `screenFrameCapture` FE | resize/compress đúng MIME và size |

### 9.2 Integration test

| Kịch bản | Kỳ vọng |
| --- | --- |
| Connect live session | FE nhận `session.ready` |
| Gửi audio test | Server forward `audio/pcm;rate=16000` sang Gemini |
| Gemini trả audio | FE phát audio 24kHz không bị rè/click nặng |
| Tool `retrieveKnowledge` | Agent trả danh sách hội nghị đúng nguồn |
| Tool `navigation` | FE nhận action navigate đúng path |
| Disconnect giữa chừng | Server đóng Gemini session và giải phóng resource |

### 9.3 Manual E2E script

1. Mở `/chatbot/livechat`.
2. Bấm Connect.
3. Bật micro và nói: “Tìm giúp tôi hội nghị về machine learning năm 2026”.
4. Kiểm tra transcript user đúng tương đối.
5. Kiểm tra agent gọi `retrieveKnowledge`.
6. Kiểm tra model trả audio và FE hiển thị text phụ đề.
7. Bật share screen ở trang `/conferences`.
8. Nói: “Mở chi tiết hội nghị đầu tiên trên màn hình”.
9. Kiểm tra agent không đoán nếu không nhận diện được item.
10. Nếu nhận diện được, FE nhận `frontend-action` navigate.
11. Nói: “Dừng phiên live”.
12. Kiểm tra server cleanup session.

### 9.4 Performance target

| Chỉ số | Mục tiêu MVP |
| --- | --- |
| Thời gian connect | < 3 giây trong mạng ổn định |
| Độ trễ audio user -> model bắt đầu phản hồi | 1-3 giây tùy Gemini Live |
| Audio chunk input | 50-100ms/chunk khuyến nghị |
| Screen frame | 1 FPS, JPEG < 250KB/frame |
| Session cleanup | < 5 giây sau disconnect |

## 10. Rủi ro và cách giảm thiểu

| Rủi ro | Tác động | Giảm thiểu |
| --- | --- | --- |
| FE lộ Gemini API key | Lộ credential | Chuyển Live API sang server bridge |
| Audio format sai | Model nghe sai/rè | Bắt buộc PCM 16-bit 16kHz mono, test bằng file mẫu |
| Screen frame quá nhiều | Tốn băng thông/cost | Giới hạn FPS, size, quality, backpressure |
| Agent hiểu nhầm “hội nghị này” | Action sai | Screen confidence guard + hỏi xác nhận |
| Tool ghi dữ liệu chạy nhầm | Ảnh hưởng user data | Confirmation trước follow/calendar/blacklist/email trong live |
| Session leak | Tốn quota/cost | Timeout, cleanup on close/error, max duration |
| Tool latency làm hội thoại đứt quãng | UX kém | Gửi status ngắn, stream audio khi có kết quả, cache context |
| ADK migration quá lớn | Delay MVP | Làm bridge + adapter trước, ADK hóa sau |

## 11. Thứ tự công việc đề xuất

#### Phase 1 ✅ Hoàn thành

1. Thêm server WebSocket `/api/live-agent` và `LiveSessionManager`.
2. Tạo bridge `@google/genai` Live API ở server với audio/text, tool call parsing, binary audio output.
3. `LiveConfig`, `liveAuth` (JWT), idle timeout 10 phút, frame size guard, graceful shutdown.

#### Phase 2 🔜 (Hiện tại)

4. Tạo `liveProtocol.ts` — binary frame helpers + JSON message types.
5. Tạo `useLiveAgentSocket.ts` — WebSocket lifecycle hook.
6. Refactor `useLiveApi.ts` — swap SDK → socket bridge, giữ event emitter.
7. Refactor `useAudioRecorder.ts` — binary frame `0x01` thay SDK blob.
8. Refactor `LiveChatAPIConfig.tsx` — bỏ SDK config, tool call qua bridge.
9. Refactor `LiveAPIContext.tsx` — `serverUrl`+`token` thay `apiKey`.
10. Tạo `useScreenShare.ts` + `screenFrameCapture.ts` — screen sharing.
11. **Tạo `useAudioAEC.ts` — WebRTC loopback AEC (không cần AudioWorklet, AEC do browser engine xử lý).**
12. **Update `audio-recorder.ts` + `audio-streamer.ts` — shared AudioContext, phát AI audio ra loa thật, AEC constraints.**
13. Update `LiveChat.tsx` — wire screen share + AEC status + token.
14. Server: thêm `src/live/tools/liveToolAdapter.ts` — server execute tool thay vì forward FE.

#### Phase 3

13. Cải tiến `liveToolAdapter` — inject `userToken`, `screenContext`, logging.
14. Thêm `liveToolSchemas` — Zod schema validation cho từng tool.
15. Thêm `liveFrontendActionMapper` — map `FrontendAction` thành event FE chuẩn.

#### Phase 4

16. Tạo `LiveHostAgent` instruction và function declarations tối thiểu.
17. Thêm guard cho action nhạy cảm và confirmation flow.
18. Bổ sung `manageFollow`, `manageCalendar`, `manageBlacklist`, `sendEmailToAdmin`.

#### Phase 5+ (future — chưa có kế hoạch cụ thể)

19. ADK migration (`@google/adk` — `LlmAgent`, `RoutedAgent`, workflow testable).

## 12. Tiêu chí hoàn thành MVP

MVP được xem là hoàn thành khi:

- FE live chat không cần nhập API key Gemini.
- Text chatbot regular vẫn chạy như trước, không regression các flow text hiện có.
- User có thể nói tiếng Việt và nhận câu trả lời bằng giọng nói.
- Server nhận audio 16kHz, gửi Gemini Live và trả audio 24kHz về FE.
- User có thể bật share screen, server nhận frame JPEG và dùng làm context.
- Live agent gọi được ít nhất 3 tool: `retrieveKnowledge`, `navigation`, `getWebsiteInfo`.
- Có status/transcript/tool log đủ để debug.
- Disconnect không để rò session Gemini.
- Các action ghi dữ liệu chưa đủ guard thì chưa bật tự động.

## 13. Kết luận

Hướng triển khai an toàn nhất là **không viết lại và không thay thế text chatbot**, mà xây một runtime Live Gateway riêng trên `Easyconf-Chatbot-Server`, chuyển media stream từ FE vào server, rồi tái sử dụng agent/tool hiện có thông qua adapter. `@google/genai` phụ trách Gemini Live realtime. `@google/adk` nên được dùng để chuẩn hóa dần phần live agent definitions, routing và workflow sau khi MVP live bridge đã ổn định.

Với cách này, hệ thống live mới giống hệ thống multi-agent hiện tại về mặt nghiệp vụ, nhưng chạy song song với text chatbot và dùng voice, audio transcript, screen context làm kênh tương tác riêng.
