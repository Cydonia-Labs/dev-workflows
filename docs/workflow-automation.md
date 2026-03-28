# Workflow Automation

How to turn best practices into enforced habits. The goal is a workflow where the right thing happens automatically — developers focus on building, and the tooling catches everything else.

## The Problem with Checklists

A 20-item security checklist in a wiki is aspirational. Within a month, developers skim it. Within three months, they skip it. The problem isn't discipline — it's friction. Every manual step is a step that eventually gets missed.

The fix is layering enforcement so that most checks are automated, AI handles the pattern-matching, and humans only verify what requires judgment.

## The Developer Sign-Off Principle

In an AI-assisted workflow, the most important question is: **who is accountable for this code?**

AI can write code. AI can review code. But AI cannot be accountable. When something breaks at 2am, a human answers the page. When a security vulnerability is discovered, a human explains how it shipped. The developer who merges the code is that human.

This means every PR requires a developer sign-off — an explicit attestation:

1. **I reviewed this.** Not "I ran a command." I read the AI's review output, understood the findings, and made deliberate decisions about each one.
2. **I understand this.** I can explain what this code does, why it does it this way, and what could go wrong. If I can't, I'm not ready to merge it.
3. **I own this.** If this code causes a problem, I take responsibility. "The AI wrote it" and "the AI reviewed it" are not defenses.

This principle applies regardless of who (or what) wrote the code. A human-written bug and an AI-written bug are both the merging developer's responsibility. The workflow automation below is designed to make this sign-off meaningful — not a checkbox to rush past, but a structured process that ensures the developer actually engaged.

Circumventing this process — skipping reviews, rubber-stamping attestations, ignoring findings — isn't cutting corners. It's a failure of professional accountability.

## Three Layers of Enforcement

```text
┌─────────────────────────────────────────────────────────┐
│  Layer 3: Human Review                                  │
│  Judgment calls, architecture decisions, threat modeling │
│  → PR template checklists (3–5 items max)               │
├─────────────────────────────────────────────────────────┤
│  Layer 2: AI-Assisted                                   │
│  Pattern matching, standard compliance, edge cases       │
│  → Claude Code /review commands, CLAUDE.md context      │
├─────────────────────────────────────────────────────────┤
│  Layer 1: Automated                                     │
│  Deterministic checks, formatting, known vulnerabilities │
│  → CI pipeline, pre-commit hooks, linters               │
└─────────────────────────────────────────────────────────┘
```

Each layer catches what the one below it can't. Together, they cover the full surface area of your coding standards without overloading any single step.

## Layer 1: Automated Checks

These run without human involvement. If they fail, the code can't merge.

### Pre-Commit Hooks

Run on every commit, before the code even reaches CI:

```yaml
# .pre-commit-config.yaml
repos:
  # Secret scanning — catches credentials before they reach git history
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  # Python linting and formatting
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.5.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  # Rust formatting and linting
  - repo: local
    hooks:
      - id: cargo-fmt
        name: cargo fmt
        entry: cargo fmt --check
        language: system
        types: [rust]
        pass_filenames: false
      - id: cargo-clippy
        name: cargo clippy
        entry: cargo clippy -- -D warnings
        language: system
        types: [rust]
        pass_filenames: false

  # TypeScript/React formatting and linting
  - repo: local
    hooks:
      - id: eslint
        name: eslint
        entry: npx eslint --fix
        language: system
        types_or: [ts, tsx]
      - id: prettier
        name: prettier
        entry: npx prettier --write
        language: system
        types_or: [ts, tsx, css, json]
```

**What this catches:**

- Hardcoded secrets and API keys
- Style violations, unused imports, unsafe patterns
- Security-related linter rules (SQL injection patterns, unsafe functions)

**Developer experience:** Runs in seconds. Auto-fixes what it can. Developer sees failures immediately, not 10 minutes later in CI.

### CI Pipeline Checks

Run on every push and PR. These are the merge gates:

```yaml
# .github/workflows/ci.yml
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Dependency vulnerability scanning
      - name: Python dependency audit
        run: pip audit

      - name: Rust dependency audit
        run: cargo audit

      - name: Node dependency audit
        run: npm audit --audit-level=moderate

      # Secret scanning (catches anything pre-commit missed)
      - name: Scan for secrets
        uses: gitleaks/gitleaks-action@v2

  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      # ... standard lint + test steps from ci-cd.md
```

**What this catches:**

- Known vulnerabilities in dependencies
- Secrets that bypassed pre-commit (force pushes, new contributors)
- Type errors, test failures, formatting drift

**Developer experience:** Runs in parallel with other CI jobs. Results visible on the PR before review.

### What to Automate

| Check | Tool | When |
| ----- | ---- | ---- |
| Secret detection | gitleaks | Pre-commit + CI |
| Code formatting | ruff, cargo fmt, prettier | Pre-commit |
| Lint rules (including security) | ruff, clippy, eslint | Pre-commit + CI |
| Type checking | mypy, tsc --noEmit | CI |
| Dependency vulnerabilities | pip audit, cargo audit, npm audit | CI |
| Test suite | pytest, cargo test, vitest | CI |

**Rule of thumb:** If a check has a deterministic yes/no answer, automate it. If it requires understanding intent, push it to Layer 2 or 3.

## Layer 2: AI-Assisted Review

AI excels at pattern matching across your codebase — things that are too nuanced for a linter but too tedious for a human to check consistently.

### CLAUDE.md as Living Standards

Your project's `CLAUDE.md` file is the bridge between the handbook and your AI assistant. It tells Claude Code what your standards are so every review is consistent:

```markdown
# CLAUDE.md

## Security Standards
- All request inputs must have Pydantic Field constraints (min_length, max_length, ge, le)
- All write endpoints require get_current_user dependency
- Error responses must not include exception details — use generic messages
- GitHub tokens must be encrypted via token_encryption service before DB storage
- Path parameters must have regex pattern validation

## Coding Standards
- Google-style docstrings on all public functions
- Unit tests for every new function (happy path + edge case + error case)
- No magic numbers — named constants with comments
```

When a developer runs `/review`, Claude checks the diff against these standards. The standards live in one place, not scattered across wiki pages.

### Custom Review Commands

Create reusable review commands that codify your checklists:

```markdown
# .claude/commands/security-review.md

Review the current git diff for security issues. Check each item:

1. **Input validation**: Every new request field has appropriate bounds
   (max_length for strings, ge/le for numbers, max_length for lists).
   Flag any unbounded user input.

2. **Authentication**: Every POST/PATCH/DELETE endpoint uses
   Depends(get_current_user). Flag any write endpoint without auth.

3. **Authorization**: If the endpoint accesses user-specific data,
   verify it checks that the requesting user owns the resource.
   Flag any missing ownership checks.

4. **Error handling**: No exception messages passed to HTTPException
   detail. Error responses must be generic. Flag any f-string
   interpolation in error details.

5. **Secrets**: No tokens, keys, or passwords in code, comments,
   or log messages. Flag any logging of sensitive fields.

6. **Data protection**: If new fields store sensitive data, verify
   encryption at rest. Flag plaintext storage of tokens or PII.

Be specific. For each issue found, show the file and line,
explain the risk, and suggest a fix.
```

```markdown
# .claude/commands/review.md

Review the current git diff against our coding standards:

1. All new public functions have doc comments (Google-style for Python,
   /// for Rust, TSDoc for TypeScript)
2. All new functions have corresponding unit tests
3. No TODO without explanation
4. No magic numbers — named constants with comments
5. Consistent naming conventions per the coding style guide
6. No unnecessary complexity — could any of this be simpler?

For each issue, cite the file and line. Suggest a fix.
```

**Developer workflow:**

```bash
# After writing code, before opening a PR
/security-review    # Catches security issues
/review             # Catches style and quality issues
```

Two commands replace a 30-item checklist. The AI reads every line of the diff against your full standards document. A human reviewer doing the same check would take 20 minutes and miss things.

### Developer Accountability: The Review Attestation

