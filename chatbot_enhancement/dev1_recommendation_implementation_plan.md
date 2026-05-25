# Plan Chi Tiết cho Dev 1 - Recommendation Pre-filter

> **Người phụ trách:** Dev 1
> **Ngày tạo:** 2026-05-25
> **Mục tiêu:** Implement Module A (Pre-filter)
> **Thời gian ước tính:** ~2-3 giờ

---

## Tổng quan công việc

Dev 1 cần thực hiện 2 phần chính:

1. **Task Xác minh** (T1, T2) - Làm trước khi code
2. **Module A: Pre-filter** (A1-A4) - Implement logic recommendation pre-filter

---

## PHẦN 1: Task Xác minh (Làm trước khi code)

### Step 1: T1 - Confirm field name `id` từ recommend API ✅ ĐÃ HOÀN THÀNH

**Mục tiêu:** Xác định field name trong response API là `"id"` hay `"_id"`.

**Kết quả:**

- Đã gọi API và xác nhận field name là `"id"` (không phải `"_id"`)
- Không cần sửa `mapRecommendationItem()` ở bước A2

---

### Step 2: T2 - Sync perPage default

**Mục tiêu:** Đảm bảo khi gọi recommend cho pre-filter, lấy đủ 100 IDs.

**Phân tích:**

- Backend mặc định `perPage=12`
- Chatbot config `CHATBOT_RECOMMEND_PAGE_SIZE` mặc định 8
- Pre-filter cần `perPage=100` để lấy đủ top 100 IDs

**Cách làm:**
Trong method `executeRecommendAndReturnIds()` (bước A2), luôn set `perPage=100`.

**File cần sửa:** `src/chatbot/services/recommendation.service.ts`

---

## PHẦN 2: Module A - Pre-filter

### Step 3: A1 - Thêm `id` vào `RecommendationItem` type ✅ ĐÃ HOÀN THÀNH

**File:** `src/chatbot/utils/recommendationState.ts`

**Thay đổi:**

```typescript
export type RecommendationItem = {
  id?: string; // <-- THÊM dòng này
  title?: string;
  acronym?: string;
  topics?: string[];
};
```

**Cách làm:**

1. Mở file `src/chatbot/utils/recommendationState.ts`
2. Tìm type `RecommendationItem`
3. Thêm field `id?: string;` vào đầu type
4. Field là optional (có `?`) để không break code cũ

**Verify:**

- [x] Type `RecommendationItem` có field `id`
- [x] Field là optional (có `?`)

---

### Step 4: A2.1 - Sửa `mapRecommendationItem()` để giữ lại `id` ✅ ĐÃ HOÀN THÀNH

**File:** `src/chatbot/services/recommendation.service.ts`

**Thay đổi:**

```typescript
function mapRecommendationItem(raw: any): RecommendationItem {
  const id = typeof raw?.id === "string" ? raw.id : "";
  const title = typeof raw?.title === "string" ? raw.title : "";
  const acronym = typeof raw?.acronym === "string" ? raw.acronym : "";
  const topics = extractTopics(raw?.topics);
  return { id, title, acronym, topics }; // <-- THÊM id vào return
}
```

**Cách làm:**

1. Mở file `src/chatbot/services/recommendation.service.ts`
2. Tìm function `mapRecommendationItem()`
3. Thêm dòng `const id = typeof raw?.id === "string" ? raw.id : "";` ở đầu function
4. Sửa return statement để bao gồm `id`

**Verify:**

- [x] Function vẫn return đúng type `RecommendationItem`
- [x] `id` được extract từ `raw.id`
- [x] Nếu `raw.id` không phải string → return empty string

---

### Step 5: A2.2 - Thêm method `executeRecommendAndReturnIds()` ✅ ĐÃ HOÀN THÀNH

**File:** `src/chatbot/services/recommendation.service.ts`

**Thay đổi:** Thêm method mới sau các method hiện có

```typescript
export async function executeRecommendAndReturnIds(params: {
  token?: string | null;
  topics?: string[];
  recentKeyword?: string;
}): Promise<{
  success: boolean;
  ids: string[];
  errorMessage?: string;
}> {
  try {
    // Gọi API với perPage=100 (hardcode theo T2)
    const response = await callRecommendationsForYou({
      token: params.token,
      page: 1,
      perPage: 100, // <-- Hardcode 100 cho pre-filter
    });

    if (!response || !response.payload || response.payload.length === 0) {
      return {
        success: false,
        ids: [],
        errorMessage: "No recommendations returned",
      };
    }

    // Extract IDs từ response
    const ids = response.payload
      .map((item) => item.id)
      .filter((id): id is string => typeof id === "string" && id.trim() !== "")
      .slice(0, 100); // Giới hạn tối đa 100 IDs

    if (ids.length === 0) {
      return {
        success: false,
        ids: [],
        errorMessage: "No valid IDs in recommendations",
      };
    }

    return { success: true, ids };
  } catch (error) {
    console.error("[executeRecommendAndReturnIds] Error:", error);
    return {
      success: false,
      ids: [],
      errorMessage: error instanceof Error ? error.message : "Unknown error",
    };
  }
}
```

