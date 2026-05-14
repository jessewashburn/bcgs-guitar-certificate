# Software Design Document
# BCGS Guitar Certificate — Adjudication Evaluation Form

**Version:** 1.0  
**Date:** 2026-05-14  
**Repo:** `jessewashburn/bcgs-guitar-certificate`  
**Deployed at:** `https://jessewashburn.github.io/bcgs-guitar-certificate/`

---

## 1. Overview

A single-page web application hosted on GitHub Pages that allows music adjudicators to complete structured evaluation forms for guitar certificate program performers. On submission, a formatted PDF of the evaluation is generated client-side and committed directly to the GitHub repository via the GitHub Contents API. No server, no login, no download required from the judge's perspective.

---

## 2. Goals & Constraints

| Goal | Detail |
|------|--------|
| Zero friction for judges | Public URL, no account, no install, works on any device |
| Auto-populated performer data | Selecting a performer fills age, level, and repertoire automatically |
| Structured scoring | 1–10 rating pills across 5 rubric categories |
| PDF output | Professional formatted evaluation PDF saved to the repo |
| No backend | Pure static HTML/CSS/JS, GitHub Pages only |
| No judge login | GitHub API auth handled via scoped PAT injected at deploy time |

---

## 3. Repository Structure

```
bcgs-guitar-certificate/
├── .github/
│   └── workflows/
│       └── deploy.yml  # Injects EVAL_PAT into index.html and deploys to Pages
├── index.html          # Entire application (HTML + CSS + JS, single file)
│                       # Contains __GH_TOKEN__ placeholder, replaced at build
├── evaluations/        # PDFs land here on submission
│   └── .gitkeep        # Placeholder to initialize the folder
├── SDD.md              # This document
└── README.md           # Optional: brief description for repo visitors
```

---

## 4. Tech Stack

| Layer | Choice | Reason |
|-------|--------|--------|
| Hosting | GitHub Pages | Free, zero config, serves from `main` branch root |
| Frontend | Vanilla HTML/CSS/JS | No build step, no dependencies to manage |
| PDF generation | jsPDF (CDN) | Client-side, no server needed, good layout control |
| Storage | GitHub Contents API | Commits PDF as base64 directly to repo |
| Auth | Fine-grained PAT (injected at deploy) | Scoped to this repo, contents write only. Stored as GitHub Actions secret `EVAL_PAT`, injected into `index.html` during the Pages build. |
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
   d. PUT request to GitHub Contents API commits the file to /evaluations/
   e. Success screen shows score summary
   f. Judge can start a new evaluation (full reset)
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

All performer data is embedded as a JavaScript object in `index.html`. No external data source. Structure:

```js
const PERFORMERS = {
  "Performer Full Name": {
    age:   "16",
    level: "Level 15",
    rep1:  "Composer — Title",
    rep2:  "Composer — Title"
  },
  // ...
}
```

**Full performer list (20 performers):**

| Name | Age | Level | Rep 1 | Rep 2 |
|------|-----|-------|-------|-------|
| Theo Anderson | 16 | Level 15 | Barrios — Mazurka appassionata | Mertz — Elegie |
| Katherine Isaev | 14 | Level 9 | Fernando Sor — Estudio No. 2, Op. 35 No. 13 | Francisco Tárrega — Lágrima (Prelude) |
| Todd Davis | 49 | Level 9 | Sor — Op. 35, No. 13 | Tárrega — Lágrima |
| Jackson Booker | 9 | Level 5 | Traditional — Waltz No. 10 | Anonymous — Greensleeves |
| Hannah S. Callahan | 16 | Level 10 | Fernando Sor — Rondo Op. 48, No. 6 | Matteo Carcassi — Op. 60, No. 7 |
| Orion Tiberius Awad | 14 | Level 7 | Mauro Giuliani — Allegro | Joseph Meissonnier — Waltz |
| Luke Nicholaides | 14 | Level 7 | Giuliani — Allegro | Carcassi — Etude 3 |
| Annalise McCombs | 7 | Level 4 | Matteo Carcassi — Andante No. 4 | Niccolò Paganini — Andante No. 5 |
| Jet Lowe | Adult | Jury Comments Only | J.S. Bach — Sarabande from Cello Suite No. 3 | G. Frescobaldi / J. Dowland |
| Emma Naik | 11 | Level 6 | Paganini — Waltz | Aguado — Study in A minor |
| Riaan Naik | 6 | Level 2 | Bach — Perpetual Motion | Bach — Tanz |
| Declan Purcell | 10 | Level 2 | S. Suzuki — Perpetual Motion | J.C. Bach — Tanz |
| Luu Li Pham | 15 | Level 15 | Joaquín Turina — Hommage à Tárrega (Garrotín & Soleares) | Sergio Assad — Suite Aquarelle; Valseana |
| Charlie Adam | 11 | Level 5 | Sor — Opus 31, No. 1 | Carulli — Waltz, Op. 121, No. 1 |
| Judah Gene Cronce | 8 | Level 4 | Niccolò Paganini — Andante No. 5 | Niccolò Paganini — Corrente |
| George Zhang | 11 | Level 6 | Mauro Giuliani — Allegro, Op. 50, No. 13 | Fernando Sor — Allegretto |
| Gleb Onishchenko | 13 | Level 10 | Francisco Tárrega — Adelita | Matteo Carcassi — Estudio in A minor, Op. 60 No. 7 |
| Lucca Farkas | 17 | Level 7 | Eduardo Sainz de la Maza — Campanas del alba | Scarlatti — Sonata, K. 209 |
| Everett H. Struble | 9 | Level 3 | F. Longay — Meadow Minuet (Suzuki Book 1) | Thomas Haynes Bayly — Long Long Ago |
| Sullivan R. Struble | 6 | Level 2 | Dr. Suzuki — Perpetual Motion (Suzuki Book 1) | J.C. Bach — Tanz (Suzuki Book 1) |

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
evaluations/Theo_Anderson_Franco_2026-05-14T10-30-00.pdf
```

---

## 9. GitHub API Integration

**Endpoint:** `PUT https://api.github.com/repos/jessewashburn/bcgs-guitar-certificate/contents/{path}`

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

