# Architecture Rules for Agents

## Inviolable Rules

1. **No direct push to `main` or `develop`** — PRs only; branch protection is not optional
2. **No hardcoded secrets** — all credentials via `${{ secrets.NAME }}` in workflows
3. **No skipping CI** — `[skip ci]` is banned except for pure docs changes (`.md` files only)
4. **No auto-merge to `main`** — human approval required on every PR to production
5. **No deleting merged branches from `main`** — preserve history

## Workflow Conventions

- **Jobs must be idempotent** — re-running a workflow must not cause duplicate comments, releases, or tags
- **Fail fast** — lint before test; if lint fails, skip test
- **Artifact retention** — test results kept 7 days, build artifacts kept 30 days
- **Timeout** — all jobs have an explicit `timeout-minutes` to prevent runaway billing

## Security Rules

- `ANTHROPIC_API_KEY` and all third-party tokens stored in repo-level secrets, never workflow env vars
- Workflows triggered by `pull_request` from forks run with read-only permissions (`read` default, no secrets)
- External PRs: Claude review runs but cannot post comments (safe degradation)
- `pull-requests: write` permission granted only to jobs that post comments
- Pin third-party actions to their SHA, not a mutable tag — audit quarterly

## PR Review Rules (from pr-review.yml)

- Claude posts inline comments for P0, P1, P2 findings only
- P3 (nits) go in the summary comment only — no inline noise
- Claude never approves or merges a PR
- Claude never runs code from the PR — static analysis only
- Duplicate comment check before every post

## Release Rules

- Versions follow Semantic Versioning: `vMAJOR.MINOR.PATCH`
- Tags are immutable once pushed — no force-push on tags
- Every release has a corresponding GitHub Release with changelog
- Changelog generated from PR titles and commit messages (Conventional Commits)

## Branch Naming (enforced by branch-guard.yml)

```
^(feature|fix|hotfix|chore|release|ci|docs|test|refactor|perf)/[a-z0-9]([a-z0-9-]*[a-z0-9])?$
```
Or: `main`, `develop`

Branches that don't match are blocked at push time with a clear error message.
