# Regulation Diff Checker – Software Requirements Specification

| Item | Value |
|---|---|
| Document | Software Requirements Specification (SRS) |
| Product | Regulation Diff Checker (local PDF ↔ Excel comparison tool) |
| Owner | Regulation Department |
| Version | 0.6.1 (Draft) – **First release scope (Core + optional modules)** |
| Base version | 0.6.0 |
| Date | 16 Oct 2026 |
| Status | For review |

## Table of Contents

1. Purpose and Scope
2. Definitions
3. Users and Use Cases
4. Input Data Structure
5. Comparison Flow
6. Functional Requirements (Core)
7. User Interface Requirements (Core)
8. Optional Module: Row View
9. Optional Module: PDF Page View
10. Non-Functional Requirements
11. Edge Cases
12. Testing and Acceptance
13. Out of Scope
14. Deferred Features (later versions)
15. Delivery Phases
16. Open Questions
17. Appendix
18. Change Log

---

## 1. Purpose and Scope

### 1.1 Purpose

The Regulation Diff Checker is a locally hosted, read-only application. It compares **one regulation PDF** with **one Regulation Read-in Excel file** and shows every difference, down to single characters. Use it for:

- **Quality checking (QC):** confirm that the Excel content matches the source PDF exactly.
- **Amendment updates:** after an editor updates the Excel for a new amendment, confirm that the updated Excel matches the new amended PDF.

### 1.2 Release Scope

This version defines the **first release**. It has 3 parts:

| Part | Content | Status |
|---|---|---|
| **Core** | Load files, Excel rebuild, PDF extraction, alignment, character diff, duplication counter, Text view, XLSX report | Required. Release is not complete without it |
| **Optional module: Row view (OPT-RV)** | Show every Excel row with its used columns, with differences on the cells | Optional. Build after Core passes acceptance |
| **Optional module: Page view (OPT-PV)** | Show the rendered PDF page with differences highlighted on the page | Optional. Build after Core passes acceptance |

Rules for optional modules:

1. Core must work fully with both modules turned off.
2. Each module is turned on or off with one setting in `config.py`. When a module is off, its UI controls are not shown.
3. A module must not change Core results (difference count, types, or source links).

Features moved out of this release are listed in Section 14.

### 1.3 Key Principle

**The tool never changes source files and never hides a difference without a count.**

---

## 2. Definitions

| Term | Meaning |
|---|---|
| Read-in File | The Excel workbook in the approved template format. The tool reads only the Standards sheet |
| DB export | An Excel file downloaded from the regulation database. Same Standards columns as the Read-in File, but the text structure can be different (see 4.4). The tool does not need to know if a file is a DB export |
| Core | The minimum feature set for the first release |
| Optional module | A feature set that can be turned on or off without an effect on Core |
| Duplication counter | A count that shows how many times the text of a difference occurs in the Excel and in the PDF (see 6.12) |
| Repeated line | A line of Excel text that occurs more times in the Excel than in the PDF |
| Repeat group | All differences caused by the same repeated line |
| Row | One row in the Standards sheet |
| Repeated / duplicated field | A column whose value repeats across many rows (Chapter, Chapter Title, Subchapter, In Front Of All, After All) but appears **once** in the PDF |
| Subpoint | The paragraph number of a row. The Subpoint, Paragraph Subpoint, and AMC subpoint columns are treated as one Subpoint field |
| Rebuild | Turning Excel rows into one continuous text that follows the PDF layout |
| Character map | A lookup from each character in the rebuilt text to its source cell, character offset, and Position |
| Block | An aligned unit for comparison (normally one Subpoint / paragraph) |
| Anchor | A stable identifier used for alignment (Subpoint number, heading number) |
| Strict mode | Comparison with no normalisation. Every code point counts |
| In-paragraph line break | A line break (CR, LF, CRLF, \_x000D\_, U+2028) inside one block, not between blocks |
| Code point | One Unicode character value, e.g. U+2019 |
| Generated character | A space or line break that PDFium adds and that is not in the PDF content stream |

---

## 3. Users and Use Cases

### 3.1 Users

| User | Need |
|---|---|
| Regulation editor | Checks their own Excel import before handover |
| QC reviewer | Checks another editor's work and records the result in the exported report |
| Team lead | Reads QC reports and summary counts |

### 3.2 Use Cases

| ID | Use case |
|---|---|
| UC-01 | QC a new regulation: compare the source PDF with the new Read-in Excel |
| UC-05 | Export a QC report for records |
| UC-07 | QC an amendment update: compare the new amended PDF with the updated Read-in Excel |

Use case IDs are kept from earlier versions for traceability.

---

## 4. Input Data Structure

### 4.1 Excel – General sheet

The tool does **not** read or use the General sheet.

### 4.2 Excel – Standards sheet

#### 4.2.1 Columns used

| Column | Role in the tool |
|---|---|
| Position | **Sort key only.** Rows are sorted by Position before the rebuild. Not compared as text |
| Chapter | Repeated heading (level 1) |
| Chapter Title | Repeated heading (level 1) |
| Subchapter | Repeated heading (level 2) |
| Subpoint / Paragraph Subpoint / AMC subpoint | Treated as **one Subpoint field**. Used as the alignment anchor |
| **Standard** | Main body text |
| **In Front Of All** | Repeated lead-in text, written once before its group |
| **After All** | Repeated closing text, written once after its group |

#### 4.2.2 Columns ignored

Regulation ID, Paragraph ID, GM ref, Tag, Origin Document, Valid From, Valid Until, Predecessor ID, and any other column not listed in 4.2.1.

#### 4.2.3 Position order

- Position values are **not always sequential**. Gaps (e.g. 10, 20, 35) are normal and are not reported.
- Rows can be in any order in the sheet.
- The tool **always runs its own sort** by numeric Position before the rebuild.

### 4.3 Known structure issues

- Chapter, Chapter Title, and Subchapter repeat on every row but appear **once** in the PDF.
- In Front Of All and After All repeat on every row of a group but appear **once** in the PDF.
- Standard often starts with the Subpoint number.
- **Tables are not in the Read-in Excel.** In this release, PDF table text shows as Missing blocks (see FR-PDF-14).
- Line breaks inside a paragraph are different in the PDF and the Excel (see 6.8).
- Cells can contain \_x000D\_, CR/LF, non-breaking spaces, and typographic quotes.

### 4.4 Excel file sources

The tool reads every Excel file with the **same rules**. It does not split, move, or remove text between columns.

