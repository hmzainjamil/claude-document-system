# Document Pre-Flight Enforcer — Zero Errors Before First Line

## ACTIVATION
Triggers: report, pdf, docx, xlsx, pptx, document, layout, format, build report, create report, generate report

## ROLE
This skill runs BEFORE any document code is written. It enforces a mandatory pre-flight checklist that prevents all known formatting failure modes at design time — before they manifest as broken files.

**No document code without completing pre-flight. This is non-negotiable.**

---

## PRE-FLIGHT CHECKLIST — RUN MENTALLY BEFORE WRITING CODE

### Step 1: Identify Format
```
FORMAT = [PDF | DOCX | XLSX | PPTX]
LIBRARY = [reportlab | python-docx | openpyxl | python-pptx]
LOAD SKILL = [reportlab-pdf-master | docx-format-master | xlsx-format-master | pptx-format-master]
```

### Step 2: Set Dimensions First (before any content)
```python
# PDF (ReportLab):
PW, PH = A4          # 595.27pt × 841.89pt
LM = RM = 12*mm      # 12mm margins
TM = BM = 15*mm
UW = PW - LM - RM    # ≈ 186mm usable width — ASSERT all tables ≤ this

# DOCX (python-docx):
section.page_width  = Inches(8.27)   # A4
section.page_height = Inches(11.69)
section.left_margin = section.right_margin = Cm(2.0)
# Usable = 21 - 2*2 = 17cm — ASSERT all col_widths sum ≤ 17cm

# XLSX (openpyxl):
# No page dims — set column widths explicitly on every column with content

# PPTX (python-pptx):
prs.slide_width  = Inches(13.33)   # 16:9 widescreen
prs.slide_height = Inches(7.5)
# Every shape: explicit left, top, width, height — no defaults
```

### Step 3: Declare ONE Data Source
```python
# ✅ CORRECT — single source of truth for all metrics in the document
DATA = {
    'client':  'Acme Corp',
    'period':  'May 2026',
    'spend':   4521.00,
    'clicks':  1440,
    'ctr':     0.0312,
    'cpc':     3.14,
    'roas':    1.63,
    'conv':    47,
}
# Every table cell, every chart series, every headline text box
# must reference DATA['key'] — NEVER hardcode the same number twice
```

### Step 4: Column Width Math (for tables)
```python
# ❌ WRONG — guessing column widths
cols = [40, 30, 20, 20, 30]  # mm — do these add up? Who knows.

# ✅ CORRECT — calculate and assert before any table code
UW_MM = float((PW - LM - RM) / mm)   # e.g. 186.0mm for A4 12mm margins
cols   = [50, 35, 25, 25, 48]         # designed to fit content
assert sum(cols) <= UW_MM - 0.5, f"Columns {sum(cols)}mm > {UW_MM}mm usable"
# Pass: cols sum = 183mm ≤ 185.5mm ✓
```

### Step 5: Plan Page Breaks (PDF only)
```python
# Plan sections BEFORE writing story:
# Section 1: Cover (1 page, hard PageBreak after)
# Section 2: Executive Summary (fits on 1 page, CondPageBreak(80*mm) after)
# Section 3: Campaign data table (21 rows × 7.76mm/row = 163mm, fits A4)
# Section 4: City breakdown (variable, CondPageBreak(80*mm) between each city)
# Rule: PageBreak() ONLY for cover. Everything else → CondPageBreak(80*mm)
```

### Step 6: Style Registry (PDF only)
```python
# Declare all styles at top of script using idempotent reg() helper:
ss = getSampleStyleSheet()
def reg(name, parent='Normal', **kw):
    try: ss.add(ParagraphStyle(name, parent=ss[parent], **kw))
    except KeyError: pass
    return ss[name]

# Register ALL styles here — never inside loops or functions that repeat:
Body    = reg('Body',    fontSize=10, leading=14)
CellSm  = reg('CellSm', fontSize=8.5, leading=11)
H1      = reg('H1',     fontSize=16, leading=20, fontName='Helvetica-Bold')
H2      = reg('H2',     fontSize=13, leading=17, fontName='Helvetica-Bold')
KPI     = reg('KPI',    fontSize=22, leading=26, fontName='Helvetica-Bold')
```

