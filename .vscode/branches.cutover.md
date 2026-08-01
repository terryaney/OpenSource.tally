# Cutover Checklist

```powershell
git push origin feature/workflow-node24 feature/globbing-documentation `
                feature/ui-tweaks feature/charts-reimagined `
                feature/merchant-composite-keys feature/experimental --force-with-lease
```

**Retire the old notes worktree.** `C:\BTR\OpenSource\tally-notes` is a worktree of the **old** repo, checked out on `feature/experimental-foundational`. Its content now lives on `playbook`, so it is dead weight — and leaving it registered means the old repo keeps a lock on that directory.

```powershell
# confirm nothing uncommitted is left behind
git -C C:\BTR\OpenSource\tally-notes status --short

git -C C:\BTR\OpenSource\tally worktree remove C:\BTR\OpenSource\tally-notes
git -C C:\BTR\OpenSource\tally worktree list          # tally-notes gone
```

`worktree remove` refuses if the tree is dirty — commit or discard first, or add `--force` once you have checked the status output. If the directory was already deleted by hand, use `git -C C:\BTR\OpenSource\tally worktree prune` instead.

**Close VS Code, then swap the folder names.**

```powershell
Rename-Item C:\BTR\OpenSource\tally          tally-archive
Rename-Item C:\BTR\OpenSource\tally-stacked  tally
```

**Repair the worktree link.** `C:\BTR\OpenSource\tally-playbook` stores an **absolute** path back to `...\tally-stacked\.git\worktrees\...`, and the repo stores an absolute path forward to the worktree. Renaming the repo breaks both. Fix both directions in one command:

```powershell
git -C C:\BTR\OpenSource\tally worktree repair C:\BTR\OpenSource\tally-playbook
git -C C:\BTR\OpenSource\tally worktree list          # both paths correct
git -C C:\BTR\OpenSource\tally-playbook status        # resolves without error
```

> **Do not rename `tally-playbook` itself.** `worktree repair` fixes a moved *repo*; a moved worktree with a simultaneously moved repo can leave it unresolvable. If you do want it renamed, do that in a separate step and re-run `worktree repair` from the new location afterwards.

**Reopen.** Point `Tally.code-workspace` at `C:\BTR\OpenSource\tally` and `C:\BTR\OpenSource\tally-playbook`, reopen it, then rebuild what a fresh clone loses — see *What a fresh clone loses*.

Delete `tally-archive` only after the first rung merges. Then delete this section.
