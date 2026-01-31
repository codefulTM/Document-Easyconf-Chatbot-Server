# Vấn đề
- Vector store hiện đang chạy song hành với PostgreSQL, tuy nhiên khi xóa vector trong vector store thì không xóa trong PostgreSQL, dẫn đến việc dữ liệu bị dư thừa, không đồng bộ.

# Giải pháp 
- Thêm cột collectionName vào bảng "Chunks" trong PostgreSQL để lưu tên collection tương ứng trong vector store.
- Thêm cột is_deleted vào bảng "Chunks" trong PostgreSQL để đánh dấu các chunk đã bị xóa trong vector store.
- Sau khi thêm các cột, cập nhật lại thay đổi cho database.
- Cập nhật lại các API CRUD chunk trên backend.
- Quy trình xóa:
    - Đầu tiên, đánh dấu các chunk cần xóa trong PostgreSQL bằng cách đặt is_deleted = true dựa trên collectionName.
    - Sau đó, xóa các vector tương ứng trong vector store dựa trên collectionName.
    - Cuối cùng, xóa các bản ghi trong PostgreSQL có is_deleted = true nếu xóa thành công trong vector store.

# Các hàm cần sửa(trong file vectorStoreService.qdrant.ts)
- deleteDocuments
- addDocuments
- clear