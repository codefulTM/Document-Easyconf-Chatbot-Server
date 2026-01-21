<!-- filepath: d:\LearningMaterial\KHTN\DoAnTotNghiep\Code\Document-Easyconf-Chatbot-Server\src\chatbot\services\rag\vectorStoreService.qdrant.md -->

# VectorStoreService (Qdrant)

Tài liệu chi tiết cho `VectorStoreService` (định nghĩa ở `vectorStoreService.qdrant.ts`). Service này quản lý việc sinh embedding (bằng `@xenova/transformers`), lưu embedding vào Qdrant, và truy vấn tìm kiếm ngữ nghĩa.

## Tổng quan

- Kết nối với Qdrant REST (`@qdrant/js-client-rest`).
- Sinh embedding qua Xenova Transformers `feature-extraction` pipeline (mặc định model `Xenova/all-MiniLM-L6-v2`).
- Lưu trữ vectors trong collection `conference_vectors` với khoảng cách `Cosine` và kích thước vector (`dimension`) là 384.
- Hỗ trợ thêm document (chunk), tìm kiếm bằng text (sinh embedding query) và duyệt/scroll collection.

## Biến môi trường liên quan

- `QDRANT_URL` — URL Qdrant (mặc định `http://localhost:6333`).
- `QDRANT_API_KEY` — API key cho Qdrant (nếu có).
- `TRANSFORMERS_MODEL` — (tùy chọn) model Xenova để dùng thay model mặc định.
- `TRANSFORMERS_CACHE_DIR` — (tùy chọn) thư mục cache/local models cho Xenova.
## Kiến trúc chính

- `VectorStoreService` là một class với các phương thức công khai chính: `init()`, `addDocuments()`, `search()`, `scrollBatch()`, `getStats()`, `clear()`, `isReady()`, `deleteDocuments()`, `close()`.
- Pipeline tạo embedding được quản lý bởi `EmbeddingService` (singleton); `VectorStoreService` gọi API batch của `EmbeddingService` để sinh vectors.
- Một pipeline embedding được lưu trong `this.embeddingPipeline`.

## Các phương thức chi tiết
### constructor()

- Khởi tạo `QdrantClient` dựa trên `QDRANT_URL` và `QDRANT_API_KEY`.
- Không còn khởi tạo pipeline embedding tại đây; `EmbeddingService` chịu trách nhiệm quản lý pipeline. Nếu cần đảm bảo pipeline đã sẵn sàng, gọi `EmbeddingService.getInstance().init()` trước khi thêm hoặc tìm kiếm vectors.
- Gọi `initializeEmbeddingPipeline()` (khởi tạo pipeline bất đồng bộ; constructor không await nó — lưu ý: pipeline có thể chưa sẵn sàng ngay sau khi instance được tạo).
### initializeEmbeddingPipeline(): Promise<void>

- (Đã loại bỏ) Việc khởi tạo pipeline embedding được chuyển hẳn sang `EmbeddingService`. Các cấu hình liên quan tới `TRANSFORMERS_MODEL` và `TRANSFORMERS_CACHE_DIR` giờ được xử lý trong `EmbeddingService`.
- Nếu thất bại, ném lỗi và log ra console.

### generateEmbeddings(texts: string[]): Promise<number[][]>

- Mục đích: chuyển mảng chuỗi `texts` thành mảng vector embedding.
- Hành vi hiện tại: `VectorStoreService` ủy quyền việc sinh embedding cho `EmbeddingService` bằng cách gọi `EmbeddingService.getInstance().generateEmbeddingsBatch(texts)`; `EmbeddingService` chịu trách nhiệm khởi tạo pipeline, áp pooling và trả về `number[][]`.

- Lưu ý: nếu cần thay đổi pooling/chuẩn hóa, hãy chỉnh trong `EmbeddingService` để thay đổi có hiệu lực cho toàn hệ thống.
- Nếu định dạng tensor không như mong đợi hoặc có lỗi: log cảnh báo/lỗi và trả về các zero vectors để hệ thống không bị crash.

Ghi chú kỹ thuật: code sử dụng indexing tuyến tính để trích giá trị từ `Float32Array` theo công thức
`index = batch_idx * seq_len * embed_dim + seq_idx * embed_dim + embed_dim_idx`, sau đó tính trung bình.
### init(): Promise<void>

- Mục đích: kiểm tra kết nối Qdrant, đảm bảo `EmbeddingService` đã khởi tạo pipeline (nếu cần), và tạo collection nếu chưa tồn tại.
- Hành vi:
  - Gọi `qdrantClient.getCollections()` để kiểm tra kết nối.
  - Gọi `await EmbeddingService.getInstance().init()` để đảm bảo pipeline embedding sẵn sàng trước khi thêm hoặc tìm kiếm vectors.
  - Nếu collection `conference_vectors` chưa tồn tại, tạo với cấu hình vectors `{ size: this.dimension, distance: 'Cosine' }`.
  - Nếu tồn tại, in ra số lượng points hiện có.

### addDocuments(chunks: Chunk[], embeddings?: number[][]): Promise<void>

