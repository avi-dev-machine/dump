# DPDP Compliance Platform — Technical Flowcharts
## RAG System & Chatbot: Techniques, Methods, Processes & Tech Stack

---

## 1. Master System Flow (Top Level)

```mermaid
flowchart TD
    A([User Query]) --> B[Frontend\nindex.html]
    B --> C{API Layer\nFastAPI}

    C --> D[Step 0\nQuery Encoder\ngemini-2.0-flash]
    D --> E[Step 1\nSemantic Retrieval\nChromaDB + BGE embeddings]
    E --> F[Step 2\nContext Builder\nWiki Page Formatter]
    F --> G[Step 3\nAnswer Generator\ngemini-2.5-flash SSE Stream]
    G --> H{Self-Correction\nCheck}

    H -- No READ flag --> I([Streamed Answer to User])
    H -- READ: page_id --> J[Step 4\nDirect Wiki Fetch\nIn-memory dict lookup]
    J --> K[Re-invoke LLM\nwith expanded context]
    K --> I

    subgraph DATA_LAYER [Knowledge Data Layer]
        L[(dpdp_titled.json\nAct 2023 Raw JSON)]
        M[(DPDP_Rules_2025.txt\nPDF extracted text)]
        N[(wiki/act/*.md\nwiki/rules/*.md\nwiki/concepts/*.md)]
        O[(ChromaDB\nwiki_db/\nPersistent Vector Store)]
        L -->|ingest_act| N
        M -->|ingest_rules| N
        N -->|build_wiki_store| O
    end

    E <--> O
    F <--> N

    style DATA_LAYER fill:#0f172a,stroke:#3b82f6,color:#e2e8f0
    style A fill:#1d4ed8,color:#fff
    style I fill:#0d9488,color:#fff
```

---

## 2. Knowledge Ingestion Pipeline

> **Purpose:** Transform raw, immutable legal PDFs/JSON into a structured, annotated wiki of Markdown pages that power retrieval.

```mermaid
flowchart TD
    subgraph RAW [Raw Sources — Never Modified]
        R1[(dpdp_titled.json\nDPDP Act 2023\nHierarchical JSON)]
        R2[(DPDP_Rules_2025.txt\nExtracted from PDF\nvia pdfplumber)]
    end

    subgraph INGEST_ACT [ingest_act — Act Ingestion]
        A1[Load JSON\nparse chapters list]
        A2[For each chapter\niterate sections]
        A3[_flatten obj\nrecursively join\nclauses + sub_clauses\n+ sub_sub_clauses]
        A4[_tags raw_text\nkeyword to tag\nmapping dict]
        A5[_act_refs raw_text\nregex: section|§ N\nextract cross-refs]
        A6[Build key_points\nfrom clause list\nclause.content 20+ chars]
        A7[Write\nwiki/act/section_N.md\nYAML frontmatter\n+ Summary + Key Points\n+ Verbatim + Cross-Refs]
        A1 --> A2 --> A3
        A3 --> A4
        A3 --> A5
        A3 --> A6
        A4 & A5 & A6 --> A7
    end

    subgraph INGEST_RULES [ingest_rules — Rules Ingestion]
        B1[Read raw TXT]
        B2[_clean_txt\nRemove Gazette headers\nRemove page separators\nCollapse blank lines]
        B3[Split on FIRST SCHEDULE\nmain_rules_text vs schedules_text]
        B4[RULE_RE regex\nMatch N. TITLE — pattern\nextract rule number + title + body]
        B5[rule_to_schedules dict\nAppend schedule content\nto matched rules]
        B6[_rule_key_points\nextract sub-rule\n1 2 3... items]
        B7[Write\nwiki/rules/rule_N.md]
        B1 --> B2 --> B3 --> B4 --> B5 --> B6 --> B7
    end

    subgraph INGEST_CONCEPTS [ingest_concepts — Synthesis Pages]
        C1[6 hardcoded concept bodies:\nconsent, data_fiduciary,\ndata_principal_rights,\npenalties, children_data,\ndata_transfer]
        C2[Combine act_refs + rules_refs\nfrom multiple sections + rules]
        C3[Write wiki/concepts/*.md\nCross-cutting synthesis\npre-answering common queries]
        C1 --> C2 --> C3
    end

    subgraph INDEX [update_index + write_overview]
        D1[Build wiki/index.md\nFull catalog: ID + path\n+ title + summary + tags]
        D2[Build wiki/overview.md\nAct structure table\nRules structure table\nAct-Rules mapping table]
    end

    R1 --> INGEST_ACT
    R2 --> INGEST_RULES
    INGEST_ACT & INGEST_RULES & INGEST_CONCEPTS --> INDEX

    style RAW fill:#1e293b,stroke:#475569,color:#e2e8f0
    style INGEST_ACT fill:#1e3a5f,stroke:#3b82f6,color:#e2e8f0
    style INGEST_RULES fill:#1a3a2a,stroke:#22c55e,color:#e2e8f0
    style INGEST_CONCEPTS fill:#3b1f4a,stroke:#a855f7,color:#e2e8f0
    style INDEX fill:#2d1f1a,stroke:#f97316,color:#e2e8f0
```

