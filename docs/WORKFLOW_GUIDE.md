# Workflow Guide

## Workflow Summary

| Workflow | File | Triggers | Key Output |
|---|---|---|---|
| CI | `ci.yml` | Push to any branch, PR to main/develop | Lint ✓, Test ✓, Typecheck ✓ |
| Claude PR Review | `pr-review.yml` | PR opened/updated to main or develop | Inline code review comments |
| Claude Assist | `claude-assist.yml` | `/claude` in any issue or PR comment | AI response in-thread |
| Release | `release.yml` | Merge to main | Changelog, tag, GitHub Release |
| Branch Guard | `branch-guard.yml` | Push to non-protected branch, PR to main/develop | Name validation |

## CI Workflow

Runs on every push and PR. Three parallel jobs after the guard:

```
guard ──┬── lint       ─┐
        ├── typecheck    ├── (lint must pass before test)
        └── audit        └── test
```

**Guard:** Detects runtime (node/python/go/rust/make), checks for committed secrets.

**Lint:** ESLint + Prettier (Node), ruff (Python), golangci-lint (Go), clippy + rustfmt (Rust). Also runs `actionlint` on all workflows.

**Typecheck:** TypeScript (`tsc --noEmit`) or mypy (Python).

**Test:** Runs your test command; uploads results as artifacts (7-day retention).

**Audit:** npm audit or pip-audit on PRs only (not blocking, but reported).

### Configuring for your stack

In `ci.yml`, set the `RUNTIME` env var:
```yaml
env:
  RUNTIME: node   # node | python | go | rust | make
```

Or leave it as `node` and let the guard auto-detect from your project files.

### Adding a custom test command

```yaml
- name: Test (custom)
  run: your-custom-test-command
```

Add it under the `test` job, alongside the existing runtime-specific steps.

## Claude PR Review

Triggers automatically when a non-draft PR is opened or updated targeting `main` or `develop`.

**What it checks:**
- Architecture and SOLID principles
- Security — OWASP Top 10, JWT, race conditions, secrets in code
- Code quality — error handling, performance, boundary conditions
- Test coverage — new code tested? regression tests for fixes?

**Severity:**
- 🔴 P0 Critical — blocks merge
- 🟠 P1 High — should fix before merge
- 🟡 P2 Medium — fix or track in follow-up issue
- 🟢 P3 Low — nit; summary only, no inline comment

**Verdict:** `REQUEST_CHANGES` (P0) → `COMMENT` (P1+) → `APPROVE` (P2/P3 or clean)

The review never approves or merges — human approval is always required on `main` PRs.

## Claude Assist

Type `/claude` followed by your request in any issue or PR comment:

```
/claude why is the CI failing on test step?
/claude review the auth middleware for security issues
/claude what's the best approach to add rate limiting?
/claude write a test for the flight log export function
/claude suggest a fix for the bug described in this issue
```

Claude will:
1. React with 👀 to acknowledge
2. Read project context (CLAUDE.md, AGENTS.md)
3. Read the issue/PR for context
4. Post a response in the thread

Claude can read files and run commands but won't push code unless you explicitly ask it to implement something.

## Release Workflow

Triggered automatically on every merge to `main`.

**How versioning works (Conventional Commits):**

| Commits since last tag | Bump |
|---|---|
| Any `feat!:` or `BREAKING CHANGE:` | Major (1.0.0 → 2.0.0) |
| Any `feat:` (no breaking) | Minor (1.0.0 → 1.1.0) |
| Any `fix:` or `perf:` | Patch (1.0.0 → 1.0.1) |
| Only `chore:`, `docs:`, `ci:`, `style:` | No release |

**What it does:**
1. Determines the version bump from commit messages
2. Generates a changelog from commits since last tag
3. Prepends to CHANGELOG.md and commits it
4. Creates an annotated git tag (`v1.2.3`)
5. Creates a GitHub Release with the changelog as release notes

If no releasable commits are found (all chores/docs), no release is created.

## Branch Guard

Runs on every push to a non-protected branch and on every PR.

**Checks:**
1. Branch name matches pattern: `<type>/<description>`
2. PR targets the correct base branch:
   - `feature/*`, `fix/*`, `chore/*`, `docs/*` → must target `develop`
   - `release/*` → must target `main`
   - `hotfix/*` → can target `main` or `develop`
3. PR description has at least 20 characters
4. PR links to an issue (advisory, not blocking)

**If branch name is invalid:**
```
❌ Invalid branch name: my-new-feature
Branch names must follow the pattern: <type>/<description>
Valid types: feature | fix | hotfix | chore | release | ci | docs | test | refactor | perf

Rename your branch:
  git branch -m my-new-feature feature/your-description
  git push origin -u feature/your-description
```

## Setting Up Branch Protection

After applying this framework to your repo, configure branch protection:

**For `main`:**
```
Settings → Branches → Add rule → Branch name pattern: main
✅ Require a pull request before merging
  ✅ Require approvals: 1
✅ Require status checks to pass before merging
  ✅ Require branches to be up to date
  Status checks: ci / Guard, ci / Lint, ci / Test, pr-review / Code review
✅ Restrict who can push to matching branches
✅ Allow force pushes: off
✅ Allow deletions: off
```

**For `develop`:**
```
Settings → Branches → Add rule → Branch name pattern: develop
✅ Require a pull request before merging
✅ Require status checks to pass before merging
  Status checks: ci / Guard, ci / Lint, ci / Test
✅ Allow force pushes: off
```
