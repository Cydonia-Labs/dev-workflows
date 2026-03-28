# Secure Coding Practices

How to write code that resists attack. The [Security Practices](security.md) doc covers operational security — secrets management, dependency scanning, HTTPS enforcement. This doc covers the code itself: how to validate input, handle authentication, protect data, and avoid the vulnerability patterns that actually get exploited.

## Principles

1. **Never trust input.** Every value crossing a system boundary is a potential attack vector until validated.
2. **Fail closed.** When in doubt, deny access. Default to "no permission" and grant explicitly.
3. **Minimize blast radius.** Limit what any single token, user, or service can access so a compromise doesn't cascade.
4. **Don't roll your own crypto.** Use battle-tested libraries for encryption, hashing, and token generation. Your custom implementation has bugs — you just haven't found them yet.

---

## 1. Input Validation

Validate at every system boundary: HTTP requests, CLI arguments, file uploads, environment variables, message queues, database results from shared tables. Reject bad input early and loudly.

### Schema Validation

Use schema validation libraries to define what valid input looks like. This is your first line of defense.

**Python (Pydantic):**

```python
from pydantic import BaseModel, Field, field_validator


class CreateProjectRequest(BaseModel):
    """Request body for creating a new project.

    Args:
        name: Human-readable project name, 1-100 characters.
        description: Optional project description, max 2000 characters.
        max_members: Maximum team size. Must be between 1 and 500.
        tags: Optional list of tags. Maximum 20 tags, each max 50 characters.
    """

    name: str = Field(min_length=1, max_length=100)
    description: str = Field(default="", max_length=2000)
    max_members: int = Field(ge=1, le=500)
    tags: list[str] = Field(default_factory=list, max_length=20)

    @field_validator("tags")
    @classmethod
    def validate_tags(cls, tags: list[str]) -> list[str]:
        """Enforce per-tag length limit.

        Args:
            tags: The list of tags to validate.

        Returns:
            The validated tag list.

        Raises:
            ValueError: If any tag exceeds 50 characters.
        """
        for tag in tags:
            if len(tag) > 50:
                raise ValueError(f"Tag must be 50 characters or fewer, got {len(tag)}")
        return tags
```

**TypeScript (Zod):**

```typescript
import { z } from "zod";

/** Maximum number of tags allowed per project. */
const MAX_TAGS = 20;

/** Maximum character length for a single tag. */
const MAX_TAG_LENGTH = 50;

/**
 * Schema for project creation requests.
 *
 * Enforces length limits on all string fields and bounds on numeric fields
 * to prevent abuse and ensure data consistency.
 */
const CreateProjectSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().max(2000).default(""),
  maxMembers: z.number().int().min(1).max(500),
  tags: z
    .array(z.string().max(MAX_TAG_LENGTH))
    .max(MAX_TAGS)
    .default([]),
});

/** Inferred type from the schema — use this instead of hand-writing the interface. */
type CreateProjectRequest = z.infer<typeof CreateProjectSchema>;
```

**Rust (serde + custom validation):**

