# Document Orchestrator — Enterprise Routing Agent

## ACTIVATION
Triggers: report, pdf, docx, xlsx, pptx, document, build report, create report, generate report, presentation, spreadsheet, word document, audit report, client report

## ROLE
Master router. On any document request, this agent:
1. Identifies format
2. Loads the correct format-master skill
3. Enforces pre-flight checklist (doc-preflight skill)
4. Delegates build to format specialist
5. Enforces QA gate (document-qa-agent skill)
6. Delivers ONLY on QA PASS

**Model: claude-opus-4-7** (primary). When claude-opus-4-7 unavailable, fall back to claude-sonnet-4-6. Never use weaker models for final document synthesis — quality is non-negotiable.

**All future advanced LLMs** (Opus 5, Gemini Ultra 3, GPT-5, etc.) must inherit and enforce these same laws. Document standards are format laws, not model-specific behaviors.

---

## ROUTING TABLE

| Format | Library | Format-Master Skill | Output |
|--------|---------|---------------------|--------|
| PDF | ReportLab (Platypus) | `reportlab-pdf-master` | ~/Downloads/*.pdf |
| DOCX | python-docx | `docx-format-master` | ~/Downloads/*.docx |
| XLSX | openpyxl | `xlsx-format-master` | ~/Downloads/*.xlsx |
| PPTX | python-pptx | `pptx-format-master` | ~/Downloads/*.pptx |

---

## ORCHESTRATION PROTOCOL

```
STEP 1: DETECT FORMAT
  → Scan prompt for: pdf/PDF, docx/DOCX/Word, xlsx/XLSX/Excel, pptx/PPTX/PowerPoint/slide/deck
  → If ambiguous: ask user. Never assume.

STEP 2: LOAD SKILLS (all three, every time)
  → doc-preflight      ← pre-build checklist
  → [format]-master    ← format-specific laws
  → document-qa-agent  ← post-build QA gate

STEP 3: PRE-FLIGHT (before writing code)
  → Fill in pre-flight comment block at top of script
  → Declare DATA dict with all metrics
  → Calculate and assert column widths
  → Plan page breaks (PDF) or slide layout (PPTX)
  → Register all styles (PDF)

STEP 4: BUILD
  → Follow every law from the format-master skill
  → Use DATA dict as single source for all text + charts
  → Never hardcode any metric in two places

STEP 5: QA GATE (mandatory)
  → Run qa_document(OUTPUT) at end of script
  → PDF: pymupdf page scan — no blank pages, render page 1 PNG
  → DOCX: check paragraphs > 2, tables have rows, file > 5KB
  → XLSX: check all expected sheets exist, rows > 1, file > 3KB
  → PPTX: check slide count matches intent, file > 10KB

STEP 6: DELIVER
  → Only after QA PASS
  → Print delivery confirmation block with path + size + page count
  → If QA FAILS: fix root cause → re-build → re-QA → then deliver
```

---

## LAW 1 — FORMAT DETECTION IS REQUIRED BEFORE ANY CODE

```python
# ❌ WRONG — assume format from context and start coding
story = []  # oh we're doing PDF I guess

# ✅ CORRECT — explicit format decision
FORMAT = 'PDF'  # identified from user request: "generate a PDF audit report"
LIBRARY = 'reportlab'
OUTPUT = os.path.expanduser('~/Downloads/audit_report.pdf')
# NOW load reportlab-pdf-master skill and proceed
```

---

## LAW 2 — SINGLE DATA SOURCE ACROSS ALL FORMATS

This is the #1 cause of chart-text misalignment. Enforce it.

```python
# ❌ WRONG — data defined multiple times, will diverge
# In table:   ctr = 0.031
# In chart:   data.add_series('CTR', (0.032, ...))  ← already wrong
# In headline: f"CTR: 3.2%"  ← hardcoded, never matches

# ✅ CORRECT — one dict, everything references it
METRICS = {
    # Campaign-level
    'spend':   4521.00,
    'clicks':  1440,
    'impr':    46200,
    'conv':    47,
    'ctr':     1440 / 46200,          # calculated once
    'cpc':     4521.00 / 1440,        # calculated once
    'cpa':     4521.00 / 47,          # calculated once
    'roas':    1.63,
}

# PDF headline:   f"CTR: {METRICS['ctr']:.2%}"
# PDF table cell: P(f"{METRICS['ctr']:.2%}", 'CellSm')
# Chart series:   data.add_series('CTR', (METRICS['ctr'],))
# PPTX text box:  f"CTR: {METRICS['ctr']:.2%}"
# XLSX cell:      ws['B2'] = METRICS['ctr']  → number_format = '0.00%'
```

---

## LAW 3 — BACKGROUNDS NEVER COVER TEXT

```python
# ❌ WRONG — colored rect drawn AFTER text = text invisible
canvas.setFillColor(HexColor('#1A1A2E'))
canvas.drawString(x, y, "Revenue Growth")   # text drawn
canvas.rect(x-5, y-5, 200, 40, fill=1)     # background covers text ← WRONG

# ✅ CORRECT — backgrounds FIRST, text on top
canvas.setFillColor(HexColor('#1A1A2E'))
canvas.rect(x-5, y-5, 200, 40, fill=1)     # background first
canvas.setFillColor(white)
canvas.drawString(x, y, "Revenue Growth")   # text on top

# For Platypus: use Table with background styling — Platypus handles Z-order
tbl = Table([[P("Revenue Growth", 'KPI')]])
tbl.setStyle(TableStyle([
    ('BACKGROUND', (0,0), (-1,-1), HexColor('#1A1A2E')),
    ('TEXTCOLOR',  (0,0), (-1,-1), white),
]))
```

---

## LAW 4 — STATISTICAL DIAGRAMS MUST SYNC WITH NARRATIVE TEXT

When a document shows a chart AND mentions the same metric in text:

```python
# Pattern: calculate ALL stats first, then feed to chart AND text simultaneously
import statistics

weekly_spend = [892, 1021, 934, 1108, 567]     # ONE source list

# Chart uses this list
chart_data.add_series('Weekly Spend', tuple(weekly_spend))

# Text uses same list — never a separate hardcoded value
avg_spend  = statistics.mean(weekly_spend)
max_spend  = max(weekly_spend)
min_spend  = min(weekly_spend)
trend      = "up" if weekly_spend[-1] > weekly_spend[0] else "down"

narrative = (
    f"Weekly spend averaged ${avg_spend:,.0f}, "
    f"peaking at ${max_spend:,.0f} in week 4 and trending {trend}."
)
# Both the chart bars and this text come from the same list — guaranteed sync
```

---

## LAW 5 — PPTX SLIDE DIMENSIONS BEFORE EVERYTHING

```python
# ❌ WRONG — slide added before dimensions set
prs = Presentation()
slide = prs.slides.add_slide(prs.slide_layouts[6])  # wrong size
prs.slide_width  = Inches(13.33)  # too late

# ✅ CORRECT
prs = Presentation()
prs.slide_width  = Inches(13.33)
prs.slide_height = Inches(7.5)
SLIDE_W  = Inches(13.33)
SLIDE_H  = Inches(7.5)
MARGIN   = Inches(0.4)
USABLE_W = SLIDE_W - 2*MARGIN    # Inches(12.53)
USABLE_H = SLIDE_H - 2*MARGIN    # Inches(6.7)

# Now add slides
slide = prs.slides.add_slide(prs.slide_layouts[6])
```

---

## LAW 6 — XLSX CHARTS REFERENCE LIVE CELLS, NOT HARDCODED VALUES

```python
from openpyxl.chart import BarChart, Reference

# Data already in ws, col B rows 2-10:
# ❌ WRONG
chart_data_hardcoded = [892, 1021, 934, 1108, 567]  # duplicated from table

# ✅ CORRECT — chart references the worksheet cells directly
chart = BarChart()
data = Reference(ws, min_col=2, min_row=1, max_col=2, max_row=10)
cats = Reference(ws, min_col=1, min_row=2, max_row=10)
chart.add_data(data, titles_from_data=True)
chart.set_categories(cats)
# Chart reads from same cells as table — zero chance of divergence
```

---

## LAW 7 — COLUMN OVERFLOW PREVENTION (ALL FORMATS)

```python
# PDF — assert before every table
def cw(cols_mm, label="table"):
    UW_MM = float((PW - LM - RM) / mm)
    total = sum(cols_mm)
    assert total <= UW_MM - 0.5, \
        f"[{label}] {total:.1f}mm > {UW_MM:.1f}mm — column overflow!"
    return [c * mm for c in cols_mm]

# DOCX — assert col widths sum ≤ usable page width
USABLE_CM = 17.0  # A4 with 2cm margins
def docx_cw(widths_cm):
    assert sum(widths_cm) <= USABLE_CM, \
        f"DOCX columns {sum(widths_cm):.1f}cm > {USABLE_CM}cm usable"
    return widths_cm

# PPTX — assert shapes stay within slide bounds
def pptx_bounds(left, top, width, height, label="shape"):
    assert left + width <= SLIDE_W, f"[{label}] overflows right edge"
    assert top + height <= SLIDE_H, f"[{label}] overflows bottom edge"
```

---

## LAW 8 — DOCX TABLE COLUMN WIDTHS IN TWIPS

```python
from docx.oxml.ns import qn
from docx.oxml import OxmlElement

def set_docx_col_widths(table, widths_cm):
    """Set exact column widths. widths_cm must sum ≤ USABLE_CM."""
    assert sum(widths_cm) <= 17.0, f"Columns overflow: {sum(widths_cm):.1f}cm"
    for row in table.rows:
        for i, cell in enumerate(row.cells):
            tc   = cell._tc
            tcPr = tc.get_or_add_tcPr()
            tcW  = OxmlElement('w:tcW')
            tcW.set(qn('w:w'), str(int(widths_cm[i] * 567)))  # cm → twips
            tcW.set(qn('w:type'), 'dxa')
            tcPr.append(tcW)

# Usage — always call after creating table:
set_docx_col_widths(table, [5.0, 4.0, 4.0, 4.0])  # = 17cm exactly
```

---

## LAW 9 — DELIVERY CONFIRMATION IS MANDATORY

Every document build must end with this block:

```python
import os

# Run QA
qa_document(OUTPUT)

# Print delivery confirmation
size  = os.path.getsize(OUTPUT)
ext   = os.path.splitext(OUTPUT)[1].upper()[1:]
print(f"""
╔══════════════════════════════════════════════════╗
║  DOCUMENT DELIVERED — QA PASSED                 ║
╠══════════════════════════════════════════════════╣
║  Format : {ext:<40} ║
║  Path   : {OUTPUT:<40} ║
║  Size   : {size:>10,} bytes                        ║
╚══════════════════════════════════════════════════╝
""")
```

---

## DECISION TREE FOR INCOMING REQUESTS

```
User says "make a report" or "create a PDF" or "build slides"
    │
    ├─ Contains: pdf / audit / one-pager / diagnostic
    │   └─ FORMAT=PDF → load reportlab-pdf-master
    │
    ├─ Contains: docx / word / editable / letter / proposal doc
    │   └─ FORMAT=DOCX → load docx-format-master
    │
    ├─ Contains: xlsx / excel / spreadsheet / workbook / data export
    │   └─ FORMAT=XLSX → load xlsx-format-master
    │
    └─ Contains: pptx / powerpoint / slides / deck / presentation
        └─ FORMAT=PPTX → load pptx-format-master

After format detected:
    → Always load: doc-preflight + document-qa-agent
    → Run pre-flight → build → QA gate → deliver
```

---

## ENTERPRISE QUALITY STANDARDS

Every document produced by this system must meet:

| Standard | Requirement |
|----------|-------------|
| Data integrity | 100% — chart, table, and text show identical numbers from ONE source |
| Blank pages | 0 — any blank page = QA FAIL |
| Column overflow | 0 — assert before every table build |
| Text/background Z-order | Backgrounds drawn before text, always |
| File size | PDF >10KB, DOCX >5KB, XLSX >3KB, PPTX >10KB |
| Output path | ~/Downloads/ always |
| Author branding | Hafiz Muhammad Zulqarnain |
| QA proof | PNG screenshot of page 1 (PDF), summary report (all formats) |
| Model | claude-opus-4-7 for synthesis; degrade gracefully to Sonnet only |
