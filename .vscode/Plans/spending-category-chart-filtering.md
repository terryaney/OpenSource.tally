# Spending by Category — Chart Filtering Interactions

**Target:** `c:\BTR\OpenSource\tally\src\tally\spending_report.js` (single-file Vue + Chart.js spending report)
**Status:** Design finalized 2026-08-02 via grill session. **Implemented, reviewed, and revised 2026-08-02** — see [Post-review revisions](#post-review-revisions). Decisions below reflect what actually shipped; superseded wording is marked inline rather than deleted.

## Use cases

- **UC1 — drill-down:** Notice a big "Home" segment in Dec '25. Click that bar segment → report (transaction details) filters to Home + Dec '25. Inspect, then one-click undo via a local "Clear Filters" button on the chart panel (no scrolling to the top filter bar).
- **UC2 — comparison:** Click Food, Home, Auto chips → report filters to those three categories; chart shows just those three series compared across whatever date range filter is active, without clutter.

## Settled design decisions

1. **Legend chips become filter toggles** (replacing today's hide/show). Three visual states: `regular` (no category filtering), `selected`, `not-selected`.
   - All chips start `regular`. First click: clicked chip → `selected`, adds its category filter, all other chips → `not-selected`.
   - Subsequent clicks are additive/subtractive on the category filter set.
   - If any click results in **all on or all off**: remove all category filters, all chips back to `regular`.
   - Accepted consequence: selecting all top-10 chips ≡ no filter (cannot express "top 10 only, exclude the Other tail").
   - Pure "hide series without filtering" behavior is intentionally removed (user declined a modifier-click fallback).
2. **Chip state = derived from `activeFilters`** (`category`-type, `include`-mode filters), bidirectional, single source of truth. Filters added by search box, canvas click, or hash restore sync the chips; removing a filter from the filter bar resets chips accordingly.
3. **Chip list = a Vue computed**, not a stored snapshot: top 10 categories computed from data with all filters applied **except include-mode `category` filters**. Auto-refreshes when date/text/merchant/tag filters change.
   - ~~Pin any actively-filtered category into the chip list even if it falls out of the top 10~~ — **REVISED, pinning was removed.** It conflicted with the sub-bullet below, and it cost horizontal chip-row real estate. The sub-bullet is now the authoritative behavior.
   - **Actual behavior:** a filter for a category inside "Other" has no chip; the legend shows all chips `not-selected` with none selected. The chip list therefore depends only on the non-category filters, which is what makes the fast path in decision 4 safe (see revision R2).
4. **Chart data source — option (a):** the category chart aggregates from data that ignores **include-mode** `category` filters. Non-selected categories render as **hidden datasets**. Chip toggles flip visibility in place via `setDatasetVisibility` + `update('none')` — no chart rebuild, no reanimation. The rest of the report (tables, cash flow, heatmap, share view) keeps using the normal fully-filtered aggregation.
   - **Exclude-mode `category` filters stay applied to the chart** (revision R1). Chips only ever create includes, so exempting includes alone is enough to keep chip toggles from disturbing the bars.
5. **Canvas click on a bar segment** adds category + the clicked bucket's date filter:
   - month grouping → `month` filter `YYYY-MM`
   - quarter grouping → range `YYYY-MM..YYYY-MM` (e.g. Q2 '24 → `2024-04..2024-06`; `monthMatches` already supports ranges)
   - year grouping → `YYYY-01..YYYY-12`
   - **compare-years mode:** resolve the year from the clicked dataset's `ttYear` + period label (clicking Feb in the 2023 stack adds `2023-02`, not the current year's Feb)
   - Date filter changes buckets, so a full chart redraw on canvas click is expected and accepted.
6. **Focused (unstack) mode unchanged:** chips only change `categoryState.focused`; no filtering. Chart stays isolated as today. Share view (grouping = none) renders no chips — unaffected. (Shipped addition: the "Other" chip is inert here too, per decision 7.)
7. **"Other" is inert everywhere:** chip has no click handler and no pointer cursor (CSS); canvas "Other" segment is a no-op with cursor kept default via `onHover`.
8. **Chart-added filters are tagged** `source: 'chart'` — both chip clicks and canvas clicks. A **"Clear Filters"** button appears in the chart panel header only while chart-tagged filters exist; clicking removes exactly those, leaving hand-typed filters alone. Tooltip: **"Clear filters added from the Spending by Category chart."**
   - Shipped addition: a **"Peek" badge** (eye icon) renders beside the chart title for exactly as long as chart-tagged filters exist. It stays visible while the panel is collapsed, so a folded chart cannot hide that it is filtering the report.
9. **Chart-added filters are NOT hash-serialized.** `filtersToHash` skips `source: 'chart'` filters entirely — a reload drops them (rather than orphaning them as untagged persistent filters). Session-convenience semantics.
10. ~~**Dedupe edge accepted as-is:** if a chart-added filter is later also typed manually, the add is a dedupe no-op, the filter keeps its chart tag, stays non-persistent, and the Clear Filters button removes it. No "promotion" rule.~~ — **SUPERSEDED by decision 12.** Manually adding any filter now clears chart-tagged filters first, so a by-hand re-add lands as a normal, hash-persisted filter. The awkward edge no longer exists.
11. **Legacy `categoryHidden` state is obsolete for this chart** (both regular and compare-mode datasets switch to filter-derived visibility). Removed outright; it was never written to `localStorage`, so there was no persisted state to migrate.
12. **Manual filtering exits "peek" mode** (added post-review, at user request). Chart-driven filters are transient. Any filter the user adds by hand — search box, a table row's `filter` button, a date preset, or the date popover's Apply — drops every `source: 'chart'` filter first. Rationale: once you filter deliberately you are no longer peeking, and stale chart filters must not linger invisibly behind the real one. Consequence: this is what supersedes decision 10.
13. **All-on / all-off is counted over chip-backed names only** (added post-review). A hand-typed filter for a category inside "Other" has no chip, so it must not count toward the all-on threshold. Reaching either end still clears **every** category filter including that chip-less one — leaving it behind would strand the legend in the all-`not-selected` state instead of `regular`, which decision 1 requires.

## Key code anchors (line numbers as of the post-review state)

| What | Where |
|---|---|
| `passesFilters` + `skipCategory` option (exempts include-mode category filters only) | `spending_report.js:1902` |
| `categoryExemptAggregations` — chart's category-exempt data | `spending_report.js:1608` |
| `selectedCategoryChips` — selection derived from `activeFilters` | `spending_report.js:1634` |
| `chipCategories` — chip list, reuses `buildCategoryBreakdown` | `spending_report.js:1651` |
| `addFilter` — `source` param + peek-mode exit | `spending_report.js:1992` |
| `filtersToHash` — skips `source: 'chart'` | `spending_report.js:2822` |
| `applyDateFilters` — also drops chart filters | `spending_report.js:2147` |
| `applyCategorySelection` / `toggleCategoryChip` | `spending_report.js:2902` / `2914` |
| `hasChartFilters` / `clearChartFilters` | `spending_report.js:2933` / `2935` |
| `compareClickToEntry` — compare-mode year resolution | `spending_report.js:3102` |
| `stripEmptyAxisSlots` — now trims a parallel `meta` array | `spending_report.js:3114` |
| `chipLegend` — tri-state + `disabled` support | `spending_report.js:3443` |
| `buildCategoryBreakdown` — shared top-10 split | `spending_report.js:3918` |
| `renderCategoryFilterLegend` | `spending_report.js:4175` |
| `updateChartFilterIndicators` — Clear Filters button + Peek badge | `spending_report.js:4188` |
| `applyCategoryChartFastVisibility` — no-rebuild toggle + divergence guard | `spending_report.js:4200` |
| `renderCategoryTrend` — stacked / compare / focused / chips / canvas click | `spending_report.js:4229` |
| `buildBuckets`, `comparePeriods` | `spending_report.js:3027`, `3054` |
| Peek badge markup / styles | `spending_report.html` chart-head · `spending_report.css` `.chart-peek-badge` |

## Implementation outline — as built

1. **Category-exempt aggregation:** `categoryExemptAggregations` mirrors `chartAggregations`' `byCategory` / `byCategoryByMonth` using `passesFilters(txn, merchant, { skipCategory: true })`. `chipCategories` derives the top 10 + "Other" from it by reusing `buildCategoryBreakdown`.
2. **Filter tagging:** `addFilter(text, type, displayText, source)`; chart interactions pass `'chart'`. `filtersToHash` skips tagged filters. `addFilter` and `applyDateFilters` clear tagged filters when the caller is not the chart (decision 12).
3. **Chip rendering:** `chipLegend` grew `state` (`regular` / `selected` / `not-selected`) and `disabled`; other legends keep the boolean `active` path. CSS adds `.legend-chip.not-selected` and `.legend-chip.disabled`.
4. **Chip click logic:** `toggleCategoryChip` computes the next selection from `selectedCategoryChips`, applies the all-on/all-off reset per decisions 1 and 13, then `applyCategorySelection` reconciles `activeFilters` to match exactly.
5. **In-place visibility:** datasets are built from the exempt aggregation with `hidden` = not-selected. `applyCategoryChartFastVisibility` flips `setDatasetVisibility` + `update('none')` on the live instance, gated by `categoryChipToggleInFlight` so only chip toggles take the fast path. It bails to a full rebuild if the live dataset labels no longer match the expected chip set (revision R2).
6. **Canvas click:** options-level `onClick` with the chart's `{ mode: 'nearest', intersect: true }` interaction → dataset label (skipping `OTHER_CATEGORY_LABEL`) + `ttYear` (compare) + bucket index → `aggregateEntryToChip`. `stripEmptyAxisSlots` trims a parallel `meta` array so bucket lookups stay index-aligned. `onHover` sets the pointer cursor except over "Other".
7. **Clear Filters button + Peek badge:** both driven by `updateChartFilterIndicators`, visible exactly while `hasChartFilters` is true.
8. **Focused mode:** chip handler unchanged (focus switching); "Other" made inert.
9. **Cleanup:** `categoryHidden` removed entirely.

## Test checklist

- Chip flows: single select, additive, subtractive, all-on reset, all-off reset. ✔
- Typed category filter syncs chips; typed Other-constituent category → all chips `not-selected`, none selected, **no pinned chip**. ✔ (verified with `#+c:Healthcare`)
- Exclude-mode category filter hides that category from the chart as well as the report. ✔ (regression R1)
- Canvas click in month/quarter/year groupings; compare mode picks the correct year; paged buckets and paged compare-years still map correctly.
- Clear Filters button and Peek badge appear/disappear together; clear only chart-added filters; manual filter-bar removal of a chart filter also updates chips and indicators. ✔
- Peek badge remains visible while the chart panel is collapsed. ✔
- Manual filter (search box / date Apply) drops chart filters and leaves exactly the hand-typed one, which persists to the hash. ✔ (decision 12)
- Reload: chart-added filters absent from hash and gone after reload; hand-typed filters persist. ✔
- Focused mode and share view unaffected; Other inert (chip + canvas), cursor correct.
- Chip toggles do not reanimate the chart; canvas click (date filter) does redraw.
- Excludes, text/tag/merchant filters still compose correctly with chip-driven category filters. ✔
- Degenerate case: when the active filters leave only one chip, clicking it is all-on ≡ no filter. Expected per decision 1. ✔

## Post-review revisions

Two defects were found by review after the first implementation pass, both confirmed at runtime against a report generated from `tests/fixtures/rule_snapshot`, both fixed.

**R1 — exclude-mode category filters were ignored by the chart.** The first `skipCategory` implementation exempted `f.type === 'category'` regardless of mode, so excluding a category removed it from the whole report but left it drawn in the chart, with its chip still reading `regular`.

```
Housing excluded → rest of report: no Housing anywhere
                 → chart: Housing visible=True total=27255 (largest bar)
```

Fixed by exempting only `mode === 'include'`. This also restores the checklist line "Excludes … still compose correctly."

**R2 — the no-rebuild fast path could leave a stale dataset.** With pinning (decision 3, since removed), deselecting a pinned tail category changed the chip list, but `applyCategoryChartFastVisibility` only flipped visibility and re-rendered the legend — never the datasets.

```
after deselecting pinned Healthcare:
  legend: 11 chips (no Healthcare)     datasets: 12 series (Healthcare still drawn)
  Other total 2742 vs baseline 3838 — Other under-reported by Healthcare's $1,097
```

Fixed twice over: removing pinning makes the chip list independent of the category selection, so it structurally cannot move during a chip toggle; and the fast path now compares live dataset labels against the expected chip set and falls back to a full rebuild on any mismatch.

**Verification:** `node --check` clean; report regenerates clean; `pytest tests/` 927 passed; Playwright drove chip toggles, canvas clicks across groupings, exclude mode, the peek lifecycle, and panel collapse with no console or page errors.

## Related plans

- `Date-Popover-Custom-Range-Rehydrate.md` — separate defect found in the same review (an applied custom `daterange` chip is deleted by opening the popover and pressing Apply). Scheduled for the ui-tweaks branch, not this one.
