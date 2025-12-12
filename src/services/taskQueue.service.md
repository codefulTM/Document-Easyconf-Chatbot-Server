# TaskQueue Service

## 1. Tổng quan (Overview)

### Mục đích chính

`TaskQueueService` đóng vai trò quản lý hàng đợi tác vụ cho mỗi request crawl, đảm bảo việc thực thi đồng thời các tác vụ được kiểm soát chặt chẽ.

### Vai trò trong kiến trúc hệ thống

- Là một trong những dịch vụ cốt lõi trong quá trình crawl dữ liệu
- Hoạt động ở cấp độ request, mỗi request crawl sẽ có một instance riêng
- Đảm bảo tải công việc được phân phối đều và hiệu quả

### Dependencies chính

- `ConfigService`: Lấy cấu hình số lượng tác vụ chạy đồng thời
- `LoggingService`: Ghi log các hoạt động của hàng đợi
- `p-queue`: Thư viện bên thứ ba quản lý hàng đợi tác vụ

## 2. Class/Component chính

### Lớp TaskQueueService

Quản lý hàng đợi tác vụ cho mỗi request crawl.

#### Constructor

```typescript
constructor(
    @inject(ConfigService) private configService: ConfigService,
    @inject(LoggingService) private loggingService: LoggingService,
)
```

#### Properties

- `private readonly logger: Logger`: Logger cho dịch vụ
- `private readonly queue: PQueue`: Instance của PQueue để quản lý hàng đợi
- `public readonly concurrency: number`: Số lượng tác vụ chạy đồng thời tối đa

#### Methods chính

1. **`add<TaskResult>(task: () => Promise<TaskResult | void>): Promise<TaskResult | void>`**

   - Thêm một tác vụ mới vào hàng đợi
   - Trả về một Promise sẽ được resolve khi tác vụ hoàn thành

2. **`onIdle(): Promise<void>`**

   - Trả về một Promise sẽ được resolve khi tất cả các tác vụ trong hàng đợi đã hoàn thành
   - Hữu ích để đợi hoàn thành toàn bộ quá trình crawl

3. **`get size(): number`**

   - Lấy số lượng tác vụ đang chờ trong hàng đợi

4. **`get pending(): number`**
   - Lấy số lượng tác vụ đang được thực thi

## 3. Flow xử lý chính

1. **Khởi tạo**

   - Đọc cấu hình `crawlConcurrency` từ `ConfigService`
   - Tạo mới một instance `PQueue` với độ ưu tiên được cấu hình

2. **Thêm tác vụ**

   - Khi một tác vụ mới được thêm qua phương thức `add()`
   - Tác vụ được đưa vào hàng đợi của `PQueue`
   - Tự động điều phối thực thi dựa trên số lượng tác vụ chạy đồng thời

3. **Theo dõi tiến trình**
   - Có thể theo dõi số lượng tác vụ đang chờ (`size`)
   - Theo dõi số lượng tác vụ đang thực thi (`pending`)
   - Sử dụng `onIdle()` để đợi hoàn thành tất cả tác vụ
