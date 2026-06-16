# RAG Refactor Plan

## Mục tiêu

Refactor hệ thống RAG của `Easyconf-Chatbot-Server` theo các yêu cầu:

- Không dùng LangChain làm orchestration layer.
- Tái sử dụng **`EmbeddingService`** hiện có để sinh embedding local.
- Refactor **`chunkingService.ts`** để dùng `RecursiveCharacterTextSplitter` của LangChain, nhưng chỉ chunk khi input >= 3000 ký tự.
- Refactor **`retrievalService.ts`** sang **Drizzle ORM** để thực hiện hybrid search trong **một query duy nhất** bằng query builder.
- Dùng **local embedding model** nhẹ, hỗ trợ đa ngôn ngữ.
- Dữ liệu RAG lấy từ **các cột trong các bảng PostgreSQL** dựa trên schema của `Easyconf-BE`.
- Dùng **CDC qua Debezium PostgreSQL Connector** để tự động đồng bộ dữ liệu.
- Chỉ **commit Kafka offset sau khi event đã được xử lý xong thành công**.
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
   - `Chunks.originalContent`
   - `Chunks.content`
   - `Chunks.embeddingId`
   - `Embeddings.id`
   - `Embeddings.embedding`
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
   - `Chunks.primaryKey`
   - `Chunks.fieldName`
   - `Chunks.originalContent`
   - `Chunks.content`
   - `Chunks.embeddingId`
   - `Embeddings.id`
   - `Embeddings.embedding`
4. Tạo thêm bảng trạng thái record riêng để giữ version kể cả khi `Chunks` đã bị xóa:
   ```sql
   CREATE TABLE rag_record_state (
      table_name text NOT NULL,
      primary_key text NOT NULL,
      last_lsn bigint NOT NULL,
      last_op text NOT NULL,
      is_deleted boolean NOT NULL DEFAULT false,
      updated_at timestamptz NOT NULL DEFAULT now(),
      PRIMARY KEY (table_name, primary_key)
   );
   ```
   - `rag_record_state` là nguồn sự thật cho version/idempotency của từng record;
   - `Chunks` có thể bị xóa khi `delete`, nhưng state của record không được mất;
   - nếu chỉ giữ version trong `Chunks` thì duplicate của một event cũ sẽ không có gì để so sánh sau khi record bị xóa. Ví dụ: xét trường hợp sau:
      - event 1: update record A
      - event 2: xóa record A
      - event 3: event 1 bị duplicated -> lúc này do event 2 làm mất lsn được lưu trong bảng Chunks khiến cho app tưởng phải tạo mới record A, trong khi thực tế là đã delete
   - `rag_record_state` giải quyết đúng case `update -> delete -> duplicate update`, vì state vẫn còn để skip event cũ.
5. Tạo unique constraint theo khóa logic của chunk để upsert an toàn:
   - `tableName + primaryKey + fieldName + originalContent + content`
6. Với trường hợp dedupe theo content giữa nhiều record, cho phép nhiều dòng `Chunks` cùng `tableName + fieldName + content` nhưng khác `primaryKey` và `id`.

Deliverable: schema vector store và chiến lược upsert.

---

