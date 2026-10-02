# Claude Document System

A pair of document QA skills and a Python CLI for checking PDF, DOCX, XLSX, and PPTX files. The repository contains one QA script and four Markdown skill documents. It does not contain the format-master skills named in the orchestrator document.

| Status | Evidence |
|---|---|
| Source reviewed | 2026-10-02; all six tracked files |
| CLI | `doc-factory.py --qa FILE` |
| Tests | No test suite or sample fixtures included |
| Dependencies | Imported on demand; no pinned dependency manifest |
| License | No license file or declared license identified |

## What it does

The CLI selects a format-specific checker by file extension:

- PDF: opens with PyMuPDF, checks non-empty pages, warns on pages with little extractable text, and renders page one to `/tmp/doc_qa_page1.png`.
- DOCX: opens with `python-docx` and checks file size and extracted text/table presence.
- XLSX: opens with `openpyxl`, checks workbook/sheet content, and can compare expected sheet names.
- PPTX: opens with `python-pptx`, checks slide count/content, and can compare an expected slide count.

These checks are structural heuristics. A PASS does not verify visual layout, factual correctness, accessibility, formulas, or suitability for delivery. In particular, image-only content may have no extractable text.

The Markdown files describe a preflight workflow, document orchestration, QA guidance, and ReportLab design rules. They are guidance documents, not an installed agent system. Some refer to model names, skills, scripts, and output policies outside this repository; those dependencies are not bundled or verified here.

## Use

Requirements depend on the file being checked:

- PDF: `pymupdf`
- DOCX: `python-docx`
- XLSX: `openpyxl`
- PPTX: `python-pptx`

Install only the package needed for your format in an isolated Python environment. The repository has no dependency lock or installer.

```sh
python3 doc-factory.py --help
python3 doc-factory.py --qa /path/to/document.pdf
python3 doc-factory.py --qa /path/to/workbook.xlsx --sheets Summary Data
python3 doc-factory.py --qa /path/to/slides.pptx --slides 10
```

The CLI accepts `--qa FILE`; `--sheets` applies to XLSX and `--slides` to PPTX. Commands are documented by source but were not executed in this review. PDF QA writes a preview image to `/tmp/doc_qa_page1.png`; each run overwrites that path.

## Repository map

| Path | Purpose |
|---|---|
| `doc-factory.py` | Format-dispatching structural QA CLI |
| `doc-preflight-SKILL.md` | Pre-build checklist guidance |
| `document-orchestrator-SKILL.md` | Document routing and workflow guidance |
| `document-qa-agent-SKILL.md` | QA checklist guidance |
| `reportlab-pdf-master-SKILL.md` | ReportLab-specific guidance |

## Limitations and documentation gaps

- Thresholds such as minimum file sizes and text counts are heuristics in source; they are not validated quality guarantees.
- No test fixtures, dependency manifest, release notes, support policy, or license are included.
- External format-master skills referenced by the guidance are not present in this repository.
- No agent runtime, model integration, document generation pipeline, or automatic delegation is implemented in the tracked files.

## Documentation

- [Security and local file handling](SECURITY.md)

## Maintainer

[hmzainjamil](https://github.com/hmzainjamil)
