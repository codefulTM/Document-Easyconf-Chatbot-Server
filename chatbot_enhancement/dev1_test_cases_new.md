# Test Cases cho Recommendation Pre-filter

> **Mục tiêu:** Test các flow thực tế có thể verify được
> **Dựa trên:** dev1_recommendation_implementation_plan.md

---

## Test Case 1: List Conference Query - Recommendation Filter Active

**Mô tả:** User query list conference - recommendation filter được kích hoạt theo mặc định

**Input:**

```
Tìm hội nghị AI
```

**Expected Output:**

**AI Response:**

```
Dưới đây là các hội nghị AI được gợi ý cho bạn:

1. ICML 2026 - International Conference on Machine Learning
   - Địa điểm: Vienna, Austria
   - Thời gian: July 2026

2. NeurIPS 2026 - Neural Information Processing Systems
   - Địa điểm: Vancouver, Canada
   - Thời gian: December 2026

3. AAAI 2026 - AAAI Conference on Artificial Intelligence
   - Địa điểm: Seattle, USA
   - Thời gian: February 2026
```

**Show Thought Process:**

```
1. Phân tích query: "Tìm hội nghị AI" → Đây là query list conference
2. Recommendation pre-filter được kích hoạt theo mặc định
3. Gọi API /for-you với perPage=100 để lấy personalized recommendations
4. Nhận được conference IDs từ recommendation API
5. Inject IDs vào filter
6. Skip entity dictionary matching (đã có pre-filter)
7. Thực hiện RAG retrieval với filter đã inject
8. Trả về kết quả từ pool personalized conferences
```

**Console Logs Expected:**

```
[RetrieveKnowledgeHandler] Pre-filtered by recommendation: { idsCount: 100, filter: { id: [...] } }
[RetrievalService] Skipping entity dictionary matching (pre-filtered by recommendation).
```

**Output:**

**Actual AI Response:**

```
Dưới đây là danh sách các hội nghị về Trí tuệ nhân tạo (AI):

ICITA: 13/10/2025 - 15/10/2025, Oslo, Na Uy
IE: 19/01/2026 - 26/01/2026, Singapore, Singapore
IJCNN: 29/06/2025 - 04/07/2025, Rome, Ý
ICAISC: 13/06/2026 - 17/06/2026, Zakopane, Ba Lan
ICTAI: 02/11/2025 - 04/11/2025, Athens, Hy Lạp
AIMSA: 17/09/2024 - 19/09/2024, Varna, Bulgaria
IC-AI: 14/12/2025, Tokyo, Nhật Bản
AISTATS: 02/05/2025 - 04/05/2025, Mai Khao, Thái Lan
EPIA: 30/09/2025 - 02/10/2025, Faro, Bồ Đào Nha
HCI: 09/11/2025 - 10/11/2025, Cardiff, Vương quốc Anh
```

**Actual Show Thought Process:**

```
Routing task to ConferenceAgent...
[ConferenceAgent] Searching knowledge base for: "AI conference" (limit=10)
[ConferenceAgent] Found 10 relevant conferences.
Saving conference result set...
```

**Comparison with Expected:**

- ✅ Chatbot returned a list of AI conferences as expected
- ❌ The actual conferences returned are different from expected (expected ICML 2026, NeurIPS 2026, AAAI 2026 but got ICITA, IE, IJCNN, etc.)
- ✅ The format is similar (list with dates and locations)
- ⚠️ The response does not appear to use recommendation filter (returns general AI conferences, not personalized recommendations)

**Screenshot:**
![Test 1 Response](test-1-complete.png)

---

## Test Case 2: Detail Query - No Recommendation Filter

**Mô tả:** User query detail về conference cụ thể (không phải list query) - recommendation filter không áp dụng

**Input:**

```
Xem thông tin ICML
```

**Expected Output:**

**AI Response:**

```
ICML (International Conference on Machine Learning) là một trong những hội nghị hàng đầu về Machine Learning:

**Thông tin chi tiết:**
- Tên đầy đủ: International Conference on Machine Learning
- Viết tắt: ICML
- Địa điểm: Vienna, Austria
- Thời gian: July 2026
- Website: https://icml.cc

**Chủ đề chính:**
- Machine Learning
- Deep Learning
- Reinforcement Learning
```

**Show Thought Process:**

```
1. Phân tích query: "Xem thông tin ICML" → Detail query (không phải list)
2. Recommendation filter không áp dụng cho detail query
3. Không gọi recommendation API
4. Entity dictionary matching để xác định "ICML"
5. RAG retrieval để lấy thông tin chi tiết
6. Trả về thông tin chi tiết về ICML
```

**Console Logs Expected:**

```
Không có log về recommendation pre-filter
```

**Output:**

**Actual AI Response:**

