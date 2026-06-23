# Packaged Performance, BuildGraph, and World Partition Proof

See also: [[ue_profiling_optimisation]], [[ue_build_modules_plugins_tools]], [[ue_world_partition_large_world_pipeline]], [[ue_device_lab_automation]], [[ue_packaged_performance_build_worldpartition_workbook]].

Version target: **UE5.3-UE5.6**. This chapter is source- and branch-sensitive. It does not claim that the current repository contains a runnable Unreal project or engine branch. It defines the proof packet a candidate or team should produce on a real target branch for packaged performance gates, BuildGraph/RunUAT release automation, and World Partition/HLOD builder output. Verify command names, flags, trace channels, builder classes, artifact locations and platform behaviour against the installed engine and project automation. [SRC-PERF-003] [SRC-PERF-009] [SRC-PERF-010] [SRC-BUILD-015] [SRC-BUILD-016] [SRC-BUILD-017] [SRC-WORLD-003]

## Scope

This chapter connects three areas that are often studied separately:

- packaged performance gates using CSV, targeted traces, memory/LLM checkpoints and device-profile proof;
- reproducible release automation using RunUAT, BuildGraph, artifacts, symbols and forced-crash evidence;
- large-world automation using World Partition builder commandlets, HLOD output, cook/package proof and packaged traversal.

The goal is not to memorise commands. The goal is to prove that a branch can build, cook, package, launch, traverse a representative world, measure target performance, catch known regressions, and produce enough evidence for another engineer to reproduce the result. [SRC-BUILD-015] [SRC-BUILD-016] [SRC-BUILD-017] [SRC-ASSET-010]

See also: [[ue_profiling_optimisation]], [[ue_build_modules_plugins_tools]], [[ue_world_partition_large_world_pipeline]], [[ue_device_lab_automation]].

## Proof Packet Overview

A target proof packet should be a small directory or build artifact set, not a prose claim. It should contain:

```text
proof_packet/
  manifest.yaml
  runuat_or_buildgraph/
    command_lines.txt
    buildgraph.xml
    graph_parameters.txt
  build_logs/
    ubt.log
    cook.log
    package.log
    smoke.log
  package/
    archive_path.txt
    cooked_manifest.txt
    staged_manifest.txt
    symbols_path.txt
  world_partition/
    builder_command.txt
    hlod_builder.log
    changed_or_stale_hlod_report.txt
    traversal_readiness.log
  performance/
    csv/
    insights/
    memory_or_llm/
    cvar_profile_dump.txt
    thresholds.yaml
  failures/
    injected_failures.md
    fixed_runs.md
  report.md
```

The minimum standard is that every important claim in the report points to an artifact: build identity, package identity, target settings, scenario, metrics, HLOD builder output, staged content, and the failing or passing gate.

## Manifest Contract

The manifest is the spine of the proof. Without it, logs and CSV files cannot be trusted later.

```yaml
proof_id: wp_perf_release_2026_06_23_001
engine:
  branch: ue5_6_project_branch
  changelist_or_commit: "123456"
project:
  branch: release_candidate
  changelist_or_commit: "abcdef0"
build:
  target: Game
  configuration: Development
  platform: Win64
  build_id: "RC_001"
  archive_root: "\\\\buildshare\\Project\\RC_001"
automation:
  mode: BuildGraph
  graph_file: Build/Graph/ProjectRelease.xml
  graph_target: PackageAndProfile
  runuat_help_captured: true
content:
  maps:
    - /Game/Maps/OpenWorld_Main
  cook_mode: clean_by_the_book
  pak_or_iostore_policy: target_project_default
world_partition:
  map: /Game/Maps/OpenWorld_Main
  hlod_builder_command: captured_in_world_partition/builder_command.txt
  data_layer_policy: docs/world_state_policy.md
performance:
  scenarios:
    - Frontend_Inventory_1000
    - OpenWorld_Traversal_A
    - CombatPeak_64AI
  csv_repetitions: 3
  warmup_seconds: 30
  sample_seconds: 120
  active_device_profile_proof: performance/cvar_profile_dump.txt
symbols:
  symbol_archive: "\\\\buildshare\\Project\\RC_001\\symbols"
  forced_crash_report: failures/forced_crash_symbolication.md
result:
  status: pass
  known_risks:
    - console profiling lane not included in this public packet
sources:
  - SRC-BUILD-015
  - SRC-BUILD-016
  - SRC-BUILD-017
  - SRC-PERF-009
  - SRC-WORLD-003
```

