# RAG Refactor Plan

## Mục tiêu

Refactor hệ thống RAG của `Easyconf-Chatbot-Server` theo các yêu cầu:

- Không dùng LangChain làm orchestration layer.
- Tái sử dụng **`EmbeddingService`** hiện có để sinh embedding local.
- Refactor **`chunkingService.ts`** để dùng `RecursiveCharacterTextSplitter` của LangChain, nhưng chỉ chunk khi input >= 3000 ký tự.
- Refactor **`retrievalService.ts`** sang **Drizzle ORM** để thực hiện hybrid search trong **một query duy nhất** bằng query builder.
- Dùng **local embedding model** nhẹ, hỗ trợ đa ngôn ngữ.
- Dữ liệu RAG lấy từ **các cột trong các bảng PostgreSQL** dựa trên schema của `Easyconf-BE`.
- Dùng **CDC mức thấp** qua `pg-logical-replication` để tự động đồng bộ dữ liệu.
- Chỉ gửi **ACK sau khi event đã được xử lý xong**.
- Hỗ trợ xử lý bất đồng bộ nhưng **không được ghi đè sai thứ tự** khi event đến out-of-order.

---

## 1. Chốt phạm vi dữ liệu RAG

1. Rà soát `Code/Easyconf-BE/prisma/schema.prisma` để chốt các bảng/cột sẽ đưa vào RAG.
2. Phân loại dữ liệu theo mức độ hữu ích cho retrieval.

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
   - `Source Reader`: đọc dữ liệu từ Postgres theo snapshot/backfill hoặc CDC event.
   - `Document Builder`: chuyển từng field thành text document trước khi chunk.
   - `Chunk + Embedding Writer`: chunk theo rule mới, sinh embedding bằng `EmbeddingService`, rồi lưu vào `Chunks`/`Embeddings`.
   - `Hybrid Retrieval Service`: dùng Drizzle ORM để dựng một query duy nhất cho hybrid search, filter, group, order, paginate.
2. Giữ raw SQL chỉ cho các phần bắt buộc như `pg_trgm`, `pgvector`, CDC bookkeeping, alias mapping, và các thao tác không biểu diễn gọn bằng query builder.
3. Định nghĩa chuẩn ID cho document để đảm bảo idempotency:
   - `Chunks.id`
   - `Chunks.tableName`
   - `Chunks.recordId`
   - `Chunks.fieldName`
   - `Chunks.contentHash`
   - `Chunks.embeddingId`
   - `Embeddings.id`
   - `Embeddings.embedding`
   - nếu cần version hóa: `doc_version = <LSN>`.
4. Định nghĩa rõ ranh giới giữa các luồng:
   - backfill đọc snapshot cũ, xử lý ở tầng code rồi ghi đích
   - CDC đọc WAL event và chỉ apply sau checkpoint khởi tạo
   - retrieval đọc từ dữ liệu đã index, không đi qua backfill/CDC trực tiếp

Deliverable: sơ đồ luồng dữ liệu và chuẩn định danh document.

---

## 3. Chọn embedding model local
- Giữ nguyên model hiện tại.

---

## 4. Thiết kế storage cho vector index

1. Dùng PostgreSQL + `pgvector` làm vector store nếu muốn đồng bộ hạ tầng.
2. Tạo bảng/vector schema riêng cho RAG, ví dụ:
   - `Chunks`
   - `Embeddings`
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

1. Khi chunking value của một field trong một bảng, áp dụng pipeline sau:
   - chunk value thành nhiều chunk nhỏ trước khi embed.
   - với mỗi chunk, tính `contentHash`.
   - nếu tồn tại bản ghi trong `Chunks` có cùng `tableName`, `recordId`, `fieldName`, `contentHash` thì skip chunk đó.
   - nếu chunk có cùng `tableName`, `fieldName`, `contentHash` nhưng khác `recordId`, chỉ tạo thêm một dòng mới trong `Chunks` với `recordId` và `id` khác, còn lại giữ nguyên.
   - chunk nào không bị skip thì embed bằng local embedding model, lưu vector vào `Embeddings`, rồi lưu `embeddingId` tương ứng vào `Chunks`.
   - sau khi xử lý xong field hiện tại, các chunk cũ cùng `tableName` + `recordId` + `fieldName` nhưng không còn xuất hiện ở lần chunk mới, cũng không được tạo mới trong đợt này và cũng không nằm trong đống chunk có sẵn bị skip thì xóa khỏi `Chunks`.
2. Định nghĩa versioning rule:
     - CDC `LSN` là nguồn sự thật cuối cùng cho ordering.

