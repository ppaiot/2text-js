# Security posture and known issues

**Assessment date:** 10 September 2026 · **Applies to:** `2-TXT-v5.html` at this commit

This document records what was reviewed, what was found, what has been fixed, and what
remains open with a rationale. It is written to be readable by a security reviewer who
has not seen the code.

---

## 1. What the tool does with your data

2TEXT-JS reads documents from a local folder, parses them **entirely inside the browser
tab**, and writes one concatenated file back to disk via a normal download. There is no
server, no account, and no upload.

| Property | Status | How it was checked |
|---|---|---|
| Outbound network calls carrying document data | **None** | no `fetch` / `XHR` / `sendBeacon` / `WebSocket` anywhere in the app |
| Other outbound requests | OCR engine + language models from jsDelivr, **only when opened from `file://`** | see section 4 |
| Server-side component | None | The tool is a single HTML file |
| Telemetry or analytics | None | as above |
| Credentials or secrets handled | None | no auth of any kind |
| Persistent storage | Tesseract caches **OCR language models** in IndexedDB | document content is never written to storage |

Document content never leaves the machine in either configuration. Served over HTTP the
tool makes no network requests at all and runs on an air-gapped host; opened from
`file://` it downloads the OCR *software* (never any document) because the bundled
copies are unreachable from a null-origin worker.

This matters for a risk comparison: the realistic alternative to a tool like this is
staff pasting confidential documents into a free online converter. That is strictly
worse.

---

## 2. Supply chain

All 10 vendored third-party files were verified **byte-identical to their upstream
releases** by SHA-256 comparison against the official CDN copies.

| File | Upstream source |
|---|---|
| `pdf.min.js`, `pdf.worker.min.js` | `cdn.jsdelivr.net/npm/pdfjs-dist@3.11.174/build/` |
| `jszip.min.js` | `cdn.jsdelivr.net/npm/jszip@3.10.1/dist/` |
| `xlsx.full.min.js` | `cdn.sheetjs.com/xlsx-0.20.3/package/dist/` |
| `tesseract.min.js` | `cdn.jsdelivr.net/npm/tesseract.js@5.1.1/dist/` |
| `tesseract.worker.min.js` | `cdn.jsdelivr.net/npm/tesseract.js@5.1.1/dist/worker.min.js` (renamed) |
| `tesseract-core-simd-lstm.wasm.js`, `tesseract-core-lstm.wasm.js` | `cdn.jsdelivr.net/npm/tesseract.js-core@5.1.1/` |
| `fra.traineddata.gz`, `eng.traineddata.gz` | `cdn.jsdelivr.net/npm/@tesseract.js-data/<lang>/4.0.0_best_int/` |

Hashes are recorded in [`vendor-manifest.sha256`](vendor-manifest.sha256). Re-verify at
any time with:

```bash
sha256sum -c vendor-manifest.sha256
```

Any future change to a vendored file must update that manifest in the same commit.

---

## 3. Findings

### F-1 · PDF.js arbitrary JavaScript execution (CVE-2024-4367) · High · **Mitigated**

PDF.js compiles glyph-rendering functions with `Function()` for speed, interpolating a
`fontMatrix` array it assumes is numeric. A crafted PDF can put a string there and break
out of the generated function body, achieving arbitrary JavaScript execution in the page
context simply by being opened.

**Why it matters here:** a document converter is precisely a tool pointed at files from
outside the organisation. Code running in the page can read every document loaded in
that session and exfiltrate it -- CORS prevents *reading* cross-origin responses, it
does not prevent *sending* data out.

**Fixed upstream in PDF.js 4.2.67.** This project ships 3.11.174.

**Mitigation applied:** `getDocument()` is now called with `isEvalSupported: false`,
which disables the vulnerable code path entirely (confirmed by the Codean Labs
write-up that published the vulnerability). Text extraction and OCR were re-tested
after the change and are unaffected.

**Why not simply upgrade — the constraint that blocks it:** PDF.js 4.x and later are
distributed **only as ES modules** (`.mjs`); there is no UMD build, and the `legacy`
build is also ESM. ES module scripts are always fetched with CORS, which `file://`
cannot satisfy, so a `<script type="module">` build fails outright when the HTML is
opened directly from disk. The primary deployment target is a locked-down workstation
where the file is double-clicked and no web server can be installed. **Upgrading PDF.js
would fix this finding and break the tool's only viable deployment.**

*Residual risk:* the mitigation disables the known exploit path, but 3.11.174 remains
an unmaintained branch and will not receive future fixes. Revisit if a UMD-compatible
build, a bundling step, or an HTTP-served deployment becomes acceptable.

### F-2 · SheetJS prototype pollution and ReDoS (CVE-2023-30533, CVE-2024-22363) · Medium · **Fixed**

