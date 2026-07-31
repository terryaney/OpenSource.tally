# Charts Reimagined — Implementation Plan (Handoff)

Reimagine the HTML spending report's KPI cards and charts. Every design decision is settled
and demonstrated in an interactive prototype; this document is the spec for implementing it
in the tally codebase. Written 2026-07-14.

## References (read these first)

- **Prototype (the spec):** `tally-notes/.vscode/Plans/Prototypes/chartsReimagined.html` —
  self-contained, open in a browser. The North Star section is the target UI; the Decision
  Log section at its bottom records every fork and why. All chart configs, the external
  tooltip handler, the year-sub-label plugin, the doughnut legend, and the pager are working
  reference implementations in plain JS + Chart.js v4 — port them, don't reinvent.
- **PR evidence screenshots:** `tally-notes/.vscode/Plans/Prototypes/screenshots/`
  (`round2-*` = final design; `round1-*` = rejected alternatives; `round2-compare-clean.png`
  shows compare-mode dividers).
- **Screenshot data manipulation playbook:** `tally-notes/.vscode/Plans/Features/Charts-Reimagined.ScreenshotDataManipulation.md`
  (temporary smoke-only KPI/label patch patterns, validation checks, and full cleanup steps).
- **Alternative concepts (not in scope):** `Prototypes/chartsReimagined-gallery.html`.

## Branch strategy

- Work on `feature/charts-reimagined`, which is **based on `feature/ui-tweaks`** (that branch
  rewrites much of the chart code and ships separately). To pick up ui-tweaks updates:
  `git checkout feature/charts-reimagined && git merge feature/ui-tweaks`. Add follow-up
  commits on ui-tweaks rather than amending, or the same conflict resolves twice.
- After ui-tweaks merges to main: `git merge main`; the PR diff then shows only this work.
- This feature is Terry's vision pitch to David — keep the diff clean and self-contained.

## Design spec (all decided — do not relitigate)

1. **KPI strip** replaces the locked Cash Flow card + Filtered View card pair: four smaller
   filter-reactive tiles spanning the top, all driven by the current filter state:
   - **Income** (green): headline = income + credits; detail lines Income and Credits, each
     with a muted `· $X/mo` sub-value.
   - **Spending** (red): detail lines Avg/month and Fixed/month.
   - **Cash Flow** (green if ≥ 0, red if negative — border and value): detail line Avg/month.
   - **Details** (neutral): headline = transaction count; detail lines Transfers $ and
     Investments $.
   - **Sparklines + trend statements** on Income/Spending/Cash Flow (not Details): full-width
     inline SVG polyline of the monthly series, line GRAY, only the endpoint dot colored
     green/red by trend direction (see `sparklineSVG` — no Chart.js). Directly below the
     spark: `↑ +N.N% this month vs prior N mo` — latest month vs the AVERAGE of the prior N
     months, N chosen by data span (>12 → 12, >6 → 6, >3 → 3, >1 → "vs last month", else
     hidden). Arrow + % colored by good/bad (Income/Cash Flow up = green; Spending up = red);
     the rest of the sentence stays gray.
2. **Chart order:** Spending by Category Trend (full width) → Cash Flow Trend (full width).
   Legends on the **bottom**. The standalone pie chart section is **removed**.
3. **Grouping buttons** on both trend charts: Years / Quarters / Months (default) / None.
   Years and Quarters only render when the filtered data spans more than one of that unit.
   - **None on the category chart = part-to-whole strip + ranked bars** (SUPERSEDES the
     earlier doughnut decision, after the gallery review). Pure HTML/CSS, no canvas: a flex
     strip (one segment per top-10+Other, width ∝ share, 2px gaps) above ranked rows —
     swatch, name, proportional bar track, `$X`, `N%`. Hovering a strip segment highlights
     its row and vice versa. Destroy the canvas chart when entering None; restore on leave
     (see `renderShareView` + the wrapper/hidden swap in the prototype).
   - **None on the cash flow chart** = one column per series (Spending, Income, Credits,
     Investment), legend hidden. None on Fixed vs Variable = two total columns.
   - **Chip legends everywhere:** every chart legend is a row of pill CHIP BUTTONS below the
     chart (Chart.js legends disabled; see `chipLegend`). Normal charts: chip toggles the
     series/category — colored dot + colored border when visible, gray when hidden; hidden
     state lives in a per-chart Set so it survives grouping/compare re-renders (this replaces
     the old `compareLegendFilter`/`legendToggleAcrossYears` legend machinery). Chips
     flex-wrap for mobile.
   - **"Unstack" toggle** (category chart, grouped modes only; hidden under None): checkbox
     beside Compare years, mutually exclusive with it (each disables — not hides — the
     other). Checked → line chart, SINGLE-SELECT EMPHASIS via the chips: exactly one category
     selected (defaults to the top-ranked one), drawn 3px in its ORIGINAL color on top;
     every other line is thin gray context. Clicking a chip moves the emphasis.
     Index-mode interaction + a vertical CROSSHAIR at the hovered bucket (`crosshairPlugin`);
     the sticky tooltip lists all series with the emphasized one bolded (`ttBoldLabel`).
   - **None-view emphasis:** hovering a strip segment or ranked row grays out every other
     segment/row/bar (same emphasis language as Unstacked).
