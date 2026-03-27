# Coding Style

Standards for writing clear, consistent, maintainable code. These aren't aesthetic preferences — they reduce bugs, speed up reviews, and make onboarding faster. Automate enforcement with linters and formatters so humans can focus on logic, not style.

## Philosophy

1. **Code is read far more than it is written.** Optimize for the reader, not the writer.
2. **Consistency beats perfection.** A mediocre convention followed everywhere is better than a perfect one followed sometimes.
3. **Automate what you can.** If a machine can enforce it, don't waste review cycles on it.
4. **Document the why, not the what.** The code shows what it does. Comments explain why it does it that way.

## Documentation

Every public interface must be documented. This is non-negotiable — undocumented code is a liability that compounds over time.

### Python: Google-Style Docstrings

```python
def rotate_refresh_token(
    user_id: str,
    current_token: str,
    ttl_seconds: int = 86400,
) -> tuple[str, datetime]:
    """Issue a new refresh token and invalidate the previous one.

    Rotation limits the damage window if a refresh token is stolen — the
    attacker's copy becomes invalid the next time the legitimate user
    refreshes.

    Args:
        user_id: The unique identifier of the authenticated user.
        current_token: The refresh token being exchanged. Must be valid
            and not yet revoked.
        ttl_seconds: Lifetime of the new token in seconds. Defaults to
            86400 (24 hours).

    Returns:
        A tuple of (new_token, expiry_datetime).

    Raises:
        TokenRevokedError: If the current token has already been used
            or explicitly revoked.
        UserNotFoundError: If the user_id does not match an active account.

    Example:
        >>> new_token, expires_at = rotate_refresh_token("user-123", old_token)
        >>> assert expires_at > datetime.now(UTC)
    """
```

**Required sections:**
- One-line summary (imperative mood: "Issue a new...", not "Issues a new...")
- Longer description if the behavior isn't obvious from the summary
- `Args:` — every parameter, with type context if not obvious from annotations
- `Returns:` — what the caller gets back and what it means
- `Raises:` — every exception the caller should handle
- `Example:` — for public API functions, show basic usage

### Rust: `///` Doc Comments

```rust
/// Rotate a refresh token, invalidating the previous one.
///
/// Rotation limits the damage window if a refresh token is stolen — the
/// attacker's copy becomes invalid the next time the legitimate user
/// refreshes.
///
/// # Arguments
///
/// * `user_id` - The unique identifier of the authenticated user.
/// * `current_token` - The refresh token being exchanged. Must be valid
///   and not yet revoked.
/// * `ttl` - Lifetime of the new token. Defaults are not supported here;
///   callers should pass `Duration::from_secs(86400)` for the standard window.
///
/// # Returns
///
/// A tuple of `(new_token, expiry)` on success.
///
/// # Errors
///
/// Returns [`TokenError::Revoked`] if the token has already been consumed.
/// Returns [`TokenError::UserNotFound`] if the user ID is invalid.
///
/// # Examples
///
/// ```
/// use std::time::Duration;
///
/// let (new_token, expires_at) = rotate_refresh_token(
///     "user-123",
///     &old_token,
///     Duration::from_secs(86400),
/// )?;
/// assert!(expires_at > Utc::now());
/// ```
pub fn rotate_refresh_token(
    user_id: &str,
    current_token: &str,
    ttl: Duration,
) -> Result<(String, DateTime<Utc>), TokenError> {
    // ...
}
```

**Required sections:**
- One-line summary
- Longer description when needed
- `# Arguments` — every parameter
- `# Returns` — what comes back
- `# Errors` — every error variant and when it occurs
- `# Examples` — runnable doc-test for public API functions

### TypeScript / React: TSDoc + JSDoc

```typescript
/**
 * Subscribe the current device to push notifications.
 *
 * Requests notification permission from the browser, registers a service worker,
 * and sends the resulting subscription to the backend. Fails gracefully on
 * browsers that don't support the Push API (returns null instead of throwing).
 *
 * @param apiBase - The base URL of the notification service (e.g., "https://api.example.com").
 * @param vapidPublicKey - The VAPID public key for push subscription authentication.
 * @returns The PushSubscription if successful, or null if the browser doesn't
 *   support push or the user denied permission.
 * @throws {NetworkError} If the backend registration call fails.
 *
 * @example
 * ```ts
 * const sub = await subscribeToPush("https://api.example.com", VAPID_KEY);
 * if (sub) {
 *   console.log("Subscribed:", sub.endpoint);
 * }
 * ```
 */
async function subscribeToPush(
  apiBase: string,
  vapidPublicKey: string,
): Promise<PushSubscription | null> {
  // ...
}
```

