# 22 — Logging Architecture

> **Back to Index**: [00_index.md](00_index.md)

---

## 22.1 Logging Strategy

ResearchAI uses Python's standard `logging` module with a custom `get_logger()` factory. Every module maintains its own named logger (`logger = get_logger(__name__)`), enabling fine-grained log filtering.

---

## 22.2 Logger Factory (`utils/logger.py`)

```python
import logging

def get_logger(name: str) -> logging.Logger:
    logger = logging.getLogger(name)
    if not logger.handlers:
        handler = logging.StreamHandler()
        formatter = logging.Formatter(
            "%(asctime)s | %(name)s | %(levelname)s | %(message)s",
            datefmt="%Y-%m-%dT%H:%M:%S"
        )
        handler.setFormatter(formatter)
        logger.addHandler(handler)
        logger.setLevel(logging.INFO)
    return logger
```

---

## 22.3 Log Types

### Application Logs
General request and operation logs:
```
2026-06-27T10:30:45 | routes.paper | INFO | Paper generation started paper_id=abc-123
2026-06-27T10:31:20 | tasks.paper_tasks | INFO | Section 'methodology' generated: 487 words
2026-06-27T10:31:25 | tasks.paper_tasks | INFO | Paper generation complete paper_id=abc-123
```

### AI Router Metrics (`ai_router_metric`)
Structured logs for every AI call:
```
ai_router_metric | task=paper_generation provider=glm model=glm-5.1 
tokens_in~1243 duration_ms=3421 success=True fallback_used=False fallback_reason=
```

Fields logged:
- `task` — feature task type
- `provider` — glm/deepseek/gemini/openai/ollama
- `model` — exact model name
- `tokens_in~` — estimated input tokens
- `duration_ms` — latency in milliseconds
- `success` — True/False
- `fallback_used` — True if primary provider failed
- `fallback_reason` — which provider failed and why

### RAG Retrieval Stats
```
RETRIEVER_STATS | namespace='project_abc' | query='methodology research design' 
| retrieved=15 | kept=9 | filtered=6 | min_score=0.1234 | max_score=0.8901
```

### Circuit Breaker Events
```
ai_router | Gemini circuit OPENED for 60s — will skip until cooldown
ai_router | Provider selected: DeepSeek (deepseek-ai/deepseek-v4-flash)
ai_router | DeepSeek HARD quota exhausted
ai_router | Gemini circuit CLOSED (reset after cooldown)
```

### Celery Task Logs
```
embed_document started  doc_id=xyz  project_id=abc  text_len=42341
embed_document: 48 chunks created for doc_id=xyz
_upsert_vectors: idempotency cleanup — purged stale vectors for doc_id=xyz
embed_document: upserted 48 vectors  doc_id=xyz  namespace=project_abc
```

### GDPR Alert Logs (Critical Level)
```
CRITICAL: GDPR ALERT: Failed to purge vectors for doc_id=xyz after 3 retries.
Manual cleanup required in namespace='project_abc'.
```

---

## 22.4 Database-Backed Cost Logs (`UsageLog`)

Every successful AI call writes a structured `UsageLog` entry:

```json
{
  "action": "api_token_usage",
  "meta": {
    "api_name": "deepseek",
    "model": "deepseek-ai/deepseek-v4-flash",
    "task_type": "plagiarism",
    "tokens_in": 1243,
    "tokens_out": 387,
    "total_tokens": 1630,
    "cost_usd": 0.000282,
    "cost_inr": 0.0238,
    "latency_ms": 892,
    "fallback_reason": ""
  }
}
```

These logs power the Super Admin **API Usage Analytics** dashboard showing per-provider costs, token consumption, and latency trends.

---

## 22.5 Security Audit Logs

Key security events are logged:

```python
# Failed login
logger.warning("AUTH: Failed login attempt for email=%s (attempt=%d)", email, user.failed_login_attempts)

# Account locked
logger.warning("AUTH: Account locked for user_id=%s until %s", user.id, user.locked_until)

# Invalid refresh token
logger.warning("AUTH: Invalid refresh token from ip=%s", request.remote_addr)

# RLS violation attempt (hypothetical)
logger.critical("RLS: Possible tenant isolation bypass detected user_id=%s", user_id)
```

---

## 22.6 GDPR Compliance Logs

```python
# Pinecone vector cleanup
logger.info("delete_document_vectors: purged vectors for doc_id=%s from namespace='%s'", doc_id, ns)
logger.info("delete_all_project_vectors: wiped namespace='%s'", namespace)

# On failure
logger.critical(
    "GDPR ALERT: Failed to purge vectors for doc_id=%s after %d retries. Manual cleanup required.",
    document_id, self.max_retries
)
```

---

## 22.7 Log Levels by Severity

| Level | When Used |
|-------|-----------|
| `DEBUG` | Verbose tracing (embedding query details, chunk sizes) |
| `INFO` | Normal operations (task started/completed, provider selected) |
| `WARNING` | Recoverable issues (provider fallback, rate limit hit, low RAG scores) |
| `ERROR` | Failures that affect a single operation (embedding failed, LLM error) |
| `CRITICAL` | Platform-level failures (GDPR cleanup failed, DB unavailable) |

---

## 22.8 Log Aggregation (Recommended Production Setup)

Currently logs go to stdout/stderr (Gunicorn default). For production:

```
Application → stdout → 
    Gunicorn → syslog → 
        Logstash/Fluent Bit → 
            Elasticsearch / CloudWatch → 
                Kibana / Grafana dashboard
```

Recommended structured logging upgrade: Replace string formatting with `structlog` or add JSON formatter for easier parsing in log aggregators.
