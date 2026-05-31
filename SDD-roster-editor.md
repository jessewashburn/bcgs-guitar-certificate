# Software Design Document — Addendum
# BCGS Guitar Certificate — In-App Roster Editor

**Version:** 1.0
**Date:** 2026-05-31
**Status:** Implemented
**Parent doc:** `SDD.md` (v1.0) — this addendum extends it; sections referenced as "§N" point at the parent.
**App repo (public):** `jessewashburn/bcgs-guitar-certificate`
**Data repo (private):** `jessewashburn/bcgs-evaluations-data`

---

## 1. Problem

Updating the performer roster for a new year currently requires a maintainer to (a) open the **private data repo** on GitHub, (b) hand-edit `performers.json` as raw JSON, and (c) trigger a Pages redeploy on the app repo so the new roster is baked into the served page (parent §7, §14). That is three steps across two repos plus knowledge of JSON syntax — too much friction for a once-a-year task that a non-technical program coordinator should be able to do.

**Goal:** an editor built into the existing form that lets an authorized person update next year's roster directly — "no more complicated than editing a CSV" — with no GitHub visit, no JSON hand-editing, and no manual redeploy.

---

## 2. Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| How edits reach the live form | **Fetch roster at runtime** | Today the roster is injected into the HTML at build time (`__PERFORMERS__`, parent §7), so a roster change is invisible until a redeploy. Fetching `performers.json` from the data repo on page load makes any save live on the next refresh — which eliminates the redeploy step entirely and is what actually removes the "go to GitHub" friction. |
| Privacy side-effect | **Improved** | With runtime fetch, student names/ages/repertoire are no longer baked into the public-served `index.html` (currently readable via View Source). They are fetched into memory over an authenticated request and never persisted in the public artifact. |
| Who can edit | **Light passphrase gate** | The form has no login and the PAT is already extractable from the page (parent §10 threat model), so this is *not* cryptographic security — it is a guard against casual/accidental edits by judges or random visitors. The passphrase is checked client-side. |
| UI form factor | **Editable table (spreadsheet-like)** | One row per performer; columns Name / Age / Level / Rep 1 / Rep 2; add-row and delete-row controls. Matches the "like editing a CSV" mental model. |
| Auth / token | **Reuse existing `EVAL_PAT`** | The deployed PAT already has Contents: Read & Write on the data repo, which covers both reading and writing `performers.json` at the repo root. No new secret, no scope change. |

---

## 3. Architecture Changes

### 3.1 Roster moves from build-time injection to runtime fetch

**Before** (parent §7): the deploy workflow fetches `performers.json` and substitutes it for the `__PERFORMERS__` placeholder; `index.html` declares `const PERFORMERS = __PERFORMERS__;`.

**After:**
- `index.html` no longer holds a roster placeholder. It declares `let PERFORMERS = {};` and a `let rosterSha = null;` (the blob SHA needed to write the file back).
- On page load the form makes an authenticated `GET` for `performers.json`, decodes it, parses it, and populates the performer dropdown.
- The deploy workflow no longer fetches the roster or substitutes `__PERFORMERS__`. The only build-time substitution that remains is `__GH_TOKEN__` (parent §10).

> The `__GH_TOKEN__` injection is unchanged. Only the roster handling moves to runtime.

### 3.2 Files touched

| File | Change |
|------|--------|
| `index.html` | Replace `PERFORMERS` constant + placeholder with runtime load; add roster-editor panel markup, styles, and JS (load / render / save). |
| `.github/workflows/deploy.yml` | Remove the "Fetch performer roster" step and the `__PERFORMERS__` half of the inject step. Drop `__PERFORMERS__` from the placeholder-presence check (keep `__GH_TOKEN__`). |
| `SDD.md` (parent) | Update §7 (roster is now fetched at runtime, edited in-app), §13 (`PERFORMERS` is now mutable + runtime-loaded), §14 ("To update performer data" steps are obsolete). |

---

## 4. Data Flow

### 4.1 Load (every page open)

```
1. Page renders with an empty performer dropdown + a "Loading roster…" state.
2. GET /repos/jessewashburn/bcgs-evaluations-data/contents/performers.json
     headers: Authorization: token <PAT>, Accept: application/vnd.github+json
     cache: no-store
3. 200 → decode base64 body (reuse b64decodeUtf8), JSON.parse → PERFORMERS,
         capture body.sha → rosterSha, populate dropdown.
   404 → treat as empty roster {} (first-ever setup); rosterSha = null.
   else → show error toast + a "Retry" affordance; dropdown stays empty,
          name/age/level fields remain manually editable as a fallback.
```

### 4.2 Edit & Save (roster editor)

```
1. User clicks "Manage roster" → passphrase prompt.
2. On correct passphrase → editor panel opens, pre-filled with current PERFORMERS
   as table rows.
3. User adds/edits/deletes rows. Each row: Name, Age, Level, Rep 1, Rep 2.
4. User clicks "Save roster":
   a. Client validation (see §7): non-empty trimmed names, no duplicate names.
   b. Build the JSON object keyed by name (same shape as parent §7).
   c. GET current sha (re-read just before write to minimise stale-sha races):
        - if rosterSha known, use it; the write below handles a stale sha via retry.
   d. PUT /contents/performers.json with:
        { message, content: base64(JSON, 2-space pretty), sha: rosterSha }
        (omit sha if creating the file for the first time)
   e. 200/201 → update in-memory PERFORMERS + rosterSha from response, re-populate
        the dropdown, close editor, success toast.
      409/422 (stale sha) → re-GET sha, retry PUT once (mirrors appendCsvRow, parent §9).
      else → error toast, leave editor open so edits aren't lost.
```

