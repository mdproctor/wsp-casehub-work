# Blackboard → Planning Migration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use executing-plans to
> implement this plan task-by-task. Each task uses ide-tooling for
> structural editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/engine#811 — Complete blackboard → planning migration
**Issue group:** child issues per repo (created in Task 1)

**Goal:** Replace every reference to `casehub-engine-blackboard` across 11 repos with `casehub-engine-planning`, verified by local builds and CI.

**Architecture:** Pure rename migration. The planning module is source-compatible — same class names, same method signatures, different package (`io.casehub.blackboard.*` → `io.casehub.engine.planning.*`). No behavioural changes. App repos have POM-only dependencies (no Java imports from blackboard).

**Tech Stack:** Maven (pom.xml edits), Java 21 imports (casehub-work only), git worktree slots

## Global Constraints

- All repos use `${casehub.version}` from the parent BOM for engine artifact versions
- `casehub-engine-planning` 0.2-SNAPSHOT is already published to GitHub Packages (verified 2026-07-28)
- Push order matters: parent → engine → casehub-work → app repos (SNAPSHOT dependency chain)
- Every commit references engine#811

---

### Task 0: Create child issues

**Files:** None (GitHub API only)

- [ ] **Step 1: Create child issues for each impacted repo**

Create one issue per repo in the appropriate GitHub repo, all referencing engine#811 as parent. Use this title pattern: `chore: migrate casehub-engine-blackboard → casehub-engine-planning`

Repos needing issues:
- `casehubio/parent`
- `casehubio/work`
- `casehubio/aml`
- `casehubio/life`
- `casehubio/soc`
- `casehubio/iot`
- `casehubio/ops`
- `casehubio/fsitrading`
- `casehubio/quarkmind`
- `casehubio/scaffold`

Body for each: `Part of casehubio/engine#811. Swap Maven dependency casehub-engine-blackboard → casehub-engine-planning.`

For casehubio/work, add: `Also migrates Java imports in engine-adapter module (3 prod + 4 test files).`

For casehubio/engine, add a cleanup issue: `Delete orphaned blackboard/ directory (not in reactor since engine#60).`

Record all issue numbers — they'll be used in commit messages.

---

### Task 1: Create work-slot with all impacted repos

**Files:** None (worktree setup)

- [ ] **Step 1: Create the worktree slot**

```
/work-slot create for engine#811
```

The slot needs these repos: parent, engine, work, aml, life, soc, iot, ops, fsitrading, quarkmind, scaffold.

- [ ] **Step 2: Open IntelliJ workspace with all repos**

```
ide_open_workspace with modules:
  - <slot>/parent
  - <slot>/engine
  - <slot>/work
  - <slot>/aml
  - <slot>/life
  - <slot>/soc
  - <slot>/iot
  - <slot>/ops
  - <slot>/fsitrading
  - <slot>/quarkmind
  - <slot>/scaffold
```

Wait for indexing to complete before proceeding.

---

### Task 2: Migrate parent BOM

**Files:**
- Modify: `<slot>/parent/pom.xml:500-504`

**Produces:** BOM entry for `casehub-engine-planning` with `${casehub.version}` — all downstream repos resolve version from this.

- [ ] **Step 1: Replace the BOM entry**

In `<slot>/parent/pom.xml`, replace:
```xml
      <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-engine-blackboard</artifactId>
        <version>${casehub.version}</version>
      </dependency>
```
with:
```xml
      <dependency>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-engine-planning</artifactId>
        <version>${casehub.version}</version>
      </dependency>
```

- [ ] **Step 2: Install to local .m2**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <slot>/parent/pom.xml install -N
```

Expected: BUILD SUCCESS. The `-N` flag installs only the parent POM (no modules).

- [ ] **Step 3: Commit**

```bash
git -C <slot>/parent add pom.xml
git -C <slot>/parent commit -m "chore: BOM — replace casehub-engine-blackboard with casehub-engine-planning

Refs casehubio/engine#811, Refs #<parent-issue>"
```

---

### Task 3: Migrate casehub-work engine-adapter

**Files:**
- Modify: `<slot>/work/engine-adapter/pom.xml:14,19`
- Modify: `<slot>/work/engine-adapter/src/main/java/io/casehub/work/engine/HumanTaskScheduleHandler.java:23-25`
- Modify: `<slot>/work/engine-adapter/src/main/java/io/casehub/work/engine/WorkItemLifecycleAdapter.java:20-22`
- Modify: `<slot>/work/engine-adapter/src/main/java/io/casehub/work/engine/PlanItemCompletionApplier.java:30-31`
- Modify: `<slot>/work/engine-adapter/src/test/java/io/casehub/work/engine/HumanTaskScheduleHandlerTest.java:22-23`
- Modify: `<slot>/work/engine-adapter/src/test/java/io/casehub/work/engine/HumanTaskScheduleHandlerAtomicityTest.java:20-21`
- Modify: `<slot>/work/engine-adapter/src/test/java/io/casehub/work/engine/WorkItemLifecycleAdapterTest.java:20-21`
- Modify: `<slot>/work/engine-adapter/src/test/java/io/casehub/work/engine/recovery/HumanTaskRecoveryServiceTest.java:19-20`

**Consumes:** Parent BOM with `casehub-engine-planning` managed (Task 2)

- [ ] **Step 1: Swap the POM dependency**

In `<slot>/work/engine-adapter/pom.xml` line 19, replace:
```xml
            <artifactId>casehub-engine-blackboard</artifactId>