- Mục đích: thêm một loạt `chunks` (đối tượng `Chunk`) vào Qdrant.
- Hành vi:
  - Lấy text từ `chunk.content` và gọi `generateEmbeddings(texts)` để sinh embedding (hiện tại luôn sinh, tham số `embeddings` chưa được dùng).
  - Sinh `id` cho mỗi point bằng `uuidv5(
    `${recordIdStr}:${chunkIndex}`,
    recordIdStr
  )` — tức dùng `recordId` làm namespace để tạo UUIDv5. (Lưu ý: nếu `recordId` trống, dùng string "unknown".)
  - Tạo payload(metadata gắn kèm với mỗi vector) gồm `content`, `source`, `chunkIndex`, `timestamp`, `recordId`, `tableName`.
  - Upsert vào Qdrant theo batch (mỗi batch size = 100).

Lưu ý: nếu muốn dùng embedding đã có sẵn (được truyền vào `embeddings`), hiện tại mã không sử dụng tham số đó — có thể cải thiện để chấp nhận embedding sẵn.

### search(queryText: string, k = 3): Promise<VectorSearchResult[]>

- Mục đích: tìm các chunk tương tự với `queryText`.
- Hành vi:
  - Gọi `generateEmbeddings([queryText])` để lấy vector truy vấn.
  - Gọi `qdrantClient.search()` với `vector` và `limit: k`, `with_payload: true`.
  - Trả về danh sách `VectorSearchResult` gồm `chunk` và `score`.

### searchByVector(queryVector: number[], k = 3): Promise<VectorSearchResult[]>

- Hiện tại hàm này gọi `this.search('', k)` — đây chỉ là placeholder/bug: nó không sử dụng `queryVector`. Nếu bạn cần tìm theo vector đã có, sửa thành gọi `qdrantClient.search()` trực tiếp với `vector: queryVector`.

### scrollBatch(offset: number, limit: number): Promise<Chunk[]>

- Sử dụng API `scroll` của Qdrant để lấy points không kèm vector, phù hợp cho các tác vụ keyword hoặc export. `scroll` là thao tác duyệt một phần các điểm trong collection của Qdrant theo từng lô.

### hasTextSearchCapability(), textSearch(...) (placeholder)

- `hasTextSearchCapability()` trả `false` (không hỗ trợ text search nội bộ).
- `textSearch()` hiện là placeholder và trả empty; code log một cảnh báo.

### getStats(), clear(), isReady(), deleteDocuments(pointIds: string[]), close()

- Các hàm tiện ích để lấy thống kê collection, xóa collection (clear), kiểm tra readiness, xóa document theo id, và đóng kết nối (no-op cho Qdrant client REST).

## Quy tắc & giả định

- `dimension` (384) phải khớp với model embedding bạn dùng. Nếu đổi model, cập nhật `dimension` tương ứng.
- UUIDv5 được sinh dựa trên `recordId` và `chunkIndex` — đảm bảo `recordId` là giá trị nhất quán nếu muốn tránh duplicate.
- Upsert theo batch (100 points) để tránh request quá lớn.
## Vấn đề cần lưu ý / Đề xuất cải tiến

- `searchByVector` cần sửa để chấp nhận vector trực tiếp.
- `addDocuments` hiện bỏ qua tham số `embeddings` nếu caller đã tính sẵn vectors — thêm logic dùng `embeddings` nếu được truyền vào sẽ tiết kiệm thời gian.
- Pipeline embedding đã được tách ra sang `EmbeddingService`; đảm bảo cập nhật cấu hình `TRANSFORMERS_MODEL`/`TRANSFORMERS_CACHE_DIR` trong `EmbeddingService` khi đổi model.
- Xử lý lỗi: hiện tại một số tình huống fallback trả zero-vectors; có thể thay bằng retry/backoff hoặc báo lỗi rõ ràng tùy yêu cầu.
- Nếu cần throughput cao, cân nhắc batch hóa lớn hơn, đồng thời kiểm tra bộ nhớ và thời gian xử lý model Xenova.
- Nếu cần throughput cao, cân nhắc batch hóa lớn hơn, đồng thời kiểm tra bộ nhớ và thời gian xử lý model Xenova.

## Ví dụ sử dụng

```ts
import { VectorStoreService } from './vectorStoreService.qdrant';
async function run() {
  const svc = new VectorStoreService();
  await svc.init(); // kiểm tra Qdrant và đảm bảo EmbeddingService đã init

  // Thêm docs (giả sử chunks là mảng Chunk từ chunking service)
  await svc.addDocuments(chunks);

  // Tìm kiếm
  const hits = await svc.search('Hội nghị về AI', 5);
  console.log(hits);
}

run().catch(console.error);
run().catch(console.error);
```

## Vận hành

- Đảm bảo Qdrant đang chạy và `QDRANT_URL` trỏ đúng địa chỉ.
- Tùy môi trường (node, browser + WASM), Xenova Transformers có thể cần cấu hình runtime; kiểm tra `@xenova/transformers` docs.
- Giữ `dimension` khớp với model embedding.

---

File này mô tả hành vi hiện tại của `vectorStoreService.qdrant.ts`, những giả định, và các gợi ý để cải thiện độ ổn định/hiệu năng.