**Required sections:**
- One-line summary
- Longer description when the behavior isn't obvious
- `@param` — every parameter, with description
- `@returns` — what the caller gets back
- `@throws` — every error the caller should handle
- `@example` — for public API functions, show basic usage

**React component documentation:**

```typescript
/** Props for the {@link NotificationBanner} component. */
interface NotificationBannerProps {
  /** The notification message to display. */
  message: string;
  /** Visual severity level. Defaults to "info". */
  severity?: "info" | "warning" | "error";
  /** Called when the user dismisses the banner. If omitted, the banner is not dismissible. */
  onDismiss?: () => void;
}

/**
 * A top-of-page banner for displaying system notifications.
 *
 * Renders a full-width alert bar with an optional dismiss button.
 * Auto-dismisses after 10 seconds for "info" severity; "warning" and
 * "error" banners persist until explicitly dismissed.
 *
 * @example
 * ```tsx
 * <NotificationBanner
 *   message="Deploy complete"
 *   severity="info"
 *   onDismiss={() => setVisible(false)}
 * />
 * ```
 */
function NotificationBanner({ message, severity = "info", onDismiss }: NotificationBannerProps) {
  // ...
}
```

### Module and File-Level Documentation

Every module should have a top-level doc comment explaining what it contains and why it exists:

**Python:**
```python
"""Token management for the authentication subsystem.

This module handles JWT creation, validation, and refresh token rotation.
All tokens are signed with RS256 using keys from the configured key store.
Refresh tokens are single-use — each rotation invalidates the previous token
to limit the window of a stolen credential.
"""
```

**Rust:**
```rust
//! Token management for the authentication subsystem.
//!
//! This module handles JWT creation, validation, and refresh token rotation.
//! All tokens are signed with RS256 using keys from the configured key store.
//! Refresh tokens are single-use — each rotation invalidates the previous token
//! to limit the window of a stolen credential.
```

**TypeScript:**
```typescript
/**
 * Push notification subscription and delivery management.
 *
 * Handles browser Push API integration, VAPID key management, and
 * subscription lifecycle (subscribe, renew, unsubscribe). All
 * subscriptions are persisted to the backend so the server can
 * send notifications when processes need user input.
 *
 * @module notifications/push
 */
```

### Inline Comments

Inline comments explain **why**, not **what**. If you need a comment to explain what the code does, the code should be rewritten to be clearer.

```python
# Good — explains a non-obvious decision
# Cap at 1000 to avoid OOM on the worker — the upstream API
# occasionally returns unbounded result sets despite pagination.
MAX_BATCH_SIZE = 1000

# Bad — restates the code
# Set max batch size to 1000
MAX_BATCH_SIZE = 1000

# Good — explains why this seemingly wrong approach is correct
# We check expiry BEFORE signature verification because signature
# verification is expensive (~2ms) and most rejections are expired
# tokens from normal usage, not attacks.
if token.is_expired():
    raise TokenExpiredError()
verify_signature(token)

# Bad — obvious from the code
# Check if token is expired
if token.is_expired():
    raise TokenExpiredError()
```

### TODO and FIXME Comments

Never leave a bare TODO. Always include what needs to be done and why:

```python
# Good
# TODO: Replace with batch insert once SQLAlchemy 2.1 ships with
# native COPY support — current row-by-row insert is O(n) roundtrips.

# Bad
# TODO: fix this later
```

## Naming Conventions

### General Principles

- Names should be intention-revealing. A reader should understand what something is without reading its implementation.
- Length should be proportional to scope. Loop variables can be short (`i`, `item`). Module-level constants should be descriptive.
- Don't abbreviate unless the abbreviation is universally understood in your domain (`url`, `id`, `db` — fine; `usr_mgr`, `val_tkn` — not fine).

### Python