```rust
use serde::Deserialize;

/// Maximum number of tags allowed per project.
const MAX_TAGS: usize = 20;

/// Maximum character length for a single tag.
const MAX_TAG_LENGTH: usize = 50;

/// Request body for creating a new project.
///
/// All string fields have length limits and numeric fields have bounds
/// to prevent abuse at the API boundary.
#[derive(Debug, Deserialize)]
pub struct CreateProjectRequest {
    /// Human-readable project name, 1-100 characters.
    pub name: String,
    /// Optional project description, max 2000 characters.
    #[serde(default)]
    pub description: String,
    /// Maximum team size. Must be between 1 and 500.
    pub max_members: u32,
    /// Optional list of tags. Maximum 20 tags, each max 50 characters.
    #[serde(default)]
    pub tags: Vec<String>,
}

impl CreateProjectRequest {
    /// Validate all fields against business rules.
    ///
    /// Call this immediately after deserialization — serde handles
    /// structure but not semantic constraints.
    ///
    /// # Errors
    ///
    /// Returns a `ValidationError` describing the first constraint violation found.
    pub fn validate(&self) -> Result<(), ValidationError> {
        if self.name.is_empty() || self.name.len() > 100 {
            return Err(ValidationError::field("name", "must be 1-100 characters"));
        }
        if self.description.len() > 2000 {
            return Err(ValidationError::field("description", "must be 2000 characters or fewer"));
        }
        if self.max_members == 0 || self.max_members > 500 {
            return Err(ValidationError::field("max_members", "must be between 1 and 500"));
        }
        if self.tags.len() > MAX_TAGS {
            return Err(ValidationError::field("tags", "maximum 20 tags allowed"));
        }
        for tag in &self.tags {
            if tag.len() > MAX_TAG_LENGTH {
                return Err(ValidationError::field("tags", "each tag must be 50 characters or fewer"));
            }
        }
        Ok(())
    }
}
```

### Key Rules

- **Bound all numeric inputs.** An unbounded integer becomes a denial-of-service vector. If a user can set `page_size=999999999`, they will.
- **Limit string and collection sizes.** Set `max_length` on every string field. Set `max_length` on every list/array field.
- **Validate formats explicitly.** Email, URL, UUID — use purpose-built validators, not regex you wrote at 2am.
- **Reject unexpected fields.** In Pydantic, use `model_config = ConfigDict(extra="forbid")`. In Zod, use `.strict()`. Unexpected fields can indicate probing or injection attempts.

---

## 2. Authentication & Session Security

### JWT Best Practices

JWTs are easy to get wrong. Follow these rules without exception:

| Rule | Why |
|------|-----|
| Use RS256 or ES256, never HS256 with a weak secret | Asymmetric algorithms let you verify tokens without exposing the signing key |
| Set short expiry (15 minutes for access tokens) | Limits the window if a token is stolen |
| Use refresh tokens for session continuity | Refresh tokens can be revoked; JWTs cannot until they expire |
| Validate `iss`, `aud`, and `exp` claims on every request | Prevents token reuse across services |
| RSA keys must be 2048+ bits; EC keys must use P-256 or P-384 | Shorter keys are vulnerable to brute force |
| Never store sensitive data in the JWT payload | JWTs are base64-encoded, not encrypted — anyone can read them |

**Python (PyJWT):**

```python
import jwt
from datetime import datetime, timedelta, timezone

# Minimum RSA key size for JWT signing — NIST recommends 2048+.
MIN_RSA_KEY_BITS = 2048

# Access tokens expire after 15 minutes to limit the damage window
# if a token is intercepted.
ACCESS_TOKEN_TTL = timedelta(minutes=15)


def create_access_token(user_id: str, signing_key: str) -> str:
    """Create a short-lived JWT access token.

    Args:
        user_id: The authenticated user's unique identifier.
        signing_key: RSA private key in PEM format.

    Returns:
        An encoded JWT string.
    """
    now = datetime.now(timezone.utc)
    payload = {
        "sub": user_id,
        "iat": now,
        "exp": now + ACCESS_TOKEN_TTL,
        "iss": "your-service-name",
        "aud": "your-service-name",
    }
    return jwt.encode(payload, signing_key, algorithm="RS256")


def verify_access_token(token: str, public_key: str) -> dict:
    """Verify and decode a JWT access token.

    Args:
        token: The encoded JWT string.
        public_key: RSA public key in PEM format.

    Returns:
        The decoded token payload.

    Raises:
        jwt.ExpiredSignatureError: If the token has expired.
        jwt.InvalidTokenError: If the token is malformed or fails validation.
    """
    return jwt.decode(
        token,
        public_key,
        algorithms=["RS256"],  # Explicit allowlist — never accept "none" or HS256
        issuer="your-service-name",
        audience="your-service-name",
        options={"require": ["exp", "iss", "aud", "sub"]},
    )
```

