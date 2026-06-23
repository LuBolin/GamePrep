# World Partition and Large-World Production Pipeline

See also: [[ue_packaged_performance_build_worldpartition_proof]], [[ue_assets_loading_cooking]], [[ue_pcg_procedural_content]], [[ue_world_partition_debugging_workbook]], [[ue_hands_on_projects]].

Version target: **UE5.3-UE5.6**. World Partition, runtime hash behavior, builder commandlets, HLOD settings, Data Layer APIs, Level Instance modes and cook output are version-sensitive. Treat this chapter as a production mental model and validation checklist; compile exact C++ calls and command lines against the target engine branch. [SRC-ASSET-010] [SRC-WORLD-001] [SRC-WORLD-002] [SRC-WORLD-003] [SRC-WORLD-004]

## Scope

This chapter deepens the first-pass asset/loading chapter. It covers the production pipeline around large UE5 worlds:

- World Partition runtime grids, cells, streaming sources and spatial loading;
- Runtime and Editor Data Layers;
- One File Per Actor source-control workflow;
- Level Instances and Packed Level Blueprints;
- World Partition HLOD;
- PCG and generated-output placement awareness;
- cook/build validation and target-device evidence;
- debugging and interview answer patterns.

It does **not** replace the broader asset chapter's hard/soft reference and cook fundamentals, the rendering chapter's full GPU cost model, or platform/device-lab chapters. Instead, it connects them into a defensible large-world workflow. [SRC-ASSET-010] [SRC-ASSET-005] [SRC-ASSET-012] [SRC-RENDER-011]

## Why This Exists

A large-world pipeline is not just a bigger map. It is an agreement among authoring, source control, streaming, representation, gameplay state, cook output and profiling:

```text
persistent world
  -> external actor files / OFPA
  -> runtime grid cells
  -> streaming sources and loading ranges
  -> Data Layer states
  -> Level Instances / Packed Level Blueprints / PCG output
  -> HLOD proxies
  -> cook/package artifacts
  -> target-device traversal evidence
```

World Partition automatically divides a persistent level into grid cells and streams cells around streaming sources, but the project still decides what is spatial, what is always loaded, what is only an editor layer, what is runtime-gated, what is represented by HLOD, and what must survive unload/reload as durable game state. [SRC-ASSET-010] [SRC-WORLD-001]

For interviews, the strongest answer is not "use World Partition". It is: "define the world ownership contract, validate streaming and cook behavior, then prove target performance and correctness with representative travel paths."

## Prerequisites And Dependents

**Prerequisites:**

- hard versus soft references, cook inclusion and packaged-only failures;
- level streaming concepts;
- Actor lifetime and cross-object reference safety;
- rendering cost dimensions and HLOD basics;
- profiling with stats, Unreal Insights and CSV;
- build/cook/package automation. [SRC-ASSET-001] [SRC-ASSET-009] [SRC-ASSET-010] [SRC-PERF-003] [SRC-BUILD-017]

**Dependent topics:**

- open-world gameplay architecture;
- procedural content and PCG;
- multiplayer relevancy and persistence;
- AI navigation and Smart Object placement;
- platform memory/I/O budgets;
- release automation and device-lab gates.

## The Mental Model

World Partition is a spatial streaming system. Data Layers are an authoring/runtime state organisation system. OFPA is a collaboration/storage system. Level Instances and Packed Level Blueprints are reusable content authoring systems. HLOD is a distant representation system. Cook/package/build automation is the release proof system.

Those systems overlap but are not interchangeable:

| Problem | Primary system | What it does not solve alone |
|---|---|---|
| "Avoid hand-authored streaming sublevels for a huge world" | World Partition | gameplay authority, persistence, target performance, bad references |
| "Let artists edit nearby actors without constant level-file conflicts" | OFPA | source-control validation or runtime streaming policy |
| "Show a destroyed village versus restored village" | Runtime Data Layers | save authority, quest truth, replicated gameplay state |
| "Reuse a camp, building cluster or POI in many places" | Level Instance / Packed Level Blueprint | high-density runtime streaming cost, mutable gameplay identity |
| "Reduce distant object cost" | HLOD | near-field content cost, material/pixel cost, gameplay collision correctness |
| "Scatter biome content" | PCG | deterministic ownership, cleanup, streaming/HLOD/Data Layer correctness |
| "Prove it works outside the editor" | Cook/package/device lab | authoring convenience or design intent |

