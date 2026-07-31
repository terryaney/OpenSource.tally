# Multi-Year Date Filtering for Spending Reports

This plan is saved in the repo at `.vscode/Plans/UI-tweaks.md` as the persisted design doc for the `feature/ui-tweaks` branch.

## Context

The report is titled "Spending Analysis 2026" and the date-filter dropdown only offers individual months, but the underlying data (and the analyzer/report pipeline) has no single-year restriction — multi-year datasets already flow through fine. The single-year assumption is confined to presentation: a frozen `{year}` in the default title, no quarter/year grouping in the date filter, and transaction rows that never show a year at all. On top of that, a real bug was found: an April 2026 transaction (a negative-value "Delta American Express Payment" transfer) is completely missing from the date-filter dropdown, even though it's visible in Merchant/Category view.

Goal: fix the dropdown bug, add Month/Quarter/Year/Custom-range date filtering (additive/OR semantics, matching the existing month-filter behavior the user already likes), fix transaction rows to show the year when needed, and stop hardcoding a single year into the default report title.

This is presentation-layer only — no changes to `section_engine.py`'s merchant-matching/rule logic (the project's guarded "critical" rule engine) and no `analyzer.py` changes, since `analyzer.py:220-260` already tracks the full multi-year month/year set with no filtering.

---

## Part 1 — Bug fix: `availableMonths` drops months absent from `sections`

