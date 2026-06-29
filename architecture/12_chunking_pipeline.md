# 12 — Chunking Pipeline

> **Back to Index**: [00_index.md](00_index.md)

---

## 12.1 Chunking Strategy

ResearchAI uses **LangChain's `RecursiveCharacterTextSplitter`** for all document chunking. This splitter respects natural language boundaries (paragraphs → sentences → words) before resorting to character-level splits.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,       # Target chunk length in characters (~250 words)
    chunk_overlap=200,     # Overlap between consecutive chunks (continuity bridge)
    length_function=len,   # Character-based length counting
    separators=["\n\n", "\n", ". ", " ", ""]
)
```

### Separator Priority
The splitter tries each separator in order and only falls back to the next if the resulting chunk exceeds `chunk_size`:

1. `"\n\n"` — Paragraph breaks (first preference)
2. `"\n"` — Line breaks
3. `". "` — Sentence boundaries
4. `" "` — Word boundaries
5. `""` — Character splits (last resort, almost never used)

This ensures chunks respect natural semantic units as much as possible.

---

## 12.2 Why 1000 Characters?

| Chunk Size | Pros | Cons |
|-----------|------|------|
| 512 chars | Precise retrieval, low token cost | May split mid-concept, fragmented context |
| **1000 chars** | **Good semantic unit, ~250 words, fits retrieval window** | **Chosen** |
| 2000 chars | Rich context | Too many tokens per LLM context injection |

**1000 chars ≈ 250 words ≈ 1-2 research paper paragraphs** — an ideal semantic granularity for academic content.

---

## 12.3 Why 200-Character Overlap?

```
Chunk 1: [.....paragraph A.....end_of_paragraph_A....first_half_of_B...]
Chunk 2: [...second_half_of_B........paragraph_B_complete.......]
```

The 200-character overlap ensures that concepts spanning chunk boundaries appear in both chunks. Without overlap:
- A sentence starting at the end of chunk N could be meaninglessly truncated
- The retriever could miss content where the key concept appears at the boundary

---

## 12.4 Vector Payload Structure

Each chunk becomes one Pinecone vector:

```python
{
    "id": "uuid-v4",                    # Random UUID (collision-proof)
    "values": [0.023, -0.145, ...],     # 768-dimensional float vector
    "metadata": {
        # ── Multi-tenant security keys (NEVER remove) ──
        "project_id": "project-uuid",   # Namespace security backup
        "user_id": "user-uuid",          # Ownership tracing
        # ── Traceability ──
        "document_id": "doc-uuid",       # Used for idempotency cleanup & citations
        "original_name": "paper.pdf",    # Original filename for display
        "chunk_index": 4,                # Position within document
        # ── Raw text (stored for LLM injection) ──
        "text": "Full text of this chunk..."
    }
}
```

**Why store text in metadata?** Pinecone does not return the vector values on retrieval by default, but it does return metadata. By storing the chunk text in metadata, we avoid a secondary DB lookup — the retrieved context is directly available.

---

## 12.5 Batch Upsert

Vectors are upserted in batches of 50:

```python
BATCH_SIZE = 50

for i in range(0, len(vectors), BATCH_SIZE):
    batch = vectors[i: i + BATCH_SIZE]
    index.upsert(vectors=batch, namespace=namespace)
```

**Why 50?** Pinecone's recommended maximum batch size for reliable upserts. Larger batches can fail silently or hit payload limits.

---

## 12.6 Typical Document Statistics

| Document Type | Pages | Chars | Chunks |
|--------------|-------|-------|--------|
| Short paper (4 pages) | 4 | ~20,000 | ~22 |
| Conference paper (8 pages) | 8 | ~40,000 | ~44 |
| Journal paper (15 pages) | 15 | ~75,000 | ~83 |
| PhD thesis chapter (40 pages) | 40 | ~200,000 | ~220 |

A typical project with 5 papers: **~110-440 vectors** in Pinecone.

---

## 12.7 Embedding Model

**Model**: `sentence-transformers/all-MiniLM-L6-v2`  
**Dimensions**: 384 (Note: The code comments say 768 in some places — the actual MiniLM-L6-v2 output is 384-dim. This should be verified against the actual index configuration.)  
**Device**: CPU (not GPU — workers run on CPU instances)  
**Loading**: Once per worker process via `utils/model_cache.py` singleton

```python
# model_cache.py
EMBEDDING_MODEL = "all-MiniLM-L6-v2"

_model_singleton: Optional[SentenceTransformer] = None

def get_embedding_model() -> SentenceTransformer:
    global _model_singleton
    if _model_singleton is None:
        _model_singleton = SentenceTransformer(EMBEDDING_MODEL, device="cpu")
        logger.info("model_cache: SentenceTransformer loaded (%s)", EMBEDDING_MODEL)
    return _model_singleton
```

**Why a singleton?** Loading SentenceTransformer takes ~2-3 seconds and ~400MB RAM. Without caching, each document embedding task would reload the model, making ingestion extremely slow. With the singleton, the model is loaded once when the first task runs and reused for all subsequent tasks in the same worker process.

---

## 12.8 Encoding Performance

```python
embeddings = model.encode(chunks).tolist()
```

- Processes all chunks in a single batch call (much faster than per-chunk calls)
- Returns numpy array → `.tolist()` converts to Python list for JSON serialization
- Typical throughput on CPU: ~50-100 chunks/second

---

## 12.9 Index Configuration

**Index Name**: `researchai-dev` (shared constant, `INDEX_NAME` in both embed and search modules)  
**Metric**: Cosine similarity (vectors normalized for cosine metric)  
**Pods**: Configured at index creation time in Pinecone dashboard  
**Namespace pattern**: `project_<project_uuid>` — one namespace per project

> [!WARNING]
> The index name `researchai-dev` suggests this is still using a development index. A production deployment should create a separate `researchai-prod` index and update the constant. Having production data in a "dev" index is a risk if the index gets purged during testing.
