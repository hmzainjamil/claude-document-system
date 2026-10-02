# Security and local file handling

## Scope

This note describes local file effects and input handling visible in `doc-factory.py`. It is not a security audit.

## Files and outputs

The CLI accepts a path supplied by its operator and opens that document with a format library. PDF checks render the first page to the fixed path `/tmp/doc_qa_page1.png`, replacing any existing file at that path. The script prints the input path and extracted content summaries to the terminal. Use only documents you are authorized to inspect, and consider whether terminal logs or shared sessions expose filenames or document text.

No network request or credential loading is present in the Python CLI source reviewed. The Markdown guidance mentions external model names and tools, but those references do not establish an integration in this repository.

## Untrusted document content

Document parsers process user-supplied files. Keep Python libraries current and run checks with least local privilege, especially for files from untrusted sources. This tool reads documents; it does not sandbox parser libraries.

## Limits

The checker uses file-size, text-presence, page/slide, sheet, and table heuristics. It does not validate visual correctness, formulas, links, macros, factual accuracy, malware, or accessibility. A PASS is not a security or delivery approval.

## Evidence

No security test suite, dependency lock, sandbox configuration, retention policy, or supported-version policy is included in the tracked repository.