## 5. Thiết kế pipeline build document trong `Chunks` từ DB(xử lý sự kiện c + r + u + d)
NOTE: Nếu một sự kiện có `lsn` bị outdated so với `rag_record_state` -> skip event
### Sự kiện r(read), c(create), u(update): chạy cùng một luồng
- lấy data trong `after`
- xác định `tableName`, `primaryKey` mà sự kiện đang ảnh hưởng
- start transaction
   - foreach `fieldName` trong `after`, nếu `fieldName` có trong `embeddingFields`(các field có hỗ trợ embedding):
      - lấy value của field
         - value là null/undefined -> xem như value bị xóa -> Xóa các chunk có cùng `tableName`, `fieldName`, `primaryKey` + cleanup các embedding mồ côi(nếu không bị chunk nào khác refer tới). -> continue;
         - cast value thành `stringValue`.
         - `stringValue` rỗng -> Xóa các chunk có cùng `tableName`, `fieldName`, `primaryKey` + cleanup các embedding mồ côi(nếu không bị chunk nào khác refer tới). -> continue;
         - Tìm toàn bộ các chunk có cùng `tableName`, `fieldName`, `primaryKey`, `originalContent` !== `stringValue`(originalContent bị lỗi thời)
            - xóa embedding mỗi chunk đang sử dụng KHI VÀ CHỈ KHI embedding đang không được sử dụng bởi chunk nào khác
            - xóa các chunk này.
         - Tìm 1 chunk có cùng `tableName`, `fieldName`, `primaryKey`, `originalContent` === `stringValue`.
            - Nếu có: continue; (tới đây là đã xong thao tác cập nhật dữ liệu trong trường hợp dữ liệu đã có sẵn. đây là trường hợp mà event bị duplicated nhưng app vẫn xử lý vì một lý do nào đó, mà thôi cứ xét luôn cho chắc).
         - Tìm tất cả các chunk có cùng `tableName`, `fieldName`, khác `primaryKey` với `primaryKey` của event hiện tại nhưng cùng `primaryKey` với nhau, có `originalContent` === `stringValue`
            - clone ra thêm các chunk có cùng giá trị các trường trong `Chunks` nhưng đổi `primaryKey`, `id`. 
            - mục đích là để khỏi chunking, khỏi lấy embedding mà vẫn có nội dung.
            - continue(tới đây là đã xong thao tác cập nhật dữ liệu, nhưng chỉ trong trường hợp tồn tại ít nhất 1 chunk này)
         - Thêm vào các chunk có cùng `tableName`, `fieldName`, `primaryKey` và `originalContent` mới = `stringValue`:
            - cắt `stringValue` thành các chunk -> `content`
            - tìm 1 chunk có cùng `content` và `embeddingId` != null
            - nếu có `embeddingId` -> tái sử dụng `embeddingId`. 
            - nếu không, lấy embedding cho mỗi chunk rồi thêm vào `Embeddings`
            - thêm mới chunk
   - update vào `rag_record_state`: cập nhật lại `lsn` của record = `lsn` của sự kiện này.
- end transaction
### Sự kiện d(delete)
- lấy data trong `before`
- xác định `tableName`, `primaryKey` mà sự kiện đang ảnh hưởng
- xác định `lsn` của sự kiện
- start transaction
   - xóa các embedding trong `Embeddings` chỉ được liên kết với 1 chunk duy nhất và chunk đó là chunk có cùng `tableName` và `primaryKey` mà sự kiện đang ảnh hưởng
   - xóa các chunk trong `Chunks` có cùng `tableName` và `primaryKey` mà sự kiện đang ảnh hưởng
   - update vào `rag_record_state`: cập nhật lại `lsn` của record = `lsn` của sự kiện này.
- end transaction

2. Định nghĩa versioning rule:
     - CDC `source.lsn` là nguồn sự thật cuối cùng cho ordering/idempotency.

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

## 7. Thiết kế CDC với Debezium

1. Dùng **Debezium PostgreSQL Connector** làm nguồn CDC, không dùng `pg-logical-replication` trong app:
   - app không đọc WAL trực tiếp;
   - Debezium quản lý replication slot, publication, offset, schema history;
   - app chỉ consume event từ topic Debezium.
2. Bật cấu hình PostgreSQL cần cho Debezium:
   - `wal_level = logical`;
   - `max_replication_slots` đủ cho connector;
   - `max_wal_senders` đủ cho connector;
   - `max_slot_wal_keep_size` mặc định là -1 -> Không cần set. -1 nghĩa là Debezium có thể giữ WAL segment không giới hạn và khi còn giữ thì Postgre sẽ không xóa các segment này.
   - user Debezium có quyền replication và quyền đọc các bảng trong phạm vi RAG.
