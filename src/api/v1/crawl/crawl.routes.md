# Crawl Routes API Documentation

## 1. Tổng quan
- **Mục đích**: File này định nghĩa các API endpoints liên quan đến chức năng crawl dữ liệu hội nghị và tạp chí.
- **Vai trò**: Đóng vai trò là lớp điều phối (router) trong kiến trúc MVC, tiếp nhận các yêu cầu HTTP và chuyển hướng đến các controller tương ứng.
- **Dependencies chính**:
  - Express.js: Framework web sử dụng để xử lý routing
  - crawl.controller: Module chứa các hàm xử lý logic cho từng endpoint

## 2. Các Endpoints chính

### `POST /crawl-conferences`
- **Mục đích**: Khởi tạo quá trình crawl thông tin hội nghị
- **Controller**: `handleCrawlConferences`
- **Ghi chú**: Được bảo vệ bởi mutex để tránh chạy đồng thời nhiều tiến trình crawl

### `POST /crawl-conferences/stop`
- **Mục đích**: Dừng quá trình crawl đang chạy
- **Controller**: `handleStopCrawl`
- **Ghi chú**: Cho phép dừng tiến trình crawl thông qua API

### `POST /crawl-journals`
- **Mục đích**: Khởi tạo quá trình crawl thông tin tạp chí (đang trong quá trình phát triển)
- **Controller**: `handleCrawlJournals`
- **Ghi chú**: Hiện chưa được triển khai đầy đủ

## 3. Kiến trúc và Thiết kế
- Sử dụng mô hình Router của Express để tổ chức các endpoints
- Mỗi endpoint được định nghĩa rõ ràng với phương thức HTTP và đường dẫn cụ thể
- Tách biệt rõ ràng giữa routing logic (trong file này) và business logic (trong controller)
- Dễ dàng mở rộng bằng cách thêm các routes mới khi cần

## 4. Ghi chú phát triển
- Các endpoint mới nên tuân thủ theo cấu trúc hiện tại
- Khi thêm endpoint mới, cần đảm bảo có xử lý lỗi đầy đủ
- Cân nhắc thêm middleware xác thực/phân quyền nếu cần thiết