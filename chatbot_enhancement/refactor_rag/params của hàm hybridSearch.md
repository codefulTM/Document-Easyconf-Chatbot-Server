# Query Parameters Specification

## Filter

Dùng để lọc dữ liệu.

### Logical Filter

```ts
type Filter =
  | AndFilter
  | OrFilter
  | NotFilter
  | ComparisonFilter;
```

```ts
interface AndFilter {
  op: "and";
  filters: Filter[];
}

interface OrFilter {
  op: "or";
  filters: Filter[];
}

interface NotFilter {
  op: "not";
  filter: Filter;
}
```

---

### Comparison Filter

```ts
interface ComparisonFilter {
  // Field phải dùng dạng qualified name để tránh mơ hồ khi nhiều bảng có cùng tên cột.
  // Format: <tableName>.<fieldName>
  field: string;

  op:
    | "eq"
    | "gt"
    | "gte"
    | "lt"
    | "lte"
    | "between"
    | "in"
    | "contains"
    | "startsWith"
    | "endsWith"
    | "isNull";

  value?: unknown;
}
```

Ví dụ:

```ts
{
  field: "ConferenceOrganizations.reviewType",
  op: "startsWith",
  value: "A"
}
```

SQL:

```sql
"ConferenceOrganizations"."reviewType" ILIKE 'A%'
```

---

## Search

Dùng cho full text search, fuzzy search, semantic search và hybrid search.

```ts
type Search =
  | FuzzySearch
  | SemanticSearch
  | HybridSearch
  | SearchGroup;
```

---

### Search Group

```ts
interface SearchGroup {
  op: "and" | "or";

  searches: Search[];
}
```

Ví dụ:

```ts
{
  op: "or",
  searches: [
    {
      field: "title",
      op: "fuzzy",
      query: "postgres"
    },
    {
      field: "content",
      op: "semantic",
      query: "vector database"
    }
  ]
}
```

---

### Fuzzy Search

```ts
interface FuzzySearch {
  field: string;

  op: "fuzzy";

  query: string;

  threshold?: number;
}
```

Ví dụ:

```ts
{
  field: "title",
  op: "fuzzy",
  query: "IE",
  threshold: 0.3
}
```

PostgreSQL:

```sql
similarity(title, 'IE') >= 0.3
```

---

### Semantic Search

```ts
interface SemanticSearch {
  field: string;

  op: "semantic";

  query: string;

  threshold?: number;
}
```

Ví dụ:

```ts
{
  field: "content",
  op: "semantic",
  query: "AI conference"
}
```

PostgreSQL:

```sql
content_embedding <=> query_embedding
```

---

### Hybrid Search

```ts
interface HybridSearch {
  field: string;

  op: "hybrid";

  query: string;

  fuzzyWeight?: number;

  semanticWeight?: number;

  threshold?: number;
}
```

Ví dụ:

```ts
{
  field: "content",
  op: "hybrid",
  query: "AI conference",
  fuzzyWeight: 0.3,
  semanticWeight: 0.7
}
```

Điểm:

```text
score =
    fuzzy_score * fuzzyWeight
  + semantic_score * semanticWeight
```

---

## Select

Dùng để chọn field và biểu thức trả về trong query.

### Field format

- `field` phải dùng qualified name để tránh mơ hồ khi nhiều bảng có cùng tên cột.
- Format chuẩn: `<tableName>.<fieldName>`
- Hỗ trợ wildcard khi cần lấy toàn bộ field của một bảng: `<tableName>.*`
- Với field ảo/tính toán, dùng `as` để đặt tên cột output.

```ts
type Select = SelectAll | SelectField | SelectExpr | SelectGroup;
```

```ts
interface SelectAll {
  op: "all";
}

interface SelectField {
  op: "field";
  field: string;
  as?: string;
}

interface SelectExpr {
  op:
    | "count"
    | "sum"
    | "avg"
    | "min"
    | "max"
    | "distinct"
    | "coalesce"
    | "concat"
    | "case"
    | "json"
    | "jsonAgg";
  field?: string;
  fields?: string[];
  value?: unknown;
  as?: string;
}

interface SelectGroup {
  op: "group";
  selects: Select[];
}
```

Ví dụ:

```ts
{
  op: "group",
  selects: [
    { op: "field", field: "Conferences.id" },
    { op: "field", field: "Conferences.title" },
    { op: "jsonAgg", field: "ConferenceOrganizations.*", as: "organizations" },
    { op: "count", field: "ConferenceFollows.id", as: "followCount" }
  ]
}
```

SQL ý tưởng:

```sql
SELECT
  c.id,
  c.title,
  json_agg(org.*) AS organizations,
  count(f.id) AS followCount
```

---

## Group By

Dùng để gom nhóm dữ liệu.

```ts
interface GroupBy {
  fields: string[];

  having?: Filter;
}
```

Ví dụ:

```ts
{
  fields: ["country", "city"]
}
```

SQL:

```sql
GROUP BY country, city
```

---

### Having

Ví dụ:

```ts
{
  fields: ["department"],
  having: {
    field: "employee_count",
    op: "gt",
    value: 10
  }
}
```

SQL:

```sql
GROUP BY department
HAVING COUNT(*) > 10
```

---

## Order By

Dùng để sắp xếp kết quả.

```ts
interface OrderBy {
  field: string;

  direction?: "asc" | "desc";

  nulls?: "first" | "last";
}
```

Ví dụ:

```ts
{
  field: "createdAt",
  direction: "desc"
}
```

SQL:

```sql
ORDER BY created_at DESC
```

---

### Multiple Order

```ts
type OrderByList = OrderBy[];
```

Ví dụ:

```ts
[
  {
    field: "department",
    direction: "asc"
  },
  {
    field: "salary",
    direction: "desc"
  }
]
```

SQL:

```sql
ORDER BY department ASC,
         salary DESC
```

---

## Query

```ts
interface Query {
  filter?: Filter;

  search?: Search;

  groupBy?: GroupBy;

  orderBy?: OrderBy[];

  limit?: number;

  offset?: number;
}
```

Ví dụ:

```ts
{
  filter: {
    op: "and",
    filters: [
      {
        field: "status",
        op: "eq",
        value: "published"
      }
    ]
  },

  search: {
    field: "content",
    op: "hybrid",
    query: "AI conference",
    fuzzyWeight: 0.3,
    semanticWeight: 0.7
  },

  groupBy: {
    fields: ["country"]
  },

  orderBy: [
    {
      field: "score",
      direction: "desc"
    }
  ],

  limit: 20,
  offset: 0
}
```
