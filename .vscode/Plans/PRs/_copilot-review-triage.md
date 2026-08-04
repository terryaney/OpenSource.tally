# Copilot review triage — PRs #98–#103

**Status: COMPLETE — 2026-08-04.** All four phases done. See *Phases 3 and 4 — done* at the
bottom for what shipped; the rest of this document is the working record of how it got there.

Working document for addressing the Copilot review comments across the six-rung stack.
A new session should be able to start here and need nothing else except `branches.md`.

---

## Where things stand

| Phase | What | Status |
|---|---|---|
| 1 | Harvest all comments, attribute each to the rung that **owns the line** | done — table below |
| 2 | Triage each issue: real bug / cosmetic / wrong / already-handled | **not started** |
| 3 | Fix bottom-up, one pass per rung | not started |
| 4 | Reply to threads, resolve | not started |

**34 review comments → ~25 distinct issues** (the stack's cumulative diffs made Copilot report several issues once per PR).

---

## The two facts that shape all the work

### 1. Fix in the rung that owns the line, not the PR the comment appeared on

Every PR targets `main`, so each diff is cumulative — #103's diff contains #98–#102's code. Copilot reviewed the whole cumulative diff each time, so **14 of the 34 comments sit on a PR whose rung did not write the line**.

Patching where the comment appears puts the fix in the wrong commit and the wrong PR. The `owned by` column below is authoritative; it comes from `git blame` on the line, mapped to the rung whose commits contain it.

> **Caveat on blame:** it identifies who wrote *that line*, which is a strong hint but not proof of who caused the problem. A new feature can break an old line without touching it — e.g. `report.py:337` blames to `main`, but the issue only exists because rung 100 added the title feature around it. Read the comment before trusting the column.

### 2. Lower-rung edits are expensive; batch them

Editing any rung below the tip means `git rebase --update-refs`, a restack of every rung above, and a rebuild of `feature/experimental` (see `branches.md` → *Work on an existing rung or independent branch* → *Editing any lower `<rung>`*).

So: **do all of one rung's fixes in a single pass, and work bottom-up.** Rung-by-rung ping-ponging pays the restack cost repeatedly.

---

## Calibration — how much to trust these comments

The only completed data point is **PR #97** (ci-repair), where Copilot left 3 comments:

- **1 was a sharp, real bug** — artifact-poisoning hole where shape-validating a PR number didn't prove ownership. Genuinely worth fixing; the fix shipped.
- **2 identified the right file but prescribed an inert remedy** — told us to add `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24`, which has been a no-op since Node 24 became the runner default on 2026-06-16. The real fix was version bumps, which those comments *did* offer as a secondary option.

**Pattern: good at locating problems, unreliable on the prescribed fix.** Treat the location as probably real and the remedy as a suggestion.

> **Verify before shipping a fix.** On #97 the first replacement fix looked correct, read plausibly, and was broken — it used `GET /repos/{repo}/commits/{sha}/pulls`, which returns results for same-repo PRs but an **empty array for fork PRs**, so it would have failed on every fork PR. Only testing it against the live API caught it. Plausible reasoning is not evidence.

---

## Work order

Bottom-up. **Rung 100 is the bulk of the work — 20 of 34 comments.**

| Order | Rung | PR | Comments owned | Distinct issues | Notes |
|---|---|---|---|---|---|
| 1 | `feature/globbing-documentation` | #98 | 1 | 1 | trivial — docs formatting |
| — | `feature/report-json-determinism` | #99 | 0 | 0 | **nothing to do** |
| 2 | `feature/ui-tweaks` | #100 | 20 | 12 | the real work |
| 3 | `feature/charts-reimagined` | #101 | 5 | 4 | |
| 4 | `feature/merchant-composite-keys` | #102 | 1 | 1 | |
| 5 | `feature/categorization` | #103 | 6 | 6 | |
| — | (pre-existing `main` code) | — | 1 | 1 | judgment call — may be out of scope |

Rung 100 also owns most of the misattributed comments, so fixing it clears feedback showing on #101, #102 and #103 at the same time.

---

## The comments, grouped by owning rung

Duplicates are collapsed; "reported on" lists every PR where the same issue was raised.

### Rung 98 — `feature/globbing-documentation` (1 issue)

| # | Location | Issue | Reported on |
|---|---|---|---|
| 98-1 | `docs/formats.html:258` | Markdown backticks render literally in an HTML page — use `<code>` elements around wildcard syntax | #98 |

### Rung 100 — `feature/ui-tweaks` (12 issues, 20 comments)

| # | Location | Issue | Reported on |
|---|---|---|---|
| 100-1 | `src/tally/report.py` (339/359/369) | **Report title is injected into `<title>` and `<h1>` unescaped.** A title containing markup (e.g. `</title><script>…`) becomes executable in the generated report, including `--no-embedded-html`. Also `str.replace` raises `TypeError` on a non-string YAML value | #100, #101, #102, #103 |
| 100-2 | `src/tally/spending_report.html` (79/82/87) | **Year tabs are click-only `<div>`s** — not keyboard-focusable, not announced as controls. Use native buttons + selected state | #100, #101, #102, #103 |
| 100-3 | `src/tally/spending_report.js` (235/237) | Date parsing accepts impossible dates (`2/31/2026`, month 13, `99/99/2026`) as valid, creating chips that can never match | #100, #102 |
| 100-4 | `src/tally/config_loader.py:103` | `report_fields` iterated without validating it is a list of strings; `null` raises uncaught `TypeError`, `memo` is treated as four field names | #101, #102 |
| 100-5 | `src/tally/spending_report.js:1510` | **Malformed date in URL hash can hang the report.** `#+dr:garbage..garbage` → `expandMonthRange` loops on `NaN-NaN` forever | #100 |
| 100-6 | `src/tally/spending_report.js:1994` | Excluded month/date chips rehydrate into the *include* set — opening the popover and clicking Apply silently flips an exclusion to an inclusion | #100 |
| 100-7 | `src/tally/spending_report.js:2632` | Category pie + category-by-month click/legend handlers removed — chart segments no longer add filters. Undocumented user-facing regression | #100 |
| 100-8 | `src/tally/spending_report.js:1391` | Per-transaction percentage fix incomplete — `filteredTotal` still raw sums including income/investment/transfers while `grossSpending` excludes them | #100 |
| 100-9 | `src/tally/spending_report.js:2288` | Three style mutations + synchronous layout reads per transaction; repeats category data across profiles. Large reports will stall | #100 |
| 100-10 | `src/tally/merchant_engine.py:436` | Dropping empty `field:` results is a **global rule-engine behavior change** — rules that emitted an empty field now emit none, affecting existing personalized rules and downstream CSV/report consumers | #100 |
| 100-11 | `src/tally/spending_report.html:28` | Reset control's only accessible name is the `↺` glyph — add `aria-label` | #102 |
| 100-12 | `docs/reference.html:499` | Sentence contradicts surrounding docs: captured columns *are* available to every rule as `field.<name>`; only a rule's `field:` output isn't an input to other rules | #102 |

### Rung 101 — `feature/charts-reimagined` (4 issues, 5 comments)

| # | Location | Issue | Reported on |
|---|---|---|---|
| 101-1 | `docs/charts.html:264` | Docs say categories map to fixed/variable; implementation classifies **merchants** by inferred recurrence with `fixed`/`variable` tag overrides | #101, #102 |
| 101-2 | `src/tally/analyzer.py:197` | Monthly inference counts distinct populated months, not elapsed calendar span — a merchant charged every January for 3 years is classified monthly before annual inference runs | #101 |
| 101-3 | `src/tally/spending_report.html:149` | Chart panel toggles use the click-only heading pattern — same a11y problem as 100-2, applies to seasonality/cash/volatility panels too | #102 |
| 101-4 | `src/tally/spending_report.js:3488` | `disabled` class only removes the click listener; generated "Other" control stays focusable and announces as enabled | #101 |

### Rung 102 — `feature/merchant-composite-keys` (1 issue)

| # | Location | Issue | Reported on |
|---|---|---|---|
| 102-1 | `src/tally/commands/explain.py:134` | **Breaking JSON contract change.** Every exact single-merchant query now returns a wrapper (`query`/`match_mode`/`merchants`) instead of a merchant object — breaks existing `tally explain --format json <merchant>` consumers even with no duplicate names | #102 |

### Rung 103 — `feature/categorization` (6 issues)

| # | Location | Issue | Reported on |
|---|---|---|---|
| 103-1 | `src/tally/merchant_utils.py:569` | Using every matching rule leaks `review` from a lower-priority broad rule onto transactions already categorized by a specific rule — breaks the documented specific/general pattern in `docs/guide.html` | #103 |
| 103-2 | `src/tally/categorization.py:305` | A `review: true` tag-only rule can match an uncategorized transaction, so it appears in both `unknowns` and `reviews` with the same key; the blank review row can overwrite an entered answer on merge | #103 |
| 103-3 | `src/tally/categorization.py:507` | When the last unknown/review row is resolved, early return leaves the stale `categorization.yaml` on disk — contradicts the merge contract, agents may act on stale answers | #103 |
| 103-4 | `src/tally/categorization.py:184` | Malformed rows silently accepted — a `useRules:` typo is ignored and the next regeneration drops the answer, since only `PRESERVED_FIELDS` are copied | #103 |
| 103-5 | `src/tally/categorization_common.py:80` | Labels de-duplicated even though multiple rules may share merchant/category/subcategory/tags with different match expressions — `useRule` then can't identify which rule was meant | #103 |
| 103-6 | `src/tally/parsers.py:94` | Identity truncated to 32 bits — ~1% collision chance around 9,000 distinct transactions; colliding transactions ordinalized as duplicates, shifting keys and reattaching answers | #103 |

### Owned by `main` — pre-existing (1 issue, judgment call)

| # | Location | Issue | Reported on |
|---|---|---|---|
| main-1 | `src/tally/report.py:337` | Title fallback applies only to the static shell and `<title>`; Vue data still has `title: None`, so mounting resets the visible heading and `document.title` to `Financial Report`. Store the resolved value | #100 |

Blames to `main`, but only manifests because rung 100 added the title feature — likely belongs in rung 100 with 100-1.

---

## Themes worth batching

Several issues are the same fix applied in more than one place:

- **Accessibility / click-only elements:** 100-2, 100-11, 101-3, 101-4. Same root pattern (non-semantic clickable elements). 101-3 explicitly says it recurs across the seasonality/cash/volatility panels.
- **Untrusted input reaching output:** 100-1 (HTML injection via title), 100-4 (`report_fields` type validation), 100-3 / 100-5 (date parsing → bad chips, infinite loop).
- **Documentation contradicting implementation:** 98-1, 100-12, 101-1.
- **Silent data-loss / overwrite risks:** 103-2, 103-3, 103-4, 103-6.

---

## Re-deriving this table

If the comments change, regenerate rather than hand-edit. The harvest script lives at
`<scratchpad>\harvest.ps1` (session-scoped — recreate from this description if gone):

1. Verify each local rung tip matches its PR head (`gh api repos/davidfowl/tally/pulls/<n> --jq .head.sha`) — blame is meaningless otherwise. All six matched at harvest time.
2. Build a commit→rung map from `git log <base>..<head> --format=%H` per rung.
3. For each comment from `gh api repos/davidfowl/tally/pulls/<n>/comments`, run
   `git blame -L <line>,<line> --porcelain <head> -- <path>` and map the resulting SHA through that table.

Rung base/head pairs:

| PR | base | head |
|---|---|---|
| 98 | `main` | `feature/globbing-documentation` |
| 99 | `feature/globbing-documentation` | `feature/report-json-determinism` |
| 100 | `feature/report-json-determinism` | `feature/ui-tweaks` |
| 101 | `feature/ui-tweaks` | `feature/charts-reimagined` |
| 102 | `feature/charts-reimagined` | `feature/merchant-composite-keys` |
| 103 | `feature/merchant-composite-keys` | `feature/categorization` |

---

## Notes for the next session

- **Start with phase 2 on rung 100** — it's 12 of the ~25 issues and it's low in the stack.
- Don't open the PRs one at a time; that's what this document exists to avoid.
- Before editing a lower rung, re-read `branches.md` → *Editing any lower `<rung>`*, and record the tip SHAs first. The verify step there is the one that must never be skipped.
- After any lower-rung edit, `feature/experimental` needs a **rebuild**, not a merge (`branches.md` → *Rebuild feature/experimental* step 1).
- Comment IDs and URLs for replying are in the harvested JSON; re-harvest if it's gone.
- Reply to threads only after the fix is verified — see the calibration note above.

---

## Phase 2 finding — PR body trimming caused at least one comment

Investigated 2026-08-04. The PR bodies were trimmed in playbook commit `5d53603`
("All PRs through Categorization submitted"); `ui-tweaks.md` went 61 -> 38 lines,
`merchant-composite-keys.md` 46 -> 28. Diff the trim with:

```powershell
git -C c:\BTR\OpenSource\tally-playbook diff 95c412c 5d53603 -- .vscode/Plans/PRs/ui-tweaks.md
```

### 100-7 is almost certainly NOT a code bug — reclassify

Copilot called it "an **undocumented** user-facing regression." It was documented; the trim deleted
the disclosure. The pre-trim `ui-tweaks.md` "At a Glance" list contained:

> - Renamed some charts, added a top 10 +other pattern, **chart click for visibility instead of filtering**

That line is absent from the posted body. The code change is real and deliberate — `spending_report.js`
goes from **4 `onClick` handlers on `feature/report-json-determinism` to 1 on `feature/ui-tweaks`**,
and the survivor still calls `addFilter` for months.

**Likely resolution: restore the sentence to the PR body and reply that it was intentional.** No code
change. Verify the intent still stands before replying — "visibility instead of filtering" is a product
decision, and Copilot is right that removing it is user-facing either way.

### Other trim casualties (not currently flagged, but disclosure gaps)

- **Available-months source change** — cut: *"The underlying month source was also corrected. Available
  months now come from `categoryView`, so a month is not lost just because its merchant was excluded
  from every configured view."* A real behavior change, now undisclosed.
- **Section View denominator explanation** — cut: the section explaining the denominator now comes from
  `filteredViewTotals.value.spending`. Relevant to **100-8**, which argues the percentage fix is
  incomplete; the cut removed the statement of what the fix was scoped to do.
- **Mobile scope caveat** — cut: pre-existing mobile issues acknowledged as deliberately out of scope.

### Not caused by the trim — genuine gaps

Checked the pre-trim bodies; these were never disclosed in either version:

- **100-10** (dropping empty `field:` results is a global rule-engine behavior change) — the old body
  mentioned only `report_fields` config and "blank values no longer create phantom badges," which is
  the adjacent symptom, not the engine change.
- **102-1** (`tally explain --format json` contract change) — the pre-trim
  `merchant-composite-keys.md` never mentions `explain`, JSON, or a contract change at all.

### Takeaway for phase 2

Three of the ~25 issues (100-7, 100-10, 102-1) are **disclosure problems, not code problems**. Check
whether a comment is asking for a code change or for the PR body to say what the code already does
deliberately — the pre-trim bodies in `95c412c` are the reference for what was already known.

---

## Disclosure restoration — DONE 2026-08-04

Full audit of all six drafts against their pre-trim versions (`git diff 95c412c 5d53603`). Important
method note: the trim's `-` lines are **not** proof of loss — several were rephrased into equivalent
`+` lines. Each item below was verified absent from the *current* draft before restoring.

| PR | Verdict | Action taken |
|---|---|---|
| #98 | nothing behavioral cut (only stack nav, replaced by the Stack table) | none |
| #99 | tag-ordering consequence lost | restored, pushed |
| #100 | 4 behavior changes + mobile caveat lost | restored, pushed |
| #101 | **no loss** — filter-surface / peek-state / Top-10 items were rephrased, not dropped | none |
| #102 | root-cause detail cut but recoverable from the diff; `Fixes #88` retained | none |
| #103 | `tally up` behavior delta lost | restored, pushed |

Restored content, all now live on GitHub:

- **#99** — "Accepted consequence: the rule info popup now lists a rule's tags alphabetically rather
  than in rule-declaration order."
- **#100** — new **Behavior changes** section: chart clicks toggle visibility instead of filtering;
  available months now from `categoryView`; Section View denominator now `filteredViewTotals.value.spending`;
  blank captured fields no longer render a phantom `+1 Memo`. Plus the mobile out-of-scope caveat.
- **#103** — new **Behavior changes on `tally up`** section: writes three files + status line; **can now
  exit non-zero on malformed YAML where it previously always exited 0**; STALE warning behavior and
  the never-deletes guarantee; `reviewComplete` is review-scoped and never gates parsing or analysis.

### 100-7 — CLOSED, no code change

Replied on [#100 discussion_r3714465548](https://github.com/davidfowl/tally/pull/100#discussion_r3714465548):
intentional interaction change, disclosure restored, and noted that #101 deliberately brings
filtering-by-chart back as an explicit filter surface. **Remove from the phase 3 code work.**

Rung 100 is therefore **11 code issues, not 12.**

---

## Two decisions still needed from you — NOT actioned

Both are undisclosed behavior changes, but unlike 100-7 they were **never** in any draft, so there is
no prior statement of intent to restore. Documenting them would mean asserting intent on your behalf,
and if the change wasn't intended the fix is code, not prose. Decide before replying to either.

**100-10 — `merchant_engine.py:436`, dropping empty `field:` results.** Copilot calls it a global
rule-engine default change affecting existing personalized rules and downstream CSV/report consumers.
The pre-trim body mentioned only `report_fields` config and "blank values no longer create phantom
badges" — the user-facing symptom, not the engine change.
→ *Intended?* Document it under #100's new Behavior changes section. *Not intended?* Code fix in rung 100.

**102-1 — `commands/explain.py:134`, `tally explain --format json` contract.** Every exact
single-merchant query now returns a `query`/`match_mode`/`merchants` wrapper instead of a merchant
object, breaking existing consumers even when no duplicate name exists. Never mentioned in any draft
of `merchant-composite-keys.md`.
→ This is a **public CLI contract break**; David may object regardless of intent. Consider whether the
wrapper can be emitted only when a query is genuinely ambiguous, preserving the old shape otherwise.

---

## Rule going forward

The trimming instinct was right — the old bodies carried real padding ("This is the strongest
correctness story in the PR"). What should never be cut is any sentence describing a **behavior
change, compatibility break, accepted trade-off, or scope caveat**. Those are the only things a
reviewer cannot recover from reading the diff, and cutting them is what produced 100-7.

---

## Both open decisions RESOLVED — 2026-08-04

**100-10 — intentional, documented, closed. No code change.**

My first characterization of this was **wrong** and has been corrected in both the PR body and on the
thread. For the record, since it changes how the issue should be understood:

- `field:` directive output lands in `MatchResult.extra_fields`.
- `field.<name>` **in a match expression** resolves to captured columns from the data file
  (`format_parser` / `parsers`), which this change does not touch.
- The two share a name but are different things, so **rule matching is unaffected**.

Real blast radius: the `+N` transaction popup (`spending_report.js`), dynamic `extra_fields` columns in
CSV export (`analyzer.py`), and transaction search. The `+N` badge is the reason for the change — a
`field:` directive runs for every transaction its rule matches, so a merchant rule capturing a memo
attached an empty memo to every transaction lacking one, producing a `+1` badge over an empty popup.

**102-1 — intentional, documented, closed. No code change.**

Forced by the fix: composite keys mean one merchant name can resolve to several entries, which a bare
object cannot express. Before/after shape is in #102's body.

Two things established while checking it:

- **The old JSON was already invalid for multi-match.** The partial-match path printed a plain-text
  `Merchants matching 'x':` header followed by concatenated JSON objects, even under `--format json`.
  The wrapper makes that path parseable for the first time.
- **The break is narrower than "API contract" suggests.** `explain` computes from config and data files
  at run time and never reads `tally up`'s output, so there is no mixed-version or persisted-format
  concern. What breaks is a user's own script parsing the old single-object shape across an upgrade.

Chose a uniform envelope over "bare object when one match, wrapper when several" — polymorphic output
means consumers branch forever instead of migrating once. Offered to revisit if David disagrees.

### Phase 3 scope is now final

| Rung | PR | Code issues remaining |
|---|---|---|
| globbing-documentation | #98 | 1 |
| report-json-determinism | #99 | 0 |
| **ui-tweaks** | **#100** | **10** |
| charts-reimagined | #101 | 4 |
| merchant-composite-keys | #102 | **0** |
| categorization | #103 | 6 |

**21 code issues**, down from the original 34 comments. Three closed as disclosure-only (100-7, 100-10,
102-1), the rest collapsed by de-duplication across the cumulative diffs.

Every disclosure-only item is now answered on its thread and reflected in the PR body on GitHub —
nothing in the list above is waiting on prose.

---

## Phases 3 and 4 — done, 2026-08-04

All 21 code issues fixed, 31 threads replied to and resolved, three PR bodies updated.

| Rung | PR | Code issues | Fix commit |
|---|---|---|---|
| `feature/globbing-documentation` | #98 | 1 | `1650e5b` |
| `feature/report-json-determinism` | #99 | 0 (restacked only) | `1f323d9` |
| `feature/ui-tweaks` | #100 | 10 | `0590b28` |
| `feature/charts-reimagined` | #101 | 4 | `76a1b73` |
| `feature/merchant-composite-keys` | #102 | 0 code; commit amended to adapt tests | `69176f8` |
| `feature/categorization` | #103 | 6 | `33644a2` |

Tests went 922 -> 1066. `feature/experimental` rebuilt from `fork-identity`; all six rungs
force-pushed and each PR head verified against its local rung.

### Three threads left unresolved on purpose

`3714465526` (100-10), `3714465548` (100-7), `3714471871` (102-1) — the disclosure-only trio.
All three already carry replies. 102-1 ends with an open offer to revisit the `explain --format
json` envelope if David disagrees, and leaving the thread open is what keeps that offer visible.

### Two findings that changed the answer

**100-1 was wider than reported.** Copilot flagged the title reaching `<title>` and `<h1>`
unescaped. It also reaches `window.spendingData`, which is embedded *inside* a `<script>`
element, so a title containing `</script>` closed that element no matter what the shell did.
A test written for the reported bug is what caught it. The `</` -> `<\/` escaping now covers the
whole payload, which also closes the same hole for merchant descriptions read from user CSVs —
pre-existing on `main`, and disclosed as such in #100's body.

**101-2's stated scenario does not reproduce.** "A merchant charged every January for 3 years is
classified monthly" is false on a normal data set: `num_months` is itself a count of distinct
populated months, so a dense data set puts the threshold at 18 and the merchant fails it
correctly. It reproduces only when the data set is as sparse as the merchant — three Januaries
in a January-only export. The underlying complaint was right even though the example was not,
and the fix targets the real defect: `months_active` measures presence, not spread.

Both are the calibration note's pattern holding: **location trustworthy, specifics not.**

### Process failure worth remembering

The plan was to commit fixes on rungs 98, 100 and 101, then restack once from rung 98.
**That silently dropped the rung-100 and rung-101 commits.** Committing on a lower rung puts
that commit off `<tip>`'s ancestry, so a rebase based below it never replays it — exit 0, no
conflict, and the rung simply absent from `Updated the following refs`. Caught only by grepping
the tip for a marker string from each fix.

Recovered by tagging the stranded commits, resetting to the recorded tips, and redoing it one
rebase per modified rung. That surfaced two genuine conflicts the batched run had hidden.
Written up in `branches.md` -> *Editing any lower `<rung>`* -> step 2, commit `8910cf1`.

### Local consequence of 103-6

Widening the transaction key invalidates any existing `categorization.yaml`. The live file at
`C:\BTR\TallySpending\tally\config` was migrated in place — 36 keys remapped, 21 answers
preserved, backup at `categorization.yaml.bak`. The old key is a strict prefix of the new one,
so the mapping is exact; it was computed from the real transactions rather than by extending a
prefix, because ordinals can legitimately change when rows that collided at 32 bits no longer
collide at 64.
