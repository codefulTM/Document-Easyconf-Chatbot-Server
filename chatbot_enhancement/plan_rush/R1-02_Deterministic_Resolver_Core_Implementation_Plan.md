# R1-02: Deterministic Resolver Core — Implementation Plan

## 1. Tổng quan

**Mã work package:** R1-02
**Ưu tiên:** Số 2
**Kế thừa từ:** P1-01 (ResultSetState V1 — đã hoàn thành)

### Mục tiêu

Loại bỏ suy đoán của LLM cho ordinal reference, đồng thời cho phép LLM lấy **full data** của nhiều conference trong 1 turn để tự suy luận temporal/ranking mà không cần resolver chuyên biệt.

| Loại        | Giải pháp                                                               |
| ----------- | ----------------------------------------------------------------------- |
| Ordinal     | `conferenceRef` object `{ list?: number, item: number }` → resolve → ID |
| Temporal    | **Không có TemporalResolver** — LLM dùng re-query pattern               |
| Ranking     | **Không có RankingResolver** — LLM dùng re-query pattern                |
| Thiếu field | LLM gọi nhiều `retrieveKnowledge(filter:{id})` song song                |

### Triết lý thiết kế

```
❌ Sai:  TemporalResolver.resolveUpcoming()   → enumerate không hết
❌ Sai:  RankingResolver.sort(ids, "rank:desc") → phải implement sorting cho mọi field
✅ Đúng: retrieveKnowledge(filter:{id: "conf1"}) lấy FULL data
         retrieveKnowledge(filter:{id: "conf2"}) lấy FULL data
         → LLM tự đọc data và suy luận "cái nào upcoming?", "cái nào rank cao hơn?"
```

**Chìa khóa:** Cho phép LLM gọi NHIỀU `retrieveKnowledge` trong cùng 1 turn (parallel function calls), mỗi cái với `filter: { id }` khác nhau, lấy full conferenceFields → LLM có đủ dữ liệu thật để trả lời.

---

## 2. Vấn đề cần giải quyết

### 2.1 Ordinal reference ("thứ 2", "cuối", "hội nghị thứ 2 trong danh sách cuối")

**Hiện tại:** LLM tự convert "thứ 2" → "2" (dễ sai), compound phải encode string `"list:-1|item:2"`

**Giải pháp:** `conferenceRef` object với 2 field `list` và `item`, LLM output number trực tiếp:

```typescript
// LLM output — backend không cần parse gì cả
conferenceRef: { item: 2 }                 // "thứ 2" → item thứ 2
conferenceRef: { item: -1 }                // "cuối" → item cuối
conferenceRef: { list: -1, item: 2 }       // "thứ 2 trong list cuối"
conferenceRef: { list: 1, item: 1 }        // "đầu tiên trong list đầu"
```

### 2.2 Non-mutation ordinal ("cho tôi submission date của hội nghị thứ 2")

**Hiện tại:** Không đi qua preToolValidator → không resolve được.

**Giải pháp:** Thêm `conferenceRef` object vào `retrieveKnowledge`. Handler resolve → filter:id → re-query full data.

### 2.3 List combine ("hội nghị thứ 2 trong danh sách cuối")

**Giải pháp:** `conferenceRef.list` + `conferenceRef.item` — LLM output structured object, backend resolve 2 tầng (list → item).

### 2.4 Thiếu field cho follow-up question

**Hiện tại:** `buildCompactConferenceList` strip field → LLM không có data để trả lời chi tiết.

**Giải pháp:** LLM gọi nhiều `retrieveKnowledge(filter:{id})` song song để lấy full data cho từng conference.

### 2.5 Save ResultSetState mỗi lần retrieveKnowledge trả list + context window priority

**Hiện tại:** Code đã save mỗi lần retrieveKnowledge trả list (dòng 254-268 trong `retrieveKnowledge.handler.ts`). Plan cũ định bỏ cơ chế này, chỉ save khi message bị đẩy ra khỏi window 50.

**Giải pháp mới:** Giữ nguyên cơ chế save mỗi lần `execute` trả về list hội nghị. Đồng thời điều chỉnh prompt để LLM ưu tiên dùng dữ liệu từ context window khi resolve ordinal reference:

- **Luôn save vào ResultSetState** mỗi khi `retrieveKnowledge` trả về list hội nghị (warm memory).
- **Khi người dùng hỏi "hội nghị thứ N":**
  - LLM kiểm tra context window: nếu thấy đủ N hội nghị → dùng thông tin trực tiếp từ context window (không cần gọi tool).
  - Nếu context window chỉ có < N hội nghị → LLM dùng `conferenceRef: { item: N }` để lấy từ ResultSetState (warm memory).

---

## 3. Kiến trúc

### 3.1 Luồng ordinal resolver

