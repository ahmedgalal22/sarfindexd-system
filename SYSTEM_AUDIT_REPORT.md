# System Audit Report — Karta System (`sarfindexed.html`)

| Field | Value |
|---|---|
| Repository | `ahmedgalal22/sarfindexd-system` |
| Inspected version | commit `3f8a70cf44fed81837c24665cc6838df8e81cd98` ("Rename kartacompany_fixed (4).html to sarfindexed.html") |
| Primary artifact | `sarfindexed.html` — 11,616 lines / 652,854 bytes |
| Inspection date | 2026-10-17 |
| Method | **Static code inspection only** (full-file reads + scripted cross-reference scans) + non-destructive syntax checks. **No runtime execution of application logic, no database was created/read/modified, no fixes were applied.** |
| Companion | `README.md` (reference manual). This report is the engineering findings register. |

> **Post-audit change addendum (2026-10-17, after the inspected commit):** the dedicated
> **Notes module was permanently removed** (Notes page, text note, dated table notes,
> external-expenses modal + client-page summary card, and all exclusively-owned
> functions/handlers/CSS — 1,046 lines deleted, 6 lines added in `sarfindexed.html`:
> 11,616 → 10,576 lines). IndexedDB was migrated **v3 → v4**: `onupgradeneeded` now deletes
> the `notes` object store if present and never creates it for fresh databases; all other
> stores/indexes/records are preserved. This migration **permanently erases legacy Notes and
> external-expenses data** on first open of an existing database. Backup no longer exports
> `notes`; `restoreData` no longer lists it, so **old backups containing `notes` restore
> successfully with that entry ignored** (store is not recreated). A saved page state of
> `notes` falls back to the default home page via the existing `if (!pageEl) return;` guard.
> Verification actually performed: `node --check` on both script blocks (pass); handler/ID
> cross-reference (108 handlers, 0 unresolved; 0 remaining references to removed IDs); the
> real extracted `initDB` + `restoreData` executed in Node against IndexedDB stubs
> (fresh/upgrade/restore-old/restore-new: 11/11 pass); the real `restorePageState` executed
> with DOM stubs for `notes`/unknown states (safe fallback, 5/5 pass). **Not** verified in a
> real browser — runtime smoke test on a disposable profile is still required. Consequently:
> **AUDIT-001 and AUDIT-002 are marked "Superseded by removal"** (the feature, not the bug,
> was removed — see §7 annotations and §10). Line numbers throughout this report remain
> references to the inspected pre-removal file; do not treat any other finding as resolved.

> **Post-audit change addendum #2 (2026-10-17, after the Notes-removal change):** the former
> **Packages page was converted into a Terms of Use-only page** in `sarfindexed.html`
> (10,576 → 10,437 lines; 599,943 → 581,446 bytes; app diff **+5 / −144**). Removed: the
> package contact modal, marketing title block, all three package cards (names, prices,
> durations, feature lists, and purchase/contact buttons), the page footer/attribution,
> package-only CSS, `openPkgModal`/`closePkgModal`, and their Escape listener. The user-facing
> sidebar/page name is now **شروط الاستخدام**, and the retained page contains only the
> pre-existing five-item Arabic Terms of Use card; its visible text was preserved verbatim
> (visible-text SHA-256 unchanged:
> `753aea71bc4836c21e6d89abe069ecd39189f209660fdf6f11004d6a4f4ade1c`). No Terms acceptance
> checkbox, acknowledgement recording, or gating mechanism existed before the change, and none
> was added. Internal identifiers `data-page="packages"` and `id="packagesPage"` were
> deliberately retained so sidebar navigation and saved page state remain compatible. No
> package IndexedDB store, package-named record field, or package-specific web-storage use was
> found; therefore **no database version, schema, persisted-record, backup-format, or migration
> change was made**. Verification actually performed: `node --check` on both extracted script
> blocks (pass); five Terms items remain and their visible-text hash matches the pre-edit
> value; 106 inline handlers resolve with 0 unresolved; references to `openPkgModal`,
> `closePkgModal`, and `pkgContactModal` are 0; every sidebar destination has a matching page
> container; and the final `packagesPage` container contains only the Terms card. **Not**
> verified in a real browser — a disposable-profile runtime smoke test is still required.
> Consequently, **AUDIT-042 is marked "Superseded by removal"** because the product no longer
> offers packages; this is not an implementation of a purchase flow. Do not treat any other
> finding as resolved by this change.

**Classification vocabulary used for every finding:**
`Confirmed defect` (the faulty behavior follows necessarily from the code as written; not
necessarily runtime-reproduced) · `Potential risk` (plausible failure needing runtime/scenario
verification) · `Missing capability` · `Unverified behavior` (could not be determined from
available artifacts) · `Business-rule clarification required` (only the owner can say what is
correct).

---

## 1. Executive summary

Karta System is a single-file, offline, Arabic RTL web app (vanilla JS + pre-generated
Tailwind-style CSS + SheetJS + IndexedDB v3) for vehicle-owner payment receipts ("kartas"),
office copies of those receipts, debt ledgers, and a 3-account treasury. There is no backend,
no build, no tests, and no network activity; all data lives in one browser profile.

The codebase is functionally broad but structurally fragile: all logic is global functions in
two inline scripts (239 function definitions), rendering is unescaped `innerHTML`, and the
core financial write path (saving a receipt) is a chain of ≥ 5 independent IndexedDB
transactions with error-swallowing. Static inspection confirmed multiple data-loss and
data-corruption defects, the most severe being: adding table notes **erases** the user's text
note and external-expenses data (AUDIT-001); note edit/delete can target the **wrong record**
(AUDIT-002); a failed treasury-balance check during receipt save leaves the UI in a
**dead state where retry silently does nothing** (AUDIT-003); receipt saving is **non-atomic**
with the money movement committed *before* the receipt (AUDIT-004); a legacy inline receipt
editor **destroys row identity and floods office copies with wrong rows** (AUDIT-005); and
several delete paths **orphan or silently destroy linked financial records**
(AUDIT-006/007/008). Credentials are stored reversibly (Base64) and the full user record
persists in localStorage (AUDIT-019).

