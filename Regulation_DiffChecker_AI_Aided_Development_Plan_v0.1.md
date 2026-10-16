# Regulation Diff Checker – AI-Aided Development Plan

| Item | Value |
|---|---|
| Document | AI-Aided Development Plan |
| Based on | Regulation_DiffChecker_Requirements_v0.6.1.md (SRS), Regulation_DiffChecker_Development_Plan_v0.1.md |
| Version | 0.1 (Draft) |
| Date | 16 Oct 2026 |
| Owner | Regulation Department |

---

## 1. Purpose

This plan tells how **one developer** builds the first release with **AI as an assistant**. The developer writes, runs, and owns all code. The AI explains, suggests, drafts, and reviews.

### 1.1 AI-aided vs agentic

| Item | AI-aided (this plan) | Agentic (Development Plan v0.1, Part B) |
|---|---|---|
| Who writes code | Developer, with AI drafts | AI agent |
| Who runs code and tests | Developer | Agent |
| Work unit | One function or one small step | One task card |
| AI tools | Chat (e.g. M365 Copilot), IDE completion (e.g. GitHub Copilot) | Coding agent in the repository |
| Speed | Slower | Faster |
| Control and learning | High: developer understands every line | Lower: developer reviews results |
| Best for | Core logic, first release, learning the code base | Repeated, well-specified work after the core is stable |

### 1.2 When to use which

1. Use **AI-aided** for Phases 0–3 (data model, rebuild, alignment, diff, counter). Errors here cause missed differences.
2. Use **AI-aided or agentic** for Phase 4 (UI, report) and Phases 5–6 (optional modules), after the core is stable.

---

## 2. Rules

1. **You own the code.** Do not commit code that you cannot explain line by line.
2. **One small step at a time.** Ask the AI for one function or one test at a time, not a full module.
3. **Run everything.** AI output is a draft until you run it and the tests pass.
4. **Tests define "correct".** You write or check every test and every golden file. The AI can suggest test cases.
5. **SRS is the source of truth.** If the AI answer conflicts with the SRS, the SRS wins. If the SRS is not clear, record a decision in `docs/decisions.md`.
6. **Check library facts.** AI can invent function names and parameters (e.g. for pypdfium2, openpyxl, PDF.js). Check each call in the official documentation or with a small test.
7. **Data rule.** Do not paste real regulation text, real Excel content, or company data into an AI tool unless it is company-approved for that data. Use synthetic fixtures in prompts.
8. **No new library without a licence check** (NFR-SEC-07).
9. **Keep the regression set green.** Run it after every change (SRS 12.5).

---

## 3. Tool Setup

| Tool | Use | Note |
|---|---|---|
| Chat assistant (company-approved) | Design questions, explanations, drafts, review, debugging | Start a new chat per phase. Paste the SRS section you work on |
| IDE completion (company-approved) | Short code, boilerplate, test cases | Accept small suggestions only. Read before you accept |
| pytest | Unit and regression tests | `pytest -q` after each change |
| Debugger (VS Code) | Step through stage outputs | Better than asking the AI to guess |
| Stage debug dumps | JSON output per pipeline stage | Paste small dumps into the AI when debugging |
| Git | One commit per step | Easy rollback of bad AI code |

### 3.1 Project context file

Create `docs/ai_context.md`. Paste it at the start of each new chat.

```markdown
# Project context – Regulation Diff Checker
- Local Python 3.11 tool. FastAPI on 127.0.0.1. Plain HTML/JS frontend.
- Compares one regulation PDF (pypdfium2) with one Excel Standards sheet (openpyxl, read-only).
- Strict mode: compare Unicode code points. No normalisation unless the SRS says so.
- Never write to input files. No network calls. No GPL/AGPL libraries.
- Data model: app/models.py (paste the relevant classes).
- Defaults and switches: config.py.
- Current phase: <phase>. Current step: <step>. SRS IDs: <IDs>.
- Answer style: short, give code with type hints, explain edge cases, list assumptions.
```

---

## 4. Step Workflow (use for every step)

1. **Understand.** Read the SRS IDs for the step. Ask the AI to explain the step and list edge cases (Prompt P-1).
2. **Design.** Write the function signatures yourself. Ask the AI to review the design (P-2).
3. **Tests.** Ask the AI for test cases (P-3). Select, fix, and add your own. Mark them with requirement IDs. Run them: they must fail.
4. **Code.** Write the code. Use AI for drafts of small parts (P-4). Read every line.
5. **Run.** Run unit tests and the regression set.
6. **Debug.** If a test fails, find the first wrong stage with the debug dumps. Ask the AI with the failing output (P-6).
7. **Review.** Ask the AI to review the final diff against the SRS IDs (P-5). Fix real problems.
8. **Commit.** One commit per step. Update `docs/traceability.md` and `docs/decisions.md`.