```
Thông tin chi tiết về hội nghị ICML (International Conference on Machine Learning):

Thời gian: 13/07/2025 - 19/07/2025
Địa điểm: Vancouver Convention Center, Vancouver, Canada
Xếp hạng (năm 2026): A*
```

**Actual Show Thought Process:**

```
Routing task to ConferenceAgent...
[ConferenceAgent] Searching knowledge base for: "ICML" (limit=5)
[ConferenceAgent] Found 2 relevant conferences.
```

**Comparison with Expected:**

- ✅ Chatbot returned detailed information about ICML as expected
- ❌ The actual details differ from expected (expected Vienna, Austria, July 2026 but got Vancouver, Canada, July 2025)
- ✅ The format is similar (detailed info with time, location, ranking)
- ✅ No recommendation filter was applied (as expected for detail query)

**Screenshot:**
![Test 2 Response](test-2-response-complete.png)
![Test 2 Thought Process](test-2-thought-process.png)

---

## Test Case 3: Non-Conference Query - No Recommendation Filter

**Mô tả:** User query không liên quan đến conference table - recommendation filter không áp dụng

**Input:**

```
Giải thích khái niệm overfitting trong machine learning
```

**Expected Output:**

**AI Response:**

```
**Overfitting (Quá khớp)** là một vấn đề phổ biến trong Machine Learning:

**Định nghĩa:**
Overfitting xảy ra khi mô hình học quá kỹ dữ liệu training, bao gồm cả noise và outliers.

**Nguyên nhân:**
- Mô hình quá phức tạp so với dữ liệu
- Dữ liệu training quá ít

**Giải pháp:**
1. Regularization (L1, L2, Dropout)
2. Early stopping
3. Data augmentation
```

**Show Thought Process:**

```
1. Phân tích query: "Giải thích khái niệm overfitting" → Không phải conference query
2. isConferenceTableQuery=false
3. Recommendation filter không áp dụng cho non-conference query
4. RAG retrieval từ knowledge base
5. Trả về giải thích về overfitting
```

**Console Logs Expected:**

```
Không có log về recommendation pre-filter
```

**Output:**

**Actual AI Response:**

```
Yêu cầu này nằm ngoài phạm vi trợ lý hội nghị. Vui lòng đặt câu hỏi liên quan đến hội nghị.
```

**Actual Show Thought Process:**
(N/A - No thought process shown as the query was rejected)

**Comparison with Expected:**

- ❌ Chatbot did NOT provide explanation about overfitting as expected
- ✅ Chatbot correctly identified this is outside the conference assistant scope
- ⚠️ The expected output assumed the chatbot would answer general ML questions, but the actual implementation restricts to conference-related queries only
- ✅ No recommendation filter was applied (as expected for non-conference query)

**Screenshot:**
![Test 3 Response](test-3-complete.png)

---

## Test Case 4: Comparison Query - No Recommendation Filter

**Mô tả:** Query so sánh (không phải list) - recommendation filter không áp dụng

**Input:**

```
So sánh ICML và NeurIPS
```

**Expected Output:**

**AI Response:**

```
Dưới đây là so sánh giữa ICML và NeurIPS:

**ICML (International Conference on Machine Learning):**
- Tập trung vào Machine Learning
- Thường tổ chức vào tháng 7
- Là hội nghị hàng đầu về ML

**NeurIPS (Neural Information Processing Systems):**
- Tập trung vào Neural Networks và Deep Learning
- Thường tổ chức vào tháng 12
- Là hội nghị hàng đầu về Deep Learning

**Khác biệt chính:**
- ICML rộng hơn về Machine Learning nói chung
- NeurIPS chuyên sâu hơn về Neural Networks
```

**Show Thought Process:**

```
1. Phân tích query: "So sánh ICML và NeurIPS" → Comparison query (không phải list)
2. Recommendation filter không áp dụng cho comparison query
3. Entity dictionary matching để xác định "ICML" và "NeurIPS"
4. RAG retrieval để lấy thông tin so sánh
5. Trả về kết quả so sánh
```

**Console Logs Expected:**

```
Không có log về recommendation pre-filter
```

**Output:**

**Actual AI Response:**

```
Dưới đây là bảng so sánh giữa ICML và NeurIPS dựa trên thông tin năm 2025:

Đặc điểm	ICML (International Conference on Machine Learning)	NeurIPS (Advances in Neural Information Processing Systems)
Thời gian	13/07/2025 - 19/07/2025	02/12/2025 - 07/12/2025
Địa điểm	Vancouver, Canada	San Diego, California, Hoa Kỳ
Xếp hạng (2026)	A*	A*

Cả hai đều là những hội nghị hàng đầu thế giới về học máy và trí tuệ nhân tạo, được đánh giá ở mức cao nhất (A*). Trong khi ICML diễn ra vào giữa năm tại Canada, thì NeurIPS được tổ chức vào cuối năm tại Hoa Kỳ.
```

