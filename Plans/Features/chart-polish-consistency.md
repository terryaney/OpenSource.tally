# Chart Polish & Consistency — HTML Report

> **Status: implemented & verified** (rev 6). Changes live in `src/tally/spending_report.{js,css,html}` on branch `feature/ui-tweaks`, staged.
>
> **Two data-correctness bugs** were found and fixed here, both the same root cause (merchant tag *union* used where per-transaction tags were needed): **rev 5** (charts + Filtered View tile) and **rev 6** (Section View percentages).
>
> Final chart names: **Cash Flow Trend** (1), **Spending by Category** (2), **Spending by Category Trend** (3).

---

# ⚠️ REV 5 — DATA CORRECTNESS BUG (the most important change in this branch)

**Every number in the report's charts and in the Filtered View tile was wrong.** This is not a cosmetic fix; it changes reported dollar amounts. Read this section before reviewing anything else.

## The bug

`spending_report.js` classified each transaction using **`merchant.tags`** — which `analyzer.py:122` builds as the **union of every tag across all of that merchant's transactions** — instead of the transaction's own **`txn.tags`** (present in the payload at `report.py:140`, and used correctly by the tag *filter* at `js:1752`).

`categorizeAmount()` (`js:46`) is a strict if/else chain: **income → investment → transfer → sign**. So a single tagged transaction at a merchant poisons every other transaction there:

- One `income`-tagged paycheck at a merchant ⇒ that merchant's union contains `income` ⇒ **all** its other transactions (purchases, refunds, transfers) were counted as `income += Math.abs(amount)` — sign and all.
- Real example from this dataset: a merchant whose union is `[monthly-bill, refund, transfer]`. Most of its transactions carry only `monthly-bill` or no tags, yet every one was classified as a **transfer** and vanished from spending.

**Two call sites passed the wrong tags:**
| Site | What it feeds |
|---|---|
| `spending_report.js:1267` (`filteredViewTotals`) | the Filtered View KPI tile |
| `spending_report.js:1448` (`chartAggregations`) | **all three charts** (spending, income, credits, per-category, per-month) |

## The fix

Pass the transaction's own tags: `categorizeAmount(txn.amount, txn.tags || [])` at both sites. This matches what Python already does (`analyzer.py:77`, `report.py:292` both classify per-transaction).

## Proof

Both code paths walk the identical 3,269 transactions — nothing was ever being *excluded*, only *misclassified*. Re-running the classification per-transaction reproduces the Python KPI totals **to the dollar**:

| | Python KPI | JS before (merchant tags) | JS after (txn tags) |
|---|---|---|---|
| Income | $213,613 | $263,212 ❌ | **$213,613** ✅ |
| Spending | $253,781 | $243,654 ❌ | **$253,781** ✅ |
| Credits | $45,595 | $28,616 ❌ | **$45,595** ✅ |
| Transfers (net) | — | −$4,426 ❌ | **+$16,339** ✅ |

After the fix the Cash Flow and Filtered View tiles agree line-for-line on unfiltered data (both net **+$5,427**), and Chart 2 / Chart 3 totals both equal **$253,781**, i.e. the real spending total.

---

# ⚠️ REV 6 — SECOND DATA CORRECTNESS BUG (Section View percentages)

Rev 5 listed `grandTotal` as "still broken, deferred". Tracing what it actually feeds showed it *is* on a rendered path, so it was fixed here.

## The bug

`grossSpending` (`js:1240`) was `grandTotal + creditsTotal`, and **both inputs make whole-merchant decisions from the tag union** — `grandTotal` drops any merchant whose union `isExcludedFromSpending`, and `creditsTotal` sums `creditMerchants`, which does the same. So one `transfer`-tagged transaction discards *all* of that merchant's ordinary spending from the denominator.

