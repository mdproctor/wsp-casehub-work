# quarkus-work → quarkus-work Rename Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rename the entire project from `quarkus-work` to `quarkus-work` — directory, GitHub repo, Maven coordinates, Java packages, class names, config prefix, REST paths, and all documentation.

**Architecture:** The rename proceeds in dependency order: GitHub repo and directory first (infrastructure), then Maven POMs, then Java source (directory moves + text replacements), then config/REST paths, then docs. Each task ends with a build or test verification so regressions surface immediately.

**Tech Stack:** Java 21, Maven 3.9, Quarkus 3.32.2, `gh` CLI, `sed`, `git mv`

---

## Rename Map

| Old | New |
|-----|-----|
| `~/claude/quarkus-work/` | `~/claude/quarkus-work/` |
| `mdproctor/quarkus-work` (GitHub) | `mdproctor/quarkus-work` |
| `work-flow/` | `work-flow/` |
| `quarkus-work-ledger/` | `quarkus-work-ledger/` |
| `quarkus-work-examples/` | `quarkus-work-examples/` |
| `quarkus-work-flow-examples/` | `quarkus-work-flow-examples/` |
| groupId `io.quarkiverse.work` | `io.quarkiverse.work` |
| artifactId `quarkus-work[-*]` | `quarkus-work[-*]` |
| Java package `io.quarkiverse.work` | `io.quarkiverse.work` |
| Class `TarkusConfig` | `WorkItemsConfig` |
| Class `TarkusProcessor` | `WorkItemsProcessor` |
| Class `WorkItemsFlow` | `WorkItemsFlow` |
| Class `WorkItemTaskBuilder` | `WorkItemTaskBuilder` |
| Config prefix `quarkus.tarkus` | `quarkus.work` |
| Feature name `"tarkus"` | `"workitems"` |
| REST path `/workitems` | `/workitems` |
| REST path `/workitems/actors` | `/workitems/actors` |

---

## Task 0: Rename GitHub repository and update git remote

**Files:**
- Git remote URL

- [ ] **Step 1: Rename the GitHub repository**

```bash
gh repo rename quarkus-work --repo mdproctor/quarkus-work --yes
```

Expected output: `✓ Renamed repository to mdproctor/quarkus-work`

- [ ] **Step 2: Update the git remote in the local repository**

```bash
cd /Users/mdproctor/claude/quarkus-work
git remote set-url origin https://github.com/mdproctor/quarkus-work.git
git remote -v
```

Expected: both fetch and push show `quarkus-work.git`

- [ ] **Step 3: Commit the remote URL change note**

No files changed — the remote update is not tracked by git. Continue to Task 1.

---

## Task 1: Rename the project directory

**⚠️ This task must be executed from OUTSIDE the project directory.** Claude Code's working directory will become invalid after the move. After this task, restart Claude Code pointing to the new path `~/claude/quarkus-work`.

- [ ] **Step 1: Move the directory from a parent shell**

```bash
cd ~/claude
mv quarkus-work quarkus-work
```

- [ ] **Step 2: Verify git history is intact**

```bash
cd ~/claude/quarkus-work
git log --oneline -3
git status --short
```

Expected: git history present, working tree clean.

- [ ] **Step 3: Verify remote is correct**

```bash
git remote -v
```

Expected: shows `quarkus-work.git` (already updated in Task 0).

**After this step: restart Claude Code with the new working directory `~/claude/quarkus-work` before continuing.**

---

## Task 2: Rename module directories

**Files modified:** `pom.xml` (parent), module directories renamed with `git mv`

Working directory for all commands: `/Users/mdproctor/claude/quarkus-work`

- [ ] **Step 1: Rename `work-flow/` → `work-flow/`**

```bash
git mv work-flow work-flow
```

- [ ] **Step 2: Rename `quarkus-work-ledger/` → `quarkus-work-ledger/`**

```bash
git mv quarkus-work-ledger quarkus-work-ledger
```

- [ ] **Step 3: Rename `quarkus-work-examples/` → `quarkus-work-examples/`**

```bash
git mv quarkus-work-examples quarkus-work-examples
```

- [ ] **Step 4: Rename `quarkus-work-flow-examples/` → `quarkus-work-flow-examples/`**

