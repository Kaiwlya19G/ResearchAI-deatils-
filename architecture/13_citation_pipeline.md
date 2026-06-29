# 13 — Citation Pipeline

> **Back to Index**: [00_index.md](00_index.md)

---

## 13.1 Overview

The citation pipeline is a **UUID-based tagging system** that tracks the source of every claim in the generated paper. Citations are injected as opaque UUID placeholders during generation and resolved to formatted strings during export or rendering.

---

## 13.2 Citation Flow

```mermaid
flowchart TD
    A["RAG retrieval\nget_context_with_metadata()"] -->|returns {text, doc_id}| B
    B["Paper section generation\ngenerate_paper_section()"] -->|generates text with source tags| C
    C["Post-processing\ncreate inline citation tags"] --> D
    D["Store: [[cite:doc-uuid]] or\n[[cite:ref-uuid://...]] in paper text"] --> E
    E["Frontend editor\nrender: [[cite:X]] → styled badge"] --> F
    F["Citation Manager Screen\nCRUD on Citation records"] --> G
    G["Export\nreplace_citations() → [1] / (Author, 2020)"]
```

---

## 13.3 Inline Citation Tag Format

Two types of UUID citation tags:

```
[[cite:f7a91b3e-4c2d-4d8e-b9c7-1234567890ab]]   ← Document-sourced citation (doc UUID)
[[cite:ref-uuid://abc123def456]]                  ← Reference-sourced citation (ref UUID)
```

The tag uses double-bracket syntax `[[...]]` to avoid conflict with Markdown and standard bracket notation.

---

## 13.4 Citation Record Schema (Database)

```python
class Citation(BaseModel):
    paper_id    = UUID FK (papers)
    project_id  = UUID FK (projects)
    document_id = UUID FK (documents)   # NULL for web/external references
    number      = Integer               # Sequential citation number (1, 2, 3...)
    authors     = String(512)           # "Smith, J., Jones, A."
    title       = String(512)           # Paper/book title
    journal     = String(255)           # Journal or conference name
    year        = Integer               # Publication year
    doi         = String(255)           # Digital Object Identifier
    url         = String(512)           # Source URL (or "ref-uuid://..." for ref citations)
    style       = String(20)            # "IEEE", "APA", "MLA", "Chicago"
    formatted   = Text                  # Pre-rendered citation string
    is_verified = Boolean               # Human-verified flag
```

---

## 13.5 Citation Generation (During Paper Generation)

When `generate_paper_section()` uses `get_context_with_metadata()`, each retrieved chunk carries a `doc_id`. After generation, the system creates Citation records for each referenced document:

```python
for chunk in chunks_used:
    doc_id = chunk["doc_id"]
    doc = Document.query.get(doc_id)
    
    citation = Citation(
        paper_id=paper.id,
        project_id=paper.project_id,
        document_id=doc.id,
        number=next_citation_number,
        authors=format_authors(doc.author_metadata),
        title=doc.original_name,
        journal=doc.journal_name,
        year=doc.publication_year,
        doi=doc.doi,
        style=paper.citation_style,
        formatted=format_citation(doc, paper.citation_style)
    )
```

---

## 13.6 Citation Styles

### IEEE
```
[1] A. Smith and B. Jones, "Title of the Paper," Journal Name, vol. 10, no. 2, pp. 45-67, 2023.
```
Inline: `[1]`, `[2]`, `[1, 3]`

### APA
```
Smith, A., & Jones, B. (2023). Title of the paper. Journal Name, 10(2), 45-67. https://doi.org/...
```
Inline: `(Smith & Jones, 2023)`, `(Smith et al., 2023)`

### MLA
```
Smith, Alice, and Bob Jones. "Title of the Paper." Journal Name, vol. 10, no. 2, 2023, pp. 45-67.
```
Inline: `(Smith and Jones 45)`

### Chicago
```
Smith, Alice, and Bob Jones. "Title of the Paper." Journal Name 10, no. 2 (2023): 45-67.
```
Inline: Footnote superscripts¹

---

## 13.7 Citation Rendering in the Editor

The frontend editor parses `[[cite:UUID]]` tags and renders them as styled badges:

```javascript
// In editor.js
function renderCitationBadge(uuid) {
    const citation = citations.find(c => c.document_id === uuid || c.url_uuid === uuid);
    if (!citation) return `<span class="citation-tag missing">[?]</span>`;
    
    const label = citation.style === "IEEE" 
        ? `[${citation.number}]`
        : `(${citation.author_short}, ${citation.year})`;
    
    return `<span class="citation-tag" data-id="${uuid}" title="${citation.formatted}">${label}</span>`;
}
```

Clicking a citation badge opens the Citation Workspace for that reference.

---

## 13.8 Citation Export Resolver

In `routes/export.py`, `_get_citation_replacement_helper()` builds a regex-based resolver:

```python
def replace_citations(text: str) -> str:
    pattern = r"\[\[\s*(?:cite:)?\s*([a-zA-Z0-9\.\/-]+)\s*\]\]"
    def repl(match):
        target_id = match.group(1).strip()
        cite_record = citation_map.get(target_id)
        if cite_record:
            return cite_record.formatted or f"[{cite_record.number}]"
        return ""   # remove unknown citations silently
    return re.sub(pattern, repl, text)
```

This resolves all `[[cite:UUID]]` tags before DOCX/PDF/LaTeX export.

---

## 13.9 Reference Discovery (`paper_genration/references.py`)

Beyond document-sourced citations, the system can discover additional web references for the paper's bibliography section. The `discover_references()` function:

1. Takes the paper's topic and keywords
2. Queries known academic databases via DOI lookup (`paper_genration/doi.py`)
3. Formats discovered references in the configured citation style
4. Returns a references list string appended to `references_text`

---

## 13.10 Citation Manager UI

**Screen**: `screenCitationWorkspace`  
**Features**:
- List all citations with source document preview
- Edit individual citation fields (authors, title, year, DOI)
- Mark citations as verified (`is_verified = True`)
- Add new citations manually
- Sort by number/author/year
- Export bibliography as separate file

---

## 13.11 Citation Accuracy Score

The `Paper.citation_accuracy` field stores a 0-100 score based on:
- % of citations with verified DOIs
- % of citations with `is_verified = True`
- Presence of required fields (authors, year, title)

This score is displayed on the Paper quality dashboard.