## What A 3-Year Engineer Should Know

A strong generalist should be able to:

1. Explain that World Partition streams grid cells around streaming sources, while Level Streaming explicitly loads/unloads level packages. [SRC-ASSET-009] [SRC-ASSET-010]
2. Decide which Actors are spatially loaded versus always loaded, and understand that references between Actors can affect loading bundles. [SRC-ASSET-010]
3. Use Runtime Data Layers for phase/event/world-state presentation while keeping durable gameplay truth in a save/quest/server system. [SRC-WORLD-001]
4. Explain why OFPA helps source-control collaboration and why encoded external actor files require changelist validation. [SRC-WORLD-002]
5. Choose Level Instances or Packed Level Blueprints for reusable static POIs, and call out runtime-mode caveats. [SRC-WORLD-004]
6. Treat HLOD as a measured proxy-representation trade-off, not a visual checkbox. [SRC-WORLD-003] [SRC-RENDER-011]
7. Debug "works in editor, fails packaged" through cook inclusion, Data Layer/runtime state, external actor/changelist correctness, HLOD generation and packaged traversal logs.
8. Build a small validation gate: conversion report, Data Layer policy check, HLOD builder output, cook/package proof and target-device traversal.

## Specialist Depth

Specialist depth starts when you can defend project-specific choices:

- runtime hash/grid configuration and cell-size/loading-range trade-offs;
- custom builder commandlet and data-validation integration;
- source-control automation around OFPA external actors;
- Data Layer authority, replication and save migration rules;
- HLOD layer hierarchy, proxy settings, material merge risks, Nanite/proxy policy and memory/build-time budgets;
- streaming source orchestration for fast travel, vehicles, cinematics and multiplayer spectators;
- PCG partition/runtime generation plus Data Layer/HLOD assignment;
- platform I/O and memory constraints for large traversal paths;
- world-partition conversion and migration from legacy/sublevel workflows.

## World Partition Design Contract

World Partition uses one persistent level and divides it into cells that are loaded/unloaded according to streaming sources. UE docs describe integration with OFPA, Data Layers, Level Instancing and HLOD. [SRC-ASSET-010]

The practical contract is:

1. **World shape:** define playable area, traversal speeds, fast-travel points, verticality, interiors and streaming corridors.
2. **Grid policy:** define cell size, loading range and whether one runtime grid is enough. The official docs warn that multiple runtime grids can negatively affect performance; do not add grids casually. [SRC-ASSET-010]
3. **Actor policy:** classify Actors as spatially loaded, always loaded, runtime-layered, editor-only, generated, proxy-only or manager/service.
4. **Reference policy:** prevent durable managers, quests, UI or save systems from hard-referencing unloadable spatial Actors. Cross-Actor references can make Actors load together. [SRC-ASSET-010]
5. **Streaming source policy:** define player, vehicle, camera, fast-travel, preview and scripted sources. Streaming source priority and target state should match gameplay expectation. [SRC-ASSET-010]
6. **Readiness policy:** do not teleport or start a scripted sequence merely because a destination transform is known. Wait until required cells/layers/collision/gameplay representatives are ready.
7. **Release proof:** run packaged traversal and fast-travel scenarios with logs, memory, loading and hitch evidence.

### Spatially Loaded Versus Always Loaded

`Is Spatially Loaded` should be treated as an ownership decision, not a convenience toggle.

Good spatial candidates:

- ordinary set dressing;
- static world geometry;
- local interactables whose durable state is stored elsewhere;
- region-local AI spawners with deterministic reconstruction;
- visual-only POI content.

Good always-loaded candidates:

- authority managers;
- world-state coordinators;
- global quest/save systems;
- subsystem-like service Actors when a real subsystem is not appropriate;
- bootstrap/entry points that cannot be tied to a cell.

Common failure: a Level Blueprint, manager Actor or data asset accidentally holds a hard reference to a spatial Actor, causing unintended always-loaded behavior or larger streaming bundles. Epic's World Partition page specifically cautions against Level Blueprint references that can mark Actors Always Loaded and suggests Blueprint Classes for many reusable behaviors. [SRC-ASSET-010]

### Streaming Source Readiness

World Partition streaming sources decide which cells are loaded. In gameplay, streaming readiness has at least four layers:

1. the cell package is loaded;
2. relevant Actors/components are registered;
3. collision/nav/gameplay systems are ready enough for the action;
4. Data Layer state and any required HLOD/visual transition policy are correct.

The official World Partition page exposes an `Is Streaming Completed` readiness concept for a streaming source. In production, wrap that concept in a gameplay-specific readiness check rather than assuming cell load alone is enough. [SRC-ASSET-010]

```cpp
// Schematic only. Verify exact class names, headers and method signatures in the target branch.
// Intent: fast-travel waits for a destination streaming source before moving the pawn.

UCLASS()
class AWorldTravelPreloader : public AActor
{
    GENERATED_BODY()

public:
    void RequestFastTravel(AController* Controller, const FVector& Destination)
    {
        PendingController = Controller;
        PendingDestination = Destination;

        SetActorLocation(Destination);

        // Target-branch API check required. The official docs expose the streaming-source
        // component and an Is Streaming Completed concept, but exact C++ names can change.
        StreamingSource->EnableStreamingSource();

        GetWorldTimerManager().SetTimer(
            PollHandle,
            this,
            &AWorldTravelPreloader::PollStreamingReady,
            0.10f,
            true);
    }

private:
    UPROPERTY(VisibleAnywhere)
    TObjectPtr<UWorldPartitionStreamingSourceComponent> StreamingSource;

    TWeakObjectPtr<AController> PendingController;
    FVector PendingDestination = FVector::ZeroVector;
    FTimerHandle PollHandle;

    void PollStreamingReady()
    {
        if (!StreamingSource->IsStreamingCompleted())
        {
            return;
        }

        if (!IsDestinationGameplayReady(PendingDestination))
        {
            return;
        }

        GetWorldTimerManager().ClearTimer(PollHandle);
        StreamingSource->DisableStreamingSource();

        if (AController* Controller = PendingController.Get())
        {
            TeleportControlledPawn(Controller, PendingDestination);
        }
    }
};
```

Interview point: this code is not about memorising API names. It shows the correct shape: enable source, poll readiness, add project-specific collision/gameplay checks, then move the player.

## Data Layers

Data Layers organise Actors and can be Editor or Runtime Data Layers. UE's Data Layer docs distinguish Data Layer Assets from world-specific Data Layer Instances and describe Runtime Data Layers as loadable/unloadable through gameplay events such as quests, progression and world events. [SRC-WORLD-001]

### Asset Versus Instance

Use the distinction:

- **Data Layer Asset:** reusable identity and type-like definition.
- **Data Layer Instance:** world-specific placement/state/configuration that references the asset.

This matters in interviews because "the swamp event layer" might be a reusable asset, while "swamp event layer in Map_A with initial loaded state X" is the instance. [SRC-WORLD-001]

### Editor Versus Runtime Layers

Editor Data Layers are organisation and authoring visibility tools. Runtime Data Layers affect runtime load/activation behavior. Do not design gameplay on the assumption that an editor-visible layer is runtime-active. [SRC-WORLD-001]

### Runtime State Pattern

A robust project usually models Data Layer changes as a state transition with gameplay gates:

```text
Unloaded
  -> LoadingRequested
  -> LoadedHidden
  -> ActivatedVisible
  -> DeactivationRequested
  -> Unloaded
```

The durable truth should usually live outside the layer:

```text
Quest/save/server state: "VillagePhase = Restored"
      |
      v
World-state presenter: chooses active Runtime Data Layers
      |
      v
World Partition/Data Layer system: streams and shows the correct actors
      |
      v
Gameplay systems: bind to stable IDs, not transient actor pointers
```

If the player destroys a bridge, the save system should not depend on "the bridge Actor happened to be loaded when the save happened." Persist a stable world-state key or object ID, then let layer/actor loading reconstruct presentation.

### Schematic Data Layer Coordinator