### Token Storage

| Environment | Storage | Why |
|-------------|---------|-----|
| Browser | `httpOnly`, `Secure`, `SameSite=Strict` cookie | Cannot be read by JavaScript — immune to XSS token theft |
| Mobile app | OS keychain (iOS Keychain, Android Keystore) | Hardware-backed secure storage |
| Server-to-server | Environment variable or secret manager | Never in source code or config files |

Never store tokens in `localStorage` or `sessionStorage`. A single XSS vulnerability exposes every token.

### OAuth Security

- **Always use the `state` parameter.** It prevents CSRF attacks against the OAuth flow. Generate a cryptographically random value, store it in the session, and verify it on callback.
- **Use PKCE (Proof Key for Code Exchange).** Required for public clients (SPAs, mobile apps). Prevents authorization code interception.
- **Validate the redirect URI server-side.** Use exact-match comparison against a pre-registered allowlist. Never allow open redirects.

**TypeScript (OAuth state + PKCE):**

```typescript
import { randomBytes, createHash } from "crypto";

/**
 * Generate a PKCE code verifier and challenge pair.
 *
 * The verifier is sent with the token exchange; the challenge is sent with
 * the authorization request. The server hashes the verifier and compares
 * it to the challenge, proving the token requester is the same entity
 * that initiated the flow.
 *
 * @returns An object containing the verifier and its SHA-256 challenge.
 */
function generatePkce(): { verifier: string; challenge: string } {
  // 32 bytes = 43 base64url characters, meeting the 43-128 char requirement.
  const verifier = randomBytes(32).toString("base64url");
  const challenge = createHash("sha256").update(verifier).digest("base64url");
  return { verifier, challenge };
}

/**
 * Generate a cryptographically random OAuth state parameter.
 *
 * Store this in the user's session and verify it matches on the OAuth callback
 * to prevent CSRF attacks.
 *
 * @returns A 32-byte random hex string.
 */
function generateOAuthState(): string {
  return randomBytes(32).toString("hex");
}
```

### Session Invalidation

- Revoke refresh tokens on password change, email change, or explicit logout.
- Maintain a server-side token denylist for immediate revocation (Redis works well).
- Set an absolute session lifetime (e.g., 7 days) even with refresh token rotation — force re-authentication periodically.

---

## 3. Authorization

Authentication proves who the user is. Authorization decides what they can do. Getting authorization wrong is the most common source of "user A can see user B's data" vulnerabilities.

### Check on Every Request

Never rely on the client to enforce access control. The UI can hide buttons, but the API must verify permissions independently.

**Python (FastAPI):**

```python
from fastapi import Depends, HTTPException, status


async def get_project(
    project_id: str,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db),
) -> Project:
    """Fetch a project, verifying the current user has access.

    Args:
        project_id: The project to retrieve.
        current_user: The authenticated user (injected by dependency).
        db: Database session (injected by dependency).

    Returns:
        The requested project.

    Raises:
        HTTPException: 404 if the project does not exist or the user
            lacks access. Using 404 instead of 403 avoids revealing
            that the resource exists to unauthorized users.
    """
    project = db.query(Project).filter(Project.id == project_id).first()

    # Return 404 for both "not found" and "not authorized" to avoid
    # leaking information about which project IDs exist.
    if not project or current_user.id not in project.member_ids:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Project not found",
        )
    return project
```

### Use Allowlists, Not Denylists

Default to denying access. Explicitly grant the minimum permissions needed.

