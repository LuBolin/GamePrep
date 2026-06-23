# Build Farm And Release Failure Workbook

Version target: **UE5.3-UE5.6**. AutomationTool, BuildGraph XML tasks, RunUAT flags, cook/stage/package layout, symbols, platform signing and CI conventions are branch/platform/studio-sensitive. Use this workbook as a debugging and interview-practice guide, then verify exact commands against the target engine and build farm. [SRC-BUILD-015] [SRC-BUILD-016] [SRC-BUILD-017] [SRC-ASSET-005] [SRC-BUILD-018]

## Purpose

This workbook turns release automation into concrete failure diagnosis. A strong candidate can explain:

- which phase failed: build, cook, stage, package, deploy, launch, smoke, symbolication or artifact archive;
- what the first causal error was;
- which artifact proves the claim;
- why the fix is not just "rerun CI";
- how to prevent recurrence with a gate or manifest.

## Release Evidence Packet

Every build-farm failure report should include:

| Field | Examples |
|---|---|
| Build identity | engine branch, project revision, changelist, build ID |
| Target | platform, configuration, target type, client/server/editor/program |
| Tooling | RunUAT command, BuildGraph XML path/target, SDK/NDK/Xcode/toolchain |
| Inputs | maps, cook filters, plugins, feature flags, Primary Asset rules |
| Outputs | package path, staged manifest, container/chunk manifest, symbols |
| Logs | UBT, UHT, cook, stage/package, platform deploy, crash/log tail |
| Environment | clean/warm workspace, cache/DDC policy, agent ID, credentials |
| Failure phase | build/cook/stage/package/deploy/launch/runtime/archive |
| Owner | gameplay, content, tools, build, rendering, platform, infrastructure |
| Recurrence guard | validation rule, smoke test, BuildGraph dependency, manifest diff |

## Scenario 1: RunUAT Reports A Generic Failure

**Symptom:** CI says `AutomationTool exiting with ExitCode=Error_Unknown`, and the team debates build-system changes without reading the full log.

**Likely phase:** hidden first causal error.

**First evidence:**

- full AutomationTool log;
- UBT/UHT/cook sub-log;
- last successful named phase;
- first `Error:` before wrapper exit;
- command line and environment.

**Investigation steps:**

1. Do not start at the final wrapper error.
2. Search upward for the first causal error.
3. Classify the failing phase before assigning owner.
4. Extract the minimum reproduction command for that phase.
5. Add a CI log summary that links final failure to first causal line.

**Fix pattern:**

Build reporting should preserve phase and first-cause context:

```text
Final: BuildCookRun failed
Cause: Cook failed loading /Game/UI/WBP_InventoryIcon
Owner: UI/content
Evidence: cook.log line 18342, asset manager audit, package manifest
```

**Interview debrief:**

> I would not debug the wrapper. I would identify the first causal phase and use the phase-specific log/artifact to assign ownership.

**Sources:** [SRC-BUILD-016] [SRC-BUILD-017]

## Scenario 2: BuildGraph Node Order Is Wrong

**Symptom:** A package job sometimes runs before validation or HLOD generation, depending on agent scheduling.

**Likely phase:** BuildGraph dependency modeling.

**First evidence:**

- BuildGraph XML target graph;
- node dependencies;
- produced/consumed artifacts;
- job timeline;
- missing or stale output.

**Hypotheses:**

1. A node consumes output without declaring dependency.
2. Two nodes write to the same artifact path.
3. A label/aggregate target hides missing dependency.
4. Agent-local files are assumed to exist across nodes.

**Investigation steps:**

1. Draw node input/output artifacts.
2. Add explicit dependency from consumer to producer.
3. Make artifacts path/versioned and archived between agents.
4. Fail fast if expected input artifact is absent or stale.
5. Add a graph dry-run or report view where supported.

**Fix pattern:**

BuildGraph should encode artifact flow, not just command order in someone's head.

**Sources:** [SRC-BUILD-015]

## Scenario 3: Clean Agent Fails, Developer Machine Passes

**Symptom:** Developer can package locally, but a clean CI agent fails during compile, cook or launch.

**Likely phase:** hidden local dependency.

**First evidence:**

- clean workspace status;
- generated/intermediate/binaries reliance;
- environment variables;
- SDK/toolchain versions;
- plugin binaries and staged third-party files;
- DDC/cache policy.

**Hypotheses:**

1. Local `Binaries` or `Intermediate` hides missing source/build rule.
2. Unity build hides missing include or transitive dependency.
3. Third-party runtime file exists locally but is not staged.
4. Cook relies on editor-loaded loose content.
5. DDC/cache hides shader/derived-data failure.

**Investigation steps:**

1. Delete local intermediates or use clean agent reproduction.
2. Build relevant targets in non-unity or stricter include mode if project policy supports it.
3. Compare staged files and plugin runtime dependencies.
4. Run cook from explicit map/asset roots.
5. Turn the missing dependency into a validation or staging rule.