3. Chuẩn bị hạ tầng Kafka/Connect:
   - Kafka broker và Kafka Connect phải cùng network với DB nguồn và app; Note: Kafka broker hiểu nôm na là database chứa message, còn Kafka Connect là một process chạy connector. Cái Debezium bản chất là một connector chạy trong Kafka Connect. Nó là plugin của Kafka Connect.
   - Kafka Connect phải load được Debezium PostgreSQL connector plugin;
   - cấu hình các topic nội bộ của Kafka Connect: `config.storage.topic`, `offset.storage.topic`, `status.storage.topic`;
   - cấu hình replication factor phù hợp môi trường dev/local;
   - xác nhận Kafka Connect REST API hoạt động ổn định trước khi register connector.
4. Tạo Kafka topic CDC với **3 partitions ngay từ đầu**:
   - số partition là cấu hình hạ tầng của topic, không phụ thuộc số lượng `{tableName}-{primaryKey}`;
   - key theo row chỉ quyết định record nào vào partition nào, không quyết định tổng số partition;
   - không đổi số partition trong giai đoạn vận hành bình thường;
   - nếu sau này cần đổi partition thì dùng TO-DO `changeKafkaPartition(numOfP)` trong `server.ts`.
5. Tạo connector chỉ theo dõi các bảng/cột nằm trong phạm vi RAG:
   - dùng include list cho schema/table;
   - loại các bảng không support RAG ngay từ connector nếu có thể;
   - vẫn validate lại allowlist ở app để tránh index nhầm khi cấu hình connector thay đổi.
6. Đăng ký connector qua Kafka Connect REST API:
   - cấu hình `connector.class = io.debezium.connector.postgresql.PostgresConnector`;
   - cấu hình `database.hostname`, `database.port`, `database.user`, `database.password`, `database.dbname`;
   - cấu hình `topic.prefix`;
   - cấu hình `plugin.name = pgoutput`;
   - cấu hình `snapshot.mode` theo mục tiêu bootstrap hiện tại;
   - nếu dùng signal table thì khai báo bảng signal ngay từ đầu;
   - kiểm tra connector status phải là `RUNNING` trước khi bật consumer app.
7. Xác nhận các quyền và bảng phục vụ Debezium:
   - user Debezium phải có quyền `REPLICATION`;
   - user Debezium phải có quyền `SELECT` trên các bảng RAG;
   - nếu dùng signal table thì user Debezium cũng phải đọc được bảng đó;
   - nếu Debezium tự tạo publication thì kiểm tra publication tương ứng đã được tạo đúng.
8. Chuẩn hóa envelope event Debezium trước khi đưa vào pipeline nội bộ:
   - `source.lsn`;
   - `source.txId`;
   - `source.schema`;
   - `source.table`;
   - primary key;
   - `op` (`c`, `u`, `d`, `r`);
   - `before`;
   - `after`;
   - `ts_ms`;
   - snapshot marker nếu event đến từ snapshot.
9. App consume Debezium topic theo consumer group riêng cho RAG indexer:
   - commit Kafka offset chỉ sau khi xử lý xong event thành công;
   - nếu event lỗi thì ghi log, và không commit Kafka offset của event đó và thoát ứng dụng;
   - không dùng offset Kafka thay thế hoàn toàn `lsn` trong dữ liệu đích, vì ordering/idempotency vẫn phải dựa trên `source.lsn`;
10. Debezium đã key event theo từng row mặc định, nên không cần thêm bước tự set Kafka message key trong app:
   - giữ key mặc định của Debezium để tránh cấu hình thừa;
   - consumer vẫn hiểu stream theo từng row/record như Debezium phát ra;
   - nếu sau này cần đổi key strategy thì chỉ làm khi có lý do kỹ thuật rõ ràng.
