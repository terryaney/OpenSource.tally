> **Stacked on [`feature/charts-reimagined`](https://github.com/terryaney/OpenSource.tally/tree/feature/charts-reimagined) — PR #TBD.** Please merge that one first (and the three it sits on).
> **This layer only:** [`feature/charts-reimagined...feature/merchant-composite-keys`](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys)

Fixes #88 allowing merchants to different categorization rules with same name.

Instead of requiring:
```
[Costco Grocery]
match: contains("COSTCO") and amount <= 200
category: Food
subcategory: Grocery

[Costco Bulk]
match: contains("COSTCO") and amount > 200
category: Shopping
subcategory: Wholesale
```

Can instead be:
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

PR correctly fixes the Python-side bug — changing `by_merchant` to use a composite `(merchant, category, subcategory)` key so same-named merchants with different categories appear as separate rows. This significantly reduces the practical impact of the JavaScript-side bug described here.

However, even after that fix, `merchant.tags` in the report data is still the **union of all transaction tags within that merchant entry**. When that union includes tags that not every transaction individually carries, `filteredViewTotals` misclassifies those transactions.  The second commit on my PR addresses this.

## Root Cause

In `spending_report.js`, both affected computed properties hoist the tags lookup outside the transaction loop:

```js
// filteredViewTotals
const tags = merchant.tags || [];          // ← merchant-level aggregate
const txns = merchant.filteredTxns || merchant.transactions || [];
for (const txn of txns) {
    const c = categorizeAmount(txn.amount || 0, tags);  // ← all txns get same tags
```

```js
// chartAggregations
const tags = merchant.tags || [];          // ← same problem
for (const txn of merchant.filteredTxns || []) {
    const c = categorizeAmount(txn.amount, tags);
```

Each individual transaction already carries its own `tags` array in the report JSON (e.g., `{"id": "...", "date": "06/22", "amount": 33.51, "tags": ["monthly-bill"]}`), so the data is available.

## Coming next (not in this PR)

1. [`feature/categorization`](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization) — PR #TBD — categorization review file, `review:` rule flag, `inventory.yaml`

Separately, [`feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair) — PR #TBD — is independent of this stack: `.github/workflows/` only.