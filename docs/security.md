# Security Practices

Security defaults for startups. You don't need a security team to avoid the most common mistakes. These practices prevent the vulnerabilities that actually get exploited in the wild.

## Principles

1. **Secure by default.** The easy path should be the secure path. If developers have to remember to do something securely, they'll eventually forget.
2. **Defense in depth.** No single layer is perfect. Overlap your defenses.
3. **Least privilege.** Every user, service, and token should have the minimum permissions needed.
4. **Assume breach.** Design so that a single compromised component doesn't give access to everything.

## Secrets Management

### The Rules

- **Never commit secrets to git.** Not even "temporarily." Git history is forever.
- **Never hardcode secrets in source code.** Not even in tests or comments.
- **Never log secrets.** Audit your logging to make sure credentials don't appear in output.
- **Rotate secrets regularly** and immediately when someone with access leaves.

### Where to Store Secrets

| Environment | Storage |
|-------------|---------|
| Local development | `.env` file (gitignored) |
| CI/CD | GitHub Actions Secrets or equivalent |
| Staging/Production | Cloud secret manager (AWS Secrets Manager, GCP Secret Manager, etc.) |

### Preventing Accidental Commits

Install a pre-commit hook to scan for secrets:

```bash
# Using gitleaks
brew install gitleaks

# Add to pre-commit config
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

Or use a simple grep-based check in a git hook:

```bash
# .git/hooks/pre-commit
#!/usr/bin/env bash
# Scan staged files for common secret patterns

PATTERNS=(
    'AKIA[0-9A-Z]{16}'           # AWS Access Key
    'sk-[a-zA-Z0-9]{48}'         # OpenAI API Key
    'ghp_[a-zA-Z0-9]{36}'        # GitHub Personal Access Token
    'password\s*=\s*["\x27][^"\x27]+'  # Hardcoded passwords
)

for pattern in "${PATTERNS[@]}"; do
    if git diff --cached --diff-filter=d | grep -qP "$pattern"; then
        echo "ERROR: Possible secret detected matching pattern: $pattern"
        echo "Review your staged changes before committing."
        exit 1
    fi
done
```

## Dependency Security

### Keep Dependencies Updated

Outdated dependencies are the #1 attack vector for most applications.

```bash
# Python — audit for known vulnerabilities
pip audit

# Rust — audit for known vulnerabilities
cargo install cargo-audit
cargo audit
```

### Automate Dependency Updates

Enable GitHub Dependabot:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"

  - package-ecosystem: "cargo"
    directory: "/"
    schedule:
      interval: "weekly"
```

### Vet New Dependencies

Before adding a new dependency, check:

- Is it actively maintained? (Last commit, open issues)
- How many transitive dependencies does it pull in?
- Does it have known vulnerabilities?
- Could you write this yourself in under an hour? (If yes, consider it)

## Input Validation

### Validate at the Boundary

Every input from outside your system is untrusted: HTTP requests, file uploads, database results from shared tables, CLI arguments, environment variables.

```python
from pydantic import BaseModel, Field


class CreateUserRequest(BaseModel):
    """Request body for creating a new user.

    Args:
        username: Alphanumeric username, 3-50 characters.
        email: Valid email address.
    """
    username: str = Field(min_length=3, max_length=50, pattern=r'^[a-zA-Z0-9_]+$')
    email: str = Field(max_length=254)
```

### Common Injection Attacks to Prevent

| Attack | Prevention |
|--------|-----------|
| SQL Injection | Use parameterized queries. Never string-format SQL. |
| Command Injection | Never pass user input to `os.system()` or `subprocess.run(shell=True)`. |
| Path Traversal | Validate and canonicalize file paths. Reject `..` components. |
| XSS | Escape HTML output. Use frameworks that escape by default. |
| SSRF | Validate and allowlist URLs before making server-side requests. |

## Authentication & Authorization

### Passwords

- **Hash with bcrypt or argon2.** Never MD5 or SHA-256 for passwords.
- **Enforce minimum length** (12+ characters). Don't impose maximum length or character restrictions.
- **Support passkeys/WebAuthn** if building a web app — it's the future and it's more secure.

### API Keys and Tokens

- **Use short-lived tokens.** JWTs with 15-minute expiry + refresh tokens.
- **Scope tokens** to the minimum permissions needed.
- **Log token usage** (not the token itself) for auditing.
- **Provide revocation.** Users must be able to revoke a compromised token immediately.

### Authorization

- **Check permissions on every request.** Don't rely on the UI to hide unauthorized actions.
- **Use allowlists, not denylists.** Default to "no access" and explicitly grant permissions.
- **Test authorization.** Write tests that verify user A cannot access user B's resources.

## HTTPS Everywhere

- **All production traffic over HTTPS.** No exceptions.
- **Redirect HTTP to HTTPS** at the load balancer or reverse proxy.
- **Use HSTS headers** to prevent downgrade attacks.
- **TLS 1.2 minimum.** Disable TLS 1.0 and 1.1.

## Logging for Security

Log enough to investigate incidents without logging sensitive data:

**Do log:**
- Authentication attempts (success and failure)
- Authorization failures
- Input validation failures
- System errors

**Don't log:**
- Passwords or tokens
- Full credit card numbers
- Personal health information
- Session IDs

## Security Review Checklist

Run through this before every release:

```
□ No secrets in code or git history
□ Dependencies are up to date and audited
□ All user input is validated
□ SQL queries are parameterized
□ Authentication and authorization are tested
□ HTTPS is enforced
□ Error messages don't leak internal details
□ Logging doesn't contain sensitive data
```

## Scaling Checklist

- [ ] .gitignore covers secrets, pre-commit secret scanning (day one)
- [ ] Dependency audit in CI (day one)
- [ ] Input validation on all API endpoints (day one)
- [ ] Dependabot or equivalent for automated updates (week one)
- [ ] Cloud secret manager for production (before first production deploy)
- [ ] Security-focused code review checklist (when you have reviewers)
- [ ] Penetration testing (before handling sensitive user data)
- [ ] Bug bounty program (when you have significant user base)
