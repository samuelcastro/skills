---
name: super-truth
description: Adversarial verification gate for AI workflow output. Takes the output of any producer skill (code review findings, fix commits, test verdicts, visual diffs, feature implementations) and spawns 2-3 adversarial sub-agents that try to prove or disprove the claims against reality. Returns a confidence score (0-100), concrete counterexamples, and a publish/halt gate decision. Use when you want to ground-truth-check another skill's output, ask "did the fix actually fix it?", "is this test brittle?", "is this finding real?", or want a publish gate before another workflow ships its result.
license: MIT
compatibility: Requires git, and an agent runtime that can spawn parallel sub-agents (Claude Code's Task tool, Codex's spawn, or equivalent). Set a per-run budget ceiling — adversarial verification runs 2-3 sub-agents per invocation.
---

# super-truth — Adversarial Verification Gate

Producer skills are optimistic: they report "done" when their internal checks pass. Their checks are usually lint, type, or rule-match — not "did this actually fix the user's problem". **super-truth is the only skill whose job is to ask "does the user-visible thing really work, in a way an adversary couldn't refute?"**

Without it, the system trusts every skill's self-report. With it, every claim earns its trust through adversarial proof.

## When to invoke

- `/super-truth verify <input>` — explicit verification request. `<input>` is one of:
  - A code review comment URL or local `comment.md` path
  - A commit SHA (typically a fix or feature merge commit)
  - A QA iteration report or a single test spec path
  - A visual-diff iteration log or screenshot diff
- `<other-skill> --truth-gate` — companion gate mode. When another producer skill finishes, it can hand its output to super-truth and refuse to publish until the gate passes. The orchestration lives in the calling skill — see "Calling super-truth from another workflow" below.
- Natural-language invocation — "did this review really find a bug or is it hallucinating?", "did that fix actually close the issue?", "is this spec brittle?", etc.

## Algorithm

### 1. Classify the input

Read the input artifact and identify the **claim shape**:

| Source | Example claim | Verifier focus |
|---|---|---|
| Code review finding | "P1: ListPage `isPending` swallows errors" | Can the bug be reproduced? Is the cited code current? Is the severity right? |
| Fix commit | "Closes #42 by adding error branch" | Does `before` reproduce the bug? Does `after` not? Did anything regress? |
| QA green spec | "dashboard route: tc-1 passes" | Does the spec assert user-visible behavior, or just status 200? Mutation test: break the feature, does the spec turn red? |
| Visual "clean" verdict | "iteration-N: STATUS clean" | Does the pixel diff math match the reviewer's prose? Are reference shots current? |
| Feature implementation | "Issue #N done" | Does the merged code satisfy the issue's acceptance criteria? Or just compile? |

### 2. Spawn 2–3 adversarial sub-agents in parallel

Each agent gets a different angle:

- **Reproducer** — tries to make the claim true *or* false from first principles. For a bug finding, write a minimal repro. For a fix, run before/after. For a QA spec, mutate the source and re-run.
- **Code-grounder** — verifies citations are accurate. Does the cited file:line still exist? Does the cited code do what the finding claims? Has it been refactored?
- **Historian** — checks `git blame`, ADR docs, prior incidents. Was this behavior intentional? Was a similar issue filed and closed wontfix? (Avoids re-litigating settled design decisions.)

Run the agents as fresh subagents so their context is independent. Cap total parallel agents at 3.

### 3. Aggregate into a verdict

Combine the agent outputs into:

```yaml
schema: super-truth/v1
input: <path or URL>
input_type: review-finding | fix-commit | qa-spec | ux-verdict | feature-impl
verified_at: 2026-05-20T20:00:00Z
confidence: 88           # 0-100
verdict: ground-truth    # ground-truth | partial | hallucination | undecidable
counterexamples:
  - finding: SE-3
    evidence: |
      Repro at docs/super-truth/cases/SE-3-repro.ts triggers the
      blank-table empty-state on a forced query error. Confirmed.
gate_decision: publish   # publish | halt | escalate-to-human
notes: |
  Reproducer + code-grounder both green. Historian flagged that
  PR #28 added a similar guard for the same component but on a
  different query — not a duplicate, but worth noting.
```

Save the verdict to `docs/super-truth/verdicts/<YYYY-MM-DD>-<input-slug>.yaml` and write a one-line summary to `docs/super-truth/log.md`.

### 4. Gate decision

- **confidence ≥ 70 AND no contradicting counterexample** → `gate_decision: publish`. Calling skill may proceed.
- **confidence 40–69 OR partial agreement among agents** → `gate_decision: escalate-to-human`. Calling skill should not auto-publish; show the verdict to the user and ask.
- **confidence < 40 OR clear counterexample** → `gate_decision: halt`. Calling skill must not publish. Surface the counterexample.

