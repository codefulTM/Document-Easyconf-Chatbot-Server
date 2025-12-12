# Tài liệu: playwright.service.ts

## 1. Tổng quan (Overview)

### Mục đích chính

`PlaywrightService` là một service singleton chịu trách nhiệm quản lý toàn bộ vòng đời của một trình duyệt Playwright. Mục đích chính của nó là khởi tạo, cấu hình và cung cấp một môi trường trình duyệt (browser context) đã được tối ưu hóa cho các tác vụ web scraping, đồng thời quản lý việc đóng trình duyệt một cách an toàn để giải phóng tài nguyên.

- **Playwright**: Là một thư viện Node.js do Microsoft phát triển, cho phép tự động hóa và điều khiển các trình duyệt hiện đại (Chromium, Firefox, WebKit) thông qua một API duy nhất. Nó rất mạnh mẽ cho việc kiểm thử tự động và cào dữ liệu web vì có thể tương tác với các trang web động như một người dùng thực thụ.
- **Web Scraping**: Là quá trình sử dụng các chương trình máy tính (bot) để tự động truy cập và trích xuất một lượng lớn dữ liệu từ các trang web. Thay vì sao chép thủ công, bot sẽ "đọc" mã HTML của trang và lấy ra thông tin cần thiết.

### Vai trò trong kiến trúc hệ thống

- **Nền tảng cho Web Scraping**: Đây là service nền tảng, cung cấp "công cụ" trình duyệt cho bất kỳ service nào khác cần tương tác với các trang web, lấy nội dung HTML, hoặc thực thi JavaScript phía client.
- **Quản lý tài nguyên tập trung**: Vì là một singleton, nó đảm bảo chỉ có một instance trình duyệt duy nhất được chạy trong toàn bộ ứng dụng, giúp tiết kiệm tài nguyên hệ thống (CPU, RAM) một cách đáng kể.
- **Tối ưu hóa hiệu suất**: Service này chủ động chặn các tài nguyên không cần thiết (hình ảnh, CSS, font, script theo dõi) thông qua cơ chế request interception, giúp tăng tốc độ tải trang và giảm băng thông sử dụng trong quá trình crawl.

### Dependencies chính

- **`tsyringe`**: Được sử dụng để triển khai pattern singleton và dependency injection.
- **`playwright`**: Thư viện cốt lõi để tự động hóa và điều khiển trình duyệt.
- **`ConfigService`**: Cung cấp các cấu hình quan trọng cho Playwright như:
    - `PLAYWRIGHT_CHANNEL`: Loại trình duyệt (ví dụ: 'chrome', 'msedge').
    - `PLAYWRIGHT_HEADLESS`: Chế độ chạy (headless hoặc có giao diện).
    - `USER_AGENT`: Chuỗi User-Agent tùy chỉnh để giả lập trình duyệt người dùng.
- **`LoggingService`**: Dùng để ghi lại các log chi tiết trong suốt quá trình hoạt động của service, từ khởi tạo đến khi đóng.

## 2. Class/Component chính

### Tên class và chức năng

- **Class**: `PlaywrightService`
- **Chức năng**: Đóng gói và trừu tượng hóa toàn bộ logic phức tạp của việc quản lý trình duyệt Playwright. Nó cung cấp các phương thức đơn giản để khởi tạo, truy cập và đóng trình duyệt, giúp các service khác không cần quan tâm đến chi tiết triển khai bên dưới.

### Constructor và injected dependencies

```typescript
constructor(
    @inject(LoggingService) private loggingService: LoggingService,
    @inject(ConfigService) private configService: ConfigService
)
```

Service này inject `LoggingService` để ghi log và `ConfigService` để lấy các cấu hình cần thiết.

### Properties và methods quan trọng

