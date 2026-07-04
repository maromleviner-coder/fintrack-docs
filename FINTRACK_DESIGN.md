# Fintrack — Personal Finance Tracker
## Design & Development Documentation

> Single-file HTML application for personal finance tracking.  
> No server, no dependencies beyond Chart.js — everything lives in one `.html` file with localStorage persistence.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Data Model](#3-data-model)
4. [Pages & Navigation](#4-pages--navigation)
5. [Import System](#5-import-system)
6. [Transaction Table](#6-transaction-table)
7. [Group Management](#7-group-management)
8. [Smart Learning Engine](#8-smart-learning-engine)
9. [Dashboard & Charts](#9-dashboard--charts)
10. [Groups Dashboard](#10-groups-dashboard)
11. [Monthly Summary](#11-monthly-summary)
12. [Export System](#12-export-system)
13. [Session Management](#13-session-management)
14. [Income vs Expense Logic](#14-income-vs-expense-logic)
15. [Key Design Decisions](#15-key-design-decisions)
16. [Known Patterns & Edge Cases](#16-known-patterns--edge-cases)

---

## 1. Project Overview

Fintrack is a single self-contained HTML file (~3,000 lines) for personal finance tracking. It imports bank and credit card CSV files, auto-categorises transactions using a statistical learning engine, and provides dashboards, monthly summaries, and export capabilities.

**Core constraints:**
- No server — runs entirely in the browser from a local file
- All persistence via `localStorage` named snapshots
- Single file: HTML + CSS + JS, Chart.js loaded from CDN

---

## 2. Architecture

```
finance-tracker.html
├── <style>          CSS (variables, layout, components)
├── <body>           Shell: sidebar nav + topbar + content div
└── <script>         All application logic (~2,500 lines JS)
    ├── State management   (emptyState, saveState, loadSnapshot)
    ├── Utility functions  (fmt, escAttr, escJS, parseDate, etc.)
    ├── Learning engine    (descNgrams, updateStats, guessGroup)
    ├── Page renderers     (renderDashboard, renderTransactions, ...)
    ├── Import handlers    (handleBankFile, confirmImport, ...)
    ├── Export handlers    (doExport, doGroupExport, ...)
    └── Event handlers     (keyboard shortcuts, context menu, ...)
```

### Layout

```
┌─ sidebar (220px) ─┬─ main (flex:1) ──────────────────────────┐
│  nav items        │  topbar (page title + action buttons)     │
│                   ├──────────────────────────────────────────│
│                   │  #content (scrollable, or tx-mode flex)  │
└───────────────────┴──────────────────────────────────────────┘
```

The transactions page uses a special `tx-mode` layout where `#content` becomes a flex column: sticky header section (stat cards + banners + toolbar + bulk bar) + scrollable `#tx-scroll` table below.

---

## 3. Data Model

### State Object

```javascript
state = {
  groups:        Group[],
  transactions:  Transaction[],
  learned:       { [ngram: string]: groupName },      // fast lookup
  learned_stats: { [ngram: string]: { [groupName]: count } }  // raw vote counts
}
```

### Group

```javascript
{
  id:       number,    // auto-incrementing
  name:     string,
  color:    string,    // hex color from PALETTE
  isIncome: boolean    // true = counted as income in summaries
                       // undefined = auto-detect from name (משכורת = income)
}
```

### Transaction

```javascript
{
  id:          string,   // "tx-NNN"
  date:        string,   // "YYYY-MM-DD"
  desc:        string,   // cleaned merchant name
  amount:      number,   // negative = expense, positive = income
  groupId:     number|null,
  source:      string,   // "Bank" | "Credit"
  comment:     string,
  edited:      boolean,
  isNew:       boolean,
  disabled:    boolean,  // excluded from all calculations
  splitParent: boolean,  // disabled split origin row
  splitIntoTwo:boolean,  // disabled via "split into 2" action
  instalmentOf:string,   // parent tx id (instalment child)
  splitFromId: string    // parent tx id (split-into-2 child)
}
```

### Persistence

- `fintrack_saves` — JSON array of `{id, name, savedAt}` (snapshot index)
- `fintrack_save_<id>` — full serialised `state` for each snapshot
- `fintrack_last` — id of last opened snapshot (auto-resume)

---

## 4. Pages & Navigation

| Page | Key | Renderer |
|------|-----|----------|
| Dashboard | `dashboard` | `renderDashboard()` |
| Groups Dashboard | `groupsdash` | `renderGroupsDashboard()` |
| Transactions | `transactions` | `renderTransactions()` |
| Monthly Summary | `summary` | `renderSummary()` |
| Group Management | `groups` | `renderGroups()` |
| Import | `import` | `renderImport()` |
| Export | `export` | `renderExport()` |
| Save Session | `session` | (welcome screen) |
| Recent Files | `recent` | (welcome screen) |

`showPage(p)` handles nav highlight, title update, and calls the renderer. On leaving the transactions page it removes the `tx-mode` class from `#content`.

---

## 5. Import System

### Supported Formats
- Bank CSV (auto-detects columns)
- Credit card CSV (auto-detects columns)
- Internal JSON export (full restore)
- Internal CSV export (transactions only)

### Column Detection (`colIdx`)

Two-pass matching:
1. Exact match on lowercased header
2. Fuzzy match — but guards against `description` matching `extended description` by skipping headers that contain a longer candidate name

### Description Selection

Always picks the **longer** of `description` vs `extended description` columns. This handles two opposite bank conventions:
- Credit card A: `extended description` has full merchant name, `description` is short
- Checking bank: `description` has full text, `extended description` is truncated

### Description Cleaning (`cleanDesc`)

Strips common bank prefixes using regex:
```
"Ext Credit Card Debit TRADER JOE S #127 LOS ALTOS CA"
  → "TRADER JOE S #127 LOS ALTOS CA"

"Withdrawal: Leviner Pmt to *8918 Choice Rewards World MC"
  → "Leviner Pmt to *8918 Choice Rewards World MC"
```

Handles both `Withdrawal ` (space) and `Withdrawal:` (colon) variants.

### Sign Validation

The expected convention is: **Debit = negative amount, Credit = positive amount**.

After parsing, the code checks `Transaction Type` column. If violations found (Debit + positive, or Credit + negative), the import is **blocked** with a clear error message asking the user to re-download from the bank.

### Review Table

Before committing, all rows are shown in a review table with:
- Status badge (new / already exists)
- Editable date, read-only description with ✎ edit button, editable amount, group dropdown
- Skip/Include toggle per row
- Duplicate rows (same date + amount + desc) pre-highlighted and pre-skipped
- Bulk controls: "Skip all duplicates" / "Include all"

**Critical implementation detail:** All inputs in the review table use `oninput` (not `onchange`) to prevent browsers firing change events during chunk-based DOM insertion.

### Chunk Rendering

With 177+ rows, building one giant `innerHTML` string caused browser crashes. The review tbody is populated in chunks of 40 rows using `setTimeout(renderChunk, 0)` between chunks, yielding control to the browser between batches.

### Duplicate Detection

`buildDupSet()` creates a Set of transaction IDs matching existing records by `date + amount + desc.trim().toLowerCase()`. Split parents and instalments are excluded.

---

## 6. Transaction Table

### Layout

`tx-mode` layout:
```
#content (flex column, overflow:hidden)
  └─ .tx-page
       ├─ .tx-header (flex-shrink:0 — never scrolls)
       │    ├─ stat cards
       │    ├─ copy/dup banners
       │    ├─ toolbar (search + filters + add button)
       │    └─ bulk bar
       └─ #tx-scroll (flex:1, overflow-y:auto — only this scrolls)
            └─ table with sticky thead
```

### Scroll Preservation

`txScrollPos` is a module-level variable that persists across re-renders. Saved from `#tx-scroll.scrollTop` at the start of `renderTransactions()`, restored after innerHTML replacement (both immediately and in a `requestAnimationFrame` fallback).

Resets to 0 on: sort column click, filter change, search input.

### Sorting

`setSort(col)` permanently sorts `state.transactions` in place and saves state. This means sort order persists across page navigation and browser refresh. `sortedVis()` simply returns the visible (filtered) array as-is — no re-sorting during render.

**Editing never re-sorts.** Changing a group, date, amount, or comment preserves row position.

### Group Copy/Paste (Ctrl+C / Ctrl+V)

1. Click any group dropdown → `setActiveGrpRow(id)` marks it and **blurs the select** (critical: moves focus away so keyboard events reach `document`)
2. `Ctrl+C` → copies `groupId` to `copiedGroupId`, shows blue banner
3. Click another row's group dropdown → marks new active row
4. `Ctrl+V` → pastes group, calls `learnRule`, re-renders

### Bulk Selection

Checkbox column + bulk bar. `toggleTx()` updates DOM in-place (no full re-render) for performance — just updates row class and checkbox, plus the bulk bar count. `bulkAssign()` keeps selection after assigning (so user can re-assign or see what changed), calls `learnRule` for each row.

### Context Menu (Right-click)

Right-click any row to get:
- ÷ Split into payments… (instalment split)
- ⧖ Split into 2 transactions
- ↩ Unsplit
- ⊘ Disable / ✓ Enable
- ✎ Edit description
- 💬 Add/Edit note
- ✕ Delete

### Split into Payments

Modal with count input and live preview. Creates N instalment transactions with `instalmentOf` pointing to the parent. Parent is marked `splitParent:true` (dimmed, excluded from totals).

Instalment badge shows `3/12` format. **Siblings are sorted by date ascending** before `findIndex` to ensure correct numbering regardless of table sort order.

### Split into 2

Disables original (marked `splitIntoTwo:true`, shown as "split origin" badge), creates 2 copies for user to edit amounts manually.

### Disabled Transactions

`r.disabled:true` rows are shown at 35% opacity with a "disabled" badge. They are excluded from **all calculations** across every page — totals, charts, group dashboard matrix, exports.

---

## 7. Group Management

### Groups Page Layout

Resizable two-column layout:
- Left: transaction list (flex:1)  
- Drag handle (8px, `cursor:col-resize`)
- Right: groups panel (default 320px, min 180px, max 600px)

`initGroupResize()` attaches `mousedown` → `mousemove` → `mouseup` listeners. The panel width is stored in `grpPanelWidth`.

### Group Row Controls (on hover)

- Colour swatch → colour picker
- Name (double-click to rename, or ✎ button)
- Usage count badge
- ↑ ↓ reorder buttons (disabled at top/bottom)
- `income` / `expense` toggle button
- ✕ delete

### Rename (`saveGrpName`)

On rename:
1. Updates `g.name` in `state.groups`
2. Updates all `state.learned` values pointing to old name
3. Re-runs `learnRule` for all transactions in that group (rebuilds keyword-format rules replacing any old raw-description keys)

### Learned Rules Panel

Shows all n-grams in `state.learned_stats` with confidence bars:
- ✓ green = active rule (passed threshold)
- ✗ grey = tracked but ambiguous
- Confidence bar (green/amber/red)
- Vote breakdown per group
- Confidence threshold slider (50%–100%)
- ⟳ Learn button (rebuilds from all transactions)
- Manual add rule form

---

## 8. Smart Learning Engine

### Data Structures

```javascript
state.learned_stats = {
  "trader joe":  { "Groceries": 15 },
  "uber eats":   { "Dining": 6 },
  "uber":        { "Transport": 8, "Travel": 5 },
}

state.learned = {
  "trader joe": "Groceries",   // passed confidence threshold
  "uber eats":  "Dining",
  // "uber" excluded — 62% confidence, below threshold
}
```

### N-gram Extraction (`descNgrams`)

For `"TRADER JOE S #127 LOS ALTOS CA"` after cleaning:
- Lowercased, numbers stripped, 2-letter trailing state stripped
- Words filtered against stop words
- Generates all 1, 2, 3-grams
- Returns longest first (specificity order)

### Learning (`updateStats`)

Called **only** from the ⟳ Learn button. Scans all assigned, non-disabled, non-split-parent transactions. For each, generates all n-grams and increments vote counts in `learned_stats`.

### Confidence Thresholds

```
LEARN_CONF_THRESHOLD = 0.70   (70% of votes must be for one group)
LEARN_MIN_OBS = 2             (minimum total observations)
LEARN_MIN_WIN = 2             (minimum winning count)
```

### `rebuildLearned(threshold)`

Sorts entries by n-gram length descending (longer = more specific). For each n-gram, checks if top group meets threshold and minimum counts. Writes to `state.learned`.

### `guessGroup(desc)`

Generates n-grams for the description, tries longest first, returns first match in `state.learned`. More specific rules (`"uber eats"`) beat less specific (`"uber"`).

### Why "Learn only via button"

Individual group assignments no longer auto-update `learned_stats`. This prevents a single mis-categorisation during a trip from permanently corrupting rules. The Learn button does a full statistical recalculation over all existing data.

---

## 9. Dashboard & Charts

### Income / Expense Definition

Income and expenses are computed from **group nets**, not raw transaction signs:

```javascript
// For each month:
//   1. Sum all transactions per group → grpNet
//   2. Income groups (isIncome flag or name contains 'משכורת') → totalInc
//   3. Expense groups → totalExp

function isIncomeGroup(g) {
  if (!g) return false;
  if (g.isIncome !== undefined) return !!g.isIncome;
  return g.name.includes('משכורת');  // fallback name detection
}
```

This ensures Dashboard, Monthly Summary, and Groups Dashboard all show consistent numbers.

### Dashboard Chart

Bar chart (Income green, Expenses red) + line (Accumulated balance, purple):
- All bars go **upward** (absolute values plotted)
- Colour indicates sign (green = income, red = expense)
- Line uses single Y-axis shared with bars (`afterFit: axis.width=90` prevents label clipping)
- Tooltip shows real signed values
- Accumulated balance line points colour: purple when positive, red when negative

---

## 10. Groups Dashboard

### Matrix Table

Rows = groups, Columns = months + Total + (no avg in table).

- Sticky first column (group name)
- Month columns show transaction count in sub-header
- Months with > 20 transactions get a ✓ badge and full opacity; others dimmed
- Qualifying months (> 20 tx) used for average calculation in mini charts

### Mini Charts (per group)

- Bar chart, absolute values, green = positive month, red = negative month
- Dashed horizontal average line (only shown when qualifying months exist)
- Average value displayed in card header: `avg -$419.73`
- Tooltip on bars shows real signed value

---

## 11. Monthly Summary

Per-month breakdown cards showing:
- **Income**: sum of net for all income groups
- **Expenses**: sum of net for all expense groups  
- **Net**: Income + Expenses
- Group chips sorted most-negative first, coloured green/red by sign

---

## 12. Export System

### Internal Data (JSON)

Full backup including:
```json
{
  "exported_at": "...",
  "version": "1.0",
  "date_range": { "from": "...", "to": "..." },
  "groups": [...],
  "learned_rules": { "trader joe": "Groceries", ... },
  "transactions": [...]
}
```

### Transaction Export (CSV / XLSX)

One row per transaction within date range. Columns: date, description, amount, group, source, comment, edited, id.

### Group Summary Export (CSV)

One row per group: group name, income, expenses, net, transaction count. Split-parent rows excluded from sums.

### Export Preview

Live preview table updates as date range or format changes.

---

## 13. Session Management

### Snapshots

- `saveSnapshot(name)` — serialises `state` to localStorage
- `loadSnapshot(id)` — deserialises, sanitises (ensures all required fields exist), auto-sets `isIncome` for משכורת groups
- `autoSave()` — silent save, only runs when `_autoSaveId` is set

### Welcome Screen

Shown on first open or when no last session. Lists recent saves with Open / Rename / Delete. User can also load from a JSON export file directly.

### Sanitisation on Load

Every load path (`loadSnapshot`, `welcomeLoadFile`) ensures:
```javascript
state.learned      = state.learned      || {};
state.learned_stats= state.learned_stats|| {};
state.groups       = state.groups       || [];
state.transactions = state.transactions || [];
// Auto-flag income groups by name if flag not explicitly set:
state.groups.forEach(g => {
  if (g.isIncome === undefined && g.name.includes('משכורת'))
    g.isIncome = true;
});
```

---

## 14. Income vs Expense Logic

The income/expense split is **group-based**, not transaction-sign-based.

### Why not transaction signs?

A salary group (`משכורת`) may contain both positive (salary deposits) and negative (tax withholdings) transactions. Using raw signs would count the tax payments as "expense" even though they belong to the income group. Group-based logic gives the net for the entire group, which is the meaningful number.

### Configuration

Each group has an `isIncome` flag toggled in the Groups page. Default detection:
- `isIncome` explicitly set → use it
- `isIncome` undefined AND name contains `"משכורת"` → treated as income
- Otherwise → expense

### Consistency

All four views use the shared `isIncomeGroup(g)` helper:
1. Dashboard stat cards (all-time totals)
2. Dashboard chart (monthly bars)
3. Monthly Summary (per-month Income/Expenses)
4. Groups Dashboard (matrix and mini charts)

---

## 15. Key Design Decisions

### Single File
No build step, no dependencies to install, works offline. The tradeoff is a large single file that requires careful editing.

### localStorage Snapshots
Multiple named sessions allow testing imports without overwriting production data. The snapshot index allows rename/delete without touching transaction data.

### "Learn only via button"
Prevents accidental learning from mis-categorisations. One-time statistical batch analysis is more reliable than incremental updates.

### Longer column wins for description
When a bank CSV has both `description` and `extended description`, always pick the longer one. Different banks use these columns in opposite ways.

### Sign validation on import
Block import if Transaction Type disagrees with amount sign. Better to fail loudly than silently import wrong data. The user is told to re-download.

### No re-sort on edit
Changing a group, date, or amount preserves row position. Only explicit column header clicks re-sort. This prevents the table "jumping" while categorising.

### `oninput` not `onchange` in review table
`onchange` fires when an input loses focus AND the value differs from when it gained focus. Some browsers fire this during chunk-based DOM insertion, corrupting amounts (stripping minus signs from number inputs). `oninput` only fires on actual user typing.

---

## 16. Known Patterns & Edge Cases

### Floating Point Amounts
All displayed amounts are rounded to 2 decimal places before colour/sign decisions using `Math.round(v * 100) / 100` to prevent `-0.000000001` showing as red when the display value is `$0.00`.

### Hebrew Group Names
The codebase uses Hebrew text directly (UTF-8). The `משכורת` (salary) keyword is used as the income group auto-detection fallback.

### Chunk Rendering Race Condition
When `showImportReview` renders rows in chunks via `setTimeout`, any `onchange` event firing during insertion could corrupt field values. Solution: use `oninput` for amount/date fields, and make description read-only with an explicit edit button.

### Instalment Badge Ordering
`findIndex` among instalment siblings must sort by date ascending first — otherwise the badge number depends on the current table sort order, not the instalment sequence.

### Credit Card Sign Convention
Some banks export credit card debits as positive amounts (wrong convention). The import validates that Debit = negative and Credit = positive, and blocks with an error if violated.

### Ctrl+C Interception
When a `<select>` has focus, browsers intercept Ctrl+C for their own copy. Fix: `setActiveGrpRow()` immediately calls `document.activeElement.blur()` after registering the row, returning focus to document so the keyboard handler receives the event.

---

*Last updated: July 2026*  
*Application: finance-tracker.html (~3,000 lines)*
