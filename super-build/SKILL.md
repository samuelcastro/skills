---
name: super-build
description: Parallel headless AI worker dispatcher for GitHub Projects. Reads issues from a configured project board's `Ready` column, runs each as an isolated headless AI worker in its own git worktree (max 3 in parallel by default), merges results back, closes the issue, advances the project card to `Done`, and notifies a chat channel on each completion. The project board is the single curation surface — drag a card to `Ready` and the loop picks it up. Use when the user says "run the build loop", "drain my Ready queue", "/super-build", or wants autonomous queue-draining for a GitHub-curated backlog.
license: MIT
compatibility: Requires git, gh, jq, and a headless coding-agent CLI (defaults to claude -p; swap to Codex CLI/OpenCode/etc. by editing scripts/super-build-dispatch.sh). Worker preamble references advisor sub-skills (superpowers:test-driven-development, /plan-eng-review, /cso, etc.) — adapt to whichever advisor skills your environment provides; the contract is multi-role majority vote, not specific skill names.
---

# super-build — Headless parallel GitHub-Project executor

You are the orchestrator. The source of truth is the **`Ready` column of the configured GitHub Project board**. Your job: dispatch every issue in `Ready` (in board order) as a parallel headless AI worker in a git worktree, merge results back to the active branch, close the issue, advance its project card to `Done`, and ping your chat channel on each completion.

**Curation contract:** the human moves cards into `Ready` when they should be worked. The loop never reads `Backlog`, `In Progress`, or `Done` cards. If `Ready` is empty, the loop reports "queue empty" and exits cleanly — no error.

**This skill is single-purpose: EXECUTE curated GitHub Project `Ready` issues.** It is INDEPENDENT of [[super-qa]] (the autonomous bug-bash loop that hardens shipped code). They share no state unless [[super-orchestrator]] explicitly sequences them. Run super-build to make forward progress on the issue queue; run super-qa to bug-bash the existing codebase. They CAN run concurrently in different orchestrator sessions, but expect merge conflicts if both touch the same files.

## Role boundary

super-build is the canonical builder. Do not use a separate `build-feature` workflow. If the user wants a one-off feature, first create or curate a GitHub issue/card with clear acceptance criteria, move it to `Ready`, then let super-build execute it.

super-build may implement, test, commit, merge worker branches, close completed issues, and move project cards to `Done`. It should not invent new product scope beyond the issue body. If the issue needs product/design/security judgment, apply the human-gate or WIP-partial path instead of guessing.

## Worker backend

super-build is **provider-agnostic**. The reference implementation uses Anthropic's `claude -p` CLI as the headless worker, but any "headless coding agent" backend works: Codex CLI, OpenCode, a self-hosted agent SDK, etc. The `scripts/super-build-dispatch.sh` reference dispatcher targets `claude -p` by default; replace the inner `claude -p` invocation with your backend of choice.

## Configuration

The orchestrator needs to know which project board to read. There are two modes:

### Standalone mode (default — feature queue)

Reads `Ready` from the repo's feature project. Resolution order:

1. **`BUILD_LOOP_PROJECT`** env var — project number (e.g. `2`).
2. **`BUILD_LOOP_OWNER`** env var — project owner (GitHub org or user).
3. **Auto-discovery fallback:** if env vars unset, run `gh project list --owner $(gh repo view --json owner -q .owner.login) --format json` and pick the project whose `title` matches the repo name, or the only open project if there's exactly one.

Surface: `🎯 Reading project: <owner>/<title> (#<number>), column "Ready"`.

### QA-loop mode (orchestrator-driven — drain `Bug` column)

When invoked by [[super-orchestrator]] as part of the QA↔Build loop, super-build drains the `Bug` column on a QA-specific project (default name `Super Ultimate QA`) rather than `Ready` on the feature project. Enabled by:

- `BUILD_LOOP_QA_MODE=1` env var, **or**
- [[super-orchestrator]] exporting it via the dispatch wrapper.

Resolution:

1. Project resolved by `SUPER_QA_PROJECT_TITLE` (default: `Super Ultimate QA`) at the configured owner — same logic as [[super-qa]].
2. Source column: `Bug` (override with `BUILD_LOOP_SOURCE_COLUMN`).
3. Target column on success: `Done` on the same QA project.

Standalone and QA-loop mode never run in the same orchestrator session by default — pick one per run. They can run concurrently in different sessions because they read different boards.

## Algorithm

### 0. Pre-flight

Before reading the board:

- Confirm the main worktree is clean: `git status --porcelain` must be empty.
- Confirm `gh auth status` works and the repo remote points at the expected GitHub repo.
- Confirm `jq`, `git`, and your worker backend CLI are available.
- Resolve and print `BUILD_LOOP_OWNER` and `BUILD_LOOP_PROJECT`.
- Run `git fetch origin` and identify the base branch from the current checkout; do **not** assume `main`.
- If this run is part of super-orchestrator, update the run manifest with preset `Build Queue`, input project, base branch, and done definition.

### 1. Read the source column

Source column is `Ready` in standalone mode, `Bug` in QA-loop mode (override with `BUILD_LOOP_SOURCE_COLUMN`):

```bash
SOURCE_COLUMN="${BUILD_LOOP_SOURCE_COLUMN:-${BUILD_LOOP_QA_MODE:+Bug}}"
SOURCE_COLUMN="${SOURCE_COLUMN:-Ready}"

gh project item-list "$BUILD_LOOP_PROJECT" --owner "$BUILD_LOOP_OWNER" --limit 200 --format json \
  | jq --arg col "$SOURCE_COLUMN" '.items[] | select(.status == $col and .content.type == "Issue")'
```

