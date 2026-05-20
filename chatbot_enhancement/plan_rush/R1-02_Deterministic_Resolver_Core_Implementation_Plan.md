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

#### Step R1: Tạo class `SaveResultSetHandler` cho ConferenceAgent

**File:** `src/chatbot/handlers/saveResultSet.handler.ts` (NEW)

```typescript
import { IFunctionHandler } from "../interface/functionHandler.interface";
import { FunctionHandlerInput, FunctionHandlerOutput } from "../shared/types";
import { container } from "tsyringe";
import { StoredIdentifier } from "../../types/resultSetState.types";
import { ResultSetStateStore } from "../../services/resultSetState/store.service";
import { getErrorMessageAndStack } from "../../utils/errorUtils";

interface SaveResultSetOutput {
  success: boolean;
  savedCount?: number;
  listName?: string | null;
  source?: "model" | "user";
  error?: string;
}

export class SaveResultSetHandler implements IFunctionHandler {
  async execute(context: FunctionHandlerInput): Promise<FunctionHandlerOutput> {
    const { args, conversationId } = context;
    const { description, orderedIdentifiers, source, listName } = args;

    // Validate description không rỗng
    if (
      !description ||
      typeof description !== "string" ||
      description.trim() === ""
    ) {
      return {
        modelResponseContent: JSON.stringify({
          success: false,
          error: "description must be a non-empty string",
        }),
      };
    }

    // Validate source là "model" hoặc "user"
    if (source !== "model" && source !== "user") {
      return {
        modelResponseContent: JSON.stringify({
          success: false,
          error: "source must be either 'model' or 'user'",
        }),
      };
    }

    // Validate orderedIdentifiers không rỗng
    if (!Array.isArray(orderedIdentifiers) || orderedIdentifiers.length === 0) {
      return {
        modelResponseContent: JSON.stringify({
          success: false,
          error: "orderedIdentifiers must be a non-empty array",
        }),
      };
    }

    try {
      // Lưu vào ResultSetState với thứ tự được bảo toàn
      const resultSetStateStore = container.resolve(ResultSetStateStore);
      await resultSetStateStore.save(
        conversationId,
        description,
        orderedIdentifiers,
        source,
      );

      const output: SaveResultSetOutput = {
        success: true,
        savedCount: orderedIdentifiers.length,
        listName: listName || null,
        source,
      };

      return {
        modelResponseContent: JSON.stringify(output),
      };
    } catch (error) {
      const { message: errorMessage } = getErrorMessageAndStack(error);
      const output: SaveResultSetOutput = {
        success: false,
        error: errorMessage,
      };
      return {
        modelResponseContent: JSON.stringify(output),
      };
    }
  }
}
```

#### Step R2: Thêm `SaveResultSetHandler` vào function registry

**File:** `src/chatbot/gemini/functionRegistry.ts`

```typescript
import { SaveResultSetHandler } from "../handlers/saveResultSet.handler";

// Trong functionRegistry object:
const functionRegistry: Record<string, IFunctionHandler> = {
  // ... các handlers hiện tại
  saveResultSet: new SaveResultSetHandler(),
  // ...
};
```

#### Step R3: Thêm function declaration cho `saveResultSet`

**File:** `src/chatbot/language/functions/english.ts`

