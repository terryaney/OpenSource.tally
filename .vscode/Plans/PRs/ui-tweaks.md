> **Stacked on [`feature/report-json-determinism`](https://github.com/terryaney/OpenSource.tally/tree/feature/report-json-determinism) — PR #TBD.** Please merge that one first (and the one it sits on).
> **This layer only:** [`feature/report-json-determinism...feature/ui-tweaks`](https://github.com/terryaney/OpenSource.tally/compare/feature/report-json-determinism...feature/ui-tweaks)

This PR improves the generated HTML spending report across correctness, navigation, and daily usability. 

**There was one headline fix to data correctness**: report-side charts, Filtered View totals, and Section View percentages now classify each transaction by its own tags instead of using a merchant-wide tag union. 

**Mobile View Note**: Given this is a locally generated html file, I feel the vast majority of users will be desktop/large display.  There were several pre-existing mobile issues that I plan to address later.

**Screenshots**: generated from simulated data so can't guarantee the math is correct, purely for visual reference.

## At a Glance

- Corrected chart/KPI classification for mixed-tag merchants
- Corrected Section View percentage using per-transaction spending totals.
- Added Month / Quarter / Year / Custom date filtering with URL hash restoration.
- Renamed some charts, added a top 10 +other pattern, chart click for visibility instead of filtering
- Added a collapsible Transaction Details container with sticky headers, pinned totals, etc.
- Persisted report layout preferences in `localStorage`, with a reset control.
- Improved transaction row layout, column sizing, and rule provenance popups.
- Rule popups received several explainability and layout improvements.
- Config and Documentation Changes
  - Added `report_fields` configuration so captured fields like memo can appear in report details without per-rule passthroughs
    - Starter settings include `report_fields: [memo]`.
    - Removed phantom `+1 Memo` indicators.
    - `report_fields` can be configured globally or per source.
  - Starter title is now `Tally Spending Analysis`.
  - `config/settings.yaml.example` includes the new report field setting.

## Important Data Correctness Fixes

### Transaction tags now drive chart and KPI classification

Previously, the JavaScript report classified transactions using `merchant.tags`, which is the union of tags across all transactions for a merchant. That meant one `income`, `transfer`, `refund`, or `investment` transaction could cause unrelated transactions at the same merchant to be classified incorrectly. This PR changes the rendered report paths to classify with each transaction's own `txn.tags`, matching the Python-side KPI logic.

Affected report surfaces:
- Filtered View KPI tile
- Cash Flow Trend
- Spending by Category
- Spending by Category Trend

**This is the strongest correctness story in the PR. It shows that the report is not just visually improved; the numbers shown in the browser now match the underlying analysis.**

**Default report totals now reconcile with Python-computed KPIs.**  

<img width="1121" height="1233" alt="image" src="https://github.com/user-attachments/assets/bdb17498-7aa4-437c-8dce-e9d473971a0a" />

### Section View percentages now use the correct spending denominator

Section View percentages previously depended on whole-merchant include/exclude decisions derived from merchant tag unions. In mixed-tag datasets, that could drop ordinary spending from the denominator.  The rendered Section View denominator now comes from `filteredViewTotals.value.spending`, which is already calculated per transaction.

## Loading and Mount Polish

The generated report now shows a clean loading shell while Vue mounts. This hides uninitialized template content and gives the page a progress indicator during startup.

**Report shows a clean loading state while the app initializes.**

<img alt="image" src="https://github.com/user-attachments/assets/98497684-afab-4160-b93e-7162d10ec0d3" />

## Date Filtering

The date filter changed from a single month dropdown into an interactive popover with multiple date selection modes.

### Date filtering now supports months, quarters, years, and custom ranges in one popover.

<img alt="image" src="https://github.com/user-attachments/assets/b87f9eee-a2be-44f8-a596-86e95fb6291d" />

### Selected months aggregate into concise quarter/year chips, while custom ranges stay day-precise

<img alt="image" src="https://github.com/user-attachments/assets/11b901a3-2e04-49b8-9e29-8fde1590c6ea" />

The underlying month source was also corrected. Available months now come from `categoryView`, so a month is not lost just because its merchant was excluded from every configured view.

## Transaction Details

**Transaction Details are now contained, collapsible, and easier to scan.**  

<img alt="image" src="https://github.com/user-attachments/assets/20774752-d03a-4655-9e77-9164478bb55c" />

**Headers and totals stay readable while only transaction rows scroll.**  

<img alt="image" src="https://github.com/user-attachments/assets/66a47034-ed4b-4d9b-afd9-56a6a4a4acae" />

**Real memo fields appear in transaction details, while blank values no longer create phantom badges.**  

<img alt="image" src="https://github.com/user-attachments/assets/7c56f70b-c561-4484-9cb7-a2fe784597f8" />

**Rule provenance popups now show the actual rule path more clearly.**

<img alt="image" src="https://github.com/user-attachments/assets/9246e05c-3c2d-4e44-8cab-13a6a4cc4b5b" />

## Coming next (not in this PR)

Each is based on the one above it, so they want merging in this order:

1. [`feature/charts-reimagined`](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined) — PR #TBD — reimagined chart layout, KPI tiles, chart docs
2. [`feature/merchant-composite-keys`](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys) — PR #TBD — composite merchant identity so one merchant can carry multiple categorizations
3. [`feature/categorization`](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization) — PR #TBD — categorization review file, `review:` rule flag, `inventory.yaml`

Separately, [`feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair) — PR #TBD — is independent of this stack: `.github/workflows/` only.