```python
# Bad — denylist. If you forget a role, they get access.
BLOCKED_ROLES = {"suspended", "banned"}

def can_access(user: User) -> bool:
    return user.role not in BLOCKED_ROLES


# Good — allowlist. Only explicitly permitted roles get access.
# New roles default to "no access" until consciously added.
PERMITTED_ROLES = frozenset({"admin", "editor", "viewer"})

def can_access(user: User) -> bool:
    """Check whether the user's role permits access.

    Args:
        user: The user to check.

    Returns:
        True if the user's role is in the allowlist.
    """
    return user.role in PERMITTED_ROLES
```

### Test Authorization Boundaries

Write explicit tests that verify user A cannot access user B's resources. These tests catch the most common authorization bugs — the ones where a developer forgets an ownership check.

```python
def test_user_cannot_access_other_users_project(client, db):
    """Verify that a user receives 404 when requesting another user's project.

    This prevents IDOR (Insecure Direct Object Reference) vulnerabilities
    where guessing or enumerating resource IDs grants unauthorized access.
    """
    alice = create_test_user(db, "alice")
    bob = create_test_user(db, "bob")
    project = create_test_project(db, owner=alice)

    response = client.get(
        f"/api/projects/{project.id}",
        headers=auth_headers(bob),
    )

    # 404, not 403 — don't confirm the resource exists.
    assert response.status_code == 404
```

---

## 4. API Security

### Rate Limiting

Without rate limiting, any endpoint is a target for brute force, credential stuffing, or denial of service.

**Solo Phase:** Use middleware-level rate limiting.

**Python (FastAPI with slowapi):**

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

# Authentication endpoints get aggressive limits because they're
# the primary target for credential stuffing attacks.
AUTH_RATE_LIMIT = "5/minute"

# General API endpoints get higher limits for normal usage.
DEFAULT_RATE_LIMIT = "60/minute"


@app.post("/auth/login")
@limiter.limit(AUTH_RATE_LIMIT)
async def login(request: Request, body: LoginRequest):
    """Authenticate a user with email and password.

    Rate limited to 5 requests per minute per IP to mitigate
    credential stuffing and brute-force attacks.
    """
    # ...
```

**Team Phase:** Move rate limiting to an API gateway (Kong, AWS API Gateway, Cloudflare) so it applies uniformly to all services.

### CORS Configuration

Lock down CORS to the minimum required. A permissive CORS policy lets any website make authenticated requests to your API.

```python
from fastapi.middleware.cors import CORSMiddleware

