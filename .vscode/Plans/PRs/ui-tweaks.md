Title: Date filtering, transaction details, and per-transaction tag classification [3/6]

> **[3/6]** Based on `feature/report-json-determinism` (#99).
> Merging this PR into `main` also lands #98 and #99 if not already merged.
> **This layer only:** [`feature/report-json-determinism...feature/ui-tweaks`](https://github.com/terryaney/OpenSource.tally/compare/feature/report-json-determinism...feature/ui-tweaks)

This PR fixes a data correctness bug in the HTML report and adds date filtering, transaction details, and layout polish.

Screenshots below are from simulated data — visual reference only.

## Data Correctness Fix

Charts, Filtered View totals, and Section View percentages previously classified transactions using `merchant.tags` (the union across all transactions for that merchant). One `income` or `transfer` transaction could misclassify every other transaction at the same merchant. This PR switches to per-transaction `txn.tags`, matching the Python-side KPI logic.

<img width="1121" height="1233" alt="image" src="https://github.com/user-attachments/assets/bdb17498-7aa4-437c-8dce-e9d473971a0a" />

## Changes

- Per-transaction tag classification for charts, Filtered View, and Section View percentages
- Month / Quarter / Year / Custom date filtering with URL hash persistence
- Collapsible Transaction Details with sticky headers and pinned totals
- Loading shell while Vue mounts
- `report_fields` config so captured fields like memo appear without per-rule passthroughs
- Layout preferences persisted in `localStorage` with a reset control
- Rule provenance popup improvements
- Starter title changed to `Tally Spending Analysis`

## Behavior changes

Calling these out explicitly — they change what the report *does*, not just how it looks, and they aren't obvious from the diff:

- **Chart clicks now toggle visibility instead of applying filters.** Clicking a category pie segment or a category-by-month bar no longer adds a category/date filter. This is deliberate, not a dropped handler. (#101 then reworks it further, making Spending by Category an explicit filter surface with tri-state legend chips.)
- **Available months now come from `categoryView`**, so a month is no longer lost just because its merchant was excluded from every configured view.
- **Section View percentages use a new denominator** — `filteredViewTotals.value.spending`, computed per transaction — instead of whole-merchant include/exclude decisions derived from merchant tag unions, which could drop ordinary spending from the denominator in mixed-tag datasets.
- **Blank captured fields no longer render**, so the phantom `+1 Memo` indicator is gone.
- **Rule `field:` directives that evaluate to empty now emit no field at all**, rather than a field with a blank value. A `field:` directive runs for every transaction its rule matches, so a merchant whose rule captures a memo previously attached an *empty* memo to every transaction that didn't have one — which is what produced a `+1` popup indicator on transactions with nothing in the popup. `0` and `False` are retained; only `None` and empty strings/lists/dicts are dropped. This affects the `+N` indicator, the dynamic `extra_fields` columns in CSV export, and report search — it does **not** affect rule matching, since `field.<name>` in a match expression reads captured columns from the data file, not values emitted by another rule's `field:` directive.

**Mobile:** this is a locally generated HTML file and I expect nearly all use to be desktop/large display. Several pre-existing mobile issues remain and are deliberately out of scope here.

## Date Filtering

<img alt="image" src="https://github.com/user-attachments/assets/b87f9eee-a2be-44f8-a596-86e95fb6291d" />

<img alt="image" src="https://github.com/user-attachments/assets/11b901a3-2e04-49b8-9e29-8fde1590c6ea" />

## Loading State

<img alt="image" src="https://github.com/user-attachments/assets/98497684-afab-4160-b93e-7162d10ec0d3" />

## Transaction Details

<img alt="image" src="https://github.com/user-attachments/assets/20774752-d03a-4655-9e77-9164478bb55c" />

<img alt="image" src="https://github.com/user-attachments/assets/66a47034-ed4b-4d9b-afd9-56a6a4a4acae" />

<img alt="image" src="https://github.com/user-attachments/assets/7c56f70b-c561-4484-9cb7-a2fe784597f8" />

<img alt="image" src="https://github.com/user-attachments/assets/9246e05c-3c2d-4e44-8cab-13a6a4cc4b5b" />

## Stack

| # | Branch | PR | Description | Diff |
|---|---|---|---|---|
| 1 | `feature/globbing-documentation` | #98 | Glob pattern docs and CLI tests | [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/globbing-documentation) |
| 2 | `feature/report-json-determinism` | #99 | Deterministic report JSON | [from PR98](https://github.com/terryaney/OpenSource.tally/compare/feature/globbing-documentation...feature/report-json-determinism) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/report-json-determinism) |
| **3** | **`feature/ui-tweaks`** | **#100** | **Date filter, transaction details, per-txn tags** | **[from PR99](https://github.com/terryaney/OpenSource.tally/compare/feature/report-json-determinism...feature/ui-tweaks) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ui-tweaks)** |
| 4 | `feature/charts-reimagined` | #101 | Reimagined charts and KPIs | [from PR100](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/charts-reimagined) |
| 5 | `feature/merchant-composite-keys` | #102 | Composite merchant keys | [from PR101](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/merchant-composite-keys) |
| 6 | `feature/categorization` | #103 | Categorization review file | [from PR102](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/categorization) |

Independent: [`feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair) — #97 — fixes fork PR builds.