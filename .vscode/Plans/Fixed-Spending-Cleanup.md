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
   - **Fix `infer_annual_amount()`** (`analyzer.py`, currently lines 166-183). It hunts a cross-product of amounts for any pair 10-14 months apart within 20%, so effectively any merchant with a few transactions over a multi-year window is marked annual. Replace with a cadence check on the merchant's own payment months — sort the months, require every consecutive gap to fall in 10-14, then validate amount similarity across those payments. Simulated over 36 months of random merchants, the current rule false-positives 100% of the time at every transaction density and the cadence rule 0%, while both real annual shapes (2 payments and 3 payments ~12 months apart) still detect.
   - This must be fixed here, not deferred: the `infer_fixed_spending` flag does not contain it, because step 2 keeps inference visible as explain/audit suggestions and step 3 surfaces it as `recurrenceSource: inferred_annual`. A broken inference would be promoted into the new audit surface as suggestion noise.
3. Improve report data shape in `c:\BTR\OpenSource\tally\src\tally\report.py` if needed:
   - Consider adding `recurrenceSource` such as `tag`, `inferred_monthly`, or `inferred_annual` so the UI can explain why a merchant appears.
4. Update report UI in `c:\BTR\OpenSource\tally\src\tally\spending_report.js` and `c:\BTR\OpenSource\tally\src\tally\spending_report.html`:
   - Keep Fixed vs Variable chart filter-reactive: sum fixed-classified transactions that pass the current filters and bucket/group them by the selected chart grouping.
   - For the fixed-budget audit, use a separate stable data source based on trailing complete months rather than reusing fully filtered chart aggregation.
   - Fixed vs Variable chart counts transactions tagged either `fixed` or `fixed-budget` as fixed; variable is spending not counted as fixed in the active filter context.
   - Audit ignores non-date filters. Date filters set one shared anchor month/date for the whole audit table; default anchor is latest complete month in the report data.
   - Audit rows share the same lookback window ending at the anchor, so totals are coherent. Do not anchor each row independently on its own last payment date.
   - `fixed-budget` math uses one shared complete-month lookback window. Default anchor is the latest complete month in report data; date filters set the anchor to the latest complete month at or before the filter end. The window includes the anchor month plus up to 11 prior complete months; current partial month is excluded.
   - Group fixed-budget audit rows by merchant display name plus category/subcategory, using the composite merchant identity from `feature/merchant-composite-keys`, so same merchant with different fixed categories appears as separate rows.
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
6. Branch placement — this feature sits **above** `feature/merchant-composite-keys` in the stack:
   - `feature/merchant-composite-keys` gives same-name merchants a composite identity by category/subcategory. `feature/ui-tweaks` (lower in the stack) already fixed chart aggregation to use per-transaction tags.
   - This plan must not depend on merchant-name-only grouping, because broad merchants like Amazon have mixed fixed/variable transactions across mixed categories.
   - Fixed classification should operate at transaction/rule/group level, then aggregate for display.
   - **Hard prerequisite: composite merchant identity.** This is a semantic dependency, not a merge conflict — the audit is *wrong*, not merely conflicted, without it. Branch from `feature/merchant-composite-keys`, never from a lower rung.
   - Accepted consequence: building on top welds `feature/merchant-composite-keys` into the stack — it can no longer be cheaply dropped or repositioned. That costs nothing real, since this feature cannot ship without it either way.
7. Update tests:
   - Keep existing `fixed`/`variable` compatibility tests, then add tests for `infer_fixed_spending` true/false behavior.
   - Add tests for `fixed` compatibility and the new fixed-budget-average behavior if a new tag/mode is chosen.
   - Add tests for irregular fixed costs, including weekly-ish or prepaid fixed services where monthly value should be based on recent spend averaged over the available lookback period rather than cadence labels.
   - Add regression tests for annual inference under the **default** `infer_fixed_spending: true`: high-volume and lumpy merchants with incidental 10-14 month amount matches must not be marked annual, while genuine 2-payment and 3-payment annual bills still are. Testing only the `infer_fixed_spending: false` path passes trivially and proves nothing.
   - `test_annual_inference_detects_similar_12_month_spacing` (`tests/test_analyzer.py:3414`) must keep passing unchanged — two payments 12 months apart is the normal shape of a real annual bill with two years of data, and any fix that requires more payment pairs breaks it.
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

**Post-Implementation Checks**

Do these after the feature works, against real `c:\BTR\TallySpending` data. None of them are blockers for shipping.