```typescript
export const englishSaveResultSetDeclaration: FunctionDeclaration = {
  name: "saveResultSet",
  description:
    "Save a set of conference identifiers (acronym, title, or ID) as a named result set for later reference by ordinal position. This allows the HostAgent to control which conferences are saved and in what order. IMPORTANT: The order of orderedIdentifiers in the array MUST be preserved exactly as provided - this order will be used for ordinal references (1st, 2nd, etc.).",
  parameters: {
    type: Type.OBJECT,
    properties: {
      description: {
        type: Type.STRING,
        description:
          "Description of this result set (e.g., 'list of AI conferences 2026', 'user's shortlist'). This will be used for semantic matching when referencing result sets.",
      },
      orderedIdentifiers: {
        type: Type.ARRAY,
        items: {
          type: Type.OBJECT,
          properties: {
            type: {
              type: Type.STRING,
              enum: ["acronym", "title", "id"],
              description: "Type of identifier: 'acronym', 'title', or 'id'.",
            },
            value: {
              type: Type.STRING,
              description:
                "The actual identifier value (e.g., 'ICML', 'International Conference on Machine Learning', 'conf-123').",
            },
          },
          required: ["type", "value"],
        },
        description:
          "Array of conference identifiers to save. Each identifier has a type and value. The order of identifiers in this array MUST be preserved - the first identifier will be position 1, second will be position 2, etc. Do NOT shuffle or reorder this array.",
        minItems: 1,
      },
      source: {
        type: Type.STRING,
        description:
          "Source of this result set: 'model' if generated by LLM, 'user' if sent by user in their message. HostAgent should use 'user' when detecting conference lists in user messages.",
        enum: ["model", "user"],
      },
      listName: {
        type: Type.STRING,
        description:
          "Optional name for this result set (e.g., 'list1', 'list2', 'primary', 'secondary'). This allows the HostAgent to reference specific result sets by name when combining multiple lists.",
        nullable: true,
      },
    },
    required: ["description", "orderedIdentifiers", "source"],
  },
};
```

#### Step R4: Bổ sung `saveResultSet` vào HostAgent function list

**File:** `src/chatbot/utils/languageConfig.ts`

```typescript
// Trong getAgentLanguageConfig() cho HostAgent:
const functionDeclarations = [
  // ... các functions hiện tại
  englishSaveResultSetDeclaration, // Thêm vào (để save user lists và control result sets)
  // ...
];

// KHÔNG thêm vào ConferenceAgent - chỉ HostAgent được phép save result sets
```

#### Step R5: ~~Cập nhật ResultSetStateService~~ (ĐÃ HOÀN THÀNH)

**File:** `src/services/resultSetState/store.service.ts`

**Đã được implement với signature hiện tại:**

```typescript
async save(
  conversationId: string,
  description: string,  // Required parameter
  orderedIdentifiers: StoredIdentifier[],  // Đã dùng StoredIdentifier[]
  source: "model" | "user",  // Required, không có default
): Promise<void> {
  // Đã sử dụng create thay vì findOneAndUpdate với upsert
  await ResultSetStateModel.create({
    conversationId,
    description,
    descriptionEmbedding,
    orderedIdentifiers,
    source,
    createdAt: now,
    expiresAt,
  });
}
```

**Đã hoàn thành:**

- Đổi từ `queryText` → `description`
- Đổi từ `queryEmbedding` → `descriptionEmbedding`
- Đổi từ `orderedConferenceIds: string[]` → `orderedIdentifiers: StoredIdentifier[]`
- Thêm field `source: "model" | "user"`
- Xóa parameter `userId` (không cần thiết)
- Xóa parameter `listName` (không được implement)
- Sử dụng `create` thay vì `findOneAndUpdate` với `upsert` (để cho phép multiple result sets với description giống nhau)

#### Step R6: Cập nhật listRef type để hỗ trợ object với description, source, và ordinal

**File:** `src/types/resultSetState.types.ts`

```typescript
export interface IResultSetResolver {
  /**
   * Resolve conference identifiers từ result set.
   *
   * @param conversationId - ID của conversation
   * @param itemOrdinal - Vị trí của item trong list (1-based, có thể negative)
   *   - undefined/null: trả về tất cả identifiers từ target list
   * @param listRef - Reference đến list (optional):
   *   - undefined/null hoặc object rỗng: dùng result set cuối cùng (mới nhất)
   *   - number: list ordinal (1-based) để chọn result set cụ thể
   *   - string: context description để semantic match với description
   *   - { description?: string, source?: "model" | "user", ordinal?: number }:
   *     - description: semantic match với description (optional)
   *     - source: filter by source (model/user) (optional)
   *     - ordinal: chọn result set thứ N trong các kết quả match (1-based) (optional)
   *     Nếu cả 3 fields đều missing → dùng result set cuối cùng
   *     Ví dụ: { description: "AI conferences", ordinal: 1 } → list AI đầu tiên match
   *     Ví dụ: { source: "user" } → list user cuối cùng
   * Dùng trong Fast Path (preToolValidator) và cho conferenceRef resolution.
   */
  resolveAll(
    conversationId: string,
    itemOrdinal?: number,
    listRef?:
      | number
      | string
      | { description?: string; source?: "model" | "user"; ordinal?: number },
  ): Promise<ResolveResult>;
}
```

