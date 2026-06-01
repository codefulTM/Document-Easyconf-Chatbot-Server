# Chung

- Note: Hùng làm 3, 4; TMinh làm 5.

<!-- 1. Thêm vào DB bảng phụ nhiều nhiều liên kết giữa conference id và user id. Có thêm một cột nữa là thứ tự của conference id xuất hiện trong recommend_user:... trong Redis. Sau đó, trong hàm retrieveKnowledge, thực hiện query trên bảng này để filter hội nghị luôn thay vì dùng API trên BE.(Tạm thời bỏ qua do hong biết có cần hong) -->

3. Handler showMoreRecommendations đang bị lặp code trong retrieveKnowledge. -> sửa.
<!-- 3. Khi chạy file main_job.py để cập nhật dữ liệu trên Redis thì đồng thời thay đổi dữ liệu trên bảng phụ nhiều nhiều.(Tạm thời bỏ qua do hong biết có cần hong) -->
4. Check lại xem có truncate response của host agent / sub agent không. -> Chỉ được yêu cầu chatbot trả về độ dài x kí tự, KHÔNG ĐƯỢC TỰ Ý cắt bỏ string. Người dùng thà thấy một response dài(trường hợp LLM không tuân theo prompt) còn hơn thấy một response bị cắt ngang giữa chừng.
5. Kiểm tra lại những chỗ gọi lưu result set cho các conference list trong code. -> Không được gọi thẳng trong code mà phải để AI gọi.

# retrieveKnowledge.handler.ts

- Note: Hùng làm 1, 2, 3, 4. TMinh làm 5, 6, 7

1. Tạm thời comment bước Áp dụng filter vị trí suy luận từ query.

   Phân tích `applyInferredLocationFilter()` — chỉ dùng regexp để suy luận quốc gia từ query, rất fragile:
   - **False positive**:
     - `\bchina\b` match cả "china" (đồ sứ) trong "china cabinet", "china pattern".
     - `\bamerican\b` match "American Express", "American Standard" dù không liên quan quốc gia.

   - **False negative (tiếng Việt)**: Không có pattern nào cho:
     "Mỹ", "Anh", "Pháp", "Đức", "Ý", "Tây Ban Nha", "Hàn Quốc", "Trung Quốc",
     "Nhật Bản", "Úc", "Nga", "Thái Lan", "Ấn Độ", "Hà Lan", "Bỉ", "Thụy Sĩ",
     "Hàn", "Nhật", "Trung", "Hoa Kỳ", "Anh Quốc"…

   - **Chỉ 13 quốc gia** — USA, UK, Vietnam, Singapore, Japan, China, Canada,
     Australia, India, Germany, France, Italy, Spain. Thiếu rất nhiều:
     Hàn Quốc, Indonesia, Malaysia, Brazil, Mexico, Netherlands, Thụy Điển,
     Na Uy, Đan Mạch, Đài Loan, UAE, Ả Rập Xê Út, Argentina, v.v.

   - **First-match wins** — Dùng `for` loop, gặp pattern đầu là return ngay.
     Query "hội nghị ở Mỹ và Nhật Bản" → chỉ trả về "United States", bỏ qua Japan.

   - **Không handle context** — "Tôi không muốn hội nghị ở Mỹ" → vẫn gắn
     `country: ["United States"]` dù user đang phủ định.

   - **normalizeCountryAlias cũng thiếu** — Chỉ 12 alias cơ bản (usa, us, uk, …),
     không map được "japan" → `normalizeCountryAlias("japan")` trả về undefined,
     giữ nguyên "japan" nhờ fallback `|| value` (may mà inferCountryFromQuery
     có pattern cho Japan nên query path tạm ổn, nhưng filter path thì không).

   → Nên dùng NLP/NER thay vì regex tay, hoặc bỏ hẳn bước suy luận này.

