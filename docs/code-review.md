# Code Review

Code review is about catching bugs, sharing knowledge, and maintaining quality. When you're solo, AI assistants fill the reviewer role. When you grow, these same standards apply to human reviewers.

## Solo Phase: AI-Assisted Review

When you're the only developer, every PR should still get a review — from your AI assistant:

1. Open a PR (even against your own `main`).
2. Have Claude Code or your AI assistant review the diff.
3. Address any findings before merging.

This catches real bugs. AI reviewers are particularly good at spotting:
- Unhandled error cases
- Security issues (SQL injection, hardcoded secrets, path traversal)
- Missing tests for new logic
- Inconsistencies with existing patterns in the codebase

## What to Look For

### Always Check

- **Correctness**: Does the code do what the PR description says it does?
- **Edge cases**: What happens with empty input, null values, maximum sizes, concurrent access?
- **Error handling**: Are errors caught and handled meaningfully, or silently swallowed?
- **Security**: Any user input that reaches a database, shell, or file system without validation?
- **Tests**: Do tests cover the happy path, at least one edge case, and at least one error case?

### Style and Design (Lower Priority)

- Does the code follow existing patterns in the codebase?
- Are names clear and intention-revealing?
- Is there unnecessary complexity that could be simplified?

**Don't bikeshed.** If a naming choice is reasonable but not what you'd pick, let it go. Save review cycles for things that affect correctness and maintainability.

## Review Etiquette (Team Phase)

### As a Reviewer

- **Review within 24 hours.** Stale PRs kill momentum. If you can't review in time, say so and let someone else pick it up.
- **Distinguish blocking from non-blocking.** Use prefixes:
  - `blocking:` Must be fixed before merge
  - `nit:` Take it or leave it
  - `question:` I don't understand this — explain or add a comment
  - `suggestion:` Consider this alternative (with reasoning)
- **Explain why**, not just what. "This should use a parameterized query" is less helpful than "This is vulnerable to SQL injection because user input reaches the query string directly."
- **Approve with nits.** If the only remaining comments are non-blocking, approve the PR so the author can merge without another round-trip.

### As an Author

- **Keep PRs small.** Under 400 lines changed. Large PRs get rubber-stamped, not reviewed.
- **Write a good description.** The reviewer shouldn't have to read every line of code to understand the intent.
- **Respond to every comment.** Even if it's just "Done" or "Good point, fixed." This shows you read the feedback.
- **Don't take it personally.** Review comments are about the code, not about you.

## Review Checklist

Use this as a mental checklist, not a form to fill out:

```
□ I understand what this PR is trying to do
□ The approach is reasonable for the problem
□ Error cases are handled
□ Tests exist and cover meaningful scenarios
□ No obvious security issues
□ No secrets or credentials in the diff
□ The code is understandable without the PR description
```

## Common Review Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|---|---|---|
| Rubber-stamping | Bugs ship, trust erodes | Block time for real review |
| Gatekeeping | PRs stall, author loses context | Review within 24 hours |
| Style wars | Waste everyone's time | Automate with formatters and linters |
| Rewrite requests | Demoralizing, blocks progress | Suggest improvements for next PR |
| No review at all | Bugs, security holes, knowledge silos | Even solo devs should use AI review |

## Scaling Checklist

- [ ] AI-assisted review on every PR (solo phase)
- [ ] Second human reviewer required (2+ engineers)
- [ ] CODEOWNERS for automatic assignment (5+ engineers)
- [ ] Documented review SLAs (response within X hours)
- [ ] Linters and formatters enforced in CI (eliminates style discussions)