```
Mutation flow:
  LLM: manageFollow(itemType="conference", action="follow",
                     conferenceRef: { item: 2 })
  → preToolValidator
    → thấy args.conferenceRef.item = 2
    → resolveConferenceRef(convId, { list: undefined, item: 2 })
      → list undefined → dùng latest result set
      → item 2 → resolveAll(convId, 2) → conf_002
    → allowed=true, identifier=conf_002 ✅

Compound mutation flow:
  LLM: manageFollow(itemType="conference", action="follow",
                     conferenceRef: { list: -1, item: 2 })
  → preToolValidator
    → resolveConferenceRef(convId, { list: -1, item: 2 })
      → list -1 → getAllValid → lấy state cuối
      → item 2 → state.orderedConferenceIds[1] → conf_002
    → allowed=true ✅

Non-mutation flow:
  LLM: retrieveKnowledge(query="submission date",
                          conferenceRef: { item: 2 })
  → retrieveKnowledge handler
    → resolveConferenceRef(convId, { list: undefined, item: 2 }) → conf_002
    → filter.id = conf_002 → re-query full RAG data
    → Trả full data (không strip)
```

**Lưu ý:** LLM có 2 cách để chỉ định conference:

- `conferenceRef: { item: N }` — ordinal trên latest result set (không cần list)
- `conferenceRef: { list: L, item: N }` — ordinal kết hợp list + item
- `identifier` + `identifierType` — cách cũ (acronym/title/id), vẫn giữ cho backward compat

### 3.2 Luồng parallel function calls (temporal/ranking)

```
Turn 1: LLM gọi retrieveKnowledge(query="AI conferences", listMode=true)
        → Handler trả compact list: [conf_A, conf_B, conf_C]

Turn 2 (cùng sub-agent loop):
  LLM thấy list, muốn biết chi tiết từng cái → gọi 3 retrieveKnowledge SONG SONG:
    fn1: retrieveKnowledge(filter={id:"conf_A"}, conferenceFields={dates, ranks, title})
    fn2: retrieveKnowledge(filter={id:"conf_B"}, conferenceFields={dates, ranks, title})
    fn3: retrieveKnowledge(filter={id:"conf_C"}, conferenceFields={dates, ranks, title})

  → LLM layer trả về functionCalls = [fn1, fn2, fn3]
  → Handler xử lý CẢ 3 (tuần tự):
      - fn1: validate → execute → response_A
      - fn2: validate → execute → response_B
      - fn3: validate → execute → response_C
  → Gom 3 response thành 1 function response turn với 3 parts
  → Gửi lại LLM

Turn 3 (cùng sub-agent loop):
  LLM đọc full data của conf_A, conf_B, conf_C
    → "conf_A có deadline gần nhất" → trả lời người dùng 🎯
    → Hoặc "conf_C rank A cao nhất" → trả lời
```

**Lợi ích:**

- Không cần TemporalResolver — LLM đọc ngày tháng thật từ DB → biết cái nào upcoming
- Không cần RankingResolver — LLM đọc rank thật → biết cái nào rank cao
- LLM trả lời dựa trên dữ liệu thật → không hallucinate
- Handle được mọi temporal mode không thể enumerate

**Model support:**

- Interface `ChatModelService` (được PooledGemini và GroqCohereHybrid implement) đã support `tools` parameter trong cả `generateTurn` và `generateStream`
- Gemini API: đã hỗ trợ multiple function calls
- Groq (llama3-groq-70b-8192-tool-use-preview): hỗ trợ multi-tool use và parallel tool calling
- Cohere (Command R+, command-a-03-2025): hỗ trợ multi-step tool use và multiple tool calls
- Backend chỉ cần sửa handler để loop qua tất cả `functionCalls[]` thay vì xử lý 1 call

### 3.3 Luồng save ResultSetState + context window priority

```
Turn N: retrieveKnowledge trả list → LUÔN save ResultSetState (warm memory)
                                   → List đồng thời nằm trong completeHistoryToSave (context window)

Turn N+K (sau đó): Người dùng hỏi "hội nghị thứ 3 là gì?"
  → LLM kiểm tra context window:
    - Nếu thấy ≥ 3 hội nghị trong context → trả lời trực tiếp, KHÔNG gọi tool
    - Nếu chỉ thấy < 3 hội nghị → gọi retrieveKnowledge(conferenceRef: { item: 3 })
      → Backend resolve từ ResultSetState → trả full data
```

---

## 4. Thay đổi cần thiết

### 4.1 ChatModelService implementations — hỗ trợ multiple function calls

**Files:**

- `src/chatbot/gemini/pooledGemini.ts` — PooledGemini.generateTurn() (dòng 242) và generateStream() (dòng 369)
- `src/chatbot/models/groqCohereHybrid.ts` — GroqCohereHybrid.generateTurn() (dòng 997) và generateStream() (dòng 1250)

**Context:** Interface `ChatModelService` (intentHandler.dependencies.ts dòng 26-97) đã định nghĩa `tools?: Tool[]` parameter trong cả `generateTurn` và `generateStream`. Cả 2 implementation đều cần cập nhật để trả về tất cả `functionCalls[]` thay vì chỉ `functionCall[0]`.

#### PooledGemini (src/chatbot/gemini/pooledGemini.ts)

**generateTurn() — hiện tại chỉ trả 1 functionCall:**