2. Logic xác định `listMode` — 4 lớp heuristic chồng chéo, đấu đá nhau.

   ```typescript
   const inferredListMode = this.shouldUseListMode(query, effectiveFilter);
   const explicitListMode =
     typeof effectiveArgs.listMode === "boolean"
       ? effectiveArgs.listMode
       : undefined;
   const hasSpecificConferenceFilter =
     this.hasExplicitConferenceScope(effectiveFilter);
   const listModeFromRouter =
     typeof explicitListMode === "boolean"
       ? explicitListMode || (!hasSpecificConferenceFilter && inferredListMode)
       : inferredListMode;
   const isSpecificConferenceIntent = this.isSpecificConferenceIntent(
     query,
     effectiveFilter,
   );
   const listMode =
     typeof effectiveArgs.listMode === "boolean" &&
     effectiveArgs.listMode === true
       ? true
       : isSpecificConferenceIntent
         ? false
         : listModeFromRouter;
   ```

   **Vấn đề:**
   - **Circular dependency**: `isSpecificConferenceIntent()` (dòng 2936) gọi lại `shouldUseListMode()`. Nếu `shouldUseListMode` trả về `true` → `isSpecificConferenceIntent` lập tức trả về `false`. Lớp bảo vệ "chặn list mode khi hỏi detail" chỉ active khi heuristic list mode đã trả về false → vô dụng đúng lúc cần nhất.

   - **`extractConferenceHints` false positive**:
     - `"show me conference at USA"` → "USA" là acronym 3 ký tự không thuộc `TOPIC_ACRONYMS` → hint = `["USA"]`. Query có "conference" (số ít), không có "conferences"/"các hội nghị"/"danh sách" → `isSpecificConferenceIntent` = `true` → tắt list mode sai. User chỉ muốn list conference ở Mỹ.
     - Quote: `"hội nghị về 'học sâu'"` → hint = `["học sâu"]`, query có "hội nghị" (số ít), không có "các hội nghị"/"danh sách" → detail intent override → list mode tắt oan.

   - **`effectiveArgs.listMode === false` bị phớt lờ**: Nếu model nói `listMode = false` (explicit), nhưng heuristic `listModeFromRouter` ra `true` → heuristic ghi đè quyết định của model. Comment bảo "giữ nguyên" nhưng logic không làm vậy.

   - **`"danh sách"` bị hardcode là plural keyword** (dòng 2944-2946): `"cho tôi danh sách thông tin chi tiết hội nghị XYZ"` → có "danh sách" → `hasPluralConferenceKeyword = true` → `isSpecificConferenceIntent = false`, dù user thực ra muốn detail 1 hội nghị.

   → **Nên đơn giản hóa**: Chỉ tin vào `explicitListMode` từ model, bỏ hết heuristic.

3. effectiveFilter = this.applyInferredLocationFilter(query, effectiveFilter); -> Check coi apply đống filter này vào thì bên retrieve() đã thêm phần xử lý mấy filter này chưa? Nếu chưa thì thêm vào. filter nằm trong cả keyword search và vector search.
4. `executeRecommendAndReturnIds()` vs `executeGetRecommendationsForUser()` — có thể gộp.

   Cả hai đều gọi `callRecommendationsForYou()` bên trong. `executeGetRecommendationsForUser` đã có sẵn `page` và `perPage` params, nên hoàn toàn có thể dùng trong loop pagination.

   So sánh:

   | Tính năng   | `executeGetRecommendationsForUser` | `executeRecommendAndReturnIds` |
   | ----------- | ---------------------------------- | ------------------------------ |
   | Pagination  | ✅ caller tự loop                  | ✅ loop sẵn bên trong          |
   | Dedup ID    | ❌ caller tự làm                   | ✅ dùng Set                    |
   | Return type | `{ items: ApiItem[] }`             | `{ ids: string[] }`            |

   `executeRecommendAndReturnIds` chỉ là wrapper tiện lợi: gộp pagination loop + dedup + extract ID vào một lần gọi. Phần pagination và dedup đều có thể tự làm ở caller với `executeGetRecommendationsForUser`.

   => **Không cần thiết**, có thể xoá và xử lý pagination + dedup trực tiếp ở handler.

5. const candidateMatch = await this.buildRecommendationCandidateIds({
   query,
   filter: effectiveFilter,
   recommendationPoolIds,
   listMode,
   usePoolAsCandidates,
   resultLimit: limit,
   });

-> Chỗ này tại sao cần gọi hàm này mà không truyền vào mảng chứa các recommendation ids ngay từ đầu?

