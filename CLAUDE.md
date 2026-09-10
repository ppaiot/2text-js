# CLAUDE.md -- 2TEXT-JS

Context file for AI assistants working on this repo. Read this instead of
re-parsing the whole codebase.

---

## What this is

A **single-file, zero-build, browser-only** document-to-corpus converter. The user
points it at a local folder; it extracts text from PDF / DOCX / PPTX / XLS(X) /
TXT / MD entirely client-side and downloads one concatenated Markdown or plain-text
file suitable for feeding to an LLM.

There is **no build system, no package.json, no tests, no bundler, no framework**.
The whole application is `2-TXT-v5.html`: markup, CSS, and vanilla ES2017+ JavaScript
in one file, with four minified libraries vendored next to it as plain `<script>` tags.

Do not introduce a build step, npm, or a framework without asking. The single-file
distribution model is the point of the tool -- anyone can copy the folder and run it.

---

## Architecture

`2-TXT-v5.html` -- 1089 lines, three parts:

- **lines 1-208** -- `<head>` (script tags + all CSS) and `<body>` markup.
- **lines 210-1086** -- the entire application script.
- Everything is at top level in the global scope. No modules, no classes.

### Global state (lines 212-229)

| Variable | Purpose |
|---|---|
| `langue` | `"fr"` or `"en"`; drives the `T` translation lookup |
| `fichiersSelectionnes` | Raw `File[]` from the directory input |
| `resultatsConversion` | `{nom, chemin, texte, taille}[]` after extraction, sorted largest-first |
| `ocrWorker`, `ocrInitialise` | Lazily created Tesseract worker; terminated after each analysis run |

`resultatsConversion` and the DOM table rows are **index-aligned** -- row `i` maps to
`resultatsConversion[i]`. Both `mettreAJourResume()` and the generate handler rely on
this. Do not sort or filter one without the other.

### Code map

| Lines | Section | Key symbols |
|---|---|---|
| 233-311 | i18n | `T` (fr/en string table), `t()`, `appliquerLangue()` |
| 316-374 | Utilities | `log()`, `extensionDe()`, `nomSimple()`, `nettoyerTexte()`, `echapperMarkdown()`, `estFichierGenere()`, `mettreAJourResume()` |
| 377-456 | PDF + OCR | `initialiserOCR()`, `augmenterContraste()`, `extraireTextePDF()` |
| 459-495 | DOCX | `extraireTexteDOCX()` |
| 502-609 | PPTX | `extraireTextePPTX()` |
| 623-800 | Excel | `motifsBruitExcel`, `estLigneBruitExcel()`, `valeurCelluleExcel()`, `extraireFeuilleExcel()`, `feuilleVersMarkdown()`, `extraireExcelMarkdown()`, `extraireExcelTexte()` |
| 808-865 | Dispatch | `extraireFichier(fichier, format)` -- extension switch |
| 869-901 | Folder selection | `boutonChoisir.onclick`, `inputRepertoire` change handler |
| 904-989 | Analysis run | `boutonAnalyser.onclick` -- the main loop |
| 992-1084 | Export | `boutonGenerer.onclick` -- builds and downloads the corpus |

### Data flow

```
folder picker -> fichiersSelectionnes (File[])
   -> filter by extension + estFichierGenere()
   -> extraireFichier() per file  ->  resultatsConversion[]
   -> sort by taille desc  ->  render table with checkboxes
   -> boutonGenerer: collect checked rows -> build string -> Blob -> <a download>
```

---

## Conventions

- **The codebase is written in French.** Identifiers, comments, and log messages are
  all French (`fichier`, `texte`, `lignes`, `nettoyer`, `extraire...`). Keep new code
  in the same language -- mixing French and English identifiers would make it worse.
- **User-facing strings go in `T` (line 233), never inline.** Both `fr` and `en` keys
  must be added together, then wired through `appliquerLangue()` if the string lands
  in static markup.
- Formatting style: no spaces around `=` or operators, 2-space indent, dense.
  Match it rather than reformatting.
- French prose (comments, UI strings) uses **proper accents**. Do not strip them.
- `log()` is the only user-visible diagnostic channel -- it appends to the `#journal`
  console pane. There is no error UI beyond it.

---

## Per-format extraction notes

**PDF** (`extraireTextePDF`, line 411)
Per page: pull the text layer; if it yields **fewer than 20 characters**, treat the
page as scanned and OCR it. Text items are joined with `" "`, so intra-page line
breaks are lost. OCR renders at scale `1.7` and binarises at grey threshold `165`
(`augmenterContraste`). Both constants are hardcoded magic numbers.

**DOCX** (`extraireTexteDOCX`, line 459)
Reads only `word/document.xml`. Walks the DOM for `w:t` / `w:tab` / `w:br` / `w:cr`
and appends `\n` after each `w:p`. Consequence: **tables are flattened** -- cell
boundaries vanish. Headers, footers, footnotes, endnotes, and comments are not read.