```typescript
// Hiện tại (cần kiểm tra code thực tế)
// Trả về { status: "function_call", functionCall: functionCalls[0] }

// Sau: trả về tất cả
return {
  status: "function_call",
  functionCall: functionCalls[0], // backward compatible (legacy field)
  functionCalls: functionCalls, // <<< MỚI: tất cả
  functionCallParts: response.candidates?.[0]?.content?.parts, // <<< MỚI
  // ... các field khác
};
```

**generateStream() — tương tự:**

```typescript
// Hiện tại
return { functionCall: functionCallsInFirstChunk[0] };

// Sau
return {
  functionCall: functionCallsInFirstChunk[0],
  functionCalls: functionCallsInFirstChunk,
  functionCallParts: firstChunk.candidates?.[0]?.content?.parts,
};
```

#### GroqCohereHybrid (src/chatbot/models/groqCohereHybrid.ts)

**generateTurn() — dòng 997:**

```typescript
// Hiện tại: cần kiểm tra cách xử lý functionCalls từ OpenAI-compatible response
// Sau khi parse từ OpenAI response, trả về:
return {
  status: "function_call",
  functionCall: parsedFunctionCalls[0], // backward compatible
  functionCalls: parsedFunctionCalls, // <<< MỚI: tất cả
  // ... runtimeFallback telemetry
};
```

**generateStream() — dòng 1250:**

```typescript
// Hiện tại: splitTextForStreaming rồi stream
// Cần parse function calls từ response và trả về tất cả
return {
  status: "function_call",
  functionCall: parsedFunctionCalls[0],
  functionCalls: parsedFunctionCalls, // <<< MỚI
  // ...
};
```

**Lưu ý:** GroqCohereHybrid parse function calls từ OpenAI-compatible format (dòng 685-752), cần đảm bảo parse được N function calls không chỉ 1.

### 4.2 Non-streaming handler — xử lý multiple function calls

**File:** `src/chatbot/handlers/hostAgent.nonStreaming.handler.ts`

Sau dòng 736, thay vì xử lý `functionCall` đơn lẻ:

```typescript
// Trước (dòng 736):
const functionCall = modelResult.functionCall;

// Sau:
const functionCallsToProcess: FunctionCall[] =
  modelResult.functionCalls?.length > 0
    ? modelResult.functionCalls
    : modelResult.functionCall
      ? [modelResult.functionCall]
      : [];

if (functionCallsToProcess.length === 0) {
  // error handling như cũ
}

// Xử lý TẤT CẢ function calls
const functionResponses: Part[] = [];

for (const fc of functionCallsToProcess) {
  // preToolValidator
  const preToolValidation = await validatePreToolInvocation({...});

  // Execute handler
  const handlerResult = await executeFunctionCall(fc, preToolValidation);

  functionResponses.push({
    functionResponse: {
      name: fc.name,
      response: handlerResult,
    },
  });
}

// Gom tất cả responses vào 1 function response turn
const combinedFunctionTurn: ChatHistoryItem = {
  role: "function",
  parts: functionResponses,     // <<< NHIỀU part
  uuid: uuidv4(),
  timestamp: new Date(),
};

// Và model turn cũng cần chứa tất cả function calls
const modelFunctionCallTurn: ChatHistoryItem = {
  role: "model",
  parts: functionCallsToProcess.map(fc => ({ functionCall: fc })),
  uuid: uuidv4(),
  timestamp: new Date(),
};
```

### 4.3 Streaming handler — refinement (đã gần đúng)

**File:** `src/chatbot/handlers/hostAgent.streaming.handler.ts`

Streaming handler đã có `functionCallParts`, nhưng chỉ validate `functionCall[0]`. Cần loop qua tất cả:

```typescript
// Dòng 1101-1106 hiện tại: chỉ xử lý 1 call
const preToolValidation = await validatePreToolInvocation({
  functionName: functionCall.name, // chỉ cái đầu tiên
  args: functionCall.args || {},
});

// Sửa thành loop:
const allCalls: FunctionCall[] =
  hostAgentLLMResult.functionCalls?.length > 0
    ? hostAgentLLMResult.functionCalls
    : functionCall
      ? [functionCall]
      : [];

const allResponses: Part[] = [];
for (const fc of allCalls) {
  const validation = await validatePreToolInvocation({
    functionName: fc.name,
    args: fc.args || {},
    language,
    agentId: "HostAgent",
  });
  // Execute + collect response vào allResponses
}

// Gom tất cả responses
```

### 4.4 Mở rộng ResultSetResolver

**File:** `src/services/resultSetState/resolver.service.ts`

Sửa method `resolveAll` đã có để merge với `resolveByContext`:

```typescript
/**
 * Resolve conference reference thành conference ID.
 * @param conversationId conversation ID
 * @param itemOrdinal item ordinal (required)
 * @param listRef list reference (optional) - có thể là number (ordinal) hoặc string (context description)
 *   - undefined/null: dùng latest result set
 *   - number: list ordinal (1-based, -1 = last)
 *   - string: context description để semantic match với queryText
 * @returns ResolveResult với resolvedId hoặc null nếu không match/ambiguity
 */
async resolveAll(
  conversationId: string,
  itemOrdinal: number,
  listRef?: number | string,
): Promise<ResolveResult>;
```

