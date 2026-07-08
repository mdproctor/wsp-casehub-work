# Relocate Work Adapter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #290 — refactor: relocate HumanTask adapter from casehub-engine to casehub-work
**Issue group:** #290

**Goal:** Move the `casehub-engine-work-adapter` module from the engine repo to work as `casehub-work-engine-adapter`, with correct package naming and no stale references.

**Architecture:** Three-phase migration: (1) rename packages in-place in engine via IntelliJ, (2) create the new module in work and copy files across, delete from engine, update all metadata, (3) fix broken `WorkItemCallerRef`, update docs.

**Tech Stack:** Java 21, Quarkus 3.32.2, Maven, IntelliJ refactoring

## Global Constraints

- All code operations via IntelliJ MCP — no bash cp/mv/rm on source files
- Engine project: `/Users/mdproctor/claude/casehub/engine`
- Work project: `/Users/mdproctor/claude/casehub/work`
- Parent BOM project: `/Users/mdproctor/claude/casehub/parent`
- IntelliJ workspace: both repos already open via `ide_open_workspace`
- Pre-release — no backward compat shims needed

---

### Task 1: Package Rename in Engine (IntelliJ)

Rename `io.casehub.workadapter` → `io.casehub.work.engine` while all code is in one project. IntelliJ handles all internal cross-references atomically.

**Files:**
- Rename (IntelliJ): all 12 source files in `engine/work-adapter/src/main/java/io/casehub/workadapter/`
- Rename (IntelliJ): all 10 test files in `engine/work-adapter/src/test/java/io/casehub/workadapter/`
- Modify: `engine/work-adapter/src/test/resources/application.properties` (package ref in `quarkus.arc.selected-alternatives`)
- Modify: `engine/runtime/src/main/java/io/casehub/engine/internal/startup/ActionGateDeploymentHealthCheck.java:66` (`Class.forName` string literal)

**Interfaces:**
- Produces: all adapter code under `io.casehub.work.engine` package, ready for Phase 2 move

- [ ] **Step 1: Rename the main package via IntelliJ**

Use `ide_refactor_rename` on the `io.casehub.workadapter` package directory. This renames all 11 files in the main package (excluding the `recovery` subpackage).

```
ide_refactor_rename:
  file: work-adapter/src/main/java/io/casehub/workadapter
  targetType: file
  newName: engine
  project_path: /Users/mdproctor/claude/casehub/engine
```

Then verify the package statement changed in a representative file:
```
ide_read_file:
  file: work-adapter/src/main/java/io/casehub/work/engine/CallerRef.java
  startLine: 1
  endLine: 5
```

- [ ] **Step 2: Rename the recovery subpackage**

The `recovery/` subpackage should already have moved with its parent. Verify:
```
ide_read_file:
  file: work-adapter/src/main/java/io/casehub/work/engine/recovery/HumanTaskRecoveryService.java
  startLine: 1
  endLine: 3
```

If the package statement still says `io.casehub.workadapter.recovery`, rename it:
```
ide_refactor_rename:
  file: work-adapter/src/main/java/io/casehub/work/engine/recovery
  targetType: file
  newName: recovery
```

- [ ] **Step 3: Rename the test package via IntelliJ**

```
ide_refactor_rename:
  file: work-adapter/src/test/java/io/casehub/workadapter
  targetType: file
  newName: engine
  project_path: /Users/mdproctor/claude/casehub/engine
```

Verify a test file:
```
ide_read_file:
  file: work-adapter/src/test/java/io/casehub/work/engine/CallerRefTest.java
  startLine: 1
  endLine: 3
```

- [ ] **Step 4: Rename the test recovery subpackage if needed**

Same check as Step 2 but for test sources.

- [ ] **Step 5: Update application.properties**

Edit `engine/work-adapter/src/test/resources/application.properties`:
- Change `io.casehub.workadapter.NoOpPreferenceProvider` → `io.casehub.work.engine.NoOpPreferenceProvider`

- [ ] **Step 6: Update ActionGateDeploymentHealthCheck string literal**

Edit `engine/runtime/src/main/java/io/casehub/engine/internal/startup/ActionGateDeploymentHealthCheck.java` line 66:
- Change `"io.casehub.workadapter.ActionGateWorkItemHandler"` → `"io.casehub.work.engine.ActionGateWorkItemHandler"`
- Update the warning message on line 72 to say `casehub-work-engine-adapter` instead of `casehub-engine-work-adapter`

- [ ] **Step 7: Grep for any remaining old package references**

```bash
grep -rn "io.casehub.workadapter" /Users/mdproctor/claude/casehub/engine --include="*.java" --include="*.properties" --include="*.xml" | grep -v target | grep -v ".git"
```

