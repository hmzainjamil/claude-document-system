# claude-document-system
3-layer enterprise document pipeline — preflight guard + format masters + QA agent for PDF/DOCX/XLSX/PPTX

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat&labelColor=555&logo=python)
![ReportLab](https://img.shields.io/badge/ReportLab-PDF-red?style=flat&labelColor=555)
![openpyxl](https://img.shields.io/badge/openpyxl-XLSX-green?style=flat&labelColor=555)
![python-docx](https://img.shields.io/badge/python--docx-DOCX-2B579A?style=flat&labelColor=555)
![Claude](https://img.shields.io/badge/Claude-Skills-cc785c?style=flat&labelColor=555)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat&labelColor=555)

[Concepts](#-concepts) · [How It Works](#️-how-it-works) · [Install](#-install) · [Usage](#-usage) · [Tips](#-tips-and-tricks-12) · [Startups](#️-startups--businesses)

---

## 🧠 CONCEPTS

| Feature | Location | Description |
|---------|----------|-------------|
| [**doc-factory.py**](doc-factory.py) | `doc-factory.py` | CLI runner — validates any document format post-build |
| [**doc-preflight**](doc-preflight-SKILL.md) | `doc-preflight-SKILL.md` | Pre-build checklist: format/dims/data_source/col_math verified before first line of code |
| [**document-orchestrator**](document-orchestrator-SKILL.md) | `document-orchestrator-SKILL.md` | Master router — loads correct format specialist, single DATA dict pattern |
| [**document-qa-agent**](document-qa-agent-SKILL.md) | `document-qa-agent-SKILL.md` | Post-build QA: renders page 1 to PNG, checks fonts, columns, math |
| [**reportlab-pdf-master**](reportlab-pdf-master-SKILL.md) | `reportlab-pdf-master-SKILL.md` | 12 hard laws for ReportLab — learned from broken reports across sessions |
| [**Column Math Guard**](doc-preflight-SKILL.md) | `abs(sum(COL_WIDTHS) - USABLE_W) < 1.0` | Assertion before every table — prevents overflow/cutoff |

### 🔥 Hot

| Feature | Location | Description |
|---------|----------|-------------|
| [**3-Layer Guard**](doc-factory.py) | `doc-factory.py` | Preflight → Build → QA: nothing ships without passing all 3 layers |
| [**Page 1 PNG Render**](document-qa-agent-SKILL.md) | `pymupdf` | QA renders first page to `/tmp/doc_qa_page1.png` — visual proof of output |
| [**12 Hard Laws**](reportlab-pdf-master-SKILL.md) | `reportlab-pdf-master` | Non-negotiable rules from real failures — zero repeat errors |

---

## ⚙️ HOW IT WORKS

```
User requests document
         ↓
LAYER 1: doc-preflight
  ├── Format confirmed (PDF/DOCX/XLSX/PPTX)
  ├── Dimensions set
  ├── Data source identified
  ├── Column math asserted: abs(sum(COL_WIDTHS) - USABLE_W) < 1.0
  └── QA comment block written
         ↓
LAYER 2: document-orchestrator
  ├── Loads correct format specialist
  ├── Single DATA dict — all metrics derived via lambda
  └── Builds document
         ↓
LAYER 3: document-qa-agent
  ├── Opens file with format library (pymupdf/python-docx/openpyxl/pptx)
  ├── Renders page 1 → /tmp/doc_qa_page1.png
  ├── Checks: fonts, column widths, no overflow, math correct
  └── Exit 0 = PASS, Exit 1 = FAIL (with reason)
```

---

## 🚀 INSTALL

```bash
git clone https://github.com/hmzainjamil/claude-document-system
cd claude-document-system
pip install reportlab pymupdf python-docx openpyxl python-pptx
cp doc-factory.py ~/.claude/bin/
cp *-SKILL.md ~/.claude/skills/  # or install via Claude skill system
```

---

## 📟 USAGE

```bash
# Post-build QA on any document
python3 ~/.claude/bin/doc-factory.py --qa ~/Downloads/report.pdf
python3 ~/.claude/bin/doc-factory.py --qa ~/Downloads/report.xlsx --sheets "Summary" "Detail"
python3 ~/.claude/bin/doc-factory.py --qa ~/Downloads/report.pptx --slides 5

# Auto-detects format from extension
# Exit 0 = PASS, Exit 1 = FAIL
```

---

## 💡 TIPS AND TRICKS (12)

[preflight](#tips-preflight) · [reportlab](#tips-reportlab) · [qa](#tips-qa) · [formats](#tips-formats)

<a id="tips-preflight"></a>■ **Preflight (3)**

| Tip | Source |
|-----|--------|
| Column math: `abs(sum(COL_WIDTHS) - USABLE_W) < 1.0` — run this assertion before every table | [HMZ](https://github.com/hmzainjamil) |
| `USABLE_W = PAGE_W - LEFT_MARGIN - RIGHT_MARGIN` — always derive, never hardcode | [HMZ](https://github.com/hmzainjamil) |
| Write the QA comment block first: `# FORMAT: PDF | DIMS: A4 | SOURCE: DATA dict` | [DigiMinds](https://github.com/hmzainjamil) |

<a id="tips-reportlab"></a>■ **ReportLab (3)**

| Tip | Source |
|-----|--------|
| Never use `wrapOn` without checking `w > 0` — causes silent zero-width cells | [HMZ](https://github.com/hmzainjamil) |
| `KeepTogether([table, spacer])` prevents table orphans across pages | [ReportLab Docs](https://reportlab.com/docs/reportlab-userguide.pdf) |
| Brand palette: extract from client URL, store as `HexColor('#RRGGBB')` at top of script | [HMZ](https://github.com/hmzainjamil) |

<a id="tips-qa"></a>■ **QA Agent (3)**

| Tip | Source |
|-----|--------|
| `pymupdf` renders page 1 to PNG — visually verify before sending to client | [HMZ](https://github.com/hmzainjamil) |
| QA exit code 1 + reason printed to stderr — pipe to `tcc` for auto-retry | [DigiMinds](https://github.com/hmzainjamil) |
| Run `doc-factory.py --qa` on EVERY document before delivering — non-negotiable | [HMZ](https://github.com/hmzainjamil) |

<a id="tips-formats"></a>■ **Format Specifics (3)**

| Tip | Source |
|-----|--------|
| XLSX: `openpyxl` column widths in characters, not pixels — multiply by 7 for pixel estimate | [HMZ](https://github.com/hmzainjamil) |
| DOCX: `python-docx` table column widths in EMUs — `Inches(1.5)` = clean notation | [python-docx Docs](https://python-docx.readthedocs.io) |
| PPTX: slide dimensions default 10×7.5 inches — always set explicitly in `Presentation()` | [HMZ](https://github.com/hmzainjamil) |

---

## ☠️ STARTUPS / BUSINESSES

| This Repo / Feature | Replaced |
|-|-|
| **3-layer doc pipeline** | [Docupilot](https://docupilot.app), [Documint](https://documint.me), [Carbone](https://carbone.io) |
| **ReportLab PDF master** | [Adobe InDesign](https://adobe.com/indesign), [Canva PDF](https://canva.com), [PDFMonkey](https://pdfmonkey.io) |
| **document-qa-agent** | [PDF.co](https://pdf.co), [iLovePDF](https://ilovepdf.com) validation — runs local, free |
| **doc-factory CLI** | [Docmosis](https://docmosis.com), [Windward](https://windward.net) — zero license fee |

---

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=hmzainjamil/claude-document-system&type=Date)](https://star-history.com/#hmzainjamil/claude-document-system&Date)

---

<div align="center">
Built by <a href="https://github.com/hmzainjamil">HMZ</a> · Part of the <a href="https://github.com/hmzainjamil/claude-ai-system">HMZ Claude AI System</a> · Zero broken documents
</div>
