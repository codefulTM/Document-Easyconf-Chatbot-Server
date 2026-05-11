# P1-01: Result Set State V1 — Implementation Plan

## 1. Tổng quan

**Mã work package:** R1-01
**Ưu tiên:** Số 1 (cao nhất)
**Scope:** In-scope (bắt buộc)
**Liên quan tới:** P1-02 (Resolver — module mới), P2-04 (3-Layer Memory — Warm Memory), system prompt (hướng dẫn LLM dùng tool resolveConferenceRef)

### Mục tiêu

Tạo một module lưu trạng thái danh sách kết quả conference theo conversation, để Resolver (P1-02) có nguồn sự thật xác định tham chiếu thứ tự ("cái thứ 2", "cái cuối", v.v.) mà không cần LLM suy đoán.

---

## 2. Kiến trúc Resolver (P1-02) — 2 tầng

Resolver có 2 tầng xử lý, hoạt động tuần tự:

### Tầng 1: Fast Path — Resolve tất cả (trong preToolValidator)

Khi LLM gọi tool mutating với identifier mơ hồ ("thứ 2"), ROUTE không qua tool mới. preToolValidator tự động resolve mà không cần thêm vòng LLM.

```
[preToolValidator] nhận manageFollow(identifier="thứ 2")
  → gọi ResultSetResolver.resolveAll(convId, "thứ 2")
    → quét TẤT CẢ result set còn hạn trong conversation
    → thử resolve "thứ 2" trên từng list
    → kết quả:
      • 0 list match → out_of_range_reference (block)
      • 1 list match duy nhất → resolve, thay identifier, allowed=true  ✅ 1 turn
      • 2+ list match → ambiguity_blocked_mutation (block)
```

### Tầng 2: Slow Path — Tool mới `resolveConferenceRef` (cho trường hợp mơ hồ)

Khi Tầng 1 block với `ambiguity_blocked_mutation`, LLM thấy lỗi → dùng tool `resolveConferenceRef` với ngữ cảnh từ conversation.

```
[LLM] thấy ambiguity_blocked_mutation
  → gọi resolveConferenceRef(ordinal="thứ 2", contextHint="AI conferences")
    → handler dùng contextHint match với query_text trong result sets
    → resolve ra conf_002
    → trả về
  → LLM gọi manageFollow(identifier="conf_002", identifierType="id")
    → preToolValidator thấy ID thật → pass  ✅ 2 turns (nhưng hiếm)
```

### Tổng kết: Resolve Strategy

