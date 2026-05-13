# P1-01: Result Set State V1 — Implementation Plan

## 1. Tổng quan

**Mã work package:** R1-01
**Ưu tiên:** Số 1 (cao nhất)
**Scope:** In-scope (bắt buộc)
**Liên quan tới:** P1-02 (Resolver — module mới), P2-04 (3-Layer Memory — Warm Memory), system prompt (hướng dẫn LLM dùng tool resolveConferenceRef)

### Mục tiêu

Tạo một module lưu trạng thái danh sách kết quả conference theo conversation, để Resolver (P1-02) có nguồn sự thật xác định tham chiếu thứ tự ("cái thứ 2", "cái cuối", v.v.) mà không cần LLM suy đoán.

**Cách tiếp cận:** LLM truyền `identifierType="ordinal"` với `identifier` là số (1-based, âm cho đếm từ cuối). Resolver xử lý số → index 0-based → lấy ID từ list thật.

| identifier | Ý nghĩa |
|---|---|
| `1` | Phần tử đầu tiên |
| `2` | Phần tử thứ 2 |
| `-1` | Phần tử cuối |
| `-2` | Phần tử áp cuối |

---

## 2. Kiến trúc Resolver (P1-02) — 2 tầng

Resolver có 2 tầng xử lý, hoạt động tuần tự:

### Tầng 1: Fast Path — Resolve tất cả (trong preToolValidator)

LLM gọi tool mutating với `identifierType="ordinal"`. preToolValidator tự động resolve số → ID thật, không cần thêm vòng LLM.

```
[preToolValidator] nhận manageFollow(identifier="2", identifierType="ordinal")
  → gọi ResultSetResolver.resolveAll(convId, 2)   // 1-based
    → quét TẤT CẢ result set còn hạn trong conversation
    → thử resolve số 2 trên từng list → index = 1 (0-based)
    → kết quả:
      • 0 list match → out_of_range_reference (block)
      • 1 list match duy nhất → resolve, thay identifier=ID, allowed=true  ✅ 1 turn
      • 2+ list match → ambiguity_blocked_mutation (block)
```

### Tầng 2: Slow Path — Tool mới `resolveConferenceRef` (cho trường hợp mơ hồ)

Khi Tầng 1 block với `ambiguity_blocked_mutation`, LLM thấy lỗi → dùng tool `resolveConferenceRef` cung cấp thêm ngữ cảnh.

```
[LLM] thấy ambiguity_blocked_mutation
  → gọi resolveConferenceRef(ordinal=2, contextHint="AI conferences")
    → handler cosine similarity match với queryEmbedding
    → resolve ra conf_002
    → trả về
  → LLM gọi manageFollow(identifier="conf_002", identifierType="id")
    → preToolValidator thấy ID thật → pass  ✅ 2 turns (nhưng hiếm)
```

### Tổng kết: Resolve Strategy

| Kịch bản                        | Tầng      | Số vòng LLM | Cơ chế                               |
| ------------------------------- | --------- | ----------- | ------------------------------------ |
| 1 list trong conversation       | Fast Path | 1           | Resolve tất cả → chỉ 1 match         |
| Nhiều list, ordinal chỉ match 1 | Fast Path | 1           | Resolve tất cả → chỉ 1 match         |
| Nhiều list, ordinal match nhiều | Slow Path | 2           | Block → LLM gọi tool với contextHint |
| Không list nào match            | Fast Path | 1 (error)   | Block → báo user không tìm thấy      |
| Tất cả list đều stale           | Fast Path | 1 (error)   | Block → stale_result_set             |

---

## 3. Data Model (Schema)

Thay đổi so với contract gốc: **bỏ `query_signature`, thêm `query_text`**.

- `query_signature` là hash 1 chiều → không match được với context từ LLM
- `query_text` lưu nguyên văn → có thể so khớp với contextHint

```typescript
interface ResultSetState {
  /** Conversation ID */
  conversationId: string;

  /** Câu query gốc (vd: "list AI conferences 2026") — dùng để match contextHint */
  queryText: string;

  /** Vector embedding của queryText để semantic matching với contextHint */
  queryEmbedding: number[];

  /** Danh sách ID conference có thứ tự — nguồn sự thật duy nhất cho ordinal resolver */
  orderedConferenceIds: string[];

  /** Thời điểm tạo (ISO-8601 UTC) */
  createdAt: string;

  /** Thời điểm hết hạn (ISO-8601 UTC) — sau 20 phút kể từ createdAt */
  expiresAt: string;

  /** Nguồn tạo (vd: "retrieval_pipeline_v1") */
  source: string;
}
```

