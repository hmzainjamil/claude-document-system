# claude-document-system

3-layer enterprise document pipeline: preflight checklist + format masters (PDF/DOCX/XLSX/PPTX) + post-build QA agent.

![PDF](https://img.shields.io/badge/PDF-ReportLab-blue?style=flat&labelColor=555) ![DOCX](https://img.shields.io/badge/DOCX-python--docx-green?style=flat&labelColor=555) ![XLSX](https://img.shields.io/badge/XLSX-openpyxl-orange?style=flat&labelColor=555) ![PPTX](https://img.shields.io/badge/PPTX-python--pptx-red?style=flat&labelColor=555)

[Concepts](#-concepts) · [How It Works](#-how-it-works) · [Install](#-install) · [Usage](#-usage) · [Config](#-configuration) · [Tips](#-tips-and-tricks-12) · [Troubleshooting](#-troubleshooting) · [Architecture](#-architecture) · [Startups](#️-startups--businesses)

---

## 🧠 CONCEPTS

| Feature | Location | Description |
|---|---|---|
| doc-preflight | `skills/doc-preflight/SKILL.md` | Pre-build checklist: dims, data source, col math, QA comment |
| PDF Master | `skills/reportlab-pdf-master/SKILL.md` | ReportLab 12 hard laws — column math, no WRAP mode, brand palette |
| DOCX Master | `skills/docx-official/SKILL.md` | python-docx patterns: styles, tables, header/footer, TOC |
| XLSX Master | `skills/xlsx-official/SKILL.md` | openpyxl patterns: formulas, charts, named ranges, data validation |
| PPTX Master | `skills/pptx-official/SKILL.md` | python-pptx patterns: master slides, animations, brand consistency |
| QA Agent | `skills/document-qa-agent/SKILL.md` | Post-build: pymupdf render, page count, font check, corruption test |
| Doc Factory | `bin/doc-factory.py` | CLI runner: `--qa file.pdf` auto-detects format, runs QA |
| Document Orchestrator | `skills/document-orchestrator/SKILL.md` | Routes to correct format specialist, enforces DATA dict pattern |
| Brand Palette Extractor | `tools/brand_extractor.py` | Extracts hex colors from client URL for branded PDFs |
| Batch Generator | `tools/batch_generator.py` | Per-prospect PDF generation with unique branding |
| Template System | `templates/` | Reusable document templates by document type |
| Audit PDF Builder | `builders/audit_pdf.py` | 11-page 360° audit PDF with client brand palette |

### 🔥 Hot

| Feature | Location | Description |
|---|---|---|
| ReportLab 12 Laws | `skills/reportlab-pdf-master/SKILL.md` | Learned from broken reports — never repeat same errors |
| QA Agent | `skills/document-qa-agent/SKILL.md` | Renders page 1 to PNG — catches invisible corruption |
| col math assertion | `skills/doc-preflight/SKILL.md` | `abs(sum(COL_WIDTHS) - USABLE_W) < 1.0` — prevents overflow |
| Brand Extractor | `tools/brand_extractor.py` | Each prospect gets their own brand palette PDF |
| Doc Factory CLI | `bin/doc-factory.py` | Single command QA any document: `python3 doc-factory.py --qa file.pdf` |

---

## ⚙️ HOW IT WORKS

```
Document Request
    │
    ▼
Layer 1: doc-preflight
    ├── Format? (PDF/DOCX/XLSX/PPTX)
    ├── Dimensions?
    ├── Data source?
    ├── Column math: sum(COL_WIDTHS) == USABLE_W?
    └── QA comment block added?

    │ (preflight passes)
    ▼
Layer 2: Format Master
    ├── PDF → reportlab-pdf-master (12 hard laws)
    ├── DOCX → docx-official
    ├── XLSX → xlsx-official
    └── PPTX → pptx-official

    │ (document built)
    ▼
Layer 3: document-qa-agent
    ├── File exists and non-zero size?
    ├── Opens without error?
    ├── Page/sheet count correct?
    ├── Font embedding (PDF)?
    ├── Page 1 rendered to /tmp/doc_qa_page1.png?
    └── Exit 0 = PASS | Exit 1 = FAIL
```

---

## 🚀 INSTALL

```bash
git clone https://github.com/hmzainjamil/claude-document-system
cd claude-document-system

pip install -r requirements.txt
# reportlab, pymupdf, python-docx, openpyxl, python-pptx, pillow, requests

# Install bin script
cp bin/doc-factory.py ~/.claude/bin/
chmod +x ~/.claude/bin/doc-factory.py

# Install skills into Claude Code
cp -r skills/* ~/.claude/skills/

# Test all formats
python3 tests/test_all_formats.py
```

---

## 📟 USAGE

```bash
# QA a PDF
python3 ~/.claude/bin/doc-factory.py --qa ~/Downloads/report.pdf

# QA an XLSX with specific sheets
python3 ~/.claude/bin/doc-factory.py --qa ~/Downloads/data.xlsx --sheets "Revenue" "Costs"

# QA a PPTX with page count check
python3 ~/.claude/bin/doc-factory.py --qa ~/Downloads/deck.pptx --slides 15

# Build audit PDF with brand palette
python3 builders/audit_pdf.py \
  --client "Acme Corp" \
  --url "https://acmecorp.com" \
  --output ~/Downloads/acme_audit.pdf

# Extract brand palette from URL
python3 tools/brand_extractor.py --url "https://example.com"

# Batch generate per-prospect PDFs
python3 tools/batch_generator.py \
  --template templates/audit_template.py \
  --prospects prospects.csv \
  --output ~/Downloads/prospect_pdfs/

# Run preflight check
python3 tools/preflight_check.py --format pdf --cols "200,150,100" --page-width 595
```

---

## ⚙️ CONFIGURATION

| Variable | Default | Description |
|---|---|---|
| `DEFAULT_PAGE_SIZE` | `A4` | `A4` or `LETTER` |
| `DEFAULT_MARGIN_PT` | `36` | Page margin in points (0.5 inch) |
| `BRAND_EXTRACTOR_TIMEOUT` | `10` | Seconds to wait for brand color extraction |
| `QA_RENDER_DPI` | `150` | DPI for page-1 render in QA agent |
| `QA_OUTPUT_DIR` | `/tmp` | Directory for QA renders |
| `BATCH_CONCURRENCY` | `4` | Parallel document generation workers |
| `FONT_DIR` | `fonts/` | Custom font directory |
| `TEMPLATE_DIR` | `templates/` | Document template directory |
| `COL_MATH_TOLERANCE` | `1.0` | Max allowable column width overflow (points) |

---

## 💡 TIPS AND TRICKS (12)

[PDF](#tips-pdf) · [XLSX](#tips-xlsx) · [QA](#tips-qa) · [Branding](#tips-branding)

<a id="tips-pdf"></a>■ **ReportLab PDF (3)**

| Tip | Source |
|---|---|
| Never use WRAP overflow mode — use fixed column widths, let text truncate | ReportLab 12 Laws |
| Always `sum(col_widths) == usable_width` before building table — overflow ruins layout | Preflight law |
| Register custom fonts with `pdfmetrics.registerFont` before any text element | ReportLab docs |

<a id="tips-xlsx"></a>■ **XLSX / openpyxl (3)**

| Tip | Source |
|---|---|
| Use `NamedStyle` for consistent formatting — apply once, reuse everywhere | openpyxl docs |
| `data_validation` on input cells prevents bad data before it reaches formulas | XLSX master |
| `write_only=True` for large sheets (10k+ rows) — 5× faster generation | openpyxl performance |

<a id="tips-qa"></a>■ **QA Agent (3)**

| Tip | Source |
|---|---|
| QA renders page 1 to PNG — visual check catches layout issues code misses | QA agent |
| Exit code 1 = hard fail — never deliver a document that failed QA | QA agent laws |
| Run QA in CI/CD pipeline for document services — catches regressions automatically | Integration guide |

<a id="tips-branding"></a>■ **Brand Palette (3)**

| Tip | Source |
|---|---|
| `brand_extractor.py` pulls primary, secondary, accent colors from any URL | Brand extractor |
| Cache brand palettes — same client URL should not be scraped repeatedly | Batch generator |
| Fallback palette: `#2C3E50, #E74C3C, #ECF0F1` — professional if extraction fails | Brand extractor |

---

## 🔧 TROUBLESHOOTING

| Issue | Fix |
|---|---|
| PDF table overflow | Run preflight: `sum(col_widths)` must equal `usable_width` |
| Font not found in PDF | Register font before use: `pdfmetrics.registerFont(...)` |
| QA render fails | `pip install pymupdf` — `import fitz` |
| XLSX formula errors | Use `=` prefix and English function names |
| PPTX layout broken | Check slide dimensions match template master |
| Brand extractor timeout | Increase `BRAND_EXTRACTOR_TIMEOUT` or use fallback palette |
| Batch gen slow | Increase `BATCH_CONCURRENCY` — check CPU cores available |
| `doc-factory.py` not found | `cp bin/doc-factory.py ~/.claude/bin/ && chmod +x` |

---

## 📊 ARCHITECTURE

```
claude-document-system/
├── skills/
│   ├── doc-preflight/SKILL.md
│   ├── reportlab-pdf-master/SKILL.md
│   ├── docx-official/SKILL.md
│   ├── xlsx-official/SKILL.md
│   ├── pptx-official/SKILL.md
│   ├── document-qa-agent/SKILL.md
│   └── document-orchestrator/SKILL.md
├── bin/
│   └── doc-factory.py          # CLI QA runner
├── builders/
│   └── audit_pdf.py            # 11-page audit PDF
├── tools/
│   ├── brand_extractor.py      # URL → hex colors
│   ├── batch_generator.py      # Per-prospect PDFs
│   └── preflight_check.py      # Column math validator
├── templates/
│   ├── audit_template.py       # Audit PDF template
│   ├── report_template.py      # Report PDF template
│   └── proposal_template.py    # Proposal PDF template
├── fonts/                      # Custom font files
├── tests/
│   └── test_all_formats.py
├── requirements.txt
└── .env.example
```

---

## 📋 REPORTLAB 12 HARD LAWS

| # | Law |
|---|---|
| 1 | Always compute `usable_width = page_width - left_margin - right_margin` |
| 2 | `sum(col_widths) == usable_width` — assertion before any table |
| 3 | Never use `OVERFLOW_MODE = WRAP` — causes unpredictable layout |
| 4 | Register all fonts before first use |
| 5 | Use `inch` units, not points, for layout measurements |
| 6 | Always set `leading` explicitly — default is too tight |
| 7 | Build table data as list-of-lists before creating `Table` object |
| 8 | Always define `TableStyle` separately from table creation |
| 9 | Use `KeepTogether` for sections that must not split across pages |
| 10 | Page numbers via `onFirstPage` / `onLaterPages` canvas callbacks |
| 11 | QA render page 1 to PNG after every build |
| 12 | Exit 0 = PASS, Exit 1 = FAIL — never deliver failing documents |

---

## ☠️ STARTUPS / BUSINESSES

| This Repo / Feature | Replaced |
|---|---|
| 3-layer document pipeline | Broken PDFs delivered to clients |
| ReportLab 12 Laws | Same column overflow bug repeated 5 sessions |
| QA Agent | Invisible corruption caught only when client opened file |
| Brand Extractor | Generic black-and-white PDFs for every client |
| Batch Generator | Manual per-prospect PDF customization |
| Doc Factory CLI | No standardized QA process |
| Preflight Checklist | Missing dimensions discovered mid-build |
| Document Orchestrator | Wrong format specialist loaded |

---

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=hmzainjamil/claude-document-system&type=Date)](https://star-history.com/#hmzainjamil/claude-document-system&Date)

---
<div align="center">Built by <a href="https://github.com/hmzainjamil">HMZ</a> · Part of HMZ Claude AI System</div>

---

## 🔬 QA METRICS TARGETS

| Check | Pass Threshold |
|---|---|
| File size | >10KB (not empty) |
| Page count | Matches expected |
| Font embedding | 100% embedded (PDF) |
| Page 1 render | PNG generated, non-blank |
| Sheet count | Matches expected (XLSX) |
| Slide count | Matches expected (PPTX) |
| Table overflow | 0 tables overflowing margins |
| Color profile | RGB (screen) or CMYK (print) |

---

## 📐 COLUMN MATH EXAMPLES

```python
# A4 PDF example
PAGE_W = 595  # A4 width in points
LEFT_M = RIGHT_M = 36  # 0.5 inch margins
USABLE_W = PAGE_W - LEFT_M - RIGHT_M  # = 523 points

# Column widths must sum to exactly USABLE_W
COL_WIDTHS = [200, 150, 100, 73]  # sum = 523
assert abs(sum(COL_WIDTHS) - USABLE_W) < 1.0  # MUST PASS

# LETTER PDF example
PAGE_W = 612
USABLE_W = 612 - 72 = 540  # 1-inch margins
COL_WIDTHS = [180, 180, 180]  # sum = 540 ✓
```

---

## 🔄 CONTRIBUTING

Contributions welcome for:
- Additional document type support (ODT, CSV, HTML)
- More QA checks in document-qa-agent
- New audit PDF templates (SEO, Meta Ads, etc.)
- Integration with cloud storage (S3, GDrive)