Strong interview signal: the candidate says which fields are branch-specific and which ones are non-negotiable. `RunUAT` flags, trace channels and HLOD builder names can vary; build ID, scenario identity, target platform, command line, artifact links and pass/fail thresholds cannot be vague.

## Target-Branch Audit Before Running The Proof

Run this audit before spending time on long builds.

| Area | What to prove | Evidence |
|---|---|---|
| RunUAT | target branch supports the intended BuildCookRun/archive/deploy/run arguments | captured `RunUAT` help or internal wrapper docs |
| BuildGraph | graph file parses, parameters resolve, artifact paths are stable | dry-run or small graph execution log |
| Cook scope | maps, Primary Assets, plugin content and generated content are intentionally included | cook settings, Asset Manager rules, package manifest |
| World Partition | map is a World Partition map and expected builder commandlet exists | editor commandlet help/log output |
| HLOD | HLOD layers are configured and builder output is generated or deliberately rejected | HLOD builder log and proxy artifact report |
| Device profile | expected profile and scalability CVars are active in packaged build | runtime dump, screenshot/log |
| CSV/trace | CSV categories and trace channels exist for target config | one tiny packaged smoke capture |
| Symbols/crash | symbols are retained and a packaged forced crash can be matched to them | symbolicated crash report |

If a row cannot be proven, do not hide it. Mark it as "not supported on this branch", "internal wrapper replaces public command", or "out of scope for this packet" with evidence.

## Packaged Performance Gate Proof

### Why Packaged Proof Is Different

Editor captures answer useful development questions, but release gates need packaged proof because target builds differ in content layout, RHI, shader/PSO caches, device profiles, storage, memory pressure, platform services and automation startup. A packaged build can be faster in core CPU code yet worse in first-use shader state, cooked asset layout, streaming, device-profile quality, or target storage. [SRC-PERF-003] [SRC-PERF-009]

The proof has four layers:

1. **Identity:** exact build, platform, config, map, scenario and settings.
2. **Signal:** stable lightweight metrics, usually CSV plus logs.
3. **Cause:** short targeted Insights, Memory Insights, GPU/platform capture, or LLM evidence only when the signal fails.
4. **Decision:** thresholds, owners, tolerance bands and the artifact needed to act.

See also: [[ue_rendering_graphics_performance]], [[ue_platform_constraints]], [[ue_android_platform_profiling_workbook]], [[ue_apple_console_profiling_workbook]].

### Scenario Shape

A good scenario is deterministic enough to compare, but realistic enough to catch production cost.

```yaml
scenario_id: OpenWorld_Traversal_A
owner: World/Performance
map: /Game/Maps/OpenWorld_Main
entry:
  mode: packaged
  command_line: "-ExecCmds=Automation RunPerfScenario OpenWorld_Traversal_A"
warmup:
  seconds: 30
  readiness:
    - map_loaded
    - device_profile_dumped
    - streaming_source_ready
measurement:
  seconds: 120
  path: spline_camera_town_to_outpost
  repetitions: 3
metrics:
  blocking:
    p95_frame_ms: 18.0
    p99_frame_ms: 28.0
    hitches_over_50ms: 0
    first_interactive_seconds: 12.0
  reporting:
    game_thread_p95_ms: true
    draw_thread_p95_ms: true
    gpu_p95_ms: true
    llm_total_mb: true
    loaded_wp_cells: true
    hlod_transition_hitches: true
artifacts:
  - csv
  - log
  - cvar_profile_dump
  - screenshot_or_video
escalation:
  on_cpu_fail: short_timing_insights
  on_memory_fail: memory_insights_plus_llm
  on_gpu_fail: gpu_visualiser_or_platform_capture
```

This is stronger than "run around and see if FPS is OK" because it defines warm-up, readiness, measurement window, thresholds and escalation.

### CSV Gate Implementation Sketch

Exact APIs and automation hooks vary. The stable design is:

```cpp
// Schematic only. Verify automation, CSV and trace APIs in the target branch.
USTRUCT()
struct FPerfGateScenario
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere)
    FName ScenarioId;

    UPROPERTY(EditAnywhere)
    float WarmupSeconds = 30.0f;

    UPROPERTY(EditAnywhere)
    float SampleSeconds = 120.0f;

    UPROPERTY(EditAnywhere)
    float P95FrameBudgetMs = 18.0f;

    UPROPERTY(EditAnywhere)
    int32 MaxHitchesOver50Ms = 0;
};

UCLASS()
class UPerfGateRunner : public UObject
{
    GENERATED_BODY()

public:
    void StartScenario(const FPerfGateScenario& Scenario)
    {
        ActiveScenario = Scenario;

        UE_LOG(LogTemp, Display, TEXT("PERF_GATE_START Scenario=%s"), *Scenario.ScenarioId.ToString());
        DumpBuildAndRuntimeIdentity();
        DumpAppliedDeviceProfileAndCVars();

        // Prefer project wrappers so command spelling is isolated to one place.
        ExecuteConsoleCommand(TEXT("csvprofile start"));
        SetTimer(Scenario.WarmupSeconds, [this]() { BeginSampleWindow(); });
    }

private:
    FPerfGateScenario ActiveScenario;

    void BeginSampleWindow()
    {
        TRACE_BOOKMARK(TEXT("PerfGate_Sample_Start"));
        UE_LOG(LogTemp, Display, TEXT("PERF_GATE_SAMPLE_START Scenario=%s"), *ActiveScenario.ScenarioId.ToString());
        SetTimer(ActiveScenario.SampleSeconds, [this]() { EndScenario(); });
    }

    void EndScenario()
    {
        TRACE_BOOKMARK(TEXT("PerfGate_Sample_End"));
        ExecuteConsoleCommand(TEXT("csvprofile stop"));
        UE_LOG(LogTemp, Display, TEXT("PERF_GATE_END Scenario=%s"), *ActiveScenario.ScenarioId.ToString());
        RequestApplicationExit();
    }
};
```

The runner does not need to know every profiler detail. It must make scenario boundaries, runtime settings and output paths machine-readable.

### Gate Result Template

```markdown
## Gate Result: OpenWorld_Traversal_A

- Build: RC_001 Win64 Development Game
- Device/Profile: DesktopTarget / Project_High
- Scenario version: 2026-06-23-a
- Repetitions: 3
- Pass/Fail: Fail

| Metric | Budget | Run 1 | Run 2 | Run 3 | Decision |
|---|---:|---:|---:|---:|---|
| P95 frame ms | 18.0 | 20.4 | 20.1 | 20.7 | fail |
| P99 frame ms | 28.0 | 33.8 | 32.4 | 34.1 | fail |
| Hitches > 50 ms | 0 | 2 | 1 | 2 | fail |
| LLM total MB | 5200 | 5060 | 5088 | 5071 | pass |
| loaded WP cells max | 24 | 23 | 23 | 23 | pass |

Escalation:
- Short Insights trace captured around frame 1832.
- HLOD transition log shows handoff from `Town_HLOD_01` to near-field actors.
- Render-thread/RHI spike increased while GPU pass time was stable.

Owner:
- World/Rendering

Next guard:
- Add HLOD transition scenario to nightly CSV gate and require HLOD builder output timestamp to match map revision.
```

## BuildGraph And RunUAT Proof

### Layering Rule

Use RunUAT/BuildCookRun directly for a simple local package proof. Use BuildGraph when the work has multiple dependent jobs, artifacts, agents, labels, reuse, symbols, crash proof and publish steps. BuildGraph should make dependencies visible; it should not hide an unreviewable shell script. [SRC-BUILD-015] [SRC-BUILD-016]

### Minimal RunUAT Proof

Capture the actual command line from the target branch. A schematic shape:

```text
RunUAT BuildCookRun
  -project=Project.uproject
  -platform=Win64
  -clientconfig=Development
  -build
  -cook
  -stage
  -pak
  -archive
  -archivedirectory=Artifacts/RC_001/Package
  -map=/Game/Maps/OpenWorld_Main
  -utf8output
```

Acceptance:

- clean workspace or clean build agent;
- no reliance on local `Binaries`, `Intermediate`, editor-loaded assets or loose content;
- logs archived with command line and environment;
- package launches outside the editor;
- smoke scenario exits with a clear pass/fail marker;
- staged/cooked manifests are retained where available. [SRC-BUILD-017]

