# Agent Instructions Reference

Use this reference when writing project-specific `AGENTS.md`, `CLAUDE.md`, relay instructions, or agent onboarding docs. It defines the kind of rules that make human-led multi-agent work reliable without turning the process into bureaucracy.

This is a template, not a replacement for project-local instructions. The project repo remains the source of truth.

## Project Role Model

Define explicit roles before multiple agents start working in parallel.

Example role model:

| Role | Responsibility |
| --- | --- |
| Human owner | Final architecture decisions, merge decisions, long-run approvals, provider or GPU budget approvals |
| Architecture agent | Architecture docs, decision memos, benchmark or scorer review, issue decomposition, PR review, high-risk contract changes |
| Implementation agent | Tactical implementation, tests-first work, fixtures, script wiring, CI cleanup, lint and test hardening |
| UI/demo agent | Code delivery, demo runtime wiring, UI/frontend work, tests, and small features |
| Operations agent | Training coordination, corpus/train/bench orchestration, long-run monitoring, operational handoffs |

The human owner remains the final decision maker. Agent agreement is useful evidence, not final authority.

## Product Framing

Every agent instruction set should state what the product is, what it is not, and which constraints must survive future changes.

Useful framing questions:

- What is the product's durable purpose?
- Which current implementation domain is only the first domain?
- Which architecture choices need to stay portable?
- Which runtime behaviors must match training, test, or demo behavior?
- Which capabilities are governed by policy rather than convenience?

Keep this section short. It should help agents avoid local optimizations that damage the product direction.

## Recurring Engineering Principles

Document project principles that should shape agent behavior.

Examples:

1. Preserve working implementation shape.
   - Unless an implementation is broken, extend it instead of replacing it.
   - Prefer compatibility layers, config adapters, and extension points.
   - Do not create one-off launchers or parallel orchestration paths without a clear reason.

2. Treat manifests and registries as canonical.
   - Tool names, permissions, capability tags, and broker actions should come from configured sources of truth.
   - Do not invent tools, permissions, or catalog entries in code.
   - If benchmark, prompt, corpus, or runtime disagrees with the manifest, flag drift before changing code.

3. Keep production-shaped traces.
   - Training-time prompts should match production/runtime prompts.
   - Generation metadata belongs outside trainable conversation fields.
   - Do not leak synthesis or orchestration metadata into model training data.

4. Make official evaluation deterministic.
   - Record seed, temperature, top_p, do_sample, model, adapter, git SHA, manifest hash, taxonomy hash, and prompt hash when relevant.

5. Do not trust aggregate scores alone.
   - Inspect per-query outputs and trace sidecars.
   - Distinguish model failures from benchmark bugs, catalog drift, prompt ambiguity, scorer defects, and runtime wiring issues.

6. Gate expensive or mutating work.
   - Require prompt audit and simulator/preflight before provider spend.
   - Use run manifests and hash pinning for large synthesis or benchmark work.
   - Use shared queues or multi-host coordination for large wall-clock jobs unless the human owner overrides.

7. Treat retrieval as governed tool use.
   - Retrieval should be invoked through explicit tools or broker behavior.
   - Retrieval outputs should use evidence envelopes or equivalent provenance.
   - Retrieval is not magical always-on context.
   - Training should preserve runtime tool behavior.

## Branch Discipline

Project instructions should include these rules:

- Never work directly on `main` unless the human owner explicitly authorizes it.
- Never work on another person's or agent's branch unless asked.
- Use focused owner-prefixed branches.
- Before editing, run `git status`, identify the current branch, and check recent commits.
- Before PR, summarize changed files, tests run, risks, and rollback plan.

Example branch names:

```text
arch/design-<topic>
arch/fix-<topic>
dev/fix-<topic>
dev/test-<topic>
ops/bench-<topic>
ui/demo-<topic>
```

## PR Review Discipline

Reviews lead with findings ordered by severity.

Review focus should include:

- manifest, taxonomy, prompt, benchmark, or runtime drift,
- scorer false positives and false negatives,
- deterministic-eval provenance gaps,
- corpus or prompt metadata leaking into trainable fields,
- missing tests for contract changes,
- paid-run, GPU-run, synthesis, or corpus mutation risk,
- production-path test gaps,
- stacked PR or squash-merge hazards.

If clean, say so clearly and identify residual risk.

## Safety And Cost Controls

Document what requires explicit human approval.

Require approval for:

- paid provider synthesis,
- GPU training,
- multi-host synthesis,
- benchmark suites over an agreed threshold,
- corpus mutation or promotion,
- live provider calibration bursts,
- production data mutation,
- irreversible infrastructure changes.

Usually allowed without special approval:

- read-only analysis,
- dry-runs,
- lint,
- unit tests,
- local targeted tests,
- local static checks.

## Coding Standards

Agent instructions should point to the project's language-specific style docs. Keep the local summary brief.

Example Python standards:

- Use Google-style docstrings for public functions, classes, and modules.
- Document parameters, returns, and raises.
- Use `pytest` for unit tests.
- Cover happy path, edge case, and failure case for new functions.
- Prefer explicit errors over silent failures.
- Run `ruff check --fix` and `ruff format` before commit when applicable.

General standards:

- Keep patches small and testable.
- Find existing patterns before adding abstractions.
- Do not add abstractions unless they remove real duplication or prevent known drift.
- Prefer production-path tests for behavior that matters in production.

