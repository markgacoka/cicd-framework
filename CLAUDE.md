# Project Context for Claude

## What This Repo Is

A CI/CD framework template. It provides GitHub Actions workflows, branch conventions, and Claude Code integration patterns that any project can adopt.

## Branch Rules

| Branch | Purpose | PR target |
|---|---|---|
| `main` | Production | — |
| `develop` | Integration | — |
| `feature/*` | New features | `develop` |
| `fix/*` | Bug fixes | `develop` |
| `hotfix/*` | Critical production fixes | `main` (then backport to `develop`) |
| `chore/*` | Maintenance, deps, refactors | `develop` |
| `release/*` | Release candidates | `main` |

Always work on a branch. Never commit directly to `main` or `develop`.

## Commit Convention (Conventional Commits)

```
feat:     New feature
fix:      Bug fix
docs:     Documentation only
style:    Formatting, missing semicolons (no logic change)
refactor: Code restructure (no feature or fix)
perf:     Performance improvement
test:     Adding or correcting tests
chore:    Build process, dependencies, tooling
ci:       CI/CD changes
```

Breaking change: add `!` after type — `feat!:` triggers a major version bump.

## PR Requirements

1. Branch named correctly (enforced by `branch-guard.yml`)
2. All CI checks pass (`ci / test`, `ci / lint`)
3. Claude PR review completed (no P0 findings)
4. At least one human approval on PRs to `main`

## When Editing Workflows

- Keep jobs idempotent — they must produce the same result on re-run
- Never hardcode secrets — use `${{ secrets.NAME }}`
- Test workflow changes on a `chore/` branch before merging to `develop`
- Document any non-obvious step with an inline comment

## Skill References

- See `~/.claude/skills/cicd/SKILL.md` for CI/CD management guidance
- See `~/.claude/skills/pr-review/SKILL.md` for PR review procedures
