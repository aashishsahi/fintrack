# Fintrack — Personal Finance PWA

A 100% offline personal finance tracker built as a single-file Progressive Web App. All data lives in the browser via IndexedDB — no server, no account, no subscription.

---

## Table of Contents

1. [Project Genesis](#project-genesis)
2. [Project Overview](#project-overview)
3. [Tech Stack](#tech-stack)
4. [Architecture](#architecture)
5. [Feature Phases](#feature-phases)
6. [Database Schema](#database-schema)
7. [Key Algorithms](#key-algorithms)
8. [UI / Design System](#ui--design-system)
9. [PWA Capabilities](#pwa-capabilities)
10. [File Structure](#file-structure)
11. [Deployment](#deployment)
12. [Development Notes](#development-notes)

---

## Project Genesis

The original requirements that shaped every design decision:

> "Can you create a free application (without any external subscriptions) to track my monthly expenses on my current iOS device?
>
> 1. It should not connect to the internet in any sense and should be completely offline (or be extremely secure, but completely offline is 1st preference), since I am tracking my finances.
> 2. It should track all my expenses across all my bank accounts.
> 3. It should also generate reports on demand about my spends, savings, and investments. The requirements or parameters to these reports should be extremely configurable and easily extendable.
> 4. It should understand where my money has gone — for example, any transaction on Zepto shows up as something else such as 'kiranakart' or similar. The system should understand what each transaction is and raise transactions that can't be identified so that I can tag them manually (preferably with a dropdown or options list)."

These requirements drove four key architectural choices:

| Requirement | Decision |
|-------------|----------|
| Completely offline | IndexedDB + Service Worker; no network calls |
| All bank accounts | Multi-account model, per-account balance tracking |
| Configurable reports | 6 report types, configurable date window + account filter, CSV export |
| Merchant identification | 3-tier MerchantEngine (user rules → dictionary → fuzzy match) + Review Queue for unknowns |

**Why a PWA instead of a native iOS app?**
A native app would require App Store submission and a developer account ($99/yr). A PWA hosted on GitHub Pages is free, installable via Safari's "Add to Home Screen", and behaves identically to a native app once installed.

**Safari / GitHub Pages constraint:**
Safari on iOS cannot open local HTML files directly from the filesystem. The app is hosted on GitHub Pages (free static hosting) so it can be accessed via a URL in Safari and installed as a home-screen PWA. After the first load, the Service Worker caches all assets — the app works fully offline from that point on.

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
| PDF parsing | PDF.js 3.11 (Mozilla) | Loaded from CDN, cached by SW for offline use |
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

### PDFParser

Offline PDF statement parser powered by [PDF.js](https://mozilla.github.io/pdf.js/) (Mozilla). PDF.js is loaded from CDN on first use and pre-cached by the Service Worker — subsequent imports work fully offline.

**Why PDFs?** All Indian banks and credit cards issue statements in PDF format. CSV export is buried in net banking portals and not available at all for many credit cards.

**Supported formats:**

| Bank | Savings Account | Credit Card |
|------|:-:|:-:|
| HDFC | ✓ | ✓ |
| ICICI | ✓ | ✓ |
| Axis | ✓ | ✓ |
| SBI | ✓ | — |
| Scapia | — | ✓ |

**Limitation:** Image-based / scanned PDFs are not supported — only digital PDFs that contain real text layers.

**How it works:**

1. PDF.js extracts all text items from every page. Each item carries an (x, y) coordinate in PDF user space.
2. Items are grouped into rows by y-coordinate (±3 pt tolerance), sorted top-to-bottom.
3. The bank is auto-detected by scanning the first 50 rows for bank name strings.
4. The header row is found by keyword matching (e.g. `NARRATION + WITHDRAWAL` → HDFC Savings).
5. Each header item's x-position becomes a column centre. Data items in subsequent rows are assigned to the nearest column by x-distance.
6. Date, description, debit, and credit columns are extracted and normalised into the same `{ date, description, amount, type }` format as CSV imports.

Output: same `{ bank, stmtType, txns[], total }` shape as BankParser — the rest of the import pipeline is identical.

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
- **Import** — PDF/CSV upload (drag-drop or file picker), bank auto-detect, preview table, deduplication report
- **Review Queue** — Unresolved transactions needing manual category tag; "apply to all similar" bulk-rule creation

**Features:**
- Parse PDF statements from HDFC, ICICI, Axis, SBI, Scapia (savings + credit card)
- Parse CSV statements from the same banks (fallback / net banking export)
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
├── index.html          # Entire app (~5,600 lines)
│   ├── <style>         # CSS design system (~784 lines)
│   └── <script>        # All JS modules (~4,800 lines)
│       ├── DB          # IndexedDB wrapper
│       ├── State       # Reactive state store
│       ├── Utils       # Date / format helpers
│       ├── CATS        # Category definitions
│       ├── MerchantEngine  # Auto-categorization
│       ├── BankParser  # CSV parser (all banks)
│       ├── PDFParser   # PDF parser via PDF.js (all banks)
│       ├── Charts      # Canvas chart renderers
│       ├── UI          # Toast / modal helpers
│       └── Screen renders (dash, txns, accounts, reports, import, review, settings)
├── manifest.json       # PWA manifest
├── service-worker.js   # Offline cache (v2 — includes PDF.js CDN URLs)
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

### iOS / Safari note

Safari on iOS **cannot open local HTML files** from the Files app. This is why GitHub Pages (or Netlify) is required — it provides an HTTPS origin so Safari can:
1. Load the app normally
2. Activate the Service Worker (which then caches everything for offline use)
3. Offer "Add to Home Screen" for native-app-like installation

Once installed and cached, the app works fully offline with no internet connection needed.

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

### Adding a New Bank PDF Format

1. Add a detection branch in `PDFParser._detect()` — match the bank name from the first 50 rows of text.
2. Write a parse function `_myBank(rows)` following the same pattern as the existing ones:
   - Call `_findHdrIdx(rows, ['KEYWORD1','KEYWORD2'], 2)` to find the header row
   - Call `_colRanges(rows[hi])` to get column descriptors
   - Loop rows, skip non-date rows, call `_assign(cols, row)` and `_cv(d, ...)` to read columns
   - Push `{ date, description, amount, type }` objects
3. Register the function in `_detect()`.
4. Update the service worker cache version if you bump anything that needs re-caching.

### PDF.js Offline Behaviour

PDF.js (`pdf.min.js` + `pdf.worker.min.js`) is loaded from the cdnjs CDN. The Service Worker pre-caches both files during its `install` event. After the first load:
- Both files are in the browser cache
- PDF parsing works fully offline
- If the cache is cleared, PDF parsing requires one internet request to re-fetch PDF.js

---

## Changelog

### v4.1 (current)
- **Fix** `deleteMapping()` crash — dead `data.budgets` reference removed
- **Fix** `catId` const-reassignment bug in `saveNew()` — changed to `let`
- **Feature** Account transfers — new "Transfer" type in Add Transaction modal; creates linked debit/credit pair, adjusts both account balances, excluded from expense/income totals
- **Feature** Splitwise CSV import — auto-detects Splitwise export format, maps Splitwise categories to Fintrack categories, imports your share per row (negative member column = your owe amount)
- **UI** Transfer transactions show `⇄` sign instead of `−` in transaction rows

---

## Roadmap

Priority tiers: **P0** = high-value, low-effort · **P1** = high-value, medium-effort · **P2** = significant effort or niche

---

### Phase 5 — Splitwise Deep Integration (P0)

The current Splitwise import handles the basic case. These improvements make it complete.

| Feature | Detail |
|---------|--------|
| **Column picker UI** | When Splitwise format is detected, show a step asking the user to pick which column header represents their name. Stores the selection for future imports. |
| **Settlement import** | Detect "payment" rows in Splitwise CSV (zero cost, one member positive, one negative) and import as income/transfer. |
| **Split tracking** | Mark imported Splitwise transactions with a "split" badge. Show a "Friends owe you" summary on the Dashboard. |
| **Reimbursement matching** | When a UPI credit arrives that matches a pending Splitwise owed amount, auto-suggest linking them. |

---

### Phase 6 — Smart Automation (P0)

Make the app work for you between imports.

| Feature | Detail |
|---------|--------|
| **Quick-add shortcuts** | Pin up to 5 frequent transactions on Dashboard (e.g. "Monthly rent ₹25,000"). One tap pre-fills the modal. Config stored in `settings` store. |
| **Bill reminders** | Promote the recurring detector from read-only to actionable. Add a "Remind me" toggle per recurring item. On PWA open, show a native `Notification` (requires permission) if a bill is due within 3 days. |
| **Auto-archive months** | Offer to freeze past months so accidental edits are blocked. Frozen months get a lock icon and are excluded from edit flows. |
| **Duplicate merge** | Detect near-duplicate transactions (same amount ± 1 day, same merchant) and offer a merge/dismiss UI in the Review Queue. |

---

### Phase 7 — Net Worth & Goals (P1)

| Feature | Detail |
|---------|--------|
| **Net worth history** | Snapshot total account balances once per month (on first open of that month). New `snapshots` DB store. New "Net Worth" chart in Reports — line chart over time. |
| **Savings goals** | Create goals with a name, target amount, and deadline. Link a specific account (e.g. "Trip to Japan — ₹80,000 by Dec 2026 — HDFC Savings"). Dashboard widget shows progress bar + days left. |
| **Debt tracker** | Track loans and EMIs. Input: principal, rate, tenure. Auto-calculate outstanding balance. Show payoff date and total interest. Link EMI transactions to loan record. |
| **Investment returns** | For investment-type transactions, add optional fields: instrument (MF/Stock/FD/Gold), units, NAV. Reports → Investments tab shows XIRR estimate. |

---

### Phase 8 — Broader Import Support (P1)

| Bank / Source | Type | Notes |
|---------------|------|-------|
| **Kotak Mahindra** | PDF + CSV | Savings + Credit card |
| **Yes Bank** | PDF + CSV | Savings |
| **IDFC First** | PDF + CSV | Savings + Credit card |
| **IndusInd** | PDF + CSV | Savings |
| **Paytm Payments Bank** | CSV | Savings |
| **Federal Bank** | PDF + CSV | Savings |
| **Bank of Baroda / PNB** | PDF | Savings |
| **SMS / notification parsing** | Text paste | Paste bank SMS alerts; regex extracts amount, merchant, account. Useful when CSV/PDF isn't available. |
| **UPI app export** | CSV | GPay / PhonePe allow CSV exports; add parsers for their formats. |
| **Zerodha P&L** | CSV | Import realized gains/losses as investment transactions. |

---

### Phase 9 — Reports & Intelligence (P1)

| Feature | Detail |
|---------|--------|
| **Cash flow calendar** | Month calendar view with daily spend dots. Tap a day to see transactions. |
| **Year-end tax summary** | Aggregate income, investments (80C-eligible), and interest earned into a one-page summary. CSV export. |
| **FI Health Score** | 0–100 score based on: savings rate (40%), emergency fund coverage (30%), investment rate (20%), budget adherence (10%). Dashboard widget with improvement tips. |
| **Peer benchmarks** | Anonymised aggregate comparisons ("You spend 20% more on food delivery than the average user"). All computed client-side from the user's own history; no data leaves the device. |
| **Custom date range** | Reports currently support 1/3/6/12-month windows. Add a custom date picker for arbitrary ranges. |
| **PDF report export** | Generate a print-ready PDF of any report using the browser's `window.print()` with a print stylesheet. |

---

### Phase 10 — UX & Polish (P0/P1 mix)

| Feature | Effort | Detail |
|---------|--------|--------|
| **Undo last transaction** | P0 | Toast after add/delete includes an "Undo" button. 5-second window before committing to DB. |
| **Bulk transaction edit** | P1 | Long-press to enter multi-select mode in Transactions list. Bulk re-categorise, bulk delete, bulk mark reviewed. |
| **Transaction notes search** | P0 | Current search covers merchant/description/amount/category. Also match notes field. |
| **Transfer filter tab** | P0 | Add "Transfers" chip to the Transactions filter row alongside Expenses/Income/Investments. |
| **Account colour themes** | P0 | Let user pick from 12 preset account card colours (currently only accent colour). |
| **Swipe to delete** | P1 | Swipe-left gesture on transaction row reveals a delete button (touch-friendly alternative to detail modal). |
| **Widget / home screen summary** | P2 | PWA doesn't support true widgets, but a "Today at a glance" share image (Canvas → PNG) the user can screenshot. |

---

### Phase 11 — Backup & Sync (P2)

All options preserve the offline-first principle — no mandatory cloud dependency.

| Option | Detail |
|--------|--------|
| **Encrypted export file** | Extend the current JSON export with AES-256-GCM encryption (password-derived key via PBKDF2). Import flow decrypts before parsing. |
| **iCloud / Google Drive backup** | Use the File System Access API or Share API to write the backup JSON to a cloud-synced folder (user picks the destination). No OAuth needed — the OS handles cloud sync. |
| **QR code sync** | For moving data between two devices on the same WiFi: encode the DB export as a QR stream (chunked). Device B scans and merges. Fully offline. |
| **Multi-device merge** | Import a backup from another device and run a conflict-resolution diff (last-write-wins per record, with a manual review queue for true conflicts). |

---

### Phase 12 — Splitwise-Level Social (P2)

For households or couples tracking shared finances.

| Feature | Detail |
|---------|--------|
| **Shared expense split** | On any transaction, tap "Split with…" → enter name + share (% or fixed). Creates a receivable record. |
| **Settlement flow** | When the person pays you back, mark the receivable as settled. Links to the incoming UPI transaction. |
| **Household view** | If two users on the same WiFi import a shared QR backup, show a merged "Household" report (combined net worth, joint spending). |

---

### Technical Debt & Housekeeping

| Item | Priority | Detail |
|------|----------|--------|
| Split `index.html` into modules | P2 | At ~6000 lines the file is getting large. Consider a build step (esbuild) that concatenates source modules but still produces a single distributable HTML. |
| Add Splitwise column-picker persistence | P0 | Store the user's chosen "my name" column in `settings` so re-imports don't re-ask. |
| `transferGroupId` in detail view | P0 | Transaction detail modal should show "Part of transfer" badge and link to the counterpart transaction. |
| Budget validation | P0 | `setBudget()` should reject negative or zero values with a toast instead of silently saving. |
| Reviewed → unreviewed | P0 | Add a "Mark as unreviewed" option in the transaction detail modal so mistakes can be corrected. |
| Service worker cache version | P1 | Bump cache version when making significant updates so stale assets are evicted. |
| iOS Safari PWA back-gesture | P1 | The bottom nav doesn't respond to iOS swipe-back. Intercept `popstate` or use a history stack. |
