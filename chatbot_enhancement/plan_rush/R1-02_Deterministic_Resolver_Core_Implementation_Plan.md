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
- `identifier` + `identifierType` — cách cũ (acronym/title/id)

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
  - Loop qua `subAgentLlmResult.functionCalls[]` (plural)
  - Validate + execute từng call tuần tự (sequential) bằng `for...of` loop
- Giữ nguyên logic validate/error handling cho mỗi call
- Gom tất cả `FunctionResponse` parts vào 1 turn duy nhất gửi lại model
- Model turn (history) chứa tất cả function calls trong 1 message với role `model`

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

1. **Kiểm tra context window trước:** Đếm số lượng DANH SÁCH hội nghị hiện có trong lịch sử hội thoại gần đây (context window).
   - Nếu số lượng DANH SÁCH hội nghị trong context window ≥ số thứ tự danh sách hội nghị người dùng yêu cầu → **trả lời trực tiếp** bằng dữ liệu từ context window, **không cần gọi tool**.
   - Ví dụ: Context window đang có 5 DANH SÁCH hội nghị, người dùng hỏi "hội nghị thứ ... của danh sách thứ 3 gần đây nhất" → LLM đọc trực tiếp từ context window và trả lời.

2. **Fallback sang ResultSetState (warm memory):** Nếu context window chỉ có ít DANH SÁCH hội nghị hơn vị trí người dùng yêu cầu → dùng `conferenceRef` để lấy từ warm memory.
   - Ví dụ: Context window chỉ có 2 DANH SÁCH hội nghị, người dùng hỏi "hội nghị thứ ... của danh sách thứ 5 gần đây nhất" → LLM gọi `retrieveKnowledge(conferenceRef: { list: "-5", item: ... })` để backend resolve từ ResultSetState.

3. **ConferenceAgent prompt:** Hướng dẫn cách dùng `conferenceRef` (list + item), re-query pattern, và quy tắc context window priority ở trên.

4. **HostAgent prompt:** Routing khi thấy user dùng positional reference, ưu tiên delegate sang ConferenceAgent.

### Step 10: Enable multiple function calling trong prompt cho tất cả agent và subagent

**Mục tiêu:** Chỉnh prompt cho tất cả agent và subagent để LUÔN LUÔN dùng multiple function calling khi có thể để tiết kiệm API call.

**Nguyên tắc:**

- Khi cần gọi nhiều function, LLM nên gọi tất cả cùng lúc trong 1 response thay vì gọi tuần tự
- Ví dụ: thay vì gọi `retrieveKnowledge` rồi đợi kết quả rồi mới gọi `getConferenceDetails`, LLM nên gọi cả 2 cùng lúc
- LLM có thể gọi nhiều function cùng lúc nếu chúng không có dependencies

**Files cần chỉnh:**

**Tiếng Anh:**

- **File:** `src/chatbot/language/functions/english.ts`
  - Thêm instruction: "When multiple function calls are needed, make all calls in a single response whenever possible to save API calls. Only make sequential calls if there are dependencies between functions."
  - Thêm ví dụ về multiple function calling

- **File:** `src/chatbot/language/instructions/english.ts`
  - Chỉnh prompt cho ConferenceAgent
  - Chỉnh prompt cho HostAgent
  - Chỉnh prompt cho các subagent khác nếu có
  - Thêm instruction về multiple function calling

**Tiếng Việt:**

- **File:** `src/chatbot/language/instructions/vietnamese.ts`
  - Chỉnh prompt cho ConferenceAgent
  - Chỉnh prompt cho HostAgent
  - Chỉnh prompt cho các subagent khác nếu có
  - Thêm instruction về multiple function calling (tiếng Việt)

**Instruction mẫu (tiếng Anh):**

```
IMPORTANT: When you need to call multiple functions, make all calls in a single response whenever possible. This saves API calls and improves performance.

Example:
- BAD: Call retrieveKnowledge, wait for result, then call getConferenceDetails
- GOOD: Call both retrieveKnowledge and getConferenceDetails in the same response

Only make sequential calls if there are dependencies between functions (e.g., you need the result of function A to call function B).
```

**Instruction mẫu (tiếng Việt):**

```
QUAN TRỌNG: Khi cần gọi nhiều function, hãy gọi tất cả cùng lúc trong 1 response khi có thể. Điều này tiết kiệm API call và cải thiện performance.

Ví dụ:
- SAI: Gọi retrieveKnowledge, đợi kết quả, rồi mới gọi getConferenceDetails
- ĐÚNG: Gọi cả retrieveKnowledge và getConferenceDetails cùng lúc trong 1 response

Chỉ gọi tuần tự nếu có dependencies giữa các function (ví dụ: cần kết quả của function A để gọi function B).
```

### Step 11: Support multiple FrontendActions

> **Lưu ý:** Frontend hiện tại chỉ hỗ trợ single `FrontendAction` mỗi message. Để enable multiple actions từ parallel function calls, cần update cả backend và frontend.

