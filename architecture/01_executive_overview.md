# 01 — Executive Architecture Overview

> **Back to Index**: [00_index.md](00_index.md)

---

## 1.1 Purpose

ResearchAI is a **production-grade, AI-powered SaaS platform** for academic research paper generation, verification, and publishing. It transforms a set of uploaded reference documents into a fully written, cited, plagiarism-checked, and exportable research paper through an automated multi-stage pipeline.

The platform serves **students, PhD researchers, professors, and academic institutions** who need to produce high-quality research papers. It abstracts away the most time-consuming phases of academic writing — source synthesis, citation management, plagiarism checking, and structural formatting — into a guided, intelligent workflow.

---

## 1.2 Major Modules

| Module | Purpose |
|--------|---------|
| **Auth & Identity** | JWT HTTPOnly cookie auth, 2FA, OAuth, session management |
| **Project Management** | Research project container, document library, settings |
| **Document Upload & OCR** | PDF/DOCX/TXT ingestion, text extraction, OCR for scanned PDFs |
| **RAG Pipeline** | LangChain chunking → SentenceTransformer embedding → Pinecone upsert |
| **AI Analysis** | Keyword/topic/gap extraction, title generation from sources |
| **Paper Generation** | Section-by-section RAG-grounded generation via multi-model router |
| **Citation Engine** | UUID-tagged inline citations, styled bibliography formatting |
| **Diagram Studio** | AI-detected diagram opportunities, Mermaid SVG + NVIDIA SDXL images |
| **Plagiarism Checker** | Celery ensemble: exact fingerprint + semantic + AI LLM verification |
| **Paraphraser** | Sentence-level AI paraphrasing with citation preservation |
| **AI Humanizer** | 4-phase forensic AI pattern removal + LLM rewrite |
| **AI Detection** | Statistical forensic + LLM zero-shot classifier ensemble |
| **Export System** | DOCX (IEEE/APA/MLA), PDF, LaTeX, Overleaf integration |
| **Notification System** | In-app notifications for async job completion |
| **Billing & Subscriptions** | Razorpay payments, plan enforcement, invoice generation |
| **Admin Panel** | Institution-scoped admin for user/paper monitoring |
| **Super Admin** | Platform-wide config, model router control, cost analytics |
| **Background Workers** | Celery + Redis task queues for all async operations |

---

## 1.3 System Boundaries

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          ResearchAI System Boundary                       │
│                                                                            │
│  ┌──────────────┐    HTTPS     ┌─────────────────────────────────────┐    │
│  │   Browser     │◄────────────►│  Flask Application Server           │    │
│  │   SPA (JS)    │             │  (app.py + Blueprints)               │    │
│  └──────────────┘             └──────────┬──────────────────────────┘    │
│                                           │                                │
│          ┌────────────────────────────────┼────────────────────────┐      │
│          ▼                    ▼            ▼                  ▼     │      │
│  ┌──────────────┐  ┌──────────────┐ ┌──────────┐  ┌──────────────┐│      │
│  │  PostgreSQL   │  │    Redis     │ │ Pinecone │  │  Celery      ││      │
│  │  (primary DB) │  │  (cache/     │ │ (vector  │  │  Workers     ││      │
│  │               │  │   broker)    │ │  store)  │  │              ││      │
│  └──────────────┘  └──────────────┘ └──────────┘  └──────────────┘│      │
│                                                                      │      │
└──────────────────────────────────────────────────────────────────────┘      │
                                                                               │
  EXTERNAL SERVICES (called by Flask & Celery workers)                        │
  ┌───────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌────────────┐       │
  │ GLM/NVIDIA│ │ DeepSeek │ │  Gemini  │ │  OpenAI   │ │   Ollama   │       │
  │ (paper    │ │ (plgrsm/ │ │ (fallbck │ │ (fallbck) │ │  (local)   │       │
  │  gen)     │ │  hmnzr)  │ │  LLM)    │ │           │ │            │       │
  └───────────┘ └──────────┘ └──────────┘ └───────────┘ └────────────┘       │
  ┌───────────┐ ┌──────────┐ ┌──────────┐                                    │
  │  AWS S3   │ │ Razorpay │ │  SMTP    │                                    │
  │  (files)  │ │ (payment)│ │  (email) │                                    │
  └───────────┘ └──────────┘ └──────────┘
