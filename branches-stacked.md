# Branch Inventory & Playbook (stacked model)

Local workflow documentation for the tally fork.

> **STATUS: not yet in effect.** This describes the model the fork moves to after the *Restart* play at the bottom runs. Until then, `branches.md` (parallel-branches-off-main model) is the live document. Delete `branches.md` and rename this file once the restart's acceptance checks pass.

Lives on the `playbook` branch — an orphan branch with no shared history with `main`, permanently checked out in the `tally-notes` worktree. It merges nowhere, so it is structurally incapable of reaching an upstream PR. See *Local machine setup*.

## The model in one picture

```
upstream/main ── main
                  │
                  ├── feature/workflow-node24                    rung 1
                  │     └── feature/globbing-documentation       rung 2
                  │           └── feature/ui-tweaks              rung 3
                  │                 └── feature/charts-reimagined rung 4
                  │                       └── feature/merchant-composite-keys  rung 5  ← stack tip
                  │
                  └── feature/experimental = main
                                           + fork-identity commit
                                           + merge(stack tip)

playbook  (orphan — notes, plans, prototypes; merges nowhere)
```

Three rules the whole document follows from:

1. **Every feature branch is based on the rung below it**, never on `main` (except rung 1).
2. **`feature/experimental` is `main` + one fork-identity commit + a merge of the stack tip.** Nothing else is ever merged into it.
3. **One upstream PR is open at a time**, from the lowest rung not yet merged.

There are no `integration(...)` commits in this model. A clash between two features is fixed *in the upper rung*, because the upper rung is authored on top of the lower one.

## Inventory

Row order = stack order, bottom to top. Each rung's base is the row above it.

| # | Branch | Base | Upstream PR | Notes |
|---|--------|------|-------------|-------|
| 1 | `feature/workflow-node24` | `main` | *(next to submit)* | Action version bumps + `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24`. Fixes upstream's broken `build` job — genuinely useful to David, smallest possible diff. |
| 2 | `feature/globbing-documentation` | rung 1 | | Docs + tests only |
| 3 | `feature/ui-tweaks` | rung 2 | | Authored against string-keyed merchants — independent of rung 5 |
| 4 | `feature/charts-reimagined` | rung 3 | | |
| 5 | `feature/merchant-composite-keys` | rung 4 | | Stack tip. Composite merchant keys; opaque `merchant_<b64>` row IDs. Was `issue/88-merchant-category-display` / PR #91. **Carries all the integration debt** — see below. |

> **Why composite-keys is on top, not at the bottom.** It is the only rung likely to draw a real objection: it changes merchant identity, which `CLAUDE.md` flags as the rule engine's core value with a historical break (952c508). Putting it last means (a) rungs 1–4 are clean cherry-picks with no cross-rung conflicts, (b) all reconciliation between the new keys and the UI/chart code lives in one rung instead of being scattered, and (c) if David rejects it, **nothing restacks** — it simply stays a fork-only rung above the tip. The four easy PRs go out first; the risky one goes last, where rejection is free.

`feature/experimental` and `playbook` are never inventory rows.

> **Renaming is free** as long as no PR is open from the branch. Once a PR is open, its head branch cannot be renamed without closing the PR — so rename *before* you submit, never after.

## The fork-identity commit

`feature/experimental`'s first commit above `main`. Message: `fork: identity and experimental build wiring`. It contains **only** what makes this fork distinguishable and self-building:

- `src/tally/_version.py` — `REPO_URL` → `terryaney/OpenSource.tally`
- `src/tally/cli.py`, `docs/index.html` — fork description/branding
- `.github/workflows/dev-build.yml` — trigger on `feature/experimental`, "Experimental Build" naming, timestamp version scheme
- `.github/workflows/release.yml` — branch refs, workflow lookup by file path, draft-SHA match validation, "Tally Experimental" titles
- `.github/workflows/build.yml` — `REPO_URL` injection, artifact naming

It contains **no notes, no plans, no prototypes, no screenshots**. Those live on `playbook`.

> **⚠ Do not rename `build-rc` → `build-release` or `rc-artifacts` → `release-artifacts`.** That rename is the *only* thing that ever conflicted with `feature/workflow-node24` — both touch the same three lines of `dev-build.yml`. Keep David's "RC" wording and the fork-identity commit merges cleanly with the stack forever. Verified: with the rename dropped, `build.yml`, `dev-build.yml` and `release.yml` all auto-merge.

---

# Plan a feature with an AI agent

Plans, prototypes and research notes live in **`tally-notes\.vscode\Plans\`** — on the `playbook` branch, tracked, in the notes worktree. Never in the main worktree.

```
.vscode\Plans\
  <topic>.md                 # loose plans
  Features\<Feature>.md      # per-feature plans
  Prototypes\<thing>.html    # throwaway protos