---

## 3. Vector Store Build & Freshness Detection

> **Problem Solved:** The index must always reflect the current wiki state. If a wiki page changes, the embeddings must be regenerated — but not unnecessarily.

```mermaid
flowchart TD
    START([build_wiki_store called]) --> Q1{Collection\ndpdp_wiki_v3\nexists in ChromaDB?}

    Q1 -- No --> REBUILD
    Q1 -- Yes --> Q2{col.count\n== 0?}
    Q2 -- Yes --> REBUILD
    Q2 -- No --> Q3[Get db_sqlite\nmtime timestamp\nfrom chroma.sqlite3]
    Q3 --> LOOP[For each .md file\nin wiki/]
    LOOP --> Q4{file.mtime\n> db_mtime?}
    Q4 -- Yes → any file newer --> REBUILD
    Q4 -- No, all files same age --> LOAD

    subgraph REBUILD [Rebuild Flow]
        R1[delete_collection\ndpdp_wiki_v3]
        R2[create_collection\ndpdp_wiki_v3\nembedding_function=BGE]
        R3[extract_wiki_chunks\nwiki_dir.rglob .md\nskip index.md + log.md]
        R4[Parse YAML frontmatter\nfrom each page\nextract: id, type, source, title, tags, path]
        R5[embed_text = full page text\nSentenceTransformer encodes\n384-dim vector per page]
        R6[col.upsert in batches of 50\nids + documents + metadatas]
        R7[Return WikiVectorStore\nwas_built = True]
        R1 --> R2 --> R3 --> R4 --> R5 --> R6 --> R7
    end

    subgraph LOAD [Load from Disk]
        L1[client.get_collection\ndpdp_wiki_v3]
        L2[Return WikiVectorStore\nwas_built = False]
        L1 --> L2
    end

    subgraph MEMORY [In-Memory Cache Built at Init]
        M1[self._pages dict\npage_id → full text\nfor all .md files]
    end

    R7 & L2 --> MEMORY
    MEMORY --> DONE([WikiVectorStore ready\nfor queries])

    style REBUILD fill:#1e3a5f,stroke:#3b82f6,color:#e2e8f0
    style LOAD fill:#1a3a2a,stroke:#22c55e,color:#e2e8f0
    style MEMORY fill:#2d1f1a,stroke:#f97316,color:#e2e8f0
```

---

## 4. RAG Query Pipeline — Full Detail

> **Core Architecture:** Wiki-Page RAG (current production system)
> The 5-step pipeline that runs on every user query.

