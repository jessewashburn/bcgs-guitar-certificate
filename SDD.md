# Software Design Document
# BCGS Guitar Certificate — Adjudication Evaluation Form

**Version:** 1.0  
**Date:** 2026-05-14  
**App repo (public):** `jessewashburn/bcgs-guitar-certificate`  
**Data repo (private):** `jessewashburn/bcgs-evaluations-data`  
**Deployed at:** `https://jessewashburn.github.io/bcgs-guitar-certificate/`

---

## 1. Overview

A single-page web application hosted on GitHub Pages that allows music adjudicators to complete structured evaluation forms for guitar certificate program performers. On submission, a formatted PDF of the evaluation is generated client-side and committed directly to a **separate private GitHub repository** via the GitHub Contents API. The app repo (which serves the page) stays public so GitHub Pages works on the free tier; the data repo stays private so submitted evaluations are not exposed to the public web. No server, no login, no download required from the judge's perspective.

---

## 2. Goals & Constraints

| Goal | Detail |
|------|--------|
| Zero friction for judges | Public URL, no account, no install, works on any device |
| Auto-populated performer data | Selecting a performer fills age, level, and repertoire automatically |
| Structured scoring | 1–10 rating pills across 5 rubric categories |
| PDF output | Professional formatted evaluation PDF saved to the private data repo |
| Student data privacy | Submitted PDFs and CSV live in a private repo, not the public app repo |
| No backend | Pure static HTML/CSS/JS, GitHub Pages only |
| No judge login | GitHub API auth handled via scoped PAT injected at deploy time |

---

## 3. Repository Structure

Two repos. The **app repo** serves the page and contains no student data. The **data repo** is private and receives every submission.

```
bcgs-guitar-certificate/             ← APP REPO (public)
├── .github/
│   └── workflows/
│       └── deploy.yml  # Injects EVAL_PAT into index.html and deploys to Pages
├── index.html          # Entire application (HTML + CSS + JS, single file)
│                       # Contains __GH_TOKEN__ placeholder, replaced at build
├── SDD.md              # This document
└── README.md           # Optional: brief description for repo visitors

bcgs-evaluations-data/               ← DATA REPO (private)
├── performers.json     # Roster (names, ages, levels, repertoire) — fetched
│                       # at build time and injected into the deployed page
└── evaluations/        # PDFs and CSV backup land here on submission
    └── evaluations.csv # Auto-created on first submission; one row per evaluation
```

The PAT (`EVAL_PAT` secret) is scoped only to the data repo. The deploy workflow uses it to **read** `performers.json` during build and to **write** evaluations on submission. Even though the PAT ends up readable in the deployed HTML, an attacker who extracts it can only read/write to the private data repo — they cannot touch the app repo, the workflow, or the PAT itself.

---

## 4. Tech Stack

| Layer | Choice | Reason |
|-------|--------|--------|
| Hosting | GitHub Pages | Free, zero config, serves from `main` branch root |
| Frontend | Vanilla HTML/CSS/JS | No build step, no dependencies to manage |
| PDF generation | jsPDF (CDN) | Client-side, no server needed, good layout control |
| Storage | GitHub Contents API | Commits PDF as base64 directly to private data repo |
| Auth | Fine-grained PAT (injected at deploy) | Scoped to data repo only, Contents: R/W. Stored as GitHub Actions secret `EVAL_PAT` on the app repo, injected into `index.html` during the Pages build. |
| Fonts | Google Fonts (Playfair Display + DM Sans) | Loaded via CDN link tag |

---

## 5. Application Flow

