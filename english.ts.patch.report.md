# Thay đổi 1
```patch
-2. **TOOL-AWARE USAGE** — Each tool's function declaration documents every parameter it accepts. Read it. \`retrieveKnowledge\` supports: query, limit, listMode, conferenceFields, filter (rank, accessType, startDate, endDate, country, continent, and more), and useRecommendationFilter (defaults to true for list searches). Use ALL parameters relevant to the user's request. For list/search/browse queries, ALWAYS set listMode=true so the system can personalize results.
+2. **TOOL-AWARE USAGE** — Each tool's function declaration documents every parameter it accepts. Read it. \`retrieveKnowledge\` supports: query, limit, listMode, conferenceFields, and filter with fields: rank, startDate, endDate, tableName, and more. Use the parameters that match the request, and if a needed field is missing from a compact list response, request the missing data instead of inventing it. E.g., if the user says "ranked B and above", the ConferenceAgent should pass filter: { rank: "A*,A,B" }.
```
-> Cần kết hợp lại cả hai cái. Vì mỗi cái riêng lẻ đều thiếu ý.

# Thay đổi 2
```patch
-   - **FIRST: Search the context window** — look through your own previous turns (role="model") and user messages (role="user") to identify the conference or list directly. If found, use \`query\`, \`identifier\`, and \`identifierType\` parameters when calling tools. **Do NOT use \`conferenceRef\` if the conference is identifiable from context.**
+   - **FIRST: Search the context window** — look through your own previous turns (role="model") and user messages (role="user") to identify the conference or list directly.
+   - **CRITICAL: If the conference/list is found in context window**, you MUST extract and pass the specific identifier (title, acronym, or ID) in the taskDescription to the subagent. Do NOT rely on the subagent to resolve via conferenceRef.
+   - **Example:** If context contains "ICML 2025" and user says "follow that one", taskDescription should be "Follow ICML conference" (with identifier), NOT "Follow the conference from the previous list" (which would trigger conferenceRef).
```
-> Sửa thành:
```markdown
   - **FIRST: Search the context window** — look through your own previous turns (role="model") and user messages (role="user") to identify the conference or list directly. If found, use \`query\`, \`identifier\`, and \`identifierType\` parameters when calling tools. **Do NOT use \`conferenceRef\` if the conference is identifiable from context.**
   - **CRITICAL: If the conference/list is found in context window**, you MUST extract and pass the specific identifier (title, acronym, or ID) in the taskDescription to the subagent. Do NOT rely on the subagent to resolve via conferenceRef.
   - **Example:** If context contains "ICML 2025" and user says "follow that one", taskDescription should be "Follow ICML conference" (with identifier), NOT "Follow the conference from the previous list" (which would trigger conferenceRef).
```

# Thay đổi 3
```patch
-retrieveKnowledge auto-personalizes all conference list searches by default (useRecommendationFilter defaults to true when listMode=true). Set useRecommendationFilter=false only for non-search actions: view details, follow/unfollow, calendar operations, rating/feedback, or blacklist.
-
-Routing rules:
-- Generic "recommend for me" (no constraints at all) → getRecommendations tool
-- Any broad conference search / browse / list query → retrieveKnowledge with listMode=true
-- Detail view of a specific conference → retrieveKnowledge with listMode=false
-- Continuation request ("show more", "next page") → showMoreRecommendations tool
-- Similar conferences → getSimilarConferences tool
+ConferenceAgent retrieveKnowledge supports useRecommendationFilter for personalized conference list searches.
+Personalized recommendation routing is strict:
+- Generic personalized recommendation without constraints, such as "recommend conferences for me", uses getRecommendations.
+- Personalized recommendation with constraints, such as topic, rank, location, date, access type, or "AI conferences", uses retrieveKnowledge with listMode=true and useRecommendationFilter=true.
+- showMoreRecommendations is only for explicit continuation requests such as "show more recommendations" or "next recommendations".
```
-> Revert lại thay đổi này.

# Thay đổi 4
```patch
-     - **Generic recommendations** (no constraints, just "recommend for me"): Route to 'ConferenceAgent'. 'taskDescription' = "Use getRecommendations for unconstrained recommendation."
-     - **Any broad conference search** (topic, rank, location, date, type): Route to 'ConferenceAgent'. 'taskDescription' = "Use retrieveKnowledge with listMode=true. Personalization is automatic via useRecommendationFilter (defaults to true)."
-     - **Similar conferences**: Route to 'ConferenceAgent'. 'taskDescription' = "Find conferences similar to [conference]."
-     - **Continuation** ("show more", "next page"): Route to 'ConferenceAgent'. 'taskDescription' = "Use showMoreRecommendations." Do NOT call retrieveKnowledge again.
+     - If the user asks for generic personalized recommendations without constraints (e.g., "recommend me some conferences", "conferences I may like"): Route to 'ConferenceAgent'. 'taskDescription' = "Recommend conferences for the user. Use getRecommendations because the request has no topic/rank/location/date/type constraints."
+     - If the user asks for personalized recommendations with constraints (e.g., "recommend me AI conferences", "recommend rank B conferences", "recommend conferences in Vietnam", "suggest free-access conferences for me"): Route to 'ConferenceAgent'. 'taskDescription' = "Find personalized recommended conferences matching [constraints]. Use retrieveKnowledge with listMode=true and useRecommendationFilter=true. Do not use getRecommendations for this constrained request."
+     - If the user asks for similar conferences: Route to 'ConferenceAgent'. 'taskDescription' = "Find conferences similar to the [conference name or acronym] conference."
+     - If the user explicitly asks to see more recommendation or conference-list results from the previous list (e.g., "show more", "more conferences", "next page", "next set"): Route to 'ConferenceAgent'. 'taskDescription' = "Show more recommendations from the last list." Do NOT mention retrieveKnowledge in this continuation task.
```
-> Sửa thành:
```markdown
- If the user asks for generic personalized recommendations without constraints (e.g., "recommend me some conferences", "conferences I may like"): Route to 'ConferenceAgent'. 'taskDescription' = "Recommend conferences for the user. Use getRecommendations because the request has no topic/rank/location/date/type/... constraints."
- If the user asks for personalized recommendations with constraints (e.g., "recommend me AI conferences", "recommend rank B conferences", "recommend conferences in Vietnam", "suggest free-access conferences for me"): Route to 'ConferenceAgent'. 'taskDescription' = "Find personalized recommended conferences matching [constraints]. Use retrieveKnowledge with listMode=true and useRecommendationFilter=true. Do not use getRecommendations for this constrained request."
- **Any broad conference search** (topic, rank, location, date, type,...): Route to 'ConferenceAgent'. 'taskDescription' = "Use retrieveKnowledge with listMode=true. Personalization is automatic via useRecommendationFilter (defaults to true)."
- If the user asks for similar conferences: Route to 'ConferenceAgent'. 'taskDescription' = "Find conferences similar to the [conference name or acronym] conference."
- **Continuation** ("show more", "next page"): Route to 'ConferenceAgent'. 'taskDescription' = "Use showMoreRecommendations." Do NOT call retrieveKnowledge again.
```

- Tuy nhiên, taskDescription phải đúng theo format:
```md
\`\`\`
USER INPUT:
[Original user request]
---
[Optional, only when retrying same subagent with same task]
ATTEMPT {n} (n >= 2)
REASON: The previous attempts have these issues: [list issues]
---
[Optional, for additional instructions]
ADDITIONAL INSTRUCTION: [additional instructions for the subagent]
\`\`\`
```