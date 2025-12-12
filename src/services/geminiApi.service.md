# Tài liệu: geminiApi.service.ts

## 1. Tổng quan (Overview)

### Mục đích chính

`GeminiApiService` là một service chuyên dụng để tương tác với Google Gemini API. Service này đóng vai trò là một lớp trừu tượng, đơn giản hóa việc gửi yêu cầu, xử lý phản hồi, và quản lý các loại lệnh gọi API khác nhau (trích xuất thông tin, xác định liên kết, xử lý CFP) cho các phần còn lại của ứng dụng.

### Vai trò trong kiến trúc hệ thống

- Là một service trung tâm trong tầng xử lý dữ liệu, chịu trách nhiệm cho mọi tác vụ cần đến khả năng của mô hình ngôn ngữ lớn (LLM).
- Được các service cấp cao hơn (ví dụ: `CrawlOrchestratorService`) gọi để làm giàu và phân tích dữ liệu thu thập được trong quá trình crawl.
- Đóng gói logic phức tạp như lựa chọn model, cơ chế thử lại (retry), sử dụng model dự phòng (fallback), và làm sạch dữ liệu trả về từ API.

### Dependencies chính

- **`ConfigService`**: Cung cấp các cấu hình liên quan đến Gemini API, chẳng hạn như danh sách các model, model dự phòng, và các tham số khác.
- **`LoggingService`**: Ghi lại log chi tiết về các hoạt động của service, giúp cho việc theo dõi và gỡ lỗi.
- **`GeminiCachePersistenceService`**: Quản lý việc cache các phản hồi từ API để tối ưu hóa hiệu suất và chi phí.
- **`GeminiApiOrchestratorService`**: Điều phối việc thực thi các lệnh gọi API, xử lý logic retry, fallback và cân bằng tải giữa các model.
- **`GeminiResponseHandlerService`**: Chịu trách nhiệm làm sạch và tiền xử lý (ví dụ: trích xuất JSON từ văn bản thô) các phản hồi nhận được từ Gemini API.

## 2. Class/Component chính

### Tên class và chức năng

- **Class**: `GeminiApiService`
- **Chức năng**: Cung cấp các phương thức công khai, dễ sử dụng để các service khác có thể yêu cầu Gemini API thực hiện các tác vụ cụ thể liên quan đến crawl dữ liệu mà không cần biết về chi tiết triển khai bên dưới.

### Constructor và injected dependencies

```typescript
constructor(
    @inject(ConfigService) private configService: ConfigService,
    @inject(LoggingService) private loggingService: LoggingService,
    @inject(GeminiCachePersistenceService) private cachePersistence: GeminiCachePersistenceService,
    @inject(GeminiApiOrchestratorService) private apiOrchestrator: GeminiApiOrchestratorService,
    @inject(GeminiResponseHandlerService) private responseHandler: GeminiResponseHandlerService,
)
```

### Properties và methods quan trọng

- **Properties**:

  - `API_TYPE_EXTRACT`, `API_TYPE_DETERMINE`, `API_TYPE_CFP`: Các hằng số định danh cho từng loại tác vụ API, giúp mã nguồn dễ đọc và bảo trì.
  - `modelIndices`: Một đối tượng dùng để theo dõi và luân chuyển việc sử dụng các model khác nhau cho mỗi loại API, nhằm mục đích phân phối tải.
  - `serviceInitialized`: Cờ (flag) để đảm bảo service đã được khởi tạo (ví dụ: tải cache) trước khi thực hiện bất kỳ lệnh gọi API nào.

