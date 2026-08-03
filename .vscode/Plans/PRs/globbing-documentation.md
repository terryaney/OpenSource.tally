> **Bottom of the stack — based directly on `main`.** Five further PRs sit on top of this one; see *Coming next*.
> **Diff:** [`main...feature/globbing-documentation`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/globbing-documentation)

Add glob pattern examples in settings.yaml.example, clarify formats docs for file globs, and add CLI tests covering multi-file matching, no-match behavior, diag visibility, and sorted processing order.

There was a stale PR (#45) that I was going to merge locally not realizing implementation was already done to support the feature, so I just created some tests and updated few small documentation files and should support closing that PR.

## Coming next (not in this PR)

Each is based on the one above it, so they want merging in this order:

1. [`feature/report-json-determinism`](https://github.com/terryaney/OpenSource.tally/compare/feature/globbing-documentation...feature/report-json-determinism) — PR #TBD — make the report JSON byte-reproducible
2. [`feature/ui-tweaks`](https://github.com/terryaney/OpenSource.tally/compare/feature/report-json-determinism...feature/ui-tweaks) — PR #TBD — date filtering, transaction details, per-transaction tag classification
3. [`feature/charts-reimagined`](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined) — PR #TBD — reimagined chart layout, KPI tiles, chart docs
4. [`feature/merchant-composite-keys`](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys) — PR #TBD — composite merchant identity so one merchant can carry multiple categorizations
5. [`feature/categorization`](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization) — PR #TBD — categorization review file, `review:` rule flag, `inventory.yaml`

Separately, [`feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair) — PR #TBD — is independent of this stack: it touches `.github/workflows/` only and fixes the `pull_request_target` break that currently makes every fork PR's build go red.
