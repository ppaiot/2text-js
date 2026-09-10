# Third-party notices

2TEXT-JS bundles the following libraries unmodified. Each is distributed under
its own licence, and each minified file retains its original licence header.

| Library | Version | Licence | Project |
|---|---|---|---|
| PDF.js | 3.11.174 | Apache-2.0 | https://github.com/mozilla/pdf.js |
| JSZip | 3.10.1 | MIT or GPLv3 (dual) | https://github.com/Stuk/jszip |
| SheetJS (`xlsx`) | 0.18.5 | Apache-2.0 | https://github.com/SheetJS/sheetjs |
| Tesseract.js | 5.1.1 | Apache-2.0 | https://github.com/naptha/tesseract.js |
| Tesseract.js core (WASM) | 5.1.1 | Apache-2.0 | https://github.com/naptha/tesseract.js-core |
| Tesseract OCR models (`fra`, `eng`) | 4.0.0_best_int | Apache-2.0 | https://github.com/tesseract-ocr/tessdata |

The bundled OCR engine builds (`tesseract-core-simd-lstm.wasm.js`,
`tesseract-core-lstm.wasm.js`) and the language models (`fra.traineddata.gz`,
`eng.traineddata.gz`) are redistributed so that OCR works without a network
connection. They are unmodified upstream artefacts.

The 2TEXT-JS source itself is MIT licensed -- see [LICENSE](LICENSE).

---

*Updated by Claude - 10-Sep-2026, 09:37 EDT*
