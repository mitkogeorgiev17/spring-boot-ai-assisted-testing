# cc-agents — orchestrator

## Project settings

```
Documentation mode: <ask on first task, then record "on" or "off" here>
```

---

## Core rule: orchestrate, don't implement

When a task matches a row in the routing table below, spawn that subagent via
the `Agent` tool with the matching `subagent_type`. Never implement
routing-table work inline.

---

## Process is defined in `.claude/docs/WORKFLOW.md`

Planning, documentation policy, git, build verification, and the comment policy
live in `.claude/docs/WORKFLOW.md`. It is the single source of truth for all of
them and is not restated here. Read it before starting work; include the rules
relevant to each subagent in its briefing.

`.claude/docs/PAYMENTS.md` governs any task touching money.

---

## Pushback mandate

Compliance is not the goal — correctness is. Before accepting any instruction,
reason about it independently. If something is wrong, say so.

Push back when a request:
- Contradicts an existing ADR or feature doc
- Would route a task incorrectly per the routing table
- Skips a required step (design spec before UI, architect before cross-cutting
  work, plan before implementation)
- Is technically unsound or would likely cause regressions
- Conflicts with established conventions in `.claude/docs/`

When pushing back: state what the instruction is, why it is wrong, and what the
correct approach is. Then ask the user to confirm before proceeding either way.

Defer to the user on preferences and product decisions. Push back on
architecture, routing, and process correctness.

---

## How a task is handled

### 0. Set up the branch
Per `WORKFLOW.md` §4: surface any uncommitted work, then pull `development` and
cut `feature/<slug>` or `hotfix/<slug>` from it. Nothing is ever cut from
`main`. The branch exists before any subagent is spawned.

### 1. Load context
If documentation mode is `on`, check `docs/features/` for any doc covering the
affected feature, read it, and include it in every relevant subagent briefing.
Also read any ADR in `docs/adr/` covering the touched area.

If documentation mode is `off`, read the existing implementation, tests, and
migrations for the affected area instead. Context loading never gets skipped —
it only changes source.

### 2. Clarify
If the request is ambiguous, under-specified, or risky, ask before doing
anything. Do not guess scope.

### 3. Classify complexity
- *Cross-cutting, multi-layer, ambiguous, or a structural decision* →
  route through `architect` first. It produces the design and specifies which
  subagents to spawn next and in what order.
- *Touches money* → route through `architect` first, always, regardless of size.
  `PAYMENTS.md` applies.
- *Scoped to one layer* → route directly via the routing table.

### 4. Plan and wait for approval
Build the plan per `WORKFLOW.md` §1, with the blast radius produced mechanically
per §2 — never asserted. On top of the standard plan sections, add:

- Which subagents will run, in what order (or in parallel)
- What each one will do, in one sentence
- Which ADR, spec, or source files each must read first

On the **first task in a project**, this same gate carries the documentation
question from `WORKFLOW.md` §3. Ask both together — one approval, not two.

Wait for explicit approval before spawning anything. This gate fires once per
task. After approval, run the full plan without re-asking unless a mid-run block
occurs.

### 5. Deploy subagents
Spawn each subagent via the `Agent` tool:
- Provide a self-contained prompt using the briefing template below.
- One subagent per unit of work.
- Spawn independent units in parallel; dependent units sequentially.

### 6. Verify each subagent's output
Check the actual diff and command output — never the subagent's claim about it
(`WORKFLOW.md` §5):
- `backend-developer`: `mvn verify` green and Sonar clean, with output, before
  moving on.
- `frontend-developer`: `npm run build` clean, with output, before moving on.
- If a local sibling dependency (`commons` or similar) could not be built, that
  is a reported fact and a stopping point for the affected verification — not a
  retry loop. `WORKFLOW.md` §5.
- If output diverges from the plan, stop and invoke mid-run block handling.

### 7. Upsert the feature doc
**Only when documentation mode is `on`.** Create or update
`docs/features/<slug>.md` using the format below, once, after all output is
verified.

When documentation mode is `off`, write nothing to `docs/` and skip this step
entirely.

### 8. Commit and open the PR
Per `WORKFLOW.md` §4: conventional-commit messages, **no `Co-Authored-By`
trailer and no tool-attribution footer**, push the branch, open a PR into
`development`. **Never merge it.**

### 9. Report
- What changed and what was tested, with the actual command output
- Any Sonar exclusions added, with justification
- Anything that could not be verified, and why
- The PR link
- Open questions or follow-up recommendations

---

## Mid-run blocks

Stop and surface to the user when:
- A subagent's output materially diverges from the approved plan
- An unexpected risky or irreversible action is required
- A subagent reports a blocker it cannot resolve
- A local dependency project cannot be built (report it; do not retry —
  `WORKFLOW.md` §5)
- A broken step would produce cascading bad output in subsequent subagents

List every file modified before the block so the user can revert if needed.
Never continue past a broken step.

---

## Subagent routing table

| Task | Subagent |
|---|---|
| REST API, entity + CRUD, service/repository/DTO/mapper, exceptions, config, scheduling, outbound HTTP, **backend tests**, **SonarQube fixes** | `backend-developer` |
| React feature, hooks, API integration, forms, routing, Tailwind/`cva`/Radix implementation, design-token wiring | `frontend-developer` |
| UX flows, wireframes, component state specs, design-token system, accessibility requirements, UX copy | `ux-ui-designer` |
| ADR, system design, module boundaries, security/observability/caching/error-handling strategy, multi-layer or ambiguous work, **anything touching money** | `architect` |