**Token:** Fine-grained PAT, scoped to `bcgs-guitar-certificate` repo, Contents: Read & Write only.

**Storage model:**
- Source on `main` contains the literal placeholder `__GH_TOKEN__` in `index.html`.
- The PAT is stored as a repository secret named `EVAL_PAT`.
- The Pages deploy workflow (`.github/workflows/deploy.yml`) substitutes the placeholder into a copy of `index.html` during the build, then publishes the result to GitHub Pages.
- The token never lands on `main` and never appears in git history.

**Threat model note:** A visitor to the deployed site can still read the PAT via View Source on the served HTML. The token's blast radius is intentionally minimal — Contents: R/W on this one repo — so the worst-case abuse remains junk file commits here. The secret-injection approach is about keeping the token out of repo source and git history (and therefore out of GitHub's secret-scanning auto-revoke path), not about hiding it from the browser.

**On success (201):** Show success screen with score breakdown.  
**On failure:** Show error toast, re-enable submit button so judge can retry.

---

## 10. UI/UX Design Spec

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

## 11. Validation Rules

| Rule | When checked |
|------|-------------|
| Adjudicator must be selected | On Next from section 1 |
| Performer must be selected | On Next from section 1 |
| "Pass to next level" must be answered | On Submit (section 7) |

All other fields are optional. Unscored rating items default to 0 and are shown as "—" in the PDF.

---

## 12. State Management

All state lives in JavaScript variables within the single HTML file. No localStorage, no cookies, no external state.

```js
const ratings = {};       // { "groupId::itemLabel": 0-10 }
let current = 1;          // active section number (1-7)
const PERFORMERS = {...}  // static lookup table
const SECTIONS = [...]    // rating group definitions
```

Reset on "New evaluation": clears all select/input/textarea values, zeros all ratings, removes pill selections, returns to section 1.

---

## 13. Deployment Steps

**One-time setup:**

1. Generate a fine-grained PAT at github.com/settings/tokens?type=beta — scope to the `bcgs-guitar-certificate` repo, Contents: Read & Write only.
2. In the repo: Settings → Secrets and variables → Actions → New repository secret. Name: `EVAL_PAT`. Value: the PAT.
3. In the repo: Settings → Pages → Source: **GitHub Actions** (not "Deploy from a branch").
4. Commit `index.html`, `evaluations/.gitkeep`, `.github/workflows/deploy.yml`, and `SDD.md` to `main`.
5. The workflow runs on push to `main`. Once it completes (~1 min), the site is live at `https://jessewashburn.github.io/bcgs-guitar-certificate/`.

**Subsequent updates:** Any push to `main` that touches `index.html` (or the workflow itself) triggers a redeploy. PDF commits under `evaluations/` are ignored by the workflow (`paths-ignore`) so they don't trigger wasted rebuilds.

**To update performer data:** Edit the `PERFORMERS` object in `index.html` and commit. The workflow injects the secret and redeploys automatically.

**To rotate the GitHub token:**
1. Generate a new fine-grained PAT (same scope).
2. Update the `EVAL_PAT` secret value in repo Settings.
3. Trigger a redeploy: either push any change to `index.html`, or run the workflow manually via the Actions tab → "Deploy to GitHub Pages" → Run workflow.
4. Revoke the old PAT.

---

## 14. Known Limitations & Future Considerations

| Item | Note |
|------|------|
| No duplicate submission guard | Two judges could theoretically submit simultaneously and collide on the same filename. Timestamp in filename makes this extremely unlikely but not impossible. Could add a random suffix if needed. |
| No offline support | Requires internet for Google Fonts, jsPDF CDN, and GitHub API. Could be made fully offline with inlined assets. |
| Token rotation | PAT has an expiration date set at creation. Needs manual rotation before expiry. Set a calendar reminder. |
| Score of 0 | Unrated items score 0. PDF shows "—" for unscored items so it's clear vs. a deliberate 0. Consider whether all items should be required. |
| No email delivery | PDFs land in the repo. Sending them to participants requires a manual step (download from repo, email). A future enhancement could add a GitHub Action that emails PDFs to participants on commit. |
