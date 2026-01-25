# Vấn đề hiện tại
- Luồng keyword search hiện tại như sau:
    - Truyền vào cho hàm query của người dùng
    - Từ query, lấy ra các keyword
    - Tìm các chunk thích hợp(có chứa keyword) trong vector database:
        - Lấy ra từng batch chunk
        - Với mỗi chunk, kiểm tra xem có chứa keyword hay không
        - Nếu có, thêm chunk vào danh sách chunk trả về cho người dùng
- Vấn đề:
    - Việc lấy ra từng batch chunk, sau đó kiểm tra từng chunk một xem có keyword hay không sẽ làm chậm quá trình tìm kiếm
    - Bên cạnh đó, cũng không thể nào kiên nhẫn quét hết toàn bộ batch được, mà sẽ có limit số batch để kiểm tra keyword -> Có thể bỏ qua các chunk có chứa keyword nếu chúng nằm ngoài số batch được kiểm tra

# Giải pháp đề xuất
- Bởi vì Qdrant không hỗ trợ tìm kiếm theo keyword, nên ta sẽ cần sự hỗ trợ của một search engine bên ngoài. Ở đây, chúng em đề xuất sử dụng Elasticsearch.
- Giới thiệu ngắn về Elasticsearch:
    - Elasticsearch là một công cụ tìm kiếm và phân tích dữ liệu mã nguồn mở, xây dựng trên Lucene.
    - Dữ liệu được lưu dưới dạng document (JSON) và được index để hỗ trợ tìm kiếm full-text nhanh chóng.
    - Hỗ trợ truy vấn theo từ khóa, cụm từ, lọc, sắp xếp và truy vấn phức tạp với độ trễ thấp.
    - Hỗ trợ fuzzy search (tìm kiếm mờ), cho phép tìm các từ gần giống với từ khóa truy vấn, rất hữu ích khi người dùng gõ sai chính tả hoặc có nhiều biến thể của từ khóa. Fuzzy search trong Elasticsearch dựa trên Levenshtein edit distance để xác định mức độ tương đồng giữa các từ.
    - Dễ dàng mở rộng theo cụm (scale horizontally) nên phù hợp với lượng dữ liệu lớn.
    - Trong giải pháp của chúng ta, Elasticsearch sẽ index nội dung của các chunk để nhanh chóng tìm ra những chunk có chứa keyword trước khi truy vấn xuống vector database.

- Luồng keyword search mới như sau:
    - Khi vector hóa một chunk và lưu vào vector database, Elasticsearch sẽ lưu trữ dữ liệu gồm:
        - Document id
        - Chunk id
        - Nội dung chunk
    - Truyền query của người dùng vào Elasticsearch để tìm kiếm các document có chứa keyword
    - Từ các document tìm được, lấy ra các chunk tương ứng dựa vào chunk id
    - Truy vấn các chunk này trong vector database để lấy embedding và thực hiện các bước tiếp theo như bình thường