**Collection:** `result_set_states`
**TTL Index:** `expiresAt` (20 phút)

---

## 4. API / Interface

```typescript
interface IResultSetStateStore {
  /** Lưu result set mới. Nếu đã tồn tại conversationId + queryText -> ghi đè. */
  save(
    conversationId: string,
    queryText: string,
    queryEmbedding: number[],
    orderedConferenceIds: string[],
    source?: string,
  ): Promise<void>;

  /** Lấy tất cả result set còn hạn của 1 conversation, sắp xếp mới nhất lên đầu */
  getAllValid(conversationId: string): Promise<ResultSetState[]>;

  /** Xóa toàn bộ result set của conversation */
  clearConversation(conversationId: string): Promise<void>;
}
```

```typescript
interface IResultSetResolver {
  /**
   * Resolve ordinal reference trên TẤT CẢ result set còn hạn.
   * ordinal là số 1-based (dương = đếm xuôi, âm = đếm từ cuối, -1 = cuối).
   * Trả về kết quả chi tiết để preToolValidator quyết định block hay pass.
   */
  resolveAll(
    conversationId: string,
    ordinal: number,
  ): Promise<ResolveAllResult>;

  /**
   * Resolve ordinal với contextHint cụ thể (từ LLM).
   * Match bằng cosine similarity giữa contextEmbedding và queryEmbedding.
   * Chọn result set có similarity cao nhất > threshold.
   * Dùng trong handler của tool resolveConferenceRef.
   */
  resolveByContext(
    conversationId: string,
    contextEmbedding: number[],
    ordinal: number,
  ): Promise<ResolveResult>;

  /** Helper sinh embedding cho 1 text string (tái sử dụng embedding service có sẵn) */
  generateEmbedding(text: string): Promise<number[]>;
}
```

```typescript
type ResolveResult = {
  resolvedId: string | null;
  reasonCode: "resolved" | "out_of_range" | "stale" | "no_match";
  confidence: "high" | "low";
};

type ResolveAllResult = {
  /** Danh sách tất cả kết quả resolve được (có thể 0, 1, hoặc nhiều) */
  matches: Array<{
    state: ResultSetState;
    resolvedId: string;
  }>;
  /** Nếu chỉ có 1 match duy nhất -> dùng cái này */
  uniqueMatch: string | null;
  /** Nếu 0 match -> out_of_range */
  totalStale: number;
};
```

---

## 5. Tool mới: `resolveConferenceRef`

### 5.1 Function Declaration

File mới: function declarations.

```typescript
export const englishResolveConferenceRefDeclaration: FunctionDeclaration = {
  name: "resolveConferenceRef",
  description:
    "Resolves an ordinal or ambiguous conference reference to a specific conference ID. " +
    "Use this when the system blocked a mutation due to ambiguity (2+ result lists match). " +
    "Provide the ordinal number and a context hint describing which search result list the user is referring to. " +
    "The system will return the exact conference ID you can use in subsequent tool calls.",
  parameters: {
    type: Type.OBJECT,
    properties: {
      ordinal: {
        type: Type.NUMBER,
        description:
          "The ordinal position of the conference, 1-based. " +
          "Positive = count from start (1 = first). " +
          "Negative = count from end (-1 = last, -2 = second to last). " +
          "Examples: 2 for 'thứ 2', -1 for 'cuối', 1 for 'đầu tiên'.",
      },
      contextHint: {
        type: Type.STRING,
        description:
          "A brief description of which search result list the user is referring to, based on conversation context. " +
          "Example: If the user previously searched for 'AI conferences 2026' and now says 'the 2nd one', " +
          "use 'AI conferences 2026' as the contextHint. The system will match this against previous search queries.",
      },
      action: {
        type: Type.STRING,
        description: "The intended action (follow, unfollow, add, remove).",
        enum: ["follow", "unfollow", "add", "remove"],
      },
    },
    required: ["ordinal", "contextHint", "action"],
  },
};
```

