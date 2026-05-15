# SKILL: reportlab-pdf-master
# Author: Hafiz Muhammad Zulqarnain
# Purpose: Zero-defect ReportLab PDF generation — 12 hard laws, no exceptions
# Version: 2.0.0 (Laws 10-12 added by enterprise doc system)

## IDENTITY

You are the ReportLab PDF specialist. Every PDF you produce must open without error, display without overflow, and contain no blank pages. You enforce 12 laws. No law is optional.

---

## LAW 1 — DIMENSIONS FIRST

Page dimensions and usable width must be the first variables in every PDF script.

❌ WRONG:
```python
doc = SimpleDocTemplate("report.pdf")  # default size, unknown margins
```

✅ CORRECT:
```python
from reportlab.lib.pagesizes import A4
from reportlab.lib.units import mm

PAGE_W, PAGE_H = A4          # 595.28 x 841.89 points
MARGIN = 12 * mm             # 33.97 points
USABLE_W = PAGE_W - 2*MARGIN # 527.34 points
print(f"[PDF] Usable width: {USABLE_W:.1f}pt ({USABLE_W/mm:.1f}mm)")
```

---

## LAW 2 — OUTPUT TO ~/Downloads/ ALWAYS

❌ WRONG:
```python
output = "report.pdf"  # saves to cwd — might not be found
```

✅ CORRECT:
```python
import os
OUTPUT_DIR = os.path.expanduser("~/Downloads")
os.makedirs(OUTPUT_DIR, exist_ok=True)
output_path = os.path.join(OUTPUT_DIR, "client_report_may2025.pdf")
```

---

## LAW 3 — TABLE CELLS ARE ALWAYS Paragraph(), NEVER STRINGS

Raw strings in table cells do not wrap. They overflow and get clipped.

❌ WRONG:
```python
data = [['Campaign Name That Is Long', '$4,521', '3.12%']]
Table(data, colWidths=[...])
```

✅ CORRECT:
```python
from reportlab.platypus import Paragraph
from reportlab.lib.styles import ParagraphStyle

cell_style = ParagraphStyle('cell', fontSize=8, leading=10, wordWrap='CJK')
hdr_style  = ParagraphStyle('hdr',  fontSize=8, leading=10, fontName='Helvetica-Bold',
                            textColor=colors.white, wordWrap='CJK')

def C(text, style=cell_style): return Paragraph(str(text), style)
def H(text):                   return Paragraph(str(text), hdr_style)

data = [[H('Campaign'), H('Spend'), H('CTR')],
        [C('Brand - Search'), C('$4,521'), C('3.12%')]]
```

---

## LAW 4 — COLUMN WIDTHS MUST SUM TO USABLE_W

❌ WRONG:
```python
Table(data, colWidths=[200, 150, 100, 100])  # 550pt > USABLE_W of 527pt — overflow
```

✅ CORRECT:
```python
COLS = [USABLE_W * p for p in [0.40, 0.20, 0.20, 0.20]]
assert abs(sum(COLS) - USABLE_W) < 1.0, f"Column sum error: {sum(COLS):.1f}"
t = Table(data, colWidths=COLS)
```

---

## LAW 5 — CondPageBreak AFTER EVERY MAJOR SECTION

❌ WRONG — table runs off bottom of page into next with no break logic:
```python
story.extend([header, table1, header2, table2])
```

✅ CORRECT:
```python
from reportlab.platypus import CondPageBreak
story.append(header1)
story.append(table1)
story.append(CondPageBreak(80 * mm))  # break if <80mm left on page
story.append(header2)
story.append(table2)
story.append(CondPageBreak(80 * mm))
```

---

## LAW 6 — KeepInFrame FOR SECTIONS THAT MUST NOT SPLIT

❌ WRONG — section splits mid-content across pages:
```python
story += [title, chart, summary_table]  # might split anywhere
```

✅ CORRECT:
```python
from reportlab.platypus import KeepInFrame
section_content = [title, chart, summary_table]
story.append(KeepInFrame(USABLE_W, 160 * mm, section_content, mode='shrink'))
```

---

## LAW 7 — AUTHOR AND METADATA SET ON DOC

```python
doc = SimpleDocTemplate(
    output_path,
    pagesize=A4,
    leftMargin=MARGIN, rightMargin=MARGIN,
    topMargin=MARGIN, bottomMargin=MARGIN,
    author="Hafiz Muhammad Zulqarnain",
    title="Report Title",
    subject="Client Name — Period",
    creator="Claude Code / ReportLab",
)
```

---

## LAW 8 — TABLE STYLE: WORD WRAP AND ALTERNATING ROWS

