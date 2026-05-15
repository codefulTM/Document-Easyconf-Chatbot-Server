# R1-02: Deterministic Resolver Core — Implementation Plan

## 1. Tổng quan

**Mã work package:** R1-02
**Ưu tiên:** Số 2
**Scope:** In-scope (bắt buộc)
**Kế thừa từ:** P1-01 (ResultSetState V1 — đã hoàn thành)
**Liên quan tới:** P2-04 (3-Layer Memory), mutation tools (manageFollow, manageCalendar, v.v.), preToolValidator

### Mục tiêu

Loại bỏ suy đoán của LLM cho **3 loại tham chiếu** chính:

| Loại | Ví dụ | Resolver chịu trách nhiệm |
|------|-------|--------------------------|
| Ordinal | "thứ 2", "cuối", "vừa rồi", "top 3" | OrdinalResolver + NL Parser |
| Temporal | "upcoming", "cfp đang mở", "deadline gần nhất" | TemporalResolver |
| Ranking | "hội nghị rank A", "sắp xếp theo ngày" | RankingResolver |

Cả 3 resolver đều có output là **danh sách conference ID có thứ tự** — deterministic, không cần LLM suy luận.

---

### Phân biệt với P1-01

P1-01 đã làm:
- ResultSetStateStore: lưu/đọc result set trên MongoDB
- ResultSetResolver.resolveAll: resolve ordinal (số 1-based) trên các result set đã lưu
- ResultSetResolver.resolveByContext: resolve ordinal + semantic match contextHint
- Fast Path trong preToolValidator: ordinal → ID (1 turn)
- Slow Path: tool resolveConferenceRef + handler

R1-02 mở rộng:
- **Ordinal NL Parser**: LLM không cần tự convert "thứ 2" → 2, "cuối" → -1 nữa — parser tự làm
- **"vừa rồi"** resolver: resolve về result set gần nhất, lấy ID cuối cùng
- **"top N"** resolver: cắt danh sách xuống N items
- **TemporalResolver**: hoàn toàn mới, filter conference theo thời gian thực (dùng DB)
- **RankingResolver**: hoàn toàn mới, sort/select conference theo policy
- **Integration hooks**: kết nối tất cả resolver vào preToolValidator và mutation tools

---

## 2. Kiến trúc tổng thể

```
┌─────────────────────────────────────────────────────────────────────┐
│                      LLM / Agent                                     │
│  "theo dõi hội nghị thứ 2"   "list upcoming AI confs"               │
└──────────┬──────────────────────────────────────────┬────────────────┘
           │                                          │
           ▼                                          ▼
┌──────────────────────┐              ┌──────────────────────────────┐
│  OrdinalResolver     │              │  TemporalResolver            │
│                      │              │                              │
│  parse("thứ 2") → 2  │              │  resolveUpcoming()           │
│  parse("cuối") → -1  │              │  resolveCfpOpen()            │
│  parse("vừa rồi")    │              │  resolveNearestDeadline()    │
│    → lastResultSet   │              │                              │
│  parse("top 3")      │              │  Kết quả: danh sách ID       │
│    → limit(3)        │              │  (không cần result set state)│
│                      │              └──────────────────────────────┘
│  Dùng ResultSetState │
│  để resolve số→ID    │              ┌──────────────────────────────┐
│                      │              │  RankingResolver             │
│  Kết quả: ID         │              │                              │
└──────────────────────┘              │  sortByRank(policy)          │
                                      │  sortByDate(order)           │
                                      │  sortByTitle(order)          │
                                      │                              │
                                      │  Kết quả: danh sách ID       │
                                      │  (sắp xếp lại thứ tự)        │
                                      └──────────────────────────────┘
```

### Luồng xử lý

```
User: "theo dõi hội nghị thứ 2"

[preToolValidator] nhận manageFollow(identifier="thứ 2", identifierType="ordinal_nl")
  → OrdinalResolver.parse("thứ 2") → { type: "ordinal", value: 2 }
  → resolveAll(convId, 2)
    → uniqueMatch → replace identifier=ID → pass ✓

---

User: "list upcoming AI conferences"

[HostAgent] detect "upcoming" keyword
  → route to ConferenceAgent
[ConferenceAgent] gọi TemporalResolver.resolveUpcoming(query, filter)
  → trả về danh sách conference ID + metadata
  → lưu vào ResultSetState (cho ordinal reference sau này)
  → trả compact list về LLM
```

### Resolve Strategy

| Kịch bản | Resolver | Input | Output |
|----------|----------|-------|--------|
| "thứ 2" | OrdinalResolver.parse + resolveAll | identifier string | conference ID |
| "cuối" | OrdinalResolver.parse + resolveAll | identifier string | conference ID |
| "vừa rồi" | OrdinalResolver.resolveLastResult | conversationId | conference ID |
| "top 3" | OrdinalResolver.resolveTop | conversationId + N | N conference IDs |
| "upcoming" | TemporalResolver.resolveUpcoming | query + filter | [conference IDs] |
| "cfp đang mở" | TemporalResolver.resolveCfpOpen | query + filter | [conference IDs] |
| "deadline gần nhất" | TemporalResolver.resolveNearestDeadline | query + filter | [conference IDs] |
| "rank A" | RankingResolver.sortByRank | list + policy | sorted [conference IDs] |
| "mới nhất" | RankingResolver.sortByDate | list + order | sorted [conference IDs] |

---

## 3. Data Model

### 3.1 Thêm temporal metadata vào ResultSetState (tuỳ chọn — có thể để ở memory layer)

