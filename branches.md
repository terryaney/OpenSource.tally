# Branch Inventory & Playbook

Local workflow documentation for the tally fork. Lives on `feature/experimental-foundational` in `OpenSource/tally` — the only branch that carries fork-local machinery, and one that never sources an upstream PR. See *Local machine setup* for how it stays readable from every other branch.

## Inventory

Row order = merge order for the rebuild play. `feature/experimental-foundational` (fork build machinery, always merged first) are implicit and never listed.

| Branch | Upstream | In Experimental | Notes |
|--------|----------|-----------------|-------|
| [issue/88-merchant-category-display](https://github.com/terryaney/OpenSource.tally/tree/issue/88-merchant-category-display) | [PR #91](https://github.com/davidfowl/tally/pull/91) | yes | Composite merchant keys; opaque `merchant_<b64>` row IDs |
| [feature/globbing-documentation](https://github.com/terryaney/OpenSource.tally/tree/feature/globbing-documentation) | [PR #93](https://github.com/davidfowl/tally/pull/93) | yes | |
| [feature/workflow-node24](https://github.com/terryaney/OpenSource.tally/tree/feature/workflow-node24) | | yes | Local-only workflow machinery |
| [feature/ui-tweaks](https://github.com/terryaney/OpenSource.tally/tree/feature/ui-tweaks) | [PR #94](https://github.com/davidfowl/tally/pull/94) | yes | **Must merge after issue/88** — see `integration(ui-tweaks)` commit |
| [feature/charts-reimagined](https://github.com/terryaney/OpenSource.tally/tree/feature/charts-reimagined) | [PR #95](https://github.com/davidfowl/tally/pull/95) | yes | **Based on feature/ui-tweaks; merge after ui-tweaks** |

> **Branch names and open PRs:** a PR's head branch cannot be renamed without closing the PR, so an inventory row keeps whatever name the PR was opened from (`issue/88-...` predates the one-branch-per-feature rule and stays until PR #91 resolves). The inventory always lists the branch the upstream PR actually tracks — never a copy.

# Plan a feature with an AI agent

Plans, prototypes and research notes live in **`tally-notes\.vscode\Plans\`** — on `-foundational`, tracked, in the notes worktree. Never in the main worktree.

```
.vscode\Plans\
  <topic>.md                 # loose plans
  Features\<Feature>.md      # per-feature plans
  Prototypes\<thing>.html    # throwaway protos
```

Point the agent at that path and let it read and write there directly while you develop on any topic branch in the main worktree. `permissions.additionalDirectories` in `.claude/settings.local.json` already grants access, so there is no per-file approval prompt.

Why this and not "write the plan into the feature branch and promote it later":

- **It cannot leak.** A plan on `-foundational` is structurally incapable of appearing in a topic branch or an upstream PR diff — that's a property of the branch graph, not a rule anyone has to remember.
- **No promote step, no cleanup.** The file is tracked from the moment it's created. Nothing to `git add -f` later, nothing left littering the main worktree.
- **No clobbering.** The rejected alternative (an ignored `Plans/` dir in the main worktree) would be silently overwritten the moment the promoted, tracked copy arrived via a merge — see *Failure mode: ignored files are silently clobbered*.
- **Always visible.** It survives every `git checkout` in the main worktree, because it isn't in the main worktree.

Use unique, descriptive filenames — the directory is shared across all features, not per-branch. Commit plans in the notes worktree whenever; it never disturbs in-flight feature work. They reach `feature/experimental` on the next `--no-ff` merge of `-foundational`, and trigger no build.

---

# Playbook

Run everything yourself; each play is a handful of git commands. Each play flags its own **🤝 Hand to Claude** moments inline, at the step where they bite.

Two rules for those handoffs, everywhere they appear: hand over the **specific conflict or failure** — the conflict hunk, the failing test output, the unexplained diff — and **never the whole play**. Everything not flagged is deliberately just git commands; routing those through an agent adds cost, not safety.

- [Update an already-included branch](#update-an-already-included-branch)
  - [Test a local build without committing](#test-a-local-build-without-committing)
- [New feature](#new-feature)
- [My PR was accepted upstream](#my-pr-was-accepted-upstream)
- [Upstream main moved](#upstream-main-moved)
- [Include someone else's upstream PR or branch](#include-someone-elses-upstream-pr-or-branch)
- [Drop an included feature](#drop-an-included-feature)

## Update an already-included branch

Applies to *any* included branch — active feature, foundational, "frozen" ones. Nothing is frozen in the merge model:

```powershell
git checkout <branch>
# ...commit changes (rebase main first only if the branch has an open PR and main moved)...
git checkout feature/experimental
git merge --no-ff <branch>                  # brings only the new commits
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

Original merge order is irrelevant for updates; it only matters during rebuilds. New semantic clash → new `integration(...)` commit.

> **🤖 Work with AI Agent if...**
> - The `--no-ff` merge stops with conflict markers and the right resolution isn't obvious. Hand over *that conflict*.
> - The merge is **clean but `pytest` fails**. That's a semantic clash between two branches that merged fine textually. The fix is an `integration(<branch>): ...` commit on `feature/experimental` — never a change to either branch. Hand over the failing test output.
> - You rebased first (the branch has an open PR and main moved) and the ground-truth diff isn't empty — see *Upstream main moved* for why that is never ignorable.

> **Exception — `feature/experimental-foundational`.** It is checked out permanently in `tally-notes`, so the `git checkout` above will fail for it. Commit to it *in that worktree* instead, then merge from the main worktree as usual:
>
> ```powershell
> git -C ..\tally-notes commit -am "<change>"    # or just work in the tally-notes folder
> git checkout feature/experimental
> git merge --no-ff feature/experimental-foundational
> git push origin feature/experimental
> ```
>
> Pushing a `.vscode/**`-only change does **not** trigger the Experimental Build — `dev-build.yml` carries a `paths-ignore` for it, which is why notes can live on a branch that merges into `feature/experimental` without burning a release.

### Test a local build without committing

For trying an implementation against real data before any commit/push/release. Two tiers:

**Tier 1 — Generate a report from the working tree (eyeball a UI change)** Run straight from source against the spending config:

```powershell
cd C:\BTR\OpenSource\tally
# Throwaway copy — never touches your normal spending_summary.html:
uv run tally up -c C:\BTR\TallySpending\tally\config -o C:\BTR\TallySpending\tally\output\spending_check.html
```

Then open `C:\BTR\TallySpending\tally\output\spending_check.html` in a browser (double-click — `file://` is fine).

- `-c / --config` points at the **config dir** — `...\tally\config`, the folder that holds `settings.yaml`, *not* the `TallySpending` root. The bare positional path (`uv run tally up C:\BTR\TallySpending`) is deprecated **and** was pointing one level too high — that path has no `settings.yaml` and errors out.
- `-o / --output` overrides the output path. **Omit it and tally writes to the configured default** (`output_dir: output` → `...\tally\output\spending_summary.html`), overwriting your normal report — so pass `-o ...spending_check.html` when you just want to look.
- The report is rendered from the **working tree** (`src/tally/spending_report.{html,css,js}`), so uncommitted edits appear immediately — just re-run to regenerate after each change.
- Iterating on CSS/JS specifically? Add `--no-embedded-html` to emit editable `spending_report.css` / `.js` next to the HTML; edit those and refresh the browser (no regen needed). See `CLAUDE.md → HTML Report Development`.
- `file://` is blocked inside the Playwright MCP sandbox, so agent-driven verification needs a loopback server (`python -m http.server <port> --bind 127.0.0.1` — the `--bind` keeps your personal data off the LAN). A normal browser has no such limit.

**Tier 2 — test as the installed tool.** Build the exe locally (same pyinstaller invocation as `build.yml`) and overwrite the installed copy:

```powershell
cd C:\BTR\OpenSource\tally
Copy-Item "$env:LOCALAPPDATA\tally\tally.exe" "$env:LOCALAPPDATA\tally\tally.official.exe"   # backup first
uv run pyinstaller --onefile src/tally/__main__.py --name tally `
  --add-data "src/tally/spending_report.html;tally" `
  --add-data "src/tally/spending_report.css;tally" `
  --add-data "src/tally/spending_report.js;tally"
Copy-Item dist\tally.exe "$env:LOCALAPPDATA\tally\tally.exe" -Force   # tally's install path
```

**Restore the official build** when done testing:

```powershell
Copy-Item "$env:LOCALAPPDATA\tally\tally.official.exe" "$env:LOCALAPPDATA\tally\tally.exe" -Force
```

Notes:
- pyinstaller builds the **working tree** — uncommitted changes included; that's the point.
- The local build skips the workflow's version injection: `tally --version` reports placeholder `0.1.0` — the tell that you're running a local build.
- Don't trust `tally update` from a local build: the placeholder `REPO_URL` in `_version.py` points at **davidfowl/tally** on main-based branches (only `feature/experimental`'s copy points at the fork), so it may fetch David's build instead of yours. The backup/restore copy avoids the whole question.
- Either copy fails if tally.exe is currently running — close it first.

## New feature

```powershell
git checkout -b feature/<name> main
# ...develop, test...
git checkout feature/experimental
git merge --no-ff feature/<name>
uv run pytest tests/ --tb=no -q             # failures here = semantic clash → integration commit
git push origin feature/experimental
```

Add an inventory row (position it after any branch it depends on). When PR time comes: `git push origin feature/<name>`, open the upstream PR from that branch, and keep developing on it — no separate PR branch.

If the merge is clean but tests fail, fix on `feature/experimental` and commit as:

```
integration(<branch>): <one-line summary>

<why: which two branches clash and what the adaptation does>
```

> **🤖 Work with AI Agent if...**
> - The `--no-ff` merge conflicts and the resolution isn't obvious.
> - The merge is **clean but tests fail** — that's the semantic clash above. Hand over the failing test output and let it write the `integration(...)` commit; the *why* line in that commit body is the whole point of the convention, and it's the part worth an agent's attention.
> - The new branch depends on an already-included one (like `feature/ui-tweaks` on `issue/88`) and you're unsure where it belongs in the inventory's merge order.

## My PR was accepted upstream

No rebuild. The branch lives:

```powershell
git fetch upstream
git checkout main
git merge --ff-only upstream/main
git push origin main
git checkout feature/experimental
git merge main                              # dedupes automatically, even squash-merges
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
git branch -d feature/<name>
git push origin --delete feature/<name>
```

Remove the inventory row. If David merged a *modified* version of your PR, expect small conflicts only on the hunks he changed — resolve those, never replay anything.

> **🤖 Work with AI Agent if...**
> - `git merge main` conflicts because David merged a **modified** version of your PR. Resolve only the hunks he changed; **never replay** your original version over his. If you catch yourself re-adding your code because the merge "lost" it, stop and hand it over.
> - You can't tell whether a difference is *upstream-intentional* or *accidentally lost*. This is the highest-value handoff in the playbook — the judgment call needs `git log -S <string>` and `git show` on the upstream history before anything is re-added. See the PR #92 lesson under *Upstream main moved*.
> - Tests fail after `git merge main` into `feature/experimental` → `integration(...)` commit.

## Upstream main moved

```powershell
git fetch upstream
git checkout main
git merge --ff-only upstream/main
git push origin main
```

For each topic branch with an **open PR** (local-only branches can wait until you touch them):

```powershell
git checkout <branch>
git rev-parse --short HEAD                  # record pre-rebase tip
git rebase main
git diff <pre-rebase-tip> --stat            # GROUND-TRUTH CHECK — see warning below
git push --force-with-lease origin <branch>
```

Then absorb into the build:

```powershell
git checkout feature/experimental
git merge main
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

> **⚠ The ground-truth check is not optional.** A rebase can silently resurrect code that upstream deliberately deleted (proven here: replaying PR #91's first commit re-added an exclusion skip that merged PR #92 had intentionally removed; only the tree-diff caught it). The diff against the pre-rebase tip must be empty, or every line of it must be explainable as intentional. If you can't explain it, hand it to Claude before pushing.

> **🤖 Work with AI Agent if...**
> - **The ground-truth diff is non-empty and you can't account for every line.** Hand it over *before* you `git push --force-with-lease` — this is the one handoff where the cost of skipping it is a silent, already-published regression.
> - You need to judge *upstream-intentional deletion* vs *accidentally lost code*. The PR #92 lesson: check `git log -S <string>` and `git show` on upstream history **before re-adding anything** a rebase or merge appears to have "dropped". A rebase replays your old commits, so it will happily re-introduce code David removed on purpose.
> - The rebase stops with conflict markers — hand over the one conflict, not the branch.
> - `git merge main` into `feature/experimental` is clean but `pytest` fails → semantic clash → `integration(...)` commit.

## Include someone else's upstream PR or branch

NOTE: This is 'rare' situation, but it does happen.

Intake — fetch the upstream ref straight into a local `contrib/` branch, then merge like any other inventory branch:

```powershell
git fetch upstream pull/<N>/head:contrib/<N>-<slug>    # for an upstream PR
git fetch upstream <branch>:contrib/<slug>              # for an upstream branch
git checkout feature/experimental
git merge --no-ff contrib/<N>-<slug>
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

Add an inventory row linking the upstream PR. Don't push `contrib/*` branches to origin — they're not your work; the fetch recreates them anytime.

**When the upstream PR/branch gets new commits** — re-fetch and re-merge; only the new commits flow in:

```powershell
git fetch upstream +pull/<N>/head:contrib/<N>-<slug>   # '+' allows update even if the author force-pushed
git checkout feature/experimental
git merge contrib/<N>-<slug>                            # force-pushed rewrites still dedupe by content
uv run pytest tests/ --tb=no -q
git push origin feature/experimental
```

**When it merges upstream** — the *My PR was accepted* play applies verbatim: merge main (content dedupes), delete the `contrib/*` branch, remove the row.

> **🤖 Work with AI Agent if...**
> - The merge conflicts. **You didn't write this code**, so the "obvious" resolution is less obvious than usual — the bar for handing over is lower here than in any other play. Hand over the conflict *plus* the upstream PR link, so the author's intent is recoverable.
> - Tests fail after a clean merge → `integration(contrib/<slug>): ...` commit.
> - The author force-pushed and the re-merge behaves strangely. Content-level dedupe usually handles it, but if the result looks wrong, don't force anything — hand it over.

## Drop an included feature

**WARNING**: This triggers a complete rebuild — the only scratch construction.

```powershell
git branch -m feature/experimental experimental-old      # temp reference, deleted at the end
git checkout -b feature/experimental main
git merge --no-ff <each remaining inventory branch, in row order>
```

- Repeated conflicts: rerere auto-stages your old resolution; verify and continue. For anything novel, `git show --remerge-diff <merge-sha>` on `experimental-old` shows how you resolved it last time.
- Re-apply still-relevant integration commits: `git log --oneline --grep='^integration(' experimental-old`, then `git cherry-pick <sha>` each one that isn't tied to the dropped branch.

```powershell
uv run pytest tests/ --tb=no -q
git push --force-with-lease origin feature/experimental   # the one sanctioned experimental force-push
git branch -D experimental-old                             # once the build is verified
```

> **🤖 Work with AI Agent if...** — this is the highest-risk play and the only one that force-pushes `feature/experimental`. Hand over **early rather than late**; `experimental-old` is your safety net right up until you `git branch -D` it, so keep it around while anything is unresolved.
> - **rerere auto-staged a resolution you can't verify.** It replays a past resolution without asking, and the branch set has *changed* — an old resolution can be silently wrong now. It is an accelerator, never an authority. Cross-check with `git show --remerge-diff <merge-sha>` on `experimental-old`.
> - A **novel** conflict appears — one you don't recognize from the original merge order.
> - **Deciding which `integration(...)` commits still apply.** Some are tied to the branch you just dropped and must **not** be cherry-picked; re-applying one of those re-introduces an adaptation for a clash that no longer exists. `git log --oneline --grep='^integration(' experimental-old` lists them; if any commit's *why* body doesn't clearly survive the drop, hand it over.
> - Tests fail at the end and you can't localize which merge caused it. Hand over the failing output **before** the force-push, not after.

# Local machine setup

Fork-local notes (this file, plans, prototypes, wish list) belong on `feature/experimental-foundational`, because it is the one branch that merges into `feature/experimental` but never sources an upstream PR. That keeps them out of every PR diff — but it also means they'd be invisible whenever a topic branch is checked out. **A second worktree solves that**, and everything below exists to make it work.

## The two worktrees

| Path | Branch | Role |
|---|---|---|
| `C:\BTR\OpenSource\tally` | whatever you're developing on | all code work; runs the playbook |
| `C:\BTR\OpenSource\tally-notes` | `feature/experimental-foundational` (**permanently**) | this file, `.vscode/Plans/`, `wishList.md` — always visible, never checked out anywhere else |

Both are opened together via `C:\BTR\Extensibility\Tally.code-workspace` (a VS Code multi-root workspace). Worktrees share one `.git`, so refs, objects, `rerere` state, and `.git/info/exclude` are common to both — but HEAD and the index are per-worktree, so committing notes never disturbs a half-finished feature.

> **⚠ `-foundational` is now pinned to the notes worktree.** Git refuses to check out one branch in two worktrees. `git checkout feature/experimental-foundational` in the main worktree **will fail** — that's by design, not breakage. All commits to it happen in `tally-notes`; the *Update an already-included branch* play notes the exception. Merging it into `feature/experimental` from the main worktree still works fine — only *checkout* is blocked.

## What is deliberately never committed

`.claude/` (Claude Code settings, permissions, agent state) is excluded via **`.git/info/exclude`**, not `.gitignore` — because `.gitignore` is tracked per-branch, and a rule committed to `-foundational` would not be in effect while a `main`-based topic branch is checked out. `.git/info/exclude` lives in the shared `.git` dir, so one entry applies to every branch **and** both worktrees, and can never itself leak into an upstream PR.

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
git worktree add ../tally-notes feature/experimental-foundational
# 3. rerere (the merge model depends on it)
git config rerere.enabled true
# 4. upstream remote
git remote add upstream https://github.com/davidfowl/tally
```

`C:\BTR\Extensibility\Tally.code-workspace` and `.claude/settings.local.json` (which grants agents access to `../tally-notes/` via `permissions.additionalDirectories`) are also untracked — recreate or restore them from backup.