1. **Annual-inference false negatives from the strict cadence rule.** The step 2 fix requires *every* consecutive gap between a merchant's payment months to fall in 10-14. That is correct for the false-positive problem it solves, but it rejects a genuinely annual bill that also has an off-cadence purchase under the same composite key — an annual domain renewal plus one small one-off from the same vendor in the same category, for example.

   - **What to check:** after the fix lands, diff `tally explain --fixed` against a pre-fix run and list merchants that lost their `annual` classification. Every merchant on that list is either a true positive the old code got right by accident, or noise the fix correctly removed. Sort them by annualized amount — only ones large enough to matter are worth acting on.
   - **What *not* to trigger on:** the number of years of data. This is not a history-volume threshold. Composite merchant identity (the hard prerequisite from step 6) already splits most stray purchases onto their own audit row by category/subcategory, so the false negative only survives when the stray transaction shares the annual bill's category *and* subcategory. That may turn out to be rare enough that nothing needs building.
   - **First-line fix is a tag, not code.** A missed annual bill is one merchant rule the user writes; an invented one is a silently wrong budget number that nobody notices. Only reach for a code change if the same shape recurs across several merchants.

   If a code change is warranted, two relaxations were analyzed:

   - *Majority-of-gaps* (require two thirds of consecutive gaps in 10-14 rather than all of them) — **probably never worth building.** With 2 or 3 payment months there are only 1 or 2 gaps, and "majority" collapses to "all," making the rule byte-for-byte equivalent to strict. It only becomes distinguishable at 4+ payment months, meaning 4+ years of history for an annual bill. Worse, it does not even fix the stray-purchase case it is usually reached for: annual + annual + one stray yields 2 gaps with 1 bad, which majority still rejects. Recorded here so it is not re-proposed and re-analyzed later.
   - *Amount-outlier filtering* (drop payments far from the dominant amount cluster, then run the cadence check on what remains) — **the only relaxation that actually addresses stray purchases.** The hazard is that it is the amount-coincidence hunt this plan just removed, sneaking back in through the filter step. It needs hard gates if built: the kept payments must account for a dominant share of the merchant's total spend (~70-80%), the count of dropped payments must be capped, and the regression suite from step 7 must still show zero false positives on high-volume and lumpy merchants. Build it only against a real observed false negative, never speculatively.

2. **Fixed-budget audit table layout.** As noted in step 4, the audit carries many columns and will likely need responsive-layout, column-priority, truncation, or detail-expansion iteration once real data is in it.

**Decisions**

- Change user-facing report/docs/help language from "recurring" to "fixed" where this feature is discussed, but leave existing internal field names like `recurrence` and `recurringMonthlyCost` alone unless new `fixed-budget` data needs clearer new fields.
- Keep `fixed` current behavior for public backward compatibility.
- Add `fixed-budget` for explicitly tagged baseline fixed costs using trailing complete-month average math.
- Do not add settings for manual merchant lists. Merchant rules already support expressive matching and tagging through `match:` expressions plus `tags:`.
- Keep an informational `Payment Pattern` column in the fixed-budget audit. This preserves visibility into annual/less-frequent payment preparation while keeping `Monthly Avg` and `Annualized` as the authoritative budget math.
- Annual inference stays deliberately high-precision and low-recall. Explicit `fixed`/`fixed-budget` tags carry recall, so inference does not need to be clever — a missed annual bill is a rule the user writes, an invented one is a silently wrong budget number.
- **Ship the strict all-gaps cadence rule with no outlier tolerance.** Relaxations were considered and deliberately deferred rather than rejected — see Post-Implementation Checks, item 1, for the known false-negative shape, the two candidate relaxations, and the trigger for revisiting.
- The annual-inference minimum of two payments is unrelated to the `fixed-budget` "limited history" and "recently started" warnings, though both are history-shortage problems. A bill seen once cannot be inferred as annual at all — there is no interval to measure. `fixed-budget` handles that case instead: an explicitly tagged bill with a single payment is included with a warning rather than dropped. This is the concrete reason to tag known annual bills rather than lean on inference for them.
- `tally explain --fixed` is the CLI audit surface. Text output mirrors the fixed-budget audit grouping/columns and adds Active/Reason; JSON includes active, reason/source, window, monthlyAvg, annualized, monthsSeen, lastPaid, and warnings. It lists explicit fixed rows plus inference candidates, marking whether each is active under `infer_fixed_spending`.