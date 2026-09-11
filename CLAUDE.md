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

`2-TXT-v5.html` -- 1692 lines, three parts:

- **lines 1-266** -- `<head>` (script tags + all CSS) and `<body>` markup.
- **lines 268-1689** -- the entire application script.
- Everything is at top level in the global scope. No modules, no classes.

### Global state (lines 270-290)

| Variable | Purpose |
|---|---|
| `langue` | `"fr"` or `"en"`; drives the `T` translation lookup |
| `entreesSource` | `{fichier, chemin, supporte, retenu, caseCocher}[]` -- one per file in the picked folder |
| `resultatsConversion` | `{nom, chemin, texte, feuilles?}[]` after extraction, sorted largest-first |
| `ocrWorker`, `ocrInitialise` | Lazily created Tesseract worker; terminated after each analysis run |
| `ocrIndisponible` | Latches when OCR init fails, so a 40-page scan does not retry 40 times; reset per run |
| `ocrAutorise` | User-controlled OCR switch, **default `false`**; gates every call into Tesseract |

`resultatsConversion` and the DOM table rows are **index-aligned** -- row `i` maps to
`resultatsConversion[i]`. `mettreAJourResume()`, `rafraichirTailles()` and the generate
handler all rely on this. Do not sort or filter one without the other.

**Entries do not all carry finished text.** A spreadsheet entry has `feuilles` (its
parsed rows), a presentation entry has `presentation` (its parsed slides), and both
leave `texte` empty; everything else carries `texte` alone.
Never read `.texte` directly -- go through `texteDe(entry)` (line 667), which renders
spreadsheets on demand from the current format and cleanup settings and memoises the
result on the entry itself (`cacheCle` / `cacheTexte` / `cacheRetirees`). `tailleDe()`
is the same thing for character counts. This is what keeps the export honest when the
user changes a control after analysing -- see known issue 2.

### Code map

| Lines | Section | Key symbols |
|---|---|---|
| 374-616 | i18n + OCR switch | `T` (fr/en), `t()`, `tn()`, `appliquerLangue()`, `majBoutonOCR()`, `boutonOCR.onclick` |
| 619-664 | Utilities | `log()`, `extensionDe()`, `nomSimple()`, `nettoyerTexte()`, `echapperMarkdown()`, `estFichierGenere()` |
| 667-733 | Rendering state | `texteDe()`, `tailleDe()`, `mettreAJourResume()`, `rafraichirTailles()` |
| 736-824 | Source selection | `entreesRetenues()`, `majCompteSelection()`, `rendreListeSource()`, `basculerSelection.onclick` |
| 827-999 | PDF + OCR | `cheminsOCR()`, `initialiserOCR()`, `augmenterContraste()`, `extraireTextePDF()` |
| 1002-1042 | DOCX | `extraireTexteDOCX()` |
| 1045-1200 | PPTX | `extraireStructurePPTX()`, `structurePPTXVersTexte()` |
| 1203-1394 | Excel | `motifsBruitExcel`, `estLigneBruitExcel()`, `valeurCelluleExcel()`, `lignesBrutesFeuille()`, `preparerLignes()`, `feuilleVersMarkdown()`, `analyserClasseur()`, `classeurVersTexte()` |
| 1397-1446 | Dispatch | `extraireFichier(fichier)` -- extension switch |
| 1449-1487 | Folder selection | `boutonChoisir.onclick`, `inputRepertoire` change handler |
| 1490-1576 | Analysis run | `boutonAnalyser.onclick` -- the main loop |
| 1579-1687 | Export | `boutonGenerer.onclick` -- builds and downloads the corpus |

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
- **Every user-visible string goes in `T` (line 374), never inline** -- and that
  includes strings written *into the generated corpus*, not just the UI. Both `fr` and
  `en` keys must be added together, then wired through `appliquerLangue()` if the
  string lands in static markup.
- Numbered strings use a `{n}` placeholder and `tn(cle, valeur)` rather than
  concatenation, so word order can differ per language.
- **Never hardcode `" : "`.** French puts a space before a colon and English does not;
  use `t("deuxPoints")` between a label and its value.
- The `.txt` envelope keys (`DATASET_TYPE`, `FILE_NAME`, `CONTENT_START`, ...) are
  **not** translated. They are format tokens for machine parsing, like HTTP headers --
  translating them would break any consumer. Only the Markdown corpus, which is prose,
  follows the language.
- Formatting style: no spaces around `=` or operators, 2-space indent, dense.
  Match it rather than reformatting.
- French prose (comments, UI strings) uses **proper accents**. Do not strip them.
- `log()` is the only user-visible diagnostic channel -- it appends to the `#journal`
  console pane. There is no error UI beyond it.

---

## Per-format extraction notes

**PDF** (`extraireTextePDF`, line 904)
Per page: pull the text layer; if it yields **fewer than 20 characters**, treat the
page as scanned and OCR it. Text items are joined with `" "`, so intra-page line
breaks are lost. OCR renders at scale `1.7` and binarises at grey threshold `165`
(`augmenterContraste`). Both constants are hardcoded magic numbers.

**DOCX** (`extraireTexteDOCX`, line 1002)
Reads only `word/document.xml`. Walks the DOM for `w:t` / `w:tab` / `w:br` / `w:cr`
and appends `\n` after each `w:p`. Consequence: **tables are flattened** -- cell
boundaries vanish. Headers, footers, footnotes, endnotes, and comments are not read.

