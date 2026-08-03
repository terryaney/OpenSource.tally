# Branch Inventory & Playbook

Local workflow documentation for the tally fork.

Lives on the `playbook` branch — an orphan branch with no shared history with `main`, permanently checked out in the `tally-playbook` worktree. It merges nowhere, and `git merge playbook` refuses by default, so nothing here reaches an upstream PR by accident. See *Local machine setup*.

## Current Inventory

Row order = stack order, bottom to top. Each rung's base is the row above it.

| # | Branch | Base | Upstream PR | Notes |
|---|--------|------|-------------|-------|
| 1 | `feature/workflow-node24` | `main` | *(next to submit)* | Action version bumps + `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24`. Fixes upstream's broken `build` job — genuinely useful to David, smallest possible diff. |
| 2 | `feature/globbing-documentation` | rung 1 | | Docs + tests only |
| 3 | `feature/ui-tweaks` | rung 2 | | Authored against string-keyed merchants — independent of rung 5 |
| 4 | `feature/charts-reimagined` | rung 3 | | |
| 5 | `feature/merchant-composite-keys` | rung 4 | | |
| 6 | `feature/categorization` | rung 5 | | |

**Contrib branches in play:** none. *(`contrib/*` branches merge into `feature/experimental` only and are never rungs — they have no base in the stack, so they are tracked here rather than as table rows. A rebuild must replay every one listed.)*

> **Renaming is free** as long as no PR is open from the branch. Once a PR is open, its head branch cannot be renamed without closing the PR — so rename *before* you submit, never after.

---

## Repository Structure

```
main (based on upstream/main)
 │
 ├── feature/workflow-node24                    				rung 1
 │     └── feature/globbing-documentation       				rung 2
 │           └── feature/ui-tweaks              				rung 3
 │                 └── feature/charts-reimagined 				rung 4
 │                       └── feature/merchant-composite-keys  	rung 5
 │                             └── feature/categorization       rung 6  ← stack tip
 │
 ├── feature/experimental = main
 │                        + fork-identity commit
 │                        + merge(stack tip)
 │
 └── playbook  (notes, plans, prototypes; merges nowhere)
```

Three rules the whole document follows from:

1. **Every feature branch is based on the rung below it**, never on `main`.
2. **`feature/experimental` is `main` + one fork-identity commit + a merge of the stack tip.** The only other thing ever merged into it is a `contrib/*` branch — see *Include someone else's upstream PR or branch*.
3. **PRs are submitted bottom-up**, each declaring the rung it sits on. Batch them as much as possible given frequency upstream reviews; do not serialize.

---

# Playbook

Run everything yourself; each play is a handful of git commands. Each play flags its own **🤖 Work with AI Agent if...** moments inline, at the step where they bite.

Two rules for those handoffs, everywhere they appear: hand over the **specific conflict or failure** — the conflict hunk, the failing test output, the unexplained diff — and **never the whole play**.

**Notation used throughout.** Substitute from the inventory table; nothing below hardcodes today's branch names.

| Placeholder | Means | Today |
|---|---|---|
| `<tip>` | top rung of the stack | `feature/merchant-composite-keys` |
| `<rung>` | the rung you are acting on | any row |
| `<rung-below>` | the row directly above it in the inventory (its base) | — |
| `<rung-above>` | the row directly below it in the inventory | — |

`<rung-below>` for rung 1 is `main`. `<rung-above>` for `<tip>` does not exist — that is what makes the tip cheap to work on.