```

---

## 1.4 External Services & Integrations

| Service | Role | Criticality |
|---------|------|-------------|
| **NVIDIA NIM API** | Primary: GLM-5.1 (paper gen), DeepSeek V4 (plg/hum/detect), Stable Diffusion XL (images) | Critical |
| **Google Gemini** | Fallback LLM for all features | High |
| **OpenAI** | Secondary fallback LLM | Medium |
| **Ollama** | Local LLM (dev/offline mode) | Dev only |
| **Pinecone** | Vector database for RAG retrieval | Critical |
| **PostgreSQL** | Primary relational database | Critical |
| **Redis** | Celery broker, task results, JWT blocklist, rate limiter | Critical |
| **AWS S3** | File storage for uploaded documents | High |
| **Razorpay** | Payment gateway (INR) | Billing |
| **SMTP / Gmail** | Email verification, OTP, notifications | High |
| **Google OAuth** | Social login | Medium |
| **Facebook OAuth** | Social login | Low |

---

## 1.5 AI Components Summary

| Component | Primary Model | Fallback Chain | Task Type |
|-----------|--------------|----------------|-----------|
| Paper Generation | GLM-5.1 (NVIDIA) | Gemini → OpenAI → Ollama | `paper_generation` |
| Plagiarism AI | DeepSeek V4-Flash | Gemini | `plagiarism` |
| Paraphraser | DeepSeek V4-Flash | Gemini → OpenAI | `paraphraser` |
| Humanizer | DeepSeek V4-Pro | Gemini | `humanizer` |
| AI Detection | DeepSeek V4-Flash | Gemini | `ai_detection` |
| Diagram Scan | DeepSeek V4-Flash (NVIDIA) | call_ai waterfall | `generic` |
| Image Generation | NVIDIA Stable Diffusion XL | None | Direct API |
| Embeddings | SentenceTransformer (local) | None | Local model |

---

## 1.6 Background Workers

Six Celery task modules run as background workers:

| Module | Tasks |
|--------|-------|
| `tasks.embed_document` | Chunk + embed + upsert to Pinecone (3 retries, 30s delay) |
| `tasks.paper_tasks` | Section-by-section paper generation pipeline |
| `tasks.plagiarism_tasks` | Ensemble plagiarism scan (chord pipeline) |
| `tasks.standalone_plagiarism_tasks` | Standalone text scan (no paper required) |
| `tasks.paraphraser_tasks` | Async paraphraser batch |
| `tasks.ai_detection_tasks` | AI detection analysis |

---

## 1.7 Storage Architecture

| Layer | Technology | Data Stored |
|-------|-----------|-------------|
| **Relational** | PostgreSQL (UUID PKs, RLS) | Users, projects, papers, citations, billing |
| **Vector** | Pinecone (per-project namespaces) | Document chunk embeddings for RAG |
| **Cache/Queue** | Redis DB 0 | Celery broker + results |
| **Rate Limit** | Redis DB 1 | Rate limiter counters + JWT blocklist |
| **File** | Local `uploads/` + AWS S3 | Raw uploaded PDFs/DOCX/TXT files |

---

## 1.8 Future Scalability Considerations

- **Horizontal scaling**: Flask is stateless (JWT via cookies). Add more Gunicorn workers or pods without session affinity.
- **Celery scaling**: Add more `celery worker` processes. Use separate queues for CPU-heavy tasks (embedding) vs. fast tasks (notifications).
- **Pinecone**: Each project namespace is independent — scales to millions of vectors. No re-architecture needed.
- **PostgreSQL RLS**: Row-Level Security policies are already implemented for multi-tenancy. Enables moving to managed PG (RDS, Supabase) without application-layer changes.
- **Multi-region**: Redis Cluster + Pinecone multi-region + PG read replicas are the three main scaling axes.
- **Model cost control**: Per-feature API keys and the circuit breaker system enable instant provider swap without code changes.
