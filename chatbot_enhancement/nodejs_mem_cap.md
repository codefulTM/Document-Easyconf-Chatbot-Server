# Node.js Memory Cap & Guard Runtime

## Feature P0-02: Node.js memory cap và guard runtime

Mục tiêu: khống chế heap để tránh crash ngẫu nhiên do rò rỉ bộ nhớ, bổ sung cảnh báo và snapshot("bản chụp nhanh" trạng thái hiện tại của hệ thống) khi sử dụng bộ nhớ vượt ngưỡng.

## Step by step thực hiện

1. Định nghĩa cấu hình memory cap(giới hạn bộ nhớ) cho tiến trình chatbot.
   - Thêm biến môi trường cấu hình: `CHATBOT_MAX_OLD_SPACE_SIZE_MB`, `CHATBOT_MEMORY_WARNING_PCT`, `CHATBOT_MEMORY_CRITICAL_PCT`.
     - `CHATBOT_MAX_OLD_SPACE_SIZE_MB`: giới hạn heap của Node.js, dùng với `--max-old-space-size`.
     - `CHATBOT_MEMORY_WARNING_PCT`: ngưỡng phần trăm heap usage để ghi cảnh báo sớm.
     - `CHATBOT_MEMORY_CRITICAL_PCT`: ngưỡng phần trăm heap usage để kích hoạt snapshot/error và phản ứng nhanh.
   - Đặt giá trị mặc định an toàn, ví dụ: `1024` MB cho `CHATBOT_MAX_OLD_SPACE_SIZE_MB`, `70`% cho cảnh báo, `85`% cho nguy hiểm. Ước tính bằng cách:
     - đo peak heap hiện tại bằng cách chạy load test cho vector search(chạy file `chunks-qdrant-search.ts`) sau đó nhân peak heap của vector search với 2(để dự phòng cho các module chatbot, crawl).
       - Note: Sau khi chạy load test, nhận được kết quả: Peak heapUsed during run: 847 MB -> 847 \* 2 = 1694 MB < 2048 MB -> chặn trên là 2048 MB hay 2 GB.
     - đảm bảo tổng RAM server vẫn đủ cho OS và các process khác

2. Cập nhật lệnh khởi chạy Node.js.
   - Sửa `package.json`, Dockerfile hoặc tập lệnh startup để thêm tham số `--max-old-space-size=${CHATBOT_MAX_OLD_SPACE_SIZE_MB}`.
   - Đảm bảo môi trường production và dev đều dùng cùng ngưỡng cấu hình nếu cần.