Render the gate decision back to the user (or your notification channel) so the human knows a check ran.

## Calling super-truth from another workflow

A producer skill that wants to gate its output adds two steps before publish:

```bash
# In the calling skill's algorithm:
#   - After producing the output artifact (e.g. a review comment.md
#     or a fix merge commit), invoke super-truth in a fresh subagent:
SUPER_TRUTH_INPUT=$(realpath <output-artifact>)
# Notify: "🔬 super-truth gating <artifact>..."
# Run /super-truth verify "$SUPER_TRUTH_INPUT" in a fresh subagent.
# Read the resulting verdict YAML at docs/super-truth/verdicts/<latest>.yaml.
case "$VERDICT_GATE" in
  publish) ;;
  escalate-to-human) exit 3 ;;   # caller asks the user to look
  halt)    exit 2 ;;             # caller refuses to publish
esac
```

The calling skill defines its own retry policy. super-truth itself is stateless and never publishes anything — it only writes verdicts.

## Worked examples

### Verifying a code review finding

Input: a review comment claiming `P1: ListPage's "isPending" branch silently swallows error-state, rendering empty-state when the API is down`.

- **Reproducer agent**: writes a Playwright spec that mocks a 500 response, navigates to the page, asserts the empty-state element is visible. Spec passes → bug is real. Saves spec for future regression.
- **Code-grounder**: greps for `isPending` in the cited file, confirms the branch exists, confirms there is no sibling `isError` branch. Cites the exact lines.
- **Historian**: `git log --all --grep="empty state"`. Finds nothing in the last 90 days. No prior wontfix. Concern is novel.

Verdict: `confidence: 92`, `verdict: ground-truth`, `gate_decision: publish`.

### Verifying a fix commit

Input: commit `abc123` claiming `Closes #42 — add isError branch to ListPage`.

- **Reproducer**: checks out `abc123^`, runs the repro spec → red. Checks out `abc123`, runs again → green. Then runs the broader regression suite at `abc123` to catch incidental breakage.
- **Code-grounder**: greps for the new `isError` branch, confirms it's present, confirms it short-circuits before the empty-state render.
- **Historian**: not strictly needed for a fix; can be skipped or used to confirm the commit author isn't undoing a recent intentional change.

Verdict: `confidence: 88`, `verdict: ground-truth`, `gate_decision: publish`.

### Verifying a "green" test spec

Input: `e2e/paths/dashboard.spec.ts` marked green in a QA iteration report.

- **Reproducer (mutation test)**: comment out the dashboard's data-fetching call, re-run the spec. If the spec still passes, the spec was checking status code, not user-visible behavior. Hallucination.
- **Code-grounder**: read the spec, count the assertions that aren't `expect(page).toHaveURL(...)` or status checks. Fewer than 3 substantive assertions = brittle.
- **Historian**: how many times has this spec changed in the last 5 commits? Specs that get rewritten often are flaky.

Verdict: `confidence: 50`, `verdict: partial`, `gate_decision: escalate-to-human`. Counterexample: "spec asserts page title but not table contents; mutation test shows it stays green when data-fetch is removed."

## Edge cases

- **Verifier can't reproduce** (env-specific, requires production data) — `verdict: undecidable`, `gate_decision: escalate-to-human`. Don't fabricate confidence.
- **All three agents disagree** (one says real, one says hallucination, one undecidable) — `verdict: partial`, lower confidence, `gate_decision: escalate-to-human`.
- **Input is stale** (head SHA in the envelope doesn't match current PR head) — exit with `verdict: undecidable, reason: stale_input`. Calling skill must refresh upstream output first.
- **Self-verification loop** — super-truth must NEVER verify another super-truth verdict. If asked, halt with "circular: super-truth output is the ground truth by design."
- **Cost ceiling** — each run spawns 2–3 verification subagents. Set a per-run token/budget ceiling (default $5 / 200k tokens) and short-circuit with `verdict: undecidable, reason: budget_exhausted` if exceeded.

## Why this matters

The confidence number is meant to be honest — under-95 is normal even for clean cases. The gate's job is to catch the cases where confidence falls below the publish threshold, not to rubber-stamp.

## Files

```
super-truth/
├── SKILL.md (this file)
└── (no helper scripts — the skill is pure orchestration)

docs/super-truth/        # consumer creates these as needed
├── verdicts/            # per-run YAML, append-only
├── log.md               # one-line summary index
└── cases/               # repro specs/scripts written by the Reproducer agent
```

## Companion skills

- `super-review` — primary producer; verdicts gate review findings before fixes act on them
- `super-build` — verifies merge commits before closing the issue
- `super-qa` — verifies green specs aren't brittle (mutation testing)
- `super-ux` — verifies "STATUS clean" matches the actual pixel diff
- `super-orchestrator` — can sequence other skills with a super-truth gate between each step