**Cách làm:**

1. Mở file `src/chatbot/services/recommendation.service.ts`
2. Tìm vị trí phù hợp (sau các method hiện có, trước export)
3. Copy và paste code method trên
4. Đảm bảo import hoặc đã có `callRecommendationsForYou` function

**Verify:**

- [ ] Gọi API với `perPage=100`
- [ ] Trả về tối đa 100 IDs
- [ ] Nếu không có ID nào → `success=false`
- [ ] Giữ nguyên retry/rate-limit behavior (dùng lại `callRecommendationsForYou`)
- [ ] Guest fallback vẫn hoạt động (nếu `callRecommendationsForYou` đã handle)

---

### Step 6: A3 - Thêm flag `skipEntityDictionary` vào `RetrievalOptions`

**File:** `src/chatbot/services/rag/retrievalService.ts`

**Thay đổi 1:** Thêm field vào interface

```typescript
export interface RetrievalOptions {
  // ... các field cũ
  skipEntityDictionary?: boolean; // <-- THÊM dòng này
}
```

**Cách làm:**

1. Mở file `src/chatbot/services/rag/retrievalService.ts`
2. Tìm interface `RetrievalOptions`
3. Thêm `skipEntityDictionary?: boolean;` vào cuối interface

**Thay đổi 2:** Bọc block entity dictionary

```typescript
// Trong method `retrieve()`, khoảng lines 160-437
// Tìm block entity dictionary và bọc như sau:

// Chỉ áp dụng logic đặc biệt khi rõ ràng đây là truy vấn cho bảng conferences
if (this.conferenceEntityDictionary && isConferenceTableQuery) {
  // NEW: skip nếu đã pre-filter từ recommendation
  if (options.skipEntityDictionary) {
    console.log(
      "[RetrievalService] Skipping entity dictionary matching (pre-filtered by recommendation).",
    );
  } else {
    // ... nguyên block entity dictionary cũ (lines 162-437)
  }
}
```

**Cách làm:**

1. Tìm method `retrieve()` trong file
2. Tìm block entity dictionary (khoảng lines 160-437)
3. Bọc block đó trong if-else như code mẫu
4. Thêm console log khi skip

**Verify:**

- [ ] Khi `skipEntityDictionary=true`, bỏ qua hoàn toàn entity matching
- [ ] Khi `skipEntityDictionary=false/undefined`, behavior giống hệt cũ
- [ ] Không ảnh hưởng đến `hybridRetrieve()` hoặc `buildCompactConferenceList()`

---

### Step 7: A4 - Thêm logic pre-filter vào `retrieveKnowledge.handler.ts`

**File:** `src/chatbot/handlers/retrieveKnowledge.handler.ts`

**Thay đổi 1:** Import function mới

```typescript
// Thêm import ở đầu file
import { executeRecommendAndReturnIds } from "../services/recommendation.service";
```

**Thay đổi 2:** Thêm logic pre-filter vào đầu method `execute()`

```typescript
// --- NEW: Recommendation Pre-filter ---
let useRecFilter = args.useRecommendationFilter === true;
let isListQuery = listMode && isConferenceTableQuery;
let skipEntityDict = false;

if (useRecFilter && isListQuery && userToken) {
  try {
    onStatusUpdate("status_update", {
      step: "fetching_recommendations",
      message: "Personalizing conference results...",
      agentId,
    });

    const recResult = await executeRecommendAndReturnIds({
      token: userToken,
      topics: extractTopicsFromQuery(query),
      recentKeyword: query,
    });

    if (recResult.success && recResult.ids.length > 0) {
      // Inject 100 IDs vào filter
      effectiveFilter = {
        ...effectiveFilter,
        id: recResult.ids.slice(0, 100),
      };
      // Skip entity dictionary
      skipEntityDict = true;

      console.log(
        "[RetrieveKnowledgeHandler] Pre-filtered by recommendation:",
        {
          idsCount: recResult.ids.length,
          filter: effectiveFilter,
        },
      );
    } else {
      // Recommend fail → fallback: vẫn chạy RAG thuần
      console.warn(
        "[RetrieveKnowledgeHandler] Recommendation pre-filter failed, falling back to plain RAG:",
        recResult.errorMessage,
      );
    }
  } catch (error) {
    console.error(
      "[RetrieveKnowledgeHandler] Error during recommendation pre-filter:",
      error,
    );
    // Fallback: RAG thuần
  }
}
// --- END NEW ---
```

**Thay đổi 3:** Thêm option khi gọi `retrievalService.retrieve()`