11. Kafka topic phải được tạo/khai báo với **3 partitions ngay từ đầu**:
   - số partition là cấu hình hạ tầng của topic, không phụ thuộc số lượng `{tableName}-{primaryKey}`;
   - key theo row chỉ quyết định record nào vào partition nào, không quyết định tổng số partition;
   - consumer xử lý song song trên 3 partition này, còn ordering/idempotency vẫn dựa vào `source.lsn`.
   - trong server.ts có thể gọi hàm này trong quá trình start app, cứ mỗi lần muốn đổi số partition thì cứ đổi numOfP thôi:
      ```
      async function changeKafkaPartition(numOfP: number) {
         if(numOfP === currentNumOfP)  return;
         await turn_off_debezium() // để không gửi event qua kafka nữa
         await wait_until_all_partitions_go_blank()
         await change_kafka_config()
         await turn_on_debezium()
      }
      ```
12. Bắt và xử lý các loại event data-change:
   - `c`: insert;
   - `u`: update;
   - `d`: delete;
    - `r`: snapshot read.
13. Schema-change không xử lý bằng cách tự parse WAL trong app(Bỏ):
    - NOTE: Do hệ thống không dùng migration, cần nghĩ cách để xử lý việc đổi tên bảng hoặc trường.
    - Debezium ghi schema history theo connector;
    - thay đổi schema phải đi qua migration có kiểm soát;
    - migration phải cập nhật RAG mapping metadata trước hoặc cùng lúc với thay đổi DB.
14. Khi đổi tên bảng hoặc trường(Bỏ):
    - cập nhật mapping metadata của `Chunks.tableName` / `Chunks.fieldName`;
    - giữ nguyên `primaryKey`, `originalContent`, `content`, `embeddingId` nếu nội dung không đổi;
    - tạo alias mapping tạm thời giữa tên cũ và tên mới để xử lý event cũ còn tồn trong topic;
    - chỉ xóa alias sau khi chắc chắn consumer đã vượt qua offset/LSN cuối cùng dùng tên cũ.
15. Khi xóa bảng hoặc trường(Bỏ):
    - migration phải đánh dấu mapping là disabled trước;
    - nếu xóa trường thì chỉ xóa chunk có `fieldName` tương ứng;
    - nếu xóa bảng thì xóa toàn bộ chunk của `tableName` đó;
    - cleanup embedding mồ côi theo rule ở mục 5.
16. Mỗi event được chuyển thành danh sách field job nội bộ:
    - các field job của cùng row được xử lý trong cùng transaction.
<!-- Đang ở bước này -->
Deliverable: Debezium connector config + RAG CDC consumer consume theo key `{tableName}-{primaryKey}` và commit offset đúng thời điểm.

---

## 8. Chống mất sự kiện, out-of-order và khóa theo key
1. Với cùng `{tableName}-{primaryKey}`, worker phải xử lý theo thứ tự Kafka gửi trong partition và kiểm tra thêm `source.lsn`:
   - event mới hơn không được ghi đè bởi event cũ hơn;
   - event cũ hơn version đã lưu phải bị skip idempotently;
    - `source.lsn` là căn cứ duy nhất để xác định thứ tự event.
2. Áp dụng rule ghi dữ liệu theo “last committed lsn wins”, không theo “task start time”.
3. Xử lý async:
   - Xử lý async cho các partition khác nhau, nhưng với mỗi partition thì xử lý tuần tự.
4. Với trường hợp các event đến từ **cùng một DB transaction** (ví dụ INSERT Conference rồi INSERT Session FK reference):
    - Dùng transaction metadata topic (`<topicPrefix>.transaction`) để biết khi nào một transaction bắt đầu và kết thúc;
    - Buffer tất cả DML events của cùng `transaction.id` vào bộ nhớ (Map<txId, DebeziumEvent[]>);
    - Khi nhận END event với `event_count`, kiểm tra buffer[txId].length === event_count:
        - Nếu đủ: sắp xếp events theo `total_order`, xử lý tuần tự trong transaction DB.
        - Nếu thiếu (event chưa kịp tới): giữ buffer, đợi thêm.
    - Buffer có timeout + cleanup (ví dụ 30s), tránh memory leak nếu END bị mất.
