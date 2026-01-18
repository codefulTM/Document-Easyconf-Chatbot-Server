# Hiện tại: 
- Luồng chat tương tác (HostAgent / API chat trực tiếp) chỉ dùng primaryGeminiApiKey (một API key duy nhất).
- Luồng batch / crawl / extract/determine/cfp thì dùng cơ chế pool + round‑robin (nhiều key), do đó nhiều API key chỉ được sử dụng cho những tác vụ đó.

# Bằng chứng từ mã nguồn
- `Gemini` wrapper (chat): tạo client trực tiếp từ `primaryGeminiApiKey` và gọi `models.generateContent` — client không dùng pool/rotation. Xem: [src/chatbot/gemini/gemini.ts](src/chatbot/gemini/gemini.ts#L1-L20)
- HostAgent khởi tạo `Gemini` với `primaryGeminiApiKey` khi tạo dependencies cho luồng chat: [src/chatbot/handlers/intentHandler.orchestrator.ts](src/chatbot/handlers/intentHandler.orchestrator.ts#L33-L41)
- Orchestration cho các tác vụ crawl/extract/determine/cfp sử dụng `GeminiApiService` → `GeminiApiOrchestratorService` (truyền `apiType`) → `GeminiClientManagerService` để chọn key theo pool/round-robin. Xem: [src/services/gemini/geminiApi.service.ts](src/services/gemini/geminiApi.service.ts#L1-L40) và [src/services/gemini/geminiApiOrchestrator.service.ts](src/services/gemini/geminiApiOrchestrator.service.ts#L1-L20)
- Logic chọn key theo pool và round-robin được thực hiện trong `GeminiClientManagerService.getApiKeyIdForApiType`. Xem: [src/services/gemini/geminiClientManager.service.ts](src/services/gemini/geminiClientManager.service.ts#L1-L40) (định nghĩa `keyPools`, `apiTypeToPoolMap`) và [src/services/gemini/geminiClientManager.service.ts](src/services/gemini/geminiClientManager.service.ts#L120-L200) (hàm `getApiKeyIdForApiType`).

# Việc tiếp theo cần làm: Thay đổi HostAgent để sử dụng pool-based GeminiClientManagerService

# Thay đổi thực hiện
- Đã thêm adapter `PooledGemini` để gọi Gemini thông qua `GeminiClientManagerService` (sử dụng pool/round-robin theo `apiType`). File mới: [src/chatbot/gemini/pooledGemini.ts](src/chatbot/gemini/pooledGemini.ts)
- Đã cập nhật HostAgent/SubAgent để khởi tạo `PooledGemini` thay vì `Gemini`, cho phép luồng chat sử dụng pool khi có `additionalGeminiApiKeys`. File: [src/chatbot/handlers/intentHandler.orchestrator.ts](src/chatbot/handlers/intentHandler.orchestrator.ts#L1-L120)
- Lưu ý: `PooledGemini.generateTurn(..., apiType)` mặc định dùng `apiType='interactive'` cho HostAgent. Các luồng batch/extract/... vẫn dùng `GeminiApiService`/orchestrator như trước.
- Cập nhật lại keyPools trong `GeminiClientManagerService`. File: [src/services/gemini/geminiClientManager.service.ts](src/services/gemini/geminiClientManager.service.ts#L30-L40)
- Cập nhật thêm API key vào .env