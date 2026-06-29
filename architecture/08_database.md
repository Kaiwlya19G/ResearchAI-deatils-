# 08 — Database Documentation

> **Back to Index**: [00_index.md](00_index.md)

---

## 8.1 ER Diagram

```mermaid
erDiagram
    institutions {
        uuid id PK
        string name
        string domain
        string plan
        int max_users
        string contact_email
        bool is_active_org
    }

    users {
        uuid id PK
        string first_name
        string last_name
        string email UK
        string password_hash
        string role
        uuid institute_id FK
        string plan
        datetime plan_expires
        bool is_suspended
        bool is_verified
        bool tfa_enabled
        string tfa_secret
        int failed_login_attempts
        datetime locked_until
        datetime last_login
    }

    refresh_tokens {
        uuid id PK
        uuid user_id FK
        string token_hash
        datetime expires_at
        datetime revoked_at
        string user_agent
        string ip_address
    }

    projects {
        uuid id PK
        uuid user_id FK
        string title
        string domain
        string citation_style
        string template
        string target_pages
        string status
        json sections_config
    }

    documents {
        uuid id PK
        uuid project_id FK
        string filename
        string file_type
        int file_size
        string s3_key
        string local_path
        text extracted_text
        bool is_scanned
        bool ocr_applied
        string status
        json author_metadata
        int publication_year
        string journal_name
        string doi
        text abstract_summary
    }

    ai_analyses {
        uuid id PK
        uuid project_id FK
        json keywords
        json topics
        json research_gaps
        text methodology
        json references
        json generated_titles
        string status
    }

    papers {
        uuid id PK
        uuid project_id FK
        string title
        text abstract
        text introduction
        text literature_review
        text research_gap
        text methodology
        text system_architecture
        text implementation
        text results
        text discussion
        text conclusion
        text future_work
        text references_text
        json raw_content
        string citation_style
        int word_count
        int page_count
        float plagiarism_score
        float ai_score
        string quality_grade
        string status
        int progress
        int version
    }

    paper_versions {
        uuid id PK
        uuid paper_id FK
        int version
        json snapshot
        uuid created_by FK
    }

    citations {
        uuid id PK
        uuid paper_id FK
        uuid project_id FK
        uuid document_id FK
        int number
        string authors
        string title
        string journal
        int year
        string doi
        string url
        string style
        text formatted
        bool is_verified
    }

    diagrams {
        uuid id PK
        uuid paper_id FK
        string diagram_type
        string title
        text description
        json data
        string image_url
    }

    scan_tasks {
        uuid id PK
        uuid paper_id FK
        uuid user_id FK
        string task_type
        string status
        int progress
        string current_step
        json results
        float overall_score
        text error_msg
    }

    subscriptions {
        uuid id PK
        uuid user_id FK
        string plan
        string status
        string payment_gateway
        string gateway_sub_id
        float amount
        string billing_cycle
        datetime starts_at
        datetime expires_at
    }

    invoices {
        uuid id PK
        uuid user_id FK
        uuid subscription_id FK
        string invoice_number UK
        float amount
        string status
        string payment_id
        datetime issued_at
    }

    usage_logs {
        uuid id PK
        uuid user_id FK
        string action
        string resource_id
        json meta
    }

    notifications {
        uuid id PK
        uuid user_id FK
        string type
        string title
        text body
        bool is_read
        string resource_id
    }

    platform_configs {
        uuid id PK
        string key UK
        text value
        string data_type
        string category
        uuid updated_by FK
    }

    api_keys {
        uuid id PK
        string name
        string key_hash
        string service
        bool is_key_active
        datetime expires_at
        uuid created_by FK
    }

    collaborators {
        uuid id PK
        uuid project_id FK
        uuid user_id FK
        string role
        bool accepted
    }

    support_assignments {
        uuid id PK
        uuid agent_id FK
        uuid user_id FK
        uuid assigned_by FK
        bool is_active
    }

    institutions ||--o{ users : "has members"
    users ||--o{ projects : "owns"
    users ||--o{ refresh_tokens : "has sessions"
    users ||--o{ invoices : "has"
    users ||--o{ subscriptions : "subscribes"
    users ||--o{ usage_logs : "generates"
    users ||--o{ notifications : "receives"
    projects ||--o{ documents : "contains"
    projects ||--o{ papers : "generates"
    projects ||--|| ai_analyses : "has"
    projects ||--o{ collaborators : "shared with"
    papers ||--o{ citations : "has"
    papers ||--o{ diagrams : "contains"
    papers ||--o{ paper_versions : "versioned by"
    papers ||--o{ scan_tasks : "scanned by"
    documents ||--o{ citations : "sourced from"
```

---

## 8.2 Table Reference

### `users`
Core identity and authorization table.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | Cryptographically secure, prevents IDOR |
| `email` | VARCHAR(255) | Unique, indexed |
| `password_hash` | VARCHAR(512) | PBKDF2-SHA256 via Werkzeug |
| `role` | VARCHAR(50) | `student/professor/phd/admin/super_admin/marketing/support_agent` |
| `institute_id` | UUID FK | NULL = individual user |
| `plan` | VARCHAR(30) | `free/starter/pro/institution` |
| `tfa_secret` | VARCHAR(32) | pyotp TOTP secret |
| `failed_login_attempts` | INT | Lockout counter (resets on success) |
| `locked_until` | TIMESTAMP | NULL = not locked |
| `deleted_at` | TIMESTAMP | NULL = active (soft delete) |