```bash
git mv quarkus-work-flow-examples quarkus-work-flow-examples
```

- [ ] **Step 5: Update parent `pom.xml` module list**

In `pom.xml`, replace the `<modules>` section with:

```xml
  <modules>
    <module>runtime</module>
    <module>deployment</module>
    <module>testing</module>
    <module>work-flow</module>
    <module>quarkus-work-ledger</module>
    <module>quarkus-work-examples</module>
    <module>quarkus-work-flow-examples</module>
    <module>integration-tests</module>
  </modules>
```

- [ ] **Step 6: Commit directory renames**

```bash
git add -A
git commit -m "refactor: rename module directories tarkus→workitems"
```

---

## Task 3: Update all Maven POM files

**Files:**
- Modify: `pom.xml` (parent)
- Modify: `runtime/pom.xml`
- Modify: `deployment/pom.xml`
- Modify: `testing/pom.xml`
- Modify: `work-flow/pom.xml`
- Modify: `quarkus-work-ledger/pom.xml`
- Modify: `quarkus-work-examples/pom.xml`
- Modify: `quarkus-work-flow-examples/pom.xml`
- Modify: `integration-tests/pom.xml`

The safest approach is a single bulk `sed` across all pom files, then manual review.

- [ ] **Step 1: Bulk replace groupId and artifactId prefixes across all poms**

```bash
find . -name "pom.xml" -not -path "*/target/*" -exec sed -i '' \
  -e 's|io\.quarkiverse\.tarkus|io.quarkiverse.work|g' \
  -e 's|quarkus-work-parent|quarkus-work-parent|g' \
  -e 's|<artifactId>quarkus-work</artifactId>|<artifactId>quarkus-work</artifactId>|g' \
  -e 's|<artifactId>quarkus-work-deployment</artifactId>|<artifactId>quarkus-work-deployment</artifactId>|g' \
  -e 's|<artifactId>quarkus-work-testing</artifactId>|<artifactId>quarkus-work-testing</artifactId>|g' \
  -e 's|<artifactId>quarkus-work-flow</artifactId>|<artifactId>quarkus-work-flow</artifactId>|g' \
  -e 's|<artifactId>quarkus-work-ledger</artifactId>|<artifactId>quarkus-work-ledger</artifactId>|g' \
  -e 's|<artifactId>quarkus-work-examples</artifactId>|<artifactId>quarkus-work-examples</artifactId>|g' \
  -e 's|<artifactId>quarkus-work-flow-examples</artifactId>|<artifactId>quarkus-work-flow-examples</artifactId>|g' \
  -e 's|<artifactId>quarkus-work-integration-tests</artifactId>|<artifactId>quarkus-work-integration-tests</artifactId>|g' \
  {} \;
```

- [ ] **Step 2: Update SCM URLs in parent pom.xml**

In `pom.xml`, update the `<scm>` block:

```xml
  <scm>
    <connection>scm:git:git@github.com:mdproctor/quarkus-work.git</connection>
    <developerConnection>scm:git:git@github.com:mdproctor/quarkus-work.git</developerConnection>
    <url>https://github.com/mdproctor/quarkus-work</url>
    <tag>HEAD</tag>
  </scm>
```

- [ ] **Step 3: Update parent pom.xml project identity**

In `pom.xml`, update:
```xml
  <artifactId>quarkus-work-parent</artifactId>
  <name>Quarkus WorkItems - Parent</name>
  <description>Human-scale WorkItem lifecycle management ...</description>
  <url>https://github.com/mdproctor/quarkus-work</url>
```

- [ ] **Step 4: Verify pom files parse correctly**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn validate 2>&1 | tail -5
```

Expected: BUILD SUCCESS (validation only, no compilation).

- [ ] **Step 5: Commit pom changes**

```bash
git add -A
git commit -m "refactor(pom): rename Maven coordinates io.quarkiverse.work→workitems"
```

---

## Task 4: Rename Java source directories

Move `io/quarkiverse/tarkus` → `io/quarkiverse/workitems` in every module's source tree using `git mv` to preserve history.

- [ ] **Step 1: Rename runtime source directory**

```bash
mkdir -p runtime/src/main/java/io/quarkiverse/workitems
git mv runtime/src/main/java/io/quarkiverse/tarkus/runtime \
        runtime/src/main/java/io/quarkiverse/workitems/runtime