3. Bổ sung runtime watcher theo heap usage — kết luận và hướng phải chuyển sang Child Process.

   Sau thử nghiệm và phân tích, lưu ý quan trọng:
   - Việc "search retained size của một hàm/object" trực tiếp trong heap snapshot mà không dùng Chrome DevTools rất khó và đắt đỏ về tài nguyên. Heap snapshot là JSON lớn, và để tính retained size đúng phải xây dominator tree hoặc dùng APIs của DevTools — cả hai đều tốn CPU / mem khi chạy trên cùng tiến trình.
   - Nếu bạn chạy phân tích snapshot trong cùng tiến trình đang bị memory pressure, hành động phân tích có thể làm tình trạng tệ hơn hoặc không khả thi. Độ chính xác khi tìm hàm/object còn phụ thuộc vào chuỗi tên trong snapshot (constructor name, function name, closure labels), và một số đối tượng native khó map trực tiếp.

   Kết luận: để phân tích retained-size an toàn và đáng tin cậy, bắt buộc phải tạo tiến trình tách rời (Child Process) để chạy module nghi ngờ — sau đó chụp heap snapshot của tiến trình con hoặc đo trực tiếp PID memory usage. Do đó ta chuyển hướng từ "tìm retained size trong same-process" sang "chạy module trong Child Process để đo và chụp snapshot".
   - Tách Child Process:
     - Nhóm 1: Các tác vụ liên quan đến chatbot(trong file hostAgent.\_\_\_.handler.ts) như xử lý message, streaming, quản lý context, v.v.
     - Nhóm 2: Các tác vụ sync(ConferenceSyncService)
     - Nhóm 3: Các tác vụ cào dữ liệu
   - Tách thành child process — ý tưởng chính (đặt ở đây để gần phần MemoryWatcher):

   - Mục tiêu: chạy các module/ tác vụ có khả năng tiêu thụ nhiều bộ nhớ hoặc chặn CPU trong tiến trình con riêng để dễ giám sát, chụp snapshot, và khởi động lại cục bộ.
   - Khi tách: chọn module/ tác vụ thoải mái tách rời (ví dụ: xử lý file lớn, download, index vector, phân tích snapshot, task tính toán nặng, hoặc thư viện native chiếm nhiều RAM).
   - Cách triển khai nhanh:
     - Dùng `child_process.fork()` để spawn tiến trình con chạy module với `MODULE_ID` trong `env`.
     - Giao tiếp qua IPC (`child.send()` / `process.on('message')`) để gửi lệnh `prepareDrain`, request snapshot, và nhận trạng thái/ lỗi.
     - Trong tiến trình con: ưu tiên viết heap snapshot cục bộ (ví dụ `heapdump`), trả metadata về parent, rồi exit an toàn.
     - Parent: theo dõi các PID bằng `pidusage` (hoặc `/proc`) để tính `totalObservedMemory` và gắn pid → module context.
   - Vấn đề vận hành:
     - Áp exponential backoff khi restart child để tránh crash-loop.
     - Lưu artifacts (snapshot, logs) ngay sau khi child exit vì heap snapshot trong child dễ phân tích mà không làm áp lực cho parent.
   - Lợi ích tóm tắt: giảm rủi ro crash toàn cục, dễ xác định thủ phạm, cho phép chụp heap snapshot an toàn và restart cục bộ.
   1. MemoryWatcher vẫn sample `process.memoryUsage()` tổng thể định kỳ (20s), **và phải cộng thêm bộ nhớ của tất cả child process đang được quản lý**.
   - Cụ thể: MemoryWatcher giữ danh sách PID của child process do `forkManager`/`fork` tạo. Mỗi sample, ngoài `process.memoryUsage().heapUsed` của tiến trình cha, gọi `pidusage` (hoặc đọc `/proc/<pid>/status`) cho từng PID để lấy `memory` (RSS) và `heap` nếu có thể; sau đó tính `totalObservedMemory = parentHeapUsed + sum(childRSS or childHeapEstimate)`.
   - So sánh `totalObservedMemory / heap_size_limit` với ngưỡng `warnPct`/`critPct` để quyết định warning/critical.
   - Khi child process đang chạy workload riêng (được spawn để đo module hoặc để thực thi module nặng), MemoryWatcher phải gắn `child.pid` vào context module tương ứng để khi child vượt ngưỡng chỉ trả về alert cho module đó.
   2. CHỈ KHI `totalObservedMemory` VƯỢT `criticalPct`:
   - Tạo `observedPerProcess`: một bản đồ theo PID chứa memory của tiến trình cha (ví dụ `heapUsed`) và memory của các tiến trình con (RSS hoặc ước lượng heap). Dữ liệu này lấy bằng `pidusage` hoặc đọc `/proc/<pid>/status`.
   - Xác định `maxObserved`: tiến trình (cha hoặc con) có giá trị heap/RSS lớn nhất trong `observedPerProcess`.
   - Chỉ thực hiện graceful shutdown trên tiến trình đó:
     - Nếu là tiến trình con: gửi `{ cmd: 'prepareDrain' }`, chờ timeout để các tác vụ in-flight hoàn thành, sau đó gửi `SIGTERM` (và `SIGKILL` nếu vẫn không dừng).
     - Nếu là tiến trình cha: thử các biện pháp giảm tải nội bộ; nếu không khả thi thì mới nâng lên khởi động lại toàn bộ.
   - Sau tắt thành công: khởi động lại tiến trình tương ứng (áp dụng exponential backoff nếu cần) và lưu artifacts (heap snapshot, logs, v.v.) để phục vụ postmortem.

   Mục đích: chỉ can thiệp vào tiến trình gây ra tiêu thụ bộ nhớ lớn nhất, giảm rủi ro ảnh hưởng đến toàn hệ thống và tập trung việc khắc phục vào thủ phạm thực tế.

---

**Chọn process để restart — dùng MAD (Median Absolute Deviation)**

Trong thống kê, để tìm "mức bình thường" một cách robust (không bị kéo lệch bởi vài giá trị lớn), ta dùng MAD — dựa trên Trung vị (Median).

Ví dụ nhanh:

- Giả sử các process có heap (MB): `20, 20, 21, 22, 45`.
- Bước 1 — Median (M): sắp xếp và lấy giá trị giữa → `21`.
- Bước 2 — Khoảng cách tuyệt đối: `|x - M|` → `1,1,0,1,24`.
- Bước 3 — MAD = median(khoảng cách) → `1`.
- Quy tắc outlier: `|x - M| > 3 * MAD` ⇒ nếu đúng thì coi `x` là ngoại lai và là ứng viên restart.

Áp dụng: `45` có deviation `24` và `24 > 3 * 1` → `45` là outlier → restart.

Ghi chú vận hành:

- Lấy mẫu trên một window (ví dụ 3–5 sample hoặc sliding window 20s) để tránh quyết định dựa trên spike ngắn.
- Nếu `MAD == 0`, dùng epsilon (ví dụ 0.1MB) hoặc chuyển sang percentile-based rule để tránh chia 0.
- Ưu tiên restart child trước; nếu parent là outlier, cố gắng mitigation in-process rồi mới escalate.
- Kết hợp với hysteresis và exponential backoff để tránh crash-loop.
- Trước restart, chụp heap snapshot và ghi log chi tiết cho postmortem.