- **Properties**:
  - `browser: Browser | null`: Giữ instance của trình duyệt Playwright sau khi đã khởi tạo.
  - `browserContext: BrowserContext | null`: Giữ instance của "bối cảnh trình duyệt". Đây là môi trường biệt lập nơi các tab (pages) được tạo ra. Toàn bộ cấu hình như viewport, user-agent, và request interception được áp dụng ở cấp độ này.
  - `isInitialized: boolean`: Một cờ (flag) để kiểm tra xem service đã khởi tạo thành công hay chưa.
  - `isInitializing: boolean`: Một cờ để ngăn chặn việc khởi tạo đồng thời từ nhiều nơi, đảm bảo chỉ có một tiến trình khởi tạo chạy tại một thời điểm.

- **Methods**:
  - `initialize(parentLogger?: Logger)`: Phương thức quan trọng nhất. Nó khởi chạy trình duyệt `chromium` với các đối số dòng lệnh được tối ưu cho môi trường server/Docker. Sau đó, nó tạo một `BrowserContext` mới và cấu hình chi tiết (vô hiệu hóa quyền, đặt viewport, user-agent, bỏ qua lỗi HTTPS, và đặc biệt là thiết lập `route` để chặn tài nguyên không cần thiết).
    - **Step by step**:
      1.  **Kiểm tra trạng thái**: Nếu service đã được khởi tạo (`isInitialized` là `true`), phương thức sẽ bỏ qua và kết thúc. Nếu đang trong quá trình khởi tạo (`isInitializing` là `true`), nó sẽ chờ cho đến khi quá trình đó hoàn tất.
      2.  **Bắt đầu khởi tạo**: Đặt cờ `isInitializing` thành `true` để ngăn các lệnh gọi đồng thời khác.
      3.  **Xác thực cấu hình**: Kiểm tra các biến môi trường quan trọng (`PLAYWRIGHT_CHANNEL`, `USER_AGENT`). Nếu thiếu, một lỗi nghiêm trọng sẽ được ném ra.
      4.  **Khởi chạy trình duyệt**: Sử dụng `chromium.launch()` để mở một trình duyệt với các đối số dòng lệnh được tối ưu hóa cho hiệu suất và môi trường Docker (ví dụ: `--no-sandbox`, `--disable-gpu`).
      5.  **Tạo Context**: Tạo một `BrowserContext` mới, đây là một môi trường duyệt web biệt lập. Các cấu hình quan trọng được áp dụng tại đây, bao gồm `userAgent`, `viewport`, và `ignoreHTTPSErrors`.
      6.  **Cấu hình chặn Request**: Thiết lập `context.route()` để can thiệp vào tất cả các yêu cầu mạng. Các tài nguyên không cần thiết cho việc lấy nội dung (như `image`, `stylesheet`, `font`, `media`) và các tên miền theo dõi/quảng cáo sẽ bị chặn (`route.abort()`) để tăng tốc độ tải trang. Các yêu cầu khác được cho phép tiếp tục (`route.continue()`).
      7.  **Hoàn tất**: Nếu tất cả các bước trên thành công, các instance `browser` và `browserContext` sẽ được lưu vào thuộc tính của class, và cờ `isInitialized` được đặt thành `true`.
      8.  **Xử lý lỗi**: Nếu có bất kỳ lỗi nào xảy ra, phương thức sẽ cố gắng đóng trình duyệt (nếu đã được khởi chạy một phần), ghi log lỗi chi tiết và ném lỗi ra ngoài để báo hiệu cho nơi gọi nó.
      9.  **Dọn dẹp**: Cờ `isInitializing` luôn được đặt lại thành `false` trong khối `finally`, bất kể thành công hay thất bại.
  - `getBrowserContext(parentLogger?: Logger)`: Phương thức công khai để các service khác lấy `BrowserContext` đã được khởi tạo. Nó sẽ báo lỗi nếu được gọi trước khi `initialize()` hoàn tất.
    - **Step by step**:
      1.  **Kiểm tra trạng thái**: Kiểm tra xem cờ `isInitialized` có phải là `true` và thuộc tính `browserContext` có tồn tại không.
      2.  **Trả về hoặc Báo lỗi**: Nếu đã khởi tạo thành công, nó sẽ trả về `browserContext`. Ngược lại, nó sẽ ném ra một lỗi để thông báo rằng service chưa sẵn sàng.
  - `close(parentLogger?: Logger)`: Đóng trình duyệt và giải phóng tất cả tài nguyên liên quan. Nó được thiết kế để xử lý các trường hợp phức tạp như được gọi khi đang khởi tạo hoặc khi đã được đóng từ trước.
    - **Step by step**:
      1.  **Kiểm tra trạng thái**: Nếu service đang trong quá trình khởi tạo, phương thức sẽ chờ cho đến khi quá trình đó kết thúc.
      2.  **Thực hiện đóng**: Nếu service đã được khởi tạo và có một instance trình duyệt đang chạy, nó sẽ gọi `browser.close()`.
      3.  **Xử lý lỗi**: Bắt và ghi lại bất kỳ lỗi nào có thể xảy ra trong quá trình đóng trình duyệt.
      4.  **Reset trạng thái**: Bất kể việc đóng có thành công hay không, phương thức sẽ luôn reset lại tất cả các thuộc tính trạng thái (`browser`, `browserContext`, `isInitialized`) về giá trị ban đầu để đảm bảo service ở trạng thái sạch sẽ cho các lần sử dụng sau (nếu có).
    
