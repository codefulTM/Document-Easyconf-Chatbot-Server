# Kế hoạch chuẩn hóa IPC: EMIT / EMIT_ACK (child <-> parent)

Mục tiêu

- Chuẩn hóa luồng "emit + ack" giữa tiến trình con (child) và tiến trình cha (parent) để: dễ duy trì, tránh duplication, hỗ trợ nhiều run/concurrency, và để parent chỉ ack khi hành động (ví dụ gửi tới socket) thực sự hoàn tất.

Tổng quan design

- Thêm 2 message chung trong `IPCCommonType` (nếu chưa có):
  - `EMIT` — payload: `{ type: EMIT, id?: string, emitId: string, event: string, payload: any }`
  - `EMIT_ACK` — payload: `{ type: EMIT_ACK, id?: string, emitId: string, success: boolean, error?: string }`
- Khóa cho pending ack: `${runId ?? 'global'}:${emitId}` để phân biệt các emit song song.

Các bước (step-by-step)

1. Cập nhật types

- Mở file `src/workers/ipc.types.ts` (hoặc file enum tương ứng) và đảm bảo có `EMIT` và `EMIT_ACK`.

2. Tạo helper child-side

- Tạo file `src/workers/workerEmit.ts` (child helper) chứa:
  - `emitToParentAwaitAck(event: string, payload: any, opts?: { id?: string, timeoutMs?: number }): Promise<any>` — trả về một `Promise`
    sẽ resolve với dữ liệu `payload` trả về bởi parent trong `EMIT_ACK`, hoặc reject khi parent trả lỗi/timeout.
  - internal `pendingEmitAcks: Map<string, {resolve: (v:any)=>void; reject:(e:any)=>void}>`.
  - `process.on('message', ...)` bắt `EMIT_ACK` và resolve/reject phù hợp.

Ví dụ rút gọn (workerEmit.ts):

```ts
import { v4 as uuidv4 } from "uuid";
import { IPCCommonType } from "./ipc.types";

type Pending = { resolve: (v: any) => void; reject: (e: any) => void };
const pendingEmitAcks = new Map<string, Pending>();

process.on("message", (msg: any) => {
  if (!msg || msg.type !== IPCCommonType.EMIT_ACK) return;
  const key = `${msg.id ?? "global"}:${msg.emitId}`;
  const p = pendingEmitAcks.get(key);
  if (!p) return;
  pendingEmitAcks.delete(key);
  if (msg.error) p.reject(new Error(msg.error));
  else p.resolve(msg.payload);
});

export function emitToParentAwaitAck(
  event: string,
  payload: any,
  opts?: { id?: string; timeoutMs?: number },
): Promise<any> {
  const emitId = uuidv4();
  const id = opts?.id ?? null;
  const key = `${id ?? "global"}:${emitId}`;
  const timeoutMs =
    opts?.timeoutMs ?? Number(process.env.CHILD_EMIT_ACK_TIMEOUT_MS) ?? 5000;

  if (!process.send) return Promise.reject(new Error("no-parent"));

  return new Promise<any>((resolve, reject) => {
    const t = setTimeout(() => {
      pendingEmitAcks.delete(key);
      reject(new Error("timeout"));
    }, timeoutMs);

    pendingEmitAcks.set(key, {
      resolve: (v: any) => {
        clearTimeout(t);
        resolve(v);
      },
      reject: (e: any) => {
        clearTimeout(t);
        reject(e);
      },
    });

    process.send({ type: IPCCommonType.EMIT, id, emitId, event, payload });
  });
}
```

3. Child: dùng helper

- Thay các chỗ child hiện đang `process.send({ type: ..., id, emitId, payload })` hoặc gửi `STREAM_CHUNK`/`STATUS`/... bằng `await emitToParentAwaitAck('chat_update', { ... }, { id: runId })` khi muốn chờ parent ack.
- Nếu không cần chờ, gọi `process.send` trực tiếp hoặc gọi helper với timeout ngắn.

4. Parent-side: `ChildProcessWrapper` / `ChildProcessManager`

- Trong `ChildProcessWrapper` (file `src/workers/childProcessManager.ts`) thêm:
  - `emitHandlers: Map<string, (wrapper, msg) => Promise<any>>`
  - `registerEmitHandler(eventName, handler)` API để đăng ký handler cụ thể (ví dụ `chat_update`, `stream_chunk`).
- Trong `child.on('message', msg => { ... })` thêm branch xử lý khi `msg.type === IPCCommonType.EMIT`:
  - lookup handler = `this.emitHandlers.get(msg.event)` hoặc fallback default.
  - await handler(this, msg) (bảo đảm try/catch)
  - reply `this.child.send({ type: IPCCommonType.EMIT_ACK, id: msg.id ?? null, emitId: msg.emitId, payload })`.

