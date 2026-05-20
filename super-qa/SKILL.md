---
name: super-qa
description: BFS route-crawler that builds your Playwright spec suite. Walks every route in your app, classifies pages by type, writes type-appropriate specs, captures evidence (screenshots, HARs, console logs, page errors), files product-readable GitHub issues for failures (with priority/area/category labels), and continues until traversal is complete or only human-gated blockers remain. Use when the user says "QA review", "bug bash", "build the e2e suite", "verify and fix", or invokes /super-qa.
license: MIT
compatibility: Requires git, gh, jq, Node.js + Playwright, and a headless coding-agent CLI for the per-iter worker (defaults to claude -p). The worker preamble references advisor sub-skills (superpowers:test-driven-development, superpowers:systematic-debugging, superpowers:verification-before-completion, playwright-best-practices, /plan-eng-review, etc.) — adapt to your environment's advisor skills.
---

# super-qa — BFS route-crawler that builds your spec suite

You are the orchestrator. Your job: drive a BFS crawl of the target app that **builds `e2e/paths/` into a comprehensive Playwright suite** while fixing any bugs it stumbles on. Each iteration is a fresh **headless agent worker** that does ONE iter and exits. There is no worktree — the worker commits its iter outputs (the queue update, the iteration report, new spec files) directly to the active branch (default `main`, but use whichever branch the orchestrator handed you). **Production-code fixes do NOT go on the active branch**; the worker opens a PR branch per fix and routes it through the reviewer pipeline. See "Fix flow — PRs, never direct-to-main" in `references/iteration-preamble.md`.

This skill is **independent of [[super-build]]**. They share no state. super-build executes new feature work from the GitHub Project board (forward progress); super-qa hardens what already ships by exhaustively walking the route tree.

After this loop has run for a while, `e2e/paths/` is the deliverable. CI runs `npm run test:e2e` on every PR. **No AI involvement in steady-state regression.** AI is only needed to extend the suite when new pages ship.

```
                AI loop (build phase)             CI (steady state)
               ─────────────────────              ──────────────────
super-qa       →  e2e/paths/    →     npm run test:e2e on PR
(drains queue.md,          grows toward          catches regressions
 fixes bugs)               comprehensive         without any AI tokens
```

## Worker backend

The reference implementation dispatches `claude -p` as the headless worker. Any backend that respects the iteration preamble works — Codex CLI, OpenCode, a custom agent harness, etc. The contract (close-out commit format, halt gates, evidence capture) is provider-independent.

## Auth bootstrap — auto-discover, never ask

Before iter 1, resolve a test login. **Never ask the user; discover or create.** Order:

1. **Cached creds** in your project's local-settings file (e.g. `.claude/settings.local.json` `.e2e.user` / `.e2e.pass`) — use them as-is.
2. **Env-file fallback** — read in order: `.env.production.local`, `.env.local`, `.env`. If your project uses Supabase, pull `SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` and auto-create a test user (e.g. `e2e-test@<owner-domain>`) via the admin API. Save creds to local settings so the next run skips bootstrap. For other auth providers, adapt the bootstrap step to your provider's admin API.
3. **Unauth-only fallback** — if neither cached creds nor admin keys exist, log "no admin key — restricting to public routes" and proceed with public/login/signup pages only. Do NOT halt.

The bootstrap is **non-interactive**. If the admin-API returns "user already exists", reset the password via the same endpoint and continue. Save creds either way.

## QA loop state board — GitHub Project as the state store

State for the QA↔Build loop lives in a GitHub Project (default title `Super Ultimate QA`, auto-discovered by title at the user/org level). It has six Status columns:

| Column   | Meaning                                              |
|----------|------------------------------------------------------|
| Queue    | Work the next iteration will pick up                 |
| Testing  | Feature currently being explored or verified         |
| Done     | Spec passing, no action needed                       |
| Bug      | Failing spec; [[super-build]] picks these up next    |
| Flaky    | Only passes on retry; quarantine + investigate       |
| Skip     | Out of scope; documented and parked                  |

