# Agent là gì?
Agent (tác tử) trong lĩnh vực AI là một hệ thống có khả năng tự động nhận biết môi trường (sense), suy nghĩ/lập kế hoạch (think/plan), và thực hiện hành động (act) để đạt mục tiêu nhất định. Agent có thể sử dụng các công cụ (tools), lưu trữ trạng thái (state/context), và tự quyết định chuỗi hành động dựa trên thông tin hiện tại thay vì chỉ làm theo kịch bản cố định.

# Trước khi có workflow hay agent
Luồng hoạt động của chatbot trước khi có tính năng workflow/agent như sau:
1. Chuẩn bị các tham số đầu vào và khởi tạo
2. Định nghĩa các helpers nội bộ
3. Vòng lặp chính(Lặp tối đa maxTurnsHostAgent lần):
- Gửi event status_update
- Gọi LLM để lấy response
- Xử lý response từ LLM:
	- Nếu lỗi, phát sự kiện chat_error
	- Nếu có function call:
		- Tạo turn role: 'model' chứa function call và đẩy vào lịch sử
		- Nếu function call là 'routeToAgent', gọi subagent tương ứng và nhận kết quả trả về, đồng thời xử lý frontend action
		- Tạo turn role: 'function' với kết quả trả về từ function call và đẩy vào lịch sử
		- Tiếp tục vòng lặp, không kết thúc
	- Nếu không có function call, chỉ có response stream:
		- Đọc từng chunk trả về bởi LLM và phát các cập nhật response về socket
		- Tạo turn role: 'model' với nội dung response đầy đủ và đẩy vào lịch sử, cũng như cập nhật response cuối cùng bên frontend
		- Kết thúc vòng lặp



# Vấn đề hiện tại
- workflow.types.ts và workflowManager.ts dường như chỉ được code cho ConferenceWorkflow. Đang gặp khó khăn khi mở rộng cho các workflow khác.

- Muốn LLM hoạt động như agent thì phải định nghĩa workflow từ trước một cách rườm rà phức tạp. Cần cách để LLM hoạt động như một agent mà không cần định nghĩa trước workflow quá chi tiết.

- Mỗi khi LLM trả về stream response, lại kết thúc vòng lặp. Điều này không phù hợp với mô hình agent, nơi LLM có thể cần thực hiện nhiều hành động liên tiếp trước khi hoàn thành nhiệm vụ.

# Giải pháp
Dưới đây là hướng tiếp cận để cho LLM hoạt động như một agent mà không cần phải định nghĩa workflow tĩnh trước đó. Mục tiêu là cung cấp một tập hợp công cụ (tools) + state/context cho LLM và để LLM quyết định hành động theo vòng lặp sense → think → act.

## 1) Kiến trúc tổng quan
- Tool registry: danh sách các hành động có thể gọi (ví dụ: `search`, `callAgent`, `emitEvent`, `upsertMemory`). Mỗi tool có tên, mô tả và wrapper function thực thi.
- Environment / State: object chứa context hiện tại, history, memory, và input từ user.
- Planner (LLM): nhận context + mô tả tools, trả về action có cấu trúc (JSON) chỉ rõ tool và input.
- Executor: xác thực và thực thi tool, trả kết quả trở lại state.
- Loop controller: lặp cho tới khi action là `DONE` hoặc đạt `maxSteps`/timeout.

## 2) Prompt & Structured Output
- Yêu cầu LLM trả về JSON theo schema cố định để dễ parse và validate, ví dụ:

```json
{
	"action": "CALL_TOOL" | "THINK" | "DONE",
	"tool": "search" | "callAgent" | null,
	"input": { ... },
	"reason": "short explanation"
}
```

- Nếu dùng API function-calling (OpenAI-like) hoặc Gemini function schema, đăng ký functions tương ứng để LLM dễ trả về kết cấu đã xác định.

## 3) Vòng lặp agent (sense → think → act)
1. Sense: thu thập input hiện tại (user message, kết quả retrieval, trạng thái).
2. Think: gọi LLM với prompt chứa mô tả tools, state tóm tắt và history → nhận action JSON.
3. Act: executor thực thi tool được chỉ định, ghi kết quả vào history và update state.
4. Lặp lại cho đến khi LLM trả `DONE` hoặc đạt giới hạn bước.

## 4) Kiểm soát & An toàn
- Giới hạn tổng bước (`maxSteps`) và tổng timeout cho toàn agent.
- Timeout/ retry cho từng action.
- Validate output của LLM theo schema; whitelist tên tool.
- Circuit-breaker / rate-limit khi gọi tool ngoài.
- Nếu LLM đề xuất tool không tồn tại: trả lỗi có kiểm soát và hỏi LLM lại.

## 5) Pattern tools / function-calling
- Định nghĩa tools với schema input rõ ràng; executor phải validate trước khi gọi.
- Ví dụ tools: `search`, `fetchProfile`, `callSubAgent`, `emitEvent`.
- Preferrable: dùng function-calling để ép LLM trả cấu trúc có thể parse ngay.