### Step 7: Plan QA Block at End
```python
# Write this at BOTTOM of script BEFORE filling in content:
OUTPUT = os.path.expanduser('~/Downloads/report.pdf')
# ... all build code here ...
doc.build(story)
qa_document(OUTPUT)  # ← this line must exist in every script
```

---

## PRE-FLIGHT COMMENT BLOCK — PASTE AT TOP OF EVERY SCRIPT

```python
#!/usr/bin/env python3
"""
# ═══════════════════════════════════════════════════════
# PRE-FLIGHT CHECKLIST — COMPLETED BEFORE WRITING CODE
# ═══════════════════════════════════════════════════════
# FORMAT   : PDF (ReportLab / Platypus)
# DIMS     : A4 | LM=RM=12mm | TM=BM=15mm | UW≈186mm
# DATA_SRC : DATA dict — single source for all metrics
# COLS_OK  : sum(cols)=183mm ≤ 185.5mm ✓ (assert in cw())
# BREAKS   : Cover=PageBreak | Sections=CondPageBreak(80mm)
# STYLES   : Registered once at top via reg() helper
# FONTS    : Helvetica / Helvetica-Bold (always available)
# QA       : qa_document(OUTPUT) at end — pymupdf page scan
# OUTPUT   : ~/Downloads/
# AUTHOR   : Hafiz Muhammad Zulqarnain
# ═══════════════════════════════════════════════════════
"""
```

Adapt for other formats:
```python
# FORMAT   : DOCX (python-docx)
# DIMS     : A4 8.27×11.69" | margins=2cm | USABLE=17cm
# COLS_OK  : sum(col_widths_cm) ≤ 17cm ✓

# FORMAT   : XLSX (openpyxl)
# DIMS     : No page dims | explicit col widths on all cols
# DATA_SRC : df / DATA dict — same for table and charts

# FORMAT   : PPTX (python-pptx)
# DIMS     : 13.33"×7.5" 16:9 | every shape: explicit LTWH
# DATA_SRC : DATA dict — chart series and text from same var
```

---

## LAW 1 — DIMENSIONS BEFORE CONTENT, ALWAYS

Set page/slide size as literally the FIRST code after imports. If dimensions come after content setup, they may be ignored.

```python
# ❌ WRONG — dimensions set late
prs = Presentation()
slide = prs.slides.add_slide(...)  # uses default 10"×7.5" — WRONG
# ... content ...
prs.slide_width = Inches(13.33)    # too late, existing slide is wrong size

# ✅ CORRECT
prs = Presentation()
prs.slide_width  = Inches(13.33)   # set BEFORE adding any slide
prs.slide_height = Inches(7.5)
slide = prs.slides.add_slide(...)  # now uses correct dimensions
```

---

## LAW 2 — DATA DICT DECLARED BEFORE ANY CONTENT FUNCTION

```python
# ✅ CORRECT structure:
import os
from reportlab.lib.pagesizes import A4
from reportlab.lib.units import mm

# 1. Page setup
PW, PH = A4
LM = RM = 12*mm
UW = PW - LM - RM

# 2. Data source — ONE dict
DATA = {
    'client': 'Acme Corp',
    'spend': 4521.00,
    # ...
}

# 3. Styles (after data so they can reference DATA if needed)
# ...

# 4. Build functions
def build_kpi_row(): ...
def build_table(): ...

# 5. Story assembly
story = []
story += build_kpi_row()
story += build_table()

# 6. Build + QA
doc.build(story)
qa_document(OUTPUT)
```

---

## LAW 3 — COLUMN MATH IS MANDATORY BEFORE TABLE CODE