rmdir runtime/src/main/java/io/quarkiverse/tarkus 2>/dev/null || true
```

- [ ] **Step 2: Rename runtime test directory**

```bash
mkdir -p runtime/src/test/java/io/quarkiverse/workitems
git mv runtime/src/test/java/io/quarkiverse/tarkus/runtime \
        runtime/src/test/java/io/quarkiverse/workitems/runtime
rmdir runtime/src/test/java/io/quarkiverse/tarkus 2>/dev/null || true
```

- [ ] **Step 3: Rename deployment source directory**

```bash
mkdir -p deployment/src/main/java/io/quarkiverse/workitems
git mv deployment/src/main/java/io/quarkiverse/tarkus/deployment \
        deployment/src/main/java/io/quarkiverse/workitems/deployment
rmdir deployment/src/main/java/io/quarkiverse/tarkus 2>/dev/null || true
```

- [ ] **Step 4: Rename testing source directory**

```bash
mkdir -p testing/src/main/java/io/quarkiverse/workitems
git mv testing/src/main/java/io/quarkiverse/tarkus/testing \
        testing/src/main/java/io/quarkiverse/workitems/testing
rmdir testing/src/main/java/io/quarkiverse/tarkus 2>/dev/null || true
```

- [ ] **Step 5: Rename work-flow source directory**

```bash
mkdir -p work-flow/src/main/java/io/quarkiverse/workitems
git mv work-flow/src/main/java/io/quarkiverse/tarkus/flow \
        work-flow/src/main/java/io/quarkiverse/workitems/flow
rmdir work-flow/src/main/java/io/quarkiverse/tarkus 2>/dev/null || true

mkdir -p work-flow/src/test/java/io/quarkiverse/workitems
git mv work-flow/src/test/java/io/quarkiverse/tarkus/flow \
        work-flow/src/test/java/io/quarkiverse/workitems/flow
rmdir work-flow/src/test/java/io/quarkiverse/tarkus 2>/dev/null || true
```

- [ ] **Step 6: Rename ledger source directory**

```bash
mkdir -p quarkus-work-ledger/src/main/java/io/quarkiverse/workitems
git mv quarkus-work-ledger/src/main/java/io/quarkiverse/tarkus/ledger \
        quarkus-work-ledger/src/main/java/io/quarkiverse/workitems/ledger
rmdir quarkus-work-ledger/src/main/java/io/quarkiverse/tarkus 2>/dev/null || true

mkdir -p quarkus-work-ledger/src/test/java/io/quarkiverse/workitems
git mv quarkus-work-ledger/src/test/java/io/quarkiverse/tarkus/ledger \
        quarkus-work-ledger/src/test/java/io/quarkiverse/workitems/ledger
rmdir quarkus-work-ledger/src/test/java/io/quarkiverse/tarkus 2>/dev/null || true
```

- [ ] **Step 7: Rename examples source directories**

```bash
# workitems-examples
mkdir -p quarkus-work-examples/src/main/java/io/quarkiverse/workitems
git mv quarkus-work-examples/src/main/java/io/quarkiverse/tarkus/examples \
        quarkus-work-examples/src/main/java/io/quarkiverse/workitems/examples
rmdir quarkus-work-examples/src/main/java/io/quarkiverse/tarkus 2>/dev/null || true

mkdir -p quarkus-work-examples/src/test/java/io/quarkiverse/workitems
git mv quarkus-work-examples/src/test/java/io/quarkiverse/tarkus/examples \
        quarkus-work-examples/src/test/java/io/quarkiverse/workitems/examples
rmdir quarkus-work-examples/src/test/java/io/quarkiverse/tarkus 2>/dev/null || true

# work-flow-examples
mkdir -p quarkus-work-flow-examples/src/main/java/io/quarkiverse/workitems
git mv quarkus-work-flow-examples/src/main/java/io/quarkiverse/tarkus/examples \
        quarkus-work-flow-examples/src/main/java/io/quarkiverse/workitems/examples
rmdir quarkus-work-flow-examples/src/main/java/io/quarkiverse/tarkus 2>/dev/null || true