3. Refactor `chunkingService.ts` theo rule mới:
    - bỏ toàn bộ logic preprocess markdown/HTML cũ.
    - chỉ dùng `RecursiveCharacterTextSplitter` để chunk text đầu vào.
    - khi input dưới 3000 ký tự thì không split, trả về một chunk duy nhất.
    - khi input từ 3000 ký tự trở lên, dùng `RecursiveCharacterTextSplitter` để chunk.
    - xóa metadata.

Deliverable: danh sách mapper cho từng bảng quan trọng.

---

## 6. Hybrid search bằng Drizzle ORM

1. Refactor `retrievalService.ts` để không dùng luồng retrieve nhiều bước kiểu hiện tại; thay bằng một query builder Drizzle duy nhất sinh ra SQL hybrid search.
2. Hàm hybrid search phải nhận đúng các tham số:
   - `filter`
   - `search`
   - `select`
   - `limit`
   - `page`
   - `perPage`
   - `groupBy`
   - `orderBy`
3. Query builder phải biểu diễn pseudo query:
   - `select *, (select * from B as a json array in a field)`
   - `from A`
   - `join B, join C, ...`
   - `where (điều kiện hybrid search)`
    - `group by` theo nhóm được sinh từ điều kiện hybrid search
   - `order by` theo điểm tổng hợp từ `where` trong từng nhóm
   - `limit` / `skip`
4. Tổng hợp score phải theo quy tắc:
   - `where A and B -> scoreA * scoreB`
   - `where A or B -> max(scoreA, scoreB)`
   - `where !A -> (threshold - scoreA) / threshold`
   - với biểu thức lồng nhau như `where A and (B or (C and D))`, xử lý từ trong ra ngoài để ra con điểm cuối cùng.
5. Hybrid search phải giữ cả:
   - structured filter
   - semantic/pgvector score
   - fuzzy/pg_trgm score
   trong cùng một query, không tách nhiều round-trip.
6. `select` phải hỗ trợ lấy full field của các bảng được chỉ định, nhưng vẫn trả về một row gốc với các bảng con dạng JSON array/JSON object theo pseudo query.
7. `recommendedIds` chỉ ảnh hưởng thứ tự/grouping, không được loại bỏ hard filter logic.
8. Thêm hàm `getSchema()` để trả schema đã chuẩn hóa cho LLM theo format:
   - Legends:
     - `i = int`
     - `f = float`
     - `b = bool`
     - `s = string`
     - `t = text`
     - `d = date`
     - `dt = datetime`
     - `e(...) = enum`
   - Capabilities:
     - `[s] = semantic search`
     - `[fs] = fuzzy + semantic search`
   - Output format:
     - `Tables:`
     - `<Table>{`
     - ` <field>:<type>[capabilities]`
     - `}`
   - `getSchema()` phải trả về đúng danh sách bảng và field được support để LLM biết field nào dùng exact, fuzzy, semantic hay cả hai.

Deliverable: RAG chain chạy được với LangChain và có thể thay retriever độc lập.

Bắt buộc tham khảo: [params của hybridSearch](./params%20của%20hàm%20hybridSearch.md)
---

## 7. Thiết kế CDC mức thấp với `pg-logical-replication`

1. Tạo bảng checkpoint CDC:
   ```sql
   CREATE TABLE cdc_state (
       id int PRIMARY KEY,
       processed_lsn pg_lsn NULL
   );
   ```
   - `processed_lsn` phải nullable để phân biệt trạng thái chưa bootstrap với trạng thái đã chạy CDC.
2. Bật logical replication cho DB nguồn(chỉnh wal_level = logical, max_replication_slots = 1, max_wal_senders = 1). Tham số thứ 1 cần cho pg-logical-replication còn 2 tham số cuối là do chỉ có 1 consumer.
3. Tạo replication slot và publication cho các bảng cần theo dõi.
4. Khi ứng dụng khởi chạy, đọc `cdc_state.processed_lsn`:
   - nếu `processed_lsn IS NULL` thì hiểu là chưa từng tạo snapshot và sync dữ liệu;
   - khi đó phải tạo snapshot cho trạng thái DB hiện tại và sync toàn bộ các bảng trong phạm vi RAG;
   - sau khi bootstrap xong, ghi `processed_lsn` về mốc LSN khởi tạo tương ứng để CDC bắt đầu từ ranh giới đó.
5. Nếu `processed_lsn IS NOT NULL`, chạy CDC worker đọc WAL kể từ `processed_lsn`.
6. Mỗi khi worker xử lý xong một `LSN` và sync xong dữ liệu tương ứng, phải cập nhật lại `cdc_state.processed_lsn`.
7. Viết CDC worker dùng `pg-logical-replication` để đọc WAL changes.
8. Bắt và xử lý đủ các loại event schema-change và data-change:
    - `INSERT`
    - `UPDATE`
    - `DELETE`
    - đổi tên bảng
    - đổi tên trường
    - xóa bảng
    - xóa trường