| Element | Convention | Example |
|---------|-----------|---------|
| Modules | `snake_case` | `token_manager.py` |
| Classes | `PascalCase` | `TokenManager` |
| Functions | `snake_case` | `rotate_refresh_token` |
| Variables | `snake_case` | `current_user` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Private | Leading underscore | `_validate_claims` |
| Type variables | Short `PascalCase` | `T`, `KeyType` |

```python
# Good — names tell you what things are
REFRESH_TOKEN_TTL_SECONDS = 86400

def revoke_all_sessions(user_id: str, reason: str) -> int:
    """Invalidate all active sessions for a user.

    Args:
        user_id: The user whose sessions to revoke.
        reason: Audit log reason (e.g., "password_change", "admin_action").

    Returns:
        The number of sessions revoked.
    """
    active_sessions = _get_active_sessions(user_id)
    for session in active_sessions:
        session.revoke(reason=reason)
    return len(active_sessions)

# Bad — cryptic names, no documentation
TTL = 86400

def revoke(uid, r):
    ss = _get(uid)
    for s in ss:
        s.revoke(reason=r)
    return len(ss)
```

### Rust

| Element | Convention | Example |
|---------|-----------|---------|
| Crates | `snake_case` | `token_manager` |
| Modules | `snake_case` | `refresh_tokens` |
| Types / Traits | `PascalCase` | `TokenManager`, `Authenticatable` |
| Functions | `snake_case` | `rotate_refresh_token` |
| Variables | `snake_case` | `current_user` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Enum variants | `PascalCase` | `TokenError::Expired` |
| Lifetimes | Short lowercase | `'a`, `'ctx` |

```rust
/// Maximum number of retry attempts before aborting the request.
///
/// Set to 3 based on p99 latency measurements — beyond this point,
/// the upstream service is likely degraded and retrying adds load.
const MAX_RETRY_COUNT: u32 = 3;

/// Errors that can occur during token operations.
#[derive(Debug, thiserror::Error)]
pub enum TokenError {
    /// The token has passed its expiration time.
    #[error("token expired at {expiry}")]
    Expired {
        /// When the token became invalid.
        expiry: DateTime<Utc>,
    },

    /// The token has been revoked (already consumed or explicitly invalidated).
    #[error("token was revoked: {reason}")]
    Revoked {
        /// Why the token was revoked.
        reason: String,
    },

    /// The token bytes could not be parsed as a valid JWT.
    #[error("malformed token")]
    Malformed,
}
```

### TypeScript / React

| Element | Convention | Example |
|---------|-----------|---------|
| Files (components) | `PascalCase.tsx` | `NotificationBanner.tsx` |
| Files (utilities) | `camelCase.ts` | `pushSubscription.ts` |
| Files (hooks) | `camelCase.ts` with `use` prefix | `useNotifications.ts` |
| Components | `PascalCase` | `NotificationBanner` |
| Functions / variables | `camelCase` | `subscribeToPush` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RECONNECT_ATTEMPTS` |
| Types / Interfaces | `PascalCase` | `NotificationPayload` |
| Enum members | `PascalCase` | `Severity.Warning` |
| CSS modules | `camelCase` | `styles.bannerContainer` |

```typescript
// Good — clear intent, documented
/** Maximum number of WebSocket reconnection attempts before showing an offline banner. */
const MAX_RECONNECT_ATTEMPTS = 5;

/** Severity levels for system notifications. */
enum Severity {
  Info = "info",
  Warning = "warning",
  Error = "error",
}

/** Props for the {@link NotificationBanner} component. */
interface NotificationBannerProps {
  /** The message to display to the user. */
  message: string;
  /** Visual severity level. Defaults to {@link Severity.Info}. */
  severity?: Severity;
}

// Bad — vague names, no types, no docs
const MAX = 5;
const NotifBanner = ({ msg, sev }: any) => { /* ... */ };
```

**React-specific conventions:**
- One component per file. The file name matches the component name.
- Co-locate tests, styles, and types with the component when they're specific to it.
- Prefix custom hooks with `use` (enforced by React's rules of hooks).
- Props interfaces are named `<ComponentName>Props`.
- Avoid `default export` — use named exports for better refactoring support and import autocompletion.

```
src/
├── components/
│   └── NotificationBanner/
│       ├── NotificationBanner.tsx       # Component
│       ├── NotificationBanner.test.tsx  # Tests
│       ├── NotificationBanner.module.css # Styles
│       └── index.ts                     # Re-export
├── hooks/
│   └── useNotifications.ts
├── services/
│   └── pushSubscription.ts
└── types/
    └── notifications.ts                 # Shared types