#### Backend changes

- **File:** `src/chatbot/shared/types.ts`
  - Thay đổi `ChatHistoryItem.action?: FrontendAction` → `actions?: FrontendAction[]`
  - Thay đổi `ResultUpdate.action?: FrontendAction` → `actions?: FrontendAction[]`
  - Thay đổi `AgentCardResponse.frontendAction?: FrontendAction` → `frontendActions?: FrontendAction[]`
  - Thay đổi `FunctionHandlerOutput.frontendAction?: FrontendAction` → `frontendActions?: FrontendAction[]`

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
  - Thay đổi `HistoryItem.action` → `actions`
  - Thay đổi `MessageDisplay.action` → `actions`

- **File:** `Easyconf-FE-Client/src/app/[locale]/chatbot/stores/messageStore/messageMappers.ts`
  - Update mapper để xử lý `actions[]` thay vì `action`

- **File:** `Easyconf-FE-Client/src/app/[locale]/chatbot/regularchat/MessageContentRenderer.tsx`
  - Update component props: `actions?: FrontendAction[]`
  - Create new component `FrontendActionsAccordion` để render multiple actions
  - Group actions theo category:
    - Navigation & Interaction: `navigate`, `confirmEmailSend`, `openMap`
    - Content Display: `displayList`, `displayConferenceSources`
    - Status Updates: `itemFollowStatusUpdated`, `itemBlacklistStatusUpdated`, `itemCalendarStatusUpdated`
    - Calendar Operations: `addToCalendar`, `removeFromCalendar`
  - Mỗi section có thể collapse/expand (accordion pattern)

- **File:** `Easyconf-FE-Client/src/app/[locale]/chatbot/regularchat/ChatMessageDisplay.tsx`
  - Update component props: `actions?: FrontendAction[]`
  - Pass `actions` xuống `MessageContentRenderer`

- **File:** `Easyconf-FE-Client/src/app/[locale]/chatbot/regularchat/FrontendActionsAccordion.tsx` (NEW)
  - Create accordion component với các sections:
    - "Navigation & Actions" - cho navigate, confirmEmailSend, openMap
    - "Content" - cho displayList, displayConferenceSources
    - "Status Updates" - cho itemFollowStatusUpdated, itemBlacklistStatusUpdated, itemCalendarStatusUpdated
    - "Calendar" - cho addToCalendar, removeFromCalendar
  - Mặc định expand section quan trọng nhất (Navigation & Actions)
  - Các section khác collapse mặc định để tiết kiệm space
  - User có thể click để expand/collapse từng section

#### Frontend action display strategy

Khi có multiple actions, frontend sử dụng **Accordion Pattern** để hiển thị:

**Accordion Structure:**

```
▼ Navigation & Actions (default expanded)
  - navigate button
  - confirmEmailSend dialog
  - openMap component

▶ Content (default collapsed)
  - displayList component
  - displayConferenceSources component

▶ Status Updates (default collapsed)
  - itemFollowStatusUpdated box
  - itemBlacklistStatusUpdated box
  - itemCalendarStatusUpdated box

▶ Calendar (default collapsed)
  - addToCalendar component
  - removeFromCalendar component
```

**Ưu điểm của Accordion Pattern:**

- Tiết kiệm vertical space - chỉ hiển thị quan trọng nhất
- User có thể expand/collapse theo nhu cầu
- Group actions logically theo category
- Flexible cho future action types
- Không bỏ qua bất kỳ action nào

**Implementation Details:**

- Mặc định expand section "Navigation & Actions" vì thường quan trọng nhất
- Các section khác collapse để tránh clutter
- Icon ▼/▶ để indicate expand/collapse state
- Smooth animation cho expand/collapse
- Persist state nếu user scroll (optional)

---

## 6. File structure