| Item | Read-in File | DB export |
|---|---|---|
| In Front Of All / After All columns | Filled where needed | Filled or empty |
| Lead-in text location | Separate column | Separate column, **or** start of Standard |
| Closing text location | Separate column | Separate column, **or** end of Standard |
| Joiner (when joined) | – | One LF (\n) |
| Effect on comparison | Normal | Lead-in / closing text inside Standard shows as **Added** text, once per row, marked **Repeated ×N (PDF ×M)** by the duplication counter (6.12) |

---

## 5. Comparison Flow

The tool has **one comparison mode: PDF (left) ↔ Excel (right)**.

1. Load PDF + Excel (read-only).
2. Excel: read used columns → sort by Position → duplicate consistency check → rebuild text + character map.
3. PDF: extract text layer and character boxes → handle generated characters → remove headers/footers/page numbers → remove ignored pages → group lines into blocks.
4. Align blocks (Subpoint / heading anchors).
5. Run a word diff, then a character diff, inside each aligned block pair.
6. Apply the line-break setting and classify differences.
7. Calculate the duplication counter for each difference (6.12).
8. Show results → export the XLSX report.

---

## 6. Functional Requirements (Core)

Priority: **M** = Must, **S** = Should. All requirements in Section 6 are Core.
IDs are kept from v0.5.2. Removed IDs are not reused. New IDs are marked **(v0.6)**.

### 6.1 File Input

| ID | Requirement | Priority |
|---|---|---|
| FR-IN-01 | The user selects a PDF file and an Excel file on the local machine with a file picker or a full local path. | M |
| FR-IN-02 | The tool opens all files **read-only** and never writes to them. | M |
| FR-IN-03 | The tool supports .xlsx. .xlsm is opened without running macros. | M |
| FR-IN-04 | The tool supports **text-based PDFs only**. | M |

### 6.2 Excel Parsing

| ID | Requirement | Priority |
|---|---|---|
| FR-XL-01 | Read only the Standards sheet. Find columns by **header name** (ignore case and extra spaces). | M |
| FR-XL-02 | If a required column (4.2.1) is missing, stop and list the missing columns. | M |
| FR-XL-03 | Read **raw cell values**, not formatted display values. | M |
| FR-XL-05 | Keep all characters exactly as stored, including \_x000D\_, CR, LF, tab, NBSP, and zero-width characters. | M |
| FR-XL-08 | Report **duplicate**, **empty**, or **non-numeric** Position values as errors. Gaps are not errors. | M |
| FR-XL-09 | Show empty Standard cells as warnings. | S |
| FR-XL-11 | Ignore all columns not listed in 4.2.1. | M |
| FR-XL-12 | Merge the Subpoint, Paragraph Subpoint, and AMC subpoint columns into one Subpoint value. If two columns have different values in the same row, show a warning. | M |
| FR-XL-13 | Use the same parsing and rebuild rules for every Excel file. Do not split Standard. Do not move text into or out of In Front Of All / After All. | M |
| FR-XL-14 | If Subchapter is empty, show a warning and put the row under **"No subchapter"** in its Chapter. Do not write a Subchapter heading for these rows. | M |

### 6.3 Excel Document Rebuild

| ID | Requirement | Priority |
|---|---|---|
| FR-RB-01 | Sort rows by numeric Position (ascending) before the rebuild. If two rows have the same Position, keep sheet row order and report the error (FR-XL-08). | M |
| FR-RB-02 | Write Chapter + Chapter Title **only when the value changes**. | M |
| FR-RB-03 | Write Subchapter only when it changes. | M |
| FR-RB-04 | Write In Front Of All **once, before the first row** of its group. | M |
| FR-RB-05 | Write After All **once, after the last row** of its group. | M |
| FR-RB-06 | Write Standard for every row. | M |
| FR-RB-07 | If **any line** of Standard starts with the Subpoint value, do not write the Subpoint again. A line is the text between two line breaks, with leading spaces removed. | M |
| FR-RB-08 | Field order and separators use built-in defaults (Appendix 17.1), stored in `config.py`. | M |
| FR-RB-09 | Build a **character map**: each rebuilt character maps to {cell address, row, column, character offset, Position}. | M |
| FR-RB-10 | Characters added by the rebuild (separators, line breaks) are marked "synthetic" and never reported as cell content. | M |

**Group definition (FR-RB-04 / 05):** a run of rows that are next to each other after the Position sort and have the same Subchapter and the same In Front Of All / After All value.

### 6.4 Duplicate Consistency Check

| ID | Requirement | Priority |
|---|---|---|
| FR-DC-01 | Before the PDF diff, compare all copies of each repeated field inside its group, character by character. | M |
| FR-DC-02 | If a copy is different, report **"Inconsistent duplicate"** with cell address, Position, majority value, different value, and the character difference. | M |
| FR-DC-04 | Report a Chapter Title that has 2 or more different values for the same Chapter. | M |
| FR-DC-06 | The rebuild uses the majority value and marks the block, so the problem is not hidden. | M |
| FR-DC-07 **(updated v0.6)** | Duplicate check results show in the warnings panel (UI-12) and in the XLSX report. | M |

### 6.5 PDF Text Extraction

| ID | Requirement | Priority |
|---|---|---|
| FR-PDF-01 | Extract text from the PDF text layer with **pypdfium2**, with page number and box for each character. | M |
| FR-PDF-02 | Keep characters exactly as extracted (no normalisation in strict mode). | M |
| FR-PDF-03 | Mark pages with no text layer **"Scanned page – not compared"**. Never skip silently. | M |
| FR-PDF-04 | Remove headers, footers, and page numbers with built-in automatic detection (FR-PDF-13). The user can view all removed text. | M |
| FR-PDF-05 | Group lines into blocks. Keep line ends inside a block as line breaks (6.8). | M |
| FR-PDF-09 | The user can enter page ranges to ignore (e.g. cover, contents, table pages) in the UI. | M |
| FR-PDF-13 | Automatic header/footer detection: text in the top or bottom area that repeats on most pages, and page number patterns (e.g. Page 3 of 40, - 3 -, a single number). | M |
| FR-PDF-14 **(updated v0.6)** | Tables are not detected in this release. PDF table text shows as Missing blocks. The user can add table pages to the ignored page ranges (FR-PDF-09). | M |
| FR-PDF-15 | Store the box of each character in **PDF user space** (origin bottom-left, unrotated page), with page size, rotation, MediaBox, and CropBox. Use `PdfTextPage.get_charbox(index, loose=False)`. Store boxes in Core even when Page view is off, so Page view needs no backend change. | M |
| FR-PDF-16 | Detect generated characters with `FPDFText_IsGenerated`. A CRLF pair counts as **one** line break. A generated line break is a line break (6.8). A generated space is marked "generated space" in the detail panel. A generated character uses the box of the previous character. | M |
| FR-PDF-18 | Open each PDF one time per run and close all PDFium objects after use. Run all PDFium calls on one worker thread. | M |
| FR-PDF-20 **(v0.6)** | Detect rotated pages (/Rotate 90, 180, 270) and show a **"Rotated page – check reading order"** warning for each. Line re-sort is deferred (Section 14, item 6). | M |

