# cicd-framework

Production-ready CI/CD framework with Claude Code AI integration.

## What's Included

| Component | Purpose |
|---|---|
| **Branch strategy** | `main → develop → feature/* / fix/* / hotfix/*` |
| **CI pipeline** | Lint, test, typecheck on every push and PR |
| **Claude PR review** | Automated code review: architecture, security (OWASP), code quality |
| **Claude assist** | `/claude <request>` in any issue or PR comment |
| **Release automation** | Auto-changelog and GitHub release on merge to `main` |
| **Branch guard** | Enforces naming conventions; blocks bad branch names |
| **Issue templates** | Bug reports and feature requests with structured fields |

## Branch Strategy

```
main        ← production; protected; deploy on merge
  └── develop      ← integration; PRs from feature/* and fix/*
        ├── feature/<ticket>-<description>   (new features)
        ├── fix/<ticket>-<description>       (bug fixes)
        └── chore/<description>             (maintenance)

hotfix/<description>  ← critical fix; PRs directly to main + develop
release/<version>     ← release candidate; PR to main
```

**Naming rules** (enforced by `branch-guard.yml`):
- `main`, `develop` — protected, no direct push
- Feature: `feature/TICKET-short-description`
- Fix: `fix/TICKET-short-description`
- Hotfix: `hotfix/short-description`
- Chore: `chore/short-description`
- Release: `release/vX.Y.Z`

## Setup for Your Project

### 1. Copy the workflows
```bash
cp -r .github/ your-project/.github/
cp CLAUDE.md AGENTS.md your-project/
```

### 2. Add the required secret
In your repo: **Settings → Secrets and variables → Actions → New repository secret**

| Secret | Value |
|---|---|
| `ANTHROPIC_API_KEY` | Your Anthropic API key |

### 3. Configure branch protection
In **Settings → Branches**, add rules for `main` and `develop`:
- ✅ Require a pull request before merging
- ✅ Require status checks to pass (`ci / test`, `ci / lint`)
- ✅ Require branches to be up to date
- ✅ Restrict who can push (admins only for `main`)

### 4. Customize the CI matrix
Edit `.github/workflows/ci.yml` — set `RUNTIME` to match your stack:
```yaml
env:
  RUNTIME: node   # node | python | go | rust | make
```

## Using Claude Assist

In any issue or PR comment, type:
```
/claude review this PR for security issues
/claude explain the failing test in ci run #42
/claude suggest a fix for the bug in issue #15
/claude what's the best approach for adding rate limiting here?
```

Claude will respond directly in the comment thread.

## Release Flow

1. Merge feature branches to `develop`
2. When ready to release, open a PR from `develop` → `main`
3. On merge to `main`, the release workflow automatically:
   - Detects the version bump (from commit messages using Conventional Commits)
   - Generates a changelog from merged PRs
   - Creates a git tag
   - Publishes a GitHub Release

**Commit convention:**
```
feat: add Strava integration          → minor version bump
fix: resolve null pointer in flight   → patch version bump
feat!: redesign dashboard API         → major version bump
chore: update dependencies            → no bump
```

## Workflow Reference

| Workflow | Triggers | What it does |
|---|---|---|
| `ci.yml` | push to any branch, PR to main/develop | Lint, test, typecheck |
| `pr-review.yml` | PR opened/updated to main or develop | Claude code review → inline comments |
| `claude-assist.yml` | `/claude` comment in any issue/PR | Claude responds in-thread |
| `release.yml` | Merge to `main` | Changelog, tag, GitHub release |
| `branch-guard.yml` | Push to any non-protected branch | Validate branch name |

## Docs

- [Branching Strategy](docs/BRANCHING_STRATEGY.md)
- [Workflow Guide](docs/WORKFLOW_GUIDE.md)