#### Step R7: Cập nhật ResultSetResolver để hỗ trợ source filtering + listRef object

**File:** `src/services/resultSetState/resolver.service.ts`

**Current code (chưa update):**

```typescript
listRef?: number | string,  // CHƯA có object support
```

**Cần update thành:**

```typescript
// Sửa method resolveAll:
async resolveAll(
  conversationId: string,
  itemOrdinal?: number,
  listRef?: number | string | { description?: string; source?: "model" | "user"; ordinal?: number },
): Promise<ResolveResult> {
  const states = await this.store.getAllValid(conversationId);

  if (states.length === 0) {
    return {
      resolvedIdentifiers: null,
      reasonCode: "stale",
      confidence: "low",
    };
  }

  // Xác định targetStates dựa trên listRef
  let targetStates: ResultSetState[];

  // Filter logic với thứ tự ưu tiên: source → description → ordinal
  if (typeof listRef === "number") {
    // listRef là number → chọn theo ordinal (1-based)
    const index = listRef - 1;
    if (index < 0 || index >= states.length) {
      return { resolvedIdentifiers: null, reasonCode: "out_of_range", confidence: "low" };
    }
    targetStates = [states[index]];
  } else if (typeof listRef === "string") {
    // listRef là string → semantic match với description
    const embedding = await this.generateEmbedding(listRef);
    targetStates = states.filter(state => {
      const similarity = this.cosineSimilarity(embedding, state.descriptionEmbedding);
      return similarity >= 0.7;
    });
    if (targetStates.length === 0) {
      return { resolvedIdentifiers: null, reasonCode: "no_match", confidence: "low" };
    }
  } else if (typeof listRef === "object" && listRef !== null) {
    // listRef là object → filter theo thứ tự: source → description → ordinal
    let filtered = states;

    // 1. Filter by source (nếu có)
    if (listRef.source) {
      filtered = filtered.filter(state => state.source === listRef.source);
    }

    // 2. Filter by description (nếu có)
    if (listRef.description) {
      const embedding = await this.generateEmbedding(listRef.description);
      filtered = filtered.filter(state => {
        const similarity = this.cosineSimilarity(embedding, state.descriptionEmbedding);
        return similarity >= 0.7;
      });
    }

    // 3. Filter by ordinal (nếu có)
    if (listRef.ordinal) {
      const index = listRef.ordinal - 1;
      if (index < 0 || index >= filtered.length) {
        return { resolvedIdentifiers: null, reasonCode: "out_of_range", confidence: "low" };
      }
      filtered = [filtered[index]];
    }

    // Nếu object rỗng (không có field nào) → dùng result set cuối cùng
    if (Object.keys(listRef).length === 0) {
      targetStates = [states[0]]; // states đã sort by createdAt desc
    } else {
      targetStates = filtered;
    }

    if (targetStates.length === 0) {
      return { resolvedIdentifiers: null, reasonCode: "no_match", confidence: "low" };
    }
  } else {
    // listRef undefined/null → dùng result set cuối cùng
    targetStates = [states[0]]; // states đã sort by createdAt desc
  }

  // Resolve itemOrdinal từ targetStates
  // ... (phần resolve itemOrdinal giữ nguyên)
}
```

#### Step R8: Thêm execution_mode parameter vào ALL function schemas

**File:** `src/chatbot/interface/functionHandler.interface.ts`

