# Multi-role advisor voting for super-build workers

super-build workers should consult a **multi-role advisor pattern** on ambiguous fix decisions: poll several role perspectives (product, engineering, security, design, QA) in parallel, take the majority vote, and break ties via smallest blast radius.

You can implement this with whatever advisor skills your environment provides. The reference convention is named **"gstack" voting** (after Garry Tan's "Use GStack, not /goal" framing — multiple expert lenses on the same decision beats a single agent's best guess), but the pattern is what matters, not the name. If your environment uses different advisor skill names (e.g. `panel-vote`, `committee-review`), substitute them throughout — keep the multi-role + majority + smallest-blast-radius contract.

## When to invoke

Use multi-role voting inside a worker session when the issue body or in-flight implementation forces a non-obvious decision:

- **Scope ambiguity** — the bug fix has multiple plausible boundaries (fix one symptom vs. refactor the call site vs. rewrite the module).
- **Compatibility tradeoff** — a fix is correct but would break a public contract or downstream consumer.
- **Security-adjacent change** — touching auth, secrets, permissions, or any data flow that crosses a trust boundary.
- **Design choice with no precedent** — the codebase has no existing pattern for what the issue asks for.

Do **not** invoke for routine work:

- Mechanical fixes (typo, lint, off-by-one, missing import).
- Bugs whose fix is dictated by an existing test or spec.
- Issues with explicit acceptance criteria that leave no judgment call.

## How to invoke

If you have a CLI advisor (e.g. `gstack`, `panel`, your own multi-role tool):

```bash
<advisor-cli> vote --topic "<one-line decision>" \
  --context "<file path or short summary>" \
  --options "A: <option>" "B: <option>" "C: <option>"
```

Otherwise, fall back to inline role-play in the worker: synthesize one sentence per role (product, engineering, security, design, QA) weighing the options, then take a majority vote. Document the vote in the commit message under a `--- gstack-vote ---` trailer:

```
fix(orders): use idempotency key from request header (closes #123)

<one-line summary>

--- gstack-vote ---
- Product: B (ship the smaller change, revisit later)
- Eng:     B (less surface area to regress)
- Security: B (no auth boundary touched)
- Design:  A (matches existing pattern in /payments)
- QA:      B (easier to write a deterministic test)
vote: B (4 of 5)
```

The vote stays attached to the commit so the orchestrator and downstream reviewers can audit why a non-obvious choice was made.

## When to escalate to human instead

Multi-role voting is a tiebreaker for **gray decisions**, not a replacement for explicit policy. Escalate to human (label issue `human-gated`, leave worktree intact, stop) when:

- The fix would require production deploy or destructive DB change.
- The vote is split with no clear majority and no smaller-blast-radius option is obvious.
- Any role explicitly raises a "this is a deal-breaker" signal (security reviewer flags an auth bypass, etc.).
- The issue itself is unclear about what "fixed" means.

Halts here cost less than a regression in production.
