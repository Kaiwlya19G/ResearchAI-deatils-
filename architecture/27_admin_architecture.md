# 27 — Admin Architecture

> **Back to Index**: [00_index.md](00_index.md)

---

## 27.1 Admin Hierarchy

```
super_admin
    │
    ├── Platform-wide access (ALL data, ALL institutions)
    ├── Model router config (provider chains, API keys)
    ├── Platform config (pricing, plan limits, feature flags)
    ├── Infrastructure monitoring
    └── Cost analytics

admin (Institution Admin)
    │
    ├── Own institution's data only (enforced by RLS)
    ├── User management (invite, suspend, role change)
    ├── Paper monitoring (flag, review)
    ├── Analytics (institution-scoped)
    └── Billing management (institution plan)

support_agent
    │
    └── Assigned users' data only (SupportAssignment table)
        ├── View user papers (read-only)
        └── View user profile (read-only)

marketing
    │
    └── Aggregated public data only
        ├── Platform statistics
        └── Landing page content
```

---

## 27.2 Institution Admin (`/api/admin`)

**Route**: `routes/admin.py`  
**Role Required**: `admin` or `super_admin`  
**Data Scope**: Enforced by `institute_id` in JWT

### Admin Dashboard
`GET /api/admin/dashboard`
```json
{
  "total_users": 245,
  "active_users_30d": 180,
  "papers_generated": 1240,
  "total_storage_gb": 12.5,
  "plan": "institution",
  "max_users": 500
}
```

### User Management
```
GET  /api/admin/users           → List institution users (paginated)
POST /api/admin/users/invite    → Invite user to institution
PUT  /api/admin/users/<id>      → Change role/status
POST /api/admin/users/<id>/suspend  → Suspend user account
```

### Paper Monitoring
```
GET  /api/admin/papers          → All papers for institution users
POST /api/admin/papers/<id>/flag → Flag paper for review
GET  /api/admin/papers/<id>     → View paper (admin read access)
```

### Analytics
```
GET /api/admin/analytics        → Usage stats (generation counts, scan counts)
GET /api/admin/monitoring       → System health indicators
```

---

## 27.3 Super Admin (`/api/super`)

**Route**: `routes/super_admin.py`  
**Role Required**: `super_admin` only  
**Data Scope**: Global (ALL institutions, ALL users)

### Super Admin Dashboard
`GET /api/super/overview`
```json
{
  "total_users": 15240,
  "total_institutions": 47,
  "papers_generated_today": 342,
  "active_celery_tasks": 8,
  "ai_cost_today_usd": 12.45,
  "revenue_this_month_inr": 485000
}
```

### Model Router Configuration
`GET/POST /api/super/model-router`

Allows super admin to change provider routing chains at runtime:
```json
{
  "paper_generation": ["glm", "gemini", "openai"],
  "plagiarism": ["deepseek", "gemini"],
  "humanizer": ["deepseek", "gemini"]
}
```

Changes can be:
1. Saved to `platform_configs` table
2. Picked up by workers on next task (or require restart for in-memory changes)

### API Usage Analytics
`GET /api/super/api-usage`
- Per-provider token usage and costs
- Per-feature cost breakdown
- Latency percentiles (p50, p95, p99)
- Fallback frequency analysis
- Daily cost trends (30 days)

### Task Queue Monitoring
`GET /api/super/task-queue`
- Active Celery tasks count
- Queue lengths
- Failed tasks (last 100)
- Worker process count

### Platform Config
`GET/POST /api/super/config`

Key-value store for platform settings:
- Feature flags (`enable_ai_illustrations`, `enable_overleaf_export`)
- Plan limits (`free_plan_pages`, `starter_plan_papers`)
- Pricing adjustments
- System messages

### Institutions Management
```
GET    /api/super/institutions          → All institutions
POST   /api/super/institutions          → Create institution
PUT    /api/super/institutions/<id>     → Update institution
POST   /api/super/institutions/<id>/suspend → Suspend org
```

---

## 27.4 Support Agent

**Route**: `routes/support.py`  
**Role Required**: `support_agent`  
**Data Scope**: Only users listed in `support_assignments.user_id` where `agent_id = jwt.sub`

### Assignment Model
```python
class SupportAssignment(BaseModel):
    agent_id = UUID FK (users)   # The support agent
    user_id  = UUID FK (users)   # The user they can assist
    assigned_by = UUID FK (users) # Who assigned this
    is_active = Boolean
```

### Support Endpoints
```
GET /api/support/my-users           → Users assigned to this agent
GET /api/support/users/<id>/papers  → Papers for assigned user
GET /api/support/users/<id>/profile → Profile for assigned user
```

All endpoints enforce: `user_id must be in agent's assignment list`.

---

## 27.5 JWT Role/Scope Summary

| Role | `scope` | DB RLS | What They Can Access |
|------|---------|--------|---------------------|
| `student` | `own` | user-scoped | Own projects/papers only |
| `professor` | `own` | user-scoped | Own projects/papers only |
| `phd` | `own` | user-scoped | Own projects/papers only |
| `admin` | `institute` | institute-scoped | All users in their institution |
| `super_admin` | `global` | unrestricted | Everything |
| `marketing` | `public` | aggregated only | Aggregated stats only |
| `support_agent` | `assigned` | assignment-scoped | Assigned users' data only |

---

## 27.6 Admin UI Screens

| Screen | Access | Description |
|--------|--------|-------------|
| `admin-dashboard` | admin+ | Institution overview |
| `admin-users` | admin+ | User list, invite, manage |
| `admin-papers` | admin+ | Paper monitoring |
| `admin-analytics` | admin+ | Usage analytics |
| `admin-monitoring` | admin+ | System health |
| `admin-billing` | admin+ | Billing/subscription management |
| `super-dashboard` | super_admin | Platform overview |
| `super-users` | super_admin | All users management |
| `super-ai` | super_admin | AI provider config |
| `super-api-usage` | super_admin | API cost analytics |
| `super-model-router` | super_admin | Provider routing config |
| `super-config` | super_admin | Platform settings |
| `super-infra` | super_admin | Infrastructure monitoring |
| `super-security` | super_admin | Security audit logs |
| `super-licensing` | super_admin | Institution licenses |
| `super-pricing` | super_admin | Plan pricing config |
| `super-payments` | super_admin | Payment monitoring |
| `super-logs` | super_admin | Application logs |
| `super-task-queue` | super_admin | Celery queue monitoring |
