# WorkflowManager — Documentation

## Overview

`WorkflowManager` là lớp trung tâm điều phối thực thi workflow nhiều bước cho chatbot, hỗ trợ retry, timeout, fallback, event tracking và tích hợp với các service như Retrieval, Gemini (LLM), agent routing. Lớp này kế thừa `EventEmitter` để phát các sự kiện trạng thái node/workflow.

---

## Khởi tạo

```ts
constructor(
  workflow: Workflow,
  options?: {
    subAgentDeps?: SubAgentHandlerCustomDeps;
    callSubAgentHandler?: (
      requestCard: AgentCardRequest,
      parentHandlerId: string,
      language: Language,
      socket: Socket
    ) => Promise<AgentCardResponse>;
    socket?: Socket;
    retrievalService?: RetrievalService;
    geminiService?: Gemini;
  }
)
```
- Nhận vào một đối tượng workflow (deep clone).
- Nhận các dependency cho agent, retrieval, LLM, socket, v.v.
- Tự động tăng `_workflowDepth` để tránh đệ quy vô hạn.
- Khởi tạo trạng thái node (`initializeNodeStates`).

---

## Các phương thức chính

### 1. `execute(): Promise<WorkflowExecutionResult>`

- Thực thi workflow từ entry point.
- Lặp qua từng node, xử lý retry, timeout, chuyển node theo kết quả/thất bại.
- Lưu kết quả vào context (`${nodeId}_result`), trạng thái node, executionPath, executionDetails.
- Phát các sự kiện: `node_start`, `node_success`, `node_error`, `node_retry`, `workflow_complete`.
- Trả về kết quả tổng hợp gồm trạng thái, context, path, duration, chi tiết từng node.

### 2. `executeNode(node: WorkflowNode): Promise<any>`

- Thực thi một node theo type:
  - `"ROUTE_TO_AGENT"`: gọi agent khác (qua handler hoặc mô phỏng).
  - `"RETRIEVE_KNOWLEDGE"`: truy vấn hệ tri thức (retrieval service hoặc mô phỏng).
  - `"DECISION"`: quyết định nhánh tiếp theo (LLM hoặc random fallback).
  - `"PARAPHRASE_QUERY"`: sinh lại query (LLM Gemini).
  - `"FALLBACK"`: xử lý fallback (error, retry, default).
  - `"END"`: kết thúc workflow.

### 3. `executeWithTimeout(node, timeoutMs)`

- Thực thi node với timeout, nếu quá thời gian sẽ ném `NodeTimeoutError`.

### 4. `executeRouteToAgent(config: RouteToAgentConfig)`

- Gọi agent khác qua handler hoặc mô phỏng.
- Hỗ trợ truyền context, taskDescription, nhận kết quả/thoughts/action từ agent.

### 5. `executeRetrieveKnowledge(config: RetrieveKnowledgeConfig)`

- Truy vấn knowledge base qua retrieval service hoặc mô phỏng.
- Hỗ trợ limit, hybrid, threshold, filter.

### 6. `executeDecision(config: DecisionConfig)`

- Nếu có Gemini, tạo prompt và gọi LLM để quyết định nhánh tiếp theo.
- Nếu không, chọn random fallback.
- Hỗ trợ parse JSON response từ LLM, validate option.

### 7. `executeParaphraseQuery(config: ParaphraseQueryConfig)`

- Gọi Gemini để sinh nhiều paraphrase cho query.
- Parse kết quả, chọn random một paraphrase, lưu vào context.

### 8. `executeFallback(config: FallbackConfig)`

- Xử lý fallback: ném lỗi, retry workflow, hoặc trả về default.

---

## Tiện ích nội bộ

- `substituteTemplateVariables(config)`: Thay thế biến `{{var}}` trong config bằng giá trị context.
- `sanitizeContextForNextNode(nextNodeId)`: Xoá các key tạm (`retryCount`, `errorType`, các `_result` không thuộc node tiếp theo).
- `summarizeWorkflowContext(overrides)`: Tóm tắt context cho prompt LLM.
- `emitEvent(type, nodeId, data)`: Phát sự kiện workflow/node.
- `getState()`: Lấy trạng thái hiện tại của workflow (để resume/debug).
- `updateContext(key, value)`: Cập nhật context.
- `getExecutionPath()`: Lấy execution path hiện tại.

---

## Sự kiện (EventEmitter)

- `"workflow_event"`: phát ra mỗi khi node/workflow thay đổi trạng thái.
  - type: `"node_start" | "node_success" | "node_error" | "node_retry" | "workflow_complete"`
  - nodeId, timestamp, data

---

## Ví dụ sử dụng

```ts
const manager = new WorkflowManager(workflow, { retrievalService, geminiService, ... });
manager.on("workflow_event", (event) => { ... });
const result = await manager.execute();
console.log(result.success, result.executionPath, result.context);
```

---

## Best Practices

- Luôn deep clone workflow khi khởi tạo để tránh side-effect.
- Theo dõi sự kiện để hiển thị tiến trình hoặc debug.
- Sử dụng context để truyền dữ liệu giữa các node.
- Đảm bảo các node có `onSuccess`/`onFailure` hợp lý để tránh dead-end.

---

## Xem thêm

- [workflow.types.md](workflow.types.md) — Định nghĩa các type, interface cho workflow engine.
- [conferenceWorkflow.ts](../examples/conferenceWorkflow.ts) — Ví dụ workflow thực tế.