### 6.6 Alignment

| ID | Requirement | Priority |
|---|---|---|
| FR-AL-01 | Align **blocks** between PDF and Excel using anchors (Subpoint, Chapter and Subchapter heading numbers). | M |
| FR-AL-02 | If no anchor is available, align with a text-similarity match. | M |
| FR-AL-03 | A missing or extra block must not move the alignment of the blocks after it. | M |
| FR-AL-04 | Report blocks on one side only as **Missing** / **Extra**. | M |

### 6.7 Character-Level Diff

| ID | Requirement | Priority |
|---|---|---|
| FR-CD-01 | Run a **two-stage diff** in each aligned block pair: (1) word diff, then (2) character diff inside each changed word run. Compare the whitespace between matched words separately. | M |
| FR-CD-02 | Compare **Unicode code points**, not visual glyphs. | M |
| FR-CD-03 | **Strict mode is the default.** No normalisation of case, space, quotes, or Unicode form. The only exception is the line-break setting (6.8). | M |
| FR-CD-04 | Use a **minimal** diff (Myers O(ND) or diff-match-patch) at both stages. If Python difflib is used as a fallback, set `autojunk=False`. | M |
| FR-CD-05 | Show the code point and Unicode name of every changed character. | M |
| FR-CD-06 | Show invisible characters as visible symbols (6.7.1). | M |
| FR-CD-07 | Every difference keeps a link to both sources: PDF page and box, Excel cell, character offset, and Position. | M |
| FR-CD-08 | Show word-level and character-level highlights together (light colour for the word, strong colour for the characters). | S |

#### 6.7.1 Invisible character display

| Character | Code point | Display |
|---|---|---|
| Space | U+0020 | · (only in "show invisible" mode) |
| No-break space | U+00A0 | ⍽ |
| Tab | U+0009 | → |
| Carriage return / \_x000D\_ | U+000D | ¤ |
| Line feed | U+000A | ¶ |
| Line separator | U+2028 | [LS] |
| Zero-width space | U+200B | [ZW] |
| Soft hyphen | U+00AD | [SHY] |
| Non-breaking hyphen | U+2011 | [NBH] |

### 6.8 Line Break Handling

| ID | Requirement | Priority |
|---|---|---|
| FR-LB-01 **(updated v0.6)** | The **Line breaks** setting has 2 options: **Strict** and **Off**. The Minor option is deferred (Section 14, item 2). | M |
| FR-LB-02 | The setting applies **only to in-paragraph line breaks**. Breaks between blocks are not affected. | M |
| FR-LB-03 | **Strict:** every line-break difference is reported as a "Line break" difference. | M |
| FR-LB-05 | **Off:** an in-paragraph line break (with spaces next to it) is treated as **one space**. No difference is reported when the other side has one space at that point. | M |
| FR-LB-06 | In **Off**, a line break against **no character** is still reported (missing space, e.g. word¶word vs wordword). | M |
| FR-LB-07 | In **Off**, the summary shows **"Line breaks ignored: N"**. | M |
| FR-LB-08 **(updated v0.6)** | A line-end hyphen (regu-¶lation vs regulation) is always reported as a difference, in both settings. | M |
| FR-LB-09 | All line-break types count as line breaks: CR, LF, CRLF, \_x000D\_, U+2028. | M |
| FR-LB-10 **(updated v0.6)** | Default setting: **Off**. The user can change it at any time; results update without reloading files. | M |
| FR-LB-11 | The selected setting is shown in the UI header and in the report. | M |

Example:

| PDF | Excel | Strict | Off |
|---|---|---|---|
| holder of a¶licence | holder of a licence | Line break | No difference, counted as ignored |
| holder of a¶licence | holder of a¤¶licence | Line break | No difference, counted as ignored |
| holder of a¶licence | holder of alicence | Line break | Missing space – shown |

### 6.9 Difference Classification

| ID | Requirement | Priority |
|---|---|---|
| FR-CL-01 | Classify every difference with one type (6.9.1). | M |
| FR-CL-04 **(updated v0.6)** | The report shows the line-break setting and the ignored page ranges. | M |

Type filter toggles are deferred (Section 14, item 3). The difference list can be sorted by type (UI-21).

#### 6.9.1 Difference types

| Type | Example |
|---|---|
| Text | licence → license |
| Quote variant | ' ↔ ’ ↔ ‘, " ↔ “ ” |
| Dash variant | - ↔ – ↔ — ↔ ‑ |
| Space variant | space ↔ NBSP, double space, trailing space |
| Line break | CR/LF/\_x000D\_ difference (Strict setting) |
| Unicode form | NFC ↔ NFD |
| Case | Licence ↔ licence |
| Punctuation | missing . , ; |
| Number | 60 → 65 |
| Missing block | Paragraph in the PDF, not in the Excel |
| Extra block | Paragraph in the Excel, not in the PDF |
| Inconsistent duplicate | See 6.4 |

### 6.10 Review Recording (simplified v0.6)

In-tool review statuses and comments are deferred (Section 14, item 1). In this release, the reviewer records the result in the exported XLSX report (FR-RP-01).

### 6.11 Reporting and Export

| ID | Requirement | Priority |
|---|---|---|
| FR-RP-01 **(updated v0.6)** | Export a QC report as **XLSX**: one row per difference with ID, type, block anchor, Excel cell, Position, PDF page, PDF text, Excel text, and code points. Add empty **Status** (data validation list: Open / OK / Error / Ignored) and **Comment** columns for the reviewer. | M |
| FR-RP-03 **(updated v0.6)** | The report header includes: file names, tool version, line-break setting, ignored page ranges, date, reviewer name (typed by the user before export), and summary counts (total differences, by type, scanned pages, rotated pages, line breaks ignored, warnings). | M |
| FR-RP-06 **(v0.6)** | Warnings (UI-12) and duplicate check results (6.4) are exported on a separate **Warnings** sheet. | M |

### 6.12 Duplication Counter (moved to Core v0.6.1)

The Excel text can have repeated lines that occur once in the PDF (e.g. a lead-in copied into every Standard cell of a DB export). The comparison does **not** remove or split this text. The duplication counter tells the reviewer that a difference is caused by repeated text.

