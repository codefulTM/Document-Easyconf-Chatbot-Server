# P0-01: Qdrant mmap on-disk và memory budget

## Mục tiêu

- Ngăn OOM và giữ retrieval ổn định trên VPS RAM thấp.
- Đảm bảo Qdrant hoạt động với profile production và có cảnh báo memory pressure.

## Phạm vi thực hiện

- Qdrant collection dùng on-disk mmap (disk-backed) thay vì load toàn bộ vào RAM.
- Cấu hình memory limit cho container và Qdrant service.
- Triển khai health check + alert level cho memory usage.
- Tạo tài liệu vận hành (runbook) và dashboard monitor.

## Kế hoạch thực hiện

### 1. Hiểu hiện trạng và đánh giá baseline

- Kiểm tra cấu hình Qdrant hiện tại (config file, docker-compose, env vars): `qdrant/`.
- Lấy số liệu baseline: RAM sử dụng, p95 retrieval latency.
  - Bật Prometheus metric exporter để thu thập:
  - Qdrant hiện có sẵn endpoint `/metrics` trên cổng HTTP (6333), không cần cổng 9647 riêng.
  - Xác nhận trong `docker-compose` bằng cách map cổng 6333:
    `- "6333:6333"`
  - Trong Prometheus config:

    ```yaml
    - job_name: "qdrant"
       static_configs:
       - targets: ["qdrant:6333"]
    ```

  - Bước chi tiết lấy mẫu:
  1. Chạy workload chuẩn (nhiều truy vấn hoặc băng thông cần kiểm tra).
  2. Kiểm tra RAM:
  - Truy vấn `qdrant_memory_resident_bytes` (Qdrant native) hoặc `process_resident_memory_bytes`.
  - Nếu dùng cAdvisor/cadvisor-exporter: `container_memory_usage_bytes{container_label_com_docker_compose_service="qdrant-old"}`.
  - Lấy giá trị max/avg ở kpi cần (ví dụ 1m, 5m), tức là dùng window theo thời gian:
  - `max_over_time(qdrant_memory_resident_bytes[5m])` (max trong 5 phút)
  - `avg_over_time(qdrant_memory_resident_bytes[1m])` (trung bình trong 1 phút)
  3. Kiểm tra p95 latency: - Truy vấn:
     `histogram_quantile(
  0.95, 
  sum by (le) (
    rate(qdrant_rest_responses_duration_seconds_bucket{action="search"}[1m])
  )
)`(p95 latency trong 1 phút) - Xếp hàng chạy 1-3 phút và ghi lại 95th percentile.

  4. So sánh trước/sau cấu hình mmap để chứng minh hiệu quả.

- Đặt cấu hình tương ứng (theo config.yaml chính thức):
  - `QDRANT__STORAGE__STORAGE_PATH=/qdrant/storage`
  - `QDRANT__STORAGE__SNAPSHOTS_PATH=/qdrant/storage/snapshots`
  - `QDRANT__STORAGE__WAL__WAL_CAPACITY_MB=64`
  - `QDRANT__STORAGE__WAL__WAL_SEGMENTS_AHEAD=1`
  - `QDRANT__STORAGE__ON_DISK_PAYLOAD=true`
  - `QDRANT__SERVICE__HOST=0.0.0.0`
  - `QDRANT__SERVICE__HTTP_PORT=6333`

### 2. Cấu hình Qdrant production profile

<!-- - Bật `ON_DISK`/`MMAP` qua biến môi trường trong `docker-compose.qdrant.yml` (ví dụ `QDRANT__STORAGE__ON_DISK=true`, `QDRANT__STORAGE__MMAP_THRESHOLD_KB=20`). -->

- Bật HNSW on-disk: `QDRANT__COLLECTION__VECTORS__HNSW_INDEX__ON_DISK=true`.
<!-- - Nếu bạn có file config YAML tĩnh thì chỉ cần set `storage.type = "mmap"` / `on_disk = true` / `mmap_path` ở đó. -->

- Cấu hình collection với on_disk đầy đủ:

  ```js
  import { QdrantClient } from "@qdrant/js-client-rest";
  const client = new QdrantClient({ host: "localhost", port: 6333 });

  await client.createCollection("full_mmap_collection", {
    vectors: {
      size: 1536,
      distance: "Cosine",
      on_disk: true, // Ép dùng mmap cho vector
    },
    hnsw_config: {
      on_disk: true, // Ép dùng mmap cho index đồ thị
      // Chỉnh threshold ở đây (đơn vị bằng bytes)
      mmap_threshold: 20480, // 20KB
    },
    optimizers_config: {
      // Ngưỡng để chuyển từ RAM sang mmap cho các segment vector
      // Nếu set = 0, sẽ cố gắng mmap ngay khi có thể.
      memmap_threshold: 20480,
    },
    on_disk_payload: true,
  });
  ```

