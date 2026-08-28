# Branching Strategy

## Overview

```
main        ← production; protected; tagged releases
  │
  └── develop      ← integration/staging; CI required
        │
        ├── feature/TICKET-description    (new features)
        ├── fix/TICKET-description        (bug fixes)
        ├── chore/description             (maintenance)
        ├── docs/description              (documentation)
        └── ci/description               (CI/CD changes)

hotfix/description    ← emergency fix; PR directly to main
release/vX.Y.Z        ← release candidate; PR to main
```

## Branch Types

| Type | Naming | Target | Use For |
|---|---|---|---|
| `feature/*` | `feature/TICKET-description` | `develop` | New features |
| `fix/*` | `fix/TICKET-description` | `develop` | Bug fixes |
| `hotfix/*` | `hotfix/description` | `main` + backport to `develop` | Critical production fixes |
| `chore/*` | `chore/description` | `develop` | Dependency updates, refactors |
| `docs/*` | `docs/description` | `develop` | Documentation only |
| `ci/*` | `ci/description` | `develop` | Workflow changes |
| `release/*` | `release/vX.Y.Z` | `main` | Release candidates |

## Naming Rules

- Lowercase only — no uppercase, no spaces
- Use hyphens to separate words — not underscores
- Include ticket/issue number when one exists: `feature/123-add-oauth`
- Keep it short but descriptive: `fix/null-flight-date` not `fix/f`

**Valid examples:**
```
feature/42-strava-sync
fix/15-null-pointer-in-flight-log
hotfix/auth-token-expiry
chore/upgrade-react-19
release/v2.1.0
```

**Invalid:**
```
Feature/AddStrava          ← uppercase
new-feature                ← no type prefix
feature/my feature         ← spaces
fix/f                      ← too vague
```

## Workflow for Each Type

### Feature development

```bash
git checkout develop
git pull origin develop
git checkout -b feature/123-your-feature

# ... make changes, commit with Conventional Commits ...
git commit -m "feat: add Strava activity sync"

# Push and open PR to develop
git push -u origin feature/123-your-feature
gh pr create --base develop --title "feat: add Strava activity sync"
```

### Bug fix

```bash
git checkout develop
git pull origin develop
git checkout -b fix/456-description

git commit -m "fix: resolve null pointer when no flights logged"
gh pr create --base develop --title "fix: resolve null pointer when no flights logged"
```

### Hotfix (critical production bug)

```bash
# Branch from main, not develop
git checkout main
git pull origin main
git checkout -b hotfix/auth-token-leak

git commit -m "fix: revoke leaked session tokens on logout"

# PR to main
gh pr create --base main --title "hotfix: revoke leaked session tokens on logout"

# After merge to main, backport to develop
git checkout develop
git pull origin develop
git cherry-pick <commit-sha>
git push origin develop
```

### Release

```bash
git checkout develop
git pull origin develop
git checkout -b release/v2.1.0

# Final QA, bump version in package.json if needed
git commit -m "chore(release): v2.1.0"

gh pr create --base main --title "Release v2.1.0"
# After merge, the release workflow auto-creates changelog + GitHub release
```

## Commit Convention (Conventional Commits)

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

| Type | Version bump | Example |
|---|---|---|
| `feat:` | minor | `feat: add Garmin heart rate zones` |
| `fix:` | patch | `fix: correct pace calculation for treadmill` |
| `feat!:` or `BREAKING CHANGE:` | major | `feat!: redesign training metrics API` |
| `chore:`, `docs:`, `ci:`, `style:` | none | `chore: upgrade eslint to v9` |
| `perf:` | patch | `perf: cache Strava token refresh` |
| `test:` | none | `test: add integration tests for flight log` |

## Protected Branch Rules

### `main`
- Require PR — no direct push (except automated release commits)
- Required status checks: `ci / lint`, `ci / test`, `pr-review / Code review`
- Require branches to be up to date before merge
- Require 1 human approval
- Restrict who can push: admins only

### `develop`
- Require PR — no direct push
- Required status checks: `ci / lint`, `ci / test`
- Squash merge recommended (clean history)
