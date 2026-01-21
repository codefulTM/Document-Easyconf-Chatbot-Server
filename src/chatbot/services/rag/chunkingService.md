# ChunkingService

## Mục đích

`ChunkingService` là một service dùng để chia nhỏ (chunk) nội dung Markdown hoặc HTML thành các đoạn nhỏ (chunk) hợp lý, phục vụ cho các bài toán RAG (Retrieval-Augmented Generation), embedding, hoặc lưu trữ dữ liệu văn bản lớn. Việc chia nhỏ giúp tối ưu hóa hiệu suất tìm kiếm, nhúng (embedding) và xử lý văn bản với các mô hình AI.

---

## Định nghĩa các kiểu dữ liệu

### Chunk

```typescript
export interface Chunk {
  content: string;
  metadata: {
    source?: string;
    tableName?: string;
    recordId?: string | number;
    heading?: string;
    [key: string]: any;
  };
  tokenCount?: number;
}
```
- `content`: Nội dung đoạn chunk (bao gồm heading và nội dung).
- `metadata`: Thông tin bổ sung về nguồn gốc, heading, bảng dữ liệu, v.v.
- `tokenCount`: Số lượng token (ước lượng, không bắt buộc).

---

## Các thuộc tính chính

- `MAX_CHUNK_SIZE`: Kích thước tối đa của một chunk (ký tự). Mặc định: 1500.
- `MIN_CHUNK_SIZE`: Kích thước tối thiểu của một chunk (ký tự). Mặc định: 100.
- `OVERLAP_SIZE`: Số ký tự chồng lặp giữa các chunk để đảm bảo ngữ cảnh. Mặc định: 200.
- `md`: Đối tượng MarkdownIt để chuyển đổi Markdown sang HTML.

---

## Các phương thức

### constructor()

Khởi tạo service, cấu hình MarkdownIt để hỗ trợ HTML, xuống dòng và tự động nhận diện link.

---

### chunkMarkdown(markdown: string, baseMetadata?: Partial<Chunk["metadata"]>): Chunk[]

- **Chức năng:**  
  Chia nhỏ nội dung Markdown thành các chunk.
- **Tham số:**
  - `markdown`: Chuỗi Markdown đầu vào.
  - `baseMetadata`: Metadata mặc định gắn vào từng chunk (tùy chọn).
- **Trả về:**  
  Mảng các chunk đã chia nhỏ.

**Quy trình:**
1. Chuyển Markdown sang HTML.
2. Gọi `chunkHtml` để xử lý tiếp.

---

### chunkHtml(html: string, baseMetadata?: Partial<Chunk["metadata"]>): Chunk[]

- **Chức năng:**  
  Chia nhỏ nội dung HTML thành các chunk.
- **Tham số:**
  - `html`: Chuỗi HTML đầu vào.
  - `baseMetadata`: Metadata mặc định gắn vào từng chunk (tùy chọn).
- **Trả về:**  
  Mảng các chunk đã chia nhỏ.

**Quy trình:**
1. Dùng cheerio để parse HTML.
2. Gọi `flattenDom` để chuyển DOM thành danh sách tuần tự các item (heading/content).
3. Gom nhóm các item thành các chunk dựa trên kích thước và ngữ nghĩa (heading).
4. Đảm bảo các chunk không vượt quá `MAX_CHUNK_SIZE`, có chồng lặp `OVERLAP_SIZE` nếu cần.
5. Gắn heading vào mỗi chunk.

---

### flattenDom($: cheerio.CheerioAPI, root: any): { type: "heading" | "content"; text: string }[]

- **Chức năng:**  
  Chuyển DOM HTML thành danh sách tuần tự các item kiểu `{ type, text }`, phân biệt heading và content.
- **Tham số:**
  - `$`: Đối tượng cheerio.
  - `root`: Node gốc để bắt đầu duyệt.
- **Trả về:**  
  Mảng các item dạng `{ type: "heading" | "content", text: string }`.

**Chi tiết:**
- Các thẻ tiêu đề (`h1`–`h6`) được nhận diện và tách ra riêng, dùng làm `heading` cho các chunk.
- Các khối nội dung (ví dụ: `p`, `li`, `div`) được thu thập theo thứ tự xuất hiện và ghi thành mục `content`.
- Duyệt DOM theo thứ tự xuất hiện (đệ quy) để giữ đúng thứ tự và ngữ cảnh của văn bản.
- Bỏ qua những node không mang nội dung hữu ích (ví dụ: `br`) để tránh tạo các chunk rỗng.

---

### pushChunk(chunks: Chunk[], content: string, heading: string, baseMetadata?: Partial<Chunk["metadata"]>)

- **Chức năng:**  
  Thêm một chunk mới vào mảng chunk, kèm metadata và kiểm tra kích thước tối thiểu.
- **Tham số:**
  - `chunks`: Mảng chunk hiện tại.
  - `content`: Nội dung chunk.
  - `heading`: Tiêu đề (heading) của chunk.
  - `baseMetadata`: Metadata mặc định (tùy chọn).

---

## Quy tắc chunking

- Mỗi chunk thường bắt đầu bằng một heading (nếu có).
- Nếu gặp heading mới, chunk hiện tại sẽ được "flush" (đẩy vào mảng).
- Nếu chunk vượt quá `MAX_CHUNK_SIZE`, sẽ cắt và chồng lặp `OVERLAP_SIZE` ký tự cuối của chunk trước vào chunk sau.
- Chunk quá nhỏ (dưới `MIN_CHUNK_SIZE`) sẽ bị bỏ qua (trừ khi là chunk đầu tiên).
- Metadata có thể mở rộng tùy ý (ví dụ: nguồn, bảng dữ liệu, id bản ghi...).

---

## Ứng dụng

- Chia nhỏ tài liệu lớn để nhúng (embedding) với mô hình ngôn ngữ.
- Tối ưu hóa truy vấn tìm kiếm ngữ nghĩa (semantic search).
- Chuẩn bị dữ liệu cho các hệ thống RAG, chatbot, hoặc lưu trữ phân mảnh.

---

## Ví dụ sử dụng

```typescript
const chunkingService = new ChunkingService();
const markdown = "# Tiêu đề\nNội dung đoạn 1\n## Mục nhỏ\nNội dung đoạn 2";
const chunks = chunkingService.chunkMarkdown(markdown, { source: "file.md" });
console.log(chunks);
/*
[
  {
    content: "Tiêu đề\n\nNội dung đoạn 1",
    metadata: { source: "file.md", heading: "Tiêu đề" }
  },
  {
    content: "Mục nhỏ\n\nNội dung đoạn 2",
    metadata: { source: "file.md", heading: "Mục nhỏ" }
  }
]
*/
```

---

## Ghi chú

- Việc ước lượng token dựa trên số ký tự (1 token ≈ 4 ký tự tiếng Anh), có thể không chính xác với các ngôn ngữ khác.
- Có thể mở rộng thêm logic chunking cho các loại tài liệu đặc biệt (bảng, code block, v.v.).
- Đảm bảo dữ liệu đầu vào hợp lệ (Markdown/HTML chuẩn) để chunking hiệu quả.
