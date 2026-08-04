> **Independent — based directly on `main`.** Touches `.github/workflows/` only and shares no file with any of my other open PRs, so it can be merged in any order.
> **Diff:** [`main...feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair)

`PR Build` has failed on **every fork pull request** since 2026-07-20, including ones that change nothing but documentation. This PR fixes that, and separately brings the workflow action pins onto Node 24 ahead of Node 20's removal this fall.

## The break

```
##[error]Refusing to check out fork pull request code from a 'pull_request_target'
workflow. This workflow runs with the base repository's GITHUB_TOKEN, secrets,
default-branch cache scope, and runner access. Fetching and executing a fork's
code in that trusted context commonly leads to "pwn request" vulnerabilities.
```

Nothing in this repository changed. Two things changed underneath it, between two runs 42 hours apart:

| | 2026-07-19 04:13 — passing | 2026-07-20 22:11 — failing |
|---|---|---|
| Runner | `2.335.1` | `2.336.0` |
| `actions/checkout@v4` resolved to | `34e1148…` | `11d5960…` |

`v4` is a moving tag, and the release it now points at refuses to check out fork code under `pull_request_target`. `pr-build.yml` did exactly that — `pull_request_target` plus an explicit `ref:` pointing at the fork's head SHA.

It isn't specific to one contributor: the most recent failure at time of writing is on `sutharion-studious-meme`, and the last green run was on a branch of mine that would fail identically today.

**Bumping to `actions/checkout@v5` does not fix this.** v5 carries the same guard. The pattern itself has to change.

## The fix — split the trusted half from the untrusted half

The problem with `pull_request_target` here is that one workflow both **executes fork code** and **holds a write token**. Those are separated:

| Workflow | Trigger | Token | Touches PR code? |
|---|---|---|---|
| `pr-build.yml` | `pull_request` | `contents: read`, no secrets | yes — builds it |
| `pr-build-comment.yml` *(new)* | `workflow_run` | `pull-requests: write` | **never** |

`pr-build.yml` keeps its `prepare` and `build` jobs and loses the `comment` job. `build.yml` needed no changes at all — under `pull_request`, checking out the head SHA is allowed.

`pr-build-comment.yml` waits for `PR Build` to complete, then posts the same install comment as before.

### Passing the PR number across

The `workflow_run` event's `pull_requests` array is empty for fork PRs, and `GET /repos/{repo}/commits/{sha}/pulls` does not associate fork-originated commits back to the PR from the base repository's side — I checked both against this repo's open PRs, and both come back empty for every fork PR while the commit itself is reachable. So the PR number cannot be recovered from the trusted side alone. It travels as a small artifact written by `prepare`.

That artifact is produced by a run over untrusted code. Under `pull_request`, GitHub resolves workflow files from the PR head, which means a fork controls what its own run writes into that artifact — including writing *someone else's* PR number to aim this privileged workflow at an unrelated PR. Shape validation alone does not prevent that, so the number is checked for **ownership**, not just format:

```bash
# Shape first, so the value is safe to put in an API path at all.
if ! [[ "$PR_NUMBER" =~ ^[0-9]+$ ]]; then exit 1; fi

# Then ownership: ask the API what that PR's head actually is, and require it
# to match the head SHA GitHub itself recorded for the triggering run.
PR_HEAD_SHA=$(gh api "repos/${REPO}/pulls/${PR_NUMBER}" --jq '.head.sha')
if [ "$RUN_EVENT" = "pull_request" ] && [ "$PR_HEAD_SHA" != "$TRUSTED_HEAD_SHA" ]; then
  exit 1
fi
```

`github.event.workflow_run.head_sha` is populated by GitHub from its own record of the triggering run, so a fork cannot forge it. A fork can claim PR #42, but it cannot make #42's head commit equal its own. `short_sha` is then derived from the API response rather than read from the artifact, so the artifact's only remaining role is supplying a number that must survive that check.

The check is skipped for `workflow_dispatch`, where `head_sha` is the dispatched branch rather than a PR head; that path can only be triggered by someone with write access, running the base repository's own copy of the workflow file.

Artifact poisoning is the one remaining attack surface in this pattern, and this is where it gets closed.

### Three things to expect

- **The comment won't post on this PR.** `workflow_run` only fires from the workflow file on the repository's default branch, so `pr-build-comment.yml` starts working once this is merged to `main`. The build half will go green here; the comment half is untestable from a PR by design.
- **There will be a red `PR Build` on this PR, from `main`'s copy of the workflow.** Until this merges, `main` still defines `PR Build` as `pull_request_target`, so every push here starts *two* runs against the same commit: the old one from `main` (which fails at checkout, exactly as described above) and the new one from this branch (which passes). Compare the `event` on each run before reading the result — the failing one is the bug this PR fixes, still firing from the base branch.
- **`pull_request` resolves workflow files from the PR head**, unlike `pull_request_target`. A useful side effect: from now on, CI changes can be validated by the PR that makes them.

## Also: Node 24 action pins

Separate commit, separate problem — Node 20 is being removed from the Actions runner this fall. Node 24 has already been the runner default since 2026-06-16, so nothing fails today; what breaks at removal is any action still declaring `using: node20` in its own metadata. The fix is version bumps, not a runtime override:

| Action | Was | Now |
|---|---|---|
| `actions/checkout` | `v4` | `v5` |
| `actions/upload-artifact` | `v4` | `v7` |
| `actions/download-artifact` | `v4` | `v8` |
| `astral-sh/setup-uv` | `v4` | `v8.1.0` |
| `actions/configure-pages` | `v4` | `v6` |
| `actions/upload-pages-artifact` | `v3` | `v5` |
| `actions/deploy-pages` | `v4` | `v5` |
| `peter-evans/find-comment` | `v3` | `v4` |
| `peter-evans/create-or-update-comment` | `v4` | `v5` |

Every JavaScript action pinned in `.github/workflows` now declares `node24`; the two remaining composites resolve to `node24` internals (`upload-pages-artifact@v5` pins `upload-artifact@v7.0.0`).

Two of these majors carry breaking changes that were checked against this repository before bumping:

- `configure-pages@v5` dropped Next.js < 13.3.0 support under the `static_site_generator` input — not used here, the step takes no inputs.
- `upload-pages-artifact@v4` stopped including dotfiles in the artifact — `docs/` contains none, so nothing is dropped. (`v5` adds an `include-hidden-files` input if that ever changes.)

`FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` was an earlier attempt at this half and has been removed. It was a pre-flip opt-in, it is inert now that Node 24 is the default, and per [actions/runner#4295](https://github.com/actions/runner/issues/4295) it does not even suppress the deprecation annotation.

## Scope

`.github/workflows/` only — ten files, no source, no tests, no documentation.

## Also in flight (independent of this PR)

A separate stack of feature work, each branch based on the one above it. None of it touches `.github/`, so this PR neither blocks nor is blocked by any of them:

1. [#98](https://github.com/davidfowl/tally/pull/98) — glob pattern docs, examples, and CLI tests — [incremental diff](https://github.com/terryaney/OpenSource.tally/compare/main...feature/globbing-documentation)
2. [#99](https://github.com/davidfowl/tally/pull/99) — make the report JSON byte-reproducible — [incremental diff](https://github.com/terryaney/OpenSource.tally/compare/feature/globbing-documentation...feature/report-json-determinism)
3. [#100](https://github.com/davidfowl/tally/pull/100) — date filtering, transaction details, per-transaction tag classification — [incremental diff](https://github.com/terryaney/OpenSource.tally/compare/feature/report-json-determinism...feature/ui-tweaks)
4. [#101](https://github.com/davidfowl/tally/pull/101) — reimagined chart layout, KPI tiles, chart docs — [incremental diff](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined)
5. [#102](https://github.com/davidfowl/tally/pull/102) — composite merchant identity so one merchant can carry multiple categorizations — [incremental diff](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys)
6. [#103](https://github.com/davidfowl/tally/pull/103) — categorization review file, `review:` rule flag, `inventory.yaml`

Because the branches are stacked but every PR targets `main`, each one's diff on GitHub includes the PRs above it — #98 is 3 files, #103 is 50. The "incremental diff" links above show just that PR's own contribution, which is the view worth reading them in.