<!-- - Chỉnh:
  - `QDRANT__WAL__ENABLE=true` (bật WAL để an toàn dữ liệu)
  - `QDRANT__WAL__PATH=/qdrant/wal` (thư mục WAL nên cùng volume `qdrant_storage`)
  - `QDRANT__SNAPSHOT__INTERVAL=5m` (hoặc giá trị hợp lý với IO và recovery) -->

### 3. Thiết lập memory limit cứng cho container

- Set `--memory` Docker compose cho service `qdrant`. Ở đây cho giới hạn memory là 1.5GB (tùy theo RAM VPS, có thể điều chỉnh thấp hơn nếu cần).
- Bật log chi tiết cho Qdrant:
  - `QDRANT__SERVICE__LOG_LEVEL=debug` trong `docker-compose.qdrant.yml`.
  - hoặc command run: `docker compose -f docker-compose.qdrant.yml run -e QDRANT__SERVICE__LOG_LEVEL=debug qdrant`.
- Kiểm tra log container để bắt memory warning sớm:
  - `docker compose -f docker-compose.qdrant.yml logs -f qdrant`.
  - tìm các từ khóa: `memory`, `oom`, `gc`, `warn`, `error`.

### 4. Health check và cảnh báo memory pressure

- Implement health check endpoint trong service gateway/chabot:
  - Prime check: Qdrant API `GET /collections/{name}/stats`.
  - Kiểm tra
    - `ram_usage_bytes`: RAM dùng thực tế của Qdrant (tránh quá tải cgroup).
    - `disk_usage_bytes`: dung lượng lưu trữ đang dùng (tránh disk full).
    - `query_count`: tổng số truy vấn đã xử lý (throughput cơ bản).
    - `qdrant_collection_points_count`: số điểm trong collection (quy mô dữ liệu).
    - `qdrant_collection_segments_count`: số segment đang load (ảnh hưởng search/optimize).
    - `qdrant_query_failures_total`: số query lỗi (sức khỏe service).
    - `qdrant_gc_duration_seconds` (nếu có): thời gian GC(Garbage Collection), spike = memory pressure(gọi GC với duration càng dài thì càng báo hiệu memory pressure càng cao).
    - `qdrant_oom_kills_total` / `container_memory_failures_total`: event OOM.
    - `qdrant_open_file_descriptors`: số file descriptor mở (mmap cần FD đủ lớn). "mở" ở đây là kết nối với file ở kernel, sẵn sàng thao tác, không phải là nạp file vào RAM.
    - `container_memory_swap`: swap container đã dùng. Khác với `disk_usage_bytes` - dùng disk để chứa data, `container_memory_swap` là dùng disk để mở rộng RAM.
    - `qdrant_up`: service có alive (1/0).
    - `qdrant_requests_in_flight`: số request đang xử lý tại một thời điểm.
- Tạo một cron job để định kỳ kiểm tra health check endpoint và log kết quả. Đồng thời, tạo alert rule như sau:
  - Cảnh báo `ram_usage_bytes > 80% limit`.
  - Cảnh báo `qdrant_oom_kills_total > 0` hoặc `container_memory_failures_total > 0`.
  - Cảnh báo `qdrant_open_file_descriptors > 90% * max_open_files = 90% * 512` hoặc trên ngưỡng tối đa.
  - Cảnh báo `qdrant_requests_in_flight > threshold = số nhân CPU * 1.5` (throttle load).
  - Cảnh báo `qdrant_disk_usage_bytes` gần đầy volume(> 75% volume).

## Deliverables

- `docs/qdrant_mmap.md` (tài liệu tổng hợp).
- Cấu hình `docker-compose.yml` và `qdrant` profile đã cập nhật.
- Runbook memory threshold + alert playbook.
- Dashboard Grafana, alert rules Prometheus.

## Definition of Done (DoD)

- Qdrant chạy ổn định dưới limit memory, không bị OOM kill.
- p95 retrieval latency giữ trong ngưỡng chấp nhận.
- Cảnh báo memory pressure hoạt động và có quy trình phản ứng.
- Tài liệu vận hành đầy đủ.

## Kết quả thực hiện

### Kết quả bước 1 (Hiểu hiện trạng và đánh giá baseline)

- Các chỉ số được lấy khi chạy các câu lệnh search liên tục trong 5 phút, với 10000+ vector trong collection và 384 chiều embedding.

- Trước khi bật mmap:
  - Max RAM usage trong 5 phút: 146825216 bytes
  - Avg RAM usage trong 5 phút: 146123161.6 bytes
  - p95 retrieval latency trong 5 phút: 0.004793676470588235 seconds < 100 ms

- Sau khi bật mmap:
  - Max RAM usage trong 5 phút: 143966208 bytes
  - Avg RAM usage trong 5 phút: 142423654.4 bytes
  - p95 retrieval latency trong 5 phút: 0.004797752247752248 seconds < 100 ms
