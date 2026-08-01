# Tally Reconciliation Workflow

## Context

Reconciling 1–30 uncategorized transactions per quarter is a permanent part of using tally — it
will never go away. Today that happens in chat: tally presents a batch of ~20 unknowns, the user
answers in free-form prose, and the agent applies the answers. The workflow has three costs:

1. **Scrolling.** After any back-and-forth, the transaction table scrolls out of view. The user
   ping-pongs through terminal history to re-read rows, and can't ask a quick side question
   without losing their place.
2. **Token bleed.** The table is re-dumped into context every batch, every cycle.
3. **Generation cost.** The review file and its JSON schema are currently produced *by the AI*,
   which is slow, expensive, and unreliable — the schema came out wrong.

The fix is a file-based review pass, structurally like `git rebase -i`: tally writes a YAML file
of unknown transactions plus a JSON Schema that drives editor autocomplete; the user fills in
answers at their own pace in an editor; the agent reads the file back and applies them. The
terminal stops being the interface.

**Hard constraint:** must work identically across Copilot, Claude, and Codex CLIs *and* their VS
Code extensions — six harness flavors. No harness-specific tooling anywhere in the design.

**Governing principle:** *Tally and skills cannot know about each other; they may not exist
together.* Tally must be fully functional with no skill installed, and no skill may ever be a
prerequisite for a tally feature. Dependency is one-way: a skill may read tally's files; tally may
never read a skill's files.

### Reference files (external to this repo)
- `C:\BTR\TallySpending\tally\config\categorization.yaml` — hand-built sample of the target shape
- `C:\BTR\TallySpending\tally\config\categorization-schema.json` — 1,747-line AI-built schema
- `C:\BTR\TallySpending\.claude\skills\tally-categorize\SKILL.md` — skill to be rewritten
- `C:\BTR\TallySpending\.claude\skills\tally-files\SKILL.md` — skill, minor change only

---

## Verified codebase facts

Confirmed by reading the code; several shaped the design:

- **Tally has no rules-file writer, and can't cheaply gain one.** The only writes are whole-file:
  `cli_utils.py:113-118` (init scaffold) and `migrations.py:128-152` (one-time CSV→`.rules`).
  `MerchantEngine.parse` (`merchant_engine.py:165`) discards every comment and blank line
  (`:179-181`). The `# === Food ===` section layout and `# Category Override` conventions exist
  only as prose in SKILL.md — tally's data model has no concept of them.
- **Unknown rule properties hard-crash the parser.** `merchant_engine.py:290-292` raises
  `MerchantParseError("Unknown property")`. **This behavior stays unchanged** (see Phase 2).
- **Identity plumbing exists but is discarded.** `_iter_rows_with_delimiter` (`parsers.py:156`)
  yields `line_num`; `parse_generic_csv` (`parsers.py:210`) has `filepath`. Both feed only
  `SkippedRow` (`parsers.py:17-21`).
- **`{tagging}`/`{memo}` are user-specific**, not tally concepts — custom captures in the user's
  `settings.yaml` `format:` string. Tally must never assume they exist.
- **PyYAML is already a dependency** (`pyproject.toml:24`); `MarkedYAMLError` exposes
  `problem_mark.line`/`.column` plus `problem` and `context`.

### Measurement (real data, 2026-07-31)
`tally discover --format json --limit 0` against `C:\BTR\TallySpending\tally\config`:
37 unknowns → 31 unique raw descriptions, distribution `{1: 30, 7: 1}`. 28 of 31 are Amazon with
unique `AMAZON MKTPL*<order-id>` suffixes. **Grouping by description would shrink a batch ~16%
and ~0% of the Amazon bulk** — batch size is irreducible. The file is the fix, not grouping.

---

## Design decisions

**Division of labor.** Tally *generates* `categorization.yaml` + `categorization-schema.json`.
The agent *applies* — edits `merchants.rules`, writes `CATEGORY:`/`TAG:` into the user's CSV
tagging column. Tally gets no rules-file writer. `newRule:` free-form prose is purely agent-side;
tally emits the field and validates nothing.

