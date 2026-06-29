# 26 — Cost Architecture

> **Back to Index**: [00_index.md](00_index.md)

---

## 26.1 Cost Tracking System

Every AI API call is logged with token counts and USD/INR cost in the `usage_logs` table. This powers the Super Admin's cost analytics dashboard and helps identify expensive operations or provider inefficiencies.

---

## 26.2 Model Pricing (from `config.py`)

| Model | Input $/1M tokens | Output $/1M tokens |
|-------|-------------------|--------------------|
| GLM-5.1 (NVIDIA) | $0.30 | $2.50 |
| MiniMax M3 (NVIDIA) | $0.25 | $2.00 |
| DeepSeek V4-Flash (NVIDIA) | $0.14 | $0.28 |
| DeepSeek V4-Pro (NVIDIA) | $0.55 | $2.20 |
| Gemini 2.0 Flash (Google) | $0.10 | $0.40 |
| GPT-4o-mini (OpenAI) | $0.15 | $0.60 |
| Ollama / local | $0.00 | $0.00 |

**INR conversion rate**: 84.5 (configurable via `Config.INR_CONVERSION_RATE`)

---

## 26.3 Cost Per Paper Generation

A typical 15-page research paper (11 sections, ~5,000 words):

| Section | ~Input Tokens | ~Output Tokens | Model | Cost (USD) |
|---------|--------------|----------------|-------|-----------|
| Abstract (150 words) | 800 | 200 | GLM-5.1 | $0.0007 |
| Introduction (500 words) | 1200 | 670 | GLM-5.1 | $0.0020 |
| Literature Review (800 words) | 1500 | 1067 | GLM-5.1 | $0.0032 |
| Research Gap (250 words) | 1200 | 333 | GLM-5.1 | $0.0012 |
| Methodology (600 words) | 1400 | 800 | GLM-5.1 | $0.0024 |
| System Architecture (350 words) | 900 | 467 | GLM-5.1 | $0.0014 |
| Implementation (400 words) | 1000 | 533 | GLM-5.1 | $0.0016 |
| Results (500 words) | 1200 | 667 | GLM-5.1 | $0.0020 |
| Discussion (350 words) | 900 | 467 | GLM-5.1 | $0.0014 |
| Conclusion (200 words) | 800 | 267 | GLM-5.1 | $0.0008 |
| Future Work (200 words) | 800 | 267 | GLM-5.1 | $0.0008 |
| **Total** | **~12,000** | **~5,700** | GLM-5.1 | **~$0.018** |

**Per paper cost ≈ $0.02 (₹1.70)** with GLM-5.1.

If falling back to Gemini:
- Per paper cost ≈ $0.003 (₹0.25) — significantly cheaper

---

## 26.4 Cost Per Feature Operation

| Operation | Tokens (Estimated) | Model | Cost/Call (USD) |
|-----------|-------------------|-------|----------------|
| Plagiarism AI verification (per match) | 600 | DeepSeek V4-Flash | $0.00017 |
| Plagiarism scan (full paper, 20 matches) | 12,000 | DeepSeek V4-Flash | $0.0034 |
| Humanizer (500 words) | 3,000 | DeepSeek V4-Pro | $0.0022 |
| AI detection (500 words, LLM) | 700 | DeepSeek V4-Flash | $0.0002 |
| Paraphraser (per paragraph) | 800 | DeepSeek V4-Flash | $0.0002 |
| Diagram opportunity scan (5 paragraphs) | 1,200 | DeepSeek V4-Flash | $0.00034 |
| AI Illustration (NVIDIA SDXL) | N/A | SDXL | $0.003-0.010 |

---

## 26.5 Monthly Cost Estimation

For 1,000 active users generating 1 paper/month each:

| Feature | Uses/Month | Cost/Use | Monthly Cost |
|---------|-----------|---------|-------------|
| Paper generation | 1,000 | $0.018 | $18 |
| Plagiarism scan | 3,000 | $0.003 | $9 |
| Humanizer | 5,000 | $0.002 | $10 |
| AI Detection | 5,000 | $0.0002 | $1 |
| Paraphraser | 10,000 | $0.0002 | $2 |
| Diagram scanning | 2,000 | $0.0003 | $0.60 |
| **Total AI** | | | **~$41/month** |

**Infrastructure costs** (additional):
- PostgreSQL (RDS small): ~$15/month
- Redis (ElastiCache): ~$15/month
- Pinecone (Starter): Free tier or ~$20/month
- AWS S3: ~$5/month (500GB)
- **Total infrastructure**: ~$55/month

**Total for 1,000 users**: ~$96/month = ~$0.10/user/month (very cost-efficient)

---

## 26.6 Cost Tracking Implementation

```python
# In ai_router.log_result():
pricing = Config.MODEL_PRICING.get(model_name)
if pricing:
    cost_usd = (tokens_in / 1_000_000 * pricing["input"]) + \
               (tokens_out / 1_000_000 * pricing["output"])
    cost_inr = cost_usd * Config.INR_CONVERSION_RATE

UsageLog(
    user_id=user_id,
    action="api_token_usage",
    meta={
        "api_name": provider,
        "model": model_name,
        "task_type": task_type,
        "tokens_in": tokens_in,
        "tokens_out": tokens_out,
        "total_tokens": tokens_in + tokens_out,
        "cost_usd": round(cost_usd, 6),
        "cost_inr": round(cost_inr, 4),
        "latency_ms": duration_ms,
    }
)
```

---

## 26.7 Super Admin Cost Dashboard

The `/api/super/api-usage` endpoint aggregates `UsageLog` records:

```python
# Cost by provider
provider_costs = db.session.execute(
    "SELECT meta->>'api_name' as provider, "
    "SUM((meta->>'cost_usd')::float) as total_usd, "
    "SUM((meta->>'total_tokens')::int) as total_tokens "
    "FROM usage_logs WHERE action='api_token_usage' "
    "GROUP BY meta->>'api_name'"
).fetchall()

# Cost by feature
feature_costs = db.session.execute(
    "SELECT meta->>'task_type' as feature, "
    "SUM((meta->>'cost_usd')::float) as total_usd "
    "FROM usage_logs WHERE action='api_token_usage' "
    "GROUP BY meta->>'task_type'"
).fetchall()
```

---

## 26.8 Cost Optimization Controls

| Control | Mechanism | Savings |
|---------|-----------|---------|
| GLM as primary (paper gen) | Cheaper than GPT-4o | ~80% vs OpenAI |
| DeepSeek V4-Flash primary | Very cheap ($0.14/$0.28) | ~90% vs GPT-4o |
| Signal-first AI detection | Skip LLM for obvious cases | 35% fewer LLM calls |
| Ollama dev mode | Zero cost local inference | 100% in dev |
| Per-feature API keys | Quota separation | Prevent one feature exhausting all |
| Circuit breaker | Skip dead providers instantly | Saves retry time/cost |