```typescript
export interface FunctionHandlerInput {
  args: any;
  userToken: string | null;
  language: Language;
  handlerId: string;
  socketId: string;
  conversationId: string;
  onStatusUpdate: (eventName: "status_update", data: StatusUpdate) => boolean;
  socket: Socket;
  functionName: string;
  executionContext?: any;
  agentId: AgentId;
  executionMode?: ExecutionMode; // Thêm execution_mode
}
```

**File:** `src/chatbot/handlers/saveResultSet.handler.ts`

```typescript
// Update function declaration để thêm execution_mode parameter
export const englishSaveResultSetDeclaration: FunctionDeclaration = {
  name: "saveResultSet",
  description: "...",
  parameters: {
    type: Type.OBJECT,
    properties: {
      description: { type: Type.STRING, ... },
      orderedIdentifiers: { type: Type.ARRAY, ... },
      source: { type: Type.STRING, ... },
      listName: { type: Type.STRING, ... },
      execution_mode: {
        type: Type.STRING,
        enum: ["sequential", "parallel"],
        description: "Execution mode: 'sequential' (default) to run this function in order, or 'parallel' to run it alongside other functions marked as parallel. The order of function calls in the response determines execution order.",
        nullable: true,
      },
    },
    required: ["description", "orderedIdentifiers", "source"],
  },
};
```

**File:** `src/chatbot/language/functions/english.ts`

```typescript
// CẬP NHẬT TẤT CẢ function declarations để thêm execution_mode

// Ví dụ cho các function khác:
export const englishSearchConferencesDeclaration: FunctionDeclaration = {
  name: "searchConferences",
  description: "...",
  parameters: {
    type: Type.OBJECT,
    properties: {
      query: { type: Type.STRING, ... },
      // ... các properties khác
      execution_mode: {
        type: Type.STRING,
        enum: ["sequential", "parallel"],
        description: "Execution mode: 'sequential' (default) or 'parallel'",
        nullable: true,
      },
    },
    required: ["query"],
  },
};

export const englishGetConferenceDetailsDeclaration: FunctionDeclaration = {
  name: "getConferenceDetails",
  description: "...",
  parameters: {
    type: Type.OBJECT,
    properties: {
      conferenceId: { type: Type.STRING, ... },
      // ... các properties khác
      execution_mode: {
        type: Type.STRING,
        enum: ["sequential", "parallel"],
        description: "Execution mode: 'sequential' (default) or 'parallel'",
        nullable: true,
      },
    },
    required: ["conferenceId"],
  },
};

// ... áp dụng cho tất cả các functions khác trong file
```

#### Step R9: Cập nhật subagent handler để xử lý execution_mode

**File:** `src/chatbot/gemini/functionRegistry.ts`

```typescript
// Trong executeFunction, thêm logic xử lý execution_mode
export async function executeFunction(
  functionCall: { name: string; args: any },
  callingAgentId: AgentId,
  handlerProcessId: string,
  conversationId: string,
  language: Language,
  onStatusUpdate: (eventName: "status_update", data: StatusUpdate) => boolean,
  socket: Socket,
  executionContext?: any,
  executionMode?: ExecutionMode, // Thêm parameter
): Promise<FunctionHandlerOutput> {
  // Pass executionMode vào handler context
  const context: FunctionHandlerInput = {
    args: normalizedArgs,
    userToken: userToken,
    language: language,
    handlerId: handlerProcessId,
    socketId: socketId,
    conversationId: conversationId,
    onStatusUpdate: onStatusUpdate,
    socket: socket,
    functionName: functionName,
    executionContext: executionContext,
    agentId: callingAgentId,
    executionMode: executionMode, // Pass executionMode
  };

  const resultFromHandler = await handler.execute(context);
  return resultFromHandler;
}
```

**File:** `src/chatbot/subagent/subagentHandler.ts` (hoặc file tương tự)

