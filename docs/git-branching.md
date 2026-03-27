# Git & Branching

How to manage your git history, branches, and pull requests — from solo development through your first team hires.

## Branch Strategy

### Solo Phase: Trunk-Based with Feature Branches

When you're the only developer, keep it simple:

```
main (always deployable)
  └── feature/short-description
  └── fix/short-description
```

- `main` is your production branch. It should always be in a deployable state.
- Create short-lived feature branches for anything that takes more than a single commit.
- Merge via squash merge to keep `main` history clean and bisectable.

**Why not just commit to main?** Even solo, branches give you a clean rollback point and let CI run before you merge. The habit also pays off immediately when you add your first teammate.

### Team Phase: Add Protected Main

When you bring on your first teammate:

- Enable branch protection on `main`: require PR reviews and passing CI.
- Add a `staging` branch if you need a pre-production environment.
- Keep feature branches short-lived (1–3 days max). Long-lived branches are where merge conflicts breed.

## Branch Naming

Use a consistent prefix so branches are scannable:

| Prefix | Use |
|--------|-----|
| `feature/` | New functionality |
| `fix/` | Bug fixes |
| `refactor/` | Code restructuring without behavior change |
| `docs/` | Documentation only |
| `ci/` | CI/CD pipeline changes |
| `experiment/` | Throwaway exploration (delete after learning) |

Examples:
```
feature/user-auth-flow
fix/token-refresh-race-condition
refactor/extract-db-connection-pool
```

## Commit Messages

Use [Conventional Commits](https://www.conventionalcommits.org/) format:

```
<type>(<scope>): <short summary>

<optional body — explain WHY, not WHAT>

<optional footer — breaking changes, issue refs>
```

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `ci`, `chore`

### Good Examples

```
feat(auth): add JWT refresh token rotation

Tokens now rotate on each refresh to limit the window of a stolen token.
Previous behavior kept the same refresh token until expiry.

Closes #42
```

```
fix(api): handle empty request body without panic

The handler assumed body was always present because the middleware
validated it — but the health check endpoint bypasses that middleware.
```

### Bad Examples

```
# Too vague — what was updated?
update stuff

# Describes what (obvious from the diff), not why
changed line 42 in auth.py

# No type, no scope, no context
fix bug
```

## Pull Request Workflow

### Creating a PR

1. Push your branch: `git push -u origin feature/your-feature`
2. Open a PR with:
   - **Title**: Same format as commit messages (`feat(scope): summary`)
   - **Description**: What changed, why, and how to test it
   - **Size**: Aim for under 400 lines changed. If it's bigger, consider splitting.

### PR Description Template

```markdown
## What

Brief description of the change.

## Why

What problem this solves or what goal it advances.

## How to Test

Steps to verify this works, or note that tests cover it.

## Notes

Anything reviewers should know — tradeoffs, follow-up work, etc.
```

### Solo PR Workflow

Even when you're the only developer, open PRs for non-trivial changes:

- It gives CI a chance to run before you merge.
- It creates a searchable history of *why* changes were made.
- It's a natural place to have your AI assistant review the diff.
- When you add teammates, the workflow is already in place.

## Git Hygiene

### Keep Commits Atomic

Each commit should represent one logical change. If you find yourself writing "and" in a commit message, consider splitting into two commits.

### Rebase Before Merge

If `main` has moved ahead of your branch:

```bash
git fetch origin
git rebase origin/main
```

This keeps the history linear and avoids merge commits that obscure the real changes.

### Clean Up After Merge

Delete branches after they're merged — both remote and local:

```bash
# Delete remote branch (usually handled by GitHub merge settings)
git push origin --delete feature/your-feature

# Delete local branch
git branch -d feature/your-feature

# Prune stale remote tracking branches
git fetch --prune
```

### Protect Sensitive Data

**Never commit secrets.** Use a `.gitignore` that covers:

```gitignore
# Environment and secrets
.env
.env.*
*.pem
*.key
credentials.json

# OS and editor
.DS_Store
*.swp
*~

# Language-specific
__pycache__/
*.pyc
target/
node_modules/
```

Consider adding a pre-commit hook or tool like `git-secrets` to catch accidental secret commits before they happen.

## Scaling Checklist

As your team grows, add these in order of need:

- [ ] Branch protection rules on `main`
- [ ] Required PR reviews (start with 1 reviewer)
- [ ] Automated CI checks as merge gates
- [ ] CODEOWNERS file for automatic reviewer assignment
- [ ] Signed commits (when security posture demands it)