The PUT writes the **whole file** (Contents API has no partial update), exactly like the CSV append in parent §9 — GET-sha → PUT-full-content with optimistic concurrency and a single retry.

---

## 5. GitHub API Integration

Reuses the existing target constants (`GH_OWNER`, `GH_REPO`) and token (`GH_TOKEN`) from `index.html`. New path constant:

```js
const ROSTER_PATH = 'performers.json'; // repo root of the data repo
```

**Read:**
```
GET https://api.github.com/repos/{GH_OWNER}/{GH_REPO}/contents/performers.json
Authorization: token {GH_TOKEN}
Accept: application/vnd.github+json
```
Response `content` is base64 (decode with the existing `b64decodeUtf8`); `sha` is retained for writes.

**Write:**
```
PUT https://api.github.com/repos/{GH_OWNER}/{GH_REPO}/contents/performers.json
{
  "message": "Update performer roster (in-app editor)",
  "content": "<base64 of pretty-printed JSON>",
  "sha": "<rosterSha — omit when creating>"
}
```

JSON is serialized with `JSON.stringify(obj, null, 2)` so the committed file stays human-diffable in the data repo.

---

## 6. UI / UX Spec

Consistent with the editorial aesthetic and color tokens in parent §11.

- **Entry point:** a small, unobtrusive "Manage roster" text link in the page footer / below the form card (visible on section 1, not mid-evaluation). Not a primary button — it should not compete with the judge's flow.
- **Passphrase prompt:** a modal (or inline panel) with a single password field + "Unlock". Wrong passphrase → error toast, stays closed.
- **Editor panel:** overlays/replaces the form card. Contains:
  - A table: header row (Name*, Age, Level, Repertoire 1, Repertoire 2, ✕) and one editable `<input>` row per performer.
  - **"+ Add performer"** button appends a blank row (focus its name field).
  - Per-row **✕** removes that row (no confirm for an unsaved row; the roster isn't written until Save).
  - **"Save roster"** (navy primary) and **"Cancel"** (secondary) at the bottom. Cancel discards edits and reverts to the in-memory roster.
- **Busy / feedback:** Save shows a "Saving…" state and disables itself during the PUT; results surface via the existing `showToast` (navy info / red error, parent §11).
- **Responsive:** the table scrolls horizontally on narrow screens, or rows collapse to stacked label/value blocks below 500px (consistent with the form's responsive behavior).

---

## 7. Validation Rules

| Rule | When |
|------|------|
| Each row's **Name** is required and trimmed | On Save |
| **Names must be unique** (they are the JSON keys) | On Save — duplicates rejected with a toast naming the collision |
| Empty rows (no name) are dropped silently | On Save |
| Age / Level / Rep1 / Rep2 are free text, optional | — (matches parent §6, where these are autofilled-but-editable) |

A roster with zero performers is permitted to save (e.g., clearing out a finished year), but the form's existing submit validation (parent §12) still requires a performer to be selected before an evaluation can be submitted.

---

## 8. Error Handling & Edge Cases

| Case | Behavior |
|------|----------|
| Roster GET fails on load | Error toast + Retry; performer dropdown empty; manual field entry still works so a judge is never hard-blocked. |
| `performers.json` missing (404) | Treated as empty roster; first Save creates the file (PUT without `sha`). |
| Stale sha on Save (409/422) | Re-GET sha, retry PUT once (mirrors parent §9). If still failing → error toast, editor stays open. |
| Two people editing concurrently | Last writer wins after the single retry. Acceptable: roster edits are a rare, near-solo admin action, unlike concurrent judge submissions. |
| Invalid JSON already in repo | `JSON.parse` failure on load → error toast; editor refuses to open (would otherwise overwrite unknown data). Manual data-repo fix required (rare). |
| Passphrase | Client-side only; documented as casual-tamper prevention, not security (parent §10 threat model already assumes the PAT is public). |

---

## 9. Impact on Parent SDD

On acceptance, update `SDD.md`:
- **§7 Performer Data** — roster is fetched at runtime and editable in-app; remove the `__PERFORMERS__` build-time injection description and the manual "edit JSON + redeploy" instructions.
- **§13 State Management** — `PERFORMERS` becomes `let` (mutable, runtime-loaded); add `rosterSha`.
- **§14 Deployment Steps** — delete "To update performer data" (now done in-app); the workflow no longer fetches the roster.
- **§15 Limitations** — note the passphrase is not real auth and concurrent roster edits are last-writer-wins.

---

## 10. Out of Scope

- Real authentication / per-user accounts.
- Server-side validation or a serverless write proxy.
- Roster version history beyond the data repo's normal git history (every Save is a commit, so history is preserved there for free).
- Bulk CSV import/export of the roster (could be a future enhancement; current editor is row-by-row).
