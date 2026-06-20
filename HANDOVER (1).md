# CSU Library Journal Finder — Handover Document for Codex

## Files included in this handover
This document is bundled in `codex-handover.zip` along with:
- `index.html` (the app)
- `journals.json` (the data, ~7.3 MB uncompressed / ~1 MB zipped)

**If Codex's upload interface rejects the zip or the JSON file (size limits vary by platform):**
- Unzip locally first and upload `index.html` + `HANDOVER.md` only to start the conversation — Codex doesn't need to read all 35,000 JSON records to understand or edit the code, since the JS reads the JSON's *structure*, not its content
- Once Codex is working in a proper repo/workspace (e.g. it creates a GitHub repo or you connect one), `journals.json` just needs to sit alongside `index.html` — you can upload it directly to that repo via the GitHub web UI exactly as described in Section 7 below, completely separately from your Codex conversation

## Purpose of this document
This is a complete handover for an AI coding agent (OpenAI Codex / ChatGPT Codex) taking over development of this project. It includes full context, history, known issues, and instructions — paste this entire document as your first message to Codex, along with the two project files (`index.html` and `journals.json`).

---

## 1. What this project is

A **static, client-side journal search tool** for Charles Sturt University (CSU) Division of Library Services. It lets staff and researchers search/filter ~35,000 academic journals by title, publisher, agreement type, Scimago ranking, CSU ranking, FOR (Field of Research) codes, ISSN, and open access eligibility status.

**Live site:** `https://roseyraine.github.io/journal-finder/`
**Current hosting:** GitHub Pages (repo: `roseyraine/journal-finder`, public)
**Intended embedding:** Pasted into a Springshare LibGuides page via `<iframe>`

There is **no backend, no database, no build step**. It is two files:
- `index.html` — the entire UI, styling, and logic (vanilla JS, no frameworks, no dependencies)
- `journals.json` — the full dataset, loaded client-side via `fetch()`

---

## 2. Files in this handover

| File | Size | Role |
|---|---|---|
| `index.html` | ~22 KB | Complete single-page app: HTML + embedded `<style>` + embedded `<script>` |
| `journals.json` | ~7.2 MB | Array of ~35,108 journal record objects |

### `journals.json` record shape
```json
{
  "Journal Title": "string",
  "ISSNs": "string — semicolon-separated, e.g. '1234-5678; 8765-4321'",
  "CSU Rank": "string — 'Q1'|'Q2'|'Q3'|'Q4'|'' (empty if unranked)",
  "Scimago Rank": "string — 'Q1'|'Q2'|'Q3'|'Q4'|''",
  "FOR Codes": "string — semicolon-separated 4-digit codes, e.g. '3103; 4102'",
  "Publisher Name": "string, often empty",
  "Agreement": "string — CAUL agreement name, often empty",
  "Eligible": "string — 'Yes'|'No'|'N/A' (CSU OA agreement eligibility)",
  "Diamond OA": "string — 'Yes'|'' (Diamond Open Access status)"
}
```

---

## 3. How the data was built (for context only — not needed to maintain the site)

The dataset was assembled outside this project from three sources and will NOT need to be rebuilt by Codex unless explicitly asked:
1. A CSU "Journal Ranking List" (internal rankings, FOR codes, ISSNs)
2. CAUL OA Agreements Title List (publisher agreements, eligibility, updated periodically by CAUL)
3. Scimago Journal Rankings 2025 CSV (Diamond OA status, Scimago quartile)

These were merged by ISSN (with title-matching as a fallback) into the flat `journals.json` structure above. **If the librarian provides an updated CSV/Excel file in future and asks you to "update the data," you'll need to ask them for the raw source file(s) — the merge logic itself lives outside this handover and was done in a separate data-processing session.** If they only give you a replacement `journals.json` directly, you can just swap the file.

---

## 4. Architecture of `index.html`

Single file, three logical sections:

### `<style>` (CSS)
- Custom properties (`:root`) define the CSU brand palette — navy `#222944`, orange `#F0572A`, charcoal `#222222`, plus supporting greens/blues/browns
- Mobile-responsive breakpoint at `640px` (hides ISSN/FOR columns on small screens)
- Sticky table header pattern — see **Known Issues** below, this has been a recurring pain point

### HTML body
- `<header>` — sticky page header with CSU branding
- `.hero` — page title and description banner
- `.filter-panel` — search input + 6 filter dropdowns (CSU OA Agreement, Filter by agreement, Diamond OA, Scimago Rank, CSU Rank, FOR code) + Clear all button
- `.stats-bar` — result count + colour legend
- `.table-wrap` → `.table-scroll` → `<table>` — the results table, paginated client-side at 50 rows/page