```
src/  # Easyconf-Chatbot-Server (Backend)
  chatbot/
    shared/
      types.ts                                # [SỬA] FrontendAction[] (Step 11)
    gemini/
      pooledGemini.ts                         # [SỬA] multiple functionCalls (Step 1)
    models/
      groqCohereHybrid.ts                     # [SỬA] multiple functionCalls (Step 1)
    handlers/
      subAgent.handler.ts                     # [SỬA] multiple function calls sequential (Step 2), collect frontendActions (Step 11)
      hostAgent.nonStreaming.handler.ts        # [SỬA] multiple function calls sequential (Step 3), collect frontendActions (Step 11)
      hostAgent.streaming.handler.ts           # [SỬA] multiple function calls sequential (Step 3), collect frontendActions (Step 11)
      retrieveKnowledge.handler.ts            # [SỬA] + conferenceRef, save RS (Step 6)
      resolveConferenceRef.handler.ts         # [XÓA] (Step 7)
    guards/
      preToolValidator.ts                     # [SỬA] xử lý conferenceRef (Step 5)
    language/
      functions/
        english.ts                            # [SỬA] + conferenceRef (Step 8), delete resolveConferenceRef (Step 7), multiple function calling instruction (Step 10)
        vietnamese.ts
        spanish.ts
      instructions/
        english.ts                            # [SỬA] context window priority (Step 9), delete resolveConferenceRef refs (Step 7), multiple function calling instruction (Step 10)
        vietnamese.ts                          # [SỬA] context window priority (Step 9), multiple function calling instruction (Step 10)
        spanish.ts
        chinese.ts                             # [SỬA] multiple function calling instruction (Step 10)
        japanese.ts                            # [SỬA] multiple function calling instruction (Step 10)
        korean.ts                              # [SỬA] multiple function calling instruction (Step 10)
        french.ts                              # [SỬA] multiple function calling instruction (Step 10)
        german.ts                              # [SỬA] multiple function calling instruction (Step 10)
        russian.ts                             # [SỬA] multiple function calling instruction (Step 10)
        arabic.ts                              # [SỬA] multiple function calling instruction (Step 10)
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
      regular-chat.types.ts                   # [SỬA] FrontendAction[] (Step 11)
    stores/messageStore/
      messageMappers.ts                       # [SỬA] handle actions[] (Step 11)
    regularchat/
      MessageContentRenderer.tsx              # [SỬA] render actions[] (Step 11)
      ChatMessageDisplay.tsx                  # [SỬA] pass actions[] (Step 11)
      FrontendActionsAccordion.tsx            # [MỚI] accordion component (Step 11)
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

---

## 11. Refactor Plan — Result Set Saving Pattern

### 11.1 Mục tiêu refactor

Thay đổi pattern lưu result set từ **backend tự động lưu** sang **ConferenceAgent/HostAgent chủ động gọi hàm lưu** để:

- Host agent có thể kiểm soát thứ tự và nội dung result set được lưu
- **Lưu cả lists từ LLM output VÀ lists từ user** (user có thể paste danh sách hội nghị họ quan tâm)
- Tránh mất thông tin khi user reference lại list họ đã gửi

### 11.2 Nguyên lý hoạt động

**Luồng hiện tại (backend tự động lưu):**

```
ConferenceAgent gọi retrieveKnowledge → Backend tự động save ResultSetState → Host agent không kiểm soát được thứ tự
```

**Luồng mới (ConferenceAgent/HostAgent chủ động lưu):**

```
Luồng 1: ConferenceAgent lưu LLM output lists
→ Host agent route đến ConferenceAgent
→ ConferenceAgent gọi 2 retrieveKnowledge trả về 2 list khác nhau (List 1: 10 kết quả, List 2: 5 kết quả)
→ Host agent muốn tổng hợp:
  - Trường hợp 1: Tổng hợp thành 1 list duy nhất (ví dụ: 9 kết quả)
    → Host agent yêu cầu ConferenceAgent gọi hàm save result set với 9 ids, CÓ THỨ TỰ
    → KHÔNG ĐƯỢC truyền lung tung
  - Trường hợp 2: Tổng hợp thành 2 list (ví dụ: List 1 lấy 5, List 2 lấy 3)
    → Host agent yêu cầu ConferenceAgent gọi hàm save result set 2 lần trong cùng 1 turn
    → Lưu List 1 trước, rồi List 2 sau
    → PHẢI ĐẢM BẢO THỨ TỰ LƯU LIST, ngay cả khi gọi song song

Luồng 2: HostAgent lưu user-sent lists
→ User gửi message: "Tôi quan tâm 3 hội nghị này: ICML, NeurIPS, AAAI — cho tôi so sánh"
→ HostAgent detect user đang gửi list hội nghị (có tên conference)
→ HostAgent gọi hàm save result set với source="user" để lưu list này
→ Sau này khi user nói "cái thứ 2 trong list tôi gửi", hệ thống có thể resolve
```

### 11.3 Implementation Steps

#### Step R0: Đổi tên trường trong ResultSetState type

**File:** `src/types/resultSetState.types.ts`

Đổi tên 2 trường để reflect đúng ý nghĩa:

- `queryText` → `description` (mô tả về list, không chỉ là query)
- `queryEmbedding` → `descriptionEmbedding` (embedding của description)

```typescript
export interface ResultSetState {
  /** Conversation ID */
  conversationId: string;

  /** Mô tả về result set (vd: "list AI conferences 2026", "user's shortlist") — dùng để match contextHint */
  description: string;

  /** Vector embedding của description để semantic matching với contextHint */
  descriptionEmbedding: number[];

  /** Danh sách ID conference có thứ tự — nguồn sự thật duy nhất cho ordinal resolver */
  orderedConferenceIds: string[];

  /** Nguồn của result set: "model" (LLM output) hoặc "user" (user sent list) */
  source: "model" | "user";

  /** Thời điểm tạo (ISO-8601 UTC) */
  createdAt: string;

