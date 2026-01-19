# Vấn đề hiện tại
- Chatbot bị phụ thuộc quá nhiều vào từ khóa trong truy vấn của người dùng. 
- Ví dụ 1:
    - Search hội nghị "Conference 1" thì hiển thị hội nghị
    - Search hội nghị "Conferrence 1" thì không hiển thị hội nghị

- Ví dụ 2:
    - Search hội nghị "International Conference" thì hiển thị hội nghị
    - Search hội nghị "Worldwide Conference" thì không hiển thị hội nghị

# Các giải pháp đề xuất
- Giải pháp 1: Fuzzy search bằng n-gram/trigram (xử lý sai chính tả)
    - Chia chuỗi thành n-gram(thường là trigram)
    - So khớp theo độ giống nhau

- Giải pháp 2: Spell correction / query normalization (sửa truy vấn đầu vào)
    - Chuẩn hóa query trước khi search
        - Lowercase
        - Xóa dấu câu
        - Sửa lỗi chính tả

- Giải pháp 3: Synonym Expansion (từ đồng nghĩa)
    - Map các từ đồng nghĩa về cùng một tập
    - Mở rộng truy vấn khi search, nghĩa là hệ thống sẽ tự động bổ sung hoặc thay thế truy vấn đó bằng tất cả các từ đồng nghĩa liên quan. Nhờ đó, kết quả tìm kiếm sẽ bao phủ rộng hơn, không bị giới hạn bởi đúng từ khóa gốc mà người dùng nhập.
    - Ví dụ:
        - International ↔ Worldwide ↔ Global
        - Conference ↔ Symposium ↔ Congress

- Giải pháp 4: Semantic Search bằng Embedding
    - Chuyển query & dữ liệu thành vector embedding
    - So khớp theo độ gần ngữ nghĩa

- Giải pháp 5: Hybrid Search(Keyword + Fuzzy + Semantic)
    - Kết hợp: Exact match, fuzzy/trigram, semantic search
    - Ví dụ chiến lược: 
        - Final score = 0.5 * keyword_score + 0.3 * fuzzy_score + 0.2 * semantic_score

# So sánh điểm mạnh và điểm yếu các giải pháp
- Giải pháp 1: Fuzzy search bằng n-gram/trigram
    - Ưu điểm:
        - Tìm được khi người dùng gõ sai chính tả -> Phù hợp cho người gõ nhanh, gõ ẩu
        - Không cần hiểu ngữ nghĩa, chỉ so sánh chuỗi ký tự -> Dễ làm, dễ debug, dễ kiểm soát
        - Hiệu quả với tên riêng, mã, tiêu đề. Vì các trường này thường không có nhiều biến thể ngữ nghĩa, chỉ cần khớp gần đúng ký tự là đủ. Sai sót chủ yếu là do lỗi gõ phím, không phải do cách diễn đạt khác nhau. Fuzzy search sẽ sửa các lỗi này mà không cần hiểu ý nghĩa sâu xa của từ.
        - Triển khai nhanh: DB đã hỗ trợ sẵn(PostgreSQL, Elasticsearch), ít phụ thuộc hạ tầng, nghĩa là chỉ cần hệ quản trị cơ sở dữ liệu phổ biến (như PostgreSQL, Elasticsearch) là đã có thể dùng được tính năng fuzzy search mà không cần xây dựng thêm dịch vụ hoặc hệ thống phức tạp bên ngoài. Không cần đầu tư thêm server, microservice hoặc các giải pháp AI/phân tích ngữ nghĩa riêng biệt.
    - Nhược điểm
        - Không hiểu từ đồng nghĩa. Ví dụ: International ≠ Worldwide, Conference ≠ Symposium. Nếu gõ khác từ thì không ra kết quả
        - Không hiểu ngữ nghĩa, chỉ biết mức độ giống nhau về mặt ký tự giữa hai chuỗi. Ví dụ: "Data Science Conference" và "AI Symposium" có thể nói về cùng một chủ đề nhưng không có nhiều ký tự giống nhau, nên fuzzy search sẽ không nhận ra mối liên hệ này.
        - Tốn bộ nhớ: một từ sinh ra nhiều n-gram, index phình to
        - Kết quả có thể khá giống nhưng sai: Có những từ giống nhau về chữ nhưng khác nghĩa hoàn toàn như cat vs catalog
        - Query càng dài thì càng nhiễu, độ chính xác giảm

- Giải pháp 2: Spell correction / query normalization
    - Ưu điểm:
        - Sửa lỗi gõ sai rất hiệu quả. Giải quyết tốt lỗi đánh máy, gõ nhanh, gõ nhầm.
        - Giữ nguyên hệ thống search phía sau: Không thay đổi DB, không đổi engine tìm kiếm, chỉ xử lý đầu vào.
        - Kết quả chính xác hơn: Khi query được sửa, việc search sẽ đúng với từ gốc, ít ra các kết quả na ná nhưng sai
        - Dễ hiểu, dễ giải thích
    - Nhược điểm:
        - Không hiểu từ đồng nghĩa. Ví dụ: International ≠ Worldwide, Conference ≠ Congress. Không giải quyết bài toán khác từ nhưng cùng nghĩa
        - Phụ thuộc vào từ điển: Dễ sửa sai nếu từ không có trong từ điển. Ví dụ: Nếu người dùng gõ từ mới, tên riêng, hoặc từ lạ không có trong từ điển, hệ thống có thể sửa sai hoặc không sửa được, dẫn đến kết quả tìm kiếm không được chính xác
        - Có thể sửa sai ý người dùng. Người dùng muốn A, hệ thống hiểu thành B. Ví dụ: Apple -> Apply, Meta -> Metal
        - Không xử lý tốt câu dài: Với câu dài, có nhiều từ, việc sửa từng từ có thể làm thay đổi ý nghĩa tổng thể của câu, dẫn đến kết quả tìm kiếm không đúng với ý định ban đầu của người dùng.