**Lifecycle**: Created on registration. Soft-deleted by admin. Hard-deleted only for GDPR "right to erasure" (future feature).

---

### `projects`
Research project container. The `id` UUID doubles as the Pinecone namespace.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | Also the Pinecone namespace: `project_<id>` |
| `user_id` | UUID FK | Owner |
| `citation_style` | VARCHAR(20) | `IEEE/APA/MLA/Chicago` |
| `template` | VARCHAR(50) | `IEEE Conference/Springer/Elsevier/...` |
| `sections_config` | JSON | Custom section definitions |
| `status` | VARCHAR(30) | `draft/processing/generating/completed/failed` |

---

### `documents`
Uploaded reference files.

| Column | Type | Notes |
|--------|------|-------|
| `extracted_text` | TEXT | Full plain-text content (used for RAG, plagiarism) |
| `is_scanned` | BOOL | PDF was scanned image (OCR was needed) |
| `ocr_applied` | BOOL | OCR was actually applied |
| `status` | VARCHAR(20) | `uploaded/processing/done/embedded/error` |
| `s3_key` | VARCHAR(512) | AWS S3 object key |
| `local_path` | VARCHAR(512) | Local filesystem path |
| `author_metadata` | JSON | Extracted author list `[{name, affiliation}]` |
| `doi` | VARCHAR(255) | Extracted DOI |

**Trigram index**: PostgreSQL `pg_trgm` extension is used on `extracted_text` for fuzzy text matching in the plagiarism scan. The trigram similarity query is: `similarity(extracted_text, :query) > 0.02`.

---

### `papers`
The generated research paper. Each standard section is its own `TEXT` column for direct SQL access.

| Column | Type | Notes |
|--------|------|-------|
| `raw_content` | JSON | `{section_key: raw_text_with_uuids}` — stores cite/diagram tags |
| `citation_style` | VARCHAR(20) | Applied during export |
| `plagiarism_score` | FLOAT | 0-100, set after scan completion |
| `ai_score` | FLOAT | 0-100, set after AI detection |
| `progress` | INT | 0-100, updated during generation |
| `current_section` | VARCHAR(60) | Section being generated ("methodology") |
| `version` | INT | Incremented on each save |
| `is_flagged` | BOOL | Admin/super_admin flag |

---

### `scan_tasks`
Tracks async plagiarism scan progress.

| Column | Type | Notes |
|--------|------|-------|
| `status` | VARCHAR(20) | `pending/processing/completed/failed` |
| `progress` | INT | 0-100 |
| `current_step` | VARCHAR(100) | Human-readable step name ("Semantic Analysis...") |
| `results` | JSON | Full scan results including flagged sentences |
| `overall_score` | FLOAT | Final plagiarism percentage |

---

### `usage_logs`
Token usage and cost tracking per API call.

| Column | Type | Notes |
|--------|------|-------|
| `action` | VARCHAR(50) | `api_token_usage/paper_generate/export/...` |
| `meta` | JSON | `{api_name, model, tokens_in, tokens_out, cost_usd, cost_inr, latency_ms}` |

---

### `platform_configs`
Key-value store for super admin platform settings.

| Column | Type | Notes |
|--------|------|-------|
| `key` | VARCHAR(100) | Unique setting name |
| `data_type` | VARCHAR(20) | `string/int/bool/json` |
| `category` | VARCHAR(50) | Groups related settings |

---

## 8.3 BaseModel — Common Fields (All Tables)

```python
id         UUID PK   (uuid4, cryptographically secure)
created_at TIMESTAMP (server_default=func.now(), timezone=True)
updated_at TIMESTAMP (auto-updates on save, timezone=True)
deleted_at TIMESTAMP (NULL = active, non-NULL = soft-deleted)
```

All models inherit these. No table has a plain integer `id`.

---

## 8.4 Indexes

| Table | Column | Index Type | Purpose |
|-------|--------|-----------|---------|
| `users` | `email` | B-tree UNIQUE | Login lookup |
| `users` | `institute_id` | B-tree | Admin scoping |
| `refresh_tokens` | `user_id` | B-tree | Session lookup |
| `refresh_tokens` | `token_hash` | B-tree | Token verification |
| `support_assignments` | `agent_id` | B-tree | RLS policy |
| `support_assignments` | `user_id` | B-tree | RLS policy |
| `documents` | `extracted_text` | GiST (pg_trgm) | Trigram fuzzy search |

---

## 8.5 Soft Delete Policy

**No hard deletes in production** (except GDPR requests). Records are soft-deleted by setting `deleted_at = now()`.

The `is_active` property on `BaseModel` returns `deleted_at is None`.

For `User`, `is_active` is overridden to also check `is_suspended`:
```python
@property
def is_active(self):
    return self.deleted_at is None and not self.is_suspended
```
