> **Stacked on [`feature/merchant-composite-keys`](https://github.com/terryaney/OpenSource.tally/tree/feature/merchant-composite-keys) — PR #TBD.** Please merge that one first (and the four it sits on). This is the top of the stack.
> **This layer only:** [`feature/merchant-composite-keys...feature/categorization`](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization)

This PR adds a second, optional way to answer "what is this merchant?" — an editable review file that `tally up` generates alongside the HTML report — plus a `review:` rule flag and a data-file register for closing the loop on files you have already checked.

Neither workflow replaces the other:

- **Prose (unchanged).** Tell an agent "all VIOC is oil change" or "these three are groceries." Still the right choice when a decision needs judgment.
- **File (new).** Answer in an editor at your own pace, with autocomplete driven by your own rules; the agent reads the file back and applies it.

The two mix freely. Set `generate_categorization_file: false` to switch the file path off entirely.

## At a Glance

- `config/categorization.yaml` — one row per uncategorized transaction; fill in `useRule`, `newRule`, or edits
- `config/categorization.hints.yaml` — deterministic match data (difflib against your own rules; no AI, no network), regenerated every run, safe to delete
- `categorization-schema.json` — drives editor autocomplete and hover
- New `review: true` flag on a merchant rule, for categorizations you want to eyeball rather than trust
- New `config/inventory.yaml` — tally's register of data files it has parsed, with a hand-set `reviewComplete` to close a file out

## Phase 1 — the review file

On re-run, rows that now match a rule drop out, rows still unknown keep your answers, and `id` renumbers. **Answers reattach by a stable key, never by id.** That key is a sha1 of source + date + amount + raw description, truncated to 8 chars, with an ordinal suffix for exact duplicates.

Malformed YAML hard-fails with line and column, leaves your file untouched, and the HTML report is still written.

**Behavior delta on `tally up`:** it now writes three files and prints one status line when unknowns exist, and can exit non-zero on malformed YAML where it previously always exited 0.

`tally discover`, the HTML report, JSON/CSV/markdown output, analysis, and rule matching are all unchanged. `merchant_engine.py` is untouched in this phase.

## Phase 2 — `review:`, and confirming a file

### The rule flag

`MerchantRule` gains an optional `review: bool = False`. The parser accepts only `review: true` or `review: false` — a conditional form like `review: amount > 500` is a parse error that points you at putting the condition in the rule's own `match:` and flagging that rule instead.

`match_info['review']` is set when **any** matching rule carries the flag, in both the categorization branch and the tag-only branch, so a tag-only rule can request review just as a categorization rule can.

Risk, stated plainly:

- The parser's hard-failure on unknown properties is **unchanged, deliberately**. A test asserts that `reviewed:` — a plausible typo — still raises "Unknown property", proving the parser was not loosened to get this in.
- `review` defaults to `False`, so every existing `merchants.rules` behaves identically. `tests/test_rule_snapshots.py` passes.
- The new `match_info` key cannot reach the report. `analyzer.py` and `report.py` both read `match_info` through explicit `.get()` calls into freshly-built dicts; verified empirically that `review` appears zero times in generated report JSON and HTML.

> **Compatibility note for release notes:** a `merchants.rules` containing `review:` will **not** load on tally builds older than this release, because the parser hard-fails on unknown properties. This is by design and was accepted.

### The inventory

`config/inventory.yaml` holds exactly four keys per entry: `path`, `source`, `registered`, `reviewComplete`. Tally auto-registers files it parses and **only ever appends** — it never modifies or removes an existing entry, so a hand-set `reviewComplete` always survives. Unknown keys are rejected outright, since a typo would otherwise silently lose the flag. A non-boolean `reviewComplete` is a hard error. Malformed YAML hard-fails, leaves the file untouched, and the report is still written.

`reviewComplete` is a **boolean, not a date**, because the only question ever asked of it is "is this done?" — and a bare `2026-08-02` loads as a YAML date object while `"2026-08-02"` loads as a string, so a hand-typed value would behave differently depending on quoting. `registered` stays a date, since tally writes it and the user never does.

Registration runs **unconditionally** on the HTML path, independent of whether the review file is generated. `registered` records when tally first saw a file, so it has to keep advancing even while review generation is off — otherwise re-enabling would backdate every file seen in the meantime to that day and lose the real first-seen date. Files registered while off carry `reviewComplete: false` and surface once review resumes.

There is deliberately **no tally command to confirm a file** — you or your agent edit `config/inventory.yaml` directly. Set `reviewComplete: false` to review a file again after appending rows to its CSV.

### Review rows

`categorization.yaml` gains a top-level `reviews:` list — transactions matched by a `review:`-flagged rule, carrying `currently` (how it is categorized right now, i.e. what stands if you do nothing) and `file` (which data file to confirm). Ids continue the same sequence as `unknowns:`, so a row number is never ambiguous across the two lists.

The status line gains the review count:

```
Categorization: 36 unknown (0 new, 36 carried forward), 2 awaiting review — confirm to close
```

**`reviewComplete` is review-scoped only and never gates parsing or analysis.** Every transaction always feeds report aggregates and `tally discover`; confirming a file only stops its review rows appearing. A test asserts spending totals are identical before and after confirmation, and that the transaction list is never filtered.

Review rows persist indefinitely — across months and files — until confirmed. Leaving a row untouched means its existing rule stands.

### Stale-file warning

Previously, setting the generation flag to false left generated files on disk silently. Now tally prints:

```
Categorization: generation is off (generate_categorization_file: false)
  <path>/categorization.yaml is STALE - written <timestamp>, not updated since.
  Ignore its contents, or delete it. Re-enable to refresh.
```

It never deletes. `categorization.yaml` holds answers you may have typed, and a config toggle must not destroy data. Reading it is read-only and never fails the run, even against a malformed or missing file.

## Naming

Two renames, both on unreleased surface, so no migration or compatibility shim exists — nothing on disk can carry the old names.

- **`categorization` → `generate_categorization_file`.** The old name read as "stop categorizing my transactions," which is tally's entire purpose. `generate_` marks it a boolean, `_file` marks what it governs, and it avoids implying a path the way a bare `categorization_file` would, since `merchants_file` / `views_file` in this codebase take paths.
- **`additionalInfo` → `aiNotes`**, in both `unknowns:` and `reviews:` rows. A blank `additionalInfo` read as an unfilled obligation the user still owed; `aiNotes` states ownership, so a blank one means "the agent hasn't written anything here." Semantics unchanged: agent-owned, filled on request, tally never writes or overwrites it, preserved across regenerations.

## Known constraints, accepted

- Because tally registers files during `tally up`, running `tally up` before the file-validation step makes a file look already-registered. Recovery is to delete the entry from `config/inventory.yaml`.
- `inventory.yaml` is hand-editable, but PyYAML does not preserve comments, so comments added to it are lost on rewrite. `ruamel.yaml` was deliberately not added as a dependency for this.
- A bug worth recording from Phase 2: parsing the `review:` property was not sufficient. `MerchantRule` is constructed with explicit keyword arguments, so the value was silently dropped until `_add_rule` was updated to pass it through — it parsed cleanly and did nothing. A test caught it.

## Testing

1019 tests pass, including `tests/test_rule_snapshots.py`. No test predating this feature was modified. The only test files touched are the ones belonging to it: `test_categorization.py`, `test_categorization_schema.py`, `test_categorization_review.py`, and `test_inventory.py`.

`rule_cache.py` was left alone; it is referenced only by its own test, so no production path reconstructs rules from cache where a new property could be silently dropped.

No tracked issue clearly matches this feature — the open issue list was checked from #88 down to #27 and has nothing about review, inventory, confirming categorizations, or the settings rename, so none is referenced here.

## Coming next (not in this PR)

Nothing — this is the top of the stack.

Separately, [`feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair) — PR #TBD — is independent of this stack: `.github/workflows/` only.
