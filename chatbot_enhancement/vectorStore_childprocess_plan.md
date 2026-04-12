Kế hoạch triển khai: Chạy `VectorStoreService` (và tùy chọn `EmbeddingService`) trong tiến trình con persist

Mục tiêu

- Chạy mọi thao tác liên quan tới vector store (upsert, search, delete, scroll, stats) trong một tiến trình con persist.
- Giữ client tới Qdrant trong tiến trình con (giảm tải I/O/latency trong tiến trình chính).
- Tách phần nặng (nếu có) như model embedding ra khỏi tiến trình chính để tránh nhân đôi memory trên nhiều tiến trình.

Nhận xét nền tảng (từ `vectorStoreService.qdrant.ts`)

- `VectorStoreService` hiện tại chỉ khởi tạo `QdrantClient` và gọi HTTP API — bản thân service nhẹ (vài MB).
- Phần nặng thật sự là `EmbeddingService` (`@xenova/transformers`) nếu model được load vào tiến trình — mỗi tiến trình load model sẽ tốn tens→hundreds MB.
- Qdrant được cấu hình `on_disk/memmap` trong code, do đó RAM lớn nằm ở Qdrant process thay vì Node, nếu cấu hình đúng.

Đề xuất tổng quan (Recommended)

- Tạo 1 tiến trình con `vectorStore` chịu trách nhiệm cho mọi tương tác với Qdrant.
- Nếu `EmbeddingService` là nhẹ/không được tải: có thể load `EmbeddingService` trong cùng tiến trình `vectorStore`.
- Nếu `EmbeddingService` tải model lớn: tách ra một tiến trình `embedding` riêng (single loader) để tránh nhân đôi model trong nhiều tiến trình; `vectorStore` gọi `embedding` qua IPC khi cần embeddings.

Chi tiết kỹ thuật

1. Giao thức RPC (Parent ↔ Child)

- Parent → Child: `{ id, type: 'RUN', method: string, payload?: any }`
- Child → Parent:
  - `FINAL_RESULT` : `{ id, type: 'FINAL_RESULT', payload }`
  - `VECTORSTORE_ERROR` (domain): `{ id, type: 'VECTORSTORE_ERROR', error }`
  - `ERROR` (control): `{ id, type: 'ERROR', error }`
  - `WORKER_READY` : `{ type: 'WORKER_READY', pid, module }`
  - `PROGRESS` (optional streaming): `{ id, type: 'PROGRESS', payload, emitId? }`
  - `EMIT_ACK` support if child wants parent to confirm socket emits.

2. File worker

- Tên gợi ý: `src/workers/vectorStore.childProcess.ts`.
- Startup:
  - import `reflect-metadata`; init `container` (tsyringe);
  - resolve `LoggingService`, `ConfigService`, khởi tạo `VectorStoreService` (container.resolve) và call `init()` nếu cần;
  - Nếu chọn gộp embeddings: resolve `EmbeddingService` và `await init()` (LƯU Ý: chỉ làm nếu chấp nhận memory cost trong tiến trình này).
  - Khi hoàn tất startup gửi `WORKER_READY`.
- Message handler: `process.on('message', async (msg) => { ... })` xử lý `RUN` theo `method`/`payload`.
- Concurrency: sử dụng `p-limit` (configurable via `WORKER_CONCURRENCY`) để giới hạn số thao tác đồng thời (search/upsert).
- PREPARE_DRAIN: khi nhận, đặt flag draining và chờ in-flight complete rồi exit.

3. Parent helper

- Thêm `callVectorStoreOnChildProcess(method, params, moduleId='vectorStore', timeoutMs?)` trong `ChildProcessManager`.
- Behavior: start hoặc reuse child, await `ensureReady()` (no-timeout or configurable), send `RUN` và resolve/reject trên `FINAL_RESULT`/`VECTORSTORE_ERROR`/`ERROR`.

4. Proxy / DI

- Tạo `src/proxies/vectorStore.proxy.ts` (singleton) mirroring public API của `VectorStoreService` (`init`, `addDocuments`, `search`, `getChunkById`, `deleteDocuments`, `deleteByRecordId`, `clear`, `getQdrantHealth`, ...).
- Các method gọi `callVectorStoreOnChildProcess(methodName, params)`.
- Đăng ký proxy trong `container.ts` để dễ chuyển đổi (container.registerSingleton<VectorStoreService>('VectorStoreService', VectorStoreProxy)) nếu muốn thay đổi hành vi.

5. Embedding model considerations

