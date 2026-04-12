# Kế hoạch: Tách phần công việc nặng sang tiến trình con — Các bước triển khai (số thứ tự)

Dưới đây là các bước chi tiết theo thứ tự (1 → N). Thực hiện tuần tự từng bước, kiểm tra và chạy smoke test sau mỗi bước lớn.

1.  Tạo scaffold worker `src/workers/crawl.childProcess.ts`

- Gắn `process.on('message')` ở top-level ngay, đặt `ready = false` và `pendingQueue: any[] = []`.
- Triển khai `handleRun(msg)` để nhận `RUN` và schedule công việc vào `p-limit` (không await khi flush queue).
- Viết `startWorker()` để init `LoggingService`, `ConfigService`, DB/PG client, và service cần thiết; sau khi init set `ready = true` và gửi `WORKER_READY`.
- Hỗ trợ message types: `RUN`, `PREPARE_DRAIN`, `UNCAUGHT_EXCEPTION`, `FINAL_RESULT`, `ERROR`.

2.  Thêm helper và mapping worker path

- Mở `src/workers/childProcessManager.ts` và thêm `WORKER_PATHS.CRAWL = 'crawl'`.
- Thêm hàm `callCrawlOnChildProcess(method: string, params: any[]): Promise<any>` theo pattern per-call listener:
- Tạo `id` unique cho mỗi call.
- Đăng ký listener một lần để lắng nghe `FINAL_RESULT` / `ERROR` cho `id` đó.
- Gửi message `RUN` tới tiến trình tương ứng: `{ id, payload: { method, params } }`.
- Trả về Promise resolve/reject khi nhận được response.

3.  Tạo `CrawlProxy` (proxy DI)

- Tạo file `src/proxies/crawl.proxy.ts` với `@singleton` class `CrawlProxy`.
- Cài đặt API methods:
- `async runConferences(payload): Promise<ProcessedRowData[] | void>` → gọi `callCrawlOnChildProcess('crawl.runConferences', [payload])`.
- `async runConferencesBackground(payload): Promise<{ accepted: true, batchRequestId }>` → gửi call non-blocking và trả acknowledgement.
- `async stopCrawl(batchRequestId): Promise<void>` → gọi `callCrawlOnChildProcess('crawl.stop', [batchRequestId])`.
- Proxy chỉ gửi dữ liệu serializable (không truyền `requestContainer`).

4.  Đăng ký proxy vào DI container

- Mở `src/container.ts` và `registerSingleton(CrawlProxy)` (hoặc tương ứng với pattern hiện có).

5.  Cập nhật `crawl.controller.ts` để dùng `CrawlProxy` (luôn bật)

- `CrawlProxy` luôn được sử dụng; không cần feature flag.
- Resolve `CrawlProxy` từ container.
- Chuẩn bị payload serializable: `{ batchRequestId, items, models: parsedApiModels, recordFile }`.
- Với `mode=sync`: gọi `await crawlProxy.runConferences(payload)` và trả kết quả cho client.
- Với `mode=async`: gọi `await crawlProxy.runConferencesBackground(payload)` → trả 202 + `batchRequestId`.
- Nếu worker không sẵn sàng hoặc có lỗi IPC, xử lý theo chính sách retry hoặc trả lỗi/ack phù hợp (ví dụ 503 hoặc 202 với trạng thái pending).

6.  Khởi động và chờ worker lúc bootstrap

- Trong `src/server.ts` (hoặc điểm bootstrap tương ứng): gọi `await ChildProcessManager.startChildProcess('crawl', WORKER_PATHS.CRAWL)` và đợi `WORKER_READY` trước khi mở port.
- Nếu worker thất bại, log và fallback theo chính sách (retry/backoff hoặc disable feature).

7.  Chunking và giới hạn payload

- Thêm config `CRAWL_CHUNK_SIZE` và `MAX_ITEMS_PER_MESSAGE`.
- Trước khi gửi payload lớn, validate kích thước `items`; nếu lớn hơn `MAX_ITEMS_PER_MESSAGE`, chia thành nhiều RUN (ví dụ `crawl.runConferencesChunk`) và gom kết quả lại.
- Nếu không thể chunk ở server, trả 413 và hướng dẫn client upload qua file/URL.

8.  Dừng job / Stop handling

- Đảm bảo worker xử lý `crawl.stop` để cancel/mark job dừng.
- `CrawlProcessManagerService.requestStop(batchRequestId)` ở parent chuyển tiếp tới child bằng `callCrawlOnChildProcess('crawl.stop', [batchRequestId])`.

9.  Background mode & persistence

- Với `mode=async`, worker chịu trách nhiệm persist kết quả (DB hoặc object storage) và trả `batchRequestId`.
- Nếu cần, worker có thể gửi các cập nhật tiến độ (`STATUS`) về parent qua IPC.

10. Kiểm thử

- Unit tests:
- Mock `childProcessManager` để test `CrawlProxy`.
- Test error handling và timeouts.
- Integration tests:
- Khởi worker thực tế trong môi trường test/dev, gửi sample `conferenceList` nhỏ và verify `FINAL_RESULT`.

11. Giám sát & health

- Ghi metric: job duration, queue length, concurrency, memory/CPU per worker.
- Parent có health check cho worker readiness (dựa trên `WORKER_READY`) và expose endpoint health nếu cần.

12. Shutdown an toàn & restart

- Hỗ trợ `PREPARE_DRAIN` để dừng nhận job mới và đợi p-limit drain.
- Triển khai restart/backoff cho worker (supervisor) để tránh crash-loop.

13. Triển khai và rollout

- Bật feature flag cho `mode=async` trước; giám sát memory/CPU và logs.
- Nếu ổn định, mở cho `mode=sync` (blocking).

14. Tài liệu & runbook

- Cập nhật README, runbook, và playbook xử lý lỗi (workers OOM, IPC timeout, partial failures).

15. Checklist triển khai trước khi mở rộng

- Unit + integration tests passed.
- Smoke tests cho payload lớn (chunking) OK.
- Metrics & alerting thiết lập.
- Rollout plan và feature flag sẵn sàng.

Nếu bạn đồng ý, mình sẽ bắt đầu thực thi bước 1 (scaffold `src/workers/crawl.childProcess.ts`).