9. Khi đổi tên bảng hoặc trường:
    - cập nhật mapping metadata của `Chunks.tableName` / `Chunks.fieldName`
    - giữ nguyên `recordId`, `contentHash`, `embeddingId` nếu nội dung không đổi
    - tạo alias mapping tạm thời giữa tên cũ và tên mới để replay/checkpoint không bị lệch
10. Khi xóa bảng hoặc trường:
    - xóa toàn bộ chunk liên quan
    - nếu xóa trường thì chỉ xử lý các chunk có `fieldName` tương ứng
    - nếu xóa bảng thì xử lý toàn bộ chunk của `tableName` đó
11. Chỉ ACK event khi đã:
     - parse xong,
     - map xong,
     - cập nhật `Chunks` và `Embeddings` xong,
     - ghi checkpoint xong.
12. Nếu xử lý fail giữa chừng, không ACK để event được redeliver.

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

1. Trước khi backfill, bật logical replication và chốt một `start_lsn` làm ranh giới đồng bộ giữa snapshot và delta.
2. Tại thời điểm bắt đầu backfill, mở một transaction nhất quán để **exported snapshot** của DB tại đúng thời điểm đó.
3. Backfill có thể chạy qua nhiều transaction con theo batch, nhưng **mỗi transaction con phải import cùng exported snapshot này trước khi đọc dữ liệu** để luôn nhìn thấy đúng “ảnh chụp cũ” của DB.
4. Khi backfill chạy với exported snapshot, người dùng vẫn có thể update dữ liệu bình thường; các thay đổi này **không làm lệch dữ liệu backfill** vì mỗi batch đều đọc theo cùng một snapshot đã export.
5. Những update/insert/delete phát sinh trong lúc backfill sẽ tiếp tục đi vào **WAL** và được CDC xử lý sau từ `start_lsn`.
6. Chunking và embed là xử lý ở tầng code ngoài transaction DB:
   - transaction chỉ dùng để đọc snapshot nhất quán từ DB nguồn
   - sau khi đọc ra dữ liệu, code mới chunk, embed, rồi ghi thẳng sang bảng đích (`Chunks`/`Embeddings`)
   - như vậy chunking không nằm trong query, nhưng vẫn được thực hiện trên dữ liệu đọc từ snapshot cũ
7. Backfill phải chạy async theo batch và ghi thẳng sang bảng đích (`Chunks`/`Embeddings`) theo cùng quy tắc idempotent/upsert.
8. Sau khi backfill xong, ghi checkpoint khởi tạo tại `start_lsn` hoặc mốc tương đương để CDC bắt đầu replay delta từ đúng ranh giới đó.
9. Từ checkpoint khởi tạo trở đi, CDC xử lý các event phát sinh sau snapshot theo thứ tự `LSN`.
10. Backfill cần chunk/batch để tránh load lớn và phải hỗ trợ resume bằng checkpoint tiến độ riêng:
   - lưu `backfill_checkpoint` theo mức `tableName + lastProcessedPk` hoặc `tableName + offset`
   - mỗi batch xử lý xong mới cập nhật checkpoint
   - khi restart thì đọc lại checkpoint cuối cùng và tiếp tục từ batch kế tiếp
   - các upsert phải idempotent để nếu batch cuối bị chạy lại cũng không tạo dữ liệu sai

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

### Phase 2: Embedding + Chunking
- Tái sử dụng `EmbeddingService`.
- Refactor `chunkingService.ts` theo `RecursiveCharacterTextSplitter` và ngưỡng 3000 ký tự.

### Phase 3: Hybrid search bằng Drizzle
- Refactor `retrievalService.ts` sang Drizzle query builder.
- Xây dựng một query duy nhất cho hybrid search, grouping, ordering, pagination.

### Phase 4: Backfill
- Build script nạp toàn bộ data từ Postgres.
- Đảm bảo idempotent upsert.

### Phase 5: CDC
- Bật logical replication.
- Viết CDC worker với ack sau xử lý.
- Thêm checkpoint store.

### Phase 6: Ordering + resilience
- Áp per-entity serialization.
- Chặn stale write theo LSN/version.
- Thêm retry/DLQ/metrics.

### Phase 7: Tối ưu
- Benchmark embedding/retrieval.
- Tuning top-k, batch size, chunking, similarity threshold.

---

## 13. Tiêu chí hoàn thành

- RAG không phụ thuộc LangChain orchestration.
- Embedding local đa ngôn ngữ hoạt động ổn định.
- Dữ liệu RAG lấy từ các bảng PostgreSQL đã chốt.
- CDC cập nhật tự động qua `pg-logical-replication`.
- ACK chỉ sau khi xử lý xong event.
- Không xảy ra stale overwrite khi event đến out-of-order.
