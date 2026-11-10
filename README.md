# GapFinder AI — Find What The Student Doesn't Know

Hackathon PS 2026 · AI Intern

A diagnostic-first study tool. It does not just mark answers right or wrong — it works out **which topics you are weak in, why, and in what order to fix them**, then rebuilds the plan after every re-test.

```
Upload  →  Test  →  Find weak spots  →  Plan  →  Re-test  →  Plan updates
```

Everything runs in one `index.html` file: vanilla HTML, CSS and JavaScript, no build step, no framework, no dependencies.

---

## Quick start

1. Double-click `index.html`. That is it — there is no install step.
2. Press **Run Judge Demo in 60 Sec**. It loads the sample corpus, sits two attempts (30% then 40%), and lands on the Gap Report, then the Revision Plan.
3. Or sign in: `demo@student.com` / `demo123` (student) or `teacher@demo.com` / `teacher123` (teacher).

To deploy: push this folder to Vercel as a static site, or drag it into the Vercel dashboard. No configuration, no framework preset, no build command.

---

## The pages

| Page | What it does |
| --- | --- |
| **Login** | Two demo accounts, one-click demo student, one-click judge demo |
| **Dashboard** | Mastery ring, weak-topic alert, next task, top 3 gaps, streak and time studied, quick actions |
| **Upload & Topic Map** | Paste text, drop a file, or load the sample. Shows the 6-topic map with source, exam weight and status, plus the full chunk index |
| **Diagnostic Quiz** | 10 MCQs, each with a citation chip (`Chapter 3, Section 2`), 10-minute timer, progress bar, Prev / Next / jump, submit confirmation |
| **Gap Report** *(hero)* | Weak / medium / strong bands with a plain-language reason, `Priority = (100 − Accuracy) × Exam Weight`, missed-question explanations |
| **Revision Plan** | Days (1–14) and hours/day (1–4) sliders, sessions sorted by priority, day-by-day schedule ending in a re-test |
| **Explain This Topic** | Answers only from retrieved chunks. The retrieved chunk and citation are shown **before** the answer. Out-of-scope questions get "Not in your uploaded material" |
| **Progress** | Attempt history and per-topic improvement between attempts |
| **Teacher View** *(bonus)* | Class-level weak topics, students below 60%, and a generated intervention plan |

A RAG explainer strip sits on every page: retrieval → generation from those chunks only → citation back to the source.

---

## File support

**Import** — parsed entirely in the browser:

| Format | Method |
| --- | --- |
| `.pdf` | Content streams are inflated with the browser's built-in `DecompressionStream`, then text operators (`Tj`, `TJ`, `Td`, `TD`, `T*`, `Tm`, `TL`) are decoded **with pen positions**, so words placed one-by-one by design tools are rejoined into real lines and paragraphs (large gaps become spaces, big vertical gaps become paragraph breaks). Font sizes are tracked so large-type lines become headings and therefore citations. Symbol-font junk from designed resumes (stray `Í®`-style glyphs, `±` used as a dash, soft hyphens, C1 controls) is cleaned automatically |
| `.docx` | The OOXML zip is read directly (central directory + raw deflate), `word/document.xml` is converted to paragraphs, and `Heading` styles become `##` headings |
| `.pptx` | Slides are read straight out of the OOXML zip (`ppt/slides/slideN.xml`). Title-placeholder text becomes `##` headings, so each slide can be cited |
| `.xlsx` | `xl/sharedStrings.xml` + `xl/worksheets/sheetN.xml` are read into rows. One heading per sheet |
| `.csv` / `.tsv` | Proper quoted-field parser. The header row becomes the citation title |
| `.rtf` | A small group-aware RTF reader: unicode (`\uN`) and hex (`\'xx`) escapes are decoded, while font tables, stylesheets and pictures are skipped |
| `.txt` / `.md` | Read as-is. `##` headings become citations |
| `.doc` / `.xls` / `.ppt` | Best-effort extraction of readable text runs from the legacy binary format. Re-save as `.docx` / `.xlsx` / `.pptx` for clean citations |

Every import is checked by **magic bytes, not just the file extension**: a `.ppt` renamed to `.txt` is still recognised and either parsed or refused with a named reason; plain `.zip` archives, images and font files (which is what embedded-binary garbage such as font tables comes from) get targeted messages instead of being dumped as text; and any binary text that slips past the signatures is caught by a garbage detector. If a file cannot be read, the app says exactly why in a red panel — a scanned PDF with no text layer, an image-only deck, an empty workbook or a legacy binary format all get different messages. It never fails silently.