Logic:

- Nếu `listRef` undefined/null → dùng latest result set (mới nhất theo thời gian)
- Nếu `listRef` là number → resolve list ordinal → chọn state cụ thể → match itemOrdinal
- Nếu `listRef` là string → semantic match với queryText → chọn state có similarity cao nhất → match itemOrdinal
- Return `{ resolvedId: string | null, reasonCode, confidence }`

### 4.6 Mở rộng preToolValidator — xử lý `conferenceRef`

**File:** `src/chatbot/guards/preToolValidator.ts`

**Luồng mới:** Trong `validateMutationArgs`, kiểm tra `args.conferenceRef` TRƯỚC khi xử lý `identifier` + `identifierType`:

```typescript
// validateMutationArgs() — thêm đầu hàm
if (isPlainObject(args.conferenceRef)) {
  const ref = args.conferenceRef as Record<string, unknown>;
  const listRef = ref.list; // có thể là number hoặc string
  const itemOrdinal = typeof ref.item === "number" ? ref.item : undefined;

  if (typeof itemOrdinal !== "number" || itemOrdinal === 0) {
    return buildBlockedResult({
      errorCode: OrchestrationSafetyErrorCode.INVALID_TOOL_ARGS,
      message: "conferenceRef.item must be a non-zero integer.",
    });
  }

  const result = await resolver.resolveAll(
    conversationId || "",
    itemOrdinal,
    listRef, // number hoặc string
  );

  if (!result.resolvedId) {
    // 0 match hoặc out of range
    return buildBlockedResult({
      errorCode: OrchestrationSafetyErrorCode.OUT_OF_RANGE_REFERENCE,
      message: `Cannot resolve conferenceRef { list: ${listRef}, item: ${itemOrdinal} }.`,
    });
  }

  // Resolve thành công → thay identifier
  normalizedArgs.identifier = result.resolvedId;
  normalizedArgs.identifierType = "id";
  return {
    allowed: true,
    node: PRE_TOOL_VALIDATOR_NODE,
    normalized_args: normalizedArgs,
  };
}

// Sau đó: xử lý identifier + identifierType như cũ (cho acronym/title/id)
// validIdentifierTypes: chỉ còn ["acronym", "title", "id"] - KHÔNG CÒN "ordinal"
```

**Lưu ý:** `conferenceRef` ưu tiên cao hơn `identifier` + `identifierType`. Khi có `conferenceRef`, bỏ qua identifier hoàn toàn.

### 4.7 Thêm `conferenceRef` param vào retrieveKnowledge

**File:** `src/chatbot/handlers/retrieveKnowledge.handler.ts`

```typescript
if (isPlainObject(args.conferenceRef)) {
  const ref = args.conferenceRef as Record<string, unknown>;
  const listRef = ref.list; // có thể là number hoặc string
  const itemOrdinal = typeof ref.item === "number" ? ref.item : undefined;

  if (typeof itemOrdinal !== "number" || itemOrdinal === 0) {
    return {
      modelResponseContent:
        "Error: conferenceRef.item must be a non-zero integer.",
    };
  }

  const result = await this.resultSetResolver.resolveAll(
    conversationId,
    itemOrdinal,
    listRef, // number hoặc string
  );

  if (!result.resolvedId) {
    return {
      modelResponseContent: JSON.stringify({
        error: "out_of_range_reference",
        message: `Cannot resolve conferenceRef { list: ${listRef}, item: ${itemOrdinal} }.`,
      }),
    };
  }

  // Filter theo ID cụ thể → lấy FULL data (không compact)
  effectiveFilter = { ...effectiveFilter, id: result.resolvedId };
}

// Bỏ qua buildCompactConferenceList khi có conferenceRef
const skipCompact = isPlainObject(args.conferenceRef);
```

**Function declaration:**

```typescript
conferenceRef: {
  type: Type.OBJECT,
  description: "Optional. Reference to a specific conference by position in a previous result list. Use when the user refers to a conference by position (e.g., 'the 2nd one', 'thứ 2', 'the last one', 'the first conference in the last list'). The system will resolve this to the actual conference ID and retrieve its full information.",
  properties: {
    list: {
      type: Type.UNION,
      description: "Optional. List reference - can be either a number (ordinal) or a string (context description). Number: list ordinal (1-based). 1 = first list, -1 = last list. String: description of the list (e.g., 'AI conference list', 'search results for 2026'). If omitted, uses the latest search result list.",
      nullable: true,
      union: [
        { type: Type.NUMBER },
        { type: Type.STRING },
      ],
    },
    item: {
      type: Type.NUMBER,
      description: "Required. Item ordinal within the list (1-based). Positive = count from start (1 = first, 2 = second), negative = count from end (-1 = last, -2 = second to last). Examples: 2 for 'the 2nd one', -1 for 'the last one'.",
    },
  },
  nullable: true,
},
```

---

## 5. Implementation Steps

### Step 1: ChatModelService implementations — multiple function calls