```mermaid
flowchart TD
    USER([User Question\ne.g. Does my startup need a DPO?]) --> S0

    subgraph S0 [STEP 0 — Query Encoding]
        E1[Build messages list\nsystem=ENCODER_PROMPT\nuser=raw_question]
        E2[gemini_complete\nmodel: gemini-2.0-flash\ntemp: 0.0  max_tokens: 300]
        E3[Parse JSON response\nstrip markdown fences\njson.loads]
        E4{Parse\nsucceeded?}
        E5[Return structured dict:\nquery_type, actors\ncore_inquiry, concepts\nsearch_queries list]
        E6[Fallback: use raw input\nas single search query]
        E1 --> E2 --> E3 --> E4
        E4 -- Yes --> E5
        E4 -- No / Exception --> E6
    end

    S0 --> S1

    subgraph S1 [STEP 1 — Semantic Retrieval]
        R1[For each search_query\nin search_queries list]
        R2[WikiVectorStore.query\ntop_k=10 per query]
        R3[ChromaDB cosine similarity\nBAAI/bge-small-en-v1.5\n384-dim vectors]
        R4[score = max 0.0  1.0 - L2_distance]
        R5[Deduplicate by page_id\nkeep highest score\nif same page appears\nfrom multiple queries]
        R6[Sort by score desc\ncap at 10 pages total]
        R7{best_score\n< 0.25?}
        R8[Set low_confidence = True\nfor all results]
        R9[Results with metadata:\npage_id, source_doc, title\nscore, tags, path]
        R1 --> R2 --> R3 --> R4 --> R5 --> R6 --> R7
        R7 -- Yes --> R8 --> R9
        R7 -- No --> R9
    end

    S1 --> S2

    subgraph S2 [STEP 2 — Context Construction]
        C1{low_confidence\nflagged?}
        C2[Prepend LOW_CONFIDENCE\nwarning string to context]
        C3[For each retrieved page\nNumbered 1 to N]
        C4[Read full page text\nstore.read_page page_id\nin-memory dict lookup]
        C5[Build header:\nN  source_label  title\npath  score  tags]
        C6[Concatenate:\nheader + full_page_text]
        C7[Join all pages\nwith --- separator]
        C8[Insert into user message:\nUSER QUERY + query metadata\n+ WIKI PAGES context block]
        C1 -- Yes --> C2 --> C3
        C1 -- No --> C3
        C3 --> C4 --> C5 --> C6 --> C7 --> C8
    end

    S2 --> S3

    subgraph S3 [STEP 3 — LLM Answer Generation]
        G1[Build messages list:\nsystem=SYSTEM_PROMPT\n+ rolling history 4 turns\n+ user=context_user_message]
        G2[POST to Gemini API\nstreamGenerateContent\nalt=sse  model=gemini-2.5-flash]
        G3[Token-by-token SSE parsing:\nfor line in resp.iter_lines\ndecode UTF-8\nparse data: JSON chunk\nextract text token]
        G4[Buffer first 150 tokens\nbefore yielding to user]
        G1 --> G2 --> G3 --> G4
    end

    S3 --> S4

    subgraph S4 [STEP 4 — Self-Correction Check]
        SC1{What appears\nin buffer?}
        SC2[think tag detected\nContext sufficient\nTIER 2 or 3 reasoning\nStart streaming]
        SC3[READ: page_ids\nContext insufficient\nLLM requests more pages]
        SC4[Buffer >= 150 chars\nNo flag detected\nContext sufficient\nStart streaming]
        SC5[Parse page_ids from\nREAD: x y z]
        SC6[store.fetch_pages\npage_ids not already\nin retrieved set\nDirect dict lookup]
        SC7[Re-invoke LLM\nWith pages + extra pages\n+ CHECK_PROMPT prefix\nDo not output READ flags]
        SC1 -- think --> SC2
        SC1 -- READ: --> SC3
        SC1 -- neither at 150 chars --> SC4
        SC3 --> SC5 --> SC6 --> SC7
    end

    SC2 & SC4 & SC7 --> OUT([Streamed Answer\nToken by Token\nCited  [Act N]  [Rules r.N]])

    style S0 fill:#1e3a5f,stroke:#3b82f6,color:#e2e8f0
    style S1 fill:#1a3a2a,stroke:#22c55e,color:#e2e8f0
    style S2 fill:#3b2a1a,stroke:#f97316,color:#e2e8f0
    style S3 fill:#3b1f4a,stroke:#a855f7,color:#e2e8f0
    style S4 fill:#1f3a3a,stroke:#14b8a6,color:#e2e8f0
```

---

## 5. Three-Tier RAG (Older Architecture) — Retrieval Detail

> The earlier implementation. Understand this because interviewers will ask about the architectural evolution.

```mermaid
flowchart TD
    Q([Search Query]) --> T1 & T2 & T3

    subgraph T1 [TIER 1 — Section Query]
        S1[ChromaDB query\nwhere tier=section\ntop_k = 3]
        S2[Returns: full section chunks\nBroad conceptual match\ne.g. ch2_s10_section]
        S1 --> S2
    end

    subgraph T2 [TIER 2 — Clause Query]
        CL1[ChromaDB query\nwhere tier=clause\ntop_k = 4]
        CL2[Returns: individual clause chunks\nClause number + parent context\ne.g. ch2_s10_cl0_clause]
        CL1 --> CL2
    end

    subgraph T3 [TIER 3 — Leaf Query]
        L1[ChromaDB query\nwhere tier=leaf\ntop_k = 3]
        L2[Returns: sub-clause chunks\nFull breadcrumb: Ch2 §10 1 b\ne.g. ch2_s10_cl0_sc1_leaf]
        L1 --> L2
    end

    T1 & T2 & T3 --> MERGE[Merge all results\nDedup by chunk_id\nKeep highest score if duplicate\nSort by score desc]

    MERGE --> PP[Parent Promotion\nFor each CLAUSE or LEAF hit\nfetch parent_section_id\nif not already in results\nAdd with score = 0.0]

    PP --> XR[Cross-Reference Expansion\nScan raw_text of retrieved chunks\nfor regex: §N or Section N\nFetch those section-tier chunks\nif not already retrieved]

    XR --> TOP[Cap at top_k = 8\nAttach low_confidence flag\nif best_score < 0.35]

    TOP --> OUT([Retrieved chunks\nready for context builder])

    subgraph EMBED [Embedding Structure per Chunk]
        EM1["[MEANING] plain English summary
        [KEYWORDS] synonym list
        [USER_QUESTIONS] how user would phrase it
        [IMPLICIT_ANSWERS] expected answer terms
        [TEXT] verbatim legal text"]
    end

    style T1 fill:#1e3a5f,stroke:#3b82f6,color:#e2e8f0
    style T2 fill:#1a3a2a,stroke:#22c55e,color:#e2e8f0
    style T3 fill:#3b1f4a,stroke:#a855f7,color:#e2e8f0
    style EMBED fill:#1e293b,stroke:#475569,color:#e2e8f0
```

