# RAG Refactor Plan

## Mục tiêu

Refactor hệ thống RAG của `Easyconf-Chatbot-Server` theo các yêu cầu:

- Dùng **LangChain** làm orchestration layer cho RAG.
- Dùng **local embedding model** nhẹ, hỗ trợ đa ngôn ngữ.
- Dữ liệu RAG lấy từ **các cột trong các bảng PostgreSQL** dựa trên schema của `Easyconf-BE`.
- Dùng **CDC mức thấp** qua `pg-logical-replication` để tự động đồng bộ dữ liệu.
- Chỉ gửi **ACK sau khi event đã được xử lý xong**.
- Hỗ trợ xử lý bất đồng bộ nhưng **không được ghi đè sai thứ tự** khi event đến out-of-order.

---

## 1. Chốt phạm vi dữ liệu RAG

1. Rà soát `Code/Easyconf-BE/prisma/schema.prisma` để chốt các bảng/cột sẽ đưa vào RAG.
2. Phân loại dữ liệu theo mức độ hữu ích cho retrieval:
   - Nhóm conference: `Conferences`, `ConferenceOrganizations`, `ConferenceDates`, `ConferenceTopics`, `ConferenceRanks`, `FieldOfResearchs`, `Ranks`, `Sources`.
   - Nhóm user-facing metadata: `Locations`, `Topics`, `Journals`, `JournalDetails`, `JournalAreas`, `JournalStatistics`, `JournalQuartiles`, `JournalTopics`, `JournalBioxBio`.
   - Nhóm trạng thái/interaction: các bảng like/follow/feedback/notification chỉ đưa vào nếu thật sự cần semantic retrieval.
3. Chốt mapping mỗi bảng thành một document RAG có cấu trúc rõ ràng: `entity_type`, `entity_id`, `version`, `updated_at`, `content`, `metadata`.

Deliverable: danh sách bảng/cột được phép index và format document chuẩn hóa.

Khảo sát data: Xem trong [data](./data%20được%20support%20cho%20RAG.md)

Note: Trong file khảo sát:
- Các bảng không được include sẽ không nằm trong phạm vi RAG. 
- Các bảng được include: 
   + Các cột được đánh số 1 sẽ chỉ áp dụng exact search.
   + Các cột được đánh số 2 sẽ chỉ áp dụng fuzzy search.
   + Các cột được đánh số 3 sẽ áp dụng fuzzy search và vector search(pg_trgm + pgvector)

---

## 2. Thiết kế kiến trúc RAG mới

1. Tách hệ thống thành 4 khối:
   - `Source Reader`: đọc dữ liệu từ Postgres.
   - `Document Builder`: chuyển row thành text document.
   - `Embedding + Vector Store`: embed và lưu vector.
   - `Retriever/QA`: LangChain chain để truy vấn và sinh câu trả lời.
2. Giữ raw SQL cho các truy vấn đặc thù như `pg_trgm`, `pgvector`, backfill, và CDC bookkeeping.
3. Định nghĩa chuẩn ID cho document để đảm bảo idempotency:
   - `doc_id = <schema>.<table>:<primary_key>`
   - nếu cần version hóa: `doc_version = <updated_at or LSN>`.

Deliverable: sơ đồ luồng dữ liệu và chuẩn định danh document.

---

## 3. Chọn embedding model local

1. Ưu tiên model local nhẹ, đa ngôn ngữ, chạy tốt trên CPU:
   - `intfloat/multilingual-e5-small` nếu cần chất lượng tốt và đa ngôn ngữ.
   - `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` nếu ưu tiên nhẹ hơn.
2. Chọn runtime embedding local phù hợp:
   - `transformers`/`@xenova/transformers` nếu muốn chạy trong Node.js.
   - hoặc tách sang service embedding riêng nếu cần ổn định hơn.
3. Chuẩn hóa input/output embedding:
   - prefix query/document theo guideline của model nếu model yêu cầu.
   - cố định dimension, metric similarity, và batch size.

Deliverable: model được chọn, benchmark sơ bộ, và quy tắc embed thống nhất.

---

## 4. Thiết kế storage cho vector index

