1. **Tổng quan (Overview)**
   - **Mục đích chính của file/module:**
     - Thực hiện bước "determine" trong pipeline crawl conference: xác định trang chính thức của hội nghị từ kết quả API (Gemini), tải nội dung trang chính, và tìm/luu các liên kết quan trọng (Call for papers, Important dates) cùng nội dung tương ứng.
   - **Vai trò trong kiến trúc hệ thống:**
     - Là service orchestrator giữa kết quả trả về từ mô hình ngôn ngữ (Gemini) và các bước crawl/đọc trang thực tế bằng Playwright. Service này đảm bảo lấy được URL chính xác, chuẩn hoá liên kết, trích xuất nội dung văn bản/ảnh, và cập nhật các trường liên quan trong `BatchEntry`.
   - **Dependencies chính:**
     - `GeminiApiService` — gọi API để xác định/điều chỉnh link (determineLinks).
     - `FileSystemService` — đọc/ghi file text được trích xuất.
     - `IConferenceLinkProcessorService` — xử lý từng link (tải trang, trích xuất text/images và lưu).
     - `playwright` (`Page`, `BrowserContext`) — thao tác trình duyệt để tải nội dung.
     - `ConfigService` — để xử lý các hành vi tuỳ theo môi trường (ví dụ fallback đọc file trong dev).
     - Các util: `normalizeAndJoinLink`, `withOperationTimeout`, `getErrorMessageAndStack`.

2. **Class/Component chính**
   - **Tên class và chức năng:**
     - `ConferenceDeterminationService` — thực hiện logic xác định trang chính thức và lưu các nội dung liên quan; triển khai `IConferenceDeterminationService`.
   - **Constructor và injected dependencies:**
     - Injected:
       - `FileSystemService` — đọc/ghi file nội dung trang.
       - `GeminiApiService` — gọi API determineLinks (API 1 / API 2).
       - `'IConferenceLinkProcessorService'` — service xử lý và lưu nội dung của một link cụ thể (`processAndSaveGeneralLink`).
       - `ConfigService` — đọc cấu hình (vd. `isProduction`) để điều chỉnh fallback.
   - **Properties và methods quan trọng:**
     - Methods công khai/quan trọng:
       - `determineAndProcessOfficialSite(api1ResponseText, originalBatch, batchIndexForApi, browserContext, determineModel, parentLogger): Promise<BatchEntry[]>` — hàm entrypoint; phân tích JSON từ API1, chuẩn hoá link, quyết định nhánh xử lý (match trong batch hay không), và trả về `BatchEntry` đã cập nhật.
     - Methods helper nội bộ:
       - `fetchAndProcessOfficialSiteInternal(page, officialWebsiteUrl, acronym, title, batchIndex, logger)` — tải trang chính, gọi `linkProcessorService.processAndSaveGeneralLink` để lưu text/images, trả về object { finalUrl, textPath, textContent, imageUrls } hoặc `null` khi thất bại.
       - `handleDetermineMatchInternal(browserContext, matchingEntry, officialWebsiteNormalized, cfpLinkFromApi1, impLinkFromApi1, batchIndex, logger)` — xử lý khi official website trùng với một entry trong batch: mở các trang CFP/IMP riêng (dùng pages riêng), xử lý song song kèm timeout, và gán kết quả vào `matchingEntry`.
       - `handleDetermineNoMatchInternal(page, officialWebsiteNormalizedFromApi1, primaryEntryToUpdate, batchIndex, determineModelForApi2, logger)` — khi không tìm thấy match trong batch: tải trang chính, lấy nội dung dùng cho lần gọi API thứ hai (API2), parse kết quả API2, chuẩn hoá link tương đối, tải và lưu CFP/IMP từ API2.
     - Các hành vi nổi bật:
       - Áp dụng timeout cho các tác vụ nặng (`withOperationTimeout`) để tránh block lâu.
       - Sử dụng `linkProcessorService.processAndSaveGeneralLink` làm điểm duy nhất trích xuất & lưu nội dung (text + image URL list).
       - Ghi log chi tiết (pino) với child logger cho từng bước.