Fix any remaining references found.

- [ ] **Step 8: Build and test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl work-adapter -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

Expected: all tests pass.

- [ ] **Step 9: Commit to engine**

```bash
git -C /Users/mdproctor/claude/casehub/engine add work-adapter/ runtime/src/main/java/io/casehub/engine/internal/startup/ActionGateDeploymentHealthCheck.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(#290): rename io.casehub.workadapter → io.casehub.work.engine

Refs casehubio/work#290"
```

---

### Task 2: Create Engine-Adapter Module in Work

Create the `engine-adapter/` module in the work repo with proper POM, then copy all source and test files from engine.

**Files:**
- Create: `work/engine-adapter/pom.xml`
- Create: `work/engine-adapter/src/main/java/io/casehub/work/engine/` (all 12 source files)
- Create: `work/engine-adapter/src/test/java/io/casehub/work/engine/` (all 10 test files)
- Create: `work/engine-adapter/src/test/resources/application.properties`
- Modify: `work/pom.xml` (add module)
- Modify: `work/docs/MODULES.md` (add entry)

**Interfaces:**
- Consumes: renamed files from Task 1
- Produces: working `casehub-work-engine-adapter` module that builds and passes tests

- [ ] **Step 1: Create the module directory structure**

```bash
mkdir -p /Users/mdproctor/claude/casehub/work/engine-adapter/src/main/java/io/casehub/work/engine/recovery
mkdir -p /Users/mdproctor/claude/casehub/work/engine-adapter/src/test/java/io/casehub/work/engine/recovery
mkdir -p /Users/mdproctor/claude/casehub/work/engine-adapter/src/test/resources
```

- [ ] **Step 2: Write the POM**

Create `work/engine-adapter/pom.xml` — re-parent under `casehub-work-parent`, with all dependencies from the original work-adapter POM. Engine deps use `${casehub.version}` instead of `${project.version}`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-work-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-work-engine-adapter</artifactId>
    <name>CaseHub Work :: Engine Adapter</name>
    <description>Bridge between casehub-work WorkItemLifecycleEvent and CaseHub engine blackboard PlanItem lifecycle</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-blackboard</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-work-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-hibernate-orm</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-work</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-persistence-memory</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-scheduler-quartz</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-work-persistence-memory</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-jdbc-h2</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ledger</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ledger-testing</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-ledger</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.awaitility</groupId>
            <artifactId>awaitility</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <parameters>true</parameters>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <systemPropertyVariables>
                        <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
                        <maven.home>${maven.home}</maven.home>
                    </systemPropertyVariables>
                </configuration>
            </plugin>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals>
                            <goal>jandex</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: Copy all source and test files from engine**

Copy each file individually (bash cp is acceptable here since these are new files being created, not in-project moves):

```bash
# Source files (12)
cp /Users/mdproctor/claude/casehub/engine/work-adapter/src/main/java/io/casehub/work/engine/*.java /Users/mdproctor/claude/casehub/work/engine-adapter/src/main/java/io/casehub/work/engine/
cp /Users/mdproctor/claude/casehub/engine/work-adapter/src/main/java/io/casehub/work/engine/recovery/*.java /Users/mdproctor/claude/casehub/work/engine-adapter/src/main/java/io/casehub/work/engine/recovery/

# Test files (10)
cp /Users/mdproctor/claude/casehub/engine/work-adapter/src/test/java/io/casehub/work/engine/*.java /Users/mdproctor/claude/casehub/work/engine-adapter/src/test/java/io/casehub/work/engine/
cp /Users/mdproctor/claude/casehub/engine/work-adapter/src/test/java/io/casehub/work/engine/recovery/*.java /Users/mdproctor/claude/casehub/work/engine-adapter/src/test/java/io/casehub/work/engine/recovery/

# Config
cp /Users/mdproctor/claude/casehub/engine/work-adapter/src/test/resources/application.properties /Users/mdproctor/claude/casehub/work/engine-adapter/src/test/resources/
```

- [ ] **Step 4: Add module to work parent POM**

Edit `work/pom.xml` — add `<module>engine-adapter</module>` before `<module>integration-tests</module>` (line 37).

- [ ] **Step 5: Sync IntelliJ and reload project**

```
ide_sync_files:
  paths: ["engine-adapter"]
  project_path: /Users/mdproctor/claude/casehub/work

ide_reload_project:
  project_path: /Users/mdproctor/claude/casehub/work
```

