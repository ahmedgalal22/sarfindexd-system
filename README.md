# Karta System (نظام الكارتة) — Reference Manual

> **Document status:** This README is a descriptive reference for the **inspected version** of the
> application only: repository `ahmedgalal22/sarfindexd-system`, commit
> `3f8a70cf44fed81837c24665cc6838df8e81cd98` (single commit: *"Rename kartacompany_fixed (4).html
> to sarfindexed.html"*), inspected on **2026-10-17** by static code inspection.
> **No runtime/behavioral testing was performed.** Anything not proven by the source code is
> explicitly marked as *unverified*, *incomplete*, or *requires owner confirmation*.
> The companion file **`SYSTEM_AUDIT_REPORT.md`** contains the engineering audit (defects,
> risks, remediation roadmap). This README describes what the code does; it does not certify
> that the application is safe for production financial data.

---

## 1. Name, purpose, scope, and status

| Item | Value (verified from source) |
|---|---|
| Product name (in UI/code) | "Karta System" (UI is Arabic, RTL) |
| Main file | `sarfindexed.html` — 11,616 lines, 652,854 bytes |
| Purpose | Single-page, offline, browser-local application for managing transport "karta" (كارتة) payment receipts for vehicle owners (أصحاب المركبات), offices (المكاتب), debts (المديونيات), a cash treasury (الخزنة), notes/expenses, printing, Excel import/export, and JSON backup/restore |
| Users | Seeded local accounts only (3 accounts are created by code on first run; see §5.1). No registration screen was found |
| Data location | **This browser, this device only** (IndexedDB + localStorage). No server, no cloud sync, no network calls exist in the code |
| Status | Working static inspection target; contains known defects listed in §13 and in `SYSTEM_AUDIT_REPORT.md`. Not validated by any automated test suite (none exists) |

**In scope (implemented in code):** login/logout, dashboard statistics, vehicle-owner
management (CRUD, search, filters, drag-to-reorder, bulk delete, Excel import/export of names,
de-duplication), per-vehicle receipt cards ("kartas") with row-level payment/delivery statuses,
vehicle debts ledger, a flexible per-vehicle "operations" table, office management (CRUD,
search, filters, period totals), per-office receipt copies and debts ledger, a receipt
("نموذج صرف" / sarf) creation form with live calculations, vehicle-number separator rows,
treasury integration on receipt save/delete (withdraw/refund), an "all receipts" browser with
filters, a treasury module (cash/Vodafone/bank accounts, deposits, withdrawals, transfers,
denomination counter, daily closing, alerts, forecast), a client/reporting page, debt-age
notifications (≥ 30 days), notes (free-text + dated table notes) and an external-expenses
table, printing (multiple paths), Excel import/export (SheetJS), full JSON backup/restore and a
7-day auto-backup prompt.

**Out of scope / not present:** any backend, multi-device sync, roles/permissions, audit
trail/history of edits, password management UI, automated tests, build tooling, or a payment
gateway. The "Packages" (الباقات) page is a static pricing/display page; no purchase or
subscription logic exists in code.

---

## 2. Verified technology stack

Verified from the actual files (not assumed):

| Layer | What is actually used | Evidence |
|---|---|---|
| Markup | One HTML file, `lang="ar"`, `dir="rtl"` | `sarfindexed.html` lines 1–9 |
| Styling | (a) Pre-generated utility CSS file `tailwind.css` (Tailwind-style class names; **not** the Tailwind CDN/play build); (b) one inline `<style>` block (lines 10–381) with custom CSS (print rules, drag styles, packages page, scrollbars) | `tailwind.css` header: *"Generated for Karta System… Covers all 351 utility classes used in index_filters.html… Compatible with @tailwindcss/browser@4… Generated: 2026-02-18"* |
| Font | Self-hosted "Cairo" via `cairo-font.css`, referencing `fonts/*.woff2` — **the `fonts/` directory is missing from the repo**, so the browser falls back to default fonts | `cairo-font.css`; `ls fonts` → absent |
| JavaScript | Vanilla JS only (no framework, no modules, no bundler). Two inline `<script>` blocks: lines 2709–10730 (main app) and 10831–11386 (receipt Excel + treasury-linked edit/delete modals). Extensive use of inline `onclick`/`onchange` handlers and `window.*` globals | Direct inspection; 239 function definitions found by static parse |
| Spreadsheet | SheetJS (`xlsx.full.min.js`), version string `0.18.5`, loaded locally at line 9 | `xlsx.full.min.js` (881,727 bytes) |
| Database | Browser **IndexedDB**, database `KartaSystemDB`, version **3** | `indexedDB.open(DB_NAME, DB_VERSION)` line 2751; constants at ~2745 |
| Secondary storage | `localStorage` for session, page-state, migration flags, treasury alert/denomination preferences | See §9.4 |
| Network | **None.** No CDN links, no `fetch`/XHR, no external URLs (only a comment in `cairo-font.css`). All three head resources are local files | Lines 7–9; static scan found 0 `fetch(`, 0 `XMLHttpRequest`, 0 `sessionStorage` |

**Missing assets referenced by the page:** `fonts/*.woff2` (Cairo font files) and `wasel.png`
(receipt header image, referenced at lines 1855 and 7930). Both are absent, so the font and the
printed header image silently fall back / appear broken.

**Known styling gap (verified by static scan):** some utility classes used in the markup do not
exist in `tailwind.css` or the inline `<style>` — e.g. arbitrary-value classes
`max-w-[98vw]`, `max-h-[96vh]`, `max-h-[90vh]`, `min-w-[90px]` and utilities `-top-1`,
`-right-1`, `bg-opacity-60`, `border-orange-300`. Those specific styles (modal size caps,
overlay opacity shade, badge offsets, one border color) will not apply. The `tailwind.css`
header also states it was generated for a differently-named source file
(`index_filters.html`), so full class coverage of the current file is **unverified**.

---

## 3. Project structure and file roles

The repository contains exactly five files (verified with `git ls-files` / `find`; no
`package.json`, no config, no tests, no `.gitignore`):

| File | Size | Role |
|---|---|---|
| `sarfindexed.html` | 652,854 bytes / 11,616 lines | The entire application: markup (≈384–2708), inline CSS (10–381), main script (2709–10730), second script (10831–11386), trailing modal markup (11388–11616) |
| `tailwind.css` | 31,696 bytes / 574 lines | Pre-generated utility CSS (see §2 caveats) |
| `cairo-font.css` | 5,336 bytes / 134 lines | `@font-face` declarations for Cairo; depends on missing `fonts/` |
| `xlsx.full.min.js` | 881,727 bytes | SheetJS 0.18.5 (Excel read/write) |
| `README.md` | this file | Reference manual (replaced a 2-line stub) |

Internal layout of `sarfindexed.html`:

- Lines 1–9: doctype, head, local resource links (`cairo-font.css`, `tailwind.css`,
  `xlsx.full.min.js`).
- Lines 10–381: inline `<style>`.
- Lines 384–2708: body markup — login screen, sidebar navigation, all pages
  (`home`, `client`, `receipt`, `allReceipts`, `vehicles`, vehicle detail, `offices`, office
  detail, `treasury`, `notes`, `packages`), datalists, early modals (expenses modal at ~2563).
- Lines 2709–10730: **Script 1** — DevTools blocker, IndexedDB init, auth, navigation/page
  state, notes, dashboard, migrations, vehicles CRUD/details/debts, karta rendering + **two
  receipt-editing implementations** (inline card editor *and*, in Script 2, a modal editor),
  operations table, receipt form + save flow, offices CRUD/details/debts/kartas, notifications,
  client page, external expenses, all-receipts + status synchronization, backup/restore,
  drag-and-drop ordering, vehicle/office Excel, de-duplication, treasury module, export-all.
- Lines 10732–10830: modal markup between the two scripts (`addVehicleModal` 10732,
  `addOfficeModal` 10774, `restoreModal` 10815).
- Lines 10831–11386: **Script 2** — receipt-form Excel export/import, delete-with-treasury
  -refund flow, receipt **edit modal** (`em*` functions).
- Lines 11388–11616: static modal markup: `receiptSaveWithdrawModal` (11389),
  `receiptDeleteRestoreModal` (11443), `editReceiptModal` (11496).

---

## 4. How to open and run

There is no build step and nothing to install.

1. Keep all five files in the **same folder** (`sarfindexed.html`, `tailwind.css`,
   `cairo-font.css`, `xlsx.full.min.js`).
2. Open `sarfindexed.html` in a modern desktop browser (Chrome/Edge/Firefox). Serving the
   folder over a local static server (e.g. `python3 -m http.server`) also works. **Note:** the
   database is scoped to the browser + origin; opening via `file://` vs `http://localhost`
   creates **separate databases**. Pick one access method and stick to it, or data will appear
   to "disappear" when switching.
3. Log in with one of the built-in accounts seeded by the code on first run (seed logic at
   ~line 2822). **Passwords are intentionally not reproduced in this manual** — obtain them
   from the project owner. Be aware they are stored only Base64-encoded (§14).
4. All data stays in that browser profile on that device. Clearing site data, changing
   browser, or using another computer means an empty (or different) database. Use the built-in
   backup (§11) before any browser maintenance.

**Runtime prerequisites:** a browser with IndexedDB and localStorage enabled (private/incognito
mode may block or discard storage). No internet connection is required at any point.

> **Operational warning (verified):** the app ships a "DevTools blocker" (lines 2712–2740)
> that tries to hide the page when developer tools open (with a Ctrl+B bypass flag). This is
> not a security control; it only complicates support/debugging.

---

## 5. Screen-by-screen guide

Navigation is a sidebar of `data-page` links; `showPage`-style logic toggles page sections and
persists the last position in `localStorage` (`karta_pageState`, saved by `savePageState`
line 2931), restoring page + selected vehicle/office + active tab after reload.

### 5.1 Login
Username/password form. On submit, the code reads the `users` store and compares the entered
password against the stored value after `atob()` decoding (with a plaintext fallback comparison
for legacy records). On success the **entire user record (including the password field)** is
kept as the session (`localStorage` key `currentUser`, restored at line 3599) and the app
shell is shown. There are **no roles**: no code path distinguishes permissions between the
seeded accounts. Logout clears the session (line ~2893).

### 5.2 Home / dashboard (الرئيسية)
Stat cards computed by `updateDashboardStats` (line 3405) from the current user's records
(counts of vehicle owners/offices/receipts and monetary aggregates derived from stored
receipts). Also hosts the debt-age **notifications** area: vehicles whose last dated debt row
still has `remaining > 0` and is ≥ 30 days old are listed (line ~8073). Dismissing a
notification only removes it from the DOM — it is **not persisted** and returns on reload.

### 5.3 Client page (العميل / التقارير)
Date-range reporting over receipts/offices (`loadClientData` line 8236,
`calculateOfficesTotal` line 8292) plus static FAQ content. Two code facts matter for reading
its numbers:
- "Total profit" is computed as **Σ `row.expense`** over receipt rows (the صرف column). Whether
  that matches the business meaning of profit is an **owner-confirmation item** (§16).
- The "collected now" figure inside `calculateOfficesTotal` is computed **without applying the
  selected date filter** — a confirmed inconsistency (see Audit AUDIT-014).

### 5.4 Receipt form — نموذج الصرف (creation)
The main data-entry screen. Header: vehicle-owner name (with datalist suggestions loaded by
`loadVehicleOwnersForReceipt` line 5806), receipt date. Body: an editable grid of rows, each
row: date, office (datalist from `loadOfficesForReceipt` line 5821), driver name (`data`),
policy (`bolsa`), car (`car`), loading (`loading`), "taktik" (تعتيق), weight, deficit,
freight rate (`noloon`), custody (`ohda`), office amount (`officeAmount`), expense
(`expense`/صرف), add (`add`), discount (`discount`), and a read-only computed **net**. Rows can
be added/removed; a special **separator row** ("🔄 رقم المركبة") splits the grid by vehicle
number and carries a running "إجمالي ما سبق" (subtotal so far). Footer totals: row count,
total, previous balance, paid, debt ("الرصيد مدين عليه"), additional, remaining.
Buttons include: save, clear/new, print (`printReceipt` line 6544 → `window.print()`),
export this form to Excel (`exportReceiptToExcel` line 10833), import Excel into the form
(`importReceiptFromExcel` line 10880).

When an owner with existing debts is chosen, "previous balance" auto-fills from the **last
debt row's `remaining`** (line ~6178); the vehicle's manually-entered `initialBalance` is *not*
used by this auto-fill (owner-confirmation item).

### 5.5 Vehicle owners (أصحاب المركبات) — list
Table of owners (`loadVehicleOwners` line 3767, `displayVehicleOwners` line 3795): name,
vehicle number, phone, received status, a debt badge showing the **last debt row's
`remaining`**, drag handle (reorder persists via `sortOrder`), checkbox for bulk delete.
Actions: add (inline form), inline edit (`editVehicleOwner` line 3865 / `saveVehicleEdit`
3908), delete (`deleteVehicleOwner` 3942 — cascades to that owner's receipts and office
copies), bulk delete (line ~4059 — **does not cascade**, see AUDIT-006), search, received
-status filter buttons, print list, export names to Excel, import names from Excel,
"remove duplicates" (`removeDuplicateVehicles`, button at line 1018).

> Inline-edit quirk (verified): `editVehicleOwner` rewrites the 8-cell row with a 6-cell edit
> row (drag + debt cells dropped), so columns visually misalign while editing (AUDIT-015).

### 5.6 Vehicle detail
Opened by clicking an owner. Header info + tabs:
- **Kartas (الكارتات):** all receipts linked to the vehicle (`vehicle.receipts[]` IDs →
  `receipts` store), rendered as cards by `loadKartas` (4639) / `displayKartas` (5032),
  newest date first. Per card: full row table incl. separator rows; per-row
  payment (الاستلام: تم صرفها/لم يتم صرفها) and delivery (التسليم) `<select>`s; receipt-level
  payment/take status selects; buttons **view details** (`viewFullReceipt`), **edit** (opens
  the Script-2 edit modal, `openEditReceiptModal(id,'vehicle')` line 5150), **delete** (opens
  the treasury-refund modal, §6.3), **print** (`printSingleReceipt` line 7913); a free-text
  receipt note with its own save button. Filters above the list: single-date search
  (`searchKartasByDate` 5195), date-range + payment/take status + free-text filter
  (`applyVehicleKartaFilter` 5382, which text-matches against `JSON.stringify(receipt)`).
  A running "vehicle total amount" reflects the filtered set.
- **Debts (المديونيات):** editable ledger stored inside the vehicle record
  (`debts[]{balance, paid, remaining, date, notes}`) with an `initialBalance` field; add/remove
  rows, save, print (`printDebtsTable` line 4502).
  ⚠️ `recalculateAllDebts` (4332) only recomputes **row 0** — other rows keep stored values
  (AUDIT-009).
- **Operations (إدارة التشغيل):** a free-form per-vehicle table (dynamic rows/columns,
  text cells `col1..colN`) in a dynamically-built modal (`openOperationsManagement` 5438,
  `saveOperationsData` 5767 stores into `vehicle.operations[]`; print via
  `printOperationsTable` 5673). It is a scratchpad — it takes no part in financial totals.

### 5.7 Offices (المكاتب) — list
Table (`loadOffices` 6680, `displayOffices` 6706): name, phone, received status, **total net**
(Σ `row.net` over the office's embedded receipt copies), **not-received net** (Σ net of rows
whose `rowPayment` ≠ "تم صرفها"), **latest debt remaining** (last `debts[]` row), plus summary
cards (total net, total remaining, collected = total net − total remaining). Actions mirror
vehicles: add, inline edit (`editOffice` 6834), delete (`deleteOffice` 6909 — deletes **only**
the office record; its embedded receipt copies/debts are lost with it and master receipts are
untouched, AUDIT-007), bulk delete (6974, same non-cascade), search, status filters, print,
Excel name import/export, de-duplication, drag-reorder, and a **period filter**
(`setOfficesListPeriod` 7002 / `applyOfficesListFilter` 7025) that recomputes per-office net
for receipts whose `receiptDate` falls in range.

### 5.8 Office detail
Tabs:
- **Kartas:** the office's **embedded receipt copies** (`office.receipts[]`), rendered by
  `loadOfficeKartas` (7433). Office copies exclude the صرف (`expense`) column, and their row
  `net` uses the office formula (§10). Row payment/delivery selects update via `_rowId` (with
  index fallback). Edit opens the same edit modal — which always loads the **master** receipt
  from the `receipts` store, even from office context (line 11120).
- **Debts:** ledger `debts[]{net, arrived, remaining, date, notes}` with
  `remaining = net − arrived` computed per row (line ~7376).

### 5.9 All receipts (جميع النماذج)
Cross-owner browser (`applyAllReceiptsFilter` 8931, `displayAllReceipts` 9067): date range,
payment status (uses `receipt.isPaid`), and take-status filters; summary totals
Σ `totals.total` and Σ `totals.remaining`. Each card offers payment/take selects, edit
(`openEditReceiptModal(id,'all')`), print, delete. "تصدير الكل" exports to Excel
(`exportAllReceiptsToExcel` line ~10592) with a `prompt()` mode choice (1/2/3) plus a summary
sheet.

### 5.10 Treasury (الخزنة)
Three accounts — cash (كاش), Vodafone Cash (فودافون), bank (البنك). Balance per account =
Σ(credit − debit) over `treasuryTransactions` excluding soft-deleted rows. Features
(lines 10064–10548): manual deposit/withdraw/transfer (`executeTreasuryOp` line ~10186 —
transfers write a mirrored debit/credit pair and withdrawals are blocked if they'd drive the
balance negative), transaction list with edit (`editTreasuryTx`; **edit has a credit/debit bug
— AUDIT-010**) and soft delete (`deleted=true`), a denomination counter (200/100/50/20/10/5 —
display aid only, persisted per user in `localStorage` `karta_denom_<user>`), **daily closing**
(line ~10421, stored in `dailyClosing`), threshold **alerts**
(`localStorage karta_alerts_<user>`), and a simple forecast = Σ of each office's last debt
`remaining`.

### 5.11 Notes (الملاحظات)
Two tabs backed by the single per-user record in the `notes` store:
- **Text note:** one big textarea (`textNote`), save (line 3019) / print.
- **Table notes:** dated notes `notes[]{date,title,content,...}` with add form (multi-row),
  search, inline edit, delete, bulk delete, print. ⚠️ Two confirmed defects live here:
  adding table notes wipes the text note + expenses data (AUDIT-001), and edit/delete act on
  the wrong note when dates are unsorted (AUDIT-002).
- **External expenses (المصروفات الخارجية):** a modal with a free-form table (amount, date,
  description + user-added columns) stored positionally as arrays in the *same* notes record
  (`expenses`, `expenseHeaders`; `saveExpenses` 8696). Total = Σ first-column amounts
  (`calculateTotalExpenses` 8674). Print uses a `document.write` window (line ~8816).

### 5.12 Packages (الباقات)
Static marketing/pricing cards. No purchase, activation, or feature-gating logic exists.

---

## 6. Detailed workflows

### 6.1 Create a receipt (end-to-end, as coded)
1. Fill the receipt form (§5.4). Nets/totals recompute live on every keystroke.
2. **Save** → `saveReceipt` (line ~6249) validates: owner name non-empty, and every row's
   office name exists in the user's offices (exact name match). It assembles the receipt
   object (reading company name/phone through optional chaining from inputs **that do not
   exist on the form**, so new receipts always store `companyName/companyPhone = ''` —
   AUDIT-011), stashes it in `_rswPendingData`, and opens the **treasury withdraw modal**
   (`receiptSaveWithdrawModal`) showing `totals.remaining` and the three account balances.
3. `rswConfirm(accountType|null)` (6358):
   a. If an account was chosen and `remaining > 0`: checks balance; writes a `withdraw`
      treasury transaction for **`totals.remaining`** (not `paid`, not `total` — see §16).
      ⚠️ If the balance is insufficient it re-opens the modal, but `_rswPendingData` was
      already set to `null` (6363), so **any retry silently does nothing** (AUDIT-003) — the
      user must cancel and press Save again.
   b. If `totals.paid > 0`: `addDebtRowFromReceipt` (6191) appends a debt row to the vehicle
      owner.
   c. Inserts the receipt into `receipts` (`store.add`).
   d. `saveOrUpdateVehicleOwner` (6422): finds owner by exact `name` (creates if missing —
      though step 2's validation makes the create-branch effectively unreachable for offices,
      the owner-create branch is reachable) and pushes the new `receiptId`.
   e. `saveToOffices` (6458): groups rows by office name, strips `expense`, recomputes office
      net (unrounded), and appends an embedded copy `{receiptId, ownerName, receiptDate,
      rows}` to each office.
   These are **≥ 5 separate IndexedDB transactions with no atomicity**; a failure mid-chain
   (e.g. tab closed) can leave a treasury withdrawal without a receipt, or a receipt without
   office copies. `saveOrUpdateVehicleOwner` also resolves its promise on transaction *error*
   (line 6453), hiding failures (AUDIT-004).
4. Success alert; optional "create new?" confirm clears the form.

### 6.2 Edit a receipt — **two different implementations exist**
- **Modal editor (Script 2, the one wired to current "تعديل" buttons):**
  `openEditReceiptModal` (11116) loads the master receipt, `em*` functions rebuild the grid
  (preserving `_rowId`, `rowPayment`, `rowDelivery` via `dataset`), `emSave` (11286) writes
  the receipt and **per-office row subsets** with the office net formula. Caveats: totals are
  re-parsed from rendered text; `totals.rowCount` is dropped; an office with no remaining rows
  keeps its old rows (`officeGroups[office.name] || r.rows`, line 11367 — AUDIT-036); company
  fields can't be edited here (not present in the modal).
- **Inline card editor (Script 1):** `buildReceiptEditFormHTML` (4676) +
  `toggleReceiptEdit` (4872) + `saveReceiptEdits` (4907). Still present and still rendered
  inside vehicle cards (hidden `.receipt-edit` block), but the visible edit buttons call the
  modal instead. If used, it is **lossy**: rebuilt rows lose `_rowId`, separator rows are
  flattened into data rows, and every office copy receives **all** rows instead of its subset
  (line 4984 — AUDIT-005). Treat the inline editor as a legacy hazard until removed.

### 6.3 Delete a receipt (with optional treasury refund)
`deleteReceipt` (7907, vehicle ctx) / office equivalent (line ~8041) → `_openRdrModal` (10981)
shows `totals.remaining` (from the master receipt, or the office copy for office context) →
`rdrConfirm(accountType|null)` (11018) optionally deposits that amount back into a treasury
account (separate transaction) → `_doDeleteReceipt` (11060) removes the receipt from
`receipts`, unlinks it from the vehicle owner, and strips office copies; office-context delete
(`_doDeleteOfficeReceipt` 11094) removes **only** that office's embedded copy. The refund
amount is parsed back from the modal's displayed text (`parseFloat(textContent)`); the format
helper `tFmt` (10098) is plain `toFixed(2)`, so this is safe for normal values today but
fragile by design.

### 6.4 Change payment / delivery / take statuses
Receipt-level: `getPaymentStatus` (9295) reads `paymentStatus ?? isPaid`-style dual fields;
`setReceiptPaymentStatus` (9325) writes **both** fields on the master receipt but only
`paymentStatus` on office copies. Take-status mirrors this asymmetry (`isTaken`/`takeStatus`,
~9409). Row-level: `setReceiptRowStatus` (9456) matches rows by `_rowId` in both the master
receipt and office copies, falling back to positional index inside office copies. Legacy
receipts acquire `_rowId`s through the boot migration (§9.3).

### 6.5 Debts editing (vehicle / office)
Open detail → Debts tab → edit rows inline (remaining recomputes per row as you type) → save
persists the whole embedded `debts[]` array. New receipts with `paid > 0` append debt rows
automatically (§6.1b). There is **no** automatic reconciliation between receipt `totals` and
debt rows beyond that append and the previous-balance auto-fill.

### 6.6 Treasury operations
Deposit/withdraw/transfer via the three account panels (`executeTreasuryOp`); before/after
balances are stamped onto each transaction. Receipt save/delete can also originate treasury
movements (§6.1, §6.3). Daily closing snapshots balances per date. Editing a past transaction
rewrites its credit/debit based on the selected type — with the `deposit`-only-credit bug
(AUDIT-010) — and does **not** restamp other transactions' before/after balances.

### 6.7 Backup / restore
See §11.

### 6.8 Excel import/export
See §11.

### 6.9 De-duplication
"إزالة المكرر" for vehicles (`removeDuplicateVehicles` ~9989) / offices (~10028): groups by
lowercased name, keeps the first record, **deletes the rest outright** — embedded receipts,
debts, and links of deleted duplicates are **not merged** (AUDIT-008).

### 6.10 Reordering lists
Drag handles on vehicle/office tables (`initDragDrop` ~9733) persist a `sortOrder` on each
record.

---

## 7. Functions and modules (overview)

There are no modules/files to import — everything is global-scope functions in two script
blocks. 239 function definitions were counted statically (including nested closures). Key
groups (line numbers refer to `sarfindexed.html` @ inspected commit; the audit report's
appendix contains the complete generated index):

| Area | Key functions (line) | Responsibility |
|---|---|---|
| Boot/DB | `initDB` (2749), boot `window.load` (~3591), `migrateFixOfficeRows` (3442), `migrateAddRowIds` (3512) | Open/upgrade DB, seed users, run data migrations, restore session + page state |
| Auth/nav | login handler (~2843+), `savePageState` (2931) | Session in localStorage, page switching/restore |
| Notes | `loadNotes`/`loadTableNotes` (3182), `displayNotes` (3204), `editNote` (3248), `deleteNote` (3315), save-text-note (3019) | Per-user notes record CRUD |
| Dashboard | `updateDashboardStats` (3405) | Aggregate stat cards |
| Vehicles | `loadVehicleOwners` (3767), `displayVehicleOwners` (3795), `editVehicleOwner` (3865), `saveVehicleEdit` (3908), `deleteVehicleOwner` (3942), bulk delete (~4059) | Owner CRUD/search/filter |
| Vehicle detail | `loadKartas` (4639), `displayKartas` (5032), `buildReceiptEditFormHTML` (4676), `saveReceiptEdits` (4907), `toggleReceiptEdit` (4872), `viewFullReceipt` (~5260), `searchKartasByDate` (5195), `applyVehicleKartaFilter` (5382), debts: `recalculateAllDebts` (4332), `addDebtRow` (4357), operations: `openOperationsManagement` (5438), `saveOperationsData` (5767) | Cards, both editors, debts, scratchpad |
| Receipt form | `loadVehicleOwnersForReceipt` (5806), `loadOfficesForReceipt` (5821), row calc `calculateNet` closure (6068), `calculateTotals` (~6100s), `saveReceipt` (6249), `rswConfirm` (6358), `addDebtRowFromReceipt` (6191), `saveOrUpdateVehicleOwner` (6422), `saveToOffices` (6458), `printReceipt` (6544) | Creation + treasury-linked save |
| Offices | `loadOffices` (6680), `displayOffices` (6706), `editOffice` (6834), `deleteOffice` (6909), `applyOfficesListFilter` (7025), `loadOfficeKartas` (7433), `printSingleReceipt` (7913), `deleteReceipt` (7907) | Office CRUD/detail/period totals |
| Notifications/client | notifications (~8073), `loadClientData` (8236), `calculateOfficesTotal` (8292) | Debt-age alerts, reporting |
| Expenses | `calculateTotalExpenses` (8674), `saveExpenses` (8696), `loadExpensesInModal` (8735), `printExpenses` (8811) | External expenses in notes record |
| All receipts/status | `applyAllReceiptsFilter` (8931), `displayAllReceipts` (9067), `getPaymentStatus` (9295), `setReceiptPaymentStatus` (9325), `setReceiptRowStatus` (9456) | Browsing + status sync across copies |
| Backup | backup (~9553), `restoreBackup` (9617), auto-backup check (~9698) | JSON export/import of all stores |
| Order/Excel/dedupe | `initDragDrop` (~9733), `loadSheetJS` (9804, **no-op**), vehicle/office Excel (~9821+), `removeDuplicateVehicles/Offices` (~9989/10028) | Utilities |
| Treasury | `tFmt` (10098), `tGetAllBalances` (~10100s), `executeTreasuryOp` (~10186), `loadTreasuryTransactions`, `saveTreasuryTxEdit` (10394), daily closing (~10421), `exportAllReceiptsToExcel` (~10592) | Accounts, closing, export-all |
| Script 2 | `exportReceiptToExcel` (10833), `importReceiptFromExcel` (10880), `_openRdrModal` (10981), `rdrConfirm` (11018), `_doDeleteReceipt` (11060), `openEditReceiptModal` (11116), `emInsertDataRow` (11153), `emCalcRow` (11219), `emSave` (11286) | Form Excel, refund-delete, edit modal |

**Global state:** `db` (2744), `currentUser` (2843), `currentVehicleId` (4084),
`currentOfficeId` (6561), filter variables (`vehicleKartaFilter` 9292 etc.),
`_rswPendingData` (6247), `_rdrPending*` (7902–7905), `_emReceiptId/_emContext` (11112–11113),
plus `window._vehicleReceiptsCache`. Duplicated implementations exist for the row-net formula
(at least 3 master variants + 3 office variants) and `calculateRemaining` (4312 vehicle debts,
7376 office debts, as nested closures).

**Error-handling style:** sparse — ~11 `try/catch` sites in 11.6k lines; user feedback is via
~103 `alert()` / 15 `confirm()` / 1 `prompt()`; many IndexedDB `onerror` handlers are absent or
swallow errors. Rendering uses `innerHTML` template strings without escaping (63 sites) plus
4 `document.write` print windows (§14).

---

## 8. Application structure & startup sequence

1. HTML loads `cairo-font.css`, `tailwind.css`, `xlsx.full.min.js`, then the two inline
   scripts (Script 2 functions override/extend globals after Script 1).
2. DevTools blocker arms immediately (2712).
3. On `window.load` (~3591): `initDB()` → seed users if `users` empty → run
   `migrateAddRowIds()` then `migrateFixOfficeRows()` (each gated by a localStorage flag) →
   restore session from `localStorage.currentUser` → restore page state
   (`karta_pageState`) → render dashboard + notes.
4. Additional `window.load` listeners (9371, 9534, 9727) wire status controls, backup
   auto-check, and drag-drop.

If IndexedDB is unavailable/blocked, `initDB` rejects and the app does not start; there is no
graceful fallback UI beyond the promise rejection path.

---

## 9. IndexedDB & persistence architecture (complete)

### 9.1 Database
- **Name:** `KartaSystemDB` · **Version:** 3 (`initDB`, line 2749–2751).
- `onupgradeneeded` creates any missing stores/indexes (creation is guarded by
  `objectStoreNames.contains`). No destructive version-change logic was found, but upgrades
  from real-world v1/v2 databases were **not testable** (no sample databases exist in the
  repo) — classified unverified in the audit.

### 9.2 Object stores

| Store | keyPath | Indexes | Record shape (as written by code) |
|---|---|---|---|
| `users` | `username` | — | `{username, password (Base64), ...}` seeded at ~2822 |
| `vehicleOwners` | `id` (autoIncrement) | `username` | `{id, name, vehicleNumber, phone, received, username, createdAt, receipts: [receiptId…], debts: [{balance, paid, remaining, date, notes}], initialBalance, operations: [{col1…colN}], sortOrder}` |
| `offices` | `id` (autoIncrement) | `username` | `{id, name, phone, received, username, createdAt, receipts: [{receiptId, ownerName, receiptDate, createdAt?, rows: […office rows]}}, debts: [{net, arrived, remaining, date, notes}], sortOrder}` |
| `receipts` | `id` (autoIncrement) | `username` | `{id, ownerName, receiptDate, companyName, companyPhone, companyInfo, username, createdAt, rows: […], totals: {total, rowCount?, previousBalance, paid, debt, additional, remaining}, isPaid, paymentStatus, isTaken, takeStatus, notes}` |
| `notes` | `username` | — | **One record per user holding four features:** `{username, notes: [{date,title,content,username,createdAt}], textNote, expenses: [[cell…]…], expenseHeaders: […], lastModified}` |
| `treasuryTransactions` | `id` (autoIncrement) | `username`, `accountType`, `date` | `{id, username, accountType: cash|vodafone|bank, opType, credit, debit, beforeBalance, afterBalance, description, date, time, timestamp, deleted}` |
| `dailyClosing` | `id` (autoIncrement) | `username`, `date` | Daily closing snapshots (created in treasury module ~10421) |

**Receipt row** (master): `{_rowId, date, office, data, bolsa, car, loading, taktik, weight,
deficit, noloon, ohda, officeAmount, expense, add, discount, net, rowPayment, rowDelivery}`.
**Separator row:** `{_type:'separator', _rowId, vehicleName, subtotal}`.
**Office-copy row:** same minus `expense`, with `net` recomputed by the office formula
(unrounded).

### 9.3 Relationships & consistency model
There are no foreign keys. Linkage is by convention:
- `vehicleOwners.receipts[]` holds receipt IDs; the receipt also stores `ownerName` as text.
- `offices.receipts[]` holds **full embedded copies** keyed by `receiptId`; the office is
  matched to rows by **office name string** (`row.office === office.name`). Renaming an office
  or owner does not rewrite historical rows/copies.
- Statuses are duplicated in master + copies and kept in sync by the §6.4 functions matching
  `_rowId`; dual field generations coexist (`isPaid`/`paymentStatus`,
  `isTaken`/`takeStatus`).
- Boot migrations: `migrateAddRowIds` (3512) back-fills `_rowId` on receipt + office-copy
  rows; `migrateFixOfficeRows` (3442) repairs office-copy row structures. Each runs once per
  browser (localStorage flags `rowIdMigrationDone`, `officeRowsMigrationDone`); **backup
  restore clears both flags**, so migrations re-run after a restore.

### 9.4 localStorage keys
`currentUser` (full user record incl. password), `karta_pageState`, `rowIdMigrationDone`,
`officeRowsMigrationDone`, `lastAutoBackup`, `karta_alerts_<username>`,
`karta_denom_<username>`. No `sessionStorage` is used.

### 9.5 Persistence semantics (important operational facts)
- Data is **per browser profile, per origin, per device**. Nothing is shared between users'
  devices; the three seeded accounts merely partition records by the `username` field/index
  within the same local database.
- Reloads preserve everything (IndexedDB), including the logged-in session and last page —
  i.e., closing the tab does **not** log out.
- All list screens re-read from IndexedDB on demand; there is no in-memory cache layer besides
  small globals. Concurrent tabs share the same DB with **no locking** — simultaneous
  read-modify-write of embedded arrays (e.g. two tabs saving receipts) is last-write-wins and
  can lose updates (unverified at runtime; evident from code structure).
- Deleting browser site data destroys the database. The only recovery path is a prior JSON
  backup (§11).

---

## 10. Financial calculations (exactly as implemented)

> This section transcribes the code's arithmetic. It does **not** assert these formulas match
> the business's accounting intent — see §16 for rules needing owner confirmation.

**Row net — master receipt** (closures at 4815 & 6068; `emCalcRow` 11219):
```
net = ((weight − deficit/1000) × noloon) − ohda − officeAmount − expense + add − discount
```
result stored after `.toFixed(2)` in form/inline/modal editors.

**Row net — office copy** (6473, 4994, 11354): same formula **without `expense`**, and
**without rounding**:
```
officeNet = ((weight − deficit/1000) × noloon) − ohda − officeAmount + add − discount
```

**Receipt totals** (`calculateTotals`, `calculateEditTotals` 4848, `emCalcTotals` 11242):
```
total     = Σ row.net            (data rows only; separators excluded)
debt      = previousBalance − paid
remaining = total − paid + additional
```
Separator `subtotal` = Σ net of the data rows above it since the previous separator.

**Debts:** vehicle row `remaining = balance − paid`; office row `remaining = net − arrived`.
"Current balance" displays everywhere use the **last** debt row's `remaining`.

**Treasury:** `balance(account) = Σ (credit − debit)` over non-deleted transactions;
withdrawals that would overdraw are blocked in `executeTreasuryOp` and in the receipt-save
path (6372). Receipt save withdraws `totals.remaining`; receipt delete can deposit
`totals.remaining` back.

**Client/reporting aggregates:** profit = Σ `row.expense`; office period net = Σ office-copy
`row.net` within `receiptDate` range; all-receipts summary = Σ `totals.total` /
Σ `totals.remaining` over the filtered set. External-expenses total = Σ first-column amounts.

**Numeric handling (verified by counting call sites):** `parseFloat` ×111, `Number` ×49,
`parseInt` ×11, `toFixed` ×105. The dominant pattern is `parseFloat(x) || 0` — invalid/empty
input silently becomes 0. **No negative-value validation, no NaN guards beyond `|| 0`, no
decimal-money type** — all arithmetic is binary floating point, with rounding applied
inconsistently (master nets rounded to 2dp; office nets and several aggregates unrounded
until display). Several save paths re-parse totals from **rendered DOM text**
(e.g. lines 4942–4947, 11291–11296) rather than recomputing from row data. Duplicate
submission is possible: save buttons are not disabled during the async save chain.

---

## 11. Import / export / backup / restore

| Capability | Where | Format & behavior (verified) |
|---|---|---|
| Full backup | ~9553 | Downloads JSON `{version:'2.0', data:{…}}` containing **all 7 stores for ALL users** (not just the logged-in user — AUDIT-035) |
| Restore | `restoreBackup` 9617 | File picker → checks only that `version` and `data` exist → **clears each store and re-inserts records preserving IDs** → clears migration flags (9645) → reloads. No schema/record validation |
| Auto-backup | ~9698 | If `localStorage.lastAutoBackup` is older than **7 days** (constant misleadingly named `TWO_WEEKS` = `7*24*60*60*1000`), prompts/offers a backup download |
| Receipt form → Excel | `exportReceiptToExcel` 10833 | Exports current form grid; separator written as `--- رقم المركبة: <name> ---` with `إجمالي ما سبق: <subtotal>` in the last column; file `<owner>_<date>.xlsx` |
| Excel → receipt form | `importReceiptFromExcel` 10880 | Reads first sheet, **replaces the entire form grid without confirmation**, recognizes separators **only** in the `--- رقم المركبة:` format, trusts file values until the next recalculation; no payment/delivery statuses or totals are imported |
| All receipts → Excel | `exportAllReceiptsToExcel` ~10592 | `prompt()` chooses mode 1/2/3 + summary sheet; separators written as `🔄 رقم المركبة: …` — **incompatible with the importer's expected prefix** (AUDIT-012) |
| Vehicle/office names ⇄ Excel | ~9821+ | Export = names only. Import adds non-duplicate names (exact trimmed match skip). Depends on global `XLSX` via the no-op `loadSheetJS` (9804) — if `xlsx.full.min.js` failed to load, these throw unhandled errors (AUDIT-018) |

**Backup safety notes (do not over-trust):** a backup file is the *only* disaster-recovery
mechanism; restores are all-or-nothing per store with no preview, no per-user selection, and
no integrity checks. A crafted backup could inject arbitrary records (including user accounts)
into the local DB (AUDIT-021). Verify a restore on a spare browser profile before relying on
it.

---

## 12. Printing

Three mechanisms coexist:
1. `window.print()` of the current page (global `@media print` CSS: A4 landscape, `.no-print`
   elements hidden) — used by lists, details pages, notes, receipt form
   (`printReceipt` 6544, `printVehicleDetails` 5377, `printOfficeDetails` 7897, etc.).
2. Generated print windows via `document.write` — debts table (~4622), operations table
   (~5751), external expenses (~8816).
3. `printSingleReceipt` (7913): builds a standalone A4-landscape receipt document.
   Known defects: it draws the header image to a canvas and then ignores the canvas,
   referencing the missing `wasel.png` instead (7930); separator rows span 8+7=15 columns in
   a 16-column table (7972) (AUDIT-013).

Because print windows interpolate stored text into HTML, printed output inherits the same
unescaped-content behavior as the screens (§14).

---

## 13. Limitations & known issues (summary — full register in the audit report)

Most consequential, all evidence-backed in `SYSTEM_AUDIT_REPORT.md`:

1. **Notes data loss:** adding table notes wipes the text note and external-expenses data
   (same DB record overwritten) — AUDIT-001.
2. **Wrong-note edits/deletes** due to sorted display vs unsorted storage indices — AUDIT-002.
3. **Receipt save retry dead-end** after insufficient treasury balance — AUDIT-003.
4. **Non-atomic receipt save** across ≥5 transactions; silent partial failures — AUDIT-004.
5. **Lossy legacy inline receipt editor** (loses row IDs/separators; floods office copies
   with all rows) — AUDIT-005.
6. **Non-cascading bulk deletes & office deletes** orphan or destroy linked data —
   AUDIT-006/007; **de-duplication deletes without merging** — AUDIT-008.
7. **Weak credential storage** (Base64 passwords; full user incl. password in localStorage)
   — AUDIT-019.
8. **Treasury edit sign-flip** for non-`deposit` credits — AUDIT-010.
9. **Excel round-trip incompatibility** between export-all and import formats — AUDIT-012.
10. **Missing assets** (`fonts/`, `wasel.png`) and **missing CSS utilities** — AUDIT-040/041.
11. No tests, no roles, no audit trail, no sync — AUDIT-022/023/045/046.

Until at least items 1–8 are addressed and verified, **do not treat this build as a system of
record for real financial data** without compensating manual backups after every work session.

---

## 14. Security & data-integrity posture ( factual, code-based )

- **Authentication is local and weak:** passwords are Base64-encoded (reversible, not hashed)
  in IndexedDB; the logged-in user object — password included — is persisted in
  `localStorage`. Anyone with access to the unlocked browser profile can read credentials and
  all business data in seconds. There is no lockout, no session expiry.
- **No authorization:** all accounts can perform all actions, including backup export of
  **every** user's data and destructive bulk operations.
- **Stored-XSS surface:** user/office/vehicle names, notes, and imported Excel/backup content
  are interpolated into `innerHTML` (63 sites) and `document.write` print documents without
  escaping. In a single-user local app this is primarily self-inflicted, but malicious
  **import files** (Excel/backup JSON) are a realistic injection vector. *Classified as a
  potential risk — not runtime-exploited during this inspection.*
- **Integrity:** no checksums, no record versioning, no edit history (except treasury
  soft-delete). Concurrent tabs can overwrite each other's embedded-array updates.
- The DevTools blocker (2712) provides **no** actual protection (bypass flag in code).

---

## 15. Testing

**Existing tests: none.** No test files, frameworks, or CI configuration exist in the
repository (verified by full file listing).

Checks actually executed during this documentation phase (non-destructive, static only):
- Full source inventory + structural scans (element/handler cross-reference: all 120 inline
  handlers resolve to defined functions; no duplicate static element IDs).
- JavaScript **syntax** validation: both inline script blocks extracted and checked with
  `node --check` (Node v22.22.3) — **both passed**. This proves parseability only; **no app
  logic was executed and no runtime behavior was tested.**

Recommended manual smoke test (before any real use, on a disposable browser profile with
fictional data): login → add office + owner → create receipt with 2 offices and a separator →
save with and without treasury withdrawal → edit via modal → change row statuses in all three
contexts → debts add/edit → treasury deposit/transfer/daily closing → Excel export/import
round-trip → backup → wipe site data → restore → print each document type. Record results;
several steps are expected to expose §13 items.

---

## 16. Unimplemented, uncertain, or owner-dependent behaviors

**Unimplemented / dead (present in UI or code but not functional):**
- Company name/phone on the receipt **creation** form (inputs don't exist; fields save empty;
  they exist only in the legacy inline editor and on stored records).
- Receipt header image upload (`imageInput/uploadedImage/imagePlaceholder` referenced at
  ~5876–5882; no such elements) and the `ownerWarningMsg` warning element (6149/6164).
- `loadSheetJS` dynamic loading (immediate no-op callback; assumes the static `<script>`
  succeeded).
- Dead functions with zero call sites: `syncDebtFromReceipt` (4455), `renderReceiptRows`
  (6230), `attachReceiptCalculations` (6092), `removeExpenseColumn` (8658).
- Packages purchase flow (display only).

**Business rules that only the owner can confirm (code behavior noted; do not "fix" blindly):**
1. Receipt-save treasury withdrawal is `totals.remaining` — should it be `paid`, `total`, or
   `remaining`? (§6.1)
2. "Profit" = Σ `expense` (صرف) on the client page — is صرف really profit, or a cost?
3. Previous-balance auto-fill ignores `initialBalance` once debt rows exist.
4. `deficit/1000` — confirm deficit unit (kg→ton?) and that weight is in tons.
5. Office net intentionally excludes `expense` and is left unrounded — confirm.
6. "Not received" for offices is derived from **row payment** (`rowPayment`) status — confirm
   payment-vs-receipt terminology mapping.
7. Notification rule (last dated debt row, remaining > 0, ≥ 30 days) and non-persistent
   dismissal — confirm desired behavior.
8. Whether office-context receipt deletion should affect the master receipt (currently:
   office copy only) and whether the refund-on-delete amount (`remaining`) is correct when a
   withdrawal was never taken at save time.

---

## 17. Glossary (UI Arabic → meaning in this system)

| Term | Meaning here |
|---|---|
| كارتة (karta) | A saved payment receipt card (one `receipts` record) shown in vehicle/office details |
| نموذج صرف (sarf form) | The receipt creation form / a receipt document |
| صاحب المركبة | Vehicle owner (`vehicleOwners`) |
| المكتب | Office (`offices`) — intermediary that receives a copy of relevant rows |
| نولون (noloon) | Freight rate per ton used in the net formula |
| عهدة (ohda) | Custody/advance amount deducted per row |
| تعتيق (taktik) | Free-text row field (loading-related note; no calculation role) |
| بوليصة (bolsa) | Policy/waybill reference (free text) |
| عجز (deficit) | Shortage amount, divided by 1000 in the net formula |
| صرف (expense/sarf column) | Per-row amount deducted in master net, excluded in office net; also summed as "profit" on the client page |
| الصافي (net) | Computed row net |
| الرصيد السابق / المدفوع / إضافي / المتبقي | Receipt totals: previous balance / paid / additional / remaining |
| المديونيات | Debts ledgers (per vehicle / per office) |
| الخزنة | Treasury (cash / Vodafone / bank accounts) |
| الاستلام / التسليم | Row-level payment / delivery statuses |
| تم الصرف / تم الأخذ | Receipt-level paid / taken statuses |
| تغيير رقم المركبة | Separator row marking a vehicle-number change within one receipt |

---

## 18. About this document

- Produced by static inspection of commit `3f8a70cf44fed81837c24665cc6838df8e81cd98` on
  2026-10-17. Line numbers refer to that version of `sarfindexed.html`.
- No application files were modified, no database was created or touched, and no real customer
  or financial data was accessed (the repo contains none).
- Runtime behavior (browser rendering, actual IndexedDB upgrades from older versions, print
  layout on physical printers, SheetJS behavior on malformed files) remains **unverified**
  and is flagged as such wherever relevant.
- Companion document: **`SYSTEM_AUDIT_REPORT.md`** — findings register (AUDIT-xxx),
  data-flow/transaction maps, prioritized remediation roadmap, and guidance for the next
  project manager.
