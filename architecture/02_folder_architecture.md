# 02 — Folder Architecture

> **Back to Index**: [00_index.md](00_index.md)

---

## Complete Project Tree

```
researchai 3/
│
├── app.py                          # Flask application factory + Celery init
├── config.py                       # All configuration (DB, JWT, Redis, AI keys, plans)
├── models.py                       # SQLAlchemy ORM — 16 domain models
├── celery_worker.py                # Celery worker entry point
├── db_session_events.py            # PostgreSQL RLS session variable injection
├── init_db.py                      # Database initialization helper
├── requirements.txt                # Python dependencies
├── .env                            # Local environment variables (gitignored)
├── .env.example                    # Environment variable template
│
├── routes/                         # Flask Blueprints — HTTP endpoint handlers
│   ├── auth.py                     # /api/auth/* — login, register, 2FA, OAuth, logout
│   ├── user.py                     # /api/user/* — profile, notifications, subscription
│   ├── project.py                  # /api/projects/* — CRUD + settings
│   ├── document.py                 # /api/documents/* — upload, OCR, library
│   ├── paper.py                    # /api/papers/* — generation, editing, diagrams, plagiarism
│   ├── export.py                   # /api/export/* — DOCX, PDF, LaTeX export
│   ├── admin.py                    # /api/admin/* — institution-scoped admin panel
│   ├── super_admin.py              # /api/super/* — platform-wide super admin
│   ├── paraphraser.py              # /api/paraphraser/* — paraphrasing endpoints
│   ├── standalone_plagiarism.py    # /api/plagiarism/* — text-only plagiarism scan
│   ├── marketing.py                # /api/marketing/* — landing page data
│   ├── support.py                  # /api/support/* — support agent endpoints
│   └── __init__.py
│
├── paper_genration/                 # Core paper generation engine (note: typo in folder name)
│   ├── generation.py               # RAG-grounded section generators (all 11 sections)
│   ├── references.py               # Reference discovery + citation formatting
│   ├── analysis.py                 # Source metadata extraction (author, DOI, journal)
│   ├── doi.py                      # DOI lookup and resolution
│   ├── parse.py                    # Section parsing utilities
│   ├── titles.py                   # AI-generated paper title suggestions
│   ├── humanize.py                 # Humanizer dispatcher (calls ai_humanizer)
│   ├── plagiarism.py               # Plagiarism wrapper for paper routes
│   ├── ai_detection.py             # AI detection dispatcher
│   └── __init__.py
│
├── utils/                          # Shared utilities and AI modules
│   ├── ai_router.py                # Multi-model routing + circuit breaker + cost logging
│   ├── ai_detector.py              # Forensic AI detection engine (stat + LLM ensemble)
│   ├── ai_humanizer.py             # 4-phase humanization engine
│   ├── ai_engine.py                # Legacy AI call helpers
│   ├── ai_ensemble.py              # Ensemble signal combiner for AI detection
│   ├── ai_signals.py               # Statistical signal extractors
│   ├── candidate_retriever.py      # Plagiarism candidate retrieval (PostgreSQL trigram)
│   ├── citation_config.py          # Citation style rules (IEEE, APA, MLA, Chicago)
│   ├── citation_processor.py       # Citation rendering and injection logic
│   ├── llm_prompts.py              # Shared prompt builders for generation
│   ├── logger.py                   # Application-wide structured logger
│   ├── model_cache.py              # SentenceTransformer singleton cache (per worker)
│   ├── parallel_manager.py         # Thread pool manager for parallel generation
│   ├── paraphrase_manager.py       # Paraphraser orchestration + citation safety
│   ├── paraphrase_store.py         # Paraphrase session persistence
│   ├── paraphraser_engine.py       # Core paraphrasing LLM engine
│   ├── pinecone_search.py          # RAG retrieval (embed query → Pinecone → context)
│   ├── plagiarism_engine.py        # MinHash fingerprinting + semantic similarity
│   ├── research_intel.py           # Research intelligence extraction helper
│   ├── scan_guard.py               # Plagiarism scan rate/abuse guard
│   ├── scan_store.py               # Scan state persistence
│   ├── score_safety.py             # Score normalization and clamp utilities
│   ├── semantic_batcher.py         # Batch semantic comparison for plagiarism
│   ├── semantic_engine.py          # Semantic similarity engine (local embeddings)
│   ├── text_extractor.py           # PDF/DOCX/TXT text extraction + OCR
│   └── __init__.py
│
├── tasks/                          # Celery background task definitions
│   ├── embed_document.py           # RAG ingestion: chunk → embed → Pinecone upsert
│   ├── paper_tasks.py              # Paper section generation pipeline
│   ├── plagiarism_tasks.py         # Ensemble plagiarism scan (chord pipeline)
│   ├── standalone_plagiarism_tasks.py  # Standalone text plagiarism scan
│   ├── paraphraser_tasks.py        # Async paraphrasing batch
│   ├── ai_detection_tasks.py       # Async AI detection task
│   └── __init__.py
│
├── static/                         # Frontend static assets
│   ├── css/
│   │   └── main.css                # Global design system (CSS variables, components)
│   └── js/
│       ├── appRouter.js            # SPA screen registry + navigation logic
│       ├── appShell.js             # Sidebar, topbar, panel switcher HTML
│       ├── api.js                  # Centralized apiFetch() wrapper
│       ├── auth.js                 # Auth state management (login/logout/CSRF)
│       ├── utils/
│       │   └── sessionState.js     # SessionState singleton (paper, project, user)
│       ├── services/
│       │   ├── dashboardService.js # Dashboard data loading
│       │   ├── editorService.js    # Paper editor data service
│       │   └── superAdminMockData.js  # Super admin mock data for dev
│       └── screens/               # One JS file per application screen
│           ├── auth.js             # Auth screen router
│           ├── authLanding.js      # Landing page
│           ├── authLogin.js        # Login screen
│           ├── authRegister.js     # Registration screen
│           ├── authOtp.js          # OTP verification screen
│           ├── auth2fa.js          # 2FA screen
│           ├── authEmailVerify.js  # Email verification screen
│           ├── authForgotPw.js     # Forgot password screen
│           ├── papers.js           # Paper list screen
│           ├── projects.js         # Projects list screen
│           ├── projectSetup.js     # New project setup wizard
│           ├── upload.js           # Document upload screen
│           ├── analysis.js         # AI analysis results screen
│           ├── generation.js       # Paper generation screen
│           ├── editor.js           # Full paper editor
│           ├── diagrams.js         # Diagram Studio (full feature)
│           ├── citations.js        # Citation manager
│           ├── citationWorkspace.js    # Citation workspace editor
│           ├── plagiarism.js       # Plagiarism check + workspace
│           ├── plagiarismWorkspace.js  # Plagiarism scan results
│           ├── paraphraserWorkspace.js # Paraphraser screen
│           ├── standaloneScanner.js    # Standalone plagiarism scanner
│           ├── humanizer.js        # AI humanizer screen
│           ├── export.js           # Export screen
│           ├── notifications.js    # Notifications screen
│           ├── user.js             # User profile/settings
│           ├── profile.js          # Profile management
│           ├── subscription.js     # Subscription management
│           ├── admin.js            # Admin panel router
│           ├── adminDashboard.js   # Admin dashboard
│           ├── adminUsers.js       # Admin user management
│           ├── adminPapers.js      # Admin paper monitoring
│           ├── adminAnalytics.js   # Admin analytics
│           ├── adminMonitoring.js  # Admin system monitoring
│           ├── adminBilling.js     # Admin billing
│           ├── super.js            # Super admin router
│           ├── superDashboard.js   # Super admin dashboard
│           ├── superUsers.js       # Super admin user management
│           ├── superAI.js          # Super admin AI/API config
│           ├── superAPIUsage.js    # Super admin API usage analytics
│           ├── superModelRouter.js # Super admin model router config
│           ├── superConfig.js      # Super admin global config
│           ├── superInfra.js       # Super admin infrastructure
│           ├── superSecurity.js    # Super admin security
│           ├── superLicensing.js   # Super admin licensing
│           ├── superPricing.js     # Super admin pricing
│           ├── superPayments.js    # Super admin payments
│           ├── superLogs.js        # Super admin logs
│           ├── superOverview.js    # Super admin overview
│           └── superTaskQueue.js   # Super admin task queue
│
├── templates/
│   └── index.html                  # Single HTML shell (SPA entry point)
│
├── migrations/                     # Flask-Migrate (Alembic) database migrations
├── uploads/                        # Local file storage for uploaded documents
├── instance/                       # Flask instance config (SQLite dev fallback)
├── scratch/                        # Developer scratch/test scripts (not production)
├── backups/                        # Database backup files
└── docs/
    └── architecture/               # This documentation suite
```