  /** Thời điểm hết hạn (ISO-8601 UTC) — sau 20 phút kể từ createdAt */
  expiresAt: string;
}
```

**Files cần cập nhật sau khi đổi tên:**

- `src/services/resultSetState/store.service.ts` — cập nhật field names + thêm field `source`
- `src/services/resultSetState/resolver.service.ts` — cập nhật field names
- Bất kỳ file nào khác tham chiếu `queryText` hoặc `queryEmbedding`

#### Step R0b: Refactor `orderedConferenceIds` thành `orderedIdentifiers` với type information

**File:** `src/types/resultSetState.types.ts`

Đổi từ `orderedConferenceIds: string[]` sang generic identifiers để support multiple identifier types (acronym, title, id) consistent với `manageFollow.handler.ts`:

```typescript
interface StoredIdentifier {
  type: "acronym" | "title" | "id";
  value: string;
}

export interface ResultSetState {
  /** Conversation ID */
  conversationId: string;

  /** Mô tả về result set (vd: "list AI conferences 2026", "user's shortlist") — dùng để match contextHint */
  description: string;

  /** Vector embedding của description để semantic matching với contextHint */
  descriptionEmbedding: number[];

  /** Danh sách identifier có thứ tự — hỗ trợ nhiều loại identifier (acronym, title, id) */
  orderedIdentifiers: StoredIdentifier[];

  /** Nguồn của result set: "model" (LLM output) hoặc "user" (user sent list) */
  source: "model" | "user";

  /** Thời điểm tạo (ISO-8601 UTC) */
  createdAt: string;

  /** Thời điểm hết hạn (ISO-8601 UTC) — sau 20 phút kể từ createdAt */
  expiresAt: string;
}
```

**Files cần cập nhật sau khi refactor:**

- `src/services/resultSetState/store.service.ts` — cập nhật logic lưu identifiers thay vì chỉ conference IDs
- `src/services/resultSetState/resolver.service.ts` — cập nhật logic resolve theo identifier type
- `src/chatbot/handlers/saveResultSet.handler.ts` — cập nhật interface accept identifiers thay vì chỉ IDs
- `src/chatbot/language/functions/english.ts` — cập nhật function declaration cho identifiers
- `src/chatbot/language/functions/vietnamese.ts` — cập nhật function declaration cho identifiers
- Bất kỳ file nào khác tham chiếu `orderedConferenceIds`

#### Step R1: Tạo hàm `saveResultSet` cho ConferenceAgent

**File:** `src/chatbot/handlers/saveResultSet.handler.ts` (NEW)

```typescript
export async function saveResultSetHandler(
  conversationId: string,
  description: string,
  orderedIdentifiers: StoredIdentifier[],
  source: "model" | "user",
  listName?: string,
): Promise<SaveResultSetOutput> {
  // Validate orderedIdentifiers là array và không rỗng
  if (!Array.isArray(orderedIdentifiers) || orderedIdentifiers.length === 0) {
    return {
      success: false,
      error: "orderedIdentifiers must be a non-empty array",
    };
  }

  // Validate conversationId
  if (!conversationId) {
    return {
      success: false,
      error: "conversationId is required",
    };
  }

  try {
    // Lưu vào ResultSetState với thứ tự được bảo toàn
    const resultSetStateStore = container.resolve(ResultSetStateStore);
    await resultSetStateStore.save(
      conversationId,
      description,
      orderedIdentifiers, // Thứ tự array được bảo toàn
      source, // "model" hoặc "user"
    );

    return {
      success: true,
      savedCount: orderedIdentifiers.length,
      listName: listName || null,
      source,
    };
  } catch (error) {
    const { message: errorMessage } = getErrorMessageAndStack(error);
    return {
      success: false,
      error: errorMessage,
    };
  }
}
```

**Interface definitions:**

```typescript
interface SaveResultSetOutput {
  success: boolean;
  savedCount?: number;
  listName?: string | null;
  source?: "model" | "user";
  error?: string;
}
```

#### Step R2: Thêm `saveResultSet` vào function registry

**File:** `src/chatbot/gemini/functionRegistry.ts`

```typescript
import { saveResultSetHandler } from "../handlers/saveResultSet.handler";

