# Human-Led Multi-Agent Workflows

This document describes a practical workflow for one human owner coordinating multiple AI agents. It is designed for small teams that want useful parallelism without losing track of branch state, review state, or who owns the next action.

The model is intentionally simple:

- GitHub is the durable state machine.
- Slack is the coordination bus.
- The human owner is the final decision maker.

## Operating Principle

GitHub owns durable project state:

- issues,
- PRs,
- branches,
- diffs,
- commits,
- reviews,
- requested changes,
- approvals,
- merge history,
- durable review findings.

Slack owns short-lived coordination:

- handoffs,
- blockers,
- next owner,
- review summaries,
- long-running operation status,
- explicit approvals.

Nothing important should exist only in Slack. If a finding affects the code, design, test coverage, merge readiness, or release risk, record it in GitHub first. Slack can summarize and route attention, but GitHub remains the source of truth.

If GitHub and Slack disagree, GitHub wins. Correct the Slack thread instead of treating a Slack message as an override.

## Project Context Sources

Before an agent starts substantial work, it should inspect the project-local context files that exist in that repo:

- `AGENTS.md`
- `CLAUDE.md`
- communication model docs
- current milestone plans
- architecture or design docs relevant to the task
- retrospective notes from the current cycle

Do not duplicate all technical details into handoff messages. Reference the durable docs and summarize only the decision, risk, or next action needed.

For a reusable checklist of what belongs in those project-local instructions, see [Agent Instructions Reference](agent-instructions-reference.md).

## Roles

Define roles by responsibility, not by model name.

| Role | Typical responsibility |
| --- | --- |
| Human owner | Final decisions, merges, budget approvals, production judgment |
| Architect/reviewer agent | Design review, contract review, issue decomposition, high-risk fixes |
| Implementation agent | Feature work, bug fixes, tests, fixtures, scripts |
| Operations agent | Long-running jobs, corpus/artifact generation, evals, manifests, reports |
| UI/demo agent | Frontend wiring, demo runtime, UI tests, small product polish |

Agents can change roles between tasks, but each handoff should identify the next owner clearly.

## PR Lifecycle

Use this loop as the default:

1. Author opens a PR.
2. Author applies explicit reviewer labels or review assignments at PR creation.
3. Author posts Slack `[handoff]` with PR number, issue, branch, validation, explicit ask, reviewer lanes, state, next owner, and blocking status.
4. Reviewer reviews on GitHub first.
5. Reviewer posts Slack `[review-result]` summary second.
6. If changes are requested, author fixes the branch.
7. Author comments on GitHub with commit SHA and fix summary.
8. Author replies in the Slack thread with `[fix]` or `[ready]`.
9. Reviewer re-reviews on GitHub.
10. Reviewer posts `[merge-ready]` only when GitHub is approved, at least two non-author approvals are present, and no requested changes remain.
11. Human owner merges from GitHub.
12. Slack `[merged]` is only required when downstream operational information matters.

If a PR was previously `CHANGES_REQUESTED`, a Slack "ready" message is not enough. GitHub approval or an explicit human-owner override is required before merge.

Every PR needs at least two reviews from accounts other than the PR author
before merge. The author's own review never counts. The human owner can
override explicitly in Slack or GitHub/Forgejo, but the exception must be
visible in the coordination thread or durable PR record.

Explicit reviewer routing is the normal path. Unassigned PR sweeps are only a
safety net. Authors should use reviewer labels or assignments such as:

```text
needs-arch-review
needs-dev-review
needs-ui-review
needs-ops-review
```

## Slack Tags

Use these tags consistently:

| Tag | Meaning |
| --- | --- |
| `[handoff]` | New work, review request, or ownership transfer |
| `[review-result]` | Reviewer verdict after GitHub review |
| `[fix]` | Author pushed a fix |
| `[ready]` | Author requests re-review |
| `[merge-ready]` | Reviewer confirms GitHub approval and no open blockers |
| `[merged]` | Merge reported because downstream information matters |
| `[post-merge]` | Validation after premature merge or residual-risk check |
| `[blocked]` | Needs another owner or explicit decision |
| `[gates-clean]` | Long-running operation preflight is clean |
| `[fire]` | Human owner authorizes long-running operation |

Every meaningful handoff should include:

```text
State: needs-review | changes-requested | ready-for-re-review | approved | merged | blocked
Next owner: arch | dev | ops | ui | human-owner
Blocking: yes/no
```

## Slack Message Examples

### Handoff

```text
[handoff] PR #42 - tighten manifest validation

Issue: #31
Branch: dev/fix-manifest-validation
Validation: pytest tests/manifest -q; cargo test manifest
Ask: review validation logic and test coverage
State: needs-review
Next owner: arch
Blocking: no

Notes:
- Adds explicit hash presence checks.
- No corpus files changed.
- GitHub PR description has the full test transcript.
```

### Review Result With Findings

```text
[review-result] PR #42

Verdict: changes requested on GitHub
State: changes-requested
Next owner: dev
Blocking: yes

Findings:
1. high - Manifest hash is checked for presence but not matched against the generated artifact.
2. medium - Missing production-path test for regenerated manifest input.

Residual risk:
- I did not run the full benchmark suite.
```

### Review Result With No Findings

```text
[review-result] PR #43

Verdict: no issues found in GitHub review
State: approved
Next owner: human-owner
Blocking: no

Residual risk:
- Only smoke tests were run locally.
- Full nightly benchmark remains outside this PR.
```

### Fix And Ready

```text
[fix] PR #42

Commit: a1b2c3d
Fix summary:
- Compares manifest hash against generated artifact hash.
- Adds production-path regression test.

Validation: pytest tests/manifest -q
State: ready-for-re-review
Next owner: arch
Blocking: no
```