```

### Boolean Naming

Booleans should read as true/false questions:

```python
# Good — reads naturally in conditionals
is_authenticated = True
has_permission = False
should_retry = attempt < MAX_RETRY_COUNT

# Bad — ambiguous
authenticated = True   # Is this a verb (was authenticated) or adjective (is authenticated)?
retry = True           # Retry what? Is this a flag or a command?
```

## Formatting and Linters

### Python: ruff

Ruff handles both linting and formatting. Single tool, fast.

```toml
# pyproject.toml
[tool.ruff]
target-version = "py312"
line-length = 99

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort (import sorting)
    "N",    # pep8-naming
    "D",    # pydocstyle
    "UP",   # pyupgrade
    "S",    # bandit (security)
    "B",    # bugbear
    "SIM",  # simplify
    "RUF",  # ruff-specific rules
]

[tool.ruff.lint.pydocstyle]
convention = "google"
```

Run:
```bash
# Check for issues
ruff check .

# Auto-fix what's fixable
ruff check --fix .

# Format code
ruff format .

# Check formatting without changing files
ruff format --check .
```

### Rust: cargo fmt + clippy

```toml
# rustfmt.toml
max_width = 99
use_field_init_shorthand = true
use_try_shorthand = true
```

Run:
```bash
# Format
cargo fmt

# Check formatting without changing files
cargo fmt --check

# Lint with all warnings as errors
cargo clippy -- -D warnings
```

### TypeScript / React: ESLint + Prettier

ESLint handles linting (catching bugs, enforcing patterns). Prettier handles formatting (spacing, semicolons, line length). Don't make them fight — let Prettier own formatting.

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": false,
  "trailingComma": "all",
  "printWidth": 99,
  "tabWidth": 2
}
```

```js
// eslint.config.js (flat config — ESLint 9+)
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import reactHooks from "eslint-plugin-react-hooks";
import reactRefresh from "eslint-plugin-react-refresh";
import prettier from "eslint-config-prettier";

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  {
    plugins: {
      "react-hooks": reactHooks,
      "react-refresh": reactRefresh,
    },
    rules: {
      "react-hooks/rules-of-hooks": "error",
      "react-hooks/exhaustive-deps": "warn",
      "react-refresh/only-export-components": "warn",
      // Require explicit return types on exported functions
      "@typescript-eslint/explicit-module-boundary-types": "error",
      // No any — use unknown and narrow
      "@typescript-eslint/no-explicit-any": "error",
      // Unused vars are errors, not warnings
      "@typescript-eslint/no-unused-vars": ["error", { argsIgnorePattern: "^_" }],
    },
  },
  // Prettier must be last — it disables formatting rules that conflict
  prettier,
);
```

Run:
```bash
# Lint
npx eslint .

# Auto-fix what's fixable
npx eslint --fix .

# Format
npx prettier --write .

# Check formatting without changes
npx prettier --check .

# Type-check (does not emit files)
npx tsc --noEmit
```

### Pre-Commit Hooks

Automate formatting so it's impossible to forget:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.5.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

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

## Constants and Magic Values

Never use unexplained literal values. Define named constants with comments explaining the choice:

```python
# Bad
if len(password) < 12:
    raise ValueError("too short")

time.sleep(0.05)

if retries > 3:
    raise TimeoutError()

# Good
# NIST SP 800-63B recommends a minimum of 8 characters; we use 12
# as a stronger baseline that balances security with usability.
MIN_PASSWORD_LENGTH = 12

# Rate limit backoff to avoid overwhelming the upstream API.
# 50ms was chosen to stay well under their 100 req/s limit.
RATE_LIMIT_BACKOFF_SECONDS = 0.05

# Beyond 3 retries, the upstream service is likely degraded
# and additional attempts add load without improving success rate.
MAX_RETRY_ATTEMPTS = 3
```

## Error Handling

### Python

- Use specific exception types, not bare `Exception`.
- Catch only what you can handle. Let everything else propagate.
- Never silently swallow exceptions.