- **File:** `src/chatbot/gemini/pooledGemini.ts`
  - Sửa `generateTurn()` (dòng 242): trả về `functionCalls[]` (sử dụng `parts` từ `GeminiInteractionResult`)
  - Sửa `generateStream()` (dòng 369): trả về `functionCalls[]`

- **File:** `src/chatbot/models/groqCohereHybrid.ts`
  - Sửa `generateTurn()` (dòng 997): trả về `functionCalls[]` từ OpenAI-compatible response
  - Sửa `generateStream()` (dòng 1250): trả về `functionCalls[]`
  - Đảm bảo parse function calls hỗ trợ N calls (dòng 685-752)

### Step 2: SubAgent handler — multiple function calls (sequential)

- **File:** `src/chatbot/handlers/subAgent.handler.ts`
  - Thay vì chỉ xử lý `subAgentLlmResult.functionCall` (single), loop qua `subAgentLlmResult.functionCalls[]` (plural)
  - Validate + execute từng call tuần tự (sequential) bằng `for...of` loop
  - Giữ nguyên logic validate/error handling cho mỗi call
  - Gom tất cả `FunctionResponse` parts vào 1 turn duy nhất gửi lại model
  - Model turn (history) chứa tất cả function calls trong 1 message với role `model`
  - Backward compat: nếu chỉ nhận `functionCall` (singular) → wrap thành `[functionCall]` để xử lý đồng nhất

### Step 3: HostAgent handlers — multiple function calls (sequential)

- **File:** `src/chatbot/handlers/hostAgent.nonStreaming.handler.ts`
  - Loop `functionCallsToProcess[]` thay vì xử lý 1 call
  - Validate + execute từng call tuần tự (sequential) bằng `for...of` loop
  - Gom tất cả `FunctionResponse` parts vào 1 turn gửi lại model
  - Model turn chứa tất cả function calls trong 1 message với role `model`

- **File:** `src/chatbot/handlers/hostAgent.streaming.handler.ts`
  - Loop `hostAgentLLMResult.functionCalls[]` (sử dụng `parts` từ `GeminiInteractionResult` cho function call parts)
  - Validate + execute từng call tuần tự (sequential)
  - Gom responses vào 1 turn duy nhất

- **Backward compat (cho cả 2 handler):** nếu model chỉ trả `functionCall` (singular) → wrap thành `[functionCall]` để pipeline xử lý đồng nhất

### Step 4: Sửa `resolveAll` trong ResultSetResolver

- **File:** `src/services/resultSetState/resolver.service.ts`
- Sửa method `resolveAll` đã có: merge với `resolveByContext`
- Signature mới: `resolveAll(convId, itemOrdinal, listRef?)` - listRef có thể là number hoặc string
- Logic:
  - Nếu listRef undefined/null → dùng latest result set (mới nhất theo thời gian)
  - Nếu listRef là number → resolve list ordinal → chọn state cụ thể → match itemOrdinal
  - Nếu listRef là string → semantic match với queryText → chọn state có similarity cao nhất → match itemOrdinal
- Xóa method `resolveByContext` (đã merge vào resolveAll)

### Step 5: Mở rộng preToolValidator — xử lý `conferenceRef`

- **File:** `src/chatbot/guards/preToolValidator.ts`
- Đầu `validateMutationArgs`: check `isPlainObject(args.conferenceRef)`
- Gọi `resolver.resolveAll(convId, itemOrdinal, listRef)` - listRef có thể là number hoặc string
- Nếu resolve thành công → replace identifier=ID, identifierType="id"
- **Xóa backward compat cho ordinal string trong identifier/identifierType** - chỉ giữ "acronym", "title", "id"
- Xóa logic `tryResolveOrdinal` - ordinal string không còn được hỗ trợ

### Step 6: Thêm `conferenceRef` param vào retrieveKnowledge

- **File:** `src/chatbot/handlers/retrieveKnowledge.handler.ts`
- Kiểm tra `isPlainObject(args.conferenceRef)` → `resolveAll(convId, itemOrdinal, listRef)` → filter:id → full data
- **LUÔN save ResultSetState** mỗi khi trả về list hội nghị (giữ nguyên code hiện tại dòng 254-268)

### Step 7: Xóa tool `resolveConferenceRef`

- **File:** `src/chatbot/handlers/resolveConferenceRef.handler.ts` - Xóa file này
- **File:** `src/chatbot/gemini/functionRegistry.ts` - Xóa `resolveConferenceRef` entry
- **File:** `src/chatbot/language/functions/english.ts` - Xóa `englishResolveConferenceRefDeclaration`
- **File:** `src/chatbot/language/functions/vietnamese.ts` - Xóa `vietnameseResolveConferenceRefDeclaration`
- **File:** `src/chatbot/language/functions/spanish.ts` - Xóa `spanishResolveConferenceRefDeclaration`
- **File:** `src/chatbot/utils/languageConfig.ts` - Xóa reference đến resolveConferenceRef declarations
- **File:** System prompts (english.ts, vietnamese.ts) - Xóa mọi reference đến `resolveConferenceRef`

### Step 8: Cập nhật function declarations

**File:** `english.ts`