| ID | Requirement | Priority |
|---|---|---|
| FR-DU-01 | For every difference that has Excel-side text (Added, Text, Extra block), find the full Excel line that contains it. A line is the text between two line breaks, after the line-break setting (6.8) is applied. | M |
| FR-DU-02 | Count how many times that line occurs in the rebuilt Excel text (**Excel ×N**) and in the PDF text (**PDF ×M**), in the same scope. Count only exact matches (all code points the same). | M |
| FR-DU-03 | Scope: the same Subchapter. Rows with no Subchapter use the **"No subchapter"** scope of their Chapter (FR-XL-14). | M |
| FR-DU-04 | Do not count lines shorter than **15 characters** (e.g. (a), Note:). The value is in `config.py`. | M |
| FR-DU-05 | Mark the difference as **Repeated** when Excel ×N is more than PDF ×M **and N is 2 or more**. Show the counter as ×N (PDF ×M). | M |
| FR-DU-06 | The counter is information only. It does not change the difference type, does not hide the difference, and does not change the total count (1.3). | M |
| FR-DU-07 | Give all differences with the same repeated line the same **repeat group ID** (R1, R2, …). For each copy, store the cell address, column (Standard / In Front Of All / After All), and Position. | M |
| FR-DU-09 **(updated v0.6.1)** | The difference list can be **sorted by repeat group** (UI-21), so all differences of one group are together. A Repeated filter toggle is deferred (Section 14, item 3). | M |
| FR-DU-11 | The summary shows **"Repeated lines: L (differences: D)"**. | M |
| FR-DU-12 **(v0.6.1)** | The XLSX report has **Repeat group**, **Excel ×N**, and **PDF ×M** columns for each difference. The reviewer can filter these columns in Excel. | M |

Example (lead-in inside Standard, Subchapter 4):

| Difference | Excel line | Counter | Copies |
|---|---|---|---|
| Added | 4.3 Typical matters… | ×6 (PDF ×1), group R1 | Standards!N20, N21, N22, N23, N24, N25 (column Standard) |
| Added | 4.3 Typical matters… + trailing space | ×1 (PDF ×0), not Repeated | Standards!N26 (column Standard) |

**Note:** In the second row, the copy with the trailing space is a separate normal difference. The near-match count (near ×K) that links it to group R1 is deferred (Section 14, item 9).

---

## 7. User Interface Requirements (Core)

### 7.1 Layout (Core, modules off)

```
┌──────────────────────────────────────────────────────────────────────────┐
│ [Open PDF] [Open Excel]  Ignore pages:[1-3]  [Run]  Reviewer:[____] [Export]│
│ Show invisible ☑   Line breaks:[Off ▼]                                     │
│ Summary: 21 total | 12 text | 4 quote | 2 space | 3 dup | LB ignored: 38  │
│          0 scanned pages | 1 rotated page | 5 warnings | Repeated lines: 3 (14)│
├────────────────────────────────────┬─────────────────────────────────────┤
│ PDF (Text)  page 7                 │ Excel (rebuilt)                     │
│ 1.2.1 The minimum age for…         │ 1.2.1 The minimum age for…          │
│ Private Pilot’s Licence            │ Private Pilot's Licence             │
│              ▲ U+2019              │              ▲ U+0027               │
│                                    │ Standards!O12 | Pos 120             │
├────────────────────────────────────┴─────────────────────────────────────┤
│ Differences   Sort:[Order ▼]                                             │
│ #1 Quote  1.2.1  p7  Standards!O12   ’ → '                               │
│ #2 Text   1.3.2  p8  Standards!O20   "valid" → "vaild"                   │
│ #3 Added  4.3.1  p9  Standards!N20   ×6 (PDF ×1)  Repeat group R1         │
├──────────────────────────────────────────────────────────────────────────┤
│ Detail: U+2019 RIGHT SINGLE QUOTATION MARK  vs  U+0027 APOSTROPHE  [Copy cell]│
├──────────────────────────────────────────────────────────────────────────┤
│ Warnings (5)                                                             │
└──────────────────────────────────────────────────────────────────────────┘
```

When an optional module is on, the header shows its switch:
- OPT-RV: `Excel view: (•) Group ( ) Row`
- OPT-PV: `PDF view: (•) Text ( ) Page   Zoom:[100% ▼]`

### 7.2 Behaviour

| ID | Requirement | Priority |
|---|---|---|
| UI-01 **(updated v0.6)** | Side-by-side panes (PDF left, Excel right) in Text view. Aligned blocks stay level; a blank space is added on the other side for a missing block. Clicking a difference scrolls both panes to it. Full synchronised scroll is deferred (Section 14, item 4). | M |
| UI-03 **(updated v0.6)** | The Excel pane shows **Group view** (rebuilt text, each heading once). Row view is optional module OPT-RV (Section 8). | M |
| UI-04 | Clicking a difference shows the detail panel: code points, Unicode names, Excel cell, Position, PDF page. | M |
| UI-06 | A **Copy cell address** button copies e.g. Standards!O12. | M |
| UI-08 **(updated v0.6)** | Colours: deleted = red, added = green, changed = amber, inconsistent duplicate = purple, synthetic text = grey. Each colour also has a second marker (underline / icon) for colour-blind users. | M |
| UI-12 | A warnings panel lists: missing columns, Position errors, Subpoint column conflicts, rows with no Subchapter, empty Standard cells, inconsistent duplicates, scanned pages, rotated pages, removed headers/footers. | M |
| UI-15 | The **Line breaks** dropdown (Strict / Off) is always visible. Changing it updates results immediately. | M |
| UI-19 **(v0.6)** | Both panes use a monospace font. | M |
| UI-20 **(v0.6)** | A busy indicator shows while the comparison runs. | M |
| UI-21 **(updated v0.6.1)** | The difference list can be sorted by order, type, block anchor, PDF page, Excel cell, or repeat group. Keys N / P go to the next / previous difference. | M |
| UI-18 **(Core v0.6.1)** | A Repeated difference shows a counter badge **×N (PDF ×M)** and its repeat group ID in the list. The detail panel lists all copies (FR-DU-07); clicking a copy goes to that cell (Text view, or Row view when OPT-RV is on). | M |
| UI-22 **(v0.6)** | Before the user starts a new comparison or closes the tool, warn if the results are not exported. | S |

---

## 8. Optional Module: Row View (OPT-RV)

Row view shows the Excel data the way it is stored: one line per row, with repeated values. Use it to find which row and column has a problem.