Where it surfaces: the `(X%)` badge next to each section header (`js:294`), which only renders when `typeTotals` is absent. Category and Subcategory views pass `:type-totals` (computed correctly per-transaction in `report.py:284-305`) and are unaffected. **Section View passes none** (`spending_report.html:247-272`), so it always fell through to `grossSpending`. Live for anyone with `views_file` set — this config has three (`Monthly Bills`, `Food & Dining`, `College Spending`).

## The fix

```js
// Gross spending (before credits). categorizeAmount() already separates positive
// spend from refunds per transaction, so this is the sum of spending-tagged
// amounts - no need to net merchants out and add their credits back in.
const grossSpending = computed(() => filteredViewTotals.value.spending);
```

Moved below `filteredViewTotals` so it reads in order. `grandTotal` and `creditsTotal` are left in place (see below) but no longer feed any rendered number.

## Proof

| | Before (merchant union) | After (per-txn) | Python `spendingTotal` |
|---|---|---|---|
| `grossSpending` | $215,322 ❌ | **$253,781** ✅ | $253,781 |

The denominator was **15% low**. Merchants dropped wholesale by their union, and the ordinary spending each one hid:

| Merchant | Union tags | Spending hidden |
|---|---|---|
| Amazon | `[monthly-bill, refund, transfer]` | $6,910 |
| Minnesota Department of Revenue | `[income]` | $2,448 |
| Best Buy | `[transfer]` | $444 |
| Salomon | `[transfer]` | $178 |
| Montana State University | `[transfer]` | $148 |

Section View percentages move accordingly: Monthly Bills **28.0% → 23.8%**, Food & Dining **20.0%**. Category/Subcategory views unchanged (Finance 4.6%, Shopping 11.4%, Food 20.1%).

## ⚠️ Still broken — NOT fixed here

All three remaining sites make **whole-merchant** include/exclude decisions from the tag union. **None of them feeds a rendered number any more**, which is why they were left alone rather than fixed blind:

- `creditMerchants` — skips a merchant entirely if its union `isExcludedFromSpending`. **Not rendered:** the "Credits Applied" section that consumed it was dropped from `spending_report.html` in **`ad9477e`** ("Add subcategory grouping toggle to HTML report", 2026-01-03) — a commit whose stated purpose was unrelated. `docs/reference.html:600` still documents the feature (`refund` tag → *"Shown in 'Credits Applied' section, nets against spending"*), so this is an upstream regression worth its own issue. A comment above the computed now records the bug and the two fix paths:
  1. **Net-negative merchants** — net each merchant's per-txn spending against its per-txn credits, list when net < 0. Preserves today's meaning, short list; but "Total Credits" still won't equal the Credits KPI, since a refund absorbed inside a net-positive merchant stays invisible. Only works when refunds arrive under their own merchant name (e.g. "Amazon Refund").
  2. **Any merchant with credits** — list when `sum(txn credits) > 0` and show that sum. Yields the identity `sum(creditAmount) === filteredViewTotals.credits` under every filter, so the section reconciles with the Credits KPI and Python's `credits_total`. Costs a longer list: a merchant can appear both as spending and as a credit (Amazon: $6,910 spent, refunds back).
- `grandTotal` — same union flaw. No longer feeds anything: `grossSpending` was repointed (rev 6), and although it is still passed as `:grand-total` to `MerchantSection`, the component never references the prop.
- income / transfer transaction counts (`incomeCount`, `transfersCount`) — count *all* of a merchant's txns if the union contains the tag. Computed and returned from `setup()`, but referenced nowhere in `spending_report.html`. Latent: wrong the moment anyone wires them to a KPI.

**Also worth a look (found while tracing, unrelated):** `build_category_view` (`report.py:243`) keys merchants by `make_merchant_id(name)` (quotes stripped, spaces → underscores). Two distinct merchant names that collapse to the same id silently overwrite each other, dropping one merchant's transactions from `categoryView`. The auto-detected "Unknown" path strips punctuation before naming so it's safe; hand-maintained `merchant_categories.csv` names are not (`Trader Joe's` vs `Trader Joes`).

---

## Context