Không cần thay đổi schema MongoDB. TemporalResolver query trực tiếp từ DB (ConferenceSyncService).

### 3.2 Error codes mới

```typescript
// Thêm vào OrchestrationSafetyErrorCode
TEMPORAL_NO_RESULTS: "temporal_no_results",
RANKING_POLICY_UNKNOWN: "ranking_policy_unknown",
ORDINAL_PARSE_FAILED: "ordinal_parse_failed",
```

### 3.3 Type definitions mới

```typescript
// ---- OrdinalResolver ----

/** Kết quả parse NL ordinal */
type OrdinalParseResult =
  | { type: "ordinal"; value: number }           // "thứ 2" → 2, "cuối" → -1
  | { type: "last_result" }                       // "vừa rồi"
  | { type: "top_n"; value: number }              // "top 3" → 3
  | { type: "unresolved" };                       // không parse được

interface IOrdinalResolver {
  /** Parse NL string thành ordinal reference */
  parse(input: string): OrdinalParseResult;

  /** Resolve "vừa rồi" → lấy ID cuối cùng từ result set gần nhất */
  resolveLastResult(conversationId: string): Promise<ResolveResult>;

  /** Resolve "top N" → lấy N ID đầu từ result set gần nhất */
  resolveTop(
    conversationId: string,
    n: number,
  ): Promise<{ resolvedIds: string[]; reasonCode: string }>;
}

// ---- TemporalResolver ----

interface ITemporalResolver {
  /** Tìm conference sắp diễn ra (conferenceDate > now) */
  resolveUpcoming(
    query: string,
    filter?: TemporalFilter,
  ): Promise<TemporalResolveResult>;

  /** Tìm conference đang mở CFP (submission deadline > now) */
  resolveCfpOpen(
    query: string,
    filter?: TemporalFilter,
  ): Promise<TemporalResolveResult>;

  /** Tìm conference có deadline gửi bài gần nhất (> now) */
  resolveNearestDeadline(
    query: string,
    filter?: TemporalFilter,
  ): Promise<TemporalResolveResult>;
}

type TemporalFilter = {
  tableName?: string;
  startDate?: string;
  endDate?: string;
  [key: string]: any;
};

type TemporalResolveResult = {
  resolvedIds: string[];
  totalCount: number;
  items: Array<{
    id: string;
    title: string;
    acronym: string;
    date?: string;
    deadline?: string;
  }>;
  reasonCode: "resolved" | "no_results" | "error";
};

// ---- RankingResolver ----

type RankingPolicy =
  | { type: "rank"; order: "asc" | "desc"; field?: string }  // sort by CORE rank
  | { type: "date"; order: "asc" | "desc" }                   // sort by conference date
  | { type: "title"; order: "asc" | "desc" };                 // sort alphabetically

type RankingResolveResult = {
  resolvedIds: string[];
  totalCount: number;
  reasonCode: "resolved" | "policy_unknown" | "error";
};

interface IRankingResolver {
  /** Sắp xếp danh sách conference theo policy */
  sort(
    conferenceIds: string[],
    policy: RankingPolicy,
  ): Promise<RankingResolveResult>;
}
```

---

## 4. API / Interface

### 4.1 OrdinalResolver

```typescript
interface IOrdinalResolver {
  parse(raw: string): OrdinalParseResult;

  resolveAll(
    conversationId: string,
    ordinal: number,
  ): Promise<ResolveAllResult>;  // đã có từ P1-01

  resolveByContext(
    conversationId: string,
    context: string | null,
    ordinal: number,
  ): Promise<ResolveResult>;      // đã có từ P1-01

  resolveLastResult(
    conversationId: string,
  ): Promise<ResolveResult>;

  resolveTop(
    conversationId: string,
    n: number,
  ): Promise<ResolveResult & { resolvedIds?: string[] }>;
}
```

### 4.2 TemporalResolver

```typescript
interface ITemporalResolver {
  resolveUpcoming(
    query: string,
    filter?: TemporalFilter,
  ): Promise<TemporalResolveResult>;

  resolveCfpOpen(
    query: string,
    filter?: TemporalFilter,
  ): Promise<TemporalResolveResult>;

  resolveNearestDeadline(
    query: string,
    filter?: TemporalFilter,
  ): Promise<TemporalResolveResult>;
}
```

TemporalResolver dùng `ConferenceSyncService` — có sẵn method:
- `getConferenceById(id)` → ConferenceRecord chứa organizations → dates

Hoặc có thể thêm method mới:

```typescript
// Thêm vào ConferenceSyncService:
findConferencesByDateRange(fromDate: string, toDate: string): Promise<ConferenceRecord[]>;
findConferencesByOpenCfp(): Promise<ConferenceRecord[]>;
```

### 4.3 RankingResolver

```typescript
interface IRankingResolver {
  /**
   * Sắp xếp danh sách conference IDs theo policy.
   * Dùng ConferenceSyncService để lấy full record cho từng ID,
   * sort theo policy, trả về thứ tự mới.
   */
  sort(
    conferenceIds: string[],
    policy: RankingPolicy,
  ): Promise<RankingResolveResult>;
}
```

---

## 5. Chi tiết các Resolver

### 5.1 OrdinalResolver — NL Parser

**File mới:** `src/services/resultSetState/ordinalParser.ts`

Parser chuyển NL thành ordinal reference, loại bỏ hoàn toàn việc LLM phải suy luận:

```typescript
class OrdinalParser {
  parse(input: string): OrdinalParseResult {
    const trimmed = input.trim().toLowerCase();

    // "vừa rồi" patterns
    if (this.matchLastResult(trimmed)) {
      return { type: "last_result" };
    }

    // "top N" patterns
    const topMatch = this.matchTopN(trimmed);
    if (topMatch) {
      return { type: "top_n", value: topMatch };
    }

    // Ordinal number patterns: "thứ 2", "số 2", "cái thứ 3"
    const ordinalMatch = this.matchOrdinalNumber(trimmed);
    if (ordinalMatch !== null) {
      return { type: "ordinal", value: ordinalMatch };
    }

    // Negative ordinal: "cuối", "áp cuối", "cuối cùng"
    const negativeMatch = this.matchNegativeOrdinal(trimmed);
    if (negativeMatch !== null) {
      return { type: "ordinal", value: negativeMatch };
    }

    return { type: "unresolved" };
  }
}
```

**Pattern matching tables:**

| Input pattern | Output |
|---------------|--------|
| "vừa rồi", "vừa mới", "just now", "just retrieved", "last result" | `{ type: "last_result" }` |
| "top 3", "top 5", "3 cái đầu", "3 items đầu" | `{ type: "top_n", value: N }` |
| "thứ 2", "thứ hai", "số 2", "cái thứ 2", "thứ 3" | `{ type: "ordinal", value: 2 }` |
| "đầu tiên", "thứ nhất", "cái đầu", "first", "the first" | `{ type: "ordinal", value: 1 }` |
| "cuối", "cuối cùng", "cái cuối", "last", "the last" | `{ type: "ordinal", value: -1 }` |
| "áp cuối", "kế cuối", "second to last" | `{ type: "ordinal", value: -2 }` |

**Ưu điểm:** LLM không cần convert "thứ 2" → 2 nữa. LLM chỉ cần truyền nguyên văn `identifier="thứ 2"`, `identifierType="ordinal_nl"`. Backend tự parse.

**Lưu ý:** `identifierType` hiện tại là `"ordinal"` (LLM tự convert). Ta thêm `"ordinal_nl"` (backend tự parse). Giữ cả 2 để backward compatible.

### 5.2 OrdinalResolver — resolveLastResult / resolveTop

```typescript
// Thêm vào ResultSetResolver (resolver.service.ts)

async resolveLastResult(conversationId: string): Promise<ResolveResult> {
  const states = await this.store.getAllValid(conversationId);
  if (states.length === 0) {
    return { resolvedId: null, reasonCode: "stale", confidence: "low" };
  }

  // Lấy result set gần đây nhất (getAllValid trả về mới nhất đầu)
  const latest = states[0];
  const lastIndex = latest.orderedConferenceIds.length - 1;

  if (lastIndex < 0) {
    return { resolvedId: null, reasonCode: "out_of_range", confidence: "low" };
  }

  return {
    resolvedId: latest.orderedConferenceIds[lastIndex],
    reasonCode: "resolved",
    confidence: "high",
  };
}

async resolveTop(
  conversationId: string,
  n: number,
): Promise<{ resolvedId: string | null; resolvedIds: string[]; reasonCode: string; confidence: "high" | "low" }> {
  const states = await this.store.getAllValid(conversationId);
  if (states.length === 0) {
    return { resolvedId: null, resolvedIds: [], reasonCode: "stale", confidence: "low" };
  }

  const latest = states[0];
  const ids = latest.orderedConferenceIds.slice(0, n);

  if (ids.length === 0) {
    return { resolvedId: null, resolvedIds: [], reasonCode: "out_of_range", confidence: "low" };
  }

  return {
    resolvedId: ids[0],
    resolvedIds: ids,
    reasonCode: "resolved",
    confidence: "high",
  };
}
```

### 5.3 TemporalResolver

**File mới:** `src/services/resultSetState/temporalResolver.service.ts`

TemporalResolver query trực tiếp từ PostgreSQL (qua ConferenceSyncService hoặc PgClient) — không dùng ResultSetState vì temporal là filter động theo thời gian thực.

```typescript
class TemporalResolver implements ITemporalResolver {
  constructor(
    private conferenceSyncService: ConferenceSyncService,
    private pgClient: PgClient,
  ) {}

  async resolveUpcoming(
    query: string,
    filter?: TemporalFilter,
  ): Promise<TemporalResolveResult> {
    // Query conferences WHERE organization.dates.type='conferenceDates'
    // AND toDate > NOW()
    // Sắp xếp theo fromDate ASC
    // Giới hạn 20 kết quả
    // Trả về danh sách ID + metadata compact
    //
    // SQL gợi ý:
    // SELECT c.id, c.title, c.acronym
    // FROM conferences c
    // JOIN organization o ON o.conferenceId = c.id
    // JOIN conference_date cd ON cd.organizationId = o.id
    // WHERE cd.type = 'conferenceDates'
    //   AND cd.toDate > NOW()
    //   AND (c.title ILIKE '%query%' OR c.acronym ILIKE '%query%')
    // ORDER BY cd.fromDate ASC
    // LIMIT 20
  }

  async resolveCfpOpen(
    query: string,
    filter?: TemporalFilter,
  ): Promise<TemporalResolveResult> {
    // Query conferences WHERE organization.dates.type='submissionDate'
    // AND toDate > NOW()
    // Sắp xếp theo toDate ASC (deadline gần nhất trước)
    // Giới hạn 20 kết quả
  }

  async resolveNearestDeadline(
    query: string,
    filter?: TemporalFilter,
  ): Promise<TemporalResolveResult> {
    // Query conferences WHERE organization.dates.type='submissionDate'
    // AND toDate > NOW()
    // Sắp xếp theo toDate ASC
    // LIMIT 1 (hoặc N theo filter)
  }
}
```

