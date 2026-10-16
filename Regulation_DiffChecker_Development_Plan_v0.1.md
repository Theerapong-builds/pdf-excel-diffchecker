# Regulation Diff Checker – Development Plan

| Item | Value |
|---|---|
| Document | Development Plan (step-by-step + agentic development) |
| Based on | Regulation_DiffChecker_Requirements_v0.6.1.md (SRS) |
| Version | 0.1 (Draft) |
| Date | 16 Oct 2026 |
| Owner | Regulation Department |

---

## 1. Purpose

This plan tells how to build the first release (Core + optional modules) so that each feature is tested and works before the next feature starts.

The plan has 2 parts:

- **Part A – Step-by-step development plan.** Phases, steps, tests, and gates.
- **Part B – Agentic development plan.** How to use AI coding agents safely to do the steps in Part A.

---

## 2. Development Rules

1. **Core first.** Do not start an optional module until the Core gate (G4) passes (SRS 1.2).
2. **Test first.** For each step: write the test, see it fail, write the code, see it pass.
3. **Gates.** Each phase ends with a gate. Do not start the next phase until the gate passes. Exception: Phase 5 and Phase 6 are independent.
4. **Regression on every change.** Run the seeded test set after every code change (SRS 12.5).
5. **Traceability.** Each test has the requirement ID(s) it checks. Each module header lists its requirement IDs.
6. **Stage dumps.** Each pipeline stage can write its output to a JSON debug file. Use these files to find which stage causes an error.
7. **Read-only.** Never write to input files. Tests check the checksum before and after each run (AC-04).
8. **Strict by default.** No normalisation unless the SRS says so (FR-CD-03).
9. **Permissive licences only.** Check the licence before you add a library (NFR-SEC-07).

---

## 3. Open Questions That Block Work

Get answers before the given point. If there is no answer, use the SRS default in `config.py` and record the decision in `docs/decisions.md`.

| Question | Blocks | Default if no answer |
|---|---|---|
| Q-07 (Python install on laptops), Q-14 (library approval) | End of Phase 0 | None – must answer |
| Q-01 (In Front Of All / After All group rule) | Phase 1, step 6 | SRS 6.3 group definition |
| Q-02 (Chapter + Title joiner) | Phase 1, step 7 | One space (SRS 17.1) |
| Q-03 (Subpoint header names) | Phase 1, step 1 | Subpoint / Paragraph Subpoint / AMC subpoint |
| Q-09 (line-break default), Q-11 (counter min length) | Phase 3 | Off; 15 characters |
| Q-05 (allowed normalisations) | Phase 3, step 5 | None (strict) |
| Q-08 (max file size) | Phase 4, step 12 | Test with the largest real file available |
| Q-06 (report save location) | Phase 4, step 8 | User selects the folder |
| Q-16 (Row view or Page view first) | Phase 5 / 6 order | Row view first (smaller) |

---

## 4. Project Structure

```
regdiff/
├─ run.py                     # one start command (NFR-US-01)
├─ config.py                  # all defaults + module switches (NFR-US-03)
├─ requirements.txt           # pinned versions
├─ THIRD_PARTY_NOTICES.md     # AC-21
├─ AGENTS.md                  # rules for AI agents (Part B)
├─ app/
│  ├─ models.py               # data model (SRS 17.2, 17.3)
│  ├─ pipeline.py             # stage runner, cache, timing log, debug dumps
│  ├─ excel/  reader.py  dupcheck.py  rebuild.py
│  ├─ pdf/    extract.py  generated.py  headerfooter.py  blocks.py  pages.py
│  ├─ align/  align.py
│  ├─ diff/   myers.py  twostage.py  linebreak.py  classify.py
│  │          counter.py  sourcelink.py  unicode_info.py
│  ├─ report/ xlsx.py
│  └─ api/    server.py       # FastAPI, 127.0.0.1 only
├─ web/
│  ├─ index.html  app.js  styles.css
│  ├─ rowview.js              # OPT-RV
│  ├─ pageview.js             # OPT-PV
│  └─ vendor/pdfjs/           # pinned pdfjs-dist (OPT-PV)
├─ tests/
│  ├─ unit/                   # one folder per app package
│  ├─ fixtures/               # reference pair, special PDFs
│  ├─ seeds/                  # S-01 … S-16 (generated)
│  ├─ golden/                 # expected results per seed (JSON)
│  ├─ holdout/                # seeds not visible to AI agents (Part B)
│  └─ e2e/                    # Playwright UI tests
├─ tools/
│  ├─ make_seeds.py  checksum.py  licence_scan.py  trace.py
└─ docs/
   ├─ SRS v0.6.1  decisions.md  traceability.md  phase_reports/
```

