# Phase 4 — draft replies to the Copilot review threads

**Status:** drafted 2026-08-04, **not posted**. Awaiting approval.

27 threads. Three are already answered and closed and are excluded:
`3714465526` (100-10), `3714465548` (100-7), `3714471871` (102-1).

Fix commits, one per rung:

| Rung | PR | Commit |
|---|---|---|
| `feature/globbing-documentation` | #98 | `1650e5b` |
| `feature/ui-tweaks` | #100 | `0590b28` |
| `feature/charts-reimagined` | #101 | `76a1b73` |
| `feature/categorization` | #103 | `33644a2` |

**14 of the 27 are duplicates** — the same issue reported again on a higher PR because
every PR's diff is cumulative. Those get the "fixed in the rung that owns the line" reply,
which also explains the stack to David rather than looking like a non-answer.

---

## A. Fixed in the rung the thread is on

### `3714432150` — #98, `docs/formats.html` (98-1)

> Fixed in `1650e5b`. The wildcards are now `<code>` elements, matching every other cell in
> that table. It was the only literal-backtick text this rung introduced.

### `3714465488` — #100, `report.py` (100-1, title injection)

> Fixed in `0590b28`, and the hole was wider than the thread describes. Escaping the shell
> was the easy half; the title also reaches `window.spendingData`, which is embedded inside a
> `<script>` element, so a title containing `</script>` closed that element regardless of the
> `<title>`/`<h1>` escaping. Both paths are now covered:
>
> - the loading shell gets `html.escape()`d text
> - the JSON payload has `</` escaped to `<\/`, which is byte-identical to a JSON parser but
>   inert to the HTML tokenizer
>
> The non-string case is handled too: `title: 2025` arrives from YAML as an `int` and used to
> raise `TypeError` inside `str.replace`. Covered by `tests/test_report_title.py`, including a
> parametrized case over both `--no-embedded-html` and the default.
>
> Worth flagging separately: the `</script>` exposure is not specific to `title`. Any string in
> `spendingData` has it, and merchant descriptions come from user CSVs, so this predates the
> rung. The escaping is applied to the whole payload rather than just the title.

### `3714465469` — #100, `report.py` (main-1, title reset on mount)

> Fixed in `0590b28`. Right diagnosis, and the two fallbacks were also different strings —
> Python fell back to `Tally Spending Analysis`, the JS computed to `Financial Report`, so an
> untitled report visibly renamed itself on mount. The title is now resolved once in
> `report.py` and stored back onto `spendingData`, so the shell and the mounted app cannot
> disagree. The JS fallback is kept but aligned, since it now only covers a hand-edited
> `spending_data.js`.

### `3714465405` — #100, `spending_report.js` (100-3, impossible dates)

> Fixed in `0590b28`. The regexes only ever proved shape, so `2/31/2026`, month 13 and
> `99/99/2026` all parsed. `parseTypedDate` now runs the result through `isRealDate`, which
> checks the day against `new Date(y, m, 0).getDate()` — day 0 of the next month is the last
> day of this one, so leap years fall out for free.

### `3714465325` — #100, `spending_report.js` (100-5, hash hang)

> Fixed in `0590b28`, and it is worse than "loops": it never terminates. `'garba'.split('-')`
> gives `[NaN]`, so `current` becomes the string `'NaN-NaN'`, and `'NaN-NaN' <= 'garba'` is
> true forever because `'N'` sorts before `'g'` — the array grows without bound until the tab
> dies. `expandMonthRange` now requires both halves to match `^\d{4}-(0[1-9]|1[0-2])$` and
> returns `[]` otherwise. Validating the input is what makes the loop provably terminate,
> rather than capping the iterations.

### `3714465381` — #100, `spending_report.js` (100-6, exclusion flips to inclusion)

> Fixed in `0590b28`. The popover now owns include-mode date chips only: `openDatePopover`
> skips anything that is not `mode === 'include'` when rehydrating, and `applyDateFilters`
> only replaces the chips it owns, so an excluded month or range survives untouched.

### `3714465357` — #100, `spending_report.js` (100-8, percentage basis)