This board is the durable, machine-readable state for the loop. `docs/super-qa/queue.md` remains the BFS route seed and audit log, but **all actionable findings land on the project board** so [[super-orchestrator]] can gate on `Bug` column non-empty without parsing markdown.

### Project resolution

Resolution order (used by `super-qa-file-bug.sh` and the orchestrator):

1. If `SUPER_QA_PROJECT_TITLE` is set, query `gh project list --owner $SUPER_QA_PROJECT_OWNER --format json` and pick the project whose title matches (case-insensitive).
2. Otherwise default to title `Super Ultimate QA`.
3. Owner defaults to `$(gh repo view --json owner -q .owner.login)` or `SUPER_QA_PROJECT_OWNER` if set.

If no matching project exists, the loop halts with a one-line error and instructs the operator to create one. Do not silently fall back to the repo's primary project — column semantics differ.

## Autonomous mode — NO `AskUserQuestion` mid-loop

super-qa is invoked by an operator who walks away (often overnight, phone-only via chat notifications). The loop must run end-to-end without the orchestrator pausing on `AskUserQuestion`.

**Decide-and-proceed, don't ask, on these classes of issue:**

- Dispatcher / harness script bugs (parser errors, missing `chmod +x`, stale lock files). Fix locally, commit with a `fix(super-qa):` prefix, then continue.
- Missing env auto-loads, leftover MCP zombies, log-dir creation.
- Lint / formatting nits introduced by the worker that block its own commit.

**Still halt on these (real forks):**

- Critical-path HUMAN GATE (dispatcher exit 4) — production money flow red >2 iters. Mandatory halt per skill contract.
- Dispatcher exits 2/3 with a worker error that isn't a script bug (e.g. backend API down, prod 503, auth credential rejected).
- Working tree dirty with changes the orchestrator didn't make (could be user's in-flight work).
- Anything that would delete or rewrite history (`git reset --hard`, force-push, branch deletion).

**Status updates instead of questions:** when you make an autonomous fix, report it in the next status message — what was broken, what you committed (with SHA), and a one-line revert path.

## The queue is the curation surface

`docs/super-qa/queue.md` is a markdown checklist the loop reads from and writes to. Six states per line:

- `[ ]` queued — will be popped next
- `[~]` in-progress (atomic claim during a worker iter; reconciled by the next iter if abandoned)
- `[x]` green spec exists (links to the spec file + iter discovered)
- `[!]` skipped permanently (with reason)
- `[b]` bug found here, children NOT pushed; will be re-tried after fix
- `[?]` flaky — passed on retry, not a bug yet (see flaky policy in `references/iteration-preamble.md`)

The queue is **append-only at the back** during normal exploration. The user can hand-edit it to reorder (turn BFS into DFS for one branch, push something to the front, etc.). The skill respects whatever the user wrote.

The seed is your app's route table (e.g. `client/src/routes.ts` for a TanStack/Next-style app, or `app/` directory entries) — every static route becomes a `[ ]` entry under "Level 0".

## Findings land in clear GitHub Issues, NOT in markdown

Every `[b]` cell triggers the worker to call `scripts/super-qa-file-bug.sh` (consumer-provided helper that adds the issue to the QA project and sets Status to `Bug`). The resulting Project card must be readable from the board without opening it.

**Title format:**

```text
<emoji> <Type> <route?> — <short action/result>
```

Examples:

```text
🐛 Bug /imports — CSV upload fails after submit
🎨 UX /settings/users — Add user button is a no-op
🧪 Tests /orders — missing coverage for failed payment state
📝 Docs /delivery — SPEC section missing for dispatch flow
```

**Required labels:**

- Type: `bug`, `feature`, `ux`, `tests`, `docs`, or `tech-debt`.
- Source: `source:qa`.
- Priority: `priority:high`, `priority:medium`, or `priority:low`.
- Area: `area:<product-area>` when known.
- QA category when relevant: `qa:functional`, `qa:visual`, `qa:network`, `qa:console`, `qa:i18n`, `qa:a11y`, `qa:data`, `qa:testability`.
- Suggested skill owner when helpful: `skill:super-build`, `skill:super-qa`, `skill:super-ux`, or `skill:super-review`.