The report has three charts: **Monthly Trend** (bar, spending/income/investment), **By Category** (doughnut), and **Category Trends by Month** (stacked bar). Bugs and inconsistencies identified:

1. **Blank-charts bug:** Chart 3 stacked synthetic "Income" and "Investment" datasets built from *tags* (`categorizeAmount()`), not real categories. Clicking one ran `addFilter('Income', 'category')`, which matched no merchant's actual category → `filteredCategoryView` emptied → all three charts blanked (they all feed off `chartAggregations`). Chart 3's legend click had the same landmine.
2. **Resize cut-off bug:** `.charts-grid` is a CSS grid (`2fr 1fr`) whose items lacked `min-width: 0`. Grid items default to `min-width: auto`, so once Chart.js rendered a canvas at some width the track could never shrink — after narrowing and re-widening the window, Chart 2 clipped and never recovered.
3. **Inconsistent interactions:** Chart 3's legend added a filter (Charts 1/2 legends were default visibility toggles); Charts 2/3 data clicks added filter chips with little benefit.
4. **(rev 2) Inconsistent category depth:** Chart 2 rendered **every** category (16 slices → overflowing, clipped legend); Chart 3 capped at **top 8** and silently dropped the rest, so its bars under-reported real monthly spending.
5. **(rev 3) Empty month columns:** `availableMonths` collects every month with *any* transaction in `categoryView`, including non-spending ones. A lone April 2026 transfer therefore drew an empty column on both Chart 1 and Chart 3.
6. **(rev 3) Chart 1 click mapped to the wrong month (pre-existing bug):** Chart 1's bar `onClick` looked up `availableMonths.value[idx]`, but its bars are labeled from `filteredMonthsForCharts`. Whenever a date filter was active the two lists diverged, so clicking a bar filed a month filter for the **wrong month**.
7. **(rev 4) Chart 1 didn't match the cash-flow definition:** `calculateCashFlow` (`js:84`) is `income - spending + credits`, yet Chart 1 plotted **Investment** (not part of cash flow) and **discarded Credits** — `chartAggregations` computed `c.credits` / `c.transferIn` / `c.transferOut` per transaction and threw them away.
8. **(rev 4) Phantom Investment legend entry:** Chart 1's three datasets were declared *statically* at init and only had `.data` reassigned by index, so an all-zero Investment series kept a permanent legend entry. This contradicted the KPI convention, where empty rows are zero-guarded (Credits `v-if="dataCreditsTotal > 0"`, Investment is its own card with `v-if="investmentTotal > 0"`).

## Decisions (confirmed with user)

