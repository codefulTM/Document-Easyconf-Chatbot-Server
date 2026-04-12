# Kế hoạch: Proxy các route `crawl` từ parent → child process (HTTP child)

Mục tiêu: khi client gọi các API crawl ở process cha, process cha sẽ proxy (forward/stream) HTTP request đến server chạy trong process con, process con sẽ trực tiếp gọi các hàm `handle...` (ví dụ `handleCrawlConferences`, `handleCrawlJournals`, `handleStopCrawl`) và trả response lại; client không biết request được thực thi ở process nào.

Tổng quan giải pháp được đề xuất

- Dùng `child_process.fork()` để spawn child Node process chạy một Express HTTP server (đặt port động). Child server sẽ mount cùng router (`createCrawlRouter` hoặc `createV1Router`) để reuse handlers hiện có.
- Parent dùng `http-proxy-middleware` (hoặc `http-proxy`) để proxy toàn bộ request/response (stream) tới child target `http://127.0.0.1:<port>`. Bằng cách này không cần serialize `req`/`res` (chúng không thể chuyển qua IPC), vì proxy stream giữ socket nguyên vẹn.

Ràng buộc kỹ thuật quan trọng

- Không thể serializable `req`/`res` qua IPC — phải dùng streaming proxy hoặc transfer socket handle. Ở đây chọn streaming proxy vì đơn giản và cross-platform.
- Child phải export/khởi tạo các route/handler theo cách có thể mount vào Express app riêng (hiện code controller của bạn là functions `handleCrawlConferences(...)` v.v. nên có thể reuse).

Các bước triển khai (chi tiết)

1. Thêm script child HTTP server

- File: `src/workers/crawlServer.childProcess.ts` (một script TypeScript nhỏ)
- Chức năng:
  - Tạo Express app, `app.use(express.json())`.
  - Import và mount `createCrawlRouter()` tại path tương ứng (ví dụ mount router tại `/api/v1/crawl`).
  - Lắng nghe port được chỉ định qua env `PORT`.
  - Sau khi `listen()` thành công, gửi IPC message `WORKER_READY` (hoặc `process.send({ type: 'WORKER_READY', pid: process.pid, module: process.env.MODULE_ID })`) để parent biết worker sẵn sàng.
  - Hỗ trợ graceful shutdown khi process nhận `PREPARE_DRAIN` message từ parent (đóng server, chờ kết thúc request hiện tại rồi exit).

2. Spawn child HTTP worker (parent)

- Tái sử dụng `ChildProcessManager` hoặc thêm helper `startHttpChild(moduleId, scriptPath)`;
- Phong cách:
  - Lấy port trống (ví dụ `getFreePort()` dùng `net.createServer().listen(0)`), sau đó `fork()` child script với env: `{ ...process.env, MODULE_ID: moduleId, PORT: String(port) }` và `execArgv: ['-r','ts-node/register']` (giữ consistent với project).
  - Lưu mapping `moduleId -> { pid, port, wrapper }` trong một `WorkerRegistry` (ví dụ `src/workers/workerRegistry.ts`) để proxy biết target.
  - Khi child gửi `WORKER_READY`, đánh dấu worker `healthy`.

3. Parent proxy middleware

- Thêm module proxy: `src/lib/childProxy.ts` (hoặc `src/middleware/childProxy.ts`) dùng `http-proxy-middleware`.
- Cấu hình proxy với `router` function hoặc `router` map để chọn target dựa trên route.
- Ví dụ: proxy tất cả path bắt đầu bằng `/api/v1/crawl` → route to `http://127.0.0.1:${port}` lấy từ `WorkerRegistry.getTarget('crawl')`.

4. Tích hợp vào router hiện tại

- Option A (ít thay đổi): Trong `src/api/v1/index.ts` hoặc `src/api/v1/crawl/crawl.routes.ts`, thay handler ban đầu bằng proxy middleware cho những route muốn forward.
  - Ví dụ, trong `createCrawlRouter()` thay `router.post('/crawl-conferences', handleCrawlConferences)` thành `router.use('/crawl-conferences', proxyTo('crawl'))` hoặc mount `router.use('/api/v1/crawl', proxyMiddleware)` ở cấp cao hơn.
- Option B (toàn cục): Mount proxy trước routers trong `createV1Router` cho các prefix cụ thể.

5. Health-check, timeouts, fallback

- Nếu worker chưa sẵn sàng hoặc unreachable, parent trả `503` hoặc enqueue request tùy yêu cầu. Nên cấu hình `proxy` với timeouts và circuit-breaker (ví dụ: nếu worker chết nhiều lần, tạm thời trả lỗi và trigger restart/backoff).

6. Graceful shutdown & cleanup

- Parent phải gửi `IPCCommonType.PREPARE_DRAIN` (hoặc custom message) rồi chờ child finish current requests.
- Child khi nhận message này sẽ `server.close()` và exit khi xong.

7. Feature toggle & rollout

- Bật bằng env var `ENABLE_CHILD_HTTP_PROXY=true` để dễ rollback.
- Deploy incremental: bật cho dev/test, chuyển một route nhỏ trước (ví dụ `/crawl-journals`), kiểm thử, sau đó chuyển các route lớn.

Gợi ý thay đổi file cụ cụ thể (đề xuất)

