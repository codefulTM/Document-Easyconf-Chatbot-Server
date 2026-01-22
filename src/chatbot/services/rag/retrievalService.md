
# RetrievalService

Tài liệu cho `RetrievalService` — module chịu trách nhiệm tìm kiếm và trả về các "chunk" (đoạn nội dung) liên quan tới một truy vấn bằng cách dùng tìm kiếm vector, tìm kiếm từ khóa, hoặc kết hợp cả hai (hybrid). File nguồn: `retrievalService.ts`.

**Tổng quan**
- **Mục đích:** cung cấp API để lấy các đoạn nội dung liên quan (Chunk) dùng cho RAG (Retrieval-Augmented Generation) với các tuỳ chọn lọc, reranking (MMR), và format đặc biệt cho dữ liệu conference.
- **Phụ thuộc chính:** `VectorStoreService` (Qdrant/chroma wrapper), `EmbeddingService`, `ConferenceSyncService`.

**Interfaces xuất khẩu**
- `RetrievalOptions` — các tuỳ chọn khi gọi `retrieve`:
	- `limit?: number` — số kết quả tối đa (mặc định 5).
	- `threshold?: number` — ngưỡng điểm (điểm distance/score phụ thuộc implementation vector store).
	- `mmrLambda?: number` — hệ số cho MMR (0..1), mặc định 0.7.
	- `filter?: { tableName?: string; source?: string; [key: string]: any }` — lọc metadata của chunk.
	- `returnFormatted?: boolean` — nếu true trả nội dung đã format (sử dụng `formatSingleChunk`).
	- `useHybrid?: boolean` — bật hybrid retrieval (vector + keyword).
	- `keywordWeight?: number` — trọng số cho keyword khi hybrid (0..1).
	- `vectorWeight?: number` — trọng số cho vector khi hybrid (0..1).

- `FormattedChunkResult` — kết quả với nội dung đã format:
	- `originalChunk: Chunk` — chunk gốc.
	- `formattedContent: string` — nội dung sau format.
	- `score?: number` — điểm (nếu có).

**Lớp chính: `RetrievalService`**
- `constructor()` — khởi tạo các service phụ trợ.
- `async init()` — khởi tạo bất kỳ resource cần thiết; idempotent (gọi nhiều lần an toàn).
- `isInitialized(): boolean` — kiểm tra đã gọi `init()` chưa.

- `async retrieve(query: string, options?: RetrievalOptions): Promise<Chunk[]>` — API chính để lấy chunks.
	- Mặc định sử dụng vector retrieval với MMR.
	- Nếu `useHybrid` = true, chuyển sang `hybridRetrieve` (kết hợp vector + keyword).

- `async retrieveFormatted(query: string, options?: RetrievalOptions): Promise<FormattedChunkResult[]>` — lấy kết quả và trả cả nội dung đã format cho từng chunk.

Private methods (tóm tắt hành vi):
- `hybridRetrieve(query, {limit, threshold, filter, keywordWeight, vectorWeight})` —
	1. Lấy kết quả vector (gấp đôi `limit`).
	2. Lấy kết quả từ khóa (keyword search).
	3. Gộp + tính score kết hợp theo `vectorWeight`/`keywordWeight`.
	4. Lọc theo `threshold` và `filter` metadata (so sánh không phân biệt hoa thường).
	5. Format và trả `limit` chunk hàng đầu (lưu `score` vào `metadata.score`).

- `vectorRetrieve(query, {limit, threshold, mmrLambda, filter})` —
	1. Lấy nhiều candidate từ vectorStore (fetchLimit = 4 * limit).
	2. Lọc bằng threshold và metadata filter.
	3. Áp dụng MMR (nếu đủ candidate) bằng `maximalMarginalRelevance`.
	4. Trả danh sách `Chunk[]` đã chọn.

