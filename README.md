# cc-agents

Portable, agent-tool-agnostic toolkit of specialized AI agent configs and
pattern docs for building **full-stack Spring Boot + React** projects with
consistent architecture, testing, and code quality.

Originally a Claude Code subagent suite, now also packaged for
**JetBrains Junie** (CLI and IDE plugin). Each tool gets its own self-contained
folder you can drop into a project.

## Repository layout

```
README.md                          this file
claude/                            Claude Code edition
├── CLAUDE.md                      orchestrator: task handling + routing
└── .claude/
    ├── agents/                    architect, backend-developer,
    │                              frontend-developer, ux-ui-designer
    └── docs/                      testing + SonarQube single source of truth
junie-cli/                         JetBrains Junie — CLI edition
└── .junie/
    ├── guidelines.md              backend engineer playbook
    └── docs/                      testing + SonarQube single source of truth
junie-plugin/                      JetBrains Junie — IDE plugin edition
└── .junie/
    ├── guidelines.md              backend engineer playbook
    └── docs/                      testing + SonarQube single source of truth
```

## What each folder contains

### `claude/` — Claude Code

Full multi-agent dev suite. Work is done by deploying subagents, not
interactive commands.

- **`CLAUDE.md`** — orchestrator that routes tasks to the right subagent
  (architect, backend-developer, frontend-developer, ux-ui-designer) and
  enforces a consistent task-handling flow.
- **`.claude/agents/`** — four specialized subagents:
  - `architect` — ADRs, system design, cross-cutting concerns
  - `backend-developer` — Spring Boot feature work + tests + SonarQube
  - `frontend-developer` — React + TypeScript implementation
  - `ux-ui-designer` — UX flows, wireframes, design tokens (specs only, no code)
- **`.claude/docs/`** — single source of truth for testing patterns and
  SonarQube resolution. Referenced by `backend-developer` on every task.

UI work chains: `ux-ui-designer` writes the spec → `frontend-developer`
implements it. Complex/cross-cutting work enters via `architect` first.

**Use it:** copy `claude/CLAUDE.md` and `claude/.claude/` into your project
root, then open it in Claude Code.

### `junie-cli/` — JetBrains Junie (CLI)

Junie edition of the **backend engineer playbook** only — no orchestrator, no
frontend/UX agents. Same Spring Boot conventions, testing patterns, and
SonarQube standards, adapted to Junie's single-agent model.

- **`.junie/guidelines.md`** — backend engineer playbook (Junie auto-loads it
  as project guidelines).
- **`.junie/docs/`** — same five testing/SonarQube docs as the Claude edition.

**Use it:** copy `junie-cli/.junie/` into your project root, then run Junie CLI
from that project.

### `junie-plugin/` — JetBrains Junie (IDE plugin)

Same content as `junie-cli/`, packaged as a separate drop so the two can evolve
independently (e.g. IDE-aware additions for the plugin edition later).

**Use it:** copy `junie-plugin/.junie/` into your project root, then use the
Junie plugin inside IntelliJ-family IDEs.

## Pattern docs (single source of truth)

Identical content lives in each tool's docs folder so each edition is
self-contained:

| File | Purpose |
|---|---|
| `INITIAL_TEST_PREQUISITES.md` | Test infrastructure setup |
| `UNIT_TESTING.md` | Service-isolation unit-test patterns |
| `INTEGRATION_TESTING.md` | HTTP + DB + WireMock integration patterns |
| `CONTROLLER_TESTING.md` | Web-layer validation-test patterns |
| `SONARQUBE.md` | SonarQube resolution standards |

These docs are never duplicated into the agent/guidelines file — the playbooks
reference them by path.

## Choosing an edition

| You use | Drop in |
|---|---|
| Claude Code (CLI or IDE) | `claude/` |
| JetBrains Junie CLI | `junie-cli/.junie/` |
| JetBrains Junie IDE plugin | `junie-plugin/.junie/` |

## Renaming the repository

The project identity is `cc-agents`. Renaming the **GitHub remote** and the
**on-disk directory** is a manual step (it can't be done safely from inside a
git worktree):

```
# on GitHub: Settings → rename repo, then locally:
git remote set-url origin <new-url>
# optionally rename the working directory
```