4. **Tooltips (grouped/stacked modes)** — one shared external HTML tooltip div appended to
   `document.body` (backdrop-filter on cards creates stacking contexts that clip children):
   - Lists **all** visible categories/series at the hovered index + a **Total** row
     (**Net** for cash flow: income + credits − spending − investment), all currency-formatted.
   - **Bolds** the hovered row.
   - **Column-anchored (sticky):** position is set only when the anchor key
     (`canvasId:dataIndex:stack`) changes; moving across segments within a column only moves
     the bold. Hide + reset key when tooltip opacity hits 0.
5. **Compare years** checkbox: off by default; visible only for Months/Quarters grouping when
   span > 1 year; capped at the **4 most recent years**.
   - Pivot layout: x-axis = period-of-year (Q1–Q4 or Jan–Dec); each year is its own stack
     (`stack: 'y' + year`; **every dataset needs an explicit stack**). Years **alternate**
     full color and a single 55% fade (newest = full, then alternate backwards) — adjacent
     bars always differ, nothing goes ghostly at 3–4 years. No descending ramp.
   - Legend deduped to the newest year's datasets (always full color) — the legend `filter`
     signature is `(legendItem, chartData)`, **not** an object with `.chart` (round-1 bug).
     Legend `onClick` must toggle the category across **all** year stacks (see
     `legendToggleAcrossYears` in the prototype), not just the newest dataset.
   - **Two-row axis:** period tick on row 1; per-year sub-labels ("2025", "2026") under each
     bar/cluster on row 2, drawn by an `afterDraw` plugin into space freed by
     `scales.x.afterFit: s => s.height += 14`. Skip a year's label at indices where it has no
     data. For cash flow, center the label under the year's series group (average bar x).
   - **Cluster dividers:** `scales.x.grid = { offset: true }` in compare mode only (draws grid
     lines *between* categories, same style as y-grid).
   - **No year caption** — the axis sub-label row carries year identity. The only caption
     text is "showing last 4 years", rendered solely when data exceeds the cap.
