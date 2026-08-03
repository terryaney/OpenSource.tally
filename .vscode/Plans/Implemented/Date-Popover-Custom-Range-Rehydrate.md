# Date Popover — Rehydrate Custom Range on Open

**Target:** `c:\BTR\OpenSource\tally\src\tally\spending_report.js`
**Status:** Not yet implemented. Split out of the code review of the Spending-by-Category chart-filtering branch (2026-08-02); intended for the **ui-tweaks** branch alongside the calendar control work.
**Severity:** Data-loss-shaped — a filter the user set silently disappears on an action that looks like a no-op.

## The bug

An applied custom `daterange` chip is not loaded back into the popover's Start/End widget when the popover reopens, and Apply rebuilds the date chips from scratch. So opening the popover and pressing Apply — changing nothing — deletes the range.

Reproduce:

1. Load a report at `#+dr:2024-01-01..2024-01-31` (or set a custom range by hand).
2. Open the Date filter popover. Start/End render empty.
3. Click **Apply**.
4. The `daterange` chip is gone and the hash is cleared.

## Why it happens

| Step | Where | Behavior |
|---|---|---|
| Open | `openDatePopover` — `spending_report.js:2108` | Expands `month`-type chips into `pendingMonths`. `daterange` chips are deliberately skipped ("stay independent"), but nothing repopulates `customStart` / `customEnd`, so both stay `null`. |
| Build | `getCustomRangeChip` — `spending_report.js:2131` | Returns `null` whenever either ref is `null`. |
| Apply | `applyDateFilters` — `spending_report.js:2147` | Drops **all** date-category chips, re-adds from `pendingMonths`, then appends `getCustomRangeChip()` — which is `null`. Net: the range chip is deleted. |

The month path round-trips correctly; only the custom range is lossy.

## Fix

In `openDatePopover`, rehydrate the widget from the active `daterange` chip before opening.

1. While looping `activeFilters`, also match `f.type === 'daterange'`.
2. `f.text.split('..')` gives two `YYYY-MM-DD` halves.
3. Parse each with the existing module-scope `parseTypedDate` (`spending_report.js:223`) — it already handles `YYYY-MM-DD` and returns the `{ y, m, d }` shape `customStart` / `customEnd` expect. Do **not** add a new parser.
4. Assign to `customStart.value` / `customEnd.value`; skip assignment if either parse returns `null` so a hand-edited hash can't wedge the widget.
5. Take the first `daterange` chip only — `getCustomRangeChip` can express exactly one, and `applyDateFilters` appends at most one.

Notes:

- `getCustomRangeChip` already normalizes reversed ranges, so no ordering work is needed on the way in.
- `removeFilter` already calls `clearCustomRange()` when a `daterange` chip is removed (`spending_report.js` — the `removed.type === 'daterange'` branch), so chip removal stays in sync. No change there.
- Leave the "daterange chips are never expanded into `pendingMonths`" rule intact — months and the custom range are independent by design; this fix only restores the widget, not the month grid.

## Test checklist

- Load `#+dr:2024-01-01..2024-01-31` → open popover → Start/End show Jan 1 2024 / Jan 31 2024.
- Same, then click Apply with no edits → chip and hash unchanged (the regression case).
- Apply with an edited range → old chip replaced by the new one, exactly one `daterange` chip present.
- Remove the range chip via its `×` → reopen popover → Start/End empty (no resurrection on Apply).
- Month selections + a custom range together → reopen → both survive an unedited Apply.
- Reversed input (End before Start) → still normalized on Apply.
- Malformed hash (`#+dr:garbage`) → popover opens with empty Start/End, no console error.
- Chart "peek" filters: `applyDateFilters` intentionally drops `source: 'chart'` filters — confirm that still holds after the change (see the chart-filtering plan, decision 12).

## Already fixed — do not redo

The companion finding from the same review, **"Top 10 Fixed" caption ordered alphabetically** (`renderFixedVariable`), was fixed on the chart-filtering branch: `fixedNames` now sorts by `recurringMonthlyCost` descending on a copied array before slicing to 10. Verified to match the Fixed Spending Audit table's cost ranking. Listed here only so it is not re-opened as ui-tweaks work.
