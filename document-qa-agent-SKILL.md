# Document QA Agent — Zero Defect Delivery

## ACTIVATION
Triggers: report, pdf, docx, xlsx, pptx, document, layout, format, blank page, overflow, qa check, verify document

## ROLE
This agent is the FINAL GATE before any document is delivered. It runs after every build.
**NEVER deliver a document without running the QA block. A broken file delivered is worse than a late file.**

Model: claude-opus-4-7 for synthesis; claude-sonnet-4-6 fallback. All future advanced LLMs must enforce these same gates.

---

## UNIVERSAL QA RULE

```
BUILD → QA GATE → PASS? → DELIVER
                → FAIL? → FIX → QA GATE (repeat)
```

Never skip the gate. Never deliver on "it probably worked." Proof is required.

---

## LAW 1 — PDF QA BLOCK (pymupdf / fitz)

Run this EXACTLY after every ReportLab / WeasyPrint / any PDF build:

```python
import fitz  # pymupdf — pip install pymupdf
import os

def qa_pdf(path):
    """Full PDF QA — raises on any failure. Returns report string."""
    assert os.path.exists(path), f"FILE NOT FOUND: {path}"
    size = os.path.getsize(path)
    assert size > 10_000, f"PDF too small ({size} bytes) — likely broken"

    doc = fitz.open(path)
    page_count = len(doc)
    assert page_count > 0, "PDF has 0 pages"

    report_lines = [f"PDF QA: {path}", f"  Pages: {page_count}", f"  Size:  {size:,} bytes"]
    failures = []

    for pn in range(page_count):
        page = doc[pn]
        text = page.get_text().strip()
        char_count = len(text)
        preview = text[:60].replace('\n', ' ')
        report_lines.append(f"  Page {pn+1}: {char_count} chars — {preview!r}")

        if char_count < 80:
            failures.append(f"Page {pn+1} BLANK or near-blank ({char_count} chars)")

    # Render page 1 as PNG proof
    page = doc[0]
    pix = page.get_pixmap(matrix=fitz.Matrix(2.0, 2.0))
    proof_path = "/tmp/doc_qa_page1.png"
    pix.save(proof_path)
    report_lines.append(f"  Proof PNG: {proof_path}")

    doc.close()

    report = '\n'.join(report_lines)
    if failures:
        raise AssertionError(f"PDF QA FAILED:\n" + '\n'.join(failures) + '\n\n' + report)

    print(f"[QA PASS] PDF\n{report}")
    return report

# Usage — always call at end of every PDF script:
output = os.path.expanduser('~/Downloads/report.pdf')
qa_pdf(output)
```

---

## LAW 2 — DOCX QA BLOCK (python-docx)

```python
from docx import Document as DocxDocument
import os

def qa_docx(path):
    """Full DOCX QA — raises on any failure."""
    assert os.path.exists(path), f"FILE NOT FOUND: {path}"
    size = os.path.getsize(path)
    assert size > 5_000, f"DOCX too small ({size} bytes) — likely broken"

    doc = DocxDocument(path)
    para_count = len(doc.paragraphs)
    table_count = len(doc.tables)

    assert para_count > 0, "DOCX has 0 paragraphs — empty document"

    report_lines = [
        f"DOCX QA: {path}",
        f"  Size:       {size:,} bytes",
        f"  Paragraphs: {para_count}",
        f"  Tables:     {table_count}",
    ]

    # Check tables have content
    failures = []
    for i, tbl in enumerate(doc.tables):
        rows = len(tbl.rows)
        cols = len(tbl.columns)
        report_lines.append(f"  Table {i+1}: {rows} rows × {cols} cols")
        if rows < 2:
            failures.append(f"Table {i+1} has only {rows} rows — likely broken")

    # Check no empty sections (paragraphs with only whitespace is OK, but
    # a document with <5 non-empty paragraphs is suspicious for a report)
    non_empty = [p for p in doc.paragraphs if p.text.strip()]
    if len(non_empty) < 3:
        failures.append(f"Only {len(non_empty)} non-empty paragraphs — document may be empty")

    report = '\n'.join(report_lines)
    if failures:
        raise AssertionError(f"DOCX QA FAILED:\n" + '\n'.join(failures) + '\n\n' + report)

    print(f"[QA PASS] DOCX\n{report}")
    return report

# Usage:
output = os.path.expanduser('~/Downloads/report.docx')
qa_docx(output)
```

---

## LAW 3 — XLSX QA BLOCK (openpyxl)

```python
from openpyxl import load_workbook
import os

def qa_xlsx(path, expected_sheets=None, min_rows=2):
    """Full XLSX QA — raises on any failure."""
    assert os.path.exists(path), f"FILE NOT FOUND: {path}"
    size = os.path.getsize(path)
    assert size > 3_000, f"XLSX too small ({size} bytes) — likely broken"

    wb = load_workbook(path)
    sheet_names = wb.sheetnames

    report_lines = [
        f"XLSX QA: {path}",
        f"  Size:   {size:,} bytes",
        f"  Sheets: {sheet_names}",
    ]

    failures = []

    if expected_sheets:
        for name in expected_sheets:
            if name not in sheet_names:
                failures.append(f"Missing sheet: '{name}'")

    for name in sheet_names:
        ws = wb[name]
        rows = ws.max_row
        cols = ws.max_column
        report_lines.append(f"  Sheet '{name}': {rows} rows × {cols} cols")
        if rows < min_rows:
            failures.append(f"Sheet '{name}' has only {rows} rows — likely empty")

    report = '\n'.join(report_lines)
    if failures:
        raise AssertionError(f"XLSX QA FAILED:\n" + '\n'.join(failures) + '\n\n' + report)

    print(f"[QA PASS] XLSX\n{report}")
    return report

# Usage:
output = os.path.expanduser('~/Downloads/report.xlsx')
qa_xlsx(output, expected_sheets=['Summary', 'Raw Data'])
```