---

## 6. LLM Tiered Reasoning System

> How the LLM self-classifies and adapts its reasoning depth to the complexity of the question.

```mermaid
flowchart TD
    Q([User Query arrives\nwith retrieved wiki pages]) --> LLM[LLM reads:\n1. System Prompt\n2. Conversation history\n3. Retrieved pages\n4. User question]

    LLM --> CLASS{Self-classify\nquery tier}

    CLASS --> T1 & T2 & T3

    subgraph T1 [TIER 1 — Simple]
        T1A["Signal: 'What is X?'\n'Who is a Y?'\n'What must Z do in case of W?'"]
        T1B[No think block\nAnswer in 1-3 sentences\nWith citations Act N]
        T1C[Example:\n'What is a Data Principal?'\n→ 'A Data Principal is any individual\nto whom personal data relates [Act §2]'"]
        T1A --> T1B --> T1C
    end

    subgraph T2 [TIER 2 — Multi-provision]
        T2A[Signal: 2-4 sub-questions\nRequires synthesis\nacross multiple pages]
        T2B["MUST use XML tags:\n<think> [EVIDENCE]... </think>"]
        T2C[After think block:\nNumbered list with citations]
        T2D[Example:\n'Does a bank need DPO?'\n→ Combines: §10 SDF definition\n+ r.13 SDF obligations\n+ §2 definitions']
        T2A --> T2B --> T2C --> T2D
    end

    subgraph T3 [TIER 3 — Complex]
        T3A[Signal: 4+ sub-questions\nSchedules + Rules + Act\nGenuine legal conflicts]
        T3B["MUST use structured think:\n[DECOMPOSITION]\n[ENTITY MAPPING]\n[EVIDENCE]\n[CLAIM GROUNDING]\n[CONFLICT RESOLUTION]\n[ANSWER SKELETON]\n[COVERAGE CHECK]"]
        T3C[Full markdown answer\nEvery sub-question numbered\nAmbiguity Rule applied]
        T3D[Example:\n'SDF + children's data breach\n→ notification timelines\n+ responsible personnel\n+ max penalties'"]
        T3A --> T3B --> T3C --> T3D
    end

    T1 & T2 & T3 --> POLICY

    subgraph POLICY [Cross-Cutting Policies Applied to All Tiers]
        P1[Cite every claim\n[Act §N clause]\n[Rules r.N sub-rule]]
        P2[Forbidden: Invent section numbers\nFabricate obligations\nChoose ambiguous interpretation]
        P3[Ambiguity Rule:\nPresent BOTH interpretations\nNever pick one\nRecommend legal counsel]
        P4[Framing: Regulatory analysis\nNOT legal advice]
    end

    style T1 fill:#1a3a2a,stroke:#22c55e,color:#e2e8f0
    style T2 fill:#1e3a5f,stroke:#3b82f6,color:#e2e8f0
    style T3 fill:#3b1f4a,stroke:#a855f7,color:#e2e8f0
    style POLICY fill:#2d1f1a,stroke:#f97316,color:#e2e8f0
```

---

## 7. API Key Pool — Quota Resilience Flow