**Bottom line:** do not adopt this build as a system of record for real financial data until
the Critical/High items in §9 are remediated and the business rules in §8 are answered. The
app is, however, small enough (one file, no dependencies to upgrade) that a focused hardening
sprint is realistic — see the roadmap.

**Counts:** 46 findings, IDs AUDIT-001…AUDIT-046, each issued once — Confirmed defects 22 ·
Potential risks 11 · Missing capabilities 6 · Unverified behaviors 2 · Business-rule
clarifications 5. *(Two of the 22 confirmed defects — AUDIT-001/AUDIT-002 — were later
superseded by removal of the Notes module; see the addendum above. All other findings
remain open as written against the inspected commit.)*

---

## 2. Architecture & entry points

- **Entry:** `sarfindexed.html` lines 1–9 load only local resources (`cairo-font.css`,
  `tailwind.css`, `xlsx.full.min.js` 0.18.5). Inline CSS 10–381. Body markup 384–2708.
  Script 1: 2709–10730. Modal markup 10732–10830 (`addVehicleModal`, `addOfficeModal`,
  `restoreModal`). Script 2: 10831–11386. Trailing modals
  11388–11616 (`receiptSaveWithdrawModal` 11389, `receiptDeleteRestoreModal` 11443,
  `editReceiptModal` 11496).
- **Boot (`window.load`, ~3591):** DevTools blocker arms at 2712 → `initDB()` (2749) opens
  `KartaSystemDB` v3 → seed 3 users if empty (~2822) → `migrateAddRowIds()` (3512) →
  `migrateFixOfficeRows()` (3442) → restore `localStorage.currentUser` (3599) → restore
  `karta_pageState` → dashboard + notes. Three more `window.load` listeners: 9371, 9534, 9727.
- **Pattern:** page sections toggled by sidebar `data-page` links; every screen re-queries
  IndexedDB directly; state in globals (`db` 2744, `currentUser` 2843, `currentVehicleId`
  4084, `currentOfficeId` 6561, `_rswPendingData` 6247, `_rdrPending*` 7902, `_em*` 11112).
- **No** module system, router, framework, sanitization library, or test harness exists.
- **Missing runtime assets:** `fonts/*.woff2`, `wasel.png` (AUDIT-040).

## 3. Screen / feature inventory (verified implementations)