---

## LAW 4 — PPTX QA BLOCK (python-pptx)

```python
from pptx import Presentation
import os

def qa_pptx(path, expected_slides=None):
    """Full PPTX QA — raises on any failure."""
    assert os.path.exists(path), f"FILE NOT FOUND: {path}"
    size = os.path.getsize(path)
    assert size > 10_000, f"PPTX too small ({size} bytes) — likely broken"

    prs = Presentation(path)
    slide_count = len(prs.slides)

    assert slide_count > 0, "PPTX has 0 slides"

    if expected_slides:
        assert slide_count == expected_slides, \
            f"Expected {expected_slides} slides, got {slide_count}"

    report_lines = [
        f"PPTX QA: {path}",
        f"  Size:   {size:,} bytes",
        f"  Slides: {slide_count}",
    ]

    failures = []
    for i, slide in enumerate(prs.slides):
        texts = []
        for shape in slide.shapes:
            if shape.has_text_frame:
                t = shape.text_frame.text.strip()
                if t:
                    texts.append(t[:50])
        preview = ' | '.join(texts[:3]) if texts else '[NO TEXT]'
        report_lines.append(f"  Slide {i+1}: {preview}")
        if not texts:
            failures.append(f"Slide {i+1} has NO text — may be blank or broken")

    report = '\n'.join(report_lines)
    if failures:
        # Warn but don't fail — some slides are intentionally image-only
        print(f"[QA WARN] PPTX:\n" + '\n'.join(failures))

    print(f"[QA PASS] PPTX\n{report}")
    return report

# Usage:
output = os.path.expanduser('~/Downloads/presentation.pptx')
qa_pptx(output, expected_slides=10)
```

---

## LAW 5 — AUTO-DETECT FORMAT AND RUN QA

```python
import os

def qa_document(path):
    """Auto-detect format and run appropriate QA. Call this at end of EVERY build."""
    ext = os.path.splitext(path)[1].lower()
    if ext == '.pdf':
        return qa_pdf(path)
    elif ext == '.docx':
        return qa_docx(path)
    elif ext == '.xlsx':
        return qa_xlsx(path)
    elif ext == '.pptx':
        return qa_pptx(path)
    else:
        raise ValueError(f"Unsupported format: {ext}")

# One-liner at end of every script:
qa_document(os.path.expanduser('~/Downloads/report.pdf'))
```

---

## LAW 6 — FAILURE RESPONSE PROTOCOL

When QA FAILS, Claude must:
1. Print the exact failure message
2. Read the relevant section of the build script
3. Fix the root cause (not just the symptom)
4. Re-run the build script
5. Re-run QA
6. Only deliver after QA PASS

**NEVER say "it should work" without running QA. NEVER deliver without PASS.**

Common failure → fix mapping:
| QA failure | Root cause | Fix |
|---|---|---|
| Page BLANK (<80 chars) | `PageBreak()` after short section | Replace with `CondPageBreak(80*mm)` |
| PDF too small | Script crashed mid-build | Check Python traceback, fix exception |
| DOCX 0 paragraphs | Wrong file path saved | Check `doc.save(output)` path |
| XLSX 1 row | Header written, loop skipped | Check data source len() > 0 |
| PPTX 0 slides | `prs.save()` before slides added | Move save() to end of script |

---

## LAW 7 — DELIVERY CONFIRMATION FORMAT

After every QA PASS, output this summary:

```
╔══════════════════════════════════════════════════╗
║  DOCUMENT DELIVERED — QA PASSED                 ║
╠══════════════════════════════════════════════════╣
║  Format : PDF / DOCX / XLSX / PPTX             ║
║  Path   : ~/Downloads/report.pdf               ║
║  Size   : 245,312 bytes                        ║
║  Pages  : 11                                   ║
║  Status : ALL PAGES HAVE CONTENT               ║
╚══════════════════════════════════════════════════╝
```

---

## STANDARD QA IMPORT BLOCK

Paste at top of every document build script:

```python
import os, sys

def qa_document(path):
    ext = os.path.splitext(path)[1].lower()
    size = os.path.getsize(path)
    if ext == '.pdf':
        import fitz
        doc = fitz.open(path)
        assert len(doc) > 0 and size > 10000, f"PDF broken: {len(doc)} pages, {size} bytes"
        blanks = [i+1 for i in range(len(doc)) if len(doc[i].get_text().strip()) < 80]
        if blanks: raise AssertionError(f"Blank pages: {blanks}")
        doc[0].get_pixmap(matrix=fitz.Matrix(2,2)).save('/tmp/doc_qa_p1.png')
        print(f"[QA PASS] PDF {len(doc)}pp {size:,}b → /tmp/doc_qa_p1.png")
    elif ext == '.docx':
        from docx import Document as D
        d = D(path); assert len(d.paragraphs) > 2 and size > 5000
        print(f"[QA PASS] DOCX {len(d.paragraphs)}p {len(d.tables)}t {size:,}b")
    elif ext == '.xlsx':
        from openpyxl import load_workbook
        wb = load_workbook(path)
        for n in wb.sheetnames:
            assert wb[n].max_row > 1, f"Sheet '{n}' is empty"
        print(f"[QA PASS] XLSX {wb.sheetnames} {size:,}b")
    elif ext == '.pptx':
        from pptx import Presentation
        p = Presentation(path); assert len(p.slides) > 0 and size > 10000
        print(f"[QA PASS] PPTX {len(p.slides)} slides {size:,}b")

# At end of script:
# qa_document(OUTPUT)
```