1. Dùng PostgreSQL + `pgvector` làm vector store nếu muốn đồng bộ hạ tầng.
2. Tạo bảng/vector schema riêng cho RAG, ví dụ:
   - `Chunks`
   - `Embeddings`
   - `rag_document_versions`
   - `rag_change_log`
3. Lưu tối thiểu:
   - `Chunks.id`
   - `Chunks.tableName`
   - `Chunks.recordId`
   - `Chunks.fieldName`
   - `Chunks.contentHash`
   - `Chunks.embeddingId`
   - `Embeddings.id`
   - `Embeddings.embedding`
   - `source_lsn`
   - `source_updated_at`
   - `status`
4. Tạo unique constraint theo khóa logic của chunk để upsert an toàn:
   - `tableName + recordId + fieldName + contentHash`
5. Với trường hợp dedupe theo content giữa nhiều record, cho phép nhiều dòng `Chunks` cùng `tableName + fieldName + contentHash` nhưng khác `recordId` và `id`.

Deliverable: schema vector store và chiến lược upsert.

---

## 5. Thiết kế pipeline build document từ DB

1. Với mỗi bảng nguồn, tạo `Document Mapper` riêng.
2. Mỗi mapper cần trả về:
   - text tự nhiên, giàu ngữ nghĩa.
   - metadata đủ để trace ngược về row gốc.
3. Không nhồi toàn bộ row thô vào một document nếu row quá lớn.
4. Khi chunking value của một field trong một bảng, áp dụng pipeline sau:
   - chunk value thành nhiều chunk nhỏ trước khi embed.
   - với mỗi chunk, tính `contentHash`.
   - nếu tồn tại bản ghi trong `Chunks` có cùng `tableName`, `recordId`, `fieldName`, `contentHash` thì skip chunk đó.
   - nếu chunk có cùng `tableName`, `fieldName`, `contentHash` nhưng khác `recordId`, chỉ tạo thêm một dòng mới trong `Chunks` với `recordId` và `id` khác, còn lại giữ nguyên.
   - chunk nào không bị skip thì embed bằng local embedding model, lưu vector vào `Embeddings`, rồi lưu `embeddingId` tương ứng vào `Chunks`.
   - sau khi xử lý xong field hiện tại, các chunk cũ cùng `tableName` + `recordId` + `fieldName` nhưng không còn xuất hiện ở lần chunk mới và cũng không được tạo mới trong đợt này thì xóa khỏi `Chunks`.
5. Với bảng nhiều quan hệ, tạo document theo 2 lớp:
   - document cấp entity chính.
   - document phụ cho relation quan trọng nếu cần.
6. Định nghĩa versioning rule:
   - nếu row có `updatedAt`, dùng làm version hint.
   - CDC `LSN` là nguồn sự thật cuối cùng cho ordering.

Deliverable: danh sách mapper cho từng bảng quan trọng.

---

## 6. Tích hợp LangChain

1. Dùng LangChain để chuẩn hóa các khối:
   - loader
   - splitter nếu cần
   - retriever
   - chain QA
2. Thiết kế chain theo hướng:
   - user query -> normalize -> retrieve top-k -> rerank nếu cần -> answer generation.
3. Nếu có hybrid search, kết hợp:
   - vector similarity
   - keyword/`pg_trgm`
   - metadata filter.
4. Tách prompt template theo domain conference/journal.

Deliverable: RAG chain chạy được với LangChain và có thể thay retriever độc lập.

---

## 7. Thiết kế CDC mức thấp với `pg-logical-replication`

1. Bật logical replication cho DB nguồn.
2. Tạo replication slot và publication cho các bảng cần theo dõi.
3. Viết CDC worker dùng `pg-logical-replication` để đọc WAL changes.
4. Chỉ ACK event khi đã:
   - parse xong,
   - map xong,
   - cập nhật `Chunks` và `Embeddings` xong,
   - ghi checkpoint xong.
5. Nếu xử lý fail giữa chừng, không ACK để event được redeliver.

Deliverable: CDC worker tối thiểu có checkpoint + ack đúng thời điểm.

---

