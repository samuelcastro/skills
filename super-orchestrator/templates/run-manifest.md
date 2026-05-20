# super-orchestrator run manifest: <run title>

## Request

- **User request:** <paste/summarize original request>
- **Inferred preset:** <Build Queue | Visual Polish | QA → UX Release Sweep | Review Loop | Release Ready | QA↔Build Loop | Custom>
- **Run owner:** <agent/session/user>
- **Created:** <YYYY-MM-DD HH:mm timezone>
- **Repo / branch:** <repo path and branch>

## Goal / done definition

- **Goal:** <one sentence outcome>
- **Done when:**
  - [ ] <done condition 1>
  - [ ] <done condition 2>
  - [ ] <no unresolved blockers except human-gated items>

## Inputs

- **GitHub Project / Ready queue:** <project URL or n/a>
- **Issues / PRs:** <links or n/a>
- **QA artifacts:** <QA-report paths or n/a>
- **UX/reference artifacts:** <screenshots/wireframes/prototypes or n/a>
- **Other context:** <docs/specs/notes>

## Selected sequence

1. <super-build | super-qa | super-ux | super-review>
2. <next step>
3. <next step>

## Loop policy

- **Default:** continue until each selected step reaches its done condition.
- **Repeat scope:** <which steps can repeat>
- **Fix authority:** <direct fix | create GitHub issue | route to another workflow | human gate>
- **No arbitrary cap:** do not stop after N attempts unless the user explicitly set a budget/timebox.

## Halt / human gates

Stop and ask before:

- [ ] production deploy or customer-visible irreversible change
- [ ] destructive database action or production data mutation
- [ ] merge conflict that cannot be resolved safely
- [ ] repeated same failure / no-progress loop
- [ ] ambiguous product/design decision
- [ ] missing external service or broken local environment
- [ ] context/tooling degradation makes verification unreliable

## Step log

### Step 1 — <workflow name>

- **Started:** <time>
- **Input:** <what it consumed>
- **Result:** <done | halted | human-gated>
- **Artifacts:** <issues, commits, reports, screenshots>
- **Next routing:** <next workflow or complete>

### Step 2 — <workflow name>

- **Started:** <time>
- **Input:** <what it consumed>
- **Result:** <done | halted | human-gated>
- **Artifacts:** <issues, commits, reports, screenshots>
- **Next routing:** <next workflow or complete>

## Final status

- **Status:** <done | halted | human-gated>
- **Summary:** <short final summary>
- **Remaining issues:** <links or none>
- **Recommended next action:** <what user/agent should do next>