- Remove **both** Income and Investment synthetic datasets from Chart 3.
- Resize fix is **pure CSS** (`min-width: 0`); no JS resize handling needed.
- Chart 3 legend becomes a plain visibility toggle (like Charts 1/2) — no filter.
- Remove data-click filter handlers from Charts 2 and 3. **Keep** Chart 1's bar click (adds month filter — useful, bug-free).
- Rename titles: Chart 2 → **"Spending by Category"**, Chart 3 → **"Monthly Spending by Category"**.
- **(rev 2)** Both category charts show **top 10 categories + an "Other" rollup**, so totals stay honest and legends stay readable.
- **(rev 3)** Each chart drops months where **all of its own series are zero** — Chart 1 drops months with no spending/income/credits/investment, Chart 3 drops months with no spending. The two axes may therefore differ (an income-only month appears in Chart 1 but not Chart 3), which is the honest behavior.
- **(rev 4)** Chart 1 plots **Spending, Income, Credits, Investment**. Credits added (it *is* part of cash flow); Investment kept (it has its own KPI card); **Transfers excluded** — they aren't cash flow, and adding them would resurrect the April 2026 transfer-only column that rev 3 removed.
- **(rev 4)** Chart 1 series render **only when they have data**, so empty ones vanish from chart and legend (matches the zero-guarded KPI rows).
- **(rev 4)** The Cash Flow **KPI tile stays unfiltered** (static whole-dataset totals from Python). The Filtered View tile is the filter-reactive one. No change.
- **(rev 4)** Final titles: Chart 1 → **"Cash Flow Trend"**, Chart 3 → **"Spending by Category Trend"** (supersedes rev 1's "Monthly Spending by Category"). Chart 2 stays "Spending by Category".

## Implemented changes

### `src/tally/spending_report.js`

1. **Chart 3 datasets:** deleted the Income and Investment dataset construction in `updateCharts`. Chart 3 is now purely spending-by-category.
2. **Chart 3 legend:** removed the custom `onClick` (which called `addFilter(category, 'category')`), so Chart.js's default visibility toggle applies — matching Charts 1 and 2.
3. **Chart 3 data click:** removed the `onClick` handler (dropped both the month and category `addFilter` calls).
4. **Chart 2 data click:** removed the `onClick` handler (dropped `addFilter(label, 'category')`).
5. **(rev 2) Top 10 + "Other":** new module constants next to `CATEGORY_COLORS`:
   ```js
   const TOP_CATEGORY_COUNT = 10;
   const OTHER_CATEGORY_LABEL = 'Other';
   const OTHER_CATEGORY_COLOR = '#6b7280';
   ```
   In `updateCharts`, a **single shared ranking** (categories sorted desc by `agg.byCategory` spend) now drives both category charts, so their series order and colors are guaranteed identical:
   - Chart 2 (pie): top 10 slices + an "Other" slice summing the remainder. Slice colors are assigned per-update (the init-time static `backgroundColor: CATEGORY_COLORS` became `[]`), so "Other" reliably gets the gray.
   - Chart 3 (stacked): the same top 10 as datasets + an "Other" dataset whose per-month value sums the remaining categories. Chart 3 previously ranked off `byCategoryByMonth` totals; it now uses the shared ranking.
   - The "Other" series is **omitted entirely** when 10 or fewer categories are present (e.g. under a category filter), so no empty gray band appears.
   - `chartAggregations.byCategory` is spending-only (income has no meaningful categories), which is why "Spending by Category" is the accurate title.
6. **(rev 3) Empty months dropped:** `updateCharts` now filters `monthsToShow` per chart — Chart 1 keeps months with any spending/income/investment, Chart 3 keeps months with spending. Both the labels and the `suggestedMax` calculation use the filtered list.
7. **(rev 3) Chart 1 click bug fixed:** added a `monthlyChartMonths` array (declared next to the chart instances) holding the months actually plotted. `updateCharts` assigns it and the bar `onClick` indexes into it instead of `availableMonths`, so the chip always matches the clicked bar. This was already wrong under date filters; dropping empty months would have made it wrong in the default view too.
8. **(rev 4) Credits aggregated per month:** `chartAggregations` now builds `creditsByMonth` (summing `categorizeAmount(...).credits`) alongside spending/income/investment and returns it. No new plumbing from Python was needed — `categorizeAmount` already produced the value; the loop was discarding it. Transfers are still discarded by design.
9. **(rev 4) Chart 1 datasets built dynamically:** init now declares `datasets: []`, and `updateCharts` builds them from a `CASH_FLOW_SERIES` spec — Spending `#4facfe`, Income `#00c9a7`, Credits `#ffa94d` (new), Investment `#7c3aed` — pushing a series only if it has a non-zero month, and computing `suggestedMax` from the included series only. Same conditional-push pattern as Chart 3's "Other" dataset. The old index-based `datasets[0..2].data` assignment is gone.
10. **(rev 4) Empty-month test includes credits:** `monthlyChartMonths` keeps a month if *any* `CASH_FLOW_SERIES` entry is non-zero there. Transfers stay out of the test, so the April 2026 transfer-only month remains hidden.
11. **Untouched:** `addFilter` itself, `categoryColorMap` (table header colors, still ranks all categories), and every other filter path (merchant/subcategory/tag clicks, search autocomplete, date popover, URL hash).

### `src/tally/spending_report.css`

7. `min-width: 0` added to `.chart-container`, with a comment explaining that the default `min-width: auto` locks grid tracks open and clips canvases on re-widen.

### `src/tally/spending_report.html`

8. Titles: "Monthly Trend" → **"Cash Flow Trend"**; "By Category" → **"Spending by Category"**; "Category Trends by Month" → **"Spending by Category Trend"**.

### Tests

9. No test changes needed — no test referenced the old titles or chart handlers. `uv run pytest tests/` → **898 passed**.

## Verification performed (Playwright, against `C:\BTR\TallySpending\tally\config`)

Dev workflow used for iteration:
```bash
uv run tally up --no-embedded-html -o <scratchpad>/dev-report/spending.html C:\BTR\TallySpending\tally\config
# file:// is blocked in Playwright — serve it:
uv run python -m http.server 8642 --directory <scratchpad>/dev-report
```

Results:
- **Blank-bug gone:** Chart 3 shows only spending categories; clicking a segment or legend item adds **0** filter chips and never blanks the charts. Legend click hides/shows the series (verified both directions, including the "Other" series).
- **Chart 2:** slice click is inert; legend still hides slices.
- **Chart 1 unchanged:** bar click still adds a month chip ("Apr 2025"), all three charts render correctly under it.
- **Top 10 + Other:** both charts render the identical 11-series list (`Food … Personal Care, Other`) with **identical colors**, and both totals equal **$243,654** — matching the report's Spending figure, i.e. nothing is silently dropped anymore. Pie legend fits in one column, no clipping.
- **Filtered state:** `#+c:Food` reduces both charts to a single "Food" series with no "Other" bucket; clearing the hash restores all 11.
- **Resize:** 1600 → 800 → 1600 px, plus the collapse-resize-expand path — canvases track their containers exactly, zero clipping on all three charts.
- **(rev 3) Empty months:** Apr 2026 (a lone transfer, no spending/income/investment) no longer appears on either chart. Both axes run Jan 2025 → Mar 2026 with **0 empty columns** on each.
- **(rev 3) Chart 1 click bug:** with `#+d:2026-02` applied, Chart 1 shows only "Feb 2026" and clicking that bar now produces a **Feb 2026** chip. Before the fix it would have filed Jan 2025 (index 0 of `availableMonths`).
- **(rev 4) Cash Flow Trend:** titles read "Cash Flow Trend" / "Spending by Category" / "Spending by Category Trend". Chart 1's legend shows **Spending, Income, Credits** and **no Investment entry** (this dataset has zero investment months). Credits bars render in all 15 months, totalling **$28,616** — matching the Credits line in the Filtered View tile. Apr 2026 stays absent; the Feb-2026 click regression still passes.

- **(rev 5) Tag-classification fix:** the JS now reconciles with Python exactly — Filtered View reads Income +$213,613 / Spending $253,781 / Credits +$45,595 / Transfers +$16,339, and both KPI tiles show net **+$5,427** on unfiltered data. Chart 1's Spending/Income/Credits series and Chart 2/3's totals all sum to the Python figures. Empty months, click-index, inert chart clicks and resize all still pass. `uv run pytest tests/` → **898 passed**.

- **(rev 6) `grossSpending` fix:** Section View percentages now use the per-transaction spending total. `grossSpending` reads **$253,781**, matching Python's `spendingTotal` exactly (was $215,322). Monthly Bills 23.8%, Food & Dining 20.0%; Category/Subcategory views unchanged. `uv run pytest tests/` → **898 passed**.

## Known follow-ups (not done)

- **Restore the "Credits Applied" section** — dropped from the template in `ad9477e` but still promised by `docs/reference.html:600`. Fix `creditMerchants`' tag-union bug as part of that (two paths documented above and in a comment on the computed).
- `grandTotal`, `incomeCount`, `transfersCount` still classify by merchant tag union. None is rendered today; fix before wiring any of them to UI.
- `make_merchant_id` collisions in `report.py:243` (silent merchant overwrite).
- (The doughnut legend overflow noted in rev 1 was resolved by the top-10 + "Other" cap.)
