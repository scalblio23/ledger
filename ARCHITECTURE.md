# Architecture

## Overview

Single-file HTML application — all HTML, CSS, and JavaScript lives in `index.html` (~2,750 lines). No build step, no npm, no bundler. Deployed as a static site on Vercel.

```
ledger/
├── index.html      # Entire application
└── vercel.json     # Rewrites all routes → index.html
```

---

## Data Storage

All data is persisted to **`localStorage`** in the user's browser. There is no backend.

| Key | Contents |
|-----|----------|
| `ledger_session` | Logged-in email |
| `ledger_data_v2_<email>` | Main ledger state (`accounts`, `entries`, `cashflow`, `payments`, `paylog`) |
| `ledger_bd_prefs_<email>` | Breakdown preferences (`overrides`, `customCats`, `typeOverrides`, `costOverrides`) |

The `store` object wraps `localStorage` with `load()` / `save()` helpers. Data is namespaced per email so multiple users on the same browser stay isolated.

---

## Authentication

No real auth. A hardcoded credential check (`USERS` map of email → password) gates access. On success, the email is written to `localStorage` as the session token and used to namespace all data keys.

---

## Pages / Navigation

Five pages rendered into `<div id="mainWrap">`, toggled via CSS class `active`:

| Page | ID | Render function |
|------|----|-----------------|
| Overview | `page-overview` | `render()` |
| Cash Flow | `page-cashflow` | `renderCashflow()` |
| Payments | `page-payments` | `renderPayments()` |
| Wages | `page-wages` | `renderWages()` |
| Breakdown | `page-breakdown` | `renderBreakdown()` |

Navigation clicks toggle the `active` class and call the appropriate render function.

---

## Core Data Model

```js
state = {
  accounts: [{ name, balance }],   // tracked bank/wallet accounts
  entries:  [{                      // all transactions
    id, type ('in'|'out'), title,
    account, date, amount,          // AUD
    origUsd, usdRate                // set when entered as USD
  }],
  cashflow: [...],
  payments: [...],
  paylog:   [...]
}
```

USD amounts are converted to AUD at a fixed rate of **1.44** at entry time. The converted AUD value is stored; the original USD and rate are preserved for display.

---

## Breakdown Module

The most complex section (~1,500 lines). State is held in module-level variables:

- `bdView` — active sub-tab (`overview` | `business` | `personal` | `pl` | `raw`)
- `bdDateFilter` — `{from, to, preset}` applied before any sub-renderer runs
- `bdOverrides` — per-entry category overrides (map of `id → category name`)
- `bdTypeOverrides` — per-entry business/personal classification
- `bdCostOverrides` — per-entry COGS/Overhead/N/A classification
- `bdCustomCats` — user-created category definitions

### Data flow through Breakdown

```
state.entries
    ↓ bdFilterEntries()       ← date filter applied here
bdEntries (filtered)
    ↓ passed to sub-renderer
renderBreakdownOverview()     ← category donut + biz/personal split
renderBreakdownType()         ← business or personal expense breakdown
renderBreakdownPL()           ← profit & loss summary + period tiles
renderBreakdownRaw()          ← spreadsheet table of all transactions
```

### P&L Calculation

```
Total Sales    = sum of non-transfer IN entries
− COGS         = sum of OUT entries where costOverride === 'cogs'
= Gross Profit
− Overhead     = sum of OUT entries where costOverride === 'overhead'
= Net Profit

Monthly tile   = Net Profit (over filtered period)
Weekly tile    = Monthly ÷ 4.3 × (dayOfMonth / 30)
Daily tile     = Weekly ÷ 7
```

### Transfer Detection

An entry is treated as a transfer (excluded from P&L and category counts) if:
1. Its title starts with `"Transfer to "` or `"Transfer from "` (auto-detected), **or**
2. Its category override is manually set to `"Transfer"`

### Undo / Redo (Raw view)

`_undoStack` / `_redoStack` hold `{field, eid, prev, next}` entries. Fields: `type`, `cost`, `category`. Keyboard shortcuts: `Ctrl+Z` / `Ctrl+Y` / `Ctrl+Shift+Z`. Only active when Raw tab is focused.

### Fill Handle (Raw view)

Drag from the handle on a selected Type or Cost cell to copy its value down a range. Uses `mousemove` / `mouseup` global listeners during drag; cleaned up on release.

---

## External Services / APIs

None at runtime. The app is fully client-side.

- **Vercel** — static hosting, auto-deploys from `main` branch
- **GitHub** — source at `scalblio23/ledger`, feature branch `claude/html-vercel-deploy-ew6lmt`

---

## Non-Obvious Design Decisions

**Single file** — deliberate. Eliminates build tooling, keeps deployment to a single `git push`, and makes the whole app auditable in one read.

**No framework** — vanilla JS with manual DOM diffing (`innerHTML` replacement per render). Fast for this data size; no reconciliation overhead.

**Re-render on every mutation** — `render()` and `renderBreakdown()` are called after every state change. No reactive bindings. Works fine for ≤ a few hundred transactions.

**Global `window.*` handlers** — inline `onclick` attributes in generated HTML can't close over local variables, so interactive handlers are attached to `window` (e.g. `window.bdMoveTx`, `window.bdApplyCost`). This is the standard workaround for `innerHTML`-based rendering.

**Sort order locked on first Breakdown render** — `bdSortedOrder` is set once so the category order in the donut/bars doesn't jump as amounts change mid-session. Cleared on page reload.

**Breakdown prefs survive page reload** — `bdOverrides`, `bdTypeOverrides`, `bdCostOverrides`, `bdCustomCats` are serialised to `localStorage` on every change via `bdSavePrefs()` and restored at boot via `bdLoadPrefs()`.