```cpp
// Schematic only. Use the target branch's Data Layer subsystem/manager API.

USTRUCT()
struct FWorldPhaseLayerPolicy
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly)
    FName PhaseName;

    UPROPERTY(EditDefaultsOnly)
    TArray<TSoftObjectPtr<UDataLayerAsset>> LayersToActivate;

    UPROPERTY(EditDefaultsOnly)
    TArray<TSoftObjectPtr<UDataLayerAsset>> LayersToUnload;
};

UCLASS()
class UWorldPhasePresenter : public UWorldSubsystem
{
    GENERATED_BODY()

public:
    void ApplyPhase(FName PhaseName)
    {
        const FWorldPhaseLayerPolicy* Policy = FindPolicy(PhaseName);
        if (!Policy)
        {
            UE_LOG(LogTemp, Warning, TEXT("Unknown world phase %s"), *PhaseName.ToString());
            return;
        }

        // Important: the save/quest/server system owns PhaseName.
        // This subsystem only presents that state through Runtime Data Layers.
        RequestLayerChanges(*Policy);
        StartReadinessProbe(*Policy);
    }
};
```

Interview point: the layer coordinator should be replayable and idempotent. If load is cancelled, travel happens, or the player reloads a save, reapplying the same phase should converge on the same runtime layer set.

## One File Per Actor

One File Per Actor stores each Actor instance in an external file during editor work, reducing source-control overlap. UE docs state that OFPA is editor-only and Actors are embedded in the level when cooked. World Partition enables OFPA by default. [SRC-WORLD-002]

### Why OFPA Helps

Without OFPA, many nearby edits can dirty the same level file. With OFPA, two artists can often edit different Actors in the same world without fighting over one monolithic map asset.

### Why OFPA Still Needs Process

OFPA external actor filenames are encoded, so a normal file list can be hard to review manually. Epic docs point to the View Changelist window and warn that partial submits can create dangling references. [SRC-WORLD-002]

Use a changelist rule:

1. Validate changed Actors through the editor/source-control view, not only raw filenames.
2. Include moved/renamed/deleted referenced Actors and folders in the same submit.
3. Run redirector/reference validation before merge.
4. Run a report-only conversion/validation commandlet for large migrations.
5. Prohibit "only submit the file I remember touching" for OFPA-heavy maps.

### OFPA Failure Pattern

Symptom:

- works on one artist's machine;
- teammate gets missing references or wrong POI state;
- CI cook fails or silently excludes an expected Actor.

Likely causes:

- external actor file omitted;
- stale redirector or rename not fixed up;
- Level Instance or Data Layer membership changed without companion files;
- binary/source-control conflict resolved by choosing one side;
- editor had the missing Actor loaded from local cache.

Debug order:

1. Identify the missing Actor by label, path and external actor package.
2. Compare source-control changelist view with raw file changes.
3. Validate references and redirectors.
4. Reopen clean workspace, run commandlet/cook, and reproduce without local unsaved files.

## Level Instances And Packed Level Blueprints

Level Instancing is a level-based workflow for reusing Actor arrangements such as buildings, rooms or POIs. Epic docs position Packed Level Blueprints as an optimised option for static dense visual arrangements. [SRC-WORLD-004]

Use this selection rule:

| Need | Better default | Reason |
|---|---|---|
| Reusable authored POI with multiple normal Actors | Level Instance | edit in context, repeat arrangement |
| Static visual kitbash with many static meshes | Packed Level Blueprint | optimised representation for dense static content |
| Highly dynamic gameplay entity with unique state | Actor/Blueprint/Data Asset pattern | stable identity and authority matter more than reuse container |
| High-density runtime streaming instances | Avoid Level Streaming mode by default | official docs note runtime cost and discourage high density in Level Streaming mode |

### Embedded Versus Level Streaming Mode

Epic's Level Instancing page describes embedded behavior under World Partition and warns that non-OFPA actors can be lost in embedded mode; Level Streaming mode has runtime cost and high density is not recommended. [SRC-WORLD-004]

Interview answer:

> I would not pick Level Instance mode only from authoring convenience. I would check whether the POI needs runtime streaming as a separate level, whether all contained actors support OFPA, whether the instance is mostly static, how many instances exist, and whether gameplay state needs stable identity outside the instance.

### Level Instance Debugging

Common bugs:

- content looks correct in authoring mode but missing after embedding;
- non-OFPA Actor content is lost or not carried as expected;
- many Level Streaming-mode instances create runtime overhead;
- gameplay systems address "the Actor inside this instance" by unstable path;
- baked lighting/HLOD/collision does not match the reusable arrangement after changes.

Debug workflow:

1. Record instance mode and target branch.
2. Validate contained Actors and OFPA support.
3. Test duplicate instances with unique gameplay IDs.
4. Cook/package and verify the instance appears in packaged traversal.
5. Run HLOD/Data Layer checks for the instance content.

