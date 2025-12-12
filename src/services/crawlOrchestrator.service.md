# CrawlOrchestrator Service

## 1. Tổng quan
- **Mục đích**: Điều phối toàn bộ quy trình crawl và xử lý dữ liệu hội nghị, quyết định luồng xử lý dựa trên trạng thái request.
- **Vai trò**: Là lớp trung tâm trong kiến trúc xử lý crawl, kết nối các dịch vụ con và đảm bảo xử lý đồng thời an toàn.
- **Dependencies chính**:
  - `ConfigService`: Quản lý cấu hình ứng dụng
  - `LoggingService`: Ghi log hệ thống
  - `GlobalConcurrencyManagerService`: 
    - **Mục đích**: Kiểm soát số lượng request xử lý đồng thời trên toàn hệ thống
    - **Cơ chế hoạt động**: Sử dụng Semaphore để giới hạn số lượng task chạy đồng thời
    - **Lợi ích**: 
      - Tránh quá tải hệ thống khi có nhiều request cùng lúc
      - Đảm bảo tài nguyên được phân bổ hợp lý giữa các request
      - Có thể cấu hình giới hạn đồng thời tùy theo khả năng của hệ thống
  - `TaskQueueService`: 
    - **Mục đích**: Quản lý hàng đợi các tác vụ xử lý hội nghị trong một request
    - **Cơ chế hoạt động**: 
      - Sử dụng hàng đợi ưu tiên (priority queue)
      - Hỗ trợ xử lý đồng thời với số lượng worker có thể cấu hình
      - Tự động xử lý lỗi và retry
    - **Lợi ích**:
      - Đảm bảo các task trong cùng request được xử lý tuần tự hoặc song song theo cấu hình
      - Giúp kiểm soát tài nguyên cấp độ request
      - Dễ dàng theo dõi tiến trình xử lý
  - `FileSystemService`: Xử lý thao tác file
  - `GeminiApiService`: Tương tác với API Gemini
  - `RequestStateService`: Quản lý trạng thái request

## 2. Lớp chính: CrawlOrchestratorService

### Constructor
```typescript
constructor(
  private configService: ConfigService,
  private loggingService: LoggingService,
  private apiKeyManager: ApiKeyManager,
  private fileSystemService: FileSystemService,
  private htmlPersistenceService: HtmlPersistenceService,
  private batchProcessingOrchestratorService: BatchProcessingOrchestratorService,
  private geminiApiService: GeminiApiService,
  private globalConcurrencyManager: GlobalConcurrencyManagerService
)
```

### Các thuộc tính quan trọng
- `configApp`: Cấu hình ứng dụng
- `baseLogger`: Logger cơ sở
- `conferenceProcessingTimeoutMs`: Thời gian chờ xử lý mỗi hội nghị (5 phút)

## 3. Phương thức chính

### `run(conferenceList, parentLogger, apiModels, batchRequestId, requestStateService, requestContainer)`

**Mục đích**: Thực thi luồng crawl và xử lý dữ liệu chính.

**Tham số**:
- `conferenceList`: Mảng dữ liệu hội nghị cần xử lý
- `parentLogger`: Logger từ request
- `apiModels`: Cấu hình model AI cho các giai đoạn xử lý
- `batchRequestId`: ID duy nhất của batch đang xử lý
- `requestStateService`: Quản lý trạng thái request
- `requestContainer`: DI container của request

**Step by step**:
1. **Khởi tạo môi trường**
   - Tạo logger riêng cho request hiện tại với context cụ thể
   - Lấy các service cần thiết từ request container:
     - `requestTaskQueue`: Quản lý hàng đợi tác vụ
     - `resultCollector`: Thu thập kết quả trong bộ nhớ
     - `resultProcessingService`: Xử lý kết quả cuối cùng
   - Kiểm tra chế độ ghi file thông qua `requestStateService`
   - Ghi log thông tin bắt đầu xử lý

2. **Chuẩn bị dữ liệu**
   - Tạo Set `processedAcronymsForThisRequest` để theo dõi các hội nghị đã xử lý
   - Reset trạng thái các service:
     - Xóa kết quả cũ từ `resultCollector`
     - Reset trạng thái `htmlPersistenceService`
     - Chuẩn bị thư mục đầu ra
     - Khởi tạo Gemini API service

3. **Tạo và lên lịch các tác vụ**
   - Với mỗi hội nghị trong danh sách:
     - Chuẩn hóa dữ liệu (xóa các ký tự đặc biệt trong Title và Acronym)
     - Tạo hàm xử lý bất đồng bộ cho từng hội nghị
     - Mỗi task được bọc trong `globalConcurrencyManager` để kiểm soát đồng thời
     - Xử lý timeout và bắt lỗi cho từng task

4. **Thực thi đồng thời**
   - Thêm tất cả task vào hàng đợi
   - Ghi log thời gian bắt đầu xử lý
   - Chờ tất cả task hoàn thành với `requestTaskQueue.onIdle()`
   - Ghi log thời gian hoàn thành

5. **Xử lý kết quả**
   - Chờ các tác vụ lưu trữ dữ liệu hoàn tất
   - Xử lý kết quả tùy theo chế độ:
     - Ghi file: Sử dụng `resultProcessingService.processOutput()`
     - Bộ nhớ: Lấy kết quả từ `resultCollector` và xử lý qua `resultProcessingService.processInMemoryData()`

6. **Xử lý lỗi**
   - Bắt và ghi log tất cả các lỗi phát sinh
   - Lưu lại lỗi để xử lý sau

7. **Dọn dẹp**
   - Trong môi trường không production: xóa các file tạm
   - Ghi log tổng kết quá trình xử lý
   - Nếu có lỗi: ném lỗi để xử lý ở tầng trên

8. **Trả kết quả**
   - Trả về mảng `allProcessedData` chứa kết quả đã xử lý
   - Đảm bảo tất cả tài nguyên đã được giải phóng trước khi kết thúc