> Fixed in `0590b28`. Correct — the denominator was moved to a spending-only basis and the
> numerator was left as a raw sum, so a section containing income, investment or transfers
> divided one basis by another and could exceed 100%. Sections and categories now carry a
> `filteredSpending` computed with the same per-transaction `categorizeAmount` classification
> `grossSpending` uses, and the section percentage uses that.
>
> The per-merchant percentage inside a category is deliberately unchanged: its numerator and
> denominator are both raw totals, so it is internally consistent.

### `3714465437` — #100, `spending_report.js` (100-9, layout thrash)

> Fixed in `0590b28`. Each measurement mutated the node then read `getBoundingClientRect`,
> forcing a synchronous layout — three per transaction, repeated across all three column
> profiles. Measurements are now memoized by the exact string measured, which collapses both
> the repeats within a profile and the duplication across profiles, since `merchant` and
> `subcategory` mode measure the same rows. The cache is cleared at the start of each
> recompute so a resize still re-measures against the new font metrics.

### `3714465575` — #100, `spending_report.html` (100-2, year tabs)

> Fixed in `0590b28`, for the month cells as well — they had the same problem a few lines
> below. Both are now `<button type="button">` with `aria-pressed` reflecting selection, and
> the year tab's pending-months dot has an `.sr-only` text equivalent, since a coloured dot
> is not announced. CSS keeps them rendering exactly as before (`background`, `font-family`,
> `line-height`, `width`); the existing Playwright tests drive them by `data-testid` and pass
> unchanged.

### `3714457890` — #101, `docs/charts.html` (101-1)

> Fixed in `76a1b73`. The docs were simply wrong: it is a per-merchant split driven by
> inferred recurrence, not a mapping of categories. Rewritten to say that, plus how the
> cadence is inferred and that `fixed`/`variable` rule tags override it.

### `3714457926` — #101, `analyzer.py` (101-2, monthly inference)

> Fixed in `76a1b73`. Worth recording what reproduces it, because the first reading did not
> hold up: `num_months` is itself a count of distinct populated months, not a calendar span,
> so on a dense data set a merchant billed every January fails the existing test and reaches
> annual inference correctly. It breaks when the data set is as sparse as the merchant —
> three Januaries in a January-only export give `months_active == num_months == 3`, which
> clears `max(3, ceil(3 * 0.5))` and books an annual premium as a monthly cost, 12x its real
> value.
>
> The deeper problem is the one you named: `months_active` measures presence, not spread, and
> the classification depended on *other merchants'* data. There is now a second test against
> the merchant's own first-to-last span, so three charges over twenty-five months cannot read
> as monthly. It is a pure tightening — it can only remove false `monthly` classifications.
>
> Symptom of the old coupling, for what it is worth: the existing tests needed a dummy
> monthly merchant present purely to hold `num_months` high enough.

### `3714457960` — #101, `spending_report.js` (101-4, disabled chip)

> Fixed in `76a1b73` — `b.disabled = true` alongside the class, so it leaves the tab order and
> announces as disabled. The existing test proved inertness by clicking it, which Playwright
> now refuses on a disabled element, so it was updated to assert `to_be_disabled()` and use a
> forced click to show there is still no side effect.

### `3714474359` — #103, `merchant_utils.py` (103-1, review leak)

> Fixed in `33644a2`. `all_matching_rules` includes rules that matched and then lost the
> specificity contest, contributing nothing — so a broad catch-all flagged `review:` pulled in
> every transaction a more specific rule had already categorized, which is the
> specific-beats-general pattern in `guide.html` working as documented. `review` now comes
> from the rules that actually applied: the category, merchant and subcategory winners, plus
> the rules that contributed tags. Tag-only review rules still surface, which was the point of
> using the wider list in the first place.

### `3714474099` — #103, `categorization.py` (103-2, same key in both lists)

> Fixed in `33644a2`, in both places. An uncategorized transaction is now excluded from the
> review list outright — it belongs in `unknowns`, which is where the answer is being asked
> for. Separately, `_load_existing` no longer lets an unanswered row displace an answered one
> when a key appears twice, so a hand-edited file cannot lose an answer either.

### `3714474030` — #103, `categorization.py` (103-3, stale file)

