# Workflow

Single source of truth for **process**: planning, documentation policy, git,
build verification, and comments in code. The orchestrator and every subagent
follow it. Nothing here is duplicated into agent files — they reference it.

---

## 1. Planning gate

No implementation code is written before the user approves a plan.

### Triggers

A plan is required for:
- A new feature
- Reworking or refactoring an existing feature
- Any schema change or migration
- Any change to a cross-cutting concern
- Anything touching payments or money — see `PAYMENTS.md`

### Exemptions (no plan needed)

- Typo or comment fix
- Symbol rename with no behavior change
- Adding/adjusting a log line
- Dependency version bump
- Single-line bugfix with an obvious cause

When in doubt, plan. When exempt, say which exemption applies and proceed.

### Required plan contents

Every plan presents these sections, in this order:

```markdown
## Goal
<one or two sentences: what will be true when this is done>

## Blast radius
<see §2 — produced by actually looking, never asserted>

## Approach
<the design: layers touched, new types, data flow, key decisions>

## Files touched
<created / modified / deleted, with a phrase each>

## Risky or irreversible actions
<schema drop, data migration, delete, force-push, external side effect — or "none">

## Test plan
<what gets tested and at which level: unit / integration / controller>

## Documentation
<the docs/ decision for this project — see §3>
```

For payments work, a **dependency failure matrix** is also required. See
`PAYMENTS.md`.

### Approval

Present the plan and stop. Approval is a single gate that covers the plan **and**
the documentation question — never ask them in two round trips.

After approval, execute the full plan without re-asking. Re-open the gate only
on a mid-run block: material divergence from the plan, an unexpected risky
action, or a blocker that cannot be resolved.

---

## 2. Blast radius

The dependency section of a plan is produced **mechanically**, by looking. Never
write "no dependencies" without having run the checks below.

For every symbol, endpoint, or table the change touches:

1. **Callers** — Grep for every changed class, method, and constant. List what
   calls it.
2. **Database** — tables touched, columns added/changed/dropped, the Liquibase
   changeset required, and whether existing rows need backfilling.
3. **Outbound calls** — every external API this path invokes, and what happens
   when each one fails (for money, this is the full matrix in `PAYMENTS.md`).
4. **Events and schedulers** — anything published, consumed, or run on a cron
   that reads or writes the touched data.
5. **Frontend consumers** — for any changed endpoint, request shape, or response
   shape, Grep the frontend service layer and list every hook and component that
   consumes it. A backend contract change is not scoped until its frontend
   consumers are listed.
6. **Sibling projects** — if a shared module (`commons` or similar) is touched,
   list every project in the workspace that depends on it.

State explicitly when a category is genuinely empty: "No schedulers touch this
table." That is a finding. "No dependencies" is not.

---

## 3. Documentation policy

**Ask once per project, not once per task.**

On the first task in a project, ask as part of the plan-approval gate:

> Should I leave a populated `docs/` folder behind? It is written for the next
> agent that works on this project — ADRs, design specs, and per-feature docs.

Record the answer in the project's `CLAUDE.md` under a `Documentation mode:`
line (`on` or `off`), and honor it on every subsequent task without asking
again. Ask again only if the user asks to change it.

### Documentation mode: on

- `architect` writes ADRs to `docs/adr/` and design docs to `docs/design/`.
- `ux-ui-designer` writes specs to `docs/design/<feature>.md`.
- The orchestrator upserts `docs/features/<slug>.md` after every task.

### Documentation mode: off

No files are written under `docs/`. The information is still produced — it is
delivered in the response instead of on disk:

- `architect` reports the decision, its consequences, and rejected alternatives
  in its final message, in the same structure an ADR would have used.
- `ux-ui-designer` returns its spec in its final message; the orchestrator
  passes that spec verbatim in the `frontend-developer` briefing.
- No feature doc is written, and no feature doc is required before reporting
  done.

### Documentation off is not context off

With no `docs/features/` to read, context loading does not disappear — it moves
to the code. Read the existing implementation, tests, and migrations for the
affected area before planning. Skipping context is never the answer.

---

## 4. Git workflow

The **orchestrator** owns the branch and the pull request. Subagents commit
nothing and open nothing. The branch exists before any subagent is spawned; the
PR is opened after all verification passes.

### Starting work

1. If the working tree has uncommitted changes, **stop and surface them**. Never
   stash, reset, or commit someone else's work to get unblocked.
2. `git checkout development`
3. `git pull`
4. `git checkout -b feature/<slug>` — or `hotfix/<slug>` for an urgent
   production fix.

**Both branch types are cut from `development`. Nothing is ever cut from
`main`.** If `development` does not exist, stop and ask which branch to use.

### Committing

Conventional commits:

```
<type>(<scope>): <subject>

<body — what changed and why, when not obvious from the subject>
```

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`, `build`.
Subject in the imperative, lowercase, no trailing period.

Commit in logical units — one commit per coherent change, not one commit per
file and not one commit for an entire multi-layer feature.

### Never on a commit

- **No `Co-Authored-By:` trailer.** Not for Claude, not for any agent, not ever.
- No "Generated with Claude Code" footer or any tool attribution.

### Opening the PR

Push the branch and open a PR **into `development`**:

```markdown
## Summary
<what this delivers, in two or three sentences>

## Changes
<bulleted list of the substantive changes>

## Blast radius
<the dependency section from the approved plan>

## Testing
<what was run and the actual result — see §5>

## Sonar
<clean, or each exclusion with its justification>
```

No tool-attribution footer in the PR body either.

### Hard stops

- **Never merge a PR.** Opening it is where the agent's job ends.
- **Never push directly to `development`** or to `main`.
- **Never force-push** — not to a feature branch, not anywhere.
- Never delete a remote branch.

---

## 5. Build verification

### Evidence before claims

"Tests pass", "the build is clean", and "Sonar is clean" are claims about
command output. Run the command, read the output, and paste the relevant lines
into the report. A claim with no output behind it is not allowed.

If a command could not be run, say so and say why. An unverified change reported
as verified is worse than an unverified change reported honestly.

### Local sibling dependency projects

A microservice often depends on another Spring Boot project in the same
workspace — commonly `commons`, holding shared DTOs, utilities, and base
classes. That artifact may live in a private repository that is unreachable
without a VPN.

When the dependency cannot be resolved and a sibling directory contains that
project's `pom.xml`:

1. Run `mvn clean install -DskipTests` in the sibling directory. **Exactly
   once.**
2. If it succeeds, continue with the main build.
3. If it fails, **stop immediately.**

On failure, report: the command run, the actual error line, what this blocks,
and what was still verified. Then continue with whatever verification remains
possible — unit tests that do not need the dependency, compilation of unaffected
modules — and state plainly what could not be verified.

**Forbidden after a failed sibling build:**

- Retrying the same command
- Trying variant goals (`mvn install`, `-U`, `-o`, `-DskipTests=false`, other
  profiles)
- Editing `settings.xml`, `~/.m2`, or repository credentials
- Adding, removing, or re-versioning the dependency to route around it
- Modifying the sibling project's source to make the build pass, unless the task
  is explicitly about that project

Maximum attempts: **one**. An unresolvable dependency is a fact to report, not a
puzzle to brute-force. Repeated build attempts have burned entire sessions and
resolved nothing.

---

## 6. Comments and documentation in code

Code explains itself through naming and structure. Comments are the exception.

### Forbidden

No comments of any kind — line comments, block comments, Javadoc, JSDoc, or
`.md` files dropped beside the code — in:

- Controllers and operations interfaces
- Entities
- DTOs, commands, and response records
- Mappers
- Repositories
- Configuration classes and property records
- Exceptions
- React components, hooks, and stores

### Allowed

**Javadoc on public service methods.** Nothing else in a service.

- Maximum three lines
- One sentence saying what the method does, then `@param` and `@return`
- No `@throws` walls, no `{@link}` chains, no `<p>` prose, no `@author`,
  no restating the code
- Public methods only — never private helpers

```java
/**
 * Creates a <feature> after validating its business rules.
 *
 * @param command the creation request
 * @return the created <feature>
 */
public <Entity>Response create(Create<Entity>Command command) {
```

**One-line JSDoc on exported frontend service functions** — the Axios call
wrappers only. Nothing on hooks, components, stores, or types.

```ts
/** Fetches a paginated list of <features>. */
export async function get<Features>(params: <Feature>Params) {
```

**Grouped one-line comments in `pom.xml`** — one line above each coherent
dependency group saying what the group is for. Not per dependency, and never a
paragraph.

```xml
<!-- Persistence -->
<dependency>...</dependency>
<dependency>...</dependency>
```

**XML comments in the Sonar suppression file**, per `SONARQUBE.md` — each entry
states its rule and why it is suppressed.

**`// Given` / `// When` / `// Then`** structural markers in tests, per the
testing docs. No other comments in tests — the test method name is the
documentation.

**"Why" comments, rarely.** A comment explaining *why* something non-obvious
exists is allowed: a workaround for a library bug, a deliberate deviation from
convention, a non-obvious regex, an ordering constraint. A comment explaining
*what* the code does is not. If the answer to "why is this here" is not in the
code, one line may say it.

### Not a comment

`@Schema(description = ..., example = ...)` on record components, the
`EXAMPLE` text blocks, and Swagger annotations on operations interfaces are
**generated API documentation**, not comments. They are required by
`backend-developer.md` and are never removed under this policy.