3. **API Endpoints (nếu là Controller)**
   - Đây là một **Service**, không phải Controller — không có route/endpoint HTTP nào được khai báo trong file này.

4. **Business Logic (LIÊN QUAN ĐẾN CRAWL DỮ LIỆU)**
   - **Các methods chính liên quan đến crawl dữ liệu:**
     - `determineAndProcessOfficialSite` — điều phối tổng thể: parse API1, normalize URL, quyết định nhánh xử lý, gọi các helper để tải/luu nội dung, gọi API2 khi cần.
     - `fetchAndProcessOfficialSiteInternal` — thực thi crawl cho trang "main" (tạo file base name, gọi linkProcessor để trích xuất text/images và lưu), trả về cả URL thực tế sau redirect.
     - `handleDetermineMatchInternal` — khi official site từ API1 khớp entry trong batch, mở các Page riêng để xử lý CFP & IMP (song song), và lưu path/content vào `BatchEntry`.
     - `handleDetermineNoMatchInternal` — khi không khớp batch: tải trang chính, lấy content làm input cho lần gọi API2 (determineLinks thứ hai), parse JSON API2, chuẩn hoá các link theo `actualFinalUrl`, sau đó tải & lưu CFP/IMP theo kết quả API2.
   - **Flow xử lý dữ liệu (tóm tắt):**
     1. Nhận `api1ResponseText` (JSON string) + `originalBatch`.
     2. Parse JSON từ API1; lấy trường "Official Website"; normalize URL.
     3. So sánh với `originalBatch` (normalize entry.mainLink) để tìm match.
        - Nếu có match: gọi `handleDetermineMatchInternal` — xử lý CFP/IMP trên các page riêng (với timeout), cập nhật `matchingEntry`.
        - Nếu không match: mở page cho URL từ API1, gọi `fetchAndProcessOfficialSiteInternal` để trích xuất nội dung trang chính; dùng nội dung này làm input cho `geminiApiService.determineLinks` (API2) để nhận CFP/IMP; normalize các link trả về từ API2 theo finalUrl; tải và lưu CFP/IMP.
     4. Lưu kết quả vào `BatchEntry` (các trường chính: `mainLink`, `conferenceTextPath`, `conferenceTextContent`, `cfpLink`, `cfpTextPath`, `cfpTextContent`, `impLink`, `impTextPath`, `impTextContent`, `imageUrls`).
     5. Trả về mảng `BatchEntry[]` (thường là mảng với 1 phần tử đã cập nhật) hoặc `mainLink = "None"` khi lỗi/không tìm được.
   - **Integration với external services:**
     - Gemini API (`GeminiApiService.determineLinks`) — gọi 2 lần: API1 (đã được gọi bên ngoài và trả về `api1ResponseText`) và API2 (khi cần, dùng nội dung trang chính làm input) để nhận json chứa các link như "Call for papers link" và "Important dates link".
     - Playwright (`BrowserContext`, `Page`) — mở page, thao tác, đọc URL cuối cùng (redirects), và truyền page vào `linkProcessorService` để trích xuất nội dung.
     - File system (`FileSystemService`) — đọc lại file nội dung nếu cần (fallback trong dev), và lưu các file text được trích xuất.
     - `IConferenceLinkProcessorService` — điểm trung tâm để crawl/parse/lưu nội dung của từng link (trả về path, content, imageUrls).

---
Ghi chú ngắn:
- File source: [Document-Easyconf-Chatbot-Server/src/services/batchProcessing/conferenceDetermination.service.ts](Document-Easyconf-Chatbot-Server/src/services/batchProcessing/conferenceDetermination.service.ts)
- File tóm tắt này được tạo tự động dựa trên nội dung của `conferenceDetermination.service.ts` và tập trung vào các phần liên quan tới crawl/link determination.