**Actual Show Thought Process:**

```
Routing task to ConferenceAgent...
[ConferenceAgent] Searching knowledge base for: "ICML and NeurIPS" (limit=2)
[ConferenceAgent] Found 1 relevant conferences.
```

**Comparison with Expected:**

- ✅ Chatbot returned a comparison between ICML and NeurIPS as expected
- ✅ The format is a table format (different from expected bullet points but still clear)
- ✅ Key information included: time, location, ranking
- ❌ The actual locations differ from expected (expected Vienna/Vancouver but got Vancouver/San Diego)
- ✅ No recommendation filter was applied (as expected for comparison query)

**Screenshot:**
![Test 4 Response](test-4-result.png)
![Test 4 Thought Process](test-4-thought-process.png)

---

## Test Case 5: List Query with Entity Name - Entity Dictionary Skipped

**Mô tả:** List query có tên conference cụ thể - entity dictionary bị skip khi recommendation filter active

**Input:**

```
Tìm hội nghị ICML
```

**Expected Output:**

**AI Response:**

```
Dưới đây là các hội nghị được gợi ý cho bạn:

1. ICML 2026 - International Conference on Machine Learning
   - Địa điểm: Vienna, Austria
   - Thời gian: July 2026

2. NeurIPS 2026 - Neural Information Processing Systems
   - Địa điểm: Vancouver, Canada
   - Thời gian: December 2026

3. AAAI 2026 - AAAI Conference on Artificial Intelligence
   - Địa điểm: Seattle, USA
   - Thời gian: February 2026
```

**Show Thought Process:**

```
1. Phân tích query: "Tìm hội nghị ICML" → List conference query (có entity name)
2. Recommendation pre-filter được kích hoạt theo mặc định
3. Gọi API /for-you
4. Nhận IDs từ recommendation API
5. Inject filter
6. Skip entity dictionary (dù có "ICML" trong query)
7. RAG retrieval với filter đã inject
8. Trả về kết quả từ pool personalized IDs
```

**Console Logs Expected:**

```
[RetrieveKnowledgeHandler] Pre-filtered by recommendation: { idsCount: 100, filter: { id: [...] } }
[RetrievalService] Skipping entity dictionary matching (pre-filtered by recommendation).
```

**Output:**

**Actual AI Response:**

```
ICML 2025 (Hội nghị Quốc tế về Học máy lần thứ 42) là hội nghị hàng đầu thế giới dành cho các chuyên gia trong lĩnh vực học máy. Dưới đây là thông tin chi tiết:

Thời gian: 13/07/2025 - 19/07/2025
Địa điểm: Trung tâm Hội nghị Vancouver, Vancouver, Canada
Hình thức: Trực tiếp
Xếp hạng: A* (năm 2026)
Các chủ đề chính: Học máy tổng quát, Học sâu (Deep Learning), Lý thuyết học máy, Hệ thống học máy, Tối ưu hóa, Các phương pháp xác suất, Học tăng cường (Reinforcement Learning), Học máy tin cậy và Học máy dựa trên ứng dụng.

ICML là nơi quy tụ cộng đồng đa dạng gồm các nhà nghiên cứu học thuật và công nghiệp, các doanh nhân, kỹ sư và sinh viên để trình bày các nghiên cứu tiên tiến nhất.
```

**Actual Show Thought Process:**

```
Routing task to ConferenceAgent...
[ConferenceAgent] Searching knowledge base for: "ICML" (limit=5)
[ConferenceAgent] Found 2 relevant conferences.
```

**Comparison with Expected:**

- ❌ Chatbot returned detailed information about ICML instead of a list of recommended conferences
- ⚠️ The query "Tìm hội nghị ICML" was interpreted as a detail query rather than a list query with entity
- ✅ The information provided is accurate and detailed
- ⚠️ It's unclear if recommendation filter was applied (behavior differs from expected)

**Screenshot:**
![Test 5 Response](test-5-final.png)
![Test 5 Thought Process](test-5-thought-process.png)

---

## Test Case 6: Multiple Sequential List Queries

**Mô tả:** Test behavior khi user gửi nhiều list query liên tiếp

**Input 1:**

```
Tìm hội nghị AI
```

**Expected Output 1:**

```
Dưới đây là các hội nghị AI được gợi ý cho bạn:
[3 conferences từ recommendation pool]
```

**Input 2:**

```
Tìm hội nghị NLP
```

**Expected Output 2:**

```
Dưới đây là các hội nghị NLP được gợi ý cho bạn:
[3 conferences từ recommendation pool]
```

**Show Thought Process:**

