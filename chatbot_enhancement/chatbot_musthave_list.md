## ✅ RAG (Retrieval-Augmented Generation)

### Must have

- **Filter kết quả RAG trả về** (lọc ra hội nghị phù hợp / loại bỏ kết quả “rác” trước khi trả cho user)
- **Cho phép cấu hình số lượng tài liệu retrieve** (hiện chỉ lấy 10, cần mở tham số để tăng/giảm)
- **Không trả quá nhiều nội dung (limit output, ví dụ <= 500 từ)** để tránh quá tải và giảm token cost
- **Giải quyết vấn đề so sánh toàn bộ câu với entity** (nên ưu tiên match substring / N‑gram / entity extraction thay vì global string similarity)
- **Bổ sung stopword list đủ cho nhiều ngôn ngữ** (bổ sung thêm stopwords ngoài tiếng Việt/Anh)
- **Cơ chế clean-up chunk + vector đồng bộ trong giao diện Admin** (giao diện/điều khiển để dọn DB chunk và vector store tương ứng khi có lỗi)
- **Cơ chế chuyển sang API Key khác khi hết quota** (hiện có nhiều key nhưng chưa tự động chuyển khi một key bị lỗi quota)

### Should have

- **Phân biệt list từ getConferences vs list hiển thị ở prompt** (tránh nhầm “hội nghị thứ X”)
- **Có thể cấu hình/hiển thị rõ ràng “source” (nguồn tài liệu) cho từng câu trả lời**

### Could have

- **Tự động nhận diện các trường hợp nên dùng RAG thay vì getConferences**
- **Re-rank kết quả theo relevance (semantic re-ranking)**
- **Cơ chế “fallback” nếu RAG retrieval quá yếu (chuyển sang trả lời template / “tôi không biết”)**

### Won’t have (khó implement / không cần ưu tiên)

- **Triển khai RAG full‑scale như hệ thống enterprise có nhiều layer (vector db phân tán, sharding, multi‑tenant)**

---

## 🤖 Chatbot (Agent) nói chung

### Must have

- **Nhớ lịch sử chat (context/state)** để xử lý refer (đại từ, “nó”, “hội nghị lúc trước”) và tránh lặp nội dung
- **Giảm tình trạng trả lặp lại cùng nội dung giữa các turn** (tránh lặp câu, lặp ý)
- **Quy tắc rõ ràng khi dùng getConferences vs RAG** (để không “dại dột” dùng getConferences khi RAG phù hợp hơn)
- **Hỗ trợ host agent non-streaming** (kiểm tra & test, tránh chỉ chạy streaming)
- **Stopword list đầy đủ** (Tiếng Việt + Tiếng Anh + các từ hay được dùng trong query học thuật)

### Should have

- **System instruction hỗ trợ đa ngôn ngữ** (ít nhất Việt/Anh — hiện chưa cập nhật đủ)
- **Cơ chế phân biệt “danh sách getConferences” vs “danh sách trong tin nhắn”** (để hiểu “hội nghị thứ x” trong ngữ cảnh nào)
- **Tối ưu trải nghiệm khi user hỏi lại hoặc mở rộng** (ví dụ “tiếp theo” / “nói thêm về…”)
- **Giới hạn output (word/token limit)** để tránh model trả quá dài
- **Tăng robustness khi hệ thống gặp lỗi (fallback messages, gracefull degradation)**

### Could have

- **Tích hợp LangGraph vào handler của host agent** (nếu mang lại lợi ích orchestration mà không làm quá tải API)
- **Mô-đun “test non-streaming” tự động/benchmark** để validate cả 2 chế độ
- **UI/UX cho user chọn “dùng RAG” vs “dùng DB search”** khi cần (một tuỳ chọn nâng cao)

### Won’t have

- **Xây hệ thống multi-agent quá phức tạp nếu không cần (trừ khi có lý do rất mạnh)**
- **Triển khai hoàn toàn theo kiểu “agent orchestration” với quá nhiều layer nếu vượt giới hạn rate (20 RPD)**
