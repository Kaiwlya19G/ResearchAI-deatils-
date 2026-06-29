# 07 — API Documentation

> **Back to Index**: [00_index.md](00_index.md)

---

## 7.1 Authentication API (`/api/auth`)

### POST `/api/auth/register`
**Auth**: None | **Rate Limit**: 10/min

**Input**:
```json
{ "first_name": "John", "last_name": "Doe", "email": "j@x.com", "password": "SecurePass123" }
```
**Output (200)**:
```json
{ "message": "Registration successful", "user": { "id": "uuid", "email": "..." } }
```
Sets `access_token_cookie`, `refresh_token_cookie`, `csrf_access_token` cookies.

**Errors**: `400` invalid email, `409` email already registered

---

### POST `/api/auth/login`
**Auth**: None | **Rate Limit**: 10/min

**Input**: `{ "email": "...", "password": "..." }`  
**Output (200)**: `{ "message": "Login successful", "user": {...}, "requires_2fa": false }`  
**Errors**: `401` invalid credentials, `403` account locked, `423` account suspended

---

### POST `/api/auth/logout`
**Auth**: Required | **Rate Limit**: 10/min

**Output (200)**: `{ "message": "Logged out successfully" }`  
Clears all JWT cookies, adds JTI to Redis blocklist.

---

### POST `/api/auth/refresh`
**Auth**: Refresh token required

**Output (200)**: `{ "message": "Token refreshed" }`  
Rotates refresh token, issues new access token.

---

### POST `/api/auth/verify-email`
**Input**: `{ "otp": "123456" }`  
**Output (200)**: `{ "message": "Email verified" }`

---

### POST `/api/auth/2fa/verify`
**Input**: `{ "code": "123456", "temp_token": "..." }`  
**Output (200)**: Full login response with cookies

---

## 7.2 User API (`/api/user`)

### GET `/api/user/me`
**Auth**: Required  
**Output**: Full user profile object including plan, institution, subscription status.

---

### PUT `/api/user/profile`
**Input**: `{ "first_name": "...", "last_name": "...", "institution": "..." }`  
**Output (200)**: `{ "message": "Profile updated", "user": {...} }`

---

### GET `/api/user/notifications`
**Output**: `{ "notifications": [...], "unread_count": 3 }`

---

### POST `/api/user/notifications/<id>/read`
**Output**: `{ "message": "Notification marked as read" }`

---

### GET `/api/user/usage`
**Output**: Token usage stats, feature usage counts, plan limits.

---

## 7.3 Projects API (`/api/projects`)

### POST `/api/projects`
**Auth**: Required  
**Input**:
```json
{
  "title": "My Research Paper",
  "domain": "Computer Science",
  "citation_style": "IEEE",
  "template": "IEEE Conference",
  "target_pages": "15-20 pages"
}
```
**Output (201)**: `{ "project": { "id": "uuid", ... } }`

---

### GET `/api/projects`
**Output**: `{ "projects": [...] }` — List of user's projects

---

### GET `/api/projects/<id>`
**Output**: Full project object with document count and paper status

---

### PUT `/api/projects/<id>`
**Input**: Partial project fields to update  
**Output**: Updated project object

---

### DELETE `/api/projects/<id>`
**Output**: `{ "message": "Project deleted" }`  
Triggers `delete_all_project_vectors` Celery task (GDPR cleanup).

---

## 7.4 Documents API (`/api/documents`)

### POST `/api/documents/<project_id>/upload`
**Auth**: Required | **Content-Type**: `multipart/form-data`  
**Input**: File(s) in `files[]` field  
**Output (200)**:
```json
{ "uploaded": ["doc-uuid-1", "doc-uuid-2"], "failed": [] }
```
Triggers `embed_document` Celery task for each file.

---

### GET `/api/documents/<project_id>`
**Output**: `{ "documents": [...] }` — All project documents with metadata

---

### DELETE `/api/documents/<project_id>/<doc_id>`
**Output**: `{ "message": "Document deleted" }`  
Triggers `delete_document_vectors` Celery task (GDPR cleanup).

---

## 7.5 Papers API (`/api/papers`)

### POST `/api/papers/<project_id>`
**Input**: `{ "title": "...", "citation_style": "IEEE" }`  
**Output (201)**: Paper object

---

### GET `/api/papers/<id>`
**Output**: Full paper object including all sections, version, status

---

### GET `/api/papers/<id>/status`
**Output**: `{ "status": "generating", "progress": 45, "current_section": "methodology" }`  
Used for polling during generation.

---

### POST `/api/papers/<id>/generate`
**Output (202)**: `{ "task_id": "celery-uuid", "status": "queued" }`  
Triggers `generate_paper_task` Celery task.

---

### PUT `/api/papers/<id>/sections/<section_key>`
**Input**: `{ "content": "New section text..." }`  
**Output (200)**: `{ "message": "Section saved" }`

