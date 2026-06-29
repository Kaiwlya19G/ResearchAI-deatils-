# ResearchAI — Architecture Documentation Index

> **Classification**: Production Architecture Audit  
> **Audience**: Senior Engineers, DevOps, Product  
> **Version**: 0.7.0 (multi-model-router branch)  
> **Last Updated**: June 2026

---

## 📁 Document Map

| # | Document | Description |
|---|----------|-------------|
| 01 | [Executive Overview](01_executive_overview.md) | Purpose, modules, system boundaries, AI components |
| 02 | [Folder Architecture](02_folder_architecture.md) | Complete project tree with module explanations |
| 03 | [Authentication & Login Flow](03_auth_login_flow.md) | JWT, cookies, 2FA, session management |
| 04 | [User Journey](04_user_journey.md) | End-to-end feature walkthroughs |
| 05 | [Frontend Architecture](05_frontend_architecture.md) | SPA routing, state, screens, API calls |
| 06 | [Backend Architecture](06_backend_architecture.md) | Flask, blueprints, middleware, services |
| 07 | [API Documentation](07_api_documentation.md) | All endpoints, schemas, auth requirements |
| 08 | [Database Documentation](08_database.md) | ER diagram, all tables, relationships |
| 09 | [AI Architecture](09_ai_architecture.md) | All AI modules, prompts, models, fallbacks |
| 10 | [Multi-Model Router](10_multi_model_router.md) | Provider routing, circuit breaker, fallback chain |
| 11 | [RAG Pipeline](11_rag_pipeline.md) | Upload → chunk → embed → retrieve → generate |
| 12 | [Chunking Pipeline](12_chunking_pipeline.md) | LangChain splitter, overlap, vector storage |
| 13 | [Citation Pipeline](13_citation_pipeline.md) | UUID injection, formatting, export |
| 14 | [Plagiarism Pipeline](14_plagiarism_pipeline.md) | Ensemble scan, semantic match, AI verification |
| 15 | [Humanizer Pipeline](15_humanizer_pipeline.md) | Phase-based rewrite, scoring, diff |
| 16 | [AI Detection Pipeline](16_ai_detection_pipeline.md) | Statistical + LLM forensic detection |
| 17 | [Diagram Studio](17_diagram_studio.md) | Section scanning, Mermaid, AI illustration |
| 18 | [Journal Template Pipeline](18_journal_templates.md) | Template parsing, style extraction, export |
| 19 | [Export Pipeline](19_export_pipeline.md) | DOCX, PDF, LaTeX, Overleaf |
| 20 | [Background Jobs](20_background_jobs.md) | Celery workers, queues, retry, monitoring |
| 21 | [Infrastructure](21_infrastructure.md) | Redis, PostgreSQL, Pinecone, S3, env vars |
| 22 | [Logging Architecture](22_logging.md) | App, API, worker, AI, cost, audit logs |
| 23 | [Error Handling](23_error_handling.md) | Failure diagrams, retry logic, fallback |
| 24 | [Security Audit](24_security_audit.md) | Auth, JWT, SQL injection, XSS, CSRF |
| 25 | [Performance Audit](25_performance_audit.md) | Caching, parallel, chunk size, DB queries |
| 26 | [Cost Architecture](26_cost_architecture.md) | Token tracking, provider cost, billing |
| 27 | [Admin Architecture](27_admin_architecture.md) | Super admin, monitoring, model router UI |
| 28 | [Deployment Architecture](28_deployment.md) | Docker, scaling, backup, disaster recovery |
| 29 | [UML Diagrams](29_uml_diagrams.md) | Class, sequence, component, activity diagrams |
| 30 | [Mermaid Diagrams](30_mermaid_diagrams.md) | System, login, RAG, plagiarism, export flows |
| 31 | [Knowledge Graph](31_knowledge_graph.md) | Module → file → function dependency graph |
| 32 | [Technical Debt Report](32_technical_debt.md) | Duplicate logic, bottlenecks, security risks |
| 33 | [Improvement Roadmap](33_improvement_roadmap.md) | Priority matrix, immediate/long-term fixes |

---

## Quick System Overview

```
┌─────────────────────────────────────────────────────────┐
│                    ResearchAI SaaS                       │
│                                                          │
│  Browser SPA ──► Flask API ──► PostgreSQL                │
│       │              │    ──► Redis (cache/queue)        │
│       │              │    ──► Pinecone (vector DB)       │
│       │              │    ──► Celery Workers             │
│       │              │    ──► AI Providers               │
│       │              │         GLM / DeepSeek / Gemini   │
│       │              │         OpenAI / Ollama / NVIDIA  │
│       │              │    ──► AWS S3 (file storage)      │
│       │              │    ──► Razorpay (payments)        │
└─────────────────────────────────────────────────────────┘
```
