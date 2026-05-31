# Báo cáo kiến trúc: Ghi âm, Share màn hình & Gửi dữ liệu lên Gemini Live

## Tổng quan luồng dữ liệu

```
[Browser]                              [Server Node.js]                    [Gemini Live API]
   │                                          │                                 │
   ├── Mic → getUserMedia(16kHz)              │                                 │
   ├── Audio PCM chunks ──WebSocket──► session.sendRealtimeInput({audio}) ──►   │
   │                                          │                                 │
   ├── Screen → getDisplayMedia(5fps)         │                                 │
   ├── Canvas 640×360 → JPEG ──WebSocket──► session.sendRealtimeInput({video})──►│
   │                                          │                                 │
   ├── Gemini audio trả về ──WebSocket──► scheduleAudioChunkPlay()              │
   │                                          │                                 │
```

## 1. Ghi âm Microphone

**File**: `src/hooks/useLiveAgent.ts:177-230` — hàm `activateAudioCapture()`

### Các bước:

1. **Lấy luồng âm thanh từ mic** qua Web API `navigator.mediaDevices.getUserMedia()` với cấu hình:
   - `echoCancellation: true`
   - `noiseSuppression: true`
   - `autoGainControl: true`
   - `channelCount: 1` (mono)

2. **Tạo AudioContext** ở sample rate **16000 Hz** — đây là rate chuẩn cho PCM đầu vào của Gemini.

3. **Kết nối đồ thị xử lý âm thanh**:
   ```
   MediaStreamSource → ScriptProcessorNode(buffer=2048) → audioCtx.destination
   ```

4. **Xử lý từng chunk âm thanh** trong callback `onaudioprocess`:
   - Lấy Float32 PCM từ `inputBuffer.getChannelData(0)`
   - Chuyển đổi sang **16-bit signed integer PCM** bằng `convertFloat32To16BitPCM()` (do Gemini yêu cầu)
   - Mã hóa base64 bằng `arrayBufferToBase64()`
   - Gửi qua WebSocket dưới dạng JSON: `{ type: "audio", data: "<base64>" }`

### Server nhận và chuyển tiếp:

**File**: `server.ts:404-408`
```typescript
session.sendRealtimeInput({
  audio: { data: parsed.data, mimeType: "audio/pcm;rate=16000" }
});
```

## 2. Share màn hình

**File**: `src/hooks/useLiveAgent.ts:253-314` — hàm `toggleScreenSharing()`

### Các bước:

1. **Gọi** `navigator.mediaDevices.getDisplayMedia()` với cấu hình video:
   - `frameRate: 5` — chỉ 5 khung hình/giây
   - Độ phân giải tối đa 1280×720

2. **Khởi tạo timer interval 1.5 giây** gọi `triggerScreenFrameCapture()`.

3. **Trong `triggerScreenFrameCapture()`** (dòng 123-174):
   - Gắn `MediaStream` vào một thẻ `<video>` ẩn
   - Tạo `<canvas>` kích thước **640×360** (giảm độ phân giải – vừa đủ cho Gemini xử lý)
   - Vẽ khung hình từ video lên canvas
   - Chuyển canvas thành **JPEG base64 với chất lượng 0.65** (`canvas.toDataURL("image/jpeg", 0.65)`)
   - Gửi qua WebSocket: `{ type: "video", data: "<base64>" }`

4. **Khi người dùng dừng sharing** (bấm nút trình duyệt hoặc nút UI): hủy interval và giải phóng stream.

### Server nhận và chuyển tiếp:

**File**: `server.ts:409-413`
```typescript
session.sendRealtimeInput({
  video: { data: parsed.data, mimeType: "image/jpeg" }
});
```

## 3. Nhận & Phát lại âm thanh từ Gemini

**File**: `src/hooks/useLiveAgent.ts:86-120` — hàm `scheduleAudioChunkPlay()`

1. Server nhận message `serverContent.modelTurn.parts[0].inlineData.data` từ Gemini
2. Server gửi xuống client: `{ type: "audio", data: "<base64>" }`
3. Client decode base64 → Int16Array → Float32Array
4. Tạo `AudioBuffer` tại sample rate **24000 Hz**
5. Lên lịch phát qua `AudioBufferSourceNode.start()` — xếp hàng đợi để phát liền mạch

## 4. WebSocket Bridge (kết nối)

**Client** → **Server** (cổng 8765, path `/api/live-agent`):
- Audio: `{ type: "audio", data: "<PCM_16bit_base64>" }`
- Video: `{ type: "video", data: "<JPEG_base64>" }`
- Text:  `{ type: "text", data: "..." }`
- File:  `{ type: "file", data: "<base64>", mimeType: "...", fileName: "..." }`

**Server** → **Client**:
- Audio: `{ type: "audio", data: "<base64>" }` — chunk giọng nói Gemini
- Transcript: `{ type: "transcript", source: "model"|"user", text: "..." }`
- Tool call: `{ type: "tool-call", name, args, status, result, id }`
- Status: `{ type: "agent-status", status, message }`

## 5. MCP Tools

Ngoài ghi âm và share màn hình, agent còn có thể gọi **MCP tools** qua Gemini Function Calling. Server (`src/server/mcp.ts`) quản lý:
- **Built-in tools**: weather, calculator, save note
- **MCP client tools**: đọc từ file cấu hình MCP, kết nối qua stdio, expose tool declarations cho Gemini

Khi Gemini gọi tool → server nhận `toolCall` → gọi `mcpManager.callMcpTool()` → trả kết quả → Gemini tiếp tục hội thoại.

## Tổng kết

| Thành phần | Công nghệ | Chi tiết |
|---|---|---|
| Ghi âm | Web Audio API (`getUserMedia`, `ScriptProcessorNode`) | 16kHz, PCM 16-bit, mono, chunk 2048 mẫu |
| Share màn hình | `getDisplayMedia` + Canvas 2D | 5fps, 640×360, JPEG 65%, gửi mỗi 1.5s |
| Giao tiếp | WebSocket (`ws` thư viện) | JSON messages, bridge với Gemini Live |
| Gemini SDK | `@google/genai` | `session.sendRealtimeInput()`, `LiveSendToolResponseParameters` |
| MCP | `@modelcontextprotocol/sdk` | Kết nối server stdio, function calling |
| Desktop | Electron | Global shortcut Ctrl+Shift+Space, system tray |
