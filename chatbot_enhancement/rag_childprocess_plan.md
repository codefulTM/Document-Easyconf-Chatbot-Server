# Kế hoạch triển khai RAG (Retrieval-Augmented Generation) child process — Cập nhật theo dịch vụ hiện có

Mục tiêu: chạy các thành phần nặng trạng thái (mô hình Embedding, adapter/khách hàng VectorStore) trong một hoặc vài tiến trình con (child processes) để:

- Tránh tải lại mô hình nhiều lần trên các tiến trình Node khác nhau.
- Dễ thu thập heap-snapshot / khởi động lại cách ly khi gặp rò rỉ bộ nhớ.
- Cải thiện khả năng quan sát và điều khiển (ready/drain/health).

Tổng quan các dịch vụ hiện có (thư mục `src/chatbot/services/rag`):

- `embeddingService.ts`: tải mô hình Xenova và sinh vector embedding (thành phần nặng nhất).
- `vectorStoreService.qdrant.ts`: adapter Qdrant (client HTTP), nhẹ hơn về bộ nhớ nhưng chịu trách nhiệm I/O và batching.
- `retrievalService.ts`: lớp điều phối (orchestration) kết hợp chunking + embedding + search.
- `chunkingService.ts`: chia văn bản thành các chunk để embed/index.
- `stringNormalize.ts`: chuẩn hóa chuỗi trước khi chunk/embed/search.
- `conferenceEntityDictionary.service.ts`: từ điển/nghiệp vụ domain (entity enrichment/normalization).
- `test/chunks-qdrant-search.ts`: script/kiểm thử hiện có.

Mục tiêu của bản cập nhật này: đánh giá vị trí triển khai (parent vs. child worker) cho từng service, xác định cách tích hợp (proxy + RPC), và liệt kê các thay đổi tối thiểu cần để di chuyển từng phần.

Đề xuất cấu hình (cập nhật: mọi service chạy chung trong một child process)

Quyết định: tất cả các service RAG trong `src/chatbot/services/rag` (gồm `embeddingService`, `vectorStoreService.qdrant`, `retrievalService`, `chunkingService`, `conferenceEntityDictionary`, ...) sẽ chạy chung trong một child process duy nhất. Lý do:

- Giảm phức tạp IPC (một endpoint duy nhất cho RAG).
- Dễ dàng chia sẻ cache/buffering nội bộ giữa các module (nếu cần).
- Giữ mô hình Embedding và VectorStore adapter trong cùng tiến trình để giảm chi phí khi co-locate.

Kiến trúc mới (tóm tắt):

- Single RAG worker: `src/workers/conferenceSync.childProcess.ts` (mở rộng hoặc tái sử dụng file hiện có) sẽ khởi tạo và vận hành:
  - `EmbeddingService` (model load, embed API),
  - `RetrievalService` (orchestration layer — điều phối luồng chunk/embed/search),
  - `VectorStore` adapter (Qdrant client, upsert/search API),
  - `ChunkingService` và `ConferenceEntityDictionary` (tuỳ cấu hình),
  - Các proxy/module export để parent sử dụng (xem phần Proxies).
- Phía parent chỉ gọi các proxy (mà thực chất gửi RPC tới single child process) để thực hiện các thao tác RAG.

Giao thức RPC (chi tiết)

- Loại tin nhắn: `RUN` (id, method, params), `FINAL_RESULT` (id, result), `ERROR` (id, error), `WORKER_READY`, `PREPARE_DRAIN`, `HEALTH`.
- Các method của worker:
  - Embedding worker: `init()`, `embed(texts: string[], options?)`, `health()`.
  - VectorStore worker: `init()`, `upsert(collection, vectors, metadata)`, `search(collection, queries, topK, filter)`, `delete(collection)`, `health()`.
- Loại lỗi domain: dùng lỗi theo từng dịch vụ, ví dụ `RAG_EMBEDDING_ERROR`, `RAG_VECTORSTORE_ERROR`, `RAG_RETRIEVAL_ERROR`, `RAG_CHUNKING_ERROR`, `RAG_CONFERENCE_DICTIONARY_ERROR`.