> Fixed in `33644a2`. When everything resolves and a file exists, it is rewritten empty rather
> than abandoned; it is still never deleted, since it is the user's file. One detail worth
> noting: the empty list is written as an explicit `unknowns: []`, because a bare `unknowns:`
> parses as `null` and `_load_existing` would reject the file it had just written. `written`
> now reports `True` in this case, and the test that asserted the old early-return behaviour
> was updated.

### `3714474127` — #103, `categorization.py` (103-4, silent typo)

> Fixed in `33644a2`. Unrecognized field names are now rejected, naming the field and
> suggesting the intended one for the common misspellings. This follows the precedent already
> set in this PR by `inventory.yaml`, which rejects unknown keys for exactly the same reason —
> a `reviewComplte` typo would otherwise silently lose the flag.

### `3714474206` — #103, `categorization_common.py` (103-5, ambiguous labels)

> Fixed in `33644a2`. Where two rules produce the same label, and only there, the match
> expression is appended so `useRule` identifies one rule rather than a set. Unique labels are
> untouched, so the common case does not get noisier. Genuinely identical rules — same label
> and same expression — still collapse to one entry, since that is a duplicate rather than an
> ambiguity.

### `3714474159` — #103, `parsers.py` (103-6, 32-bit identity)

> Fixed in `33644a2` — widened to 64 bits. The consequence is worth spelling out: a collision
> is indistinguishable from a genuine duplicate, so `assign_transaction_keys` ordinalizes the
> two rows, their keys shift, and a stored answer reattaches to the wrong transaction. Silent
> in every direction.

---

## B. Duplicates — fixed in the rung that owns the line

Every PR targets `main`, so each diff is cumulative and these lines belong to a rung below the
PR the comment landed on. Each reply names the owning rung so the thread is followable.

**Template** (substituted per row):

> This line belongs to `<rung>` (#<pr>), which this PR is stacked on — the diff here is
> cumulative, so it shows up again. Fixed there in `<sha>`; <one clause>.

| Comment | On PR | Owning rung | Fix | One clause |
|---|---|---|---|---|
| `3714457782` | #101 | `feature/ui-tweaks` #100 | `0590b28` | title is escaped for the shell and `</` escaped in the embedded JSON |
| `3714457827` | #101 | `feature/ui-tweaks` #100 | `0590b28` | `report_fields` is validated; a bare string is one name, empty is none, anything else is rejected |
| `3714457860` | #101 | `feature/ui-tweaks` #100 | `0590b28` | year tabs and month cells are buttons with `aria-pressed` |
| `3714471791` | #102 | `feature/ui-tweaks` #100 | `0590b28` | as above, both escaping paths |
| `3714471848` | #102 | `feature/ui-tweaks` #100 | `0590b28` | `report_fields` validated |
| `3714471908` | #102 | `feature/ui-tweaks` #100 | `0590b28` | `parseTypedDate` rejects dates the calendar does not have |
| `3714471941` | #102 | `feature/ui-tweaks` #100 | `0590b28` | year tabs and month cells are buttons |
| `3714472009` | #102 | `feature/ui-tweaks` #100 | `0590b28` | the sentence was wrong and is rewritten |
| `3714472086` | #102 | `feature/ui-tweaks` #100 | `0590b28` | `aria-label` added to the reset control |
| `3714472052` | #102 | `feature/charts-reimagined` #101 | `76a1b73` | charts docs rewritten to describe the per-merchant recurrence split |
| `3714474261` | #103 | `feature/ui-tweaks` #100 | `0590b28` | both escaping paths |
| `3714474311` | #103 | `feature/ui-tweaks` #100 | `0590b28` | year tabs and month cells are buttons |

### `3714471971` — #102, `spending_report.html` (101-3, chart panel toggles)

Not a duplicate — this one is owned by #101 and needs its own reply.

> This line belongs to `feature/charts-reimagined` (#101), which this PR is stacked on. Fixed
> there in `76a1b73`, across all six panels — category, seasonality, cash, volatility, fixed
> and audit. The `<h3>` stays a heading so document navigation is unchanged, with a
> chrome-stripped `<button>` inside it carrying `aria-expanded` and `aria-controls`. The peek
> badge became a real `<button>` too; it already had `role="button"` but was neither focusable
> nor operable.

---

## Then

Resolve each thread after its reply posts. Nothing here is waiting on further code.
