## Plan: Fixed Spending Cleanup

HARD STOP: DO not implement this plan.  User needs to discuss with you:
- Need to address fixed-annual tagging that is done in personal data
- Need to address fixed-budget tagging that is done in personal data, only applied to known NON-MONTHLY bills...for known monthly bills, use fixed only?

Current behavior: the report counts merchants as recurring when `analyzer.py` assigns `recurrence` and positive `recurring_monthly_cost`; the UI then counts every filtered merchant with those fields. The current 83 count is likely inflated because the annual fallback can mark broad/high-volume merchants recurring when any similar pair of payments appears 10-14 months apart.

Recommended approach: preserve all current behavior by default. Keep existing `fixed` and `variable` tag functionality, keep CV/annual inference active by default, add `infer_fixed_spending` as an opt-out boolean, and add a new explicit `fixed-budget` tag for baseline fixed costs averaged over trailing complete months. Only `infer_fixed_spending: false` and explicit use of `fixed-budget` change behavior.

**Steps**

1. Document current precedence and confirm desired behavior: `fixed` tag wins, `variable` blocks, monthly inference next, annual inference last; UI counts non-null `recurrence` with positive monthly cost.
2. Update fixed-spending semantics in `c:\BTR\OpenSource\tally\src\tally\analyzer.py`:
   - Keep existing `fixed` and `variable` tag behavior unchanged for backward compatibility.
   - Add/support `fixed-budget` as a new explicit tag for baseline fixed costs using trailing complete-month average math.
   - Add `infer_fixed_spending` setting: when false, CV/annual heuristics do not feed active fixed chart totals, but remain visible as explain/audit suggestions.
   - Document known `fixed-budget` pitfalls: fewer than 12 complete months of data can overstate sparse annual-ish payments; newly started bills after the lookback window can be understated until enough months accrue.
3. Improve report data shape in `c:\BTR\OpenSource\tally\src\tally\report.py` if needed:
   - Consider adding `recurrenceSource` such as `tag`, `inferred_monthly`, or `inferred_annual` so the UI can explain why a merchant appears.
