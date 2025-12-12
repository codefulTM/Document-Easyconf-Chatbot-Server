# Gemini API Orchestrator Service

## 1. Tổng quan

### Mục đích chính

- Điều phối và quản lý các cuộc gọi API đến mô hình Gemini
- Xử lý logic fallback từ model chính sang model dự phòng khi cần thiết
- Quản lý cấu hình và tài nguyên cho việc thực thi các yêu cầu API

### Vai trò trong kiến trúc hệ thống

- Là lớp trung gian giữa các dịch vụ cấp cao hơn (như crawl service) và các dịch vụ thực thi API Gemini cấp thấp hơn
- Đảm bảo tính sẵn sàng cao thông qua cơ chế retry và fallback
- Tích hợp với các dịch vụ quản lý rate limiting, retry và thực thi SDK

### Dependencies chính

- `@google/genai`: SDK chính thức của Google cho Gemini API
- `tsyringe`: Hỗ trợ dependency injection
- `pino`: Ghi log
- Các dịch vụ nội bộ:
  - `GeminiModelOrchestratorService`: Quản lý các mô hình Gemini
  - `GeminiRateLimiterService`: Kiểm soát tốc độ gọi API
  - `GeminiRetryHandlerService`: Xử lý logic thử lại
  - `GeminiSdkExecutorService`: Thực thi các lệnh gọi SDK

## 2. Class chính: GeminiApiOrchestratorService

### Chức năng

Quản lý vòng đời của các yêu cầu API đến Gemini, xử lý fallback và retry, đồng thời cung cấp giao diện thống nhất cho các dịch vụ khác sử dụng.

### Constructor

```typescript
constructor(
    @inject(ConfigService) private configService: ConfigService,
    @inject(LoggingService) private loggingService: LoggingService,
    @inject(GeminiModelOrchestratorService) private modelOrchestrator: GeminiModelOrchestratorService,
    @inject(GeminiRateLimiterService) private rateLimiters: GeminiRateLimiterService,
    @inject(GeminiRetryHandlerService) private retryHandler: GeminiRetryHandlerService,
    @inject(GeminiSdkExecutorService) private sdkExecutor: GeminiSdkExecutorService
)
```

### Properties

- `serviceBaseLogger`: Logger chính cho service
- `generalApiTypeSettings`: Cấu hình chung cho các loại API
- `defaultMaxRetriesForFallback`: Số lần thử lại mặc định cho fallback

## 3. Các phương thức chính

### 1. `prepareFewShotParts(apiType: string, configForApiType: GeneralApiTypeConfig, parentLogger: Logger): Part[]`

- **Mục đích**: Chuẩn bị các ví dụ few-shot từ cấu hình
- **Step by step**:
  1. Khởi tạo một mảng rỗng để lưu trữ các phần few-shot
  2. Tạo một logger con để theo dõi quá trình xử lý
  3. Kiểm tra xem có dữ liệu đầu vào/ra hợp lệ không:
     - Nếu không có dữ liệu hoặc dữ liệu rỗng, ghi log và trả về mảng rỗng
  4. Lấy danh sách các khóa đầu vào và sắp xếp lại theo thứ tự số (input1, input2, ...)
  5. Với mỗi khóa đầu vào:
     - Trích chỉ số từ khóa đầu vào (ví dụ: từ "input1" lấy ra "1")
     - Tạo khóa đầu ra tương ứng (ví dụ: "output1")
     - Lấy giá trị đầu vào và đầu ra tương ứng
     - Nếu giá trị đầu vào tồn tại, thêm vào mảng kết quả
     - Nếu giá trị đầu ra tương ứng tồn tại, thêm vào mảng kết quả
     - Ghi log cảnh báo nếu thiếu giá trị đầu vào hoặc đầu ra
  6. Kiểm tra và ghi log kết quả:
     - Nếu mảng kết quả rỗng
     - Nếu số lượng phần tử lẻ (không phù hợp cho few-shot)
     - Thông báo thành công với số lượng cặp ví dụ
  7. Xử lý ngoại lệ nếu có, ghi log lỗi và trả về mảng rỗng
  8. Trả về mảng các phần few-shot đã chuẩn bị

