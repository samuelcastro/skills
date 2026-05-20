# super-orchestrator preset decision tree

Use this when a user asks super-orchestrator to choose a sequence from a natural-language request.

## Principle

Suggest the smallest preset that satisfies the request. Do not force the full Release Ready sequence when the user only asked for build, QA, UX polish, or review.

## Decision tree

1. **Does the user mention building, ready issues, GitHub Project queue, tickets, or implementation work?**
   - Yes → suggest **Build Queue** (super-build).
   - If the user also says release-ready, continue evaluating later steps.

2. **Does the user mention bug bash, functionality, testing, QA, verify/fix, broken pages, or "make sure it works"?**
   - Yes → include **super-qa**.

3. **Does the user mention UI polish, screenshots, wireframes, design match, visual fidelity, layout, pixels, or UX?**
   - Yes → include **super-ux**.

4. **Does the user mention PR readiness, code review, architecture, security review, merge readiness, or final judgment?**
   - Yes → include **super-review**.

5. **Does the user say "release ready", "production ready", "ship ready", or similar broad language?**
   - Suggest **Release Ready**: `super-build? → super-qa → super-ux → super-review`.
   - Check GitHub `Ready` queue first:
     - if relevant cards exist, include super-build;
     - if queue is empty, skip super-build and start at super-qa.

6. **Does the user give an explicit sequence?**
   - Use the explicit sequence unless it conflicts with safety or a required prerequisite.

## Confirmation behavior

Confirm the inferred sequence when:

- multiple presets match;
- the run will push commits, create issues, or change project board state;
- the sequence includes destructive/risky actions;
- the goal/done definition is unclear.

Do not ask for confirmation when:

- the next action is read-only inspection;
- the prompt clearly maps to one low-risk preset;
- the user has already provided explicit sequence and scope.

## Output routing

- Findings from **super-qa** can be fixed directly if safe/in scope, and also logged in local markdown state. Create GitHub issues/cards for bugs that should survive beyond the current run.
- Findings from **super-ux** can be fixed directly when the reference target is clear. Create/surface issues for subjective visual decisions.
- Findings from **super-review** should be routed to super-build, super-qa, or super-ux rather than silently fixed by the reviewer.
