# Onboarding

Getting a new team member productive as fast as possible. This guide serves double duty: it's a checklist for onboarding new hires, and it's a test of whether your development environment is actually reproducible.

## Why This Matters Early

Even as a solo founder, write your onboarding doc now:

- It forces you to automate and document your setup.
- When you hire your first engineer, they can self-serve instead of consuming your time for a week.
- If your laptop dies, it's your own disaster recovery plan.

## First Day Checklist

### Access Provisioning

```
□ GitHub org — added to team with appropriate permissions
□ Cloud provider console (AWS, GCP, etc.) — scoped to their role
□ Communication tools (Slack, Discord, etc.)
□ CI/CD secrets — they should NOT have direct access to production secrets
□ Internal tools and dashboards
□ Password manager — shared vaults for team credentials
```

**Principle of least privilege:** Start with minimal access. Add permissions as needed, not preemptively.

### Development Environment

This should take under 30 minutes if your setup is documented properly:

```
□ Clone the repo
□ Run the setup script (make setup)
□ Copy .env.example to .env and fill in local values
□ Run the test suite (make test) — should pass on first try
□ Run the app locally (make dev) — should work
□ Make a trivial change, push a branch, open a PR — CI should pass
```

If any of these steps fail or require undocumented manual intervention, **fix the setup process**, not the symptom.

### Context Loading

```
□ Read the README and CONTRIBUTING guide
□ Read this engineering handbook (you're reading it)
□ Review the last 10 merged PRs to understand current work patterns
□ Review open issues to understand current priorities
□ Pair with the founder/lead on one task to learn the workflow live
```

## First Week Guide

### Day 1–2: Set Up and Ship Something Small

- Complete the environment setup checklist.
- Pick a small, well-defined task (fix a typo, add a test, update a dependency).
- Go through the full workflow: branch, code, PR, review, merge.
- This validates their setup works end-to-end and gets them comfortable with the process.

### Day 3–4: First Real Task

- Pick a feature or bug that's well-scoped and doesn't require deep system knowledge.
- Pair or check in frequently — the goal is learning the codebase, not independence yet.
- Review their PR thoroughly — this is a teaching opportunity, not a gatekeeping one.

### Day 5: Retro

- What was confusing about the setup process?
- What's undocumented that should be?
- What would they change about the workflow?

**Update the docs based on their answers.** Every new hire makes the onboarding better for the next one.

## Knowledge Transfer

### What to Document vs. What to Transfer Live

| Document | Transfer Live |
|----------|--------------|
| Setup instructions | Architecture decisions and their history |
| Coding conventions | Why certain tradeoffs were made |
| Deploy process | Where the dragons are (fragile areas) |
| API contracts | Customer/user context |

Documentation answers "what" and "how." Live conversation answers "why" and "what have we tried."

### Architecture Overview

Write a brief architecture doc that covers:

```markdown
## System Architecture

### Components
- [Component A] — what it does, what it talks to
- [Component B] — what it does, what it talks to

### Data Flow
How a request flows through the system, from user action to response.

### Key Decisions
- Why we chose [technology X] over [technology Y]
- Why [component] is structured this way
- Known limitations and planned improvements
```

This doesn't need to be polished. A rough diagram and a few paragraphs save hours of explanation.

## Offboarding Checklist

When someone leaves, reverse the access provisioning:

```
□ Revoke GitHub org access
□ Revoke cloud provider access
□ Revoke CI/CD and deploy access
□ Rotate any shared secrets they had access to
□ Remove from communication tools
□ Transfer ownership of any services or resources they managed
```

**Do this the same day they leave.** Deferred offboarding is a security risk.

## Scaling Checklist

- [ ] Setup script that works from scratch (day one — this is for you too)
- [ ] README with getting-started instructions (day one)
- [ ] This onboarding checklist completed and tested (before first hire)
- [ ] Architecture overview document (before first hire)
- [ ] Offboarding checklist (before first hire)
- [ ] Buddy/mentor system (5+ engineers)
- [ ] Formal onboarding program with milestones (10+ engineers)
