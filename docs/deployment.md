# Deployment

How to get code from `main` to production safely. Start with simple, manual-but-documented deploys and automate as you grow.

## Principles

1. **Deployments should be boring.** If deploying makes you nervous, your process has gaps.
2. **Every deploy is reversible.** You should always be able to roll back in under 5 minutes.
3. **Deploy small, deploy often.** Small deploys are easier to debug when something goes wrong.
4. **Never deploy on Friday.** (Unless you have monitoring and are willing to fix things over the weekend.)

## Solo Phase: Simple Deploy Script

When you're the only developer, a deploy script is enough:

```bash
#!/usr/bin/env bash
# scripts/deploy.sh — Deploy to production
set -euo pipefail

ENVIRONMENT="${1:?Usage: ./scripts/deploy.sh <staging|production>}"
DEPLOY_SHA="$(git rev-parse --short HEAD)"

echo "Deploying ${DEPLOY_SHA} to ${ENVIRONMENT}..."

# Pre-deploy checks
echo "Running tests..."
make test

echo "Running linter..."
make lint

# Build
echo "Building..."
# Python: build container or package
# Rust: cargo build --release --target x86_64-unknown-linux-gnu

# Deploy (customize for your infrastructure)
# Examples:
# - scp binary to server
# - docker push + restart
# - fly deploy
# - railway up

echo "Deployed ${DEPLOY_SHA} to ${ENVIRONMENT} at $(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

### Deploy Checklist (Manual)

Before every production deploy:

```
□ All tests pass locally
□ CI is green on main
□ Changes have been tested on staging (if you have one)
□ Database migrations are backward-compatible
□ No secrets in the code
□ You know how to roll back if something breaks
```

## Automated Deployment via CI

When you're ready to automate, add a deploy step to your CI pipeline:

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    # ... your existing CI checks ...

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        run: ./scripts/deploy.sh staging
        env:
          DEPLOY_TOKEN: ${{ secrets.STAGING_DEPLOY_TOKEN }}

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      # Require manual approval for production deploys
      # Configure this in GitHub Settings → Environments
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        run: ./scripts/deploy.sh production
        env:
          DEPLOY_TOKEN: ${{ secrets.PROD_DEPLOY_TOKEN }}
```

**Key detail:** The `environment: production` with a required reviewer means production deploys pause for manual approval. This gives you an automated pipeline with a human gate for production.

## Rollback Strategy

### Know Your Rollback Before You Deploy

Every deployment approach needs a corresponding rollback:

| Deploy Method | Rollback Method |
|---------------|-----------------|
| Binary on server via SCP | Keep previous binary, swap symlink |
| Docker container | `docker pull <previous-tag> && docker restart` |
| Platform deploy (Fly, Railway) | Platform rollback command |
| Database migration | Backward-compatible migrations (no destructive changes) |

### Binary Rollback with Symlinks

```bash
# Deploy structure on server
~/app/
├── current -> releases/v42/     # symlink to active release
├── releases/
│   ├── v41/                     # previous release (keep for rollback)
│   └── v42/                     # current release
└── shared/                      # config, logs, persistent data

# Deploy new version
mkdir -p ~/app/releases/v43/
cp binary ~/app/releases/v43/
ln -sfn ~/app/releases/v43 ~/app/current
sudo systemctl restart myapp

# Rollback
ln -sfn ~/app/releases/v42 ~/app/current
sudo systemctl restart myapp
```

### Database Migrations

**Golden rule: migrations must be backward-compatible.**

This means:
- **Adding a column**: Safe. Add with a default value or allow NULL.
- **Removing a column**: Do it in two deploys. First deploy: stop reading the column. Second deploy: drop it.
- **Renaming a column**: Do it in three steps. Add new column → migrate data → remove old column.
- **Adding a table**: Safe.
- **Dropping a table**: Only after confirming nothing reads from it.

Why? If you need to roll back your code, the old code needs to work with the current database schema.

## Release Tagging

Tag every production deploy:

```bash
# After successful deploy
git tag -a v1.2.3 -m "Release v1.2.3: brief description"
git push origin v1.2.3
```

Use [Semantic Versioning](https://semver.org/):
- **MAJOR**: Breaking changes to your API or user-facing behavior
- **MINOR**: New features, backward-compatible
- **PATCH**: Bug fixes, backward-compatible

For early-stage products, `v0.x.y` signals that the API is still evolving.

## Monitoring After Deploy

Every deploy should be followed by a brief observation period:

1. **Check logs** for errors or unexpected warnings.
2. **Check key metrics** — response times, error rates, resource usage.
3. **Smoke test** the critical user paths manually.
4. **Set a timer** — if nothing breaks in 15 minutes, you're probably fine.

If you don't have monitoring yet, at minimum check your application logs for the first few minutes after deploying.

## Scaling Checklist

- [ ] Documented deploy process (even if manual) (day one)
- [ ] Deploy script that runs tests before deploying (day one)
- [ ] Rollback plan documented and tested (before first production deploy)
- [ ] Release tags on every production deploy (day one)
- [ ] Automated deploy to staging from CI (when you have staging)
- [ ] Manual approval gate for production deploys (when you have a team)
- [ ] Health checks and basic monitoring (before trusting automated deploys)
- [ ] Blue-green or canary deployments (high-traffic applications)
