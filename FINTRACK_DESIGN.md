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
14. [Income vs Expense vs Transfer Logic](#14-income-vs-expense-vs-transfer-logic)
15. [Transaction Grouping (Year/Month)](#15-transaction-grouping-yearmonth)
16. [Key Design Decisions](#16-key-design-decisions)
17. [Known Patterns & Edge Cases](#17-known-patterns--edge-cases)

---

## 1. Project Overview

Fintrack is a single self-contained HTML file (~3,100 lines) for personal finance tracking. It imports bank and credit card CSV files, auto-categorises transactions using a statistical learning engine, and provides dashboards, monthly summaries, and export capabilities.

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
└── <script>         All application logic (~2,700 lines JS)
    ├── State management   (emptyState, saveState, loadSnapshot)
    ├── Utility functions  (fmt, escAttr, escJS, parseDate, etc.)
    ├── Learning engine    (descNgrams, updateStats, guessGroup)
    ├── Income/transfer    (isIncomeGroup, isTransferGroup, monthlyGroupTotals)
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
  learned:       { [ngram: string]: groupName },
  learned_stats: { [ngram: string]: { [groupName]: count } }
}
```

### Group

```javascript
{
  id:         number,    // auto-incrementing
  name:       string,
  color:      string,    // hex color from PALETTE
  isIncome:   boolean,   // true = counted as income in summaries
  isTransfer: boolean    // true = excluded from income/expense totals
}
```

**Group type detection (auto-fallback for old saves):**
- `isTransfer: true` → transfer group (excluded from Income/Expense)
- `isIncome: true` → income group
- `isIncome: undefined` + name contains `'משכורת'` → treated as income
- `isTransfer: undefined` + name starts with `'Transfer to'` → treated as transfer
- Otherwise → expense group

### Transaction

```javascript
{
  id:           string,   // "tx-NNN"
  date:         string,   // "YYYY-MM-DD"
  desc:         string,   // cleaned merchant name
  amount:       number,   // negative = expense, positive = income
  groupId:      number|null,
  source:       string,   // "Bank" | "Credit"
  comment:      string,
  edited:       boolean,
  isNew:        boolean,
  disabled:     boolean,  // excluded from all calculations
  splitParent:  boolean,  // disabled split origin row
  splitIntoTwo: boolean,  // disabled via "split into 2" action
  instalmentOf: string,   // parent tx id (instalment child)
  splitFromId:  string    // parent tx id (split-into-2 child)
}
```

### Persistence

- `fintrack_saves` — JSON array of `{id, name, savedAt, txCount}` (snapshot index)
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
2. Fuzzy match — guards against `description` matching `extended description`

### Description Selection

Always picks the **longer** of `description` vs `extended description` columns. This handles two opposite bank conventions:
- Credit card A: `extended description` has full merchant name
- Checking bank: `description` has full text, `extended description` is truncated

### Description Cleaning (`cleanDesc`)

Strips common bank prefixes. Handles both `Withdrawal ` (space) and `Withdrawal:` (colon) variants, `ACH Debit/Deposit`, `POS Transaction`, `Ext Credit Card Debit/Credit`, `Descriptive Deposit/Withdrawal`, `Deposit:`.

### Sign Validation

Expected convention: **Debit = negative, Credit = positive**.

After parsing, checks `Transaction Type` column. If violations found, import is **blocked** with error asking user to re-download from bank.

### Review Table

Before committing, all rows shown with:
- Status badge (new / already exists)
- Editable date, read-only description with ✎ edit button, editable amount, group dropdown
- Skip/Include toggle per row
- Duplicate rows pre-highlighted and pre-skipped
- Bulk: "Skip all duplicates" / "Include all"

**Critical:** All inputs use `oninput` (not `onchange`) to prevent browsers firing change events during chunk-based DOM insertion, which was corrupting amount signs.

### Chunk Rendering

Review tbody populated in chunks of 40 rows using `setTimeout(renderChunk, 0)` between batches.

---

## 6. Transaction Table

### Layout (tx-mode)

```
#content (flex column, overflow:hidden)
  └─ .tx-page
       ├─ .tx-header (sticky — never scrolls)
       │    ├─ stat cards
       │    ├─ copy/dup banners
       │    ├─ toolbar (search + filters + expand/collapse + add)
       │    └─ bulk bar
       └─ #tx-scroll (only this scrolls)
            └─ table with sticky thead
```

### Year/Month Grouping

Transactions are grouped into collapsible year and month rows:

```
▼ 2026  [12 months · 847 tx]  Income: +$141k  Expenses: -$156k  Net: -$14k
  ▼ June 2026  [28 tx]  Income: +$20,817  Expenses: -$24,697  Net: -$3,879
    ☐  01-Jun-2026  Credit Dividend  +0.10  ...
  ▶ May 2026  [31 tx]  ...
▶ 2025  [2 months · 134 tx]  ...
```

**State:** `expandedYears: Set<string>`, `expandedMonths: Set<string>` — module-level, persist across re-renders. Auto-expands most recent year+month on first load.

**Multiple open simultaneously:** clicking a year/month toggles it independently. Collapsing a year also collapses all its months.

**Toolbar:** `⊞ All` expands everything, `⊟ All` collapses everything.

### Scroll Preservation

`txScrollPos` is module-level. Saved from `#tx-scroll.scrollTop` before re-render, restored after (plus `requestAnimationFrame` fallback). Resets to 0 on sort/filter/search changes.

### Sorting

`setSort(col)` permanently sorts `state.transactions` in place. Editing never re-sorts.

### Group Copy/Paste (Ctrl+C / Ctrl+V)

1. Click group dropdown → `setActiveGrpRow(id)` + **blur the select** (critical: returns focus to document so keyboard events work)
2. `Ctrl+C` → copies `groupId`, shows blue banner
3. Click another row's group dropdown
4. `Ctrl+V` → pastes group, calls `learnRule`

### Bulk Selection

`toggleTx()` updates DOM in-place (no full re-render) for performance. `bulkAssign()` keeps selection after assigning.

### Context Menu (Right-click)

- ÷ Split into payments…
- ⧖ Split into 2 transactions (disables original, creates 2 editable copies)
- ↩ Unsplit
- ⊘ Disable / ✓ Enable
- ✎ Edit description
- 💬 Add/Edit note
- ✕ Delete

### Disabled Transactions

`r.disabled:true` → shown at 35% opacity, excluded from **all calculations** everywhere.

---

## 7. Group Management

### Resizable Panel

Two-column layout with drag handle. `initGroupResize()` handles mouse events. Width stored in `grpPanelWidth` (default 320px).

### Group Type Button

Cycles through three states (click to advance):
- `expense` (grey border)
- `income` (green border)  
- `transfer` (group-colour dashed border, excluded from Income/Expenses)

### Rename (`saveGrpName`)

1. Updates `g.name`
2. Updates all `state.learned` values pointing to old name
3. Re-runs `learnRule` for all transactions in that group

### Learned Rules Panel

Shows all n-grams in `state.learned_stats` with confidence bars, vote breakdown, threshold slider (50%–100%), ⟳ Learn button, manual add form.

---

## 8. Smart Learning Engine

### Data Structures

```javascript
state.learned_stats = {
  "trader joe": { "Groceries": 15 },
  "uber eats":  { "Dining": 6 },
  "uber":       { "Transport": 8, "Travel": 5 },
}
state.learned = {
  "trader joe": "Groceries",  // passed confidence threshold
  "uber eats":  "Dining",
  // "uber" excluded — 62% below threshold
}
```

### N-gram Extraction (`descNgrams`)

Lowercases, strips numbers/stop-words/state codes, generates all 1–3-grams, returns longest first.

### Thresholds

```
LEARN_CONF_THRESHOLD = 0.70  (70% of votes must be one group)
LEARN_MIN_OBS = 2            (minimum total observations)
LEARN_MIN_WIN = 2            (minimum winning count)
```

### Learn Only Via Button

Individual assignments don't auto-update `learned_stats`. The ⟳ Learn button does a full statistical recalculation, preventing single mis-categorisations from corrupting rules.

### `guessGroup(desc)`

Tries n-grams longest-first → returns first match in `state.learned`. More specific rules (`"uber eats"`) beat less specific (`"uber"`).

---

## 9. Dashboard & Charts

### Income / Expense / Transfer Definition

All views share `isIncomeGroup(g)` and `isTransferGroup(g)` helpers:

```javascript
function isTransferGroup(g) {
  return g ? !!g.isTransfer : false;
}
function isIncomeGroup(g) {
  if (!g || isTransferGroup(g)) return false;
  if (g.isIncome !== undefined) return !!g.isIncome;
  return g.name.includes('משכורת');
}
```

### Dashboard Chart

Bar chart (Income green, Expenses red) + Accumulated balance line (purple). Single Y-axis (`afterFit: axis.width=90`). All use group-based logic via `monthlyGroupTotals()`.

---

## 10. Groups Dashboard

### Matrix Table

Rows = groups, columns = months + Total. Sticky first column. Months with >20 transactions get ✓ badge (used for average).

### Mini Charts

Bar chart per group: absolute values, green/red by sign, dashed average line from qualifying months (>20 tx).

---

## 11. Monthly Summary

### Chart

Bar chart of Net per month — uses same group-based logic as month blocks (not raw transaction signs). Transfer groups excluded from net calculation.

### Month Blocks

Per month: Income / Expenses / Net stats + group chips.

**Transfer chips** shown with dashed border, neutral color, tooltip "Transfer (excluded from Income/Expenses)".

**Chip sort:** most negative first.

---

## 12. Export System

### Internal Data (JSON)

Full backup: groups, learned rules, transactions, date range, version.

### Transaction Export (CSV / XLSX)

One row per transaction. Split-parents excluded.

### Group Summary (CSV)

One row per group: name, income, expenses, net, count.

---

## 13. Session Management

### Snapshots

`saveSnapshot(name)` → `loadSnapshot(id)` → auto-sanitises on every load:

```javascript
state.learned       = state.learned       || {};
state.learned_stats = state.learned_stats || {};
state.groups        = state.groups        || [];
state.transactions  = state.transactions  || [];
// Auto-set flags for old saves:
state.groups.forEach(g => {
  if (g.isIncome === undefined && g.name.includes('משכורת'))
    g.isIncome = true;
  if (g.isTransfer === undefined && g.name.toLowerCase().includes('transfer to'))
    g.isTransfer = true;
});
```

---

## 14. Income vs Expense vs Transfer Logic

Three group types, consistent across all views:

| Type | Dashboard | Monthly Summary | Groups Dashboard | Transactions |
|------|-----------|-----------------|------------------|--------------|
| Income | ✓ Counted | ✓ Counted | ✓ Shown | ✓ Shown |
| Expense | ✓ Counted | ✓ Counted | ✓ Shown | ✓ Shown |
| Transfer | ✗ Excluded | ✗ Excluded (shown in chips) | ✓ Shown | ✓ Shown |

**Default transfer groups:** "Transfer to Savings", "Transfer to Investments".

**Why transfer groups?** Moving money to savings/investments is neither income nor expense — it would distort both totals. Excluding them gives accurate Income and Expense figures while still tracking where the money went.

---

## 15. Transaction Grouping (Year/Month)

### Motivation

With 1,500+ transactions spanning multiple years, a flat list is unusable. Collapsible year/month groups let the user focus on one period at a time.

### Implementation

`buildGroupedRows(rows, grpOpts, dupIds)` builds the tbody HTML:
- Iterates years descending
- Within each year, iterates months descending  
- Year/month rows are `<tr>` with `colspan="8"` and `onclick` handlers
- Transaction rows are normal `txRow()` output — all editing works unchanged

### State

```javascript
let expandedYears  = new Set();  // {"2026"}
let expandedMonths = new Set();  // {"2026-06", "2026-05"}
```

Auto-expands most recent year + month on first render.

### Performance

Only expanded months render full `txRow()` HTML. With 1,500 transactions and one month open (28 rows), initial render is near-instant.

---

## 16. Key Design Decisions

**Single File** — No build step, works offline, easy to share.

**localStorage Snapshots** — Multiple named sessions allow testing without overwriting production data.

**Longer column wins** — Always pick the longer of `description` vs `extended description`. Different banks use these columns in opposite ways.

**Sign validation on import** — Block if Transaction Type disagrees with amount sign. Better to fail loudly.

**`oninput` not `onchange` in review table** — `onchange` fires during chunk-based DOM insertion, corrupting amounts (strips minus signs from number inputs).

**Learn only via button** — Prevents mis-categorisations during a trip from corrupting rules permanently.

**No re-sort on edit** — Changing group/date/amount preserves row position. Only explicit column header clicks re-sort.

**Group-based Income/Expense** — Using raw transaction signs would count tax withholdings inside a salary group as expenses. Group nets give the meaningful number.

**Transfer group type** — Moving money between accounts should not inflate Income or Expenses. A dedicated transfer type excludes these from totals while keeping them visible everywhere else.

---

## 17. Known Patterns & Edge Cases

**Floating point amounts** — `Math.round(v * 100) / 100` before colour decisions prevents `-0.000000001` showing as red.

**Hebrew group names** — UTF-8 throughout. `'משכורת'` (salary) used as income auto-detection fallback keyword.

**Instalment badge ordering** — Siblings sorted by date ascending before `findIndex` — prevents badge number depending on current table sort order.

**Credit card sign convention** — Some banks export debits as positive. Import validates and blocks with error if violated.

**Ctrl+C interception** — `setActiveGrpRow()` calls `document.activeElement.blur()` immediately so keyboard events reach `document` handler, not the focused `<select>`.

**Chunk rendering race condition** — `oninput` instead of `onchange` on review table amount/date inputs prevents browser from firing change events during DOM insertion.

**Year/month grouping + search** — When search/filter is active, only groups containing matching transactions are shown. Expand/collapse state preserved independently.

---

*Last updated: July 2026*  
*Application: finance-tracker.html (~3,100 lines)*

---

## 18. Changelog — Latest Fixes

### Transfer group consistency in Transactions tab
- `groupStats(txRows)` helper added — computes income/expenses using `isIncomeGroup`/`isTransferGroup`, transfer groups excluded
- Year/month group header rows in `buildGroupedRows` now use `groupStats()` — previously used raw transaction signs, causing July 2026 to show wrong Net
- Transactions page stat cards (Income/Expenses/Net) also switched to `groupStats()`
- All four views (Dashboard, Monthly Summary, Groups Dashboard, Transactions) now use identical income/expense/transfer logic

### Monthly Summary chart fix
- "Net per month" bar chart was using raw transaction signs while month blocks used group-based logic
- Chart now uses same group-based calculation — Net bar matches the number shown in the month block below it

### Dashboard chart sign fixes
- Removed `Math.max(0,inc)` and `Math.min(0,exp)` clamping from `monthlyGroupTotals()` and `groupStats()` — income groups can net negative in some months (e.g. tax withholding month), expense groups can net positive (refunds)
- Dashboard chart bars now use `Math.abs()` for height with dynamic colour (green when income positive, red when negative)
- Stored original signed arrays `monthlyIncSigned`/`monthlyExpSigned` alongside absolute bar heights — tooltip callback uses `dataIndex` to look up real signed value, fixing incorrect `Expenses: -$0.00` display

---

## 19. Yearly Summary Page

### Overview
A new sidebar page ("Yearly summary") that analyses each year's finances, projects full-year figures for partial years, and breaks down actuals and projections per group.

**Minimum data threshold:** a year must have at least 3 months of data to appear.

---

### Layout

One card per year, most recent first. Each card has:

- Year label + status badge (Complete year / Partial — N of 12 months)
- From/To month range dropdowns (user-adjustable)
- 3 stat cards: Income / Expenses / Net — each shows Actual and (for partial years) Projected
- Optional warning if current calendar month is in the range
- Per-group breakdown table with mini bar charts

---

### Projection Formula

For partial years (selected range < 12 months):

```
avg_per_month = actual / months_in_range
projected_12  = actual + (avg_per_month * remaining_months)
```

Where remaining_months = 12 - months_in_range.

This gives: actual-so-far + estimated future, not just a scaled total.
For complete years: no projection shown — actual IS the answer.

---

### Range Selection

Each year card has From / To month dropdowns.

Defaults:
- From = earliest month with data in that year
- To = last complete month — if current calendar month is in this year, default To is the previous month (avoids partial-month distortion)

Behaviour on change: immediately recalculates all stats and redraws mini charts for that year.

State: yearRanges = { "2027": { from: "2027-01", to: "2027-04" } } — persists within session.

---

### Current Month Warning

If the current calendar month falls within the selected range:
"April 2027 is in progress — projection may be understated. Consider adjusting the end month."

---

### Per-group Breakdown

Three sections per year card: Income groups, Expense groups, Transfers.

Groups with zero activity in the selected range are hidden.

Columns:
- Group (coloured pill)
- Monthly trend (mini bar chart, one bar per month, green/red by sign, signed tooltip)
- Actual (sum over selected range)
- Avg / month (actual / months in range)
- Projected (12mo) — hidden for complete years

Groups sorted by absolute actual descending.

---

### Key Functions

| Function | Purpose |
|----------|---------|
| renderYearlySummary() | Main renderer — builds all year cards, draws charts |
| setYearRange(y, field, val) | Dropdown handler — re-renders single year body + charts |
| renderYearBody(...) | Returns HTML for one year's stats + group tables |
| drawYearCharts(y, from, to, tx) | Draws mini Chart.js bar charts for all active groups |

State variables: yearRanges, yrChartInstances, MIN_MONTHS_FOR_YEAR = 3

---

## 20. Monthly Summary Chart Fix

The Net per month bar chart had mismatched colours because data was reversed but backgroundColor/borderColor arrays were not.

Fix: compute netReversed = [...netByMonth].reverse() first, then derive all three arrays from the same reversed source. Also added proper signed tooltip: Net: +$4,123.45 / Net: -$8,234.56.

---

## 21. Transfer Group Consistency Fixes

- groupStats() helper added — computes income/expenses using isIncomeGroup/isTransferGroup, transfer groups excluded
- Year/month group header rows in buildGroupedRows now use groupStats()
- Transactions page stat cards also use groupStats()
- Dashboard chart stores original signed arrays (monthlyIncSigned/monthlyExpSigned) for correct tooltip display
- Removed Math.max(0,inc) and Math.min(0,exp) clamping — income groups can net negative in some months
