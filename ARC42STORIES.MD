# casehub-work — ARC42STORIES.MD

**Spec:** Arc42Stories v0.1
**Profile:** CaseHub — Foundation tier
**Profile ref:** `../parent/docs/arc42stories-casehub-profile.md` · fallback: `https://raw.githubusercontent.com/casehubio/parent/main/docs/arc42stories-casehub-profile.md`
**Build position:** After casehub-platform-api (api module depends on it); core module has zero other casehubio deps; ledger module depends on casehub-ledger. Publishes before engine and all application-tier apps.
**Consumed by:** casehub-engine (casehub-work-adapter Quarkus extension), all application-tier harness apps
**Depends on:** `casehub-work-api` → casehub-platform-api. `casehub-work` core → zero casehubio deps. `casehub-work-ledger` module → casehub-ledger.

---

## §1 Introduction and Goals

### Description

`casehub-work` is the human task lifecycle foundation module. It models WorkItems — formal human task assignments with inbox management, SLA enforcement via configurable deadlines, delegation, group routing, and policy-driven escalation.

Core capabilities: `WorkItem` (human task entity with status, candidateGroups, claimDeadline, callerRef), `WorkItemService` (create, claim, complete, reject, cancel, extend), `ExpiryLifecycleService` (deadline enforcement — triggers SlaBreachPolicy and fires SlaBreachEvent), `SlaBreachPolicy` SPI (domain-defined escalation decision: Fail | EscalateTo | Extend | Chained), `BreachDecision` sealed interface, `WorkItemLifecycleEvent` (CDI event on status transitions), `casehub-work-adapter` (Quarkus extension that bridges WorkItemLifecycleEvent to engine PlanItem completion via HumanTaskScheduleHandler + WorkItemLifecycleAdapter).

`SlaBreachPolicy` lives in `casehub-work-api` — every consumer already has it on their classpath; the SPI is intrinsically work-bounded (ADR-0007 in casehub-devtown).

### Stakeholders

| Stakeholder | Interest |
|---|---|
| Application-tier harness apps | Consume this module's foundation primitives |
| Platform team | Maintain correctness and backward compatibility |
| LLM sessions (Claude) | Navigate architecture and implement extensions confidently |

### Quality Goals

| Goal | Scenario |
|---|---|
| SLA enforcement | Every WorkItem with a claimDeadline triggers ExpiryLifecycleService on expiry — no deadline silently ignored |
| Stateless escalation | SlaBreachPolicy reads current candidateGroups to detect tier — no external state storage needed for multi-tier escalation |
| Engine integration | casehub-work-adapter bridges WorkItemLifecycleEvent to engine PlanItem without requiring application code |

---

## §2 Constraints

### Platform
Java 21 (on Java 26 JVM), Quarkus 3.32.2. Published to GitHub Packages as `io.casehub:casehub-work:0.2-SNAPSHOT`.

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

### Dependencies
Resolved from GitHub Packages: `https://maven.pkg.github.com/casehubio/*`. CI authentication via `GITHUB_TOKEN`.

---

## §3 Context and Scope

🔲 C4Context diagram — add in dedicated session.

**Boundary rules:** casehub-work owns only the concerns described in §1. Domain logic (business rules, application-specific workflows) belongs in application-tier repos (devtown, aml, clinical, life, drafthouse, quarkmind).

See `../parent/docs/PLATFORM.md` Capability Ownership table for the full boundary map.

---

## §4 Solution Strategy

🔲 Core architectural patterns and module structure — complete in dedicated session from DESIGN.md and repo architecture.

---

## §5 Building Block View

🔲 C4Container diagram and module structure — complete in dedicated session.

---

## §6 Runtime View

🔲 Key runtime scenarios — complete in dedicated session.

---

## §7 Deployment View

Published to GitHub Packages. CI: GitHub Actions. No GraalVM native target (foundation modules).

---

## §8 Crosscutting Concepts

References to governing protocols:

| Concern | Protocol / Reference |
|---|---|
| Module structure | `docs/protocols/universal/module-tier-structure.md` |
| Flyway migrations | `docs/protocols/universal/flyway-migration-rules.md` |
| CDI displacement (`@DefaultBean`) | `docs/protocols/casehub/alternative-extension-patterns.md` |
| Capability ownership | `../parent/docs/PLATFORM.md` Capability Ownership table |
| Architectural patterns | `../parent/docs/ARCHITECTURE.md` |

🔲 Anti-patterns inline (Symptom → Cause → Fix) — complete in dedicated session. Must be present per profile rules; do not defer to AGENTIC-HARNESS-GUIDE.md reference alone.

---

## §9 Journeys and Chapters

🔲 **Bootstrap — full Journeys and Chapters pending dedicated session.**

Foundation modules use Chapters to track delivery milestones of the foundation capability itself (transport types, SPI increments, new adapters) rather than integrating external foundation modules.

Read `../parent/docs/arc42stories-spec.md §4 Solution Strategy` — "Foundation: Chapters = delivery milestones of the foundation capability itself."

The full §9 entry requires:
- Reading DESIGN.md (or equivalent) for the delivery arc
- Mapping milestones to Chapters
- Writing §9.4 Layer/Chapter entries with all 18 required subsections

---

## §10 Architectural Decisions

Per the Arc42Stories spec: only decisions not captured inline elsewhere. Should be sparse.

🔲 Cross-cutting decisions — complete in dedicated session.

---

## §11 Quality Requirements

See §1 Quality Goals for the primary scenarios.

🔲 Full quality requirements table — complete in dedicated session.

---

## §12 Risks and Technical Debt

🔲 Active risks and technical debt — complete in dedicated session.

---

## §13 Glossary

| Term | Definition |
|---|---|
| `WorkItem` | Human task entity with inbox, claim deadline (SLA), delegation, escalation, and completion lifecycle. |
| `SlaBreachPolicy` | SPI in casehub-work-api. Called by ExpiryLifecycleService when a WorkItem expires. Returns a BreachDecision. |
| `BreachDecision` | Sealed interface: Fail | EscalateTo | Extend | Chained. Models escalation response to SLA breach. |
| `casehub-work-adapter` | Quarkus extension — bridges WorkItemLifecycleEvent to engine PlanItem completion. HumanTaskScheduleHandler creates WorkItems from HumanTaskScheduleEvent; WorkItemLifecycleAdapter signals PlanItem. |
| `ExpiryLifecycleService` | Runs SlaBreachPolicy on expired WorkItems and fires SlaBreachEvent(context, leafDecision). |

---

*Bootstrap document — generated 2026-06-02. §3–§9 require a dedicated session to complete from repository source material.*