```typescript
// Cập nhật logic để xử lý execution_mode từ function calls
async function executeFunctionCalls(functionCalls: Array<{ name: string; args: any }>) {
  const results: FunctionHandlerOutput[] = [];
  let i = 0;

  while (i < functionCalls.length) {
    const call = functionCalls[i];
    const executionMode = call.args?.execution_mode || "sequential";

    if (executionMode === "parallel") {
      // Tìm tất cả các calls liền kề có execution_mode="parallel"
      const parallelCalls = [call];
      let j = i + 1;
      while (j < functionCalls.length && (functionCalls[j].args?.execution_mode === "parallel")) {
        parallelCalls.push(functionCalls[j]);
        j++;
      }

      // Execute parallel calls cùng lúc
      const parallelResults = await Promise.all(
        parallelCalls.map(c => executeFunction(c, ...))
      );
      results.push(...parallelResults);
      i = j;
    } else {
      // Execute sequential
      const result = await executeFunction(call, ...);
      results.push(result);
      i++;
    }
  }

  return results;
}
```

#### Step R9a: Cập nhật HostAgent handlers để hỗ trợ execution_mode

**File:** `src/chatbot/hostAgent/nonStreaming.handler.ts`

```typescript
// Cập nhật logic để xử lý execution_mode từ function calls
async function executeHostFunctionCalls(functionCalls: Array<{ name: string; args: any }>) {
  const results: FunctionHandlerOutput[] = [];
  let i = 0;

  while (i < functionCalls.length) {
    const call = functionCalls[i];
    const executionMode = call.args?.execution_mode || "sequential";

    if (executionMode === "parallel") {
      // Tìm tất cả các calls liền kề có execution_mode="parallel"
      const parallelCalls = [call];
      let j = i + 1;
      while (j < functionCalls.length && (functionCalls[j].args?.execution_mode === "parallel")) {
        parallelCalls.push(functionCalls[j]);
        j++;
      }

      // Execute parallel calls cùng lúc
      const parallelResults = await Promise.all(
        parallelCalls.map(c => executeFunction(c, ...))
      );
      results.push(...parallelResults);
      i = j;
    } else {
      // Execute sequential
      const result = await executeFunction(call, ...);
      results.push(result);
      i++;
    }
  }

  return results;
}
```

**File:** `src/chatbot/hostAgent/streaming.handler.ts`

```typescript
// Áp dụng logic tương tự như nonStreaming.handler.ts
```

**Lưu ý:** Cập nhật mọi tham chiếu đến `queryText` và `queryEmbedding` trong resolver thành `description` và `descriptionEmbedding`.

#### Step R10: Xóa auto-save trong retrieveKnowledge handler

**File:** `src/chatbot/handlers/retrieveKnowledge.handler.ts`

```typescript
// Xóa hoặc comment out code auto-save (dòng 254-268 trong plan cũ)
// HostAgent sẽ chủ động gọi saveResultSet khi cần
```

#### Step R11: Cập nhật HostAgent prompt (Tiếng Anh)

**File:** `src/chatbot/language/instructions/english.ts`

Thêm vào section routing cho ConferenceAgent:

```
### RESULT SET SAVING — CONTROLLED BY HOST AGENT

**When ConferenceAgent returns result lists:**
You (HostAgent) are responsible for calling `saveResultSet` to save these lists for later ordinal reference.

**When ConferenceAgent returns a result list:**
- Extract the conference identifiers from the result (can be IDs, titles, acronyms)
- Call `saveResultSet` yourself with the list of identifiers
- The identifiers MUST be passed in the correct order (first identifier = position 1, second identifier = position 2, etc.)
- DO NOT pass identifiers in random order
- Example: saveResultSet({ orderedIdentifiers: ["ICML", "NeurIPS", "AAAI"], source: "model" })

**When USER sends a conference list:**
If the user sends a message containing conference names/IDs (e.g., "I'm interested in ICML, NeurIPS, AAAI"):
- Extract the conference identifiers (can be IDs, titles, acronyms, or a mix)
- Call `saveResultSet` yourself with `source: "user"` to save this list
- This ensures user-sent lists are trackable for later ordinal references
- Example: saveResultSet({ orderedIdentifiers: ["ICML", "NeurIPS", "AAAI"], source: "user" })

**Important Rules:**
- Always preserve the exact order of identifiers passed to saveResultSet
- Identifiers can be any string type: ID, title, acronym, or a mix
- When saving user lists, ALWAYS use `source: "user"`
- When saving model-generated lists, use `source: "model"`
```

