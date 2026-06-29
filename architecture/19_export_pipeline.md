# 19 — Export Pipeline

> **Back to Index**: [00_index.md](00_index.md)

---

## 19.1 Supported Export Formats

| Format | Route | Library | Features |
|--------|-------|---------|----------|
| **DOCX** | `POST /api/export/<id>/docx` | python-docx | Full formatting, 2-column IEEE, author info |
| **PDF** | `POST /api/export/<id>/pdf` | python-docx → docx2pdf | Generated via DOCX conversion |
| **LaTeX** | `POST /api/export/<id>/latex` | string template | IEEEtran/LLNCS document class |
| **Overleaf** | `POST /api/export/<id>/overleaf` | ZIP + Overleaf API | Direct upload to Overleaf |

---

## 19.2 Export Pre-Processing (All Formats)

Before any format-specific rendering, the export pipeline:

1. **Resolves citations**: Replace all `[[cite:UUID]]` with formatted citation strings
2. **Removes diagram tags**: `[[diagram:UUID]]` → description text or removed
3. **Applies section order**: Based on template configuration
4. **Loads paper sections**: From individual `Paper.<section>` columns
5. **Fetches user info**: Author name from `User.first_name + last_name`

### Citation Resolution

```python
replace_citations = _get_citation_replacement_helper(paper_id)

# Applied to every section text:
section_text = replace_citations(paper.methodology)
# "...demonstrated by [cite:f7a91b]..." → "...demonstrated by [1]..."
```

Pattern matched:
```python
pattern = r"\[\[\s*(?:cite:)?\s*([a-zA-Z0-9\.\/-]+)\s*\]\]"
```

---

## 19.3 DOCX Export Architecture

### Document Structure
```
DOCX Document
├── Page Setup (margins, columns)
├── Title (18pt Bold, Times New Roman, centered)
├── Authors (11pt, centered)
├── Institution (10pt, italic, centered)
├── Abstract (indented, 9pt for IEEE)
├── Keywords (italic, comma-separated)
├── Section Headings (10-12pt Bold, ALL CAPS for IEEE)
└── Section Body Paragraphs (10-12pt, first-line indent)
    └── Citations resolved: [[cite:X]] → [1]
```

### Section Rendering Loop
```python
for section_key in section_order:
    section_text = getattr(paper, section_key, "") or ""
    section_text = replace_citations(section_text)
    
    if section_key == "references_text":
        _add_references_section(doc, paper, citation_map, template)
        continue
    
    # Add section heading
    heading = doc.add_heading('', level=1)
    heading.add_run(SECTION_LABELS[section_key])
    
    # Add paragraphs
    for paragraph in section_text.split('\n\n'):
        p = doc.add_paragraph(paragraph.strip())
        # Apply template typography
```

### File Response
```python
output = io.BytesIO()
doc.save(output)
output.seek(0)

return send_file(
    output,
    as_attachment=True,
    download_name=f"{paper.title or 'research_paper'}.docx",
    mimetype="application/vnd.openxmlformats-officedocument.wordprocessingml.document"
)
```

---

## 19.4 LaTeX Export

Generates a complete `.tex` file with:
- Appropriate `\documentclass` for template
- `\author`, `\title`, `\maketitle`
- Each section as `\section{...}` with content
- Bibliography as `\bibitem` entries
- Packaged as ZIP: `paper.tex` + `references.bib`

```python
# Bibliography
bib_entries = []
for c in citations:
    bib_entries.append(f"""
@article{{{c.id},
  author  = {{{c.authors}}},
  title   = {{{c.title}}},
  journal = {{{c.journal or ''}}},
  year    = {{{c.year or ''}}},
  doi     = {{{c.doi or ''}}},
}}""")

# In-text citation replacement
def latex_cite_replacement(text, citation_map):
    def repl(match):
        uuid = match.group(1)
        cite = citation_map.get(uuid)
        return f"\\cite{{{cite.id}}}" if cite else ""
    return re.sub(pattern, repl, text)
```

---

## 19.5 Overleaf Export

Creates a ZIP file with the LaTeX content and uploads it to Overleaf via their API:

```python
# Create ZIP in memory
import zipfile
zip_buffer = io.BytesIO()
with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zf:
    zf.writestr("main.tex", latex_content)
    zf.writestr("references.bib", bib_content)
zip_buffer.seek(0)

# Upload to Overleaf (API)
overleaf_response = requests.post(
    "https://www.overleaf.com/api/v1/projects",
    files={"file": ("project.zip", zip_buffer, "application/zip")},
    headers={"Authorization": f"Bearer {overleaf_token}"}
)
project_url = overleaf_response.json()["project_url"]
return jsonify({"overleaf_url": project_url})
```

---

## 19.6 Export Usage Logging

Every export is logged to `UsageLog`:
```python
UsageLog(
    user_id=request.user_id,
    action="export",
    resource_id=str(paper.id),
    meta={"format": "docx", "template": template, "word_count": paper.word_count}
)
```

---

## 19.7 Plan Limits at Export

Export checks plan page limits:
```python
plan = user.plan  # free | starter | pro | institution
limits = Config.PLANS[plan]  # {"pages": 40, ...}

if limits["pages"] != -1 and paper.page_count > limits["pages"]:
    return jsonify({"error": f"Paper exceeds your plan's {limits['pages']} page limit"}), 403
```
