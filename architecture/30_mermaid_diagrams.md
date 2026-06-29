# 30 — Mermaid Diagrams

> **Back to Index**: [00_index.md](00_index.md)

---

## 30.1 System Architecture Diagram

```mermaid
graph TB
    subgraph USER ["User Layer"]
        B[Browser SPA]
    end

    subgraph WEB ["Web Tier"]
        N[Nginx + SSL]
        F1[Flask Gunicorn 1]
        F2[Flask Gunicorn 2]
    end

    subgraph ASYNC ["Async Tier"]
        R[Redis Broker]
        CW1[Celery Worker\nEmbed + Gen]
        CW2[Celery Worker\nPlagiarism + AI]
    end

    subgraph DATA ["Data Tier"]
        PG[(PostgreSQL)]
        PC[(Pinecone)]
        S3[(AWS S3)]
    end

    subgraph AI ["AI Providers"]
        NVIDIA[NVIDIA NIM\nGLM + DeepSeek + SDXL]
        GEMINI[Google Gemini]
        OPENAI[OpenAI]
        OLLAMA[Ollama Local]
    end

    B --> N
    N --> F1
    N --> F2
    F1 & F2 --> PG
    F1 & F2 --> R
    F1 & F2 --> PC
    F1 & F2 --> S3
    R --> CW1 & CW2
    CW1 & CW2 --> PG & PC
    F1 & F2 & CW1 & CW2 --> NVIDIA
    NVIDIA -. fallback .-> GEMINI
    GEMINI -. fallback .-> OPENAI
    OPENAI -. fallback .-> OLLAMA
```

---

## 30.2 Auth + Token Flow

```mermaid
sequenceDiagram
    participant B as Browser
    participant F as Flask
    participant R as Redis
    participant D as PostgreSQL

    Note over B,D: Login Flow
    B->>F: POST /api/auth/login {email, password}
    F->>D: SELECT user WHERE email=?
    D-->>F: User record
    F->>F: check_password(hash)
    F->>F: create_access_token + create_refresh_token
    F->>D: INSERT refresh_token(SHA256(raw))
    F-->>B: 200 Set-Cookie: access_token + refresh_token + csrf

    Note over B,D: API Call Flow
    B->>F: POST /api/papers/{id}/generate\nCookie: access_token\nHeader: X-CSRF-TOKEN
    F->>F: jwt_required() → verify cookie
    F->>R: GET blocklist:{jti}
    R-->>F: nil (not revoked)
    F->>D: SET LOCAL app.current_user_id='{uuid}'
    F->>F: Route handler executes
    F-->>B: 202 Accepted

    Note over B,D: Logout Flow
    B->>F: POST /api/auth/logout
    F->>F: GET jti from current JWT
    F->>R: SETEX blocklist:{jti} {ttl_seconds} "1"
    F-->>B: 200 + unset_jwt_cookies
```

---

## 30.3 RAG Pipeline Flow

```mermaid
flowchart LR
    subgraph UPLOAD ["Upload Phase"]
        A[User uploads PDF] --> B[text_extractor\nExtract text]
        B --> C[LangChain splitter\n1000 char chunks]
        C --> D[SentenceTransformer\nEncode 384-dim]
        D --> E[Pinecone upsert\nnamespace: project_uuid]
    end

    subgraph GENERATE ["Generation Phase"]
        F[Generate section] --> G[Section query string]
        G --> H[Encode query\nSentenceTransformer]
        H --> I[Pinecone.query top_k=15]
        I --> J[Filter score > 0.20\nDiversify: max 2/doc]
        J --> K[Build grounded prompt\nwith source tags]
        K --> L[call_ai - GLM-5.1]
        L --> M[Save section text\nto Paper DB]
    end

    E --> I
```

---

## 30.4 Multi-Model Router Flow

