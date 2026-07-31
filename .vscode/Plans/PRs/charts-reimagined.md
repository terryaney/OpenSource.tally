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

**Review Guide**

<img alt="image" src="https://github.com/user-attachments/assets/ff28116f-021a-4bf6-8fde-12bef973783e" />

Please use [charts.html](https://htmlpreview.github.io/?https://raw.githubusercontent.com/terryaney/OpenSource.tally/refs/heads/feature/charts-reimagined/docs/charts.html) as the primary feature walkthrough instead of reproducing all details in this PR body.  That page documents the chart/KPI behavior and intent implemented on this branch.