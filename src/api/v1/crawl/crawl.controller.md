# Tổng quan về crawl.controller.ts

## 1. Tổng quan
- **Mục đích chính**: Điều phối các yêu cầu crawl dữ liệu hội nghị và tạp chí
- **Vai trò**: Là controller xử lý các API liên quan đến crawl dữ liệu, đóng vai trò trung gian giữa client và các service xử lý crawl
- **Dependencies chính**:
  - `CrawlOrchestratorService`: Điều phối quá trình crawl
  - `LoggingService`: Ghi log hệ thống
  - `ConfigService`: Quản lý cấu hình
  - `LogAnalysisCacheService`: Quản lý cache phân tích log
  - `CrawlProcessManagerService`: Quản lý tiến trình crawl
  - `RequestStateService`: Theo dõi trạng thái request

## 2. Class/Component chính
### `handleCrawlConferences`
- **Chức năng**: Xử lý yêu cầu crawl thông tin hội nghị
- **Constructor và dependencies**:
  - Sử dụng dependency injection để nhận các service cần thiết
  - Tạo child container riêng cho mỗi request
- **Các phương thức chính**:
  - `performCrawl`: Thực hiện quá trình crawl dữ liệu
  - `cleanupRequestResources`: Dọn dẹp tài nguyên sau khi xử lý
- **Step by step**:
  - Tạo child container mới từ root container
  - Khởi tạo logging service và tạo request ID duy nhất
  - Thiết lập logger cho request

  - Lấy các tham số từ request body:
    - `description`: Mô tả ngắn về yêu cầu crawl
    - `items`: Mảng chứa danh sách các hội nghị cần crawl
    - `models`: Cấu hình model AI cho các bước xử lý (determineLinks, extractInfo, extractCfp)
    - `recordFile`: Cờ ghi lại dữ liệu vào file (true/false)
  - Xác định chế độ thực thi (sync/async) từ query parameter
  - Ghi log thông tin request
  - Khởi tạo và lấy các service cần thiết
  - Định nghĩa hàm performCrawl:
    - Lấy thông tin dataSource từ query parameters. Nếu dataSource khác 'client', ném lỗi
    - Khởi tạo đối tượng parsedApiModels với giá trị mặc định
    - Nếu có modelsFromPayload từ request:
      - Kiểm tra từng key trong modelsFromPayload
      - Nếu giá trị là 'tuned' hoặc 'non-tuned', lưu vào parsedApiModels
      - Nếu không hợp lệ, thêm vào danh sách lỗi
    - Nếu modelsFromPayload không hợp lệ, sử dụng mô hình mặc định
    - Kiểm tra conferenceList có phải là mảng không
    - Nếu mảng rỗng, ghi log cảnh báo và trả về mảng rỗng
    - Gọi phương thức run của orchestrator với các tham số:
      - Danh sách hội nghị
      - Logger
      - Các mô hình API
      - ID của batch request
      - Dịch vụ trạng thái request
      - Container của request
    - Trả về mảng processedResults chứa dữ liệu đã xử lý
  - Nếu executionMode là 'sync' thì gọi hàm performCrawl đồng bộ, nếu không phải 'sync' thì xử lý bất đồng bộ

### `handleStopCrawl`
- **Chức năng**: Dừng quá trình crawl đang chạy
- **Tham số**: `batchRequestId` - ID của request cần dừng
- **Step by step**:
  - Lấy thông tin batchRequestId từ phần thân của request
  - Nếu không có batchRequestId thì báo lỗi 400 và kết thúc hàm
  - Lấy instance của CrawlProcessManagerService từ container và gọi phương thức requestStop của instance này. Nếu thành công -> Trả về mã 200

### `handleCrawlJournals`
- **Chức năng**: Xử lý yêu cầu crawl thông tin tạp chí
- **Mô tả**: Tương tự như `handleCrawlConferences` nhưng dành cho dữ liệu tạp chí


## 3. API Endpoints
### POST `/crawl-conferences`
- **Mục đích**: Khởi tạo quá trình crawl thông tin hội nghị
- **Tham số**:
  - `mode`: `sync` (đồng bộ) hoặc `async` (bất đồng bộ, mặc định)
  - `description`: Mô tả yêu cầu
  - `items`: Danh sách các hội nghị cần crawl
  - `models`: Cấu hình model AI sử dụng
- **Response**:
  - Đồng bộ (sync): Trả về kết quả ngay lập tức
  - Bất đồng bộ (async): Trả về 202 Accepted với batchRequestId

### POST `/stop-crawl`
- **Mục đích**: Dừng quá trình crawl đang chạy
- **Tham số**:
  - `batchRequestId`: ID của request cần dừng

### POST `/crawl-journals`
- **Mục đích**: Khởi tạo quá trình crawl thông tin tạp chí
- **Tham số**: Tương tự như `/crawl-conferences`

## 4. Business Logic
### Xử lý crawl dữ liệu
1. **Khởi tạo**:
   - Tạo request ID duy nhất
   - Khởi tạo logger cho request
   - Xác thực và chuẩn hóa dữ liệu đầu vào

2. **Xử lý chính**:
   - Phân tích và xác thực cấu hình model AI
   - Gọi `CrawlOrchestratorService` để thực hiện crawl
   - Xử lý kết quả và lỗi

3. **Dọn dẹp**:
   - Giải phóng tài nguyên
   - Đóng logger
   - Xóa cache nếu cần

### Xử lý lỗi
- Bắt và ghi log chi tiết các lỗi xảy ra
- Trả về thông báo lỗi phù hợp cho client
- Đảm bảo giải phóng tài nguyên kể cả khi có lỗi

### Tích hợp với services khác
- **LoggingService**: Ghi log chi tiết quá trình xử lý
- **CrawlOrchestratorService**: Điều phối quá trình crawl
- **LogAnalysisCacheService**: Quản lý cache kết quả phân tích
- **RequestStateService**: Theo dõi trạng thái request