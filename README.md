# dev-workflows

A practical engineering handbook for solo founders scaling to teams.

This repo documents development workflows, best practices, and standards that start simple for one person and grow with your team. Every recommendation is grounded in real-world tradeoffs — not cargo-culted from big-company playbooks.

## Who This Is For

- Solo founders writing code and shipping product
- Small teams (2–5 engineers) establishing their first shared practices
- Anyone who wants sane defaults before they need a 50-page engineering wiki

## Philosophy

1. **Start lean, add structure when it hurts.** Don't adopt a process until the absence of that process causes a real problem.
2. **Automate the boring stuff.** If a human has to remember to do it, it will eventually be forgotten.
3. **Document decisions, not just rules.** Every standard should explain *why* it exists so future-you (or new teammates) can decide when to break it.
4. **AI-assisted, human-accountable.** Use AI coding assistants aggressively, but own every line that ships.

## Contents

| Section | Description |
|---------|-------------|
| [Environment Setup](docs/environment-setup.md) | Local dev, staging, production environment management |
| [Onboarding](docs/onboarding.md) | New engineer checklist, access provisioning, first-week guide |
| [Secure Coding](docs/secure-coding.md) | Input validation, auth, API security, data protection, XSS prevention |
| [Security Practices](docs/security.md) | Secrets management, dependency scanning, secure defaults |
| [Git & Branching](docs/git-branching.md) | Branch strategy, commit conventions, PR workflow |
| [Coding Style](docs/coding-style.md) | Documentation standards, naming, formatting, error handling |
| [Testing](docs/testing.md) | Test strategy, coverage expectations, when to test what |
| [Code Review](docs/code-review.md) | Review standards, what to look for, async review tips |
| [CI/CD](docs/ci-cd.md) | Pipeline design, automated checks, deployment gates |
| [Deployment](docs/deployment.md) | Release process, rollback strategy, deploy checklist |
| [Workflow Automation](docs/workflow-automation.md) | Enforcing standards via CI, AI review commands, and PR templates |
| [AI-Assisted Development](docs/ai-assisted-dev.md) | Working with Claude Code and other AI coding assistants |
| [Human-Led Multi-Agent Workflows](docs/multi-agent-workflows.md) | Coordinating multiple agents through GitHub, Slack handoffs, review gates, and long-running approvals |
| [Agent Instructions Reference](docs/agent-instructions-reference.md) | Template rules for project-local agent roles, communication, safety gates, and runtime awareness |

## Stack

This handbook is written with the following stack in mind, but most practices are language-agnostic:

- **Python** — ML/AI workloads, APIs, scripting, anywhere libraries matter more than speed
- **Rust** — Performance-critical services, security tooling, systems programming
- **React + TypeScript** — PWA-first web apps with push notifications, responsive across mobile and desktop

## Contributing

This is an open resource. If something helped you or you have a better approach, open a PR. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

[MIT](LICENSE)