Mẫu code (rút gọn):

```ts
// inside ChildProcessWrapper constructor
this.emitHandlers = new Map();

this.child.on('message', (msg:any) => {
  if (!msg || !msg.type) return;
  if (msg.type === IPCCommonType.EMIT) {
    (async () => {
      const handler = this.emitHandlers.get(msg.event);
      try {
        const res = handler ? await handler(this, msg) : { success: true };
        this.child.send({ type: IPCCommonType.EMIT_ACK, id: msg.id ?? null, emitId: msg.emitId, payload: res });
      } catch (err) {
        this.child.send({ type: IPCCommonType.EMIT_ACK, id: msg.id ?? null, emitId: msg.emitId, payload: { success: false, error: String(err) } });
      }
    })();
    return;
  }
  // existing message handling...
});

// registration helper
public registerEmitHandler(evt:string, handler:(w:ChildProcessWrapper,msg:any)=>Promise<any>) {
  this.emitHandlers.set(evt, handler);
}
```

5. Ví dụ handler: forward tới socket.io và ack khi client ack

```ts
manager
  .getChildByModuleId(ModuleId.HOST_AGENT)
  ?.registerEmitHandler("stream_chunk", async (wrapper, msg) => {
    const runId = msg.id;
    const socketId = runIdToSocketId.get(runId);
    if (!socketId) return { success: false, error: "no-socket" };
    return new Promise((resolve) => {
      io.to(socketId).emit("stream_chunk", msg.payload, (clientAck: any) => {
        if (clientAck && clientAck.ok) resolve({ success: true });
        else
          resolve({ success: false, error: clientAck?.error ?? "client-nack" });
      });
      // optionally local timeout here
    });
  });
```

6. Symmetry: parent -> child emit + ack

- Nếu parent cần emit tới child và chờ ack, implement tương tự:
  - parent tạo `emitId`, gửi `EMIT` tới child.
  - child thêm case `EMIT` (cũng có thể reuse same handler registry on child-side), trả `EMIT_ACK` về parent.

7. Migration checklist (thực hiện theo thứ tự)

- [ ] Backup / commit current state.
- [ ] Thêm `EMIT`/`EMIT_ACK` vào `ipc.types`.
- [ ] Tạo `src/workers/workerEmit.ts` (child helper).
- [ ] Import helper vào `hostAgent.childProcess.ts`, `rag.childProcess.ts`, `crawlServer.childProcess.ts` và thay thế 1-2 emit quan trọng (ví dụ `chat_update`, `stream_chunk`) để dùng helper.
- [ ] Cập nhật `ChildProcessWrapper` trong `childProcessManager.ts`: thêm `emitHandlers`, `registerEmitHandler` và branch `EMIT` để trả `EMIT_ACK`.
- [ ] Đăng ký handlers tiêu chuẩn (stream/result/status) khi khởi tạo manager.
- [ ] Viết smoke tests: child emit test event -> parent ACK -> child resolves; parent emit -> child ACK.
- [ ] Tinh chỉnh timeout qua env `CHILD_EMIT_ACK_TIMEOUT_MS` hoặc config.
- [ ] Triển khai dần: chuyển các emit khác sang helper, monitor logs.

8. Test & vận hành

- Manual smoke: bật parent trong dev, fork 1 child, chạy 1 request thử streaming, confirm child `emitToParentAwaitAck` resolves và parent forwarded message tới socket (và socket ack) trước khi parent trả EMIT_ACK.
- Viết unit test cho `workerEmit` bằng cách mock `process.send` và gửi `EMIT_ACK` message vào `process.on('message')`.

9. Lưu ý vận hành

- Chọn timeout mặc định ngắn (3–5s) để tránh block child lâu.
- Nếu parent quá tải, trả nhanh `success:false` với `error: 'parent-busy'` giúp child degrade gracefully.
- Bảo đảm `id` (runId) được set cho mỗi run để map tới socket/connection.

Tài liệu tham khảo nhanh

- File cần chỉnh: `src/workers/childProcessManager.ts`, `src/workers/hostAgent.childProcess.ts`, `src/workers/rag.childProcess.ts`, `src/workers/crawlServer.childProcess.ts`, `src/workers/ipc.types.ts`, `src/workers/workerEmit.ts` (mới).

---

Nếu bạn muốn, tôi sẽ tiếp tục tạo patch code cho:

- `src/workers/workerEmit.ts` (thực tế) và
- cập nhật `childProcessManager.ts` với `emitHandlers` + một ví dụ handler `stream_chunk`.

Bạn muốn tôi tạo patch tiếp không?