---

# Part A – Step-by-Step Development Plan

## 5. Phase Overview

| Phase | Goal | Main SRS IDs | Gate | Size |
|---|---|---|---|---|
| 0 – Foundation | Project, data model, test set, harness | NFR-US-03, NFR-SEC-07, 12.1 | G0 | M |
| 1 – Excel | Read, validate, duplicate check, rebuild, character map | FR-IN, FR-XL, FR-DC, FR-RB | G1 | M |
| 2 – PDF + Alignment | Extract, clean, block, align | FR-PDF, FR-AL | G2 | L |
| 3 – Diff engine | Two-stage diff, line breaks, classes, source links, counter | FR-CD, FR-LB, FR-CL, FR-DU | G3 | L |
| 4 – UI + Report | Text view, panels, XLSX report, real-file test | UI-*, FR-RP, NFR-SEC | G4 = **Core release** | L |
| 5 – OPT-RV | Row view | UI-RV-* | G5 | S |
| 6 – OPT-PV | Page view | UI-PV-*, NFR-SEC-06 | G6 | M |

Phase 0 is new. Phases 1–6 follow SRS Section 15.

---

## 6. Phase 0 – Foundation

**Goal:** Make the test system work before any feature code.

### Steps

1. Create the repository, a Python 3.11 virtual environment, and `requirements.txt` with pinned versions (FastAPI, uvicorn, openpyxl, pypdfium2, pytest; dev only: reportlab, playwright, pip-licenses).
2. Create `config.py` with all defaults: module switches (`OPT_RV = False`, `OPT_PV = False`), `LINE_BREAKS = "off"`, `COUNTER_MIN_LEN = 15`, column header names, field order and separators (SRS 17.1), header/footer zone size, page number patterns.
3. Create `models.py` with dataclasses for: Row, CharMapEntry (17.3), PdfChar (text, page, box, generated), Block, Difference + Repeat (17.2), Warning. Add `to_json()` / `from_json()`.
4. Create `pipeline.py`: one function per stage (SRS 5), a cache for stage outputs, a timing log per stage (NFR-PF-01), and an optional JSON debug dump per stage.
5. Create the **reference pair**:
   1. **Synthetic pair:** write a reference text with chapters, subchapters, lead-ins, After All text, lists ((a), (i), —), quotes, multi-line paragraphs, a header, a footer, and page numbers. Generate the Excel with openpyxl and the PDF with reportlab.
   2. **Real pair:** one real PDF (e.g. HK CAD) and its QC-approved Read-in Excel.
6. Create `tools/make_seeds.py`. It copies the reference Excel and applies each seed S-01 to S-16 (SRS 12.1). For each seed it writes a golden file `tests/golden/S-xx.json` with the expected difference type, cell address, Position, and count.
7. Create special PDF fixtures: one scanned page (image only), one rotated page (/Rotate 90), one page with a CropBox, one page with a MediaBox origin not at (0, 0).
8. Create the regression harness: a parameterised pytest that runs each seed and compares results with the golden file. It prints **missed** and **extra** differences.
9. Create `tools/checksum.py` and a pytest fixture that checks input checksums before and after each run (AC-04).
10. Create `tools/licence_scan.py` (pip-licenses). It fails on GPL / AGPL. Create `THIRD_PARTY_NOTICES.md` (AC-21).
11. Create `tools/trace.py`. It reads `@pytest.mark.req("FR-XL-08")` markers and writes `docs/traceability.md`.
12. Write `AGENTS.md` (Part B, Section 14).

