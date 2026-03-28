# Testing

Testing strategy for a codebase that starts with one developer and grows. The goal is confidence that your code works — not coverage metrics for their own sake.

## Testing Philosophy

1. **Test behavior, not implementation.** Tests should verify what your code does, not how it does it internally. This lets you refactor freely.
2. **Fast tests run often.** If your test suite takes more than 30 seconds, you'll stop running it. Keep unit tests fast.
3. **Every bug gets a test.** When you fix a bug, write a test that would have caught it. This prevents regressions and builds coverage where it matters most.
4. **AI assistants write the first draft.** Let Claude Code generate test scaffolding, then review and adjust. AI is good at generating edge cases you might miss.

## Test Pyramid

```
        ╱  E2E  ╲          Few, slow, high confidence
       ╱─────────╲
      ╱Integration ╲       Some, moderate speed
     ╱───────────────╲
    ╱   Unit Tests    ╲    Many, fast, focused
   ╱───────────────────╲
```

- **Unit tests**: Test a single function or module in isolation. Fast, many of these.
- **Integration tests**: Test that components work together (API + database, service + service).
- **End-to-end tests**: Test full user flows. Slow and brittle — use sparingly for critical paths.

## Language-Specific Setup

### Python (pytest)

Directory structure mirrors source:
```
project/
├── src/
│   └── auth/
│       ├── __init__.py
│       └── tokens.py
└── tests/
    └── auth/
        ├── __init__.py
        └── test_tokens.py
```

Run all tests:
```bash
pytest
```

Run with coverage:
```bash
pytest --cov=src --cov-report=term-missing
```

Key conventions:
- Test files: `test_<module>.py`
- Test functions: `test_<behavior_being_tested>`
- Use fixtures for shared setup, not setUp/tearDown classes
- Use `pytest.raises` for expected exceptions
- Use `pytest.mark.parametrize` for testing multiple inputs

Example:
```python
import pytest
from src.auth.tokens import create_token, validate_token


def test_create_token_returns_valid_jwt():
    """Verify that create_token produces a token that passes validation."""
    token = create_token(user_id="user-123", ttl_seconds=3600)
    claims = validate_token(token)
    assert claims["sub"] == "user-123"


def test_validate_token_rejects_expired_token(frozen_time):
    """Verify that expired tokens are rejected, not silently accepted."""
    token = create_token(user_id="user-123", ttl_seconds=1)
    frozen_time.tick(seconds=2)
    with pytest.raises(TokenExpiredError):
        validate_token(token)


@pytest.mark.parametrize("bad_input", ["", None, "not-a-jwt", "eyJ.eyJ.bad"])
def test_validate_token_rejects_malformed_input(bad_input):
    """Verify that garbage input raises ValidationError, not an unhandled exception."""
    with pytest.raises(ValidationError):
        validate_token(bad_input)
```

### Rust (cargo test)

Unit tests live inline with the source:
```rust
// src/auth/tokens.rs

/// Creates a signed JWT for the given user.
///
/// # Examples
///
/// ```
/// let token = create_token("user-123", Duration::from_secs(3600));
/// assert!(validate_token(&token).is_ok());
/// ```
pub fn create_token(user_id: &str, ttl: Duration) -> String {
    // ...
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn create_token_produces_valid_jwt() {
        let token = create_token("user-123", Duration::from_secs(3600));
        let claims = validate_token(&token).expect("token should be valid");
        assert_eq!(claims.sub, "user-123");
    }

    #[test]
    fn validate_token_rejects_expired() {
        let token = create_token("user-123", Duration::from_secs(0));
        std::thread::sleep(Duration::from_millis(10));
        assert!(matches!(
            validate_token(&token),
            Err(TokenError::Expired)
        ));
    }

    #[test]
    fn validate_token_rejects_malformed_input() {
        for bad in &["", "not-a-jwt", "eyJ.eyJ.bad"] {
            assert!(matches!(
                validate_token(bad),
                Err(TokenError::Malformed)
            ));
        }
    }
}
```

Integration tests go in `tests/`:
```
project/
├── src/
│   └── lib.rs
└── tests/
    └── api_integration.rs
```

Run all tests:
```bash
cargo test
```

Run with output visible:
```bash
cargo test -- --nocapture
```

### TypeScript / React (Vitest + React Testing Library)

Directory structure co-locates tests with components:
```
src/
├── components/
│   └── NotificationBanner/
│       ├── NotificationBanner.tsx
│       └── NotificationBanner.test.tsx
├── hooks/
│   ├── useNotifications.ts
│   └── useNotifications.test.ts
└── services/
    ├── pushSubscription.ts
    └── pushSubscription.test.ts
```

Run all tests:
```bash
npx vitest run
```

Run in watch mode during development:
```bash
npx vitest
```

Run with coverage:
```bash
npx vitest run --coverage
```

Key conventions:
- Test files: `<module>.test.ts` or `<Component>.test.tsx`, co-located with source
- Use `describe` blocks to group related tests
- Use `it` or `test` with behavior-describing names
- Use React Testing Library's `screen` queries — test what the user sees, not component internals
- Mock API calls with `msw` (Mock Service Worker), not by mocking fetch directly

Example — testing a utility:
```typescript
import { describe, it, expect } from "vitest";
import { formatNotificationTime } from "./formatNotificationTime";