## 3. Business Logic (Service)

### Các methods chính LIÊN QUAN ĐẾN CRAWL DỮ LIỆU và chức năng

- **`initialize(...)`**: Không trực tiếp crawl, nhưng là bước chuẩn bị không thể thiếu. Nó tạo ra môi trường trình duyệt sẵn sàng cho việc crawl, đặc biệt là việc cấu hình chặn request.
- **`getBrowserContext(...)`**: Cung cấp "cánh cửa" để các service khác (ví dụ: `PageContentExtractorService`) có thể tạo một trang mới (`newPage()`) và bắt đầu quá trình crawl thực tế (điều hướng đến URL, lấy nội dung).

### Flow xử lý dữ liệu

Flow của service này không phải là xử lý dữ liệu mà là quản lý vòng đời của công cụ xử lý:

1.  **Yêu cầu khởi tạo**: Một service cấp cao hơn gọi `playwrightService.initialize()`.
2.  **Kiểm tra trạng thái**: Service kiểm tra xem nó đã hoặc đang được khởi tạo chưa để tránh thực hiện lại. Nó có cơ chế chờ nếu một tiến trình khởi tạo khác đang chạy.
3.  **Khởi chạy trình duyệt**: Launch một instance trình duyệt `chromium` với các cấu hình tối ưu.
4.  **Tạo và cấu hình Context**: Tạo một `BrowserContext` mới. Đây là bước quan trọng nhất, nơi các thiết lập về hiệu suất và "tàng hình" được áp dụng:
    - Chặn các loại tài nguyên: `image`, `media`, `font`, `stylesheet`.
    - Chặn các tên miền theo dõi và quảng cáo phổ biến.
    - Đặt `userAgent` tùy chỉnh.
5.  **Hoàn tất khởi tạo**: Đánh dấu `isInitialized = true`.
6.  **Cung cấp Context**: Các service khác gọi `getBrowserContext()` để lấy context và sử dụng nó để tạo các trang (`page`) và thực hiện crawl.
7.  **Đóng trình duyệt**: Khi ứng dụng kết thúc, `close()` được gọi để đóng trình duyệt và dọn dẹp tài nguyên.

### Integration với external services

- Service này không tích hợp trực tiếp với dịch vụ bên ngoài nào. Thay vào đó, nó là công cụ để các service khác tích hợp với các trang web bên ngoài (đối tượng cần crawl).