### 5.2 Handler

**File:** `src/chatbot/handlers/resolveConferenceRef.handler.ts`

```typescript
class ResolveConferenceRefHandler implements IFunctionHandler {
  async execute(context: FunctionHandlerInput): Promise<FunctionHandlerOutput> {
    const { args } = context;
    const conversationId = this.extractConversationId(context);
    const ordinal = args.ordinal as number;       // number, 1-based
    const contextHint = args.contextHint as string;

    // Sinh embedding cho contextHint để semantic matching
    const contextEmbedding =
      await resultSetResolver.generateEmbedding(contextHint);

    // Bước 1: Cosine similarity giữa contextEmbedding và queryEmbedding
    const contextMatch = await resultSetResolver.resolveByContext(
      conversationId,
      contextEmbedding,
      ordinal,
    );
    if (contextMatch.resolvedId) {
      return {
        modelResponseContent: JSON.stringify({
          resolved: true,
          conferenceId: contextMatch.resolvedId,
          confidence: "high",
          matchType: "semantic",
        }),
      };
    }

    // Bước 2: Fallback — resolveAll
    const allMatch = await resultSetResolver.resolveAll(
      conversationId,
      ordinal,
    );
    if (allMatch.uniqueMatch) {
      return {
        modelResponseContent: JSON.stringify({
          resolved: true,
          conferenceId: allMatch.uniqueMatch,
          confidence: "medium",
          matchType: "fallback_all",
        }),
      };
    }

    // Không resolve được
    return {
      modelResponseContent: JSON.stringify({
        resolved: false,
        message: "Could not resolve the conference reference.",
      }),
    };
  }
}
```

### 5.3 Register vào functionRegistry

```typescript
// functionRegistry.ts
const functionRegistry: Record<string, IFunctionHandler> = {
  // ... existing handlers ...
  resolveConferenceRef: new ResolveConferenceRefHandler(),
};
```

---

## 6. Integration Points

### 6.1 Router Integration (Save)

**File:** `src/chatbot/handlers/retrieveKnowledge.handler.ts`
**Dòng:** 161-204 (trong khối `if (isConferenceListMode)`, trước `return`)

```typescript
if (isConferenceListMode) {
  const compactItems = await this.retrievalService.buildCompactConferenceList(
    results.map((r) => r.originalChunk),
    { maxItems: limit },
  );

  // >>> LƯU RESULT SET <<<
  const ids = compactItems.map((item) => item.id);
  const embedding = await this.resultSetResolver.generateEmbedding(query);
  await this.resultSetStateStore.save(conversationId, query, embedding, ids);

  const compactPayload = {
    mode: "conference_list",
    count: compactItems.length,
    items: compactItems,
  };
  // ...
  return { modelResponseContent: compactText };
}
```

**Khi nào gọi save:**

- Chỉ gọi khi `listMode=true` và `isConferenceListMode === true`
- Chỉ gọi khi retrieve thành công, `compactItems.length > 0`
- `queryText` = chính là `query` string (không hash)
- `queryEmbedding` = embedding vector của `query` (sinh bằng embedding service tái sử dụng từ codebase)
- Cần inject `IResultSetStateStore` và `IResultSetResolver` vào constructor của `RetrieveKnowledgeHandler`

### 6.2 Fast Path Integration (preToolValidator)

**File:** `src/chatbot/guards/preToolValidator.ts`
**Dòng:** 426-515 (trong hàm `validateMutationArgs`, trước `hasAmbiguousReference` check)