```python
# Good — specific, informative, documented
class InsufficientBalanceError(Exception):
    """Raised when a transaction exceeds the available account balance.

    Attributes:
        available: The current balance.
        requested: The amount that was attempted.
    """

    def __init__(self, available: Decimal, requested: Decimal) -> None:
        self.available = available
        self.requested = requested
        super().__init__(
            f"Insufficient balance: {available} available, {requested} requested"
        )


def withdraw(account_id: str, amount: Decimal) -> Decimal:
    """Withdraw funds from an account.

    Args:
        account_id: The account to withdraw from.
        amount: The amount to withdraw. Must be positive.

    Returns:
        The new account balance after withdrawal.

    Raises:
        ValueError: If amount is not positive.
        InsufficientBalanceError: If the account balance is too low.
        AccountNotFoundError: If the account_id is invalid.
    """
    if amount <= 0:
        raise ValueError(f"Withdrawal amount must be positive, got {amount}")

    balance = get_balance(account_id)
    if balance < amount:
        raise InsufficientBalanceError(available=balance, requested=amount)

    return execute_withdrawal(account_id, amount)


# Bad — catches everything, hides bugs
def withdraw(account_id, amount):
    try:
        balance = get_balance(account_id)
        return execute_withdrawal(account_id, amount)
    except Exception:
        return None  # Caller has no idea what went wrong
```

### Rust

- Use `Result<T, E>` for recoverable errors.
- Use descriptive error enums with `thiserror`.
- Reserve `panic!` / `unwrap` for truly unrecoverable states and never in library code.

```rust
use thiserror::Error;

/// Errors that can occur during a withdrawal.
#[derive(Debug, Error)]
pub enum WithdrawError {
    /// The requested amount exceeds the available balance.
    #[error("insufficient balance: {available} available, {requested} requested")]
    InsufficientBalance {
        /// Current account balance.
        available: Decimal,
        /// Amount that was requested.
        requested: Decimal,
    },

    /// The account ID does not match any active account.
    #[error("account not found: {0}")]
    AccountNotFound(String),

    /// The withdrawal amount was zero or negative.
    #[error("amount must be positive, got {0}")]
    InvalidAmount(Decimal),
}

/// Withdraw funds from an account.
///
/// # Arguments
///
/// * `account_id` - The account to withdraw from.
/// * `amount` - The amount to withdraw. Must be positive.
///
/// # Returns
///
/// The new account balance after withdrawal.
///
/// # Errors
///
/// Returns [`WithdrawError::InvalidAmount`] if amount is not positive.
/// Returns [`WithdrawError::InsufficientBalance`] if the balance is too low.
/// Returns [`WithdrawError::AccountNotFound`] if the account ID is invalid.
///
/// # Examples
///
/// ```
/// let new_balance = withdraw("acct-123", Decimal::new(5000, 2))?;
/// assert!(new_balance >= Decimal::ZERO);
/// ```
pub fn withdraw(account_id: &str, amount: Decimal) -> Result<Decimal, WithdrawError> {
    if amount <= Decimal::ZERO {
        return Err(WithdrawError::InvalidAmount(amount));
    }

    let balance = get_balance(account_id)
        .map_err(|_| WithdrawError::AccountNotFound(account_id.to_string()))?;

    if balance < amount {
        return Err(WithdrawError::InsufficientBalance {
            available: balance,
            requested: amount,
        });
    }

    execute_withdrawal(account_id, amount)
}
```

### TypeScript / React

- Use `unknown` instead of `any`. Narrow with type guards.
- Handle async errors explicitly — don't let promises fail silently.
- Use discriminated unions for error states in components.

```typescript
// Good — explicit error states, no silent failures
/** Result type for operations that can fail with a known error. */
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

/** Errors that can occur during push subscription. */
enum PushErrorKind {
  /** The browser does not support the Push API. */
  NotSupported = "NOT_SUPPORTED",
  /** The user denied notification permission. */
  PermissionDenied = "PERMISSION_DENIED",
  /** The backend rejected the subscription registration. */
  RegistrationFailed = "REGISTRATION_FAILED",
}

/** A structured push subscription error. */
interface PushError {
  /** The category of failure. */
  kind: PushErrorKind;
  /** A human-readable explanation. */
  message: string;
}