// Trong functionRegistryMap:
"saveResultSet": {
  handler: saveResultSetHandler,
  allowedAgents: ["ConferenceAgent", "HostAgent"],  // Cả 2 agent đều được phép gọi (HostAgent để save user lists)
},
```

#### Step R3: Thêm function declaration cho `saveResultSet`

**File:** `src/chatbot/language/functions/english.ts`

```typescript
export const englishSaveResultSetDeclaration: FunctionDeclaration = {
  name: "saveResultSet",
  description:
    "Save a set of conference IDs as a named result set for later reference by ordinal position. This allows the HostAgent to control which conferences are saved and in what order. IMPORTANT: The order of conferenceIds in the array MUST be preserved exactly as provided - this order will be used for ordinal references (1st, 2nd, etc.).",
  parameters: {
    type: Type.OBJECT,
    properties: {
      conferenceIds: {
        type: Type.ARRAY,
        items: { type: Type.STRING },
        description:
          "Array of conference IDs to save. The order of IDs in this array MUST be preserved - the first ID will be position 1, second ID will be position 2, etc. Do NOT shuffle or reorder this array.",
        minItems: 1,
      },
      source: {
        type: Type.STRING,
        description:
          "Source of this result set: 'model' if generated by LLM (default), 'user' if sent by user in their message. HostAgent should use 'user' when detecting conference lists in user messages.",
        enum: ["model", "user"],
        nullable: true,
      },
      listName: {
        type: Type.STRING,
        description:
          "Optional name for this result set (e.g., 'list1', 'list2', 'primary', 'secondary'). This allows the HostAgent to reference specific result sets by name when combining multiple lists.",
        nullable: true,
      },
    },
    required: ["conferenceIds"],
  },
};
```

**File:** `src/chatbot/language/functions/vietnamese.ts`

```typescript
export const vietnameseSaveResultSetDeclaration: FunctionDeclaration = {
  name: "saveResultSet",
  description:
    "Lưu một set conference IDs dưới dạng result set có tên để tham chiếu sau bằng vị trí ordinal. Điều này cho phép HostAgent kiểm soát những conference nào được lưu và theo thứ tự nào. QUAN TRỌNG: Thứ tự của conferenceIds trong array PHẢI được bảo toàn chính xác như được cung cấp - thứ tự này sẽ được dùng cho tham chiếu ordinal (thứ 1, thứ 2, v.v.).",
  parameters: {
    type: Type.OBJECT,
    properties: {
      conferenceIds: {
        type: Type.ARRAY,
        items: { type: Type.STRING },
        description:
          "Array của conference IDs để lưu. Thứ tự của IDs trong array này PHẢI được bảo toàn - ID đầu tiên sẽ là vị trí 1, ID thứ hai sẽ là vị trí 2, v.v. KHÔNG được xáo trộn hoặc reorder array này.",
        minItems: 1,
      },
      source: {
        type: Type.STRING,
        description:
          "Nguồn của result set này: 'model' nếu được tạo bởi LLM (default), 'user' nếu được gửi bởi user trong message của họ. HostAgent nên dùng 'user' khi phát hiện list hội nghị trong message của user.",
        enum: ["model", "user"],
        nullable: true,
      },
      listName: {
        type: Type.STRING,
        description:
          "Tên tùy chọn cho result set này (ví dụ: 'list1', 'list2', 'primary', 'secondary'). Điều này cho phép HostAgent tham chiếu đến các result set cụ thể theo tên khi kết hợp nhiều list.",
        nullable: true,
      },
    },
    required: ["conferenceIds"],
  },
};
```

#### Step R4: Bổ sung `saveResultSet` vào ConferenceAgent và HostAgent function lists

**File:** `src/chatbot/utils/languageConfig.ts`

```typescript
// Trong getAgentLanguageConfig() cho ConferenceAgent:
const functionDeclarations = [
  // ... các functions hiện tại
  englishSaveResultSetDeclaration, // Thêm vào
  // ...
];

// Trong getAgentLanguageConfig() cho HostAgent:
const functionDeclarations = [
  // ... các functions hiện tại
  englishSaveResultSetDeclaration, // Thêm vào (để save user lists)
  // ...
];
```

#### Step R5: Cập nhật ResultSetStateService để hỗ trợ named result sets + new fields

**File:** `src/services/resultSetState/store.service.ts`

```typescript
// Sửa method save hoặc thêm method mới:
async saveResultSet(
  conversationId: string,
  userId: string,
  conferenceIds: string[],
  source: "model" | "user" = "model",
  listName?: string,
): Promise<void> {
  // Bảo toàn thứ tự của conferenceIds
  const orderedConferenceIds = [...conferenceIds];

  // Generate embedding cho description (dùng service có sẵn)
  const description = listName || `result_set_${Date.now()}`;
  const descriptionEmbedding = await this.embeddingService.generateEmbedding(description);

  const state: ResultSetState = {
    conversationId,
    userId,
    orderedConferenceIds,  // Thứ tự được bảo toàn
    description,  // Đổi từ queryText
    descriptionEmbedding,  // Đổi từ queryEmbedding
    source,  // Thêm field source
    createdAt: new Date().toISOString(),
    expiresAt: new Date(Date.now() + 20 * 60 * 1000).toISOString(),  // 20 phút
  };

  await this.model.create(state);
}
```

#### Step R6: Cập nhật ResultSetResolver để hỗ trợ named result sets + new field names

**File:** `src/services/resultSetState/resolver.service.ts`

```typescript
// Sửa method resolveAll:
async resolveAll(
  conversationId: string,
  itemOrdinal: number,
  listRef?: number | string,
): Promise<ResolveResult> {
  // Nếu listRef là string → tìm theo listName (nếu có) hoặc description
  if (typeof listRef === "string") {
    // Thử tìm theo listName trước (nếu schema có field này)
    const state = await this.model.findOne({
      conversationId,
      listName: listRef,
    });
    if (state) {
      return this.resolveFromState(state, itemOrdinal);
    }

    // Nếu không tìm theo listName, thử semantic match với description
    const states = await this.model.find({ conversationId });
    // Semantic matching logic với descriptionEmbedding
    const matched = this.semanticMatch(listRef, states);
    if (matched) {
      return this.resolveFromState(matched, itemOrdinal);
    }

    return { resolvedId: null, reasonCode: "LIST_NOT_FOUND", confidence: 0 };
  }

  // Logic hiện tại cho number (ordinal) và undefined/null
  // Cập nhật để dùng description thay vì queryText
  // ...
}
```

**Lưu ý:** Cập nhật mọi tham chiếu đến `queryText` và `queryEmbedding` trong resolver thành `description` và `descriptionEmbedding`.

#### Step R7: Xóa auto-save trong retrieveKnowledge handler

**File:** `src/chatbot/handlers/retrieveKnowledge.handler.ts`

```typescript
// Xóa hoặc comment out code auto-save (dòng 254-268 trong plan cũ)
// ConferenceAgent sẽ chủ động gọi saveResultSet khi cần
```

#### Step R8: Cập nhật HostAgent prompt (Tiếng Anh)

**File:** `src/chatbot/language/instructions/english.ts`

Thêm vào section routing cho ConferenceAgent:

```
### RESULT SET SAVING — CONTROLLED BY HOST AGENT