- [Plan a feature with an AI agent](#plan-a-feature-with-an-ai-agent)
- [New feature](#new-feature)
- [Work on an existing rung](#work-on-an-existing-rung)
- [Submit PRs](#submit-prs)
- [Upstream main moved](#upstream-main-moved) — also covers **my PR was accepted upstream**
- [Include someone else's upstream PR or branch](#include-someone-elses-upstream-pr-or-branch)
- [My PR was rejected](#my-pr-was-rejected)
- [Squash a rung](#squash-a-rung)
- [Rebuild feature/experimental](#rebuild-featureexperimental)
- [When to abandon upstream](#when-to-abandon-upstream)

## Plan a feature with an AI agent

Plans, prototypes and research notes live in **`tally-playbook\.vscode\Plans\`** — on the `playbook` branch, tracked, in the notes worktree. Never in the main worktree.

```
.vscode\Plans\
  <topic>.md                 # loose plans
  Features\<Feature>.md      # per-feature plans
  Prototypes\<thing>.html    # throwaway protos
```

Point the agent at that path and let it read and write there directly while you develop on any rung in the main worktree.

Why this and not "write the plan into the feature branch and promote it later":

- **It cannot leak by accident.** `playbook` shares no history with `main`, so `git merge playbook` refuses outright — getting a plan into a rung takes a deliberate `--allow-unrelated-histories` or `cherry-pick`. The guard is git's, not a rule you have to remember.
- **Always visible.** It survives every `git checkout` in the main worktree, because it isn't in the main worktree.
- Use unique, descriptive filenames — the directory is shared across all features, not per-branch. Commit notes whenever; it never disturbs in-flight feature work.

## New feature

A new feature always goes on **top of the stack**, never on `main`:

```powershell
git checkout -b feature/<name> <tip>
# ...develop, test...
git checkout feature/experimental
git merge feature/<name>
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

Add an inventory row at the bottom of the table. `feature/<name>` is now `<tip>`.

> **Deliberate consequence:** everything you write from now on sits above every unmerged PR. If upstream never merges anything, that is fine — the stack still builds and `feature/experimental` still works. It only costs you if you later want to submit the new feature *before* something beneath it, which requires the *rejected* play's restack.

> **🤖 Work with AI Agent if...** the feature really belongs lower in the stack (it's independent of the rungs beneath it and you'd rather submit it sooner). Moving it down is a `git rebase --onto` plus a restack of everything above — hand over the intent, not the branch.

## Work on an existing rung

### Editing `<tip>`

Nothing sits above it, so there is nothing to replay.

```powershell
git checkout <tip>
# ...commit changes...
git checkout feature/experimental
git merge <tip>
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
git push --force-with-lease origin <tip>        # only if it has an open PR
```

### Editing any lower `<rung>`

Every rung above must be replayed onto your edit. One `--update-refs` run does all of them.

**1. Record the tip — and `<rung>` itself.** `--update-refs` overwrites every rung ref in one pass, so there is no "before" afterwards unless you captured it now. `<rung>`'s own hash is only needed by the amend variant below, but an amend destroys it too — capture both and throw one away.

```powershell
git rev-parse --short <tip>                     # write this down
git rev-parse --short <rung>                    # <rung-before> — write this down too
```

**2. Edit and replay.** Which command you use depends on whether you **added** a commit or **amended** the existing one. `<rung>` is where you committed; `<tip>` tells the rebase how far up to replay. Every rung between them moves too.

**2a. You added a commit.** The rung's old tip is still an ancestor of your new one, so `<rung>..<tip>` contains only the rungs above:

```powershell
git checkout <rung>
# ...commit changes...
git rebase --update-refs <rung> <tip>
```

**2b. You amended the rung's last commit.** The pre-amend commit is *still an ancestor of `<tip>`* — nothing above has been replayed yet — so `<rung>..<tip>` still contains it. The 2a command would replay the **pre-amend** version on top of the **post-amend** one; patch-id dedupe can't drop it, because the amend is exactly what changed its content. Use `--onto` to separate where the replay lands from where it starts:

```powershell
git checkout <rung>
# ...commit --amend...
git rebase --update-refs --onto <rung> <rung-before> <tip>
```

Same shape as the *rejected* play's step 2: `--onto <lands here> <starts here, exclusive> <replays up to here>`.

> **⚠ Symptom you reached for 2a on an amend:** the rebase stops with `could not apply <old-hash>... <the commit message you just rewrote>`, conflicting in every file the amend touched. That is the old commit being replayed onto its own replacement. `git rebase --abort` and re-run with 2b — nothing is lost, and the abort restores every ref.
>
> Reading the conflict as real and resolving it is the trap: you would be hand-merging two versions of your own commit, and the result is whatever survives that merge rather than what you amended to.

**3. Verify — `<tip>` only.** The base of this replay is your own edit, so you already know what should have changed.

```powershell
git diff <recorded-tip> <tip> --stat
```

**Expect:** exactly the change you just made, nothing else.
**STOP if:** anything else appears. The rebase dropped or resurrected something.

**4. Absorb and push.**

```powershell
git push --force-with-lease origin <every rung from `<rung>` up to `<tip>` that has an open PR>
git checkout feature/experimental
git merge <tip>
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

Rungs *below* `<rung>` did not move — do not push them.

> **Worked example — 2a, adding a commit to rung 3 of 5.** `<rung>` = `feature/ui-tweaks`, `<tip>` = `feature/merchant-composite-keys`.
> Rung 3 moves when you **commit** to it in step 2. `git rebase --update-refs feature/ui-tweaks feature/merchant-composite-keys` then replays rungs 4 and 5 onto it and moves those two refs. Rungs 1–2 are untouched. Step 4 pushes rungs 3, 4 and 5.

> **Worked example — 2b, amending rung 4 of 6.** `<rung>` = `feature/charts-reimagined`, `<rung-before>` = `2477dea`, `<tip>` = `feature/categorization`.
> After `git commit --amend`, rung 4 points at the new commit but rungs 5 and 6 still descend from `2477dea`. `git rebase --update-refs --onto feature/charts-reimagined 2477dea feature/categorization` replays rungs 5 and 6 — `2477dea` itself excluded — onto the amended commit and moves both refs. Rung 4's ref is the `--onto` target, so `--update-refs` leaves it alone. Rungs 1–3 are untouched.
> Running 2a here instead (`git rebase --update-refs feature/charts-reimagined feature/categorization`) puts `2477dea` back in the range and conflicts immediately — this is the mistake the ⚠ above describes, and it has actually happened.

> **🤖 Work with AI Agent if...**
> - Step 3's diff has lines you can't account for. Hand over *that diff*, **before** the force-push.
> - `--update-refs` stops with conflict markers. Hand over the conflict plus which two rungs are involved — the resolution belongs in the upper rung. **Include the `could not apply <hash>... <subject>` line** — if that subject is a commit you just amended, the answer is the 2a/2b mix-up, not a real conflict.
> - The merge into `feature/experimental` is clean but `pytest` fails. That means the rung itself is wrong. Fix it in the rung and re-run from step 2.

### Test a local build without committing

For trying an implementation against real data before any commit/push/release. Two tiers:

**Tier 1 — test as the installed tool.**

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

**Tier 2 — Generate a report from the working tree (eyeball a UI change)**

```powershell
cd C:\BTR\OpenSource\tally
# Throwaway copy — never touches your normal spending_summary.html:
uv run tally up -c C:\BTR\TallySpending\tally\config -o C:\BTR\TallySpending\tally\output\spending_check.html
```

- Iterating on CSS/JS specifically? Add `--no-embedded-html` to emit editable `spending_report.css` / `.js` next to the HTML; edit those and refresh the browser (no regen needed). See `CLAUDE.md → HTML Report Development`.
- `file://` is blocked inside the Playwright MCP sandbox, so agent-driven verification needs a loopback server (`python -m http.server <port> --bind 127.0.0.1` — the `--bind` keeps your personal data off the LAN). A normal browser has no such limit.

## Submit PRs

Every PR targets `main`, so each diff is cumulative: rung 3's PR shows rungs 1–3. As long as rungs stay squashed to a commit or two each, that is a sentence of explanation, not a wall — **submit in batches, do not serialize**. Upstream visits are quarterly; one merge per visit is a decade.

```powershell
git push origin <rung>
gh pr create --repo davidfowl/tally --base main --head terryaney:<rung> `
  --title "<title>" --body-file .vscode\Plans\PRs\<rung>.md
```

**Per-layer diff links.** A compare URL between two of your own branches renders only the upper one's commits — the isolated view a native stack would give:

```
https://github.com/terryaney/OpenSource.tally/compare/<rung below>...<this rung>
```

Bottom rung uses `main...<rung>`. Branch names containing `/` work as-is; no escaping needed.

**PR description template** — first two lines carry the dependency, so it is legible without opening anything:

```markdown
**Stacked on `feature/globbing-documentation`** — please merge that one first.
Standalone diff for this layer only: [feature/globbing-documentation...feature/ui-tweaks](https://github.com/terryaney/OpenSource.tally/compare/feature/globbing-documentation...feature/ui-tweaks)

## What this does

<one paragraph>

## Why it's separate

<why this is its own layer rather than folded into the one below>

## Coming next (not in this PR)

- [`feature/charts-reimagined`](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined) — reimagined chart layout, KPI tiles, chart docs
- [`feature/merchant-composite-keys`](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys) — composite merchant identity so one merchant can carry multiple categorizations
```

**Conventions**

- Branch names are `feature/<kebab-slug>` and never change once a PR is open — renaming a head branch closes the PR. Rename *before* submitting.
- PR title = simple descriptive title. No `[1/5]` or `Part 2:` prefixes; the "Stacked on" line carries the ordering and survives reordering.
- Drop the "Stacked on" line for the bottom rung; drop "Coming next" when nothing above is ready to preview. When a rung merges upstream, the rung above it becomes `main...<rung>`.
- Update the inventory row with the PR link as soon as it is open.

## Upstream main moved

Covers both cases, because they are the same event and can happen together: `git fetch upstream` returns commits, and one of them may or may not be a rung of yours. Steps 3 and 6 are gated — step 2 tells you whether they apply, so don't decide up front.

**1. Record every rung tip and old `main`.** `--update-refs` overwrites every rung ref in one pass, and `main` is about to fast-forward; without these there is nothing to compare against.

```powershell
git rev-parse --short main                       # <old-main> — write this down

git branch --format='%(refname:short)' |
  Where-Object { $_ -notin 'main','playbook','feature/experimental' -and $_ -notlike 'contrib/*' } |
  ForEach-Object { "{0,-40} {1}" -f $_, (git rev-parse --short $_) } |
  Tee-Object ..\tally-playbook\pre-rebase-tips.txt
```

Every local branch except `main`, `playbook`, `feature/experimental` and `contrib/*` is a rung, so this needs no editing when the stack changes. `contrib/*` branches are excluded deliberately — they merge into the build branch, never into the stack, so they are not rebased and their SHAs are not verified here. Output is alphabetical, not stack order — it is a lookup table for step 5, not a sequence.

**2. Take upstream's new main, then check what landed.**

```powershell
git fetch upstream
git checkout main
git merge --ff-only upstream/main
git push origin main

git log --oneline <old-main>..main               # what upstream landed
git cherry -v main <rung>                        # repeat per rung — '-' = already upstream
```

**`git log` alone does not prove a rung landed.** Upstream squashes, renames and edits, so your rung's subject may not appear at all. `git cherry` compares patch IDs and is reliable in one direction:

| `git cherry` output | Meaning |
|---|---|
| `-` | that commit is already upstream, applied **verbatim** — the rung merged |
| `+` | either not merged, **or** merged with modifications |

A `-` → steps 3 and 6 apply to that rung. All `+` and nothing recognisable in the log → skip both. A `+` on a rung you *believe* merged is exactly the modified-merge case; step 3 will show you what changed.

**3. Only if rungs of yours merged — check whether upstream modified them.** Otherwise go to step 4.

Do this before restacking; a rebase over a modified merge is where upstream's edits get silently reverted. **Run it once per merged rung** — batched submissions mean two or more can land in the same window.

```powershell
git diff --name-only <old-main> <merged-rung>    # files that PR touched — keep this list
git diff <merged-rung> main -- <those files>     # how upstream's version differs from yours
```

**Empty** → merged verbatim. Go to step 4, nothing special.
**Non-empty** → every line is an edit upstream made. Read it now; you will hit these as conflicts in step 4. Three rules, in priority order:

1. **Upstream's version wins on any line they changed.** Even if yours looks better — reopen it as a follow-up rung instead of quietly restoring it.
2. **Your version wins only where a *later rung* deliberately changed the same line** — the upper rung's own work, not a replay of the merged rung's.
3. **Never re-add something the merge appears to have "lost."** A rebase replays your old commits, so code deleted on purpose comes back looking like a conflict resolution. Confirm intent first:

```powershell
git log -S "<the exact line>" --oneline upstream/main
git show <that commit>
```

Deleted deliberately upstream → leave it out. No such commit → it may genuinely be lost; hand it over.

**4. Restack.**

```powershell
git rebase --update-refs main <tip>
```

If a rung merged **verbatim**, this completes with no conflict. If upstream **modified** it, replaying that rung conflicts against upstream's version — and this is where the whole play can silently go wrong:

> **⚠ During a rebase, `ours` and `theirs` are the opposite of what you expect.** Verified:
>
> | Side | During `git rebase` |
> |---|---|
> | `--ours` / stage 2 | **upstream's** version (the new base) |
> | `--theirs` / stage 3 | **your** commit being replayed |
>
> Rule 1 says upstream wins, so the correct resolution is **`git checkout --ours -- <file>`**. Reaching for `--theirs` — the intuitive choice, since it is "their" merge you are adapting to — **reverts upstream's edit, exits 0, leaves a clean `git status`, and reports nothing.** That is exactly the silent failure step 5 exists to catch.

```powershell
git checkout --ours -- <conflicted file>       # keep upstream's version
git add <conflicted file>
git rebase --continue
```

**5. Verify — every rung, individually.**

A plain `git diff <recorded-sha> <rung>` is **not** the check. Each rung's tree now contains upstream's new commits, so that diff is non-empty by design and tells you nothing. What must hold is narrower: *each rung still contributes exactly its own changes, on top of a new base.*

Compare tree hashes. Paste once, then run bottom-up — each rung is checked against the **new** rung below it:

```powershell
function TreeCheck($oldBase, $newBase, $oldTip, $newTip) {
  $exp = (git merge-tree --write-tree --merge-base=$oldBase $newBase $oldTip) -split "`n" | Select-Object -First 1
  $act = git rev-parse "$newTip^{tree}"
  if ($exp -eq $act) { "ok  $newTip" } else { "DIFFERENT - STOP  $newTip" }
}

# lowest surviving rung — its base is main
TreeCheck <old-main> main <recorded-rung1> <rung1>
# every rung above — base is the rung below it
TreeCheck <recorded-rung1> <rung1> <recorded-rung2> <rung2>
TreeCheck <recorded-rung2> <rung2> <recorded-rung3> <rung3>
#   ...continue up to <tip>
```

**Expect:** `ok` for every rung.
**STOP if:** any prints `DIFFERENT`. Do not push. Inspect with `git diff <recorded-sha> <rung>` and account for every line — anything that is not upstream's own new work is code a replay resurrected or dropped.

**One legitimate exception.** If step 3 found upstream modifications and rule 1 forced you to take their version where a rung of yours also touched those lines, that rung *will* print `DIFFERENT` — the adaptation is real. Confirm it is only that:

```powershell
git diff main <tip> -- <the file list from step 3>
```

**Expect:** only changes your *upper* rungs make on purpose.
**STOP if:** any upstream edit appears reverted. That failure ships silently and is the reason step 3 exists.

> **This is the one check that must never be skipped.** A rebase replays your old commits, so it will happily re-introduce code upstream deleted on purpose — this has actually happened here, and only the tree diff caught it. The stack makes it worse: one `--update-refs` run can resurrect deleted code at *any* rung, and a bad resolution low in the stack propagates silently through every rung above. Running the check is free; only explaining a failure costs anything.

**6. Only if rungs of yours merged — retire them.** Otherwise go to step 7. Repeat for **each** rung from step 2's list:

```powershell
git branch -d <merged rung>
git push origin --delete <merged rung>
```

- Remove its inventory row.
- Update the "Stacked on" line and compare link in any open PR that pointed at it. The lowest surviving rung now sits on `main`; if several merged at once, only the *lowest* survivor moves to `main` — the rest still point at the rung below them.

**7. Push the stack and rebuild the build branch.**

```powershell
git push --force-with-lease origin <each surviving rung with an open PR>
```

Then [rebuild `feature/experimental`](#rebuild-featureexperimental) — step 4 rewrote every rung SHA, so the build branch is holding commits that no longer exist. Do not try to `git merge` the new tip into the old branch; it dedupes by content but leaves both copies in history.

> **🤖 Work with AI Agent if...**
> - Any step-5 diff is non-empty and you can't account for every line. Hand it over *before* `git push --force-with-lease`.
> - Step 4 conflicts on a line upstream modified and rule 1 vs rule 2 isn't obvious. Hand over the hunk plus which rung it stopped on.
> - You need to judge *upstream-intentional deletion* vs *accidentally lost code*. Highest-value handoff in this document — check `git log -S <string>` and `git show` against upstream history **before re-adding anything** a rebase appears to have "dropped".

## Include someone else's upstream PR or branch

NOTE: This is a 'rare' situation, but it does happen.

Contrib work does **not** enter the stack — it merges straight into `feature/experimental`, so it can never contaminate a PR diff:

Pick one fetch; `<contrib-branch>` below is whichever local name it created.

```powershell
git fetch upstream pull/<N>/head:contrib/<N>-<slug>    # for an upstream PR
git fetch upstream <branch>:contrib/<slug>             # for an upstream branch

git checkout feature/experimental
git merge --no-ff <contrib-branch>
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

This is the **one** exception to "nothing but the fork-identity commit and the stack tip is merged into experimental." Record it on the *Contrib branches in play* line under the inventory — **not** as an inventory row; a `contrib/*` branch has no base in the stack, so it doesn't fit that table. A rebuild re-derives `feature/experimental` from scratch and will drop the merge unless you know to replay it.

**When it gets new commits** — re-fetch and re-merge; only the new commits flow in:

```powershell
# whichever form you used originally — '+' allows update even if the author force-pushed
git fetch upstream +pull/<N>/head:contrib/<N>-<slug>
git fetch upstream +<branch>:contrib/<slug>

git checkout feature/experimental
git merge <contrib-branch>                             # force-pushed rewrites still dedupe by content
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

**When it merges upstream** — run [Upstream main moved](#upstream-main-moved). The content arrives via `main`, reaches the rungs when they restack, and reaches the build branch through the rebuild; nothing about it needs special handling. Then:

```powershell
git branch -D <contrib-branch>
```

and clear it from the *Contrib branches in play* line under the inventory.

Don't push `contrib/*` branches to origin — they're not your work; the fetch recreates them anytime.

> **🤖 Work with AI Agent if...**
> - The merge conflicts. **You didn't write this code**, so the "obvious" resolution is less obvious than usual — the bar for handing over is lower here than in any other play. Hand over the conflict *plus* the upstream PR link.
> - The author force-pushed and the re-merge behaves strangely. Content-level dedupe usually handles it, but if the result looks wrong, don't force anything — hand it over.

## My PR was rejected

Drop the rung; everything above it restacks onto the rung below.

> **If the rejected rung is `<tip>`, none of this applies.** Nothing sits above it, so rejection costs one decision — keep it as a fork-only rung or delete it — and zero restacking. That is why the most contentious rung belongs on top.

**1. Record `<tip>`.**

```powershell
git rev-parse --short <tip>                  # write this down
```

**2. Restack over the gap.**

```powershell
git rebase --update-refs --onto <rung-below> <rejected rung> <tip>
```

**3. Verify — `<tip>` only.** The new base is a rung you already had, so the only change should be the rejected rung's code disappearing.

```powershell
git diff <recorded-tip> <tip> --stat
```

**Expect:** exactly the rejected rung's code gone, nothing else.
**STOP if:** anything else moved.

**4. Delete and push.**

```powershell
git branch -D <rejected rung>
git push origin --delete <rejected rung>
git push --force-with-lease origin <each rung that was ABOVE the rejected one>
```

Rungs below the rejected one did not move — do not push them.

**5. Fix the paperwork.**

- Remove the rejected rung's inventory row.
- In every open PR that was above it: update the "Stacked on" line and the compare link. The rung directly above the gap now sits on `<rung-below>` — or on `main`, if the rejected rung was rung 1.

Then [rebuild `feature/experimental`](#rebuild-featureexperimental) — the SHAs it holds no longer exist.

Keep the rejected code if you still want it locally: don't delete the branch, move it *above* the tip instead (`git rebase --onto <tip> <rung-below> <rejected rung>`) and mark the inventory row "local only, not for upstream."

> **🤖 Work with AI Agent if...**
> - Upper rungs *depended* on the rejected rung's code. The restack will conflict everywhere, and the right answer may be to fold the rejected rung's essential parts into the upper rung. Hand over the conflict plus what upstream objected to.

## Squash a rung

Rungs should reach upstream as one commit, or a small number of clearly separable ones.

**Squashing `<tip>` is cheap** — nothing sits above it. **Squashing any lower rung replays every rung above it**, so do it as early as you can; the cost grows with each rung you add.

**1. Record every rung tip.** Recovery needs all of them, not just the one you are squashing.

```powershell
git branch --format='%(refname:short)' |
  Where-Object { $_ -notin 'main','playbook','feature/experimental' -and $_ -notlike 'contrib/*' } |
  ForEach-Object { "{0,-40} {1}" -f $_, (git rev-parse --short $_) }
```

**2. Squash.** The range must run to `<tip>`, not to the rung — otherwise only the rung moves and every rung above is stranded on the pre-squash commits.

```powershell
git rebase -i --update-refs <rung-below> <tip>
```

VS Code opens the commit list oldest-first with a dropdown per line. It contains **every commit reachable from `<tip>` but not from `<rung-below>`** — `<rung-below>`'s own commits are not listed — with an `update-ref refs/heads/<branch>` line after the last commit of each rung:

```
pick a811b5d # rung1: a
pick ec085c7 # rung1: b
update-ref refs/heads/feature/rung1
pick 2a0e67c # rung2: a
update-ref refs/heads/feature/rung2
pick 1e70337 # rung3: a
```

Set the target rung's commits to `s` (squash); leave everything else `pick`. A second editor then opens with all the squashed messages — edit down to the final one, save, close the tab.

| Mark | Effect |
|---|---|
| `pick` | keep as its own commit |
| `s` squash | fold into the line above; opens the combined-message editor |
| `f` fixup | fold into the line above, discard its message silently |
| `e` edit | stop here, then `git rebase --continue` |

> **Never delete or squash across an `update-ref` line.** Those lines are what move the intermediate branch refs; folding a commit across one merges two rungs into a single branch.

A fixup that sits later in history than its parent can be moved — cut its line, paste it under the parent, mark `f`. If that conflicts, `git rebase --abort`, drop that one reorder, retry. For future work avoid the problem entirely: `git commit --fixup=<sha>`, then `git rebase -i --autosquash --update-refs <rung-below> <tip>`.

**3. Verify — content unchanged, commits fewer.**

```powershell
git diff <recorded-tip> <tip>                     # MUST be empty
git log --oneline <rung-below>..<rung>            # the squashed rung: expected count
uv run pytest tests/ --tb=no -q
```

**Expect:** empty diff, fewer commits on `<rung>`, tests pass.
**STOP if:** the diff is non-empty. Squashing must never change content.

**Recovery.** Mid-rebase: `git rebase --abort`. After it completed, **detach HEAD first** — the rebase leaves you on `<tip>`, and git refuses to force-move a branch that a worktree has checked out:

```powershell
git checkout --detach
git branch -f <rung> <recorded-sha>               # repeat for every rung that moved
git checkout <tip>
```

Without the detach you get `fatal: cannot force update the branch '<tip>' used by worktree at ...`. Note it says *worktree*, not *current branch* — the same refusal applies to any branch checked out in `tally-playbook`. Do not substitute `git reset --hard`; it moves whichever branch is checked out, which is not the one you are trying to restore.

**4. Push and absorb.**

```powershell
git push --force-with-lease origin <every rung from `<rung>` up to `<tip>` that has an open PR>
```

Then [rebuild `feature/experimental`](#rebuild-featureexperimental) — the rung SHAs changed.

> **Never squash a rung that has an open PR** unless you intend to force-push it — the PR diff changes wholesale under the reviewer.

> **🤖 Work with AI Agent if...** a reorder conflicts, or step 3's diff is non-empty and you can't see what changed.

## Rebuild feature/experimental

`feature/experimental` is `main` + the `fork:` commit + a merge of the stack tip. When a rebase rewrites rung SHAs, the branch is holding commits that no longer exist — so you throw it away and re-derive it. Two commands, because the `fork:` commit sits directly on `main` and is tagged `fork-identity`.

**Safety net first**, if this rebuild is at all risky — the first command below clobbers the ref, so there is no undo afterwards:

```powershell
git branch experimental-old feature/experimental      # delete once tests pass
```

```powershell
git checkout -B feature/experimental fork-identity
git merge <tip>
git merge --no-ff <each contrib/* branch still in play>     # rare; usually none
uv run pytest tests/ --tb=no -q
git push --force-with-lease origin feature/experimental
```

**When you need this — only after a rebase.** These rewrite rung SHAs:

| Play | Rewrites SHAs? |
|---|---|
| Work on a rung → editing `<tip>` | no — just adds a commit |
| New feature (on top of the stack) | no |
| Submit PRs · Include someone else's PR | no |
| Work on a rung → editing a **lower** rung | **yes** |
| Upstream main moved | **yes** |
| My PR was rejected | **yes** |
| Squashing an existing rung | **yes** |

For the "no" rows, just `git merge <branch>` into `feature/experimental` — no rebuild.

**Never `git merge <tip>` into the *existing* branch after a rebase.** It dedupes by content so it looks like it worked, but both the old and new copies of every rung stay in the history and the next rebuild inherits the mess. Re-derive from the tag instead.

**Only two things may live on `feature/experimental` alone:** the `fork:` commit and any `contrib/*` merges. Both halves need checking before you discard the branch — `--no-merges` hides merges, so it cannot see the second half:

```powershell
git log main..feature/experimental --no-merges --oneline
#   → rung commits + exactly one `fork:` commit, nothing else

git log main..feature/experimental --merges --oneline
#   → one merge per absorb since the last rebuild, plus one per contrib/* branch
```

Anything extra in the first list is work committed to the wrong branch — move it to a rung before rebuilding, or you lose it.

**The second list is normally several merges, not one.** Every *New feature* and *Editing `<tip>`* absorb adds another merge of the stack tip; they accumulate until the next rebuild collapses them to one. What you are looking for is a merge that is neither a stack-tip merge nor a `contrib/*` merge — that is something absorbed into the build branch that shouldn't have been. Note every `contrib/*` merge you find; the rebuild command above has to replay each one.


> **🤖 Work with AI Agent if...**
> - Either `git log main..feature/experimental` listing turns up something unexpected — a commit that is neither a rung commit nor the `fork:` commit, or a merge that is neither `<tip>` nor a `contrib/*`. Hand over that list *before* rebuilding; the rebuild discards whatever it is.
> - `git merge <tip>` conflicts against the `fork:` commit. It shouldn't; the only overlap is `dev-build.yml`, and it is conflict-free as long as the `build-rc`/`rc-artifacts` naming stayed upstream's. A conflict there means that rename crept back in.
> - `fork-identity` doesn't resolve. Find it with `git log --oneline --grep='^fork:' feature/experimental` and re-tag before rebuilding.

## When to abandon upstream

**Not a trigger: David going quiet.** His stated cadence is *"my interest spikes when I need to run it, maybe 4 times a year"* (2026-07-14). Months of silence is the documented normal, not a signal. Do not restructure anything because a PR sat unreviewed.

**Real triggers, in order of cost:**

| Event | Response | Cost |
|---|---|---|
| Rung rejected, nothing above depends on it | *My PR was rejected* play | minutes |
| Rung rejected, upper rungs depend on it | Same play, but fold essentials upward | hours |
| Upstream main moved | *Upstream main moved* play | minutes, plus per-rung ground-truth checks |
| Upstream merges a *modified* version of a rung | *Upstream main moved* step 3, resolve only their hunks | hours, highest judgment risk |
| **You decide the tax isn't worth it** | Collapse: merge the whole stack into one branch, stop rebasing on upstream, stop opening PRs | one afternoon |

The last row is always available and costs nothing to defer. Nothing in this model traps you upstream — the stack is a *presentation* choice, and `feature/experimental` is a working build regardless of whether a single PR ever merges.

---

# Local machine setup

## The two worktrees

| Path | Branch | Role |
|---|---|---|
| `C:\BTR\OpenSource\tally` | whatever rung you're developing on | all code work; runs the playbook |
| `C:\BTR\OpenSource\tally-playbook` | `playbook` (**permanently**) | this file, `.vscode/Plans/`, `wishList.md` — always visible, never checked out anywhere else |

Both are opened together via `C:\BTR\Extensibility\Tally.code-workspace` (a VS Code multi-root workspace). Worktrees share one `.git`, so refs, objects, `rerere` state, and `.git/info/exclude` are common to both — but HEAD and the index are per-worktree, so committing notes never disturbs a half-finished feature.

> **⚠ `playbook` is pinned to the `tally-playbook` worktree.** Git refuses to check out one branch in two worktrees. `git checkout playbook` in the main worktree **will fail** — that's by design, not breakage. All commits to it happen in `tally-playbook`.

> **⚠ Renaming or moving either directory breaks the worktree link** — both store absolute paths. Repair with `git -C <repo> worktree repair <worktree-path>`, which fixes both directions.

> **`playbook` is an orphan branch** — it shares no commit history with `main`. Two consequences:
>
> - **`git merge playbook` fails by default**: `fatal: refusing to merge unrelated histories`. Accidental propagation into a rung is blocked by git, not by remembering a rule. Deliberate propagation is still possible via `--allow-unrelated-histories` or `git cherry-pick`; don't.
> - **`git log main..playbook` returns every commit on `playbook`**, since none are reachable from `main`. It is valid, just useless — don't reach for it to compare the two.

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
git worktree add ../tally-playbook playbook
# 3. rerere (the stack model depends on it — every rebase replays old resolutions)
git config rerere.enabled true
# 4. remotes
git remote add upstream https://github.com/davidfowl/tally
# 5. rebase.updateRefs so plain `git rebase` doesn't strand mid-stack refs
git config rebase.updateRefs true
# 6. the fork-identity tag — clones do NOT fetch tags that were never pushed
git fetch origin --tags
git tag -l fork-identity                     # must print; Rebuild feature/experimental depends on it
```

If `fork-identity` is missing, recreate and push it so the next clone has it:

```powershell
git tag fork-identity $(git log --format='%H' --grep='^fork:' feature/experimental | Select-Object -Last 1)
git push origin fork-identity
```

`C:\BTR\Extensibility\Tally.code-workspace` and `.claude/settings.local.json` (which grants agents access to `../tally-playbook/` via `permissions.additionalDirectories`) are also untracked — recreate or restore them from backup.