---

Ví dụ triển khai (Chỉ phương án B - fork + monitor + graceful restart):

Bước 1 — File worker: `download-worker.js`

```javascript
// download-worker.js
const fs = require("fs");

const MODULE_ID = process.env.MODULE_ID || "download-worker";

process.on("message", async (data) => {
  if (data && data.cmd === "prepareDrain") {
    // finish in-flight quickly, then exit
    await new Promise((r) =>
      setTimeout(r, Math.min(data.timeoutMs || 5000, 5000)),
    );
    process.exit(0);
  }

  try {
    const { filePath } = data;
    const fileContent = fs.readFileSync(filePath); // heavy op for demo
    // optionally write heap snapshot here using heapdump or inspector
    // const heapdump = require('heapdump'); heapdump.writeSnapshot(`/tmp/${MODULE_ID}-${Date.now()}.heapsnapshot`);
    process.send({ status: "done", size: fileContent.length });
    process.exit(0);
  } catch (err) {
    process.send({ status: "error", error: String(err) });
    process.exit(1);
  }
});
```

Bước 2 — Parent: fork + monitor + graceful restart

```javascript
// parent.js
const { fork } = require("child_process");
const pidusage = require("pidusage");

// simple ForkMap for pid -> module context
const ForkMap = new Map();

function startWorker(filePath, moduleId = "download-worker") {
  const child = fork("./download-worker.js", {
    env: { MODULE_ID: moduleId },
  });
  ForkMap.set(child.pid, { moduleId, child });

  child.send({ filePath });

  const checker = setInterval(async () => {
    try {
      const stats = await pidusage(child.pid);
      if (stats.memory > 200 * 1024 * 1024) {
        // request graceful drain
        child.send({ cmd: "prepareDrain", timeoutMs: 5000 });
        setTimeout(() => {
          try {
            child.kill("SIGTERM");
          } catch (e) {}
        }, 7000);
        clearInterval(checker);
      }
    } catch (e) {
      clearInterval(checker); // child likely exited
    }
  }, 1000);

  child.on("message", (msg) => console.log("child message", msg));
  child.on("exit", () => {
    ForkMap.delete(child.pid); /* restart/backoff here */
  });

  return child;
}
```

4. Thêm log cảnh báo và snapshot ngắn.
   - Khi sử dụng vượt `warning` threshold, ghi log warning chi tiết: heapUsed, heapTotal, RSS, external.
   - Khi vượt `critical` threshold, ghi log snapshot ngắn của hiện trạng bộ nhớ.

   Giải thích các trường memory (để dễ phân tích):
   - `RSS` ([Resident Set Size]): tổng bộ nhớ vật lý hiện đang được tiến trình chiếm (heap + stack + native allocations + mmaped files). Đây là con số hữu ích để biết tiến trình đang dùng bao nhiêu RAM trên hệ thống.
   - `external`: bộ nhớ được cấp phát bởi các thành phần native (C/C++) liên kết với đối tượng JS — ví dụ `Buffer`, `ArrayBuffer` và các native addon; không phải là heap JS, nhưng vẫn chiếm RAM thực.
   - `heapUsed`: dung lượng heap JS đang dùng (byte).
   - `heapTotal`: dung lượng heap JS hiện đã được V8 cấp phát (byte).
   - Snapshot ngắn có thể lưu: `process.memoryUsage()`, `process.cpuUsage()`, `process.uptime()`, trạng thái các queue/worker nếu có.

5. Kích hoạt cơ chế phản ứng nhanh.
   - Với mức `critical`:
     - Ưu tiên: chỉ restart module bị phình [module: một thành phần chức năng có thể tách ra chạy độc lập] thay vì restart toàn bộ tiến trình.
     - Áp exponential backoff cho restart module để tránh crash-loop [crash-loop: restart liên tục do lỗi chưa fix] — `exponential backoff` [tăng dần thời gian chờ giữa các lần thử theo hàm mũ để tránh vòng lặp khởi động lại nhanh].

## Gợi ý triển khai

- Nếu dùng Kubernetes hoặc Docker, cấu hình `resources.limits.memory` tương ứng với giá trị `--max-old-space-size`.
- Nếu có healthcheck, hãy bổ sung kiểm tra `memory pressure` và trả `unhealthy` khi vượt ngưỡng nguy hiểm.
- Đảm bảo watcher không gây quá nhiều overhead và chỉ log chi tiết khi vượt ngưỡng.

---