**When ConferenceAgent returns multiple result lists:**
You (HostAgent) control how these lists are saved for later ordinal reference:

**Scenario 1: Combine into single list**
- If ConferenceAgent returns 2 lists (List 1: 10 results, List 2: 5 results) and you want to display 9 specific results to the user:
  → Route to ConferenceAgent with ADDITIONAL_INSTRUCTION requesting it to call `saveResultSet` with the 9 specific conference IDs
  → The IDs MUST be passed in the correct order (first ID = position 1, second ID = position 2, etc.)
  → DO NOT pass IDs in random order

**Scenario 2: Keep as separate lists**
- If you want to display 2 separate lists (List 1: 5 results, List 2: 3 results):
  → Route to ConferenceAgent with ADDITIONAL_INSTRUCTION requesting it to call `saveResultSet` TWICE in the SAME turn
  → First call: saveResultSet({ conferenceIds: [ids_for_list1], listName: "list1" })
  → Second call: saveResultSet({ conferenceIds: [ids_for_list2], listName: "list2" })
  → CRITICAL: List 1 MUST be saved BEFORE List 2
  → Even when calling saveResultSet in parallel (multiple function calls), ensure the calls are ordered in the response so list1 is processed first

**When USER sends a conference list:**
If the user sends a message containing conference names/IDs (e.g., "I'm interested in ICML, NeurIPS, AAAI"):
- Extract the conference IDs (you may need to resolve names to IDs first)
- Call `saveResultSet` yourself with `source: "user"` to save this list
- Use a descriptive listName like "user's shortlist" or "user's AI conferences"
- This ensures user-sent lists are trackable for later ordinal references

**Important Rules:**
- Always preserve the exact order of conference IDs passed to saveResultSet
- When saving multiple lists, use listName parameter to distinguish them (e.g., "list1", "list2")
- The order of saveResultSet calls determines the order of result sets for later reference
- When saving user lists, ALWAYS use `source: "user"`
```

#### Step R9: Cập nhật HostAgent prompt (Tiếng Việt)

**File:** `src/chatbot/language/instructions/vietnamese.ts`

Thêm vào section routing cho ConferenceAgent:

```
### LƯU RESULT SET — ĐƯỢC CONTROL BỞI HOST AGENT

**Khi ConferenceAgent trả về nhiều result lists:**
Bạn (HostAgent) kiểm soát cách các lists này được lưu để tham chiếu ordinal sau này:

**Tình huống 1: Tổng hợp thành 1 list duy nhất**
- Nếu ConferenceAgent trả về 2 lists (List 1: 10 kết quả, List 2: 5 kết quả) và bạn muốn hiển thị 9 kết quả cụ thể cho người dùng:
  → Route đến ConferenceAgent với ADDITIONAL_INSTRUCTION yêu cầu nó gọi `saveResultSet` với 9 conference IDs cụ thể
  → Các IDs PHẢI được truyền theo đúng thứ tự (ID đầu tiên = vị trí 1, ID thứ hai = vị trí 2, v.v.)
  → KHÔNG ĐƯỢC truyền IDs theo cách ngẫu nhiên

