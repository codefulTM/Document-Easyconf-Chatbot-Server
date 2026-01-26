
# Workflow Types — Documentation

Mô tả các kiểu dữ liệu và giao diện (interfaces) dùng bởi workflow engine, theo định nghĩa trong `workflow.types.ts`.

## Overview
- **Purpose:** Định nghĩa cấu trúc cho các node workflow, trạng thái thực thi, kết quả và lỗi. Giúp chuẩn hoá cách biểu diễn các workflow phức tạp, dễ lưu trữ, resume và debug.
- **Audience:** Developer cần tạo/đọc/thi hành workflow, hoặc xây UI để hiển thị/biên tập workflow.

## NodeType
Union các loại node hiện có:

```ts
type NodeType =
	| "ROUTE_TO_AGENT"
	| "RETRIEVE_KNOWLEDGE"
	| "DECISION"
	| "PARAPHRASE_QUERY"
	| "FALLBACK"
	| "END";
```

Mô tả ngắn:
- `ROUTE_TO_AGENT`: chuyển yêu cầu sang một agent cụ thể.
- `RETRIEVE_KNOWLEDGE`: truy vấn hệ tri thức (vector DB / RDBMS).
- `DECISION`: LLM/logic đưa ra nhánh tiếp theo.
- `PARAPHRASE_QUERY`: sinh lại/biên dịch query.
- `FALLBACK`: xử lý khi lỗi hoặc không tìm được kết quả.
- `END`: kết thúc workflow.

## WorkflowNode
Trường hợp một node trong workflow:

```ts
interface WorkflowNode {
	id: string;
	type: NodeType;
	config: Record<string, any>;
	onSuccess?: string;
	onFailure?: string;
	retryCount?: number;
	maxRetries?: number;
	retryDelay?: number;
	timeout?: number; // ms
}
```

- `id`: định danh duy nhất cho node.
- `config`: cấu hình tuỳ theo `type` (ví dụ `RetrieveKnowledgeConfig`).
- `onSuccess`/`onFailure`: id node kế tiếp.
- `retryCount`, `maxRetries`, `retryDelay`: cơ chế thử lại.
- `timeout`: timeout node (ms).

## Workflow
Định nghĩa toàn bộ workflow:

```ts
interface Workflow {
	id: string;
	name: string;
	description?: string;
	entryPoint: string; // ID của node bắt đầu
	nodes: Record<string, WorkflowNode>;
	context: Record<string, any>;
	metadata?: { createdAt: string; version: string; tags?: string[] };
}
```

`context` dùng để lưu dữ liệu chạy ngang qua các node (input, intermediate results, external ids,…).

## WorkflowExecutionResult
Thông tin trả về sau khi thực thi một workflow:

```ts
interface WorkflowExecutionResult {
	workflowId: string;
	success: boolean;
	finalNode: string;
	context: Record<string, any>;
	executionPath: string[];
	duration: number; // ms
	executionDetails?: Array<{
		nodeId: string;
		nodeType: string;
		input?: any;
		output?: any;
		duration: number;
		status: "success" | "error" | "pending";
		error?: string;
	}>;
	error?: string;
}
```

`executionDetails` hữu ích để debug và audit.

## Config types (node-specific)

- `RouteToAgentConfig` — cấu hình khi route đến agent:

```ts
interface RouteToAgentConfig {
	agentId: string;
	taskDescription: string;
	parameters?: Record<string, any>;
}
```

- `RetrieveKnowledgeConfig` — truy xuất tri thức:

```ts
interface RetrieveKnowledgeConfig {
	query: string;
	options?: { limit?: number; useHybrid?: boolean; threshold?: number; filter?: Record<string, any> };
}
```

- `DecisionConfig` + `DecisionOption` — cấu hình node ra quyết định:

```ts
interface DecisionConfig {
	condition?: string; // LLM prompt for decision
	options: DecisionOption[];
	fallbackOption?: string;
}

interface DecisionOption {
	value: string;
	label: string;
	nextNode: string;
	condition?: string;
}
```

- `ParaphraseQueryConfig` — paraphrase/sanitize query:

```ts
interface ParaphraseQueryConfig {
	originalQuery: string;
	strategy?: "simple" | "detailed" | "focused";
	targetKeywords?: string[];
}
```

- `FallbackConfig` — xử lý khi lỗi/không tìm được dữ liệu:

```ts
interface FallbackConfig {
	message?: string;
	action?: "return_error" | "return_default" | "retry_workflow";
	defaultResponse?: any;
}
```

## Events & State

- `WorkflowEvent` — sự kiện theo dõi trạng thái node/workflow:

```ts
type WorkflowEventType =
	| "node_start"
	| "node_success"
	| "node_error"
	| "node_retry"
	| "workflow_complete";

interface WorkflowEvent { type: WorkflowEventType; nodeId: string; timestamp: number; data?: any }
```

- `NodeState` và `WorkflowState` — lưu trạng thái để resume/persistence:

```ts
interface NodeState { retryCount: number; lastAttempt: number; status: "pending" | "success" | "error" }

interface WorkflowState {
	workflowId: string;
	currentNodeId: string;
	context: Record<string, any>;
	executionPath: string[];
	nodeStates: Record<string, NodeState>;
}
```

## Error classes

- `WorkflowError` — lỗi chung kèm nodeId.
- `NodeTimeoutError` — lỗi timeout node.
- `WorkflowExecutionError` — lỗi khi thực thi workflow (kèm path/context).

Ví dụ (ts):

```ts
class WorkflowError extends Error { constructor(message: string, public nodeId: string, public originalError?: Error) { super(message); } }

class NodeTimeoutError extends WorkflowError { constructor(nodeId: string, timeout: number) { super(`Node ${nodeId} timed out after ${timeout}ms`, nodeId); } }

class WorkflowExecutionError extends Error { constructor(message: string, public workflowId: string, public executionPath: string[], public context: Record<string, any>) { super(message); } }
```

## Example: simple workflow

```ts
const wf: Workflow = {
	id: 'wf1',
	name: 'Search and Route',
	entryPoint: 'node1',
	nodes: {
		node1: { id: 'node1', type: 'RETRIEVE_KNOWLEDGE', config: { query: '{{input}}' }, onSuccess: 'node2' },
		node2: { id: 'node2', type: 'DECISION', config: { options: [{ value: 'hasAnswer', nextNode: 'node3' }, { value: 'noAnswer', nextNode: 'node4' }] } },
		node3: { id: 'node3', type: 'ROUTE_TO_AGENT', config: { agentId: 'agentA' }, onSuccess: 'END' },
		node4: { id: 'node4', type: 'FALLBACK', config: { message: 'No results' }, onSuccess: 'END' },
	},
	context: {},
};
```

## Best practices
- Đặt `entryPoint` rõ ràng và dùng `onSuccess/onFailure` để mô tả flow.
- Lưu `context` có keys rõ ràng (e.g., `userQuery`, `retrievedChunks`, `agentResponse`).
- Sử dụng `timeout` và `maxRetries` cho các node có gọi mạng / LLM.
- Hoàn thiện `executionDetails` để dễ audit và tái thực hiện khi cần.

---

Tệp này mô tả các kiểu chính; khi mở rộng node mới, cập nhật cả `NodeType`, cấu hình node tương ứng và phần **Best practices**.

