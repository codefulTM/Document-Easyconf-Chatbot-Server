Kế hoạch triển khai: Chạy mọi hàm của `ConferenceSyncService` trong 1 child process persist

Mục tiêu

- Mọi lời gọi tới `ConferenceSyncService` (từ service/controller/...) sẽ được thực thi trong một tiến trình con (child process) persist.
- Child process giữ DI, DB pool, cache, và phục vụ nhiều request liên tiếp (không spawn/kill mỗi lần gọi).

Tóm tắt approach (Hướng 1 - recommended)

- Parent (main process) fork một child persistent bằng `child_process.fork`.
- Parent gửi RPC messages tới child: `{ id, type: 'RUN', method, params }`.
- Child thực thi method trên `ConferenceSyncService` rồi trả `FINAL_RESULT` hoặc `CONFERENCE_SYNC_ERROR`.
- Parent resolve/reject Promise tương ứng khi nhận response.

1. Thiết kế giao thức RPC (message format)

- Sử dụng các hằng `ipc.types.ts` đã có để giữ nhất quán với hệ thống hiện tại.
  - Parent → Child (request):
    - `{ id: string, type: 'RUN', method: string, params: any[] }`

- Child → Parent (responses/events):
  - Progress: `{ id, type: 'PROGRESS', payload }`
  - Result (final): `{ id, type: 'FINAL_RESULT', payload }` (kết quả đầy đủ)
  - Error (domain): `{ id, type: 'CONFERENCE_SYNC_ERROR', error }` (nếu là lỗi nội dung/chat)
  - Error (control): `{ id, type: 'ERROR', error }` (control-level / IPC error)
  - Ready: `{ type: 'WORKER_READY', pid, module }`
  - Ghi chú: luôn dùng `id` (UUID) để correlate. Nếu child gửi nhiều event con (ví dụ progress), dùng `emitId` cho từng emit nếu cần ACK (`EMIT_ACK`).

2. Tạo worker (child process)

- File gợi ý: `src/workers/conferenceSync.childProcess.ts`
- Startup:
  - Import `reflect-metadata`, khởi tạo `container` (tsyringe), resolve `PgClient`, `ConferenceSyncService`.
  - Warm-up: gọi init trên các service cần thiết (vd. connect/builder, load cache).
  - Khi sẵn sàng gửi `{ type: 'WORKER_READY', pid, module: 'conferenceSync' }`.
- Message handler (rút gọn):
  - Lắng nghe `process.on('message', async (msg) => { ... })`.
  - Nếu `msg.type === 'RUN'`:
    - Map `method` → kiểm tra tồn tại trên `ConferenceSyncService`.
    - Gọi `await svc[method](...msg.params)`.
    - Gửi kết quả `{ id, type: 'FINAL_RESULT', payload }` hoặc lỗi `{ id, type: 'CONFERENCE_SYNC_ERROR', error }`.
  - Hỗ trợ `PREPARE_DRAIN` để hoàn tất in-flight tasks rồi exit.
- Concurrency: child có thể xử lý đồng thời (với limit) hoặc tuần tự (queue). Implement semaphore/p-limit khi cần.

3. Thêm helper ở parent (`ChildProcessManager`)

- Tạo hàm `callConferenceSyncOnChildProcess(method: string, params: any[], timeoutMs?: number)`.
- Behavior:
  - Get or start child process with module id `'conferenceSync'`.
  - Create UUID `id`, attach `child.on('message', onMessage)`.
  - Send request: `child.send({ type: 'RUN', id, method, params })`.
  - Handle messages:
    - `FINAL_RESULT` with same `id` → resolve(payload)
    - `CONFERENCE_SYNC_ERROR` → reject(Error)
    - `PROGRESS` → optional callback để forward logs/socket
  - Timeout: reject if vượt `timeoutMs`.
  - Cleanup: `child.off('message', onMessage)` và clear timeout.

4. Proxy / Replace callers

- Tạo `ConferenceSyncProxy` (module nhỏ) hoặc trực tiếp thay các nơi `container.resolve(ConferenceSyncService)` gọi proxy helper.
- Proxy method trả Promise giống API gốc.
- Ví dụ replacement:
  - Trước: `await conferenceSyncService.syncConference(id)`
  - Sau: `await callConferenceSyncOnChildProcess('syncConference', [id])`
