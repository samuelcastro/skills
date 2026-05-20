---
name: super-review
description: PR, code, architecture, and release-readiness reviewer. Inspects a branch or PR, classifies findings by severity (blocker / should-fix / nit / human-gate), routes fixes to other producer skills instead of silently changing code, and produces a merge-readiness verdict with verification evidence. Use when the user says "review this branch", "review loop", "make sure this is merge-ready", "PR review", or asks for code, architecture, security, QA evidence, or release-readiness judgment.
license: MIT
compatibility: Requires git and gh. Most effective when paired with super-build / super-qa / super-ux so blocker findings can be routed to a producer skill instead of fixed in-place.
---

# super-review — PR/code readiness reviewer

**super-review** is the reviewer/router workflow. It checks whether a branch or PR is safe to merge, records actionable findings, and routes fixes to the right producer skill.

super-review is conservative with claims: only say **merge-ready** when review evidence is clean and verification has passed. If evidence is missing, say what is unverified.

## When to use

- PR or branch review before merge.
- Code, architecture, security, data-model, or migration judgment.
- Release-readiness checks after a build, QA, or UX pass.
- The **Review Loop** preset in [[super-orchestrator]].
- A final pass that needs risks, blockers, and human gates summarized.

Do **not** use this as the primary implementation workflow. Route fixes to:

- **[[super-build]]** for feature/task implementation from a GitHub Project `Ready` queue.
- **[[super-qa]]** for functional bugs, broken behavior, failing test paths, or missing QA coverage.
- **[[super-ux]]** for visual fidelity, layout, screenshots, wireframe match, or design-system drift.

## Inputs

Accept any of these inputs:

- current branch or local diff;
- GitHub PR number or URL;
- commit range;
- user-provided file list;
- QA report, screenshots, or orchestrator run manifest;
- release goal / done definition.

If the input is ambiguous, default to reviewing the current branch against its upstream/base branch. Ask only when the base branch, PR, or target scope materially changes the result.

## Review flow

1. **Establish scope**
   - Identify branch, base branch, PR, changed files, and user goal.
   - Check working tree status before reviewing.
   - If there are unrelated dirty files, stop and ask before touching them.

2. **Inspect changes**
   - Read the diff and the affected modules.
   - Check the project's conventions from its `CLAUDE.md`, `AGENTS.md`, `README.md`, or other root-level guidance documents. Examples of project-specific rules to look for:
     - clock / time abstractions (e.g. `clock.now()` instead of `new Date()`);
     - service vs. repository boundaries (services own business logic, repositories own data access);
     - job-handler transaction ownership;
     - money/currency type conventions (e.g. `numeric(12,2)` over `float`);
     - date vs. timestamp conventions for calendar days;
     - structured error patterns (e.g. `AppError({ error_code, context })`);
     - schema-validated JSON writes (Zod, Pydantic, etc.).

3. **Classify findings**
   - **Blocker:** correctness, data loss, security, auth, migrations, money, customer-visible broken behavior, or failing required tests.
   - **Should fix:** maintainability, missing tests, risky edge cases, accessibility, i18n, observability, or design drift that is clearly in scope.
   - **Nit / optional:** style or cleanup that does not block merge.
   - **Human gate:** product/design/ops decision that cannot be safely guessed.

4. **Route fixes**
   - If a blocker is an implementation task, hand it to [[super-build]].
   - If a blocker is a functional regression, hand it to [[super-qa]].
   - If a blocker is visual/design fidelity, hand it to [[super-ux]].
   - If the user explicitly authorizes super-review to fix, make the smallest safe patch, verify it, and clearly report that review also changed code.

5. **Verify evidence**
   - Run the smallest meaningful verification for the touched area.
   - Prefer targeted tests first; run broader suites when the change crosses boundaries.
   - For upload/import flows, do not call it complete from UI success or HTTP 200 alone; verify jobs reach terminal state and destination records are saved.
   - If verification is skipped, state why and mark merge-readiness as unverified.

6. **Report**
   - Lead with the final status: `merge-ready`, `blocked`, `human-gated`, or `unverified`.
   - Include findings grouped by severity.
   - Include verification commands and results.
   - Include which producer skill should own each fix.

## Output format

```markdown
## super-review result: <merge-ready | blocked | human-gated | unverified>

- Scope: <branch/PR/files reviewed>
- Base: <base branch/commit if known>
- Verification: <commands + pass/fail/skipped>

### Blockers
- [ ] <finding> → route to <super-build | super-qa | super-ux | human>

### Should fix
- [ ] <finding> → route to <workflow>

### Human gates
- <decision needed>

### Merge-readiness
<clear statement of whether this can merge now, and why>
```

For short status notifications (chat/Slack/Telegram), keep it phone-friendly:

```markdown
**super-review: blocked**

- **Scope:** PR #123 / current branch
- **Blockers:** 2
- **Verified:** `npm test -- --run imports`
- **Next:** route functional bug to super-qa, schema decision to human gate
```

## Review Loop behavior

When [[super-orchestrator]] runs **Review Loop**, use this sequence:

1. super-review inspects branch/PR and writes findings.
2. super-orchestrator routes each actionable finding to super-build, super-qa, or super-ux.
3. The owning workflow fixes and verifies its scope.
4. super-review runs again against the updated branch.
5. Stop only when no blocking review findings remain, or unresolved items are explicitly human-gated.

super-review should not silently push fixes during Review Loop unless the user or orchestrator explicitly grants that authority.

## Common pitfalls

- Calling a branch **fixed** or **merge-ready** before tests or evidence prove it.
- Treating UI success or HTTP 200 as enough evidence for background jobs, uploads, or imports.
- Mixing reviewer findings with broad refactors.
- Creating duplicate GitHub issues without checking whether the finding is already tracked.
- Letting super-review become another alias for super-build; keep review authority separate from implementation authority.

## Done condition

super-review is done when one of these is true:

- no blocking findings remain and the branch/PR has enough verification evidence to call it merge-ready;
- all unresolved findings are explicitly human-gated;
- required evidence cannot be collected because tooling/service access is unavailable, and the output clearly marks the result as unverified.