### BuildGraph Proof Shape

```xml
<!-- Schematic only. Verify exact task names in the target branch. -->
<BuildGraph>
  <Property Name="Project" DefaultValue="Project.uproject" />
  <Property Name="Platform" DefaultValue="Win64" />
  <Property Name="Config" DefaultValue="Development" />
  <Property Name="ArchiveRoot" DefaultValue="$(RootDir)/Artifacts/$(BuildId)" />

  <Node Name="BuildGame">
    <!-- Build target binaries. -->
    <Produces>$(ArchiveRoot)/Build</Produces>
  </Node>

  <Node Name="CookWorld" Requires="BuildGame">
    <!-- Cook selected maps and Primary Asset scope. -->
    <Produces>$(ArchiveRoot)/Cook</Produces>
  </Node>

  <Node Name="BuildWorldPartitionHLOD" Requires="CookWorld">
    <!-- Some teams build before cook, some after content validation. Verify project policy. -->
    <Produces>$(ArchiveRoot)/WorldPartition/HLOD</Produces>
  </Node>

  <Node Name="Package" Requires="CookWorld;BuildWorldPartitionHLOD">
    <Produces>$(ArchiveRoot)/Package</Produces>
  </Node>

  <Node Name="SmokeRun" Requires="Package">
    <Produces>$(ArchiveRoot)/Smoke</Produces>
  </Node>

  <Node Name="PerfGate" Requires="SmokeRun">
    <Produces>$(ArchiveRoot)/Performance</Produces>
  </Node>

  <Node Name="ArchiveSymbols" Requires="BuildGame">
    <Produces>$(ArchiveRoot)/Symbols</Produces>
  </Node>

  <Node Name="ForcedCrashProof" Requires="Package;ArchiveSymbols">
    <Produces>$(ArchiveRoot)/CrashProof</Produces>
  </Node>
</BuildGraph>
```

The graph is intentionally schematic. The proof question is: if `PerfGate` fails, can another engineer find the package, command line, CSV, logs, symbols and HLOD builder output from the graph artifacts without asking who ran the job?

### Release Failure Classification

| Failure | First proof question | Recurrence guard |
|---|---|---|
| BuildGraph passed but package missing map | Did cook scope include the map and dependencies? | map/Primary Asset validation gate |
| Package launches locally but not clean machine | Which staged runtime file or prerequisite was missing? | staged-file manifest check |
| CI package differs from local package | Which graph parameter, SDK, DDC/cache, plugin or config differed? | manifest diff and clean-agent baseline |
| Perf CSV missing | Did scenario exit before CSV flush or write to wrong sandbox path? | runner end marker and artifact existence check |
| Crash report unsymbolicated | Do build ID and symbol archive match the package? | forced-crash proof node |
| HLOD stale after map change | Did builder run after relevant actor/HLOD layer changes? | stale-HLOD detection and graph dependency |

## World Partition Builder And HLOD Proof

### What The Proof Must Show

World Partition proof has two sides:

- **correctness:** cells, Runtime Data Layers, spatial/always-loaded policy, references, Level Instances, PCG output and fast-travel readiness behave in packaged runtime;
- **performance:** HLOD/streaming choices reduce the intended cost without adding unacceptable memory, build time, visual artefacts or handoff hitches.

The HLOD builder log alone is not enough. It proves a build step ran, not that the output improved the shipped experience. Pair builder output with packaged traversal and render/memory metrics. [SRC-ASSET-010] [SRC-WORLD-003]

### Builder Command Evidence

Schematic command shapes:

```text
# Conversion report before migration.
UnrealEditor-Cmd.exe Project.uproject /Game/Maps/OpenWorld_Main
  -run=WorldPartitionConvertCommandlet
  -ReportOnly
  -Unattended
  -NullRHI

# HLOD builder lane.
UnrealEditor-Cmd.exe Project.uproject /Game/Maps/OpenWorld_Main
  -run=WorldPartitionBuilderCommandlet
  -Builder=WorldPartitionHLODsBuilder
  -Unattended
  -AllowCommandletRendering
```

Validate exact executable, map path syntax, `-Builder` value and rendering requirements in the target branch. Some HLOD generation paths may need rendering-capable agents rather than `NullRHI`; do not assume one command works for every project.