---

## 5. Phase Plan with AI Use

The phases, steps, and gates are the same as Development Plan v0.1, Part A. This section shows **how to use AI** in each phase and **what you must do yourself**.

AI use level: **High** = AI drafts most code; **Medium** = AI drafts parts, you write the core; **Low** = you write it, AI explains and reviews only.

### 5.1 Phase 0 – Foundation

| Step | AI use | AI helps with | You do |
|---|---|---|---|
| Repo, venv, requirements, config.py | High | Folder layout, pinned requirements, config draft | Check versions and licences |
| Data model (models.py) | Medium | Dataclass draft from SRS 17.2 / 17.3 | Check every field against the SRS |
| Pipeline skeleton, timing, dumps | High | Stage runner, cache, JSON dump code | Check cache keys |
| Synthetic reference pair | Medium | reportlab and openpyxl generator code | **Write the reference text yourself** |
| Seed generator + golden files | **Low** | Code to apply seed edits | **Write golden files yourself** |
| Special PDF fixtures | Medium | reportlab code for rotate, CropBox, MediaBox | Open each PDF and check |
| Harness, checksum, licence scan, trace tool | High | pytest parametrise, pip-licenses, marker scan | Run and check output |

**Gate G0:** same as Development Plan v0.1.

### 5.2 Phase 1 – Excel

| Step | AI use | AI helps with | You do |
|---|---|---|---|
| Reader + header match | High | openpyxl read-only code | Test `.xlsm`, missing columns |
| `_x000D_` escape check | **Low** | Explain how openpyxl stores escapes | **Test with a real cell yourself**; record result |
| Position check + sort | High | Validation and stable sort | Check error messages |
| Subpoint merge, No subchapter | Medium | Merge logic | Check conflict rules |
| Duplicate consistency check | Medium | Majority value + character diff | Check group scope |
| Group detection + rebuild | **Low** | Explain approach, review | **Write it yourself** (core logic) |
| Character map | **Low** | Suggest the invariant test | **Write it yourself** |

**Gate G1:** same as Development Plan v0.1.

### 5.3 Phase 2 – PDF and Alignment

| Step | AI use | AI helps with | You do |
|---|---|---|---|
| Extraction + boxes | Medium | pypdfium2 code draft | **Check each API call** in the pypdfium2 docs |
| Generated characters | **Low** | Explain FPDFText_IsGenerated and raw API use | Test CRLF and generated space on fixtures |
| Scanned / rotated / ignored pages | High | Page checks, range parser | Test bad input |
| Header / footer removal | Medium | Zone and repeat detection, page number regex | Check removed-text list on the real PDF |
| Block grouping | Medium | Anchor regex, gap rules | Check block dumps by eye |
| Alignment | **Low** | Explain LCS on anchors, similarity fallback | **Write it yourself**; test S-05, S-06 |

**Gate G2:** same as Development Plan v0.1.

### 5.4 Phase 3 – Diff Engine

| Step | AI use | AI helps with | You do |
|---|---|---|---|
| Myers minimal diff | Medium | Explain the algorithm step by step; draft | Trace on small examples; minimality test |
| Two-stage diff | **Low** | Review, edge cases | **Write it yourself**; reconstruction invariant |
| Line-break setting | **Low** | Generate cases from SRS 6.8 table | **Write it yourself** |
| Classification | Medium | Rule order draft, one test per type | Fix rule order; record in decisions.md |
| Unicode details | High | `unicodedata` code, symbol map | Check display-only rule |
| Source links | Medium | Mapping code | Check every difference has a link |
| Duplication counter | **Low** | Generate cases from S-15, S-16 | **Write it yourself**; AC-14, 16, 25, 31 |
| Fast re-run | High | Cache reuse | Measure time (< 3 s) |

**Gate G3:** same as Development Plan v0.1, plus: you can explain the diff, line-break, and counter code without AI.

### 5.5 Phase 4 – UI, Report, Core Release

| Step | AI use | AI helps with | You do |
|---|---|---|---|
| FastAPI server + run.py | High | Endpoints, 127.0.0.1 binding | Check binding and no CDN |
| Header, Text view, panels | High | HTML/CSS/JS drafts from SRS 7.1 | Check colours + second markers, N / P keys |
| XLSX report | Medium | openpyxl writer, data validation | Check report count = screen count |
| Unsaved warning | High | JS draft | Test |
| Playwright UI tests | High | Test drafts | Check they test behaviour |
| Real-file test | **None** (real data) | – | Run locally; make synthetic fixtures from problems |

