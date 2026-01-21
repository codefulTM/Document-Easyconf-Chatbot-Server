<!-- filepath: d:\LearningMaterial\KHTN\DoAnTotNghiep\Code\Document-Easyconf-Chatbot-Server\src\chatbot\services\rag\embeddingService.md -->

# EmbeddingService

Tài liệu cho `EmbeddingService` (được định nghĩa trong `embeddingService.ts`). Service này dùng để tải model embedding tại runtime và tạo các vector embedding cho chuỗi văn bản, phục vụ cho RAG, tìm kiếm ngữ nghĩa, hoặc lưu trữ embedding.

## Tổng quan

- `EmbeddingService` là một singleton: chỉ tạo một instance trong ứng dụng.
 - Cung cấp các phương thức chính:
  - `init()` — khởi tạo pipeline (tải model).
  - `generateEmbedding(text)` — sinh embedding cho một chuỗi.
  - `generateEmbeddings(texts)` — sinh embedding cho nhiều chuỗi tuần tự (hiện làm tuần tự).
  - `generateEmbeddingsBatch(texts)` — API theo lô (batch): sinh embedding cho nhiều chuỗi trong một lần gọi (hiệu năng cao hơn).
  - `generateEmbeddings(texts)` — sinh embedding cho nhiều chuỗi tuần tự.

## Kiến trúc lớp

```ts
export class EmbeddingService {
  private static instance: EmbeddingService;
  private pipe: FeatureExtractionPipeline | null = null;
  private readonly modelName = 'Xenova/all-MiniLM-L6-v2';
  private constructor() { }
  public static getInstance(): EmbeddingService { ... }
  public async init(): Promise<void> { ... }
  public async generateEmbedding(text: string): Promise<number[]> { ... }
  public async generateEmbeddings(texts: string[]): Promise<number[][]> { ... }
}
```

### Singleton

- `getInstance()` trả về instance duy nhất, đảm bảo model/pipeline chỉ được tải 1 lần trong vòng đời tiến trình.

## Chi tiết các phương thức

### init()

- Mục đích: tải model và khởi tạo pipeline `feature-extraction`.
- Hành vi:
  - Nếu `this.pipe` đã tồn tại thì không làm gì (idempotent).
  - Nếu tải thất bại, ném lỗi để caller xử lý.
  - Ghi log lúc bắt đầu và khi tải xong.

Lưu ý vận hành:
- Tải model có thể tốn thời gian và tài nguyên; gọi `init()` ở bước khởi động ứng dụng nếu có thể.

### generateEmbedding(text: string): Promise<number[]>

- Mục đích: trả về vector embedding (mảng số) cho `text`.
- Kiểm tra đầu vào: nếu `text` rỗng hoặc chỉ khoảng trắng, ném lỗi.
- Nếu `pipe` chưa được khởi tạo sẽ gọi `init()` tự động.
  - `pooling: 'mean'`: mỗi token (từ hoặc đơn vị con) có một vector embedding riêng; `mean` lấy trung bình các vector token đó để tạo một vector đại diện cho toàn câu. Nói ngắn: lấy trung bình các embedding token để thành embedding câu.
  - `normalize: true` chuẩn hóa vector (unit length), hữu ích khi dùng cosine similarity.
- Kết quả `result.data` thường là `Float32Array`; phương thức trả về `Array.from(result.data)` để dễ lưu/ghi và so sánh.
### generateEmbeddings(texts: string[]): Promise<number[][]>

- Mục đích: lần lượt tạo embedding cho từng chuỗi trong `texts`.
- Hiện tại phương thức làm tuần tự (vòng for..of). Khuyến nghị: nếu có nhiều văn bản, dùng `generateEmbeddingsBatch` để tận dụng batch processing và giảm overhead.

### generateEmbeddingsBatch(texts: string[]): Promise<number[][]>

- Mục đích: sinh embedding cho nhiều văn bản trong một lần gọi, tận dụng `@xenova/transformers` pipeline để xử lý batch.
- Hành vi:
  - Đảm bảo pipeline đã được khởi tạo (`init()` nếu cần).
  - Gọi `pipe(texts, { pooling: 'mean', normalize: true })` để nhận kết quả tensor-like.
  - Hỗ trợ cả hai dạng output:
    - `dims = [batch, dim]` (2D): mỗi hàng là một vector embedding.
    - `dims = [batch, seq_len, dim]` (3D): áp dụng average pooling theo `seq_len` để tạo embedding câu.
  - Trả về `number[][]` tương ứng với mỗi input.
  - Trong trường hợp lỗi hoặc định dạng bất thường, trả về zero vectors làm fallback (kèm log lỗi).
- Hiện tại phương thức làm tuần tự (vòng for..of). Nếu cần throughput cao hơn có thể thay bằng gọi song song hoặc batch nếu model/pipeline hỗ trợ.

## Trích dẫn lỗi & xử lý

- `init()` và `generateEmbedding()` đều bắt lỗi và log lỗi chi tiết rồi bubble lỗi lên caller (rethrow). Caller nên bắt và xử lý lỗi khi cần.

## Hiệu năng & vận hành

- Model `all-MiniLM-L6-v2` là một model nhẹ phù hợp cho embedding nhanh và chi phí thấp, nhưng vẫn cần tài nguyên (CPU / GPU / WASM) tùy môi trường.
- Khuyến nghị:
  - Gọi `await EmbeddingService.getInstance().init()` khi ứng dụng khởi động.
  - Tránh tải model nhiều lần (đó là lý do dùng singleton).
  - Nếu cần xử lý lượng lớn văn bản, cân nhắc batch hóa hoặc chạy nhiều worker.

## Cải tiến có thể thực hiện

- Batch inference: nếu `@xenova/transformers` hỗ trợ truyền mảng input, ta có thể tối ưu `generateEmbeddings` bằng calls theo batch.
- Song song hóa có kiểm soát: chạy nhiều request đồng thời với pool size cố định.
- Cache embedding cho các văn bản lặp lại (ví dụ key = hash(text)).

## Ví dụ sử dụng

```ts
import { EmbeddingService } from './embeddingService';

async function example() {
  const svc = EmbeddingService.getInstance();
  await svc.init();
  const emb = await svc.generateEmbedding('Hello world');
  console.log('embedding length', emb.length);

  const batch = await svc.generateEmbeddings(['A','B','C']);
  console.log('batch embeddings', batch.length);
}

example().catch(console.error);
```

## Cài đặt & môi trường

- `@xenova/transformers` có thể chạy trên nhiều backend (WASM, node, GPU) — xem tài liệu chính thức để cấu hình runtime phù hợp.
- Đảm bảo cài đặt đúng phiên bản và các phụ thuộc cần thiết trong `package.json`.

## Ghi chú cuối

- File mã nguồn `embeddingService.ts` đã xử lý hầu hết các kiểm tra cơ bản; tài liệu này nhằm giải thích hành vi, khuyến nghị vận hành và hướng mở rộng.
