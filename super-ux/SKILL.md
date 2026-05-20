---
name: super-ux
description: Autonomous visual-diff iteration orchestrator. Runs N rounds of a Builder↔Reviewer ping-pong where each iteration is two fresh headless AI workers in their own git worktrees — a Builder that applies fixes against a reference design (wireframe, prototype, or design-system page), and a Reviewer that pixel-diffs the live app against the reference at a configurable viewport and emits structured findings. Halts on STATUS clean, halt condition, or max iterations. Use when the user says "design QA", "visual QA", "wireframe match", "screenshot review", "run design-fidelity loop", "design loop", or invokes /super-ux.
license: MIT
compatibility: Requires git, Node.js, a pixel-diff tool (pixelmatch or equivalent), a headless screenshot capture script for your dev server, and a headless coding-agent CLI for the per-iter Builder/Reviewer workers (defaults to claude -p). Consumer provides docs/design-loop/targets.md, reference shots under docs/design-loop/reference/, and the screenshot + diff scripts.
---

# super-ux — Autonomous visual-diff iteration orchestrator

You are the orchestrator. Run **N iterations** (default 8, max configurable). Each iteration is two fresh headless AI workers in their own git worktrees: a **Builder** (writes code fixes) and a **Reviewer** (writes findings YAML, never touches source). Halt on `STATUS: clean`, `STATUS: halt`, or max iters.

## Algorithm

1. Determine iter count (CLI arg, default 8, ceiling configurable).
2. Ensure the aggregate branch exists (default `design-loop/aggregate`, cut from `${BASE_BRANCH:-main}` — set `BASE_BRANCH` in env or local config to override per-repo).
3. Notify: `🎨 design-loop starting — N iterations`.
4. For each iter:
   - Dispatch Builder → merge `design-loop/build-N` into aggregate
   - Dispatch Reviewer → merge `design-loop/review-N` into aggregate
   - Parse the Reviewer's STATUS line:
     - `STATUS: clean` → success, exit
     - `STATUS: halt (<reason>)` → halt, exit
     - `STATUS: dirty (F findings)` → continue
5. Notify halt/success at exit.

## Invocation

- `super-ux` / `/super-ux` → 8 iterations
- `/super-ux 3` → 3 iterations
- `/super-ux --resume` → continue from `state.json`'s `lastIter + 1`

## Cost cap

- Default 8 iters × ~$2/iter (with Claude `claude -p` backend) = ~$16/run. Use whatever per-iter estimate fits your worker backend.
- Halts early on `STATUS: clean` (often before 8 iters).
- Halts on stuck-pattern (same `finding_id` in 2 consecutive iters).

## Required project files (consumer creates these)

The skill assumes your repo has:

- `docs/design-loop/targets.md` — coverage list, lines like `<slug> <lang>` (e.g. `dashboard en`).
- `docs/design-loop/reference/<slug>-<lang>.png` — committed reference shots (the design source of truth).
- `docs/design-loop/feedback.md` — open findings (the Reviewer rewrites this each iter).
- `docs/design-loop/iteration-N.md` — per-iter log (the Reviewer writes this).
- `docs/design-loop/state.json` — resume marker.
- A way to capture live screenshots from your dev server (script under `scripts/`).
- A pixel-diff command (e.g. a `pixelmatch` wrapper) that writes `docs/design-loop/diff/<slug>-<lang>.{png,json}`.

See `references/builder-prompt.md` and `references/reviewer-prompt.md` for the per-role worker contracts.

## Worker dispatch

Both workers use the same dispatch shape:

```bash
# Pseudocode — adapt to your worker backend (Claude Code, Codex CLI, OpenCode, etc.)
WORKER_PROMPT="$(cat references/<role>-prompt.md) <per-iter footer>"
git worktree add -b design-loop/<role>-<N> .worktrees/design-loop-<role>-<N> "$BASE_BRANCH"
(cd .worktrees/design-loop-<role>-<N> && <worker-backend> --prompt "$WORKER_PROMPT")
git merge --no-ff design-loop/<role>-<N>
git worktree remove .worktrees/design-loop-<role>-<N>
git branch -d design-loop/<role>-<N>
```

## Stop conditions

- `STATUS: clean` → success.
- `STATUS: halt` → exit non-zero (manual review needed).
- Max iters reached without clean → halt with summary.
- Worker non-zero exit / missing close-out commit → halt.
- Merge conflict → halt.

## Companion skills

- [[super-qa]] — bug-bash on functional bugs (read its SKILL.md to understand the shared loop pattern).
- [[super-orchestrator]] — coordinates multi-skill releases.
- [[super-truth]] — verifies "STATUS clean" matches the actual pixel diff before publish.