| ID | Requirement | Priority |
|---|---|---|
| UI-RV-01 | When OPT-RV is on, a **Group / Row** switch shows in the Excel pane header. Default: Group. The selected difference stays the same when the view changes. | M |
| UI-RV-02 | Row view shows every row in **Position order** with: Position, cell row number, Chapter, Chapter Title, Subchapter, Subpoint, In Front Of All, Standard, After All. Repeated values are shown on every row. | M |
| UI-RV-03 | Row view uses the character map (FR-RB-09) to draw each difference on the correct cell and characters. Synthetic characters are not shown. | M |
| UI-RV-04 | Inconsistent duplicates (6.4) are highlighted on the different cell in purple. | M |
| UI-RV-08 **(v0.6.1)** | Cells that hold a copy of a repeated line show the repeat group ID (e.g. R1) next to the cell. | S |
| UI-RV-05 | Selecting a difference in the list scrolls Row view to that row and highlights the cell. | M |
| UI-RV-06 | Rows with no Subchapter show "No subchapter" in the Subchapter column. | S |
| UI-RV-07 | Long cells fold to 3 lines, with an "expand" option. | S |

---

## 9. Optional Module: PDF Page View (OPT-PV)

Page view shows the original PDF page, not extracted text. The tool draws highlight boxes on a layer above the page. The tool does not change the PDF file.

- **Page view:** confirm the location and context of a difference on the real page.
- **Text view:** check invisible characters and exact code points.

### 9.1 Requirements

| ID | Requirement | Priority |
|---|---|---|
| UI-PV-01 | When OPT-PV is on, a **Text / Page** switch shows in the PDF pane header. Default: Text. The selected difference stays the same when the view changes. | M |
| UI-PV-02 | Render each page in the browser with **PDF.js** (canvas). Draw highlights in a separate HTML or SVG overlay. The backend sends the PDF to the browser from 127.0.0.1 as read-only bytes. | M |
| UI-PV-03 | Convert each pypdfium2 box (FR-PDF-15) to screen coordinates with `viewport.convertToViewportRectangle([x0, y0, x1, y1])`. Normalise the result (min/max). Add 1–2 px padding. Do not write a separate conversion. | M |
| UI-PV-04 | Show each difference type as in 9.2. Use the colours and second markers of UI-08. | M |
| UI-PV-05 | Clicking a highlight selects the difference. Selecting a difference (or N / P) goes to its page, scrolls the box into view, and flashes it for 1 second. | M |
| UI-PV-06 | Hovering a highlight shows a tooltip: type, PDF text and code points, Excel text and code points, Excel cell. | M |
| UI-PV-08 **(updated v0.6)** | Highlights follow the line-break setting. Changing it updates the highlights without rendering the page again. | M |
| UI-PV-09 | Page navigation: previous / next page, go to page number, zoom (fit width, fit page, 50–400 %). | M |
| UI-PV-10 | Render pages only when they come into view. Keep a limited number of rendered pages in memory. | M |
| UI-PV-12 | For invisible-character differences, the tooltip says the character cannot be seen on the page and shows a **Show in Text view** link. | M |
| UI-PV-13 | The difference list works the same in both views. | M |
| UI-PV-14 | The PDF.js text layer is **not used** for comparison or highlights. Only pypdfium2 text is compared. | M |
| UI-PV-15 **(v0.6)** | If PDF.js cannot open a PDF, Page view shows "Page view not available for this file". Text view and Core results still work. | M |

Synchronised scroll in Page view (old UI-PV-07) and status-based highlight styles (old UI-PV-11) are deferred.

### 9.2 Display by difference type

| Difference | Display on the page |
|---|---|
| Text, quote, dash, number, case, space variant | Amber box on the changed glyphs, light amber box on the full word |
| Missing block (PDF only) | Red box around the full paragraph |
| Added / extra in Excel (no PDF glyph) | Thin green bar at the next PDF character, full line height, drawn as a box in PDF user space. Tooltip shows the added Excel text |
| Repeated difference (6.12) | Insertion bar with the counter badge ×N (PDF ×M) in the left margin of the line |
| Line break / missing space | Marker at the end of the PDF line |
| Removed header / footer | Grey striped overlay |
| Ignored page range | Full-page grey overlay, label "Ignored page" |
| Scanned page | Full-page grey overlay, label "Scanned page – not compared" |

### 9.3 Page view limits

| Limit | Handling |
|---|---|
| Invisible characters cannot be seen on a page | Highlight the position only. Show code points in the tooltip (UI-PV-12) |
| A ligature is one glyph for 2 or more characters | Highlight the full glyph |
| Poor font metrics in some PDFs | Box padding (UI-PV-03) |
| Rotated page | Highlights are correct (PDF.js viewport). Reading order can be wrong: see FR-PDF-20 warning |

### 9.4 PDF Engine Split (OPT-PV)

| Task | Engine | Where |
|---|---|---|
| Text, character boxes, generated characters | pypdfium2 | Backend (Core) |
| Page rendering, zoom, lazy loading | PDF.js | Browser (OPT-PV) |
| Box to screen conversion | PDF.js viewport | Browser (OPT-PV) |
| Highlights | HTML/SVG overlay | Browser (OPT-PV) |

Rules:
1. **One source of truth for text:** only pypdfium2 text is compared.
2. **One coordinate space:** all boxes are in PDF user space, origin bottom-left.
3. **Same page box:** PDF.js and PDFium both use the CropBox (confirmed in the v0.5.2 prototype, finding F-3).

---

## 10. Non-Functional Requirements

### 10.1 Security and Privacy

| ID | Requirement |
|---|---|
| NFR-SEC-01 | Runs fully on the local machine. The web server binds to **127.0.0.1 only**. |
| NFR-SEC-02 | No upload of files or data to any internet or cloud service. No telemetry. |
| NFR-SEC-03 | All file access is read-only. |
| NFR-SEC-04 | No macros or embedded scripts are run from source files. |
| NFR-SEC-05 | Must pass company IT security review for local tools. |
| NFR-SEC-06 (OPT-PV) | pdfjs-dist is bundled with a pinned version and loads from 127.0.0.1. No CDN. Use the core library only, with `isEvalSupported: false`. |
| NFR-SEC-07 | All third-party libraries have permissive licences (Apache-2.0, BSD, MIT). **No AGPL or GPL libraries** (e.g. PyMuPDF). A THIRD_PARTY_NOTICES file lists each library, version, and licence. |

### 10.2 Performance

| ID | Requirement |
|---|---|
| NFR-PF-01 **(updated v0.6)** | Measure and log the run time for each stage. No target in this release (target deferred). |
| NFR-PF-02 | UI navigation stays responsive (< 200 ms) during review. |
| NFR-PF-04 | Cells with more than 32,000 characters are handled without crashing. |
| NFR-PF-05 | Changing the line-break setting updates results in under 3 seconds without re-reading files. |
| NFR-PF-06 (OPT-PV) | A page shows with its highlights in under 1 second at 100 % zoom. |
| NFR-PF-07 (OPT-PV) | The backend sends character boxes per page, only for pages in view and only for boxes with a difference. |