#### Step R12: Cập nhật HostAgent prompt (Tiếng Việt)

**File:** `src/chatbot/language/instructions/vietnamese.ts`

Thêm vào section routing cho ConferenceAgent:

```
### LƯU RESULT SET — ĐƯỢC CONTROL BỞI HOST AGENT

**Khi ConferenceAgent trả về result lists:**
Bạn (HostAgent) chịu trách nhiệm gọi `saveResultSet` để lưu các lists này cho tham chiếu ordinal sau này.

**Khi ConferenceAgent trả về result list:**
- Trích xuất conference identifiers từ kết quả (có thể là ID, title, acronym, hoặc kết hợp)
- Tự gọi `saveResultSet` với list identifiers
- Các identifiers PHẢI được truyền theo đúng thứ tự (identifier đầu tiên = vị trí 1, identifier thứ hai = vị trí 2, v.v.)
- KHÔNG ĐƯỢC truyền identifiers theo cách ngẫu nhiên
- Ví dụ: saveResultSet({ orderedIdentifiers: ["ICML", "NeurIPS", "AAAI"], source: "model" })

**Khi USER gửi list hội nghị:**
Nếu user gửi message chứa tên/ID hội nghị (ví dụ: "Tôi quan tâm ICML, NeurIPS, AAAI"):
- Trích xuất conference identifiers (có thể là ID, title, acronym, hoặc kết hợp)
- Tự gọi `saveResultSet` với `source: "user"` để lưu list này
- Điều này đảm bảo user-sent lists có thể track cho các tham chiếu ordinal sau này
- Ví dụ: saveResultSet({ orderedIdentifiers: ["ICML", "NeurIPS", "AAAI"], source: "user" })

**Quy tắc quan trọng:**
- Luôn bảo toàn đúng thứ tự của identifiers được truyền cho saveResultSet
- Identifiers có thể là bất kỳ string type nào: ID, title, acronym, hoặc kết hợp
- Khi lưu user lists, LUÔN LUÔN dùng `source: "user"`
- Khi lưu model-generated lists, dùng `source: "model"`
```

Thêm instruction mới vào section context window priority:

```

### OPTIMIZED LIST SEARCH — SEARCH IN CONTEXT WINDOW FIRST

When the user refers to a conference list or specific conference (e.g., "the list from earlier", "the AI conferences you showed me", "that list from the last turn", "ICML from my list"), search for that list/conference in the context window FIRST, with the following priority:

**Search Scope (in order):**

1. Messages with role="model" - focus on the LAST text part within each message
2. Messages with role="user" - user may have explicitly listed conferences in their message

**Why this optimization?**

- Conference lists are typically displayed in model responses (last text part)
- Users may also explicitly list conferences in their messages (source="user" lists)
- This reduces search scope and improves accuracy
- Only fallback to ResultSetState if context window search fails

**Example:**

```

History:
[
{ role: "user", parts: [{ text: "I'm interested in ICML, NeurIPS, AAAI" }] },
{ role: "model", parts: [{ functionCall: {...} }, { text: "Here are 10 AI conferences: [list content]" }] },
{ role: "function", parts: [{ functionResponse: {...} }] },
{ role: "model", parts: [{ text: "Based on the results..." }] }
]

User: "Show me the 2nd conference from that list"
→ Search in role="model" messages (last text part)
→ Found list in the second model message
→ Use ordinal reference on that list

User: "Tell me more about NeurIPS from my list"
→ Search in role="user" messages
→ Found user's explicit list: "ICML, NeurIPS, AAAI"
→ Use NeurIPS from that list