**Priority words:**

- `priority:high` — urgent, release-blocking, data-loss, security/auth, money, or critical feature broken.
- `priority:medium` — important feature degraded, important edge case, a11y/i18n issue, or user confusion that should be fixed soon.
- `priority:low` — polish, cosmetic, copy, documentation, testability, or cleanup.

See `references/iteration-preamble.md` for the full issue body template, fingerprint/dedupe contract, and filing-script contract.

## What counts as "green" — non-blank guards + forensics

A 200 response with a blank body is a bug, not a green test. Every spec under `e2e/paths/` enforces:

- **No console errors** (`level === 'error'` only).
- **No uncaught page errors** (`page.on('pageerror', ...)`).
- **No 5xx network responses** during the run.
- **No 401/403** on auth-required pages.
- **Type-specific non-blank guard** (e.g. list-page: ≥1 row OR empty-state element; dashboard: ≥1 widget with non-`—` data) — see `references/iteration-preamble.md` "Per-spec expectations" for the canonical list.

If any of those fire → cell is `[b]` → bug filed.

Per-spec forensics captured by a shared report fixture:

| Artifact | Path |
|----------|------|
| Screenshot per `report.step()` | `docs/super-qa/report/<slug>/tc-N/<locale>/*.jpg` |
| Console errors | `docs/super-qa/report/<slug>/tc-N/<locale>/console.log` |
| Page errors | `docs/super-qa/report/<slug>/tc-N/<locale>/pageerrors.log` |
| Network HAR | `docs/super-qa/report/<slug>/tc-N/<locale>/network.har` |
| Network summary | `docs/super-qa/report/<slug>/tc-N/<locale>/network.json` |
| Sentry probe | `docs/super-qa/report/<slug>/tc-N/sentry-events.json` |

Implement a shared fixture (e.g. `e2e/lib/report-fixture.ts`) that captures all of the above. Disk writes are gated behind an env flag (e.g. `SUPER_QA_FORENSICS=1`) that the dispatcher sets per-iter.

## Algorithm

### 1. Determine iteration count
- If user provides arg (e.g. `/super-qa 5`), use that count.
- If `--resume`, start at `max(existing iteration-*.md) + 1`.
- Default: **10 iterations, sequential**. (Concurrency is unsafe — workers share `queue.md`. To run parallel work, branch and run a second orchestrator.)

Notify once: `🐛 super-qa starting — N iterations`.

### 2. For each iteration N (sequential)

**2a. Pre-flight**
- `next_n = max(existing iteration-*.md numbers in docs/super-qa/iter/) + 1`. If none exist, start at 1.
- Confirm working tree is clean (`git status` has no staged/unstaged tracked changes — untracked files are OK). If dirty → halt and notify.
- Verify `docs/super-qa/queue.md` exists. If not, refuse to dispatch and report — the queue is hand-seeded once via this skill's setup.

**2b. Dispatch iteration worker**
```
bash scripts/super-qa-dispatch.sh <next_n>
```

Run with backgrounding (e.g. `run_in_background: true` in your agent harness, or `&` in shell) so the orchestrator can poll. The dispatcher:
- Composes prompt = `references/iteration-preamble.md` + per-iteration footer (iter num, base SHA, mandatory final-commit format).
- Runs the headless worker backend in repo root (no worktree creation).
- Verifies the worker produced a `super-qa: iter N` commit on the current branch.
- Exit codes: `0` (iter complete) / `2` (worker non-zero) / `3` (no done-commit) / `4` (HUMAN GATE) / `5` (WIP-CHECKPOINT — wall-clock hit mid-fix, picks up next iter).

Notify: `🔍 iter N dispatched`.

**2c. Wait, then advance**

Poll the dispatcher. On exit 0:
- Read the close-out commit subject to extract `(X bugs, Y items, Z PRs opened)`.
- Notify: `✅ iter N done — X bugs, Y items processed`.
- Loop to next iteration.

