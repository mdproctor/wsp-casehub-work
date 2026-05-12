# WorkItemTemplateService callerRef Overload — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `callerRef` parameter to `WorkItemTemplateService.instantiate()` and `toCreateRequest()` so the casehub-engine work adapter can set routing metadata when instantiating a template on behalf of a case binding.

**Architecture:** Two new overloads — existing 4-arg methods delegate to new 5-arg methods with `callerRef = null`, preserving all existing call sites. The multi-instance path logs a warning and ignores `callerRef` (tracked in casehubio/work#166). Pure addition: no existing code changes beyond the delegation wiring.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache, AssertJ, JUnit 5 (`quarkus-junit`).

---

## File map

| File | Action |
|---|---|
| `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java` | Modify — add 5-arg `toCreateRequest()` and `instantiate()` overloads; delegate existing 4-arg methods |
| `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateServiceTest.java` | Modify — add 2 unit tests for `toCreateRequest()` callerRef |
| `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateInstantiateTest.java` | Create — `@QuarkusTest` for `instantiate()` callerRef (requires CDI + JPA) |

---

### Task 1: Failing unit tests for `toCreateRequest()` callerRef

**Files:**
- Modify: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateServiceTest.java`

- [ ] **Step 1.1: Add two failing tests after the existing `toCreateRequest` block**

  Open `WorkItemTemplateServiceTest.java`. After the `toCreateRequest_nullFields_whenTemplateHasNoDefaults` test and before the `parseLabels` section, add:

  ```java
  @Test
  void toCreateRequest_setsCallerRef_whenProvided() {
      final WorkItemTemplate t = template("IRB Review");
      final String callerRef = "case:550e8400-e29b-41d4-a716-446655440000/pi:irb-gate";
      final WorkItemCreateRequest req =
          WorkItemTemplateService.toCreateRequest(t, null, null, "system", callerRef);
      assertThat(req.callerRef()).isEqualTo(callerRef);
  }

  @Test
  void toCreateRequest_callerRefNull_whenNotProvided() {
      final WorkItemTemplate t = template("IRB Review");
      final WorkItemCreateRequest req =
          WorkItemTemplateService.toCreateRequest(t, null, null, "system");
      assertThat(req.callerRef()).isNull();
  }
  ```

  No imports needed — `WorkItemCreateRequest`, `WorkItemTemplate`, and `assertThat` are already imported.

- [ ] **Step 1.2: Run tests to confirm they fail with the expected error**

  ```bash
  scripts/mvn-test runtime -Dtest=WorkItemTemplateServiceTest
  ```

  Expected: `toCreateRequest_setsCallerRef_whenProvided` fails with
  `no suitable method found for toCreateRequest(WorkItemTemplate,null,null,String,String)`.
  `toCreateRequest_callerRefNull_whenNotProvided` passes (the 4-arg form already returns null).

---

### Task 2: Implement `toCreateRequest()` 5-arg overload — make unit tests pass

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java`

- [ ] **Step 2.1: Replace the existing `toCreateRequest()` body with a delegating overload and add the new 5-arg method**

  The existing 4-arg method becomes a one-liner delegate. Replace the entire existing `toCreateRequest` static method (currently ~20 lines ending with the `return new WorkItemCreateRequest(...)`) with:

  ```java
  public static WorkItemCreateRequest toCreateRequest(
          final WorkItemTemplate template,
          final String titleOverride,
          final String assigneeIdOverride,
          final String createdBy) {
      return toCreateRequest(template, titleOverride, assigneeIdOverride, createdBy, null);
  }

  /**
   * Convert a template and optional overrides into a {@link WorkItemCreateRequest}.
   *
   * @param template the template providing defaults
   * @param titleOverride if non-null and non-blank, used as the title; otherwise template name
   * @param assigneeIdOverride if non-null, set as the direct assignee
   * @param createdBy the actor triggering the instantiation
   * @param callerRef opaque routing key set by engine adapters (null for human-initiated creation)
   * @return the create request ready for {@link WorkItemService#create}
   */
  public static WorkItemCreateRequest toCreateRequest(
          final WorkItemTemplate template,
          final String titleOverride,
          final String assigneeIdOverride,
          final String createdBy,
          final String callerRef) {

      final String title = (titleOverride != null && !titleOverride.isBlank())
              ? titleOverride
              : template.name;

      return new WorkItemCreateRequest(
              title,
              template.description,
              template.category,
              null, // formKey — not templated
              template.priority,
              assigneeIdOverride,
              template.candidateGroups,
              template.candidateUsers,
              template.requiredCapabilities,
              createdBy,
              template.defaultPayload,
              null, // claimDeadline — use config default unless template overrides
              null, // expiresAt — use config default unless template overrides
              null, // followUpDate
              null, // labels — applied separately so addLabel fires LABEL_ADDED events
              null, // confidenceScore — template-spawned items have no AI confidence
              callerRef,
              template.defaultClaimBusinessHours,
              template.defaultExpiryBusinessHours);
  }
  ```

  The existing Javadoc on `toCreateRequest` can be removed (it's now on the new 5-arg method).

- [ ] **Step 2.2: Run unit tests to confirm they pass**

  ```bash
  scripts/mvn-test runtime -Dtest=WorkItemTemplateServiceTest
  ```

  Expected: all 13 tests pass (11 existing + 2 new). Time: < 5s.

- [ ] **Step 2.3: Commit**

  ```bash
  git add runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java \
           runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateServiceTest.java
  git commit -m "feat(template): 5-arg toCreateRequest() with callerRef support

  Existing 4-arg overload delegates to new 5-arg with callerRef=null.
  All existing call sites unchanged.

  Refs #165"
  ```

---

### Task 3: Failing `@QuarkusTest` for `instantiate()` callerRef

**Files:**
- Create: `runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateInstantiateTest.java`

- [ ] **Step 3.1: Create the test file**

  ```java
  package io.casehub.work.runtime.service;

  import static org.assertj.core.api.Assertions.assertThat;

  import io.casehub.work.runtime.model.WorkItem;
  import io.casehub.work.runtime.model.WorkItemTemplate;
  import io.quarkus.test.junit.QuarkusTest;
  import jakarta.inject.Inject;
  import jakarta.transaction.Transactional;
  import org.junit.jupiter.api.Test;

  /**
   * Integration tests for WorkItemTemplateService.instantiate() callerRef overload.
   * Requires CDI + JPA — see WorkItemTemplateServiceTest for pure-unit mapping tests.
   */
  @QuarkusTest
  class WorkItemTemplateInstantiateTest {

      @Inject
      WorkItemTemplateService templateService;

      @Test
      void instantiate_setsCallerRef_onCreatedWorkItem() {
          final WorkItemTemplate template = persistedTemplate("IRB Review", null);
          final String callerRef = "case:550e8400-e29b-41d4-a716-446655440000/pi:irb-gate";

          final WorkItem workItem =
              templateService.instantiate(template, null, null, "system:engine", callerRef);

          assertThat(workItem.callerRef).isEqualTo(callerRef);
      }

      @Test
      void instantiate_multiInstanceTemplate_ignoresCallerRef() {
          final WorkItemTemplate template = persistedTemplate("Parallel Review", 3);
          final String callerRef = "case:550e8400-e29b-41d4-a716-446655440000/pi:review-gate";

          final WorkItem parent =
              templateService.instantiate(template, null, null, "system:engine", callerRef);

          assertThat(parent).isNotNull();
          assertThat(parent.callerRef).isNull();
      }

      @Transactional
      WorkItemTemplate persistedTemplate(final String name, final Integer instanceCount) {
          final WorkItemTemplate t = new WorkItemTemplate();
          t.name = name;
          t.candidateGroups = "reviewers";
          t.createdBy = "admin";
          t.instanceCount = instanceCount;
          t.persist();
          return t;
      }
  }
  ```

- [ ] **Step 3.2: Run tests to confirm they fail**

  ```bash
  scripts/mvn-test runtime -Dtest=WorkItemTemplateInstantiateTest
  ```

  Expected: both tests fail — `instantiate_setsCallerRef_onCreatedWorkItem` fails with
  `no suitable method found for instantiate(..., String)`.

---

### Task 4: Implement `instantiate()` 5-arg overload — make integration tests pass

**Files:**
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java`

- [ ] **Step 4.1: Add the delegating 4-arg overload and new 5-arg `instantiate()` method**

  The existing `@Transactional instantiate()` method becomes a one-liner delegate. Replace the entire existing `instantiate()` method body with:

  ```java
  @Transactional
  public WorkItem instantiate(
          final WorkItemTemplate template,
          final String titleOverride,
          final String assigneeIdOverride,
          final String createdBy) {
      return instantiate(template, titleOverride, assigneeIdOverride, createdBy, null);
  }

  /**
   * Instantiate a {@link WorkItemTemplate} into a new PENDING {@link WorkItem}.
   *
   * @param template the template to instantiate; must not be null
   * @param titleOverride optional title; defaults to template name
   * @param assigneeIdOverride optional direct assignee; overrides candidateGroups routing
   * @param createdBy the actor (user or system) triggering instantiation
   * @param callerRef opaque routing key for engine adapters; null for human-initiated creation.
   *                  Ignored for multi-instance templates — see casehubio/work#166.
   * @return the newly created PENDING WorkItem with all template defaults applied
   */
  @Transactional
  public WorkItem instantiate(
          final WorkItemTemplate template,
          final String titleOverride,
          final String assigneeIdOverride,
          final String createdBy,
          final String callerRef) {

      if (template.instanceCount != null) {
          if (callerRef != null) {
              LOG.warnf(
                  "callerRef '%s' ignored for multi-instance template %s — not yet supported (see casehubio/work#166)",
                  callerRef, template.id);
          }
          return multiInstanceSpawnService.get().createGroup(template, titleOverride, createdBy);
      }

      final WorkItemCreateRequest request =
          toCreateRequest(template, titleOverride, assigneeIdOverride, createdBy, callerRef);
      WorkItem workItem = workItemService.create(request);

      final List<WorkItemLabel> labels = parseLabels(template);
      for (final WorkItemLabel label : labels) {
          workItem = workItemService.addLabel(workItem.id, label.path, label.appliedBy);
      }

      return workItem;
  }
  ```

  Verify `LOG` is already declared in the class (`private static final Logger LOG = Logger.getLogger(WorkItemTemplateService.class)`). If not, add it after the class declaration.

- [ ] **Step 4.2: Run all runtime tests to confirm everything passes**

  ```bash
  scripts/mvn-test runtime
  ```

  Expected: all tests pass including the 2 new unit tests and 2 new integration tests. Time: < 60s.

- [ ] **Step 4.3: Install to local Maven repo so dependents can resolve the updated artifact**

  ```bash
  scripts/mvn-install runtime
  ```

- [ ] **Step 4.4: Commit**

  ```bash
  git add runtime/src/main/java/io/casehub/work/runtime/service/WorkItemTemplateService.java \
           runtime/src/test/java/io/casehub/work/runtime/service/WorkItemTemplateInstantiateTest.java
  git commit -m "feat(template): 5-arg instantiate() with callerRef support

  Existing 4-arg overload delegates to new 5-arg with callerRef=null.
  Multi-instance path logs a warning and ignores callerRef (see #166).
  All existing callers (WorkItemScheduleService, REST endpoint, tests) unchanged.

  Closes #165"
  ```