```

**Important:**

- Search in BOTH role="model" (last text part) AND role="user" messages
- Do NOT search in role="function" messages for conference lists
- Focus on the LAST text part for model messages
- Nếu không tìm thấy list trong context window, fallback sang dùng conferenceRef từ ResultSetState với listRef phù hợp (có thể specify source filter: { description: "...", source: "model" | "user" })

```

#### Step R13: Xóa ConferenceAgent saveResultSet instruction (Tiếng Anh)

**File:** `src/chatbot/language/instructions/english.ts`

Xóa hoặc comment out instruction về SAVE RESULT SET vì ConferenceAgent không được phép save result sets.

```typescript
// Xóa section "### SAVE RESULT SET — CONTROLLED BY HOST AGENT"
// ConferenceAgent không cần instruction này vì chỉ HostAgent được phép save result sets
```

#### Step R14: Xóa ConferenceAgent saveResultSet instruction (Tiếng Việt)

**File:** `src/chatbot/language/instructions/vietnamese.ts`

Xóa hoặc comment out instruction về LƯU RESULT SET vì ConferenceAgent không được phép save result sets.

```typescript
// Xóa section "### LƯU RESULT SET — ĐƯỢC CONTROL BỞI HOST AGENT"
// ConferenceAgent không cần instruction này vì chỉ HostAgent được phép save result sets
```

#### Step R15: Cập nhật HostAgent prompt — Optimized list search pattern (Tiếng Anh)

**File:** `src/chatbot/language/instructions/english.ts`

Thêm instruction mới vào section context window priority:

```
### OPTIMIZED LIST SEARCH — SEARCH IN CONTEXT WINDOW FIRST

When the user refers to a conference list or specific conference (e.g., "the list from earlier", "the AI conferences you showed me", "that list from the last turn", "ICML from my list"), search for that list/conference in the context window FIRST, with the following priority:

**Search Scope (in order):**
1. Messages with role="model" - focus on the LAST text part within each message
2. Messages with role="user" - user may have explicitly listed conferences in their message

**Why this optimization?**
- Conference lists are typically displayed in model responses (last text part)
- Users may also explicitly list conferences in their messages (source="user" lists)
- This reduces search scope and improves accuracy
- Only fallback to ResultSetState if context window search fails

**Example:**
```

History:
[
{ role: "user", parts: [{ text: "I'm interested in ICML, NeurIPS, AAAI" }] },
{ role: "model", parts: [{ functionCall: {...} }, { text: "Here are 10 AI conferences: [list content]" }] },
{ role: "function", parts: [{ functionResponse: {...} }] },
{ role: "model", parts: [{ text: "Based on the results..." }] }
]

User: "Show me the 2nd conference from that list"
→ Search in role="model" messages (last text part)
→ Found list in the second model message
→ Use ordinal reference on that list

User: "Tell me more about NeurIPS from my list"
→ Search in role="user" messages
→ Found user's explicit list: "ICML, NeurIPS, AAAI"
→ Use NeurIPS from that list

```

**Important:**
- Search in BOTH role="model" (last text part) AND role="user" messages
- Do NOT search in role="function" messages for conference lists
- Focus on the LAST text part for model messages
- If not found in context window, fallback to instructing the subagent to use the `conferenceRef` parameter when calling functions to refer to a conference/conference list (can specify source filter: { description: "...", source: "model" | "user" })
```

#### Step R16: Cập nhật HostAgent prompt — Optimized list search pattern (Tiếng Việt)

**File:** `src/chatbot/language/instructions/vietnamese.ts`

Thêm instruction mới vào section context window priority:

```
### TỐI ƯU HÓA TÌM KIẾM LIST — TÌM TRONG CONTEXT WINDOW TRƯỚC

Khi người dùng nhắc đến một list hội nghị hoặc hội nghị cụ thể (ví dụ: "list từ trước", "các hội nghị AI bạn đã show tôi", "list đó từ turn cuối", "NeurIPS từ list của tôi"), hãy tìm kiếm list/hội nghị đó trong context window TRƯỚC, với thứ tự ưu tiên sau:

**Phạm vi tìm kiếm (theo thứ tự):**
1. Messages với role="model" - tập trung vào phần text CUỐI CÙNG trong mỗi message
2. Messages với role="user" - user có thể liệt kê hội nghị trong message của họ

**Tại sao tối ưu hóa này?**
- List hội nghị thường hiển thị trong model responses (phần text cuối)
- User cũng có thể liệt kê hội nghị trong messages của họ (source="user" lists)
- Điều này giảm phạm vi tìm kiếm và tăng độ chính xác
- Chỉ fallback sang ResultSetState nếu tìm trong context window thất bại

**Ví dụ:**
```

History:
[
{ role: "user", parts: [{ text: "Tôi quan tâm ICML, NeurIPS, AAAI" }] },
{ role: "model", parts: [{ functionCall: {...} }, { text: "Đây là 10 hội nghị AI: [nội dung list]" }] },
{ role: "function", parts: [{ functionResponse: {...} }] },
{ role: "model", parts: [{ text: "Dựa trên kết quả..." }] }
]

User: "Cho tôi xem hội nghị thứ 2 từ list đó"
→ Tìm trong role="model" messages (phần text cuối)
→ Tìm thấy list trong message thứ hai của model
→ Dùng tham chiếu ordinal trên list đó

User: "Cho tôi biết thêm về NeurIPS từ list của tôi"
→ Tìm trong role="user" messages
→ Tìm thấy list rõ ràng của user: "ICML, NeurIPS, AAAI"
→ Dùng NeurIPS từ list đó

```

**Quan trọng:**
- Tìm trong CẢ role="model" (phần text cuối) VÀ role="user" messages
- KHÔNG tìm trong role="function" messages cho list hội nghị
- Tập trung vào phần text CUỐI CÙNG cho model messages
- Nếu không tìm thấy trong context window, fallback sang việc hướng dẫn subagent sử dụng tham số `conferenceRef` khi gọi hàm để tham chiếu đến một hội nghị/danh sách hội nghị (có thể chỉ định source filter: { description: "...", source: "model" | "user" })
```

### 11.4 Test Plan cho Refactor |

| 8 | HostAgent detect user list và lưu với source="user" | Success, source="user" |
| 9 | ResultSetState có field description thay vì queryText | Field name đã đổi thành description |
| 10 | ResultSetState có field source | Field source tồn tại với giá trị "model" hoặc "user" |
| 11 | HostAgent lưu user list → user reference "cái thứ 2 trong list tôi gửi" → resolve thành công | Resolve thành công, trả về đúng conference ID |

### 11.5 File structure updates

```

src/ # Easyconf-Chatbot-Server (Backend)
types/
resultSetState.types.ts # [SỬA] đổi tên queryText→description, queryEmbedding→descriptionEmbedding, thêm source (Step R0)
chatbot/
handlers/
saveResultSet.handler.ts # [MỚI] (Step R1)
retrieveKnowledge.handler.ts # [SỬA] xóa auto-save (Step R7)
gemini/
functionRegistry.ts # [SỬA] thêm saveResultSet, cho phép cả ConferenceAgent và HostAgent (Step R2)
language/
functions/
english.ts # [SỬA] thêm declaration với source parameter (Step R3)
vietnamese.ts # [SỬA] thêm declaration với source parameter (Step R3)
instructions/
english.ts # [SỬA] thêm result set saving instruction + user list detection (Step R8, R10)
vietnamese.ts # [SỬA] thêm result set saving instruction + user list detection (Step R9, R11)
utils/
languageConfig.ts # [SỬA] thêm saveResultSet declaration cho cả ConferenceAgent và HostAgent (Step R4)
services/
resultSetState/
store.service.ts # [SỬA] hỗ trợ named result sets + new fields (description, descriptionEmbedding, source) (Step R5)
resolver.service.ts # [SỬA] hỗ trợ named result sets + new field names (Step R6)

```

```

```

```

```