describe("formatNotificationTime", () => {
  it("shows 'just now' for timestamps under 60 seconds ago", () => {
    const now = Date.now();
    const fiftySecondsAgo = now - 50_000;
    expect(formatNotificationTime(fiftySecondsAgo)).toBe("just now");
  });

  it("shows relative minutes for timestamps under one hour ago", () => {
    const now = Date.now();
    const tenMinutesAgo = now - 600_000;
    expect(formatNotificationTime(tenMinutesAgo)).toBe("10m ago");
  });

  it("shows the date for timestamps older than 24 hours", () => {
    // Use a fixed timestamp to avoid flaky date formatting
    const twoDaysAgo = new Date("2026-03-25T12:00:00Z").getTime();
    expect(formatNotificationTime(twoDaysAgo)).toContain("Mar 25");
  });
});
```

Example — testing a React component:
```typescript
import { describe, it, expect, vi } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { NotificationBanner } from "./NotificationBanner";

describe("NotificationBanner", () => {
  it("renders the notification message", () => {
    render(<NotificationBanner message="Deploy complete" />);
    expect(screen.getByText("Deploy complete")).toBeInTheDocument();
  });

  it("calls onDismiss when the close button is clicked", async () => {
    const onDismiss = vi.fn();
    render(
      <NotificationBanner message="Deploy complete" onDismiss={onDismiss} />,
    );

    await userEvent.click(screen.getByRole("button", { name: /dismiss/i }));
    expect(onDismiss).toHaveBeenCalledOnce();
  });

  it("does not render a close button when onDismiss is not provided", () => {
    render(<NotificationBanner message="Deploy complete" />);
    expect(screen.queryByRole("button", { name: /dismiss/i })).not.toBeInTheDocument();
  });

  it("applies error styling for error severity", () => {
    render(<NotificationBanner message="Build failed" severity="error" />);
    // Test visible behavior, not CSS class names
    expect(screen.getByRole("alert")).toBeInTheDocument();
  });
});
```

Example — testing a custom hook:
```typescript
import { describe, it, expect } from "vitest";
import { renderHook, act } from "@testing-library/react";
import { useNotificationPermission } from "./useNotificationPermission";

describe("useNotificationPermission", () => {
  it("returns the current permission state", () => {
    const { result } = renderHook(() => useNotificationPermission());
    // Default in test environment is "default" (not yet asked)
    expect(result.current.permission).toBe("default");
  });

  it("updates permission after requesting", async () => {
    const { result } = renderHook(() => useNotificationPermission());

    await act(async () => {
      await result.current.requestPermission();
    });

    expect(["granted", "denied"]).toContain(result.current.permission);
  });
});
```

Vitest configuration:
```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    setupFiles: "./src/test/setup.ts",
    css: true,
  },
});
```

```typescript
// src/test/setup.ts
import "@testing-library/jest-dom/vitest";
```

## What to Test

### Always Test

- **Public API surface**: Every public function should have at least one test.
- **Happy path**: Does it work with normal, expected input?
- **Edge cases**: Empty collections, zero values, maximum sizes, boundary conditions.
- **Error cases**: Invalid input, missing resources, network failures.
- **Security-relevant logic**: Authentication, authorization, input validation, crypto operations.

### Don't Bother Testing

- **Implementation details** that might change during refactoring.
- **UI layout** (unless it's a critical user flow — use E2E for that).
- **React component snapshots** — they break on every visual change and test nothing meaningful. Test behavior instead.

## Test Quality Checklist

A good test:

- [ ] Has a clear name that describes the expected behavior
- [ ] Tests one thing (one assertion per logical concept)
- [ ] Fails for the right reason when the code breaks
- [ ] Doesn't depend on test execution order
- [ ] Doesn't depend on external services (use fakes/mocks for unit tests)
- [ ] Runs in under 1 second (unit) or under 30 seconds (integration)

## When to Write Tests

- **Before fixing a bug**: Write a failing test that reproduces the bug, then fix it.
- **During feature development**: Write tests as you implement, not after. Tests written after the fact tend to just verify what the code already does, not what it should do.
- **When refactoring**: Ensure tests exist for the behavior you're about to change *before* you change it.

## Using AI to Write Tests

AI assistants are excellent at generating test scaffolding:

1. Write your function first.
2. Ask the AI to generate tests — it's good at thinking of edge cases.
3. **Review every generated test.** AI sometimes writes tests that pass for the wrong reason or test implementation details rather than behavior.
4. Run the tests and verify they actually fail when you break the code.

## Scaling Checklist

- [ ] Unit tests for all public functions (day one)
- [ ] CI runs tests on every PR (day one)
- [ ] Integration tests for critical paths (when you have external dependencies)
- [ ] Coverage reporting (when you want visibility, not as a gate)
- [ ] E2E tests for critical user flows with Playwright (when you have a UI)
- [ ] MSW (Mock Service Worker) for frontend API mocking (when frontend calls APIs)
- [ ] Test data factories/fixtures (when test setup becomes repetitive)