mkdir -p quarkus-work-flow-examples/src/test/java/io/quarkiverse/workitems
git mv quarkus-work-flow-examples/src/test/java/io/quarkiverse/tarkus/examples \
        quarkus-work-flow-examples/src/test/java/io/quarkiverse/workitems/examples
rmdir quarkus-work-flow-examples/src/test/java/io/quarkiverse/tarkus 2>/dev/null || true
```

- [ ] **Step 8: Rename integration-tests source directory**

```bash
mkdir -p integration-tests/src/test/java/io/quarkiverse/workitems
git mv integration-tests/src/test/java/io/quarkiverse/tarkus \
        integration-tests/src/test/java/io/quarkiverse/workitems
```

- [ ] **Step 9: Commit directory moves**

```bash
git add -A
git commit -m "refactor: move Java source directories tarkus→workitems"
```

---

## Task 5: Update all Java package declarations and imports

All `.java` files now live at the new path but still contain `io.quarkiverse.work` in their `package` and `import` statements. Fix with bulk `sed`.

- [ ] **Step 1: Bulk replace package and import statements**

```bash
find . -name "*.java" -not -path "*/target/*" -exec sed -i '' \
  's|io\.quarkiverse\.tarkus|io.quarkiverse.work|g' {} \;
```

- [ ] **Step 2: Verify no stale references remain in source**

```bash
grep -r "io\.quarkiverse\.tarkus" --include="*.java" \
  $(find . -name "*.java" -not -path "*/target/*") | head -20
```

Expected: no output. If any remain, they are in `target/` or missed — investigate before continuing.

- [ ] **Step 3: Spot-check a file to confirm the replacement**

```bash
head -5 runtime/src/main/java/io/quarkiverse/workitems/runtime/model/WorkItem.java
```

Expected: `package io.quarkiverse.work.runtime.model;`

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "refactor: update all Java package declarations and imports tarkus→workitems"
```

---

## Task 6: Rename Tarkus-named classes

Rename four classes: `TarkusConfig`, `TarkusProcessor`, `WorkItemsFlow`, `WorkItemTaskBuilder`.

**Files:**
- Rename + modify: `runtime/.../config/TarkusConfig.java` → `WorkItemsConfig.java`
- Rename + modify: `deployment/.../deployment/TarkusProcessor.java` → `WorkItemsProcessor.java`
- Rename + modify: `work-flow/.../flow/WorkItemsFlow.java` → `WorkItemsFlow.java`
- Rename + modify: `work-flow/.../flow/WorkItemTaskBuilder.java` → `WorkItemTaskBuilder.java`
- All files that import or reference these class names

- [ ] **Step 1: Rename TarkusConfig → WorkItemsConfig**

```bash
git mv runtime/src/main/java/io/quarkiverse/workitems/runtime/config/TarkusConfig.java \
        runtime/src/main/java/io/quarkiverse/workitems/runtime/config/WorkItemsConfig.java
```

- [ ] **Step 2: Rename TarkusProcessor → WorkItemsProcessor**

```bash
git mv deployment/src/main/java/io/quarkiverse/workitems/deployment/TarkusProcessor.java \
        deployment/src/main/java/io/quarkiverse/workitems/deployment/WorkItemsProcessor.java
```

- [ ] **Step 3: Rename WorkItemsFlow → WorkItemsFlow**

```bash
git mv work-flow/src/main/java/io/quarkiverse/workitems/flow/WorkItemsFlow.java \
        work-flow/src/main/java/io/quarkiverse/workitems/flow/WorkItemsFlow.java
```

- [ ] **Step 4: Rename WorkItemTaskBuilder → WorkItemTaskBuilder**

```bash
git mv work-flow/src/main/java/io/quarkiverse/workitems/flow/WorkItemTaskBuilder.java \
        work-flow/src/main/java/io/quarkiverse/workitems/flow/WorkItemTaskBuilder.java
```

- [ ] **Step 5: Update all class name references across Java source**

```bash
find . -name "*.java" -not -path "*/target/*" -exec sed -i '' \
  -e 's|TarkusConfig|WorkItemsConfig|g' \
  -e 's|TarkusProcessor|WorkItemsProcessor|g' \
  -e 's|WorkItemsFlow|WorkItemsFlow|g' \
  -e 's|WorkItemTaskBuilder|WorkItemTaskBuilder|g' \
  {} \;
```