- Thêm file mới: `src/workers/crawlServer.childProcess.ts` (server chạy Express, mount crawl router)
- Thêm file mới: `src/workers/workerRegistry.ts` (map moduleId→{pid,port,status})
- Thêm/extend `ChildProcessManager` (hoặc new helper) để start http child với port env
  - Nếu chỉnh `ChildProcessManager.startChildProcess`, thêm optional arg `env?: Record<string,string>` để có thể truyền `PORT`.
- Thêm file proxy: `src/lib/childProxy.ts` sử dụng `http-proxy-middleware`.
- Thay đổi nhẹ `src/api/v1/crawl/crawl.routes.ts` hoặc `src/api/v1/index.ts` để mount proxy cho các endpoint đã đánh dấu.

Ví dụ code — CHILD (minimal)

```ts
// src/workers/crawlServer.childProcess.ts
import "reflect-metadata";
import express from "express";
import createCrawlRouter from "../api/v1/crawl/crawl.routes";

const app = express();
app.use(express.json({ limit: "50mb" }));
// Mount the same crawl router so handlers run in child process
app.use("/api/v1/crawl", createCrawlRouter());

const port = Number(process.env.PORT || 4001);
const server = app.listen(port, () => {
  if (process.send) {
    process.send({
      type: "WORKER_READY",
      pid: process.pid,
      module: process.env.MODULE_ID,
    });
  }
  console.log(`crawlServer child process listening ${port}`);
});

process.on("message", (msg: any) => {
  if (msg && msg.type === "PREPARE_DRAIN") {
    server.close(() => process.exit(0));
    // consider setTimeout to force-exit if not closed after N ms
  }
});
```

Ví dụ code — PARENT proxy middleware (minimal)

```ts
// src/lib/childProxy.ts
import { createProxyMiddleware } from "http-proxy-middleware";
import WorkerRegistry from "../workers/workerRegistry";

export function crawlProxyMiddleware() {
  return createProxyMiddleware({
    changeOrigin: true,
    router: (req) => {
      const target = WorkerRegistry.getTargetForModule("crawl");
      return target || "http://127.0.0.1:4001";
    },
    onError: (err, req, res) => {
      res.statusCode = 502;
      res.end(
        JSON.stringify({ message: "Worker unreachable", error: String(err) }),
      );
    },
    logLevel: "warn",
    // Keep streaming behavior default (no body buffering)
  });
}
```

Trong `src/api/v1/crawl/crawl.routes.ts`

```ts
import { crawlProxyMiddleware } from "../../lib/crawlProxy";

const router = Router();
if (process.env.ENABLE_CHILD_HTTP_PROXY === "true") {
  // Proxy all crawl routes to child
  router.use("/", crawlProxyMiddleware());
} else {
  router.post("/crawl-conferences", handleCrawlConferences);
  router.post("/crawl-conferences/stop", handleStopCrawl);
  router.post("/crawl-journals", handleCrawlJournals);
}
```

Kiểm thử & rollout

- Local: bật `ENABLE_CHILD_HTTP_PROXY=true`, start parent; parent sẽ fork một child thông qua helper `workerRegistry.spawn('crawl')` (cần implement) và mount proxy.
- Test: gửi POST `/api/v1/crawl/crawl-conferences` với small payload, kiểm tra child logs và response.
- Streaming/large payload: test với CSV lớn; nếu performance/timing issue thì hãy dùng temp-file path passing: parent lưu tạm file và truyền path trong body tới child.

Chú ý vận hành & pitfalls

- Đồng bộ middleware phụ thuộc vào process-level singletons (ex: container, DB client): child sẽ có own DI container instance. OK, nhưng cần cấu hình kết nối DB/Redis để child có quyền truy cập và credentials.
- Logging: child phải tạo request-specific logger dựa trên `batchRequestId` giống parent.
- Security: giới hạn child chỉ bind `127.0.0.1` để tránh public exposure; proxy in parent có thể check authorization if needed.
- Lifecyle: implement restart/backoff để tránh crash-loop; parent phải detect child dead và spawn lại hoặc trả lỗi 5xx.

Tài liệu tham khảo trong repo

- `src/workers/childProcessManager.ts` — hiện đã có logic fork/IPC; có thể tái sử dụng để track pid/module -> dùng làm mẫu cho start/stop policy.
- `src/workers/hostAgent.childProcess.ts`, `src/workers/rag.childProcess.ts` — ví dụ worker init, readiness signaling và PREPARE_DRAIN handling.

Kết luận ngắn

- Giải pháp proxy HTTP giữ được client URL không đổi và cho phép child sử dụng `req`/`res` Express trực tiếp (không serialize). Đây là cách an toàn, đơn giản và cross-platform so với việc transfer socket handle trực tiếp. Nên bắt đầu bằng 1 route nhỏ, test kỹ (timeouts, streaming), rồi mở rộng.

Nếu OK, tôi có thể: (A) scaffold các file mẫu (`crawlServer.childProcess.ts`, `workerRegistry.ts`, `childProxy.ts`) trong repo và cập nhật `crawl.routes.ts` để bật proxy theo `ENABLE_CHILD_HTTP_PROXY`; hoặc (B) chỉ tạo file plan này và để bạn review trước khi code.