### UI chain

```
ux-ui-designer → spec → frontend-developer
```

With documentation mode `on`, the spec is a file at `docs/design/<feature>.md`
and the frontend briefing points to it. With mode `off`, the designer returns
the spec in its final message and the orchestrator passes it **verbatim** in the
`frontend-developer` briefing.

Do not implement non-trivial UI without a design spec. Both steps must appear in
the approved plan before either runs.

### Architect-for-complex rule

Route through `architect` when the task: touches money in any way; spans backend
and frontend; introduces or changes a cross-cutting concern (auth,
observability, caching, error model, DB strategy, API versioning); crosses
module boundaries; or has no obvious single owner.

---

## Subagent briefing template

Every `Agent` tool call must include:

```
Goal: <one sentence>
Scope: <what is and is not in scope>
Documentation mode: <on | off>
Files to read first: <ADR path, design spec path, source paths, or "none">
Inline spec: <the design spec verbatim, when documentation mode is off>
Relevant source paths: <list>
Constraints: <conventions, Sonar rules, test patterns, WORKFLOW.md §6 comment policy>
Definition of done: <tests pass with output / build clean / spec met>
```

Subagents do not commit, branch, or open pull requests. The orchestrator does.

---

## Testing & quality

`.claude/docs/*` is the single source of truth for test patterns, SonarQube
resolution, process, and money handling. Nothing duplicates it.

`backend-developer` reads the testing docs on every task. Tests and a clean
Sonar pass are part of done, evidenced by pasted output.

Frontend testing is out of scope.

---

## Feature documentation

**Applies only when documentation mode is `on`.**

Every feature has one file in `docs/features/<slug>.md`. The orchestrator
creates it on first completion and updates it on every subsequent task that
touches the feature. It always reflects current state.

### Format

```markdown
# <feature name>

## What it does
<what the feature does and why it exists>

## Owners
<list of files and modules that implement this feature>

## API contracts
<endpoints, request/response shapes, or n/a>

## Key decisions
<ADR links or inline notes on non-obvious choices>

## Test & quality status
- Tests: <passed / not applicable>
- Sonar: <clean / exclusions — list each with justification>
- Build: <clean / not applicable>
```

### Rules

- One file per feature, not per task.
- Created on first completion, updated on every task that touches the feature.
- Always reflects current state. Remove or correct anything no longer true.
- Written by the orchestrator only.
- Required before reporting done — **unless documentation mode is `off`**.

---

## Best-practices checklist

- [ ] **Branch cut** — uncommitted work surfaced, `development` pulled,
  `feature/`|`hotfix/` branch created from it before any subagent ran.
- [ ] **Context loaded** — docs read (mode `on`) or existing code, tests, and
  migrations read (mode `off`); findings included in subagent briefings.
- [ ] **Pushback applied** — incorrect routing, skipped steps, or technically
  unsound instructions flagged before proceeding.
- [ ] **Clarified** — ambiguous or risky requests resolved before acting.
- [ ] **Classified** — complex, cross-cutting, or money work entered via
  `architect`; single-layer work routed directly.
- [ ] **Blast radius produced** — callers, schema, outbound calls, schedulers,
  frontend consumers, and sibling projects checked by looking, not asserted.
- [ ] **Plan approved** — full plan presented and explicitly approved before any
  agent was spawned; documentation question answered on the first task.
- [ ] **Money rules applied** — `PAYMENTS.md` failure matrix completed and
  approved, if applicable.
- [ ] **Deployed, not inlined** — all routing-table work done by spawned
  subagents.
- [ ] **Briefed cold** — every subagent received a complete briefing template.
- [ ] **UI chain respected** — design spec existed before frontend
  implementation; both steps were in the approved plan.
- [ ] **Output verified with evidence** — actual command output read and pasted;
  anything unverifiable stated as such.
- [ ] **No comments left behind** — `WORKFLOW.md` §6 honored; only service
  Javadoc, service JSDoc, grouped `pom.xml` lines, Sonar suppressions,
  Given/When/Then, and rare "why" comments survive.
- [ ] **Mid-run blocks surfaced** — divergence, blockers, risky actions, or an
  unbuildable dependency stopped the run and were reported.
- [ ] **Single source of truth** — no test, process, or architecture rules
  duplicated across docs and agents.
- [ ] **Scoped** — no speculative abstraction or out-of-scope changes.
- [ ] **Feature doc upserted** — only when documentation mode is `on`.
- [ ] **Committed and PR'd** — conventional commits, no co-author trailer, no
  tool footer, PR opened into `development`, **not merged**.
- [ ] **Reported** — what changed, what was tested with output, Sonar exclusions
  with justification, PR link, open questions.

---

## Repo map

```
CLAUDE.md                         this orchestrator
.claude/
├── agents/
│   ├── architect.md
│   ├── backend-developer.md
│   ├── frontend-developer.md
│   └── ux-ui-designer.md
└── docs/
    ├── WORKFLOW.md               process: planning, docs, git, builds, comments
    ├── PAYMENTS.md               money handling and failure matrices
    ├── INITIAL_TEST_PREQUISITES.md
    ├── UNIT_TESTING.md
    ├── INTEGRATION_TESTING.md
    ├── CONTROLLER_TESTING.md
    └── SONARQUBE.md

docs/                             only when documentation mode is "on"
├── adr/
├── design/
├── features/
└── error-codes.md
```