### 5.4 RankingResolver

**File mới:** `src/services/resultSetState/rankingResolver.service.ts`

```typescript
class RankingResolver implements IRankingResolver {
  constructor(private conferenceSyncService: ConferenceSyncService) {}

  async sort(
    conferenceIds: string[],
    policy: RankingPolicy,
  ): Promise<RankingResolveResult> {
    // Lấy full record cho từng ID
    const records = await Promise.all(
      conferenceIds.map(id => this.conferenceSyncService.getConferenceById(id)),
    );
    const validRecords = records.filter(Boolean) as ConferenceRecord[];

    // Sort theo policy
    switch (policy.type) {
      case "rank":
        // Sort theo CORE rank value (A > B > C)
        // Dùng rank table trong conference record
        // Cần mapping: "A" → 1, "B" → 2, "C" → 3, ...
        break;
      case "date":
        // Sort theo fromDate của organization gần đây nhất
        break;
      case "title":
        // Sort alphabetically
        break;
    }

    return {
      resolvedIds: validRecords.map(r => String(r.id)),
      totalCount: validRecords.length,
      reasonCode: "resolved",
    };
  }
}
```

---

## 6. Integration Points

### 6.1 Cập nhật identifierType — thêm "ordinal_nl"

**File:** `src/chatbot/language/functions/english.ts`

Thêm `"ordinal_nl"` vào enum của tất cả mutation tools (bên cạnh `"ordinal"` hiện tại):

```typescript
enum: ["acronym", "title", "id", "ordinal", "ordinal_nl"],
```

**Phân biệt:**
- `"ordinal"`: LLM tự convert "thứ 2" → "2", "cuối" → "-1"
- `"ordinal_nl"`: LLM truyền nguyên văn "thứ 2", backend tự parse

### 6.2 Cập nhật preToolValidator — xử lý "ordinal_nl"

**File:** `src/chatbot/guards/preToolValidator.ts`

Trong khối xử lý ordinal, thêm nhánh `ordinal_nl`:

```typescript
if (identifierType.trim() === "ordinal_nl") {
  // Dùng OrdinalParser.parse() thay vì parseInt()
  const parseResult = ordinalParser.parse(identifier as string);

  switch (parseResult.type) {
    case "ordinal":
      // resolveAll với số → Fast Path như cũ
      break;
    case "last_result":
      // resolveLastResult → lấy ID từ result set gần nhất
      break;
    case "top_n":
      // resolveTop → lấy N ID đầu
      break;
    case "unresolved":
      // fallback về ambiguity check
      break;
  }
}
```

### 6.3 TemporalResolver Integration — tool mới hoặc tự động trong retrieveKnowledge

**Cách 1 (khuyên dùng):** Mở rộng `retrieveKnowledge` handler — thêm param `temporalMode` (cùng với `listMode`):

```typescript
args: {
  query: "AI conferences",
  filter: { tableName: "conferences" },
  listMode: true,
  temporalMode: "upcoming" | "cfp_open" | "nearest_deadline" | null,
}
```

Khi `temporalMode !== null`: handler gọi TemporalResolver thay vì retrievalService.

**Cách 2:** Tool riêng `resolveTemporalConference`. Đơn giản nhưng thêm 1 tool LLM phải học.

Khuyên dùng Cách 1 vì tái sử dụng flow retrieveKnowledge hiện tại.

### 6.4 RankingResolver Integration

RankingResolver là **post-processing step** — nó sắp xếp lại kết quả đã có.

**Cách 1:** Thêm param `rankBy` vào `retrieveKnowledge`:

```typescript
args: {
  query: "AI conferences",
  filter: { tableName: "conferences" },
  listMode: true,
  rankBy: "rank:desc" | "date:asc" | "title:asc" | null,
}
```

Handler: sau khi có result list, gọi RankingResolver.sort() trước khi trả về.

**Cách 2:** Integration vào mutating flow — khi LLM gọi mutation với `sortBy` param.

### 6.5 Lưu ResultSetState sau Temporal/Ranking resolve

Khi TemporalResolver hoặc RankingResolver trả về danh sách, cần lưu vào ResultSetState để ordinal reference sau này có thể dùng:

```typescript
// Trong retrieveKnowledge handler, sau khi temporal/ranking resolve:
if (temporalMode || rankBy) {
  const ids = resolvedItems.map(item => item.id);
  await this.resultSetStateStore.save(conversationId, query, embedding, ids);
}
```

---

## 7. System Prompt cập nhật

### 7.1 ConferenceAgent — hướng dẫn temporal/ranking/ordinal_nl

**File:** `src/chatbot/language/instructions/english.ts` (và các ngôn ngữ khác)

Thêm section mới trong ConferenceAgent instructions:

```
*   **Temporal References (Upcoming, CFP, Deadlines):**
    *   When the user asks for upcoming conferences (e.g., "upcoming AI conferences",
        "hội nghị sắp tới"), use 'retrieveKnowledge' with \`temporalMode: "upcoming"\`.
    *   When the user asks for conferences with open CFP (e.g., "cfp đang mở",
        "conferences accepting papers"), use \`temporalMode: "cfp_open"\`.
    *   When the user asks for nearest deadline (e.g., "deadline gần nhất",
        "submission soon"), use \`temporalMode: "nearest_deadline"\`.

*   **Ranking / Sorting References:**
    *   When the user asks to sort by rank (e.g., "rank A", "CORE rank", "sorted by rank"),
        use \`rankBy: "rank:desc"\`.
    *   When the user asks to sort by date (e.g., "newest first", "mới nhất"),
        use \`rankBy: "date:desc"\`.
    *   When the user asks to sort by title (e.g., "alphabetically", "A-Z"),
        use \`rankBy: "title:asc"\`.

*   **Ordinal References (Slow Path - NL mode):**
    *   When the user refers to a conference by position (e.g., "the 2nd one",
        "thứ 2", "cuối", "vừa rồi", "top 3"):
    *   Call the mutation tool with \`identifierType="ordinal_nl"\` and
        \`identifier\` as the **exact phrase the user used** (e.g., "thứ 2",
        "cuối", "vừa rồi", "top 3"). Do NOT convert to a number.
    *   The system will parse and resolve automatically.
    *   If blocked with \`ambiguity_blocked_mutation\`, use \`resolveConferenceRef\`.
```

### 7.2 HostAgent — routing temporal/ranking

HostAgent không cần học temporal cụ thể. HostAgent chỉ cần route:
```
- "hội nghị sắp tới" → ConferenceAgent (temporal)
- "rank A" → ConferenceAgent (ranking)
```

---

## 8. Error Handling

### 8.1 Error codes mới

| Error code | Khi nào | HTTP analogy |
|------------|---------|--------------|
| `temporal_no_results` | TemporalResolver không tìm thấy kết quả | 404 |
| `ranking_policy_unknown` | Policy không hợp lệ (e.g., "rank:invalid") | 400 |
| `ordinal_parse_failed` | OrdinalParser không parse được input | 400 |

### 8.2 Error handling trong từng resolver

**OrdinalParser:**
- Input rỗng → `{ type: "unresolved" }`
- "top" không có số → `{ type: "unresolved" }`
- "thứ" không có số → `{ type: "unresolved" }`

**TemporalResolver:**
- Query rỗng → vẫn query được (tất cả upcoming)
- DB query timeout → catch error → `reasonCode: "error"`
- Không có kết quả → `reasonCode: "no_results"`, items rỗng

**RankingResolver:**
- conferenceIds rỗng → `reasonCode: "error"`
- Policy unknown → `reasonCode: "policy_unknown"`
- Conference record không tìm thấy → bỏ qua ID đó, vẫn sort phần còn lại

---

## 9. Unit Test Plan

### 9.1 OrdinalParser

| # | Test case | Input | Expected |
|---|-----------|-------|----------|
| 1 | "thứ 2" | `"thứ 2"` | `{ type: "ordinal", value: 2 }` |
| 2 | "thứ hai" | `"thứ hai"` | `{ type: "ordinal", value: 2 }` |
| 3 | "số 3" | `"số 3"` | `{ type: "ordinal", value: 3 }` |
| 4 | "first" | `"first"` | `{ type: "ordinal", value: 1 }` |
| 5 | "the first" | `"the first one"` | `{ type: "ordinal", value: 1 }` |
| 6 | "cuối" | `"cuối"` | `{ type: "ordinal", value: -1 }` |
| 7 | "áp cuối" | `"áp cuối"` | `{ type: "ordinal", value: -2 }` |
| 8 | "last" | `"last"` | `{ type: "ordinal", value: -1 }` |
| 9 | "vừa rồi" | `"vừa rồi"` | `{ type: "last_result" }` |
| 10 | "just now" | `"just now"` | `{ type: "last_result" }` |
| 11 | "top 3" | `"top 3"` | `{ type: "top_n", value: 3 }` |
| 12 | "5 cái đầu" | `"5 cái đầu"` | `{ type: "top_n", value: 5 }` |
| 13 | "3 items đầu" | `"3 items đầu"` | `{ type: "top_n", value: 3 }` |
| 14 | Input rỗng | `""` | `{ type: "unresolved" }` |
| 15 | Input không liên quan | `"ABC"` | `{ type: "unresolved" }` |
| 16 | "đầu tiên" | `"đầu tiên"` | `{ type: "ordinal", value: 1 }` |
| 17 | "cái đầu" | `"cái đầu"` | `{ type: "ordinal", value: 1 }` |
| 18 | "second to last" | `"second to last"` | `{ type: "ordinal", value: -2 }` |
| 19 | "thứ 1" | `"thứ 1"` | `{ type: "ordinal", value: 1 }` |
| 20 | "top N" (không số) | `"top"` | `{ type: "unresolved" }` |

### 9.2 OrdinalResolver (resolveLastResult)

| # | Test case | Expected |
|---|-----------|----------|
| 21 | 1 result set, 3 items → lấy cuối | resolvedId = item thứ 3 |
| 22 | 1 result set, 0 items | out_of_range |
| 23 | 0 result set | stale |
| 24 | 2 result set, lấy từ mới nhất | resolvedId từ latest state |

### 9.3 OrdinalResolver (resolveTop)

| # | Test case | Expected |
|---|-----------|----------|
| 25 | 1 result set, top 2 → 2 items | resolvedIds.length = 2 |
| 26 | 1 result set, top 10 > items | resolvedIds.length = items.length |
| 27 | 0 result set | resolvedIds = [] |
| 28 | top 0 | resolvedIds = [] |

### 9.4 TemporalResolver

| # | Test case | Expected |
|---|-----------|----------|
| 29 | resolveUpcoming với query có kết quả | resolvedIds.length > 0 |
| 30 | resolveUpcoming không có kết quả | reasonCode = "no_results" |
| 31 | resolveCfpOpen với query | resolvedIds chỉ chứa conference có CFP mở |
| 32 | resolveNearestDeadline | 1 hoặc N kết quả, sorted ASC |
| 33 | resolveUpcoming với filter tableName khác | fallback hoặc empty |
| 34 | TemporalResolver kết nối DB lỗi | reasonCode = "error", không crash |

