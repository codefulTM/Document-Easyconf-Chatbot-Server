Google ADK làm việc tương tự OpenAI Function Calling: LLM **không tự suy luận kiểu dữ liệu từ code Python**, mà ADK sẽ chuyển function của bạn thành một **JSON Schema** rồi gửi schema đó cho Gemini. Gemini dựa vào schema này để biết tham số nào bắt buộc, kiểu dữ liệu gì, object có những field nào, v.v. ([Google GitHub][1])

Ví dụ đơn giản:

```python
def create_user(
    name: str,
    age: int,
    email: str | None = None
):
    """Create a user."""
```

ADK sẽ sinh schema gần giống:

```json
{
  "type": "object",
  "properties": {
    "name": { "type": "string" },
    "age": { "type": "integer" },
    "email": { "type": "string" }
  },
  "required": ["name", "age"]
}
```

Gemini nhìn schema này và biết:

* `name` bắt buộc, kiểu string
* `age` bắt buộc, kiểu integer
* `email` optional, kiểu string

([Google GitHub][1])

---

### Với object phức tạp

Thường dùng `Pydantic BaseModel`:

```python
from pydantic import BaseModel

class UserInfo(BaseModel):
    name: str
    age: int
    address: str | None = None

def create_user(user: UserInfo):
    """Create user."""
```

Pydantic sẽ tạo JSON Schema dạng:

```json
{
  "type": "object",
  "properties": {
    "name": {
      "type": "string"
    },
    "age": {
      "type": "integer"
    },
    "address": {
      "type": "string"
    }
  },
  "required": ["name", "age"]
}
```

ADK hỗ trợ sử dụng Pydantic model làm tham số tool và sinh schema từ model đó. ([GitHub][2])

---

### Thêm mô tả cho từng field

```python
from pydantic import BaseModel, Field

class UserInfo(BaseModel):
    name: str = Field(
        description="User full name"
    )
    age: int = Field(
        description="User age in years"
    )
    address: str | None = Field(
        default=None,
        description="Home address"
    )
```

Mô tả này cũng được đưa vào schema và Gemini sử dụng để hiểu ý nghĩa của từng trường tốt hơn. ([Medium][3])

---

### Điều gì xảy ra khi Gemini gọi tool?

Gemini sẽ trả về:

```json
{
  "user": {
    "name": "Minh",
    "age": 22
  }
}
```

Sau đó ADK dùng Pydantic để parse và validate:

```python
UserInfo(
    name="Minh",
    age=22
)
```

Nếu Gemini trả:

```json
{
  "user": {
    "name": "Minh",
    "age": "abc"
  }
}
```

thì validation sẽ fail vì `age` không phải số. ([Pydantic][4])

---

Nói ngắn gọn:

**Function signature + type hints + Pydantic model → JSON Schema → Gemini nhìn schema → Gemini sinh arguments đúng cấu trúc → ADK/Pydantic validate trước khi gọi tool.** ([Google GitHub][1])

Nếu bạn đang dùng Google ADK Python, mình có thể cho ví dụ đầy đủ với nested object, list, enum, optional field và xem chính xác schema JSON mà ADK tạo ra.

[1]: https://google.github.io/adk-docs/tools-custom/function-tools/?utm_source=chatgpt.com "Overview - Agent Development Kit (ADK)"
[2]: https://github.com/google/adk-python/issues/55?utm_source=chatgpt.com "Union types don't work? · Issue #55 · google/adk-python"
[3]: https://medium.com/%40gururaser/structured-outputs-with-tools-google-agent-development-kit-adk-136b94be0576?utm_source=chatgpt.com "Structured Outputs with Tools | Google ADK"
[4]: https://pydantic.dev/docs/validation/2.4/concepts/models/?utm_source=chatgpt.com "Models | Pydantic Docs"
