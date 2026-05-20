---
name: super-orchestrator
description: Workflow sequence and loop coordinator for AI engineering skills. Composes build / QA / UX / review workflows into a prompt-driven or parameterized sequence with explicit inputs, outputs, goals, loop policy, stop conditions, and human gates. Use when the user says "orchestrator loop", "run these workflows in sequence", "loop super-build and super-qa", "release-ready", or sets up an unattended multi-step run.
license: MIT
compatibility: Requires git, gh, jq, and a headless coding-agent CLI for the per-step workers (defaults to claude -p; swap via SUPER_WORKER_PROVIDER env var). Coordinates the other super-* skills in this collection — install them alongside for the full preset menu (Build Queue, Visual Polish, QA→UX Release Sweep, Review Loop, Release Ready, QA↔Build Loop).
---

# super-orchestrator — workflow sequence and loop coordinator

super-orchestrator can be triggered with explicit parameters or from a natural-language prompt. If the prompt has enough information, infer the workflow sequence and run it; if key execution details are missing, ask only for the missing parameters that materially change the run.

The reference implementation is bash-based (a pure shell loop). It dispatches one headless AI worker per iter via a per-workflow dispatch wrapper. Adapt to your runtime — the contract (worker outputs, halt gates, manifest) is what matters, not the exact shell.

## Worker backend policy

super-orchestrator and the workflows it composes are **provider-agnostic**. Use the phrase **headless agent worker** in new docs/prompts, with any specific backend (Claude Code, Codex CLI, OpenCode, a self-hosted agent SDK) as a configurable choice rather than the only path.

Suggested preferred backend order for long autonomous runs:

1. **Subscription-based headless backends** (e.g. Codex CLI under ChatGPT/Codex OAuth) — default candidate when you want to avoid per-call API billing.
2. **Interactive AI assistants with agent teams / subagents** — when you want AI-coding quality without an unattended `-p` loop.
3. **Per-call API CLIs** (`claude -p`, OpenAI API CLIs) — premium/compatibility fallback, especially for tasks where one model performs best. Verify billing/auth gates before using.
4. **OpenCode / local / other API providers** — optional fallbacks when explicitly configured and budget-gated.

Adapter contract for scripts:

```bash
SUPER_WORKER_PROVIDER="${SUPER_WORKER_PROVIDER:-auto}" # auto|codex-cli|claude-interactive|claude-p|opencode|...
```

Worker outputs should stay provider-independent: `STATUS: done`, `STATUS: wip-partial`, `STATUS: human-gate`, or `STATUS: failed`, plus changed files, verification evidence, logs/artifacts, and any budget/usage notes.

## Input model

super-orchestrator does **not** require a rigid parameter block every time. It supports two setup styles:

### 1. Explicit parameter block

Use this when the user already knows the sequence:

```text
super-orchestrator
Goal: get region list pages production-ready
Inputs: GitHub Project Ready issues + current QA report
Steps: super-build → super-qa → super-ux → super-review
Loop: continue each step until its done condition is met
Output: PR-ready summary, commits pushed, risks, human gates
```

### 2. Prompt-driven setup

Use this when the user gives a natural-language outcome:

```text
Run super-orchestrator and get the admin console ready for release.
Pick the best sequence and ask before risky actions.
```

In prompt-driven mode, infer a draft workflow from the preset list below, then ask only for missing details that materially affect execution. Do not force the user to provide boilerplate if the prompt already identifies the goal.

## Presets

Load `references/preset-decision-tree.md` when choosing a sequence from a natural-language request.

1. **Build Queue** — [[super-build]] over GitHub Project `Ready` issues. Done when the selected `Ready` queue is empty, or only blocked/human-gated cards remain.
2. **Visual Polish** — [[super-ux]] over screenshots/wireframes/reference pages. Done when reviewer status is clean, or remaining deltas need human design judgment.
3. **QA → UX Release Sweep** — [[super-qa]] → [[super-ux]]. Done when functional traversal is complete with no actionable bugs, then visual review is clean.
4. **Review Loop** — [[super-review]] → route findings to the right fix workflow → [[super-review]]. Done when review has no blocking issues and PR is merge-ready, or unresolved items are human-gated. super-review primarily logs/reviews; fixes should route through super-build, super-qa, or super-ux unless explicitly authorized.
5. **Release Ready** — suggested sequence `super-build? → super-qa → super-ux → super-review`. Include super-build when the GitHub Project has relevant `Ready` issues; skip it when the queue is empty and the goal is validation/polish/review.
6. **QA↔Build Loop** — state-machine loop driven by a QA-specific GitHub Project (default name `Super Ultimate QA`). Each iter inspects column counts and dispatches super-build (drain `Bug`) or super-qa (extend coverage). Done when `Bug` and `Queue` columns are empty AND the QA queue file has no remaining `[ ]` routes. See "QA↔Build state machine" below.

## QA↔Build state machine (preset 6)

State lives on a GitHub Project (default title `Super Ultimate QA`) with six Status columns:

| Column   | Meaning                                              |
|----------|------------------------------------------------------|
| Queue    | Work the next iteration will pick up                 |
| Testing  | Feature currently being explored or verified         |
| Done     | Spec passing, no action needed                       |
| Bug      | Failing spec; super-build picks these up next        |
| Flaky    | Only passes on retry; quarantine + investigate       |
| Skip     | Out of scope; documented and parked                  |

Each orchestrator iter is one state-machine step.

### Per-iter logic