4. Update report UI in `c:\BTR\OpenSource\tally\src\tally\spending_report.js` and `c:\BTR\OpenSource\tally\src\tally\spending_report.html`:
   - Keep Fixed vs Variable chart filter-reactive: sum fixed-classified transactions that pass the current filters and bucket/group them by the selected chart grouping.
   - For the fixed-budget audit, use a separate stable data source based on trailing complete months rather than reusing fully filtered chart aggregation.
   - Fixed vs Variable chart counts transactions tagged either `fixed` or `fixed-budget` as fixed; variable is spending not counted as fixed in the active filter context.
   - Audit ignores non-date filters. Date filters set one shared anchor month/date for the whole audit table; default anchor is latest complete month in the report data.
   - Audit rows share the same lookback window ending at the anchor, so totals are coherent. Do not anchor each row independently on its own last payment date.
   - `fixed-budget` math uses one shared complete-month lookback window. Default anchor is the latest complete month in report data; date filters set the anchor to the latest complete month at or before the filter end. The window includes the anchor month plus up to 11 prior complete months; current partial month is excluded.
   - Group fixed-budget audit rows by merchant display name plus category/subcategory (and, after PR #91, compatible composite identity), so same merchant with different fixed categories appears as separate rows.
   - Replace the old authoritative cadence column with an informational `Payment Pattern` column. Payment Pattern is a best-effort guess such as Monthly-ish, Annual-ish, Sparse, Irregular, or Unknown; it does not drive `fixed-budget` monthly-average math.
   - Audit columns should be fixed-budget oriented: Merchant, Category/Subcategory, Monthly Avg, Annualized, Payment Pattern, Window, Months Seen, Last Paid, Warnings.
   - Ship two `fixed-budget` warnings first: limited history when fewer than 12 complete months are available in the lookback, and recently started fixed-budget item when activity begins late in the window.
   - Include a visible footnote/help note on the fixed-budget audit explaining that the audit ignores non-date filters, date filters set the shared anchor, and the calculation uses the anchor month plus up to 11 prior complete months.
   - UI risk: the fixed-budget audit table has many columns, so the report UI will likely need post-implementation design iteration. Expect to adjust responsive layout, column priority, truncation, wrapping, or detail expansion after seeing real data.
5. Update docs/help/templates:
   - `c:\BTR\OpenSource\tally\src\tally\commands\reference.py`
   - `c:\BTR\OpenSource\tally\src\tally\commands\workflow.py`
   - `c:\BTR\OpenSource\tally\src\tally\commands\diag.py`
   - `c:\BTR\OpenSource\tally\src\tally\templates.py`
   - `c:\BTR\OpenSource\tally\docs\reference.html`, `docs\guide.html`, and chart docs as needed.
6. Account for PR #91 / issue #88 multi-category merchant work:
   - PR #91 changes same-name merchants toward composite merchant identity by category/subcategory and fixes report chart aggregation to use per-transaction tags.
   - The fixed-spending plan should not depend on old merchant-name-only grouping, because broad merchants like Amazon can have mixed fixed/variable transactions and mixed categories.
   - Fixed classification should operate at transaction/rule/group level, then aggregate for display.
   - PR #91 / issue #88 is a hard prerequisite. Do not implement this fixed-spending feature until PR #91 lands or its changes are otherwise incorporated into the target branch.
   - Do not make code changes for this feature before PR #91 lands, including `infer_fixed_spending` and temporary `fixed-budget` alias behavior. Pre-PR #91 work should stay limited to planning/docs unless explicitly revisited.
7. Update tests:
   - Keep existing `fixed`/`variable` compatibility tests, then add tests for `infer_fixed_spending` true/false behavior.
   - Add tests for `fixed` compatibility and the new fixed-budget-average behavior if a new tag/mode is chosen.
   - Add tests for irregular fixed costs, including weekly-ish or prepaid fixed services where monthly value should be based on recent spend averaged over the available lookback period rather than cadence labels.
   - Add a regression test for high-volume/lumpy merchants with incidental 10-14 month amount matches so they do not inflate active fixed totals when `infer_fixed_spending: false`.
   - Add/adjust HTML/report tests if chart labels or audit behavior changes.
8. Update personal config after behavior is settled:
   - In `c:\BTR\TallySpending\tally\config\merchants.rules`, use `fixed-budget` for baseline fixed costs that should use trailing complete-month average math.
   - Keep existing `fixed` tags only where current legacy fixed behavior is desired.
   - Keep `views.rules` tag views aligned with the fixed tag vocabulary.

**Verification**

1. Run focused analyzer tests: `uv run pytest tests/test_analyzer.py -k recurrence -v` from `c:\BTR\OpenSource\tally`.
2. Run report HTML tests covering the chart/audit if touched: `uv run pytest tests/test_report_html.py -v`.
3. Generate a report against `c:\BTR\TallySpending` and verify the fixed count is explainable and no longer unexpectedly inflated when `infer_fixed_spending: false`.
4. Use `tally explain --fixed -vv` or generated report details to confirm individual merchants show why they are fixed or fixed candidates.

**Decisions**

- Change user-facing report/docs/help language from "recurring" to "fixed" where this feature is discussed, but leave existing internal field names like `recurrence` and `recurringMonthlyCost` alone unless new `fixed-budget` data needs clearer new fields.
- Keep `fixed` current behavior for public backward compatibility.
- Add `fixed-budget` for explicitly tagged baseline fixed costs using trailing complete-month average math.
- Do not add settings for manual merchant lists. Merchant rules already support expressive matching and tagging through `match:` expressions plus `tags:`.
- Keep an informational `Payment Pattern` column in the fixed-budget audit. This preserves visibility into annual/less-frequent payment preparation while keeping `Monthly Avg` and `Annualized` as the authoritative budget math.
- `tally explain --fixed` is the CLI audit surface. Text output mirrors the fixed-budget audit grouping/columns and adds Active/Reason; JSON includes active, reason/source, window, monthlyAvg, annualized, monthsSeen, lastPaid, and warnings. It lists explicit fixed rows plus inference candidates, marking whether each is active under `infer_fixed_spending`.