### 9.5 RankingResolver

| # | Test case | Expected |
|---|-----------|----------|
| 35 | sort rank:desc | Kết quả sorted A > B > C |
| 36 | sort rank:asc | Kết quả sorted C > B > A |
| 37 | sort date:desc | Mới nhất trước |
| 38 | sort title:asc | A-Z |
| 39 | sort với conferenceIds rỗng | reasonCode = "error" |
| 40 | sort với policy unknown | reasonCode = "policy_unknown" |
| 41 | sort với ID không tồn tại | Bỏ qua ID đó, sort phần còn lại |

### 9.6 Integration — preToolValidator ordinal_nl

| # | Test case | Expected |
|---|-----------|----------|
| 42 | manageFollow identifier="thứ 2", identifierType="ordinal_nl" | Đúng parse → resolve → pass |
| 43 | manageFollow identifier="vừa rồi", identifierType="ordinal_nl" | resolveLastResult → pass |
| 44 | manageFollow identifier="top 3", identifierType="ordinal_nl" | resolveTop → pass (có thể cần mutation hỗ trợ multiple IDs) |
| 45 | manageFollow identifier="ABC", identifierType="ordinal_nl" | parse fail → ambiguity block |

---

## 10. Implementation Steps (Thứ tự code)

### Step 1: Tạo OrdinalParser

- **File:** `src/services/resultSetState/ordinalParser.ts`
- Export class `OrdinalParser` implements `IOrdinalParser`
- Pattern matching tables cho Vietnamese + English
- Test với tất cả patterns trong test plan

**Lưu ý:** KHÔNG dùng NLP/thư viện bên ngoài — chỉ dùng regex + string matching, siêu nhẹ.

### Step 2: Thêm resolveLastResult + resolveTop vào ResultSetResolver

- **File:** `src/services/resultSetState/resolver.service.ts`
- Thêm 2 method mới
- Export qua interface

### Step 3: Tạo TemporalResolver

- **File:** `src/services/resultSetState/temporalResolver.service.ts`
- Class `TemporalResolver` implements `ITemporalResolver`
- Query PostgreSQL trực tiếp qua `PgClient`
- 3 method: `resolveUpcoming`, `resolveCfpOpen`, `resolveNearestDeadline`
- Dùng `ConferenceSyncService` để format kết quả

**Chi tiết SQL cho TemporalResolver:**

```sql
-- resolveUpcoming
SELECT DISTINCT c.id, c.title, c.acronym
FROM conferences c
JOIN organization o ON o."conferenceId" = c.id
JOIN conference_date cd ON cd."organizationId" = o.id
WHERE cd.type = 'conferenceDates'
  AND cd."toDate" > NOW()
  AND (c.title ILIKE $1 OR c.acronym ILIKE $1)
ORDER BY cd."fromDate" ASC
LIMIT 20;

-- resolveCfpOpen
SELECT DISTINCT c.id, c.title, c.acronym
FROM conferences c
JOIN organization o ON o."conferenceId" = c.id
JOIN conference_date cd ON cd."organizationId" = o.id
WHERE cd.type = 'submissionDate'
  AND cd."toDate" > NOW()
  AND (c.title ILIKE $1 OR c.acronym ILIKE $1)
ORDER BY cd."toDate" ASC
LIMIT 20;

-- resolveNearestDeadline
SELECT c.id, c.title, c.acronym, cd."toDate" as deadline
FROM conferences c
JOIN organization o ON o."conferenceId" = c.id
JOIN conference_date cd ON cd."organizationId" = o.id
WHERE cd.type = 'submissionDate'
  AND cd."toDate" > NOW()
  AND (c.title ILIKE $1 OR c.acronym ILIKE $1)
ORDER BY cd."toDate" ASC
LIMIT 1;
```

### Step 4: Tạo RankingResolver

- **File:** `src/services/resultSetState/rankingResolver.service.ts`
- Class `RankingResolver` implements `IRankingResolver`
- Rank mapping: `A+ → 1, A → 2, A- → 3, B+ → 4, B → 5, B- → 6, C → 7`
- Sort theo từng policy
- Xử lý record không tìm thấy (skip gracefully)

### Step 5: Unit test

- **File:** `src/services/resultSetState/__tests__/ordinalParser.test.ts`
- **File:** `src/services/resultSetState/__tests__/temporalResolver.test.ts`
- **File:** `src/services/resultSetState/__tests__/rankingResolver.test.ts`
- **File:** `src/services/resultSetState/__tests__/resolver.service.test.ts` (update — thêm test cho resolveLastResult + resolveTop)

### Step 6: Export module

- **File:** `src/services/resultSetState/index.ts` — thêm export `OrdinalParser`, `TemporalResolver`, `RankingResolver`

### Step 7: Cập nhật function declarations — thêm "ordinal_nl"

- **File:** `src/chatbot/language/functions/english.ts`
- Thêm `"ordinal_nl"` vào enum identifierType của 6 mutation tools
- Cập nhật description:
  ```
  "The type of the identifier: 'acronym', 'title', 'id', 'ordinal', or 'ordinal_nl'. "
  + "Use 'ordinal' when you have already converted a position to a number (e.g., '2' for 'the 2nd one', "
  + "'-1' for 'the last one'). "
  + "Use 'ordinal_nl' when you want the system to parse the user's exact natural language phrase "
  + "(e.g., 'thứ 2', 'cuối', 'vừa rồi', 'top 3'). "
  + "When 'ordinal_nl', the identifier must be the exact phrase the user used."
  ```