### HLOD Acceptance Matrix

| Check | Passing evidence | Failing evidence |
|---|---|---|
| Builder executed | log includes map, builder, target revision and success marker | command silently skipped target map |
| Output freshness | proxy output timestamp/revision matches source changes | stale proxy still shipped after actor/material change |
| Visual quality | far/transition/near screenshots accepted by owner | material merge artefacts, missing geometry, bad lighting |
| Runtime handoff | no unacceptable P95/P99 transition hitch | HLOD hides near-field actor activation spike |
| Cost target | intended primitive/draw/material/shadow cost falls | memory/proxy cost rises without frame benefit |
| Gameplay boundary | proxy not treated as authoritative gameplay/collision unless designed | interactable/collision missing after transition |
| Cook/package | HLOD artifacts present in packaged build | editor-only or uncooked proxies |

### World Partition Traversal Proof

The traversal proof should log:

```text
WP_TRAVERSAL_START Scenario=OpenWorld_Traversal_A Build=RC_001
WP_CELL_STATE Time=12.3 Loaded=18 Activated=14 Pending=3
WP_DATA_LAYER State Town_Normal=Active Town_Burned=Unloaded
WP_HLOD_STATE Region=Town Proxy=Town_HLOD_01 Visible=true
WP_FAST_TRAVEL_REQUEST Destination=Outpost_Entry
WP_READINESS Destination=Outpost_Entry Streaming=true Collision=true GameplayActors=true
WP_FAST_TRAVEL_COMMIT Destination=Outpost_Entry TimeToReadyMs=842
WP_TRAVERSAL_END Scenario=OpenWorld_Traversal_A Result=Pass
```

Exact log fields are project-defined. The important thing is that streaming readiness, Data Layer state, HLOD visibility/transition and gameplay readiness are visible in packaged logs. A screenshot of the editor World Partition window is not packaged proof.

## Full Pipeline Acceptance Criteria

Treat the whole proof as passing only when all rows are satisfied or explicitly rejected with evidence:

| Gate | Acceptance |
|---|---|
| Target audit | branch commands/APIs/tooling verified or rejection documented |
| Build | clean build succeeds for target/config/platform |
| Cook | intended maps/assets cook; missing-rule injection fails correctly |
| Package | staged/package/archive output produced with manifest |
| Launch | package launches outside editor on target or constrained profile |
| Smoke | deterministic smoke scenario reaches pass marker |
| World builder | WP/HLOD builder command and output captured |
| Traversal | packaged traversal and fast-travel readiness scenario passes |
| Performance | CSV gate has repeated runs, thresholds and profile/CVar proof |
| Escalation | failing signal has smallest causal trace/capture |
| Memory | LLM or memory evidence covers key checkpoints where relevant |
| Crash | forced packaged crash symbolicates against archived symbols |
| Failure injection | at least one missing cook, stale HLOD, wrong profile or CSV-missing failure is caught |
| Report | artifact links, owners, decisions and residual risks are recorded |

## Debugging Workflows

### Packaged Gate Fails But Editor Looks Fine

1. Confirm the failing package identity and active profile/CVars.
2. Compare package and editor content paths: cooked assets, staged files, shader/PSO state, map list and plugins.
3. Use CSV to locate scenario and window.
4. Capture a short trace around the failing window.
5. Classify CPU, Draw/RHI, GPU, memory, I/O, shader/PSO, streaming, HLOD transition or platform wait.
6. Make one causal change.
7. Rerun the same packaged scenario three times.
8. Add a recurrence guard to the gate.

### BuildGraph Green But Artifact Is Wrong

1. Check manifest parameters and graph output paths.
2. Confirm the node that produced the artifact, not just the final green status.
3. Compare local and CI graph properties.
4. Inspect cook/stage/archive manifests.
5. Verify downstream nodes depend on the artifact-producing node.
6. Add an artifact existence and identity check.

### HLOD Builder Passes But Runtime Hitches

1. Confirm the builder output is in the packaged artifact.
2. Measure far, transition and near-field ranges separately.
3. Check whether the hitch is proxy load, near actor activation, material/shader first use, texture streaming, nav/collision setup or gameplay BeginPlay.
4. Compare primitive/draw/material-section counts before and after HLOD.
5. If HLOD only hides a near-field activation spike, fix streaming/readiness/activation rather than only changing proxy quality.