5. Lưu ý: transaction metadata chỉ cần thiết khi có **FK dependency trong cùng transaction**.
    - Các event không cùng transaction (row update riêng lẻ) xử lý bình thường, không qua buffer.
6. Nếu một event fail:
    - ghi log lỗi với raw Debezium event, normalized metadata, stack trace, `topic`, `partition`, `offset`, `lsn`, và stream key;
    - không commit Kafka offset của event lỗi;
    - dừng app lại.

Deliverable: cơ chế khóa theo `{tableName}-{primaryKey}`, chống stale write, buffer transaction để xử lý FK dependency, không chặn toàn bộ pipeline khi một record stream lỗi.

---

## 9. Chiến lược commit offset, retry và DLQ

1. Với Debezium, “ACK” của app tương ứng với **commit Kafka offset** của consumer.
2. Chỉ commit offset sau khi:
   - parse event xong;
   - map sang field job xong;
   - cập nhật `Chunks` và `Embeddings` xong;
   - cập nhật `rag_record_state` xong.
3. Không commit offset cho event lỗi:
   - event lỗi được giữ lại trong Kafka bằng cách không commit offset;
   - app log lý do lỗi và stop app.
4. Retry chính là redelivery từ Kafka:
   - khi app restart hoặc consumer rebalance, Kafka gửi lại offset chưa commit;
   - event được xử lý lại từ đầu và vẫn đi qua idempotency/`lastLsn` check.
5. DLQ không phải luồng mặc định cho lỗi xử lý event(Tức là không dùng DLQ):
   - chỉ dùng DLQ nếu có quyết định vận hành riêng cho event không thể xử lý nhưng vẫn muốn tiến offset;
   - nếu ghi DLQ và commit offset thì phải được coi là ngoại lệ có kiểm soát, không phải behavior mặc định;
   - mặc định của plan này là giữ event lỗi bằng Kafka offset chưa commit để không mất dữ liệu.
6. Nếu crash sau khi write nhưng trước commit offset:
   - event sẽ được Kafka redeliver;
   - write phải idempotent để không tạo bản ghi lỗi.
7. Checkpoint/offset phải phản ánh event đã xử lý xong thành công, không phải event chỉ mới đọc được hoặc đã bị skip vì đang blocked.
8. Cần cấu hình consumer để tránh commit tự động:
   - tắt auto commit;
   - commit thủ công sau transaction DB thành công;
   - nếu dùng KafkaJS/NestJS wrapper thì phải kiểm tra rõ API pause/resume/commit offset để không vô tình ack message lỗi.

Deliverable: semantics at-least-once an toàn, khóa lỗi theo `{tableName}-{primaryKey}`, không commit offset cho event lỗi.

---

## 10. Incremental Snapshot và bootstrap bằng Debezium (TO-DO: Chỉ implement khi muốn re-index lại chunk từ đầu bằng cách yêu cầu Debezium tạo snapshot)

1. Dùng **Debezium Incremental Snapshot** thay cho backfill tự quản bằng exported snapshot:
   - Debezium chia dữ liệu thành nhiều chunk nhỏ;
   - app nhận snapshot event như event bình thường (`op = r`);
   - dữ liệu snapshot và WAL mới được đan xen an toàn bằng cơ chế watermarking của Debezium.
2. Bật signal table hoặc signaling channel cho Debezium để trigger incremental snapshot theo bảng:
   - app/admin gửi signal snapshot cho các bảng trong phạm vi RAG;
   - có thể snapshot lại một bảng hoặc một nhóm bảng mà không dừng CDC.