```

Point the agent at that path and let it read and write there directly while you develop on any rung in the main worktree. `permissions.additionalDirectories` in `.claude/settings.local.json` already grants access, so there is no per-file approval prompt.

Why this and not "write the plan into the feature branch and promote it later":

- **It cannot leak.** `playbook` is an orphan branch that merges into nothing. A plan there is structurally incapable of appearing in a rung or an upstream PR diff — that's a property of the branch graph, not a rule anyone has to remember.
- **No promote step, no cleanup.** The file is tracked from the moment it's created.
- **No clobbering.** The rejected alternative (an ignored `Plans/` dir in the main worktree) would be silently overwritten the moment a tracked copy arrived via a merge — see *Failure mode: ignored files are silently clobbered*.
- **Always visible.** It survives every `git checkout` in the main worktree, because it isn't in the main worktree.
- **No build side effects.** `playbook` never reaches `feature/experimental`, so committing notes can never trigger a build. (The old `paths-ignore: .vscode/**` hack in `dev-build.yml` is no longer needed and should be removed from the fork-identity commit.)

Use unique, descriptive filenames — the directory is shared across all features, not per-branch. Commit notes whenever; it never disturbs in-flight feature work.

---

# Playbook

Run everything yourself; each play is a handful of git commands. Each play flags its own **🤖 Work with AI Agent if...** moments inline, at the step where they bite.

Two rules for those handoffs, everywhere they appear: hand over the **specific conflict or failure** — the conflict hunk, the failing test output, the unexplained diff — and **never the whole play**.

- [Work on a rung](#work-on-a-rung)
  - [Test a local build without committing](#test-a-local-build-without-committing)
- [New feature](#new-feature)
- [Submit the next PR](#submit-the-next-pr)
- [My PR was accepted upstream](#my-pr-was-accepted-upstream)
- [My PR was rejected](#my-pr-was-rejected)
- [Upstream main moved](#upstream-main-moved)
- [Include someone else's upstream PR or branch](#include-someone-elses-upstream-pr-or-branch)
- [Rebuild feature/experimental](#rebuild-featureexperimental)
- [Restart checklist](#restart-checklist--delete-this-section-when-done) — one-time, disposable
- [When to abandon upstream](#when-to-abandon-upstream)

## The ground-truth check

Every play that rebases records tips **before** and diffs **after**. `--update-refs` moves all rung refs in one pass, so there is no "before" to compare against unless you captured it first. Capture with:

```powershell
git rev-parse --short feature/workflow-node24 feature/globbing-documentation `
  feature/merchant-composite-keys feature/ui-tweaks feature/charts-reimagined
```

**How many rungs you must diff depends on where the new base came from:**

| The rebase's new base is… | Check | Why |
|---|---|---|
| **your own edit** to a lower rung (*Work on a rung*, *My PR was rejected*) | **tip only** | You know what changed — you wrote it. The tip diff should show exactly your edit and nothing else. |
| **upstream** (*Upstream main moved*, *My PR was accepted*) | **every rung, individually** | A replayed commit can silently resurrect code David deliberately deleted. Only a per-rung tree diff catches it, and a bad resolution at a low rung propagates invisibly through every rung above. |

Nothing in the upstream-base case is optional. See the PR #92 warning under *Upstream main moved*.

> **⚠ Never skip a ground-truth check to save AI tokens.** The check itself is free — it is `git diff --stat`, run by you, costing nothing. Only *explaining* a non-empty result costs anything, and that cost is the entire point: a diff you can't account for is the one failure mode in this model that ships silently and stays shipped.

> **🤖 Work with AI Agent if...**
> - A diff is non-empty and you can't account for **every line**. Hand over *that diff* and the rung it belongs to — not the branch, not the play.
> - You can account for the lines but can't tell whether a deletion was *upstream-intentional* or *accidentally lost*. `git log -S <string>` and `git show` on upstream history answer it; the judgment is worth an agent.

## Work on a rung

Editing the **top** rung is trivial — nothing sits above it:

```powershell
git checkout feature/charts-reimagined
# ...commit changes...
git checkout feature/experimental
git merge feature/charts-reimagined         # fast-forward-ish; no --no-ff needed
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

Editing a **middle** rung means everything above it must be replayed:

```powershell
# record every tip first — the ground-truth check needs them
git rev-parse --short feature/workflow-node24 feature/globbing-documentation `
  feature/merchant-composite-keys feature/ui-tweaks feature/charts-reimagined

git checkout feature/ui-tweaks
# ...commit changes...
git rebase --update-refs feature/ui-tweaks feature/charts-reimagined
```

`--update-refs` replays every rung above and **moves each branch ref** in one pass. Then verify and absorb:

```powershell
git diff <old-charts-tip> feature/charts-reimagined --stat    # GROUND-TRUTH CHECK — tip only; base is your own edit
git push --force-with-lease origin feature/charts-reimagined  # and any other moved rung with a PR
git checkout feature/experimental
git merge feature/charts-reimagined
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

The ground-truth diff should show **exactly** the change you just made, nothing more. Anything extra means the rebase resurrected or dropped something.

> **🤖 Work with AI Agent if...**
> - The ground-truth diff contains lines you can't account for. Hand it over *before* the force-push.
> - `--update-refs` stops with conflict markers. Hand over *that conflict* plus which two rungs are involved — the resolution belongs in the upper rung.
> - The merge into `feature/experimental` is clean but `pytest` fails. In this model that is **not** an integration commit — it means the rung itself is wrong. Fix it in the rung and re-propagate.

### Test a local build without committing

For trying an implementation against real data before any commit/push/release. Two tiers:

**Tier 1 — Generate a report from the working tree (eyeball a UI change)**

```powershell
cd C:\BTR\OpenSource\tally
# Throwaway copy — never touches your normal spending_summary.html:
uv run tally up -c C:\BTR\TallySpending\tally\config -o C:\BTR\TallySpending\tally\output\spending_check.html
```

Then open `C:\BTR\TallySpending\tally\output\spending_check.html` in a browser (double-click — `file://` is fine).

- `-c / --config` points at the **config dir** — `...\tally\config`, the folder that holds `settings.yaml`, *not* the `TallySpending` root. The bare positional path is deprecated **and** points one level too high — that path has no `settings.yaml` and errors out.
- `-o / --output` overrides the output path. **Omit it and tally writes to the configured default** (`output_dir: output` → `...\tally\output\spending_summary.html`), overwriting your normal report — so pass `-o ...spending_check.html` when you just want to look.
- The report is rendered from the **working tree** (`src/tally/spending_report.{html,css,js}`), so uncommitted edits appear immediately — re-run to regenerate.
- Iterating on CSS/JS specifically? Add `--no-embedded-html` to emit editable `spending_report.css` / `.js` next to the HTML; edit those and refresh the browser (no regen needed). See `CLAUDE.md → HTML Report Development`.
- `file://` is blocked inside the Playwright MCP sandbox, so agent-driven verification needs a loopback server (`python -m http.server <port> --bind 127.0.0.1` — the `--bind` keeps your personal data off the LAN). A normal browser has no such limit.

**Tier 2 — test as the installed tool.**

```powershell
cd C:\BTR\OpenSource\tally
Copy-Item "$env:LOCALAPPDATA\tally\tally.exe" "$env:LOCALAPPDATA\tally\tally.official.exe"   # backup first
uv run pyinstaller --onefile src/tally/__main__.py --name tally `
  --add-data "src/tally/spending_report.html;tally" `
  --add-data "src/tally/spending_report.css;tally" `
  --add-data "src/tally/spending_report.js;tally"
Copy-Item dist\tally.exe "$env:LOCALAPPDATA\tally\tally.exe" -Force
```

**Restore the official build** when done:

```powershell
Copy-Item "$env:LOCALAPPDATA\tally\tally.official.exe" "$env:LOCALAPPDATA\tally\tally.exe" -Force
```

Notes:
- pyinstaller builds the **working tree** — uncommitted changes included; that's the point.
- The local build skips the workflow's version injection: `tally --version` reports placeholder `0.1.0` — the tell that you're running a local build.
- Don't trust `tally update` from a local build: `REPO_URL` in `_version.py` points at **davidfowl/tally** on every rung (only `feature/experimental` carries the fork-identity commit that redirects it), so it may fetch David's build instead of yours. The backup/restore copy avoids the whole question.
- Either copy fails if tally.exe is currently running — close it first.

## New feature

A new feature always goes on **top of the stack**, never on `main`:

```powershell
git checkout -b feature/<name> feature/charts-reimagined     # current tip
# ...develop, test...
git checkout feature/experimental
git merge feature/<name>
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

Add an inventory row at the bottom of the table. The new branch is now the stack tip.

> **Deliberate consequence:** everything you write from now on sits above every unmerged PR. If David never merges anything, that is fine — the stack still builds and `feature/experimental` still works. It only costs you if you later want to submit the new feature *before* something beneath it, which requires the *rejected* play's restack.

> **🤖 Work with AI Agent if...** the feature really belongs lower in the stack (it's independent of the rungs beneath it and you'd rather submit it sooner). Moving it down is a `git rebase --onto` plus a restack of everything above — hand over the intent, not the branch.

## Submit the next PR

One at a time, from the lowest rung without a merged PR.

```powershell
git push origin <rung>
gh pr create --repo davidfowl/tally --base main --head terryaney:<rung>
```

Because GitHub cross-fork PRs can only target a branch **in the upstream repo**, the PR diff shows every commit from `main` up through that rung — including rungs beneath it that haven't merged yet. That is why only one is open at a time: the next rung's diff is unreadable until the current one lands.

Update the inventory row with the PR link.

## My PR was accepted upstream

The merged rung dissolves into `main` and disappears from every diff above it.

```powershell
git fetch upstream
git checkout main
git merge --ff-only upstream/main
git push origin main

# record all tips before restacking
git rev-parse --short feature/globbing-documentation feature/merchant-composite-keys `
  feature/ui-tweaks feature/charts-reimagined

git rebase --update-refs main feature/charts-reimagined

# GROUND-TRUTH CHECK — base is upstream, so EVERY rung, individually
git diff <old-globbing-tip> feature/globbing-documentation --stat
git diff <old-keys-tip>     feature/merchant-composite-keys --stat
git diff <old-ui-tip>       feature/ui-tweaks --stat
git diff <old-charts-tip>   feature/charts-reimagined --stat

git push --force-with-lease origin <each remaining rung>

git branch -d <merged rung>
git push origin --delete <merged rung>

git checkout feature/experimental
git merge main                              # dedupes automatically, even squash-merges
git merge feature/charts-reimagined
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

Remove the inventory row. Submit the next rung.

If David merged a *modified* version, expect small conflicts only on the hunks he changed — resolve those, never replay anything.

> **🤖 Work with AI Agent if...**
> - `git merge main` conflicts because David merged a **modified** version of your PR. Resolve only the hunks he changed; **never replay** your original version over his. If you catch yourself re-adding your code because the merge "lost" it, stop and hand it over.
> - You can't tell whether a difference is *upstream-intentional* or *accidentally lost*. This is the highest-value handoff in the playbook — the judgment call needs `git log -S <string>` and `git show` on the upstream history before anything is re-added. See the PR #92 lesson under *Upstream main moved*.

## My PR was rejected

Drop the rung; everything above it restacks onto the rung below.

```powershell
git rev-parse --short <every rung above the rejected one>

git rebase --onto <rung below> <rejected rung> <stack tip> --update-refs
git diff <old-tip> <stack tip> --stat        # GROUND-TRUTH CHECK — expect only the rejected rung's code to be gone

git branch -D <rejected rung>
git push origin --delete <rejected rung>
git push --force-with-lease origin <each surviving rung>
```

Then [rebuild `feature/experimental`](#rebuild-featureexperimental) — the merge SHAs it holds no longer exist.

Keep the rejected code if you still want it locally: don't delete the branch, move it *above* the tip instead (`git rebase --onto <tip> <rung below> <rejected rung>`) and mark the inventory row "local only, not for upstream."

> **🤖 Work with AI Agent if...**
> - Upper rungs *depended* on the rejected rung's code. The restack will conflict everywhere, and the right answer may be to fold the rejected rung's essential parts into the upper rung. Hand over the conflict plus what David objected to.
>
> **Not applicable to rung 5 (`feature/merchant-composite-keys`).** Nothing sits above it, so rejection costs one decision — keep it as a fork-only rung or drop it — and zero restacking. That is why it is on top.

## Upstream main moved

```powershell
git fetch upstream
git checkout main
git merge --ff-only upstream/main
git push origin main
```

Then restack. Capture **every** rung tip first — `--update-refs` rebases them all in one pass, so there is no per-rung "before" to compare against afterward unless you recorded it:

```powershell
git rev-parse --short feature/workflow-node24 feature/globbing-documentation `
  feature/merchant-composite-keys feature/ui-tweaks feature/charts-reimagined > ..\tally-notes\pre-rebase-tips.txt

git rebase --update-refs main feature/charts-reimagined

# GROUND-TRUTH CHECK, once per rung, against the recorded tips
git diff <old-node24-tip>  feature/workflow-node24 --stat
git diff <old-globbing-tip> feature/globbing-documentation --stat
git diff <old-keys-tip>     feature/merchant-composite-keys --stat
git diff <old-ui-tip>       feature/ui-tweaks --stat
git diff <old-charts-tip>   feature/charts-reimagined --stat

git push --force-with-lease origin <the rung with the open PR>
```

Then [rebuild `feature/experimental`](#rebuild-featureexperimental).

> **⚠ The ground-truth check is not optional.** A rebase can silently resurrect code that upstream deliberately deleted (proven here: replaying PR #91's first commit re-added an exclusion skip that merged PR #92 had intentionally removed; only the tree-diff caught it). Each diff must be empty, or every line of it must be explainable as intentional. If you can't explain it, hand it to Claude before pushing.
>
> **The stack makes this worse, not better** — one `--update-refs` run can resurrect deleted code at *any* rung, and a bad resolution at rung 2 propagates silently through rungs 3–5. That is the price of the model, and the per-rung diff is the only thing that catches it.

> **🤖 Work with AI Agent if...**
> - **Any ground-truth diff is non-empty and you can't account for every line.** Hand it over *before* you `git push --force-with-lease`.
> - You need to judge *upstream-intentional deletion* vs *accidentally lost code*. The PR #92 lesson: check `git log -S <string>` and `git show` on upstream history **before re-adding anything** a rebase appears to have "dropped".
> - The rebase stops with conflict markers — hand over the one conflict and say which rung it stopped on.

## Include someone else's upstream PR or branch

NOTE: This is a 'rare' situation, but it does happen.

Contrib work does **not** enter the stack — it merges straight into `feature/experimental`, so it can never contaminate a PR diff:

```powershell
git fetch upstream pull/<N>/head:contrib/<N>-<slug>    # for an upstream PR
git fetch upstream <branch>:contrib/<slug>              # for an upstream branch
git checkout feature/experimental
git merge --no-ff contrib/<N>-<slug>
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

This is the **one** exception to "nothing but the fork-identity commit and the stack tip is merged into experimental." Record it in the rebuild play's checklist, because a rebuild will drop it otherwise.

**When it gets new commits** — re-fetch and re-merge; only the new commits flow in:

```powershell
git fetch upstream +pull/<N>/head:contrib/<N>-<slug>   # '+' allows update even if the author force-pushed
git checkout feature/experimental
git merge contrib/<N>-<slug>                            # force-pushed rewrites still dedupe by content
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

**When it merges upstream** — the *My PR was accepted* play applies; the content dedupes on `git merge main`, then delete the `contrib/*` branch.

Don't push `contrib/*` branches to origin — they're not your work; the fetch recreates them anytime.

> **🤖 Work with AI Agent if...**
> - The merge conflicts. **You didn't write this code**, so the "obvious" resolution is less obvious than usual — the bar for handing over is lower here than in any other play. Hand over the conflict *plus* the upstream PR link.
> - The author force-pushed and the re-merge behaves strangely. Content-level dedupe usually handles it, but if the result looks wrong, don't force anything — hand it over.

## Rebuild feature/experimental

Needed whenever the stack's SHAs were rewritten (any rebase). Cheap and scriptable — this is the payoff of the stacked model.

The `fork:` commit sits directly on `main` and is tagged `fork-identity`, so a rebuild is two commands — point the branch back at the tag and re-merge the tip:

```powershell
git checkout -B feature/experimental fork-identity
git merge <current stack tip>
git merge --no-ff <each contrib/* branch still in play>     # rare; see checklist below
uv run pytest tests/ --tb=no -q
git push --force-with-lease origin feature/experimental
```

**When you need this:** only when the stack's SHAs were rewritten — an upstream rebase, or inserting a feature *below* an existing rung. Adding a feature *on top* of the stack needs no rebuild at all; just `git merge feature/<new>` into experimental.

If you'd rather keep a safety net for a risky rebuild, `git branch experimental-old feature/experimental` first and delete it once tests pass.

Checklist of things that live **only** on `feature/experimental` and must survive a rebuild:

1. The fork-identity commit (cherry-pick it — find it with `git log --oneline --grep='^fork:' experimental-old`)
2. Any `contrib/*` merges

Nothing else should ever be there. If `git log main..experimental-old --no-merges` shows anything outside that list, it belongs in a rung and was put in the wrong place.

> **🤖 Work with AI Agent if...**
> - The cherry-picked fork-identity commit conflicts. It shouldn't — if the `build-rc`/`rc-artifacts` rename stayed out of it, `build.yml`/`dev-build.yml`/`release.yml` all auto-merge with `feature/workflow-node24`. A conflict here means the rename crept back in.
> - `git log main..experimental-old --no-merges` shows commits that aren't on the checklist. Those are lost work — figure out which rung they belong to before you delete `experimental-old`.

## Restart checklist — DELETE THIS SECTION WHEN DONE

One-time migration from parallel-off-main branches to the stack. Pure procedure; the *why* lives in the sections above. Work top to bottom, tick as you go, stop on any **STOP**.

Everything runs in `C:\BTR\OpenSource\tally-stacked`. `C:\BTR\OpenSource\tally` is the archive — untouched, still your working build, and your safety net until step 9.

**Progress:** ☑ 0 · ☑ 1 · ☐ 2 · ☐ 3 · ☐ 4 · ☐ 5 · ☐ 6 · ☐ 7 · ☐ 8 · ☐ 9 · ☐ 10

**Before any `rebase -i`:** `echo $env:GIT_EDITOR` must be empty. If it prints anything, `Remove-Item Env:GIT_EDITOR`. Otherwise git accepts the todo list unchanged and the rebase silently does nothing.

**How `rebase -i` works:** VS Code opens the commit list, oldest first, with a dropdown on each line (Pick / Squash / Fixup / Edit / Drop). Change the first word or use the dropdown, `Ctrl+S`, then `Ctrl+W` to close the tab — git waits for the close.

| Mark | Effect |
|---|---|
| `pick` | keep as its own commit |
| `s` squash | fold into the line above; **opens one editor with every folded message** so you can write the combined message |
| `f` fixup | fold into the line above and discard its message silently |
| `e` edit | stop here so you can change something, then `git rebase --continue` |

These checklists use **`s`** — after you close the todo list, a second editor opens containing all the squashed messages under a `# This is a combination of N commits.` header. Delete what you don't want, write the real subject on line 1, save, close. No separate `git commit --amend` needed.

Recover from any mess with `git reset --hard old/<branch>`, then retry.

**The verification helper.** Paste this once per shell session; every rung uses it:

```powershell
function TreeCheck($oldBase, $below, $oldTip, $newTip) {
  $exp = (git merge-tree --write-tree --merge-base=$oldBase $below $oldTip) -split "`n" | Select-Object -First 1
  $act = git rev-parse "$newTip^{tree}"
  if ($exp -eq $act) { "ok  $newTip" } else { "DIFFERENT - STOP  $newTip" }
}
```

It builds the tree the new rung *should* have (rung below + the old rung's own changes) and compares tree hashes. Exact, and unaffected by squashing.

> **`$oldBase` is the OLD branch's base, and `--merge-base` is not optional.** Let git infer it and you get `main`, which double-applies everything the old branch was stacked on. Real example: checking rung 4 without it produced an "expected" tree with **412 extra lines** in `spending_report.*` and the test files — code `charts-reimagined` had deliberately deleted, resurrected because `ui-tweaks` was merged in twice. That is the PR #92 hazard wearing a different hat.
>
> `$oldBase` is `main` for rungs 1, 2, 3 and 5 — but `old/feature/ui-tweaks` for rung 4, because `old/feature/charts-reimagined` was branched from it.

> Do **not** verify by comparing `git diff` output between old and new branches. Patch text carries `index <hash>..<hash>` lines and `@@` line numbers that shift whenever two rungs touch the same file — you get differences that mean nothing. (Real example: `globbing` adds 5 lines to `config/settings.yaml.example` above a `ui-tweaks` hunk in that same file, producing 6 phantom differences across 4,438 identical content lines.) Compare trees, not patches.

---

### 0. Setup ☑

```powershell
git clone https://github.com/terryaney/OpenSource.tally.git C:\BTR\OpenSource\tally-stacked
cd C:\BTR\OpenSource\tally-stacked
git remote add old C:\BTR\OpenSource\tally
git remote add upstream https://github.com/davidfowl/tally
git fetch old
git fetch upstream
git config rerere.enabled true
git config rebase.updateRefs true

uv sync --extra dev                            # pytest is an optional extra, NOT installed by `uv run`
uv run pytest tests/ --tb=no -q                # baseline: main must be green before building anything

git rev-parse main upstream/main old/main      # three identical SHAs
```

**STOP if** the SHAs differ, a fetch errored, or the baseline tests fail.

> A fresh clone's `.venv` has runtime deps only. Without `uv sync --extra dev` you get `error: Failed to spawn: pytest — program not found`. Add `--extra build` as well when you need pyinstaller for a Tier 2 local build.

---

### 1. Rung 1 — `feature/workflow-node24` ☑

```powershell
git checkout -b feature/workflow-node24 main
git cherry-pick main..old/feature/workflow-node24

git range-diff main...old/feature/workflow-node24 main...feature/workflow-node24
git log --oneline main..feature/workflow-node24                    # 1
```

**STOP if** conflict, any non-`=` line, or count ≠ 1. No squash — already one commit.

---

### 2. Rung 2 — `feature/globbing-documentation` ☐

```powershell
git checkout -b feature/globbing-documentation feature/workflow-node24
git cherry-pick main..old/feature/globbing-documentation

git range-diff main...old/feature/globbing-documentation `
               feature/workflow-node24...feature/globbing-documentation
git log --oneline feature/workflow-node24..feature/globbing-documentation   # 3
```

**STOP if** conflict, any non-`=` line, or count ≠ 3.

```powershell
git rebase -i feature/workflow-node24
```

```
pick  <sha>  docs/tests: document and validate glob file patterns
s     <sha>  tests: stabilize glob CLI assertions
s     <sha>  tests: fix single-file glob fallback expectation
```

A second editor opens with all three messages — edit down to one, save, close.

```powershell
TreeCheck main feature/workflow-node24 old/feature/globbing-documentation feature/globbing-documentation
git log --oneline feature/workflow-node24..feature/globbing-documentation    # 1
```

**STOP if** it prints `DIFFERENT` or count ≠ 1. Do not start step 3 until both pass.

---

### 3. Rung 3 — `feature/ui-tweaks` ☐

```powershell
git checkout -b feature/ui-tweaks feature/globbing-documentation
git cherry-pick main..old/feature/ui-tweaks

git range-diff main...old/feature/ui-tweaks `
               feature/globbing-documentation...feature/ui-tweaks
git log --oneline feature/globbing-documentation..feature/ui-tweaks          # 12
uv run pytest tests/ --tb=no -q
```

**STOP if** conflict, any non-`=` line, count ≠ 12, or tests fail.

```powershell
git rebase -i feature/globbing-documentation
```

```
pick  1c7c046  UI Tweaks
s     eb5b245  UI Tweaks - Report Fields Setting
s     1929f38  Fix two data-correctness bugs, then polish the charts
s     346cbca  Transaction Details - container + collapser polish
s     e004d7e  Fix view-toggle cascade in Transaction Details
s     3342761  Transaction Details Mobile Wrap
s     e43dd78  Polish Row UX and Transaction Column Sizing
s     2b56528  Save Layout Settings to Local Storage
s     91e8f8d  Fixed 'report fields' badge rendering
s     dfece33  Remove dead getTransactionYears/showYear plumbing
s     f42e42b  Fix merchant filter regression from row-UX/date polish
s     2538433  Margin on field/memo badge
```

*(SHAs shown are the originals; yours will differ after the cherry-pick — match on subject line.)*

The combined-message editor opens with all 12 messages. Write the real summary on line 1 and **keep the two data-correctness bug fixes from `1929f38` in the body** — they vanish from the log otherwise, and a commit titled "UI tweaks" that quietly changes reported numbers is a reviewer-trust problem.

```powershell
TreeCheck main feature/globbing-documentation old/feature/ui-tweaks feature/ui-tweaks
git log --oneline feature/globbing-documentation..feature/ui-tweaks          # 1
uv run pytest tests/ --tb=no -q
```

**STOP if** it prints `DIFFERENT`, count ≠ 1, or tests fail.

---

### 4. Rung 4 — `feature/charts-reimagined` ☐

```powershell
git checkout -b feature/charts-reimagined feature/ui-tweaks
git cherry-pick old/feature/ui-tweaks..old/feature/charts-reimagined

git range-diff old/feature/ui-tweaks...old/feature/charts-reimagined `
               feature/ui-tweaks...feature/charts-reimagined
git log --oneline feature/ui-tweaks..feature/charts-reimagined               # 12
uv run pytest tests/ --tb=no -q
```

**STOP if** conflict, any non-`=` line, count ≠ 12, or tests fail.

**4a — squash 12 → 2:**

```powershell
git rebase -i feature/ui-tweaks
```

```
pick  3baa99a  Added new reimagined chart(s) layout
s     08f78fe  Added chart tests
s     4d72e79  Improve responsive label behavior, resize rerendering, compare-year paging
s     13add0c  Removing empty xaxis items
s     848f6c3  Documentation Updates
pick  99104c3  Restore KPI test hooks for report compatibility
s     4a2e2fb  KPI Sparklines are based on last month containing data
s     72e035a  KPI mistakenly used 'transfer only' months as anchor
s     9078a1d  Changed KPI to be month based instead of all time
s     9b88d67  Reconcile KPI detail math with trend baseline
s     897b37f  Exclude transfer-only merchants from fixed outputs
s     daebc8f  Stabilize chart behavior and default-hide empty legend series
```

**Two** combined-message editors open in sequence — one per group. Suggested subjects: *"Add reimagined chart layout"* and *"Correct KPI math and stabilize chart rendering"*.

**4b — split the layout commit by path (it is 31 files / +3,647 lines):**

```powershell
git rebase -i feature/ui-tweaks       # mark the LAYOUT commit `e`, leave the KPI commit `pick`
```

Git stops with the layout commit applied. Then:

```powershell
git rev-parse HEAD                    # record this SHA
git reset HEAD~1

git add -A src tests config
git commit -m "Add reimagined chart layout"

git add -A docs
git commit -m "Document chart gallery and refresh screenshots"

git status --short                    # EMPTY
git diff <recorded-sha> HEAD          # EMPTY
git rebase --continue
```

`-A` is required — it stages the `docs/demo.gif` rename and the `docs/screenshot.png` deletion. Plain `git add docs` misses both.

```powershell
TreeCheck old/feature/ui-tweaks feature/ui-tweaks old/feature/charts-reimagined feature/charts-reimagined
git log --oneline feature/ui-tweaks..feature/charts-reimagined               # 3
git status --short                                                            # EMPTY
uv run pytest tests/ --tb=no -q
```

**STOP if** it prints `DIFFERENT`, count ≠ 3, status shows anything, or tests fail.

---

### 5. Rung 5 — `feature/merchant-composite-keys` ☐

**Do not cherry-pick this rung.** Replaying `old/issue/88` onto the stack means hand-merging ui-tweaks' merchant-cell restructure against markup written for main — hundreds of conflicted lines to arrive at a 13-line result. `old/feature/experimental` already holds that reconciliation, tested: it is 88 plus its four integration commits, applied to exactly this code.

Take the reconciled state directly. These 8 files are the complete rung-5 delta (~711 lines):

```powershell
git checkout -b feature/merchant-composite-keys feature/charts-reimagined

git checkout old/feature/experimental -- `
  src/tally/analyzer.py src/tally/commands/explain.py src/tally/report.py `
  src/tally/spending_report.js tests/test_analyzer.py tests/test_cli.py `
  tests/test_report.py tests/test_report_html.py

git status --short                      # exactly those 8, all M or A
uv run pytest tests/ --tb=no -q
git commit -m "Fix incorrect category display for duplicate merchant names"

git diff feature/merchant-composite-keys old/feature/experimental --stat
#   → ONLY .github/workflows/*, docs/index.html, _version.py, cli.py, .vscode/**
```

Safe because the only difference between `feature/charts-reimagined` and `old/feature/experimental` is 88 + its integration commits + fork identity + notes — and fork identity touches none of those 8 files. That last diff is the proof, and it is the same check step 8 runs.

Split `src/` from `tests/` afterward if you want two commits for review (same path-split recipe as step 4b).

`cc9872c` (date tests) needs **no placement — it is redundant**. Verified: its assertions are already in `feature/charts-reimagined`, arriving via `f42e42b` in ui-tweaks. It was the same fix authored twice, once on experimental and once on the branch. Ignore it.

**STOP if** a cherry-pick conflicts or tests fail. Hand over the conflict plus the *why* body of the commit being applied.

---

### 6. Fork identity + `feature/experimental` ☐

```powershell
git checkout -b feature/experimental main
git cherry-pick 058963d 4e84065 bd97e91
```

- `058963d` workflows + `_version.py` REPO_URL (73 lines)
- `4e84065` `docs/index.html` + `cli.py` description (2 lines)
- `bd97e91` `release.yml` workflow-file lookup (2 lines)

Then edit `.github/workflows/dev-build.yml` and change **exactly two lines** (~line 69) back to David's wording:

```yaml
      # Download RC artifacts
      - name: Download RC artifacts
```

> **Change nothing else in that file.** Those two lines sit immediately above `uses: actions/download-artifact`, which rung 1 bumps `@v4` → `@v8` — that adjacency is the *only* thing that ever conflicted. The job name `build-release`, the `needs:` lists, `path: release-artifacts`, `pattern: tally-*-experimental-release`, the echo text and the zip glob all merge cleanly and **must stay as-is**. Reverting `pattern:` to `tally-*-rc` breaks artifact matching outright: `build.yml` emits `tally-<platform>-experimental-release`, so the download finds nothing and the release job fails with zero artifacts.

There is no `paths-ignore` to remove — it existed only in `f5a773a`, which this step skips.

```powershell
git commit -am "fork: keep upstream RC wording on the download step"
git rebase -i main                    # squash all 4 into "fork: identity and experimental build wiring"
git tag fork-identity                 # rebuilds reset to this tag — do not skip
git merge feature/merchant-composite-keys
uv run pytest tests/ --tb=no -q
```

**Never build this commit by diffing against `main`** — `old/feature/experimental-foundational` is based on `main~1`, so a diff shows 11 files including a revert of merged PR #92. The true content is 6 files, +75/−68. Cherry-pick is immune.

**STOP if** the cherry-picks conflict, or `git log main..feature/experimental --no-merges` shows anything but the one `fork:` commit.

---

### 7. `playbook` orphan branch ☐

```powershell
git checkout --orphan playbook
git rm -rf .
```

Copy from `C:\BTR\OpenSource\tally-notes\.vscode\` into `.vscode\`: `Plans\`, `branches.md`, `wishList.md`. (This file becomes the new `branches.md` — delete the old one and this section with it.)

```powershell
git add -A
git commit -m "playbook: notes, plans, prototypes"
git push origin playbook
git worktree add ..\tally-notes-new playbook
```

---

### 8. Acceptance — all four must pass ☐

```powershell
uv run pytest tests/ --tb=no -q

# the stack reproduces the old build, modulo fork identity
git diff feature/merchant-composite-keys old/feature/experimental --stat
#   → ONLY fork-identity files and .vscode/**

# each rung's tree matches what it should be (see TreeCheck in the primer above)
TreeCheck main                  main                           old/feature/workflow-node24        feature/workflow-node24
TreeCheck main                  feature/workflow-node24        old/feature/globbing-documentation feature/globbing-documentation
TreeCheck main                  feature/globbing-documentation old/feature/ui-tweaks              feature/ui-tweaks
TreeCheck old/feature/ui-tweaks feature/ui-tweaks              old/feature/charts-reimagined      feature/charts-reimagined
#   → all four "ok" (rung 5 is not checkable this way — it absorbed the integration commits)

# the build works
git checkout feature/experimental
uv run tally up -c C:\BTR\TallySpending\tally\config -o C:\BTR\TallySpending\tally\output\spending_check.html
```

Final shape: 6 commits across rungs 1–4 (1 + 1 + 1 + 3), plus rung 5 and the `fork:` commit.

**STOP if** anything fails. Do not proceed to step 9 — the archive is still your only good copy.

---

### 9. Cutover ☐

```powershell
git push origin feature/workflow-node24 feature/globbing-documentation `
                feature/ui-tweaks feature/charts-reimagined `
                feature/merchant-composite-keys feature/experimental --force-with-lease
```

Close VS Code, then:

```powershell
Rename-Item C:\BTR\OpenSource\tally         tally-archive
Rename-Item C:\BTR\OpenSource\tally-stacked tally
Rename-Item C:\BTR\OpenSource\tally-notes-new tally-notes
Remove-Item C:\BTR\OpenSource\tally-notes -Recurse -Force    # the OLD worktree
```

Reopen `Tally.code-workspace`. Then rebuild what a fresh clone loses — see *What a fresh clone loses*.

---

### 10. Upstream ☐

Close PRs #91, #93, #94, #95, copying anything worth keeping into `.vscode\Plans\` first. Then open one PR from `feature/workflow-node24` — it fixes David's broken `build` job, so it is the easiest thing he will ever be asked to merge.

Delete `tally-archive` only after the first rung merges. Then delete this section.

---

> **🤖 Hand over when:** a rung 1–4 cherry-pick conflicts (the premise says they can't — something is wrong); a tree diff is non-empty; rung 5 conflicts and the integration commit's *why* body doesn't resolve it; step 8's first check lists code you don't recognise (work that existed only on `old/feature/experimental` and is about to be lost).
## When to abandon upstream

**Not a trigger: David going quiet.** His stated cadence is *"my interest spikes when I need to run it, maybe 4 times a year"* (2026-07-14). Months of silence is the documented normal, not a signal. Do not restructure anything because a PR sat unreviewed.

**Real triggers, in order of cost:**

| Event | Response | Cost |
|---|---|---|
| Rung rejected, nothing above depends on it | *My PR was rejected* play | minutes |
| Rung rejected, upper rungs depend on it | Same play, but fold essentials upward | hours |
| Upstream main moved | *Upstream main moved* play | minutes, plus per-rung ground-truth checks |
| David merges a *modified* version of a rung | *My PR was accepted*, resolve only his hunks | hours, highest judgment risk |
| **You decide the tax isn't worth it** | Collapse: merge the whole stack into one branch, stop rebasing on upstream, stop opening PRs | one afternoon |

The last row is always available and costs nothing to defer. Nothing in this model traps you upstream — the stack is a *presentation* choice, and `feature/experimental` is a working build regardless of whether a single PR ever merges.

---

# Local machine setup

## The two worktrees

| Path | Branch | Role |
|---|---|---|
| `C:\BTR\OpenSource\tally` | whatever rung you're developing on | all code work; runs the playbook |
| `C:\BTR\OpenSource\tally-notes` | `playbook` (**permanently**) | this file, `.vscode/Plans/`, `wishList.md` — always visible, never checked out anywhere else |

Both are opened together via `C:\BTR\Extensibility\Tally.code-workspace` (a VS Code multi-root workspace). Worktrees share one `.git`, so refs, objects, `rerere` state, and `.git/info/exclude` are common to both — but HEAD and the index are per-worktree, so committing notes never disturbs a half-finished feature.

> **⚠ `playbook` is pinned to the notes worktree.** Git refuses to check out one branch in two worktrees. `git checkout playbook` in the main worktree **will fail** — that's by design, not breakage. All commits to it happen in `tally-notes`.

> **`playbook` is an orphan branch.** It shares no commits with `main` and merges into nothing. That is deliberate: notes cannot reach a PR diff, cannot trigger a build, and cannot bloat the stack. It also means `git log main..playbook` is meaningless and `git merge playbook` should never be run.

## What is deliberately never committed

`.claude/` (Claude Code settings, permissions, agent state) is excluded via **`.git/info/exclude`**, not `.gitignore` — because `.gitignore` is tracked per-branch, and a rule committed to one rung would not be in effect while another is checked out. `.git/info/exclude` lives in the shared `.git` dir, so one entry applies to every branch **and** both worktrees, and can never itself leak into an upstream PR.

## ⚠ Failure mode: ignored files are silently clobbered

Git treats ignored files as expendable. If upstream ever commits a file at a path you hold as ignored-and-untracked — say David adds `.claude/settings.json` and you have one — then `git merge upstream/main` **overwrites yours silently**: fast-forward, exit 0, no prompt. Verified behaviour, not theory:

| Your local file | Upstream adds same path | Result |
|---|---|---|
| untracked, **not** ignored | | Merge **aborts**: "untracked working tree files would be overwritten". You are safe. |
| untracked **and ignored** | | **Silently overwritten.** Content is gone — never tracked, so no blob, no reflog, no recovery. |

Accepted knowingly: the trade is that `.claude/` can never be swept into a commit by `git add .`, which matters more. Real exposure is small — the only file held there is `settings.local.json`, which by convention nobody commits. The risk grows if `.claude/agents/*` or `.claude/commands/*` are added locally and upstream adds identically-named ones. **If a pull ever quietly resets Claude config, this is why.**

## What a fresh clone loses

None of this is in git. After a re-clone, rebuild by hand:

```powershell
# 1. .git/info/exclude — re-add:  .claude/   (plus any **/.claude/* runtime entries)
# 2. the notes worktree
git worktree add ../tally-notes playbook
# 3. rerere (the stack model depends on it — every rebase replays old resolutions)
git config rerere.enabled true
# 4. remotes
git remote add upstream https://github.com/davidfowl/tally
# 5. rebase.updateRefs so plain `git rebase` doesn't strand mid-stack refs
git config rebase.updateRefs true
```

`C:\BTR\Extensibility\Tally.code-workspace` and `.claude/settings.local.json` (which grants agents access to `../tally-notes/` via `permissions.additionalDirectories`) are also untracked — recreate or restore them from backup.
