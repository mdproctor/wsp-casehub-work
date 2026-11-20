---
layout: post
title: "Giving social cognition a database"
date: 2026-09-23
entry_type: note
subtype: diary
projects: [casehubio/blocks]
tags: [jpa, social-cognition, maven, jackson, spring-boot]
---

I came into this session expecting to pick up from where the previous session left off — Task 3 of the blocks-social-jpa plan, refactoring `StrategyLearningOrchestrator` to use the new `StrategyStore` SPI instead of reaching past it to `CbrRecordStore`.

What I didn't expect was spending the first hour untangling why blocks wouldn't compile.

## The phantom jar

The symptom was straightforward: `cannot find symbol: CbrRecordStore`. The class existed in `~/.m2/repository` — I checked. It was right there in the jar. But Maven wasn't using that jar.

The slot has its own `.m2` repo configured via `slot-settings.xml`, and Maven was resolving from there — not from the global repo. The slot's copy of `casehub-neocortex-memory-api` had been rebuilt from a neocortex checkout that had already landed a vocabulary rename (`CbrCase` → `CbrRecord`), while blocks still used the old names. Two jars, same coordinates, different contents. Classic SNAPSHOT divergence, but the slot-local repo made it invisible.

The fix was mechanical — copy the compatible jars from the global repo into the slot's `.m2`. But the diagnosis took longer than it should have because I was looking at the wrong jar the whole time.

## The actual work

With compilation restored, the refactoring went cleanly. `StrategyLearningOrchestrator` had six direct touchpoints with `CbrRecordStore` — two write paths in `doTick()`, a count query in `countAgentCases()`, and retrieval calls in both `doReflect()` and `doReflectAsync()`. Each one mapped directly onto the new `StrategyStore` SPI methods (`storeEvidence`, `evidenceCount`, `recentEvidence`) that the previous session had added. The `analyzeTrends()` and `summarizePerSubject()` methods needed a bridge — `EngagementEvidence` fields converted to `FeatureValue` maps for the existing `TrendAnalyzer` — but the mapping is straightforward. 2,180 tests green after the change.

Then the three new modules. `social-jpa-common` carries the framework-neutral pieces: five JPA entities, a Flyway V1 migration, Jackson `AttributeConverter`s for the JSON TEXT columns, and a `SocialJsonMapper` with polymorphic serialization for the `NarrativeFragment` sealed hierarchy. That last one surfaced a genuine gotcha — Jackson's mixin `@JsonTypeInfo` on a sealed interface doesn't fire during serialization via `writeValueAsString(Object)` because Java erases the declared type. The discriminator appears during deserialization (where you pass a `TypeReference`) but silently vanishes during serialization. The fix: either use `writerFor(TypeReference)` for the typed path, or register the mixin on each concrete subtype.

`social-jpa` provides the Quarkus CDI stores — four implementations with `@Alternative @Priority(3)`, displacing the `@DefaultBean` CBR stores when the JPA module is on the classpath. Zero configuration for consumers. 19 integration tests against H2 with Flyway.

`social-spring-jpa` mirrors the pattern for Spring Boot — five Spring Data repositories, four store implementations, and an `@AutoConfiguration` with `@ConditionalOnMissingBean`. Spring Boot 4.x made this slightly harder than expected: `@EntityScan` moved from `spring-boot-autoconfigure` to a new `spring-boot-persistence` module at `org.springframework.boot.persistence.autoconfigure`. And mixing Quarkus's `slf4j-jboss-logmanager` with Spring Boot's Logback on the test classpath caused the classic "LoggerFactory is not a Logback LoggerContext" crash — fixed by excluding the Quarkus logging from the `blocks-core` dependency.

## What this opens up

The CBR-backed social stores were a pragmatic starting point — serialize everything as feature vectors and query via similarity. It worked, but the abstraction leak was real: `StrategyLearningOrchestrator` was constructing `CbrFeatureRecord` instances and calling `cbrStore.store()` directly, bypassing the `StrategyStore` SPI entirely for engagement writes. The JPA stores close that gap. Every engagement data path now flows through the store abstraction, and consumers get SQL-queryable columns instead of opaque feature vectors.

Next is wiring `social-jpa` into wacky-manor as the first consumer — add the dependency, point Flyway at both migration locations, and the `@Alternative @Priority(3)` stores take over automatically.