```
with:
```xml
            <artifactId>casehub-engine-planning</artifactId>
```

- [ ] **Step 2: Update the POM description**

In `<slot>/work/engine-adapter/pom.xml` line 14, replace:
```xml
    <description>Bridge between casehub-work WorkItemLifecycleEvent and CaseHub engine blackboard PlanItem lifecycle</description>
```
with:
```xml
    <description>Bridge between casehub-work WorkItemLifecycleEvent and CaseHub engine planning PlanItem lifecycle</description>
```

- [ ] **Step 3: Swap imports in all 7 Java files**

Use `ide_replace_text_in_file` on each file in the engine-adapter module. Two replacements per file:

Replace `io.casehub.blackboard.plan` → `io.casehub.engine.planning.plan` in:
- `src/main/java/io/casehub/work/engine/HumanTaskScheduleHandler.java` (lines 23-24)
- `src/main/java/io/casehub/work/engine/WorkItemLifecycleAdapter.java` (lines 20-21)
- `src/main/java/io/casehub/work/engine/PlanItemCompletionApplier.java` (line 30)
- `src/test/java/io/casehub/work/engine/HumanTaskScheduleHandlerTest.java` (line 22)
- `src/test/java/io/casehub/work/engine/HumanTaskScheduleHandlerAtomicityTest.java` (line 20)
- `src/test/java/io/casehub/work/engine/WorkItemLifecycleAdapterTest.java` (line 20)
- `src/test/java/io/casehub/work/engine/recovery/HumanTaskRecoveryServiceTest.java` (line 19)

Replace `io.casehub.blackboard.registry` → `io.casehub.engine.planning.registry` in:
- `src/main/java/io/casehub/work/engine/HumanTaskScheduleHandler.java` (line 25)
- `src/main/java/io/casehub/work/engine/WorkItemLifecycleAdapter.java` (line 22)
- `src/main/java/io/casehub/work/engine/PlanItemCompletionApplier.java` (line 31)
- `src/test/java/io/casehub/work/engine/HumanTaskScheduleHandlerTest.java` (line 23)
- `src/test/java/io/casehub/work/engine/HumanTaskScheduleHandlerAtomicityTest.java` (line 21)
- `src/test/java/io/casehub/work/engine/WorkItemLifecycleAdapterTest.java` (line 21)
- `src/test/java/io/casehub/work/engine/recovery/HumanTaskRecoveryServiceTest.java` (line 20)

- [ ] **Step 4: Build engine-adapter to verify**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <slot>/work/pom.xml install -pl engine-adapter -am
```

Expected: BUILD SUCCESS. The `-am` flag builds dependencies (core, api) that engine-adapter needs.

- [ ] **Step 5: Commit**

```bash
git -C <slot>/work add engine-adapter/
git -C <slot>/work commit -m "chore: migrate engine-adapter from casehub-engine-blackboard to casehub-engine-planning

Swap Maven dependency and Java imports (3 prod + 4 test files).
Source-compatible — same class names, same method signatures, new package.

Refs casehubio/engine#811, Refs #<work-issue>"
```

---

### Task 4: Migrate app repos (POM-only)

**Files:**
- Modify: `<slot>/aml/app/pom.xml` (line ~140)
- Modify: `<slot>/life/app/pom.xml` (line ~129)
- Modify: `<slot>/soc/app/pom.xml` (line ~123)
- Modify: `<slot>/iot/pom.xml` (line ~123)
- Modify: `<slot>/iot/webapp/pom.xml` (line ~75)
- Modify: `<slot>/ops/app/pom.xml` (line ~35)
- Modify: `<slot>/fsitrading/app/pom.xml` (line ~128)
- Modify: `<slot>/quarkmind/pom.xml` (lines 109-115)

**Consumes:** Parent BOM with `casehub-engine-planning` managed (Task 2)

- [ ] **Step 1: Swap artifact ID in each pom.xml**

For each repo, use `ide_replace_text_in_file` to replace `casehub-engine-blackboard` → `casehub-engine-planning` in the pom.xml files listed above.

- [ ] **Step 2: Fix quarkmind stale comment**