# Explicit origin allowlist — never use "*" with credentials.
# Each origin must be a full URL including scheme and port.
ALLOWED_ORIGINS = [
    "https://app.yourservice.com",
    "https://staging.yourservice.com",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,      # No wildcards
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],  # Only methods you actually use
    allow_headers=["Authorization", "Content-Type"],  # Only headers you actually need
)
```

### Security Headers

Set these headers on every response. Most frameworks have middleware for this.

```python
from fastapi import FastAPI
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Add security headers to all responses.

    These headers instruct browsers to enforce transport security,
    prevent content type sniffing, and block framing attacks.
    """

    async def dispatch(self, request: Request, call_next) -> Response:
        """Process the request and add security headers to the response.

        Args:
            request: The incoming HTTP request.
            call_next: The next middleware or route handler.

        Returns:
            The response with security headers added.
        """
        response = await call_next(request)

        # Force HTTPS for 1 year, including subdomains.
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        # Prevent browsers from guessing content types — stops MIME-sniffing attacks.
        response.headers["X-Content-Type-Options"] = "nosniff"
        # Block the page from being embedded in iframes — prevents clickjacking.
        response.headers["X-Frame-Options"] = "DENY"
        # Disable the Referer header on cross-origin requests to avoid leaking URLs.
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        # Opt out of browser features you don't use to reduce attack surface.
        response.headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()"

        return response
```

### Error Message Sanitization

Never return internal details in API error responses. Attackers use stack traces, database error messages, and file paths to map your system.

```python
from fastapi import Request
from fastapi.responses import JSONResponse
import logging

logger = logging.getLogger(__name__)


async def global_exception_handler(request: Request, exc: Exception) -> JSONResponse:
    """Catch unhandled exceptions and return a generic error.

    Logs the full exception server-side for debugging, but returns
    only a generic message to the client to avoid leaking internals.

    Args:
        request: The incoming HTTP request.
        exc: The unhandled exception.

    Returns:
        A JSON response with a generic error message and 500 status.
    """
    # Full details in server logs — for your eyes only.
    logger.exception("Unhandled exception on %s %s", request.method, request.url.path)

    # Generic message to client — no stack traces, no table names, no file paths.
    return JSONResponse(
        status_code=500,
        content={"detail": "An internal error occurred. Please try again later."},
    )


app.add_exception_handler(Exception, global_exception_handler)
```

---

## 5. Data Protection

### Encrypt Sensitive Data at Rest

Tokens, PII (email, phone, address), and API keys stored in your database should be encrypted. If your database is compromised, encryption is the difference between a breach and an inconvenience.

```python
from cryptography.fernet import Fernet


class FieldEncryptor:
    """Encrypt and decrypt sensitive database fields using Fernet (AES-128-CBC).

    The encryption key must come from a secret manager — never hardcode it.

    Args:
        key: A Fernet-compatible encryption key (base64-encoded 32-byte key).

    Example:
        >>> encryptor = FieldEncryptor(key=os.environ["FIELD_ENCRYPTION_KEY"])
        >>> encrypted = encryptor.encrypt("user@example.com")
        >>> encryptor.decrypt(encrypted)
        'user@example.com'
    """

    def __init__(self, key: str) -> None:
        self._fernet = Fernet(key.encode())

    def encrypt(self, plaintext: str) -> str:
        """Encrypt a string value.

        Args:
            plaintext: The value to encrypt.

        Returns:
            The encrypted value as a base64-encoded string.
        """
        return self._fernet.encrypt(plaintext.encode()).decode()

    def decrypt(self, ciphertext: str) -> str:
        """Decrypt a previously encrypted value.

        Args:
            ciphertext: The encrypted value to decrypt.

        Returns:
            The original plaintext string.

        Raises:
            cryptography.fernet.InvalidToken: If the ciphertext is
                invalid or was encrypted with a different key.
        """
        return self._fernet.decrypt(ciphertext.encode()).decode()
```

### Never Log Secrets

Audit every logging call to ensure secrets, tokens, and PII are excluded.

```python
# Bad — the token ends up in your log aggregator, which is backed up,
# replicated, and accessible to anyone with dashboard access.
logger.info("User authenticated", extra={"token": access_token})

# Good — log the event and a non-sensitive identifier.
logger.info("User authenticated", extra={"user_id": user.id})
```

### Parameterized Queries

SQL injection remains in the OWASP Top 10 because developers still concatenate strings into queries. Use parameterized queries everywhere — no exceptions.

**Python (SQLAlchemy):**

```python
from sqlalchemy import select, text
from sqlalchemy.orm import Session


def get_user_by_email(db: Session, email: str) -> User | None:
    """Look up a user by email address.

    Uses parameterized queries to prevent SQL injection.

    Args:
        db: Active database session.
        email: The email address to search for.

    Returns:
        The matching User or None.
    """
    # Good — ORM query. SQLAlchemy handles parameterization.
    stmt = select(User).where(User.email == email)
    return db.execute(stmt).scalar_one_or_none()


def search_users_raw(db: Session, name_fragment: str) -> list[User]:
    """Search users by name using a raw SQL query.

    Even with raw SQL, use bound parameters — never f-strings or format().

    Args:
        db: Active database session.
        name_fragment: Partial name to search for (case-insensitive).

    Returns:
        A list of matching Users.
    """
    # Good — bound parameter with :name syntax.
    stmt = text("SELECT * FROM users WHERE name ILIKE :name")
    result = db.execute(stmt, {"name": f"%{name_fragment}%"})
    return result.all()

    # NEVER DO THIS:
    # stmt = f"SELECT * FROM users WHERE name ILIKE '%{name_fragment}%'"
```

**Rust (sqlx):**

```rust
/// Look up a user by email address.
///
/// Uses sqlx's compile-time checked query macros, which both prevent
/// SQL injection and verify the query against the database schema
/// at compile time.
///
/// # Arguments
///
/// * `pool` - Database connection pool.
/// * `email` - The email to search for.
///
/// # Errors
///
/// Returns `sqlx::Error` if the query fails.
pub async fn get_user_by_email(
    pool: &sqlx::PgPool,
    email: &str,
) -> Result<Option<User>, sqlx::Error> {
    sqlx::query_as!(
        User,
        "SELECT id, name, email FROM users WHERE email = $1",
        email
    )
    .fetch_optional(pool)
    .await
}
```

### Prevent Path Traversal

Never construct file paths from user input without validation. A `../../../etc/passwd` in a filename parameter is a classic attack.

```python
import os
from pathlib import Path

# Base directory where user uploads are stored.
UPLOAD_DIR = Path("/var/app/uploads")


def get_upload_path(filename: str) -> Path:
    """Resolve a filename to a safe absolute path within the upload directory.

    Rejects any filename that would escape the upload directory via
    path traversal (e.g., "../../../etc/passwd").

    Args:
        filename: The user-provided filename.

    Returns:
        The resolved absolute path within UPLOAD_DIR.

    Raises:
        ValueError: If the resolved path is outside UPLOAD_DIR.
    """
    # resolve() collapses ".." and symlinks to an absolute path.
    safe_path = (UPLOAD_DIR / filename).resolve()

    # Verify the resolved path is still inside the upload directory.
    if not safe_path.is_relative_to(UPLOAD_DIR):
        raise ValueError(f"Path traversal detected: {filename}")

    return safe_path
```

---

## 6. Frontend Security

### Sanitize Rendered HTML

If you render user-generated content as HTML, you must sanitize it. React escapes JSX by default, but `dangerouslySetInnerHTML` and markdown renderers bypass that protection.

**TypeScript (React with DOMPurify):**

```typescript
import DOMPurify from "dompurify";

/**
 * Render user-provided HTML content safely.
 *
 * Sanitizes the HTML through DOMPurify to strip script tags, event handlers,
 * and other XSS vectors before rendering. Only use this when you genuinely
 * need to render HTML — prefer plain text or markdown rendering when possible.
 *
 * @param html - The raw HTML string to sanitize and render.
 * @returns A JSX element containing the sanitized HTML.
 */
function SafeHtml({ html }: { html: string }): JSX.Element {
  // DOMPurify strips <script>, onerror, javascript: URIs, and other XSS vectors.
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ["b", "i", "em", "strong", "a", "p", "br", "ul", "ol", "li", "code", "pre"],
    ALLOWED_ATTR: ["href", "target", "rel"],
  });

  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

**When using react-markdown**, configure allowed elements:

```typescript
import ReactMarkdown from "react-markdown";

/**
 * Render user-provided markdown safely.
 *
 * Restricts the rendered HTML to a safe subset of elements. Disallows
 * raw HTML in the markdown source to prevent XSS via embedded scripts.
 *
 * @param content - The raw markdown string.
 */
function SafeMarkdown({ content }: { content: string }): JSX.Element {
  return (
    <ReactMarkdown
      // Block raw HTML in the markdown source.
      skipHtml={true}
      allowedElements={["p", "strong", "em", "a", "ul", "ol", "li", "code", "pre", "h1", "h2", "h3"]}
    >
      {content}
    </ReactMarkdown>
  );
}
```

### Content Security Policy

CSP tells the browser exactly which sources of scripts, styles, and other resources are allowed. A strong CSP is the most effective defense against XSS.

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.yourservice.com;
  font-src 'self';
  frame-ancestors 'none';
```

Start strict and loosen only when you have a specific, documented reason.

### Never Store Secrets in Client Code

Client-side code is public. Every variable, every API key, every config value is visible to anyone who opens the browser dev tools.

```typescript
// DANGEROUS — Vite exposes any env var prefixed with VITE_ to the client bundle.
// This API key is now visible in the browser's Sources tab.
const apiKey = import.meta.env.VITE_SECRET_API_KEY;

// SAFE — proxy sensitive API calls through your backend.
// The browser talks to your API; your API talks to the third-party service
// using credentials that never leave the server.
const response = await fetch("/api/proxy/external-service", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ query: userInput }),
});
```

The same applies to Next.js (`NEXT_PUBLIC_`), Create React App (`REACT_APP_`), and any other framework that exposes env vars to the client bundle. If a variable is prefixed for client access, assume it is public.

### XSS Prevention Checklist

- Use React's default escaping for all text content (never use `dangerouslySetInnerHTML` without DOMPurify).
- Set a strict Content Security Policy.
- Sanitize all user-generated HTML and markdown before rendering.
- Encode data in URLs with `encodeURIComponent()`.
- Never inject user input into `eval()`, `Function()`, `innerHTML`, or `document.write()`.

---

## 7. Secure Error Handling

The error handling patterns in the [Coding Style](coding-style.md) doc cover correctness and clarity. This section covers the security dimension: what errors reveal to attackers.

### Generic Messages to Clients, Details in Server Logs

**Python (FastAPI):**

```python
from fastapi import HTTPException
import logging

logger = logging.getLogger(__name__)


async def process_payment(payment_id: str) -> PaymentResult:
    """Process a payment by ID.

    Args:
        payment_id: The payment to process.

    Returns:
        The payment result.

    Raises:
        HTTPException: 500 with a generic message if processing fails.
    """
    try:
        return await payment_gateway.charge(payment_id)
    except PaymentGatewayError as exc:
        # Full details in logs — includes gateway error code, transaction ID,
        # and any debug info from the payment provider.
        logger.error(
            "Payment processing failed for %s: %s (gateway_code=%s)",
            payment_id,
            exc.message,
            exc.gateway_code,
        )

        # Generic message to client — no gateway internals, no error codes
        # that could help an attacker probe the payment system.
        raise HTTPException(
            status_code=500,
            detail="Payment processing failed. Please try again.",
        ) from exc
```

**Rust (axum):**

```rust
use axum::{http::StatusCode, response::IntoResponse, Json};
use serde_json::json;
use tracing::error;

/// Handle a payment processing error.
///
/// Logs the full error chain server-side for debugging, but returns
/// only a generic message to the client.
pub async fn process_payment(
    payment_id: String,
) -> Result<Json<PaymentResult>, impl IntoResponse> {
    match payment_gateway::charge(&payment_id).await {
        Ok(result) => Ok(Json(result)),
        Err(err) => {
            // Full context in structured logs.
            error!(
                payment_id = %payment_id,
                error = %err,
                "Payment processing failed"
            );

            // Generic response to client.
            Err((
                StatusCode::INTERNAL_SERVER_ERROR,
                Json(json!({"detail": "Payment processing failed. Please try again."})),
            ))
        }
    }
}
```

### Never Expose Stack Traces in Production

Configure your framework to suppress stack traces in production. This should be a deployment checklist item, not something left to chance.

```python
# FastAPI / Uvicorn
# In production, set debug=False and use the global exception handler
# from section 4 to catch any unhandled exceptions.
app = FastAPI(debug=False)  # Never debug=True in production
```

```typescript
// Express.js — production error handler
// The 4-parameter signature tells Express this is an error handler.
app.use((err: Error, req: Request, res: Response, _next: NextFunction) => {
  // Log the full error server-side.
  console.error(`[${req.method} ${req.path}]`, err);

  // Generic response — no err.message, no err.stack.
  res.status(500).json({ detail: "An internal error occurred." });
});
```

---

## 8. Code Review for Security

### What to Look For

Every code review should include a security pass. You don't need to be a security expert — most vulnerabilities follow predictable patterns.

**Input handling:**
- Is every external input validated with schema validation?
- Are there length limits on strings and bounds on numbers?
- Is user input ever interpolated into SQL, shell commands, or file paths?

**Authentication and authorization:**
- Does every endpoint verify the user's identity?
- Does every data access check that the user owns or has permission to access the resource?
- Are there tests proving user A cannot access user B's data?

**Data exposure:**
- Do error messages reveal internal details (table names, file paths, stack traces)?
- Are secrets logged anywhere?
- Is PII being stored unencrypted?
- Does the API return more data than the client needs?

**Dependencies:**
- Does a new dependency introduce a large attack surface?
- Is the dependency actively maintained?
- Has `pip audit` / `cargo audit` / `npm audit` been run?

### OWASP Top 10 Quick Reference

Keep this list in mind during reviews. These are the most commonly exploited vulnerability categories:

| # | Vulnerability | What to Check |
|---|--------------|---------------|
| 1 | Broken Access Control | Authorization checked on every request? IDOR tests? |
| 2 | Cryptographic Failures | Sensitive data encrypted? Strong algorithms? Keys managed properly? |
| 3 | Injection | Parameterized queries? No shell=True? No string-formatted SQL? |
| 4 | Insecure Design | Threat model considered? Rate limiting on sensitive endpoints? |
| 5 | Security Misconfiguration | Debug mode off? Default credentials removed? CORS locked down? |
| 6 | Vulnerable Components | Dependencies audited? Dependabot enabled? |
| 7 | Authentication Failures | Strong password policy? Account lockout? MFA available? |
| 8 | Data Integrity Failures | JWT signatures verified? CI/CD pipeline integrity? |
| 9 | Logging Failures | Auth events logged? Secrets excluded from logs? |
| 10 | SSRF | Server-side requests validated against allowlist? |

### Solo Phase

When you're the only reviewer, use this abbreviated checklist on every PR:

```
□ All input validated at the boundary (schema + bounds)
□ Authorization checked on every data access
□ No secrets in code, logs, or error responses
□ SQL queries parameterized
□ New dependencies audited
□ Error messages don't leak internals
```

### Team Phase

Add security-specific review requirements:

- Tag PRs that touch auth, payments, or PII for explicit security review.
- Require two approvals for changes to authentication or authorization logic.
- Run SAST (static analysis security testing) in CI — tools like `bandit` (Python), `cargo audit` (Rust), and `eslint-plugin-security` (TypeScript) catch common patterns automatically.

---

## Scaling Checklist

- [ ] Input validation with schema libraries on all API endpoints (day one)
- [ ] Parameterized queries everywhere, no string-formatted SQL (day one)
- [ ] JWT best practices: RS256, short expiry, claim validation (day one)
- [ ] Security headers middleware (HSTS, X-Content-Type-Options, X-Frame-Options) (day one)
- [ ] Rate limiting on authentication endpoints (day one)
- [ ] Generic error responses in production, no stack traces (day one)
- [ ] CORS locked down to specific origins (day one)
- [ ] Content Security Policy header (week one)
- [ ] Field-level encryption for PII and tokens at rest (before storing user data)
- [ ] Authorization boundary tests in the test suite (before multi-user access)
- [ ] SAST tools in CI pipeline: bandit, cargo audit, eslint-plugin-security (when you have CI)
- [ ] Security-focused code review checklist (when you have reviewers)
- [ ] Penetration testing by a third party (before handling financial or health data)
- [ ] OWASP Top 10 training for all engineers (before team exceeds 5 people)
