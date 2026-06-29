# 17 — Diagram Studio

> **Back to Index**: [00_index.md](00_index.md)

---

## 17.1 Overview

Diagram Studio is an AI-powered diagramming feature within the paper editor. It scans the paper for ideal diagram placement opportunities, generates appropriate diagrams or AI illustrations, and inserts them as `[[diagram:UUID]]` tags in the paper text.

**Route**: `routes/paper.py` (diagram endpoints)  
**Screen**: `screens/diagrams.js`  
**Sidebar Position**: Last item in the "Tools" sidebar section (renamed from "Diagrams & Graphs")

---

## 17.2 Diagram Types

| Type | Generation Method | Output |
|------|-----------------|--------|
| Flowchart | NVIDIA `qwen/qwen2-72b-instruct` → Mermaid code | SVG (client-side render) |
| Sequence Diagram | Same LLM → Mermaid code | SVG |
| Mind Map | Same LLM → Mermaid code | SVG |
| Architecture Diagram | Same LLM → Mermaid code | SVG |
| Graph/Chart | Same LLM → Mermaid code | SVG |
| **AI Illustration** | NVIDIA Stable Diffusion XL | PNG (base64 or URL) |

---

## 17.3 Opportunity Scanning

### API
`POST /api/papers/<id>/diagrams/opportunities`

### Batch Processing Architecture

The paper text is split into paragraphs, batched into groups of 5, and scanned concurrently:

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

batches = [paragraphs[i:i+5] for i in range(0, len(paragraphs), 5)]
all_opportunities = []

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = {
        executor.submit(scan_batch, batch, idx * 5): idx
        for idx, batch in enumerate(batches)
    }
    for future in as_completed(futures):
        result = future.result()
        all_opportunities.extend(result)
```

### Scanning Prompt (Per Batch)

```python
prompt = f"""
You are an expert academic diagram consultant analyzing a research paper.

Analyze these paragraphs from a research paper and identify which paragraphs would
most benefit from a visual diagram. Consider whether the content involves:
- Processes or workflows (→ Flowchart)
- System interactions (→ Sequence Diagram)  
- Hierarchical structures (→ Mind Map)
- Data flows (→ Architecture Diagram)
- Complex technical concepts that need illustration (→ AI Illustration)

Paragraphs:
{formatted_paragraphs}

Return a JSON array of opportunities:
[{{
    "paragraph_id": 0,
    "confidence": 0-100,
    "recommended_type": "Flowchart|Sequence Diagram|Mind Map|Architecture Diagram|AI Illustration",
    "reason": "Why this paragraph needs a diagram",
    "suggested_title": "Descriptive title for the diagram"
}}]

Only include paragraphs with confidence >= 70.
"""
```

### Model Used for Scanning

Primary: Direct call to NVIDIA NIM API with DeepSeek V4-Flash via `call_diagram_ai()`:
```python
def call_diagram_ai(prompt: str) -> str:
    api_key = os.environ.get("NVIDIA_API_KEY_SCAN") or os.environ.get("NVIDIA_API_KEY")
    # Direct NVIDIA NIM call with OpenAI-compatible SDK
    client = openai.OpenAI(api_key=api_key, base_url="https://integrate.api.nvidia.com/v1")
    response = client.chat.completions.create(
        model="deepseek-ai/deepseek-v4-flash",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=1024,
        temperature=0.2
    )
    return response.choices[0].message.content
```

Fallback: `call_ai(prompt, task_type="generic")` (full waterfall router)

---

## 17.4 Diagram Generation

### API
`POST /api/papers/<id>/diagrams/generate`

**Input**:
```json
{
  "paragraph_text": "The system processes requests through...",
  "diagram_type": "Flowchart",
  "theme": "academic",
  "style": "IEEE Style",
  "orientation": "Vertical"
}
```

### Mermaid Diagram Generation

For all structural diagram types:

```python
prompt = f"""
Generate a {diagram_type} diagram in Mermaid.js syntax for the following research content.

Style: {style} ({orientation} orientation)
Theme: {theme}

Content to visualize:
{paragraph_text}

Rules:
- Return ONLY valid Mermaid.js syntax, nothing else
- No markdown code fences
- Use clear, concise node labels
- Maximum 12 nodes for readability
- For flowcharts use: graph TD (top-down) or graph LR (left-right)
"""

