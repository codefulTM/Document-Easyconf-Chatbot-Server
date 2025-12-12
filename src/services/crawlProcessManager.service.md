# CrawlProcessManager Service

## 1. Tổng quan

### Mục đích chính

- Quản lý vòng đời của các tiến trình crawl dữ liệu
- Điều khiển việc dừng các tiến trình đang chạy
- Theo dõi trạng thái của các batch request

### Vai trò trong kiến trúc hệ thống

- Lớp dịch vụ đơn (singleton) đảm bảo chỉ có một instance duy nhất trong toàn bộ ứng dụng
- Đóng vai trò trung tâm trong việc kiểm soát các tiến trình crawl
- Tích hợp với các service khác thông qua cơ chế dependency injection

### Dependencies chính

- `tsyringe`: Để đánh dấu service là singleton
- `pino`: Để ghi log các hoạt động

## 2. Class chính: CrawlProcessManagerService

### Chức năng

Quản lý trạng thái và vòng đời của các tiến trình crawl thông qua cơ chế cờ dừng (stop flag) cho từng batch request.

### Constructor

```typescript
constructor();
```

Không có tham số vì sử dụng dependency injection của tsyringe.

### Properties

```typescript
private stopFlags: Map<string, boolean>
```

- Lưu trữ trạng thái dừng cho từng batch request
- Key: batchRequestId (string)
- Value: boolean (true nếu có yêu cầu dừng)

## 3. Methods quan trọng

### 1. `requestStop(batchRequestId: string, logger: Logger): void`

- **Mục đích**: Đánh dấu một batch request cụ thể cần được dừng
- **Tham số**:
  - `batchRequestId`: ID của batch cần dừng
  - `logger`: Đối tượng Logger để ghi log
- **Hành động**: Đặt cờ dừng thành true cho batch tương ứng

### 2. `isStopRequested(batchRequestId: string): boolean`

- **Mục đích**: Kiểm tra xem một batch request có đang được yêu cầu dừng hay không
- **Tham số**:
  - `batchRequestId`: ID của batch cần kiểm tra
- **Trả về**:
  - `true`: Nếu có yêu cầu dừng
  - `false`: Nếu không có yêu cầu dừng hoặc batch không tồn tại

### 3. `clearStopFlag(batchRequestId: string, logger: Logger): void`

- **Mục đích**: Xóa cờ dừng sau khi tiến trình đã hoàn tất việc dừng
- **Tham số**:
  - `batchRequestId`: ID của batch đã dừng
  - `logger`: Đối tượng Logger để ghi log
- **Hành động**: Xóa entry tương ứng với batchId khỏi Map

## 4. Luồng xử lý

1. Khi có yêu cầu dừng một tiến trình crawl:

   ```typescript
   crawlProcessManager.requestStop(batchId, logger);
   ```

2. Trong vòng lặp chính của tiến trình crawl:

   ```typescript
   if (crawlProcessManager.isStopRequested(batchId)) {
     // Thực hiện các thao tác dọn dẹp
     break; // Thoát khỏi vòng lặp
   }
   ```

3. Sau khi dừng xong:
   ```typescript
   crawlProcessManager.clearStopFlag(batchId, logger);
   ```

## 5. Tích hợp với các service khác

- **LoggingService**: Sử dụng để ghi lại các sự kiện quan trọng như yêu cầu dừng, xóa cờ, v.v.
- **CrawlOrchestratorService**: Sử dụng để kiểm tra và xử lý các yêu cầu dừng trong quá trình crawl

## 6. Xử lý lỗi

- Tự động xử lý các trường hợp batchId không tồn tại
- Ghi log đầy đủ các sự kiện quan trọng
- Đảm bảo không có memory leak bằng cách xóa các cờ không cần thiết