`xlsx` 0.18.5 was vulnerable to prototype pollution via a crafted spreadsheet
(CVE-2023-30533, fixed in 0.19.3) and to regular-expression denial of service
(CVE-2024-22363, fixed in 0.20.2).

**Fixed by upgrading to 0.20.3.** Note that fixed SheetJS releases are published on
`cdn.sheetjs.com` only -- the `xlsx` package on npm is no longer maintained and still
carries the vulnerable versions. Upgrade verified: Markdown and text rendering, the
noise filter, and the formatted-value (`cell.w`) path all produce byte-identical output
to 0.18.5.

### F-3 · No patch mechanism for vendored code · Low · **Partially addressed**

Dependencies are vendored deliberately, so nothing updates itself and a new CVE will
not surface on its own. `vendor-manifest.sha256` now makes it possible to tell exactly
what is shipped; it does not make anything update. Dependency versions should be
reviewed on a schedule.

### F-4 · Output aggregation · Informational · **Accepted**

The generated corpus concatenates every selected document into one file. That file is a
higher-value target than the scattered originals and is easier to over-share by
accident. Treat the output at the classification of the most sensitive document that
went into it.

---

## 4. Deployment risk: `file://` versus served

OCR needs a Web Worker, and a worker on a `file://` page has an opaque (`null`) origin,
which is not permitted to load local files. Bundled OCR assets are therefore unreachable
when the HTML is opened directly. Two workarounds are under consideration; **their risk
profiles are not equivalent.**

| Option | What it does | Blast radius |
|---|---|---|
| **A · CDN fallback on `file://`** | loads the OCR engine and models from jsDelivr at runtime | **this tool, this session** |
| **B · `--allow-file-access-from-files`** | Chrome flag lifting file-origin isolation | **every local file the user can read, browser-wide, for the whole session** |

**Option A** replaces "no third-party runtime trust" with "trusting jsDelivr at
runtime". A compromised CDN or package would get access to the documents being
processed. It **cannot be protected with Subresource Integrity**: `importScripts()`
accepts no integrity attribute. It also discloses usage metadata (IP, timing) to the
CDN. No document content is transmitted.

**Option B** collapses `file://` origin isolation for the entire browser session. Any
other local HTML opened in that browser -- a saved page, an email attachment -- gains
the ability to read any file the user can read and send it anywhere.

**These two findings compound.** Option B together with an unpatched F-1 turns
"a malicious PDF reads the documents you fed it" into "a malicious PDF reads arbitrary
files on disk and exfiltrates them". The F-1 mitigation above is therefore a
**prerequisite**, not an optional extra, if Option B is ever adopted.

Recommendation: prefer Option A unless the corporate network blocks the CDN. If Option
B is unavoidable, confine it to a dedicated browser profile and shortcut used only for
this tool.

**Option A is now implemented** (`cheminsOCR()`): `file://` falls back to the CDN,
HTTP keeps using the bundled assets. Measured in all three states -- `file://` with the
CDN reachable (OCR works), `file://` with `cdn.jsdelivr.net` blackholed (scanned pages
skipped, **the rest of the document still converts**, actionable message logged), and
HTTP with the CDN blackholed (OCR works entirely from local assets). Option B remains
available for offline OCR from `file://` and is **not** required for the tool to be
useful there.

---

## 5. Action plan

| # | Action | Status |
|---|---|---|
| 1 | Set `isEvalSupported: false` on `getDocument()` | **Done** |
| 2 | Upgrade SheetJS to a fixed release (0.20.3) | **Done** |
| 3 | Record SHA-256 manifest of vendored files | **Done** |
| 4 | Upgrade PDF.js to >= 4.2.67 | **Blocked** -- ESM-only, incompatible with `file://` (see F-1) |
| 5 | Deployment Option A (CDN fallback on `file://`) | **Done** -- Option B no longer required for OCR to work from the filesystem |
| 6 | Isolate OCR failures per page so one bad page cannot discard a whole document | **Done** |
| 7 | Review dependency versions on a schedule | Open |

---

## 6. Reporting a problem

This is a personal project distributed as-is. Open an issue at
<https://github.com/ppaiot/2text-js/issues>. Do not include confidential documents in a
bug report -- a description of the file type and structure is enough.

---

## References

- CVE-2024-4367 analysis: <https://codeanlabs.com/2024/05/cve-2024-4367-arbitrary-js-execution-in-pdf-js/>
- CVE-2024-4367 advisory: <https://advisories.gitlab.com/npm/pdfjs-dist/CVE-2024-4367/>
- CVE-2023-30533 advisory: <https://cdn.sheetjs.com/advisories/CVE-2023-30533>
- CVE-2024-22363 advisory: <https://cdn.sheetjs.com/advisories/CVE-2024-22363>

---

*Updated by Claude - 11-Sep-2026, 09:12 EDT*
