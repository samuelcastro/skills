# Super Family Skills

[![skills.sh](https://skills.sh/b/samuelcastro/skills)](https://skills.sh/samuelcastro/skills)

A family of six [Agent Skills](https://agentskills.io) for running autonomous AI engineering loops — builders, QA crawlers, design reviewers, code reviewers, orchestrators, and a ground-truth verifier — all driven from a GitHub Project board.

The skills are **provider-agnostic** (the reference implementation uses Anthropic's `claude -p`, but any "headless coding agent" backend works: Codex CLI, OpenCode, a self-hosted agent SDK, etc.) and **composable** — each one is single-purpose and they coordinate via a separate orchestrator.

## Skills in this collection

| Skill | What it does |
|---|---|
| [`super-build`](./super-build) | Parallel headless worker dispatcher for GitHub Projects. Reads `Ready` issues, runs each as an isolated worker in its own git worktree (3 in parallel by default), merges back, closes the issue. |
| [`super-qa`](./super-qa) | BFS route-crawler that builds your Playwright spec suite. Walks every route, classifies pages, writes specs, captures evidence, files product-readable issues for failures. |
| [`super-ux`](./super-ux) | Autonomous visual-diff iteration orchestrator. Builder ↔ Reviewer ping-pong that pixel-diffs each page against a wireframe/prototype reference and iterates until visually clean. |
| [`super-review`](./super-review) | PR / code / architecture / release-readiness reviewer. Classifies findings by severity, routes fixes to other skills, produces merge-readiness verdicts with verification evidence. |
| [`super-truth`](./super-truth) | Adversarial verification gate. Takes the output of any producer skill, spawns adversarial sub-agents that try to disprove the claims against reality, returns a confidence score + publish/halt decision. |
| [`super-orchestrator`](./super-orchestrator) | Workflow sequence and loop coordinator. Composes the other skills into a prompt-driven or parameterized sequence with explicit inputs, outputs, goals, loop policy, stop conditions, and human gates. |

## Install

Use the [skills.sh CLI](https://www.skills.sh/docs/cli) to install any individual skill or the whole collection:

```bash
# Install a single skill
npx skills add samuelcastro/skills/super-build

# Install the orchestrator that ties them together
npx skills add samuelcastro/skills/super-orchestrator
```

The CLI fetches the skill folder from this repo and installs it into your local agent's skills directory (e.g. `~/.claude/skills/` for Claude Code). After install, the skill is discoverable by name + description and activates when its description matches the task.

## Layout

This repo follows the [Agent Skills specification](https://agentskills.io/specification):

```
samuelcastro/skills/
├── super-build/
│   ├── SKILL.md
│   ├── references/
│   │   ├── worker-preamble.md
│   │   └── gstack-voting.md
│   └── scripts/
│       └── super-build-dispatch.sh
├── super-orchestrator/
│   ├── SKILL.md
│   ├── references/
│   │   ├── preset-decision-tree.md
│   │   └── visual-review-rubric.md
│   └── templates/
│       └── run-manifest.md
├── super-qa/
│   ├── SKILL.md
│   └── references/
│       └── iteration-preamble.md
├── super-review/
│   └── SKILL.md
├── super-truth/
│   └── SKILL.md
└── super-ux/
    ├── SKILL.md
    └── references/
        ├── builder-prompt.md
        └── reviewer-prompt.md
```

## How they fit together

```
┌────────────────────────────────────────────────────────┐
│              super-orchestrator                        │
│  (composes the others into a sequence per preset)      │
└──────┬─────────────┬─────────────┬─────────────┬───────┘
       │             │             │             │
       ▼             ▼             ▼             ▼
   super-build   super-qa     super-ux     super-review
   (drains      (BFS route   (pixel-diff   (PR/branch
    Ready        crawler +    Builder↔     readiness
    queue in     spec         Reviewer     reviewer +
    parallel     builder)     loop)        router)
    worktrees)
       │             │             │             │
       └─────────────┴──────┬──────┴─────────────┘
                            ▼
                       super-truth
              (adversarial verifier — gates any
               producer skill's claims before publish)
```

**Typical Release-Ready sequence:** `super-build? → super-qa → super-ux → super-review`.

**QA↔Build state-machine loop:** alternates super-build (drain `Bug` column) and super-qa (extend coverage) on a single GitHub Project until both columns are empty.

See [`super-orchestrator/SKILL.md`](./super-orchestrator/SKILL.md) for the full preset menu and the run-manifest contract.

## Conventions

All six skills share a few conventions worth knowing:

- **Provider-agnostic workers.** The reference dispatchers run `claude -p`, but the contract (close-out commit format, exit codes, halt gates, manifest fields) is provider-independent. Swap in Codex CLI, OpenCode, or your own SDK without changing the orchestration.
- **GitHub Projects as the state store.** Curation happens in the `Ready` column (for super-build) or the `Bug`/`Queue` columns (for the QA↔Build loop). Drag a card, the loop picks it up. No external queue.
- **Worktree isolation for parallel workers.** super-build creates a fresh worktree per issue so up to 3 workers can run concurrently without stepping on each other.
- **PRs, never direct-to-main, for fixes.** super-qa fixes always go through a PR with reviewer skills (`/review`, `/plan-eng-review`, `/security-review`). The asymmetric risk of an AI writing both the spec and the fix is too high.
- **Notifications instead of `AskUserQuestion`.** Long autonomous runs never block on user input. Status updates land in your chat channel; the loop halts only on a real human-gate event.
- **Multi-role advisor voting on judgment calls.** Workers consult several advisor perspectives (product / engineering / security / design / QA) on gray decisions, take the majority vote, break ties via smallest blast radius, and stamp the rationale into the commit trailer.

## License

[MIT](./LICENSE).
