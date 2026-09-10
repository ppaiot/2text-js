# 2TEXT-JS

A single-page, browser-only tool that walks a folder of documents and concatenates
everything into **one Markdown or plain-text corpus file** -- the kind of input you
feed to an LLM.

No install, no build step, no server-side processing. Open the HTML file, pick a
folder, and every document is parsed locally in your browser.

---

## What it does

1. You pick a local folder (recursively, via the browser's directory picker).
2. Every supported file is parsed **in the browser** and converted to text.
3. You get a review table -- per-file character counts, with checkboxes to include
   or exclude each document.
4. You export a single concatenated `corpus_concatene.md` or `corpus_concatene.txt`
   with a document index and per-document metadata headers.

### Supported formats

| Extension | How it is read |
|---|---|
| `.pdf` | Text layer via PDF.js; pages with no usable text layer fall back to OCR |
| `.docx` | Unzipped with JSZip, text pulled from `word/document.xml` |
| `.pptx` | Unzipped with JSZip, slide text plus speaker notes |
| `.xls` / `.xlsx` | SheetJS; rendered as Markdown tables or tab-separated text |
| `.txt` | Read as-is, line endings normalised |
| `.md` | Passed through untouched (only line endings normalised) |

Anything else in the folder is ignored. Previously generated corpus files
(`corpus_concatene.*`, `documents_concat.*`) are skipped automatically so you can
re-run in place without feeding the output back into itself.

### Extras

- **Bilingual, end to end** -- French / English, toggled with one button (French is
  the default). The toggle covers the generated corpus too: section headings, slide
  and sheet markers all follow the chosen language. The `.txt` envelope keys
  (`DATASET_TYPE`, `FILE_NAME`, ...) stay fixed, since they are there for machines
  to parse.
- **OCR for scanned PDFs** -- Tesseract.js with the `fra+eng` models. Pages are
  rendered at 1.7x, binarised, then recognised. A progress bar tracks it.
- **Excel noise filter** (optional, on by default, Markdown output only) -- drops rows
  that are obviously administrative rather than data: legends, instructions,
  translations, version history, "confidential", "please read", "click here", and
  similar. Empty rows and fully empty columns are always dropped.
- **Wide-sheet handling** -- sheets with more than 12 columns are emitted as labelled
  lists instead of an unreadable Markdown table.

---

## Running it

The tool needs to be served over HTTP. Opening `2-TXT-v5.html` directly from the
filesystem (`file://`) will break the PDF.js worker and the OCR worker, because
browsers block workers on `file://` origins.

From the project folder:

```bash
python -m http.server 8000
```

Then open <http://localhost:8000/2-TXT-v5.html>.

Any static file server works equally well (`npx serve`, `php -S`, VS Code Live
Server, etc.).

**Browser requirement:** the folder picker uses `webkitdirectory`, so you need a
Chromium-based browser (Chrome, Edge, Brave) or a recent Firefox. Safari support is
unreliable.

---

## Privacy

Every document is parsed inside the browser tab. **No file content is ever uploaded
anywhere.**

**The tool makes no network requests at all.** The OCR engine, its WebAssembly core,
and the French and English language models are all bundled in this repository, so
scanned PDFs are recognised entirely offline too. You can run the whole thing on an
air-gapped machine.

(Earlier versions fetched the OCR assets from the jsDelivr CDN on first use. That is
no longer the case -- see `initialiserOCR()` in the source.)

---

## Output format

**Markdown** -- a corpus header, an index table mapping IDs (`D001`, `D002`, ...) to
filenames, then one `# [D00n] filename` section per document with source path
metadata and the extracted content.

**Plain text** -- a machine-readable envelope instead:

```
DATASET_TYPE: MULTI_DOCUMENT_TEXT
FORMAT_VERSION: 2
DOCUMENT_COUNT: 12

===== DOCUMENT INDEX =====
1 | report.pdf
...

===== DOCUMENT START =====
FILE_INDEX: 1
FILE_NAME: report.pdf
FILE_PATH: input/report.pdf
CONTENT_START
...
CONTENT_END
===== DOCUMENT END =====
```

Documents are ordered largest-first by character count.

---

## Project layout

```
2-TXT-v5.html          the entire application -- markup, styles, and logic
pdf.min.js             PDF.js 3.11.174
pdf.worker.min.js      PDF.js worker
jszip.min.js           JSZip 3.10.1 (docx/pptx unzipping)
xlsx.full.min.js       SheetJS 0.20.3
tesseract.min.js       Tesseract.js 5.1.1
tesseract.worker.min.js          OCR worker
tesseract-core-simd-lstm.wasm.js OCR engine, SIMD build (used when supported)
tesseract-core-lstm.wasm.js      OCR engine, non-SIMD fallback
fra.traineddata.gz               French OCR model
eng.traineddata.gz               English OCR model
```

All dependencies are vendored so the tool keeps working without a package manager.

---

## Third-party licences

| Library | Licence |
|---|---|
| PDF.js | Apache-2.0 |
| JSZip | MIT / GPLv3 dual |
| SheetJS (`xlsx`) | Apache-2.0 |
| Tesseract.js | Apache-2.0 |

Each retains its own licence; the bundled minified files carry their original
headers. Full detail in [THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md).

---

## Security

Documents are parsed entirely in the browser and never leave the machine. Vendored
dependencies are pinned and hash-verified (`sha256sum -c vendor-manifest.sha256`).

Known issues, the reasoning behind them, and the action plan are documented in
[SECURITY.md](SECURITY.md) -- read it before deploying this anywhere sensitive.

---

## Licence

MIT -- see [LICENSE](LICENSE).

---

*Updated by Claude - 10-Sep-2026, 19:48 EDT*