AI assistance doesn't transfer accountability. When a developer runs `/security-review`, they're not delegating responsibility to the AI — they're using a tool to help them review, and then attesting: *I looked at this, I understood the findings, and I'm accountable for what ships.*

This is critical when AI is writing code. If Claude Code wrote the implementation and Claude Code reviewed it, but the developer never engaged with either output, nobody is actually accountable. Circumventing this process — skipping the review, rubber-stamping the attestation, ignoring findings — isn't laziness. It's a failure of professional responsibility.

The enforcement model has two parts that work together:

#### Part 1: Local attestation (developer accountability)

When `/security-review` completes, it writes a marker file recording the commit SHA that was reviewed:

```bash
# .claude/commands/security-review.md (append to end of the review prompt)

After completing the review, write the current HEAD commit SHA
to .claude/last-review-sha so CI can verify this review was performed.
```

This marker is the developer's signature. It says: "I ran this review at this commit and I own the result."

#### Part 2: CI verification + independent review (safety net)

CI does two things: verifies the developer engaged, and runs its own independent check.

```yaml
# .github/workflows/ai-review.yml
name: AI Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  verify-developer-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Verify developer ran security review
        run: |
          if [ ! -f .claude/last-review-sha ]; then
            echo "::error::Security review not performed. Run /security-review before pushing."
            exit 1
          fi

          REVIEW_SHA=$(cat .claude/last-review-sha)
          HEAD_SHA=$(git rev-parse HEAD)

          if [ "$REVIEW_SHA" != "$HEAD_SHA" ]; then
            echo "::error::Security review is stale (reviewed $REVIEW_SHA, HEAD is $HEAD_SHA). Re-run /security-review."
            exit 1
          fi

  independent-ai-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # CI runs its own review as a safety net
      # This catches issues the developer missed or that were
      # introduced after the local review
      - name: AI Security Review
        run: |
          git diff origin/main...HEAD > /tmp/diff.txt
          # Feed diff to your AI review tool of choice
```

**Why both?** Because they serve different purposes:

- The **attestation check** confirms the developer personally engaged with the review. It's not about catching bugs — it's about maintaining accountability. You can't claim "the AI approved it" if you never ran the review.
- The **independent CI review** is a safety net that catches things the developer's review might have missed, or issues introduced by commits after the local review.

Neither alone is sufficient. Without the attestation, developers can let CI's AI review become their excuse for not reviewing their own code. Without the CI review, developers who rush through the local review have no backstop.

### What AI Reviews Catch