## HLOD In World Partition

HLOD replaces groups of distant Actors with proxy representations to reduce distant object cost. World Partition has HLOD Layer assets and builder workflows; HLOD also connects to the broader rendering discussion around draw calls, material cost, memory and visual transitions. [SRC-WORLD-003] [SRC-RENDER-011]

### Choosing HLOD Layers

HLOD Layer choices are not only "on/off":

- **Instancing-like layers:** useful when many compatible repeated meshes can share representation.
- **Merged layers:** combine meshes/materials, reducing object/draw overhead but potentially losing per-object culling/material nuance.
- **Simplified layers:** generate simplified proxy geometry, trading build time, memory and visual fidelity.

The World Partition HLOD docs describe settings for mesh merge, material merge, physics, landscape culling, Nanite generated meshes and proxy quality. They also warn about material merge artefacts and sampling/memory trade-offs. [SRC-WORLD-003]

### HLOD Decision Matrix

| Question | Why it matters |
|---|---|
| What cost is HLOD meant to reduce? | Draw/RHI, primitive count, material cost, shadow cost, memory or streaming? |
| What distance range uses the proxy? | Too near causes visible artefacts; too far may not repay build/memory cost. |
| Is collision needed? | Visual proxy collision can be wrong for gameplay; physics merge settings need proof. |
| Are materials world-position-dependent? | Equivalent/material merging can create artefacts. [SRC-WORLD-003] |
| Is Nanite involved? | Nanite changes geometry cost but not all material/pixel/shadow/memory costs. |
| Does landscape culling apply? | Hidden faces under terrain can be removed where appropriate. [SRC-WORLD-003] |
| How is it built in CI? | Stale HLOD proxies are a release risk, not just an editor annoyance. |

### HLOD Debugging

Symptom: distant town is cheap, but player arrives and hitches badly.

Likely causes:

- near-field cell activation still spawns too many Actors/components;
- HLOD proxy hides the cost until handoff;
- source meshes/materials are expensive when loaded;
- collision/nav/AI activation occurs at the same moment;
- streaming, shader/PSO and texture load all converge.

Debug order:

1. Capture far, transition and near positions.
2. Record primitive/draw counts, Game/Draw/RHI/GPU, memory and loading traces.
3. Toggle HLOD visibility/debug view in the target branch.
4. Compare proxy memory/build output against near-field content cost.
5. Fix the actual limiting resource; HLOD is not a cure for every handoff hitch.

## PCG And Generated Large-World Content

PCG output in a World Partition world needs ownership, cleanup, streaming and representation rules. The PCG overview notes World Partition support, including generated Actors assigned to the same Data Layer and HLOD Layer as the PCG component in relevant workflows. [SRC-PCG-001]

Large-world PCG checklist:

1. Decide editor-generated, partitioned, runtime or hybrid generation.
2. Give every generated output stable generation metadata: graph version, seed, input identity and owner.
3. Separate graph execution cost from output cost: Actors/components/instances/collision/nav/materials.
4. Verify Data Layer/HLOD assignment for generated output.
5. Test cleanup/regeneration across cell unload/reload.
6. Test packaged cook output; do not infer from editor preview.
7. Record target traversal evidence at 1x/10x/100x scale.

## Build, Cook And Automation Gates

World-building mistakes often escape editor testing because the editor has loaded assets, local unsaved changes, flexible file discovery and different runtime paths. A strong pipeline runs commandlets/build steps and packages representative maps. [SRC-ASSET-005] [SRC-BUILD-016] [SRC-BUILD-017]

### Commandlet And Builder Evidence

Example command shapes, not copy-paste guarantees:

```text
# Report conversion impact before migrating a legacy/sublevel map.
UnrealEditor.exe Project.uproject Map.umap -run=WorldPartitionConvertCommandlet -ReportOnly

# Generate a conversion ini for review before committing a migration.
UnrealEditor.exe Project.uproject Map.umap -run=WorldPartitionConvertCommandlet -GenerateIni

# Build World Partition HLODs in an automation lane.
UnrealEditor.exe Project.uproject Map.umap -run=WorldPartitionBuilderCommandlet -Builder=WorldPartitionHLODsBuilder

# Cook/package representative maps for proof.
RunUAT BuildCookRun -project=Project.uproject -platform=TargetPlatform -clientconfig=Development -cook -stage -pak -build
```