```mermaid
flowchart TD
    CALL([Gemini API call needed]) --> PICK[_next_key\npick GEMINI_KEY_POOL pool_idx\nincrement pool_idx mod 5]

    PICK --> POST[requests.post\nto Gemini endpoint\nwith selected key\ntimeout = 10 connect 120 read]

    POST --> RESP{HTTP\nStatus Code}

    RESP -- 200 OK --> SUCCESS([Return response\nor yield SSE tokens])

    RESP -- 429 Rate Limit\nor 503 Unavailable --> RETRY{attempts\n< pool size\n= 5?}

    RETRY -- Yes, try next --> PICK

    RETRY -- No, all keys failed --> ERR([Raise RuntimeError\nAll Gemini keys\nquota-exceeded])

    RESP -- ConnectTimeout --> TIMEOUT[Print connection\ntimeout warning] --> RETRY
    RESP -- ReadTimeout --> RTIME[Print read\ntimeout warning] --> RETRY

    RESP -- Other 4xx or 5xx --> HARD([Raise RuntimeError\nGemini status_code\nresponse text])

    subgraph POOL [Key Pool State]
        K1[Key 1\nIndex 0]
        K2[Key 2\nIndex 1]
        K3[Key 3\nIndex 2]
        K4[Key 4\nIndex 3]
        K5[Key 5\nIndex 4]
        K1 --> K2 --> K3 --> K4 --> K5 --> K1
    end

    style POOL fill:#1e293b,stroke:#f97316,color:#e2e8f0
    style SUCCESS fill:#0d9488,color:#fff
    style ERR fill:#dc2626,color:#fff
    style HARD fill:#dc2626,color:#fff
```

---

## 8. Self-Correction [READ:] Loop — Detailed Flow

```mermaid
flowchart TD
    START([LLM starts generating\nfirst tokens]) --> BUF[Buffer incoming\ntokens\nbuffer = '']

    BUF --> TOK[Receive next token\nbuffer += token]

    TOK --> C1{think in\nbuffer?}
    C1 -- Yes --> STREAM[decided = True\nYield buffer\nthen stream all\nremaining tokens directly]

    C1 -- No --> C2{READ in\nbuffer?}

    C2 -- Yes --> BRACKET{Has closing\nbracket ]?}
    BRACKET -- Not yet and\nbuffer < 300 chars --> TOK

    BRACKET -- Yes --> PARSE[FETCH_RE regex\nextract page_id list\nfrom READ: x y z]

    PARSE --> FILTER[Filter: remove page_ids\nalready in retrieved set\nusing set of page_ids]

    FILTER --> FETCH[store.fetch_pages\npages_ids_to_fetch\nIn-memory dict lookup\nO 1 per page]

    FETCH --> RE_INVOKE[Build new messages\noriginal pages + extra pages\n+ CHECK_PROMPT\nDo NOT output READ flags\nMUST use think tags\nCite provisions]

    RE_INVOKE --> RE_STREAM[gemini_stream_iter\nwith expanded context\nStream final answer]

    RE_STREAM --> DONE([Answer streamed\nUser sees seamless\ncomplete response])

    C2 -- No --> C3{buffer >=\n150 chars\nFLAG_LIMIT?}
    C3 -- Yes --> STREAM
    C3 -- No --> TOK

    subgraph NOTE [What CHECK_PROMPT Does]
        N1[1. Do NOT output READ flags\n   prevents infinite loop]
        N2[2. MUST use think tags\n   forces structured reasoning]
        N3[3. Cite provisions\n   maintains anti-hallucination]
    end

    STREAM --> DONE

    style START fill:#1d4ed8,color:#fff
    style DONE fill:#0d9488,color:#fff
    style NOTE fill:#1e293b,stroke:#a855f7,color:#e2e8f0
```

---

## 9. Conversation Memory Management

```mermaid
flowchart LR
    subgraph TURN1 [Turn 1]
        U1[User: raw question 1]
        A1[LLM: full answer 1\nup to 4096 tokens]
    end
    subgraph TURN2 [Turn 2]
        U2[User: follow-up question]
        A2[LLM: answer 2]
    end
    subgraph TURN3 [Turn 3]
        U3[User: ...]
        A3[LLM: ...]
    end
    subgraph TURN4 [Turn 4]
        U4[User: ...]
        A4[LLM: ...]
    end

    subgraph HISTORY [history list — what gets stored]
        H1["role:user\ncontent: core_inquiry\n[:200 chars]"]
        H2["role:assistant\ncontent: full_response\n[:500 chars]"]
        H3["role:user\ncontent: core_inquiry\n[:200 chars]"]
        H4["role:assistant\ncontent: full_response\n[:500 chars]"]
    end

    subgraph MSG_BUILD [Messages sent to LLM on each turn]
        M1[system: SYSTEM_PROMPT]
        M2[history entries\nmax 8 messages\n4 user + 4 assistant]
        M3[user: context_block\nwith wiki pages\n+ current query]
        M1 --> M2 --> M3
    end

    TURN1 --> HISTORY
    HISTORY --> MSG_BUILD

    subgraph WHY [Design Rationale]
        W1[core_inquiry not raw input\n→ shorter, more semantic\n→ lower token cost]
        W2[200 char user truncation\n+ 500 char assistant truncation\n→ summary not verbatim]
        W3[len history > 8\n→ history = history -8:\n→ rolling window: oldest dropped]
        W4[4 turns sufficient for\nfollow-ups while keeping\ntotal prompt tokens bounded]
    end

    style HISTORY fill:#1e293b,stroke:#3b82f6,color:#e2e8f0
    style MSG_BUILD fill:#1a3a2a,stroke:#22c55e,color:#e2e8f0
    style WHY fill:#2d1f1a,stroke:#f97316,color:#e2e8f0
```