| Pattern | Why Linters Miss It |
| ------- | ------------------ |
| Missing input validation on a new field | Linters don't know your validation policy |
| Endpoint without auth that should have it | No static rule can determine intent |
| Error message leaking internal details | Requires understanding what's "internal" |
| Missing ownership check (user A accessing user B's data) | Requires understanding the data model |
| Inconsistent patterns vs. rest of codebase | Requires cross-file context |
| Missing tests for edge cases | Requires understanding the function's purpose |

## Layer 3: Human Review

Humans review what machines and AI can't — design decisions, threat models, and judgment calls.

### PR Template

Keep the human checklist short. Everything automatable should already be handled by Layers 1 and 2:

```markdown
## PR Checklist

### Accountability
- [ ] I ran `/security-review` and `/review` on the final commit
- [ ] I read the findings and addressed or explicitly dismissed each one
- [ ] I understand the changes well enough to debug them at 2am

### Security (if applicable)
- [ ] New endpoints: auth requirements are correct for the use case
- [ ] User data: access is scoped to the authenticated user
- [ ] New dependencies: reviewed for maintainability and security posture

### Design
- [ ] Approach is reasonable for the problem (not over/under-engineered)
- [ ] Breaking changes are documented
```

**Why the accountability section?** When AI writes code and AI reviews code, the human is the accountability layer. The first three checkboxes aren't about catching bugs — CI handles that. They're about the developer affirming: *I own this change.* If a developer can't check those boxes honestly, the PR isn't ready.

### When Human Review Matters Most

- **New authentication or authorization patterns** — Are the security boundaries correct?
- **New external integrations** — What data flows where? What's the trust boundary?
- **Schema or API changes** — Are they backward-compatible? Do they leak internal structure?
- **New dependencies** — Is this library maintained? What's the attack surface?

These are judgment calls that require understanding the business context, not pattern matching.

## Wiring It Together

Here's what happens when a developer makes a change, from keystroke to merge:

```text
Developer writes code (with or without AI assistance)
       │
       ▼
Pre-commit hooks run (Layer 1 — automated)
  ├── gitleaks: no secrets ✓
  ├── ruff/eslint: code quality ✓
  └── prettier/fmt: formatting ✓
       │
       ▼
Developer runs /security-review + /review (Layer 2 — developer accountability)
  ├── AI checks diff against CLAUDE.md standards
  ├── Developer reads findings and fixes issues
  ├── Developer re-runs until clean
  └── Review marker written to .claude/last-review-sha
       │
       ▼
Developer pushes branch, opens PR
  └── Fills in accountability checklist: "I reviewed this. I own it."
       │
       ▼
CI runs (Layer 1 + Layer 2 — automated verification)
  ├── Tests, linting, type checking, dependency audit ✓
  ├── Verify review marker matches HEAD SHA ✓
  └── Independent AI security review (safety net) ✓
       │
       ▼
Human reviews PR (Layer 3 — judgment calls)
  ├── Checks design and architecture
  ├── Confirms accountability checklist is checked
  └── Approves or requests changes
       │
       ▼
Merge to main → auto-deploy
```

The developer's manual effort is: write code, run two review commands, read the output, and check the accountability boxes. Everything else is automated. The total added time is 2-5 minutes per PR — the cost of owning your work.

## Claude Code Hooks

Claude Code supports hooks — shell commands that run automatically in response to events. Use these to enforce standards without relying on developer memory:

```json
// .claude/settings.json
{
  "hooks": {
    "postToolUse": [
      {
        "tool": "Write",
        "command": "echo 'Remember to run /security-review before committing'"
      }
    ]
  }
}
```

For more advanced automation, hooks can run linters after every file edit, ensuring the developer sees issues immediately rather than at commit time.

## Tracking Compliance

### For Solo Developers

Don't overthink this. If your CI is green and you're running AI reviews, you're ahead of most teams. Track by habit:

- Pre-commit hooks installed and working
- AI review commands exist and you use them
- CI pipeline includes security checks

### For Teams

When you have multiple developers, add visibility:

- **PR merge requirements**: CI must pass, at least one review, AI review checkbox checked
- **Monthly audit**: Pick 5 random merged PRs. Did they follow the process? If not, why? Fix the process, not the people.
- **Dependency update cadence**: Dependabot PRs are reviewed and merged within a week

## Common Anti-Patterns

| Anti-Pattern | Why It Fails | Do This Instead |
| ------------ | ------------ | --------------- |
| 50-item manual checklist | No one reads it after week 1 | Automate 80%, AI checks 15%, humans check 5% |
| "Review your own code" as a policy | Developers are blind to their own assumptions | AI review catches what self-review misses |
| Blocking CI with slow security scans | Developers push `--no-verify` | Run expensive scans in parallel, not blocking |
| Security review only on "security PRs" | Every PR can introduce vulnerabilities | Security checks on every PR, scoped to the diff |
| Mandatory training instead of tooling | Training fades, tools don't | Training builds understanding, tools enforce it |

## Scaling Checklist

- [ ] Pre-commit hooks installed (gitleaks, linters, formatters) (day one)
- [ ] CI pipeline includes dependency audit and secret scanning (day one)
- [ ] CLAUDE.md documents project security and coding standards (day one)
- [ ] Custom `/review` and `/security-review` commands created (week one)
- [ ] PR template with focused checklist (before first hire)
- [ ] Branch protection requiring CI pass + review (before first hire)
- [ ] Automated AI review on every PR via CI (when team > 2)
- [ ] Monthly compliance spot-check (when team > 5)
- [ ] Formal security review process for sensitive changes (when handling user data)
