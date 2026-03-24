## 🛠️ Các cách tối ưu RAM trong Qdrant

---

### 1️⃣ Dùng `on_disk` cho vectors (quan trọng nhất)

```json
// Khi tạo collection
{
  "vectors": {
    "size": 1536,
    "distance": "Cosine",
    "on_disk": true // vectors lưu trên disk, không load hết vào RAM
  }
}
```

- RAM giảm đáng kể, chỉ cache phần đang dùng
- Latency tăng nhẹ (~vài ms), chấp nhận được với 160k vectors

---

### 2️⃣ Dùng `memmap_threshold` để kiểm soát khi nào dùng mmap

```yaml
# config/config.yaml của Qdrant
storage:
  memmap_threshold_kb: 20000 # Segments > 20MB sẽ dùng mmap thay vì RAM
```

---

### 3️⃣ Scalar Quantization (giảm ~4x RAM)

```json
{
  "quantization_config": {
    "scalar": {
      "type": "int8", // float32 (4 bytes) -> int8 (1 byte)
      "quantile": 0.99,
      "always_ram": false // quantized vectors cũng lưu disk
    }
  }
}
```

$$\text{RAM: } 984\text{ MB} \xrightarrow{\text{int8}} \approx \textbf{246 MB}$$

Độ chính xác giảm rất ít (~1-2% recall).

---

### 4️⃣ Product Quantization (giảm RAM mạnh hơn nữa)

```json
{
  "quantization_config": {
    "product": {
      "compression": "x16", // x4, x8, x16, x32, x64
      "always_ram": false
    }
  }
}
```

- `x16` compress 16x nhưng recall giảm nhiều hơn
- Nên test recall trước khi dùng

---

### 5️⃣ Tối ưu payload (metadata)

- Chỉ index các field payload **thực sự cần filter**:

```json
{
  "field_name": "tableName",
  "field_schema": "keyword"
}
```

- Payload không index không chiếm RAM index

---

### 6️⃣ Giảm HNSW graph size

```json
{
  "hnsw_config": {
    "m": 8, // default 16, giảm xuống 8 -> giảm ~30% RAM graph
    "ef_construct": 64 // trade-off: build chậm hơn nhưng RAM ít hơn
  }
}
```

---

## 📊 Tổng kết tiết kiệm RAM

| Kỹ thuật          | RAM tiết kiệm                 |
| ----------------- | ----------------------------- |
| `on_disk: true`   | ~70-80% (chỉ cache hot data)  |
| Scalar int8       | ~75% vector RAM               |
| `m=8` thay `m=16` | ~30% HNSW graph               |
| Kết hợp cả 3      | **< 300 MB** cho 160k vectors |

---

> Với project của bạn, khuyến nghị dùng `on_disk: true` + `scalar int8` là đủ — đơn giản, hiệu quả, latency không ảnh hưởng đáng kể.