#### a) Thêm `conferenceRef` vào `retrieveKnowledge`

```typescript
conferenceRef: {
  type: Type.OBJECT,
  description: "Optional. Reference to a specific conference by position...",
  properties: {
    list: {
      type: Type.UNION,
      description: "Optional. List reference - can be either a number (ordinal) or a string (context description)...",
      nullable: true,
      union: [
        { type: Type.NUMBER },
        { type: Type.STRING },
      ],
    },
    item: { type: Type.NUMBER, description: "..." },
  },
  nullable: true,
},
```

#### b) Thêm `conferenceRef` vào 6 mutation tools (manageFollow, manageCalendar, manageBlacklist, countConferenceFollowed, rateConference, getConferenceFeedback)

```typescript
conferenceRef: {
  type: Type.OBJECT,
  description: "Optional. Alternative to identifier+identifierType. Use this when the user refers to a conference by position in a previous result list (e.g., 'the 2nd one', 'thứ 2', 'the last one'). When provided, identifier and identifierType are ignored.",
  properties: {
    list: {
      type: Type.UNION,
      description: "Optional. List reference - can be either a number (ordinal) or a string (context description). Number: list ordinal (1-based). 1 = first list, -1 = last list. String: description of the list (e.g., 'AI conference list', 'search results for 2026'). If omitted, uses the latest search result list.",
      nullable: true,
      union: [
        { type: Type.NUMBER },
        { type: Type.STRING },
      ],
    },
    item: {
      type: Type.NUMBER,
      description: "Required. Item ordinal within the list (1-based). Positive = from start (1 = first), negative = from end (-1 = last).",
    },
  },
  nullable: true,
},
```

#### c) Xóa mọi dấu vết về ordinal references

- Xóa mọi mention của "ordinal" trong `identifier.description` và `identifierType.description`
- Không thêm hint "Use conferenceRef instead" - chỉ xóa ordinal references hoàn toàn

### Step 9: System prompt — context window priority + conferenceRef

**Nguyên tắc quan trọng:** Khi người dùng dùng ordinal reference ("hội nghị thứ 2", "cái thứ 3", "hội nghị cuối cùng"), LLM phải tuân theo quy tắc ưu tiên context window:

1. **Kiểm tra context window trước:** Đếm số lượng hội nghị hiện có trong lịch sử hội thoại gần đây (context window).
   - Nếu số lượng hội nghị trong context window ≥ vị trí người dùng yêu cầu → **trả lời trực tiếp** bằng dữ liệu từ context window, **không cần gọi tool**.
   - Ví dụ: Context window đang có 5 hội nghị, người dùng hỏi "hội nghị thứ 3" → LLM đọc trực tiếp từ context window và trả lời.

2. **Fallback sang ResultSetState (warm memory):** Nếu context window chỉ có ít hội nghị hơn vị trí người dùng yêu cầu → dùng `conferenceRef` để lấy từ warm memory.
   - Ví dụ: Context window chỉ có 2 hội nghị, người dùng hỏi "hội nghị thứ 5" → LLM gọi `retrieveKnowledge(conferenceRef: { item: 5 })` để backend resolve từ ResultSetState.

3. **ConferenceAgent prompt:** Hướng dẫn cách dùng `conferenceRef` (list + item), re-query pattern, và quy tắc context window priority ở trên.

4. **HostAgent prompt:** Routing khi thấy user dùng positional reference, ưu tiên delegate sang ConferenceAgent.

### Step 10: Support multiple FrontendActions

> **Lưu ý:** Frontend hiện tại chỉ hỗ trợ single `FrontendAction` mỗi message. Để enable multiple actions từ parallel function calls, cần update cả backend và frontend.

#### Backend changes

- **File:** `src/chatbot/shared/types.ts`
  - Thay đổi `ChatHistoryItem.action?: FrontendAction` → `actions?: FrontendAction[]`
  - Thay đổi `ResultUpdate.action?: FrontendAction` → `actions?: FrontendAction[]`
  - Thay đổi `AgentCardResponse.frontendAction?: FrontendAction` → `frontendActions?: FrontendAction[]`
  - Thay đổi `FunctionHandlerOutput.frontendAction?: FrontendAction` → `frontendActions?: FrontendAction[]`
  - Giữ backward compat: vẫn có field `action?: FrontendAction` (deprecated) để frontend cũ không bị lỗi

- **File:** `src/chatbot/handlers/subAgent.handler.ts` (line 468-471)
  - Collect tất cả `functionOutput.frontendAction` vào array `frontendActions[]`
  - Thay vì chỉ gán `subAgentFrontendAction = functionOutput.frontendAction` (ghi đè), dùng `frontendActions.push(functionOutput.frontendAction)`
  - Return `frontendActions` thay vì `frontendAction`

- **File:** `src/chatbot/handlers/hostAgent.nonStreaming.handler.ts`
  - Update logic để collect `frontendActions[]` từ tất cả function calls
  - Gom vào single response

- **File:** `src/chatbot/handlers/hostAgent.streaming.handler.ts`
  - Update logic để collect `frontendActions[]` từ tất cả function calls

#### Frontend changes

