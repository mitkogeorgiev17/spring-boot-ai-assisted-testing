# SonarQube Resolution Standards

How to analyze and resolve SonarQube issues. This is the single source of
truth — `backend-developer` follows it directly, non-interactively, as part of
finishing a feature. No approval gates, no menus.

## Core rules

- **Never suppress with `@SuppressWarnings`** — exclusions go in `pom.xml`
  `<properties>` only.
- **Never change code that alters business logic** to satisfy a rule without
  explicitly flagging it in the work report.
- **Run one command at a time** — never chain with `&&`.
- **Always run analysis before and after fixing**; report before/after counts.

## Infrastructure

Required `pom.xml` properties:

```xml
<properties>
    <sonar.host.url>http://localhost:9000</sonar.host.url>
    <sonar.projectKey>my-project</sonar.projectKey>
    <sonar.token>my-token</sonar.token>
</properties>
```

Required Maven plugin (inside `<build><plugins>`):

```xml
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>3.11.0.3922</version>
</plugin>
```

If infrastructure is missing, set it up first; ask the user only for the real
values of `sonar.host.url`, `sonar.projectKey`, and `sonar.token`.

## Workflow

1. **Analyze**: run `mvn sonar:sonar` (single command).
2. **Fetch issues**: extract `sonar.token` / `sonar.host.url` /
   `sonar.projectKey` from `pom.xml`, then:
   ```
   curl -s -u <token>: "<host>/api/issues/search?componentKeys=<key>&statuses=OPEN,CONFIRMED,REOPENED&ps=500"
   ```
3. **Resolve** each issue — fix (preferred) or ignore via `pom.xml`.
4. **Re-analyze**: `mvn sonar:sonar`; compare before/after.

## Fix strategies by rule type

### Code smells (apply the minimal change)
- Unused imports → remove the import
- Unused private fields/methods → remove (verify no reflection usage first)
- Field injection → convert to constructor injection
- Raw types → add generic type parameters
- Missing `final` on locals → add `final`
- String concatenation in logging → parameterized logging

### Bugs
- Null pointer risks → add null checks / `Optional`
- Resource leaks → try-with-resources
- `equals`/`hashCode` contract → implement both consistently

### Security hotspots
- Treat as high-risk: state the risk and recommended fix in the report.
- Never silently rewrite security-relevant code to clear a hotspot — flag it.

## Ignoring issues via pom.xml

Use the `sonar.issue.ignore.multicriteria` mechanism in `<properties>`.

```xml
<!-- SonarQube Issue Exclusions -->
<!-- java:S1135 - TODO comments tracked externally -->
<sonar.issue.ignore.multicriteria>e1,e2</sonar.issue.ignore.multicriteria>
<sonar.issue.ignore.multicriteria.e1.ruleKey>java:S1135</sonar.issue.ignore.multicriteria.e1.ruleKey>
<sonar.issue.ignore.multicriteria.e1.resourceKey>**/*</sonar.issue.ignore.multicriteria.e1.resourceKey>

<!-- java:S1068 - Unused private fields (Lombok false positive) -->
<sonar.issue.ignore.multicriteria.e2.ruleKey>java:S1068</sonar.issue.ignore.multicriteria.e2.ruleKey>
<sonar.issue.ignore.multicriteria.e2.resourceKey>**/*</sonar.issue.ignore.multicriteria.e2.resourceKey>
```

Rules:
1. If `sonar.issue.ignore.multicriteria` does not exist, create it with the
   first entry as `e1`.
2. If it exists, append the next sequential key (`e2`, `e3`, …) and update the
   comma-separated list.
3. Always add an XML comment above each entry stating the rule and why it is
   ignored.
4. `resourceKey` scope options:
   - `**/*` — everywhere (default)
   - `**/test/**` — test files only
   - `**/src/main/java/<pkg path>/**` — a specific package
   - a specific file path — a single file
5. Default to the narrowest scope that resolves the issue; widen only with a
   stated reason.

## Done criteria

- Selected issues fixed or scoped-ignored with justifying comments.
- `pom.xml` updated for any exclusions.
- Re-analysis confirms resolution; no new issues introduced.
- Report lists before/after counts and every exclusion added with its reason.