### CSV Gate Is Noisy

1. Remove accidental variability: dynamic camera, random seed, network, bots, thermal state, background load, resolution scaling and VSync.
2. Add warm-up and readiness markers.
3. Use repeated runs and compare distributions.
4. Promote only stable, actionable metrics to blockers.
5. Keep volatile diagnostics as reporting-only signals.

## Interview Answer Patterns

### "How would you prove a packaged performance gate is production-ready?"

> I would start with a manifest: build ID, branch, target, config, platform, device profile, scenario and thresholds. Then I would run deterministic packaged scenarios with warm-up and fixed sample windows, capture CSV for repeatable metrics, and keep short causal traces for failures. The gate must prove active CVars/profile, package identity and artifact paths. I would fail on stable actionable metrics such as repeated P95/P99 frame time, hitch count, first-interactive time or memory growth, not one noisy average FPS sample.

### "How do BuildGraph and RunUAT fit a release pipeline?"

> RunUAT/AutomationTool is the command-line orchestration surface for build, cook, stage, package, deploy and run. BuildGraph is useful when those operations become a dependency graph with parameters, agents, artifacts, symbols, crash proof and publish steps. I would use direct RunUAT for a local or simple package proof, and BuildGraph when the team needs repeatable build-farm evidence and artifact flow. The graph should make dependencies explicit, especially package, symbols, smoke, performance and crash-proof nodes.

### "What is a strong World Partition/HLOD proof?"

> I would not stop at "World Partition is enabled". I would prove spatial versus always-loaded policy, Runtime Data Layer state, HLOD builder output, packaged cook inclusion, and traversal readiness. For HLOD I would measure the intended cost reduction, visual quality, memory/proxy cost and transition hitches. A packaged traversal and fast-travel scenario should log cell/layer/readiness state and capture CSV or trace evidence.

### "How do you handle stale or bad HLOD output?"

> First I verify that the builder ran for the correct map and that output is in the package. Then I compare source actor/material changes to proxy freshness, inspect far/transition/near visuals, and measure whether the problem is proxy generation, cook inclusion, shader/material cost, memory, or near-field activation. The recurrence guard is a builder dependency plus stale-output check, not asking artists to remember a manual rebuild.

## Common Weak Answers

- "I tested in PIE and it was fine."
- "The CI job was green, so the package is valid."
- "Average FPS is above 60."
- "HLOD is enabled, so distant objects are solved."
- "Delete Intermediate and DDC."
- "Use BuildGraph for everything."
- "Cook all content to fix missing packaged assets."
- "The Device Profile exists in source, so it must be active."
- "The crash report came in, so symbols are fine."

## Hands-On Verification Task

Build a compact proof packet on any UE5 project with one representative map:

1. Define one packaged performance scenario and one World Partition/HLOD traversal scenario.
2. Capture target branch command help or local wrapper docs for RunUAT/BuildGraph and builder commandlets.
3. Package from a clean workspace or clean agent.
4. Archive build logs, command lines, package path, symbols and manifest.
5. Run a packaged smoke scenario and prove active Device Profile/CVars.
6. Run CSV for three repetitions of the performance scenario.
7. Capture one short causal trace for the worst frame or an injected regression.
8. Run World Partition/HLOD builder or document why the map does not support it.
9. Run packaged traversal and log cell/layer/readiness/HLOD state.
10. Inject two failures: missing cook/stage asset and stale/missing HLOD or wrong profile.
11. Prove the gate catches each failure and records an owner.
12. Write a report with artifact links, pass/fail table, residual risks and next gate.

## Source Map

- `SRC-PERF-003`, `SRC-PERF-009` and `SRC-PERF-010` support Unreal Insights, CSV and memory/LLM-oriented performance evidence.
- `SRC-BUILD-015`, `SRC-BUILD-016` and `SRC-BUILD-017` support BuildGraph, AutomationTool and build/cook/package/deploy/run separation.
- `SRC-BUILD-018` supports crash-reporting and symbolication as a release-quality gate.
- `SRC-ASSET-010` and `SRC-WORLD-003` support World Partition, builder commandlet and HLOD workflow concepts.
- `SRC-PERF-007`, `SRC-PERF-008` and `SRC-PLAT-005` connect target proof to device profiles, scalability and platform constraints.