The official World Partition page documents conversion commandlet options, builder commandlets for minimap/HLOD and cooking World Partition worlds through the cook commandlet flow. The exact executable, map path, builder names, flags and platform arguments should be checked against the installed engine. [SRC-ASSET-010] [SRC-WORLD-003] [SRC-BUILD-016] [SRC-BUILD-017]

### Large-World CI Gate

A realistic gate starts small:

1. **Source-control validation:** changed external actor files are represented by meaningful Actor/changelist data; no partial OFPA submit.
2. **Reference validation:** no prohibited always-loaded references from managers/Level Blueprints to spatial Actors.
3. **Data Layer validation:** every Runtime Data Layer has owner, initial state policy, save/quest/server relationship and test scenario.
4. **Level Instance validation:** mode, OFPA support and packaged behavior recorded.
5. **PCG validation:** generated-output owner, cleanup, Data Layer/HLOD assignment and runtime-generation policy recorded.
6. **HLOD build:** HLOD builder runs in automation and reports changed/stale proxies.
7. **Cook/package:** representative map cooks in a clean workspace.
8. **Traversal smoke:** packaged build traverses or teleports through key cells and records readiness/logs.
9. **Telemetry:** CSV/Insights/LLM/platform profiler evidence exists for one representative pass.
10. **Ownership:** every failure category has an owner: world art, tech art, gameplay, build, rendering, performance or platform.

## Debugging Workflows

### Teleport Into Missing World

Symptom:

- player teleports;
- terrain or collision absent;
- character falls, clips or sees proxy-only content;
- issue is rare on fast machines but common on target hardware.

Workflow:

1. Verify the teleport path uses a streaming source or equivalent preload, not immediate movement.
2. Log source location, priority, target state, requested cells and readiness.
3. Wait for World Partition readiness and project-specific collision/nav/gameplay readiness.
4. Confirm Runtime Data Layer state at destination.
5. Capture loading and hitch trace around teleport.
6. Add an automation scenario for the exact path.

Weak fix: add a blind delay.

Strong fix: define readiness conditions, preload range/source policy and failure fallback.

### Data Layer Works In Editor, Not Packaged

Workflow:

1. Confirm the layer is Runtime, not Editor-only.
2. Check the Data Layer Asset/Instance distinction.
3. Verify initial runtime state and any world-state code that toggles it.
4. Verify referenced assets are cooked.
5. Run packaged scenario with logging for phase, layer state and map name.
6. Check whether local editor state or unsaved files masked the failure.

### Too Much Always Loaded

Workflow:

1. List always-loaded Actors/classes and references.
2. Look for Level Blueprint or manager hard references to spatial Actors.
3. Find cross-cell references that bundle content.
4. Move durable state to managers/save systems and use stable IDs/soft refs/events.
5. Reprofile memory/loading and verify expected cells unload.

### HLOD Looks Good But Gameplay Is Broken

Workflow:

1. Distinguish visual proxy from gameplay Actor/collision/nav presence.
2. Test far, transition and near range.
3. Verify HLOD Layer settings for collision/physics/material merge.
4. Confirm near-field cells load before interaction is allowed.
5. Record debug view and profiler counters.

### OFPA Changelist Broke The Map

Workflow:

1. Use editor/source-control changelist view to map encoded files to Actors.
2. Find omitted external actor files.
3. Validate redirectors/references.
4. Reopen clean workspace and reproduce.
5. Add pre-submit validation or review checklist.

### Conversion Or Migration Is Unstable

Workflow:

1. Run report-only conversion first.
2. Generate/review conversion ini.
3. Fix unstable GUID/reference issues rather than skipping validation as a default.
4. Convert a copy/suffix first.
5. Cook/package and run traversal before deleting old structure.

The World Partition docs list conversion options such as `-ReportOnly`, `-GenerateIni`, `-ConversionSuffix`, `-OnlyMergeSubLevels` and `-SkipStableGUIDValidation`. Treat skip options as emergency/diagnostic tools, not normal migration policy. [SRC-ASSET-010]

## Profiling Workflow

Profile World Partition as a pipeline:

1. **Scenario:** fixed route, fixed camera speed, fast-travel cases, vehicle speed if relevant, same build/device.
2. **Streaming:** cell load/unload timeline, source state, pending packages, I/O and hitches.
3. **Memory:** resident world memory, textures, meshes, HLOD proxies, external systems, LLM tags where available.
4. **CPU:** Actor/component registration, construction scripts, BeginPlay-like spikes, nav/collision setup, AI spawn.
5. **Render:** primitives, draws, material passes, shadows, HLOD transition, Nanite/LOD/ISM/HISM policy.
6. **Cook/package:** map inclusion, chunk/install bundle rules, HLOD/minimap/generated artefacts.
7. **Device:** target hardware, active Device Profile/CVars, thermals and repeated-run variance. [SRC-PERF-003] [SRC-PERF-009] [SRC-PLAT-005]

Useful metrics:

- loaded cell count;
- always-loaded Actor count;
- total Actor/Component count per region;
- cell activation time;
- fast-travel time to ready;
- package/IO bytes during traversal;
- peak memory and memory delta per region;
- primitive/draw/material-section count;
- HLOD proxy memory and visual artefact notes;
- hitch P95/P99 and worst frame with causal trace.

## Common Bugs

- Treating World Partition as automatic world design.
- Putting gameplay truth inside unloadable Actors without durable IDs.
- Using Runtime Data Layers as save authority instead of presentation/streaming state.
- Accidentally keeping spatial Actors always loaded through hard references or Level Blueprint references.
- Assuming editor-visible Data Layers are runtime-active.
- Submitting partial OFPA external actor changes.
- Using high-density Level Streaming-mode Level Instances without profiling.
- Forgetting HLOD builder/cook steps in CI.
- Letting HLOD proxy hide a near-field activation hitch.
- Testing only in editor with already-loaded content.
- Assuming PCG output has correct Data Layer/HLOD/cook assignment.
- Adding multiple runtime grids without measuring the performance impact.

## Common Misconceptions

**"World Partition replaces level streaming."**  
World Partition is the UE5 large-world default for many projects, but explicit level streaming can still be valid for bounded interiors, menus, arenas or special flows. [SRC-ASSET-009] [SRC-ASSET-010]

**"If it is in a Runtime Data Layer, gameplay state is solved."**  
Runtime Data Layers can load/unload or activate world content. They do not by themselves define authority, save migration, replication or stable identity. [SRC-WORLD-001]

**"OFPA means source control conflicts disappear."**  
OFPA reduces overlap, but encoded filenames and partial submits need validation. [SRC-WORLD-002]

**"HLOD is always a performance win."**  
HLOD can reduce distant object cost, but it adds build output, memory, proxy transitions and artefact risk. It does not fix near-field gameplay activation cost. [SRC-WORLD-003] [SRC-RENDER-011]

**"Packed Level Blueprints are for every reusable thing."**  
They are strongest for static dense visual arrangements. Mutable gameplay entities still need stable runtime identity and authority. [SRC-WORLD-004]

## System Design Implications

Large-world architecture should separate:

- **definition:** Data Assets, Data Tables, PCG graph parameters, biome/world-state definitions;
- **durable state:** save/server/quest systems with stable IDs and schema versions;
- **presentation:** spatial Actors, Data Layer activation, Level Instances, HLOD, PCG outputs;
- **runtime services:** streaming source coordinators, region managers, AI/nav spawn managers;
- **validation:** commandlets, data validators, package gates and device scenarios.

For multiplayer, be especially careful:

- The server owns durable state and authoritative gameplay results.
- Clients may have different visible/loaded cells, but cannot own truth because they streamed something first.
- Replication relevancy and World Partition streaming are related performance concepts but not the same system.
- Late joiners need reconstruction from durable world state, not from "which Data Layer happened to be visible on one client."

For saves:

- store stable world-state keys, destroyed/collected IDs and schema versions;
- avoid serialising transient Actor pointers from spatial cells;
- when content changes, migrate IDs and handle missing definitions gracefully.

## Interview Answer Patterns

### "How would you design an open world in UE5?"

> I would start from traversal speed, memory and platform budgets, then define World Partition grid/source policy, spatial versus always-loaded Actors, Data Layer world-state policy, reusable POI rules through Level Instances or Packed Level Blueprints, HLOD layers, and cook/build validation. Gameplay truth lives in save/server/quest systems, not only in unloadable Actors. I would run packaged traversal and fast-travel scenarios with loading, memory, hitch and visual-transition evidence before calling the design complete.

