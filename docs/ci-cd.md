# CI/CD

Continuous Integration and Continuous Deployment — automating the path from code to production. Start simple, add complexity only when you need it.

## Philosophy

- **CI from day one.** Even solo, automated checks catch things you'll miss. The cost is 15 minutes of setup; the payoff is permanent.
- **Fast feedback loops.** If CI takes more than 10 minutes, developers stop waiting for it and merge anyway.
- **The pipeline is code.** Version your CI config in the repo. Never configure pipelines through a web UI alone.
- **Failed pipeline = blocked merge.** CI is only useful if you actually enforce it.

## Solo Phase: Minimal CI

Start with GitHub Actions. One workflow file covers the basics:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  check-python:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[dev]"
      - run: ruff check .
      - run: ruff format --check .
      - run: pytest --tb=short

  check-rust:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
      - run: cargo test

  check-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"
      - run: npm ci
      - run: npx prettier --check .
      - run: npx eslint .
      - run: npx tsc --noEmit
      - run: npx vitest run
```

This gives you:
- Linting and formatting checks (catches style issues automatically — no review debates)
- Full test suite on every PR and push to main
- Clear pass/fail signal before you merge

### What This Costs

- ~15 minutes to set up
- Free tier on GitHub Actions covers most solo/small-team usage
- Runs in 2–5 minutes for small-to-medium projects

## Pipeline Stages

As your project grows, structure your pipeline in stages:

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│   Lint   │───▶│  Build   │───▶│   Test   │───▶│  Deploy  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
  seconds         minutes         minutes         minutes
```

### Stage 1: Lint (Fast Feedback)

Run first because it's fast and catches obvious issues:

- **Python**: `ruff check`, `ruff format --check`
- **Rust**: `cargo fmt --check`, `cargo clippy -- -D warnings`
- **TypeScript**: `prettier --check`, `eslint`, `tsc --noEmit`
- **General**: Check for secrets with `trufflehog` or `gitleaks`

### Stage 2: Build

Verify the code compiles and dependencies resolve:

- **Python**: `pip install -e ".[dev]"` (catches dependency issues)
- **Rust**: `cargo build` (catches compilation errors)
- **TypeScript**: `npm ci` (clean install from lockfile — catches dependency issues)

### Stage 3: Test

Run the full test suite:

- Unit tests (always)
- Integration tests (when you have them)
- Coverage report (optional, for visibility)

### Stage 4: Deploy

See [Deployment](deployment.md) for details. Key principle: deployment should be a pipeline stage, not a manual process.

## Branch Protection

Set these up on `main` as soon as you have CI working:

```
Settings → Branches → Branch protection rules → main

✅ Require a pull request before merging
✅ Require status checks to pass before merging
   → Select your CI checks
✅ Require branches to be up to date before merging
✅ Do not allow bypassing the above settings
```

**Why "do not allow bypassing"?** If you can bypass protection rules, you will — at 2am when something is broken and you "just need to push this fix." Future-you will regret it.

## Secrets in CI

- Store secrets in GitHub Actions Secrets (Settings → Secrets and variables → Actions).
- Never echo secrets in CI output.
- Use environment-scoped secrets for production credentials.
- Rotate secrets on a schedule, not just when someone leaves.

```yaml
# Referencing secrets in workflows
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  API_KEY: ${{ secrets.API_KEY }}
```

## Caching for Speed

Slow pipelines get ignored. Cache dependencies:

```yaml
# Python
- uses: actions/setup-python@v5
  with:
    python-version: "3.12"
    cache: "pip"

# Rust
- uses: actions/cache@v4
  with:
    path: |
      ~/.cargo/registry
      ~/.cargo/git
      target/
    key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

# Node.js / TypeScript
- uses: actions/setup-node@v4
  with:
    node-version: "22"
    cache: "npm"
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| CI takes 20+ minutes | People merge without waiting | Parallelize, cache, split jobs |
| Flaky tests | People ignore failures | Fix or quarantine flaky tests immediately |
| No branch protection | CI runs but doesn't gate merges | Enable protection rules |
| Secrets in logs | Credential leaks | Audit CI output, use masking |
| Only testing on Linux | Mac/Windows bugs ship | Add matrix builds when relevant |

## Scaling Checklist

- [ ] Basic CI with lint + test (day one)
- [ ] Branch protection requiring CI pass (day one)
- [ ] Dependency caching for speed (week one)
- [ ] Secret scanning in CI (when you have any secrets)
- [ ] Automated deployment from CI (when you have staging/prod)
- [ ] Matrix builds for multiple OS/versions (when users are cross-platform)
- [ ] Separate staging and production deploy pipelines (when you have both environments)
