# Refactoring Reports

## 1. Bỏ matchType, chuyển sang dùng score cho Retrieval Service

### Vấn đề

- `matchType` chỉ có 2 giá trị: `title_or_acronym` và `summary_or_topics` - quá đơn giản
- `matchType` không phản ánh hybrid search - khi hybrid search, tất cả các trường được search cùng lúc → không biết match theo trường nào
- `matchType` chỉ có ý nghĩa cho entity dictionary matching, không phản ánh hybrid search
- Hybrid search results không có matchType → LLM không biết độ phù hợp

### Giải pháp

Chuyển sang dùng score để ưu tiên kết quả:

- Boost score theo entity dictionary match
- Handler chỉ dùng score để sắp xếp
- LLM ưu tiên kết quả có score cao

### Thay đổi chi tiết

#### 1. Bỏ matchType logic trong retrievalService.ts

- **strongDictionaryMatch fallback** (dòng 398-410):
  - Trước: Gán `matchType: "title_or_acronym"` vào metadata
  - Sau: Gán `score: 1.0` vào metadata (không boost, luôn cao nhất)
- **fuzzyAmbiguousCandidate** (dòng 453-465):
  - Trước: Gán `matchType: "title_or_acronym"` vào metadata
  - Sau: Gán `score: 0.6` vào metadata (ưu tiên trung bình)

- **Bỏ annotateConferenceMatchTypes** (dòng 545-532):
  - Comment lại toàn bộ block gọi `annotateConferenceMatchTypes`
  - Không cần gán matchType nữa

#### 2. Bỏ matchType logic trong retrieveKnowledge.handler.ts (dòng 256-283)

- Bỏ logic đếm `directCount` và `relatedCount` dựa trên matchType
- Bỏ logic gán `matchLabel` dựa trên matchType
- Thay `matchType` bằng `score` trong enrichedItems
- Bỏ `directMatches` và `relatedMatches` từ payload
- Payload chỉ có `score`, không có `matchType`

#### 3. Điều chỉnh hybridRetrieve để xử lý query undefined (dòng 950-989)

- **Skip vector search khi query undefined hoặc hasFilterById**:
  - Gộp điều kiện: `if (!query || hasFilterById)`
  - Skip vector search và combineAndRank khi không có query hoặc có filter.id
- **Return keyword results trực tiếp**:
  - Khi query undefined hoặc hasFilterById, return keyword results
  - Thêm score vào metadata cho tất cả chunk

#### 4. Thêm score vào metadata ở tất cả return paths trong hybridRetrieve

- **Nhánh keywordResults** (dòng 982-988):

  ```typescript
  return keywordResults.map((item) => ({
    ...item.chunk,
    metadata: {
      ...item.chunk.metadata,
      score: item.score,
    },
  }));
  ```

- **Nhánh combinedResults** (dòng 1003-1009):
  ```typescript
  return combinedResults.map((item) => ({
    ...item.chunk,
    metadata: {
      ...item.chunk.metadata,
      score: item.score,
    },
  }));
  ```

#### 5. Sửa keywordSearch để cho phép query undefined (dòng 1137-1143)

- **Signature**: `query: string` → `query?: string` (hoặc `query: string | undefined`)
- **Handle undefined**: `const keywords = query ? TextUtils.extractKeywords(query) : []`
- **Logic hiện tại đã có sẵn**:
  - Khi `keywords = []` → `hasKeyword = false`
  - SQL dùng `"1"` thay vì `similarity(ch.content, $1)` khi `hasKeyword = false`
  - Score = 1 cho tất cả chunk khi không có keyword

#### 6. Chuyển options thành optional trong hybridRetrieve (dòng 925-934)

- **Signature**: `options: RetrievalOptions` → `options?: RetrievalOptions`
- **Destructuring**: `} = options;` → `} = options || {};`
- Cho phép gọi `hybridRetrieve(query)` mà không cần options

#### 7. Sửa điều kiện skip entity dictionary matching (dòng 143)

- **Trước**: `if (listMode || hasConferenceDateFilter || hasFilterById)`
- **Sau**: `if (listMode || hasConferenceDateFilter || (hasFilterById && !query))`
- Khi có filter.id và có query → vẫn chạy entity dictionary matching để boost score

#### 8. Comment lại fallbackConferenceSearchFromDb (dòng 496-532)

- Entity dictionary matching đã đủ mạnh
- Tránh redundancy với keyword search
- Comment lại toàn bộ block

#### 9. Comment lại summary logging trong hybridRetrieve (dòng 971-1017)

- Tránh performance impact từ DB fetch
- Comment lại toàn bộ block

#### 10. Sửa lỗi TypeScript trong findBestConferenceMatch (dòng 314)

- **Trước**: `findBestConferenceMatch(query, ...)`
- **Sau**: `findBestConferenceMatch(query || "", ...)`
- Fix lỗi TypeScript khi query có thể là undefined

#### 11. Sửa lỗi syntax trong fallbackConferenceSearchFromDb (dòng 1327)

- **Trước**: `return? []`
- **Sau**: `return []`
- Fix syntax error

### Kết quả

- Hệ thống đơn giản hơn, chỉ dùng score để ưu tiên kết quả
- Entity dictionary match được ưu tiên tự nhiên qua score cao
- LLM chỉ cần ưu tiên theo score
- Phản ánh đúng hybrid search (score aggregate)
- Tránh redundancy và performance impact

### Files thay đổi

- `src/chatbot/services/rag/retrievalService.ts`
- `src/chatbot/handlers/retrieveKnowledge.handler.ts`
