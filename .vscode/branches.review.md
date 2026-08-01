# Review brief — `branches.md` (tally fork git workflow)

**Document under review:** `C:\BTR\OpenSource\tally-playbook\.vscode\branches.md` (~600 lines)
**Ask:** review for **git workflow correctness**. Do the commands do what the prose claims? Are there sequences that lose work, produce wrong history, or fail silently?

---

## What this document is

The operating manual for a **fork** of `davidfowl/tally`. One person (Terry) maintains a personal build with five features on top of upstream, while feeding those features back as pull requests.

It is not a general git guide. Every play is a specific, ordered sequence for this repository's topology.

## The topology

```
main (= upstream/main, never diverges)
 │
 ├── feature/workflow-node24                        rung 1
 │     └── feature/globbing-documentation           rung 2
 │           └── feature/ui-tweaks                  rung 3
 │                 └── feature/charts-reimagined    rung 4
 │                       └── feature/merchant-composite-keys   rung 5  ← "<tip>"
 │
 ├── feature/experimental  = main + one `fork:` commit + merge(<tip>)
 │
 └── playbook  (orphan branch — this doc, plans, prototypes; merges nowhere)
```

Three invariants the whole document derives from:

1. Every feature branch is based on the rung below it, never on `main` (except rung 1).
2. `feature/experimental` is `main` + the `fork:` commit + a merge of `<tip>`. Nothing else.
3. PRs are submitted bottom-up, each declaring the rung it sits on.

## Why a stack rather than parallel branches

The previous model had all five branches off `main` in parallel. Cross-branch clashes were reconciled after the fact with `integration(...)` commits on the build branch. Two of those existed for `ui-tweaks` vs the merchant-key work.

The stack eliminates that class of commit: an upper rung is *authored* on top of the lower one, so there is nothing to reconcile afterwards. The document was rewritten around this in July 2026, and the five branches were rebuilt from the old ones by cherry-pick + `range-diff` verification.

**Accepted cost:** rungs lose independence. Any rebase — upstream moving, editing a middle rung, dropping a rung — replays everything above and rewrites SHAs. That is the central tradeoff, and it is why so much of the document is about verifying rebases.

## Design decisions a reviewer should not "fix"

These look odd out of context and were chosen deliberately:

| Decision | Reason |
|---|---|
| `feature/merchant-composite-keys` is the **top** rung, not the bottom | It changes merchant identity and is the likeliest to be rejected. On top, rejection costs zero restacking. |
| `playbook` is an **orphan** branch | Notes cannot reach a PR diff or trigger a build — a property of the graph, not a rule to remember. |
| `.claude/` excluded via `.git/info/exclude`, not `.gitignore` | `.gitignore` is tracked per-branch; the exclude file is shared across all branches and both worktrees, and can never leak into a PR. |
| Fork identity is **one commit**, tagged `fork-identity` | Makes rebuilding the build branch two commands. |
| PR bodies live in `.vscode\Plans\PRs\` on `playbook` | Versioned, editable outside the browser, survive a closed/reopened PR. |
| The fork-identity commit keeps upstream's `build-rc` / `rc-artifacts` naming | Renaming those is the only thing that ever conflicted with rung 1's action-version bumps. |
| GitHub's native stacked PRs are **not** used | Docs state cross-fork stacks are unsupported; a PR's base must exist in the base repo. Path A in *Submit PRs* covers the case where push access is granted. |

## What to review hardest

1. **`git rebase --update-refs` usage** — used in *Work on an existing rung*, *Upstream main moved*, *My PR was rejected*, *Squash a rung*. Are the argument orders right? Does it move the refs the prose claims, and only those?
2. **The verification steps.** Every rebasing play records tips beforehand and diffs afterwards. Are the recorded/compared refs correct? Is "expect empty" actually right in each case?
3. **`git rebase --onto <rung-below> <rejected rung> <tip> --update-refs`** in *My PR was rejected* — argument order and whether `--update-refs` behaves as claimed with `--onto`.
4. **Push guidance.** Several plays say to force-push "rungs above X" and explicitly not those below. Is the boundary stated correctly each time?
5. **Rebuild `feature/experimental`.** `git checkout -B feature/experimental fork-identity` then `git merge <tip>`. The document forbids merging the new tip into the *existing* branch after a rebase (content dedupes, but both copies stay in history). Is that characterisation accurate?
6. **Orphan-branch claims.** `git checkout --orphan`, and the assertions that `git log main..playbook` is meaningless and `git merge playbook` must never run.
7. **The clobbering failure mode** (*Failure mode: ignored files are silently clobbered*) — the claim is that untracked-**and-ignored** files are silently overwritten by a merge, while untracked-**not-ignored** files cause the merge to abort. Stated as verified behaviour; worth confirming.

## Known gaps / open items

- `.git/info/exclude` is currently **empty** in the working clone — the entry was lost in a re-clone. *What a fresh clone loses* documents restoring it; it hasn't been done yet.
- The `fork-identity` tag exists locally but may not be pushed. Step 6 of *What a fresh clone loses* covers recreating it.
- One planned feature (`Fixed-Spending-Cleanup`) will become rung 6, above composite-keys, because it semantically depends on composite merchant identity.

## Conventions in the document

- **PowerShell only.** No bash/WSL. Reviewers proposing bash equivalents will be rejected.
- `<tip>`, `<rung>`, `<rung-below>`, `<rung-above>` are defined in a notation table under `# Playbook` and used throughout instead of literal branch names. The inventory table is the substitution source.
- Every play is numbered steps with **Expect:** / **STOP if:** at the point they apply, and a `🤖 Work with AI Agent if...` block listing the narrow cases worth escalating.
- Plays are meant to be followed top-to-bottom without cross-referencing other sections. Duplication between plays is intentional.