### Gate G0

- [ ] Seeds S-01 to S-16 and golden files are generated.
- [ ] The harness runs. All seeds fail with "not implemented" (expected).
- [ ] Licence scan passes.
- [ ] Q-07 and Q-14 have answers.

---

## 7. Phase 1 – Excel

**Goal:** Rebuilt Excel text is correct and each character traces to its cell.

### Steps

1. **Reader** (FR-IN-01 to 03, FR-XL-01 to 03, FR-XL-11): open read-only, read only the Standards sheet, find columns by header name (ignore case and extra spaces), stop and list missing columns. Open .xlsm without macros.
2. **Escape check** (FR-XL-05): make a test cell with `_x000D_`, CR, LF, tab, NBSP, and zero-width space. Record exactly what openpyxl returns (openpyxl can unescape `_x000D_` to CR). Write the result in `docs/decisions.md`. Make sure both forms show as ¤ and count as a line break (6.7.1, FR-LB-09).
3. **Position check and sort** (FR-XL-08, FR-RB-01): report duplicate, empty, and non-numeric Position values. Do not report gaps. Sort by numeric Position with a stable sort.
4. **Row fields** (FR-XL-09, 12, 14): merge the 3 Subpoint columns and warn on conflicts. Warn on empty Standard. Put rows with empty Subchapter under "No subchapter".
5. **Duplicate consistency check** (FR-DC-01, 02, 04, 06, 07): compare all copies of each repeated field in its group. Find the majority value. Report "Inconsistent duplicate" with the character difference. Report a Chapter with 2 or more Chapter Titles.
6. **Group detection** (SRS 6.3 group definition): rows next to each other after the sort, with the same Subchapter and the same In Front Of All / After All value.
7. **Rebuild** (FR-RB-02 to 08, FR-XL-13): write headings only when they change, In Front Of All once at group start, After All once at group end, Standard for each row, Subpoint only if no line of Standard starts with it. Use the majority value for repeated fields.
8. **Character map** (FR-RB-09, 10): one entry per rebuilt character. Mark separators and added line breaks as synthetic.

### Tests

| Test | Checks |
|---|---|
| Appendix 17.1 example as a unit test | Sort + rebuild layout |
| Character map invariant: for each non-synthetic character, `cell_value[char_offset] == character` | FR-RB-09 |
| S-07, S-08 | Inconsistent duplicate found with correct cell |
| S-09, S-13 | Sort result |
| Cell with 40,000 characters | NFR-PF-04 |
| Missing column, duplicate Position, Subpoint conflict, empty Subchapter | Warnings and errors |

### Gate G1

- [ ] Rebuilt text is correct on S-07, S-08, S-09, S-13 (SRS Phase 1 exit).
- [ ] Character map invariant passes on all seeds and the real Excel.
- [ ] Escape behaviour is recorded in `decisions.md`.

---

## 8. Phase 2 – PDF Extraction and Alignment

**Goal:** Clean PDF blocks align with Excel blocks on the reference pair.

### Steps

1. **Extract** (FR-PDF-01, 02, 15, 18): use pypdfium2 on one worker thread. For each character store text, page, box (`get_charbox(index, loose=False)`), page size, rotation, MediaBox, CropBox. Close all PDFium objects.
2. **Generated characters** (FR-PDF-16): use `FPDFText_IsGenerated` (pypdfium2 raw API). Count CRLF as one line break. Mark generated spaces. Give a generated character the box of the previous character.
3. **Page checks** (FR-PDF-03, 20): mark pages with no text as "Scanned page – not compared". Warn for rotated pages.
4. **Ignored pages** (FR-PDF-09): parse ranges like `1-3, 7, 10-12`. Reject bad input with a clear message.
5. **Header / footer removal** (FR-PDF-04, 13): text in the top or bottom zone that repeats on most pages, and page number patterns. Keep a list of all removed text for the warnings panel.
6. **Blocks** (FR-PDF-05): group lines into blocks. Start a new block at an anchor pattern (Subpoint or heading number) or at a large vertical gap. Keep line ends inside a block as line breaks.
7. **Alignment** (FR-AL-01 to 04):
   1. Match anchors in order (longest common subsequence on the anchor list). This stops a missing block from moving the blocks after it.
   2. Between matched anchors, align blocks with no anchor by text similarity.
   3. Report blocks on one side only as Missing / Extra.

