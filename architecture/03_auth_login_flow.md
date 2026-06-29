# 03 — Authentication & Login Flow

> **Back to Index**: [00_index.md](00_index.md)

---

## 3.1 Authentication Overview

ResearchAI uses **Flask-JWT-Extended** in **HTTPOnly cookie mode**. Tokens are never exposed to JavaScript — they ride in `HttpOnly; SameSite=Lax; Secure` cookies, making XSS token theft impossible.

### Token Types

| Token | Lifetime | Storage | Scope |
|-------|----------|---------|-------|
| Access Token | 24 hours | HTTPOnly cookie (`/`) | All API calls |
| Refresh Token | 30 days | HTTPOnly cookie (`/api/auth/`) | Token rotation only |

### CSRF Protection
Every state-changing request (`POST`, `PUT`, `DELETE`) must include a `X-CSRF-TOKEN` header. The CSRF token is embedded in a readable (non-HttpOnly) cookie that JavaScript can access and forward as a header. This is the **double-submit pattern**.

---

## 3.2 Login Sequence Diagram

```mermaid
sequenceDiagram
    participant Browser
    participant Flask as Flask /api/auth/login
    participant Redis as Redis Blocklist
    participant DB as PostgreSQL

    Browser->>Flask: POST /api/auth/login {email, password}
    Flask->>DB: User.query.filter_by(email=email)
    DB-->>Flask: user record

    alt Account Locked
        Flask-->>Browser: 403 {error: "Account locked"}
    end

    Flask->>Flask: check_password(password_hash)

    alt Wrong Password
        Flask->>DB: record_failed_login() [increment counter, lock at 5]
        Flask-->>Browser: 401 {error: "Invalid credentials"}
    end

    alt 2FA Enabled
        Flask-->>Browser: 200 {requires_2fa: true, temp_token}
        Browser->>Flask: POST /api/auth/2fa/verify {code}
        Flask->>Flask: pyotp.TOTP(tfa_secret).verify(code)
    end

    Flask->>Flask: create_access_token(user_id, claims)
    Flask->>Flask: create_refresh_token(user_id)
    Flask->>DB: store_refresh_token(SHA256(refresh_token))
    Flask->>DB: record_successful_login()
    Flask-->>Browser: 200 + Set-Cookie: access_token, refresh_token, csrf_token
```

---

## 3.3 Token Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Active: Login Success
    Active --> Refreshing: Access token expires (24h)
    Refreshing --> Active: POST /api/auth/refresh → new access token
    Refreshing --> LoggedOut: Refresh token expired (30d) or revoked
    Active --> Revoked: POST /api/auth/logout
    Revoked --> [*]: JTI added to Redis blocklist (TTL = remaining lifetime)
    LoggedOut --> [*]
```

### JWT Claims Structure
Every JWT access token carries these additional claims (embedded at login, no DB hit needed per request):

```json
{
  "sub": "uuid-of-user",
  "role": "student",
  "email": "user@example.com",
  "scope": "own",
  "institute_id": null,
  "exp": 1234567890,
  "jti": "unique-token-id"
}
```

**Scope values**: `global` (super_admin), `institute` (admin), `public` (marketing), `assigned` (support_agent), `own` (all other roles).

---

## 3.4 Authentication Middleware (`token_required`)

Every protected route uses the `@token_required` decorator defined in `routes/user.py`:

```python
def token_required(f):
    @wraps(f)
    @jwt_required()
    def decorated(*args, **kwargs):
        request.user_id = get_jwt_identity()
        request.user_claims = get_jwt()
        return f(*args, **kwargs)
    return decorated
```

Flask-JWT-Extended handles:
1. Extracting the JWT from the `access_token_cookie`
2. Verifying signature and expiry
3. Calling `check_if_token_revoked()` → Redis O(1) lookup
4. Populating `flask.g` with identity

---

## 3.5 Registration Flow

```mermaid
flowchart TD
    A[POST /api/auth/register] --> B{Valid email format?}
    B -- No --> C[400 Invalid email]
    B -- Yes --> D{Email already exists?}
    D -- Yes --> E[409 Email taken]
    D -- No --> F[Create User record]
    F --> G[set_password - bcrypt hash]
    G --> H[is_verified = False]
    H --> I[Send OTP email]
    I --> J[200 + Set cookies]
    J --> K[Browser shows OTP screen]
    K --> L[POST /api/auth/verify-email]
    L --> M{OTP valid?}
    M -- Yes --> N[is_verified = True]
    M -- No --> O[400 Invalid OTP]