- buildRecommendationCandidateIds() đang khá dư thừa vì có thể truyền thẳng recommendationPoolIds vào retrieve() rồi giao với filter sau.
- Bên trong hàm này lại gọi retrieve() nhiều lần và còn tách thêm nhiều nhánh con như structured filter, topic RAG, strict topic alias, fallback RAG, khiến luồng rất nặng.
- Trong khi mục tiêu thực tế có vẻ chỉ là lấy giao giữa danh sách recommendation gốc với các điều kiện filter/topic, việc này có thể tự làm bằng vài dòng map/filter/Set.
- retrieve() và keywordSearch() vốn đã có lọc DB rồi, nên phần logic bổ sung trong buildRecommendationCandidateIds() cần được xem lại vì có dấu hiệu trùng lặp.

6.  const candidateIds = candidateMatch.candidateIds;
    const finalMatch = this.fillByRecommendationChunks(
    recommendationPoolIds,
    candidateIds,
    limit,
    );
    const continuationMatch = this.fillByRecommendationChunks(
    recommendationPoolIds,
    candidateIds,
    recommendationPoolIds.length,
    );
    const finalIds = finalMatch.finalIds;
    const outsidePoolDroppedCount = this.countOutsideRecommendationPool(
    recommendationPoolIds,
    candidateIds,
    );

- Đoạn này đang lấy giao giữa recommendationPoolIds và candidateIds thêm một lần nữa, trong khi candidateIds về lý thuyết đã được sinh ra từ chính recommendationPoolIds ở bước trước.
- fillByRecommendationChunks() bị gọi 2 lần với cùng một input chính, chỉ khác limit, nên phần giao tập ở đây trông như đang lặp lại logic kiểm tra phạm vi.
- Nếu invariant của candidateMatch.candidateIds đã đúng, thì bước countOutsideRecommendationPool() và việc giao lại với recommendationPoolIds có dấu hiệu dư thừa.
- Cần xem lại có thật sự cần một lớp phòng thủ nữa ở đây hay không, vì hiện tại luồng đang làm rối và khó theo dõi hơn cần thiết.

7. Điều kiện này:
   if (finalIds.length === 0 || orderedFinalResults.length > 0) {

- Hơi bị vô nghĩa đúng không??? Chứng minh:
  finalIds.length === 0 || orderedFinalResults.length > 0
  <=> orderedFinalResults.length === 0 || orderedFinalResults.length > 0 (vì các id trong orderedFinalResults là tập con của finalIds)
  <=> orderedFinalResults.length >= 0 (luôn đúng vì độ dài của mảng luôn >= 0)
  => Điều kiện luôn xảy ra. Mà luôn xảy ra thì thêm vào làm gì á?

# english.ts

- Note: File này thì lúc Hùng + TMinh fix xong tất cả các lỗi trên thì mới sửa.

- Sửa lại prompt để cứ mỗi lần filter muốn filter theo thông tin gì đó thì -> include trường đó trong conferenceFields để LLM bắt buộc phải đọc lại những kết quả mình đã lọc và verify lại cho người dùng -> kết hợp cả tầng code và tầng LLM để lọc -> tăng độ chính xác. cái thứ hai là phải thêm vào support cho trường continent trong filter, do hiện tại truyền vào hàm thì có truyền mà thân hàm ignore cũng như không.
- Còn lại coi thêm trong english.ts.patch.report.md và english.ts.patch

# Idea: hostAgent.streaming.handler.ts

- Note: File này thì lúc Hùng + TMinh fix xong tất cả các lỗi trên thì mới sửa.

- Khi subagent yêu cầu hàm retrieveKnowledge với params truyền vào là listMode = true -> đặt cờ shouldSaveResultSet = true
- Khi LLM muốn kết thúc workflow -> chạy code để check xem cờ shouldSaveResultSet có bật không. Nếu bật thì check coi trong các turn của LLM có gọi hàm saveResultSet lần nào hay chưa. Nếu chưa thì gửi thêm một turn nữa cho chatbot yêu cầu nó gọi hàm saveResultSet. Lặp lại đến khi nào hàm saveResultSet được gọi rồi mới thôi.
