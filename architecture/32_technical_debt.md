# 32 — Technical Debt Report

> **Back to Index**: [00_index.md](00_index.md)

---

## 32.1 Debt Summary

| Category | Issues Found | Severity |
|----------|-------------|---------|
| Missing DB Indexes | 6 tables missing FK indexes | 🔴 High |
| No Input Schema Validation | Manual validation in ~20 routes | 🔴 High |
| Dev Index in Production | Pinecone uses `researchai-dev` | 🔴 High |
| Gemini Rate Limiter Per-Worker | Not cross-worker synchronized | 🟡 Medium |
| No Secrets Manager | API keys only in .env | 🟡 Medium |
| Paper Generation Sequential | Sections generated one at a time | 🟡 Medium |
| Large `paper_bp` Monolith | 2756-line single Blueprint file | 🟡 Medium |
| No Response Sanitization | LLM output stored raw | 🟡 Medium |
| Frontend No Code Splitting | All JS loaded upfront | 🟡 Medium |
| No Pagination on all endpoints | Some lists return unbounded results | 🟠 Low-Med |
| Hardcoded model names | Model strings duplicated in code | 🟠 Low |
| No integration tests | Only unit tests exist | 🟠 Low |
| `debug=True` default | DEBUG defaults to True | 🔴 High |

---

## 32.2 Critical Issues

### 🔴 Missing Database Indexes
**Impact**: Full table scans on every paper/citation/notification fetch. Will cause severe performance degradation at scale (>1,000 users).

```sql
-- Add in next migration
CREATE INDEX ix_papers_project_id ON papers(project_id);
CREATE INDEX ix_citations_paper_id ON citations(paper_id);
CREATE INDEX ix_diagrams_paper_id ON diagrams(paper_id);
CREATE INDEX ix_scan_tasks_paper_id ON scan_tasks(paper_id);
CREATE INDEX ix_notifications_user_id ON notifications(user_id);
CREATE INDEX ix_usage_logs_user_id ON usage_logs(user_id);
CREATE INDEX ix_usage_logs_created_at ON usage_logs(created_at DESC);
```

---

### 🔴 No Input Schema Validation Library
**Impact**: Inconsistent validation, missing edge cases, hard to audit security.

**Current state**:
```python
# Ad-hoc validation in every route
text = data.get("text", "")
if not text:
    return jsonify({"error": "text is required"}), 400
```

**Recommended**: Marshmallow schemas or Pydantic:
```python
from marshmallow import Schema, fields, validate

class HumanizeSchema(Schema):
    text = fields.Str(required=True, validate=validate.Length(min=1, max=50000))
    intensity = fields.Str(load_default="medium", validate=validate.OneOf(["light","medium","high"]))
```

---

### 🔴 Dev Index Used in Production
**Impact**: Risk of data loss if the dev index is cleared during testing.

**Fix**: Create `researchai-prod` index and update the constant:
```python
# utils/pinecone_search.py + tasks/embed_document.py
INDEX_NAME = os.environ.get("PINECONE_INDEX", "researchai-prod")
```

---

### 🔴 `DEBUG=True` Default
**Impact**: Debug mode in production exposes stack traces and the Werkzeug debugger.

```python
# config.py — DANGEROUS default
DEBUG = os.environ.get("DEBUG", "True") == "True"
```

**Fix**:
```python
DEBUG = os.environ.get("DEBUG", "False") == "True"
```

---

## 32.3 Medium Issues

### 🟡 Gemini Rate Limiter Not Cross-Worker
The `_GeminiLimiter` is a module-level singleton (per-process). With 4 Celery workers and 10 RPM limit:
- **Intended**: 10 requests/minute total
- **Actual**: 40 requests/minute (10 per worker × 4 workers)

**Fix**: Use Redis for cross-worker rate tracking:
```python
import time
import redis

def is_gemini_allowed(redis_client) -> bool:
    key = "gemini_rpm_counter"
    count = redis_client.incr(key)
    if count == 1:
        redis_client.expire(key, 60)
    return count <= 10
```

---