## 6) Ví dụ TypeScript (minimal)
```ts
type Action = { action: 'CALL_TOOL' | 'THINK' | 'DONE'; tool?: string; input?: any; reason?: string };
type Tool = { name: string; description: string; call: (input:any,state:any)=>Promise<any> };

class LlMAgent {
	tools: Record<string,Tool>;
	state: any;
	maxSteps = 10;
	llmCall: (prompt:string)=>Promise<string>;

	constructor(tools:Tool[], llmCall:(p:string)=>Promise<string>, initState={}){
		this.tools = Object.fromEntries(tools.map(t=>[t.name,t]));
		this.llmCall = llmCall; this.state = initState;
	}

	buildPrompt(){
		const toolList = Object.values(this.tools).map(t=>`- ${t.name}: ${t.description}`).join('\n');
		return `Available tools:\n${toolList}\nState:\n${JSON.stringify(this.state)}\nReturn JSON action.`;
	}

	async parseAction(text:string):Promise<Action>{
		try{ const m = text.match(/\{[\s\S]*\}/m); return JSON.parse(m?m[0]:text) as Action }catch(e){ return { action:'THINK', reason:'parse_error' } }
	}

	async run(userInput:string){
		this.state.lastUserInput = userInput; const history:any[] = [];
		for(let step=0; step<this.maxSteps; step++){
			const prompt = this.buildPrompt()+`\nHistory:${JSON.stringify(history)}\nUser:${userInput}`;
			const raw = await this.llmCall(prompt);
			const act = await this.parseAction(raw);
			if(act.action==='DONE') return { result:this.state, reason:act.reason };
			if(act.action==='CALL_TOOL' && act.tool && this.tools[act.tool]){
				let res; try{ res = await this.tools[act.tool].call(act.input,this.state) }catch(err){ res = { error:String(err) } }
				history.push({ step, action:act, result:res }); this.state[act.tool] = res; continue;
			}
			history.push({ step, action: act });
		}
		return { error:'max_steps_reached', state:this.state };
	}
}
```

## 7) Mẹo thực tế
- Bắt đầu với tập tool nhỏ (3–5 tools).
- Giữ temperature thấp cho các hành động điều khiển.
- Nén/tóm tắt history để tránh token blowup.
- Thêm human-in-the-loop cho hành động phá hoại.

## Kết luận & bước tiếp theo
- Thay vì cố gắng định nghĩa toàn bộ workflow tĩnh, hãy cung cấp cho LLM bộ công cụ và state; LLM sẽ quyết định chuỗi hành động theo runtime.

# Các bước thực hiện
1. Comment lại function "runWorkflow" trong workflowManager.ts để tạm thời vô hiệu hóa workflow tĩnh.
2. Chỉnh sửa lại prompt trong file english.ts và vietnamese.ts để không sử dụng "runWorkflow" nữa mà thay vào đó sử dụng kiến trúc agent mới.
3. Comment lại công cụ "runWorkflow" trong english.ts để tránh việc gọi công cụ này.
4. Sửa lại prompt để yêu cầu LLM trả về response dưới dạng:
{
  done: boolean
}
[Nội dung chính]
Bên cạnh đó, sửa prompt để gợi ý LLM một số workflow phổ biến mà LLM có thể áp dụng. Ví dụ:
- Workflow tìm kiếm thông tin về hội nghị: 
	- 1. Nhận yêu cầu từ người dùng
	- 2. Thử route đến conference agent
	- 3. Nếu không được, dùng công cụ retrieveKnowledge để tìm thông tin hội nghị
	- 4. Nếu không được, dịch yêu cầu người dùng sang tiếng Anh sau đó thử lại retrieveKnowledge
	- 5. Tổng hợp kết quả và trả về cho người dùng
(Bước này tạm thời chỉ mới chỉnh sửa english.ts, vietnamese.ts - Thêm step 2 và 3, sau này sẽ chỉnh sửa các file khác nếu cần)
5. Sửa lại hàm handleStreaming trong hostAgent.streaming.handler.ts để thực hiện vòng lặp agent:
	- Lấy input: history, config, system instruction, tools.
	- Gọi LLM với prompt mới để biết cần done hay chưa, để biết hành động kế tiếp là gì.
	- Dựa vào trường done, kết thúc hoặc không kết thúc. Thực hiện cơ chế phát hiện sớm trường done. Nếu như trường done là true thì kết thúc vòng lặp. Cơ chế phát hiện sớm trường done: 
		- Khi nhận chunk đầu, thêm vào buffer string
		- Trích xuất chuỗi JSON từ buffer string
		- Nếu parse thành công và có trường "done" -> chấp nhận
		- Nếu parse không thành công, tiếp tục thêm chunk vào buffer string cho đến khi đạt giới hạn ký tự (ví dụ 1000 ký tự), nếu vẫn không parse được -> fallback: 
			- Nếu có function call -> done = false
			- Nếu chỉ có partial token(vd: 'don', 'done: tr') -> done = false
   - Lặp lại cho đến khi nhận được DONE hoặc đạt giới hạn bước.

6. Sửa lại response của LLM để loại bỏ {done: boolean} ra khỏi phần content trả về cho người dùng cuối.

*Cập nhật 04/02/2026*:
1. Chỉnh sửa lại system instruction trong english.ts và vietnamese.ts để LLM không đính kèm {done: boolean} trong content trả về nữa. Khi thực hiện xong workflow, dùng function call "finishWorkflow" để báo cho hệ thống biết đã hoàn thành workflow.
2. Thêm function call "finishWorkflow" trong english.ts để báo cho hệ thống biết đã hoàn thành workflow(Nhớ sửa lại languageConfig.ts để đăng ký function này).
3. Sửa lại hàm handleStreaming trong hostAgent.streaming.handler.ts để kiểm tra function call "finishWorkflow" thay vì trường {done: boolean} trong content.