### Tests

| Test | Checks |
|---|---|
| Reference pair: no generated character in the compared text | AC-22 (early check) |
| Header/footer fixture: all headers, footers, page numbers removed; no body text removed | AC-12 (early check) |
| Scanned and rotated fixtures | FR-PDF-03, 20 |
| Ignored page parser: good and bad input | FR-PDF-09 |
| S-05, S-06: exactly one Missing / one Extra block, no shift after it | FR-AL-03, AC-03 (early check) |
| Real PDF: dump blocks and check by eye against the PDF | Reading order |

### Gate G2

- [ ] All blocks align on the reference pair (0 Missing, 0 Extra).
- [ ] S-05 and S-06 give exactly one Missing / Extra block.
- [ ] Removed header/footer list is correct on the real PDF.

---

## 9. Phase 3 – Diff Engine

**Goal:** Find every seeded difference with the correct type, source link, and counter.

### Steps

1. **Minimal diff** (FR-CD-04): implement Myers O(ND) (or use diff-match-patch). Use difflib only as a fallback with `autojunk=False`.
2. **Two-stage diff** (FR-CD-01, 02): tokenise into words and whitespace. Run a word diff, then a character diff inside each changed word run. Compare whitespace between matched words separately. Compare code points.
3. **Line-break setting** (FR-LB-01 to 11): apply the setting only to in-paragraph breaks.
   - Strict: report each line-break difference.
   - Off: treat a break plus spaces next to it as one space. Still report a break against no character (FR-LB-06). Always report a line-end hyphen (FR-LB-08). Count "Line breaks ignored".
4. **Classification** (FR-CL-01, 6.9.1): use a fixed rule order and record it in `decisions.md`. Proposed order: Line break → Unicode form (NFC equal) → Case (casefold equal) → Quote → Dash → Space → Number → Punctuation → Text.
5. **Unicode details** (FR-CD-05, 06): code point and name with `unicodedata`. The invisible-symbol map (6.7.1) is for display only. It never goes into the diff.
6. **Source links** (FR-CD-07): map each difference to PDF page and boxes, and to Excel cell, offset, and Position (character map). For Added text with no PDF glyph, set `insert_point`.
7. **Duplication counter** (FR-DU-01 to 07, 11): find the full Excel line, count exact matches in Excel and PDF in the Subchapter scope ("No subchapter" scope when empty), skip lines shorter than `COUNTER_MIN_LEN`, mark Repeated when N > M and N ≥ 2, give repeat group IDs, and store all copies. The counter never changes type or total count.
8. **Fast re-run** (NFR-PF-05, FR-LB-10): when the line-break setting changes, re-run from the diff stage with cached Excel and PDF data.

### Tests

| Test | Checks |
|---|---|
| Reconstruction invariant: equal + deleted = PDF text; equal + inserted = Excel text (all seeds + 1,000 random string pairs with a fixed seed) | 0 missed differences (NFR-AC-01) |
| Minimality: edit count ≤ difflib (autojunk=False) on random pairs | FR-CD-04 |
| SRS 6.8 example table (3 rows × 2 settings) | FR-LB-03 to 06 |
| One test per difference type (6.9.1) | FR-CL-01 |
| Every difference has a valid source link | NFR-AC-02 |
| S-15, S-16 counters and groups | AC-14, 25, 31 |
| Counter on/off gives the same total and types | AC-16 |
| Setting change time < 3 s on the largest real file | NFR-PF-05 |

### Gate G3

