"+ - If the current user query is not strongly related to the previous query or active result set, you MUST ask a clarification before using prior context: ask whether the user wants to start a completely new query or continue from the previous query/context."
-> Không hiểu ý đồ của câu prompt này là gì?

"+ - If the user wants to filter/sort/refine/paginate a list you just returned, do it from memory — do NOT route or call tools."
-> Lỡ chatbot trả về list không chứa thông tin cần thiết để filter hay sort thì sao? Ví dụ như người dùng yêu cầu filter theo rank mà list trước đó không có trường rank cho mỗi hội nghị -> phải route để gọi tool -> mâu thuẫn.\

"**RESULT SET SAVING — AUTO-SAVED FOR MODEL RESULTS**"
-> Sửa thành **RESULT SET SAVING — CONTROLLED BY HOST AGENT**

"Do NOT call \`saveResultSet\` for model-generated ConferenceAgent result lists. ConferenceAgent auto-saves real UUIDs from \`retrieveKnowledge\` for later ordinal references."
-> Sửa thành "You (HostAgent) must call \`saveResultSet\` to save these lists for later ordinal reference."

"-- Extract the conference identifiers from the result (can be IDs, titles, acronyms)
-- Call \`saveResultSet\` yourself with the list of identifiers
-- The identifiers MUST be passed in the correct order (first identifier = position 1, second identifier = position 2, etc.)
-- DO NOT pass identifiers in random order
-- The \`description\` parameter MUST be natural English prose describing the list content (e.g., "list of AI conferences for 2025", "user's shortlist of machine learning conferences"). **NEVER use slug/code formats like "user_relevant_conf", "search_results_1", or similar.**
-- Example: saveResultSet({ description: "list of top AI conferences", orderedIdentifiers: [{type: "acronym", value: "ICML"}, {type: "acronym", value: "NeurIPS"}, {type: "acronym", value: "AAAI"}], source: "model" })
+- Present the returned list directly to the user.
+- Do not manually save model-generated identifiers; this can duplicate state or save weaker acronyms instead of UUIDs."
-> Revert lại thay đổi này.

"-- When saving model-generated lists, use \`source: "model"\`
+- Do not save model-generated lists manually."
-> Revert lại thay đổi này.

"+**E1 — Filter existing list (Context-First)**

- History: You just returned a list of 10 conferences from a previous routeToAgent call.
- User: "filter the conference list by september"
- ✓ CORRECT: Filter from memory. Among the 10, only some match September. Return those.
- ✗ WRONG: Call routeToAgent or retrieveKnowledge again. The data is already in context.
- Why: Data already exists in conversation history. Filtering is a memory operation."
  -> Fix lại để nếu data đã có trong context thì không gọi routeToAgent, nhưng nếu data KHÔNG TỒN TẠI trong context(thiếu thông tin về thời gian), phải gọi retrieveKnowledge lại.

"+ - For conference list queries (\`listMode = true\`), expect a compact list payload: **id, title, acronym, date, location**. Full address, ranks, and status are NOT included in list mode to fit more results. Do **not** invent or expand missing long fields."

-> Có thể gây nhầm lẫn. Chatbot hoàn toàn CÓ THỂ expand thêm trường mới sử dụng tham số conferenceFields trong hàm retrieveKnowledge -> mâu thuẫn.

"- - If the user asks for personalized recommendations or "recommend for me": Route to 'ConferenceAgent'. 'taskDescription' = "Recommend conferences for the user." Include topics or a recent keyword if provided.

-     - If the user asks for generic personalized recommendations without constraints (e.g., "recommend me some conferences", "conferences I may like"): Route to 'ConferenceAgent'. 'taskDescription' = "Recommend conferences for the user. Use getRecommendations because the request has no topic/rank/location/date/type constraints."
-     - If the user asks for personalized recommendations with constraints (e.g., "recommend me AI conferences", "recommend rank B conferences", "recommend conferences in Vietnam", "suggest free-access conferences for me"): Route to 'ConferenceAgent'. 'taskDescription' = "Find personalized recommended conferences matching [constraints]. Use retrieveKnowledge with listMode=true and useRecommendationFilter=true. Do not use getRecommendations for this constrained request.""

"- - If the user asks to see more results: Route to 'ConferenceAgent'. 'taskDescription' = "Show more recommendations from the last list."

-     - If the user explicitly asks to see more recommendation or conference-list results from the previous list (e.g., "show more", "more conferences", "next page", "next set"): Route to 'ConferenceAgent'. 'taskDescription' = "Show more recommendations from the last list." Do NOT mention retrieveKnowledge in this continuation task."
  -> Các chỗ này mặc dù đúng ý nhưng mà sai format. Format của taskDescription coi lại trong file english.ts.

"-8. **RESULT SET SAVING — CONTROLLED BY HOST AGENT**
+8. **RESULT SET SAVING — AUTO-SAVED FOR MODEL RESULTS**"
-> Revert thay đổi này.

"- You (HostAgent) must call \`saveResultSet\` to save these lists for later ordinal reference.

- Do NOT call \`saveResultSet\` for model-generated ConferenceAgent result lists. ConferenceAgent auto-saves real UUIDs from \`retrieveKnowledge\` for later ordinal references."
  -> Revert thay đổi này.

"- - Extract the conference identifiers from the result (can be IDs, titles, acronyms)

- - Call \`saveResultSet\` yourself with the list of identifiers
- - The identifiers MUST be passed in the correct order (first identifier = position 1, second identifier = position 2, etc.)
- - DO NOT pass identifiers in random order
- - The \`description\` parameter MUST be natural English prose describing the list (e.g., "list of AI conferences for 2025", "user's shortlist of machine learning conferences"). **NEVER use slug/code formats like "user_relevant_conf", "search_results_1", or similar.**
- - Example: saveResultSet({ description: "list of top AI conferences", orderedIdentifiers: [{type: "acronym", value: "ICML"}, {type: "acronym", value: "NeurIPS"}, {type: "acronym", value: "AAAI"}], source: "model" })

* - Present the returned list directly to the user.
* - Do not manually save model-generated identifiers; this can duplicate state or save weaker acronyms instead of UUIDs."
    -> Revert thay đổi này.

"- - When saving model-generated lists, use \`source: "model"\`

- - Do not save model-generated lists manually."
    -> Revert thay đổi này.

"- \* **For list-style queries asking for conferences about a topic/keyword (e.g., "list of conferences about AI"), you MUST use Self-RAG via the 'retrieveKnowledge' tool with \`listMode = true\` and \`filter = { tableName: 'conferences' }\`.**

-        *   **In general, use the 'retrieveKnowledge' tool for conference search.** For list requests, keep the response compact (id, title, acronym, date, location, status).
-        *   Request detailed retrieval only when the user explicitly asks for detailed conference content.

*
*        *   **\`conferenceFields\` CONTROLS RESPONSE SIZE.** The HostAgent has a 2500-char limit for function responses. Using too many fields truncates the response and silently drops items.
*
*        *   **When \`listMode = true\` (broad list/search/browse):**
*            *   Set ONLY these fields in \`conferenceFields\`:
*                - Required: \`id: true\`, \`title: true\`, \`acronym: true\`
*                - Location: \`locations.cityStateProvince: true\`, \`locations.country: true\`
*                - Date: \`dates.fromDate: true\`, \`dates.toDate: true\`, \`dates.sortOrder: "desc"\`, \`dates.limit: 1\`
*            *   Set ALL other fields to \`false\`: \`status\`, \`address\`, \`dates.name\`, \`ranks\`, \`topics\`.
*            *   Use \`filter: { tableName: "conferences" }\`. Pass ADDITIONAL filter params (rank, startDate, endDate, country, continent, cityStateProvince, location, accessType) when the user specifies criteria.
*            *   For broad logged-in list searches, set \`useRecommendationFilter: true\` unless the task is a specific conference lookup or explicitly asks for unpersonalized/global results.
*            *   For constrained personalized recommendation requests, always call \`retrieveKnowledge\` with \`listMode: true\` and \`useRecommendationFilter: true\`. Examples: "recommend me AI conferences", "recommend rank B conferences", "recommend conferences in Vietnam", "suggest free-access conferences for me". Do NOT use \`getRecommendations\` for constrained recommendation requests.
*            *   For location-constrained recommendations, ALWAYS pass a structured location filter. Use \`filter.country\` for countries. Normalize "USA", "U.S.", "America", and "United States" to \`country: ["United States"]\`.
*            *   **Quick reference:**
*                | Scenario | listMode | filter | conferenceFields |
*                |----------|----------|--------|-----------------|
*                | Generic personalized recommendation | use getRecommendations | n/a | n/a |
*                | Constrained personalized recommendation | true | { constraints, tableName } | compact |
*                | List by topic | true | { rank?, tableName } | compact (no address/ranks) |
*                | List by date range | true | { startDate?, endDate?, tableName } | compact |
*                | List by rank | true | { rank, tableName } | compact |
*                | List/recommend by country | true | { country, tableName } | compact |
*                | Detail (1 conference) | false | { tableName } | ALL fields |
*                | Compare 2+ conferences | false | { tableName } | ALL fields |
*
*        *   **When \`listMode = false\` (detail / compare):**
*            *   Set ALL relevant \`conferenceFields\` to \`true\`: \`id\`, \`title\`, \`acronym\`, \`status\`, \`organizations.locations.address\`, full \`organizations.dates\`, \`ranks\`, \`topics\`.
*            *   For **compare 2 conferences**: call \`retrieveKnowledge\` separately for each item with \`listMode = false\`."
  -> Hơi nguy hiểm ở mấy chỗ này: Set ONLY these fields in \`conferenceFields\` sẽ làm cho ngay cả khi người dùng muốn show những thông tin mà họ tự chọn thì chatbot cũng sẽ chỉ làm theo ý system prompt. Bên cạnh đó filter theo location cũng khá nguy hiểm tại vì theo tui nhớ không lầm thì nó chỉ chống sai chính tả thôi. Lỡ mà trong data tên thành phố là "Thành phố Hồ Chí Minh" mà chatbot truyền vào filter là "TPHCM", "TP.HCM" hay "Sài Gòn" các kiểu là chết.
