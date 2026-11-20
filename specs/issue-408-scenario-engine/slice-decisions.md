# Slice Map Decisions

## SD1: Progressive slice structure

**Choice:** 6 thin slices, each teaching a specific Blocks agentic-AI pattern grouping
**Alternatives:**
- Fewer, larger slices covering more patterns each — harder to learn from, overwhelming
- One comprehensive example app — defeats the purpose of progressive teaching
**Rationale:** People learn platform capabilities one pattern at a time. Each slice is the thinnest possible demonstration of that pattern. Full applications (devtown, aml, clinical) exist for comprehensive reference.
**Exploration:** quick
**Status:** captured

## SD2: Slice map

**Choice:**
1. Helpdesk — Case, Worker, WorkItem, SEQUENCE, basic routing
2. Code/doc review — DEBATE, conversation protocol, convergence, epistemic ground, speech acts
3. Incident response — SUPERVISOR, oversight gates, trust scoring, routing rationale
4. Project planning — HTN, decomposition, compound plans, DAG execution, sub-cases
5. Multi-agent collaboration — Agent Mesh, channel semantics, mesh participation, VOTING
6. Summarisation & observation — tiered summarisation, event accumulation, observation pipeline

**Rationale:** Natural groupings — patterns within each slice need each other to make sense (e.g., DEBATE needs the conversation model, SUPERVISOR needs trust). Progressive complexity from foundation to advanced.
**Exploration:** quick
**Status:** captured

## SD3: Slice 1 learning points

**Choice:** Six concepts for the helpdesk slice:
1. A Case is a defined unit of work (CaseDefinition YAML, engine runs it)
2. AI is just another Worker (same abstraction as human workers)
3. Bindings connect triggers to capabilities (declarative, not imperative)
4. Work Items are the human handoff (queue → claim → resolve)
5. Routing is a platform decision (engine picks the worker)
6. Two perspectives, one system (end-user view + ops view)

**Rationale:** Enough to understand the platform's core model without overwhelming. Trust, oversight, and advanced patterns deferred to later slices.
**Exploration:** quick
**Status:** captured

## SD4: Deployment model

**Choice:** Single embedded Quarkus deployment, all in-memory, no external services
**Alternatives:**
- Multi-service deployment — realistic but slow startup, complex setup
- Docker Compose — portable but heavyweight for a teaching demo
**Rationale:** Fast startup, instant teardown, runnable with `mvn quarkus:dev`. Engine, work, and workers all in-process. Ephemeral state matches the scenario-driven model.
**Exploration:** quick
**Status:** captured

## SD5: Dependency approach

**Choice:** Maven dependencies from existing platform repos (casehub-engine, casehub-work, casehub-worker) — already set up via `${casehub.version}` and GitHub packages repo
**Rationale:** The examples repo POM already has the pattern. Just add dependency entries.
**Exploration:** quick
**Status:** captured

## SD6: Build vs borrow

**Choice:** Thinnest possible slice. Borrow logic and UIs from full applications (devtown, aml, clinical) rather than building bespoke demo code.
**Rationale:** The slices teach the platform, not the domain. Domain logic is a commodity — reuse what exists. The platform wiring is the lesson.
**Exploration:** quick
**Status:** captured

## SD7: Trust deferred

**Choice:** Trust scoring, graduated autonomy, oversight gates belong in slice 3 (incident response), not slice 1
**Rationale:** Trust requires accumulated data — observations, scores building over time. Can't demonstrate meaningfully in a 4-step scenario. Slice 1 uses deterministic routing.
**Exploration:** quick
**Status:** captured