- [ ] AC-01, AC-02, AC-03, AC-09, AC-10, AC-14, AC-16, AC-22, AC-25, AC-31 pass in the harness.
- [ ] Reconstruction invariant passes.
- [ ] Holdout seeds pass (Part B, Section 15).

---

## 10. Phase 4 – UI, Report, and Core Release

**Goal:** A reviewer can do a full QC job with the tool and export the report.

### Steps

1. **Server** (NFR-SEC-01): FastAPI bound to 127.0.0.1. Endpoints: run comparison, get results, change line-break setting, get warnings, export report. Return a busy state while a run is active.
2. **Start** (NFR-US-01): `python run.py` starts the server and opens the browser.
3. **Header and summary** (7.1, UI-15, FR-LB-11, NFR-US-05): file pickers or paths, ignored pages, Run, reviewer name, Export, Show invisible, Line breaks dropdown, summary counts, version number.
4. **Text view** (UI-01, 03, 08, 19, 20): two monospace panes, aligned blocks level, blank space for missing blocks, colours plus a second marker, busy indicator.
5. **Difference list** (UI-18, UI-21): sort by order, type, anchor, page, cell, repeat group. N / P keys. Counter badge ×N (PDF ×M).
6. **Detail panel** (UI-04, UI-06, UI-18): code points, names, cell, Position, page, Copy cell address, list of copies.
7. **Warnings panel** (UI-12): all warning types.
8. **XLSX report** (FR-RP-01, 03, 06, FR-DU-12): one row per difference, Status data validation (Open / OK / Error / Ignored), Comment, repeat columns, header with settings and counts, Warnings sheet. Write a new file only. Refuse a path that is an input file.
9. **Unsaved warning** (UI-22).
10. **Network check** (AC-05): search `web/` and `app/` for external URLs; watch connections with `netstat` during a full run.
11. **UI tests** (Playwright, Python): load a seed, click a difference, check scroll and detail, change the setting, export, compare the report count with the screen count (AC-07, AC-32).
12. **Real-file test** (AC-08): run on HK CAD and UK CAA Part-FCL files. For each real problem, make a small new fixture and add it to the regression set.
13. **User test:** one QC reviewer does one real job with the tool. Record findings in `docs/phase_reports/`.

### Gate G4 – Core Release

- [ ] AC-04, AC-05, AC-07, AC-08, AC-12, AC-21, AC-26, AC-32 pass.
- [ ] All G3 criteria still pass with both modules off.
- [ ] User test has no open blocker.
- [ ] Tag the release (e.g. `v1.0.0-core`).

---

## 11. Phase 5 – OPT-RV Row View

1. Add the Group / Row switch when `OPT_RV = True` (UI-RV-01). Hide all controls when off.
2. Show every row in Position order with all used columns (UI-RV-02, 06).
3. Draw differences on cells with the character map. Hide synthetic characters (UI-RV-03).
4. Purple highlight on inconsistent duplicates (UI-RV-04). Repeat group ID next to cells (UI-RV-08).
5. Scroll to the row on difference select (UI-RV-05). Keep the same selection when the view changes.
6. Fold long cells to 3 lines with expand (UI-RV-07).

**Gate G5:** AC-27, AC-28, AC-29. G4 regression still passes.

---

## 12. Phase 6 – OPT-PV Page View

1. Bundle a pinned `pdfjs-dist` (core library only, `isEvalSupported: false`). Add the version to THIRD_PARTY_NOTICES (NFR-SEC-06).
2. Add an endpoint that sends the PDF bytes from 127.0.0.1 (UI-PV-02).
3. Add an endpoint that sends boxes per page, only for pages in view and only boxes with a difference (NFR-PF-07).
4. Render pages lazily with a limited page cache (UI-PV-10). Add page navigation and zoom (UI-PV-09).
5. Convert boxes with `viewport.convertToViewportRectangle`, normalise min/max, add 1–2 px padding (UI-PV-03).
6. Draw the overlay for each difference type (SRS 9.2), with UI-08 colours and markers.
7. Click, select, scroll, flash (UI-PV-05). Tooltip (UI-PV-06, 12).
8. Update highlights on a line-break setting change without re-render (UI-PV-08).
9. Show "Page view not available for this file" if PDF.js fails (UI-PV-15).