```mermaid
flowchart TD
    IN["call_ai(prompt, task_type='plagiarism')"] --> DEV{DEV_LLM_PROVIDER\n= ollama?}
    DEV -- Yes --> OLL["Ollama local"]
    DEV -- No --> CHAIN["chain = [deepseek, gemini]"]

    CHAIN --> DS_CHECK{DeepSeek\ncircuit open?}
    DS_CHECK -- Open --> GEM_CHECK
    DS_CHECK -- Closed --> DS["Try DeepSeek V4-Flash\n(NVIDIA NIM)"]
    DS -- Success --> LOG["Log usage to DB\nReturn text"]
    DS -- 429/Error --> DS_OPEN["Open circuit 60s"]
    DS_OPEN --> GEM_CHECK

    GEM_CHECK{Gemini\ncircuit open?} -- Open --> FINAL_OLL
    GEM_CHECK -- Closed --> RATE{Gemini rate\nlimit OK?}
    RATE -- Limit reached --> FINAL_OLL
    RATE -- OK --> GEM["Try Gemini 2.0 Flash"]
    GEM -- Success --> LOG
    GEM -- Quota Error --> GEM_OPEN["Open circuit 3600s"]
    GEM_OPEN --> FINAL_OLL

    FINAL_OLL["Last resort: Ollama"]
    FINAL_OLL -- Success --> LOG
    FINAL_OLL -- Fail --> ERR["raise AIUnavailableError"]
```

---

## 30.5 Plagiarism Scan Pipeline

```mermaid
flowchart TD
    A["Start Plagiarism Scan"] --> B["preprocess_task\nSentence tokenization\nProgress: 15%"]

    B --> C1["exact_match_scan\nPostgreSQL trigram\n+ MinHash fingerprints\nProgress: 40%"]
    B --> C2["semantic_scan\nSentenceTransformer\ncosine similarity\nProgress: 40%"]

    C1 & C2 --> D["ai_verification_task\nLLM confirms/dismisses\nborderline matches\nProgress: 75%"]

    D --> E["aggregate_results\nMerge + deduplicate\nCalculate score\nProgress: 100%"]

    E --> F["ScanTask.status = completed\nNotification sent"]
```

---

## 30.6 AI Humanizer Flow

```mermaid
flowchart TD
    IN["Input text\n+ intensity level"] --> PROTECT["Protect [[cite:UUID]] tags"]

    PROTECT --> P1["Phase 1: Pattern Removal\n30+ regex replacements\nContractions + jargon"]

    P1 --> INTENSITY{Intensity\nmedium or high?}
    INTENSITY -- Yes --> P2["Phase 2: Burstiness Control\nSplit long sentences\nMerge short sentences"]
    INTENSITY -- No --> P3

    P2 --> P3["Phase 3: LLM Deep Rewrite\nDeepSeek V4-Pro\nForensic prompt"]

    P3 --> REFINE["Self-refinement:\nScore before vs after\nKeep best version"]

    REFINE --> RESTORE["Restore [[cite:UUID]] tags"]
    RESTORE --> P4["Phase 4: Word-Level Diff\ndifflib token comparison"]
    P4 --> OUT["Return:\nhumanized text\nbefore/after scores\nword_diff annotations"]
```

---

## 30.7 Export Flow

```mermaid
flowchart TD
    A["POST /api/export/{id}/docx"] --> B["Load Paper + Citations\nLoad User info"]
    B --> C["_get_citation_replacement_helper()"]
    C --> D["Apply section order\nfor template"]
    D --> E["For each section:"]
    E --> F["section_text = replace_citations(text)\n[[cite:UUID]] → [1] or (Author, 2020)"]
    F --> G["add_heading(section_title)"]
    G --> H["add_paragraph(text)"]
    H --> I{More sections?}
    I -- Yes --> E
    I -- No --> J["Generate bibliography\nCitation.formatted strings"]
    J --> K["Apply template formatting\n(margins, fonts, columns)"]
    K --> L["doc.save() → BytesIO"]
    L --> M["send_file() binary DOCX"]
```

---

## 30.8 Frontend SPA Navigation

```mermaid
stateDiagram-v2
    [*] --> Landing: First visit
    Landing --> Login: Click Login
    Landing --> Register: Click Register
    Register --> OTP: Submit form
    OTP --> Dashboard: Verify OTP
    Login --> Dashboard: Success
    Login --> TwoFA: 2FA required

    Dashboard --> Projects: Click My Projects
    Dashboard --> NewProject: Click New Project
    NewProject --> Upload: Create project
    Upload --> Analysis: Embed complete
    Analysis --> Generation: Click Generate
    Generation --> Editor: Generation complete
    
    Editor --> Citations: Click Citations
    Editor --> DiagramStudio: Click Diagram Studio
    Editor --> Plagiarism: Click Check Plagiarism
    Editor --> Humanizer: Click Humanizer
    Editor --> Export: Click Export

    state DiagramStudio {
        [*] --> ScanOpportunities
        ScanOpportunities --> SelectLocation
        SelectLocation --> GenerateDiagram
        GenerateDiagram --> InsertInPaper
    }
```