- [ ] **Step 6: Build and test in work**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -f /Users/mdproctor/claude/casehub/work/pom.xml
```

Expected: all tests pass. If dependency resolution fails, check that engine artifacts are available in local `.m2` (may need `mvn install` in engine first).

- [ ] **Step 7: Update docs/MODULES.md**

Add under "Integration Modules (built)":
```
| `engine-adapter/` | CaseHub engine adapter; creates WorkItems from HumanTask/ActionGate bindings, translates lifecycle events back to PlanItem transitions |
```

- [ ] **Step 8: Commit to work**

```bash
git -C /Users/mdproctor/claude/casehub/work add engine-adapter/ pom.xml docs/MODULES.md
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(#290): add casehub-work-engine-adapter module

Relocates adapter from casehub-engine work-adapter.
Refs #290"
```

---

### Task 3: Clean Up Engine

Remove the old module from engine, update all docs and metadata.

**Files:**
- Delete: `engine/work-adapter/` (entire directory)
- Modify: `engine/pom.xml` (remove module line)
- Modify: `engine/docs/DESIGN.md` (update HumanTaskBinding section)
- Modify: `engine/CLAUDE.md` (lines 135, 276, 278, 287, 423–452)

**Interfaces:**
- Consumes: Task 2 complete (work module exists and passes tests)
- Produces: clean engine with no stale work-adapter references

- [ ] **Step 1: Remove work-adapter module from engine parent POM**

Edit `engine/pom.xml` — remove the line `<module>work-adapter</module>` (line 105).

- [ ] **Step 2: Delete the work-adapter directory**

```bash
rm -rf /Users/mdproctor/claude/casehub/engine/work-adapter
```

(This is a directory delete, not a source file move — bash rm is appropriate.)

- [ ] **Step 3: Update engine docs/DESIGN.md**

Edit `engine/docs/DESIGN.md`:
- Line 180: change `### HumanTaskBinding (casehub-engine-work-adapter)` → `### HumanTaskBinding (casehub-work-engine-adapter — relocated to work repo)`
- Line 189: update reference from `work-adapter` to `casehub-work-engine-adapter (work repo)`

- [ ] **Step 4: Update engine CLAUDE.md**

Multiple edits:
1. Line 135: remove `casehub-work-adapter` from the `NoOpLedgerEntryRepository` applied-to list
2. Lines 276, 278: update `work-adapter` → `engine-adapter (work repo)` in gate mechanism narrative
3. Line 287: update `casehub-engine-work-adapter` → `casehub-work-engine-adapter`
4. Lines 423–452: replace the entire `## casehub-work-adapter Module` section with:
   ```
   ## casehub-work-adapter Module (relocated)

   Relocated to `casehub-work-engine-adapter` in the casehub-work repo. See work's CLAUDE.md for module documentation.
   ```

- [ ] **Step 5: Update parent BOM**

Edit `/Users/mdproctor/claude/casehub/parent/pom.xml` line 512:
- Change `<artifactId>casehub-engine-work-adapter</artifactId>` → `<artifactId>casehub-work-engine-adapter</artifactId>`

- [ ] **Step 6: Grep for stale references across engine**

```bash
grep -rn "io.casehub.workadapter\|casehub-engine-work-adapter" /Users/mdproctor/claude/casehub/engine --include="*.java" --include="*.properties" --include="*.xml" --include="*.md" | grep -v target | grep -v ".git" | grep -v "docs/specs/"
```

Historical specs are intentionally excluded (paths were correct when written). Fix any other stale references.

- [ ] **Step 7: Build engine without work-adapter**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl runtime -f /Users/mdproctor/claude/casehub/engine/pom.xml -DskipTests
```

Expected: compiles cleanly.

- [ ] **Step 8: Commit to engine and parent**

```bash
git -C /Users/mdproctor/claude/casehub/engine add -A
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(#290): remove work-adapter module — relocated to casehub-work

Module relocated to casehub-work-engine-adapter in casehub-work repo.
Refs casehubio/work#290"

git -C /Users/mdproctor/claude/casehub/parent add pom.xml
git -C /Users/mdproctor/claude/casehub/parent commit -m "refactor(#290): update BOM — casehub-engine-work-adapter → casehub-work-engine-adapter

Refs casehubio/work#290"
```

---

### Task 4: Delete WorkItemCallerRef and Fix Docs

Delete the broken `WorkItemCallerRef` utility from work-api. Update CLAUDE.md and ARC42STORIES.MD.

**Files:**
- Delete (IntelliJ safe-delete): `work/api/src/main/java/io/casehub/work/api/WorkItemCallerRef.java`
- Delete (IntelliJ safe-delete): `work/api/src/test/java/io/casehub/work/api/WorkItemCallerRefTest.java`
- Modify: `work/CLAUDE.md` (remove utility paragraph, add engine-adapter section)
- Modify: `work/ARC42STORIES.MD` (fix §3 callerRef format, add module to §5)

**Interfaces:**
- Consumes: Task 2 (engine-adapter module exists with `CallerRef` sealed hierarchy)
- Produces: clean work-api with no broken utility, accurate docs

- [ ] **Step 1: Safe-delete WorkItemCallerRef via IntelliJ**

```
ide_refactor_safe_delete:
  file: api/src/main/java/io/casehub/work/api/WorkItemCallerRef.java
  target_type: file
  project_path: /Users/mdproctor/claude/casehub/work
```

- [ ] **Step 2: Safe-delete WorkItemCallerRefTest via IntelliJ**

```
ide_refactor_safe_delete:
  file: api/src/test/java/io/casehub/work/api/WorkItemCallerRefTest.java
  target_type: file
  project_path: /Users/mdproctor/claude/casehub/work
```

- [ ] **Step 3: Build work-api to confirm clean deletion**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -f /Users/mdproctor/claude/casehub/work/pom.xml
```

- [ ] **Step 4: Update CLAUDE.md — remove utility paragraph**

Remove the `## casehub-work-api Utilities` section that documents `WorkItemCallerRef.parseCaseId`.

- [ ] **Step 5: Update CLAUDE.md — add engine-adapter module note**

Add under the appropriate section:
```markdown
## engine-adapter Module

CaseHub engine adapter (`casehub-work-engine-adapter`). Bridges casehub-work WorkItem lifecycle with CaseHub engine's blackboard PlanItem lifecycle. Creates WorkItems from HumanTask/ActionGate YAML bindings, translates lifecycle events back to PlanItem transitions.

Activated by adding `casehub-work-engine-adapter` to consumer classpath. Package: `io.casehub.work.engine`.
```

- [ ] **Step 6: Fix ARC42STORIES.MD §3 callerRef format**

Search for the incorrect callerRef format documentation in §3 and update:
- Old: `caseId:planItemId`
- New: `case:{caseId}/pi:{planItemId}` for HumanTask bindings, `case:{caseId}/gate:{gateId}` for ActionGate bindings

- [ ] **Step 7: Add engine-adapter to ARC42STORIES.MD §5 if module listing exists**

Check §5 for a module table. If one exists, add:
```
| `engine-adapter/` | CaseHub engine bridge — HumanTask/ActionGate WorkItem lifecycle |
```

- [ ] **Step 8: Commit to work**

```bash
git -C /Users/mdproctor/claude/casehub/work add api/ CLAUDE.md ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/work commit -m "refactor(#290): delete WorkItemCallerRef, update docs for engine-adapter

- Delete broken WorkItemCallerRef from work-api (CallerRef sealed hierarchy is canonical)
- Fix ARC42STORIES.MD §3 callerRef format documentation
- Add engine-adapter module documentation to CLAUDE.md
Refs #290"
```

---

### Task 5: Final Verification

Cross-repo verification that nothing is stale.

**Files:** None (read-only verification)

- [ ] **Step 1: Grep work for old package**

```bash
grep -rn "io.casehub.workadapter" /Users/mdproctor/claude/casehub/work --include="*.java" --include="*.properties" --include="*.xml" | grep -v target | grep -v ".git"
```

Expected: zero results.

- [ ] **Step 2: Grep engine for old package**

```bash
grep -rn "io.casehub.workadapter" /Users/mdproctor/claude/casehub/engine --include="*.java" --include="*.properties" --include="*.xml" | grep -v target | grep -v ".git"
```

Expected: zero results.

- [ ] **Step 3: Grep for old artifact ID across both repos**

```bash
grep -rn "casehub-engine-work-adapter" /Users/mdproctor/claude/casehub/work --include="*.xml" --include="*.md" | grep -v target | grep -v ".git" | grep -v "docs/specs/"
grep -rn "casehub-engine-work-adapter" /Users/mdproctor/claude/casehub/engine --include="*.xml" --include="*.md" | grep -v target | grep -v ".git" | grep -v "docs/specs/"
```

Expected: zero results (historical specs excluded).

- [ ] **Step 4: Full build of engine-adapter in work**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter -f /Users/mdproctor/claude/casehub/work/pom.xml
```

Expected: all tests pass.

- [ ] **Step 5: Verify IntelliJ sees the new module**

```
ide_find_class:
  query: HumanTaskScheduleHandler
  project_path: /Users/mdproctor/claude/casehub/work
```

Expected: finds `io.casehub.work.engine.HumanTaskScheduleHandler` in `engine-adapter` module.
