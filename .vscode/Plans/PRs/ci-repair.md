> **Independent — based directly on `main`.** Touches `.github/workflows/` only and shares no file with any of my other open PRs, so it can be merged in any order.
> **Diff:** [`main...feature/ci-repair`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/ci-repair)

`PR Build` has failed on **every fork pull request** since 2026-07-20, including ones that change nothing but documentation. This PR fixes that, and separately brings the workflow action pins up to date ahead of Node 20's removal.

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

`v4` is a moving tag, and the release it now points at refuses to check out fork code under `pull_request_target`. `pr-build.yml` did exactly that — `pull_request_target` plus `ref: <fork head sha>`.

It isn't specific to one contributor: the most recent failure at time of writing is on `sutharion-studious-meme`, and the last green run was on a branch of mine that would fail identically today.

**Bumping to `actions/checkout@v5` does not fix this.** v5 carries the same guard. The pattern itself has to change.

## The fix — split the trusted half from the untrusted half

The problem with `pull_request_target` here is that one workflow both **executes fork code** and **holds a write token**. Those are separated:

| Workflow | Trigger | Token | Touches PR code? |
|---|---|---|---|
| `pr-build.yml` | `pull_request` | `contents: read`, no secrets | yes — builds it |
| `pr-build-comment.yml` *(new)* | `workflow_run` | `pull-requests: write` | **never** |

`pr-build.yml` keeps its `prepare` and `build` jobs unchanged and loses the `comment` job. `build.yml` needed no changes at all — under `pull_request`, checking out the head SHA is allowed.

`pr-build-comment.yml` waits for `PR Build` to complete, then posts the same install comment as before.

### Passing the PR number across

The `workflow_run` event's `pull_requests` array is empty for fork PRs, so the PR number can't be read from the event. It travels as a small artifact written by `prepare`.

That artifact is produced by a run over untrusted code, so the comment workflow treats it as untrusted input and validates before use:

```bash
if ! [[ "$PR_NUMBER" =~ ^[0-9]+$ ]]; then echo "Refusing malformed pr_number"; exit 1; fi
if ! [[ "$SHORT_SHA" =~ ^[0-9a-f]{7}$ ]]; then echo "Refusing malformed short_sha"; exit 1; fi
```

Artifact poisoning is the one remaining attack surface in this pattern, and this is where it gets closed.

### Two things to expect

- **The comment won't post on this PR.** `workflow_run` only fires from the workflow file on the repository's default branch, so `pr-build-comment.yml` starts working once this is merged to `main`. The build half will go green here; the comment half is untestable from a PR by design.
- **`pull_request` resolves workflow files from the PR head**, unlike `pull_request_target`. A useful side effect: from now on, CI changes can be validated by the PR that makes them.

## Also: Node 24 action pins

Separate commit, separate problem — Node 20 is being removed from the Actions runner. Nothing is failing because of it yet (the runner already forces these actions onto Node 24 and emits only a `##[warning]`), so this half is preventative.

- `actions/checkout@v4` → `@v5` across all workflows
- `actions/upload-artifact@v4` → `@v7`, `actions/download-artifact@v4` → `@v8`
- `astral-sh/setup-uv@v4` → `@v8.1.0`
- `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: "true"` on `build.yml`, `dev-build.yml`, `test.yml`

The env var is a transition-period switch covering any action not pinned here; it can be dropped once Node 20 is fully gone.

## Scope

`.github/workflows/` only — nine files, no source, no tests, no documentation.

## Also in flight (independent of this PR)

A separate stack of feature work, each PR based on the one above it. None of it touches `.github/`, so this PR neither blocks nor is blocked by any of them:

1. [`feature/globbing-documentation`](https://github.com/terryaney/OpenSource.tally/compare/main...feature/globbing-documentation) — PR #TBD — glob pattern docs, examples, and CLI tests
2. [`feature/report-json-determinism`](https://github.com/terryaney/OpenSource.tally/compare/feature/globbing-documentation...feature/report-json-determinism) — PR #TBD — make the report JSON byte-reproducible
3. [`feature/ui-tweaks`](https://github.com/terryaney/OpenSource.tally/compare/feature/report-json-determinism...feature/ui-tweaks) — PR #TBD — date filtering, transaction details, per-transaction tag classification
4. [`feature/charts-reimagined`](https://github.com/terryaney/OpenSource.tally/compare/feature/ui-tweaks...feature/charts-reimagined) — PR #TBD — reimagined chart layout, KPI tiles, chart docs
5. [`feature/merchant-composite-keys`](https://github.com/terryaney/OpenSource.tally/compare/feature/charts-reimagined...feature/merchant-composite-keys) — PR #TBD — composite merchant identity so one merchant can carry multiple categorizations
6. [`feature/categorization`](https://github.com/terryaney/OpenSource.tally/compare/feature/merchant-composite-keys...feature/categorization) — PR #TBD — categorization review file, `review:` rule flag, `inventory.yaml`
