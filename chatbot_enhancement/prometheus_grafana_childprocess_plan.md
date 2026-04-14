# Kế hoạch tích hợp Prometheus + Grafana (Option B: Aggregator)

## Tóm tắt

Kế hoạch này chuyển sang Option B: mọi child không mở port `/metrics` riêng mà gửi metrics qua IPC (process.send) tới parent; parent thu thập/biên dịch các metric, expose một endpoint `/metrics` duy nhất mà Prometheus scrape. Mục tiêu: giảm surface area (một target cho Prometheus), tránh mở nhiều port và đơn giản hoá discovery trong môi trường Docker / Docker Compose.

## Mục tiêu

- **Tập trung**: Prometheus scrape chỉ endpoint của parent.
- **Low-overhead**: child gửi snapshot metric nhẹ qua IPC (JSON), không dùng prom-client HTTP endpoint ở mỗi child.
- **Labeling**: metrics aggregated sẽ có nhãn `pid`, `worker`, `env` để phân biệt nguồn.
- **Extensibility**: hỗ trợ cả default Node.js metrics và vài custom metrics (requests, tasks).

## Giả định

- Parent và child đều chạy Node.js; parent đã có HTTP server sẵn.
- Có thể thay đổi `src/workers/childProcessManager.ts` để inject/forward IPC payloads.
- Môi trường deploy: docker-compose (phiên bản development) và có thể mở rộng cho k8s (Service Discovery không cần cho children).

## Kiến trúc (Option B — Aggregator)

- Child processes: thu thập local snapshots (ví dụ `process.memoryUsage()`, `process.cpuUsage()`, custom counters) và gửi định kỳ qua `process.send({ type: 'METRICS', payload })`.
- Parent: giữ một `prom-client` `Registry` + các `Gauges`/`Counters` per-metric, cập nhật các giá trị khi nhận IPC từ child. Parent expose `/metrics` (prom-client) cho Prometheus.
- Prometheus: scrape chỉ endpoint parent (ví dụ `parent:9500/metrics`).

## Implementation steps (chi tiết)

1. Cài đặt dependency ở parent (và devDependency nếu cần ở child):

```bash
npm install prom-client
```

2. Child: thu thập metrics và gửi IPC (ví dụ `src/child/metrics-ipc.ts`)

```js
// src/child/metrics-ipc.ts
const INTERVAL_MS = Number(process.env.METRICS_INTERVAL_MS || 10000);

function collectSnapshot() {
  const mem = process.memoryUsage();
  const cpu = process.cpuUsage();
  // các bộ đếm tuỳ chỉnh do child quản lý
  const custom = {
    requestsProcessed: global.__requestsProcessed || 0,
  };
  return { pid: process.pid, mem, cpu, custom, ts: Date.now() };
}

setInterval(() => {
  if (process.send)
    process.send({
      type: "METRICS",
      payload: collectSnapshot(),
      worker: process.env.WORKER_NAME,
    });
}, INTERVAL_MS);

// Xuất helper để có thể tăng bộ đếm từ các module khác trong child
module.exports = { collectSnapshot };
```

Ghi chú:

- Giữ payload nhỏ (chỉ số dạng số). Tránh gửi toàn bộ text `register.metrics()` từ mỗi child — việc phân tích/gom hợp sẽ phức tạp và tốn băng thông IPC.

3. Parent: bộ thu (aggregator) và endpoint `/metrics` (ví dụ `src/parent/metrics-aggregator.ts`)