### "World Partition versus Level Streaming?"

> Level Streaming gives explicit level package load/unload control. World Partition divides a persistent UE5 world into grid cells driven by streaming sources and integrates with OFPA, Data Layers and HLOD. I choose based on world shape, collaboration, authoring and runtime control. World Partition is strong for large continuous worlds; explicit streaming can still be better for bounded or highly scripted spaces.

### "How do Data Layers fit gameplay progression?"

> Runtime Data Layers can present world phases, events and progression by loading or activating sets of Actors. I keep the durable phase in a quest/save/server state model, then use a presenter to drive Runtime Data Layer state. That prevents unloaded Actors from becoming the only source of truth and makes save/load, late join and content migration testable.

### "How do you stop fast travel from landing in an unloaded area?"

> I create or move a streaming source to the destination, request the right target state/range, wait for streaming completion and project-specific readiness like collision/nav/key gameplay Actors, then teleport. If readiness times out, I show a loading state or fallback. I test the path in a packaged build on target hardware, not only in editor PIE.

### "What do you check before adopting HLOD?"

> I first name the cost: object count, draw/RHI, shadow, material, streaming or memory. Then I choose HLOD layers and proxy settings, build them in automation, compare far/transition/near ranges and measure memory plus visual artefacts. I do not use HLOD to hide near-field actor activation or gameplay collision problems.

### "What can go wrong with OFPA?"

> OFPA reduces level-file conflicts by storing actors externally in the editor, but the filenames are encoded and partial submits can break references. I use editor changelist validation, redirector/reference checks, clean workspace cook/package and source-control policy rather than reviewing only raw file paths.

## Hands-On Verification Task

Build a small "large world pipeline" proof map or prototype project:

1. Create a World Partition map with three regions: town, wilderness and outpost.
2. Define one always-loaded world-state service and keep durable region state there.
3. Create Runtime Data Layers for at least two world phases, such as `Town_Normal` and `Town_Burned`.
4. Add one Level Instance POI and one Packed Level Blueprint/static visual cluster; document why each was chosen.
5. Enable OFPA and create a source-control/changelist validation note for changed external Actors.
6. Add one PCG or simulated generated-output region; record seed/input/output owner and Data Layer/HLOD policy.
7. Create an HLOD Layer for distant region visuals and build HLOD output.
8. Implement a schematic fast-travel preloader or scripted streaming source and readiness check.
9. Package/cook the map, launch packaged build and run a traversal plus fast-travel scenario.
10. Capture loaded-cell behavior, memory, hitch, Actor/Component count, HLOD transition and Data Layer state evidence.
11. Inject failures: missing runtime layer activation, partial OFPA submit, bad hard reference keeping a region loaded, stale HLOD, and fast travel without readiness.
12. Write a five-minute interview memo: requirements, grid/layer/source policy, durable-state design, validation gates, profiler evidence and remaining risks.

For scenario practice, use [[ue_world_partition_debugging_workbook]]. It contains packaged-world debugging cases for fast travel, Data Layers, OFPA, Level Instances, HLOD, PCG output, cook inclusion and validation gates.

For a broader release-quality proof that combines World Partition/HLOD builder output with BuildGraph/RunUAT packaging, packaged CSV gates, symbols and forced-crash evidence, use [[ue_packaged_performance_build_worldpartition_proof|Packaged Performance, BuildGraph, and World Partition Proof]] and [[ue_packaged_performance_build_worldpartition_workbook]].

## Source Notes

- `SRC-ASSET-010` is the base World Partition source for cells, streaming sources, spatial loading, conversion/build/cook commandlet concepts and related workflow anchors.
- `SRC-WORLD-001` is the Data Layers source for Asset/Instance and Editor/Runtime distinctions.
- `SRC-WORLD-002` is the OFPA/source-control workflow source.
- `SRC-WORLD-003` is the World Partition HLOD source for layer/proxy settings and builder workflow.
- `SRC-WORLD-004` is the Level Instancing source for Level Instance/Packed Level Blueprint and runtime mode caveats.
- `SRC-RENDER-011`, `SRC-PERF-003`, `SRC-PERF-009`, `SRC-BUILD-016`, `SRC-BUILD-017` and `SRC-ASSET-005` connect the world-building workflow to rendering, profiling and release automation evidence.