---

## Module Explanations

### `app.py` — Application Factory
The Flask application factory. Creates the Flask app, initializes all extensions (SQLAlchemy, JWT, Redis, Limiter, Celery), registers all blueprints, configures JWT token callbacks, and sets up the CORS policy and rate limiter. This is the only entry point for both the web server and Celery workers.

### `config.py` — Configuration Class
Single source of truth for all configuration. Reads from environment variables (fails fast at startup if required vars like `DATABASE_URL` or `JWT_SECRET_KEY` are missing). Contains the connection pool settings, JWT lifetimes, plan definitions, and model pricing table.

### `models.py` — ORM Layer
Defines all 16 SQLAlchemy models inheriting from `BaseModel`. Every model gets UUID primary keys, soft delete, and timezone-aware timestamps automatically. This file owns the entire data shape of the application.

### `db_session_events.py` — PostgreSQL RLS
Injects the current user's ID into PostgreSQL session variables on every request, enabling Row-Level Security policies to automatically enforce tenant isolation at the database level.

### `routes/` — HTTP API Layer
Each Blueprint maps to a domain area. Routes are thin — they validate input, call service/utility functions, and return JSON. Business logic lives in `paper_genration/` and `utils/`.

### `paper_genration/` — Generation Engine
The core AI generation module. `generation.py` contains one function per paper section. Each function performs a RAG retrieval from Pinecone, builds a grounded prompt, and calls `call_ai()`. `references.py` handles citation discovery and formatting.

### `utils/ai_router.py` — Multi-Model Router
The heart of the AI reliability system. Implements per-provider circuit breakers, a Gemini rate limiter (token bucket), and a waterfall routing system that tries providers in configured order. Logs token usage and cost to `UsageLog` on every successful call.

### `utils/model_cache.py` — Embedding Model Cache
Loads the SentenceTransformer model once per worker process and caches it as a module-level singleton. Prevents the ~2-3 second model load time from occurring on every document embedding operation.

### `tasks/` — Background Workers
All long-running operations. Each task module is a Celery `@shared_task` with retry logic, progress tracking, and database status updates. The plagiarism scan uses a Celery `chord` for parallel execution.

### `static/js/` — Frontend SPA
A pure JavaScript Single Page Application. No React/Vue/Angular — all screens are render functions that return HTML strings. `appRouter.js` manages navigation. `api.js` wraps all HTTP calls. `sessionState.js` is the client-side state singleton.
