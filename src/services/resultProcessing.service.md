# Tài liệu: resultProcessing.service.ts

## 1. Tổng quan

### Mục đích chính

- Xử lý và chuẩn hóa dữ liệu hội nghị từ nhiều nguồn khác nhau
- Chuyển đổi dữ liệu thô thành định dạng chuẩn để lưu trữ và xuất báo cáo
- Đảm bảo tính nhất quán và hợp lệ của dữ liệu

### Vai trò trong kiến trúc hệ thống

- Lớp dịch vụ (Service) trong kiến trúc ứng dụng
- Chịu trách nhiệm xử lý dữ liệu đầu vào trước khi lưu trữ hoặc xuất báo cáo
- Nằm ở tầng xử lý nghiệp vụ (Business Logic Layer)

### Dependencies chính

- `@json2csv/node`: Chuyển đổi dữ liệu JSON sang định dạng CSV
- `fs` và `path`: Làm việc với hệ thống file
- `stream`: Xử lý dữ liệu dạng stream
- `tsyringe`: Dependency injection
- Các service nội bộ:
  - `ConfigService`: Cấu hình ứng dụng
  - `LoggingService`: Ghi log hệ thống
  - `FileSystemService`: Thao tác với hệ thống file

## 2. Class chính: ResultProcessingService

### Chức năng

- Xử lý và chuẩn hóa dữ liệu hội nghị từ nhiều nguồn
- Chuyển đổi giữa các định dạng dữ liệu (JSONL → Object → CSV)
- Đảm bảo tính hợp lệ và nhất quán của dữ liệu

### Constructor

```typescript
constructor(
  private configService: ConfigService,
  private loggingService: LoggingService,
  private fileSystemService: FileSystemService
)
```

### Properties quan trọng

- `VALID_CONTINENTS`: Tập hợp các châu lục hợp lệ
- `VALID_TYPES`: Các loại hình hội nghị được chấp nhận
- `YEAR_REGEX`: Biểu thức chính quy kiểm tra định dạng năm
- `CSV_FIELDS`: Danh sách các trường dữ liệu đầu ra

### Methods chính

#### 1. `_processApiResponse`

- Xử lý dữ liệu phản hồi từ API
- Chuẩn hóa và tổ chức lại cấu trúc dữ liệu
- Xử lý các trường dữ liệu ngày tháng và thông tin chi tiết

#### 2. `_transformSingleRecord`

- Chuyển đổi một bản ghi thô thành dạng đã xử lý
- Thực hiện validation và chuẩn hóa dữ liệu
- Xử lý các trường hợp đặc biệt và giá trị mặc định
- Có phạm vi rộng hơn `_processApiResponse`, bao gồm cả việc đọc dữ liệu từ nhiều nguồn và thực hiện validation
- `_transformSingleRecord` sử dụng `_processApiResponse` như một bước trong quy trình xử lý lớn hơn. Nó dùng để xử lý extractInfo(thông tin chi tiết về hội nghị, phân biệt với determineInfo - Các liên kết liên quan và cfpInfo - Thông tin về lời kêu gọi nộp bài).

#### 3. `_processJsonlStream`

- Đọc và xử lý dữ liệu từ file JSONL
- Sử dụng stream để xử lý file lớn hiệu quả
- Gọi `_transformSingleRecord` cho từng dòng dữ liệu

#### 4. `processInMemoryData`

- Xử lý mảng dữ liệu trong bộ nhớ
- Phù hợp với lượng dữ liệu vừa và nhỏ
- Trả về mảng dữ liệu đã xử lý

#### 5. `_writeCSVAndCollectDataInternal`

- Ghi dữ liệu đã xử lý ra file CSV
- Hỗ trợ ghi tăng dần (streaming) - kỹ thuật xử lý dữ liệu mà không cần tải toàn bộ dữ liệu vào bộ nhớ cùng một lúc
- Thu thập thống kê cơ bản về quá trình xử lý:
  - `recordsProcessed`: Tổng số bản ghi đã xử lý
  - `recordsWrittenToCsv`: Số bản ghi đã ghi thành công vào CSV
  - `skippedCount`: Số bản ghi bị bỏ qua (tính bằng `recordsProcessed - recordsWrittenToCsv`)
- Các thông tin chi tiết khác được ghi vào log thông qua hệ thống logging