## Runtime Awareness

Agents should know where code actually runs.

Document:

- target operating system,
- deployment host or cluster class,
- required interpreters and toolchains,
- GPU or accelerator assumptions,
- base model or runtime family when relevant,
- paths that are valid in production versus local development.

Instruction to include:

Do not assume macOS paths, local-only execution, or the editing machine's hardware. If the target runtime differs from the editing machine, say so in PR notes and handoffs.

## Slack Relay And Agent Communication

If agents use a relay, document the architecture precisely.

Include:

- which process owns Slack Socket Mode or equivalent inbound connection,
- whether agents run receivers, listeners, or both,
- how addressed messages route to each agent,
- agent IDs,
- ports,
- host users or runtime locations,
- where receiver logs live,
- how to check receiver health.

Example relay table:

| Agent ID | Port | Role |
| --- | ---: | --- |
| `arch` | `8701` | architecture/review |
| `dev1` | `8702` | implementation |
| `ops` | `8703` | operations/long-run coordination |
| `ui` | `8704` | implementation/UI/demo |

Each agent owns its own receiver health.

## Comms Working Model

Use this principle:

GitHub is the durable state machine. Slack is the coordination bus.

Canonical surfaces:

| Question | Surface |
| --- | --- |
| PR or issue state | GitHub |
| Diff, commit, or file changes | GitHub |
| Review approval state | GitHub |
| Who is blocked or what is next | Slack |
| Long-run handshakes | Slack with checklist, then durable report in GitHub |

If GitHub and Slack disagree, GitHub wins and Slack gets corrected.

## Required Slack Tags

Use these tags:

- `[handoff]`: author requests review or action.
- `[review-result]`: reviewer verdict after GitHub review.
- `[fix]`: author pushed fix.
- `[ready]`: author requests re-review.
- `[merge-ready]`: reviewer confirms GitHub approval and no blockers.
- `[merged]`: owner reports merge only when downstream info matters.
- `[post-merge]`: post-merge validation or residual risk.
- `[blocked]`: needs decision or another owner.
- `[gates-clean]`: preflight clean for long-running operation.
- `[fire]`: human owner authorizes long-running operation.

Every handoff should include:

```text
State: needs-review | changes-requested | ready-for-re-review | approved | merged | blocked
Next owner: arch | dev | ops | ui | human-owner
Blocking: yes/no
```

## Required PR Flow

1. GitHub PR or comment first.
2. Author applies explicit reviewer labels or assignments at PR creation.
3. Slack handoff second, naming reviewer lanes.
4. Reviewer reviews on GitHub first.
5. Reviewer posts Slack summary second.
6. Owner fixes branch.
7. Owner comments on GitHub with commit SHA.
8. Owner posts Slack `[fix]` or `[ready]`.
9. Reviewer re-reviews on GitHub.
10. Reviewer posts `[merge-ready]` only after the PR has at least two
    non-author approvals and no requested changes.
11. Human owner merges.

A Slack `[ready]` message alone is not merge-ready.

Every PR needs at least two reviews from accounts other than the PR author
before merge. The author's own review never counts. The human owner can
override explicitly in Slack or GitHub/Forgejo, but the override must be visible
and durable enough for the team to understand the exception.

Authors should not rely on unassigned PR fallback alerts as the normal route.
Use explicit reviewer labels or assignments, for example:

```text
needs-arch-review
needs-dev-review
needs-ui-review
needs-ops-review
```

## Monitoring And SLA

Each agent should:

- verify receiver health at session start,
- check inbox before Slack posts,
- check inbox after meaningful work units,
- check inbox before "done", "ready", or "standing by" messages,
- check inbox before starting new work.

Routine action SLA: 15-20 minutes during active work.

Blocking, `[gates-clean]`, and `[fire]`: surface or act within about 5 minutes.

Reading is not acting. Addressed messages require action, blocker, or explicit acknowledgement.

Exception: do not acknowledge no-action status messages indefinitely. Avoid acknowledgement loops.

## Long-Running Operation Handshake

Before training, benchmark, synthesis, demo fire, migration, or other long-running operation, post `[gates-clean]` with:

- prompt/runtime parity,
- adapter, model, dataset, or artifact path,
- corpus path when relevant,
- manifest and prompt hashes,
- trace capture and provenance plan,
- confirmation that latest `main` contains required fixes,
- tokenized trainability when relevant,
- budget and wall-clock estimate,
- output or report path,
- abort or rollback criteria.

The human owner replies `[fire]`.

## Generated Work Contract Packages

A generation family, benchmark family, prompt suite, or corpus slice is a contract package, not just templates plus lint.

Each family should define:

- registry entry,
- sub-pools and quotas,
- required anchors,
- seed pool,
- prompt renderer and hash,
- acceptance lint,
- shared-backstop policy,
- provider eligibility,
- zero-spend simulator,
- canary and full-fire gates.

Keep the detailed contract in project design docs. Reference those docs from workflow instructions instead of duplicating every rule.

## Retrospective Practice

Every major cycle should maintain a running retrospective.

Capture:

- good,
- bad,
- wish-we'd-never-seen-that,
- actionable changes.

Do not wait until the end of a cycle. Capture lessons while they are fresh.
