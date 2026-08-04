Title: Make report JSON deterministic [2/6]

> **[2/6]** Based on `feature/globbing-documentation` (#98).
> Merging this PR into `main` also lands #98 if not already merged.
> **This layer only:** [`feature/globbing-documentation...feature/report-json-determinism`](https://github.com/terryaney/OpenSource.tally/compare/feature/globbing-documentation...feature/report-json-determinism)

`tally up` writes `<output>.json` beside the HTML report and diffs the next run against it — that file is the baseline for the "Changes since last run" message. The same config against the same data should produce the same bytes, but currently doesn't — `pattern.tags` serializes in arbitrary order because Python randomizes hash seeds per process.

## Changes

- **`analyzer.py`** — sort `pattern.tags` unconditionally; the existing `isinstance(set)` guard never fired because the value arrives as a list
- **`run.py`** — extract `collect_source_names()` to deduplicate the report subtitle's source list (order-preserving)
- **`tests/test_analyzer.py`** — four tests covering both fixes

Two source files, one test file. No rule-engine change, no change to HTML rendering or any other export format.

## Stack

| # | Branch | PR | Description | Diff |
|---|---|---|---|---|
| 1 | `feature/globbing-documentation` | #98 | Glob pattern docs and CLI tests | [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/globbing-documentation) |
| **2** | **`feature/report-json-determinism`** | **#99** | **Deterministic report JSON** | **[from PR98](https://github.com/terryaney/OpenSource.tally/compare/feature/globbing-documentation...feature/report-json-determinism) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/report-json-determinism)** |
| 3 | `feature/ui-tweaks` | #100 | Date filter, transaction details, per-txn tags | [from PR99](https://github.com/terryaney/OpenSource.tally/compare/feature/report-json-determinism...feature/ui-tweaks) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ui-tweaks) |
| 4 | `feature/charts-reimagined` | #101 | Reimagined charts and KPIs | [from PR100](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/charts-reimagined) |
| 5 | `feature/merchant-composite-keys` | #102 | Composite merchant keys | [from PR101](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/merchant-composite-keys) |
| 6 | `feature/categorization` | #103 | Categorization review file | [from PR102](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/categorization) |

Independent: [`feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair) — #97 — fixes fork PR builds.