On dispatcher exit 2/3/4/5:
- Notify with `tail -50` of `.planning/super-build-logs/super-qa-iter-N.log`.
- Halt the loop (unless `--continue-on-error` was passed).
- Exit 4 (HUMAN GATE) is mandatory halt regardless — see "critical paths" below.
- Exit 5 (WIP) is **not** halt; the next iter's regression phase finds the red spec and finishes the fix. Continue.

### 3. Termination check (after every iter)

Stop when **any** of:

- Queue has no `[ ]` items left — natural completion.
- User-supplied iteration count `N` is reached (the `N` in `/super-qa N`).
- Dispatcher halt gate fires (HUMAN GATE, dirty tree, dispatcher exit 2/3/4).
- User interrupts.

**Notification trigger (NOT a stop):** after 3 consecutive iters with zero bugs found, send a summary like *"diminishing returns: 3 iters, 0 bugs, N items still in queue — continuing"*. The loop continues until the queue actually drains (or `N` is reached). Zero bugs in recent iters does not prove the unexplored remainder of the queue is clean.

Coverage % is reported every iter as a progress indicator, never a gate.

### 4. Final report

After termination (or on halt):
- Aggregate: total iters, items moved from `[ ]` → `[x]` / `[b]` / `[!]`, total bugs found, total fixed, queue size now.
- Send summary linking to all `docs/super-qa/iter/iteration-*.md` and the current `docs/super-qa/report/QA-REPORT.md`.

## One iteration, in plain English

The worker (per iter N) runs three phases:

**Phase 1 — Regression.** Run all existing specs against everything. Any red spec? Apply the retry policy (re-run once in isolation). Real reds get filed as bugs in `iteration-N.md`, fixed via TDD on a PR branch (never direct-to-main), re-run until green.

**Phase 2 — Explore.** Until the budget is hit (default: 5 cells popped OR 30 min wall-clock):
1. Pop the top `[ ]` item from `queue.md` and mark it `[~]` (atomic claim).
2. Classify via a URL-pattern heuristic (e.g. `references/page-types.md`).
3. Write a spec at `e2e/paths/<slug>.spec.ts` using the test recipe for that type.
4. Run it. Apply retry policy.
   - **Green** → mark `[x]`. Walk the rendered page; push newly-discovered children (links / buttons / dialogs / tabs) to the back of the queue. Cap children pushed per page at **10**.
   - **Red** → file the bug. Mark `[b]`. Do NOT push children. Fix on a PR branch with reviewer skills; never direct-to-main.

**Phase 3 — Report.** Write `docs/super-qa/iter/iteration-N.md` (bugs found, items processed, queue size before/after, coverage snapshot). Regenerate the QA dashboard. Commit `super-qa: iter N (X bugs, Y items, Z PRs opened)`.

See `references/iteration-preamble.md` for the full worker contract (TC schemas, forensics expectations, fix-flow rules, filing semantics, UX visual-review pass, recovery semantics).

## The bug-handling rule (non-blocking)

> When a spec goes red, the worker logs the bug, marks the item `[b]`, **stops expanding into that subtree**, and continues with the rest of the iter's batch. Other siblings in the queue still get tested.

A bug on `/customers/:id` shouldn't block exploration of `/orders` or `/dashboard`. Maximum coverage per iter; one bug never halts the loop.

## Critical paths (HUMAN GATE)

`docs/super-qa/critical-paths.md` is a hand-curated list of money-flow specs that MUST always be green (login, create order, take payment, run cutoff snapshot, send driver email — adapt to your app). The user maintains it; the skill never auto-edits it.

If any critical-path spec has been red for **>2 consecutive iters**, the dispatcher exits **4 (HUMAN GATE)**. The loop halts. A human investigates. This protects production-critical flows from being silently broken by in-flight queue items.

## Multi-step user journeys — `docs/super-qa/flows.md`

`critical-paths.md` is the *money-flow guard* (small, hand-curated, mandatory human gate if red >2 iters). It is **not** a coverage layer.