```typescript
if (isMutatingAction) {
  // ... check identifier, identifierType ...

  // >>> FAST PATH: resolveAll TRƯỚC KHI BLOCK <<<
  // identifierType === "ordinal" → LLM đã extract số, Resolver xử lý
  if (identifierType?.trim() === "ordinal") {
    const ordinalNum = parseInt(identifier as string, 10);
    if (isNaN(ordinalNum)) {
      // Identifier không phải số → parse lỗi, block
      return buildBlockedResult({
        errorCode: OrchestrationSafetyErrorCode.INVALID_TOOL_ARGS,
        message: `${functionName} received an invalid ordinal identifier.`,
        userMessage: "Số thứ tự không hợp lệ. Vui lòng thử lại.",
      });
    }

    const result = await resultSetResolver.resolveAll(
      conversationId,
      ordinalNum,
    );

    if (result.uniqueMatch) {
      // Chỉ 1 match -> resolve ngay, không cần LLM
      normalizedArgs.identifier = result.uniqueMatch;
      normalizedArgs.identifierType = "id";
      return {
        allowed: true,
        node: PRE_TOOL_VALIDATOR_NODE,
        normalized_args: normalizedArgs,
      };
    }

    if (result.matches.length === 0) {
      // 0 match -> out of range
      return buildBlockedResult({
        errorCode: OrchestrationSafetyErrorCode.OUT_OF_RANGE_REFERENCE,
        message: "...",
        userMessage: "...",
      });
    }
    // 2+ match -> fallback xuống ambiguity check bên dưới
  }

  if (hasAmbiguousReference(identifier)) {
    return buildBlockedResult({
      errorCode: OrchestrationSafetyErrorCode.AMBIGUITY_BLOCKED_MUTATION,
      // ...
    });
  }
}
```

**Chú ý:** Chỉ chạy resolveAll khi `identifierType === "ordinal"`. Các identifierType khác (`"acronym"`, `"title"`, `"id"`) không chạy resolveAll, để ambiguity check xử lý như hiện tại.

### 6.3 Fast Path — Cập nhật identifierType enum trong tool declarations cũ

**File:** Tất cả các file language functions (`english.ts`, `vietnamese.ts`, `spanish.ts`, ...)

Thêm `"ordinal"` vào enum `identifierType` của các mutation tool:
- `manageFollow`
- `manageCalendar`
- `manageBlacklist`
- `rateConference`
- `getConferenceFeedback`
- `countConferenceFollowed`

```typescript
identifierType: {
  type: Type.STRING,
  enum: ["acronym", "title", "id", "ordinal"],  // + "ordinal"
  description: "...",
}
```

Việc này cho phép Gemini sinh `identifierType="ordinal"` khi user dùng số thứ tự.

### 6.4 Slow Path Integration (tool resolveConferenceRef)

**File:** `src/chatbot/language/functions/english.ts` (và các file language khác)
**File:** `src/chatbot/gemini/functionRegistry.ts`

- Thêm tool declaration mới `resolveConferenceRef` (xem Section 5)
- Register handler trong functionRegistry
- System prompt: thêm 1 đoạn hướng dẫn LLM dùng tool này

### 6.5 Warm Memory Integration (P2-04)

```
MemoryManagerLite
  -> warm_memory.result_set_ref = conversationId:queryText
  -> warm_memory.active_entities = orderedConferenceIds
```

---

## 7. Thay đổi cần thiết

### 7.1 Thêm "ordinal" vào identifierType enum của các mutation tool

**File:** Tất cả file language functions (`english.ts`, `vietnamese.ts`, `spanish.ts`, ...)

Thêm `"ordinal"` vào enum của các tool có identifierType:
- `manageFollow`
- `manageCalendar`
- `manageBlacklist`
- `rateConference`
- `getConferenceFeedback`
- `countConferenceFollowed`

### 7.2 Thêm tool declaration mới `resolveConferenceRef` + handler

- File: `src/chatbot/language/functions/english.ts` — thêm `englishResolveConferenceRefDeclaration`
- File: `src/chatbot/language/functions/vietnamese.ts` — thêm tương tự
- File: `src/chatbot/gemini/functionRegistry.ts` — register handler

### 7.3 Thêm 1 đoạn trong system prompt (tất cả ngôn ngữ)

**File:** `src/chatbot/language/instructions/vietnamese.ts`
**File:** `src/chatbot/language/instructions/english.ts`
**File:** `src/chatbot/language/instructions/spanish.ts`

Thêm đoạn (ví dụ tiếng Việt):