- `formatSingleChunk(chunk)` — nếu `chunk.metadata.tableName === 'conferences'` và có `recordId`, lấy thông tin hội nghị đầy đủ từ `ConferenceSyncService` và format bằng `formatConference()`, ngược lại trả `chunk.content` gốc.

- `formatChunks(chunks)` — format danh sách chunk (gọi `formatSingleChunk` tuần tự).

- `maximalMarginalRelevance(query, candidates, lambda, k)` —
	- Cài đặt MMR: cần embedding của query và các candidate; hiện implementation tái-embedding các candidate bằng `EmbeddingService` (ghi chú: tốn chi phí; trong production nên lưu vectors để tránh re-embed).
	- Trả danh sách candidate tối đa `k` sao cho cân bằng relevance và diversity.

- `cosineSimilarity(vecA, vecB)` — hàm tiện ích tính cosine similarity.

- `keywordSearch(query, limit)` —
	- Trích keywords bằng `extractKeywords`.
	- Nếu store hỗ trợ text search, gọi `vectorStore.textSearch`, nếu không dùng `scrollAndFilter` để quét batch và tính điểm match từ khóa.

- `searchKeywordsInDatabase(keywords, limit)` — wrapper, chọn chiến lược textSearch hoặc scrollAndFilter.

- `scrollAndFilter(keywords, limit)` — quét theo batch (batchSize = 100) từ vector store, tính điểm match tỉ lệ trên số keywords, trả top matches.

- `extractKeywords(query)` — triển khai đơn giản: loại bỏ stopwords, ký tự đặc biệt, lấy tối đa 10 từ dài >2.

- `combineAndRank(vectorResults, keywordResults, vectorWeight, keywordWeight)` — gộp hai nguồn theo chunk id, tính score kết hợp: vectorScore*vectorWeight + keywordScore*keywordWeight, sắp theo giảm dần.

- `getChunkId(chunk)` — tạo id tạm cho chunk (dùng tableName + slice nội dung), để map khi gộp kết quả.

Hành vi logging / debug
- Module có nhiều log console để debug: báo số kết quả, bước lọc, mismatch metadata, sample scores, warnings khi scroll vượt ngưỡng.

Lưu ý quan trọng & khuyến nghị
- MMR hiện tại re-embed các candidate để tính diversity — có thể tốn chi phí. Tốt nhất là cập nhật `VectorStoreService` để trả kèm vectors hoặc giữ cache vectors. (Trong code có chú thích các option A/B/C).
- `threshold` usage: phụ thuộc store (một số store trả distance nhỏ = tốt hơn); kiểm tra kỹ semantics của `vectorStore.search()` trong dự án.
- Metadata filter: hybrid retrieval áp dụng so sánh case-insensitive; vectorRetrieve so sánh strict equality — chú ý điểm khác biệt.
- `getChunkId` dùng slice nội dung để nhận dạng — không hoàn hảo; nếu có `id` duy nhất trong metadata nên dùng thay thế.

Ví dụ sử dụng (TypeScript)

```ts
const svc = new RetrievalService();
await svc.init();

// Vector-only (mặc định, có MMR)
const chunks = await svc.retrieve('How to submit a paper', { limit: 5 });

// Hybrid (vector + keyword), trả formatted content
const formatted = await svc.retrieveFormatted('important conference deadline', {
	useHybrid: true,
	limit: 3,
	vectorWeight: 0.7,
	keywordWeight: 0.3,
});
```

Vị trí file nguồn
- Xem mã nguồn tại: `retrievalService.ts` (trong repo) để biết chi tiết implementation và các console.log debug.

Các bước tiếp theo gợi ý
- Nếu cần tối ưu: cập nhật `VectorStoreService` để trả vectors hoặc cache vectors, sửa `getChunkId` để dùng `metadata.recordId` nếu có.

---
Tài liệu này cung cấp tóm tắt API, hành vi và các lưu ý vận hành cho `RetrievalService`.

