# Batch Processing Orchestrator Service

## 1. Tổng quan (Overview)

### Mục đích chính

Dịch vụ `BatchProcessingOrchestratorService` đóng vai trò điều phối quá trình xử lý hàng loạt (batch processing) cho việc thu thập và cập nhật thông tin hội nghị. Nó quản lý việc xử lý các liên kết hội nghị, phân phối tác vụ cho các dịch vụ con và đảm bảo tài nguyên được giải phóng đúng cách.

### Vai trò trong kiến trúc hệ thống

- Lớp trung tâm điều phối giữa các thành phần xử lý link và thực thi tác vụ
- Quản lý vòng đời của các trang Playwright
- Điều phối luồng xử lý cho cả hai tác vụ chính: cập nhật hội nghị (update) và lưu hội nghị mới (save)
- Đảm bảo xử lý bất đồng bộ và quản lý tài nguyên hiệu quả

### Dependencies chính

- `ConfigService`: Cấu hình hệ thống
- `LoggingService`: Ghi log
- `FileSystemService`: Thao tác với hệ thống tệp
- `IConferenceLinkProcessorService`: Xử lý các liên kết hội nghị
- `IUpdateTaskExecutorService`: Thực thi tác vụ cập nhật
- `ISaveTaskExecutorService`: Thực thi tác vụ lưu mới
- `RequestStateService`: Quản lý trạng thái yêu cầu
- `InMemoryResultCollectorService`: Thu thập kết quả xử lý

## 2. Class chính: BatchProcessingOrchestratorService

### Constructor

```typescript
constructor(
    private readonly configService: ConfigService,
    loggingService: LoggingService,
    private readonly fileSystemService: FileSystemService,
    private readonly conferenceLinkProcessorService: IConferenceLinkProcessorService
)
```

### Các thuộc tính quan trọng

- `serviceBaseLogger`: Logger chính của service
- `batchesDir`: Thư mục lưu trữ các batch xử lý
- `tempDir`: Thư mục tạm
- `errorLogPath`: Đường dẫn file log lỗi
- `activeBatchSaves`: Tập hợp các tác vụ lưu đang hoạt động

### Các phương thức chính

#### 1. `processConferenceUpdate`

Xử lý cập nhật thông tin hội nghị hiện có.

**Tham số chính:**

- `browserContext`: Ngữ cảnh trình duyệt Playwright
- `conference`: Dữ liệu hội nghị cần cập nhật
- `parentLogger`: Logger từ tầng gọi
- `apiModels`: Các mô hình AI sử dụng
- `requestStateService`: Quản lý trạng thái yêu cầu
- `requestContainer`: Container dependency injection
- `resultCollector`: Thu thập kết quả xử lý
- `processedAcronymsSet`: Tập hợp các acronym đã xử lý

**Luồng xử lý:**

1. Tạo các trang mới cho các loại link (main, CFP, IMP)
2. Xử lý song song các loại link
3. Thu thập và xử lý kết quả
4. Ủy quyền cho `UpdateTaskExecutor` xử lý tiếp

#### 2. `processConferenceSave`

Xử lý lưu thông tin hội nghị mới.

**Tham số chính:**

- `browserContext`: Ngữ cảnh trình duyệt Playwright
- `conference`: Dữ liệu hội nghị mới
- `links`: Danh sách các liên kết cần xử lý
- `parentLogger`: Logger từ tầng gọi
- `apiModels`: Các mô hình AI sử dụng
- `requestStateService`: Quản lý trạng thái yêu cầu
- `requestContainer`: Container dependency injection
- `resultCollector`: Thu thập kết quả xử lý
- `processedAcronymsSet`: Tập hợp các acronym đã xử lý

**Luồng xử lý:**

1. Duyệt qua từng link trong danh sách
2. Xử lý từng link với timeout
3. Tạo batch dữ liệu để xử lý
4. Ủy quyền cho `SaveTaskExecutor` xử lý tiếp

#### 3. `awaitCompletion`

Đợi tất cả các tác vụ bất đồng bộ hoàn thành.

#### 4. `_closePages` (private)

Đóng các trang Playwright đang mở một cách an toàn.

## 3. Xử lý nghiệp vụ chính

### 1. Xử lý cập nhật hội nghị

- **Mục đích**: Cập nhật thông tin hội nghị hiện có
- **Luồng xử lý**:
  1. Tạo các trang mới cho các loại link (main, CFP, IMP)
  2. Xử lý song song các link với timeout
  3. Thu thập kết quả và xử lý lỗi
  4. Gọi `UpdateTaskExecutor` để xử lý tiếp

### 2. Xử lý lưu hội nghị mới

- **Mục đích**: Thu thập và lưu thông tin hội nghị mới
- **Luồng xử lý**:
  1. Duyệt qua từng link trong danh sách
  2. Xử lý từng link với timeout
  3. Tạo batch dữ liệu
  4. Gọi `SaveTaskExecutor` để xử lý tiếp

### 3. Quản lý tài nguyên

- Tự động đóng các trang Playwright sau khi sử dụng
- Xử lý lỗi và timeout
- Ghi log chi tiết quá trình xử lý

## 4. Tích hợp với các dịch vụ khác

- **ConferenceLinkProcessorService**: Xử lý các loại link khác nhau
- **UpdateTaskExecutor**: Thực thi tác vụ cập nhật
- **SaveTaskExecutor**: Thực thi tác vụ lưu mới
- **RequestStateService**: Theo dõi trạng thái yêu cầu
- **InMemoryResultCollector**: Thu thập kết quả xử lý

## 5. Xử lý lỗi

- Ghi log chi tiết khi có lỗi
- Đảm bảo giải phóng tài nguyên ngay cả khi có lỗi
- Hỗ trợ timeout cho các thao tác xử lý link