In `<slot>/quarkmind/pom.xml` lines 109-111, replace:
```xml
        <!-- casehub-engine-blackboard: WritablePanelImpl, CaseContextImpl — CDI beans for
             the blackboard layer. Engine runtime satisfies the SPI dependencies (EventLogRepository,
             JobScheduler etc.) that previously blocked this dependency. -->
```
with:
```xml
        <!-- casehub-engine-planning: planning layer CDI beans. Engine runtime satisfies
             the SPI dependencies (EventLogRepository, JobScheduler etc.). -->
```

- [ ] **Step 3: Verify each repo compiles**

Run `mvn compile` for each app repo to verify dependency resolution:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <slot>/aml/pom.xml compile -pl app
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <slot>/life/pom.xml compile -pl app
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <slot>/soc/pom.xml compile -pl app
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <slot>/iot/pom.xml compile -pl webapp
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <slot>/ops/pom.xml compile -pl app
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <slot>/fsitrading/pom.xml compile -pl app
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <slot>/quarkmind/pom.xml compile
```

Expected: BUILD SUCCESS for each. If any fails, the planning artifact is not resolving — check parent BOM install (Task 2).

- [ ] **Step 4: Commit each repo**

One commit per repo, each referencing engine#811 and the repo's child issue:

```bash
git -C <slot>/aml add app/pom.xml && git -C <slot>/aml commit -m "chore: migrate casehub-engine-blackboard → casehub-engine-planning

Refs casehubio/engine#811, Refs #<aml-issue>"
```

Repeat for life, soc, iot, ops, fsitrading, quarkmind (for iot, stage both `pom.xml` and `webapp/pom.xml`; for quarkmind, the comment fix is included).

---

### Task 5: Migrate scaffold template

**Files:**
- Modify: `<slot>/scaffold/pom.xml` (line ~65)

- [ ] **Step 1: Swap artifact ID**

Replace `casehub-engine-blackboard` → `casehub-engine-planning` in `<slot>/scaffold/pom.xml`.

- [ ] **Step 2: Commit**

```bash
git -C <slot>/scaffold add pom.xml
git -C <slot>/scaffold commit -m "chore: scaffold template — casehub-engine-blackboard → casehub-engine-planning

New projects will generate with the correct dependency.

Refs casehubio/engine#811, Refs #<scaffold-issue>"
```

---

### Task 6: Delete orphaned blackboard directory in engine

**Files:**
- Delete: `<slot>/engine/blackboard/` (entire directory)

- [ ] **Step 1: Verify blackboard is not in the reactor**

```bash
grep "<module>blackboard</module>" <slot>/engine/pom.xml
```

Expected: no output (blackboard is not a module).

- [ ] **Step 2: Delete the orphaned directory**

```bash
rm -rf <slot>/engine/blackboard/
```

- [ ] **Step 3: Commit**

```bash
git -C <slot>/engine add -A blackboard/
git -C <slot>/engine commit -m "chore: delete orphaned blackboard/ directory

Module was superseded by casehub-engine-planning in engine#60.
Not in the reactor — directory was left behind during the rename.

Refs casehubio/engine#811, Closes #<engine-cleanup-issue>"
```

---

### Task 7: Push in dependency order

- [ ] **Step 1: Push parent (BOM first)**

```bash
git -C <slot>/parent push origin main
```

Wait for CI to publish the BOM SNAPSHOT.

- [ ] **Step 2: Push engine (cleanup)**

```bash
git -C <slot>/engine push origin main
```

- [ ] **Step 3: Push casehub-work**

```bash
git -C <slot>/work push origin main
```

Wait for CI to publish the engine-adapter SNAPSHOT.

- [ ] **Step 4: Push app repos and scaffold (parallel)**

```bash
git -C <slot>/aml push origin main
git -C <slot>/life push origin main
git -C <slot>/soc push origin main
git -C <slot>/iot push origin main
git -C <slot>/ops push origin main
git -C <slot>/fsitrading push origin main
git -C <slot>/quarkmind push origin main
git -C <slot>/scaffold push origin main
```

- [ ] **Step 5: Verify AML CI is green**

```bash
gh run list --repo casehubio/aml --limit 1 --json status,conclusion --jq '.[0]'
```

Expected: `"conclusion": "success"`. This was the original failure that surfaced the issue.

---

### Task 8: Update engine#811 and close child issues

- [ ] **Step 1: Close each child issue with the commit SHA**

For each repo, close its child issue:
```bash
gh issue close <N> --repo casehubio/<repo> --comment "Landed as <SHA> on main."
```

- [ ] **Step 2: Close the root issue**

```bash
gh issue close 811 --repo casehubio/engine --comment "Migration complete. All 11 repos migrated from casehub-engine-blackboard to casehub-engine-planning. AML CI green."
```

- [ ] **Step 3: Update casehub-work CLAUDE.md**

Update the engine-adapter section in CLAUDE.md to reference `casehub-engine-planning` instead of `casehub-engine-blackboard`.