### 🟡 `paper_bp` 2756-Line Monolith
`routes/paper.py` handles paper CRUD, generation, editing, citations, plagiarism, humanizer, AI detection, and diagrams. This violates Single Responsibility Principle and makes debugging difficult.

**Recommended split**:
```
routes/paper_core.py      (~300 lines) — CRUD
routes/paper_generation.py (~300 lines) — generation + analysis
routes/paper_editor.py    (~400 lines) — editing, sections, versioning
routes/paper_citations.py (~400 lines) — citation management
routes/paper_analysis.py  (~500 lines) — plagiarism, humanizer, detection
routes/paper_diagrams.py  (~600 lines) — diagram studio
```

---

### 🟡 Sequential Paper Generation
Currently all 11 sections are generated one after another. This takes 2-15 minutes when sections could be parallelized.

**Parallel grouping possible**:
```
Group 1 (parallel): introduction, abstract (independent)
Group 2 (parallel): literature_review, research_gap (after abstract)
Group 3 (parallel): methodology, system_architecture (independent)
Group 4 (parallel): results, discussion, implementation (independent)
Group 5 (parallel): conclusion, future_work (after results)
```

---

### 🟡 LLM Output Stored Without Sanitization
```python
paper.introduction = call_ai(prompt, ...)  # Raw LLM output directly to DB
```

Risk: LLM could produce:
- HTML/script tags (XSS if rendered unsanitized in frontend)
- Markdown artifacts (backticks, headers)
- Control characters

**Fix**: Strip markdown and sanitize before saving:
```python
import bleach
text = call_ai(prompt)
text = re.sub(r'```[a-z]*\n?', '', text)  # Remove code fences
text = bleach.clean(text, tags=[], strip=True)  # Strip HTML
paper.introduction = text.strip()
```

---

## 32.4 Low-Medium Issues

### 🟠 Unbounded List Returns
Several endpoints return all records without pagination:
```python
# This could return thousands of citations
citations = Citation.query.filter_by(paper_id=paper_id).all()
return jsonify({"citations": [c.to_dict() for c in citations]})
```

**Fix**: Add pagination:
```python
page = request.args.get('page', 1, type=int)
per_page = min(request.args.get('per_page', 50, type=int), 100)
citations = Citation.query.filter_by(paper_id=paper_id).paginate(page=page, per_page=per_page)
```

---

### 🟠 Hardcoded Model Names
Model strings are duplicated across `config.py`, `ai_router.py`, and individual route files:
```python
# In ai_router.py
model="deepseek-ai/deepseek-v4-flash"

# In paper.py (diagram generation)
model="qwen/qwen2-72b-instruct"
```

**Fix**: Define all model names as constants in `config.py`:
```python
class ModelNames:
    DEEPSEEK_FLASH = "deepseek-ai/deepseek-v4-flash"
    DEEPSEEK_PRO   = "deepseek-ai/deepseek-v4-0324"
    GLM            = "glm-5.1"
    QWEN_72B       = "qwen/qwen2-72b-instruct"
    SDXL           = "stabilityai/stable-diffusion-xl"
    GEMINI_FLASH   = "gemini-2.0-flash"
```

---

### 🟠 Typo in Core Folder Name
```
paper_genration/   ← should be paper_generation/
```
This typo is in the actual filesystem path. Renaming would require updating all imports. Low priority but affects code quality impression.

---

## 32.5 Debt Priority Matrix

```
HIGH IMPACT, LOW EFFORT (Do First):
  ✓ Add missing DB indexes (30-min migration)
  ✓ Fix DEBUG default to False
  ✓ Rename Pinecone index env var

HIGH IMPACT, HIGH EFFORT (Plan for Sprint):
  ✓ Add Marshmallow/Pydantic validation
  ✓ Split paper_bp monolith
  ✓ Parallel paper generation
  ✓ Redis-backed Gemini rate limiter

LOW IMPACT, LOW EFFORT (Opportunistic):
  ✓ Centralize model name constants
  ✓ Add pagination to list endpoints
  ✓ Sanitize LLM output before DB save
```