**PPTX** (`extraireTextePPTX`, line 502)
Collects all `a:t` nodes per `ppt/slides/slideN.xml`, prefixed with
`--- DIAPOSITIVE N ---`, then the matching `ppt/notesSlides/notesSlideN.xml`. The
slide-to-notes mapping assumes index parity (`slideN` <-> `notesSlideN`), which holds
for simple decks but is **not guaranteed** by OOXML -- the correct way is to follow
the relationship files in `ppt/slides/_rels/`. Falls back to dumping all notes if no
slide text was found.

**Excel** (line 623 onwards)
`extraireFeuilleExcel()` reads the declared `!ref` range cell by cell, preferring
`cell.w` (formatted display text) over `cell.v` (raw value). Drops empty rows, then
optionally noise rows, then fully empty columns.
`feuilleVersMarkdown()` switches to a labelled-list rendering when a sheet has
**more than 12 columns**. Row 1 is always assumed to be the header row.
Formulas are not evaluated -- SheetJS returns cached values only. Merged cells are
not expanded. Charts, images, and comments are ignored.

---

## Known issues / gotchas

1. **The vendored Tesseract runtime is not actually used.** `tesseract.worker.min.js`,
   `tesseract-core.wasm`, and `tesseract-core.wasm.js` sit in the folder, but
   `initialiserOCR()` (line 377) never passes `workerPath` / `corePath` / `langPath`,
   so Tesseract.js 5.1.1 falls back to its jsDelivr CDN defaults. OCR therefore needs
   an internet connection. To make it offline, pass all three paths plus the
   `fra.traineddata.gz` / `eng.traineddata.gz` files (which are **not** in the repo).

2. **Output format is captured at analysis time, not export time.** Excel files are
   the only format whose extraction depends on `formatSortie` (Markdown tables vs.
   tab-separated). `boutonAnalyser.onclick` snapshots `formatSortie.value` at line 918.
   If the user analyses as Markdown, then switches the dropdown to text and exports,
   the Excel sections stay in Markdown. Re-analysis is required and nothing tells the
   user that.

3. **`file://` does not work.** PDF.js sets `workerSrc` to a relative path (line 211)
   and Tesseract spawns a worker; both are blocked on `file://` origins. Must be
   served over HTTP.

4. **Everything runs on the main thread, serially.** A large folder freezes the UI.
   There is no cancel button and no way to resume a partial run.

5. **The whole corpus is held in memory** as one JavaScript string before the Blob is
   created. Very large folders can exhaust the tab.

6. **No file-level picking.** `webkitdirectory` takes a whole folder; per-file
   selection only happens after extraction, via the results table.

7. **The `estFichierGenere()` guard** (line 345) is name-based only and matches any
   filename containing `documents_concat` or `corpus_concat`.

8. **The version number lives only in the filename** (`2-TXT-v5.html`). There is no
   version constant in the code and no changelog.

9. **No `background` on `body`, but `color:#222` is set.** In a browser or embedded
   view rendering with a dark ground, the page inherits the dark background and the
   dark text becomes unreadable. Fix by setting an explicit `background:#fff` on
   `body`, or by adding a real dark-theme palette.

10. **The Excel noise filter is Markdown-only.** `extraireFichier()` (line 808) only
    passes `nettoyageExcel.checked` on the Markdown branch; the text branch calls
    `extraireExcelTexte()`, which hardcodes `nettoyer=false`. The checkbox silently
    does nothing when the output format is text.

---

## Likely next steps

Ideas that fit the tool's shape, roughly in order of value:

- Wire up the local Tesseract worker/core/lang paths so OCR works offline (issue 1).
- Re-extract Excel files on format change, or warn the user (issue 2).
- Add a version constant and surface it in the UI and the generated corpus header.
- Per-document token estimate alongside the character count.
- Drag-and-drop as an alternative to the folder picker.
- Extra formats: `.doc`, `.rtf`, `.csv`, `.html`, `.epub`.
- Preserve DOCX table structure as Markdown tables.
- Follow `ppt/slides/_rels/` for correct slide-to-notes mapping.
- A cancel button and per-file progress rather than a single OCR bar.
- Configurable OCR languages instead of the hardcoded `fra+eng`.

---

## Working agreements

- Keep it a single HTML file with vendored libraries. No build step.
- No telemetry, no network calls beyond the existing Tesseract CDN fetch. This tool's
  selling point is that documents never leave the browser.
- **This is a public repository.** Never commit personal data, real document samples,
  local absolute paths, credentials, or client names. Test fixtures must be synthetic.
- Verify changes by serving the folder over HTTP and exercising the real UI; there is
  no test suite to lean on.

---

*Updated by Claude - 10-Sep-2026, 09:05 EDT*