```bash
# Resolve project once per run
OWNER="${SUPER_QA_PROJECT_OWNER:-$(gh repo view --json owner -q .owner.login)}"
TITLE="${SUPER_QA_PROJECT_TITLE:-Super Ultimate QA}"
PROJECT_NUMBER=$(gh project list --owner "$OWNER" --format json \
  | jq --arg t "$TITLE" '.projects[] | select(.title|ascii_downcase == ($t|ascii_downcase)) | .number')

# Read column counts
ITEMS_JSON=$(gh project item-list "$PROJECT_NUMBER" --owner "$OWNER" --limit 200 --format json)
bug_count=$(echo "$ITEMS_JSON"   | jq '[.items[] | select(.status=="Bug")]   | length')
queue_count=$(echo "$ITEMS_JSON" | jq '[.items[] | select(.status=="Queue")] | length')
routes_left=$(grep -c '^- \[ \]' docs/super-qa/queue.md 2>/dev/null || echo 0)

# Route
if   [[ $bug_count   -gt 0 ]]; then dispatch super-build with BUILD_LOOP_QA_MODE=1
elif [[ $queue_count -gt 0 || $routes_left -gt 0 ]]; then dispatch super-qa
else echo "📭 board drained — exit clean"; exit 0
fi
```

### Required env (set once per run, or in your project's local config)

```bash
SUPER_QA_PROJECT_TITLE="Super Ultimate QA"
SUPER_QA_PROJECT_OWNER=<your-org>      # defaults to repo owner
```

### Halt gates

Standard "Done and loop policy" halts apply. Plus:

- If `Bug` count doesn't decrease for **3 consecutive super-build iters**, halt and escalate — workers are stuck on the same card.
- If `Queue` count doesn't increase after a super-qa iter AND `routes_left > 0`, halt — QA worker is failing to harvest children.
- If both `Bug` and `Queue` are zero but `routes_left > 0`, allow up to 5 super-qa iters to harvest from queue.md before declaring done.

### Notification cadence

- 1 message at start: resolved project, initial column counts.
- 1 per iter: which lane (`🔧 super-build → Bug` or `🔍 super-qa → Queue`), and delta from previous iter.
- 1 final: column counts at end, total time, queue.md routes remaining.

## Sequence policy

Do **not** force one universal sequence. Suggest the best preset first, show the inferred sequence, and proceed when either:

- the user explicitly approves it; or
- the user request clearly maps to a low-risk preset and the next actions are reversible/local.

Ask before risky actions: production deploys, destructive DB changes, external customer-visible changes, merging conflicted branches, or ambiguous product decisions.

## Done and loop policy

Default loop behavior is **continue until done**, not "stop after N attempts". Individual workflows define done conditions:

- **super-build:** GitHub Project `Ready` queue is clear, or remaining cards are blocked/human-gated.
- **super-qa:** target traversal is complete and no new actionable functional bugs are found.
- **super-ux:** screenshot/wireframe/reference review is clean, or remaining differences require human design judgment.
- **super-review:** no blocking review findings remain and PR/branch is merge-ready, or unresolved items are human-gated.

Workflow authority:

- **super-qa** keeps local markdown/state for traversal and bug logs. It may create GitHub issues/cards for actionable bugs that should remain in the project queue, and it may fix in-scope bugs directly when safe.
- **super-ux** records visual findings and may fix direct visual mismatches when the target is clear. If the finding needs product/design judgment, create/surface an issue or human gate instead of guessing.
- **super-review** should primarily produce findings and route fixes to the right workflow. It should not silently push fixes unless explicitly authorized.

Allowed halt gates are safety gates, not arbitrary iteration caps:

- no progress / same failure repeats and needs human judgment;
- merge conflict or dirty working tree that cannot be resolved safely;
- destructive database or production action would be required;
- required external service is unavailable;
- user-defined budget/time window is reached;
- context/tooling degradation would make claims unreliable.

## super-truth gates

When the prompt says **super-truth**, load/use the [[super-truth]] workflow. Treat it as a phase-boundary verification gate, not another implementation pass.

Use super-truth before:

- closing/moving Build issues to Done when the fix claim is non-trivial;
- accepting 3 consecutive zero-bug super-qa iterations;
- accepting super-ux `clean`/visual-complete claims;
- accepting super-review release-ready claims;
- sending the final "done/release-ready" notification.

Gate outputs are `gate_decision: publish|halt|escalate-to-human`. In autonomous mode, `halt` or `escalate-to-human` stops the run with concrete evidence and a resume command; do not call `AskUserQuestion` mid-loop.

## Run manifest

Write a lightweight manifest when a run spans more than one workflow or lasts more than one iteration. This makes the run auditable and resumable.

Recommended path:

```text
docs/super-orchestrator/runs/<YYYY-MM-DD>-<slug>.md
```

Start from `templates/run-manifest.md`. Record the user request, inferred preset, goal/done definition, inputs, outputs, selected sequence, loop policy, halt gates, step results, links to issues/PRs/reports/screenshots, and final status (`done`, `halted`, or `human-gated`).

Use the term **Loop** publicly: super-orchestrator owns sequencing, retries, loop policy, and stop conditions. Individual workflows can still iterate internally, but the public menu names should stay capability-focused: **super-build**, **super-qa**, **super-ux**, **super-review**.

## Companion skills

- [[super-build]] — drains GitHub Project `Ready` issues in parallel worktrees.
- [[super-qa]] — BFS route-crawler that builds and runs your Playwright spec suite.
- [[super-ux]] — visual-diff Builder↔Reviewer ping-pong.
- [[super-review]] — PR/code/architecture readiness reviewer.
- [[super-truth]] — adversarial verification gate.