**Interview debrief:**

> A clean-agent failure is often the first honest build. I would not copy local generated files into CI; I would fix the missing declared dependency.

**Sources:** [SRC-BUILD-001] [SRC-BUILD-002] [SRC-BUILD-017]

## Scenario 4: Editor-Only Module Reaches Shipping

**Symptom:** Shipping build fails to link or launches with missing class/package from an editor module.

**Likely phase:** module/plugin boundary.

**First evidence:**

- `.Build.cs` dependencies;
- plugin/module descriptor types;
- generated build target;
- staged package content;
- runtime asset references to editor-only classes.

**Hypotheses:**

1. Runtime module depends on UnrealEd/PropertyEditor/AssetTools.
2. Runtime asset references a class defined only in an Editor module.
3. `WITH_EDITOR` guards code but `.Build.cs` still declares an editor dependency.
4. plugin descriptor marks module type/loading phase incorrectly.

**Investigation steps:**

1. Build Game/Server/Shipping, not only Editor.
2. Inspect runtime module dependencies.
3. Audit assets for editor-only class references.
4. Move editor functionality to an Editor module and keep runtime schema clean.
5. Add package smoke test for editor exclusion.

**Sources:** [SRC-BUILD-002] [SRC-BUILD-005] [SRC-BUILD-009]

## Scenario 5: Third-Party Runtime Library Missing

**Symptom:** Packaged game runs on developer machine but fails on a clean test machine with missing DLL/shared library or plugin binary.

**Likely phase:** staging/runtime dependency.

**First evidence:**

- package/stage manifest;
- plugin descriptor;
- external module Build.cs;
- platform runtime library path;
- launch log or OS loader error.

**Hypotheses:**

1. Library linked for build but not staged.
2. RuntimeDependency path is platform-specific and wrong.
3. Developer machine has global install hiding the missing file.
4. Shipping config strips or excludes a needed plugin/binary.

**Investigation steps:**

1. Launch on a clean machine/device.
2. Compare expected and staged runtime files.
3. Fix Build.cs/runtime dependency or staging rule.
4. Archive staged manifest and validate before smoke launch.

**Sources:** [SRC-BUILD-003] [SRC-BUILD-005] [SRC-BUILD-017]

## Scenario 6: Asset Exists In Editor But Missing In Package

**Symptom:** A UI icon, weapon mesh or event asset is missing only in packaged build.

**Likely phase:** cook inclusion/dependency closure.

**First evidence:**

- missing asset path;
- hard/soft reference chain;
- Primary Asset rules/labels/chunks;
- cook log and manifest;
- package container/chunk membership.

**Hypotheses:**

1. Soft reference not managed by Asset Manager/cook rule.
2. Rare path only loaded in editor preview.
3. redirector or moved asset hides path in editor.
4. chunk/install bundle rule excludes it from the tested package.

**Investigation steps:**

1. Reproduce in clean packaged build.
2. Determine intended cook root.
3. Use Asset Manager/Primary Asset or explicit cook rule as appropriate.
4. Verify container/chunk manifest.
5. Add packaged smoke for the rare path.

**Sources:** [SRC-ASSET-001] [SRC-ASSET-003] [SRC-ASSET-005] [SRC-ASSET-006] [SRC-ASSET-007]

## Scenario 7: Symbols Do Not Match Crash

**Symptom:** Forced packaged crash generates a report, but the callstack is unsymbolicated or points to wrong functions.

**Likely phase:** artifact/symbol contract.

**First evidence:**

- build ID;
- package path;
- symbol archive path;
- crash context/log;
- exact executable/binary metadata;
- symbol upload/lookup logs.

**Hypotheses:**

1. Symbols from a different build/configuration were archived.
2. Crash reporter endpoint/path is wrong.
3. Package was rebuilt after symbols were generated.
4. symbols were not uploaded or were stripped unexpectedly.

**Investigation steps:**

1. Force a known packaged crash.
2. Verify build ID and symbol archive match.
3. Symbolicate locally if possible.
4. Add a release gate that fails if forced crash cannot symbolicate.

**Sources:** [SRC-BUILD-018] [SRC-BUILD-017]

## Scenario 8: CI Cache Hides A Build Error

**Symptom:** Incremental CI passes, but nightly clean or new branch build fails.

**Likely phase:** stale cache/intermediate dependency.

**First evidence:**

- clean versus incremental job logs;
- cached directories;
- generated headers;
- unity/non-unity difference;
- DDC/build cache policy.

**Hypotheses:**

1. Generated header or stale object file masks missing dependency.
2. Unity build hides include order/dependency issue.
3. cache key does not include toolchain/plugin/config changes.
4. DDC hides derived-data generation failure.

**Investigation steps:**

1. Compare clean and incremental command lines.
2. Remove cache and reproduce.
3. Fix missing include/dependency/generation input.
4. Adjust cache key or add periodic clean job.