```python
from reportlab.lib import colors
from reportlab.platypus import TableStyle

NAVY = colors.HexColor("#1A1A59")
GREY = colors.HexColor("#F5F5F5")
WHITE = colors.white

table_style = TableStyle([
    # Header row
    ('BACKGROUND',  (0,0), (-1,0), NAVY),
    ('TEXTCOLOR',   (0,0), (-1,0), WHITE),
    ('FONTNAME',    (0,0), (-1,0), 'Helvetica-Bold'),
    ('FONTSIZE',    (0,0), (-1,0), 8),
    # Data rows
    ('FONTSIZE',    (0,1), (-1,-1), 8),
    ('ROWBACKGROUNDS', (0,1), (-1,-1), [WHITE, GREY]),
    # Grid
    ('GRID',        (0,0), (-1,-1), 0.25, colors.HexColor("#CCCCCC")),
    ('VALIGN',      (0,0), (-1,-1), 'TOP'),
    ('TOPPADDING',  (0,0), (-1,-1), 4),
    ('BOTTOMPADDING',(0,0), (-1,-1), 4),
    ('LEFTPADDING', (0,0), (-1,-1), 6),
    ('RIGHTPADDING',(0,0), (-1,-1), 6),
    # WORDWRAP is handled by Paragraph() cells — no TableStyle needed
])
t.setStyle(table_style)
```

---

## LAW 9 — QA BLOCK MANDATORY AT END

Every PDF script ends with QA. No exceptions.

```python
# At end of every PDF script:
import fitz, os

def qa_pdf(path):
    assert os.path.exists(path), f"File not found: {path}"
    assert os.path.getsize(path) > 10240, f"File too small: {path}"
    doc = fitz.open(path)
    assert len(doc) > 0, "Zero pages"
    for i, page in enumerate(doc):
        chars = len(page.get_text().strip())
        assert chars > 0, f"Blank page detected: page {i+1}"
    # Render page 1 as proof
    pix = doc[0].get_pixmap(dpi=96)
    pix.save("/tmp/qa_page1.png")
    doc.close()
    size_kb = os.path.getsize(path) / 1024
    print(f"[QA] PASS | {len(fitz.open(path))} pages | {size_kb:.1f}KB | {path}")

qa_pdf(output_path)
print(f"\n✓ DELIVERED: {output_path}")
```

---

## LAW 10 — BACKGROUND COLORS AND OVERLAPPING ELEMENTS

Backgrounds drawn AFTER text will cover the text. Always draw backgrounds first.

❌ WRONG — adding a colored flowable after a text paragraph:
```python
story.append(header_text_paragraph)    # text placed first
story.append(colored_background_rect)  # ← covers the text above it
```

❌ WRONG — canvas.drawRect() after canvas.drawString() on same area:
```python
def on_page(canvas, doc):
    canvas.drawString(x, y, "Revenue: $4,521")
    canvas.setFillColor(navy)
    canvas.rect(0, y-5, PAGE_W, 30, fill=1)  # ← covers text
```

✅ CORRECT — all backgrounds in onFirstPage/onLaterPages callback, drawn BEFORE story content:
```python
def draw_page_chrome(canvas, doc):
    """Called before Platypus places any story content on the page."""
    canvas.saveState()

    # 1. Draw all backgrounds first
    canvas.setFillColorRGB(0.10, 0.10, 0.35)  # dark navy
    canvas.rect(0, PAGE_H - 50*mm, PAGE_W, 50*mm, fill=1, stroke=0)  # header bg

    canvas.setFillColorRGB(0.95, 0.95, 0.95)  # light grey
    canvas.rect(0, 0, PAGE_W, 20*mm, fill=1, stroke=0)  # footer bg

    # 2. Draw text/lines on top of backgrounds
    canvas.setFillColorRGB(1, 1, 1)  # white text
    canvas.setFont('Helvetica-Bold', 16)
    canvas.drawString(MARGIN, PAGE_H - 28*mm, "Performance Report")

    canvas.setFillColorRGB(0.5, 0.5, 0.5)  # grey footer text
    canvas.setFont('Helvetica', 7)
    canvas.drawString(MARGIN, 8*mm, f"Page {doc.page}")

    canvas.restoreState()

# Backgrounds drawn in callback — story content placed after:
doc.build(story, onFirstPage=draw_page_chrome, onLaterPages=draw_page_chrome)
```

✅ CORRECT — for Platypus: use KeepInFrame so content sections never bleed into backgrounds:
```python
# If using colored frames as Flowable (e.g. Table with BACKGROUND):
# Draw the colored row as part of TableStyle, not as a separate rect after the table
table_style = TableStyle([
    ('BACKGROUND', (0,0), (-1,0), colors.HexColor("#1A1A59")),  # header bg IN table
    ...
])
```

---

## LAW 11 — DATA SYNC: ONE SOURCE ALWAYS

Every number shown in the PDF — in tables, charts, and headline text — must come from ONE dict. Never hardcode the same metric in two places.

❌ WRONG — same metric typed multiple times, diverges:
```python
# Headline text:
story.append(Paragraph("CTR improved to 3.12%", style))

# Table row 200 lines later:
data_row = [C('CTR'), C('3.1%')]   # ← different! which is right?

# Chart series further down:
bar_chart.data = [[0.031, 0.029]]   # ← 0.031 not 0.0312 — diverged again
```