**PPTX** (line 1045 onwards) -- **parsed once, rendered on demand**, same split as Excel.

`extraireStructurePPTX()` collects all `a:t` nodes per `ppt/slides/slideN.xml` plus the
matching `ppt/notesSlides/notesSlideN.xml`, returning
`{diapositives:[{numero, textes, notes}], notesOrphelines}`. No marker text is produced
here. `structurePPTXVersTexte()` renders `--- DIAPOSITIVE N ---` / `--- SLIDE N ---`
and the notes heading at display and export time, because those markers are translated.
Falls back to rendering orphan notes when no slide yielded text.

The slide-to-notes mapping assumes index parity (`slideN` <-> `notesSlideN`), which
holds for simple decks but is **not guaranteed** by OOXML -- the correct way is to
follow the relationship files in `ppt/slides/_rels/`.

**Excel** (line 1203 onwards) -- **parsed once, rendered on demand.**

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
   (line 736) passes `workerPath`, `corePath:"."` and `langPath:"."`, so every OCR
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
   dropdown after analysing. The memo key is `format|nettoyer|langue` -- the language
   is in there because sheet and slide markers are translated, so switching language
   invalidates a cached render exactly as switching format does. Presentations ride
   the same mechanism via `presentation`. If you add another control that affects
   rendered output, it must go
   into that key too, and the language toggle must keep calling `rafraichirTailles()`
   so the displayed sizes follow.

3. **OCR is opt-in, and that is a privacy control, not a preference.** `ocrAutorise`
   starts `false`. `extraireTextePDF()` checks it *before* `initialiserOCR()`, so with
   the switch off Tesseract is never constructed and **no network request is possible**
   -- verified by asserting `ocrWorker === null` after a run with a scanned page. This
   matters because on `file://` enabling OCR means executing third-party CDN code in the
   same page as the user's documents (see finding F-1 reasoning in `SECURITY.md`). Do
   not flip the default to `true` for convenience, and do not move the check after
   `initialiserOCR()`.

   The warning text is protocol-dependent (`majBoutonOCR()`, line 574): on `file://` it
   says the engine comes from a CDN, over HTTP it says the engine is local. Showing one
   fixed string would be false in one of the two cases.

4. **`file://` cannot reach the bundled OCR assets -- handled, do not "simplify".**
   A worker on a `file://` page has an opaque (`null`) origin and may not read local
   files, so `workerPath` / `corePath` / `langPath` are unreachable there whatever value
   they are given. A cross-origin `https` script *is* loadable from a null origin, so
   `cheminsOCR()` (line 827) returns `{}` on `file://`, letting Tesseract fall back to
   its own jsDelivr defaults. Served over HTTP it returns local paths and stays fully
   offline. Both branches are measured, not assumed -- see `SECURITY.md` section 4.

   PDF.js is unaffected: it loses its worker too but silently falls back to the main
   thread, which is why PDFs still parse from `file://`.

   **Do not "upgrade PDF.js to fix CVE-2024-4367".** 4.x and later ship ES modules
   only, and module scripts are CORS-fetched, which `file://` cannot satisfy -- the
   upgrade would break the tool's primary deployment. The vulnerable path is disabled
   via `isEvalSupported:false` instead (`extraireTextePDF`, line 904). See `SECURITY.md`
   finding F-1 before touching either.

5. **OCR failure is per page, never per document.** `initialiserOCR()` (line 854)
   returns a boolean and never throws; the OCR block in `extraireTextePDF()` is wrapped
   so an unreadable page increments `ignorees` and moves on. This is load-bearing: the
   original code let one failed page discard every page already extracted, which cost a
   user a 36-page deck because page 36 needed OCR. `ocrIndisponible` latches after the
   first initialisation failure so a 40-page scan does not retry 40 times, and resets at
   the start of each analysis run so changed connectivity is retried.

6. **Everything runs on the main thread, serially.** A large folder freezes the UI.
   There is no cancel button and no way to resume a partial run.

7. **The whole corpus is held in memory** as one JavaScript string before the Blob is
   created. Very large folders can exhaust the tab.

8. **There are two selection stages, and they are not redundant.** The folder listing
   (`rendreListeSource()`, line 761) ticks files *before* conversion, so unwanted files
   never cost an OCR pass. The results table then drops files *after* conversion, once
   their character counts are visible. Removing either one loses a real capability.
   `webkitdirectory` still takes a whole folder -- individual files cannot be picked
   from the OS dialog.

   Selection lives in `entree.retenu`, **not** in the checkbox. `rendreListeSource()`
   rebuilds the list on every language change, and fresh checkboxes would default back
   to ticked and silently discard the user's choice.

9. **The `estFichierGenere()` guard** (line 648) is name-based only and matches any
   filename containing `documents_concat` or `corpus_concat`.

10. **The version number lives only in the filename** (`2-TXT-v5.html`). There is no
   version constant in the code and no changelog.

11. **The page is deliberately light-only.** `:root` sets `color-scheme:light` and an
   explicit white background, and `body` repeats the background. This is load-bearing:
   the rest of the stylesheet assumes a light ground (`#f7f7f7` zones, white cells,
   `#ddd` borders), so removing either declaration makes the page unreadable against a
   dark host. There is no dark theme -- adding one means moving the palette to CSS
   custom properties and redefining them under `prefers-color-scheme`, which is a
   design decision, not a bug fix.

12. **The Excel noise filter is Markdown-only, by design.** `classeurVersTexte()`
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

*Updated by Claude - 10-Sep-2026, 10:52 EDT*