/**
 * Subscribe the current browser to push notifications.
 *
 * @param vapidKey - The VAPID public key for authentication.
 * @returns A Result containing the subscription or a structured error.
 */
async function subscribeToPush(vapidKey: string): Promise<Result<PushSubscription, PushError>> {
  if (!("PushManager" in window)) {
    return {
      ok: false,
      error: { kind: PushErrorKind.NotSupported, message: "Push API not available" },
    };
  }

  const permission = await Notification.requestPermission();
  if (permission !== "granted") {
    return {
      ok: false,
      error: { kind: PushErrorKind.PermissionDenied, message: "User denied notifications" },
    };
  }

  // ... registration logic
}

// Bad — any types, silent catch, no structure
async function subscribe(key: any) {
  try {
    const sub = await doSubscribe(key);
    return sub;
  } catch (e) {
    console.log(e);
    return null; // Caller can't distinguish "not supported" from "network error"
  }
}
```

**React error boundaries** for component-level failures:
```typescript
// Use error boundaries for unexpected render errors.
// Use Result types for expected, handleable failures (API calls, permissions).
// Never catch errors in render — let them bubble to the boundary.
```

## Import Organization

### Python

Group imports in this order, separated by blank lines:

```python
# 1. Standard library
import os
from datetime import datetime, timezone
from decimal import Decimal

# 2. Third-party packages
import httpx
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel, Field

# 3. Local application imports
from src.auth.tokens import create_token
from src.db.connection import get_session
```

Ruff with the `I` rule set enforces this automatically.

### Rust

```rust
// 1. Standard library
use std::collections::HashMap;
use std::time::Duration;

// 2. External crates
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use thiserror::Error;

// 3. Crate-internal modules
use crate::auth::TokenManager;
use crate::db::ConnectionPool;
```

`cargo fmt` handles ordering within groups.

### TypeScript

```typescript
// 1. React and framework imports
import { useEffect, useState } from "react";

// 2. Third-party libraries
import { z } from "zod";

// 3. Internal shared modules (absolute imports via path alias)
import { apiClient } from "@/services/api";
import type { NotificationPayload } from "@/types/notifications";

// 4. Relative imports (co-located files)
import { NotificationBanner } from "./NotificationBanner";
import styles from "./Dashboard.module.css";
```

Configure path aliases so imports stay clean as the project grows:
```json
// tsconfig.json (relevant section)
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

ESLint with `eslint-plugin-import` can enforce this ordering automatically.

## Code Organization

### Function Length

If a function is longer than ~40 lines, it's probably doing too much. Extract logical steps into well-named helper functions — but only if they represent a meaningful concept, not just "lines 20–35."

### File Length

If a file exceeds ~500 lines, look for natural boundaries to split on. A file should contain one cohesive concept (a module, a service, a set of related utilities).

### Nesting Depth

If code is indented more than 3 levels deep, refactor. Use early returns to flatten logic:

```python
# Bad — deeply nested, hard to follow
def process_order(order):
    if order is not None:
        if order.status == "pending":
            if order.items:
                if validate_inventory(order.items):
                    return execute_order(order)
                else:
                    raise OutOfStockError()
            else:
                raise EmptyOrderError()
        else:
            raise InvalidStatusError()
    else:
        raise ValueError("order is None")

# Good — early returns, flat structure
def process_order(order):
    """Execute a pending order after validating inventory.

    Args:
        order: The order to process. Must be in "pending" status.

    Returns:
        The completed order confirmation.

    Raises:
        ValueError: If order is None.
        InvalidStatusError: If order is not in "pending" status.
        EmptyOrderError: If the order has no items.
        OutOfStockError: If inventory is insufficient.
    """
    if order is None:
        raise ValueError("order is None")
    if order.status != "pending":
        raise InvalidStatusError()
    if not order.items:
        raise EmptyOrderError()
    if not validate_inventory(order.items):
        raise OutOfStockError()

    return execute_order(order)
```

## Scaling Checklist

- [ ] Linter and formatter configured and passing (day one)
- [ ] Pre-commit hooks for automatic formatting (day one)
- [ ] Doc comment standards agreed on (day one)
- [ ] Linter rules enforced in CI (day one)
- [ ] Consistent naming conventions documented (this file)
- [ ] New team members pointed here during onboarding (before first hire)