- **Methods**:
  - `init()`: Phương thức khởi tạo bất đồng bộ, cần được gọi một lần trước khi sử dụng service.
  - `extractInformation(...)`: Đây là một phương thức công khai (public) đóng vai trò là một "wrapper" đơn giản. Chức năng chính của nó là cung cấp một điểm truy cập (entry point) rõ ràng cho việc trích xuất thông tin cấu trúc (ngày tháng, địa điểm, chủ đề...) từ nội dung của một trang web hội nghị.
    - **Step by step**: 
      - 1. **Nhận yêu cầu**: Phương thức được gọi từ một service bên ngoài với các tham số `params`, `crawlModel`, và `parentLogger`.
      - 2. **Ủy quyền xử lý**: Ngay lập tức gọi phương thức `executeApiCallLogic`, truyền tất cả các tham số nhận được cùng với hằng số `this.API_TYPE_EXTRACT` để chỉ định loại tác vụ.
      - 3. **Trả về kết quả**: Chờ `executeApiCallLogic` hoàn thành và trả về đối tượng `ApiResponse` mà nó nhận được.
  - `extractCfp(...)`: Tương tự như `extractInformation`, đây là một phương thức public đóng vai trò "wrapper". Nó chuyên dùng để yêu cầu trích xuất thông tin từ các trang "Call for Papers" (CFP), ví dụ như các mốc thời gian quan trọng và nội dung chi tiết của lời mời.
    - **Step by step**: 
      - 1. **Nhận yêu cầu**: Phương thức được gọi với các tham số `params`, `crawlModel`, và `parentLogger`.
      - 2. **Ủy quyền xử lý**: Ngay lập tức gọi phương thức `executeApiCallLogic`, truyền tất cả các tham số nhận được cùng với hằng số `this.API_TYPE_CFP`.
      - 3. **Trả về kết quả**: Chờ `executeApiCallLogic` hoàn thành và trả về đối tượng `ApiResponse`.
  - `determineLinks(...)`: Là một phương thức public "wrapper" thứ ba. Nhiệm vụ của nó là yêu cầu phân tích một danh sách các URL để xác định và phân loại các liên kết quan trọng nhất, chẳng hạn như đâu là trang chủ, đâu là trang CFP.
    - **Step by step**: 
      - 1. **Nhận yêu cầu**: Phương thức được gọi với các tham số `params`, `crawlModel`, và `parentLogger`.
      - 2. **Ủy quyền xử lý**: Ngay lập tức gọi phương thức `executeApiCallLogic`, truyền tất cả các tham số nhận được cùng với hằng số `this.API_TYPE_DETERMINE`.
      - 3. **Trả về kết quả**: Chờ `executeApiCallLogic` hoàn thành và trả về đối tượng `ApiResponse`.
  - `executeApiCallLogic(...)`: Đây là phương thức riêng tư (private) và là "trái tim" của `GeminiApiService`. Nó chứa toàn bộ logic để thực hiện một lệnh gọi API hoàn chỉnh, từ khâu chuẩn bị, lựa chọn model, gửi yêu cầu, cho đến xử lý và làm sạch phản hồi. Việc tập trung logic vào đây giúp tránh lặp lại mã và đảm bảo một quy trình nhất quán.
    - **Step by step**: 
      - 1. **Khởi tạo Logger & Kiểm tra**: Tạo một logger chuyên dụng cho lần thực thi và gọi `ensureInitialized()` để chắc chắn rằng service đã sẵn sàng hoạt động.
      - 2. **Lựa chọn Model**: Dựa trên `apiType` và `initialCrawlModelType` ('tuned' hoặc 'non-tuned'), phương thức lấy ra danh sách model phù hợp từ `ConfigService`. Nó sử dụng cơ chế round-robin (thông qua `this.modelIndices`) để chọn một model từ danh sách, nhằm phân phối tải đều.
      - 3. **Xác định Nội dung Yêu cầu (`userContent`)**: Kiểm tra xem tham số `params.contents` (dành cho yêu cầu đa phương thức như ảnh + text) có tồn tại không. Nếu có, `userContent` sẽ là `contents`. Nếu không, nó sẽ sử dụng `params.batch` (dành cho yêu cầu chỉ có văn bản). Nếu cả hai đều không có, phương thức sẽ ghi log lỗi và trả về một phản hồi rỗng.
      - 4. **Gọi Orchestrator**: Gọi phương thức `apiOrchestrator.orchestrateApiCall` và truyền vào `userContent` cùng với tất cả các thông tin cần thiết khác (tên model, loại API, thông tin log...). `GeminiApiOrchestratorService` sẽ chịu trách nhiệm thực hiện cuộc gọi API thực tế và xử lý logic retry/fallback.
      - 5. **Xử lý Kết quả từ Orchestrator**:
          - **Nếu thành công**: Kết quả thô (`responseText`) được chuyển đến `responseHandler.cleanJsonResponse` để loại bỏ các ký tự không hợp lệ và trích xuất chuỗi JSON.
          - **Nếu thất bại**: Ghi log lỗi chi tiết về lý do thất bại.
      - 6. **Trả về `ApiResponse`**: Trả về đối tượng `ApiResponse` chứa `responseText` đã được làm sạch và `metaData` (nếu thành công) hoặc một đối tượng rỗng (nếu thất bại).
      - 7. **Bắt lỗi ngoại lệ (Catch Block)**: Nếu có bất kỳ lỗi không mong muốn nào xảy ra trong toàn bộ quá trình, nó sẽ được bắt lại, ghi log chi tiết và một `ApiResponse` mặc định sẽ được trả về.