```
Khi người dùng dùng tham chiếu mơ hồ như "thứ 2", "cái cuối", "cái đầu tiên"
để chỉ một hội nghị trong danh sách kết quả tìm kiếm trước đó:
1. Chuyển đổi tham chiếu thành số:
   - "thứ 2", "thứ hai" → 2
   - "đầu tiên", "cái đầu" → 1
   - "cuối", "cái cuối" → -1
   - "thứ N" → N
2. Gọi tool quản lý (manageFollow, manageCalendar, manageBlacklist...)
   với identifierType="ordinal" và identifier là số vừa chuyển.
3. Nếu bị chặn với lỗi "ambiguity_blocked_mutation", hãy dùng hàm
   resolveConferenceRef với ordinal (số), contextHint, và action.
4. Sau khi nhận được conferenceId từ resolveConferenceRef, gọi lại tool quản lý
   với identifier=conferenceId và identifierType="id".
```

---

## 8. Storage Layer

### MongoDB Collection: `result_set_states`

```javascript
// Indexes
db.result_set_states.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
db.result_set_states.createIndex({ conversationId: 1, createdAt: -1 });
db.result_set_states.createIndex(
  { conversationId: 1, queryText: 1 },
  { unique: true },
);
```

**Chính sách:**

- TTL index trên `expiresAt` — MongoDB tự động xóa document sau 20 phút
- `conversationId + queryText` là unique — ghi đè nếu trùng (cùng query thì chỉ giữ 1)
- Không dùng SQL cho state này

---

## 9. Error Handling

### Error codes (giữ nguyên từ section 9 của plan gốc)

| Error code                   | Khi nào                                           |
| ---------------------------- | ------------------------------------------------- |
| `stale_result_set`           | Tất cả result set đều quá hạn                     |
| `out_of_range_reference`     | Ordinal vượt quá số lượng items trong tất cả list |
| `ambiguity_blocked_mutation` | Nhiều list match ordinal — cần LLM resolve        |
| `missing_fields`             | Thiếu identifier                                  |
| `invalid_tool_args`          | Sai schema                                        |

### Format response

```json
{
  "request_id": "string",
  "error_code": "stale_result_set",
  "message": "Danh sách kết quả đã hết hạn. Vui lòng thực hiện lại tìm kiếm.",
  "node": "pre_tool_validator",
  "details": {
    "conversation_id": "conv_123",
    "total_states_found": 2,
    "all_stale": true
  }
}
```

---

## 10. Unit Test Plan

### ResultSetStateStore

| #   | Test case                                        | Expected                                   |
| --- | ------------------------------------------------ | ------------------------------------------ |
| 1   | save + getAllValid                               | Lưu 1 result set → getAllValid trả về đúng |
| 2   | save với nhiều result set khác queryText         | getAllValid trả về tất cả, mới nhất đầu    |
| 3   | getAllValid khi quá TTL                          | Trả về mảng rỗng                           |
| 4   | getAllValid khi 1 cái hết hạn, 1 cái còn         | Chỉ trả cái còn hạn                        |
| 5   | save ghi đè khi trùng conversationId + queryText | Dữ liệu mới thay cũ                        |
| 6   | save với orderedConferenceIds rỗng               | Vẫn lưu được                               |

### ResultSetResolver (resolveAll)

| #   | Test case                                | Expected                               |
| --- | ---------------------------------------- | -------------------------------------- |
| 7   | 1 list, ordinal=2 → resolve được         | uniqueMatch = conf_002                 |
| 8   | 2 list, ordinal=2 match cả 2             | matches.length = 2, uniqueMatch = null |
| 9   | 2 list, ordinal=2 chỉ match được 1       | uniqueMatch = conf_xxx                 |
| 10  | Ordinal=-1 (cuối) trên list 3 items      | resolve ra item thứ 3 (index 2)        |
| 11  | Ordinal=1 (đầu tiên)                     | resolve ra item đầu (index 0)          |
| 12  | Ordinal=5 trên list chỉ có 3 items       | matches.length = 0                     |
| 13  | Không có result set nào                  | matches.length = 0                     |
| 14  | Tất cả result set stale                  | matches.length = 0                     |

### ResultSetResolver (resolveByContext)

| #   | Test case                                                                            | Expected          |
| --- | ------------------------------------------------------------------------------------ | ----------------- |
| 15  | contextEmbedding vs queryEmbedding similarity > threshold                            | resolved          |
| 16  | contextEmbedding vs queryEmbedding similarity < threshold                            | resolvedId = null |
| 17  | contextHint paraphrase khác (e.g. "AI conferences" vs "AI list") nhưng embedding gần | resolved          |
| 18  | Cosine similarity > threshold nhưng ordinal out of range                             | resolvedId = null |