- Nếu `EmbeddingService` tải model lớn, KHÔNG nên load model trong mỗi tiến trình. Các lựa chọn:
  - Option A (recommended): Tạo tiến trình `embedding` singleton giữ `EmbeddingService` và expose RPC `embed(texts[])` → trả embedding vectors. `vectorStore` gọi `embedding` qua IPC.
  - Option B: Chỉ load `EmbeddingService` trong `vectorStore` (chỉ nếu bạn muốn hợp nhất và chấp nhận memory tăng tại worker).
  - Option C: Keep embedding in main process and call it synchronously — this defeats purpose of offloading memory; not recommended.

6. Readiness & ordering

- Worker gửi `WORKER_READY` khi container + services đã init.
- Parent `startChildProcess` trả wrapper có `ready`/`ensureReady()` promise. Các helper (proxy/init) phải `await` readiness before sending RPC.
- `server.ts` should start child and await ready before calling `vectorStoreProxy.init()` or before allowing traffic that uses it.

7. Error handling & restart

- Đặc biệt phân biệt DOMAIN vs OPERATIONAL errors: `VECTORSTORE_ERROR` cho lỗi nghiệp vụ; `ERROR`/`UNCAUGHT_EXCEPTION` cho lỗi control.
- Khi child crash: reject in-flight, restart với exponential backoff; non-idempotent operations phải retry cẩn trọng.

8. Observability

- Log requestId, method, duration, errors inside worker.
- Expose health endpoint via proxy: `getQdrantHealth()` và map ra /health/qdrant.
- Metrics: in-flight, queue-length (p-limit queue), avg latency, errors/sec.

9. Tests & rollout

- Unit test helper bằng cách mock `child.on('message')` events.
- Smoke test: start server, call endpoint that triggers `vectorStoreProxy.addDocuments` và `search`.
- Crash test: kill worker trong khi có request in-flight, đảm bảo parent restart và in-flight requests fail fast.

10. Code sketches (parent helper)

```ts
export const callVectorStoreOnChildProcess = async (
  method: string,
  params: any[] = [],
  moduleId = "vectorStore",
  timeoutMs = 60_000,
) => {
  const manager = container.resolve(ChildProcessManager);
  let w = manager.getChildByModuleId(moduleId);
  if (!w) w = manager.startChildProcess(moduleId, WORKER_PATHS.VECTOR_STORE);
  await w.ensureReady(); // wait until WORKER_READY
  const child = w.child;
  return new Promise((resolve, reject) => {
    const id = uuidv4();
    const onMessage = (msg: any) => {
      if (!msg) return;
      if (msg.id != null && msg.id !== id) return;
      if (msg.id == null) {
        if (
          msg.type === IPCCommonType.ERROR ||
          msg.type === IPCCommonType.UNCAUGHT_EXCEPTION
        ) {
          cleanup();
          reject(new Error(String(msg.error || msg)));
        }
        return;
      }
      if (msg.type === IPCCommonType.FINAL_RESULT) {
        cleanup();
        resolve(msg.payload);
      } else if (msg.type === IPCCommonType.VECTORSTORE_ERROR) {
        cleanup();
        reject(new Error(msg.error));
      } else if (msg.type === IPCCommonType.ERROR) {
        cleanup();
        reject(new Error(msg.error));
      }
    };
    child.on("message", onMessage);
    child.send({ type: IPCCommonType.RUN, id, payload: { method, params } });
    const timer = setTimeout(() => {
      cleanup();
      reject(new Error("timeout"));
    }, timeoutMs);
    const cleanup = () => {
      child.off("message", onMessage);
      clearTimeout(timer);
    };
  });
};
```

11. Checklist (thực hiện theo thứ tự)

- [ ] Add `src/workers/vectorStore.childProcess.ts` (worker)
- [ ] Add `WORKER_PATHS.VECTOR_STORE` and `callVectorStoreOnChildProcess` to `childProcessManager.ts`
- [ ] Add `src/proxies/vectorStore.proxy.ts` and register in DI if desired
- [ ] Start `vectorStore` child in `server.ts` and wait readiness before allowing calls
- [ ] If embedding model heavy: implement `embedding` worker and update `vectorStore` to call it
- [ ] Add concurrency limit (`p-limit`) and PREPARE_DRAIN handling
- [ ] Add health/metrics endpoints and tests

Ghi chú cuối

- Mục tiêu chính: tránh duplicate heavy memory (model) giữa nhiều tiến trình. Nếu `EmbeddingService` không tải model, tác động bộ nhớ rất nhỏ; nếu nó tải model, hãy tách model ra 1 tiến trình duy nhất.
