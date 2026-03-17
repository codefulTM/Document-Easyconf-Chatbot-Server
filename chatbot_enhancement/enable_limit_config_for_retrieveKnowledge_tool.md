# Plan: Cho phép Gemini gọi tool retrieveKnowledge với tham số `limit` (mặc định = 10)

## Mục tiêu

- Gemini có thể gọi tool `retrieveKnowledge` (hoặc tương tự) từ trong prompt / function call.
- Khi gọi tool, nó sẽ gửi thêm tham số `limit` với giá trị mặc định là `10` (nếu không được chỉ định).
- Cấu hình `limit` có thể được điều chỉnh (overridable) thông qua config chung hoặc meta prompt.

## Bước 1: Xác định tool `retrieveKnowledge` hiện tại

1. Tool function (OpenAI function declaration) được định nghĩa tại:
   - `Code/Easyconf-Chatbot-Server/src/chatbot/language/functions/english.ts` (`englishRetrieveKnowledgeDeclaration`).
   - Nó định nghĩa `name: "retrieveKnowledge"` và hiện tại chỉ có tham số `query`.
2. Phần xử lý thực thi tool được ở:
   - `Code/Easyconf-Chatbot-Server/src/chatbot/handlers/retrieveKnowledge.handler.ts` (hàm `execute()` hoặc tương tự).
3. Kiểm tra định nghĩa input của tool: tham số nào đang có, kiểu dữ liệu, bắt buộc hay tuỳ chọn.

## Bước 2: Thêm tham số `limit` vào định nghĩa tool và xử lý trong handler

1. Cập nhật schema/tool definition: thêm một trường `limit` (số nguyên) với mô tả rõ ràng.
2. Sửa hàm `execute()` trong `retrieveKnowledge.handler.ts` (hoặc đoạn code xử lý tool) để:
   - Đọc giá trị `limit` từ tham số đầu vào.
   - Nếu `limit` không được truyền, dùng giá trị mặc định `10`.
   - (Tùy chọn) Kiểm tra `limit` trong khoảng hợp lý (vd. 1..100) và ghi log nếu cần.
3. Thiết lập giá trị mặc định = 10 tại chỗ gọi (hoặc trong hàm triển khai tool) nếu giá trị không được truyền.

## Bước 3: Điều chỉnh prompt / function call của Gemini (tuỳ chọn)

- Nếu Gemini đã được cấu hình để gọi function / tool dựa vào schema (function declaration), thì không cần chỉnh prompt thủ công.
- Nếu bạn vẫn muốn đảm bảo Gemini truyền `limit`, có thể thêm ghi chú vào meta prompt nhưng không bắt buộc.