For each `Ready` item:
- Capture `content.number` (issue #), `content.title`, `content.body`, `content.labels` (via separate `gh issue view N --json labels` if `item-list` doesn't include them).
- **Skip** issues with the `loop:in-progress` label (in flight in another orchestrator), `loop:halted` label (manually paused), or `human-gated` label (requires manual handling).
- Parse `body` for `Depends on: #N1, #N2` lines (case-insensitive). If any dep is still open AND not in the current `Ready` set, the issue is blocked — leave it for a future run.
- Parse `body` for an optional `Skills:` line (e.g. `Skills: superpowers:test-driven-development, superpowers:verification-before-completion`). If absent, the worker uses defaults from the preamble.

**Ordering:**
1. **Board order first** — `gh project item-list` returns items in the human's manual board order. Respect that. If the user wants Issue X before Issue Y, they drag X above Y on the board.
2. **Tiebreaker (rare):** within the same drag-position, sort by priority extracted from title regex `^\[?(P[1-4])\]?`: `P1` < `P2` < `P4` < `P3` < no priority. Then issue number ascending.

If `Ready` is empty: print `📭 Project board "Ready" column is empty — nothing to dispatch.` and exit 0. This is not an error; it means the human hasn't curated work yet.

### 2. Ensure required labels exist (idempotent)

Once per run, before first dispatch:
```bash
gh label create loop:in-progress --color FFA500 --description "Currently being worked on by super-build" 2>/dev/null || true
gh label create loop:halted --color B60205 --description "Manually paused — super-build will skip" 2>/dev/null || true
gh label create human-gated --color B60205 --description "Requires manual handling — super-build will not auto-execute" 2>/dev/null || true
```

### 3. Dispatch wave (max 3 concurrent)

Notify your chat channel once per wave: `▶️ super-build dispatching: Issues [#N1, #N2, #N3] (parallel)`.

For each issue N in the ready set (up to 3 at a time):

a. **Lock the issue:**
```bash
gh issue edit N --add-label loop:in-progress
```

b. **Announce on the issue** (so a human reading the issue page knows it's being worked):
```bash
gh issue comment N --body "🤖 Dispatched by super-build — worker spinning up in worktree \`.worktrees/issue-N\` on branch \`loop/issue-N\`. Skills: <skills-line-from-body-or-defaults>."
```

c. **Dispatch:**
```bash
BASE_BRANCH=<current-branch> bash scripts/super-build-dispatch.sh N
```

Run with backgrounding (e.g. `run_in_background` in your agent harness, or `&` in shell). Capture each shell ID.

The reference dispatcher at `scripts/super-build-dispatch.sh` handles:
- `git worktree add -b loop/issue-N .worktrees/issue-N <resolved-base-ref>` (resolves `BASE_BRANCH` to either the local branch or `origin/<branch>` if remote-only)
- `gh issue view N --json title,body,labels` to compose the worker prompt
- prepend `references/worker-preamble.md` + append working-directory footer
- `cd` into worktree and exec the worker backend (default `claude -p`)
- verify the worker produced a `chore(loop): close #N` commit on its branch

Dispatcher exit codes:
- `0` — success (close-commit present)
- `2` — worker non-zero, no close/WIP marker
- `3` — worker exit 0 but no `chore(loop): close #N` commit
- `4` — HUMAN GATE TRIPPED (log contains the literal marker)
- `5` — WIP-PARTIAL (worker stopped at an intentional partial slice)
- `64` — usage / config error (bad args, missing env, repo or branch lookup failure, gh API error, worktree-path conflict). Treat as halt-and-investigate; the orchestrator does NOT retry on this.

**Base branch:** the orchestrator's currently-checked-out branch. Pass via `BASE_BRANCH` env var to the dispatcher. Don't assume `main`.

### 4. Wait + reconcile

Poll each in-flight shell. As each finishes:

**On dispatcher exit 0 (success):**
- `cd <repo-root>`
- `git merge --no-ff loop/issue-N -m "merge: loop/issue-N (closes #N)"`
- `gh issue close N --comment "Closed by super-build in $(git rev-parse --short HEAD)"`
- `gh issue edit N --remove-label loop:in-progress`
- `git worktree remove .worktrees/issue-N`
- `git branch -D loop/issue-N`
- Move the project item to `Done` if GitHub does not do it automatically on issue close.
- Notify: `✅ super-build issue #N closed (merged)`.
- Recompute the ready set; if new issues are now unblocked, dispatch in the next wave (respecting the 3-concurrent throttle).

**On dispatcher exit 5 (intentional WIP-PARTIAL — merge as partial, do NOT close):**
- `cd <repo-root>`
- `git merge --no-ff loop/issue-N -m "merge: loop/issue-N partial — <slice from worker's final message> (#N)"` — the issue number reference (no `closes #` keyword) means GitHub will not auto-close the issue.
- `gh issue edit N --remove-label loop:in-progress --add-label human-gated`
- `gh issue comment N --body "✅ Foundation/partial slice merged as $(git rev-parse --short HEAD). Issue stays open with \`human-gated\` label until the rest of the implementation lands."`
- Before cleanup, archive the branch as a tag: `git tag archive/loop-issue-N-$(date +%Y%m%d-%H%M%S) loop/issue-N` (so `git branch -D` is recoverable).
- `git worktree remove .worktrees/issue-N`
- `git branch -D loop/issue-N`
- Notify: `🟡 Issue #N partial merged — issue stays open (human-gated)`.
- **Continue dispatching the next issue in the wave.** A WIP-PARTIAL is NOT a halt. The worker did intentional, scoped work and the orchestrator advances.

**On dispatcher exit 2 or 3 (worker failed or no done-commit):**
- `gh issue edit N --remove-label loop:in-progress`
- `gh issue comment N --body "❌ super-build worker failed (exit code <X>). Last 50 lines of log:\n\n\`\`\`\n$(tail -50 .planning/super-build-logs/issue-N.log)\n\`\`\`\nWorktree at \`.worktrees/issue-N\` left intact for human inspection."`
- Notify with `tail -50` of `.planning/super-build-logs/issue-N.log`.
- Do NOT merge; leave the worktree intact for human inspection.
- Halt the loop.

**On dispatcher exit 4 (HUMAN GATE TRIPPED):**
- `gh issue edit N --remove-label loop:in-progress --add-label human-gated`
- `gh issue comment N --body "🔴 HUMAN GATE TRIPPED — needs manual handling. See worktree \`.worktrees/issue-N\`."`
- Notify: `🔴 Issue #N tripped HUMAN GATE — needs manual handling`.
- Halt the loop.

### 5. Final report

When the selected GitHub Project `Ready` queue is empty, or only blocked/skipped cards remain: send a summary listing issues closed this run, issues still skipped (`human-gated` / `loop:halted` / blocked dependencies), and any halts. Suggest: "Run [[super-qa]] to confirm closed issues are truly complete."

## Issue contract

Every `Ready` issue should be executable without live clarification. Preferred issue body shape:

```markdown
## Goal
<one-sentence outcome>

## Acceptance Criteria
- [ ] <observable behavior or artifact>
- [ ] <test/verification expectation>

## Notes / Constraints
- Depends on: #123, #456
- Skills: superpowers:test-driven-development, superpowers:verification-before-completion
- Human gates: <deploy, destructive DB action, product decision, etc.>
```

super-build treats acceptance criteria as the completion contract. Workers must not edit checkboxes to make the issue look complete. If the issue is too vague to execute, label/comment it as `human-gated` instead of guessing.

## Constraints

- **Maximum 3 parallel workers** at a time (resource throttle; override via `BUILD_LOOP_MAX_PARALLEL`).
- **Never auto-touch issues with `human-gated` label** (production cutover, secrets, irreversible ops).
- **Never modify code in the main worktree** while workers are running. Only run `gh issue` commands and merge/cleanup operations.
- **Merge conflicts** between concurrent workers' branches → halt loop, notify with conflict files, leave `loop:in-progress` label so a human can resolve. Don't auto-resolve.
- **Notification cadence:** 1 message at start, 1 per dispatch wave, 1 per completion (success/fail), 1 final summary. Don't spam.
- **Workers MUST use the right skills.** The worker preamble enforces: workers parse the `Skills:` line from the issue body if present, otherwise use defaults (`superpowers:test-driven-development`, `superpowers:verification-before-completion`). All decision points use a multi-role advisor pattern with majority vote (tie → smallest blast radius). See `references/gstack-voting.md` for when to invoke vs. when to escalate, and the required `--- gstack-vote ---` commit trailer.

## Worker preamble

The preamble at `references/worker-preamble.md` is the worker contract: decision policy, HUMAN GATE handling, the per-issue gate contract, and final-commit format. The dispatcher prepends it to every worker prompt. See that file for the verbatim text.

## Recovery / re-entry

If the user re-invokes super-build after a partial run:
- Re-read `gh issue list`. Issues already closed are skipped automatically.
- If `.worktrees/issue-N` exists for an N that's still open: a previous worker was interrupted. Default: notify the user and ask before auto-resuming (a re-dispatch overwrites the previous attempt's branch).
- If the `loop:in-progress` label is set on issue N but no worktree exists: stale lock from a crashed orchestrator. Remove the label and treat as ready.

## Stop conditions

- Selected GitHub Project `Ready` queue is empty, or all remaining `Ready` cards are blocked / `human-gated` / `loop:halted`.
- Any worker fails (dispatcher exit 2 or 3).
- HUMAN GATE TRIPPED (dispatcher exit 4).
- Merge conflict.
- User interrupts.

**Not a stop condition:** WIP-PARTIAL (dispatcher exit 5). The orchestrator merges the partial, leaves the issue open with `human-gated`, and continues dispatching the next issue in the wave. WIP-PARTIAL is intentional, bounded work.

## Invocation patterns

- `super-build` / `/super-build` → process all GitHub Project `Ready` issues in board order
- `/super-build --only N` → execute only issue #N (smoke-test mode, ignores `loop:in-progress` label on that issue if you set `FORCE=1`)
- `/super-build --dry-run` → print the dispatch plan (issues, order, parallelism waves) without invoking workers, without setting labels, without commenting

## Companion skills

- [[super-qa]] is the autonomous bug-bash iteration loop. Independent of this skill — they share no state unless [[super-orchestrator]] explicitly sequences them.
- [[super-orchestrator]] composes super-build with QA, UX, and review workflows.
- [[super-truth]] verifies that the close-commit actually satisfies the issue's acceptance criteria.