### 10.3 Accuracy

| ID | Requirement |
|---|---|
| NFR-AC-01 | In strict mode, **0 missed differences** on the seeded test set (Section 12). |
| NFR-AC-02 | Every difference traces to an exact source location. |
| NFR-AC-03 | If the tool cannot compare a part (scanned page, ignored page, parse error), it says so on screen and in the report. |

### 10.4 Usability and Maintenance

| ID | Requirement |
|---|---|
| NFR-US-01 **(updated v0.6)** | Runs on Windows 10/11 from a Python 3.11+ install with one start command (e.g. `python run.py`), which opens the browser at http://127.0.0.1:<port>. Packaging is deferred. |
| NFR-US-03 **(updated v0.6)** | Works out of the box. All default rules and module switches are in one `config.py` file. |
| NFR-US-04 | Logs go to a local log folder. Logs do not contain full regulation text unless debug mode is on. |
| NFR-US-05 | Version number shown in the UI and in every report. |

### 10.5 Technology

| Layer | Option | Licence | Part |
|---|---|---|---|
| Backend | Python 3.11+, FastAPI | MIT | Core |
| Excel | openpyxl (read-only, raw values) | MIT | Core |
| PDF text + character boxes | pypdfium2 (PDFium) | Apache-2.0 / BSD-3-Clause | Core |
| Diff | Myers minimal diff (own code) or diff-match-patch; difflib (`autojunk=False`) as fallback only | – / Apache-2.0 / PSF | Core |
| Report | openpyxl (write new file only) | MIT | Core |
| Frontend | Plain HTML/JS | – | Core |
| PDF page view | pdfjs-dist (bundled) | Apache-2.0 | OPT-PV |

Not used in this release: pdfplumber (table detection deferred), PyInstaller (packaging deferred), PyMuPDF (AGPL).

---

## 11. Edge Cases

| # | Case | Expected behaviour |
|---|---|---|
| 1 | Same Chapter Title with a different quote in one row | Inconsistent duplicate (FR-DC-02) |
| 2 | Standard (or a line of Standard) starts with the Subpoint | Subpoint not repeated (FR-RB-07) |
| 3 | Subpoint missing in Standard text | Text difference |
| 4 | Paragraph missing in Excel | Missing block; following blocks stay aligned |
| 5 | Duplicate Position value | Error before comparing (FR-XL-08) |
| 6 | Position gaps or rows not in sheet order | Tool sorts by Position. No error |
| 7 | Table in the PDF | Missing block. User adds the page to ignored ranges (FR-PDF-14) |
| 8 | Line-end hyphen in PDF | Reported as a difference (FR-LB-08) |
| 9 | Different line breaks inside the same paragraph | Line-break setting (6.8) |
| 10 | \_x000D\_ inside a cell | Shown as ¤. Treated as a line break |
| 11 | Line break in PDF, no space in Excel | Missing space in both settings (FR-LB-06) |
| 12 | Lists with —, (a), (i) | Compared as text; dash variants classified |
| 13 | Footnotes | Compared as text where extracted |
| 14 | Scanned page | Marked not compared (FR-PDF-03) |
| 15 | Empty In Front Of All / After All | Nothing written |
| 16 | GM / AMC rows between rules | Rebuilt in Position order |
| 17 | Different Subpoint values in two Subpoint columns | Warning (FR-XL-12) |
| 18 | Header/footer not detected | Shows as Extra text; user adds ignored pages |
| 19 | Lead-in or closing text inside Standard (DB export) | Shows as Added, Repeated ×N (FR-DU-05) |
| 20 | Row with empty Subchapter | Warning; row under "No subchapter"; counter uses "No subchapter" scope |
| 26 **(v0.6.1)** | Lead-in in In Front Of All **and** inside Standard | Written once from the column and once per row from Standard. Repeated ×N+1 |
| 27 **(v0.6.1)** | Lead-in inside Standard for a group with one row | Same layout as the PDF. No difference, no counter |
| 28 **(v0.6.1)** | Repeated line shorter than 15 characters | Not counted (FR-DU-04). Compared as normal |
| 29 **(v0.6.1)** | One copy of a repeated lead-in has a different character | Separate normal difference, not in the repeat group |
| 21 | PDFium adds a space or line break | Not compared as a real character (FR-PDF-16) |
| 22 | Rotated page | Warning (FR-PDF-20). OPT-PV highlights are still on the correct glyphs |
| 23 | (OPT-PV) Invisible-character difference | Position highlighted; tooltip links to Text view |
| 24 | (OPT-PV) Difference spans two lines or pages | One box per line; all boxes flash together |
| 25 | (OPT-PV) PDF.js cannot open the PDF | Message in Page view; Core still works |

---

## 12. Testing and Acceptance

### 12.1 Seeded Test Set

Create a reference PDF + Excel pair that matches exactly. Then create copies with known errors:

| Seed | Error |
|---|---|
| S-01 | One ' changed to ’ |
| S-02 | One space changed to NBSP |
| S-03 | One \_x000D\_ added inside a paragraph |
| S-04 | One letter swapped (valid → vaild) |
| S-05 | One Subpoint row deleted |
| S-06 | One row added |
| S-07 | One Chapter Title copy with a different character |
| S-08 | One In Front Of All copy with a trailing space |
| S-09 | Two rows with swapped Position values |
| S-10 | One number changed (60 → 65) |
| S-11 | One - changed to – |
| S-12 | One zero-width space added |
| S-13 | Rows shuffled in the sheet, Position gaps (content unchanged) |
| S-14 | One space removed at a PDF line-break point (a¶licence vs alicence) |
| S-15 | DB export copy of the reference pair: In Front Of All empty, lead-in joined into Standard with LF |
| S-16 | S-15 with one lead-in copy that has a trailing space |

### 12.2 Acceptance Criteria – Core

