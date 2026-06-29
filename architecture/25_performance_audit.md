# 25 — Performance Audit

> **Back to Index**: [00_index.md](00_index.md)

---

## 25.1 Performance Profile

| Operation | Typical Duration | Bottleneck |
|-----------|-----------------|-----------|
| Login | 50-200ms | bcrypt password check |
| Document upload + extraction | 1-10s | PDF text extraction |
| Document embedding (per doc) | 10-60s | SentenceTransformer encoding |
| Paper generation (per section) | 3-15s | LLM API latency |
| Full paper generation (11 sections) | 2-15 min | Sequential LLM calls |
| Plagiarism scan | 30s-5min | Semantic similarity for long papers |
| AI detection | 5-60s | LLM classifier + statistics |
| Humanizer | 5-30s | LLM rewrite API latency |
| DOCX export | 1-5s | python-docx rendering |

---

## 25.2 Current Performance Optimizations

### Embedding Model Cache
```python
# model_cache.py — loaded once per worker process
_model_singleton = SentenceTransformer("all-MiniLM-L6-v2", device="cpu")
```
Without this: 2-3 second model load on every embedding operation.  
With this: ~50ms per embedding operation after first load.

### Concurrent Diagram Scanning
```python
with ThreadPoolExecutor(max_workers=5) as executor:
    futures = {executor.submit(scan_batch, batch): idx for idx, batch in enumerate(batches)}
```
5 batches of 5 paragraphs scan in parallel, reducing diagram scanning from serial to parallel.

### Plagiarism Parallel Chord
Exact match and semantic scan run concurrently via Celery chord, halving total scan time.

### Connection Pooling
```python
SQLALCHEMY_ENGINE_OPTIONS = {
    "pool_size": 10,
    "max_overflow": 20,
    "pool_pre_ping": True,
}
```
Pre-ping prevents wasted time on stale connections.

### Signal-First AI Detection Bypass
```python
# Skip LLM call for obvious cases
if stat_score <= 15 or stat_score >= 85:
    return local_only_result  # No LLM call
```
Saves ~5-30s LLM latency for ~35% of detection requests.

---

## 25.3 Database Query Performance

### Current Indexing
- `users.email` — B-tree unique index (login: O(log n))
- `documents.extracted_text` — GiST trigram index (plagiarism candidate retrieval)
- `refresh_tokens.token_hash` — B-tree index (token verification)

### Missing Indexes (Performance Gaps)
> [!WARNING]
> The following queries run without proper indexes and will degrade with scale:

| Query | Missing Index | Impact |
|-------|--------------|--------|
| `Paper.query.filter_by(project_id=...)` | `papers.project_id` | Full table scan |
| `Citation.query.filter_by(paper_id=...)` | `citations.paper_id` | Full table scan |
| `Diagram.query.filter_by(paper_id=...)` | `diagrams.paper_id` | Full table scan |
| `ScanTask.query.filter_by(paper_id=...)` | `scan_tasks.paper_id` | Full table scan |
| `Notification.query.filter_by(user_id=...)` | `notifications.user_id` | Full table scan |
| `UsageLog.query.filter_by(user_id=...)` | `usage_logs.user_id` | Full table scan |

**Recommended migration**:
```python
# In a new migration
op.create_index('ix_papers_project_id', 'papers', ['project_id'])
op.create_index('ix_citations_paper_id', 'citations', ['paper_id'])
op.create_index('ix_diagrams_paper_id', 'diagrams', ['paper_id'])
op.create_index('ix_scan_tasks_paper_id', 'scan_tasks', ['paper_id'])
op.create_index('ix_notifications_user_id', 'notifications', ['user_id'])
op.create_index('ix_usage_logs_user_id', 'usage_logs', ['user_id'])
```

---

## 25.4 LLM Latency Optimization

### Current Approach
Sequential section generation: Each section waits for the previous to complete.

### Potential Improvement: Parallel Section Generation
Independent sections could be generated concurrently:
```python
# Sections with no dependencies on each other
PARALLEL_SECTIONS = [
    ["introduction", "literature_review", "research_gap"],  # Group 1
    ["methodology", "system_architecture"],                  # Group 2 (after Group 1)
    ["results", "discussion"],                               # Group 3
    ["conclusion", "future_work"],                           # Group 4
]

# Run each group in parallel via ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=3) as executor:
    futures = [executor.submit(generate_section, s, ctx) for s in group]
```

This could reduce generation time from 15 minutes to 5-6 minutes.

---

## 25.5 RAG Performance

### Current: top_k * 3 Over-Retrieval
```python
response = index.query(
    vector=query_vector,
    top_k=top_k * 3,   # Retrieve 3x more, then filter/diversify down to top_k
    include_metadata=True,
    namespace=namespace,
)
```
This retrieves 15 vectors to select the best 5 — ensuring diversity but adding network overhead.

### MIN_RAG_SIMILARITY = 0.20
Low threshold ensures maximum context coverage. If precision is preferred over recall, raise to 0.35.

---

## 25.6 Caching Opportunities (Not Yet Implemented)

> [!TIP]
> These caching layers could significantly improve response times:

| Data | Cache Key | TTL | Benefit |
|------|----------|-----|---------|
| User profile | `user:<uuid>` | 300s | Avoids DB hit on every request |
| Project list | `projects:<user_id>` | 60s | Dashboard loads faster |
| Paper status | `paper_status:<paper_id>` | 5s | Reduces polling DB hits |
| AI detection result | `ai_detect:<hash(text)>` | 3600s | Same text → same score |

A Redis-backed LRU cache using `flask-caching` would be the easiest implementation path.

---

## 25.7 Frontend Performance

### Current Approach
- All JS loaded inline in `index.html` (no code splitting)
- No lazy loading — all screens loaded upfront
- No client-side caching of API responses

### Optimization Opportunities
1. **Lazy load screen JS files** — only load screen code when navigated to
2. **Client-side cache** — cache paper/project data in `sessionStorage` for instant navigation
3. **Debounce editor saves** — currently saves on every keystroke, should debounce 1-2s
4. **Polling optimization** — exponential backoff on status polls instead of fixed 3s
