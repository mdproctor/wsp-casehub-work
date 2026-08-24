## D1: Reflection over JavaParser

**Choice:** Use Java reflection for the model-canonical generator (both SchemaWriter and future TypeScriptWriter)
**Alternatives:**
- JavaParser — parses source files for richer info (Javadoc, builder patterns), but adds complexity (source file paths, CombinedTypeSolver for cross-repo types) without capability reflection doesn't already provide
- Hybrid (JavaParser for engine, reflection fallback for cross-repo) — two code paths for no gain
**Rationale:** Java 17+ reflection handles everything: records via `getRecordComponents()`, sealed interfaces via `getPermittedSubclasses()`, generics via `getGenericType()`, Jakarta Validation annotations (runtime-retained). Cross-repo types resolve naturally from the classpath. Descriptions come from annotations (`@JsonPropertyDescription`), not Javadoc.
**Trade-offs:** No Javadoc access — but descriptions via annotations are explicit, versioned, and don't rot. Acceptable trade.
**Sources:** CaseDefinition.java (1019 lines, references types from 4 repos), CasehubRuleFactory.java (existing codegen), spec 2026-08-23-typescript-programming-model-design.md
**Exploration:** quick
**Status:** captured