---

## 10. Complete Tech Stack — Choice Justification

```mermaid
mindmap
  root((DPDP Platform\nTech Stack))
    LLM Layer
      Gemini 2.5 Flash Answer
        Why: Best reasoning per dollar
        Why: Native streaming SSE
        Why: 1M token context window
        Why: No SDK dependency via HTTP
      Gemini 2.0 Flash Encoder
        Why: Fastest Gemini model
        Why: Structured JSON output
        Why: temp=0.0 deterministic
    Embedding
      BAAI bge-small-en-v1.5
        Why: MTEB retrieval score 51.7
        Why: Beats MiniLM-L6 score 40.1
        Why: Asymmetric retrieval training
        Why: Legal text superior
        384 dimensions
    Vector DB
      ChromaDB
        Why: Embedded no server
        Why: Persistent PersistentClient
        Why: Metadata filtering support
        Why: Python native
        Why: Auto HNSW index
        Alternative rejected: Pinecone cost
        Alternative rejected: Weaviate complexity
    Backend
      FastAPI planned
        Why: Async native asyncio
        Why: StreamingResponse built-in
        Why: Pydantic auto-validation
        Why: Auto Swagger docs
        Why: Uvicorn high performance
      requests library now
        Why: Direct HTTP control
        Why: Key rotation flexibility
        Why: SSE stream parsing control
    Data Format
      Structured JSON
        dpdp_titled.json
          Hierarchical: ch - sec - clause - sub
          Why: Machine-readable Act
          Why: Enables multi-tier indexing
        newtry.json
          Nested compliance tree 100+ nodes
          Why: Visual mind map data
      Markdown Wiki
        YAML frontmatter metadata
        Why: Human + machine readable
        Why: Git-versionable
        Why: mtime freshness detection
    NLP Processing
      pdfplumber
        Why: Gazette PDF extraction
        Why: Preserves text structure
      Regex parsing
        _clean_txt Gazette headers
        _parse_rules_txt rule headings
        _tags keyword mapping
        _act_refs cross-ref extraction
    Frontend
      Vanilla HTML CSS JS
        Why: Zero build step
        Why: Single file deployment
        Why: Offline capable
        Data embedded inline
      Dark Light mode
        CSS custom properties
        data-theme dark override
      Highlight animation
        CSS keyframes
        highlightPulse
```

---

## 11. Anti-Hallucination Architecture

```mermaid
flowchart TD
    QUERY([Query Received]) --> L1

    subgraph L1 [Layer 1 — Grounding Instruction]
        I1[System Prompt FORBIDDEN list:\nDo not invent section numbers\nDo not fabricate obligations\nDo not state must unless text says so\nDo not choose ambiguous interpretation]
        I2[System Prompt ALLOWED list:\nReason from text freely\nSynthesise across pages\nDraw logical conclusions\nExplain in plain language]
    end

    L1 --> L2

    subgraph L2 [Layer 2 — Retrieval Confidence Scoring]
        S1[Every retrieved page gets\ncosine similarity score\n0.0 to 1.0]
        S2{best_score\n< threshold\n0.25 wiki\n0.35 3-tier}
        S3[low_confidence = True\nSet on all results]
        S4[Prepend to context:\nLOW_CONFIDENCE: Answer only\nfrom explicit text below\nDo not infer]
        S1 --> S2
        S2 -- Below threshold --> S3 --> S4
        S2 -- Above threshold --> S5[Normal context\nno restriction]
    end

    L2 --> L3

    subgraph L3 [Layer 3 — Citation Enforcement]
        C1[Every factual claim must be\ncited as Act §N or Rules r.N]
        C2[LLM cannot reference a section\nnot present in retrieved pages\nFORBIDDEN]
        C3[Citation format strictly specified\nin system prompt]
    end

    L3 --> L4

    subgraph L4 [Layer 4 — Self-Correction READ Loop]
        R1[LLM can signal READ: page_id\ninstead of guessing\nwhen context is insufficient]
        R2[System fetches requested pages\nRe-invokes with complete context]
        R3[CHECK_PROMPT forbids\nfurther READ flags\nprevents infinite loop]
    end

    L4 --> L5

    subgraph L5 [Layer 5 — Ambiguity Resolution Rule]
        A1{Does any retrieved\nprovision resolve\nthe ambiguity?}
        A2[Cite provision\nand answer directly]
        A3[Present BOTH\ninterpretations with\noperational implications]
        A4[Never pick one interpretation\nover the other arbitrarily]
        A5[Recommend qualified\nlegal counsel]
        A1 -- Yes --> A2
        A1 -- No --> A3 --> A4 --> A5
    end

    L5 --> OUT([Answer with citations\nConfidence-calibrated\nGrounded in retrieved text\nNever hallucinated])

    style L1 fill:#1e3a5f,stroke:#3b82f6,color:#e2e8f0
    style L2 fill:#1a3a2a,stroke:#22c55e,color:#e2e8f0
    style L3 fill:#3b2a1a,stroke:#f97316,color:#e2e8f0
    style L4 fill:#3b1f4a,stroke:#a855f7,color:#e2e8f0
    style L5 fill:#1f3a3a,stroke:#14b8a6,color:#e2e8f0
    style OUT fill:#0d9488,color:#fff
```