- [ ] **Step 6: Update the test workflow class name reference**

The test file `work-flow/src/test/java/.../flow/TestTarkusWorkflow.java` extends `WorkItemsFlow`. Rename the test file too:

```bash
git mv work-flow/src/test/java/io/quarkiverse/workitems/flow/TestTarkusWorkflow.java \
        work-flow/src/test/java/io/quarkiverse/workitems/flow/TestWorkItemsWorkflow.java
```

Then fix the class name inside:
```bash
sed -i '' 's|class TestTarkusWorkflow|class TestWorkItemsWorkflow|g' \
  work-flow/src/test/java/io/quarkiverse/workitems/flow/TestWorkItemsWorkflow.java
```

Update references to `TestTarkusWorkflow` in `HumanTaskIntegrationTest.java`:
```bash
sed -i '' 's|TestTarkusWorkflow|TestWorkItemsWorkflow|g' \
  work-flow/src/test/java/io/quarkiverse/workitems/flow/HumanTaskIntegrationTest.java
```

- [ ] **Step 7: Verify no Tarkus class name references remain**

```bash
grep -r "TarkusConfig\|TarkusProcessor\|WorkItemsFlow\|WorkItemTaskBuilder\|TestTarkusWorkflow" \
  --include="*.java" $(find . -name "*.java" -not -path "*/target/*") | head -10
```

Expected: no output.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "refactor: rename TarkusConfig/Processor/Flow/TaskBuilder → WorkItemsConfig/Processor/Flow and WorkItemTaskBuilder"
```

---

## Task 7: Update config prefix, feature name, and REST paths

**Files:**
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/config/WorkItemsConfig.java`
- Modify: `work-flow/src/main/java/io/quarkiverse/workitems/deployment/WorkItemsProcessor.java`
- Modify: `runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemResource.java`
- Modify: `quarkus-work-ledger/src/main/java/io/quarkiverse/workitems/ledger/api/LedgerResource.java`
- Modify: `quarkus-work-ledger/src/main/java/io/quarkiverse/workitems/ledger/api/ActorTrustResource.java`
- Modify: all `application.properties` files

- [ ] **Step 1: Update ConfigMapping prefix in WorkItemsConfig**

In `runtime/src/main/java/io/quarkiverse/workitems/runtime/config/WorkItemsConfig.java`:

Replace:
```java
@ConfigMapping(prefix = "quarkus.tarkus")
```
With:
```java
@ConfigMapping(prefix = "quarkus.work")
```

- [ ] **Step 2: Update feature name in WorkItemsProcessor**

In `deployment/src/main/java/io/quarkiverse/workitems/deployment/WorkItemsProcessor.java`:

Replace:
```java
private static final String FEATURE = "tarkus";
```
With:
```java
private static final String FEATURE = "workitems";
```

- [ ] **Step 3: Update REST paths**

```bash
# WorkItemResource: /workitems → /workitems
sed -i '' 's|@Path("/workitems")|@Path("/workitems")|g' \
  runtime/src/main/java/io/quarkiverse/workitems/runtime/api/WorkItemResource.java

# LedgerResource: /workitems/{id} → /workitems/{id}
sed -i '' 's|@Path("/workitems/{id}")|@Path("/workitems/{id}")|g' \
  quarkus-work-ledger/src/main/java/io/quarkiverse/workitems/ledger/api/LedgerResource.java

# ActorTrustResource: /workitems/actors → /workitems/actors
sed -i '' 's|@Path("/workitems/actors")|@Path("/workitems/actors")|g' \
  quarkus-work-ledger/src/main/java/io/quarkiverse/workitems/ledger/api/ActorTrustResource.java
```

- [ ] **Step 4: Update config prefix in all application.properties (non-target)**

```bash
find . -name "application.properties" -not -path "*/target/*" -exec sed -i '' \
  's|quarkus\.tarkus\.|quarkus.work.|g' {} \;
```

- [ ] **Step 5: Update config prefix in any Javadoc or string literals**