```python
def cw(cols, label="table"):
    """Assert column widths fit usable width. Returns widths in points."""
    UW_MM = float((PW - LM - RM) / mm)
    assert sum(cols) <= UW_MM - 0.5, \
        f"[{label}] Columns {sum(cols):.1f}mm > {UW_MM:.1f}mm usable width. Reduce column widths."
    return [c * mm for c in cols]

# Usage — fail loud before building broken table:
COL_CAMPAIGN = cw([50, 18, 18, 18, 18, 22, 22, 20], label="campaign_table")
# AssertionError: [campaign_table] Columns 186.0mm > 185.5mm usable width.
# → Reduce to [48, 18, 18, 18, 18, 22, 22, 20] = 184mm ✓
```

---

## LAW 4 — LONG TEXT MUST BE Paragraph() — NO EXCEPTIONS

```python
# Anything going into a ReportLab Table cell > 15 chars must be Paragraph():
P = lambda text, style='CellSm': Paragraph(str(text), ss[style])

# ❌ WRONG
row = ['retirement calculator, 401k calculator, roth ira calculator', '$1.55', 'YES']

# ✅ CORRECT
row = [P('retirement calculator, 401k calculator, roth ira calculator'), '$1.55', 'YES']
# Short values (numbers, YES/NO, codes) can stay as plain strings
```

---

## LAW 5 — CHART AND TABLE USE SAME VARIABLE

```python
# ❌ WRONG — data defined twice, will diverge
chart_ctrs = [0.031, 0.042, 0.028]   # for chart
table_ctrs = [0.032, 0.041, 0.029]   # for table ← already wrong!

# ✅ CORRECT
ctrs = [0.031, 0.042, 0.028]          # ONE list
avg_ctr = sum(ctrs) / len(ctrs)       # ONE calculation

# Chart:
chart_data.add_series('CTR', tuple(ctrs))

# Table cell:
P(f"Avg CTR: {avg_ctr:.1%}", 'CellSm')

# Headline:
f"Average CTR: {avg_ctr:.1%}"         # same variable
```

---

## LAW 6 — PAGE BREAKS: CondPageBreak NOT PageBreak

```python
from reportlab.platypus import CondPageBreak, PageBreak

# ❌ WRONG — creates blank page if section fits on current page
story.append(PageBreak())

# ✅ CORRECT — only breaks if < 80mm remains on current page
story.append(CondPageBreak(80*mm))

# Only use hard PageBreak() for:
# 1. After cover page (always full page)
# 2. Before a section that must start on a fresh page (e.g., legal appendix)
```

---

## LAW 7 — NARROW ACCENT COLUMNS NEED ZERO PADDING

```python
# Color bar accent column (e.g., 5mm wide):
# 5mm - 6pt left - 6pt right = 5mm - 4.23mm = 0.77mm → NEGATIVE crash

# ✅ CORRECT — zero padding on narrow color columns
tbl.setStyle(TableStyle([
    ('LEFTPADDING',   (0, 0), (0, -1), 0),
    ('RIGHTPADDING',  (0, 0), (0, -1), 0),
    ('TOPPADDING',    (0, 0), (0, -1), 0),
    ('BOTTOMPADDING', (0, 0), (0, -1), 0),
    ('BACKGROUND',    (0, 0), (0, -1), HexColor('#1A1A2E')),
]))
```

---

## LAW 8 — OUTPUT PATH AND BRANDING

```python
import os

OUTPUT  = os.path.expanduser('~/Downloads/report.pdf')  # ALWAYS ~/Downloads/
AUTHOR  = 'Hafiz Muhammad Zulqarnain'
HEADER  = 'Hafiz Muhammad Zulqarnain · May 2026 · Confidential'
COVER   = 'HAFIZ MUHAMMAD ZULQARNAIN · PERFORMANCE INTELLIGENCE UNIT'

# Never save to Desktop, current dir, /tmp, or any other path.
# Always print the full path after saving:
doc.build(story)
print(f"Saved: {OUTPUT} ({os.path.getsize(OUTPUT):,} bytes)")
```