### 2. `prepareForApiCall(modelNameToUse: string, apiType: string, initialUserContent: ContentListUnion, effectiveCrawlModelType: CrawlModelType, parentLogger: Logger): Promise<ModelExecutionConfig | null>`

- **Mục đích**: Chuẩn bị cấu hình cho lệnh gọi API
- **Step by step**:
  - Lấy rate limiter cho model hiện tại
  - Lấy config của API type hiện tại
  - Xử lý input đầu vào:
    - Nếu input đầu vào là string -> giữ nguyên
    - Nếu input đầu vào là mảng hoặc là một Content object đơn lẻ -> lấy phần text đầu tiên hoặc nếu không có text thì cho originalBatchPrompt = [multimodal content]
    - Không xác định được loại input -> [unknown content type]
  - Xác định xem lời gọi API là dành cho tuned hay non-tuned model:
    - Đối với tuned model:
      - Đặt MIME type thành text/plain
      - Xóa response schema
      - Lấy prefix cho model tuned. Cần xem lại: Tại sao prefixForTuned lại lấy từ `systemInstructionPrefixForNonTunedModel`?
      - Xử lý thêm prefix vào finalUserContent
    - Đối với non-tuned model:
      - Cài đặt MIME type, mặc định là application/json
      - Áp dụng response schema nếu MIME type là JSON
      - Lấy system instruction
      - Lấy các ví dụ few-shot
    - Nếu model chứa "2.5" thì chỉnh thinking config: Đặt thinking budget thành 8000 token
    - Sau khi đã chuẩn bị xong tất cả các thông số cần thiết(system instruction, few-shot examples, generation config,...), gửi các thông số đến modelOrchestrator.prepareModel() để chuẩn bị model
    - Kết quả trả về bao gồm tất cả các thông tin cần thiết để thực hiện lời gọi API thực tế

### 3. `orchestrateApiCall(params: InternalCallGeminiApiParams, parentServiceMethodLogger: Logger): Promise<OrchestrationResult>`

- **Mục đích**: Điều phối toàn bộ quá trình gọi API, xử lý fallback và retry
- **Step by step**:

  1. **Khởi tạo và thiết lập ban đầu**

     - Trích xuất các tham số từ `InternalCallGeminiApiParams`
     - Tạo logger cho quá trình xử lý
     - Khởi tạo kết quả mặc định

  2. **Giai đoạn 1: Thử với model chính (Primary Model)**

     - Ghi log bắt đầu quá trình gọi API với model chính
     - Kiểm tra nếu có tên model chính được cung cấp
     - Chuẩn bị cấu hình cho model chính thông qua `prepareForApiCall`
     - Nếu chuẩn bị thành công:
       - Tạo hàm gọi API với `executeSdkCall`
       - Thực hiện gọi API với cơ chế retry (tối đa 1 lần thử)
       - Nếu thành công, trả về kết quả
       - Nếu thất bại, ghi log lỗi và tiếp tục
     - Nếu chuẩn bị thất bại, ghi log lỗi

  3. **Giai đoạn 2: Thử với model dự phòng (Fallback Model)**

     - Kiểm tra nếu model chính thất bại và có model dự phòng
     - Nếu không có model dự phòng:
       - Ghi log thông báo không có model dự phòng
       - Trả về lỗi với thông tin từ lỗi của model chính
     - Nếu có model dự phòng:
       - Xác định loại crawl model phù hợp cho model dự phòng
       - Ghi log bắt đầu quá trình gọi API với model dự phòng
       - Chuẩn bị cấu hình cho model dự phòng
       - Thực hiện gọi API với cơ chế retry đầy đủ

  4. **Xử lý kết quả**

     - Nếu có kết quả từ model dự phòng, trả về kết quả đó
     - Nếu không, trả về lỗi với thông tin từ lỗi cuối cùng
     - Ghi log kết thúc quá trình xử lý

  5. **Xử lý lỗi**
     - Bắt và xử lý các ngoại lệ phát sinh trong quá trình thực thi
     - Ghi log chi tiết lỗi
     - Đảm bảo luồng thực thi không bị gián đoạn bởi lỗi