```js
const client = require("prom-client");
const register = new client.Registry();

// Định nghĩa các Gauge/counter mà parent sẽ cập nhật theo dữ liệu từ child
const gaugeHeapUsed = new client.Gauge({
  name: "easyconf_child_heap_used_bytes",
  help: "heapUsed (bytes) của child",
  labelNames: ["pid", "worker", "env"],
});
const gaugeRss = new client.Gauge({
  name: "easyconf_child_rss_bytes",
  help: "rss (bytes) của child",
  labelNames: ["pid", "worker", "env"],
});
const gaugeRequests = new client.Gauge({
  name: "easyconf_child_requests_total",
  help: "số requests đã xử lý (ước lượng)",
  labelNames: ["pid", "worker", "env"],
});

register.registerMetric(gaugeHeapUsed);
register.registerMetric(gaugeRss);
register.registerMetric(gaugeRequests);

function handleChildMetrics(message) {
  // Chỉ xử lý message dạng METRICS
  if (message?.type !== "METRICS") return;
  const { pid, mem, custom } = message.payload;
  const labels = {
    pid: String(pid),
    worker: message.worker || message.payload.worker || "child",
    env: process.env.NODE_ENV || "dev",
  };
  gaugeHeapUsed.set(labels, mem.heapUsed);
  gaugeRss.set(labels, mem.rss);
  if (typeof custom?.requestsProcessed === "number") {
    gaugeRequests.set(labels, custom.requestsProcessed);
  }
}

// Kết nối hàm này với sự kiện 'message' của child trong `childProcessManager`

// Expose /metrics: parent HTTP server nên trả về `register.metrics()`
```

Ghi chú tích hợp: `register` chỉ chứa các metric được parent đăng ký (aggregated); parent có thể đồng thời đăng các default metrics cho chính nó bằng `collectDefaultMetrics({ register })`. 4. `childProcessManager.ts`: forward child messages to aggregator

- When forking children, add a single `child.on('message', msg => metricsAggregator.handleChildMetrics(msg))` handler. Use a single dispatcher to avoid listener leaks.
- Ensure children get an env var `WORKER_NAME` (or `MODULE`) so parent can label metrics.

5. Cấu hình Prometheus (chỉ scrape parent)

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "easyconf_parent"
    metrics_path: /metrics
    static_configs:
      - targets: ["parent:9500"]
```

6. Docker Compose (dev)

```yaml
version: "3.8"
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"

  parent:
    build: .
    command: node dist/parent.js
    environment:
      - NODE_ENV=development
    ports:
      - "9500:9500"
```

7. Grafana

- Data source: Prometheus → `http://prometheus:9090`.
- Dashboard panels: `easyconf_child_heap_used_bytes`, `easyconf_child_rss_bytes`, `easyconf_child_requests_total` with template variable `pid` or `worker`.

8. Kiểm thử & xác thực

- Khởi parent và các child; kiểm tra rằng `http://parent:9500/metrics` chứa các metric có nhãn `pid`/`worker`.
- Dùng `DEBUG_CHILD_SPAWN=1` để phát hiện các fork bất thường nếu cần.
- Dùng lệnh sau để kiểm tra định dạng Prometheus:

```bash
curl http://localhost:9500/metrics
```

9. Bảo mật & vận hành

- Chỉ mở endpoint `/metrics` của parent ra mạng nội bộ (không public internet).
- Hạn chế cardinality của nhãn (không đưa các ID động như request-id thành label).

## Files cần xem / cập nhật

- [Code/Document-Easyconf-Chatbot-Server/src/workers/childProcessManager.ts](Code/Document-Easyconf-Chatbot-Server/src/workers/childProcessManager.ts)
- [Code/Document-Easyconf-Chatbot-Server/src/child/metrics-ipc.ts](Code/Document-Easyconf-Chatbot-Server/src/child/metrics-ipc.ts) (gợi ý tạo file)
- [Code/Document-Easyconf-Chatbot-Server/src/parent/metrics-aggregator.ts](Code/Document-Easyconf-Chatbot-Server/src/parent/metrics-aggregator.ts) (gợi ý tạo file)
- `prometheus.yml`, `docker-compose.yml` (cập nhật để scrape parent)

## Ước lượng thời gian

- PR mẫu (child IPC + parent aggregator): 2–4 giờ
- Prometheus + Grafana (dev): 1–2 giờ
- Kiểm thử + rollout: 1–2 giờ

## Next steps (đề xuất)

1. Tôi sẽ tạo PR mẫu với:

- `src/child/metrics-ipc.ts` (child sender),
- `src/parent/metrics-aggregator.ts` (parent registry + handler),
- sửa `childProcessManager.ts` để hook `child.on('message', ...)` vào aggregator.

2. Cập nhật `prometheus.yml` và `docker-compose.yml` dev để scrape parent.
3. Chạy local thử nghiệm: build + start parent, spawn vài child, kiểm tra `http://localhost:9500/metrics`.

Nếu bạn đồng ý, tôi sẽ tiếp tục với bước (1) và mở PR mẫu.
