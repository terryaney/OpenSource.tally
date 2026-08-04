Title: Composite merchant keys for same-name multi-category rules [5/6]

> **[5/6]** Based on `feature/charts-reimagined` (#101).
> Merging this PR into `main` also lands #98, #99, #100, and #101 if not already merged.
> **This layer only:** [`feature/charts-reimagined...feature/merchant-composite-keys`](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys)

Fixes #88 — same-named merchant rules with different categories now work:

```
[Costco]
match: contains("COSTCO") and amount <= 200
category: Food
subcategory: Grocery

[Costco]
match: contains("COSTCO") and amount > 200
category: Shopping
subcategory: Wholesale
```

## Changes

- **`analyzer.py`** — `by_merchant` uses a composite `(merchant, category, subcategory)` key so same-named merchants with different categories appear as separate report rows
- **`spending_report.js`** — `filteredViewTotals` and `chartAggregations` now use per-transaction `txn.tags` instead of the merchant-level tag union, fixing misclassification when one transaction's tags don't apply to all transactions at that merchant
- **`commands/explain.py`** — follows `by_merchant` to composite keys, so a name that now resolves to several entries can return all of them

## Behavior change — `tally explain --format json` output shape

Intentional, and required by the fix: one merchant name can now legitimately resolve to more than one entry, which a single object cannot express. Merchant queries now emit a wrapper:

```json
{
  "query": "Costco",
  "match_mode": "exact",
  "matched_names": ["Costco"],
  "merchants": [ { "name": "Costco", "category": "Food", ... } ]
}
```

where it previously emitted a bare merchant object. `match_mode` is `exact`, `case_insensitive`, or `partial`.

This also repairs output that was already invalid: the pre-existing partial-match path printed a human-readable `Merchants matching 'x':` header followed by several concatenated JSON objects, even under `--format json`, which no parser could consume. That path now emits one well-formed document.

Nothing persisted changes — `explain` computes from config and data files at run time and reads no output of `tally up`, so there is no mixed-version concern. The only thing that breaks is a script parsing the old single-object shape from a previous release, and the wrapper is deliberately uniform rather than switching shape on match count, so such a script has one form to move to rather than two.

## Stack

| # | Branch | PR | Description | Diff |
|---|---|---|---|---|
| 1 | `feature/globbing-documentation` | #98 | Glob pattern docs and CLI tests | [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/globbing-documentation) |
| 2 | `feature/report-json-determinism` | #99 | Deterministic report JSON | [from PR98](https://github.com/terryaney/OpenSource.tally/compare/feature/globbing-documentation...feature/report-json-determinism) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/report-json-determinism) |
| 3 | `feature/ui-tweaks` | #100 | Date filter, transaction details, per-txn tags | [from PR99](https://github.com/terryaney/OpenSource.tally/compare/feature/report-json-determinism...feature/ui-tweaks) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ui-tweaks) |
| 4 | `feature/charts-reimagined` | #101 | Reimagined charts and KPIs | [from PR100](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/charts-reimagined) |
| **5** | **`feature/merchant-composite-keys`** | **#102** | **Composite merchant keys** | **[from PR101](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/merchant-composite-keys)** |
| 6 | `feature/categorization` | #103 | Categorization review file | [from PR102](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/categorization) |

Independent: [`feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair) — #97 — fixes fork PR builds.