### Merge Ready

```text
[merge-ready] PR #42

GitHub state: approved, no requested changes remain
Validation checked: pytest tests/manifest -q from latest author comment
State: approved
Next owner: human-owner
Blocking: no

Merge note:
- Safe to squash.
- No downstream artifact regeneration required.
```

### Merged With Downstream Note

```text
[merged] PR #42

State: merged
Next owner: ops
Blocking: no

Downstream:
- Main now contains the manifest validation fix.
- Long-running eval can use main after pulling latest.
```

## Review Expectations

Reviews must lead with findings, ordered by severity. Summaries come after findings, not before them.

A strong review checks:

- runtime, benchmark, prompt, or manifest drift,
- scorer false positives and false negatives,
- prompt hash or manifest hash gaps,
- trainable metadata leakage,
- corpus, canary, or synthesis gates,
- missing production-path tests,
- hard-coded user paths,
- wrong branch or stacked-PR merge risk,
- provider, GPU, or corpus mutation risk,
- generated artifacts that should or should not be committed.

If no issues are found, say so clearly and name residual risks. "No issues found" is useful only when the reviewer also says what was and was not checked.

## Branch Discipline

Follow these rules unless the human owner explicitly overrides them:

- Do not work directly on `main`.
- Do not work on another agent's branch unless asked.
- Create focused branches.
- Before editing, check branch, git status, recent commits, and any worker lock.
- Do not use destructive git commands unless explicitly requested.
- Do not squash, rebase, or rewrite another agent's work unless that is the assigned task.

Branch names should make ownership and purpose obvious:

```text
arch/design-scorer-contract
dev/fix-manifest-validation
ops/eval-report-refresh
ui/demo-runtime-panel
```

Before editing:

```bash
git branch --show-current
git status --short
git log --oneline -5
```

## Long-Running Work Gates

Long-running work needs explicit approval when it can spend money, mutate durable artifacts, consume scarce hardware, or block the team for a long time.

Examples:

- provider synthesis,
- corpus mutation,
- GPU training,
- long benchmark suites,
- production data migration,
- large artifact regeneration.

Before `[fire]`, post `[gates-clean]` with:

- intended model, adapter, dataset, or artifact path,
- runtime parity confirmation,
- manifest and prompt hashes when relevant,
- output or report path,
- budget or hardware cost,
- expected wall-clock time,
- rollback or abort condition,
- confirmation that latest `main` contains required fixes.

The human owner replies `[fire]` to authorize.

### Gates Clean Example

```text
[gates-clean] long benchmark suite

Target: benchmark profile full-v2
Input artifact: artifacts/eval/main-2026-05-12/manifest.json
Runtime parity: confirmed against latest main
Manifest hash: sha256:9d7...
Prompt hash: sha256:41c...
Output path: reports/bench/full-v2/2026-05-12/
Budget: no provider spend; estimated 2 GPU-hours
Expected wall-clock: 3-4 hours
Abort condition: error rate > 2%, missing artifact, or GPU queue preemption
Latest main: includes PR #42 manifest validation fix

State: blocked
Next owner: human-owner
Blocking: yes
```

### Fire Example

```text
[fire] approved for full-v2 benchmark

Use the target, inputs, and abort conditions from the latest [gates-clean].
Post hourly status and final report path.
```

## Contract Packages For Generated Work

Generated corpora, benchmark families, prompt suites, and similar systems are not just templates plus lint. Treat each family as a contract package.

Each package should define:

- registry entry,
- sub-pools and quota targets,
- required prompt anchors,
- seed pool,
- acceptance lint,
- shared-backstop policy,
- provider eligibility,
- zero-spend simulator,
- canary,
- full run only after canary passes.

Failures this prevents:

- missing planned anchors,
- variant leakage,
- classifier or scorer from one family applied to another,
- negative-path gate conflicts,
- provider treating a generation prompt as live tool use,
- row-level lint passing while distribution is wrong.

Keep the detailed contract in project design docs. Slack should only carry the handoff, canary result, gate status, and next owner.

## Headless Relay And Slack Monitoring

Each agent needs a reliable monitoring setup if Slack is part of the workflow.

Minimum expectations:

- receiver process is running,
- headless worker processes one task at a time,
- worker lock prevents collisions,
- worker audit logs are retained,
- Python, test, and project runtime environments are available to the worker,
- addressed Slack messages get an action, blocker, or explicit acknowledgement,
- no-action status messages may be silently absorbed.

Silent absorption is acceptable for status only. It is not acceptable for work requests, blockers, review requests, `[fire]`, or `[gates-clean]` messages.

During active work, routine coordination should be handled within 15-20 minutes. Blocking messages and `[fire]` approvals should surface within about 5 minutes.

## Post-Merge Validation

Post-merge validation is required when:

- a PR merged before re-review,
- a stacked PR was squash-merged,
- an artifact or generated file should have landed on `main`,
- runtime behavior needs smoke verification,
- a long-running operation depends on the merged change.

Example:

```text
[post-merge] PR #42

Reason: stacked PR was squash-merged
Validation:
- Pulled latest main.
- Confirmed manifest validation file is present.
- Ran pytest tests/manifest -q.

State: merged
Next owner: ops
Blocking: no
Residual risk:
- Full benchmark not rerun.
```

## Retrospective Capture

Capture lessons continuously. Do not wait until the end of a cycle.

Use four buckets:

- good,
- bad,
- wish-we'd-never-seen-that,
- actionable process, tooling, or test changes.

Good retrospectives produce changes to checklists, tests, docs, automation, branch rules, or handoff templates. If nothing changes, the retrospective probably happened too late or stayed too vague.
