Title: Document and validate glob file patterns [1/6]

> **[1/6]** Based directly on `main`.
> **Diff:** [`main...feature/globbing-documentation`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/globbing-documentation)

Add glob pattern examples in settings.yaml.example, clarify formats docs for file globs, and add CLI tests covering multi-file matching, no-match behavior, diag visibility, and sorted processing order.

There was a stale PR (#45) that I was going to merge locally not realizing implementation was already done to support the feature, so I just created some tests and updated few small documentation files and should support closing that PR.

## Stack

| # | Branch | PR | Description | Diff |
|---|---|---|---|---|
| **1** | **`feature/globbing-documentation`** | **#98** | **Glob pattern docs and CLI tests** | **[from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/globbing-documentation)** |
| 2 | `feature/report-json-determinism` | #99 | Deterministic report JSON | [from PR98](https://github.com/terryaney/OpenSource.tally/compare/feature/globbing-documentation...feature/report-json-determinism) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/report-json-determinism) |
| 3 | `feature/ui-tweaks` | #100 | Date filter, transaction details, per-txn tags | [from PR99](https://github.com/terryaney/OpenSource.tally/compare/feature/report-json-determinism...feature/ui-tweaks) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ui-tweaks) |
| 4 | `feature/charts-reimagined` | #101 | Reimagined charts and KPIs | [from PR100](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/charts-reimagined) |
| 5 | `feature/merchant-composite-keys` | #102 | Composite merchant keys | [from PR101](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/merchant-composite-keys) |
| 6 | `feature/categorization` | #103 | Categorization review file | [from PR102](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/categorization) |

Independent: [`feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair) — #97 — fixes fork PR builds.