- **File:** `src/chatbot/language/functions/vietnamese.ts` — tương tự
- **File:** `src/chatbot/language/functions/spanish.ts` — tương tự

### Step 8: Cập nhật preToolValidator — Fast Path cho ordinal_nl

- **File:** `src/chatbot/guards/preToolValidator.ts`
- Mở rộng `tryResolveOrdinal` hoặc tạo `tryResolveOrdinalNL`
- Luồng xử lý:

```typescript
if (identifierType.trim() === "ordinal_nl") {
  const parseResult = ordinalParser.parse(identifier as string);

  switch (parseResult.type) {
    case "ordinal": {
      // Dùng resolveAll như ordinal thường
      const result = await resolver.resolveAll(convId, parseResult.value);
      // ... xử lý uniqueMatch / 0 match / nhiều match như cũ
    }
    case "last_result": {
      const result = await resolver.resolveLastResult(convId);
      if (result.resolvedId) {
        normalizedArgs.identifier = result.resolvedId;
        normalizedArgs.identifierType = "id";
        return { allowed: true, ... };
      }
      // fallback → out_of_range_reference
    }
    case "top_n": {
      // top N — resolve nhiều ID
      const result = await resolver.resolveTop(convId, parseResult.value);
      if (result.resolvedIds.length > 0) {
        // Nếu tool support multiple IDs, pass array
        // Nếu không, lấy ID đầu tiên
        normalizedArgs.identifier = result.resolvedIds[0];
        normalizedArgs.identifierType = "id";
        return { allowed: true, ... };
      }
    }
    case "unresolved": {
      // fallback → ambiguity block
      return buildBlockedResult({
        errorCode: OrchestrationSafetyErrorCode.ORDINAL_PARSE_FAILED,
        message: `Could not parse ordinal reference: "${identifier}".`,
        ...
      });
    }
  }
}
```

### Step 9: Mở rộng retrieveKnowledge handler — temporalMode + rankBy

- **File:** `src/chatbot/handlers/retrieveKnowledge.handler.ts`
- Thêm 2 param mới vào args: `temporalMode` và `rankBy`
- Khi `temporalMode` có value:
  - Bỏ qua retrievalService.retrieve()
  - Gọi TemporalResolver thay thế
  - Lưu kết quả vào ResultSetState
- Khi `rankBy` có value:
  - Giữ nguyên retrievalService.retrieve()
  - Sau đó gọi RankingResolver.sort() để sắp xếp lại
  - Lưu kết quả đã sort vào ResultSetState

```typescript
// Trong execute(), trước hoặc sau retrieve:

// Temporal path
if (temporalMode) {
  const temporalResult = await this.temporalResolver.resolve(query, {
    mode: temporalMode,
    filter: effectiveFilter,
  });
  // ... format + save + return
}

// Ranking post-processing
if (rankBy && results.length > 0) {
  const ids = results.map(r => r.metadata?.recordId || r.metadata?.id);
  const ranked = await this.rankingResolver.sort(ids, parseRankBy(rankBy));
  // Re-order results theo ranked.resolvedIds
}
```

### Step 10: Thêm function declaration params mới

- **File:** `src/chatbot/language/functions/english.ts` (retrieveKnowledge declaration)
- Thêm 2 params:

```typescript
temporalMode: {
  type: Type.STRING,
  description: "Optional. Filter conferences by temporal criteria: 'upcoming' (future conferences), 'cfp_open' (conferences with open call for papers), 'nearest_deadline' (conferences with nearest submission deadline). When set, overrides normal semantic search.",
  enum: ["upcoming", "cfp_open", "nearest_deadline"],
  nullable: true,
},

rankBy: {
  type: Type.STRING,
  description: "Optional. Sort/rank criteria for the result list: 'rank:desc' (best rank first), 'rank:asc' (worst rank first), 'date:desc' (newest first), 'date:asc' (oldest first), 'title:asc' (A-Z), 'title:desc' (Z-A).",
  nullable: true,
},
```

### Step 11: Cập nhật system prompt

- **File:** `src/chatbot/language/instructions/english.ts`
- **File:** `src/chatbot/language/instructions/vietnamese.ts`
- **File:** `src/chatbot/language/instructions/spanish.ts`

Thêm section cho:
1. ConferenceAgent: temporal mode, ranking mode, ordinal_nl (xem Section 7)
2. HostAgent: routing temporal/ranking requests

### Step 12: Register TemporalResolver + RankingResolver trong DI container

- **File:** `src/chatbot/di/container.ts` (hoặc tương đương)
- `container.registerSingleton(TemporalResolver)`
- `container.registerSingleton(RankingResolver)`
- `container.registerSingleton(OrdinalParser)`

---

## 11. File structure đề xuất