```
1. Judge opens URL
2. Section 1 loads — judge selects adjudicator name and performer name
3. On performer select → age, level, rep1, rep2 auto-fill from embedded data
4. Judge steps through sections 2–7 using Next / Back buttons
5. On Submit (end of section 7):
   a. Client-side validation (adjudicator, performer, pass/fail recommendation required)
   b. jsPDF builds a formatted PDF from all form state
   c. PDF is base64-encoded
   d. PUT request to the GitHub Contents API commits the PDF to `bcgs-evaluations-data/evaluations/`
   e. GET → mutate → PUT round-trip appends a row to `bcgs-evaluations-data/evaluations/evaluations.csv` (creates the file with a header on first submission)
   f. Success screen shows score summary
   g. Judge can start a new evaluation (full reset)

The CSV append is intentionally non-blocking: if the PDF write succeeds but the CSV append fails, the success screen still renders and a warning toast surfaces the CSV failure. The PDF is the canonical record; the CSV is a queryable backup.
```

---

## 6. Form Sections

### Section 1 — Performer Information
| Field | Type | Required | Notes |
|-------|------|----------|-------|
| Adjudicator name | Dropdown | Yes | Franco Platino, Alejandro Bustamante, Connor Milstead |
| Performer name | Dropdown | Yes | All 20 performers; triggers autofill on change |
| Performer age | Text | Yes | Auto-filled, editable |
| Auditioning for level | Text | Yes | Auto-filled, editable |
| Selected repertoire 1 | Text | Yes | Auto-filled, editable |
| Selected repertoire 2 | Text | Yes | Auto-filled, editable |

### Section 2 — Scales (6 items, 1–10 each)
- Accuracy of notes
- General RH mechanics
- Quality and consistency of tone
- Consistency of articulation
- Consistency of tempo
- General LH mechanics and quality of LH shifts (if applicable)

### Section 3 — Performance: General (3 items, 1–10 each)
- Score reading (pitch and rhythmic accuracy)
- Tempo
- Memorization

### Section 4 — Performance: Technique (4 items, 1–10 each)
- Posture / position
- RH position / efficiency / execution
- Overall tone quality
- LH position / efficiency / execution

### Section 5 — Performance: Interpretation (7 items, 1–10 each)
- Phrasing
- Use of tone color
- Articulation
- Vibrato
- Rubato
- Projection
- Overall dynamic range

### Section 6 — Performance: Presentation (3 items, 1–10 each)
- Stage presence
- Composure
- Attitude

### Section 7 — Comments & Recommendations
| Field | Type | Required |
|-------|------|----------|
| General comments | Textarea | No |
| Recommendations | Textarea | No |
| Recommend passing to next level? | Dropdown (Yes/No) | Yes |
| Recommend trying this level again? | Dropdown (Yes/No) | No |
| Recommend trying for level ___ | Text | No |

**Score totals:**
- Max per item: 10
- Section maxes: Scales 60, General 30, Technique 40, Interpretation 70, Presentation 30
- **Grand total max: 230**

---

## 7. Performer Data

The roster lives in **`performers.json` at the root of the private data repo** (`bcgs-evaluations-data`). The public app repo and this SDD do not contain any performer names, ages, levels, or repertoire — that's the privacy boundary.

`index.html` declares `const PERFORMERS = __PERFORMERS__;` with a literal placeholder. The Pages deploy workflow fetches `performers.json` from the data repo (authenticated with the same `EVAL_PAT` it already uses for submissions) and substitutes the JSON in place of the placeholder during the build. The deployed page contains the real data; the source on `main` of the app repo does not.

**File format:** standard JSON, keyed by full performer name:

```json
{
  "Performer Full Name": {
    "age":   "16",
    "level": "Level 15",
    "rep1":  "Composer — Title",
    "rep2":  "Composer — Title"
  }
}
```

**To update the roster:** edit `performers.json` in the data repo and commit. Then trigger a Pages redeploy on the app repo — either push any change to `index.html`/workflow, or run the workflow manually via Actions → "Deploy to GitHub Pages" → Run workflow. The data repo's pushes do not auto-trigger a redeploy because the workflow lives on the app repo and only watches that repo's `main`.

---

## 8. PDF Generation