---

## 12. Architectural Evolution: 3-Tier → Wiki-Page RAG

```mermaid
flowchart LR
    subgraph OLD [Three-Tier RAG — vectorstore.py]
        O1[Raw JSON\ndpdp_titled.json]
        O2[Extract chunks\nSECTION tier\nCLAUSE tier\nLEAF tier]
        O3[Index in ChromaDB\nall-MiniLM-L6-v2\n3 separate queries per search]
        O4[Parent Promotion\n+ Cross-ref Expansion]
        O5[Multiple small chunks\nto LLM context]
        O1 --> O2 --> O3 --> O4 --> O5
    end

    subgraph NEW [Wiki-Page RAG — core/vectorstore.py]
        N1[Raw JSON + Rules TXT]
        N2[ingest.py\nTransform to wiki pages\nOne page per section/rule\nWith summaries + cross-refs]
        N3[Index in ChromaDB\nBAGI/bge-small-en-v1.5\n1 query per search term]
        N4[READ: self-correction\nloop replaces cross-ref expansion]
        N5[Fewer richer documents\nto LLM context]
        N1 --> N2 --> N3 --> N4 --> N5
    end

    subgraph TRADEOFFS [Why the Migration]
        T1[GAIN: Richer per-doc context\nGAIN: Simpler retrieval\nGAIN: Better embedding model\nGAIN: Human-readable wiki\nGAIN: Git-versionable knowledge]
        T2[COST: Less sub-clause granularity\nCOST: READ loop adds latency\non context misses\nCOST: Ingest step required]
    end

    OLD -->|Replaced by| NEW
    NEW --> TRADEOFFS

    style OLD fill:#1e293b,stroke:#475569,color:#94a3b8
    style NEW fill:#1e3a5f,stroke:#3b82f6,color:#e2e8f0
    style TRADEOFFS fill:#1a3a2a,stroke:#22c55e,color:#e2e8f0
```

---

## 13. End-to-End Sequence: Real Query Trace

