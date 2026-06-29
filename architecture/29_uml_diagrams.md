# 29 — UML Diagrams

> **Back to Index**: [00_index.md](00_index.md)

---

## 29.1 Component Diagram

```mermaid
graph LR
    subgraph FRONTEND ["Frontend (Browser SPA)"]
        HTML["index.html\n(SPA Shell)"]
        Router["appRouter.js\n(Screen Registry)"]
        API["api.js\n(HTTP Wrapper)"]
        State["sessionState.js\n(Client State)"]
        Screens["screens/*.js\n(50 screens)"]
        Shell["appShell.js\n(Layout)"]
    end

    subgraph BACKEND ["Backend (Flask)"]
        App["app.py\n(Factory)"]
        Auth["auth_bp\n(/api/auth)"]
        Papers["paper_bp\n(/api/papers)"]
        Export["export_bp\n(/api/export)"]
        Admin["admin_bp\n(/api/admin)"]
        Super["super_bp\n(/api/super)"]
    end

    subgraph SERVICES ["Services (Business Logic)"]
        AIRouter["ai_router.py\n(Provider Routing)"]
        RAG["pinecone_search.py\n(Context Retrieval)"]
        Generator["generation.py\n(Paper Gen)"]
        Humanizer["ai_humanizer.py\n(Humanization)"]
        Detector["ai_detector.py\n(AI Detection)"]
        Plagiarism["plagiarism_engine.py\n(Plagiarism)"]
    end

    subgraph WORKERS ["Workers (Celery)"]
        EmbedTask["embed_document\n(RAG Ingestion)"]
        PaperTask["paper_tasks\n(Generation)"]
        PlagTask["plagiarism_tasks\n(Chord Pipeline)"]
    end

    subgraph DATA ["Data (Storage)"]
        PG["PostgreSQL\n(ORM models)"]
        Redis["Redis\n(Broker + Blocklist)"]
        Pinecone["Pinecone\n(Vectors)"]
        S3["AWS S3\n(Files)"]
    end

    HTML --> Router --> Screens
    Screens --> API --> BACKEND
    BACKEND --> SERVICES
    BACKEND --> WORKERS
    SERVICES --> DATA
    WORKERS --> DATA
    AIRouter --> NVIDIA["NVIDIA NIM"]
    AIRouter --> Gemini["Google Gemini"]
```

---

## 29.2 Class Diagram — Domain Models

```mermaid
classDiagram
    class BaseModel {
        +UUID id
        +DateTime created_at
        +DateTime updated_at
        +DateTime deleted_at
        +bool is_active
        +soft_delete()
    }

    class User {
        +String email
        +String password_hash
        +String role
        +String plan
        +bool tfa_enabled
        +int failed_login_attempts
        +DateTime locked_until
        +set_password(plain_pw)
        +check_password(plain_pw) bool
    }

    class Project {
        +UUID user_id
        +String title
        +String domain
        +String citation_style
        +String template
        +JSON sections_config
    }

    class Document {
        +UUID project_id
        +String filename
        +Text extracted_text
        +String status
        +JSON author_metadata
        +String doi
    }

    class Paper {
        +UUID project_id
        +String title
        +Text abstract
        +Text introduction
        +Text methodology
        +Text results
        +Text conclusion
        +int progress
        +String status
        +float plagiarism_score
    }

    class Citation {
        +UUID paper_id
        +UUID document_id
        +int number
        +String authors
        +String formatted
        +String style
    }

    class Diagram {
        +UUID paper_id
        +String diagram_type
        +String title
        +JSON data
    }

    class ScanTask {
        +UUID paper_id
        +String status
        +int progress
        +String current_step
        +JSON results
        +float overall_score
    }

    class Notification {
        +UUID user_id
        +String type
        +String title
        +bool is_read
    }

    class Subscription {
        +UUID user_id
        +String plan
        +String status
        +DateTime expires_at
    }

    BaseModel <|-- User
    BaseModel <|-- Project
    BaseModel <|-- Document
    BaseModel <|-- Paper
    BaseModel <|-- Citation
    BaseModel <|-- Diagram
    BaseModel <|-- ScanTask
    BaseModel <|-- Notification
    BaseModel <|-- Subscription

    User "1" --> "many" Project : owns
    Project "1" --> "many" Document : contains
    Project "1" --> "many" Paper : generates
    Paper "1" --> "many" Citation : has
    Paper "1" --> "many" Diagram : contains
    Paper "1" --> "many" ScanTask : tracked by
    User "1" --> "many" Notification : receives
    User "1" --> "1" Subscription : has
```