```typescript
// Tìm chỗ gọi this.retrievalService.retrieve()
// Thêm skipEntityDictionary option:

const results = await this.retrievalService.retrieve(query, {
  filter: effectiveFilter,
  limit,
  keywordWeight: 0.3,
  vectorWeight: 0.7,
  listMode,
  skipEntityDictionary: skipEntityDict, // <-- THÊM dòng này
});
```

**Thay đổi 4:** Thêm helper function `extractTopicsFromQuery()`

```typescript
// Thêm helper function (nếu chưa có)
function extractTopicsFromQuery(query: string): string[] {
  // Simple heuristic: extract capitalized words that might be topics
  // Có thể cải thiện sau với NLP
  const words = query.split(/\s+/);
  const topics = words.filter(
    (w) =>
      w.length > 2 &&
      /^[A-Z]/.test(w) &&
      !/^(the|a|an|for|with|about|find|show|list|search)/i.test(w),
  );
  return topics;
}
```

**Cách làm:**

1. Mở file `src/chatbot/handlers/retrieveKnowledge.handler.ts`
2. Thêm import `executeRecommendAndReturnIds`
3. Tìm method `execute()`
4. Thêm logic pre-filter ở đầu method (sau khi đã có `listMode`, `isConferenceTableQuery`, `effectiveFilter`)
5. Tìm chỗ gọi `retrievalService.retrieve()` và thêm option
6. Thêm helper function `extractTopicsFromQuery()` nếu chưa có

**Verify:**

- [ ] Khi `useRecommendationFilter=true + listMode`: gọi recommend → inject filter → skip entity dict
- [ ] Khi `useRecommendationFilter=false`: behavior giống hệt cũ
- [ ] Khi recommend API lỗi → chạy RAG thuần, user không thấy lỗi
- [ ] Khi recommend trả về 0 IDs → chạy RAG thuần
- [ ] Status update "Personalizing conference results..." hiển thị cho user
- [ ] Console log ghi rõ khi recommend thành công/fail

---

## PHẦN 3: Testing & Verification

### Step 8: Integration Test Checklist

Sau khi hoàn thành tất cả các step, chạy checklist sau:

**Test Module A:**

- [ ] User: "Tìm hội nghị AI" → request `/for-you` được gọi với `perPage=100`
- [ ] Response có `id` → filter được inject vào RAG
- [ ] Entity dictionary không chạy (confirm bằng console log)
- [ ] RAG trả về kết quả trong pool 100 IDs
- [ ] User: "Xem thông tin ICML" → `useRecommendationFilter=false` → không gọi recommend
- [ ] Tắt recommend API → "tìm hội nghị AI" vẫn trả về RAG thuần

**Console Logs Check:**

- [ ] `[RetrieveKnowledgeHandler] Pre-filtered by recommendation:` xuất hiện khi recommend thành công
- [ ] `[RetrieveKnowledgeHandler] Recommendation pre-filter failed, falling back to plain RAG:` xuất hiện khi recommend fail
- [ ] `[RetrievalService] Skipping entity dictionary matching (pre-filtered by recommendation).` xuất hiện khi skip entity dict

---

## PHẦN 4: Dependencies & Coordination

### Dependencies với Dev 3:

- Dev 3 cần cập nhật `retrieveKnowledge` function declaration để thêm `useRecommendationFilter` parameter
- **Coordinate:** Dev 3 hoàn thành C1 trước, sau đó Dev 1 mới bắt đầu Step 7

### Merge Order:

1. Dev 1: Step 1-7 (Module A)
2. Dev 3: C1-C3 (Prompt updates)

---

## Tóm tắt Step-by-Step

| Step     | Mô tả                                      | File                         | Ước tính thời gian |
| -------- | ------------------------------------------ | ---------------------------- | ------------------ |
| 1        | T1 - Confirm field name API                | -                            | ✅ Đã xong         |
| 2        | T2 - Sync perPage default                  | recommendation.service.ts    | 10 phút            |
| 3        | A1 - Thêm `id` vào type                    | recommendationState.ts       | 5 phút             |
| 4        | A2.1 - Sửa `mapRecommendationItem`         | recommendation.service.ts    | 10 phút            |
| 5        | A2.2 - Thêm `executeRecommendAndReturnIds` | recommendation.service.ts    | 30 phút            |
| 6        | A3 - Thêm `skipEntityDictionary`           | retrievalService.ts          | 20 phút            |
| 7        | A4 - Thêm logic pre-filter                 | retrieveKnowledge.handler.ts | 45 phút            |
| 8        | Integration Test                           | -                            | 30 phút            |
| **Tổng** |                                            |                              | **~2.5 giờ**       |

---

## Notes

- **Quan trọng:** Đợi Dev 3 hoàn thành C1 trước khi làm Step 7
- **Testing:** Mỗi step xong nên test nhanh trước khi tiếp tục
- **Logs:** Sử dụng console log để debug khi gặp lỗi
- **Fallback:** Luôn có fallback khi recommend fail → không crash app
