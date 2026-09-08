# cc-agents

Claude Code agent suite for building **full-stack Spring Boot + React** projects
with consistent architecture, testing, code quality, and workflow.

## Repository layout

```
README.md                          this file
claude-code/                       drop-in Claude Code setup
├── CLAUDE.md                      orchestrator: task handling + routing
└── .claude/
    ├── agents/                    architect, backend-developer,
    │                              frontend-developer, ux-ui-designer
    └── docs/                      process, money, testing, SonarQube
```

**Use it:** copy `claude-code/CLAUDE.md` and `claude-code/.claude/` into your
project root, then open it in Claude Code.

## What's in it

Work is done by deploying subagents, not interactive commands.

- **`CLAUDE.md`** — orchestrator that routes tasks to the right subagent and
  enforces a consistent task-handling flow. Holds the project's
  `Documentation mode` setting.
- **`.claude/agents/`** — four specialized subagents:
  - `architect` — design decisions, cross-cutting concerns, money designs
  - `backend-developer` — Spring Boot feature work + tests + SonarQube
  - `frontend-developer` — React + TypeScript implementation
  - `ux-ui-designer` — UX flows, wireframes, design tokens (specs only, no code)
- **`.claude/docs/`** — single source of truth. Nothing in it is duplicated into
  the agent files; the playbooks reference it by path.

UI work chains: `ux-ui-designer` writes the spec → `frontend-developer`
implements it. Complex, cross-cutting, or money work enters via `architect`
first.

## Docs (single source of truth)

| File | Purpose |
|---|---|
| `WORKFLOW.md` | Planning gate, docs policy, git flow, build verification, comment policy |
| `PAYMENTS.md` | Money handling, dependency failure matrices, payment testing |
| `INITIAL_TEST_PREQUISITES.md` | Test infrastructure setup |
| `UNIT_TESTING.md` | Service-isolation unit-test patterns |
| `INTEGRATION_TESTING.md` | HTTP + DB + WireMock integration patterns |
| `CONTROLLER_TESTING.md` | Web-layer validation-test patterns |
| `SONARQUBE.md` | SonarQube resolution standards |

## How work runs

**Plan first.** Every feature and rework is planned and approved before any code
is written. The plan states its goal, its **blast radius** — callers, schema,
outbound calls, schedulers, frontend consumers, sibling projects, all found by
grepping rather than assumed — its approach, its risks, and its test plan.

**Documentation is opt-in.** On the first task in a project you are asked once
whether to leave a populated `docs/` folder behind for the next agent. The
answer is recorded in `CLAUDE.md` and honored from then on. With docs off, ADRs
and design specs are still produced — they are delivered in the response instead
of written to disk.

**Money gets its own rules.** Anything touching payments routes through
`architect` and carries a dependency failure matrix: for every outbound call,
what happens on decline, `4xx`, `5xx`, duplicate, malformed body, and — the one
that matters — timeout. A test per row.

**Git flow is fixed.** Pull `development`, cut `feature/<slug>` or
`hotfix/<slug>` from it (never from `main`), commit in conventional-commit
units, open a PR into `development`. The agent never merges, never force-pushes,
and never pushes to `development` directly. **No `Co-Authored-By` trailer and no
tool-attribution footer**, on commits or PRs.

**Claims need evidence.** "Tests pass" means pasted `mvn verify` output. A local
sibling dependency project (`commons` and friends) gets exactly **one**
`mvn clean install` attempt — if it fails, that is reported and the run stops
looking for a way around it.

**Comments stay out of the code.** No comments in controllers, entities, DTOs,
mappers, config, or components. Allowed: thin Javadoc on public service methods,
one-line JSDoc on exported frontend service functions, grouped one-liners in
`pom.xml`, Sonar suppression justifications, Given/When/Then in tests, and rare
"why" lines where something genuinely non-obvious needs justifying.

## Renaming the repository

The project identity is `cc-agents`. Renaming the **GitHub remote** and the
**on-disk directory** is a manual step (it can't be done safely from inside a
git worktree):

```
# on GitHub: Settings → rename repo, then locally:
git remote set-url origin <new-url>
# optionally rename the working directory
```
