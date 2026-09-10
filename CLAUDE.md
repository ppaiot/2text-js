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

`2-TXT-v5.html` -- 1172 lines, three parts:

- **lines 1-208** -- `<head>` (script tags + all CSS) and `<body>` markup.
- **lines 210-1169** -- the entire application script.
- Everything is at top level in the global scope. No modules, no classes.

### Global state (lines 212-229)

| Variable | Purpose |
|---|---|
| `langue` | `"fr"` or `"en"`; drives the `T` translation lookup |
| `fichiersSelectionnes` | Raw `File[]` from the directory input |
| `resultatsConversion` | `{nom, chemin, texte, feuilles?}[]` after extraction, sorted largest-first |
| `ocrWorker`, `ocrInitialise` | Lazily created Tesseract worker; terminated after each analysis run |

`resultatsConversion` and the DOM table rows are **index-aligned** -- row `i` maps to
`resultatsConversion[i]`. `mettreAJourResume()`, `rafraichirTailles()` and the generate
handler all rely on this. Do not sort or filter one without the other.

**Entries do not all carry finished text.** A spreadsheet entry has `feuilles` (its
parsed rows) and an empty `texte`; everything else has `texte` and no `feuilles`.
Never read `.texte` directly -- go through `texteDe(entry)` (line 370), which renders
spreadsheets on demand from the current format and cleanup settings and memoises the
result on the entry itself (`cacheCle` / `cacheTexte` / `cacheRetirees`). `tailleDe()`
is the same thing for character counts. This is what keeps the export honest when the
user changes a control after analysing -- see known issue 2.

### Code map

| Lines | Section | Key symbols |
|---|---|---|
| 233-311 | i18n | `T` (fr/en string table), `t()`, `appliquerLangue()` |
| 322-367 | Utilities | `log()`, `extensionDe()`, `nomSimple()`, `nettoyerTexte()`, `echapperMarkdown()`, `estFichierGenere()` |
| 370-428 | Rendering state | `texteDe()`, `tailleDe()`, `mettreAJourResume()`, `rafraichirTailles()` |
| 431-527 | PDF + OCR | `initialiserOCR()`, `augmenterContraste()`, `extraireTextePDF()` |
| 530-566 | DOCX | `extraireTexteDOCX()` |
| 573-680 | PPTX | `extraireTextePPTX()` |
| 694-885 | Excel | `motifsBruitExcel`, `estLigneBruitExcel()`, `valeurCelluleExcel()`, `lignesBrutesFeuille()`, `preparerLignes()`, `feuilleVersMarkdown()`, `analyserClasseur()`, `classeurVersTexte()` |
| 888-933 | Dispatch | `extraireFichier(fichier)` -- extension switch |
| 936-968 | Folder selection | `boutonChoisir.onclick`, `inputRepertoire` change handler |
| 971-1056 | Analysis run | `boutonAnalyser.onclick` -- the main loop |
| 1059-1167 | Export | `boutonGenerer.onclick` -- builds and downloads the corpus |

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

**PDF** (`extraireTextePDF`, line 482)
Per page: pull the text layer; if it yields **fewer than 20 characters**, treat the
page as scanned and OCR it. Text items are joined with `" "`, so intra-page line
breaks are lost. OCR renders at scale `1.7` and binarises at grey threshold `165`
(`augmenterContraste`). Both constants are hardcoded magic numbers.

**DOCX** (`extraireTexteDOCX`, line 530)
Reads only `word/document.xml`. Walks the DOM for `w:t` / `w:tab` / `w:br` / `w:cr`
and appends `\n` after each `w:p`. Consequence: **tables are flattened** -- cell
boundaries vanish. Headers, footers, footnotes, endnotes, and comments are not read.

**PPTX** (`extraireTextePPTX`, line 573)
Collects all `a:t` nodes per `ppt/slides/slideN.xml`, prefixed with
`--- DIAPOSITIVE N ---`, then the matching `ppt/notesSlides/notesSlideN.xml`. The
slide-to-notes mapping assumes index parity (`slideN` <-> `notesSlideN`), which holds
for simple decks but is **not guaranteed** by OOXML -- the correct way is to follow
the relationship files in `ppt/slides/_rels/`. Falls back to dumping all notes if no
slide text was found.

**Excel** (line 694 onwards) -- **parsed once, rendered on demand.**

*Parse phase*, during analysis: `analyserClasseur()` reads the workbook and returns
`[{nom, lignesBrutes}]` per sheet. `lignesBrutesFeuille()` walks the declared `!ref`
range cell by cell, preferring `cell.w` (formatted display text) over `cell.v` (raw
value), and drops fully empty rows. Only format-independent work happens here.

