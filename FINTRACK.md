# Fintrack — Personal Finance PWA

A 100% offline personal finance tracker built as a single-file Progressive Web App. All data lives in the browser via IndexedDB — no server, no account, no subscription.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Tech Stack](#tech-stack)
3. [Architecture](#architecture)
4. [Feature Phases](#feature-phases)
5. [Database Schema](#database-schema)
6. [Key Algorithms](#key-algorithms)
7. [UI / Design System](#ui--design-system)
8. [PWA Capabilities](#pwa-capabilities)
9. [File Structure](#file-structure)
10. [Deployment](#deployment)
11. [Development Notes](#development-notes)

---

## Project Overview

Fintrack is a mobile-first personal finance app for tracking expenses, income, investments, and budgets. It was built iteratively in four phases:

| Phase | Theme | Key Additions |
|-------|-------|---------------|
| 1 | Core | Manual transactions, accounts, dashboard, categories |
| 2 | Import & Auto-Categorization | Bank CSV import, merchant intelligence, deduplication |
| 3 | Reports & Analytics | 6 report views, budgets, CSV export |
| 4 | Security & Polish | PIN lock, category manager, live search, recurring detector |

**Design principles:**
- Entirely offline — no network calls after first load
- Single HTML file — no build toolchain, no npm, no bundler
- Mobile-first — max-width 480 px, safe-area notch support
- Zero external dependencies — custom CSS, Canvas charts, Web Crypto

---

## Tech Stack

| Layer | Choice | Notes |
|-------|--------|-------|
| Language | Vanilla JavaScript (ES2020+) | No frameworks |
| Storage | IndexedDB | All user data, offline-first |
| Styling | Custom CSS with design tokens | No Tailwind / Bootstrap |
| Charts | Native Canvas 2D API | No Chart.js or D3 |
| Security | Web Crypto API (SHA-256) | PIN hashing |
| PWA | Service Worker + Web Manifest | Offline cache, home-screen install |

---

## Architecture

The app is structured as cooperating modules inside a single `<script>` block.

### DB Module

IndexedDB wrapper providing promise-based CRUD over 7 object stores.

```
DB.getAll(store)
DB.get(store, key)
DB.put(store, record)
DB.putBulk(store, records[])
DB.remove(store, key)
DB.clearStore(store)
DB.checkHashes(hashes[])   // deduplication
DB.storeHashes(hashes[])
```

### State Object

Central mutable store that caches the current DB snapshot. Refresh on every screen transition.

```javascript
State = {
  accounts: [],
  transactions: [],
  mappings: [],
  activeScreen: "dash",
  activeMonth: "YYYY-MM",
  txFilter: "all",        // all | expense | income | investment | unreviewed
  searchQuery: "",
  reportPrefs: { tab, monthsWindow },
}
State.refresh()           // re-reads from DB
State.txInMonth()         // transactions for activeMonth
State.accountById(id)
State.txById(id)
```

### Utils Module

| Function | Purpose |
|----------|---------|
| `uid()` | Random unique ID |
| `today()` | Current date as YYYY-MM-DD |
| `fmtDate(d)` | Display format (e.g. "15 May") |
| `monthLabel(ym)` | "May 2024" |
| `shiftMonth(ym, n)` | Add/subtract months |
| `currentYM()` | Current YYYY-MM |
| `fmtAmt(n)` | INR with K/L/Cr abbreviation |
| `parseAmt(s)` | String → float |
| `txHash(date, amt, desc)` | Dedup fingerprint |

### CATS Module

26 built-in categories. Each:
```javascript
{ id, name, icon, type: "expense"|"income"|"investment", color }
```
- O(1) lookup via `CATS.byId` map
- Phase 4: mutable via `_updateOrAdd()`, `_remove()`
- Phase 2: name→id mapping for CSV auto-categorization

### MerchantEngine

Three-tier merchant resolution pipeline (runs in order, first match wins):

```
1. User-defined mappings (stored in DB)
2. Built-in dictionary  (100+ merchants)
3. Fuzzy token match    (5+ char prefix)
```

Input: raw bank description string  
Output: `{ merchant, category, icon, brand, src }` or `null`

Pre-processing strips UPI/NEFT/IMPS prefixes before matching.

Built-in dictionary covers: Swiggy, Zomato, Amazon, Flipkart, Netflix, Spotify, YouTube, Airtel, Jio, ICICI, HDFC, Uber, Ola, BookMyShow, Myntra, and 85+ more.

### BankParser

CSV tokenizer with RFC 4180 compliance (quoted fields, CRLF). Auto-detects bank format by sniffing header columns.

Supported formats:

| Bank | Statement Types |
|------|----------------|
| HDFC | Savings account, Credit card |
| ICICI | Savings account, Credit card |
| Axis | Savings account |
| SBI | Savings account |
| Scapia | Credit card |
| Generic | Unknown CSV (best-effort) |

Output: `{ bank, stmtType, txns[], total }`

Date parsing handles: `DD/MM/YYYY`, `MMM DD YYYY`, `DD-MM-YYYY`.

### Charts

All charts rendered to `<canvas>` — no external library.

| Function | Chart Type | Used In |
|----------|-----------|---------|
| `donut()` | Donut/pie | Spending breakdown |
| `bars()` | Vertical bars | Monthly trends (dashboard) |
| `line()` | Multi-line | Savings rate, income/expense trends |
| `hbars()` | Horizontal bars | Top merchants |
| `groupedBars()` | Side-by-side bars | Income vs expense |

### UI Module

- `UI.toast(msg, type)` — success / error / info / warning, 2.8 s auto-dismiss
- `UI.openModal(id)` / `UI.closeModal(id)` — modal management
- Bottom nav: 6 tabs with badge support (unreviewed count)

---

## Feature Phases

### Phase 1 — Core

**Screens:**
- **Dashboard** — Net worth hero card, monthly stats (spent / earned / invested / savings), category donut, 6-month trend bars, recent transactions list, budget alerts
- **Transactions** — Month-based list grouped by date, type filter, add/edit modal
- **Accounts** — 4 account types (savings, current, credit, investment), balance cards, add/edit/delete
- **Settings** — Account shortcuts, data export/import

**Core behaviour:**
- Manual transaction entry (date, amount, category, account, notes)
- Account balance auto-updated on add/edit/delete
- Month navigation (← →)

---

### Phase 2 — Import & Auto-Categorization

**New screens:**
- **Import** — CSV upload (drag-drop or file picker), bank auto-detect, preview table, deduplication report
- **Review Queue** — Unresolved transactions needing manual category tag; "apply to all similar" bulk-rule creation

**Features:**
- Parse 5 bank formats
- Merchant auto-categorization via MerchantEngine
- Transaction deduplication: hash = `date|amount|description`, skip already-seen hashes
- User merchant mappings saved to DB (`mappings` store)
- Nav badge shows unreviewed count

---

### Phase 3 — Reports & Analytics

**Reports tab — 6 sub-views:**

| Tab | Content |
|-----|---------|
| Spending | Category donut + table with % share, smart insights |
| Trends | Grouped bars (income vs expense), monthly table |
| Savings | Savings rate % line chart, 50/30/20 benchmark |
| Merchants | Top 15 ranked horizontal bars, category + tx count |
| Investments | Monthly investment bars, breakdown by type |
| Budgets | Progress bars per category, set/edit limits |

**Controls:**
- Date window: 1 / 3 / 6 / 12 months
- Account filter: All accounts or single account
- CSV export for every report

**Smart insights:**
- Spending delta alerts (this month vs last)
- Savings rate vs 20% target
- Anomaly detection in spending patterns

---

### Phase 4 — Security & Polish

**PIN Lock:**
- 4-digit PIN, numeric keypad UI
- Hash stored as SHA-256(PIN + salt) in `settings` store
- Lock screen shown on boot if PIN is set
- Set / change / remove flows with confirmation step

**Category Manager (in Settings):**
- Full CRUD for categories
- Emoji picker (60 emoji options)
- Color picker (18 colors)
- Type: expense / income / investment
- Transactions auto-remapped when a category is deleted

**Live Search:**
- Search icon in Transactions header
- Searches: merchant, description, notes, amount, category name
- Results grouped by date, matching text highlighted
- 180 ms debounce

**Recurring Detector:**
- Identifies subscriptions: same merchant appearing in 2+ distinct months with coefficient of variation < 15% on amounts
- Shown on Dashboard with estimated next due date
- Highlighted if due within 7 days

---

## Database Schema

| Object Store | Key Path | Indexes | Purpose |
|-------------|----------|---------|---------|
| `accounts` | `id` | — | Bank accounts & cards |
| `transactions` | `id` | `by_date`, `by_account`, `by_status` | All financial entries |
| `categories` | `id` | — | Expense / income / investment categories |
| `mappings` | `id` | `by_pattern` | User merchant-to-category rules |
| `hashes` | `h` | — | Dedup fingerprints |
| `budgets` | `id` | — | Per-category monthly spend limits |
| `settings` | `k` | — | PIN hash, app preferences |

### Transaction Record

```javascript
{
  id:          "uid",
  date:        "2024-05-15",          // YYYY-MM-DD
  description: "SWIGGY ORDER 12345",  // raw bank text
  merchant:    "Swiggy",              // resolved name
  categoryId:  "c-fd",
  accountId:   "acc-hdfc",
  type:        "expense",             // expense | income | investment | transfer
  amount:      450.00,
  notes:       "Dinner with team",
  status:      "reviewed",            // reviewed | unreviewed
  source:      "hdfc",               // manual | bank name
  hash:        "abc123"              // dedup key
}
```

### Account Record

```javascript
{
  id:      "acc-hdfc",
  name:    "HDFC Savings",
  type:    "savings",                 // savings | current | credit | investment
  balance: 125000.00,
  color:   "#0A84FF"
}
```

### Budget Record

```javascript
{
  id:         "bgt-c-fd",
  categoryId: "c-fd",
  limit:      5000.00                 // monthly limit in INR
}
```

---

## Key Algorithms

### Merchant Resolution

```
cleanDesc = stripUPIPrefixes(rawDescription)

for mapping in userMappings (sorted by specificity):
    if cleanDesc.includes(mapping.pattern): return mapping

for entry in builtinDictionary:
    for pattern in entry.patterns:
        if cleanDesc.includes(pattern): return entry

tokens = cleanDesc.split(" ").filter(t => t.length >= 5)
for token in tokens:
    if fuzzyMatch(token, dictionaryNames): return match

return null
```

### Deduplication

```
hash = SHA256-like( date + "|" + amount + "|" + description )
existingHashes = DB.checkHashes([hash])
if hash in existingHashes: skip transaction
else: import + DB.storeHashes([hash])
```

### Budget Alert

```
spent    = sum(expense transactions this month where categoryId == budget.categoryId)
pct      = spent / budget.limit * 100
if pct >= 100: show critical alert
if pct >= 80:  show warning alert
```

### Savings Rate

```
savings_rate = (income - expenses) / income * 100
benchmark    = 20%   (50/30/20 rule)
```

### Recurring Detection

```
group transactions by merchant
for each merchant:
    months = distinct calendar months with a transaction
    if len(months) >= 2:
        amounts = [tx.amount for tx in group]
        cv      = stddev(amounts) / mean(amounts) * 100
        if cv < 15:
            mark as recurring
            next_due = last_transaction_date + 1 month
```

---

## UI / Design System

### Color Tokens

| Token | Value | Usage |
|-------|-------|-------|
| `--bg` | `#0B0B0E` | App background (dark) |
| `--surface` | `#16161A` | Card / modal surfaces |
| `--ac` | `#E8A64E` | Accent (orange) — CTAs, highlights |
| `--tx` | `#EDE9E3` | Primary text |
| `--tx2` | `#8E8E93` | Secondary / muted text |
| `--rd` | `#FF375F` | Expenses / negative |
| `--gn` | `#32D74B` | Income / positive |
| `--bl` | `#0A84FF` | Investments |
| `--og` | `#FF9F0A` | Warnings |

### Layout

- Base spacing unit: 12 px
- CSS Grid for screen layouts
- Mobile-first, max-width: 480 px
- Bottom navigation bar (6 tabs): Dash, Transactions, Reports, Import, Review, Settings
- `env(safe-area-inset-*)` for notch / home indicator clearance

### Typography

System font stack: `-apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif`

---

## PWA Capabilities

| Feature | Implementation |
|---------|---------------|
| Offline support | Service Worker caches all assets on first load |
| Home screen install | `manifest.json` with standalone display mode |
| App icons | `icon-192.png` (192×192), `icon-512.png` (512×512) |
| No install required | Works in any modern browser tab |
| iOS install | Safari → Share → Add to Home Screen |

The Service Worker uses a cache-first strategy — network is never required after the initial load.

---

## File Structure

```
fintrack/
├── index.html          # Entire app (~5,400 lines)
│   ├── <style>         # CSS design system (~784 lines)
│   └── <script>        # All JS modules (~4,600 lines)
│       ├── DB          # IndexedDB wrapper
│       ├── State       # Reactive state store
│       ├── Utils       # Date / format helpers
│       ├── CATS        # Category definitions
│       ├── MerchantEngine  # Auto-categorization
│       ├── BankParser  # CSV parser
│       ├── Charts      # Canvas chart renderers
│       ├── UI          # Toast / modal helpers
│       └── Screen renders (dash, txns, accounts, reports, import, review, settings)
├── manifest.json       # PWA manifest
├── service-worker.js   # Offline cache
├── icon-192.png        # App icon (small)
├── icon-512.png        # App icon (large)
├── README.txt          # Deployment guide
└── FINTRACK.md         # This file
```

---

## Deployment

### GitHub Pages (recommended)

1. Create a new GitHub repository
2. Upload all files from this folder
3. Go to **Settings → Pages → Deploy from branch → main**
4. Your app is live at `https://<username>.github.io/<repo>/`

### Netlify

Drag-drop the project folder into [netlify.com/drop](https://netlify.com/drop) — live in seconds.

### Local

Open `index.html` directly in any modern browser. Service Worker requires a server origin (localhost or HTTPS) to activate, but all core features work via `file://`.

---

## Development Notes

### Adding a New Bank Format

1. Add a detection function in `BankParser` that matches the bank's CSV headers
2. Return a normalized array of `{ date, description, amount, type }` objects
3. Register the detector in the `BankParser.parse()` dispatch table

### Adding a New Merchant

Add an entry to the built-in dictionary in `MerchantEngine`:

```javascript
{
  p: ["pattern1", "pattern2"],  // substrings to match (lowercase)
  n: "Display Name",
  c: "category-id",
  i: "emoji",
  b: "brand-slug"
}
```

### Adding a New Report

1. Add a tab button in the Reports screen HTML
2. Write a render function `renderMyReport(txns, opts)`
3. Register it in the tab-switch handler

### Extending the DB Schema

1. Add the new store to the `onupgradeneeded` handler with the correct key path and indexes
2. Bump the DB version number
3. Add CRUD helpers to the `DB` module

### No Build Step

There is intentionally no bundler, transpiler, or package manager. All features use browser-native APIs available in Chrome 90+, Firefox 90+, Safari 15+. Edit `index.html` directly and reload.
