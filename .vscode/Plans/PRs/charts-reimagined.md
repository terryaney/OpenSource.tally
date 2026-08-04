Title: Reimagined charts and KPI experience [4/6]

> **[4/6]** Based on `feature/ui-tweaks` (#100).
> Merging this PR into `main` also lands #98, #99, and #100 if not already merged.
> **This layer only:** [`feature/ui-tweaks...feature/charts-reimagined`](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined)

Reimagines the KPI strip and chart experience in the HTML spending report.

<img alt="image" src="https://github.com/user-attachments/assets/ff28116f-021a-4bf6-8fde-12bef973783e" />

Full walkthrough: [charts.html](https://htmlpreview.github.io/?https://raw.githubusercontent.com/terryaney/OpenSource.tally/refs/heads/feature/charts-reimagined/docs/charts.html) — that page documents every chart/KPI behavior implemented on this branch.

## Changes

- KPI strip refresh with anchor-month correctness and sparkline/trend hardening
- Spending by Category is now a filter surface: tri-state legend chips, clicking a bar drills into that category + date range
- Chart-driven filters are transient "peek" state — not persisted to URL hash, dropped when the user adds a manual filter
- Cash Flow, Recurring vs Variable, Seasonality, Volatility chart refinements
- Top 10 Fixed ranked by monthly cost instead of alphabetically
- Empty bucket cleanup across all charts
- KPI compatibility hooks for existing test selectors
- New `docs/charts.html` documentation page

## Key implementation details

- `categoryExemptAggregations` / `passesFilters(txn, merchant, { skipCategory })` — category chart bars use an aggregation that ignores include-mode category filters, so selecting a category dims peers rather than removing them
- `toggleCategoryChip` / `applyCategorySelection` — `activeFilters` is the single source of truth; no separate hidden-set
- `applyCategoryChartFastVisibility` — chip toggles flip dataset visibility on the live Chart instance instead of rebuilding it

## Behavior changes

Added while addressing review feedback. These change what the charts *say*, not just how they look.

- **Fewer merchants are classified as monthly recurring.** Monthly inference tested only how many distinct months a merchant appeared in, against a threshold derived from how many distinct months the *whole data set* covered. That measures presence, not spread, and it made one merchant's classification depend on unrelated merchants' data. A merchant now also has to be dense over its own first-to-last span, so three charges spread across twenty-five months can no longer read as monthly.

  The failure this fixes: with a data set as sparse as the merchant — three Januaries in a January-only export — an annual premium cleared the old threshold and was booked as a *monthly* cost, twelve times its real value, in both Fixed vs Variable and the Fixed Spending Audit. On a dense data set the old check was already correct, so most reports will not move.

  It is a pure tightening: it can only remove false `monthly` classifications, never add one. `fixed`/`variable` rule tags still override the inference entirely.

- **Chart panel headings are buttons.** The collapse toggles were click handlers on `<h3>`, so they were unreachable by keyboard across all six panels. The heading stays a heading — document navigation is unchanged — with a chrome-stripped button inside it. Rendering is unchanged.

- **A disabled legend chip is genuinely disabled.** The "Other" chip carried a `disabled` class that removed its click listener but left it focusable and announced as an enabled control.

## Stack

| # | Branch | PR | Description | Diff |
|---|---|---|---|---|
| 1 | `feature/globbing-documentation` | #98 | Glob pattern docs and CLI tests | [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/globbing-documentation) |
| 2 | `feature/report-json-determinism` | #99 | Deterministic report JSON | [from PR98](https://github.com/terryaney/OpenSource.tally/compare/feature/globbing-documentation...feature/report-json-determinism) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/report-json-determinism) |
| 3 | `feature/ui-tweaks` | #100 | Date filter, transaction details, per-txn tags | [from PR99](https://github.com/terryaney/OpenSource.tally/compare/feature/report-json-determinism...feature/ui-tweaks) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ui-tweaks) |
| **4** | **`feature/charts-reimagined`** | **#101** | **Reimagined charts and KPIs** | **[from PR100](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/charts-reimagined)** |
| 5 | `feature/merchant-composite-keys` | #102 | Composite merchant keys | [from PR101](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/merchant-composite-keys) |
| 6 | `feature/categorization` | #103 | Categorization review file | [from PR102](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/categorization) |

Independent: [`feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair) — #97 — fixes fork PR builds.