- **File:** `Easyconf-FE-Client/src/app/[locale]/chatbot/lib/regular-chat.types.ts`
  - Thay đổi `action?: FrontendAction` → `actions?: FrontendAction[]`
  - Giữ backward compat: `action?: FrontendAction` (deprecated)
  - Thay đổi `HistoryItem.action` → `actions`
  - Thay đổi `MessageDisplay.action` → `actions`

- **File:** `Easyconf-FE-Client/src/app/[locale]/chatbot/stores/messageStore/messageMappers.ts`
  - Update mapper để xử lý `actions[]` thay vì `action`
  - Nếu backend gửi `action` (deprecated) → wrap thành `[action]`

- **File:** `Easyconf-FE-Client/src/app/[locale]/chatbot/regularchat/MessageContentRenderer.tsx`
  - Update component props: `actions?: FrontendAction[]`
  - Render tất cả actions (có thể dùng queue, priority, hoặc execute tuần tự)
  - Conflict resolution: nếu có conflict actions (ví dụ: navigate vs openMap), cần logic để quyết định

- **File:** `Easyconf-FE-Client/src/app/[locale]/chatbot/regularchat/ChatMessageDisplay.tsx`
  - Update component props: `actions?: FrontendAction[]`
  - Pass `actions` xuống `MessageContentRenderer`

#### Frontend action execution strategy

Khi có multiple actions, frontend cần quyết định cách thực thi:

**Option A: Execute tuần tự (sequential)**

- Execute action 1, chờ hoàn thành, rồi action 2, ...
- Ưu điểm: dễ debug, dễ rollback
- Nhược điểm: chậm, user phải đợi từng action

**Option B: Execute song song (parallel)**

- Execute tất cả actions cùng lúc
- Ưu điểm: nhanh
- Nhược điểm: khó debug, conflicts (navigate vs openMap)

**Option C: Priority-based**

- Mỗi action type có priority
- Ví dụ: `navigate` > `openMap` > `displayList`
- Chỉ execute action có priority cao nhất
- Ưu điểm: đơn giản, tránh conflicts
- Nhược điểm: bỏ qua các action khác

**Khuyến nghị:** Bắt đầu với **Option C (Priority-based)** vì:

- Đơn giản implement
- Tránh conflicts
- Phù hợp với use case hiện tại (thường chỉ cần 1 action quan trọng)

Priority order đề xuất:

1. `navigate` — cao nhất (thay đổi route)
2. `confirmEmailSend` — cần user interaction
3. `openMap` — mở map
4. `displayList`, `displayConferenceSources` — hiển thị UI
5. `addToCalendar`, `removeFromCalendar` — calendar operations
6. `itemFollowStatusUpdated`, `itemBlacklistStatusUpdated`, `itemCalendarStatusUpdated` — status updates

---

## 6. File structure

```
src/  # Easyconf-Chatbot-Server (Backend)
  chatbot/
    shared/
      types.ts                                # [SỬA] FrontendAction[] (Step 10)
    gemini/
      pooledGemini.ts                         # [SỬA] multiple functionCalls (Step 1)
    models/
      groqCohereHybrid.ts                     # [SỬA] multiple functionCalls (Step 1)
    handlers/
      subAgent.handler.ts                     # [SỬA] multiple function calls sequential (Step 2), collect frontendActions (Step 10)
      hostAgent.nonStreaming.handler.ts        # [SỬA] multiple function calls sequential (Step 3), collect frontendActions (Step 10)
      hostAgent.streaming.handler.ts           # [SỬA] multiple function calls sequential (Step 3), collect frontendActions (Step 10)
      retrieveKnowledge.handler.ts            # [SỬA] + conferenceRef, save RS (Step 6)
      resolveConferenceRef.handler.ts         # [XÓA] (Step 7)
    guards/
      preToolValidator.ts                     # [SỬA] xử lý conferenceRef (Step 5)
    language/
      functions/
        english.ts                            # [SỬA] + conferenceRef (Step 8), delete resolveConferenceRef (Step 7)
        vietnamese.ts
        spanish.ts
      instructions/
        english.ts                            # [SỬA] context window priority (Step 9), delete resolveConferenceRef refs (Step 7)
        vietnamese.ts
        spanish.ts
    utils/
      languageConfig.ts                       # [XÓA] resolveConferenceRef declaration refs (Step 7)
  services/
    resultSetState/
      resolver.service.ts                     # [SỬA] modify resolveAll, delete resolveByContext (Step 4)
      store.service.ts                        # KHÔNG ĐỔI
      index.ts                                # KHÔNG ĐỔI
      __tests__/
        resolver.service.test.ts              # [SỬA]

Easyconf-FE-Client/  # Frontend
  src/app/[locale]/chatbot/
    lib/
      regular-chat.types.ts                   # [SỬA] FrontendAction[] (Step 10)
    stores/messageStore/
      messageMappers.ts                       # [SỬA] handle actions[] (Step 10)
      messageMappers.ts                       # [SỬA] handle actions[] (Step 9)
    regularchat/
      MessageContentRenderer.tsx              # [SỬA] render actions[] (Step 9)
      ChatMessageDisplay.tsx                  # [SỬA] pass actions[] (Step 9)
```

