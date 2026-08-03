> **Stacked on [`feature/globbing-documentation`](https://github.com/terryaney/OpenSource.tally/tree/feature/globbing-documentation) — PR #TBD.** Please merge that one first.
> **This layer only:** [`feature/globbing-documentation...feature/report-json-determinism`](https://github.com/terryaney/OpenSource.tally/compare/feature/globbing-documentation...feature/report-json-determinism)

`tally up` writes `<output>.json` beside the HTML report and diffs the next run against it — that file is the baseline for the "Changes since last run" message. It therefore has to be reproducible: the same config against the same data must produce the same bytes.

It currently isn't. Regenerating a report from an unchanged config produces a file of **identical length but a different hash**, because `pattern.tags` serializes in arbitrary order. This PR makes the output deterministic.

## At a Glance

- `analyzer.py` — sort `pattern.tags` unconditionally; the existing `isinstance(set)` guard was dead code
- `run.py` — extract `collect_source_names()` and deduplicate the report subtitle's source list
- `tests/test_analyzer.py` — four tests covering both fixes

## The defect

In `build_merchant_json()`:

```python
pattern_tags = match_info.get('tags', [])
if isinstance(pattern_tags, set):      # never fires
    pattern_tags = sorted(pattern_tags)
```

`MerchantEngine.MatchResult.tags` is a `Set[str]`, but `merchant_utils.normalize_merchant` already does `list(result.tags)` when it builds `match_info`. By the time the value reaches the exporter it is a **list** carrying the set's arbitrary iteration order, so the guard never fires and the sort is skipped. Python randomizes its string hash seed per process, so that order changes from run to run.

The fix sorts unconditionally:

```python
pattern_tags = sorted(match_info.get('tags', []))
```

This mirrors the top-level merchant `tags` field a few lines above, which handles both a set and a list and always sorts — that field was already deterministic and is unchanged here.

**Why sort at the export boundary rather than at the source.** Making `MatchResult.tags` order-preserving would mean editing `merchant_engine.py` / `merchant_utils.py`. Sorting on the way out is contained to the exporter, matches the sibling field, and cannot affect categorization.

**Accepted consequence:** the rule info popup now lists a rule's tags alphabetically rather than in rule-declaration order — consistent with the top-level tags list, which was already alphabetical.

## Report subtitle source list

`run.py` collected subtitle sources with an inline comprehension that could repeat a name when several sources share one. That collection moved into a small named helper so it can be tested:

```python
def collect_source_names(data_sources):
    return list(dict.fromkeys(
        s.get('name', 'Unknown') for s in data_sources if not s.get('_supplemental', False)
    ))
```

`dict.fromkeys` deduplicates while preserving the order sources are declared in `settings.yaml`, so the subtitle reads in the same order as the config file rather than being re-sorted.

This one does not affect the JSON — `source_names` only reaches `write_summary_file_vue`, never `export_json`.

## Verification

Two runs must be **separate processes**; Python randomizes the string hash seed per process, so two calls inside one interpreter agree even while the bug is present.

```powershell
uv run tally up --quiet -o "$scratch\a\r.html" -c <config-dir>
uv run tally up --quiet -o "$scratch\b\r.html" -c <config-dir>
(Get-FileHash "$scratch\a\r.json").Hash -eq (Get-FileHash "$scratch\b\r.json").Hash
```

Before the fix this returns `False` with both files at identical byte length — that equal-size/different-hash pair is the signature. After, it returns `True`.

## Tests

- `pattern.tags` sorted when `match_info['tags']` is a list in non-alphabetical order
- `pattern.tags` sorted when it arrives as a set — the path the dead guard used to cover
- `collect_source_names` deduplicates without reordering
- `collect_source_names` excludes supplemental sources

Nothing in the existing suite referenced `source_names`, which is why its behavior could drift unnoticed.

## Scope

Two source files, one test file. No rule-engine change, no change to categorization, HTML rendering, or any other export format.

## Coming next (not in this PR)

Each is based on the one above it, so they want merging in this order:

1. [`feature/ui-tweaks`](https://github.com/terryaney/OpenSource.tally/compare/feature/report-json-determinism...feature/ui-tweaks) — PR #TBD — date filtering, transaction details, per-transaction tag classification
2. [`feature/charts-reimagined`](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined) — PR #TBD — reimagined chart layout, KPI tiles, chart docs
3. [`feature/merchant-composite-keys`](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys) — PR #TBD — composite merchant identity so one merchant can carry multiple categorizations
4. [`feature/categorization`](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization) — PR #TBD — categorization review file, `review:` rule flag, `inventory.yaml`

Separately, [`feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair) — PR #TBD — is independent of this stack: `.github/workflows/` only.
