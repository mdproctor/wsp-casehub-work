---
title: "Closing the Frontend Gap — When Build Items Need a REST Endpoint"
date: 2026-10-29
entry_type: article
subtype: explanation
projects: [casehubio/agency]
author: mdp
tags: [quarkus, rest, deployment, build-items, frontend, cognition, casehub-mosaic]
---

# Closing the Frontend Gap — When Build Items Need a REST Endpoint

The mosaic frontend has had a cognition-view for weeks — thirteen typed
ConfigSpec interfaces rendering drive axes, mood baselines, personality
evolution parameters, the works. Beautiful cards with progress bars and
status badges. The only problem: no data source. The backend discovers
`CognitionDefinition` from YAML at build time, wraps it in a
`CognitionConfigBuildItem`, and then... nothing. The build item exists.
Nobody consumes it.

This is a pattern I've seen repeat across the agentic-yaml module. Four
build items — patterns, cognition, prompt optimisation, world config —
all discovered from `META-INF/*.yaml`, all produced by `AgenticYamlProcessor`
build steps, all sitting in the deployment pipeline with nowhere to go at
runtime. The frontend needs JSON. The backend has the data. The gap is the
plumbing between them.

## The Quarkus Deployment Pipeline as Data Source

The interesting design question isn't "how do we write a REST endpoint" —
it's where the data comes from. These aren't runtime-queried entities.
They're static configuration, loaded once at build time and immutable
for the lifetime of the application. A traditional approach would
deserialize the YAML at startup, register a CDI bean, and serialize to
JSON on every request. But that's three unnecessary steps when Quarkus
already parsed the YAML during augmentation.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 720 220" style="max-width:720px;font-family:'Inter','Segoe UI',system-ui,sans-serif">
  <defs>
    <marker id="ah" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto"><path d="M0,0 L8,3 L0,6" fill="#7f849c"/></marker>
  </defs>
  <rect x="0" y="0" width="720" height="220" rx="8" fill="#1e1e2e"/>
  <!-- Build time -->
  <text x="160" y="24" font-size="11" fill="#7f849c" text-anchor="middle" font-weight="600" letter-spacing="0.5">BUILD TIME</text>
  <rect x="20" y="36" width="130" height="44" rx="6" fill="#313244" stroke="#45475a"/>
  <text x="85" y="62" font-size="10" fill="#cdd6f4" text-anchor="middle">cognition.yaml</text>
  <rect x="20" y="90" width="130" height="44" rx="6" fill="#313244" stroke="#45475a"/>
  <text x="85" y="116" font-size="10" fill="#cdd6f4" text-anchor="middle">patterns/*.yaml</text>
  <rect x="20" y="144" width="130" height="44" rx="6" fill="#313244" stroke="#45475a"/>
  <text x="85" y="170" font-size="10" fill="#cdd6f4" text-anchor="middle">prompt-opt.yaml</text>
  <line x1="150" y1="58" x2="190" y2="108" stroke="#7f849c" marker-end="url(#ah)"/>
  <line x1="150" y1="112" x2="190" y2="108" stroke="#7f849c" marker-end="url(#ah)"/>
  <line x1="150" y1="166" x2="190" y2="118" stroke="#7f849c" marker-end="url(#ah)"/>
  <rect x="190" y="80" width="150" height="56" rx="6" fill="#313244" stroke="#89b4fa" stroke-width="1.5"/>
  <text x="265" y="104" font-size="10" fill="#89b4fa" text-anchor="middle" font-weight="600">AgenticYamlProcessor</text>
  <text x="265" y="120" font-size="9" fill="#7f849c" text-anchor="middle">@BuildStep</text>
  <line x1="340" y1="108" x2="380" y2="108" stroke="#7f849c" marker-end="url(#ah)"/>
  <rect x="380" y="80" width="120" height="56" rx="6" fill="#313244" stroke="#f5c2e7" stroke-width="1.5"/>
  <text x="440" y="100" font-size="10" fill="#f5c2e7" text-anchor="middle" font-weight="600">JSON.valueToTree</text>
  <text x="440" y="116" font-size="9" fill="#7f849c" text-anchor="middle">pre-serialize</text>
  <text x="440" y="130" font-size="9" fill="#7f849c" text-anchor="middle">all specs → String</text>
  <!-- Runtime -->
  <text x="600" y="24" font-size="11" fill="#7f849c" text-anchor="middle" font-weight="600" letter-spacing="0.5">RUNTIME</text>
  <line x1="500" y1="108" x2="540" y2="108" stroke="#7f849c" marker-end="url(#ah)"/>
  <rect x="540" y="72" width="160" height="72" rx="6" fill="#313244" stroke="#a6e3a1" stroke-width="1.5"/>
  <text x="620" y="96" font-size="10" fill="#a6e3a1" text-anchor="middle" font-weight="600">AgencySpecsResource</text>
  <text x="620" y="112" font-size="9" fill="#7f849c" text-anchor="middle">GET /api/agency/specs</text>
  <text x="620" y="128" font-size="9" fill="#7f849c" text-anchor="middle">returns pre-built String</text>
  <!-- Divider -->
  <line x1="520" y1="36" x2="520" y2="200" stroke="#45475a" stroke-dasharray="4,3"/>
</svg>

The approach: serialize everything to JSON at build time using Jackson's
`valueToTree` and `writeValueAsString`, pass the resulting string through
a Quarkus recorder, and serve it verbatim at runtime. Zero per-request
serialization. Zero runtime deserialization. The endpoint returns a
pre-built string — it's as fast as a static file serve.

## The Naming Mismatch

The one wrinkle was a field name divergence between backend and frontend.
The Java record uses `personality` as the field name (backed by
`PersonalityEvolutionConfigSpec`), but the TypeScript `CognitionConfig`
type uses `personalityEvolution`. Both names are defensible — the Java
side is terse, the frontend side is explicit.

`@JsonProperty("personalityEvolution")` handles the JSON output. But it
also changes the deserialization name, which breaks existing YAML files
that use `personality:`. The fix is `@JsonAlias("personality")` — Jackson
accepts the old name during deserialization while serializing to the new
name. One annotation pair, backward compatible, no consumer changes.

```java
public record CognitionDefinition(
        @Nullable DriveConfigSpec drive,
        @Nullable MoodConfigSpec mood,
        @JsonProperty("personalityEvolution") @JsonAlias("personality")
        @Nullable PersonalityEvolutionConfigSpec personality,
        // ... 10 more sections
```

## Pre-Serialization vs. Runtime Serialization

The decision to pre-serialize at build time rather than hold the Java
records at runtime was driven by a specific constraint: Quarkus
recorders.

Recorders work by replaying method calls at runtime — the build step
records a call like `recorder.createSpecsBean(data)`, and Quarkus
generates bytecode to replay it during application startup. The arguments
need to be serializable to bytecode. Simple types (strings, primitives,
lists of simple types) work. But `PatternDefinition` is a sealed
interface with nine subtypes, each containing nested sealed hierarchies
(`RoutingSpec`, `TerminationSpec`, `AgentRefSpec`). Asking the recorder
to serialize this graph would be fragile at best.

Pre-serializing to a JSON string at build time sidesteps the problem
entirely. The recorder receives a `String` — the simplest possible type.
Jackson's `@JsonProperty` annotations are respected during
`valueToTree`, so the `personalityEvolution` rename happens at the right
moment. And the runtime endpoint is trivial:

```java
@Path("/api/agency/specs")
@Produces(MediaType.APPLICATION_JSON)
public class AgencySpecsResource {

    private final AgencySpecsBean specsBean;

    @Inject
    public AgencySpecsResource(AgencySpecsBean specsBean) {
        this.specsBean = specsBean;
    }

    @GET
    public String get() {
        return specsBean.json();
    }
}
```

Constructor injection means the resource is directly unit-testable
without CDI — construct it with a known bean, assert the output. No
container startup, no mocking.

## The Endpoint Shape

Rather than one endpoint per spec type, a single `GET /api/agency/specs`
returns everything:

```json
{
  "patterns": [ ... ],
  "cognition": { "drive": { ... }, "personalityEvolution": { ... }, ... },
  "promptOptimisation": { "pipelines": { ... } },
  "world": null
}
```

Absent specs are `null`. The total payload for a typical deployment is a
few kilobytes — these are configuration declarations, not runtime data.
A single fetch is simpler for the frontend and avoids the overhead of
multiple requests for data that changes only on redeployment.

## What This Unlocks

The cognition-view in casehub-mosaic can now render real agent
configurations. Thirteen cognitive modules — drives, mood, personality
evolution, user models, strategy learning, mental models, narrative
synthesis, goal proposal, goal escalation, norm detection, collective
goals, retention, cognitive goals — each with typed config fields, axis
weight visualisations, and duration formatting. The view was ready. The
data wasn't. Now it is.

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 720 300" style="max-width:720px;font-family:'Inter','Segoe UI',system-ui,sans-serif">
  <rect x="0" y="0" width="720" height="300" rx="8" fill="#1e1e2e"/>
  <!-- Header -->
  <text x="20" y="30" font-size="12" fill="#7f849c" font-weight="600" letter-spacing="0.5">COGNITION</text>
  <text x="115" y="30" font-size="11" fill="#f5c2e7" font-weight="500">cognition</text>
  <text x="360" y="30" font-size="10" fill="#7f849c" text-anchor="middle">8 of 13 cognitive modules configured</text>
  <!-- Grid of cards -->
  <!-- Drive -->
  <rect x="20" y="46" width="160" height="70" rx="6" fill="rgba(245,194,231,0.04)" stroke="rgba(245,194,231,0.25)"/>
  <text x="36" y="64" font-size="10" fill="#cdd6f4" font-weight="600">⚡ Drive</text>
  <rect x="142" y="54" width="28" height="14" rx="3" fill="rgba(166,227,161,0.15)"/>
  <text x="156" y="64" font-size="8" fill="#a6e3a1" text-anchor="middle" font-weight="600">ON</text>
  <text x="36" y="80" font-size="9" fill="#7f849c" font-family="monospace">changeThreshold</text>
  <text x="170" y="80" font-size="9" fill="#cdd6f4" text-anchor="end" font-family="monospace">0.1</text>
  <text x="36" y="94" font-size="9" fill="#7f849c" font-family="monospace">maxIntensity</text>
  <text x="170" y="94" font-size="9" fill="#cdd6f4" text-anchor="end" font-family="monospace">1.0</text>
  <text x="36" y="108" font-size="9" fill="#7f849c" font-family="monospace">minIntensity</text>
  <text x="170" y="108" font-size="9" fill="#cdd6f4" text-anchor="end" font-family="monospace">0.0</text>
  <!-- Mood -->
  <rect x="190" y="46" width="160" height="70" rx="6" fill="rgba(245,194,231,0.04)" stroke="rgba(245,194,231,0.25)"/>
  <text x="206" y="64" font-size="10" fill="#cdd6f4" font-weight="600">🎭 Mood</text>
  <rect x="312" y="54" width="28" height="14" rx="3" fill="rgba(166,227,161,0.15)"/>
  <text x="326" y="64" font-size="8" fill="#a6e3a1" text-anchor="middle" font-weight="600">ON</text>
  <text x="206" y="80" font-size="9" fill="#7f849c" font-family="monospace">decayTimeConst</text>
  <text x="340" y="80" font-size="9" fill="#89b4fa" text-anchor="end" font-family="monospace">PT30M</text>
  <text x="206" y="94" font-size="9" fill="#7f849c" font-family="monospace">maxDisplacement</text>
  <text x="340" y="94" font-size="9" fill="#cdd6f4" text-anchor="end" font-family="monospace">0.8</text>
  <!-- Personality -->
  <rect x="360" y="46" width="160" height="70" rx="6" fill="rgba(245,194,231,0.04)" stroke="rgba(245,194,231,0.25)"/>
  <text x="376" y="64" font-size="10" fill="#cdd6f4" font-weight="600">🔄 Personality</text>
  <rect x="482" y="54" width="28" height="14" rx="3" fill="rgba(166,227,161,0.15)"/>
  <text x="496" y="64" font-size="8" fill="#a6e3a1" text-anchor="middle" font-weight="600">ON</text>
  <text x="376" y="80" font-size="9" fill="#7f849c" font-family="monospace">decayFactor</text>
  <text x="510" y="80" font-size="9" fill="#cdd6f4" text-anchor="end" font-family="monospace">0.95</text>
  <text x="376" y="94" font-size="9" fill="#7f849c" font-family="monospace">l2Ceiling</text>
  <text x="510" y="94" font-size="9" fill="#cdd6f4" text-anchor="end" font-family="monospace">0.3</text>
  <!-- Narrative -->
  <rect x="530" y="46" width="170" height="70" rx="6" fill="rgba(245,194,231,0.04)" stroke="rgba(245,194,231,0.25)"/>
  <text x="546" y="64" font-size="10" fill="#cdd6f4" font-weight="600">📖 Narrative</text>
  <rect x="662" y="54" width="28" height="14" rx="3" fill="rgba(166,227,161,0.15)"/>
  <text x="676" y="64" font-size="8" fill="#a6e3a1" text-anchor="middle" font-weight="600">ON</text>
  <text x="546" y="80" font-size="9" fill="#7f849c" font-family="monospace">maxEpisodes</text>
  <text x="690" y="80" font-size="9" fill="#cdd6f4" text-anchor="end" font-family="monospace">50</text>
  <text x="546" y="94" font-size="9" fill="#7f849c" font-family="monospace">maxThemes</text>
  <text x="690" y="94" font-size="9" fill="#cdd6f4" text-anchor="end" font-family="monospace">10</text>
  <!-- Second row - inactive cards -->
  <rect x="20" y="126" width="160" height="44" rx="6" fill="rgba(255,255,255,0.03)" stroke="rgba(255,255,255,0.08)"/>
  <text x="36" y="148" font-size="10" fill="#cdd6f4" font-weight="600">👤 User Model</text>
  <rect x="142" y="138" width="28" height="14" rx="3" fill="rgba(255,255,255,0.05)"/>
  <text x="156" y="148" font-size="8" fill="#585b70" text-anchor="middle" font-weight="600">OFF</text>
  <rect x="190" y="126" width="160" height="44" rx="6" fill="rgba(245,194,231,0.04)" stroke="rgba(245,194,231,0.25)"/>
  <text x="206" y="148" font-size="10" fill="#cdd6f4" font-weight="600">📈 Strategy</text>
  <rect x="312" y="138" width="28" height="14" rx="3" fill="rgba(166,227,161,0.15)"/>
  <text x="326" y="148" font-size="8" fill="#a6e3a1" text-anchor="middle" font-weight="600">ON</text>
  <rect x="360" y="126" width="160" height="44" rx="6" fill="rgba(245,194,231,0.04)" stroke="rgba(245,194,231,0.25)"/>
  <text x="376" y="148" font-size="10" fill="#cdd6f4" font-weight="600">🧠 Mental Model</text>
  <rect x="482" y="138" width="28" height="14" rx="3" fill="rgba(166,227,161,0.15)"/>
  <text x="496" y="148" font-size="8" fill="#a6e3a1" text-anchor="middle" font-weight="600">ON</text>
  <rect x="530" y="126" width="170" height="44" rx="6" fill="rgba(255,255,255,0.03)" stroke="rgba(255,255,255,0.08)"/>
  <text x="546" y="148" font-size="10" fill="#cdd6f4" font-weight="600">🎯 Goal Proposal</text>
  <rect x="662" y="138" width="28" height="14" rx="3" fill="rgba(255,255,255,0.05)"/>
  <text x="676" y="148" font-size="8" fill="#585b70" text-anchor="middle" font-weight="600">OFF</text>
  <!-- Third row -->
  <rect x="20" y="180" width="160" height="44" rx="6" fill="rgba(245,194,231,0.04)" stroke="rgba(245,194,231,0.25)"/>
  <text x="36" y="202" font-size="10" fill="#cdd6f4" font-weight="600">⬆ Escalation</text>
  <rect x="142" y="192" width="28" height="14" rx="3" fill="rgba(166,227,161,0.15)"/>
  <text x="156" y="202" font-size="8" fill="#a6e3a1" text-anchor="middle" font-weight="600">ON</text>
  <rect x="190" y="180" width="160" height="44" rx="6" fill="rgba(255,255,255,0.03)" stroke="rgba(255,255,255,0.08)"/>
  <text x="206" y="202" font-size="10" fill="#cdd6f4" font-weight="600">📏 Norm Detection</text>
  <rect x="312" y="192" width="28" height="14" rx="3" fill="rgba(255,255,255,0.05)"/>
  <text x="326" y="202" font-size="8" fill="#585b70" text-anchor="middle" font-weight="600">OFF</text>
  <rect x="360" y="180" width="160" height="44" rx="6" fill="rgba(255,255,255,0.03)" stroke="rgba(255,255,255,0.08)"/>
  <text x="376" y="202" font-size="10" fill="#cdd6f4" font-weight="600">🤝 Collective Goal</text>
  <rect x="482" y="192" width="28" height="14" rx="3" fill="rgba(255,255,255,0.05)"/>
  <text x="496" y="202" font-size="8" fill="#585b70" text-anchor="middle" font-weight="600">OFF</text>
  <rect x="530" y="180" width="170" height="44" rx="6" fill="rgba(245,194,231,0.04)" stroke="rgba(245,194,231,0.25)"/>
  <text x="546" y="202" font-size="10" fill="#cdd6f4" font-weight="600">💾 Retention</text>
  <rect x="662" y="192" width="28" height="14" rx="3" fill="rgba(166,227,161,0.15)"/>
  <text x="676" y="202" font-size="8" fill="#a6e3a1" text-anchor="middle" font-weight="600">ON</text>
  <!-- Fourth row -->
  <rect x="20" y="234" width="160" height="44" rx="6" fill="rgba(255,255,255,0.03)" stroke="rgba(255,255,255,0.08)"/>
  <text x="36" y="256" font-size="10" fill="#cdd6f4" font-weight="600">💡 Cognitive Goal</text>
  <rect x="142" y="246" width="28" height="14" rx="3" fill="rgba(255,255,255,0.05)"/>
  <text x="156" y="256" font-size="8" fill="#585b70" text-anchor="middle" font-weight="600">OFF</text>
</svg>

More broadly, this pattern — build-time pre-serialization for static
config — applies to any Quarkus extension that needs to expose
deployment-discovered metadata via REST. The YAML is the source of
truth, the deployment pipeline is the parser, and the endpoint is just
a string pass-through. No runtime cost, no serialization complexity,
no impedance mismatch between the build model and the runtime model.
