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
- **OCR for scanned PDFs, off by default** -- a switch in the top-left corner turns it
  on. Tesseract.js with the `fra+eng` models; pages are rendered at 1.7x, binarised,
  then recognised. With the switch off, scanned pages are skipped and the rest of every
  document still converts -- and Tesseract is never loaded, so nothing is fetched. See
  [Privacy](#privacy) for why the default is off.
- **Excel noise filter** (optional, on by default, Markdown output only) -- drops rows
  that are obviously administrative rather than data: legends, instructions,
  translations, version history, "confidential", "please read", "click here", and
  similar. Empty rows and fully empty columns are always dropped.
- **Wide-sheet handling** -- sheets with more than 12 columns are emitted as labelled
  lists instead of an unreadable Markdown table.

---

## Running it

**The simple way: double-click `2-TXT-v5.html`.** Every format works. OCR is off by
default; switching it on fetches the OCR engine from a CDN, so that one feature needs an
internet connection. Nothing else does, and no document content is ever sent anywhere.

**The offline way: serve the folder over HTTP.** OCR then uses the bundled engine and
models, and the tool makes no network requests at all. From the project folder:

```bash
python -m http.server 8000
```

Then open <http://localhost:8000/2-TXT-v5.html>. Any static file server works equally
well (`npx serve`, `php -S`, VS Code Live Server, etc.).

Why the difference: OCR runs in a Web Worker, and a worker on a `file://` page is not
permitted to read local files, so the bundled OCR assets are unreachable there. See
[SECURITY.md](SECURITY.md) section 4.

**Browser requirement:** the folder picker uses `webkitdirectory`, so you need a
Chromium-based browser (Chrome, Edge, Brave) or a recent Firefox. Safari support is
unreliable.

---

## Privacy

Every document is parsed inside the browser tab. **No file content is ever uploaded
anywhere.**

**Served over HTTP, the tool makes no network requests at all.** The OCR engine, its
WebAssembly core and the French and English language models are bundled here, so even
scanned PDFs are recognised offline. That configuration runs on an air-gapped machine.

**Opened directly from disk (`file://`), OCR alone needs the internet** -- which is why
it is off until you ask for it. Tesseract runs recognition in a Web Worker, and a worker
on a `file://` page gets an opaque origin that browsers forbid from reading local files,
so the bundled OCR assets are unreachable no matter what path they are given. With the
switch on, the tool loads *only* the OCR engine and language models from the jsDelivr
CDN instead.

**With the switch off, Tesseract is never loaded at all** -- no worker, no request. If a
document might be sensitive, leave OCR off and everything stays on your machine; you
still get every text-layer page and every DOCX, PPTX, XLSX, TXT and MD file, and scanned
pages are simply reported as skipped.

Either way, **no document content is ever transmitted.** The only thing fetched is the
OCR software. Everything else -- PDF text layers, DOCX, PPTX, XLSX, TXT, MD -- works
offline from `file://` with no network at all.

If the CDN is unreachable (a corporate proxy, say), scanned pages are skipped with a
message in the log and **the rest of the document is still converted**. Serve the folder
over HTTP to get offline OCR back. See [SECURITY.md](SECURITY.md) for the full picture.

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

*Updated by Claude - 11-Sep-2026, 10:26 EDT*