### ResolveConferenceRefHandler

| #   | Test case                        | Expected                                           |
| --- | -------------------------------- | -------------------------------------------------- |
| 19  | LLM gọi với đúng contextHint     | Trả về resolved=true + conferenceId                |
| 20  | LLM gọi với contextHint sai      | Fallback resolveAll → nếu vẫn sai → resolved=false |
| 21  | LLM gọi với ordinal không hợp lệ | resolved=false                                     |

### Fast Path (preToolValidator)

| #   | Test case                                                          | Expected                                             |
| --- | ------------------------------------------------------------------ | ---------------------------------------------------- |
| 22  | manageFollow với identifier="2", identifierType="ordinal", 1 list  | allowed=true, identifier bị thay bằng ID             |
| 23  | manageFollow với identifier="2", identifierType="ordinal", 2 list  | allowed=false, ambiguity_blocked_mutation            |
| 24  | manageFollow với identifier="5", identifierType="ordinal", 0 list  | allowed=false, out_of_range_reference                |
| 25  | manageFollow với identifier="ICML", identifierType="acronym"       | Không chạy resolveAll, chạy hasAmbiguousReference cũ |

---

## 11. Sequence Diagram (Luồng xử lý)

### Fast Path (1 turn)

```
User                    Orchestrator          preToolValidator      Resolver          MongoDB
 |                           |                      |                   |                |
 |--- "follow thứ 2" ------>|                      |                   |                |
 |                           |--- LLM generate ---->|                   |                |
 |                           |   manageFollow       |                   |                |
 |                           |   (identifier="2",   |                   |                |
 |                           |    identifierType=   |                   |                |
 |                           |    "ordinal")        |                   |                |
 |                           |                      |--- resolveAll --->|                |
 |                           |                      |   (ordinal=2)     |--- query ----->|
 |                           |                      |                   |<-- [state1] ---|
 |                           |                      |                   |                |
 |                           |                      |<-- uniqueMatch ---|                |
 |                           |                      |   = conf_002      |                |
 |                           |                      |                   |                |
 |                           |<-- allowed=true -----|                   |                |
 |                           |   identifier=conf_002|                   |                |
 |                           |   identifierType=id  |                   |                |
 |                           |--- manageFollow ---->|                   |                |
 |                           |   (conf_002)         |                   |                |
 |<-- response --------------|                      |                   |                |
```

### Slow Path (2 turns, khi mơ hồ thật sự)

```
User                    Orchestrator          preToolValidator      Resolver          MongoDB
 |                           |                      |                   |                |
 |--- "follow thứ 2" ------>|                      |                   |                |
 |                           |--- LLM generate ---->|                   |                |
 |                           |   manageFollow       |                   |                |
 |                           |   (identifier="2",   |                   |                |
 |                           |    identifierType=   |                   |                |
 |                           |    "ordinal")        |                   |                |
 |                           |                      |--- resolveAll --->|                |
 |                           |                      |   (ordinal=2)     |--- query ----->|
 |                           |                      |                   |<-- [state1] ---|
 |                           |                      |                   |<-- [state2] ---|
 |                           |                      |<-- 2 matches -----|                |
 |                           |                      |                   |                |
 |                           |<-- BLOCK ------------|                   |                |
 |                           |   ambiguity_blocked  |                   |                |
 |                           |                      |                   |                |
 |   LLM thấy block -> dùng tool resolveConferenceRef                   |                |
 |                           |                      |                   |                |
 |--- resolveConfRef ------->|                      |                   |                |
 |   (ordinal=2,             |--- handler call -----|                   |                |
 |    contextHint="AI confs")|                      |--- resolveBy ---->|                |
 |                           |                      |   QueryText()     |--- query ----->|
 |                           |                      |                   |<-- [state1] ---|
 |                           |                      |<-- conf_002 ------|                |
 |                           |<-- conf_002 ---------|                   |                |
 |                           |                      |                   |                |
 |   LLM có ID thật -> gọi manageFollow(conf_002, "id")                |                |
 |                           |--- manageFollow ---->|                   |                |
 |                           |   (conf_002)         |                   |                |
 |<-- response --------------|                      |                   |                |
```

---