- Giải pháp 3: Synonym Expansion
    - Ưu điểm:
        - Giải quyết tốt bài toán "khác từ nhưng cùng nghĩa"
        - Dễ kiểm soát kết quả: Có thể biết rõ từ nào được mở rộng, không sinh kết quả "ảo"
        - Không cần AI, không cần học máy, chỉ cần từ điển -> Dễ triển khai và bảo trì
        - Rất hiệu quả trong domain rõ ràng: Hội nghị, Y tế, Giáo dục, Thương mại,... Bởi vì từ vựng ổn định, ít biến đổi -> hiệu quả cao
        - Tăng trải nghiệm người dùng: Gõ từ nào cũng ra kết quả mong muốn. Người dùng không cần "đoán đúng từ hệ thống"
    - Nhược điểm:
        - Phải xây dựng và bảo trì thủ công, nghĩa là đội phát triển phải tự tay tạo ra danh sách các nhóm từ đồng nghĩa(synonym list) và cập nhật nó khi có từ mới, từ chuyên ngành, hoặc khi phát hiện thiếu sót. Hệ thống không tự động học hoặc cập nhật các từ đồng nghĩa này.
        - Không mở rộng được hết mọi trường hợp: Từ đồng nghĩa có thể rất đa dạng và phong phú, việc liệt kê hết tất cả các từ đồng nghĩa có thể rất khó khăn, đặc biệt là trong các lĩnh vực rộng lớn hoặc phức tạp.
        - Dễ gây nhiễu nếu map sai: Có thể trả về kết quả không đúng mong muốn nếu từ gần nghĩa nhưng không giống hoàn toàn hoặc ngữ cảnh khác nhau. Ví dụ: Nếu map "bank" (ngân hàng) với "river bank" (bờ sông) vào cùng một nhóm, khi người dùng tìm "bank" với ý là ngân hàng, hệ thống có thể trả về kết quả liên quan đến bờ sông, gây nhiễu và làm giảm độ chính xác của tìm kiếm.
        - Khó mở rộng khi dữ liệu lớn: Domain càng rộng -> synonym càng nhiều -> khó quản lý

- Giải pháp 4: Semantic Search bằng Embedding
    - Ưu điểm:
        - Hiểu ngữ nghĩa, không phụ thuộc từ khóa
        - Giải quyết tốt bài toán từ đồng nghĩa và diễn đạt khác nhau
        - Phù hợp với câu hỏi tự nhiên, ví dụ như "Hội nghị quốc tế về AI tổ chức khi nào?"
        - Không cần xây dựng synonym thủ công: Không cần liệt kê từng cặp từ, mô hình đã "học sẵn" -> Tiết kiệm công sức bảo trì
    - Nhược điểm:
        - Kết quả khó đoán, khó giải thích: Không biết vì sao 2 kết quả lại match, khó debug
        - Có thể trả về kết quả chỉ "hơi liên quan" nhưng không chính xác 100%
        - Tốn tài nguyên hơn search truyền thống: Phải lưu vector, phải chạy similarity search
        - Không giỏi xử lý sai chính tả nhỏ: Thường vẫn cần spell correction / trigram hỗ trợ
        - Phụ thuộc vào chất lượng model

- Giải pháp 5: Hybrid Search(Keyword + Fuzzy + Semantic)
    - Ưu điểm:
        - Giải quyết được hầu hết các trường hợp tìm kiếm:
            - Gõ đúng -> ra kết quả
            - Gõ sai -> vẫn ra
            - Khác từ nhưng cùng nghĩa -> vẫn ra
        - Kết quả vừa chính xác vừa thông minh
        - Trải nghiệm người dùng tốt nhất
        - Giảm rủi ro bỏ sót dữ liệu quan trọng: Một kỹ thuật fail -> kỹ thuật khác bù -> Ít khi "search không ra gì"
    - Nhược điểm:
        - Kiến trúc phức tạp hơn 
        - Khó điều chỉnh trọng số các thành phần. Ví dụ: Keyword bao nhiêu điểm, Semantic bao nhiêu điểm, Fuzzy bao nhiêu điểm? -> Cần thử nghiệm, tinh chỉnh nhiều lần
        - Tốn tài nguyên hơn: vừa index text, vừa index vector
        - Khó debug khi kết quả không như ý: Không rõ lỗi do keyword, fuzzy hay semantic
        - Overkill với hệ thống nhỏ: Dữ liệu ít, query đơn giản -> không cần thiết phải dùng quá nhiều kỹ thuật phức tạp
        
# Lựa chọn giải pháp
- Chọn giải pháp 5: Hybrid Search(Keyword + Fuzzy + Semantic)
- Lý do:
    - Kết hợp ưu điểm của các phương pháp: sửa lỗi chính tả, nhận biết đồng nghĩa, hiểu ngữ nghĩa, khớp gần đúng.
    - Đảm bảo người dùng gõ đúng, gõ sai, dùng từ đồng nghĩa hay diễn đạt tự nhiên đều có thể ra kết quả.
    - Giảm rủi ro bỏ sót dữ liệu quan trọng: nếu một kỹ thuật không match thì kỹ thuật khác có thể bù lại.
    - Trải nghiệm người dùng tốt nhất, phù hợp cho hệ thống chatbot thực tế với dữ liệu đa dạng.