**Sources:** [SRC-BUILD-001] [SRC-BUILD-002] [SRC-BUILD-013] [SRC-ASSET-008]

## Scenario 9: Package Launch Smoke Passes Wrong Map

**Symptom:** CI package smoke passes, but the intended gameplay map crashes on startup.

**Likely phase:** smoke scenario coverage.

**First evidence:**

- launch command line;
- map list;
- game default map/config;
- smoke test readiness marker;
- packaged logs.

**Hypotheses:**

1. Smoke test launches frontend or blank map only.
2. map inclusion differs between local and CI.
3. readiness marker fires before map content is actually ready.
4. test does not exercise plugin/content path that failed.

**Investigation steps:**

1. Name the intended release-critical maps.
2. Add explicit map launch commands.
3. Add readiness marker after gameplay map load and first interactive state.
4. Archive map/scenario in manifest.

**Sources:** [SRC-ASSET-005] [SRC-BUILD-017] [SRC-PERF-011]

## Scenario 10: Android Package Fails After SDK Change

**Symptom:** Build farm starts failing Android packaging after SDK/NDK/Gradle/toolchain update.

**Likely phase:** platform toolchain.

**First evidence:**

- exact SDK/NDK/JDK/Gradle/AGP versions;
- Unreal target version requirements;
- build agent environment;
- package signing config;
- first causal platform-tool error.

**Investigation steps:**

1. Pin toolchain versions in build metadata.
2. Compare against Unreal's target setup requirements.
3. Reproduce on clean agent.
4. Update project/toolchain together; do not let one agent auto-upgrade silently.
5. Add preflight check for expected versions.

**Sources:** [SRC-PLAT-001] [SRC-PLAT-004] [SRC-PLAT-007]

## Scenario 11: Telemetry Artifact Missing

**Symptom:** Performance job passes/fails but has no CSV, trace, screenshot or profile proof attached.

**Likely phase:** test/automation artifact harvest.

**First evidence:**

- command line enabling telemetry;
- output paths;
- app crash or timeout before flush;
- runner artifact pull logs;
- run manifest.

**Hypotheses:**

1. telemetry was never enabled.
2. output path differs on device/build agent.
3. app crashed before flush.
4. runner did not pull artifacts.
5. artifact was overwritten by another run.

**Investigation steps:**

1. Make artifact paths part of run manifest.
2. Add scenario start/end markers and flush policy.
3. Fail the job if required artifact is missing.
4. Attach minimal log tail when artifact capture fails.

**Sources:** [SRC-PERF-009] [SRC-PERF-011] [SRC-PLAT-009]

## Scenario 12: Flaky Agent Or Product Failure?

**Symptom:** One build agent intermittently fails packaging or smoke launch.

**Likely phase:** product versus infrastructure classification.

**First evidence:**

- same build on different agent;
- same agent on known-good build;
- disk/network/credentials/device availability;
- deterministic product logs;
- repeated failure signature.

**Classification:**

- **Product failure:** stable cook, package, crash or smoke failure tied to build/content.
- **Build infrastructure:** agent path, disk, credentials, network, cache or tool install problem.
- **Flaky device/lab:** install, launch, connection or thermal/device-health issue across builds.

**Fix pattern:**

Rerun only for defined infrastructure classes. Do not rerun deterministic product crashes until green.

**Sources:** [SRC-BUILD-015] [SRC-BUILD-016] [SRC-PLAT-009]

## Interview Drill Questions

Answer each with phase, evidence, owner and recurrence guard:

1. `RunUAT` exits with a generic error. Where do you start?
2. Package works locally but not on a clean agent.
3. Shipping build links an editor-only symbol.
4. Third-party DLL is missing only on QA machines.
5. A crash report arrives without symbols.
6. Rare soft-referenced content is missing in package.
7. CI smoke passes the frontend but not the gameplay map.
8. Android packaging fails after an SDK update.
9. Performance gate has no CSV artifact.
10. One agent fails intermittently.

## Strong Answer Pattern

> I classify the failure phase first, then find the first causal log line and the artifact that proves it. I do not treat a final RunUAT wrapper error or red CI badge as the diagnosis. The fix should update the build graph, manifest, validation rule, package staging rule or smoke scenario so the same class of failure is caught earlier with an owner.

## Source Notes

- `SRC-BUILD-015`, `SRC-BUILD-016` and `SRC-BUILD-017` anchor BuildGraph, AutomationTool and build/cook/package/deploy/run operations.
- `SRC-ASSET-005`, `SRC-ASSET-006`, `SRC-ASSET-007` and `SRC-ASSET-008` anchor package/cook/chunk/redirector/DDC evidence.
- `SRC-BUILD-018` anchors crash reporting and symbolication evidence.
- `SRC-PLAT-009`, `SRC-PERF-009` and `SRC-PERF-011` connect build farm output to target automation and telemetry harvest.