```

---

## 3.6 Password Security

- Passwords are hashed using **Werkzeug's `generate_password_hash`** (PBKDF2-SHA256 by default).
- Plain-text passwords are **never stored or logged**.
- Account lockout: **5 failed attempts → 15-minute lockout** (`locked_until` field on User model).
- Anti-enumeration: both "email not found" and "wrong password" return the same generic error message.

---

## 3.7 OAuth Flow (Google / Facebook)

```mermaid
sequenceDiagram
    Browser->>Flask: GET /api/auth/oauth/google
    Flask-->>Browser: Redirect to Google OAuth consent
    Browser->>Google: User grants permission
    Google-->>Browser: Redirect to /api/auth/oauth/google/callback?code=X
    Browser->>Flask: GET /api/auth/oauth/google/callback?code=X
    Flask->>Google: Exchange code for token + profile
    Flask->>DB: Find or create User(oauth_provider=google, oauth_id=...)
    Flask-->>Browser: 200 + Set-Cookie (same as normal login)
```

---

## 3.8 Two-Factor Authentication (2FA / TOTP)

- Uses **pyotp** (RFC 6238 TOTP implementation).
- `tfa_secret` stored in the `users` table (encrypted at rest by PG disk encryption).
- When `tfa_enabled = True`: login returns `{requires_2fa: true}`, browser redirects to 2FA screen.
- POST `/api/auth/2fa/verify` validates the 6-digit code against `pyotp.TOTP(secret).verify()`.
- 30-second time window is enforced automatically by pyotp.

---

## 3.9 Logout Flow

```mermaid
flowchart LR
    A[POST /api/auth/logout] --> B[Get JTI from current access token]
    B --> C[Redis.setex - blocklist:jti - TTL=remaining lifetime]
    C --> D[unset_jwt_cookies - clear all cookies]
    D --> E[200 Logged out]
```

The JTI (JWT ID) is stored in Redis with a TTL equal to the token's remaining valid lifetime. When the token would have expired, the Redis key auto-deletes — no manual cleanup needed.

---

## 3.10 Token Refresh Flow

```mermaid
sequenceDiagram
    Browser->>Flask: POST /api/auth/refresh (refresh_token cookie)
    Flask->>Flask: jwt_required(refresh=True)
    Flask->>DB: Check refresh_token.token_hash in DB
    Flask->>DB: Revoke old refresh token (revoked_at = now)
    Flask->>Flask: create_access_token(user_id)
    Flask->>Flask: create_refresh_token(user_id) [rotation]
    Flask->>DB: Store new refresh_token hash
    Flask-->>Browser: 200 + new access_token + new refresh_token cookies
```

Token rotation ensures that stolen refresh tokens become invalid after one use.

---

## 3.11 Protected Route Authorization Matrix

| Route Pattern | Required Role | Scope Check |
|---------------|--------------|-------------|
| `/api/user/*` | Any authenticated | `own` — user_id match |
| `/api/projects/*` | Any authenticated | Project.user_id == jwt.sub |
| `/api/papers/*` | Any authenticated | Paper → Project.user_id == jwt.sub |
| `/api/admin/*` | `admin` or `super_admin` | institute_id match |
| `/api/super/*` | `super_admin` only | Global |
| `/api/paraphraser/*` | Any authenticated | own |
| `/api/plagiarism/*` | Any authenticated | own |

---

## 3.12 PostgreSQL Row-Level Security (RLS)

`db_session_events.py` listens for SQLAlchemy `before_cursor_execute` events and injects:

```sql
SET LOCAL app.current_user_id = '<uuid>';
SET LOCAL app.current_role = '<role>';
SET LOCAL app.current_scope = '<scope>';
```

PostgreSQL RLS policies then enforce these at the DB level automatically, providing a second layer of data isolation even if Flask-level checks are bypassed.