**Tình huống 2: Giữ thành 2 lists riêng biệt**
- Nếu bạn muốn hiển thị 2 lists riêng biệt (List 1: 5 kết quả, List 2: 3 kết quả):
  → Route đến ConferenceAgent với ADDITIONAL_INSTRUCTION yêu cầu nó gọi `saveResultSet` HAI LẦN trong CÙNG MỘT turn
  → Lần gọi đầu tiên: saveResultSet({ conferenceIds: [ids_cho_list1], listName: "list1" })
  → Lần gọi thứ hai: saveResultSet({ conferenceIds: [ids_cho_list2], listName: "list2" })
  → QUAN TRỌNG: List 1 PHẢI được lưu TRƯỚC List 2
  → Ngay cả khi gọi saveResultSet song song (nhiều function calls cùng lúc), đảm bảo các calls được sắp xếp trong response để list1 được xử lý trước

**Khi USER gửi list hội nghị:**
Nếu user gửi message chứa tên/ID hội nghị (ví dụ: "Tôi quan tâm ICML, NeurIPS, AAAI"):
- Trích xuất conference IDs (có thể cần resolve tên sang ID trước)
- Tự gọi `saveResultSet` với `source: "user"` để lưu list này
- Dùng listName mô tả như "user's shortlist" hoặc "user's AI conferences"
- Điều này đảm bảo user-sent lists có thể track cho các tham chiếu ordinal sau này

**Quy tắc quan trọng:**
- Luôn bảo toàn đúng thứ tự của conference IDs được truyền cho saveResultSet
- Khi lưu nhiều lists, dùng parameter listName để phân biệt (ví dụ: "list1", "list2")
- Thứ tự của các lần gọi saveResultSet quyết định thứ tự của result sets để tham chiếu sau này
- Khi lưu user lists, LUÔN LUÔN dùng `source: "user"`
```

#### Step R10: Cập nhật ConferenceAgent prompt (Tiếng Anh)

**File:** `src/chatbot/language/instructions/english.ts`

Thêm instruction mới:

```
### SAVE RESULT SET — CONTROLLED BY HOST AGENT

You have the ability to call `saveResultSet` to save conference IDs for later ordinal reference. However, this should ONLY be done when explicitly instructed by the HostAgent via ADDITIONAL_INSTRUCTION in the taskDescription.

When HostAgent instructs you to save result sets:
- Preserve the EXACT order of conference IDs passed to saveResultSet
- If saving multiple lists, use the listName parameter to distinguish them
- Follow the ordering specified by HostAgent even when making parallel function calls
```

#### Step R11: Cập nhật ConferenceAgent prompt (Tiếng Việt)

**File:** `src/chatbot/language/instructions/vietnamese.ts`

Thêm instruction mới:

```
### LƯU RESULT SET — ĐƯỢC CONTROL BỞI HOST AGENT

Bạn có khả năng gọi `saveResultSet` để lưu conference IDs để tham chiếu ordinal sau này. Tuy nhiên, điều này CHỈ nên được thực hiện khi được HostAgent hướng dẫn rõ ràng qua ADDITIONAL_INSTRUCTION trong taskDescription.

Khi HostAgent hướng dẫn bạn lưu result sets:
- Bảo toàn ĐÚNG thứ tự của conference IDs được truyền cho saveResultSet
- Nếu lưu nhiều lists, dùng parameter listName để phân biệt
- Tuân theo thứ tự được chỉ định bởi HostAgent ngay cả khi gọi nhiều function cùng lúc
```

#### Step R12: Cập nhật HostAgent prompt — Optimized list search pattern

**File:** `src/chatbot/language/instructions/english.ts`

Thêm instruction mới vào section context window priority:

```
### OPTIMIZED LIST SEARCH — SEARCH IN MODEL RESPONSES ONLY

When the user refers to a conference list that you have previously shown to them (e.g., "the list from earlier", "the AI conferences you showed me", "that list from the last turn"), search for that list ONLY in messages with:
- role: "model"
- The LAST text part within that message

**Why this optimization?**
- Conference lists are typically displayed in model responses (not user messages or function responses)
- The last text part in a model message is where the list content is usually shown
- This reduces search scope and improves accuracy

**Example:**
```

History:
[
{ role: "user", parts: [{ text: "Find AI conferences" }] },
{ role: "model", parts: [{ functionCall: {...} }, { text: "Here are 10 AI conferences: [list content]" }] },
{ role: "function", parts: [{ functionResponse: {...} }] },
{ role: "model", parts: [{ text: "Based on the results..." }] }
]

User: "Show me the 2nd conference from that list"
→ Search in role="model" messages
→ Focus on the LAST text part of each model message
→ Found list in the second model message (with functionCall + text)
→ Use ordinal reference on that list

```

**Important:**
- Do NOT search in role="user" messages for conference lists
- Do NOT search in role="function" messages for conference lists
- Focus on the LAST text part because lists are typically the final content in a model response
- If no list found in model messages, fallback to using conferenceRef from ResultSetState
```

**File:** `src/chatbot/language/instructions/vietnamese.ts`

Thêm instruction mới vào section context window priority:

```
### TỐI ƯU HÓA TÌM KIẾM LIST — CHỈ TÌM TRONG MODEL RESPONSES