**Library:** jsPDF 2.5.1 (loaded from cdnjs.cloudflare.com)  
**Format:** US Letter, portrait  
**Generated entirely client-side before the API call.**

### PDF Layout

```
[Gold rule]
BCGS Certificate Program                    ← Playfair Display / Times serif, 20pt
ADJUDICATION EVALUATION FORM               ← 9pt uppercase muted label
[Gold rule]

Left column:              Right column:
  Adjudicator               Repertoire 1
  Performer                 Repertoire 2
  Age                       Date
  Level

[Light rule]

[Navy header bar] SCALES
  Accuracy of notes ........................ [score pill]
  General RH mechanics ..................... [score pill]
  ... (all items, alternating row shading)
  Section total: XX / 60

[Navy header bar] PERFORMANCE — GENERAL
  ...
  Section total: XX / 30

[Repeat for Technique, Interpretation, Presentation]

[Gold rule]
[Gold filled bar] TOTAL SCORE    XXX / 230

General Comments
[text]

Recommendations
[text]

[Bordered box]
  Pass to next level: Yes/No   Try again: Yes/No
  Recommended level: ___

[Footer rule]
BCGS Certificate Program  ·  Adjudicator Name  ·  Date
```

### PDF Filename Convention

```
evaluations/{PerformerName}_{AdjudicatorFirstName}_{ISO-timestamp}.pdf

Example:
evaluations/Performer_Name_Adjudicator_2026-05-14T10-30-00.pdf
```

---

## 9. CSV Backup

**File:** `bcgs-evaluations-data/evaluations/evaluations.csv` (in the private data repo) — auto-created on first submission, one row appended per submission. Lives alongside the PDFs and serves as a queryable backup / index of all evaluations.

**Format:** RFC 4180 (CRLF line endings, all fields quoted, embedded `"` doubled). UTF-8.

**Columns** (in order):

| Column | Source |
|--------|--------|
| `timestamp` | ISO-8601 (`new Date().toISOString()`) |
| `adjudicator` | Section 1 |
| `performer` | Section 1 |
| `age` | Section 1 (autofilled, editable) |
| `level` | Section 1 (autofilled, editable) |
| `rep1` | Section 1 (autofilled, editable) |
| `rep2` | Section 1 (autofilled, editable) |
| `Scales :: <item>` (×6) | Section 2 ratings, 0–10 each |
| `Performance — General :: <item>` (×3) | Section 3 ratings |
| `Performance — Technique :: <item>` (×4) | Section 4 ratings |
| `Performance — Interpretation :: <item>` (×7) | Section 5 ratings |
| `Performance — Presentation :: <item>` (×3) | Section 6 ratings |
| `<section> total` (×5) | Computed section totals |
| `grand_total` | Sum of section totals (max 230) |
| `pass_to_next_level` | Section 7 dropdown |
| `try_this_level_again` | Section 7 dropdown |
| `recommend_level` | Section 7 text |
| `comments` | Section 7 textarea |
| `recommendations` | Section 7 textarea |
| `pdf_filename` | Path of the PDF written in the preceding API call |

**Header columns** are emitted by `csvHeader()` in `index.html`; the column count and order are derived from `SECTIONS`, so adding a rubric item automatically extends the CSV schema (existing rows will simply have fewer trailing fields — readers must tolerate jagged rows or you backfill manually).

**Append mechanism:** GitHub's Contents API has no native append, so each write is:

1. `GET /repos/.../contents/evaluations.csv` → decode base64 body, capture `sha`. A 404 means this is the first submission.
2. Concatenate `existing || csvHeader()` + `newRow`.
3. `PUT` the full file content with the captured `sha`.

A 409 / 422 response indicates the SHA was stale (another judge submitted between our GET and PUT). The client retries once. If it still fails, the CSV append is reported as a soft error — the PDF was already saved, so the submission as a whole succeeds and a warning toast surfaces the CSV failure.