**Tests:** screenshot tests on the rotated, CropBox, and MediaBox fixtures; highlight count per page = difference count per page; checksum before and after.

**Gate G6:** AC-17, AC-18, AC-19, AC-20, AC-23, AC-30. G4 regression still passes.

---

## 13. Definition of Done (each step)

1. Tests for the step exist and have requirement ID markers.
2. Unit tests and the full regression set pass.
3. Licence scan passes (if a library was added).
4. `docs/traceability.md` is updated (`tools/trace.py`).
5. Decisions are recorded in `docs/decisions.md`.
6. A human reviewed and merged the change.

---

# Part B – Agentic Development Plan

## 14. Approach

- **Human = owner.** The human selects tasks, approves tests, reviews code, and passes gates.
- **Agent = worker.** The agent does one task card at a time, inside the limits in Section 16.
- Use only AI coding tools on the company-approved list.
- **Data rule:** give agents only synthetic fixtures. Run real regulation files on the local machine only, unless IT approves otherwise.

### 14.1 AGENTS.md (put in the repository root)

```markdown
# Rules for AI agents – Regulation Diff Checker

## Source of truth
- Requirements: docs/Regulation_DiffChecker_Requirements_v0.6.1.md
- Decisions: docs/decisions.md
- Use requirement IDs (e.g. FR-XL-08) in code comments and test markers.

## Hard rules
1. Never write to input PDF or Excel files. Open them read-only.
2. Strict mode: never normalise text unless the SRS says so.
3. Never hide a difference without a count.
4. Only pypdfium2 text is compared. Never use PDF.js text for comparison.
5. No GPL/AGPL libraries. No new library without human approval.
6. No network calls. Server binds to 127.0.0.1 only. No CDN.
7. Do not edit tests/golden/, tools/make_seeds.py, or the SRS.
8. All defaults go in config.py.

## Commands
- Unit tests:     pytest tests/unit -q
- Regression:     pytest tests/seeds -q
- Licence scan:   python tools/licence_scan.py
- Traceability:   python tools/trace.py

## Done means
- All commands above pass. Paste the test output in your final message.
- List changed files and the requirement IDs covered.
- List any assumption you made.
```

---

## 15. Agent Roles

Use a **separate agent session** for each role. The agent that writes the code must not write its own acceptance tests.

| Role | Input | Output | Human check |
|---|---|---|---|
| **Planner** | SRS section, phase | Task cards (Section 17) | Approve task list |
| **Test writer** | Task card, SRS IDs | Failing tests + fixtures | **Approve tests before code** |
| **Implementer** | Task card, approved tests | Code that passes the tests | Code review |
| **Reviewer** | Diff of the change, task card, SRS IDs | Review notes: SRS gaps, hidden normalisation, read-only risk, licence | Read notes, decide |
| **Debugger** | Failing test, stage debug dumps | Root cause + small fix | Code review |

**Holdout set:** keep some seed variants (e.g. same seed types on other paragraphs) in `tests/holdout/`. Do not show them to agents. Run them at each gate. This finds code that is fitted to the visible tests only.

---

## 16. Agent Loop and Guardrails

### 16.1 Loop for each task card

1. Human selects the next task card (dependencies done).
2. Test writer agent writes tests. Human approves.
3. Implementer agent writes code until the tests pass. It runs all commands in AGENTS.md.
4. Reviewer agent reviews the change against the SRS IDs.
5. Human reviews, runs the regression set locally, and merges.
6. Update `docs/traceability.md` and the task card status.

### 16.2 Guardrails