> Concrete example: *"Can a hospital use treatment data to train an AI model?"*

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend
    participant ENC as Encoder\ngemini-2.0-flash
    participant VEC as ChromaDB\nWikiVectorStore
    participant CTX as Context Builder
    participant ANS as Answer LLM\ngemini-2.5-flash
    participant WIKI as Wiki Pages\nIn-Memory Cache

    U->>FE: "Can a hospital use treatment data to train an AI model?"
    FE->>ENC: POST /api/chat {query}

    Note over ENC: temp=0.0, max_tokens=300
    ENC-->>FE: {query_type: situation,\nactors: hospital + patients,\nconcepts: consent purpose limitation\nlegitimate use research,\nsearch_queries: [\n  "purpose limitation consent new use",\n  "legitimate uses without consent section 7",\n  "research archiving exemption rule 16"\n]}

    FE->>VEC: query("purpose limitation consent new use", top_k=10)
    VEC-->>FE: [section_6 0.88, section_7 0.85, consent 0.82...]
    FE->>VEC: query("legitimate uses without consent section 7", top_k=10)
    VEC-->>FE: [section_7 0.91, section_4 0.79...]
    FE->>VEC: query("research archiving exemption rule 16", top_k=10)
    VEC-->>FE: [rule_16 0.87, section_17 0.74...]

    Note over FE: Dedup + sort by score → top 10 pages

    FE->>WIKI: read_page("section_7")
    WIKI-->>FE: Full section_7.md content
    FE->>WIKI: read_page("rule_16")
    WIKI-->>FE: Full rule_16.md content
    Note over FE: Repeat for all 10 pages

    FE->>CTX: build_context_block(pages)
    CTX-->>FE: Numbered context string\n[1] [Act] Section 7 | score:0.91...\n[2] [Rules] Rule 16 | score:0.87...\n...

    FE->>ANS: POST streamGenerateContent\n{system: SYSTEM_PROMPT,\nhistory: last 4 turns,\nuser: query + context}

    Note over ANS: Buffer first 150 tokens

    ANS-->>FE: token: "<think>"
    Note over FE: <think> detected → context sufficient\ndecided = True → start streaming

    ANS-->>FE: token stream: <think>TIER 3 reasoning...\n[DECOMPOSITION] [EVIDENCE]\n[CLAIM GROUNDING]...\n</think>

    ANS-->>FE: token stream: "Fresh consent is not automatically..."

    Note over ANS: No [READ:] flag triggered\nAll relevant pages retrieved

    FE->>U: Stream tokens to browser (SSE)
    U->>U: Sees answer building word by word

    Note over FE: Update history:\nuser: core_inquiry[:200]\nassistant: answer[:500]
```

---

## Quick Reference: Technique Map

| Technique | Where Used | Why This Technique |
|-----------|-----------|-------------------|
| **RAG (Retrieval-Augmented Generation)** | Main chatbot pipeline | Grounds LLM to actual legal text; prevents hallucination of provisions |
| **Semantic Embedding** | ChromaDB + BGE | Bridges the gap between user language and legal language |
| **Asymmetric Retrieval** | BAAI/bge-small-en-v1.5 | Query is short+question-form; documents are long+declarative. BGE trained for this |
| **YAML Frontmatter** | Wiki pages | Machine-readable metadata (source, tags, cross-refs) embedded in human-readable files |
| **Keyword-to-Tag Mapping** | `_tags()` in ingest.py | Enables metadata filtering in vector DB; classifies sections by compliance domain |
| **Regex Cross-Ref Extraction** | `_act_refs()`, `_rule_refs()` | Detects `§N` / `Section N` / `rule N` patterns to build the knowledge graph |
| **Three-Tier Indexing** | vectorstore.py (older) | Solves granularity mismatch: Section for context, Clause for specificity, Leaf for precision |
| **Parent Promotion** | vectorstore.py | Ensures sub-clause matches always include full section context |
| **Wiki-Page Indexing** | core/vectorstore.py | Pre-compiled pages are context-complete; one rich document beats multiple small chunks |
| **mtime Freshness Detection** | build_wiki_store() | Auto-detects stale index without needing a manual manifest |
| **Token Buffering** | _buffered_stream() | Detects `<think>` vs `[READ:]` in first 150 tokens before passing through |
| **SSE Streaming** | gemini_stream_iter() | Token-by-token delivery — responsive UX, no wait for full response |
| **Round-Robin Key Pool** | _next_key() | ~5x free-tier quota; automatic 429 failover |
| **Rolling History Window** | chat_loop() | 4-turn (8-message) context; uses `core_inquiry` (not raw input) for efficiency |
| **Tiered LLM Reasoning** | SYSTEM_PROMPT TIER 1/2/3 | Matches answer depth to question complexity; avoids over-reasoning simple queries |
| **Ambiguity Resolution Rule** | SYSTEM_PROMPT | Prevents arbitrary legal interpretation; presents both readings when text is genuinely unclear |
| **Self-Correction Loop** | ask_wiki_llm() | LLM signals context gaps; system fetches + re-invokes transparently |
| **Benchmark Framework** | run_benchmark.py | Measures system quality on 25 mixed-capability legal questions; saves after every Q |
| **Wiki Linter** | lint_wiki() | Detects broken cross-refs and orphan pages; maintains knowledge base integrity |
| **PDF Preprocessing** | _clean_txt() | Removes Gazette headers, page separators, Hindi text artifacts from pdfplumber output |
| **Data-Inline HTML** | fix_viewer.py | Embeds JSON into HTML at build time; zero-server, offline-capable viewer |

---

*Flowchart document for BDS Cube Technologies DPDP Compliance Platform*
*Sources: `chatbot/chat.py` | `chatbot/core/vectorstore.py` | `chatbot/core/ingest.py` | `chatbot/vectorstore.py` | `run_benchmark.py` | `fix_viewer.py`*
