# InMemoryResultCollector Service

## 1. Tổng quan (Overview)

### Mục đích chính

`InMemoryResultCollectorService` đảm nhiệm việc thu thập và lưu trữ tạm thời kết quả crawl trong bộ nhớ, giúp quản lý dữ liệu một cách hiệu quả trong quá trình xử lý.

### Vai trò trong kiến trúc hệ thống

- Hoạt động như một bộ đệm lưu trữ tạm thời kết quả crawl
- Được tạo mới cho mỗi request crawl (container-scoped)
- Hỗ trợ song song với cơ chế lưu file

### Dependencies chính

- `LoggingService`: Ghi log các hoạt động thu thập dữ liệu
- `InputRowData`: Interface định nghĩa cấu trúc dữ liệu đầu vào

## 2. Class/Component chính

### Lớp InMemoryResultCollectorService

Quản lý bộ nhớ tạm để lưu trữ kết quả crawl.

#### Decorator

```typescript
@scoped(Lifecycle.ContainerScoped)
```

#### Constructor

```typescript
constructor(@inject(LoggingService) loggingService: LoggingService)
```

#### Properties

- `private results: InputRowData[]`: Mảng lưu trữ các bản ghi đã thu thập
- `private readonly collectorId: string`: Định danh duy nhất cho mỗi instance
- `private readonly logger: Logger`: Logger cho dịch vụ

#### Methods chính

1. **`add(record: InputRowData): void`**

   - Thêm một bản ghi mới vào bộ nhớ
   - Ghi log thông tin về bản ghi được thêm
   - Cập nhật số lượng bản ghi hiện tại

2. **`get(): InputRowData[]`**

   - Trả về toàn bộ kết quả đã thu thập
   - Trả về bản sao của mảng để đảm bảo tính toàn vẹn dữ liệu

3. **`clear(): void`**
   - Xóa toàn bộ dữ liệu đã thu thập
   - Hữu ích khi cần reset bộ nhớ

## 3. Flow xử lý dữ liệu

1. **Khởi tạo**

   - Tạo một collector ID ngẫu nhiên
   - Khởi tạo logger với context riêng
   - Khởi tạo mảng rỗng để lưu kết quả

2. **Thu thập dữ liệu**

   - Khi có dữ liệu mới, phương thức `add()` được gọi
   - Dữ liệu được thêm vào mảng `results`
   - Thông tin log được ghi lại để theo dõi

3. **Truy xuất dữ liệu**
   - Phương thức `get()` trả về toàn bộ dữ liệu đã thu thập
   - Trả về bản sao để tránh thay đổi trực tiếp dữ liệu gốc

## 4. Tích hợp với các thành phần khác

### Làm việc với CrawlOrchestratorService

- Được sử dụng bởi `CrawlOrchestratorService` để thu thập kết quả crawl
- Mỗi request crawl có một instance riêng để đảm bảo tính độc lập

### Tích hợp với LoggingService

- Sử dụng `LoggingService` để ghi lại các sự kiện quan trọng
- Mỗi instance có một logger với context riêng để dễ dàng theo dõi

## 5. Ghi chú sử dụng

- Phù hợp cho việc xử lý dữ liệu tạm thời trong quá trình crawl
- Không nên sử dụng cho dữ liệu cần lưu trữ lâu dài
- Cần xử lý cẩn thận khi làm việc với dữ liệu lớn để tránh tràn bộ nhớ
- Mỗi request nên sử dụng một instance riêng để tránh xung đột dữ liệu
