> **Stacked on [`feature/ui-tweaks`](https://github.com/terryaney/OpenSource.tally/tree/feature/ui-tweaks) — PR #TBD.** Please merge that one first (and the two it sits on).
> **This layer only:** [`feature/ui-tweaks...feature/charts-reimagined`](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined)

This PR reimagines the KPI and chart experience in the HTML spending report while preserving the same core budgeting and analysis intent.

**Features / Changes**

1. KPI strip and chart interaction refresh
2. Category chart behavior updates, including empty bucket cleanup
3. Cash Flow, Recurring vs Variable, Seasonality, Volatility, and Recurring views refinements
4. Responsive and chart state behavior improvements
5. Documentation updates, including the new charts documentation page and sitemap image metadata
6. Test updates for chart/report behavior
7. KPI compatibility hooks restored for report_html test selectors used by existing tests
8. Post-hook KPI behavior hardening (anchor month and sparkline/trend correctness)
9. KPI detail-row math reconciliation so trend baseline and displayed 12-month average align
10. Spending by Category chart is now a filter surface: legend chips are tri-state (regular / selected / not-selected) instead of show/hide, and clicking a bar segment drills into that category plus that bucket's date range
11. Chart-driven filters are transient "peek" state — surfaced by a Peek Mode badge and a Clear Filters button, not persisted to the URL hash, and dropped as soon as the user adds a filter by hand
12. Top 10 Fixed is ranked by monthly cost rather than alphabetically

**Review Guide**

<img alt="image" src="https://github.com/user-attachments/assets/ff28116f-021a-4bf6-8fde-12bef973783e" />

Please use [charts.html](https://htmlpreview.github.io/?https://raw.githubusercontent.com/terryaney/OpenSource.tally/refs/heads/feature/charts-reimagined/docs/charts.html) as the primary feature walkthrough instead of reproducing all details in this PR body.  That page documents the chart/KPI behavior and intent implemented on this branch.

Three pieces of `spending_report.js` are worth reading closely, since the rest follows from them:

- `categoryExemptAggregations` / `passesFilters(txn, merchant, { skipCategory })` — the Spending by Category chart sources its bars from an aggregation that ignores *include*-mode category filters, so selecting a category dims its peers rather than deleting them from the canvas. Exclude-mode category filters still apply.
- `toggleCategoryChip` / `applyCategorySelection` — `activeFilters` is the single source of truth for chip state; there is no separate hidden-set anymore. All-on and all-off both collapse back to "no category filter".
- `applyCategoryChartFastVisibility` — a chip toggle flips dataset visibility on the live Chart instance instead of destroying and rebuilding it, to avoid reanimating on every click. It bails to a full rebuild if the dataset list no longer matches the chip list.

## Coming next (not in this PR)

Each is based on the one above it, so they want merging in this order:

1. [`feature/merchant-composite-keys`](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys) — PR #TBD — composite merchant identity so one merchant can carry multiple categorizations
2. [`feature/categorization`](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization) — PR #TBD — categorization review file, `review:` rule flag, `inventory.yaml`

Separately, [`feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair) — PR #TBD — is independent of this stack: `.github/workflows/` only.