| Kịch bản | Tầng | Số vòng LLM | Cơ chế |
|---|---|---|---|
| 1 list trong conversation | Fast Path | 1 | Resolve tất cả → chỉ 1 match |
| Nhiều list, ordinal chỉ match 1 | Fast Path | 1 | Resolve tất cả → chỉ 1 match |
| Nhiều list, ordinal match nhiều | Slow Path | 2 | Block → LLM gọi tool với contextHint |
| Không list nào match | Fast Path | 1 (error) | Block → báo user không tìm thấy |
| Tất cả list đều stale | Fast Path | 1 (error) | Block → stale_result_set |

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
   * Trả về kết quả chi tiết để preToolValidator quyết định block hay pass.
   */
  resolveAll(
    conversationId: string,
    ordinal: string,
  ): Promise<ResolveAllResult>;

  /**
   * Resolve ordinal với contextHint cụ thể (từ LLM).
   * Match qua 2 bước: exact match queryText → fallback cosine similarity với queryEmbedding.
   * Dùng trong handler của tool resolveConferenceRef.
   */
  resolveByContext(
    conversationId: string,
    contextHint: string,
    contextEmbedding: number[],
    ordinal: string,
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

Chỉ thêm 1 file: function declarations (không sửa các tool mutating hiện có)

```typescript
export const englishResolveConferenceRefDeclaration: FunctionDeclaration = {
  name: "resolveConferenceRef",
  description:
    "Resolves an ordinal or ambiguous conference reference to a specific conference ID. " +
    "Use this when the user says things like 'the 2nd one', 'cái thứ 3', 'the last one', 'cái cuối', " +
    "'the first one', v.v. and the system could not automatically resolve it. " +
    "Provide the ordinal reference and a context hint describing which search result list the user is referring to. " +
    "The system will return the exact conference ID you can use in subsequent tool calls.",
  parameters: {
    type: Type.OBJECT,
    properties: {
      ordinal: {
        type: Type.STRING,
        description:
          "The ordinal reference from the user's message, exactly as spoken. " +
          "Examples: 'thứ 2', 'thứ 3', 'cuối', 'đầu tiên', 'last one', 'second one'.",
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
    const ordinal = args.ordinal as string;
    const contextHint = args.contextHint as string;

    // Sinh embedding cho contextHint để semantic matching
    const contextEmbedding = await resultSetResolver.generateEmbedding(contextHint);

    // Bước 1: Exact match + cosine similarity với queryEmbedding
    const contextMatch = await resultSetResolver.resolveByContext(
      conversationId, contextHint, contextEmbedding, ordinal
    );
    if (contextMatch.resolvedId) {
      return { modelResponseContent: JSON.stringify({
        resolved: true,
        conferenceId: contextMatch.resolvedId,
        confidence: contextMatch.confidence,
        matchType: contextMatch.confidence === "high" ? "exact" : "semantic",
      })};
    }

    // Bước 2: Fallback — resolveAll
    const allMatch = await resultSetResolver.resolveAll(conversationId, ordinal);
    if (allMatch.uniqueMatch) {
      return { modelResponseContent: JSON.stringify({
        resolved: true,
        conferenceId: allMatch.uniqueMatch,
        confidence: "medium",
        matchType: "fallback_all",
      })};
    }

    // Không resolve được
    return { modelResponseContent: JSON.stringify({
      resolved: false,
      message: "Could not resolve the conference reference.",
    })};
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
  const ids = compactItems.map(item => item.id);
  const embedding = await this.resultSetStateStore.generateEmbedding(query);
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
- Cần inject `IResultSetStateStore` vào constructor của `RetrieveKnowledgeHandler`

### 6.2 Fast Path Integration (preToolValidator)

**File:** `src/chatbot/guards/preToolValidator.ts`
**Dòng:** 426-515 (trong hàm `validateMutationArgs`, trước `hasAmbiguousReference` check)

```typescript
if (isMutatingAction) {
  // ... check identifier, identifierType ...

  // >>> FAST PATH: resolveAll TRƯỚC KHI BLOCK <<<
  // Nếu identifier có vẻ là ordinal reference ("thứ 2", "cuối", etc.)
  if (isOrdinalReference(identifier)) {
    const result = await resultSetResolver.resolveAll(conversationId, identifier);
    
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

**Chú ý:** `isOrdinalReference()` parse identifier xem có phải số thứ tự không. Nếu không phải (vd identifier là acronym thật) thì không chạy resolveAll, để ambiguity check xử lý như hiện tại.

### 6.3 Slow Path Integration (tool resolveConferenceRef)

**File:** `src/chatbot/language/functions/english.ts` (và các file language khác)
**File:** `src/chatbot/gemini/functionRegistry.ts`

- Chỉ thêm tool declaration mới, không sửa tool cũ
- System prompt: thêm 1 đoạn hướng dẫn LLM dùng tool này

### 6.4 Warm Memory Integration (P2-04)

```
MemoryManagerLite
  -> warm_memory.result_set_ref = conversationId:queryText
  -> warm_memory.active_entities = orderedConferenceIds
```

---

## 7. Integration: Không sửa tool declarations cũ — Chỉ sửa system prompt

**Nguyên tắc:** Zero touch vào các tool declaration hiện có (manageFollow, manageCalendar, manageBlacklist, v.v.). Chỉ 2 thay đổi:

### 7.1 Thêm 1 tool declaration mới + handler

- File: `src/chatbot/language/functions/english.ts` — thêm `englishResolveConferenceRefDeclaration`
- File: `src/chatbot/language/functions/vietnamese.ts` — thêm tương tự
- File: `src/chatbot/gemini/functionRegistry.ts` — register handler

### 7.2 Thêm 1 đoạn trong system prompt (tất cả ngôn ngữ)

**File:** `src/chatbot/language/instructions/vietnamese.ts`
**File:** `src/chatbot/language/instructions/english.ts`
**File:** `src/chatbot/language/instructions/spanish.ts`

Thêm đoạn (ví dụ tiếng Việt):

```
Khi người dùng dùng tham chiếu mơ hồ như "thứ 2", "cái cuối", "cái đầu tiên"
để chỉ một hội nghị trong danh sách kết quả tìm kiếm trước đó:
1. Ưu tiên gọi trực tiếp tool quản lý (manageFollow, manageCalendar, manageBlacklist...)
   với identifier là tham chiếu mơ hồ — hệ thống có thể tự động resolve.
2. Nếu bị chặn với lỗi "ambiguity_blocked_mutation" hoặc "out_of_range_reference",
   hãy dùng hàm resolveConferenceRef với:
   - ordinal: tham chiếu số thứ tự từ người dùng (giữ nguyên)
   - contextHint: mô tả ngắn kết quả tìm kiếm bạn nghĩ người dùng đang nói tới
   - action: hành động người dùng muốn thực hiện
3. Sau khi nhận được conferenceId từ resolveConferenceRef, gọi lại tool quản lý
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

| Error code | Khi nào |
|---|---|
| `stale_result_set` | Tất cả result set đều quá hạn |
| `out_of_range_reference` | Ordinal vượt quá số lượng items trong tất cả list |
| `ambiguity_blocked_mutation` | Nhiều list match ordinal — cần LLM resolve |
| `missing_fields` | Thiếu identifier |
| `invalid_tool_args` | Sai schema |

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

| # | Test case | Expected |
|---|---|---|
| 1 | save + getAllValid | Lưu 1 result set → getAllValid trả về đúng |
| 2 | save với nhiều result set khác queryText | getAllValid trả về tất cả, mới nhất đầu |
| 3 | getAllValid khi quá TTL | Trả về mảng rỗng |
| 4 | getAllValid khi 1 cái hết hạn, 1 cái còn | Chỉ trả cái còn hạn |
| 5 | save ghi đè khi trùng conversationId + queryText | Dữ liệu mới thay cũ |
| 6 | save với orderedConferenceIds rỗng | Vẫn lưu được |

### ResultSetResolver (resolveAll)

| # | Test case | Expected |
|---|---|---|
| 7 | 1 list, ordinal "thứ 2" → resolve được | uniqueMatch = conf_002 |
| 8 | 2 list, ordinal match cả 2 | matches.length = 2, uniqueMatch = null |
| 9 | 2 list, ordinal chỉ match được 1 | uniqueMatch = conf_xxx |
| 10 | Ordinal "cuối" trên list 3 items | resolve ra item thứ 3 |
| 11 | Ordinal "đầu tiên" | resolve ra item đầu |
| 12 | Ordinal "thứ 5" trên list chỉ có 3 items | matches.length = 0 |
| 13 | Không có result set nào | matches.length = 0 |
| 14 | Tất cả result set stale | matches.length = 0 |

### ResultSetResolver (resolveByQueryText)

| # | Test case | Expected |
|---|---|---|
| 15 | contextHint match chính xác queryText | resolved |
| 16 | contextHint match fuzzy queryText | resolved (nếu implement fuzzy) |
| 17 | contextHint không match gì | resolvedId = null |
| 18 | contextHint match nhưng ordinal out of range | resolvedId = null |

### ResolveConferenceRefHandler

| # | Test case | Expected |
|---|---|---|
| 19 | LLM gọi với đúng contextHint | Trả về resolved=true + conferenceId |
| 20 | LLM gọi với contextHint sai | Fallback qua fuzzy → nếu vẫn sai → resolved=false |
| 21 | LLM gọi với ordinal không hợp lệ | resolved=false |

### Fast Path (preToolValidator)

| # | Test case | Expected |
|---|---|---|
| 22 | manageFollow với identifier="thứ 2", chỉ 1 list | allowed=true, identifier bị thay bằng ID |
| 23 | manageFollow với identifier="thứ 2", 2 list match | allowed=false, ambiguity_blocked_mutation |
| 24 | manageFollow với identifier="thứ 2", 0 list match | allowed=false, out_of_range_reference |
| 25 | manageFollow với identifier="ICML" (không phải ordinal) | Không chạy resolveAll, chạy hasAmbiguousReference cũ |

---

## 11. Sequence Diagram (Luồng xử lý)

### Fast Path (1 turn)

```
User                    Orchestrator          preToolValidator      Resolver          MongoDB
 |                           |                      |                   |                |
 |--- "follow thứ 2" ------>|                      |                   |                |
 |                           |--- LLM generate ---->|                   |                |
 |                           |   manageFollow       |                   |                |
 |                           |   ("thứ 2")          |                   |                |
 |                           |                      |--- resolveAll --->|                |
 |                           |                      |                   |--- query ----->|
 |                           |                      |                   |<-- [state1] ---|
 |                           |                      |                   |                |
 |                           |                      |<-- uniqueMatch ---|                |
 |                           |                      |   = conf_002      |                |
 |                           |                      |                   |                |
 |                           |<-- allowed=true -----|                   |                |
 |                           |   identifier=conf_002|                   |                |
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
 |                           |   ("thứ 2")          |                   |                |
 |                           |                      |--- resolveAll --->|                |
 |                           |                      |                   |--- query ----->|
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
 |   (ordinal="thứ 2",       |--- handler call -----|                   |                |
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
- File: `src/services/resultSetState/store.service.ts`
- Implement `IResultSetStateStore`: `save`, `getAllValid`, `clearConversation`
- `getAllValid` phải lọc theo TTL (so sánh `expiresAt` với hiện tại)

### Step 4: Tạo ResultSetResolver
- File: `src/services/resultSetState/resolver.service.ts`
- Implement `IResultSetResolver`: `resolveAll`, `resolveByContext`, `generateEmbedding`
- `resolveByContext`: exact match queryText → fallback cosine similarity với queryEmbedding
- `generateEmbedding`: kiểm tra codebase có sẵn embedding utility không (RetrievalService, Gemini embedding API, etc.), tái sử dụng nếu có
- Core logic: parse ordinal ("thứ 2", "cuối", "đầu tiên", "thứ N") → index → get ID

### Step 5: Unit test Store + Resolver
- File: `src/services/resultSetState/__tests__/store.service.test.ts`
- File: `src/services/resultSetState/__tests__/resolver.service.test.ts`

### Step 6: Export module
- File: `src/services/resultSetState/index.ts`

### Step 7: Router Integration (Save)
- File: `src/chatbot/handlers/retrieveKnowledge.handler.ts` (dòng ~165)
- Inject `IResultSetStateStore` → gọi `save(conversationId, query, embedding, ids)`

### Step 8: Fast Path Integration (preToolValidator)
- File: `src/chatbot/guards/preToolValidator.ts` (dòng ~500)
- Thêm `isOrdinalReference()` check
- Gọi `resultSetResolver.resolveAll()` trước `hasAmbiguousReference`
- Nếu uniqueMatch → thay identifier, return allowed
- Nếu 0 match → return `out_of_range_reference`
- Nếu nhiều match → fallback ambiguity check

### Step 9: Tạo tool resolveConferenceRef
- File: `src/chatbot/language/functions/english.ts` — thêm declaration
- File: `src/chatbot/language/functions/vietnamese.ts` — thêm declaration
- File: `src/chatbot/language/functions/spanish.ts` — thêm declaration
- File: `src/chatbot/handlers/resolveConferenceRef.handler.ts` — handler
- File: `src/chatbot/gemini/functionRegistry.ts` — register

### Step 10: Update system prompt (tất cả ngôn ngữ)
- File: `src/chatbot/language/instructions/english.ts`
- File: `src/chatbot/language/instructions/vietnamese.ts`
- File: `src/chatbot/language/instructions/spanish.ts`
- Thêm hướng dẫn dùng resolveConferenceRef khi bị ambiguity_blocked

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
      resolver.service.ts               # IResultSetResolver (resolveAll, resolveByQueryText)
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
      preToolValidator.ts               # + Fast Path (resolveAll trước ambiguity check)
```

---

## 14. Định nghĩa hoàn thành (DoD)

- [x] Module `ResultSetStateStore` lưu/đọc result set trên MongoDB
- [x] TTL index hoạt động (document tự xóa sau 20 phút)
- [x] `getAllValid()` trả đúng result set còn hạn, không trả stale
- [x] Module `ResultSetResolver.resolveAll()` quét tất cả result set, trả uniqueMatch hoặc null
- [x] Module `ResultSetResolver.resolveByContext()` match contextHint với queryText (exact) + queryEmbedding (cosine similarity)
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
- [x] Không sửa bất kỳ tool declaration cũ nào (manageFollow, manageCalendar, v.v.)
- [x] Unit test pass cho Store + Resolver + Fast Path
- [x] `stale_result_set` trả đúng format error code chuẩn

---

## 15. Phụ thuộc

| Phụ thuộc | Ghi chú |
|---|---|
| MongoDB connection | Đã có trong codebase, collection mới `result_set_states` |
| Conversation ID | Cần confirm cách lấy từ context hiện tại |
| Ordinal parser | Module parse "thứ 2", "cuối", "đầu tiên", "top N" |
| Fuzzy match (optional) | Dùng cho Step 3 của resolveConferenceRef handler (có thể để sau) |