---

## 29.3 Sequence Diagram — Paper Generation

```mermaid
sequenceDiagram
    participant UI as Browser
    participant Flask as Flask API
    participant Celery as Celery Worker
    participant Pinecone as Pinecone
    participant LLM as LLM Provider (GLM)
    participant DB as PostgreSQL

    UI->>Flask: POST /api/papers/{id}/generate
    Flask->>DB: Create/update Paper status=queued
    Flask->>Celery: generate_paper_task.delay(paper_id)
    Flask-->>UI: 202 {task_id}

    loop Every 3 seconds
        UI->>Flask: GET /api/papers/{id}/status
        Flask->>DB: Paper.progress
        Flask-->>UI: {progress: 45, current_section: "methodology"}
    end

    Celery->>DB: Paper.status = generating
    loop For each section
        Celery->>Pinecone: query(section_query, namespace=project_uuid)
        Pinecone-->>Celery: [{text, doc_id}] chunks
        Celery->>Celery: build_grounded_prompt(chunks, instruction)
        Celery->>LLM: chat.completions.create(prompt, max_tokens=1500)
        LLM-->>Celery: section_text
        Celery->>DB: Paper.{section} = section_text
        Celery->>DB: Paper.progress += step
    end

    Celery->>DB: Paper.status = completed, progress = 100
    Celery->>DB: Notification(user_id, "Paper ready")
    UI->>Flask: GET /api/papers/{id}/status
    Flask-->>UI: {status: "completed", progress: 100}
    UI->>UI: nav("editor")
```

---

## 29.4 Sequence Diagram — Plagiarism Scan

```mermaid
sequenceDiagram
    participant UI as Browser
    participant Flask as Flask API
    participant Chord as Celery Chord
    participant PreTask as preprocess_task
    participant ExactTask as exact_match_scan
    participant SemTask as semantic_scan
    participant AITask as ai_verification
    participant AggTask as aggregate_results
    participant DB as PostgreSQL

    UI->>Flask: POST /api/papers/{id}/plagiarism/scan
    Flask->>DB: ScanTask(status=pending)
    Flask->>Chord: chord(preprocess → [exact, semantic] → ai_verify → aggregate)
    Flask-->>UI: 202 {scan_task_id}

    Chord->>PreTask: preprocess_task(text)
    PreTask->>DB: ScanTask.progress=15, step="Tokenizing..."
    PreTask-->>Chord: {sentences, clean_text}

    par Parallel execution
        Chord->>ExactTask: exact_match_scan(prep_data)
        ExactTask->>DB: trigram similarity query
        ExactTask-->>Chord: {exact_matches}
    and
        Chord->>SemTask: semantic_scan(prep_data)
        SemTask->>SemTask: SentenceTransformer.encode()
        SemTask-->>Chord: {semantic_matches}
    end

    Chord->>AITask: ai_verification([exact+semantic matches])
    AITask->>AITask: call_ai(verify each match)
    AITask-->>Chord: {verified_matches}

    Chord->>AggTask: aggregate_results(all_results)
    AggTask->>DB: ScanTask.status=completed, results={...}, overall_score=14.7
    AggTask->>DB: Notification(user_id, "Plagiarism scan complete")
```

---

## 29.5 Activity Diagram — User Registration

```mermaid
flowchart TD
    START([User opens app]) --> LAND[View Landing Page]
    LAND --> REG[Click Register]
    REG --> FORM[Fill registration form]
    FORM --> VALIDATE{Validate input}
    VALIDATE -- Invalid --> ERROR[Show inline error]
    ERROR --> FORM
    VALIDATE -- Valid --> API[POST /api/auth/register]
    API --> DUP{Email duplicate?}
    DUP -- Yes --> DUP_ERR[409 Email taken]
    DUP_ERR --> FORM
    DUP -- No --> CREATE[Create User in DB]
    CREATE --> OTP_EMAIL[Send OTP email]
    OTP_EMAIL --> OTP_SCREEN[OTP verification screen]
    OTP_SCREEN --> ENTER_OTP[User enters OTP]
    ENTER_OTP --> VERIFY{OTP valid?}
    VERIFY -- No --> OTP_ERR[Invalid OTP message]
    OTP_ERR --> ENTER_OTP
    VERIFY -- Yes --> MARK[is_verified = True]
    MARK --> COOKIES[Set auth cookies]
    COOKIES --> DASHBOARD[Redirect to Dashboard]
    DASHBOARD --> END([User is logged in])
```