```
src/
  services/
    resultSetState/
      ordinalParser.ts                      # [MỚI] NL → OrdinalParseResult
      temporalResolver.service.ts           # [MỚI] Upcoming / CfpOpen / NearestDeadline
      rankingResolver.service.ts            # [MỚI] Sort by rank/date/title policy
      resolver.service.ts                   # [SỬA] + resolveLastResult, resolveTop
      store.service.ts                      # KHÔNG ĐỔI
      index.ts                              # [SỬA] + export mới
      __tests__/
        ordinalParser.test.ts               # [MỚI]
        temporalResolver.test.ts            # [MỚI]
        rankingResolver.test.ts             # [MỚI]
        resolver.service.test.ts            # [SỬA] + test resolveLastResult, resolveTop
        store.service.test.ts               # KHÔNG ĐỔI
  chatbot/
    handlers/
      retrieveKnowledge.handler.ts          # [SỬA] + temporalMode + rankBy
      resolveConferenceRef.handler.ts       # KHÔNG ĐỔI
    guards/
      preToolValidator.ts                   # [SỬA] + ordinal_nl fast path
    language/
      functions/
        english.ts                          # [SỬA] + "ordinal_nl" enum + temporalMode + rankBy
        vietnamese.ts                       # [SỬA] + "ordinal_nl" enum
        spanish.ts                          # [SỬA] + "ordinal_nl" enum
      instructions/
        english.ts                          # [SỬA] + system prompt temporal/ranking/ordinal_nl
        vietnamese.ts                       # [SỬA] + system prompt temporal/ranking/ordinal_nl
        spanish.ts                          # [SỬA] + system prompt temporal/ranking/ordinal_nl
    di/
      container.ts                          # [SỬA] + register mới
```

---

## 12. Definition of Done (DoD)

- [x] `OrdinalParser.parse()` xử lý đúng tất cả NL patterns (20+ test cases)
- [x] `OrdinalParser.parse()` cho "vừa rồi" → `{ type: "last_result" }`
- [x] `OrdinalParser.parse()` cho "top N" → `{ type: "top_n", value: N }`
- [x] `ResultSetResolver.resolveLastResult()` hoạt động đúng
- [x] `ResultSetResolver.resolveTop()` hoạt động đúng
- [x] `TemporalResolver.resolveUpcoming()` query DB → trả danh sách ID
- [x] `TemporalResolver.resolveCfpOpen()` query DB → trả danh sách ID
- [x] `TemporalResolver.resolveNearestDeadline()` query DB → trả deadline gần nhất
- [x] `RankingResolver.sort()` sort theo rank policy
- [x] `RankingResolver.sort()` sort theo date policy
- [x] `RankingResolver.sort()` sort theo title policy
- [x] `"ordinal_nl"` đã thêm vào identifierType enum của tất cả mutation tools
- [x] preToolValidator xử lý `ordinal_nl` → parse → resolve (Fast Path)
- [x] preToolValidator xử lý "vừa rồi" → resolveLastResult
- [x] preToolValidator xử lý "top N" → resolveTop
- [x] `retrieveKnowledge` handler hỗ trợ `temporalMode` param
- [x] `retrieveKnowledge` handler hỗ trợ `rankBy` param
- [x] System prompt cập nhật cho ConferenceAgent (temporal, ranking, ordinal_nl)
- [x] System prompt cập nhật cho HostAgent (routing)
- [x] Unit test pass ≥ 95%
- [x] Error codes mới (temporal_no_results, ranking_policy_unknown, ordinal_parse_failed) đã thêm
- [x] Temporal/Ranking result được lưu vào ResultSetState

---

## 13. Phụ thuộc

| Phụ thuộc | Ghi chú |
|-----------|---------|
| P1-01 (ResultSetState V1) | Đã hoàn thành — ordinal resolver + store có sẵn |
| ConferenceSyncService | Đã có — dùng cho TemporalResolver query DB |
| PgClient | Đã có — temporal query SQL qua pgClient |
| EmbeddingService | Đã có — không cần thay đổi |
| MongoDB + PostgreSQL | Đã có — không cần migration mới |
| preToolValidator | Đã có ordinal fast path — cần mở rộng cho ordinal_nl |

---

## 14. Rủi ro & Mitigation

| Rủi ro | Xác suất | Mitigation |
|--------|----------|------------|
| LLM không dùng ordinal_nl dù đã hướng dẫn | Medium | Giữ backward compatible với "ordinal" — nếu LLM vẫn dùng ordinal cũ, vẫn hoạt động |
| TemporalResolver SQL performance với nhiều conference | Low | Đánh index trên conference_date.type + toDate, limit 20 |
| RankingResolver load nhiều conference cùng lúc (N request) | Medium | Batch load records (IN query) thay vì N single queries |
| OrdinalParser bỏ sót pattern | Low | Dễ dàng thêm pattern mới — chỉ là thêm string vào mảng |

---

## 15. Ghi chú implementation

### 15.1 OrdinalParser implementation tips

- Đặt các pattern array ở class level (const) để dễ maintain
- Kiểm tra priority: "vừa rồi" > "top N" > "cuối" > "thứ N"
- Tiếng Việt không dấu: "thu 2", "thu hai", "cuoi" — thêm pattern cho flexible matching

### 15.2 TemporalResolver implementation tips

- TemporalResolver KHÔNG dùng ResultSetState — nó query trực tiếp DB
- Sau khi temporal resolve, handler NÊN lưu kết quả vào ResultSetState để ordinal reference sau này có thể dùng
- Dùng `conference_date` table với `type` field để phân biệt loại date
- Field type values: `"conferenceDates"`, `"submissionDate"`, `"cameraReadyDate"`, `"notificationDate"`
- Cần normalize ILIKE query parameter: `%${query.toLowerCase()}%`

### 15.3 RankingResolver implementation tips

- Rank value mapping (dựa trên CORE/rank name):
  ```
  "A*" → 0, "A+" → 0, "A" → 1, "A-" → 2,
  "B+" → 3, "B" → 4, "B-" → 5,
  "C+" → 6, "C" → 7, "C-" → 8,
  "national" → 9, null → 99
  ```
- Date sort: lấy `organization.dates` filter `type === "conferenceDates"`, sort by `fromDate`
- Title sort: localeCompare hoặc simple string comparison