✅ CORRECT — single DATA dict, all references pull from it:
```python
# ── SINGLE SOURCE OF TRUTH ───────────────────────────────────────
DATA = {
    'client':      "Acme Corp",
    'period':      "May 2025",
    'impressions': 46154,
    'clicks':      1440,
    'spend':       4521.00,
    'conversions': 87,
    'revenue':     7395.00,
}
# Derived metrics — computed once, never typed
DATA['ctr']       = round(DATA['clicks'] / DATA['impressions'], 4)
DATA['cpc']       = round(DATA['spend']  / DATA['clicks'], 2)
DATA['conv_rate'] = round(DATA['conversions'] / DATA['clicks'], 4)
DATA['roas']      = round(DATA['revenue'] / DATA['spend'], 2)

# Print for verification before building:
for k, v in DATA.items():
    print(f"  {k}: {v}")

# All references use DATA:
# Headline:
story.append(Paragraph(f"CTR: {DATA['ctr']:.2%}", style))

# Table row:
data_row = [C('CTR'), C(f"{DATA['ctr']:.2%}")]

# Chart:
bar_chart.data = [[DATA['ctr'], 0.0289, 0.0301]]  # only current period from DATA

# NEVER:
# story.append(Paragraph("CTR: 3.12%", style))        # ← hardcoded, will diverge
# data_row = [C('CTR'), C('3.1%')]                    # ← different rounding, bug
```

Helper for consistent formatting:
```python
def fmt(key, format_type='pct'):
    """Format DATA[key] consistently everywhere."""
    v = DATA[key]
    if format_type == 'pct':   return f"{v:.2%}"
    if format_type == 'money': return f"${v:,.2f}"
    if format_type == 'int':   return f"{v:,}"
    if format_type == 'float': return f"{v:.2f}"
    return str(v)

# Usage:
story.append(Paragraph(f"CTR: {fmt('ctr')}", style))       # → "3.12%"
story.append(Paragraph(f"Spend: {fmt('spend','money')}", style)) # → "$4,521.00"
data_row = [C('CTR'), C(fmt('ctr'))]                        # same value
```

---

## LAW 12 — FONT EMBEDDING AND ENCODING

❌ WRONG — using font names with spaces, or non-standard fonts without registration:
```python
canvas.setFont('Arial Bold', 12)      # Not available in ReportLab by default
canvas.setFont('Times New Roman', 10) # May not render on all systems
canvas.drawString(x, y, "€ Sign")    # Euro sign may fail with latin-1 canvas
```

✅ CORRECT — use Helvetica family (always available), register custom fonts explicitly:
```python
# Safe built-in fonts (always available in ReportLab):
FONT_REGULAR = 'Helvetica'
FONT_BOLD    = 'Helvetica-Bold'
FONT_ITALIC  = 'Helvetica-Oblique'
FONT_BOLDITALIC = 'Helvetica-BoldOblique'

# For custom fonts, register before use:
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
import os

font_path = os.path.expanduser("~/.fonts/Inter-Regular.ttf")
if os.path.exists(font_path):
    pdfmetrics.registerFont(TTFont('Inter', font_path))
    FONT_REGULAR = 'Inter'
else:
    FONT_REGULAR = 'Helvetica'  # fallback

# UTF-8 encoding for canvas.drawString():
def safe_string(text):
    """Encode text safely for canvas.drawString() — handles special chars."""
    return str(text).encode('latin-1', errors='replace').decode('latin-1')

canvas.setFont(FONT_BOLD, 14)
canvas.drawString(x, y, safe_string("Revenue: $4,521 | ROAS: 1.63×"))

# For Platypus Paragraph (recommended — handles encoding automatically):
style = ParagraphStyle('body', fontName=FONT_REGULAR, fontSize=10)
story.append(Paragraph("Revenue: $4,521 | ROAS: 1.63×", style))
# Paragraph handles UTF-8 natively — no manual encoding needed
```

Custom font registration template:
```python
def register_font_family(name, regular_path, bold_path=None, italic_path=None):
    """Register a font family. Falls back to Helvetica if files missing."""
    from reportlab.pdfbase import pdfmetrics
    from reportlab.pdfbase.ttfonts import TTFont
    
    if not os.path.exists(regular_path):
        print(f"[FONT] {name} not found at {regular_path}, using Helvetica")
        return 'Helvetica', 'Helvetica-Bold'
    
    pdfmetrics.registerFont(TTFont(name, regular_path))
    bold_name = name
    if bold_path and os.path.exists(bold_path):
        pdfmetrics.registerFont(TTFont(f'{name}-Bold', bold_path))
        bold_name = f'{name}-Bold'
    
    print(f"[FONT] Registered: {name} ({regular_path})")
    return name, bold_name
```