A file that is readable but very short still imports: it gets an amber *"Short file — only N readable characters"* note under the green panel rather than being refused, so a small spreadsheet or a short handout is never blocked. Only files with essentially no text (fewer than 20 readable characters) are rejected.

**Export** — every report can be downloaded as:

- **PDF** — a real A4 PDF written byte by byte in JavaScript (Helvetica, correct xref table, multipage, aligned table columns)
- **Word `.docx`** — a genuine OOXML package with real `w:tbl` tables
- **Plain text `.txt`**

Available for the Gap Report, the Revision Plan and the Class Report.

---

## How the engine works

1. **Chunking** — the material is split on blank lines. Lines that look like headings become citation labels; long paragraphs are cut at sentence boundaries.
2. **Topic mapping** — each chunk is matched against the 6 syllabus topics by keyword, which produces the topic map and the "not found in material" warning.
3. **Retrieval** — a small TF-IDF variant. Body term frequency (weight 0.6) plus heading terms (weight 1.6, IDF-scaled) plus a metadata boost when the query names a heading. Cosine-normalised by query length, with a 1.2 cut-off so off-topic questions retrieve nothing.
4. **Answering** — the local fallback is extractive: the two best-matching sentences from the top chunk. An optional Groq key can phrase it better, but the system prompt restricts generation to the retrieved chunks.
5. **Diagnosis** — per-topic accuracy, band thresholds (weak < 50, medium 50–79, strong ≥ 80), and `Priority Score = (100 − Accuracy) × Exam Weight`.
6. **Planning** — repetitions scale with priority (≥ 12 → 3 reps, ≥ 7 → 2, ≥ 2 → 1), sessions are interleaved across rounds, slots per day come from the hours slider, and the last day always ends with a re-test that rebuilds the plan.

---

## Data and privacy

- Nothing is uploaded anywhere. All parsing, scoring and export happen in the tab.
- Progress, material, plan and streak are stored in `localStorage` under `gapfinder.v1`.
- The sample corpus (6 topics, 18 chunks, 10 cited questions) ships inside the file, so the whole app works offline with no network access.

## Groq (optional)

No API key is included in this repository — a key inside a static HTML file is public the moment it deploys. The Explain page has a key field that stores your key in `localStorage` only. Answers stay grounded in the retrieved chunks either way; without a key the local extractive answerer is used. For production, put the key behind a small serverless proxy.

---

## Honest limitations

- A scanned PDF has no text layer, so there is nothing to extract. The app detects this and says so; run OCR first.
- Legacy `.doc` / `.xls` are binary formats and only get a best-effort read.
- `.xlsx` reads cell text only — formulas evaluate to their cached values, and charts or images are ignored.
- Retrieval is lexical, not embedding-based. It is strong on the bundled corpus and reasonable on well-headed notes, but a single giant paragraph with no headings will produce weak citations.
- File size: 158 KB uncompressed, ~46 KB gzipped. It is over the 100 KB target in the brief because the design system, the full sample corpus, a PDF writer, a DOCX writer and three file parsers all live inline. Minifying would recover roughly 8 KB, not 58 KB.

## Verification

- 50 headless assertions across two suites: chunking (18 chunks / 6 topics), retrieval accuracy and the off-topic cut-off, attempt scoring, gap ordering, plan generation, the analyze success and failure panels, CSV / DOCX / XLSX / PDF import, a 2-sheet 19-row spreadsheet, the short-file and empty-file branches, all 8 page renderers, and PDF + DOCX round-trips. Zero failures.
- Generated PDFs opened with `pypdf`: text extracted correctly, 1–2 pages each.
- Generated DOCX files opened with `python-docx`: paragraphs and tables parsed, real `w:tbl` grids with `w:tblGrid`.
- Browser QA on a live server: every nav item, the quiz flow, PDF / DOCX / XLSX / CSV uploads through the real file picker, the analyze error path, PDF + Word + text export from all three reports, the RAG refusal path, and a 390 × 844 mobile pass with no horizontal overflow. Zero console errors and zero warnings.

## Project structure

```
index.html   the entire application (markup, design system, engine, parsers, exporters)
README.md    this file
```