```bash
find . -name "*.java" -not -path "*/target/*" -exec sed -i '' \
  's|quarkus\.tarkus\.|quarkus.work.|g' {} \;
```

- [ ] **Step 6: Update REST path references in tests and integration tests**

```bash
find . -name "*.java" -not -path "*/target/*" -exec sed -i '' \
  -e 's|/workitems|/workitems|g' \
  -e 's|/workitems/actors|/workitems/actors|g' \
  {} \;
```

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "refactor: update config prefix quarkus.tarkus→workitems, REST paths /tarkus/*→/workitems/*, feature name"
```

---

## Task 8: Build verification — compile and test

- [ ] **Step 1: Install quarkus-ledger (required prerequisite)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -DskipTests \
  -f ~/claude/quarkus-ledger/pom.xml 2>&1 | tail -5
```

Expected: BUILD SUCCESS

- [ ] **Step 2: Full clean install without tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -DskipTests 2>&1 | tail -20
```

Expected: BUILD SUCCESS across all 9 modules. If compilation fails, read the error, identify which class or import is wrong, fix with targeted `sed` or manual edit, and retry.

Common failure patterns:
- Remaining `io.quarkiverse.work` in a file missed by the bulk sed → `grep -r "io\.quarkiverse\.tarkus" --include="*.java" . | grep -v target`
- `TarkusConfig`/`WorkItemsFlow` reference not replaced → `grep -r "Tarkus[A-Z]" --include="*.java" . | grep -v target`
- Flyway migration directory mismatch → check `runtime/src/main/resources/db/migration/`

- [ ] **Step 3: Run all tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime,quarkus-work-ledger,work-flow,quarkus-work-examples,quarkus-work-flow-examples 2>&1 | grep -E "Tests run|BUILD|FAIL"
```

Expected: all modules BUILD SUCCESS, 0 failures.

- [ ] **Step 4: Fix any test failures**

Tests most likely to fail:
- REST Assured tests hitting `/workitems` — check for hardcoded path strings in test `@Test` methods
- `@QuarkusTest` config binding failures — `quarkus.work.*` property in test application.properties not yet renamed
- Class injection failures (`@Inject WorkItemsConfig`) — check field types match renamed class

