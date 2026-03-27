# Environment Setup

How to manage local development, staging, and production environments. The goal: any developer (including future-you after reformatting your laptop) can go from zero to running code in under 15 minutes.

## Local Development

### Reproducible Setup

Your project should have a single entry point for getting started:

```bash
git clone <repo>
cd <repo>
make setup    # or: ./scripts/setup.sh
```

This script should:
1. Check for required system dependencies and tell the user what's missing
2. Create virtual environments / install toolchains
3. Install project dependencies
4. Set up local config from a template
5. Run a smoke test to verify everything works

Example `Makefile`:
```makefile
.PHONY: setup dev test lint clean

setup:
	@echo "Checking dependencies..."
	@command -v python3.12 >/dev/null || (echo "Python 3.12 required" && exit 1)
	@command -v cargo >/dev/null || (echo "Rust toolchain required" && exit 1)
	@command -v node >/dev/null || (echo "Node.js required" && exit 1)
	python3.12 -m venv .venv
	.venv/bin/pip install -e ".[dev]"
	npm ci
	@test -f .env || cp .env.example .env
	@echo "Setup complete. Run 'make dev' to start."

dev:
	.venv/bin/python -m src.main

dev-frontend:
	npm run dev

test:
	.venv/bin/pytest
	cargo test
	npx vitest run

lint:
	.venv/bin/ruff check .
	.venv/bin/ruff format --check .
	cargo fmt --check
	cargo clippy -- -D warnings
	npx prettier --check .
	npx eslint .
	npx tsc --noEmit

clean:
	rm -rf .venv target/ __pycache__/ .pytest_cache/ node_modules/ dist/
```

### Python Environment

Use virtual environments. Always.

```bash
# Create
python3.12 -m venv .venv

# Activate
source .venv/bin/activate

# Install with dev dependencies
pip install -e ".[dev]"
```

Pin dependencies for reproducibility:
- `pyproject.toml` — direct dependencies with version ranges
- `requirements.lock` or `uv.lock` — exact pinned versions for reproducible installs

```toml
# pyproject.toml
[project]
dependencies = [
    "fastapi>=0.110",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "ruff>=0.4",
    "pytest-cov",
]
```

### Rust Environment

```bash
# Install via rustup (if not present)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Project setup is automatic — cargo handles it
cargo build
cargo test
```

Pin the Rust toolchain per project:
```toml
# rust-toolchain.toml
[toolchain]
channel = "stable"
components = ["clippy", "rustfmt"]
```

### TypeScript / React Environment

Use Node.js LTS and npm for package management:

```bash
# Install Node.js via nvm (if not present)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install 22
nvm use 22
```

Project setup with Vite (fast builds, PWA support out of the box):

```bash
# Create a new React + TypeScript project
npm create vite@latest frontend -- --template react-ts
cd frontend
npm install
```

Key configuration files:
```
frontend/
├── package.json           # Dependencies and scripts
├── package-lock.json      # Pinned dependency tree (always commit this)
├── tsconfig.json          # TypeScript compiler options
├── vite.config.ts         # Build tool configuration
├── eslint.config.js       # Linting rules
├── .prettierrc            # Formatting rules
└── src/
    ├── main.tsx           # Entry point
    └── App.tsx            # Root component
```

Pin the Node version per project:
```
# .nvmrc
22
```

Then any developer can run `nvm use` to switch to the correct version.

#### PWA Configuration

For push notifications and installability, add the Vite PWA plugin:

```bash
npm install -D vite-plugin-pwa
```

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { VitePWA } from "vite-plugin-pwa";

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      registerType: "autoUpdate",
      manifest: {
        name: "Your App",
        short_name: "App",
        display: "standalone",
        background_color: "#ffffff",
        theme_color: "#000000",
      },
      workbox: {
        // Cache API responses for offline support
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/api\./,
            handler: "NetworkFirst",
            options: {
              cacheName: "api-cache",
              expiration: { maxEntries: 50, maxAgeSeconds: 300 },
            },
          },
        ],
      },
    }),
  ],
});
```

This gives you:
- **Installable PWA** — "Add to Home Screen" on mobile, app-like experience
- **Push notifications** — via the Web Push API and service worker
- **Offline support** — cached API responses and static assets
- **Auto-update** — new versions deploy without app store review

## Environment Variables

### The .env Pattern

```bash
# .env.example — committed to git, documents every variable
DATABASE_URL=postgres://localhost:5432/myapp_dev
API_KEY=your-api-key-here
LOG_LEVEL=debug
ENVIRONMENT=development

# .env — NOT committed, contains real values
# Created by copying .env.example during setup
```

**Rules:**
- `.env.example` is committed. It documents every variable the app needs, with safe placeholder values.
- `.env` is gitignored. It holds real credentials for your local setup.
- The app should fail loudly at startup if a required variable is missing — not silently use a default.

**Frontend note:** Vite exposes environment variables prefixed with `VITE_` to client-side code. Never put secrets in `VITE_` variables — they're embedded in the JavaScript bundle and visible to anyone.

```bash
# .env.example additions for frontend
VITE_API_BASE_URL=http://localhost:8000
VITE_VAPID_PUBLIC_KEY=your-vapid-public-key-here
# Note: VAPID *private* key stays server-side only, never prefixed with VITE_
```

### Loading Environment Variables

Python:
```python
from os import environ

def get_required_env(key: str) -> str:
    """Retrieve a required environment variable or fail fast.

    Args:
        key: The environment variable name.

    Returns:
        The environment variable value.

    Raises:
        RuntimeError: If the variable is not set.
    """
    value = environ.get(key)
    if value is None:
        raise RuntimeError(f"Required environment variable {key} is not set")
    return value
```

## Environments

### Development (Local)

- Runs on your machine
- Uses local or containerized databases
- Debug logging enabled
- Hot reload where possible

### Staging

- Mirrors production infrastructure as closely as possible
- Uses production-like data (anonymized if it contains PII)
- Deployed automatically from `main` (or a `staging` branch)
- Where you catch integration issues that local dev misses

### Production

- Deployed through CI/CD pipeline only — never manually
- Minimal logging (no debug, no PII in logs)
- Monitoring and alerting enabled
- Secrets managed through your cloud provider's secret manager

### Environment Parity

The closer your environments are to each other, the fewer surprises in production:

| Concern | Dev | Staging | Prod |
|---------|-----|---------|------|
| Database | Local/container | Same engine, smaller instance | Full instance |
| Secrets | .env file | CI secrets | Secret manager |
| Logging | Debug level | Info level | Warn/Error level |
| Data | Seed/fixture data | Anonymized prod-like | Real |

## Docker for Local Services

When your app depends on external services (databases, caches, queues), use Docker Compose for local dev:

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: myapp_dev
      POSTGRES_PASSWORD: localdev
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

```bash
# Start services
docker compose up -d

# Stop services
docker compose down

# Reset everything (destroys data)
docker compose down -v
```

## Scaling Checklist

- [ ] Setup script that works from a fresh clone (day one)
- [ ] `.env.example` documenting all required variables (day one)
- [ ] Makefile or equivalent with `setup`, `dev`, `test`, `lint` targets (day one)
- [ ] `.nvmrc` pinning Node version (day one for frontend projects)
- [ ] Docker Compose for local services (when you need databases/caches)
- [ ] Staging environment (when you need pre-production validation)
- [ ] Infrastructure as Code for staging/prod (when manual setup becomes error-prone)