## Q: Cho tôi hỏi nếu như socket server nhận sự kiện 'send_message' từ client, rồi gọi hàm handleStreaming() trong hostAgent.\_\_\_.handler.ts nhưng mà hàm đó yêu cầu gửi child.send() đến child process VectorStoreService thì làm sao để tôi nhận kết quả trả về từ child process để mà tiếp tục chạy phần còn lại của hàm handleStreaming?

A:
Khi bạn tách `VectorStoreService` ra một **Child Process** (tiến trình con) riêng, việc gọi hàm không còn là "gọi và đợi" (synchronous) thông thường nữa. Vì các tiến trình nằm ở các vùng nhớ khác nhau, bạn phải sử dụng cơ chế **IPC (Inter-Process Communication)** (giao tiếp giữa các tiến trình) dựa trên sự kiện.

Để nhận kết quả trả về trong hàm `handleStreaming` mà không làm hỏng logic của bạn, cách tốt nhất là sử dụng **Promise** (một đối tượng đại diện cho kết quả của một tác vụ bất đồng bộ trong tương lai) kết hợp với `async/await`.

Dưới đây là cách triển khai chi tiết:

### 1. Tại Child Process (VectorStoreService)

Tiến trình này cần lắng nghe tin nhắn, xử lý và gửi ngược kết quả lại cho tiến trình cha kèm theo một mã định danh (`id`) để tiến trình cha biết kết quả này là của yêu cầu nào.

```typescript
// Trong file vector-store-worker.ts
process.on("message", async (msg: any) => {
  if (msg.type === "VECTOR_SEARCH") {
    const result = await vectorStoreService.search(msg.data);

    // Gửi trả kết quả về kèm ID của yêu cầu ban đầu
    process.send({
      type: "VECTOR_SEARCH_RESULT",
      id: msg.id,
      data: result,
    });
  }
});
```

### 2. Tại Main Process (Nơi chạy Socket & HostAgent)

Bạn cần một cơ chế để "đợi" phản hồi từ tiến trình con. Chúng ta sẽ dùng một `Map` để lưu trữ các hàm `resolve` (hàm kết thúc một Promise thành công).

```typescript
import { fork, ChildProcess } from "child_process";
import { v4 as uuidv4 } from "uuid"; // Thư viện tạo ID duy nhất

const vectorWorker = fork("vector-store-worker.ts");
const pendingRequests = new Map<string, (value: any) => void>();

// Lắng nghe mọi phản hồi từ worker
vectorWorker.on("message", (msg: any) => {
  if (msg.type === "VECTOR_SEARCH_RESULT") {
    const resolve = pendingRequests.get(msg.id);
    if (resolve) {
      resolve(msg.data); // Kích hoạt Promise đang đợi
      pendingRequests.delete(msg.id); // Xóa sau khi xong
    }
  }
});

// Hàm wrapper để gọi sang worker theo kiểu async/await
function callVectorStore(data: any): Promise<any> {
  return new Promise((resolve) => {
    const id = uuidv4(); // Tạo mã định danh cho yêu cầu này
    pendingRequests.set(id, resolve);
    vectorWorker.send({ type: "VECTOR_SEARCH", id, data });
  });
}
```

### 3. Trong hàm `handleStreaming` của bạn

Bây giờ, hàm của bạn có thể viết cực kỳ gọn gàng giống như đang gọi hàm bình thường:

```typescript
// hostAgent.___.handler.ts
export async function handleStreaming(payload: any) {
  try {
    // Gọi sang child process và đợi kết quả
    // Đây là lúc Node.js tạm dừng hàm này để làm việc khác, không gây nghẽn
    const vectorData = await callVectorStore(payload.query);

    // Sau khi có kết quả từ child process, hàm tiếp tục chạy tại đây
    console.log("Kết quả từ Vector Process:", vectorData);

    // Tiếp tục logic streaming của bạn...
    // socket.emit(...)
  } catch (error) {
    console.error("Lỗi khi gọi Vector Process", error);
  }
}
```

### Tại sao nên dùng cách này?

1.  **Non-blocking (Không gây nghẽn):** Trong khi chờ `VectorStoreService` xử lý, **Event Loop** (vòng lặp sự kiện) của API server vẫn rảnh để nhận thêm các sự kiện `send_message` khác từ người dùng.
2.  **Tính tuần tự:** Code của bạn vẫn trông giống như chạy từ trên xuống dưới, dễ đọc và bảo trì.
3.  **An toàn:** Việc dùng `id` (UUID) đảm bảo rằng nếu 10 người dùng nhắn tin cùng lúc, kết quả của người A sẽ không bị trả nhầm cho người B.

**Lưu ý nhỏ:** Nếu bạn dùng **Tsyringe**, bạn có thể đóng gói logic `callVectorStore` này vào một class, rồi `inject` class đó vào `HostAgent`. Khi đó, `HostAgent` thậm chí không cần biết là nó đang gọi một Child Process ở bên dưới!