**Race window:** GET → PUT is a few hundred ms. With three judges submitting sequentially through the same scheduled program, collisions are unlikely. The single retry covers the realistic worst case.

---

## 10. GitHub API Integration

**Endpoint:** `PUT https://api.github.com/repos/jessewashburn/bcgs-evaluations-data/contents/{path}`

The host `index.html` is served from the public app repo, but every API call from the form targets the **private data repo** above. The `GH_OWNER`, `GH_REPO`, and `GH_PATH` constants in `index.html` define this target.

**Headers:**
```
Authorization: token {GH_TOKEN}
Content-Type: application/json
Accept: application/vnd.github+json
```

**Body:**
```json
{
  "message": "Add evaluation: {Performer} — {Adjudicator}",
  "content": "{base64-encoded PDF}"
}
```

**Token:** Fine-grained PAT, scoped to `bcgs-evaluations-data` only, Contents: Read & Write. No access to the app repo, no other permissions.

**Storage model:**
- Source on the app repo's `main` contains the literal placeholder `__GH_TOKEN__` in `index.html`.
- The PAT is stored as a repository secret named `EVAL_PAT` on the **app repo** (`bcgs-guitar-certificate`), since that's where the Pages workflow runs.
- The Pages deploy workflow (`.github/workflows/deploy.yml`) substitutes the placeholder into a copy of `index.html` during the build, then publishes the result to GitHub Pages.
- The token never lands on `main` of either repo and never appears in git history.

**Threat model note:** A visitor to the deployed site can still read the PAT via View Source on the served HTML. With the split-repo architecture, the token's blast radius is *only* the private data repo — an attacker can write junk commits into `bcgs-evaluations-data` but cannot touch the app source, modify the form, or rotate their own privileges. The secret-injection approach keeps the token out of repo source and git history; the split-repo design caps the worst-case damage even if the token leaks from the browser.

**On success (201):** Show success screen with score breakdown.  
**On failure:** Show error toast, re-enable submit button so judge can retry.

---

## 11. UI/UX Design Spec

**Aesthetic:** Refined editorial — warm cream backgrounds, gold accents, Playfair Display serif headings, DM Sans body. Feels like a music conservatory program, not a generic form.

**Color tokens:**
```css
--ink:        #1a1714   /* primary text */
--ink-soft:   #4a4540   /* secondary text */
--ink-muted:  #8a837a   /* labels, placeholders */
--cream:      #faf8f4   /* page background */
--cream-warm: #f3efe8   /* summary backgrounds */
--cream-dark: #e8e2d8   /* dividers, unselected states */
--gold:       #b8860b   /* primary accent */
--gold-light: #d4a017   /* hover states */
--gold-pale:  #fdf8ec   /* autofill field background */
--border:     #d4cfc6   /* field borders */
--navy:       #2c3e50   /* section headers, submit button */
```

**Progress bar:** 7 segments, gold fill for completed, gold-light for current, cream-dark for upcoming.

**Rating pills:** 10 circular buttons (1–10) per item. Unselected: white with gray border. Selected: gold fill, white text. Row background shifts to white with border when any pill selected.

**Autofill visual:** Fields populated from performer selection get a gold-pale background and italic text. Clears on manual edit.

**Toast notifications:** Fixed bottom-center, navy background for info, red for errors. Auto-dismisses after 3.5s.

**Navigation:** Back / Next buttons. Back disabled on section 1. Next becomes "Submit →" on section 7.

**Success screen:** Replaces nav and progress bar. Shows adjudicator + performer, score breakdown grid (one cell per section), grand total, and "New evaluation" button which fully resets all state.

**Responsive:** Works on tablet and phone. Field rows collapse to single column below 500px. Pill size reduces on small screens.

---

## 12. Validation Rules

| Rule | When checked |
|------|-------------|
| Adjudicator must be selected | On Next from section 1 |
| Performer must be selected | On Next from section 1 |
| "Pass to next level" must be answered | On Submit (section 7) |

