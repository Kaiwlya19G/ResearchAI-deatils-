# 24 — Security Audit

> **Back to Index**: [00_index.md](00_index.md)

---

## 24.1 Security Summary

| Security Control | Status | Implementation |
|-----------------|--------|----------------|
| JWT XSS Protection | ✅ Implemented | HTTPOnly cookies (JS cannot read) |
| CSRF Protection | ✅ Implemented | Double-submit cookie pattern |
| SQL Injection | ✅ Protected | SQLAlchemy ORM (parameterized queries) |
| IDOR Prevention | ✅ Protected | UUID PKs + ownership checks |
| Account Lockout | ✅ Implemented | 5 failures → 15-minute lockout |
| Token Revocation | ✅ Implemented | Redis blocklist (O(1) lookup) |
| Token Rotation | ✅ Implemented | Refresh token rotated on use |
| Rate Limiting | ✅ Implemented | Flask-Limiter (Redis-backed) |
| Password Hashing | ✅ Implemented | PBKDF2-SHA256 (Werkzeug) |
| Multi-Tenant Isolation | ✅ Implemented | Namespace + metadata (Pinecone), RLS (PostgreSQL) |
| Soft Deletes | ✅ Implemented | No hard deletes (audit trail preserved) |
| GDPR Vector Cleanup | ✅ Implemented | Async Celery tasks purge Pinecone on delete |
| Input Validation | ⚠️ Partial | Manual validation, no schema library |
| HTTPS in Production | ⚠️ Config required | `JWT_COOKIE_SECURE = True` in prod config |
| API Key Storage | ⚠️ Env vars only | No secrets manager (Vault/AWS SM) |
| Password in Logs | ✅ Protected | Passwords never logged |

---

## 24.2 Authentication Security

### JWT HTTPOnly Cookies
```python
JWT_TOKEN_LOCATION = ["cookies"]
JWT_COOKIE_HTTPONLY = True        # JS cannot read — XSS proof
JWT_COOKIE_SECURE = True          # HTTPS only (production)
JWT_COOKIE_SAMESITE = "Lax"      # Blocks cross-site requests
JWT_COOKIE_CSRF_PROTECT = True    # Double-submit CSRF pattern
```

### CSRF Defense
Every state-changing request requires `X-CSRF-TOKEN` header:
```javascript
// Frontend reads non-HttpOnly CSRF cookie
const csrfToken = document.cookie.match(/csrf_access_token=([^;]+)/)?.[1];
fetch('/api/...', { headers: { 'X-CSRF-TOKEN': csrfToken } });
```
If the header is missing or wrong, Flask-JWT-Extended returns 401.

### Account Lockout
```python
if user.failed_login_attempts >= 5:
    user.locked_until = datetime.now(timezone.utc) + timedelta(minutes=15)
```
Generic error message returned to prevent account enumeration:
```python
# SAME error message for "email not found" and "wrong password"
return jsonify({"error": "Invalid credentials"}), 401
```

---

## 24.3 Authorization

### Ownership Verification (IDOR Prevention)
Every route that accesses a resource verifies ownership:
```python
paper = Paper.query.get_or_404(paper_id)
project = Project.query.filter_by(
    id=paper.project_id,
    user_id=request.user_id  # JWT identity
).first_or_404()
# If user doesn't own this project → 404 (not 403, to prevent resource enumeration)
```

### Role-Based Access
JWT claims carry the user's role and scope. Admin routes check role:
```python
claims = get_jwt()
if claims.get("role") not in ("admin", "super_admin"):
    return jsonify({"error": "Insufficient permissions"}), 403
```

### Row-Level Security (PostgreSQL)
`db_session_events.py` sets PostgreSQL session variables before each query:
```sql
SET LOCAL app.current_user_id = '<uuid>';
SET LOCAL app.current_role = '<role>';
SET LOCAL app.current_scope = '<scope>';
```

PostgreSQL RLS policies then auto-enforce data isolation. Even if Flask's ownership check fails (bug), the DB itself blocks cross-tenant reads.

---

## 24.4 Injection Attacks

### SQL Injection
All database queries use SQLAlchemy's parameterized query system:
```python
# SAFE: ORM query (parameterized internally)
Paper.query.filter_by(id=paper_id, project_id=project.id).first()

# SAFE: Raw SQL with parameters
sql = sa_text("SELECT * FROM documents WHERE similarity(text, :q) > 0.02")
db.session.execute(sql, {"q": clean_text})  # :q is a bound parameter
```

No string interpolation in SQL queries.

### Prompt Injection
AI prompts include untrusted user content (paper text, document content). Mitigation:
- System prompt is hardcoded (not user-controllable)
- User content goes in the `user` role message, not `system`
- LLM outputs are treated as untrusted text, never executed

> [!WARNING]
> A determined user could craft document content designed to manipulate the LLM's output for their paper. Full prompt injection prevention would require output filtering or sandboxed generation.

---

## 24.5 Data Privacy

### Sensitive Data Storage
| Data | Storage | Protection |
|------|---------|-----------|
| Passwords | PostgreSQL `password_hash` | PBKDF2-SHA256 |
| 2FA secrets | PostgreSQL `tfa_secret` | DB encryption at rest |
| Refresh tokens | PostgreSQL `token_hash` | SHA-256 hash (plaintext never stored) |
| API keys | PostgreSQL `key_hash` | Hashed |
| JWT access tokens | HTTPOnly cookie | Client-side: inaccessible to JS |
| Documents | S3 / local disk | S3: server-side encryption |

### GDPR Controls
- **Soft deletes**: Records preserved with `deleted_at` (audit trail)
- **Right to erasure**: Pinecone vectors deleted on document/project delete
- **Data export**: `GET /api/user/export` (future feature)
- **Anonymization**: Admin can anonymize user data (`email = anon-uuid@deleted.com`)

---

## 24.6 Network Security

### CORS
```python
CORS(app, 
    resources={r"/api/*": {"origins": [frontend_url]}},
    supports_credentials=True  # Required for cookies
)
```

Only `frontend_url` (from env) can make CORS requests. `*` is never allowed.

### Rate Limiting
```python
# Global defaults
limiter = Limiter(
    default_limits=["2000 per hour", "100 per minute"]
)

# Auth endpoints: tighter limit
limiter.limit("10 per minute")(auth_bp)
```

Rate limits are enforced per IP address and backed by Redis (persistent across restarts).

---

## 24.7 Known Security Gaps (Technical Debt)

> [!CAUTION]
> The following items should be addressed before a major production launch.

1. **No input schema validation library**: Manual validation in every route handler is error-prone. Should add Marshmallow or Pydantic.

2. **API keys in environment variables**: No secrets rotation, no audit trail. Should integrate AWS Secrets Manager or HashiCorp Vault.

3. **Dev index in production**: Pinecone uses `researchai-dev` index. Should be `researchai-prod` in production.

4. **No request signing for webhooks**: The Razorpay/plagiarism callback endpoints at `/api/plagiarism/callback` have TODO security checks — currently accepts any POST.

5. **`debug=True` in development**: `Config.DEBUG = os.environ.get("DEBUG", "True") == "True"` — defaults to debug mode. Production must explicitly set `DEBUG=False`.

6. **No output sanitization on AI responses**: LLM outputs are stored directly to DB and rendered in the frontend. Should strip any HTML/script content from LLM responses.