*Render phase*, at display and export: `classeurVersTexte(feuilles, format, nettoyer)`
produces Markdown or tab-separated text synchronously. `preparerLignes()` applies the
noise filter and then prunes empty columns -- **in that order, and at render time**,
because dropping noise rows can empty a column that was previously populated. Caching
the pruned columns from a previous render would be wrong.

`feuilleVersMarkdown()` switches to a labelled-list rendering when a sheet has **more
than 12 columns**. Row 1 is always assumed to be the header row. The noise filter
applies to Markdown output only (`nettoyageActif` in `classeurVersTexte`), matching
the checkbox label.

Formulas are not evaluated -- SheetJS returns cached values only. Merged cells are not
expanded. Charts, images, and comments are ignored.

---

## Known issues / gotchas

1. **OCR is fully offline, and that constrains the OEM setting.** `initialiserOCR()`
   (line 431) passes `workerPath`, `corePath:"."` and `langPath:"."`, so every OCR
   asset is served from the project folder and nothing touches the jsDelivr CDN.
   `corePath` is a **directory**, so Tesseract picks the build itself: with OEM `1`
   (`LSTM_ONLY`, passed to `createWorker`) it resolves `lstmOnly=true` and requests
   `tesseract-core-simd-lstm.wasm.js`, falling back to `tesseract-core-lstm.wasm.js`
   without SIMD. **Only those two builds are vendored.** Changing the OEM argument to
   `0` or `2` would make it request `tesseract-core-simd.wasm.js` /
   `tesseract-core.wasm.js`, which are not in the repo -- OCR would 404 and hang. If
   you ever need the legacy engine, vendor those two files as well.

2. **Spreadsheet rendering is deferred on purpose -- do not "optimise" it away.**
   Excel is the only format whose output depends on `formatSortie` and
   `nettoyageExcel`. Analysis stores parsed rows, not rendered text, and
   `texteDe()` renders at display and export time from the *current* control values.
   Storing a rendered string during analysis (the previous behaviour) silently
   produced Markdown tables inside `.txt` corpora whenever the user changed the
   dropdown after analysing. The memo key is `format|nettoyer`; if you add another
   control that affects Excel output, it must go into that key too.

3. **`file://` does not work.** PDF.js sets `workerSrc` to a relative path (line 211)
   and Tesseract spawns a worker; both are blocked on `file://` origins. Must be
   served over HTTP.

4. **Everything runs on the main thread, serially.** A large folder freezes the UI.
   There is no cancel button and no way to resume a partial run.

5. **The whole corpus is held in memory** as one JavaScript string before the Blob is
   created. Very large folders can exhaust the tab.

6. **No file-level picking.** `webkitdirectory` takes a whole folder; per-file
   selection only happens after extraction, via the results table.

7. **The `estFichierGenere()` guard** (line 351) is name-based only and matches any
   filename containing `documents_concat` or `corpus_concat`.

8. **The version number lives only in the filename** (`2-TXT-v5.html`). There is no
   version constant in the code and no changelog.

9. **The page is deliberately light-only.** `:root` sets `color-scheme:light` and an
   explicit white background, and `body` repeats the background. This is load-bearing:
   the rest of the stylesheet assumes a light ground (`#f7f7f7` zones, white cells,
   `#ddd` borders), so removing either declaration makes the page unreadable against a
   dark host. There is no dark theme -- adding one means moving the palette to CSS
   custom properties and redefining them under `prefers-color-scheme`, which is a
   design decision, not a bug fix.

10. **The Excel noise filter is Markdown-only, by design.** `classeurVersTexte()`
    computes `nettoyageActif=(format==="md") && nettoyer`, matching the checkbox
    label ("... lors de la génération Markdown"). Toggling the checkbox now takes
    effect immediately in Markdown mode -- previously it was read once during analysis
    and silently ignored afterwards. In text mode it is still intentionally inert.

---

## Likely next steps

Ideas that fit the tool's shape, roughly in order of value:

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
- **Automated/headless verification gotcha:** PDF.js schedules its rasterisation loop
  with `requestAnimationFrame`, so `page.render()` **never resolves** in a tab the
  browser considers hidden (`document.hidden === true`) -- it hangs silently with no
  error. This makes OCR look broken when it is fine. To exercise the render path from
  an automation context, pass `intent:"print"` in the render params, which switches
  PDF.js to promise-based scheduling. Real users with a visible window are unaffected.

---

*Updated by Claude - 10-Sep-2026, 10:02 EDT*