| ID | Criterion |
|---|---|
| AC-01 | All seeds S-01 to S-12 and S-14 are found in strict mode, with the correct type and cell address. |
| AC-02 | The reference pair shows **0 differences** with line breaks **Off** (no false results from repeated fields). |
| AC-03 | Missing / added rows do not cause extra differences in the blocks after them. |
| AC-04 | Source files are not changed (same checksum before and after use). |
| AC-05 | No network traffic outside 127.0.0.1. |
| AC-07 | The XLSX report matches the on-screen difference list (same count). |
| AC-08 | Tested on real files from at least 2 authorities (e.g. HK CAD and UK CAA Part-FCL). |
| AC-09 | S-13 shows **0 differences**. |
| AC-10 | Reference pair with **Off** shows a correct "Line breaks ignored" count. S-14 is still found. |
| AC-12 | Headers/footers are removed with built-in rules only. |
| AC-21 | Licence scan shows no AGPL or GPL runtime library. THIRD_PARTY_NOTICES is complete. |
| AC-22 | Reference pair: no difference is caused by a PDFium generated character. |
| AC-14 | S-15: every difference is Repeated with the correct counter, and the Repeated lines count equals the number of lead-in groups with 2 or more rows. No other differences. |
| AC-16 | The counter does not change the total count or the difference types (FR-DU-06). |
| AC-25 | S-15: lead-in differences show the correct ×N (PDF ×1). A changed line that is in the Excel once (×1) is not marked Repeated (FR-DU-05). |
| AC-31 **(v0.6.1)** | S-16: the copy with the trailing space is a separate normal difference with the correct cell address. The other copies stay in their repeat group. |
| AC-32 **(v0.6.1)** | The XLSX report repeat columns (FR-DU-12) match the on-screen counters. |
| AC-26 **(v0.6)** | Core passes all criteria above with both optional modules **off**. |

### 12.3 Acceptance Criteria – OPT-RV

| ID | Criterion |
|---|---|
| AC-27 **(v0.6)** | Row view shows every row in Position order (S-13). |
| AC-28 **(v0.6)** | For seeds S-01 to S-12, each difference is on the correct cell and characters in Row view. |
| AC-29 **(v0.6)** | Turning OPT-RV on or off does not change the difference count. |

### 12.4 Acceptance Criteria – OPT-PV

| ID | Criterion |
|---|---|
| AC-17 | Every difference from S-01 to S-12 and S-14 has a highlight on the correct glyph or position. |
| AC-18 | Highlights are correct on a rotated page and on a page with a CropBox. |
| AC-19 | The number of highlights on a page matches the difference count for that page. |
| AC-20 | The PDF file is the same before and after Page view use (same checksum). |
| AC-23 | Highlights are correct on a page with a MediaBox origin not at (0, 0). |
| AC-30 **(v0.6)** | Turning OPT-PV on or off does not change the difference count. |

### 12.5 Regression

Run the seeded test set automatically after every code change.

---

## 13. Out of Scope

- Editing, fixing, or saving PDF or Excel files.
- Automatic correction of differences.
- PDF ↔ PDF and Excel ↔ Excel comparison.
- Self-check mode (Excel only).
- Saving and reopening review sessions.
- Reading the General sheet.
- Scanned PDFs and OCR. Docling JSON input.
- Selecting or detecting the Excel file type.
- Splitting lead-in or closing text out of Standard.
- Comparing PDF tables.
- Network drive paths, recent file list, locked files, formula handling.
- Cloud hosting or multi-user use.
- .xls, .docx, or HTML sources.
- Translation or language checks.

---

## 14. Deferred Features (later versions)

These features are in v0.5.2 but are **not** in this release. Add them back in this order:

| # | Feature | v0.5.2 IDs | Release 1 workaround |
|---|---|---|---|
| 1 | In-tool review status and comments, and one status for a full repeat group | FR-RV-01, 02, 06, 07, 08, FR-DU-10 | Status and Comment columns in the XLSX report |
| 2 | Line-break **Minor** setting | FR-LB-04, AC-11 | Use Off |
| 3 | Type filter toggles, Repeated filter toggle, and hidden counts | FR-CL-02, CL-03, FR-DU-09 (filter) | Sort the list by type or repeat group; filter in the XLSX |
| 4 | Navigation tree and full synchronised scroll (Text and Page view) | UI-01 (full), UI-02, UI-PV-07, UI-PV-13 (tree) | Click a difference to scroll both panes |
| 5 | Table detection and exclusion (pdfplumber) | FR-PDF-12, UI-16, AC-13, Q-15 | Ignore table pages |
| 6 | Line re-sort on rotated pages | FR-PDF-19, AC-24 | Rotated page warning (FR-PDF-20) |
| 7 | Hyphen and ligature flags | FR-PDF-06, 08, 17 | Reported as normal differences |
| 8 | Moved block detection | FR-AL-05 | Shows as Missing + Extra |
| 9 | Near-match count (near ×K) for repeated lines | FR-DU-08, AC-15 | The changed copy shows as a separate normal difference (AC-31) |
| 10 | Extra duplicate checks (type label, broken groups) | FR-DC-03, DC-05 | None |
| 11 | HTML, TXT, and PDF exports | FR-RP-02, 04, 05, FR-RB-11 | XLSX only |
| 12 | UI extras: full shortcuts, search, block folding, font zoom, progress stages, Cancel | UI-07, 09, 10, 11, 13, 14 | N / P keys, busy indicator, monospace font |
| 13 | Status-based highlight styles in Page view | UI-PV-11 | None |
| 14 | Optional profiles (including counter scope override) | 6.12 of v0.5.2 (FR-CF-01 to 06) | Defaults in `config.py` |
| 15 | Packaging (.exe) and one-click start | NFR-US-01, 02 | Python start command |
| 16 | Performance targets | NFR-PF-01 | Measure and log only |

---

## 15. Delivery Phases

| Phase | Content | Exit criteria |
|---|---|---|
| 0 (done v0.5.2) | Spike: pypdfium2 boxes drawn on PDF.js | Engine choice confirmed |
| 1 – Core | Excel parsing, Position sort, rebuild, character map, duplicate check | Rebuilt text correct on S-07, S-08, S-09, S-13 |
| 2 – Core | PDF extraction, generated characters, header/footer removal, ignored pages, alignment | Blocks align on the reference pair |
| 3 – Core | Two-stage diff, line-break setting, classification, invisible characters, duplication counter | AC-01, AC-02, AC-03, AC-09, AC-10, AC-22, AC-14, AC-16, AC-25, AC-31 |
| 4 – Core | Text view UI, difference list (with counter badge), detail and warnings panels, XLSX report | AC-04, AC-05, AC-07, AC-12, AC-21, AC-26, AC-32. Test on real files (AC-08) |
| 5 – OPT-RV | Row view | AC-27 to AC-29 |
| 6 – OPT-PV | Page view | AC-17 to AC-20, AC-23, AC-30 |

Phases 5 and 6 are independent. Do them in either order.

---

## 16. Open Questions