All other fields are optional. Unscored rating items default to 0 and are shown as "—" in the PDF.

---

## 13. State Management

All state lives in JavaScript variables within the single HTML file. No localStorage, no cookies, no external state.

```js
const ratings = {};       // { "groupId::itemLabel": 0-10 }
let current = 1;          // active section number (1-7)
const PERFORMERS = {...}  // injected at deploy time from data repo
const SECTIONS = [...]    // rating group definitions
```

Reset on "New evaluation": clears all select/input/textarea values, zeros all ratings, removes pill selections, returns to section 1.

---

## 14. Deployment Steps

**One-time setup:**

1. Create the **data repo**: `bcgs-evaluations-data`, **Private**, initialize with any file (a README is fine).
2. Create `performers.json` at the root of the data repo with the roster (see §7 for format).
3. Generate a fine-grained PAT at github.com/settings/personal-access-tokens/new:
   - Resource owner: `jessewashburn`
   - Repository access: **Only select repositories** → `bcgs-evaluations-data` (NOT the app repo)
   - Repository permissions → **Contents: Read and write**
4. On the **app repo** (`bcgs-guitar-certificate`): Settings → Secrets and variables → Actions → New repository secret. Name: `EVAL_PAT`. Value: the PAT from step 3.
5. On the **app repo**: Settings → Pages → Source: **GitHub Actions** (not "Deploy from a branch").
6. Commit `index.html`, `.github/workflows/deploy.yml`, and `SDD.md` to `main` of the app repo.
7. The workflow runs on push to `main`. It fetches `performers.json` from the data repo, injects roster + PAT into `index.html`, and deploys. Once it completes (~1 min), the site is live at `https://jessewashburn.github.io/bcgs-guitar-certificate/`. The first submission auto-creates `evaluations/evaluations.csv` and the first PDF in the data repo.

**Subsequent updates:** Any push to the app repo's `main` that touches `index.html` (or the workflow itself) triggers a redeploy. The data repo is **not** watched — pushing changes to `performers.json` or `evaluations/` does not redeploy on its own.

**To update performer data:** Edit `performers.json` in the data repo and commit. Then trigger a redeploy on the app repo: either push any change to `index.html`/workflow, or run the workflow manually via the Actions tab → "Deploy to GitHub Pages" → Run workflow.

**To rotate the GitHub token:**
1. Generate a new fine-grained PAT (same scope — data repo only, Contents: R/W).
2. Update the `EVAL_PAT` secret value on the app repo.
3. Trigger a redeploy: push any change to `index.html`, or run the workflow manually via the Actions tab → "Deploy to GitHub Pages" → Run workflow.
4. Revoke the old PAT at github.com/settings/personal-access-tokens.

---

## 15. Known Limitations & Future Considerations

| Item | Note |
|------|------|
| No duplicate submission guard | Two judges could theoretically submit simultaneously and collide on the same PDF filename. Timestamp in filename makes this extremely unlikely but not impossible. Could add a random suffix if needed. |
| CSV concurrent-write race | Appending to `evaluations.csv` is a GET-then-PUT cycle with optimistic SHA matching. Simultaneous submits can race; one retry covers the realistic case. If both retries lose, the CSV row is dropped (the PDF still saves). A more robust design would queue writes through a serverless function — out of scope for this version. |
| No offline support | Requires internet for Google Fonts, jsPDF CDN, and GitHub API. Could be made fully offline with inlined assets. |
| Token rotation | PAT has an expiration date set at creation. Needs manual rotation before expiry. Set a calendar reminder. |
| Score of 0 | Unrated items score 0. PDF shows "—" for unscored items so it's clear vs. a deliberate 0. Consider whether all items should be required. |
| No email delivery | PDFs land in the repo. Sending them to participants requires a manual step (download from repo, email). A future enhancement could add a GitHub Action that emails PDFs to participants on commit. |