---

## 7. Test Plan

### Multiple function calls (sequential)

| #   | Test case                            | Expected                    |
| --- | ------------------------------------ | --------------------------- |
| 1   | LLM trả về 3 function calls cùng lúc | Cả 3 đều được xử lý tuần tự |
| 2   | 3 responses gom thành 1 turn         | function turn có 3 parts    |
| 3   | 1 call fail → các call khác vẫn chạy | Không ảnh hưởng lẫn nhau    |
| 4   | LLM trả về 0 function call           | Fallback error như cũ       |

### conferenceRef resolution

| #   | Test case                                        | Expected               |
| --- | ------------------------------------------------ | ---------------------- |
| 5   | `{ item: 2 }` → 1 list, đủ items                 | allowed=true           |
| 6   | `{ item: -1 }` → item cuối                       | allowed=true           |
| 7   | `{ list: -1, item: 1 }` → list cuối, item đầu    | allowed=true           |
| 8   | `{ list: 2, item: -1 }` → list thứ 2, item cuối  | allowed=true           |
| 9   | `{ item: 2 }` → 2+ list match                    | ambiguity_blocked      |
| 10  | `{ item: 99 }` → out of range                    | out_of_range_reference |
| 11  | `{ item: 2 }` → 0 list trong conversation        | out_of_range_reference |
| 12  | `{ item: 0 }` → invalid                          | invalid_tool_args      |
| 13  | `{ list: 2, item: 1 }` → list index out of range | out_of_range_reference |

### conferenceRef trong retrieveKnowledge

| #   | Test case                                    | Expected                |
| --- | -------------------------------------------- | ----------------------- |
| 14  | `conferenceRef: { item: 2 }` → 1 match       | filter:id, full data    |
| 15  | `conferenceRef: { item: 99 }` → out of range | error                   |
| 16  | `conferenceRef: { item: 2 }` → 2+ match      | ambiguity_blocked error |

---

## 8. Điểm mới so với plan cũ

| Mục                      | Plan cũ                   | Plan mới                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------ |
| TemporalResolver         | Có — 3 method hardcode    | **Bỏ** — LLM re-query pattern                                            |
| RankingResolver          | Có — sort policy          | **Bỏ** — LLM tự sort từ full data                                        |
| `"ordinal_nl"` enum      | Thêm enum mới             | **Bỏ** — `conferenceRef` object đã đủ                                    |
| Compound string          | `"list:-3\|item:2"` parse | **Bỏ** — `conferenceRef: { list, item }` object                          |
| `ordinal` (number) param | Trong retrieveKnowledge   | **Bỏ** — `conferenceRef` object thay thế                                 |
| `conferenceRef` object   | Không có                  | **Thêm** — structured param ở 7 functions                                |
| ChatModelService layer   | Single functionCall       | **Sửa** — PooledGemini & GroqCohereHybrid support multiple functionCalls |
| Handler loop             | Xử lý 1 call/lần          | **Sửa** — loop tất cả calls trong 1 turn                                 |
| Re-query pattern         | Không có                  | **Thêm** — LLM gọi N retrieveKnowledge(filter:id) song song              |
| Save ResultSetState      | Trong retrieveKnowledge   | **Giữ** — save mỗi lần trả list                                          |

---

## 9. DoD

- [x] `conferenceRef` object (`{ list?: number|string, item: number }`) có trong tất cả function declarations:
  - retrieveKnowledge
  - manageFollow, manageCalendar, manageBlacklist
  - countConferenceFollowed, rateConference, getConferenceFeedback
- [x] `ResultSetResolver.resolveAll(convId, itemOrdinal, listRef?)`
  - listRef undefined/null → dùng latest result set
  - listRef number → resolve list ordinal → match item
  - listRef string → semantic match → match item
- [x] preToolValidator: kiểm tra `args.conferenceRef` → `resolveAll` → replace identifier=ID
- [x] retrieveKnowledge handler: kiểm tra `args.conferenceRef` → resolve → filter:id → full RAG data; **luôn save ResultSetState** mỗi khi trả list
- [x] Tool `resolveConferenceRef` đã xóa (handler, declaration, system prompt references)
- [x] Method `resolveByContext` đã xóa (merge vào resolveAll)
- [x] Gemini layer trả về `functionCalls[]` + `functionCallParts`
- [x] Non-streaming handler loop qua tất cả function calls
- [x] Streaming handler loop qua tất cả function calls
- [x] Gom N function responses thành 1 turn với N parts
- [x] System prompt: context window priority — LLM kiểm tra context window trước khi dùng `conferenceRef`
- [x] Không còn TemporalResolver + RankingResolver trong codebase
- [x] Unit test pass ≥ 95%

---

## 10. Phụ thuộc

| Phụ thuộc                 | Ghi chú                                                 |
| ------------------------- | ------------------------------------------------------- |
| P1-01 (ResultSetState V1) | Đã hoàn thành                                           |
| Gemini API                | Đã support multiple functionCalls — chỉ cần sửa handler |
