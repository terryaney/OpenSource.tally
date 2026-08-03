# Report JSON — Make Output Deterministic

**Target:** `c:\BTR\OpenSource\tally\src\tally\analyzer.py`, `c:\BTR\OpenSource\tally\src\tally\commands\run.py`
**Status:** Not implemented. Two independent one-line fixes.
**Scope note:** Pure Python (analyzer + CLI). Unrelated to the chart-filtering front-end changeset — keep it a separate commit.

## Why

`tally up` writes `<output>.json` beside the HTML and diffs the next run against it ([`run.py:333`](../../../tally/src/tally/commands/run.py)). That file is the baseline for the "Changes since last run" message, so it has to be reproducible: same config + same data must produce the same bytes.

It currently does not. Regenerating a report from an unchanged config produces a file of **identical length but a different hash** — two fields serialize in arbitrary order because they come from Python `set`s, whose iteration order varies with the per-process string hash seed.

Harmless for the diff itself (`compare_reports` compares tags as sets, so it won't invent tag changes) but it makes the JSON useless as a byte-comparable dev-validation artifact, which is exactly what `spending_check.json` is for.

## Fix 1 — `pattern.tags` serialized unsorted

**Where:** `analyzer.py`, in the merchant export helper, "Add pattern match info if available".

```python
    match_info = data.get('match_info')
    if match_info:
        pattern_tags = match_info.get('tags', [])
        if isinstance(pattern_tags, set):      # <-- guard never fires
            pattern_tags = sorted(pattern_tags)
```

The guard is dead. `MerchantEngine.MatchResult.tags` is a `Set[str]`, but `merchant_utils.normalize_merchant` already does `list(result.tags)` when building `match_info`, so by the time it reaches here it is a **list** carrying the set's arbitrary order. The sort is skipped and that order lands in the JSON.

**Change:** sort unconditionally.

```python
        pattern_tags = sorted(match_info.get('tags', []))
```

This mirrors what the top-level merchant `tags` field a few lines above already does (it handles both set and list and always sorts) — that field is already deterministic and needs no change.

**Why sort at the export boundary rather than fix the source:** making `MatchResult.tags` order-preserving would mean editing `merchant_engine.py` / `merchant_utils.py`, which `CLAUDE.md` puts under change control (new behavior behind a flag, snapshot tests, default unchanged). Sorting on the way out is contained, matches the sibling field, and cannot alter categorization.

**Accepted consequence:** the info popup lists a rule's tags alphabetically instead of in rule-declaration order. Consistent with the top-level tags list, which is already alphabetical.

## Fix 2 — report subtitle source order

**Where:** `run.py`, "Collect source names for the report subtitle".

```python
source_names = list(set(s.get('name', 'Unknown') for s in data_sources if not s.get('_supplemental', False)))
```

**Change:** dedupe while preserving the order sources are declared in `settings.yaml`.

```python
source_names = list(dict.fromkeys(
    s.get('name', 'Unknown') for s in data_sources if not s.get('_supplemental', False)
))
```

Sorting would also be deterministic, but declaration order is what a reader expects in the subtitle.

## Verify

Determinism check — regenerate twice into scratch and compare hashes. Never point `-o` at a real output dir for this; it would rewrite a live baseline.

```powershell
$s = "$env:TEMP\tally-determinism"
New-Item -ItemType Directory -Force -Path "$s\a","$s\b" | Out-Null
uv run tally up --quiet -o "$s\a\r.html" -c <config-dir>
uv run tally up --quiet -o "$s\b\r.html" -c <config-dir>
(Get-FileHash "$s\a\r.json").Hash -eq (Get-FileHash "$s\b\r.json").Hash   # must be True
```

Each run must be a **separate process** — Python randomizes the string hash seed per process, so two calls inside one interpreter will agree even while the bug is present. Before the fix this returns `False`; equal file sizes with differing hashes is the signature.

To locate a mismatch, find the first differing character offset and print ~90 chars either side; the difference will appear inside a `"pattern": { … "tags": [...] }` block.

## Test

Add to `tests/test_analyzer.py`:

- Build a merchant whose `match_info['tags']` is a list in deliberately non-alphabetical order; assert the exported `pattern.tags` comes back sorted.
- Assert `pattern.tags` is sorted when `match_info['tags']` is passed as a `set` too (the path the dead guard used to cover).
- For `run.py`: given data sources declared `B`, `A`, `B`, assert `source_names == ['B', 'A']` — dedupe without reordering.

A subprocess-based two-run hash comparison would be the strongest test but is slow and environment-sensitive; the unit tests above cover the actual defects.

## Out of scope

- `analyzer.py` top-level merchant `tags` — already sorted, leave alone.
- `infer_annual_amount()` recurrence threshold — separate finding, deferred to the fixed-annotation branch.