**File shape.** One row per transaction, flat top-level `unknowns:` list, **no grouping** —
*the AI is the collapsing mechanism, not tally.* Natural-language batching ("all VIOC is oil
change across all sources") beats tally-side grouping because it works across *different*
descriptions and needs no split hatch. `source:` is a display field, not a nesting level. Sort by
source ascending, then date descending. `id:` is an ephemeral display label, renumbered every
regeneration — "process 7, 9, 13" always means the file currently on screen.

**Hints vs annotations.** Tally always emits deterministic `hints:` — fuzzy nearest existing
rules with scores, prior-period categorization of the same description, occurrence count, amount
spread, refund flag. No LLM. `additionalInfo:` is a *separate* agent-owned field, filled on
demand only ("annotate"), and preserved across regenerations. The default path costs zero tokens
and the whole feature works with no AI at all.

**Merge, not overwrite — read once, at the end.** `tally up` runs normally and writes the report,
then generation reads the existing `categorization.yaml`. Rows whose transactions now match a
rule are dropped; rows still unknown carry forward their answers and `additionalInfo` verbatim;
new unknowns append. There is no early parse — failing fast buys nothing when the report is
written regardless, so there is exactly one read point.

Identity = `sha1(source + date + amount + raw_description)` + occurrence ordinal, deliberately
**excluding tagging/memo** so the agent writing `CATEGORY:` doesn't change a row's identity. Each
row carries a machine-owned `key:`. Failed applies are detected via a `generated:` header
timestamp compared against source/rules-file mtimes — no per-row stamp, no writer needed, works
on the zero-AI path.

**Failure behavior.** Malformed YAML → hard-fail. The report is already written by then; emit
full `MarkedYAMLError` detail (line, column, `problem`, `context`, and what to do), leave the file
completely untouched, exit non-zero. Never back up, move, or regenerate — forcing the user to
merge two files is worse than fixing the syntax.

**Trigger surface.** Generation defaults ON when `merchants_file` is set and unknowns > 0;
switchable off. Writes nothing when unknowns are zero. Tally prints one status line with a
breakdown, because the `⚠ still unknown after apply` count is how failed-apply detection reaches
the user without opening anything.

> **No existing tally behavior changes.** `tally discover` keeps its current table output exactly
> as-is. The constraint is only that this feature *adds* no new table/listing output and no new
> listing command — if the user wants rows dumped in chat 15 at a time, the agent reads the YAML
> and renders them.

**Review (Phase 2).** `review: true` is a bare boolean on a rule — no conditions; granularity
lives in the rule's own `match:` (a Best-Buy-in-December rule carries the flag, the general Best
Buy rule doesn't). Review rows surface for transactions in files not yet stamped `reviewedOn`,
and **persist indefinitely** — months and files later — until stamped. Resolution is file-level:
leave a review row untouched and its existing rule stands.

---

## Phase 1 — Core loop

### 1a. Transaction identity
`src/tally/parsers.py` — surface `source_name`, `filepath`, and `raw_description` onto transaction
dicts at construction (data already available at `parsers.py:249`). Add a helper computing
`sha1(source + date + amount + raw_description)` truncated to 8 chars, plus an occurrence ordinal
for exact duplicates. Excludes tagging/memo by construction.

### 1b. Generator — new module `src/tally/categorization.py`
Largely a re-shaping of `cmd_discover`, not new analysis. **Reuse, don't reinvent:**
- `commands/discover.py:102-113` — per-description `count`, `total`, `has_negative`, examples
- `commands/discover.py:134-152` — `suggest_pattern`, `suggest_merchant_name`,
  `suggest_merchants_rule`
- `commands/explain.py:212` — `get_close_matches` for nearest-rule hints (follow this pattern
  rather than `expr_parser.py:315`'s `SequenceMatcher`, which is the `fuzzy()` operator)
- `merchant_utils.get_all_rules` / `MerchantEngine.rules` — name/category/subcategory/tags for
  the schema enums

Responsibilities: read existing YAML → merge by key → emit YAML + schema → return status counts.

### 1c. Schema generation
`$defs` + `$ref` for the `useRule` enum (currently duplicated verbatim, accounting for most of
1,747 lines). Drop the `""` enum entry — `null` already covers an empty value. `description` on
**every** property so Ctrl+Space shows a popup; the current `tags` popup is the quality bar.
`tags` becomes an **array with `items.enum`** — this eliminates the "yellow squiggle expected"
caveat entirely. Navigation comment block rewritten to: copy `useRule: ` → Ctrl+F → F3 between
items → paste to replace → Ctrl+Space. **The header line must change** — "do not hand-edit" is
now backwards, since hand-editing is the interface; replace with a machine-owned-fields note.

### 1d. Wiring
- `src/tally/commands/run.py` — after the HTML is written, run generation.
- `src/tally/config_loader.py` + `config/settings.yaml.example` — the on/off setting.
- `src/tally/cli.py` — AGENTS.md template update (required by CLAUDE.md).

### 1e. Skill rewrite (`tally-categorize`, external repo)
- **Phrasebook section, verbatim and quotable** — first-class, not a footnote, because the user is
  deliberately trading file structure for conversational instruction:
  `"all VIOC is oil change across all sources, process rest of my answers"` ·
  `"treat 5-10 as Health & Fitness / Tennis, I've answered the rest"` · `"process file"` ·
  `"process 1-5"` · `"annotate"` · `"I've reviewed all, they can stay as matched"`
- Surface `annotate` proactively (e.g. "N rows have weak hints — say 'annotate'")
- Drop schema/YAML generation steps entirely — tally owns those now
- Guarded editor open: `if TERM_PROGRAM == "vscode"` → `code -r <path>`, else print the path.
  Environment check, not harness check — works in all six flavors. **The skill must never depend
  on the file being open.**
- Remove the "yellow squiggle expected" line (fixed by `items.enum`)

---

## Phase 2 — Review + inventory

### `review: true`
New optional bare-boolean rule property, parsed and exposed on `MerchantRule`.

**The parser's hard-failure on unknown properties stays unchanged** — no tolerant-parse change,
no `merchant_engine.py` behavior modification beyond recognizing one new property name. Consequence
to document in release notes: a `merchants.rules` file containing `review:` will not load on tally
builds older than this release. Accepted. `tests/test_rule_snapshots.py` must still pass.

### `config/inventory.yaml` — tally-owned
```yaml
files:
  - path: data/amazon-chase-visa-2026-Q2.CSV
    source: Amazon Chase Visa
    registered: 2026-07-31
    reviewedOn: 2026-08-02
```
That is the whole schema — no unknown-key tolerance, no extra fields. Tally auto-registers any
data file it discovers that isn't already listed, and only ever appends entries. Hand-editable by
design (clearing `reviewedOn` forces reprocessing after a CSV append), so it gets the same
hard-fail contract as `categorization.yaml`. Documented limitation: PyYAML does not preserve
comments; `ruamel.yaml` deliberately not added.

**`reviewedOn` is review-scoped only.** It must never gate parsing or analysis — every transaction
feeds `analyze_spending`'s aggregates (`analyzer.py:86-137`), so skipping stamped files would
silently corrupt totals and monthly averages and blind `tally discover`.

### Who stamps `reviewedOn`
The agent or the user, by editing `config/inventory.yaml` directly. **No new tally command.** The
user reviews the file, and if nothing needs changing tells the agent "I've reviewed all, they can
stay as matched" — the agent writes the date. Until then review rows keep appearing, indefinitely.

When a file has zero unknowns but did surface review rows, the status line says so explicitly:
```
Categorization: 0 unknown, 3 awaiting review — confirm to close
  config/categorization.yaml
```

### `tally-files` skill — one-line change
Its newness test stays "file not present in the inventory." It now reads `config/inventory.yaml`
instead of `tally/data/inventory.yaml`, and no longer writes rows (tally registers them).

> **Known ordering constraint, accepted by the user.** Because tally registers files on `tally up`,
> running `tally up` *before* the files skill makes that file look already-registered and it will
> never be validated. The workflow is: download → files skill → categorize skill (which runs
> `tally up`). Recovery is to hand-delete the entry. The user has an instruction file that blocks
> ad-hoc `tally` usage outside the skills.

---

## Verification

**Tests** (`tests/test_analyzer.py`, plus a new `tests/test_categorization.py`):
- Merge preserves un-applied answers and `additionalInfo` across regeneration
- Applied rows drop out; new unknowns append; `id:` renumbers
- Identity is stable when tagging/memo change, distinct for true duplicates (ordinal)
- Malformed YAML: file untouched, HTML still written, non-zero exit, error names line/column
- Zero unknowns writes no file; disabled setting prints nothing
- Schema: `$ref` resolves, no `""` in enums, every property has `description`
- Phase 2: `review: true` parses; `reviewedOn` gates review rows but never aggregates; review rows
  persist across multiple `tally up` runs until stamped
- **`tests/test_rule_snapshots.py` must pass** before any `merchant_engine.py` commit

**End-to-end** against `C:\BTR\TallySpending` (37 known unknowns, 31 unique descriptions):
```
uv run tally up C:\BTR\TallySpending\tally\config
```
Confirm the status line, then open `config/categorization.yaml` — verify schema autocomplete on
`useRule` (Ctrl+Space), array autocomplete on `tags` with no squiggle, and hover popups on every
field. Fill 5 rows, apply, re-run, confirm those 5 are gone and the other 32 retain their answers.
Corrupt the YAML deliberately and confirm the report is still written and the file untouched.

**Report changes** — per CLAUDE.md, use Playwright MCP to verify the HTML report is unaffected.