Kế hoạch tích hợp theo service (chi tiết bước theo bước)

1. Phân tích và chuẩn bị mã nguồn (refactor nhẹ)
   - Mở `embeddingService.ts` để xác định `init()` và API công khai (batch size, options).
   - Mở `vectorStoreService.qdrant.ts` để xác định các method dùng (`upsert`, `search`) và cấu hình hiện có.
   - Thêm các adapter nhỏ nếu thiếu để cung cấp chữ ký RPC đơn giản.

2. Thiết kế và triển khai worker đơn (Single RAG worker)
   - Dùng một file worker duy nhất: `src/workers/rag.childProcess.ts` (mở rộng file hiện có hoặc tạo mới theo convention).
   - Trong file này triển khai các thành phần sau:
     - Khởi tạo `EmbeddingService` và cung cấp API `init`, `embed(texts, opts)`, `health()`.
     - Khởi tạo `VectorStore` (Qdrant client) và cung cấp API `init`, `upsert`, `search`, `delete`, `health()`.
     - Triển khai `ChunkingService` và `ConferenceEntityDictionary` nếu cần để cho phép pre-processing/enrichment nội bộ trước khi embed/index.
     - Triển khai các proxy-class/instance và export chúng từ cùng file để parent có thể import trực tiếp (ví dụ `embeddingProxy`, `vectorstoreProxy`, `chunkingProxy`, `conferenceEntityDictionaryProxy`).
   - Tính năng worker:
     - Emit `WORKER_READY` sau khi init xong.
     - Hỗ trợ `PREPARE_DRAIN` để hoàn tất các tác vụ đang chạy trước khi shutdown.
     - Sử dụng `p-limit` hoặc semaphore để giới hạn concurrency cho các tác vụ nặng; áp cơ chế batching nội bộ theo `EMBED_BATCH_SIZE`.
   - Lợi ích: giảm IPC, cho phép chia sẻ cache/batching trong tiến trình, và đơn giản hóa quản lý tiến trình con.

3. Helpers phía parent & proxies
   - Thêm `WORKER_PATHS.RAG` vào `childProcessManager`.
   - Cài đặt helper `callEmbeddingOnChildProcess(method, params, timeout)`, `callVectorStoreOnChildProcess(...)`,...
   - Quy tắc proxy: mỗi service trong `src/chatbot/services/rag` phải có một proxy tương ứng (ngoại trừ `stringNormalize.ts`), ví dụ: `embeddingProxy`, `vectorstoreProxy`, `chunkingProxy`, `conferenceEntityDictionaryProxy`, v.v. Các proxy chịu trách nhiệm:
     - Chuẩn hóa API phía parent (signature thân thiện với RPC).
     - Gọi helper `call*OnChildProcess` để thực thi trên worker.
     - Map lỗi domain từ worker về lỗi có ý nghĩa phía parent.
   - Yêu cầu triển khai cụ thể: các proxy sẽ được triển khai (đóng gói) cùng file với `rag.childProcess.ts` — tức là export thêm các lớp/proxy từ `src/workers/rag.childProcess.ts` để giữ logic worker/proxy tập trung với nhau.
   - Bind các proxy bằng `tsyringe` và cập nhật `retrievalService` để sử dụng proxy thay vì gọi trực tiếp service.

4. Luồng retrieval & chunking
   - Không thay đổi logic bên trong `retrievalService`; chỉ thay instance hiện tại bằng `retrievalProxy` (proxy này forward các method cũ sang single RAG worker). Luồng xử lý giữ nguyên: `stringNormalize` -> `chunkingService` -> `embeddingProxy.embed` -> `vectorstoreProxy.search/upsert`.
   - `conferenceEntityDictionary` chạy trong child process (được khởi tạo trong single RAG worker); parent gọi qua `conferenceEntityDictionaryProxy` khi cần enrichment.