## 8. Chống mất sự kiện và xử lý out-of-order

1. Mỗi event phải có:
   - `lsn`
   - `txid`
   - `schema/table`
   - `pk`
   - `op`
   - `commit_timestamp` nếu available.
2. Tạo **per-entity ordering guard**:
   - mọi update của cùng `doc_id` phải đi qua một hàng đợi/lock riêng.
   - đảm bảo update của một entity được commit theo thứ tự LSN.
3. Áp dụng rule ghi dữ liệu theo “last committed LSN wins”, không theo “task start time”.
4. Nếu xử lý async:
   - cho phép song song giữa các entity khác nhau,
   - nhưng serialize các event cùng `doc_id`.
5. Trước khi write, kiểm tra version hiện tại trong store:
   - nếu event có `lsn` cũ hơn version đã lưu, bỏ qua.
   - nếu event mới hơn, upsert.
6. Khi một event làm thay đổi một field đã được chunk trước đó, worker phải xử lý theo khóa logic của chunk:
   - match theo `tableName + recordId + fieldName + contentHash` để tránh ghi đè sai.
   - chỉ xóa các chunk cũ cùng `tableName + recordId + fieldName` khi chunk đó thực sự không còn thuộc snapshot mới.

Deliverable: cơ chế chống ghi đè sai thứ tự cho cùng một entity.

---

## 9. Chiến lược ACK an toàn

1. ACK chỉ sau khi transaction nội bộ hoàn tất.
2. Nếu lưu nhiều document từ một event, coi đó là một unit of work.
3. Checkpoint phải phản ánh event đã xử lý xong, không phải event đã đọc được.
4. Nếu crash sau khi write nhưng trước ACK:
   - event sẽ được replay,
   - write phải idempotent để không tạo bản ghi lỗi.

Deliverable: semantics at-least-once an toàn, không mất event.

---

## 10. Backfill và bootstrap

1. Trước khi bật CDC realtime, chạy backfill toàn bộ dữ liệu nguồn.
2. Backfill phải ghi cùng một format với CDC event.
3. Sau backfill, ghi checkpoint khởi tạo để CDC chỉ xử lý delta mới.
4. Backfill cần chunk/batch để tránh load lớn.

Deliverable: script bootstrap index từ DB hiện có.

---

## 11. Observability và vận hành

1. Ghi log theo `doc_id`, `lsn`, `txid`, `event_type`, `duration_ms`.
2. Thêm metrics:
   - event lag
   - processing latency
   - retry count
   - dropped stale event count
   - vector upsert success/fail
3. Có dead-letter path cho event lỗi parse/hệ thống.
4. Có cơ chế replay từ checkpoint khi cần rebuild index.

Deliverable: dashboard/metrics tối thiểu cho CDC và RAG pipeline.

---

## 12. Kế hoạch triển khai theo phase

### Phase 1: Nền tảng
- Chọn embedding model local.
- Dựng schema vector store.
- Tạo document mapper cho vài bảng chính.

### Phase 2: LangChain RAG
- Thay retrieval layer hiện tại bằng LangChain chain.
- Tích hợp vector store và prompt template.

### Phase 3: Backfill
- Build script nạp toàn bộ data từ Postgres.
- Đảm bảo idempotent upsert.

### Phase 4: CDC
- Bật logical replication.
- Viết CDC worker với ack sau xử lý.
- Thêm checkpoint store.

### Phase 5: Ordering + resilience
- Áp per-entity serialization.
- Chặn stale write theo LSN/version.
- Thêm retry/DLQ/metrics.

### Phase 6: Tối ưu
- Benchmark embedding/retrieval.
- Tuning top-k, batch size, chunking, similarity threshold.

---

## 13. Tiêu chí hoàn thành

- RAG chạy bằng LangChain.
- Embedding local đa ngôn ngữ hoạt động ổn định.
- Dữ liệu RAG lấy từ các bảng PostgreSQL đã chốt.
- CDC cập nhật tự động qua `pg-logical-replication`.
- ACK chỉ sau khi xử lý xong event.
- Không xảy ra stale overwrite khi event đến out-of-order.