**Root cause** (`src/tally/spending_report.js:1337-1368`): `availableMonths` sources from `spendingData.value.sections` whenever any views are configured (`views_file` in settings.yaml), and only falls back to `categoryView` when `sections` is completely empty — never merges. `sections` comes from `section_engine.classify_merchants()` (evaluates each merchant's aggregate data against view filter expressions) — a merchant matching **zero** views is silently absent from `sections`, by design (catch-all filters are an intentional feature, not something to change). `categoryView` (`report.py:236-309`, `build_category_view()`) is built from `stats['by_merchant']` — **all** merchants, unfiltered by views — so it is always a superset of every month `sections` could ever contain.

**Fix:** Change `availableMonths` to source exclusively from `categoryView`, removing the `sections`-branch entirely (not a merge — `categoryView` is already a strict superset, so merging would be redundant work):

```js
const availableMonths = computed(() => {
    const months = new Set();
    const categoryView = spendingData.value.categoryView || {};
    for (const category of Object.values(categoryView)) {
        for (const subcat of Object.values(category.subcategories || {})) {
            for (const merchant of Object.values(subcat.merchants || {})) {
                for (const txn of merchant.transactions || []) {
                    months.add(txn.month);
                }
            }
        }
    }
    return Array.from(months).sort().map(m => ({ key: m, label: formatMonthLabel(m) }));
});
```

No existing test exercises this branch (`tests/test_report_html.py` fixtures never configure `views_file`, confirmed via grep — `availableMonths`/dropdown month lists aren't asserted anywhere today).

**New regression test** in `tests/test_report_html.py`: a fixture with `views_file` pointing at a `views.rules` that excludes at least one merchant from every view, asserting that merchant's month still appears in the date-filter UI (Part 2's month grid) rather than silently vanishing.

This is a prerequisite for Part 2, not a standalone fix made obsolete by it: Part 2's year tabs and month grid are both built from `availableMonths`, so they'd inherit this exact bug unfixed.

---

## Part 2 — Multi-granularity date filter (Month / Quarter / Year / Custom)

### Design source
This implements the "Structured Categories + Drill-Down Calendar" design (Option 6) prototyped and interaction-tested at `.vscode/prototype/datefilter.html` — a working, browser-verified vanilla-JS reference for every interaction below (shared pending/aggregation logic ~js:1259-1420, drill calendar ~js:1421-1600, wiring ~js:1680-1724, markup ~html:592-651). Port its logic into Vue's reactive idioms rather than re-deriving the design; don't modify the prototype file itself, it's reference-only.

### Current mechanics
- `spending_report.html:42-47`: one `<optgroup label="Individual Months">` populated from `availableMonths`, wired to `addMonthFilter` → `addFilter(text, 'month', displayText)` (js:1461-1463, 1440-1446), pushing `{ text, type: 'month', mode, displayText }` onto `activeFilters`.
- `passesFilters` (js:1372-1395): filters are grouped **by `type`**, OR'd within a type, AND'd across types. This is why multiple month selections currently union together — and why quarter/year/custom must NOT become their own distinct AND'd types, or "Q1 2026 + all of 2025" would incorrectly AND instead of union.
- `monthMatches`/`expandMonthRange` (js:1432-1438, 1609-1621) already support an unused `"YYYY-MM..YYYY-MM"` range-string format via lexicographic comparison (works because `YYYY-MM` is fixed-width) — Part 2 makes this plumbing load-bearing for the first time (see Aggregation below).

### Chip data model (unchanged from original sketch)
Keep `month` as the literal chip `type` for Month/Quarter/Year (reusing the existing range-string support — **zero changes needed** to `matchesFilter`'s `'month'` case or `monthMatches`), and add one new `daterange` type only for day-precision custom ranges:

- Month: `{ type: 'month', text: '2026-04', displayText: 'Apr 2026' }`
- Quarter: `{ type: 'month', text: '2026-04..2026-06', displayText: 'Q2 2026' }`
- Year: `{ type: 'month', text: '2026-01..2026-12', displayText: '2026' }`
- Custom: `{ type: 'daterange', text: '2026-01-05..2026-04-13', displayText: 'Jan 5 – Apr 13, 2026' }`

To keep `month` and `daterange` OR'd together (not AND'd), introduce a `filterCategory(type)` helper (`month`/`daterange` → `'date'`, else passthrough) and group by `filterCategory(f.type)` instead of raw `f.type` in `passesFilters` (js:1372-1395). `matchesFilter`'s per-chip `switch` still keys on the real `type`, so add one new `case 'daterange'` there using a day-precision comparison (see helper below).

### UI structure
Replace the current single `<optgroup>` dropdown (`spending_report.html:42-47`) with a popover (triggered by the existing "+ Date filter" button) containing, top to bottom:
1. **This row** — 3 pills: This Month / This Quarter / This Year (computed from today's date, not a static list).
2. **Last row** — 3 pills: Last Month / Last Quarter / Last Year. No divider between this row and the one above (a shared `.quick-grid`-style `border-bottom` meant for other single-row usages must be suppressed for these two rows specifically — the prototype hit this exact bug: `proto-quick-grid-3col` overrides `border-bottom: none` for this reason).
3. **Year tabs** — up to 3 most recent years present in the data, oldest→newest left to right, plus a **Clear** link button (clears all pending months) right-aligned in the same row. A tab shows a small dot when it contains any pending month even while a different tab is active.
4. A **3-column month grid** for the active year tab, showing whichever months actually have data that year (reuses `availableMonths` from Part 1).
5. **Start / End** custom-range inputs, each with a drill-down calendar (below).
6. Footer — **Clear all filters** (destructive: wipes pending, applied, and the custom range widget, then closes) and **Apply** (aggregates + commits + closes). No other Clear button in the footer — a duplicate lower-left "Clear" was tried and removed during prototyping since it was redundant with #3's Clear and confusing.

### Pending-state model (port directly from the prototype)
- Internal "pending" state is always a **flat set of individual calendar months** (`{type:'month', key:'YYYY-MM'}`) — never separate quarter/year entries — regardless of whether a month arrived via a grid click or a This/Last Quarter/Year pill.
- A This/Last preset pill's active/highlighted state is a **derived coverage check**, not a stored flag: active whenever every one of its constituent months is currently in the pending set (prototype: `isItemActive`/`monthsForItem`). This is what makes manually picking all 3 months of a quarter automatically highlight the matching Quarter pill, with zero extra bookkeeping — and is why quarter/year presets don't need their own pending entry type at all.
- Clicking a preset toggles all of its months at once: fills in whichever are missing if not yet fully active, or removes all of them if already fully active (prototype: `toggleItem`).
- Reopening the popover expands whatever's currently *applied* (which may already be aggregated into Year/Quarter chips) back into the flat pending month set for further editing (prototype: trigger click handler).

### Aggregation on Apply (new behavior vs. the original sketch)
When Apply is clicked, the flat pending month set is greedily re-compressed into the fewest chips, in priority order: (1) any year whose all 12 months are present → one Year chip; (2) any quarter (within remaining months) whose all 3 months are present → one Quarter chip; (3) whatever's left → one chip per month. Port `aggregateMonthKeys` from the prototype directly — it already outputs `{type, key, label}` entries in the exact shape needed to become `{type:'month', text, displayText}` chips per the data model above. This re-aggregates on **every** Apply, so incrementally adding months across multiple open/edit/apply cycles keeps consolidating (apply Jan+Feb as two chips, later reopen, add March, re-apply → collapses to one Quarter chip). The custom Start/End range is read and appended **after** aggregation, completely independent of it — never merged with or counted toward month/quarter/year coverage, per explicit requirement.

### Drill-down calendar widget (Start/End)
Each of Start/End is a text input — supports free-form typing, parsed via `YYYY-MM-DD` / `M/D/YYYY` regex with a `new Date(...)` fallback; unparseable text gets an `.invalid` visual state without blocking typing — plus a calendar icon that opens a three-view picker (port `createDrillCalendar` from the prototype):
- **Days view**: header shows Month + Year as separate clickable "drill" buttons (plus ‹ › to step one month at a time); a 7-column day grid below. Clicking a day commits it and closes the picker.
- **Months view** (drilled in from Days via the Month button): header shows Year as a clickable drill button (‹ › steps one year); a 3-column × 4-row grid of month abbreviations. Clicking a month drills back into Days view for that month.
- **Years view** (drilled in from Days or Months via the Year button): header shows a 12-year page (‹ › pages by 12 years); a 3×4 grid of years. Clicking a year drills back into Months view for that year.

**Cross-year range label**: show the year on both Start and End when they fall in different calendar years (e.g. "Dec 15, 2025 – Feb 10, 2026"); show it once, on the end, when both share a year (e.g. "Mar 1 – Apr 13, 2026"). Port `fmtRange` from the prototype.

### New code (`spending_report.js`)
1. Today/quarter helpers, `thisLastPresetDefs()`-equivalent, `monthsForItem`/`isItemActive`/`toggleItem`, and `aggregateMonthKeys` — ported near-verbatim from the prototype's shared logic, adapted to read from `availableMonths` (post-Part-1-fix) instead of the prototype's hardcoded fixture array, and to push results into `activeFilters` (existing chip array) instead of a local `applied` array. Year tabs cap at 3 most recent (no separate cap needed for months/quarters — the month grid is naturally bounded to ≤12 per visible year tab, and quarters only ever appear as computed This/Last labels or aggregation output, never a flat enumerated list).
2. **`txnFullDate(txn)` helper** (near `monthMatches`) reconstructing day precision the same way `analyzer.py:618-627`'s CSV export already does: `` `${txn.month}-${txn.date.slice(3,5)}` `` (mirrors the existing `.slice(3,5)` pattern already used in `getTransactions`'s sort comparator, js:388-389 — `txn.date` is always the zero-padded `MM/DD` from `strftime('%m/%d')`, so this is safe).
3. **`matchesFilter`**: add `case 'daterange'` using `txnFullDate(txn)` compared against the split `start..end`.
4. **`numFilteredMonths`**/`filteredMonthsForCharts` (js:1076-1091, 1196-1213): change their `f.type === 'month'` check to `filterCategory(f.type) === 'date'`, expanding `daterange` chips to whole-month buckets for chart/average purposes (an acceptable approximation already implicit everywhere else in the app, since there's no finer-than-month averaging concept — the actual displayed totals stay day-accurate because they come from `passesFilters`-filtered transactions, not the bucket).
5. **Hash round-trip**: `filterTypeChar` (js:1580-1582) and the two inline type-char maps in `filtersToHash`/`hashToFilters` (js:1676, 1687, fallback at 1693) need a `daterange` entry, or a restored custom-range chip silently mis-maps to `'category'`.
6. **`getDisplayText`** (js:1303-1306): currently assumes any `type === 'month'` filter text is a plain `YYYY-MM`; needs to detect plain month vs. `YYYY-MM..YYYY-MM` (year-boundary → year label, else → quarter label) vs. `daterange` (cross-year-aware range formatting per above) for correct label restoration after a hash-based page reload.
7. The drill-down calendar widget and its parse/format helpers (new, ported per above).

### Markup (`spending_report.html:42-47`)
Replace the single `<optgroup>` `<select>` with the popover structure described above. Add `data-testid` attributes throughout (the project's Playwright tests consistently use `get_by_test_id`) — mirror the prototype's `id`/class naming (`proto-quick-grid`, `proto-year-tab`, `proto-month-cell`, `proto-cal-*`) translated to the report's existing conventions.

### CSS (`spending_report.css`)
Port the prototype's popover/pill/grid/drill-calendar styles (inline `<style>` in `.vscode/prototype/datefilter.html`) onto the report's existing CSS custom properties (`--bg-card`, `--accent-blue`, etc.) — the prototype was deliberately built using the report's own theme variables, so this should be close to a direct port. Add `.filter-chip.daterange` alongside the existing `.filter-chip.month` rules so custom-range chips visually match the month/quarter/year "date family."

### No Python changes needed for this part.

---

## Part 3 — Transaction rows: show year only when the list spans multiple years

**Bug**: `formatDate` (js:1555-1566) never emits a year in either of its branches, and the only call site (`spending_report.js:302`, inside `MerchantSection`'s template, `<span class="txn-date">{{ formatDate(txn.date) }}</span>`) only passes `txn.date` (`MM/DD`, no year — the year lives separately in `txn.month`, `YYYY-MM`; confirmed in `analyzer.py:99-101` and `report.py:130-141`, no fully-qualified date field is serialized).

**Chosen behavior**: show the year only when the specific transaction list being rendered (per merchant/subcategory row, i.e. `getTransactions(item)` in `MerchantSection`, js:384-392) actually spans more than one calendar year — compact by default, year appears automatically when needed. Verified this doesn't break the one existing test that asserts exact date strings (`tests/test_report_html.py:172-187`, `test_transactions_sorted_by_date_descending`) — its Amazon fixture transactions are all within 2024, so the multi-year condition is false and the unmodified `["Mar 1", "Feb 1", "Jan 10", "Jan 5"]` assertion still holds.

**Implementation** (`spending_report.js`):
1. Update `formatDate(dateStr, monthStr, showYear)` to append `, YYYY` (from `monthStr.slice(0,4)`) when `showYear` is true.
2. Add a memoized helper on `MerchantSection`, e.g. `getTransactionYears(item)`, that computes `new Set(getTransactions(item).map(t => t.month?.slice(0,4)))`.size > 1` **once per item** rather than once per transaction row (avoids O(n²) re-derivation across a merchant's full row list). Cache the result in a component-level `WeakMap` keyed by the `item` object reference — `WeakMap` specifically (not `Map`) so stale entries are garbage-collected automatically as `item` objects are recreated on every `filteredCategoryView`/`filteredSectionView` recompute, avoiding unbounded memory growth.
3. Update the template call site (js:302) to `formatDate(txn.date, txn.month, getTransactionYears(item))`.

No Python changes — `txn.month` (which carries the year) is already serialized per-transaction today.

---

## Part 4 — Title default: stop hardcoding a year

- `src/tally/templates.py:6`: `title: "{year} Spending Analysis"` → `title: "Tally Spending Analysis"` (confirmed only `{year}` occurrence in the file).
- `src/tally/cli_utils.py`: since `STARTER_SETTINGS` no longer has a `{year}` placeholder, simplify `f.write(STARTER_SETTINGS.format(year=current_year))` → `f.write(STARTER_SETTINGS)`, and remove the now-unused `current_year = datetime.datetime.now().year` and `import datetime` (confirmed no other use in that function).
- **Not touched**: the deprecated `year:` config field and its `commands/run.py:57-62` fallback (`f"{year} Financial Report"`), `config_loader.py`'s deprecation warning, `commands/diag.py`'s year display — all independent of `STARTER_SETTINGS`, must keep working for existing users who still set `year:` explicitly (settings.yaml backward compatibility).
- `tests/test_report_html.py:143` (`"2024 Financial Report"`) goes through the legacy `year:` config path via its fixture's `year: 2024` setting — decoupled from `STARTER_SETTINGS`, remains valid unchanged.
- No existing test asserts the literal `STARTER_SETTINGS` title text, so this change needs no test updates.

---

## Verification

1. **Manual Playwright MCP check** (required before considering this done, per project convention — this is a browser-rendered feature):
   - Generate a report from a config spanning ≥2 calendar years with a `views_file` excluding at least one merchant/month; confirm that month now appears in the month grid (Part 1).
   - Open the date-filter popover; confirm This/Last pills, year tabs (≤3, most recent), and the active year's month grid all render correctly, with no divider between the This and Last rows.
   - Click individual months that together complete a quarter (e.g. Apr+May+Jun); confirm the matching Quarter pill lights up *before* Apply is clicked (coverage-based highlight).
   - Apply two individual months, reopen, add a third that completes a quarter, Apply again; confirm the chip strip collapses to one Quarter chip instead of three month chips (aggregation). Repeat with a full year's worth of months to confirm collapse to a Year chip.
   - Confirm the "Clear" button next to the year tabs only unselects months (grid + This/Last Month), and "Clear all filters" wipes everything and closes the popover.
   - Open the Start calendar, drill Month→Years and Years→Months→Days, paginate years by 12, pick a day; confirm it commits into the text input and closes. Repeat for End.
   - Pick a Start/End range spanning two calendar years; confirm the chip shows the year on both ends (not just the end).
   - Add a Year chip and a Quarter chip from a *different* year; confirm totals reflect the union of both periods, not zero/empty (the core additive semantic).
   - Reload the page with Quarter/Year/Custom chips active via URL hash; confirm labels restore correctly (the case most likely to regress if `getDisplayText`/type-char maps aren't updated).
   - Expand a merchant whose transactions span multiple years (e.g. the Delta Amex Payment example) and confirm rows now show the year; expand a single-year merchant and confirm rows stay compact (no year).
   - Confirm `tally init` produces `title: "Tally Spending Analysis"` with no year.

2. **Automated tests** (`tests/test_report_html.py`):
   - New regression test for Part 1 (views_file-excluded merchant's month still appears in the month grid).
   - New test class covering: This/Last pills and year-tabbed month grid render correctly; a quarter pill highlights once its 3 months are individually selected; applying months incrementally across two sessions collapses to a quarter/year chip on the second Apply; Year+Quarter selections from different periods union rather than cancel; the drill-down calendar commits a day-precision date and its chip filters accurately (reuse the existing multi-month Jan–Mar 2024 CSV fixture, `tests/test_report_html.py:62-76`); a cross-year custom range shows the year on both ends; a multi-year fixture asserting a merchant's transaction rows show `", YYYY"` while a single-year merchant's rows don't.
   - No `tests/test_analyzer.py` changes needed — no analyzer/report.py logic changes in this plan.

### Critical files
- `src/tally/spending_report.js`
- `src/tally/spending_report.html`
- `src/tally/spending_report.css`
- `src/tally/templates.py`
- `src/tally/cli_utils.py`
- `tests/test_report_html.py`
- `.vscode/prototype/datefilter.html` — reference implementation for Part 2, not modified
