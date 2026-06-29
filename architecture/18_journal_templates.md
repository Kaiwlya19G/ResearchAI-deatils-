# 18 — Journal Template Pipeline

> **Back to Index**: [00_index.md](00_index.md)

---

## 18.1 Overview

ResearchAI supports multiple journal/conference formatting templates that control page layout, font sizes, column count, section ordering, and citation style applied during export.

---

## 18.2 Supported Templates

| Template | Style | Columns | Font |
|----------|-------|---------|------|
| IEEE Conference | Two-column, 8.5"×11" | 2 | Times New Roman 10pt |
| IEEE Journal | Two-column, 8.5"×11" | 2 | Times New Roman 10pt |
| Springer LNCS | Single-column, A4 | 1 | Times New Roman 12pt |
| Elsevier | Single-column, A4 | 1 | Times New Roman 12pt |
| APA 7th Edition | Double-spaced, 1-inch margins | 1 | Times New Roman 12pt |
| MLA 9th Edition | Double-spaced, 1-inch margins | 1 | Times New Roman 12pt |
| Chicago 17th Edition | Notes-bibliography | 1 | Times New Roman 12pt |
| Custom | User-defined | Any | Any |

---

## 18.3 Template Selection

Templates are set at the project level and can be changed at export time:

```python
# In project setup:
project.template = "IEEE Conference"
project.citation_style = "IEEE"

# At export:
data = request.get_json() or {}
template = data.get("template", project.template)  # Export can override project template
```

---

## 18.4 DOCX Template Application (`routes/export.py`)

The DOCX export applies template settings via `python-docx`:

### Page Setup
```python
section = doc.sections[0]
if template == "IEEE Conference":
    section.top_margin    = Inches(0.75)
    section.bottom_margin = Inches(1.0)
    section.left_margin   = Inches(0.75)
    section.right_margin  = Inches(0.75)
elif template in ("Springer LNCS", "Elsevier"):
    section.top_margin    = Inches(1.0)
    section.bottom_margin = Inches(1.0)
    section.left_margin   = Inches(1.25)
    section.right_margin  = Inches(1.25)
```

### Column Layout
```python
if "IEEE" in template:
    # Two-column layout via OOXML
    sectPr = section._sectPr
    cols = OxmlElement('w:cols')
    cols.set(qn('w:num'), '2')
    cols.set(qn('w:space'), '720')  # 0.5" between columns
    sectPr.append(cols)
```

### Typography
```python
# Section headings
heading = doc.add_heading('', level=1)
heading_run = heading.add_run(section_title)
heading_run.font.name = 'Times New Roman'
heading_run.font.size = Pt(10 if "IEEE" in template else 12)
heading_run.font.bold = True
heading.paragraph_format.space_before = Pt(6)
heading.paragraph_format.space_after = Pt(3)

# Body text
para = doc.add_paragraph()
para_run = para.add_run(section_text)
para_run.font.name = 'Times New Roman'
para_run.font.size = Pt(10 if "IEEE" in template else 12)
para.paragraph_format.line_spacing = 1.0 if "IEEE" in template else 2.0  # APA/MLA = double-spaced
para.paragraph_format.first_line_indent = Inches(0.25)
```

### Abstract Formatting
IEEE Abstract uses a special indented single-column format:
```python
abstract_para = doc.add_paragraph()
abstract_para.paragraph_format.left_indent = Inches(0.25)
abstract_para.paragraph_format.right_indent = Inches(0.25)
abstract_run = abstract_para.add_run("Abstract — " + paper.abstract)
abstract_run.font.name = 'Times New Roman'
abstract_run.font.size = Pt(9)  # IEEE uses 9pt for abstract
abstract_run.font.italic = True
```

---

## 18.5 Section Order by Template

Different journals have different standard section orders:

| Template | Section Order |
|----------|--------------|
| IEEE/Springer | Abstract, Keywords, Introduction, Literature Review, Methodology, System Architecture, Implementation, Results, Discussion, Conclusion, Future Work, References |
| APA | Abstract, Introduction, Method, Results, Discussion, References |
| Custom | Configurable via `project.sections_config` JSON |

```python
SECTION_ORDERS = {
    "IEEE Conference": [
        "abstract", "keywords_text", "introduction", "literature_review",
        "research_gap", "methodology", "system_architecture", "implementation",
        "results", "discussion", "conclusion", "future_work", "references_text"
    ],
    "APA 7th Edition": [
        "abstract", "introduction", "methodology", "results", "discussion", "references_text"
    ],
}

section_order = SECTION_ORDERS.get(template, SECTION_ORDERS["IEEE Conference"])
```

---

## 18.6 LaTeX Template Generation

For LaTeX export, a `.tex` file is generated with the appropriate document class:

```python
if "IEEE" in template:
    doc_class = "\\documentclass[10pt,conference]{IEEEtran}"
    packages = "\\usepackage{cite}\\usepackage{graphicx}\\usepackage{amsmath}"
elif "Springer" in template:
    doc_class = "\\documentclass{llncs}"
    packages = "\\usepackage{graphicx}"
else:
    doc_class = "\\documentclass[12pt]{article}"
    packages = "\\usepackage{cite}\\usepackage{graphicx}\\usepackage{geometry}"
    packages += "\n\\geometry{margin=1in}"

latex_content = f"""
{doc_class}
{packages}

\\title{{{paper.title}}}
\\author{{{author_name}}}

\\begin{{document}}
\\maketitle

\\begin{{abstract}}
{paper.abstract}
\\end{{abstract}}

...
\\end{{document}}
"""
```

---

## 18.7 `sections_config` — Custom Sections

The `project.sections_config` JSON field allows users to define custom section structures:

```json
{
  "sections": [
    { "key": "introduction", "title": "1. Introduction", "enabled": true, "order": 1 },
    { "key": "methodology", "title": "2. Materials and Methods", "enabled": true, "order": 2 },
    { "key": "custom_section_1", "title": "3. Experimental Setup", "enabled": true, "order": 3 }
  ]
}
```

Custom sections are generated by adapting the standard section generation pipeline with the custom title and a generic context prompt.