3. Cấu hình chunk size phù hợp:
   - chunk nhỏ để tránh giữ transaction/read quá lâu;
   - chunk đủ lớn để giảm overhead;
   - có thể tune riêng theo bảng lớn/nhỏ.
4. Debezium watermarking đảm bảo:
   - snapshot đọc theo từng khoảng khóa chính;
   - WAL event phát sinh trong lúc snapshot vẫn được capture;
   - event snapshot cũ không ghi đè event WAL mới hơn nhờ `source.lsn` và `lastLsn` check ở app.
5. App xử lý snapshot event và streaming event chung một pipeline:
   - đều đi qua document builder;
   - đều đi qua chunking + embedding writer;
   - đều dùng idempotent upsert và “last committed lsn wins”.
6. Khi bootstrap lần đầu:
   - tạo Debezium connector;
   - cho connector bắt đầu stream WAL;
   - trigger incremental snapshot cho các bảng RAG;
   - app consume cả `r/c/u/d` event và build index dần.
7. Không cần ghi checkpoint khởi tạo kiểu `start_lsn` trong app:
   - Debezium quản lý offset và snapshot progress;
   - app vẫn lưu `rag_record_state` để audit, idempotency và resume nội bộ.
8. Nếu snapshot bị gián đoạn:
   - Debezium resume từ offset/snapshot progress đã lưu;
   - app vẫn idempotent nếu nhận lại event;
9. Chunking và embed vẫn là xử lý ở tầng code:
   - Debezium chỉ cung cấp row-level event;
   - code mới build document, chunk, embed, rồi ghi vào `Chunks`/`Embeddings`;
   - transaction ghi đích phải nhỏ, theo event `{tableName}-{primaryKey}` hoặc batch nhỏ có rollback rõ ràng.
10. Với bảng rất lớn, cho phép chạy incremental snapshot theo từng đợt:
   - trigger theo bảng ưu tiên;
   - theo dõi lag, số stream đang blocked, throughput embedding;
   - tạm pause/resume snapshot nếu hệ thống embedding hoặc DB đích quá tải.
11. Vì incremental snapshot chia dữ liệu thành các khối nhỏ:
   - không cần tự viết backfill đọc toàn bảng bằng exported snapshot;
   - snapshot chunk được đan xen với WAL mới thông qua thuật toán watermarking của Debezium;
    - app chỉ cần đảm bảo idempotency, `lastLsn` guard, và commit offset sau khi xử lý thành công.

Deliverable: cấu hình Debezium incremental snapshot + pipeline bootstrap index bằng snapshot event có watermarking.

---

## 11. Kế hoạch triển khai theo phase

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

### Phase 4: Incremental Snapshot
- Cấu hình Debezium incremental snapshot cho các bảng RAG.
- Xử lý snapshot event chung pipeline với streaming event.
- Đảm bảo idempotent upsert.

### Phase 5: CDC
- Bật Debezium PostgreSQL Connector.
- Viết RAG CDC consumer với commit offset sau xử lý.
- Thêm `rag_record_state`.

### Phase 6: Ordering + resilience
- Áp khóa theo `{tableName}-{primaryKey}`.
- Chặn stale write theo `lsn`/version.
- Thêm `blockedStreams`, logging lỗi, metrics, và manual recovery flow.

### Phase 7: Tối ưu
- Benchmark embedding/retrieval.
- Tuning top-k, batch size, chunking, similarity threshold.

---

## 13. Tiêu chí hoàn thành

- RAG không phụ thuộc LangChain orchestration.
- Embedding local đa ngôn ngữ hoạt động ổn định.
- Dữ liệu RAG lấy từ các bảng PostgreSQL đã chốt.
- CDC cập nhật tự động qua Debezium PostgreSQL Connector.
- Kafka offset chỉ commit sau khi xử lý xong event thành công.
- Không xảy ra stale overwrite khi event đến out-of-order.