Fix, then re-run the failing module only:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module> 2>&1 | tail -20
```

- [ ] **Step 5: Commit (if any fixes applied in this task)**

```bash
git add -A && git commit -m "fix: post-rename compilation and test fixes"
```

---

## Task 9: Update all documentation

**Files:**
- Modify: `README.md`
- Modify: `docs/DESIGN.md`
- Modify: `docs/api-reference.md`
- Modify: `docs/integration-guide.md`
- Modify: `CLAUDE.md`
- Modify: `HANDOFF.md`
- Modify: `adr/0001-extract-ledger-infrastructure-to-quarkus-ledger.md`
- Modify: `adr/INDEX.md`
- Modify: `work-flow/README.md`
- Modify: `quarkus-work-examples/README.md`
- Modify: `quarkus-work-flow-examples/README.md`
- Modify: `blog/*.md`
- Modify: `docs/specs/*.md`
- Modify: `docs/superpowers/specs/*.md`
- Modify: `docs/superpowers/plans/*.md`

- [ ] **Step 1: Bulk replace "quarkus-work" and "Quarkus WorkItems" in all markdown**

```bash
find . -name "*.md" -not -path "*/target/*" -exec sed -i '' \
  -e 's|quarkus-work|quarkus-work|g' \
  -e 's|Quarkus WorkItems|Quarkus WorkItems|g' \
  -e 's|quarkus\.tarkus\.|quarkus.work.|g' \
  -e 's|io\.quarkiverse\.tarkus|io.quarkiverse.work|g' \
  -e 's|/workitems|/workitems|g' \
  -e 's|/workitems/actors|/workitems/actors|g' \
  {} \;
```

- [ ] **Step 2: Update the CLAUDE.md project type and GitHub repo reference**

In `CLAUDE.md`, find and update the GitHub repo line:
```
**GitHub repo:** mdproctor/quarkus-work
```

Also update the Build and Test section references to renamed modules:
```bash
# Run tests (flow integration)
JAVA_HOME=$(...) mvn test -pl work-flow

# Run tests (examples module)  
JAVA_HOME=$(...) mvn test -pl quarkus-work-examples
```

- [ ] **Step 3: Replace remaining "Tarkus" references in markdown**

```bash
# Replace "Tarkus" (capitalized, as a proper noun for the extension) but NOT "work-flow" 
# (already handled above) or identifiers that are correctly named
find . -name "*.md" -not -path "*/target/*" -exec sed -i '' \
  -e 's|\bTarkus\b|WorkItems|g' \
  -e 's|\btarkus\b|workitems|g' \
  {} \;
```

**Review the changes carefully** — some references like "WorkItemsFlow" in code blocks may now read "WorkItemsFlow" which is correct. But check for "Worktarkusitems" or other mangled strings:

```bash
grep -r "tarkus\|Tarkus" --include="*.md" . | grep -v target | grep -v ".git" | head -30
```

Fix any oddities manually.

- [ ] **Step 4: Update HANDOFF.md**

The HANDOFF.md will contain references to `quarkus-work`. After the bulk sed, verify it reads correctly and update the "What Changed This Session" section to note the rename.

- [ ] **Step 5: Commit all doc updates**

```bash
git add -A
git commit -m "docs: update all documentation for quarkus-work→quarkus-work rename"
```

---

## Task 10: Migrate Claude memory path reference and final cleanup

- [ ] **Step 1: Update CLAUDE.md reference to memory path (if any)**

Check if CLAUDE.md references the project path `quarkus-work` anywhere:

```bash
grep -n "quarkus-work\|quarkus-work" CLAUDE.md | head -10
```

Fix any remaining references manually.

- [ ] **Step 2: Check for any remaining tarkus references in Java or properties**

```bash
grep -r "tarkus\|Tarkus" --include="*.java" --include="*.properties" \
  $(find . -not -path "*/target/*" -not -path "*/.git/*") | grep -v "quarkus-work\|workitems" | head -20
```

Expected: no output (or only in intentional contexts like commit messages or external library names that shouldn't change).

- [ ] **Step 3: Final full build + test run**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | tail -15
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 4: Push to GitHub**

```bash
git push origin main
```

Expected: push succeeds to `https://github.com/mdproctor/quarkus-work.git`

- [ ] **Step 5: Final commit summary**

```bash
git log --oneline -8
```

Verify the commit trail makes sense. Done.

---

## Self-Review

### Spec coverage
- ✅ GitHub repo rename (Task 0)
- ✅ Project directory rename (Task 1)
- ✅ Module directory renames (Task 2)
- ✅ Maven coordinate changes — groupId, artifactIds (Task 3)
- ✅ Java source directory moves (Task 4)
- ✅ Java package declarations and imports (Task 5)
- ✅ Class renames: TarkusConfig, TarkusProcessor, WorkItemsFlow, WorkItemTaskBuilder (Task 6)
- ✅ Config prefix `quarkus.tarkus` → `quarkus.work` (Task 7)
- ✅ Feature name `"tarkus"` → `"workitems"` (Task 7)
- ✅ REST paths `/workitems` → `/workitems`, `/workitems/actors` → `/workitems/actors` (Task 7)
- ✅ All application.properties (Task 7)
- ✅ Build verification (Task 8)
- ✅ Test verification (Task 8)
- ✅ All documentation (Task 9)
- ✅ Claude data files (Task 10)
- ✅ Final push (Task 10)

### Known risks not covered by the plan
- `META-INF/maven/io.quarkiverse.work/quarkus-work/pom.xml` — this is a generated file in the installed jar, not a source file. It will regenerate correctly after `mvn clean install`. Do not edit it manually.
- `quarkus-ledger` dependency: `io.quarkiverse.ledger:quarkus-ledger` is a separate project — its coordinates do NOT change. The `quarkus-work-ledger` module's dependency on it stays as `io.quarkiverse.ledger`.
- GitHub issue URLs embedded in commit messages or docs reference `mdproctor/quarkus-work` — GitHub auto-redirects these. Leave them as historical record.
- The `~/.claude/projects/-Users-mdproctor-claude-quarkus-work/` directory contains conversation history files (`.jsonl`), not memory. No migration needed — Claude will create a new project entry for the renamed path automatically.
