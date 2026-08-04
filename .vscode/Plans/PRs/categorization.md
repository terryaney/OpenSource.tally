Title: Categorization review file, review flag, and file inventory [6/6]

> **[6/6]** Based on `feature/merchant-composite-keys` (#102). Top of the stack.
> Merging this PR into `main` also lands #98, #99, #100, #101, and #102 if not already merged.
> **This layer only:** [`feature/merchant-composite-keys...feature/categorization`](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization)

This PR adds a second, optional way to answer "what is this merchant?" — an editable review file that `tally up` generates alongside the HTML report — plus a `review:` rule flag and a data-file register for closing the loop on files you have already checked. 

**This works best with a AI skill.  I've included a sample skill file at bottom of this description.**

- **Prose (unchanged).** Tell an agent "all VIOC is oil change." Still the right choice when a decision needs judgment.
- **File (new).** Answer in an editor at your own pace, with autocomplete driven by your own rules to eliminate AI cycle/token burn.

Set `generate_categorization_file: false` to switch the file path off entirely.

## What it looks like

**Rule and Memo**
<img width="1161" height="460" alt="tally-memo" src="https://github.com/user-attachments/assets/97b80f13-d951-4713-8d7d-1dcdb060a6a0" />

**Creating a new Category Tagging Rule**

Note: I have rules in my `merchant.rules` file:

```
is_amazon = contains("Amazon.com*") or contains("AMAZON MKTPL*") or contains("AMAZON MKTPLACE PMTS")

[Amazon]
match: is_amazon and contains(field.tagging, "CATEGORY: Health / Supplements")
category: Health & Fitness
subcategory: Supplements
```

<img width="1161" height="460" alt="tally-newCategoryRule" src="https://github.com/user-attachments/assets/c9c05ade-94fd-407a-95b4-7037e8dab3b1" />

**Use Rule Suggestion**
<img width="1161" height="460" alt="tally-useRule" src="https://github.com/user-attachments/assets/45dbe4d4-4076-4ca1-9d11-f3075da2967c" />

## Changes

**Review file** — `config/categorization.yaml` lists one row per uncategorized transaction. Fill in `useRule`, `newRule`, or edits. On re-run, matched rows drop out, unknown rows keep your answers. Answers reattach by a stable key (sha1 of source + date + amount + raw description), never by id. Malformed YAML hard-fails with line and column, leaves your file untouched, and the HTML report is still written.

**Hints file** — `config/categorization.hints.yaml` provides deterministic match suggestions (difflib against your rules; no AI, no network). Regenerated every run, safe to delete.

**Schema** — `categorization-schema.json` drives editor autocomplete and hover.

**`review: true` rule flag** — marks a rule whose categorizations you want to eyeball. Matched transactions appear in a `reviews:` list in `categorization.yaml` with their current categorization. Setting `reviewComplete` on the data file in `inventory.yaml` closes them out.

**Data inventory** — `config/inventory.yaml` auto-registers every data file tally parses. Set `reviewComplete: true` to stop review rows appearing for that file. Tally only ever appends entries, never modifies or removes them.

## Behavior changes on `tally up`

- Writes three files and prints one status line when unknowns exist
- **Can now exit non-zero on malformed `categorization.yaml` or `inventory.yaml`, where it previously always exited 0**
- With `generate_categorization_file: false`, prints a STALE warning naming the file and when it was written, rather than leaving generated files on disk silently. It never deletes them — the file holds answers you may have typed, and a config toggle must not destroy data
- `reviewComplete` is **review-scoped only and never gates parsing or analysis**. Every transaction always feeds report aggregates and `tally discover`; confirming a file only stops its review rows appearing

`tally discover`, the HTML report, JSON/CSV/markdown output, analysis, and rule matching are otherwise unchanged.

## Compatibility

- `review:` in `merchants.rules` will not load on older tally builds (parser hard-fails on unknown properties — by design)
- `review` defaults to `false`, so every existing `merchants.rules` behaves identically
- `tests/test_rule_snapshots.py` passes unchanged

## Testing

1019 tests pass. No pre-existing test was modified. New test files: `test_categorization.py`, `test_categorization_schema.py`, `test_categorization_review.py`, `test_inventory.py`.

## Stack

| # | Branch | PR | Description | Diff |
|---|---|---|---|---|
| 1 | `feature/globbing-documentation` | #98 | Glob pattern docs and CLI tests | [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/globbing-documentation) |
| 2 | `feature/report-json-determinism` | #99 | Deterministic report JSON | [from PR98](https://github.com/terryaney/OpenSource.tally/compare/feature/globbing-documentation...feature/report-json-determinism) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/report-json-determinism) |
| 3 | `feature/ui-tweaks` | #100 | Date filter, transaction details, per-txn tags | [from PR99](https://github.com/terryaney/OpenSource.tally/compare/feature/report-json-determinism...feature/ui-tweaks) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ui-tweaks) |
| 4 | `feature/charts-reimagined` | #101 | Reimagined charts and KPIs | [from PR100](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/charts-reimagined) |
| 5 | `feature/merchant-composite-keys` | #102 | Composite merchant keys | [from PR101](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/merchant-composite-keys) |
| **6** | **`feature/categorization`** | **#103** | **Categorization review file** | **[from PR102](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization) · [from main](https://github.com/terryaney/OpenSource.tally/compare/main...feature/categorization)** |

Independent: [`feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair) — #97 — fixes fork PR builds.

## Sample Skill File

Note: Still iterating over size and doing prompt testing.

````
---
name: 'tally-categorize'
description: 'Run tally, review the categorization.yaml file it generates for unknown transactions, apply answers, and loop until 0 unknowns remain.Run tally, review the categorization.yaml file it generates for unknown transactions, apply answers, and loop until 0 unknowns remain.'
---

You are categorizing unknown transactions and managing merchant rules. Tally generates the review
file; you apply it. Run the report, open the review file, interpret the user's natural-language
instructions, apply them to `merchants.rules` and the CSV tagging column, and loop until 0 unknowns
remain.

## Files and division of labor

Tally generates `categorization.yaml`, `categorization.hints.yaml`, and
`categorization-schema.json` as part of `tally up`, whenever `merchants_file` is set and unknowns
exist. You never build any of them — that used to be this skill's job and it isn't anymore. Tally
has no rules-file writer and never will: `merchants.rules` edits and CSV tagging-column edits are
entirely yours.

- **`categorization.yaml`** — the answer file, the one you and the user edit. Top-level `unknowns:`
  and `reviews:` lists.
- **`categorization.hints.yaml`** — read-only deterministic match data, regenerated every run and
  safe to delete. Correlate an entry to an answer row by `key` (stable across runs — prefer this)
  or `id`. If the file is missing, degrade gracefully: treat every row as having no `nearest` entry
  rather than erroring.
- Both open with a `state:` block (`generated`, `totalSources`, `totalUnknowns`, `totalReviews`).

**`id:` is a display label**, renumbered from 1 on every regeneration; `key:` is the stable
identity but not what users say out loud. "Process 7, 9, 13" always means the file **currently on
screen** — re-read before acting on any id-based instruction, even if you just wrote to the file.
`reviews:` continues the same id sequence as `unknowns:`, so a row number refers to exactly one row
across both lists.

## Workflow

1. Run `tally up`. If `settings.yaml` has `generate_categorization_file: false`, or tally prints a
   "generation is off" / STALE warning, `categorization.yaml` is stale — do not read or act on it.
   Fall back to the prose workflow instead: `tally discover`, reading `merchants.rules`, and
   conversation with the user.
2. Read tally's status line: the unknown/carried-forward/dropped breakdown, an "awaiting review"
   count when `reviews:` is non-empty, and a "still unknown after apply" count if a previous apply
   pass silently failed. 0 unknown **and** 0 awaiting review → skip to step 7. If unknowns are 0 but
   review rows remain, there is still work waiting.
3. Open `categorization.yaml` — tally prints its path, and the companions sit next to it. Check the
   `TERM_PROGRAM` environment variable (an environment check, not a harness check, so it behaves
   identically under Claude, Copilot, or Codex, terminal or VS Code extension): if it is `vscode`,
   open with `code -r <path>` to reuse the window; otherwise print the path and let the user open
   it. Never block on the file being open, and never assume it stayed open.


   **Editor note (first open only):** Mention that editing in VS Code with the
   [YAML extension by Red Hat](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml)
   is recommended for Ctrl+Space autocomplete on `useRule` values, but any editor with YAML
   language support will work.
4. Read the file once, then immediately run the **Policy** pass below — it applies to `unknowns:`
   rows only, never `reviews:` rows. Tell the user how many unknown rows are waiting, how many
   review rows await confirmation, how many you auto-applied, and how many you annotated. Do not
   dump rows into chat unless asked; if asked ("show me the next 15"), read the file and render the
   slice yourself. Tally will never gain a listing command for this, and `tally discover`'s table is
   a different, tally-owned output that this feature does not extend.
5. Wait for the user — typed answers in the file, a chat instruction, or both. See **Phrasebook**.
6. Re-read the YAML fresh, apply what was asked (see **Applying answers**), then go to step 1.
7. If `merchants.rules` was modified this session, regenerate the all-rules report via
   `/tally-rules`. Then open the HTML report in the browser for the user.

## Policy: auto-apply vs. annotate thresholds

**These numbers are yours to tune — edit them, not the prose around them.**

```
AUTO_APPLY_MIN_SCORE = 0.95   # nearest[0].score must be at least this
AUTO_APPLY_MAX_RULES = 1      # nearest[0].rules must equal this (unambiguous merchant)
```

For every unresolved `unknowns:` row, look at `nearest` in `categorization.hints.yaml` (matched by
`key`) and pick exactly one outcome:

1. **Auto-apply** — `nearest[0].score >= AUTO_APPLY_MIN_SCORE` AND `nearest[0].rules ==
   AUTO_APPLY_MAX_RULES`. That entry's `useRule` value is unambiguous (one merchant, one variant);
   apply it immediately, exactly as you would a user-filled `useRule`, without waiting to be asked.
2. **Annotate** — the row is not auto-appliable AND you can say something the user cannot already
   see. Write it into `aiNotes`. Apply the value test below before writing anything.
3. **Leave alone** — everything else, including every row where you have nothing to add. Don't
   touch `aiNotes`, don't apply anything.

**Never auto-apply on `score` alone.** `score: 1.0` with `rules: 25` (e.g. Amazon) means the
merchant name matched verbatim and tells you nothing about the category. Both terms, always.

### The value test for `aiNotes`

**A blank `aiNotes` is a good outcome. Never fill it for coverage.** The user reads this file row by
row; a note that tells them nothing costs them attention and buries the notes that matter.

Before writing, ask: *does this say anything the user could not get by reading the row and its
hints?* If no, leave it blank.

- **Never restate the hints.** `"Amazon has 25 category-specific rules; purchase details needed"` is
  just `rules: 25` in prose. So is "no prior-period match", "seen 3 times", "this is a refund". The
  hints file already says all of that, and the user can read it.
- **Do write** when you bring in something the data doesn't carry: what an opaque merchant name
  actually is (`"VIOC is Valvoline Instant Oil Change"`), a reading of the `memo`/tagging text, a
  pattern across rows (`"ids 4, 7, 9 look like one order split across three charges"`), or a genuine
  category guess with a reason that isn't already on screen.
- **When the answer isn't in the data, say nothing.** Amazon order IDs are opaque — no amount of
  reasoning reveals what was bought. That is not a row awaiting a better guess; it is a row only the
  user can answer.
- **Report the pattern once in chat, not N times in the file.** "24 Amazon rows — the category
  depends on the item and isn't in the data; tell me what they were" is useful once and noise
  twenty-four times.

If a pass produces no notes at all, that is a correct result — say so and move on.

Run this pass automatically after reading a freshly generated file (workflow step 4), not only when
asked. Tally never writes `aiNotes` and never overwrites yours — it survives regeneration
until the row is resolved.

## Phrasebook

The user is trading file structure for conversational instruction on purpose — this is the primary
interface, not a fallback. The AI is the collapsing mechanism, not tally: natural-language batching
has to work across *different* descriptions of the same intent, not just the examples below. Treat
these as patterns to generalize from, not a fixed command grammar.

- **"all VIOC is oil change across all sources, process rest of my answers"** — two instructions in
  one. First, find every row whose merchant matches "VIOC" regardless of its `source:` field and
  apply the oil-change categorization (an existing rule via `useRule` if one fits, otherwise
  `edits.category`) to all of them — "across all sources" means don't scope the match to one
  `source:`. Second, "process rest of my answers" means also apply every other row that already has
  an answer filled in, exactly like "process file".
- **"treat 5-10 as Health & Fitness / Tennis, I've answered the rest"** — apply
  `edits.category: Health & Fitness / Tennis` directly to ids 5 through 10 yourself (the user is
  telling you the answer in chat, not asking you to type it into the file), then process every other
  row the user has already filled in, same as "process file".
- **"process file"** — apply every row that has any answer filled in (`useRule`, `newRule`, or any
  `edits.*` field). Entirely blank rows are skipped and reappear next generation.
- **"process 1-5"** — apply only ids 1 through 5. Leave every other row untouched for this pass,
  even ones with answers already filled in — the user is explicitly scoping this batch.
- **"annotate"** — re-run or widen the annotate step from **Policy** on demand, e.g. after editing
  the thresholds, or to cover rows the default pass skipped (`"annotate 5-12"`, `"annotate the
  Amazons"`). It never applies or processes anything; it only fills `aiNotes`.
- **"I've reviewed all, they can stay as matched"** — see **Review rows**. Set `reviewComplete:
  true` in `tally/config/inventory.yaml` for every file currently surfacing a review row, not just one.

## Review rows

Some transactions aren't unknown — they matched a rule the user flagged with `review: true` (a
narrowly-scoped rule, e.g. "Best Buy in December", not the merchant's general rule) — but the file
they came from hasn't been confirmed yet. They surface in the `reviews:` list, carrying `id`, `key`,
`source`, `date`, `merchant`, `amount`, `currently` (how the matched rule categorizes it now), and
`file` (the data-file path to mark complete), plus the same answer fields as an unknown row.

- **Leaving a review row untouched confirms the existing rule.** There is nothing to "apply" — the
  rule already matched and already wrote the categorization. The row is only asking for a look.
- **To change it instead**, the user fills in `useRule`/`newRule`/`edits` exactly like an unknown
  row, and you apply it the same way.
- **Resolution is file-level and indefinite.** A review row keeps reappearing — across months and
  across runs — until someone sets `reviewComplete: true` for that row's `file` in
  `tally/config/inventory.yaml`. Edit that file directly; there is no tally command for this, so do not
  invent one. To close out "I've reviewed all", collect the distinct `file` values across the
  current `reviews:` list and mark every one of them complete.

## Applying answers

For each row with any answer filled in:

| Field | Action |
|---|---|
| `useRule` only | Find that rule in `merchants.rules`. Update its `match:` expression minimally to also match this transaction (add an `or contains(...)` clause). If the rule's match checks `field.tagging` for a `CATEGORY:` or `TAG:` pattern, auto-add that tagging to the CSV row. |
| `newRule` only | Free-form prose — tally emits this field and validates nothing. Parse the instruction yourself and create the rule per **Rule Insertion Logic**. |
| `useRule` **and** `newRule` | Not a conflict — two different axes. `useRule` says *which rule*; `newRule` says *what to do with it*. Picking `useRule` from autocomplete is how the user names a rule unambiguously, so never discard it and re-derive the target from the prose. The prose governs the action: "add to this rule" means widen its `match:`, "also tag it x" means add the tag, and so on. |
| Genuinely contradictory | If the prose asks for something incompatible with the named rule (e.g. `useRule: [Amazon] Shopping / Books` plus "actually make this a new merchant"), **ask** — do not pick one. Silently guessing here is how `merchants.rules` gets quietly wrong. |
| `edits.category` | Add `CATEGORY: X / Y` to the CSV tagging column for that transaction's row. |
| `edits.tags` | For each tag: add `TAG: x` to the CSV tagging column. If the tag doesn't exist as a tag-only rule and isn't within edit-distance 2 of an existing tag, create a new tag-only rule. If within edit-distance 2, warn the user about a possible typo first. Tally emits this field as `[]`, not blank, on a fresh row — treat `[]` as no tags. |
| `edits.memo` | Add text to the CSV memo column for that transaction's row. |
| All blank | Skip — the row reappears next generation, carrying forward untouched. |

Multiple fields can be filled simultaneously (e.g. `newRule` + `edits.category`). Never write to
either file's machine-owned fields.

## Tagging Column Semantics

CSV data files use a `{tagging}` column (separate from `{memo}`) to hold annotation directives that
drive rule matching. The tagging column is never displayed in the report.

- Format: `CATEGORY: Category / Subcategory` and/or `TAG: tagname`
- Multiple entries are comma-separated: `CATEGORY: Health & Fitness / Tennis, TAG: fixed-budget`
- Known shorthand: Normally, `CATEGORY: Health / X` resolves directly to `Health & Fitness / X` and
  is used in match expressions for a specific merchant.

## Rule Insertion Logic

**Never create `[Category Override - ...]` bracket names.** When a non-Amazon CSV row has a
`CATEGORY: X / Y` in its tagging column, add a rule to the `# --- Non-Amazon category overrides ---`
block inside `# === CATEGORY: tagging overrides ===`.
- Bracket name: use merchant's existing name or clean merchant name.
- Match: `contains("DESCRIPTION_PATTERN") and contains(field.tagging, "CATEGORY: X / Y")` with **no
  source filter**.

**Insert into the correct category section — never append.** The `merchants.rules` file has a fixed
layout:
1. Preserved infrastructure at top: Field Transforms, Variables, Tag-only rules, CC Payments /
   Transfers, Family Account Transfers, Check Number / Deposit Reference Rules, CATEGORY: tagging
   overrides. Never add generic merchant rules here.
2. Below: one section per `category:` value (`# === Auto ===`, `# === Food ===`), sorted
   alphabetically.
   - Insert into the matching section. **Never** create `# === X (continued) ===`, batch headers,
     or append to EOF.
   - If the category doesn't exist, create one new `# === Category ===` header in alphabetical
     position.
   - **Same-merchant pairs across categories:** keep adjacent in the primary rule's section.
     `# Category Override` comment above rules whose category differs.
   - **Within-pair order:** specific filters before generic (first-match-wins).
````