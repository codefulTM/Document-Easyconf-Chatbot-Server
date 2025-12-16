# Conference Processor Service

## 1. Tổng quan (Overview)

### Mục đích chính

Dịch vụ `ConferenceProcessorService` đóng vai trò trung tâm trong việc xử lý thông tin hội nghị, quyết định luồng xử lý phù hợp (cập nhật hoặc thu thập mới) dựa trên dữ liệu đầu vào, và điều phối quá trình xử lý thông qua các dịch vụ con.

### Vai trò trong kiến trúc hệ thống

- Lớp điều phối chính cho việc xử lý thông tin hội nghị
- Quyết định luồng xử lý (update/save) dựa trên dữ liệu đầu vào
- Tích hợp với các dịch vụ tìm kiếm và lưu trữ
- Quản lý vòng đời của quá trình xử lý một hội nghị

### Dependencies chính

- `ConfigService`: Cấu hình hệ thống
- `LoggingService`: Ghi log
- `GoogleSearchService`: Tìm kiếm thông tin hội nghị
- `HtmlPersistenceService`: Xử lý lưu trữ và cập nhật dữ liệu
- `FileSystemService`: Thao tác với hệ thống tệp
- `RequestStateService`: Quản lý trạng thái yêu cầu
- `InMemoryResultCollectorService`: Thu thập kết quả xử lý

## 2. Class chính: ConferenceProcessorService

### Constructor

```typescript
constructor(
    private configService: ConfigService,
    private loggingService: LoggingService,
    private googleSearchService: GoogleSearchService,
    private htmlPersistenceService: HtmlPersistenceService,
    private fileSystemService: FileSystemService
)
```

### Các thuộc tính quan trọng

- `searchQueryTemplate`: Mẫu câu truy vấn tìm kiếm
- `year1`, `year2`, `year3`: Các năm sử dụng cho tìm kiếm (năm trước, năm hiện tại, năm sau)
- `unwantedDomains`: Danh sách các domain không mong muốn
- `skipKeywords`: Từ khóa cần bỏ qua
- `maxLinks`: Số lượng liên kết tối đa cần xử lý
- `serviceBaseLogger`: Logger chính của service

### Các phương thức chính

#### 1. `process`

Phương thức chính xử lý thông tin hội nghị.

**Tham số chính:**

- `conference`: Dữ liệu hội nghị cần xử lý
- `taskIndex`: Chỉ số của tác vụ
- `parentLogger`: Logger từ tầng gọi
- `apiModels`: Các mô hình AI sử dụng
- `batchRequestId`: ID của batch request
- `requestStateService`: Quản lý trạng thái yêu cầu
- `requestContainer`: Container dependency injection
- `resultCollector`: Thu thập kết quả xử lý
- `processedAcronymsSet`: Tập hợp các acronym đã xử lý

**Luồng xử lý:**

1. Xác định loại xử lý (update hoặc save) dựa trên dữ liệu đầu vào
2. Nếu có đủ thông tin liên kết (mainLink, cfpLink, impLink):
   - Thực hiện luồng cập nhật (update flow)
   - Gọi `htmlPersistenceService.processUpdateFlow`
3. Nếu không đủ thông tin liên kết:
   - Thực hiện tìm kiếm Google để lấy liên kết
   - Lọc và giới hạn số lượng kết quả
   - Thực hiện luồng lưu mới (save flow)
   - Gọi `htmlPersistenceService.processSaveFlow`
4. Xử lý kết quả và ghi log phù hợp

## 3. Xử lý nghiệp vụ chính

### 1. Xử lý cập nhật (Update Flow)

- **Điều kiện kích hoạt**: Khi hội nghị đã có đầy đủ các liên kết (mainLink, cfpLink, impLink)
- **Quy trình**:
  1. Tạo đối tượng `ConferenceUpdateData` từ dữ liệu đầu vào
  2. Gọi `htmlPersistenceService.processUpdateFlow` để xử lý cập nhật
  3. Ghi log kết quả cập nhật

### 2. Xử lý lưu mới (Save Flow)

- **Điều kiện kích hoạt**: Khi hội nghị chưa có đầy đủ thông tin liên kết
- **Quy trình**:
  1. Tạo câu truy vấn tìm kiếm dựa trên tiêu đề và từ viết tắt hội nghị
  2. Thực hiện tìm kiếm Google thông qua `googleSearchService`
  3. Lọc kết quả tìm kiếm loại bỏ các domain không mong muốn
  4. Giới hạn số lượng kết quả theo cấu hình
  5. Lưu kết quả tìm kiếm
  6. Gọi `htmlPersistenceService.processSaveFlow` để xử lý lưu mới
  7. Ghi log kết quả xử lý

### 3. Xử lý lỗi

- Bắt và ghi log các lỗi phát sinh trong quá trình xử lý
- Phân biệt giữa lỗi tìm kiếm và lỗi xử lý
- Đảm bảo thông tin lỗi chi tiết được ghi lại

## 4. Tích hợp với các dịch vụ khác

### 1. GoogleSearchService

- Thực hiện tìm kiếm thông tin hội nghị
- Trả về danh sách các liên kết liên quan

### 2. HtmlPersistenceService

- Xử lý lưu trữ và cập nhật dữ liệu hội nghị
- Cung cấp hai luồng xử lý chính: update flow và save flow

### 3. FileSystemService

- Lưu trữ kết quả tìm kiếm
- Hỗ trợ ghi log và lưu trữ dữ liệu trung gian

### 4. RequestStateService và InMemoryResultCollectorService

- Theo dõi trạng thái yêu cầu
- Thu thập và quản lý kết quả xử lý

## 5. Xử lý đặc biệt

- Tự động xác định năm hiện tại và các năm liên quan cho việc tìm kiếm
- Hỗ trợ tái xử lý (re-crawl) thông qua originalRequestId
- Kiểm tra và xử lý các trường hợp đặc biệt như thiếu dữ liệu