## 3. Business Logic (Service)

### Các methods chính LIÊN QUAN ĐẾN CRAWL DỮ LIỆU và chức năng

- **`extractInformation(params, crawlModel, parentLogger)`**: Gửi yêu cầu đến Gemini API để trích xuất thông tin chi tiết về một hội nghị (ngày tháng, địa điểm, chủ đề,...) từ nội dung trang web.
- **`extractCfp(params, crawlModel, parentLogger)`**: Gửi yêu cầu để trích xuất thông tin từ trang "Call for Papers" (CFP), chẳng hạn như tóm tắt và nội dung chi tiết của lời mời nộp bài.
- **`determineLinks(params, crawlModel, parentLogger)`**: Gửi yêu cầu để xác định và phân loại các đường link quan trọng (trang chính, trang CFP, trang ngày quan trọng) từ một danh sách các URL tìm được.

### Flow xử lý dữ liệu

1.  **Nhận yêu cầu**: Một service khác gọi một trong các phương thức public (ví dụ: `extractInformation`).
2.  **Gọi Logic Chung**: Phương thức public này gọi `executeApiCallLogic` và truyền vào loại tác vụ (`apiType`).
3.  **Kiểm tra khởi tạo**: `executeApiCallLogic` đảm bảo service đã được `init()`.
4.  **Lựa chọn Model**: Dựa trên `apiType` và `crawlModelType` (tuned/non-tuned), service chọn một model từ danh sách đã cấu hình trong `ConfigService`. Cơ chế luân phiên (round-robin) được sử dụng để chọn model tiếp theo cho các lần gọi sau.
5.  **Xác định Nội dung**: Phương thức xác định nội dung sẽ được gửi đi. Nó ưu tiên `contents` (cho các yêu cầu đa phương thức, ví dụ: hình ảnh và văn bản) hoặc sử dụng `batch` (cho các yêu cầu chỉ có văn bản).
6.  **Điều phối API Call**: Dữ liệu được chuyển xuống `GeminiApiOrchestratorService`. Service này sẽ thực hiện lệnh gọi API thực tế, quản lý việc thử lại nếu có lỗi và tự động chuyển sang model dự phòng nếu cần.
7.  **Xử lý Phản hồi**: Kết quả thô từ `GeminiApiOrchestratorService` được chuyển đến `GeminiResponseHandlerService` để làm sạch (ví dụ: loại bỏ các ký tự không hợp lệ, trích xuất chuỗi JSON).
8.  **Kiểm tra và Ghi Log**: Service thực hiện các bước kiểm tra bổ sung trên dữ liệu đã làm sạch (ví dụ: xác thực cấu trúc JSON) và ghi log chi tiết về toàn bộ quá trình.
9.  **Trả kết quả**: Dữ liệu cuối cùng (thường là một chuỗi JSON sạch) được trả về cho service đã gọi ban đầu.

### Integration với external services

- **Google Gemini API**: Đây là dịch vụ bên ngoài cốt lõi mà `GeminiApiService` tích hợp. Toàn bộ service được xây dựng để hoạt động như một client chuyên dụng cho Gemini API, trừu tượng hóa các chi tiết phức tạp của nó.
