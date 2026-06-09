# CDC Dictionary

## Logical Replication

Là cơ chế PostgreSQL phát các thay đổi dữ liệu theo kiểu **stream sự kiện** thay vì chỉ để client tự đi query lại.

Hiểu đơn giản:
- DB không chỉ lưu dữ liệu cuối cùng.
- Mỗi lần có `INSERT`, `UPDATE`, `DELETE`, PostgreSQL có thể phát ra event tương ứng.
- Ứng dụng CDC đọc các event đó để đồng bộ dữ liệu sang hệ thống khác, ví dụ search index, vector store, cache.

Điểm mạnh:
- bắt thay đổi gần realtime
- không cần quét toàn bảng liên tục
- phù hợp để tự động cập nhật index

---

## Replication Slot

Là “điểm giữ trạng thái đọc” của logical replication.

Hiểu đơn giản:
- PostgreSQL phát event thay đổi liên tục.
- replication slot giúp DB biết consumer đã đọc tới đâu.
- nếu consumer chưa ACK hoặc chưa đọc xong, DB sẽ giữ lại WAL cần thiết.

Tác dụng:
- tránh mất event khi consumer tạm ngưng
- đảm bảo có thể đọc tiếp từ đúng điểm cũ

Lưu ý:
- nếu consumer chết quá lâu mà không đọc tiếp, WAL có thể bị giữ lại nhiều, làm tốn dung lượng.

---

## Publication

Là “danh sách bảng được phép phát thay đổi” trong logical replication.

Hiểu đơn giản:
- không phải mọi bảng trong DB đều cần phát event.
- publication nói cho PostgreSQL biết: bảng nào được stream ra ngoài.

Ví dụ:
- chỉ publish `Conferences`, `ConferenceOrganizations`, `Journals`...
- các bảng không liên quan thì bỏ qua.

Tác dụng:
- giảm noise
- giảm tải CDC
- chỉ theo dõi đúng nguồn dữ liệu cần sync

---

## LSN

`LSN` là viết tắt của `Log Sequence Number`.

Đây là “địa chỉ” hoặc “mốc” của một vị trí trong WAL.

Hiểu đơn giản:
- mỗi thay đổi trong PostgreSQL được ghi vào WAL theo thứ tự.
- `LSN` cho biết event nào đến trước, event nào đến sau.

Trong CDC:
- dùng `LSN` để checkpoint
- dùng `LSN` để chống out-of-order
- dùng `LSN` để biết event nào là mới hơn

---

## TXID

`TXID` là `Transaction ID`.

Hiểu đơn giản:
- một transaction có thể gồm nhiều thay đổi.
- `TXID` là mã của transaction đó.

Tác dụng:
- gom các event cùng transaction lại với nhau
- biết event nào thuộc cùng một commit logic
- hữu ích khi cần debug hoặc replay

---

## PK

`PK` là `Primary Key`, khóa chính của bản ghi.

Hiểu đơn giản:
- đây là định danh duy nhất của một row.
- ví dụ: `conference.id`, `journal.id`.

Trong CDC:
- dùng `PK` để biết event đang tác động lên bản ghi nào
- dùng làm khóa để upsert/delete đúng row

---

## OP

`OP` là loại thao tác xảy ra trên row.

Thường gặp:
- `INSERT`: thêm mới
- `UPDATE`: cập nhật
- `DELETE`: xóa

Tác dụng:
- giúp worker biết phải tạo, sửa hay xóa dữ liệu downstream

---

## Commit Timestamp

Là thời điểm transaction được commit xong.

Hiểu đơn giản:
- transaction có thể bắt đầu sớm nhưng commit muộn.
- commit timestamp phản ánh lúc dữ liệu thật sự “chốt” vào DB.

Tác dụng:
- hỗ trợ debug thứ tự sự kiện
- làm metadata cho audit
- nhưng khi chống out-of-order, `LSN` thường đáng tin hơn timestamp

---

## Backfill

Là bước **nạp dữ liệu ban đầu** từ DB hiện có vào hệ thống đích trước khi chạy CDC realtime.

Hiểu đơn giản:
- trước khi nghe event thay đổi, phải copy toàn bộ dữ liệu nền ban đầu.
- nếu không backfill, hệ thống đích sẽ bị thiếu dữ liệu lịch sử.

Ví dụ:
- load toàn bộ conference/journal hiện có vào `Chunks` và `Embeddings`
- sau đó mới bật CDC để chỉ xử lý phần thay đổi mới

---

## Bootstrap

Là quá trình khởi tạo hệ thống đồng bộ từ đầu.

Trong bối cảnh CDC/RAG, bootstrap thường gồm:
- tạo schema bảng đích
- tạo replication slot/publication nếu cần
- backfill dữ liệu ban đầu
- ghi checkpoint khởi tạo
- bắt đầu đọc CDC realtime

Hiểu ngắn gọn:
- backfill là một phần của bootstrap
- bootstrap là toàn bộ quy trình khởi động hệ thống từ trạng thái trống đến trạng thái sẵn sàng chạy realtime

---

## Cách đúng nhất cho backfill + CDC

Đây là cách an toàn nhất khi dữ liệu nguồn vẫn có người dùng ghi trong lúc bootstrap:

1. **Bật logical replication trước**
   - tạo `publication`
   - tạo `replication slot`
   - bắt đầu đọc WAL để không bỏ lỡ thay đổi mới

2. **Chốt một mốc LSN làm ranh giới snapshot**
   - lấy `start_lsn` ngay trước khi backfill chạy
   - mốc này là điểm để CDC bắt đầu replay delta

3. **Backfill đọc dữ liệu theo snapshot ổn định**
   - backfill chỉ đọc dữ liệu theo trạng thái tại snapshot đó
   - nên chạy async và theo batch

4. **Trong lúc backfill, CDC chỉ buffer event**
   - không apply trực tiếp vào `Chunks`/`Embeddings` nếu chưa xong backfill
   - vẫn tiếp tục nhận event từ WAL để không mất dữ liệu

5. **Backfill xong thì replay delta từ `start_lsn` trở đi**
   - các event phát sinh sau snapshot sẽ được áp dụng theo thứ tự `LSN`
   - những thay đổi đã xuất hiện trong backfill sẽ được CDC xác nhận lại bằng version/LSN để tránh ghi đè sai

6. **Upsert phải idempotent**
   - nếu backfill và CDC cùng chạm một row/field, bản mới hơn thắng
   - dùng `tableName + recordId + fieldName + contentHash + LSN` để quyết định ghi hay bỏ qua

Nói ngắn gọn:
- **Backfill** lấy ảnh chụp ổn định của dữ liệu hiện tại
- **CDC** xử lý mọi thay đổi sau ảnh chụp đó
- **LSN** là ranh giới để ghép 2 phần này lại mà không mất dữ liệu, không trùng logic

---

## Tóm tắt ngắn

- `logical replication`: stream thay đổi từ PostgreSQL
- `replication slot`: giữ vị trí đọc, tránh mất event
- `publication`: chọn bảng được phát event
- `LSN`: mốc thứ tự trong WAL
- `TXID`: mã transaction
- `PK`: khóa chính của row
- `OP`: kiểu thao tác (`INSERT`/`UPDATE`/`DELETE`)
- `commit_timestamp`: thời điểm commit
- `backfill`: nạp dữ liệu ban đầu
- `bootstrap`: quy trình khởi tạo toàn bộ pipeline