**Gate G4 (Core release):** same as Development Plan v0.1.

### 5.6 Phases 5 and 6 – Optional Modules

| Phase | AI use | Note |
|---|---|---|
| 5 – Row view | High | UI work on the stable character map |
| 6 – Page view | Medium | AI drafts PDF.js code. **You check** `convertToViewportRectangle`, `isEvalSupported: false`, and the bundled version. Test on rotated, CropBox, MediaBox fixtures |

---

## 6. Prompt Library

Replace `<…>`. Paste `docs/ai_context.md` first in a new chat.

### P-1 Understand a step
```
SRS text: <paste requirement rows>.
Explain what this step must do in simple steps.
List all edge cases, including those in SRS Section 11 for these IDs.
List anything not clear in the SRS.
```

### P-2 Review my design
```
Step: <step>. SRS IDs: <IDs>.
My design: <function signatures + short notes>.
Find gaps against the SRS, edge cases I missed, and simpler options.
Do not write the full code.
```

### P-3 Suggest tests
```
Function: <signature + docstring>. SRS IDs: <IDs>.
Suggest pytest cases: normal, edge, and error input.
Use small inline synthetic strings. Mark each with @pytest.mark.req("<ID>").
Give the expected output for each case and explain why.
```

### P-4 Draft a small part
```
Write only <function name> with this signature: <signature>.
Rules: <strict mode / read-only / config.py defaults>.
Use only <library + version>. If you are not sure an API exists, say so.
Keep it under <N> lines. Add short comments with SRS IDs.
```

### P-5 Review my change
```
SRS IDs: <IDs>. Change: <diff>.
Check: SRS coverage, hidden normalisation, lost differences, writes to input files,
network calls, wrong library API use, missing tests.
List problems as Blocker / Major / Minor. Do not rewrite the code.
```

### P-6 Debug
```
Failing test: <name>. Output: <paste>.
Stage dump (small, synthetic): <paste>.
Expected: <…>. Actual: <…>.
Find the most likely root cause. Give 2–3 checks I can run to confirm it.
```

### P-7 Explain code (learning)
```
Explain this code step by step with a small trace example: <code>.
Show the values of the main variables at each step.
```

### P-8 Check a library call
```
In <library> version <version>, does <function / parameter> exist?
What does it return? Give the documentation page name.
If you are not sure, say so.
```

---

## 7. Checks for AI Output

Use this list before you accept AI code.

| # | Check | Why |
|---|---|---|
| 1 | Every library call exists in the pinned version | AI can invent APIs |
| 2 | No `strip()`, `lower()`, `normalize()`, `replace()` on compared text unless the SRS allows it | Strict mode (FR-CD-03) |
| 3 | No `difflib` with default `autojunk` | FR-CD-04 |
| 4 | Files opened read-only; report writes a new file only | FR-IN-02, AC-04 |
| 5 | No external URL, CDN, or telemetry | NFR-SEC-01, 02 |
| 6 | No silent `except: pass`; errors become warnings | NFR-AC-03 |
| 7 | Each difference keeps a source link | FR-CD-07 |
| 8 | Defaults in `config.py`, not hard-coded | NFR-US-03 |
| 9 | Tests check output values, not only "no error" | Real test coverage |
| 10 | You can explain each line | Ownership |

---

## 8. Weekly Rhythm

1. **Start of week:** list the steps for the week from the phase plan. Check blocking open questions (Development Plan v0.1, Section 3).
2. **Each day:** 1–3 steps with the step workflow (Section 4). Commit each step.
3. **End of week:**
   1. Run the full regression set and the licence scan.
   2. Ask the AI to review the week's changes against the SRS sections (P-5).
   3. Update `docs/traceability.md`.
   4. Write 3–5 lines in `docs/phase_reports/`: done, problems, decisions, AI mistakes found.

---

## 9. Record AI Mistakes

Keep `docs/ai_mistakes.md`. For each mistake, record: date, step, what the AI said, what was correct, and how you found it. Add a check to Section 7 if the mistake can happen again. This list shows where AI is safe to use and where it needs more checks.

---

## 10. Risks

| Risk | Action |
|---|---|
| Invented library API | Check docs (P-8); small test per new call |
| AI adds hidden normalisation | Check list item 2; reconstruction invariant test |
| Plausible code that fails on edge cases | Tests first; seed and holdout sets |
| Developer depends on AI for core logic | Write rebuild, alignment, diff, line-break, and counter code yourself; G3 explain check |
| Real data pasted into a non-approved AI tool | Synthetic fixtures only in prompts (Rule 7) |
| Long chats lose context | New chat per phase; paste `ai_context.md` and SRS sections |
| AI answer conflicts with the SRS | SRS wins; record decision |