`docs/super-qa/flows.md` is a separate, broader hand-curated list of *multi-step user journeys* the BFS crawler doesn't reliably exercise on its own. Examples:

- create user → set role → user logs in → sees their assigned region
- import CSV → review preview → confirm import → verify rows in `/orders`
- create order → assign driver → driver email mock fires → order moves to dispatched

The user maintains `flows.md`; the skill never auto-edits it. The worker generates or updates `e2e/flows/<slug>.spec.ts` for each entry. Specs in `e2e/flows/` run alongside `e2e/paths/` in Phase 1 regression — same fixture, same forensics, same `[b]` rules apply.

Flows take precedence over individual route specs when both exist for the same endpoint — a green `e2e/paths/orders.spec.ts` doesn't override a red `e2e/flows/create-order-end-to-end.spec.ts`.

## Coexistence with [[super-build]]

`super-qa-file-bug.sh` files bugs into the QA project's `Bug` column. This is a separate board from super-build's standalone feature queue, so the two skills can run concurrently without column races.

- **Standalone super-build** keeps reading `Ready` from its configured feature project. Untouched by super-qa.
- **Orchestrator-driven super-build in QA-loop mode** drains the `Bug` column on the QA project and moves cards to `Done`. super-orchestrator gates on this column being empty before kicking off the next QA wave.

If you intentionally want a single board for both lanes, set `BUILD_LOOP_PROJECT=$SUPER_QA_PROJECT_NUMBER` and `BUILD_LOOP_QA_MODE=1`. Don't do this by accident — the column semantics differ.

## Test target & safety rails

The default target is the app target configured by `BASE_URL`. Treat production URLs as production and prefer preview/staging for mutating flows.

Safety rails (enforced by `references/iteration-preamble.md`):
- Test user: configure via app env (e.g. `QA_BOT_EMAIL` / `QA_BOT_PASSWORD`).
- All written test data is prefixed `[TEST] ` so it's greppable.
- Sentry tag `source=super-qa` on errors.
- DB resets DISABLED (`RESET_DB=false`) — would wipe prod.
- Email sending mocked or hard-skipped unless `BASE_URL` is a verified staging origin.
- Override the target via `BASE_URL=…` env var on the dispatcher.

## Files

```
super-qa/
├─ SKILL.md                            ← orchestrator instructions (this file)
└─ references/
   └─ iteration-preamble.md            ← worker contract (verbatim prompt)

# Consumer creates these in their project:
scripts/
├─ super-qa-dispatch.sh                ← thin dispatcher
└─ super-qa-file-bug.sh                ← issue filer with dedupe + project promote

docs/super-qa/
├─ README.md                           ← operator's guide
├─ queue.md                            ← THE queue (loop reads/writes)
├─ page-types.md                       ← taxonomy + per-page overrides
├─ critical-paths.md                   ← hand-curated money-flow guard
├─ flows.md                            ← hand-curated multi-step journeys
└─ iter/
   ├─ iteration-1.md                   ← per-iter bug log + report
   └─ ...

e2e/paths/                             ← THE deliverable (one spec per place)
e2e/flows/                             ← multi-step journey specs

docs/super-qa/report/
└─ QA-REPORT.md                        ← regenerated every iter
```

## Recovery / re-entry

- Default: starts a new batch numbered after the last `iteration-*.md`.
- `--resume`: only run iters whose number is greater than `max(existing)`.
- A leftover `wip:` commit on the current branch (from a dispatcher exit 5) is OK — the next iter's regression phase finds the red spec and finishes the fix.

## Invocation patterns

- `super-qa` / `/super-qa` → 10 iters, sequential, default budget
- `/super-qa 5` → 5 iters
- `/super-qa --resume` → continue from `max(existing) + 1`
- `/super-qa --continue-on-error` → don't halt on a single iter failure

## Companion skills

- [[super-build]] executes pending issues from the GitHub Project board. Independent of this skill.
- [[super-orchestrator]] composes super-qa with build/UX/review workflows.
- [[super-truth]] verifies that "green" specs aren't brittle (mutation testing).