| Guardrail | Rule |
|---|---|
| Task size | One task card = one module or one small feature. Aim for under 300 changed lines |
| Protected files | `tests/golden/`, `tests/holdout/`, `tools/make_seeds.py`, SRS, `AGENTS.md` – human changes only |
| Test changes | The implementer must not change approved tests. If a test seems wrong, stop and report |
| Retries | After 3 failed attempts, stop and report the cause. Do not weaken tests |
| Dependencies | New library = human approval + licence scan |
| Context | Give the agent only the SRS sections for the task, plus `AGENTS.md`, `models.py`, `config.py` |
| Evidence | The agent pastes test output. "Tests pass" without output is not accepted |
| Commits | One commit per task card, message starts with the task ID (e.g. `T-1.3`) |

---

## 17. Task Cards

### 17.1 Task card template

```markdown
## T-<phase>.<n> – <title>
- SRS IDs: <IDs>
- Depends on: <task IDs>
- Files allowed to change: <paths>
- Inputs: <fixtures, models>
- Output: <functions / endpoints>
- Acceptance tests: <test files / seeds>
- Out of scope: <items>
- Human check: Low | Medium | High
```

### 17.2 Task list

| Task | Title | SRS IDs | Depends on | Human check |
|---|---|---|---|---|
| T-0.1 | Repo, venv, pinned requirements, config.py | NFR-US-03 | – | Low |
| T-0.2 | Data model (models.py) | 17.2, 17.3 | T-0.1 | **High** |
| T-0.3 | Pipeline skeleton, cache, timing, dumps | NFR-PF-01 | T-0.2 | Medium |
| T-0.4 | Synthetic reference pair generator | 12.1 | T-0.1 | **High** |
| T-0.5 | Seed generator + golden files S-01 to S-16 | 12.1 | T-0.4 | **High (human-owned)** |
| T-0.6 | Special PDF fixtures | 11 (#14, 22) | T-0.4 | Medium |
| T-0.7 | Regression harness, checksum fixture | 12.5, AC-04 | T-0.5 | Medium |
| T-0.8 | Licence scan, trace tool, THIRD_PARTY_NOTICES | NFR-SEC-07, AC-21 | T-0.1 | Low |
| T-1.1 | Excel reader + header match + escape check | FR-IN-01–03, FR-XL-01–03, 05, 11 | T-0.7 | Medium |
| T-1.2 | Position check + sort | FR-XL-08, FR-RB-01 | T-1.1 | Low |
| T-1.3 | Subpoint merge, No subchapter, empty Standard | FR-XL-09, 12, 14 | T-1.1 | Low |
| T-1.4 | Duplicate consistency check | FR-DC-01–07 | T-1.3 | Medium |
| T-1.5 | Group detection + rebuild + character map | FR-RB-02–10, FR-XL-13 | T-1.4 | **High** |
| T-2.1 | PDF extraction + boxes + worker thread | FR-PDF-01, 02, 15, 18 | T-0.7 | Medium |
| T-2.2 | Generated characters | FR-PDF-16 | T-2.1 | Medium |
| T-2.3 | Scanned / rotated pages, ignored ranges | FR-PDF-03, 09, 20 | T-2.1 | Low |
| T-2.4 | Header / footer removal | FR-PDF-04, 13 | T-2.1 | Medium |
| T-2.5 | Line → block grouping | FR-PDF-05 | T-2.4 | Medium |
| T-2.6 | Block alignment | FR-AL-01–04 | T-1.5, T-2.5 | **High** |
| T-3.1 | Myers minimal diff | FR-CD-04 | T-0.2 | Medium |
| T-3.2 | Two-stage diff + reconstruction invariant | FR-CD-01, 02 | T-3.1 | **High** |
| T-3.3 | Line-break setting | FR-LB-01–11 | T-3.2 | **High** |
| T-3.4 | Classification + Unicode details | FR-CL-01, FR-CD-05, 06 | T-3.2 | Medium |
| T-3.5 | Source links | FR-CD-07, NFR-AC-02 | T-2.6, T-3.2 | Medium |
| T-3.6 | Duplication counter | FR-DU-01–07, 11 | T-3.5 | **High** |
| T-3.7 | Fast re-run on setting change | NFR-PF-05 | T-3.3 | Low |
| T-4.1 | FastAPI server + run.py | NFR-SEC-01, NFR-US-01 | T-3.7 | Medium |
| T-4.2 | Header, summary, Text view | UI-01, 03, 08, 15, 19, 20 | T-4.1 | Medium |
| T-4.3 | Difference list, detail, warnings panels | UI-04, 06, 12, 18, 21 | T-4.2 | Medium |
| T-4.4 | XLSX report | FR-RP-01, 03, 06, FR-DU-12 | T-3.6 | **High** |
| T-4.5 | Unsaved warning + Playwright UI tests | UI-22, AC-07, AC-32 | T-4.3, T-4.4 | Medium |
| T-4.6 | Real-file test + new fixtures | AC-08 | T-4.5 | **Human only (real data)** |
| T-5.x | Row view (one card per Section 11 step) | UI-RV-01–08 | G4 | Medium |
| T-6.x | Page view (one card per Section 12 step) | UI-PV-01–15 | G4 | Medium |

Tasks in the same phase with no dependency between them can run in parallel agent sessions (e.g. T-1.2 and T-1.3; T-2.2, T-2.3, and T-2.4; T-3.1 can start in Phase 1).

---

## 18. Prompt Templates

### 18.1 Test writer

```
Read AGENTS.md and SRS sections <sections>.
Task card: <paste>.
Write pytest tests only. Do not write feature code.
Mark each test with @pytest.mark.req("<ID>").
Cover: normal case, each edge case in SRS Section 11 for these IDs, and error input.
Use only fixtures in tests/fixtures and tests/seeds.
The tests must fail now. Show the pytest output.
```

### 18.2 Implementer

```
Read AGENTS.md, app/models.py, config.py, and SRS sections <sections>.
Task card: <paste>.
Make the approved tests in <test files> pass. Do not change these tests.
Change only: <allowed files>.
Run all commands in AGENTS.md and paste the output.
List assumptions and any SRS text that was not clear.
If you fail 3 times, stop and report the cause.
```

### 18.3 Reviewer

```
Read AGENTS.md and SRS sections <sections>.
Review this change: <diff>.
Check:
1. Each SRS ID in the task card is fully done.
2. No hidden normalisation or skipped difference.
3. No write to input files. No network call.
4. Every difference keeps a source link.
5. No new library without a licence check.
6. Tests check behaviour, not only that the code runs.
List problems by severity: Blocker / Major / Minor.
```

### 18.4 Debugger

```
Read AGENTS.md.
Failing test: <name + output>.
Stage dumps: <paths>.
Find the first stage where the output is wrong. Explain the root cause.
Propose the smallest fix. Do not change tests.
```

---

## 19. Human Checkpoints

| When | Human does |
|---|---|
| Each task card | Approve tests, review code, merge |
| T-0.2, T-0.5 | Write or fully check the data model and golden files (they define "correct") |
| End of each phase | Run gate checks, run holdout seeds, write a short phase report in `docs/phase_reports/` |
| T-4.6 | Run real files locally. Turn each real problem into a synthetic fixture for agents |
| G4 | QC reviewer user test, release tag |

---

## 20. Risks

| Risk | Effect | Action |
|---|---|---|
| openpyxl unescapes `_x000D_` | Wrong display or missed difference | T-1.1 escape test; record in decisions.md |
| PDFium reading order wrong on some layouts | False Missing / Extra blocks | Block dumps on real PDFs in Phase 2; rotated warning |
| Header/footer rule removes body text | Missed differences | Removed-text list in UI; fixture with body text in top zone |
| Two types fit one difference | Unstable classes | Fixed rule order in decisions.md + one test per type |
| No anchors in a section | Bad alignment | Similarity fallback tests; real-file test |
| Agent fits code to visible seeds | Passes tests, fails real files | Holdout seeds, reconstruction invariant, real-file test |
| Agent changes tests to pass | False pass | Protected files; human approves test changes |
| Real regulation data sent to a cloud agent | Policy breach | Synthetic fixtures only for agents (Section 14) |
| Large files are slow | Poor review experience | Stage timing log; test with the largest file (Q-08) |
