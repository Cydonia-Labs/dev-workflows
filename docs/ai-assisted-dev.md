# AI-Assisted Development

How to work effectively with Claude Code and other AI coding assistants. AI doesn't replace engineering judgment — it amplifies it. The goal is to move faster while maintaining quality.

## Philosophy

1. **AI-assisted, human-accountable.** You are responsible for every line of code that ships, whether you wrote it or an AI did. Review AI output with the same rigor you'd apply to a junior developer's PR.
2. **Use AI for leverage, not autopilot.** AI excels at generating boilerplate, exploring options, and catching things you miss. It struggles with novel architecture decisions, ambiguous requirements, and knowing your business context.
3. **Invest in prompting skill.** The quality of AI output is directly proportional to the quality of your instructions. Being precise with AI is a core engineering skill now.
4. **Keep the human in the loop for decisions.** Let AI handle implementation; keep design decisions and tradeoff calls with the human.

## Setting Up Claude Code

### Project Configuration

Create a `CLAUDE.md` at the root of your project to give Claude persistent context:

```markdown
# Project Context

## What This Project Does
Brief description of the product/service.

## Stack
- Python 3.12, FastAPI, SQLAlchemy
- Rust for performance-critical services
- PostgreSQL, Redis

## Coding Standards
- Python: Google-style docstrings, ruff for linting
- Rust: cargo clippy, cargo fmt
- All public functions must have doc comments
- All new functions must have unit tests

## Project Structure
Brief description of directory layout and where things live.

## Common Commands
- make setup — initial project setup
- make dev — run locally
- make test — run all tests
- make lint — run linters
```

This gives Claude context about your project without you repeating it every conversation. Keep it updated as your project evolves.

### Custom Slash Commands

Create reusable prompts for tasks you do repeatedly:

```
.claude/commands/
├── review.md      — review the current diff against coding standards
├── test.md        — generate tests for a given file
└── document.md    — add documentation to undocumented functions
```

Example `.claude/commands/review.md`:
```markdown
Review the current git diff and check for:
1. Missing error handling
2. Missing or inadequate tests
3. Security issues (injection, hardcoded secrets, path traversal)
4. Violations of the coding standards in CLAUDE.md
5. Logic errors or edge cases

Be specific. For each issue, explain what's wrong and suggest a fix.
```

## Where AI Excels

### Code Generation

AI is strongest when the task is well-defined and the output is verifiable:

- **Boilerplate**: API endpoints, database models, serialization code
- **Tests**: Generating test cases, especially edge cases you might miss
- **Transformations**: Porting code between languages, updating to new API versions
- **Documentation**: Doc comments, README sections, API documentation

### Code Review

AI catches things humans skim over:

- Unhandled error paths
- Security issues (SQL injection, XSS, path traversal)
- Missing input validation
- Inconsistencies with existing code patterns
- Off-by-one errors and boundary conditions

**Workflow:** After writing code, ask Claude to review it before opening a PR. This catches the obvious stuff so human reviewers (or future teammates) can focus on design and architecture.

### Research and Exploration

- "How does library X handle Y?"
- "What are the tradeoffs between approach A and approach B?"
- "Show me three ways to implement this, with pros and cons"
- "What edge cases should I consider for this function?"

### Debugging

- "Here's the error, here's the relevant code — what's going wrong?"
- "This test is flaky — what could cause intermittent failures?"
- "This query is slow — how can I optimize it?"

## Where AI Needs Supervision

### Architecture Decisions

AI will confidently propose architectures without understanding your constraints:

- Team size and skill distribution
- Budget and infrastructure constraints
- Compliance requirements
- Expected scale and growth trajectory
- Existing technical debt

**Use AI to explore options, but make the decision yourself.** Ask it to lay out tradeoffs, not to choose.

### Security-Critical Code

AI-generated code involving authentication, authorization, cryptography, or input validation should be reviewed line by line. Common issues:

- Using deprecated or weak cryptographic algorithms
- Missing authorization checks on endpoints
- Insufficient input validation
- Race conditions in session handling

### Business Logic

AI doesn't know your domain. For business rules, be extremely specific in your prompts, and verify the output against your actual requirements — not just "does it compile and pass tests."

## Effective Prompting

### Be Specific

```
# Bad — vague, AI will guess at everything
"Build a user authentication system"

# Good — specific, constrained, verifiable
"Add a POST /auth/login endpoint to the FastAPI app in src/api/auth.py.
It should accept {email, password} in the request body, verify against
the users table using bcrypt, return a JWT with a 15-minute expiry,
and return 401 with a generic error message for invalid credentials.
Add tests covering: successful login, wrong password, nonexistent user,
and missing request fields."
```

### Provide Context

- Reference specific files and functions
- Explain what already exists and what you're adding to
- State your constraints (performance requirements, compatibility, etc.)
- Share error messages and stack traces completely

### Break Down Large Tasks

Instead of "build the entire feature," break it into steps:

1. Design the data model
2. Implement the database layer
3. Add the API endpoints
4. Write tests
5. Review and refine

Each step gives you a checkpoint to verify before continuing.

### Review AI Output Critically

Questions to ask about AI-generated code:

- Does it actually solve the problem, or does it just look like it does?
- Are there edge cases it missed?
- Does it match the patterns used elsewhere in this codebase?
- Are the tests testing behavior, or just mirroring the implementation?
- Would I be comfortable debugging this code at 2am?

## Team Guidelines for AI Use

When you bring on teammates, establish clear norms:

### What's Expected

- AI-generated code goes through the same review process as human-written code.
- Commit messages should reflect what the code does, not that AI wrote it (but a co-authored-by line is good practice for transparency).
- Teammates should understand AI-generated code well enough to modify and debug it independently.

### What's Not Acceptable

- Committing AI output without reading and understanding it.
- Using AI to generate code in areas you're not qualified to review (e.g., generating crypto code without security knowledge).
- Skipping tests because "the AI said it was correct."
- Using AI to inflate commit counts or PR volume without proportional value.

## Multi-Agent Workflows

For complex tasks, use multiple AI passes:

1. **Generate**: Have AI write the initial implementation.
2. **Review**: Have AI (or a different model) review the output for bugs and issues.
3. **Test**: Have AI generate comprehensive tests.
4. **Simplify**: Have AI review for unnecessary complexity and suggest simplifications.

Each pass catches things the previous one missed.

## Tools and Configuration

### Recommended Setup

- **Claude Code**: Primary AI assistant for coding tasks. Runs in your terminal with full project context.
- **CLAUDE.md**: Project configuration file that persists context across sessions.
- **Claude Code hooks**: Automate common checks (lint, test) after AI makes changes.

### Project-Level AI Conventions

Document your AI usage conventions in `CLAUDE.md`:

```markdown
## AI Conventions
- All AI-generated code must have corresponding tests
- AI code review on every PR (use /review command)
- Complex architecture decisions require human sign-off
- AI-generated commits include Co-Authored-By trailer
```

## Scaling Checklist

- [ ] CLAUDE.md with project context (day one)
- [ ] AI review on every PR (day one)
- [ ] Custom slash commands for repeated tasks (week one)
- [ ] Team guidelines for AI use (before first hire)
- [ ] Multi-model review for security-critical code (when handling sensitive data)