Khi người dùng nhắc đến một list hội nghị mà bạn đã từng show cho họ trước đó (ví dụ: "list từ trước", "các hội nghị AI bạn đã show tôi", "list đó từ turn cuối"), chỉ tìm kiếm list đó trong các tin nhắn có:
- role: "model"
- Phần text CUỐI CÙNG trong tin nhắn đó

**Tại sao tối ưu hóa này?**
- List hội nghị thường được hiển thị trong model responses (không phải trong user messages hay function responses)
- Phần text cuối cùng trong một model message thường là nơi nội dung list được show
- Điều này giảm phạm vi tìm kiếm và cải thiện độ chính xác

**Ví dụ:**
```

History:
[
{ role: "user", parts: [{ text: "Tìm hội nghị AI" }] },
{ role: "model", parts: [{ functionCall: {...} }, { text: "Dưới đây là 10 hội nghị AI: [nội dung list]" }] },
{ role: "function", parts: [{ functionResponse: {...} }] },
{ role: "model", parts: [{ text: "Dựa trên kết quả..." }] }
]

Người dùng: "Cho tôi xem hội nghị thứ 2 trong list đó"
→ Tìm trong tin nhắn role="model"
→ Focus vào phần text CUỐI CÙNG của mỗi model message
→ Tìm thấy list trong model message thứ 2 (có functionCall + text)
→ Dùng tham chiếu ordinal trên list đó

```

**Quan trọng:**
- KHÔNG tìm trong tin nhắn role="user" cho list hội nghị
- KHÔNG tìm trong tin nhắn role="function" cho list hội nghị
- Focus vào phần text CUỐI CÙNG vì list thường là nội dung cuối cùng trong một model response
- Nếu không tìm thấy list trong model messages, fallback sang dùng conferenceRef từ ResultSetState
```

### 11.4 Test Plan cho Refactor

| #   | Test case                                                                                    | Expected                                             |
| --- | -------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| 1   | ConferenceAgent gọi saveResultSet với 5 IDs                                                  | Success, savedCount=5                                |
| 2   | ConferenceAgent gọi saveResultSet với listName="list1"                                       | Success, listName="list1"                            |
| 3   | Host agent yêu cầu lưu 9 IDs theo thứ tự                                                     | Thứ tự được bảo toàn chính xác                       |
| 4   | Host agent yêu cầu lưu 2 lists trong cùng 1 turn                                             | Cả 2 lists được lưu, list1 trước list2               |
| 5   | ConferenceAgent gọi saveResultSet với empty array                                            | Error: conferenceIds must be non-empty array         |
| 6   | Resolver tìm result set theo listName="list1"                                                | Tìm thấy, trả về đúng list                           |
| 7   | Auto-save trong retrieveKnowledge đã xóa                                                     | Không còn auto-save tự động                          |
| 8   | HostAgent detect user list và lưu với source="user"                                          | Success, source="user"                               |
| 9   | ResultSetState có field description thay vì queryText                                        | Field name đã đổi thành description                  |
| 10  | ResultSetState có field source                                                               | Field source tồn tại với giá trị "model" hoặc "user" |
| 11  | HostAgent lưu user list → user reference "cái thứ 2 trong list tôi gửi" → resolve thành công | Resolve thành công, trả về đúng conference ID        |

### 11.5 File structure updates

```
src/  # Easyconf-Chatbot-Server (Backend)
  types/
    resultSetState.types.ts                     # [SỬA] đổi tên queryText→description, queryEmbedding→descriptionEmbedding, thêm source (Step R0)
  chatbot/
    handlers/
      saveResultSet.handler.ts                 # [MỚI] (Step R1)
      retrieveKnowledge.handler.ts              # [SỬA] xóa auto-save (Step R7)
    gemini/
      functionRegistry.ts                       # [SỬA] thêm saveResultSet, cho phép cả ConferenceAgent và HostAgent (Step R2)
    language/
      functions/
        english.ts                              # [SỬA] thêm declaration với source parameter (Step R3)
        vietnamese.ts                           # [SỬA] thêm declaration với source parameter (Step R3)
      instructions/
        english.ts                              # [SỬA] thêm result set saving instruction + user list detection (Step R8, R10)
        vietnamese.ts                            # [SỬA] thêm result set saving instruction + user list detection (Step R9, R11)
    utils/
      languageConfig.ts                         # [SỬA] thêm saveResultSet declaration cho cả ConferenceAgent và HostAgent (Step R4)
  services/
    resultSetState/
      store.service.ts                          # [SỬA] hỗ trợ named result sets + new fields (description, descriptionEmbedding, source) (Step R5)
      resolver.service.ts                       # [SỬA] hỗ trợ named result sets + new field names (Step R6)
```