## 12. Implementation Steps (Thứ tự code)

### Step 1: Tạo type/interface

- File: `src/types/resultSetState.types.ts`
- `ResultSetState`, `IResultSetStateStore`, `IResultSetResolver`, `ResolveResult`, `ResolveAllResult`

### Step 2: Tạo MongoDB model

- File: `src/models/resultSetState.model.ts`
- Mongoose schema + model + TTL index

### Step 3: Tạo ResultSetStateStore

- ResultSetStateStore là tầng lưu trữ — chỉ biết CRUD MongoDB (save, getAllValid, clearConversation). Không có logic nghiệp vụ.
- File: `src/services/resultSetState/store.service.ts`
- Implement `IResultSetStateStore`: `save`, `getAllValid`, `clearConversation`
- `getAllValid` phải lọc theo TTL (so sánh `expiresAt` với hiện tại)

### Step 4: Tạo ResultSetResolver

- ResultSetResolver là tầng xử lý nghiệp vụ — nhận số (1-based) → chuyển thành index 0-based → dùng Store lấy danh sách → map ra ID conference. Nó cũng có resolveByContext dùng cosine similarity để match embedding với contextHint từ LLM.
- File: `src/services/resultSetState/resolver.service.ts`
- Implement `IResultSetResolver`: `resolveAll`, `resolveByContext`
- **KHÔNG implement `generateEmbedding`** — đã có sẵn `EmbeddingService.generateEmbedding()` trong codebase (`src/chatbot/services/rag/embeddingService.ts`), Resolver chỉ cần dùng `container.resolve(EmbeddingService)` để gọi
- **Ordinal là `number`** (không phải string):
  - `ordinal > 0`: đếm xuôi, 1-based → index = `ordinal - 1`
  - `ordinal < 0`: đếm từ cuối, -1 = cuối → index = `length + ordinal`
- `resolveByContext`:
  - Dùng `EmbeddingService.generateEmbedding(contextHint)` để lấy `contextEmbedding` nếu cần (hoặc nhận sẵn từ param)
  - Cosine similarity giữa `contextEmbedding` và `queryEmbedding` của từng result set
  - Chọn result set có similarity > threshold (vd 0.7)
  - Resolve số ordinal → index → resolve ID
- Core logic: `ordinal → index → get ID`
- **Dependency:** Inject `IResultSetStateStore` vào constructor — Resolver cần store để gọi `getAllValid()`

### Step 5: Unit test Store + Resolver

- File: `src/services/resultSetState/__tests__/store.service.test.ts`
- File: `src/services/resultSetState/__tests__/resolver.service.test.ts`

### Step 6: Export module

- File: `src/services/resultSetState/index.ts`

### Step 7: Router Integration (Save)

- File: `src/chatbot/handlers/retrieveKnowledge.handler.ts` (dòng ~165)
- Inject `IResultSetStateStore` và `IResultSetResolver` vào constructor
- Gọi `this.resultSetResolver.generateEmbedding(query)` để lấy embedding
- Gọi `this.resultSetStateStore.save(conversationId, query, embedding, ids)`

### Step 8: Thêm "ordinal" vào identifierType enum mutation tools

- File: Tất cả file language functions (english.ts, vietnamese.ts, spanish.ts, ...)
- Thêm `"ordinal"` vào enum `identifierType` của các tool:
  `manageFollow`, `manageCalendar`, `manageBlacklist`, `rateConference`,
  `getConferenceFeedback`, `countConferenceFollowed`

### Step 9: Fast Path Integration (preToolValidator)

- File: `src/chatbot/guards/preToolValidator.ts` (dòng ~500)
- Check `identifierType === "ordinal"` thay vì `isOrdinalReference()`
- `parseInt(identifier)` để lấy số
- Gọi `resultSetResolver.resolveAll(conversationId, ordinalNum)` — truyền number
- Nếu uniqueMatch → thay identifier=ID, identifierType="id", return allowed
- Nếu 0 match → return `out_of_range_reference`
- Nếu nhiều match → fallback ambiguity check

### Step 10: Tạo tool resolveConferenceRef