```
Query 1:
1. Phân tích: List conference query
2. Recommendation pre-filter được kích hoạt theo mặc định
3. Gọi API /for-you
4. Nhận IDs
5. Inject filter và skip entity dictionary
6. RAG retrieval
7. Trả về kết quả

Query 2:
1. Phân tích: List conference query khác
2. Recommendation pre-filter được kích hoạt lại
3. Gọi API /for-you
4. Nhận IDs
5. Inject filter mới và skip entity dictionary
6. RAG retrieval
7. Trả về kết quả mới
```

**Console Logs Expected:**

```
[RetrieveKnowledgeHandler] Pre-filtered by recommendation: { idsCount: 100, filter: { id: [...] } }
[RetrievalService] Skipping entity dictionary matching (pre-filtered by recommendation).
[x2 cho 2 queries]
```

**Output:**

**Actual AI Response 1 (Tìm hội nghị AI):**

```
Dưới đây là danh sách các hội nghị về Trí tuệ nhân tạo (AI):

IC-AI: 14/12/2025, Tokyo, Nhật Bản
IE: 19/01/2026 - 26/01/2026, Singapore, Singapore
ICTAI: 02/11/2025 - 04/11/2025, Athens, Hy Lạp
EPIA: 30/09/2025 - 02/10/2025, Faro, Bồ Đào Nha
AIMSA: 17/09/2024 - 19/09/2024, Varna, Bulgaria
ICITA: 13/10/2025 - 15/10/2025, Oslo, Na Uy
ISAIM: 06/01/2026 - 08/01/2026, Fort Lauderdale, Florida, Hoa Kỳ
AISTATS: 02/05/2025 - 04/05/2025, Mai Khao, Thái Lan
IJCNN: 29/06/2025 - 04/07/2025, Rome, Ý
ICAISC: 13/06/2026 - 17/06/2026, Zakopane, Ba Lan
```

**Actual AI Response 2 (Tìm hội nghị NLP):**

```
Hiện tại tôi không tìm thấy danh sách hội nghị tập trung chuyên biệt vào NLP, nhưng dưới đây là một số hội nghị công nghệ có các chủ đề liên quan chặt chẽ đến trí tuệ nhân tạo và xử lý dữ liệu:

ICAART: 04/03/2026 - 06/03/2026, Marbella, Tây Ban Nha
PoEM: 02/12/2025 - 04/12/2025, Geneva, Thụy Sĩ
WORLDCOMP: 20/07/2025 - 23/07/2025
IJCNN: 29/06/2025 - 04/07/2025, Rome, Ý
ICAISC: 13/06/2026 - 17/06/2026, Zakopane, Ba Lan
ISDA: 30/07/2025, Settat, Ma-rốc
ACMMM: 26/10/2025 - 30/10/2025, Dublin, Ireland
ICITA: 13/10/2025 - 15/10/2025, Oslo, Na Uy
ICLR: 23/04/2025 - 27/04/2025, Singapore, Singapore
HCI: 09/11/2025 - 10/11/2025, Cardiff, Vương quốc Anh
```

**Actual Show Thought Process:**

```
Query 1: [ConferenceAgent] Searching knowledge base for: "AI conference" (limit=10)
Query 2: [ConferenceAgent] Searching knowledge base for: "NLP" (limit=10)
```

**Comparison with Expected:**

- ✅ Both queries returned lists of conferences as expected
- ❌ The conferences returned are general AI/tech conferences, not from a personalized recommendation pool
- ⚠️ The second query for NLP returned conferences with AI/data processing topics rather than pure NLP conferences
- ⚠️ It's unclear if recommendation filter was being applied for each query

**Screenshot:**
![Test 6 Query 1](test-6-query1-final.png)
![Test 6 Query 2](test-6-query2-complete.png)

---

## Summary Checklist

### Recommendation Filter Active Tests

- [x] Test Case 1: List Conference Query - Recommendation Filter Active
- [x] Test Case 5: List Query with Entity Name - Entity Dictionary Skipped
- [x] Test Case 6: Multiple Sequential List Queries

### No Filter Tests

- [x] Test Case 2: Detail Query - No Recommendation Filter
- [x] Test Case 3: Non-Conference Query - No Recommendation Filter
- [x] Test Case 4: Comparison Query - No Recommendation Filter

### Total: 6 Test Cases

---

## Notes

**Các test case đã loại bỏ (không thực tế):**

- API Failure Fallback: Không thể tạo điều kiện API fail một cách determinstic
- Empty Response: Không biết input nào làm response empty
- Topics Extraction: Plan không còn truyền topics parameter nữa
- Entity Dictionary Skip Confirmation: Không thể verify từ UI (chỉ xem console log)
- Recent Keyword: Không thể verify từ UI
- Less Than 100 Recommendations: Không thể tạo điều kiện determinstic
- Special Characters: Edge case không quan trọng
- Empty Query: Vô nghĩa, user sẽ không gửi query rỗng