### `<script>` (vanilla JS, no libraries)
Key functions:
- `fetch(DATA_URL)` — loads `journals.json` on page load
- `buildDropdowns(data)` — dynamically populates the Agreement and FOR Code filter options from the dataset (so they don't need to be hardcoded)
- `applyFilters()` — filters `allData` based on search box + all 6 dropdowns, then sorts, then calls `render()`
- `render()` — slices the current page (50 rows) and writes table HTML via `innerHTML`
- `renderPagination()` — builds prev/next + numbered page buttons with ellipsis truncation
- Sort logic strips leading non-alphabetic characters so titles like `"2D Materials"` or `"(mt) Marine Technology"` sort under their first real letter rather than floating to the top

---

## 5. Known issues / recent bug history (IMPORTANT — read this before making changes)

### Issue: rows appearing above the sticky table header
**Status: partially fixed, may recur — this is the most important thing to verify first.**

Multiple fix attempts were made in the previous session:
1. First attempt: fixed the *sort key logic* (was stripping digits incorrectly, causing numeric-titled journals to sort first) — this was a real bug and is fixed
2. Second attempt: changed `.table-wrap`/`.table-scroll` from `overflow: hidden` to `overflow: visible`/`auto`, and made the table header sticky relative to an internal scroll container (`max-height: 70vh` on `.table-scroll`) rather than the page scroll
3. Third attempt (most recent): user reported the bug **still occurring** even after fix #2 — specifically, the very first row of the `tbody` rendering visually above the sticky `<thead>`. The final fix applied was:
   - Explicit `background`, `position: relative`, `z-index: 1` on every `tbody tr`
   - `z-index: 50` plus `transform: translateZ(0)` + `will-change: transform` on `<thead>` to force it onto its own GPU-composited layer

**This last fix had NOT yet been confirmed working by the user when handover occurred.** If the user reports this bug again:
- Check whether it happens in the standalone page or specifically inside the LibGuides iframe (iframe-specific stacking context bugs are a different class of problem)
- Consider as a more robust alternative: replacing the CSS sticky-header pattern entirely with a JS-based "shadow header" that's a separate fixed-position element synced to scroll, OR removing `position: sticky` from the table header altogether if the bug persists (simpler, less elegant, but bulletproof)
- Test directly in Microsoft Edge specifically, since that's the user's browser and where the bug was last observed

### Issue: duplicate journal titles
Already fixed — 535 true duplicates (same title + same ISSN) were removed from the dataset during cleaning.

### Issue: filter labels
Already renamed per user request:
- "CSU Eligible" → "CSU OA Agreement"
- "Agreement" → "Filter by agreement"
- Added new filters: "FOR code", and keyword search now covers ISSN too

---

## 6. Things the user may ask Codex to do next

- Fix the sticky header bug if it recurs (see above)
- Update `journals.json` when CAUL or Scimago release new data (will need source files re-merged — ask for them)
- Add/remove/reorder table columns
- Add new filter dropdowns
- Change branding/colours
- Help migrate hosting away from GitHub Pages (the user has asked about Netlify, Vercel, Cloudflare Pages, or a CSU-hosted server as alternatives — purely a matter of uploading the same two static files elsewhere, no code changes needed)
- Possibly rebuild the data pipeline if given new source spreadsheets — but treat this as a new task requiring the original CSV/Excel files, not something inferable from this handover alone

---

## 7. Hosting / deployment notes

- **Current:** GitHub Pages, repo `roseyraine/journal-finder`, deployed from `main` branch root
- **To update the live site:** overwrite `index.html` and/or `journals.json` in the repo (via GitHub web UI: Add file → Upload files → commit) — auto-deploys in 1–2 minutes
- **No build step, no npm, no compilation** — files are served as-is
- **LibGuides embed code** (already provided to user separately):
  ```html
  <iframe src="https://roseyraine.github.io/journal-finder/"
    width="100%" height="800px" style="border:none;border-radius:4px;"
    title="CSU Library Journal Finder" loading="lazy"></iframe>
  ```

---

## 8. Brand reference (if asked to adjust styling)

```
Navy (primary):     #222944
Orange (accent):    #F0572A
Orange (dark):      #DA3D0F
Charcoal (text):    #222222
Green:              #3C8364
Blue:               #5675BC
Pink (muted):       #E9CECA
Brown/tan (light):  #C7B8A0
Font: Arial throughout (no custom webfonts loaded)
```

---

## 9. Suggested first message to Codex

> "I'm handing over an existing static web project — a journal search tool for a university library, currently hosted on GitHub Pages. Please read the attached HANDOVER.md fully before making any changes. The two project files (`index.html` and `journals.json`) are attached. Confirm you understand the current sticky-header bug history before I describe what I need next."

This ensures Codex reads context before acting rather than guessing at the codebase cold.