6. **Fixed vs Variable Spending** (full width, after Cash Flow Trend): stacked bars,
   Fixed `#4facfe` + Variable `#ffa94d`, legend bottom, external tooltip. Footnote lists the
   merchants currently classified fixed (doubles as a classification audit).
   **Full control parity with the trends:** grouping pills (Years/Quarters/Months/None) +
   Compare years (one stack per year, alternating fade, dividers, year sub-labels — reuses
   the same helpers; see the prototype's `renderFixedVariable`). None = two total columns.
6b. **Spending Seasonality heatmap** (full width, after Fixed vs Variable): pure HTML/CSS
   grid, one row per top-10 category (no "Other" row — a grab-bag has no rhythm), one column
   per month. Cell = single-hue red ramp, 6 alpha steps 0.06→0.95, normalized to that ROW's
   own min–max (green→red was rejected: worst CVD pair, and a low month isn't "good").
   Hover tooltip: amount + row min–max (non-sticky path of the shared tooltip div). Pages at
   24 columns with the same Prev/Next pager (`renderPagedHeatmap` in the prototype).
7. **Expense Volatility** (split row, left): horizontal **native floating bars** `[min, max]`
   of monthly spend per category (no boxplot plugin), average marker = overlay scatter dataset
   with `pointStyle: 'line'`, `rotation: 90`, y = the exact category label string. Top 8 by
   range (max − min); header chip "Top 8 · by monthly swing"; no "Other" bar; tooltip prints
   min/avg/max.
8. **Subscription & Recurring Audit** (split row, right): table — Merchant, Category, Cadence
   badge, Monthly, Annualized with a CSS bar track behind the value (width ∝ max annualized).
   Top 10 by annualized + overflow row `+ N more · $X/yr`; total row covers **all** recurring
   merchants. Header chip "Top 10 · by annualized". Split row stacks to full width < 1000px.
9. **Density guard:** charts display at most **24 buckets** per page. When the grouping
   produces more, render pager links `‹ Prev 24 months` / `Next 24 months ›` (noun follows
   grouping) + a range caption ("May '24 – Apr '26 · showing 24 of 48 months"). Default page =
   most recent. No scrollbars. Reset to the newest page when filters change the month set.
10. The existing search box / filter chips / date filter / sections below charts are
    **untouched** — KPIs and charts consume the already-computed filtered aggregations.

## Rev2 changes (2026-07-19)

The following are explicit post-prototype product decisions and override earlier wording where
they conflict:

1. **Category toggle naming:** rename the category chart checkbox label from **Unstack** to
  **Focused**. Behavior stays the same (single-series emphasis lines; mutually exclusive with
  Compare years).
2. **Chart-level collapse controls + persistence:** every chart card gets a chevron collapse
  control. Collapsed cards hide same-row controls. Persist chart UI state in local settings:
  - expanded/collapsed state per chart card
  - grouping selection (when applicable)
  - compare-years toggles
  - category focused state
  If no saved settings exist, all chart cards default to expanded.
3. **Split-row collapse layout rule:** Expense Volatility and Subscription & Recurring Audit are
  side-by-side only when both are expanded. If one is collapsed, the pair switches to single
  column; collapsed card header renders first, expanded card fills full width below it.
4. **Chart order + title updates:**
  - Spending by Category
  - Spending Seasonality
  - Expense Volatility + Subscription & Recurring Audit
  - Cash Flow
  - Fixed vs Variable Spending
5. **Fixed/Variable monthly model change:** Fixed is no longer a flat repeated baseline.
  Compute per-month fixed spend by summing spending-tagged transactions for merchants currently
  classified recurring in that filtered month. Variable remains `spending - fixed` per month.
6. **Recurring audit row layout:** for overflow/summary rows, span the left label across the
  first three columns to prevent wrapping of the recurring-merchants label.
7. **Chevron placement consistency:** chart-level collapse toggles use the same visual language
  as Transaction Details and section headers: a small chevron to the left of the chart title,
  not a separate right-side icon button.
8. **Compact chart rhythm:** reduce inter-chart vertical spacing and reduce chart header/control
  vertical padding by roughly 50-60% from the prior pass. Keep chart card top and bottom padding
  symmetric so title/controls and footer spacing feel balanced.
9. **Header row alignment:** ensure Seasonality and Expense Volatility header rows (title,
  descriptive chip, chevron) are vertically centered.
10. **Volatility bar thickness:** set Expense Volatility floating-bar height to match
  Seasonality heatmap cell height.
11. **Volatility category count:** use top 10 categories (not top 8) for Expense Volatility.
12. **Volatility footnote copy:** restore a second line: "Wide bars are the
    budget-breakers that need a sinking fund."
13. **Seasonality footer formatting:** use proper-case "Low"/"High" and move
    "Each row relative to its own range." to a new line.
14. **Focused animation behavior:** allow line-draw animation on first focused-view render,
    then disable animation for focused-category switches so category emphasis updates are instant.
15. **Category tooltip ordering:** Spending by Category tooltip rows are sorted highest-to-lowest
  by value at the hovered bucket, and zero-value rows are omitted.
16. **Cash/Fixed tooltip ordering:** Cash Flow and Fixed vs Variable tooltips keep top-to-bottom
  row order consistent with legend left-to-right order.
17. **Multiline footer markup/spacing:** multiline chart footers use `div` line blocks (not `p`),
  with minimal vertical gap between lines.
18. **Volatility row spacing:** restore visible vertical spacing between volatility rows while
  preserving the intended bar thickness target.

## Python changes (additive only — no rule-engine behavior change)

The client needs data it doesn't get today. Facts from exploration (2026-07-14):
`analyzer.py:133-169` computes per-merchant `months_active`, `avg_when_active`, `cv`,
`is_consistent`; `report.py` exports `calcType`/`monthsActive`/`isConsistent` (lines ~172-174,
currently unused by the JS) but **not** `cv`. There is **no stored Monthly/Annual cadence
label** — derive one:

1. In `analyzer.py`, compute per-merchant `recurrence`:
   - **Tag override first:** a `fixed` tag forces fixed; a `variable` tag forces variable
     (tags already flow rule → txn → merchant union; see `merchant_engine.py` `_resolve_tags`).
   - Else infer **monthly-recurring**: `months_active >= max(3, ceil(num_months * 0.5))` and
     `cv < 0.3` (mirrors the sample view expressions in `templates.py:216-226`).
   - Else infer **annual-recurring**: 2+ transactions ~12 months apart with similar amounts —
     if too fuzzy for v1, ship monthly-only + tag override and note annual as a follow-up.
2. In `report.py` (`build_section_merchants` area, lines ~166-181), export per merchant:
   `cv`, `recurrence: 'monthly' | 'annual' | null`, `recurringMonthlyCost` (avg_when_active
   for monthly; amount/12 for annual). The audit table and fixed/variable chart derive
   entirely from these + transactions already exported.
3. Document the `fixed` / `variable` tags in `config/settings.yaml.example` and update the
   AGENTS.md template in `cli.py` (CLAUDE.md requirement for user-facing features).
4. **Do not touch** `merchant_engine.py` / `merchant_utils.py` matching behavior. Run
   `tests/test_rule_snapshots.py` anyway.

## JS/HTML/CSS changes (`src/tally/spending_report.{html,js,css}`)

- **HTML:** replace the summary-grid card trio (lines ~120-186) with the 4 KPI tiles (keep or
  update `data-testid`s and any tests using them); reorder chart containers, delete the pie
  container, add grouping-pill rows + compare checkbox + caption/pager divs, add the Fixed vs
  Variable container and the Volatility/Audit split row, add `<div id="ext-tooltip">` before
  scripts.
- **JS:** the reactive spine stays — `chartAggregations` (line ~1457) →
  `watch(chartAggregations, updateCharts)` (~2409). Changes:
  - Grouping/pager/compare state per chart (Vue refs); bucket aggregation helpers
    (month/quarter/year keys off `agg` month data — port `buckets()`/`sumFor()`/
    `comparePeriods()`/`pagedBuckets()` from the prototype).
  - **Destroy + recreate** charts on grouping/compare/page change (structural transitions);
    plain `update()` remains fine for filter-driven data refreshes within a mode.
  - Port from the prototype verbatim where possible: `extTooltipHandler` (sticky anchor),
    `yearSubLabelsPlugin`, the compare dataset builders, `sparklineSVG`, `renderShareView`,
    `renderCatLines` (Unstack), `renderHeatmap`/`renderPagedHeatmap`, the volatility and
    audit renderers, and the pager.
  - Colors: keep `CATEGORY_COLORS` / top-10+Other ranking (ui-tweaks) — assign hues from the
    overall ranking once so regrouping/filtering never repaints a surviving category.
  - Stacked segments get `borderColor` = page surface + `borderWidth: 1` (segment gaps).
  - On theme toggle, re-read `Chart.defaults.color` / `borderColor` from CSS vars and
    re-render all charts.
- **CSS:** port `.kpi-*`, `.head-chip`, `.chart-pager`, `.audit-table`/`.bar-track`,
  `#ext-tooltip` styles from the prototype (they already use the report's CSS variables).

## Suggested phases (each independently verifiable)

1. Python export + tests (`recurrence`, `cv`, tag override) — verify via `--format json`.
2. KPI strip (uses existing `filteredViewTotals`) — no chart work yet.
3. Trend charts: reorder, legends bottom, grouping pills, None modes, external sticky tooltip.
4. Compare years (pivot + two-row axis + dividers + caption).
5. New charts: Fixed vs Variable, Volatility, Audit (needs phase 1).
6. Density paging + filter-change page reset.

## Known gotchas (paid for in the prototype — don't rediscover)

- Legend `filter` receives `(legendItem, chartData)`; `legendItem.chart` does not exist.
- Compare mode: every dataset must carry an explicit `stack` or Chart.js merges them.
- Scatter average markers on a category axis: the point's `y` must exactly equal the label.
- External tooltip must live on `document.body` (backdrop-filter stacking contexts clip it).
- Set `Chart.defaults.color` **before** first render and re-read on theme toggle, or light
  mode gets grey-on-white ticks.
- Chart.js CDN is unpinned (currently v4) in `spending_report.html:352-353` — the APIs used
  (floating bars, external tooltip signature, `interaction`) are v4-stable.
- Palette note: the report palette's worst CVD pair is `#3b82f6`↔`#a855f7` (category ranks
  9/10); acceptable with the secondary encodings present (legend, gaps, labeled tooltips).

## Testing & verification (CLAUDE.md requirements)

- `tests/test_analyzer.py`: recurrence classification (monthly inference bounds, cv gate,
  `fixed`/`variable` tag override wins, export shape).
- `tests/test_rule_snapshots.py` must pass untouched.
- Playwright MCP against a generated report (`uv run tally up --no-embedded-html -o ...` for
  fast CSS/JS iteration), before committing:
  1. Zero console errors after a full interaction pass.
  2. KPI tiles react to a filter chip; Cash Flow tile flips red with a spending-heavy filter.
  3. Grouping pills: Years/Quarters hidden for single-unit spans; None → doughnut (category)
     with $/% legend + visible-slice recalc, per-series columns (cash flow).
  4. Sticky tooltip: frozen per column, bold follows mouse, Total/Net row, repositions on
     column change, hides on leave.
  5. Compare: dividers, two-row axis, 4-year cap caption, deduped legend.
  6. Pager on >24-bucket data; absent otherwise; resets to newest page on filter change.
  7. Both themes.
- Commit referencing the tracked issue (`Fixes #<n>`) — create one for this feature if none
  exists before the PR.