- Đảm bảo các chỗ gọi xử lý Promise (await) bình thường.

5. Error handling & restart

- Distinguish:
  - Domain error (vd: not found) → return CONFERENCE_SYNC_ERROR but do not crash.
  - Operational error (e.g., uncaught exception) → child may crash; parent phải recover.
- Parent policy:
  - Khi child crash: reject in-flight requests; restart child (exponential backoff).
  - Optionally: requeue idempotent requests.

6. Graceful shutdown

- Parent gửi `PREPARE_DRAIN` khi shutdown; child hoàn thành in-flight rồi exit.
- Child trả `READY_TO_EXIT` hoặc exit code 0.

7. Observability

- Log mỗi requestId, method, start/end, duration, error.
- Metrics: in-flight, avg latency, errors/sec.
- Nếu child gửi progress events, parent có thể proxy sang socket và gửi EMIT_ACK trả về child để child biết event đã được gửi.

8. Tests & rollout

- Unit tests cho helper: mock child messages.
- Smoke-test: start server, call endpoint that triggers `ConferenceSyncService` proxy method, verify result.
- Crash recovery test: kill child process during requests and ensure restart + meaningful error for in-flight.

9. Code sketches (rút gọn)

- Parent helper (childProcessManager additions):

```ts
export const callConferenceSyncOnChildProcess = async (
  method: string,
  params: any[] = [],
  timeoutMs = 60000,
) => {
  const manager = container.resolve(ChildProcessManager);
  let w = manager.getChildByModuleId("conferenceSync");
  if (!w) {
    w = manager.startChildProcess("conferenceSync", WORKER_PATHS.HOST_AGENT);
  }
  const child = w.child;
  return new Promise((resolve, reject) => {
    const id = uuidv4();
    const onMessage = (msg: any) => {
      if (!msg || msg.id !== id) return;
      if (msg.type === "FINAL_RESULT") {
        cleanup();
        resolve(msg.payload);
      } else if (msg.type === "CHAT_ERROR" || msg.type === "ERROR") {
        cleanup();
        reject(new Error(msg.error || msg.payload || String(msg)));
      } else if (msg.type === "PROGRESS") {
        /* optional: forward progress */
      }
    };
    const cleanup = () => {
      child.off("message", onMessage);
      clearTimeout(timer);
    };
    child.on("message", onMessage);
    child.send({ type: "RUN", id, method, params });
    const timer = setTimeout(() => {
      cleanup();
      reject(new Error("timeout"));
    }, timeoutMs);
  });
};
```

- Child minimal:

```ts
// conferenceSync.childProcess.ts (example using existing IPC types)
import "reflect-metadata";
import "../container";
import { container } from "tsyringe";
import { ConferenceSyncService } from "../chatbot/services/sync/conferenceSyncService";

process.on("message", async (msg) => {
  if (!msg || msg.type !== "RUN") return;
  const { id, method, params } = msg;
  try {
    const svc = container.resolve(ConferenceSyncService);
    const result = await svc[method](...(params || []));
    process.send?.({ id, type: "FINAL_RESULT", payload: result });
  } catch (err) {
    // domain-level chat errors -> CHAT_ERROR; other control errors -> ERROR
    process.send?.({ id, type: "ERROR", error: String(err) });
  }
});
process.send?.({
  type: "WORKER_READY",
  pid: process.pid,
  module: "conferenceSync",
});
```

10. Checklist ngắn (thực hiện theo thứ tự)

- [ ] Tạo `src/workers/conferenceSync.childProcess.ts` (worker)
- [ ] Thêm `callConferenceSyncOnChildProcess` trong `src/workers/childProcessManager.ts`
- [ ] Start child `conferenceSync` trong `src/server.ts` (single process)
- [ ] Thay 1-2 callers ví dụ để dùng proxy (smoke test)
- [ ] Implement concurrency limit & queue trong child
- [ ] Add logging/metrics
- [ ] Tests + smoke run
- [ ] Add graceful drain + restart/backoff

Muốn mình tiếp tục và hiện thực hóa (code) các bước đầu (tạo worker + helper + update `server.ts` + thay 1 caller ví dụ) không? Nếu có, mình sẽ implement A1 và chạy quick smoke (nếu bạn muốn).