| # | Question | Owner |
|---|---|---|
| Q-01 | Confirm the meaning of **In Front Of All** and **After All** and the exact group rule. | Regulation Dept |
| Q-02 | Should Chapter and Chapter Title be joined with a space or a line break? | Regulation Dept |
| Q-03 | Confirm the exact Subpoint column header names. Can more than one be filled in the same row? | Regulation Dept |
| Q-05 | Which normalisations, if any, are allowed in the final QC sign-off? | QC lead |
| Q-06 | Where should exported reports be saved? | Team lead |
| Q-07 | Can the tool run from a Python install on user laptops (packaging deferred)? | IT |
| Q-08 | Maximum expected file sizes (pages, rows)? | Regulation Dept |
| Q-09 **(updated v0.6)** | Is **Off** the correct default for the line-break setting in this release? | QC lead |
| Q-12 | If OPT-PV is built, should the default PDF view be Text or Page? | QC lead |
| Q-11 | Is 15 characters the correct minimum line length for the counter (FR-DU-04)? | QC lead |
| Q-14 | Confirm with Legal / IT that the libraries in 10.5 are approved. | IT / Legal |
| Q-16 **(v0.6)** | Which optional module is needed first: Row view or Page view? | QC lead |

---

## 17. Appendix

### 17.1 Rebuild example (built-in defaults)

Excel rows (sheet order, not sorted):

| Sheet row | Position | Chapter | Chapter Title | Subchapter | Subpoint | Standard |
|---|---|---|---|---|---|---|
| 2 | 30 | CHAPTER 1 | GENERAL INFORMATION | 1.2 AGE LIMITS | 1.2.1 | 1.2.1 The minimum age… |
| 3 | 10 | CHAPTER 1 | GENERAL INFORMATION | 1.1 INTRODUCTION | 1.1.1 | 1.1.1 The Air Navigation… |
| 4 | 15 | CHAPTER 1 | GENERAL INFORMATION | 1.1 INTRODUCTION | 1.1.2 | 1.1.2 This publication… |

After the sort (Position 10, 15, 30), rebuilt text:

```
CHAPTER 1 GENERAL INFORMATION        ← from Position 10 only
1.1 INTRODUCTION                     ← from Position 10 only
1.1.1 The Air Navigation…            ← Standards!O3, Pos 10 (Subpoint not repeated)
1.1.2 This publication…              ← Standards!O4, Pos 15
1.2 AGE LIMITS                       ← from Position 30 (value changed)
1.2.1 The minimum age…               ← Standards!O2, Pos 30
```

Built-in default order: Chapter + " " + Chapter Title (when changed) → Subchapter (when changed) → In Front Of All (group start) → Subpoint (only if no line of Standard starts with it) → Standard → After All (group end). Each item starts on a new line.

### 17.2 Difference record (data model)

```json
{
  "id": 17,
  "type": "quote_variant",
  "block_anchor": "1.2.1",
  "pdf":   { "page": 7, "bbox": [72.1, 410.2, 80.4, 421.0],
             "boxes": [[72.1, 410.2, 80.4, 421.0]],
             "page_size": [595.0, 842.0], "rotation": 0,
             "mediabox": [0, 0, 595.0, 842.0], "cropbox": [0, 0, 595.0, 842.0],
             "coord_origin": "pdf-user-space-bottom-left", "engine": "pypdfium2",
             "generated": false,
             "word_bbox": [41.6, 410.2, 96.3, 421.0],
             "insert_point": null,
             "text": "’", "codepoints": ["U+2019"] },
  "excel": { "cell": "Standards!O12", "offset": 154, "position": 120,
             "text": "'", "codepoints": ["U+0027"] },
  "context_pdf":   "Private Pilot’s Licence",
  "context_excel": "Private Pilot's Licence",
  "repeat": null
}
```

Repeated difference (lead-in inside Standard):

```json
"repeat": { "group_id": "R1", "excel_count": 6, "pdf_count": 1,
            "scope": "Subchapter: 4 APPLICATION",
            "copies": [ { "cell": "Standards!N20", "column": "Standard", "position": 200 },
                        { "cell": "Standards!N21", "column": "Standard", "position": 210 } ] }
```

- All boxes are [left, bottom, right, top] in PDF user space. Core stores them; only OPT-PV uses them.
- `insert_point` is [page, x, line_top, line_bottom] for Added differences with no PDF glyph; `bbox` is then null.
- `repeat` is null when the difference is not Repeated. `near_count` from v0.5.2 is removed (Section 14, item 9).
- The `status`, `comment`, `reviewer`, and `updated` fields from v0.5.2 are removed in this release (Section 14, item 1).

### 17.3 Character map entry

```json
{ "rebuilt_offset": 2051, "cell": "Standards!O12", "row": 12, "column": "Standard",
  "char_offset": 154, "position": 120, "synthetic": false }
```

---

## 18. Change Log

| Version | Date | Change |
|---|---|---|
| 0.1 – 0.5.2 | 16 Oct 2026 | See Regulation_DiffChecker_Requirements_v0.5.2.md |
| 0.6.0 | 16 Oct 2026 | First release scope defined: **Core** plus optional modules **Row view (OPT-RV)** and **Page view (OPT-PV)** (1.2). Moved 16 feature groups to Deferred (Section 14), with workarounds. Line breaks: Strict / Off only, default Off (FR-LB-01, LB-10). Tables not detected; table pages ignored by user (FR-PDF-14); pdfplumber removed from this release. Added FR-PDF-20 (rotated page warning), FR-RP-06 (Warnings sheet), UI-19 to UI-22, UI-RV-01 to 07, UI-PV-15, AC-26 to AC-30, Q-16. Updated FR-DC-07, FR-RP-01 (Status and Comment columns in the report), FR-RP-03, FR-LB-08, UI-01, UI-03, UI-08, UI-PV-08, NFR-PF-01, NFR-US-01, NFR-US-03. FR-PDF-15 boxes stay in Core so Page view needs no backend change. Data model: removed repeat and review fields. Delivery phases re-planned (Section 15). |
| 0.6.1 | 16 Oct 2026 | **Duplication counter moved to Core** (new 6.12): FR-DU-01 to 07, FR-DU-09 (sort by repeat group), FR-DU-11, new FR-DU-12 (repeat columns in XLSX report). UI-18 moved to Core; UI-21 sorts by repeat group; added UI-RV-08 and Page view Repeated display. Updated 2 (definitions), 4.4, 5 (flow step 7), 7.1 layout, edge case 19; added edge cases 26–29. Added seeds S-15, S-16 and AC-14, AC-16, AC-25, AC-31, AC-32. Q-11 back. Data model `repeat` field back (no `near_count`). Deferred list re-ordered: near ×K (FR-DU-08) and group status (FR-DU-10) stay deferred. Phases 3 and 4 updated. |