- File: `src/chatbot/language/functions/english.ts` — thêm declaration (ordinal là Type.NUMBER)
- File: `src/chatbot/language/functions/vietnamese.ts` — thêm declaration (ordinal là Type.NUMBER)
- File: `src/chatbot/language/functions/spanish.ts` — thêm declaration (ordinal là Type.NUMBER)
- File: `src/chatbot/handlers/resolveConferenceRef.handler.ts` — handler
- File: `src/chatbot/gemini/functionRegistry.ts` — register

### Step 11: Update system prompt (tất cả ngôn ngữ)

- File: `src/chatbot/language/instructions/english.ts`
- File: `src/chatbot/language/instructions/vietnamese.ts`
- File: `src/chatbot/language/instructions/spanish.ts`
- Thêm hướng dẫn:
  1. Chuyển "thứ 2", "cuối", "đầu tiên" → số (2, -1, 1)
  2. Gọi mutation tool với identifierType="ordinal"
  3. Nếu bị ambiguity_blocked → dùng resolveConferenceRef

---

## 13. File structure đề xuất

```
src/
  types/
    resultSetState.types.ts             # Type definitions
  models/
    resultSetState.model.ts             # MongoDB schema + model
  services/
    resultSetState/
      store.service.ts                  # IResultSetStateStore
      resolver.service.ts               # IResultSetResolver (resolveAll, resolveByContext)
      index.ts                          # Export
      __tests__/
        store.service.test.ts
        resolver.service.test.ts
  chatbot/
    handlers/
      resolveConferenceRef.handler.ts   # Handler cho tool mới
    language/
      functions/
        english.ts                      # + englishResolveConferenceRefDeclaration
        vietnamese.ts                   # + vietnameseResolveConferenceRefDeclaration
        spanish.ts                      # + spanishResolveConferenceRefDeclaration
      instructions/
        english.ts                      # + system prompt hướng dẫn
        vietnamese.ts                   # + system prompt hướng dẫn
        spanish.ts                      # + system prompt hướng dẫn
    gemini/
      functionRegistry.ts               # + register resolveConferenceRef
    guards/
      preToolValidator.ts               # + Fast Path (check identifierType === "ordinal")
```

---

## 14. Định nghĩa hoàn thành (DoD)

- [x] Module `ResultSetStateStore` lưu/đọc result set trên MongoDB
- [x] TTL index hoạt động (document tự xóa sau 20 phút)
- [x] `getAllValid()` trả đúng result set còn hạn, không trả stale
- [x] Module `ResultSetResolver.resolveAll(ordinal: number)` quét tất cả result set, resolve số 1-based (dương = xuôi, âm = ngược) → trả uniqueMatch hoặc null
- [x] Module `ResultSetResolver.resolveByContext()` match contextEmbedding với queryEmbedding bằng cosine similarity
- [x] Module `ResultSetResolver.generateEmbedding()` tái sử dụng embedding service có sẵn
- [x] Router gọi `save()` khi có list mode query (retrieveKnowledge.handler.ts)
- [x] Fast Path: preToolValidator chạy resolveAll trước ambiguity check
- [x] Fast Path: uniqueMatch → tự động thay identifier → pass (1 turn)
- [x] Fast Path: 0 match → trả `out_of_range_reference`
- [x] Fast Path: 2+ match → fallback ambiguity block
- [x] Tool `resolveConferenceRef` được định nghĩa trong function declarations
- [x] Tool `resolveConferenceRef` có handler hoạt động đúng
- [x] Tool `resolveConferenceRef` được register trong functionRegistry
- [x] System prompt được cập nhật (tất cả ngôn ngữ)
- [x] Đã thêm `"ordinal"` vào identifierType enum của tất cả mutation tools (manageFollow, manageCalendar, v.v.)
- [x] Unit test pass cho Store + Resolver + Fast Path
- [x] `stale_result_set` trả đúng format error code chuẩn

---

## 15. Phụ thuộc

| Phụ thuộc              | Ghi chú                                                          |
| ---------------------- | ---------------------------------------------------------------- |
| MongoDB connection     | Đã có trong codebase, collection mới `result_set_states`         |
| Conversation ID        | Cần confirm cách lấy từ context hiện tại                         |
| Ordinal parser         | Resolve số 1-based (dương + âm) → index — đơn giản, không cần NLP |
| Fuzzy match (optional) | Dùng cho Step 3 của resolveConferenceRef handler (có thể để sau) |