| # | Screen | Implemented features (traced to code) | Data touched |
|---|---|---|---|
| 1 | Login | atob-compare vs `users`; session → localStorage; logout | R: users |
| 2 | Home/dashboard | `updateDashboardStats` (3405); ≥30-day debt notifications (~8073, dismiss = DOM-only) | R: all stores |
| 3 | Client/reports | date-range aggregates (8236/8292); static FAQ | R: receipts, offices |
| 4 | Receipt form | live row nets/totals; add/remove rows; vehicle separators; datalists; save (§6.1 README); clear; print; Excel export/import of the form grid | C: receipts, vehicleOwners, offices, treasuryTransactions |
| 5 | Vehicles list | CRUD inline, search, received-filter, drag reorder (`sortOrder`), bulk delete, print, name Excel in/out, dedupe | CRUD: vehicleOwners (+cascades vary by path) |
| 6 | Vehicle detail | karta cards w/ row+receipt statuses, filters (date/range/status/text via JSON.stringify), full-view modal, edit (2 impls), delete-with-refund, print, per-receipt note; debts ledger; operations scratchpad; print | CRUD: receipts, vehicleOwners, offices, treasury |
| 7 | Offices list | CRUD inline, search, filters, period filter + totals (net/remaining/collected, not-received), drag reorder, bulk delete, print, name Excel, dedupe | CRUD: offices |
| 8 | Office detail | embedded-copy karta cards w/ statuses, filters, edit (loads master), office-copy delete, debts ledger (`remaining=net−arrived`) | CRUD: offices, receipts |
| 9 | All receipts | date + isPaid + take filters; Σtotal/Σremaining; status selects; edit/print/delete; export-all Excel (modes 1/2/3 + summary) | CRUD: receipts, offices |
| 10 | Treasury | 3 accounts; deposit/withdraw/transfer (mirrored pair, overdraw block); tx list/edit/soft-delete; denominations (display only); daily closing; alerts (localStorage); forecast = Σ office last-debt remaining | CRUD: treasuryTransactions, dailyClosing |
| 11 | Notes | **REMOVED 2026-10-17 (post-audit)** — formerly: text note; dated table notes (add/search/edit/delete/bulk/print); external-expenses modal + client-page summary card. The `notes` store is dropped by the v4 migration (see addendum). Per-receipt/debt note fields in #6/#8 were preserved (independent features) | ~~CRUD: notes~~ |
| 12 | Terms of Use | **CONVERTED 2026-10-17 (post-audit)** — formerly Packages static pricing display. Now read-only display of the pre-existing five-item Arabic Terms of Use only; package cards, prices, feature lists, purchase/contact actions, contact modal, package CSS, and `openPkgModal`/`closePkgModal` removed. Internal page id remains `packages`/`packagesPage` for navigation/saved-state compatibility (see addendum #2). | — |
| — | Global | JSON backup (all stores/all users), restore (clear+reinsert), 7-day auto-backup prompt, page-state restore | all stores |

UI-vs-behavior mismatches are called out per finding (notably AUDIT-011 company fields,
AUDIT-013 print header, AUDIT-014 client "collected", AUDIT-016 dismissals, AUDIT-017
auto-backup naming).

## 4. Function inventory & relationships

239 function definitions (static regex parse incl. nested closures; duplicate names:
`calculateNet` ×2 at 4815/6068 — both nested; `calculateRemaining` ×2 at 4312/7376 — nested in
respective debt modules). All 120 inline `on*` handlers resolve to defined functions; no
duplicate static element IDs (only template-generated ones).

**Call-graph hotspots (verified by reading):**
- `saveReceipt` (6249) → modal → `rswConfirm` (6358) → `tGetAllBalances`, treasury `add`,
  `addDebtRowFromReceipt` (6191) → `receipts.add` → `saveOrUpdateVehicleOwner` (6422),
  `saveToOffices` (6458) → `updateDashboardStats`, `clearReceipt`.
- Rendering: `loadKartas`(4639)→`displayKartas`(5032)→`buildReceiptEditFormHTML`(4676)+
  `initializeReceiptEditForms`(4793); `loadOfficeKartas`(7433); `applyAllReceiptsFilter`(8931)
  →`displayAllReceipts`(9067). Status writes funnel into `setReceiptPaymentStatus`(9325) /
  `setReceiptRowStatus`(9456) which fan out to master + office copies and then re-render up to
  three lists.
- Edit: `openEditReceiptModal`(11116)→`emInsertDataRow`(11153)/`emInsertSeparatorRow`(11189)
  →`emCalcRow`(11219)/`emCalcTotals`(11242)→`emSave`(11286). Legacy parallel path:
  `toggleReceiptEdit`(4872)→`attachReceiptEditCalculations`(4799)→`saveReceiptEdits`(4907).
- Delete: `deleteReceipt`(7907)→`_openRdrModal`(10981)→`rdrConfirm`(11018)→
  `_doDeleteReceipt`(11060)/`_doDeleteOfficeReceipt`(11094).
- Treasury: `executeTreasuryOp`(~10186), `loadTreasuryTransactions`, `saveTreasuryTxEdit`
  (10394), daily closing (~10421). Backup: ~9553 / `restoreBackup`(9617).

The **complete generated function index (name → line)** is in Appendix A.

**Error handling/validation profile:** ~11 `try/catch` sites (2766, 2932, 2937, 9542, 9565,
9633, 9675, 9716, 9839, 9929, 10885); alert×103 / confirm×15 / prompt×1; `parseFloat`×111,
`Number`×49, `parseInt`×11, `toFixed`×105, typically `parseFloat(x)||0`; no systematic
negative/NaN checks; several `onerror` handlers missing or resolving-as-success (6453).

## 5. IndexedDB schema & data-flow map

DB `KartaSystemDB` **v3** at inspection (2751) — **now v4 after the 2026-10-17 removal; the
`notes` store below no longer exists (addendum)**. Stores: `users`(keyPath `username`), `vehicleOwners`
(`id` autoInc, idx `username`), `offices` (`id`, idx `username`), `receipts` (`id`, idx
`username`), `notes` (keyPath `username` — one record per user containing `notes[]`,
`textNote`, `expenses`, `expenseHeaders`), `treasuryTransactions` (`id`; idx `username`,
`accountType`, `date`), `dailyClosing` (`id`; idx `username`, `date`; creation guarded at
2805–2810).

**Data flow (receipt lifecycle):**
```
FORM rows ──saveReceipt──▶ _rswPendingData ──rswConfirm──┬─▶ treasuryTransactions.add (withdraw remaining)
                                                          ├─▶ vehicleOwners.debts.push (if paid>0)
                                                          ├─▶ receipts.add ──▶ id
                                                          ├─▶ vehicleOwners.receipts.push(id) [match by name]
                                                          └─▶ offices[].receipts.push(copy) [match by office name, expense stripped, office-net]
EDIT (modal) emSave ──▶ receipts.put + offices copies re-subset by office name   [single tx: receipts+offices]
EDIT (legacy) saveReceiptEdits ──▶ receipts.put + ALL rows into every office copy, _rowId lost  [single tx]
STATUS setReceipt*Status ──▶ receipts.put + offices copies put (match _rowId, fallback index)
DELETE rdrConfirm ──▶ [treasury deposit remaining] ──▶ receipts.delete + unlink vehicle + strip office copies
BACKUP ──▶ JSON(all 7 stores, all users) ; RESTORE ──▶ clear all + put (IDs preserved) + reset migration flags + reload
```
localStorage: `currentUser` (incl. password), `karta_pageState`, `rowIdMigrationDone`,
`officeRowsMigrationDone`, `lastAutoBackup`, `karta_alerts_<u>`, `karta_denom_<u>`.

Concurrency: none — read-modify-write of whole embedded arrays; multi-tab = last-write-wins.
Recovery: only manual JSON backup. Per-device/per-origin isolation is total.

## 6. Financial calculation & transaction-flow map

```
row.net (master)  = ((weight − deficit/1000)·noloon) − ohda − officeAmount − expense + add − discount   [.toFixed(2)]
row.net (office)  = ((weight − deficit/1000)·noloon) − ohda − officeAmount + add − discount             [unrounded]
totals.total      = Σ data-row nets        debt = previousBalance − paid        remaining = total − paid + additional
separator.subtotal= Σ nets since previous separator
vehicle debt.remaining = balance − paid          office debt.remaining = net − arrived
treasury.balance(acct) = Σ(credit − debit) over deleted=false
SAVE money flow:  treasury.withdraw(totals.remaining) [optional, pre-save] ; vehicle debt row += f(paid) [if paid>0]
DELETE money flow: treasury.deposit(totals.remaining) [optional] — independent of whether a withdrawal ever occurred
client.profit = Σ row.expense ; office period net = Σ office-copy row.net in range ; forecast = Σ offices' last debt.remaining
```
Divergence risks: ≥6 separate implementations of the net formula; office nets unrounded while
master nets rounded; totals sometimes re-parsed from DOM text (4942, 11291); dual status
fields (`isPaid`/`paymentStatus`, `isTaken`/`takeStatus`) written asymmetrically to copies
(9325).

## 7. Findings register

> Severity is an engineering judgment for prioritization; classification states the evidence
> type. All paths are `sarfindexed.html` unless noted. "Static" = proven by code reading;
> runtime reproduction was out of scope (documentation phase; no DB may be seeded).

### Confirmed defects

**AUDIT-001 — Confirmed defect — Critical. — STATUS (2026-10-17): SUPERSEDED BY REMOVAL of the
Notes module (see addendum). The defect was not individually fixed; the feature and its
`notes` store were deleted.** Adding table notes **deletes the text note and
all external-expenses data**. Evidence: the `notes` record jointly holds `notes[]`,
`textNote` (written at 3028), `expenses`/`expenseHeaders` (8715/8718); the add-notes handler
writes `store.put({username, notes: existingNotes, lastModified})` at **3161–3165**, replacing
the whole record. Impact: silent, irreversible loss of two features' data on a routine action.

**AUDIT-002 — Confirmed defect — High. — STATUS (2026-10-17): SUPERSEDED BY REMOVAL of the
Notes module (see addendum).** Note edit/delete/bulk act on the **wrong note**.
`displayNotes` sorts by date desc (3217) and embeds the *sorted* index in buttons (3229/3234);
`editNote`/`saveNoteEdit`/`deleteNote` re-read the *stored, unsorted* array and apply that
index (3255, 3298, 3324). Impact: edits/deletes hit a different record than the one displayed.

**AUDIT-003 — Confirmed defect — High.** Receipt-save retry is a dead end after
insufficient-balance. `rswConfirm` nulls `_rswPendingData` at 6363, *then* checks balance
(6372) and re-shows the modal (6374); every retry exits at the `if (!_rswPendingData) return`
guard (6360) with no message. Impact: user believes another account can be picked; nothing is
saved and nothing says so.

**AUDIT-004 — Confirmed defect — Critical.** Receipt save is **non-atomic across ≥ 5
transactions**, ordered money-first: treasury withdrawal (6379) → debt append (6402) → receipt
insert (6405) → owner link (6411) → office copies (6412). `saveOrUpdateVehicleOwner` resolves
on `transaction.onerror` (6453), so link failures are invisible. Impact: crash/close mid-chain
yields money moved without a receipt, receipts without copies, or unlinked receipts — with a
success alert shown whenever the receipt insert itself succeeded.

**AUDIT-005 — Confirmed defect — High.** Legacy inline editor `saveReceiptEdits` (4907) is
lossy and divergent: rebuilt rows have **no `_rowId`** (4921–4940), separator rows become data
rows (the edit form at 4721 renders every row as inputs), and **every office copy receives all
rows** (`rows.map` unfiltered at 4984) — unlike `emSave`'s per-office subset (11348–11356).
Impact: row-status sync breaks (falls back to index), office balances inflate with other
offices' rows.

**AUDIT-006 — Confirmed defect — High.** Vehicle **bulk delete** (~4059) deletes only
`vehicleOwners` records, while single delete `deleteVehicleOwner` (3942) cascades to receipts
and office copies. Impact: orphaned receipts remain in `receipts`/office copies and in
all-receipts/aggregates.

**AUDIT-007 — Confirmed defect — High.** `deleteOffice` (6909) and office bulk delete (6974)
delete only the office record — destroying its embedded receipt copies and debt ledger with
it, no warning listing affected receipts, master receipts untouched (rows still reference the
office by name). Impact: one click destroys an office's financial history; subsequent receipt
saves for that office name fail validation or recreate a blank office depending on path.

**AUDIT-008 — Confirmed defect — High.** `removeDuplicateVehicles` (~9989) /
`removeDuplicateOffices` (~10028) keep the first record per lowercased name and delete the
rest **without merging** receipts/debts/links. Impact: the duplicates' embedded financial data
is lost; button label gives no hint of this.

**AUDIT-009 — Confirmed defect — Medium.** `recalculateAllDebts` (4332) recomputes only
`index === 0` (4346–4352); all other rows are skipped. Impact: stale `remaining` values
persist after edits that rely on recalculation.

**AUDIT-010 — Confirmed defect — High.** `saveTreasuryTxEdit` (10394) computes
`isCredit = (opType === 'deposit')` (10401). Any credit-side transaction whose `opType` is not
exactly `deposit` (e.g. transfer-in records created by `executeTreasuryOp`, receipt-delete
refunds) is rewritten as a **debit** when edited. Impact: editing history can flip money
direction.

**AUDIT-011 — Confirmed defect — Medium.** New receipts always store empty company fields:
`saveReceipt` reads `companyName/companyPhone` via `?.` from inputs that do not exist in the
form (getElementById-vs-static-ID audit; ~6252). The fields exist on records/edit-print, so
company info can never be captured at creation. Related dead UI refs: image-upload elements
(~5876–5882), `ownerWarningMsg` (6149/6164).

**AUDIT-012 — Confirmed defect — Medium.** Excel formats are mutually incompatible:
`exportAllReceiptsToExcel` writes separators as `🔄 رقم المركبة: …`; `importReceiptFromExcel`
recognizes only `--- رقم المركبة:` (10902 — matching `exportReceiptToExcel`'s format at
10845). Impact: export-all files cannot be re-imported; separators become garbage data rows.

**AUDIT-013 — Confirmed defect — Low.** `printSingleReceipt` (7913): draws header to canvas
then discards it, emitting `<img src="wasel.png">` (7930, file missing); separator row spans
8+7=15 of 16 columns (7972–7973). Impact: broken header image + table misalignment in print.

**AUDIT-014 — Confirmed defect — Medium.** Client page `calculateOfficesTotal` (8292)
computes its "collected now" figure without the selected date range applied elsewhere on the
same screen. Impact: same-screen numbers use different periods; misleading reporting.

**AUDIT-015 — Confirmed defect — Low.** `editVehicleOwner` (3865) replaces the 8-cell row
with 6 cells. Impact: column misalignment during inline edit (office inline edit at 6843 does
preserve cell count).

**AUDIT-017 — Confirmed defect — Low.** Auto-backup cadence is 7 days but the constant is
named `TWO_WEEKS` (9698: `7 * 24 * 60 * 60 * 1000`). Impact: maintainer confusion; verify
intended cadence with owner (§8).

**AUDIT-018 — Confirmed defect — Medium.** `loadSheetJS` (9804) is a no-op that immediately
invokes its callback assuming global `XLSX`. If `xlsx.full.min.js` fails to load (missing
file when the HTML is moved alone), every Excel feature throws an unhandled `ReferenceError`
with no user message. Impact: silent feature death in a plausible deployment mistake.

**AUDIT-019 — Confirmed defect — Critical (security).** Passwords are stored Base64-encoded
(seed ~2822–2829; verified decodable — values not reproduced here), compared via `atob` with a
plaintext fallback, and the **full user record including password** is persisted in
localStorage `currentUser` (restored 3599). Impact: trivial credential disclosure to anyone
with profile access or any future XSS; session never expires.

**AUDIT-035 — Confirmed defect — High (privacy/integrity).** Backup exports **all 7 stores
for all users** (9553/9562) regardless of who is logged in; restore replaces all users' data.
Impact: any account can exfiltrate/overwrite every other account's data on the device.

**AUDIT-036 — Confirmed defect — Medium.** `emSave` office-copy update falls back to stale
rows: `officeGroups[office.name] || r.rows` (11367). Removing the last row(s) of an office in
the editor leaves the office copy unchanged. Impact: office totals keep deleted rows.

**AUDIT-040 — Confirmed defect — Medium.** Referenced assets are absent from the repo:
`fonts/*.woff2` (font falls back) and `wasel.png` (1855, 7930). Verified by directory listing.

**AUDIT-041 — Confirmed defect — Low.** Used utility classes absent from both CSS sources —
e.g. `max-w-[98vw]`, `max-h-[96vh]`, `max-h-[90vh]`, `min-w-[90px]`, `-top-1`, `-right-1`,
`bg-opacity-60`, `border-orange-300` (grep counts: 0 in `tailwind.css`/inline style, ≥1 in
markup). Modal size caps and badge offsets silently don't apply. Root cause: `tailwind.css`
was generated for a different source filename per its own header.

### Potential risks

**AUDIT-020 — Potential risk — High.** Stored-XSS surface: 63 `innerHTML` interpolation sites
with unescaped record text + 4 `document.write` print docs (4622, 5751, 8011, 8816). Primary
vector: imported Excel/backup content rendered later. Not exploited during inspection
(static-only mandate); treat as real until escaping/sanitization is added.

**AUDIT-021 — Potential risk — High.** Restore validates only `version`+`data` presence
(9617+), then bulk-inserts attacker-controllable records (including `users`). Combined with
AUDIT-019/020, a malicious backup = credential + script injection. Also: restore clears
migration flags (9645) — re-migration idempotency on restored data is unproven.

**AUDIT-025 — Potential risk — High.** Formula duplication/divergence: master net ×3 (4815,
6068, 11219), office net ×3 (4994, 6473, 11354), totals ×3; rounding differs by path. Any
future formula change must be applied in ≥6 places or totals diverge by entry path.

**AUDIT-026 — Potential risk — Medium.** Numeric robustness: `parseFloat||0` everywhere silences
bad input; negatives accepted; money in binary float; totals re-parsed from DOM text (4942,
11291); refund amount re-parsed from modal text (11022 — safe today only because `tFmt`=10098
is locale-free `toFixed(2)`). Duplicate submission: save buttons never disabled during async
chains.

**AUDIT-033 — Potential risk — Medium.** Multi-tab concurrency: whole-record
read-modify-write (vehicle.receipts push 6446, office copies map) with no versioning →
last-write-wins lost updates.

**AUDIT-037 — Potential risk — Medium.** `openEditReceiptModal` always loads from `receipts`
(11120) even with `context='office'`; office copies whose master was deleted/bulk-orphaned
cannot be edited (alert "not found"), and office users unknowingly edit the master record.

**AUDIT-038 — Potential risk — Medium.** `importReceiptFromExcel` wipes the form grid without
confirmation (10897); trusts cell values (incl. `net`, 10952) until recalculation; Excel date
cells may arrive as serial numbers — untested.

**AUDIT-039 — Potential risk — Medium.** Dual-generation status fields with asymmetric writes
(master gets both `isPaid`+`paymentStatus`, office copies only `paymentStatus`, 9325; take
status mirrors this). Any reader using the "other" field sees stale status (all-receipts
filter keys off `isPaid`, 8931).

**AUDIT-043 — Potential risk — Low.** Unguarded string ops on possibly-missing fields, e.g.
`v.name.toLowerCase()` / `vehicleNumber.toLowerCase()` in searches (3783, 6695) — one
malformed/imported record can throw and blank a list.

**AUDIT-031 — Potential risk — Low.** DevTools blocker (2712–2740, Ctrl+B bypass) obstructs
legitimate debugging/support and gives false security confidence.

**AUDIT-032 — Potential risk — Low.** Listener accumulation: `updateBulkDelete*Visibility`
re-adds `change` listeners to every checkbox on each call (e.g. 6959–6961; vehicles ~4044).
Harmless today (idempotent handler), but grows with list refreshes.

### Missing capabilities

**AUDIT-022 — Missing capability — High.** Zero automated tests / CI (verified: no test files
in repo). Financial formulas and DB flows have no regression safety net.
**AUDIT-023 — Missing capability — Medium.** No roles/authorization; no user-management UI;
no password change. **AUDIT-024 — Missing capability — Low.** Dead code with zero call sites:
`syncDebtFromReceipt` (4455), `renderReceiptRows` (6230), `attachReceiptCalculations` (6092),
`removeExpenseColumn` (8658) — removal candidates that currently mislead maintainers.
**AUDIT-042 — Missing capability — Low — Superseded by removal 2026-10-17.** Packages page has no purchase/activation logic. *(Historical finding retained for context: the Packages feature has since been removed and the page converted to Terms of Use only — see Post-audit change addendum #2 and inventory row #12. This does not implement a purchase flow.)*
**AUDIT-045 — Missing capability — Medium.** No audit trail: edits/deletes of financial
records leave no history (treasury soft-delete excepted). **AUDIT-046 — Missing capability —
Medium.** No sync/server backup; single-browser data residency (operational constraint, see
README §9.5).

### Unverified behaviors

**AUDIT-034 — Unverified behavior — High.** IndexedDB upgrade from real v1/v2 production
databases: `onupgradeneeded` creates missing stores, but no legacy DB fixture exists to prove
data survives. Do not ship schema changes before obtaining a production backup fixture.
**AUDIT-016 — Unverified behavior — Low.** Notification dismissal is DOM-only (verified in
code); whether non-persistence is *intended* is unknown — confirm (§8).
**AUDIT-041b (under AUDIT-041) — Unverified:** full Tailwind class coverage — heuristic scan
found 523 static classes vs 401 CSS selectors but cannot reliably separate JS-hook classes
(`em-*`, `edit-*`); visual QA required. (Folded into AUDIT-041; not a separate ID.)

*Note: AUDIT-016 is classified Unverified (intent unknown), the DOM-only mechanism itself is
code-verified.*

### Business-rule clarifications required

**AUDIT-028** — Should receipt-save withdraw `remaining` (current, 6365), `paid`, or `total`
from the treasury? And should delete-refund deposit `remaining` even when no withdrawal was
taken at save? **AUDIT-029** — Is "profit" truly Σ `expense`/صرف (client page), or is صرف a
cost and profit defined otherwise? **AUDIT-030** — Previous-balance rule: last debt
`remaining` (current, ~6178) vs `initialBalance` for first receipt — confirm intended
precedence. **AUDIT-044** — Units: is `weight` in tons and `deficit` in kg (hence `/1000`)?
Which fields may legitimately be negative? **AUDIT-027** — Office "not received" is computed
from row **payment** status (6748–6762): is that the intended business meaning? Client-page
period semantics and office-net exclusion/rounding intent are covered under AUDIT-014 and
AUDIT-025 respectively and restated as owner questions in §8. *(Business-rule IDs issued:
027, 028, 029, 030, 044.)*

## 8. Critical unanswered business-rule questions (for the owner)

1. Treasury on receipt save: withdraw `remaining` vs `paid` vs `total`? On delete: refund
   only if a withdrawal actually happened? (AUDIT-028)
2. Definition of profit on the client page (Σ صرف?) and the date semantics of "collected
   now". (AUDIT-029, AUDIT-014)
3. Previous-balance source of truth and the role of `initialBalance`. (AUDIT-030)
4. Units for weight/deficit; allowed sign ranges per field. (AUDIT-044)
5. Is office net's exclusion of `expense` and non-rounding intentional accounting, or
   drift? (AUDIT-025)
6. Should office "received/not-received" track payment or physical receipt? (AUDIT-027)
7. Should deleting from office context affect the master receipt? Should office/owner
   renames rewrite history? (AUDIT-037, §5 linkage)
8. Auto-backup cadence: 7 days (code) or two weeks (name)? (AUDIT-017)
9. Are the three seeded accounts meant to be equal-privilege? Must backups be per-user?
   (AUDIT-023, AUDIT-035)

## 9. Automated checks actually executed (results)

Executed 2026-10-17, non-destructively (working copies in `/tmp`; the repository and any
browser data were not touched):

| Check | Tool | Result |
|---|---|---|
| JS syntax, Script 1 (380,704 chars extracted) | `node --check` (Node v22.22.3) | **PASS** (parse only) |
| JS syntax, Script 2 (32,100 chars) | `node --check` | **PASS** (parse only) |
| Inline-handler → function cross-reference | Python static scan | 120 handlers, **0 unresolved** |
| `getElementById` literals vs static IDs | Python static scan | 209 literals / 287 static IDs; missing-but-referenced inputs found (AUDIT-011); `txEditModal` is dynamically created (not missing) |
| Duplicate static IDs | Python | **None** (only `${…}` templates) |
| Dead-code scan | Python | 4 definition-only functions (AUDIT-024) |
| CSS class presence spot-checks | grep | AUDIT-041 classes confirmed absent |
| Asset existence | `ls`/`find` | `fonts/`, `wasel.png` **absent** |
| Automated test suite | — | **Does not exist; nothing to run** |

**Explicitly not done:** no browser execution, no IndexedDB creation, no functional,
calculation, print, import/export, or backup/restore runtime verification. No finding above
claims runtime reproduction.

**Missing safeguards/tests to add first:** formula unit tests (net/totals/office-net),
save-chain integration test on a throwaway profile, backup→restore round-trip test, upgrade
test with a real legacy backup fixture, XSS escaping tests on all render paths.

## 10. Prioritized remediation roadmap

**Critical (before any real financial data):**
1. ~~AUDIT-001 notes record overwrite~~ — **done by alternative means (2026-10-17): the Notes
   module + store were removed entirely instead of merge-writing** (see addendum). No
   regression test exists yet for the migration in a real browser — add one.
2. AUDIT-004 atomic receipt save (single multi-store transaction or compensating rollback);
   stop swallowing errors (6453).
3. AUDIT-019 credential storage: salted hash (e.g. WebCrypto PBKDF2), never persist password/
   full user in localStorage; add session expiry.
4. AUDIT-035 per-user backup scoping (or explicit admin-only with warning).

**High:**
5. ~~AUDIT-002 stable note IDs (index→id)~~ — **moot: Notes module removed (2026-10-17)**.
6. AUDIT-003 keep pending data until success/cancel.
7. AUDIT-005 remove or repair the legacy inline editor; one edit path only. 8. AUDIT-006/007/
008 cascading/merge semantics + confirm dialogs listing impact. 9. AUDIT-010 treasury edit
sign logic. 10. AUDIT-020 output escaping for all `innerHTML`/`document.write` paths.
11. AUDIT-021 restore validation (schema whitelist, reject `users` overwrite or confirm).
12. AUDIT-025 single shared net/totals implementation. 13. AUDIT-022 seed a test harness.
14. AUDIT-034 obtain legacy-DB backup fixture; test upgrade before schema changes.

**Medium:** AUDIT-009, 011, 012, 014, 018, 026, 033, 036, 037, 038, 039, 040 (fonts), 045,
046 (document the constraint for users), 023.
**Low:** AUDIT-013, 015, 016, 017, 024, 031, 032, 041, 042, 043.

**Do-not-change-without-analysis zones:** the `_rowId` status-sync protocol (9456) and its
migrations (3442/3512) — restored/legacy data depends on them; the dual status fields
(`isPaid` etc.) — filters and copies read different generations; office-copy embedding format
(backup compatibility); `tFmt`'s plain format (parsers at 11022/4942/11291 depend on it);
seeded usernames (existing DBs key data by them).

## 11. Guidance for the Next AI Project Manager

- **Ground truth:** `sarfindexed.html` @ `3f8a70c` is the whole system. Trust README.md +
  this report's line numbers only against that commit; re-grep before citing after any edit.
- **Never test against a production browser profile.** All verification must use a disposable
  profile + fictional data, because restore/wipe operations are destructive and backups are
  all-user. Obtain a real (anonymized) backup JSON fixture early — AUDIT-034 blocks safe
  schema work without it.
- **Sequencing:** fix Critical data-loss/atomicity first (AUDIT-001/002 now superseded by the
  Notes removal; start with AUDIT-003/004), then delete
  -path semantics (006/007/008) **together with owner answers** to §8 Q1/Q7 — these change
  money semantics and must not be "fixed" by assumption. Do not let any agent renumber or
  close audit IDs; reference them in every task/PR.
- **Two-editor trap:** until AUDIT-005 is resolved, any task touching receipt editing must
  state which editor it targets (modal = current UI; inline = legacy hazard). Reject changes
  that patch only one net-formula copy (AUDIT-025 lists all sites).
- **Change control:** require, per change: affected AUDIT IDs, stores touched, migration
  impact, backup-compatibility note, and a manual test script result on a throwaway profile.
  The repo has no tests — budget the harness (AUDIT-022) into the first sprint.
- **Known unknowns to re-verify at runtime (first QA pass):** actual print layouts
  (AUDIT-013), Excel serial-date import (AUDIT-038), multi-tab behavior (AUDIT-033), CSS
  visual coverage (AUDIT-041), and every §8 business rule against the owner's real workflow.
- **Security posture to communicate:** local-only ≠ safe. Until AUDIT-019/020/021/035 are
  fixed, treat device/profile access as full data access, and backup files as sensitive
  (they contain all users' data and encoded passwords).
- **Out-of-scope guardrail:** this phase was documentation-only by directive; no code was
  changed. Await explicit owner/PM approval before remediation work begins.

---

## Appendix A — Complete function index (generated by static parse)

239 definitions, `name — line`, in source order. Nested closures are included; a name
appearing twice denotes two definitions (see §4). This index is mechanical output — verify
against the file before acting on any single entry.

<!-- FUNCTION_INDEX_START -->
```text
  2749  initDB                                   7740  calculateOfficeReceived
  2822  addDefaultUsers                          7756  calculateOfficeNotReceived
  2873  showError                                7771  recalcOfficeTotal
  2880  showDashboard                            7792  calculateOfficeTotalNet
  2931  savePageState                            7805  applyOfficeKartaFilter
  2936  restorePageState                         7846  searchOfficeKartasByDate
  2977  switchNotesTab                           7897  printOfficeDetails
  3003  loadNotes                                7907  deleteReceipt
  3040  loadTextNote                             7913  printSingleReceipt
  3114  removeNoteRow                            8021  saveReceiptNote
  3182  loadTableNotes                           8040  deleteOfficeReceipt
  3204  displayNotes                             8047  saveOfficeReceiptNote
  3248  editNote                                 8073  toggleNotificationsPanel
  3286  saveNoteEdit                             8091  checkOverdueDebts
  3315  deleteNote                               8135  updateNotificationBell
  3349  updateBulkDeleteNotesVisibility          8146  displayNotificationsPanel
  3405  updateDashboardStats                     8194  navigateToVehicle
  3442  migrateFixOfficeRows                     8202  dismissNotificationItem
  3512  migrateAddRowIds                         8231  applyClientFilter
  3614  openAddVehicleModal                      8236  loadClientData
  3621  closeAddVehicleModal                     8292  calculateOfficesTotal
  3625  buildVehicleRow                          8372  calculateTotalPending
  3636  modalAddVehicleRow                       8395  updateFAQData
  3642  modalSaveVehicles                        8404  toggleFAQ
  3705  removeAddRow                             8418  updateFAQ6
  3767  loadVehicleOwners                        8498  setClientPeriod
  3795  displayVehicleOwners                     8521  setOfficePeriod
  3865  editVehicleOwner                         8544  setVehiclePeriod
  3908  saveVehicleEdit                          8569  openExpensesModal
  3942  deleteVehicleOwner                       8576  closeExpensesModal
  4039  updateBulkDeleteVisibility               8581  addExpenseRow
  4087  showVehicleDetails                       8627  addExpenseColumn
  4125  backToVehiclesList                       8647  removeExpenseRow
  4133  switchVehicleTab                         8658  removeExpenseColumn
  4161  loadDebts                                8674  calculateTotalExpenses
  4219  createDebtRowHTML                        8687  attachExpenseCalculations
  4268  attachInitialBalanceListener             8696  saveExpenses
  4297  attachDebtCalculations                   8735  loadExpensesInModal
  4306  attachSingleRowCalculations              8811  printExpenses
  4312  calculateRemaining                       8908  setAllReceiptsPeriod
  4332  recalculateAllDebts                      8931  applyAllReceiptsFilter
  4357  addDebtRow                               9027  updateReceiptStatus
  4384  removeDebtRow                            9067  displayAllReceipts
  4401  saveDebts                                9229  highlightRows
  4455  syncDebtFromReceipt                      9295  getPaymentStatus
  4502  printDebtsTable                          9299  filterReceiptsByStatus
  4639  loadKartas                               9304  setupKartaFilters
  4676  buildReceiptEditFormHTML                 9325  setReceiptPaymentStatus
  4793  initializeReceiptEditForms               9366  updatePaymentStatusFromSelect
  4799  attachReceiptEditCalculations            9379  getTakeStatus
  4815  calculateNet                             9383  filterReceiptsByTakeStatus
  4848  calculateEditTotals                      9388  setupTakeFilters
  4872  toggleReceiptEdit                        9409  setReceiptTakeStatus
  4907  saveReceiptEdits                         9451  updateTakeStatusFromSelect
  5024  getDayName                               9456  setReceiptRowStatus
  5032  displayKartas                            9530  updateRowPaymentStatus
  5195  searchKartasByDate                       9531  updateRowDeliveryStatus
  5257  viewFullReceipt                          9541  manualBackup
  5377  printVehicleDetails                      9553  createBackup
  5382  applyVehicleKartaFilter                  9580  getAllFromStore
  5438  openOperationsManagement                 9592  downloadBackup
  5594  addOperationRow                          9606  openRestoreModal
  5628  removeOperationRow                       9611  closeRestoreModal
  5640  addOperationColumn                       9617  restoreBackup
  5673  printOperationsTable                     9671  restoreData
  5767  saveOperationsData                       9697  scheduleAutoBackup
  5806  loadVehicleOwnersForReceipt              9715  performAutoBackup
  5821  loadOfficesForReceipt                    9735  initDragDrop
  5838  setCurrentDate                           9773  saveVehiclesOrder
  5892  addVehicleChangeRow                      9788  saveOfficesOrder
  5945  fillFromPreviousRow                      9804  loadSheetJS
  5959  addReceiptRow                            9809  exportVehiclesToExcel
  6033  removeReceiptRow                         9831  importVehiclesFromExcel
  6045  updateRowNumbers                         9899  exportOfficesToExcel
  6053  attachRowCalculation                     9921  importOfficesFromExcel
  6068  calculateNet                             9989  removeDuplicateVehicles
  6092  attachReceiptCalculations               10028  removeDuplicateOffices
  6100  calculateTotals                         10079  tCalcBalance
  6191  addDebtRowFromReceipt                   10091  tGetAllBalances
  6230  renderReceiptRows                       10098  tFmt
  6249  saveReceipt                             10101  loadTreasury
  6358  rswConfirm                              10128  tRenderDenominations
  6422  saveOrUpdateVehicleOwner                10144  tUpdateDenomTotal
  6458  saveToOffices                           10157  saveCashDenominations
  6526  clearReceipt                            10165  switchTreasuryTab
  6544  printReceipt                            10186  executeTreasuryOp
  6565  openAddOfficeModal                      10260  loadTreasuryTransactions
  6569  closeAddOfficeModal                     10316  deleteTreasuryTx
  6572  buildOfficeRow                          10332  editTreasuryTx
  6582  modalAddOfficeRow                       10394  saveTreasuryTxEdit
  6588  modalSaveOffices                        10421  executeDailyClosing
  6620  removeOfficeRow                         10453  tLoadClosingHistory
  6680  loadOffices                             10486  saveAlerts
  6706  displayOffices                          10497  tLoadAlerts
  6834  editOffice                              10505  tCheckAlerts
  6877  saveOfficeEdit                          10526  tLoadForecast
  6909  deleteOffice                            10552  _buildReceiptRows
  6955  updateBulkDeleteOfficesVisibility       10592  exportAllReceiptsToExcel
  7002  setOfficesListPeriod                    10726  openPkgModal
  7025  applyOfficesListFilter                  10727  closePkgModal
  7065  displayOfficesWithFilter                10833  exportReceiptToExcel
  7189  showOfficeDetails                       10880  importReceiptFromExcel
  7221  backToOfficesList                       10981  _openRdrModal
  7229  switchOfficeTab                         11018  rdrConfirm
  7254  loadOfficeDebts                         11060  _doDeleteReceipt
  7326  addOfficeDebtRow                        11094  _doDeleteOfficeReceipt
  7359  removeOfficeDebtRow                     11116  openEditReceiptModal
  7369  attachSingleOfficeDebtRow               11148  closeEditReceiptModal
  7376  calculateRemaining                      11153  emInsertDataRow
  7387  attachOfficeDebtCalculations            11189  emInsertSeparatorRow
  7393  saveOfficeDebts                         11212  emAttachRowCalc
  7433  loadOfficeKartas                        11219  emCalcRow
  7476  displayOfficeKartas                     11220  v
  7617  calculateVehicleDelivered               11229  emUpdateSeparatorSubtotals
  7634  calculateVehicleNotDelivered            11242  emCalcTotals
  7651  calculateVehicleReceived                11257  emAddRow
  7668  calculateVehicleNotReceived             11263  emAddSeparator
  7685  recalcVehicleTotal                      11274  emRemoveRow
  7708  calculateOfficeDelivered                11286  emSave
  7724  calculateOfficeNotDelivered
```
<!-- FUNCTION_INDEX_END -->

## Appendix B — Inspection coverage statement

Inspected in full: all 5 repository files; every line range of `sarfindexed.html` was read
across the inspection (HTML, CSS, both scripts, trailing modals), supplemented by scripted
cross-reference scans (functions, handlers, element IDs, storage calls, parse/format counts,
try/catch inventory). Not inspectable: any production IndexedDB contents (none in repo; none
created), the missing `fonts/` binaries and `wasel.png` (absent), SheetJS internals beyond
version identification (minified third-party, used as-is), and all runtime behavior.
version identification (minified third-party, used as-is), and all runtime behavior.