mermaid_code = call_diagram_ai(prompt)
```

**Output**:
```json
{ "code": "graph TD\n  A[Input] --> B[Process]\n  B --> C[Output]", "title": "System Flow", "is_image": false }
```

### AI Illustration Generation

For `AI Illustration` type:

```python
# Context-aware prompt generation
context_prompt = f"""
Based on this research content: "{paragraph_text}"
Generate a detailed prompt for an academic illustration image that:
- Shows a technical diagram or schematic
- Is appropriate for a research paper figure
- Visualizes the key concepts described

Keep the description under 77 tokens.
"""
image_prompt = call_ai(context_prompt, task_type="generic", max_tokens=100)

# Call NVIDIA Stable Diffusion XL
response = requests.post(
    "https://ai.api.nvidia.com/v1/genai/stabilityai/stable-diffusion-xl",
    headers={
        "Authorization": f"Bearer {nvidia_key}",
        "Content-Type": "application/json",
        "Accept": "application/json"
    },
    json={
        "text_prompts": [{"text": image_prompt + " academic research diagram, technical illustration, white background"}],
        "cfg_scale": 7,
        "seed": 0,
        "sampler": "K_DPM_2_ANCESTRAL",
        "steps": 25,
        "width": 1024,
        "height": 1024,
    }
)

# Extract base64 PNG
image_b64 = response.json()["artifacts"][0]["base64"]
image_data = f"data:image/png;base64,{image_b64}"
```

**Output**:
```json
{ "code": "data:image/png;base64,...", "title": "AI Generated Illustration", "is_image": true }
```

---

## 17.5 Diagram Storage & Insertion

### Save Diagram
`POST /api/papers/<id>/diagrams`

Creates a `Diagram` record in the database:
```python
diagram = Diagram(
    paper_id=paper.id,
    diagram_type=data["diagram_type"],  # "Flowchart" | "AI Illustration" | ...
    title=data["title"],
    description=data["description"],
    data={
        "code": data["code"],           # Mermaid code or base64 PNG
        "is_image": data["is_image"],   # True for AI Illustration
        "theme": data.get("theme"),
        "style": data.get("style"),
    },
    image_url=data.get("image_url")     # Saved PNG URL (if stored to S3)
)
```

### Tag Insertion
The "Save & Insert" button calls:
`POST /api/papers/<id>/diagrams/insert`

Which inserts `[[diagram:<diagram-uuid>]]` at the target paragraph location in the paper's `raw_content`:

```python
def insert_diagram_tag(paper, section_key, paragraph_index, diagram_id):
    paragraphs = paper.raw_content.get(section_key, "").split("\n\n")
    tag = f"\n\n[[diagram:{str(diagram_id)}]]\n\n"
    paragraphs.insert(paragraph_index + 1, tag)
    paper.raw_content[section_key] = "\n\n".join(paragraphs)
    db.session.commit()
```

---

## 17.6 Frontend UI

### Properties Panel Logic (`diagrams.js`)

The properties panel shows different controls based on diagram type:

**For Mermaid diagrams**:
- Diagram Type selector
- Theme selector (default, neutral, dark, forest, base)
- Style selector (IEEE Style, APA Style, Research Style)
- Orientation (Vertical, Horizontal)
- Mermaid code editor (raw code view + syntax highlighting)

**For AI Illustration** (custom panel — no Mermaid controls):
- Caption input
- Image style selector (Realistic, Schematic, Blueprint, Infographic)
- Re-generate button with custom prompt
- Preview at full resolution

### Mermaid.js Rendering

Client-side rendering via the Mermaid.js library loaded in `index.html`:
```html
<script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
```

```javascript
mermaid.initialize({ startOnLoad: false, theme: selectedTheme });
const { svg } = await mermaid.render('diagram-' + diagramId, mermaidCode);
container.innerHTML = svg;
```

---

## 17.7 Version History

Each regeneration of a diagram is stored as a version entry in `diagram.data.versions`:

```json
{
  "versions": [
    { "timestamp": "2026-01-01T10:00:00Z", "code": "graph TD..." },
    { "timestamp": "2026-01-01T10:05:00Z", "code": "graph LR..." }
  ],
  "current_version": 1
}
```

Users can restore any previous version from the sidebar history panel.
