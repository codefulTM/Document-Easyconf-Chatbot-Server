# Flowchart `RetrieveKnowledgeHandler.execute`

```mermaid
flowchart TD
    A([Start]) --> B[Read args, userToken, callbacks, ids]
    B --> C[Resolve requestId]
    C --> D[Clone args into effectiveArgs]
    D --> E{conferenceRef present?}

    E -- Yes --> F[Resolve result set by conversationId + item]
    F --> G{Resolved identifiers found?}
    G -- No --> G1[Return error: out_of_range_reference]
    G -- Yes --> H[Merge ids/acronyms/titles into effectiveFilter]

    E -- No --> H

    H --> I{conferenceFields valid?}
    I -- No --> I1[Log validation error and return invalid conferenceFields]
    I -- Yes --> J[Normalize filter and default tableName=conferences]

    J --> K[Apply inferred location filter]
    K --> L[Infer listMode and specific conference intent]
    L --> M{query missing and no exact lookup?}
    M -- Yes --> M1[Log error and return missing query]
    M -- No --> N[Send status: retrieving_knowledge]

    N --> O[Clamp limit to 1..100]
    O --> P{Should use recommendation pre-filter?}

    P -- No --> U[Run plain retrieval]
    P -- Yes --> Q[Extract topics and classify recommendation mode]
    Q --> R[Fetch recommendation ids]
    R --> S{Recommendation ids available?}
    S -- No --> U
    S -- Yes --> T[Build candidate ids and intersect with pool]
    T --> T1{Final ids loaded and results found?}
    T1 -- No --> U
    T1 -- Yes --> T2[Build compact conference list from recommendation results]
    T2 --> V[Log flow, send behavior event, return compact JSON]

    U --> U1[Call retrievalService.retrieve(query, filter, limit, weights, listMode)]
    U1 --> W{Results found?}
    W -- No --> W1[Log no results and return "No relevant information found"]
    W -- Yes --> X[Build compact conference list]
    X --> Y[Enrich items with index/score/source when needed]
    Y --> Z[Compose compact payload JSON]
    Z --> AA[Send retrieval_complete status]
    AA --> AB[Log flow step]
    AB --> AC{Conference list mode?}
    AC -- Yes --> AD[Send SEARCH_CONFERENCE behavior event]
    AC -- No --> AE{Should track conference detail?}
    AE -- Yes --> AF[Resolve tracking name and send CLICK_CONFERENCE_TITLE event]
    AE -- No --> AG[Skip tracking]
    AD --> AH[Return compact JSON]
    AF --> AH
    AG --> AH

    V --> AH
    G1 --> END([End])
    I1 --> END
    M1 --> END
    W1 --> END
    AH --> END

    %% Error handling
    U1 -. throws .-> ERR[Catch error]
    R -. throws .-> ERR
    F -. throws .-> ERR
    ERR --> ERR1[Log retrieval_error and return error message]
    ERR1 --> END
```