---

### POST `/api/papers/<id>/analyze`
**Output**: `{ "keywords": [...], "topics": [...], "research_gaps": [...], "generated_titles": [...] }`

---

### POST `/api/papers/<id>/humanize`
**Input**: `{ "text": "...", "intensity": "medium" }`  
**Output**:
```json
{
  "humanized": "...",
  "human_score_before": 12,
  "human_score_after": 78,
  "improvement": 66,
  "word_diff": [...]
}
```

---

### POST `/api/papers/<id>/detect-ai`
**Input**: `{ "text": "..." }`  
**Output**:
```json
{
  "overall_score": 78,
  "classification": "AI-generated",
  "confidence": "high",
  "signal_breakdown": {...},
  "sentence_analysis": [...]
}
```

---

### POST `/api/papers/<id>/plagiarism/scan`
**Output (202)**: `{ "scan_task_id": "uuid" }`  
Triggers Celery chord pipeline.

---

### GET `/api/papers/<id>/plagiarism/status`
**Output**:
```json
{
  "status": "processing",
  "progress": 60,
  "current_step": "Semantic Similarity Analysis...",
  "results": null
}
```

---

### POST `/api/papers/<id>/diagrams/opportunities`
**Output**:
```json
{
  "opportunities": [
    {
      "section_key": "methodology",
      "section_name": "Methodology",
      "paragraph_index": 2,
      "confidence": 92,
      "recommended_type": "Flowchart",
      "reason": "...",
      "suggested_title": "..."
    }
  ]
}
```

---

### POST `/api/papers/<id>/diagrams/generate`
**Input**:
```json
{
  "paragraph_text": "...",
  "diagram_type": "Flowchart",
  "theme": "academic",
  "style": "IEEE Style",
  "orientation": "Vertical"
}
```
**Output (200)**:
```json
{ "code": "graph TD\n  A --> B", "title": "AI Generated Flowchart", "is_image": false }
```
For `AI Illustration`:
```json
{ "code": "data:image/png;base64,...", "title": "AI Generated Illustration", "is_image": true }
```

---

### POST `/api/papers/<id>/diagrams`
**Input**: Diagram data object  
**Output (201)**: Diagram record + paper updated with `[[diagram:UUID]]` tag

---

### GET `/api/papers/<id>/citations`
**Output**: `{ "citations": [...] }`

---

### POST `/api/papers/<id>/citations`
**Input**: Citation metadata object  
**Output (201)**: Citation record

---

## 7.6 Export API (`/api/export`)

### POST `/api/export/<paper_id>/docx`
**Input**: `{ "template": "IEEE Conference" }`  
**Output**: Binary DOCX file download (`application/vnd.openxmlformats-officedocument.wordprocessingml.document`)

---

### POST `/api/export/<paper_id>/pdf`
**Output**: Binary PDF file download

---

### POST `/api/export/<paper_id>/latex`
**Output**: Binary ZIP file with `.tex` and `.bib` files

---

### POST `/api/export/<paper_id>/overleaf`
**Output**: Redirect URL to Overleaf with project pre-populated

---

## 7.7 Paraphraser API (`/api/paraphraser`)

### POST `/api/paraphraser/<paper_id>/paraphrase`
**Input**: `{ "text": "...", "sentences": [...] }`  
**Output**: `{ "paraphrased": "...", "changes": [...] }`

---

## 7.8 Admin API (`/api/admin`)

All routes require `role = admin or super_admin`.

### GET `/api/admin/users`
**Output**: Paginated user list scoped to admin's institution

### GET `/api/admin/papers`
**Output**: Paper monitoring data for institution users

### GET `/api/admin/analytics`
**Output**: Usage analytics for the institution

---

## 7.9 Super Admin API (`/api/super`)

All routes require `role = super_admin`.

### GET `/api/super/overview`
**Output**: Platform-wide metrics (users, papers, revenue, API costs)

### GET `/api/super/api-usage`
**Output**: Per-provider token usage, costs, latency stats from `UsageLog`

### POST `/api/super/config`
**Input**: `{ "key": "...", "value": "..." }`  
**Output**: `{ "message": "Config updated" }`

### GET `/api/super/task-queue`
**Output**: Celery task status overview (pending, active, failed counts)

### GET `/api/super/logs`
**Output**: Recent application log entries

---

## 7.10 Common Response Codes

| Code | Meaning |
|------|---------|
| `200` | Success |
| `201` | Created |
| `202` | Accepted (async task started) |
| `400` | Bad request / validation error |
| `401` | Not authenticated |
| `403` | Forbidden / wrong role |
| `404` | Resource not found |
| `409` | Conflict (duplicate) |
| `429` | Rate limit exceeded |
| `500` | Internal server error |
| `503` | Service unavailable (all AI providers down) |