5. Khởi động server & readiness
   - Khởi động single RAG worker sớm trong `server.ts` (ví dụ: `rag` worker), `await worker.ready` trước khi thực hiện các RPC.
   - Thêm flag môi trường để tuỳ chỉnh cấu hình chạy (ví dụ: `RAG_COLOCATE_VECTORSTORE=true` để cấu hình nội bộ của worker nếu cần).

6. Kiểm thử & đo lường
   - Thêm `scripts/bench/embed_memory_check.ts` để khởi tạo `EmbeddingService` và in RSS sau khi `init()`.
   - Thêm smoke test `tests/smoke/rag-smoke.spec.ts` chạy luồng embed -> upsert -> search với Qdrant dev.

7. Quan sát và độ tin cậy
   - `health()` của worker trả các chỉ số cơ bản (RSS, `heapUsed`, số inflight request).
   - Parent implement cơ chế restart/backoff và expose health của worker trong `/health` chính.

Các file/code cần tạo/sửa (cụ thể)

- `src/workers/rag.childProcess.ts` (sửa/mở rộng): triển khai Single RAG worker và export các proxy (embeddingProxy, vectorstoreProxy, chunkingProxy, conferenceEntityDictionaryProxy, ...).
- Rename: đổi tên file worker thành `src/workers/rag.childProcess.ts` (thực hiện rename khi scaffold xong) và cập nhật tất cả `WORKER_PATHS`/tham chiếu trong `childProcessManager` và `server.ts`.
- `src/workers/childProcessManager.ts` (sửa): thêm `WORKER_PATHS.RAG`/`WORKER_PATHS.CONFERENCE_SYNC` và helper gọi RPC tới single worker; cập nhật `ensureReady()` logic nếu cần.
- `src/chatbot/services/retrievalService.ts` (sửa): đổi để gọi các proxy export từ `rag.childProcess.ts`.
- `scripts/bench/embed_memory_check.ts` (mới): script đo RSS khi khởi tạo `EmbeddingService` (dùng để quyết định tuning).
- `tests/smoke/rag-smoke.spec.ts` (mới): smoke test chạy luồng embed -> upsert -> search với Qdrant dev.

Kế hoạch di chuyển thực tế (bước an toàn)

1. Bổ sung các proxy exports vào `src/workers/rag.childProcess.ts` (ví dụ `embeddingProxy`, `vectorstoreProxy`, `chunkingProxy`, `conferenceEntityDictionaryProxy`) mà không thay đổi ngay `retrievalService` — cho phép parent thử gọi proxy mới song song.
2. Chạy script đo bộ nhớ (`scripts/bench/embed_memory_check.ts`) để xác nhận RSS khi load mô hình và điều chỉnh `EMBED_BATCH_SIZE`/`WORKER_CONCURRENCY`.
3. Cập nhật `retrievalService` để chuyển dần các lời gọi embedding/upsert/search sang các proxy (proxy chuyển tiếp RPC tới single worker).
4. Thêm smoke tests và monitoring; triển khai dần bằng feature flag.

Các điểm quyết định và giá trị mặc định khuyến nghị

- Mặc định: chạy toàn bộ RAG services trong một child process duy nhất (`rag` worker). Giữ mọi proxy và logic worker tập trung để đơn giản hóa vận hành.
- Kích thước batch khởi điểm: `EMBED_BATCH_SIZE=32` (tối ưu theo latency và bộ nhớ).
- Số concurrency khuyên dùng: `WORKER_CONCURRENCY=2-4` cho các tác vụ embedding/processing trong worker, tuỳ CPU và mô hình.

Hành động tiếp theo tôi sẽ làm nếu bạn đồng ý

1. Tạo `scripts/bench/embed_memory_check.ts` để đo RSS của `EmbeddingService` (rủi ro thấp, nhanh). HOẶC
2. Scaffold / mở rộng `src/workers/rag.childProcess.ts` để chứa toàn bộ RAG worker và export các proxy (nhiều công hơn nhưng tiến tới di chuyển ngay).

---

File cập nhật tại: chatbot_enhancement/rag_childprocess_plan.md
