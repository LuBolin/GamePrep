# World Partition Debugging Workbook

See also: [[ue_world_partition_large_world_pipeline]], [[ue_packaged_performance_build_worldpartition_proof]], [[ue_pcg_procedural_content]], [[ue_assets_loading_cooking]].

Version target: **UE5.3-UE5.6**. This workbook is a practice companion to [[ue_world_partition_large_world_pipeline]]. Exact commandlets, debug views, Data Layer C++ APIs and builder flags must be verified in the target branch. [SRC-ASSET-010] [SRC-WORLD-001] [SRC-WORLD-002] [SRC-WORLD-003] [SRC-WORLD-004]

## How To Use This Workbook

Each scenario is written like an interview debugging prompt:

1. State the symptom in product language.
2. Classify the phase: authoring, source control, streaming, runtime state, rendering, cook/package, target device or automation.
3. Name the first evidence to collect.
4. Form a short hypothesis list.
5. Use a deterministic reproduction.
6. Fix the owner contract, not just the visible symptom.
7. Produce a reusable gate or checklist item.

The point is not to memorise all World Partition UI. The point is to build a repeatable investigation pattern that survives different UE minor versions and studio tooling.

## Evidence Packet Template

For every large-world bug, capture:

| Field | Example |
|---|---|
| Build identity | engine branch, project revision, target platform, config |
| Map and region | `OpenWorld_Main`, `Town_North`, cell/debug ID if available |
| Scenario | traversal path, fast-travel ID, world phase, player speed |
| Streaming state | active sources, requested/loaded cells, readiness result |
| Data Layer state | desired phase, runtime active/loaded state, source of truth |
| Source-control state | changelist, external actor files, changed Actor labels |
| Cook/package state | cook log, staged map/assets, HLOD builder output |
| Runtime telemetry | log markers, CSV, Insights/load trace, memory counters |
| Rendering state | HLOD visible/hidden, primitive/draw/material-section counts |
| Repro rule | exact steps, cold/warm run, target device or device profile |
| Owner | gameplay, world art, tech art, rendering, build, platform or tools |

If the evidence packet cannot identify phase and owner, the debugging process is not finished.

## Scenario 1: Fast Travel Lands In Empty Space

**Symptom:** Fast travel to the outpost works on a developer desktop but sometimes drops the character through missing terrain on target hardware.

**Likely phase:** runtime streaming/readiness, then target I/O/performance.

**First evidence:**

- fast-travel request time;
- destination streaming source location/range/priority;
- cell load/readiness log;
- collision/nav readiness marker;
- Runtime Data Layer state;
- worst-frame load trace around teleport.

**Hypotheses:**

1. Teleport moves the pawn immediately after setting a destination transform.
2. Streaming source exists but loading range is too small for collision/nav.
3. Runtime Data Layer containing collision or destination POI is not active.
4. The target device's storage/memory profile makes the race visible.
5. A previous editor run had the cells already loaded, hiding the bug.

**Investigation steps:**

1. Reproduce from a cold packaged launch, not PIE.
2. Add a request ID to the fast-travel flow: `FastTravelId`, destination, source state, readiness decision.
3. Move/create a destination streaming source and wait for streaming completion.
4. Add a project readiness check: collision below target, nav if required, key spawn/interactable present, Runtime Data Layer active.
5. Add timeout/fallback path that does not leave the pawn in invalid space.
6. Run the same scenario on a low target tier and record time-to-ready.

**Fix pattern:**

Fast travel should be a small state machine:

```text
Request
  -> DestinationSourceEnabled
  -> StreamingReady
  -> GameplayReady
  -> CommitTeleport
  -> CleanupSource
```

**Interview debrief:**

> I would avoid a blind delay. The fix is a streaming-source readiness contract plus project-specific collision/gameplay checks and packaged target proof.

**Sources:** [SRC-ASSET-010] [SRC-PERF-003] [SRC-PERF-009]

## Scenario 2: Region Never Unloads

**Symptom:** Memory stays high after leaving the town. Debug UI suggests the town content remains resident.

**Likely phase:** reference/lifetime policy.

**First evidence:**

- always-loaded Actor list;
- loaded cell list before/after leaving;
- hard reference paths to spatial Actors;
- Level Blueprint references;
- manager/service references to town Actors;
- loaded-object/memory diff.

**Hypotheses:**

1. A manager Actor hard-references a spatial Actor.
2. Level Blueprint holds direct Actor references, causing always-loaded behavior.
3. Cross-cell Actor references bundle the town with another region.
4. A debug/test actor is always loaded and references the whole POI.
5. A soft reference was synchronously loaded and retained by a long-lived owner.

**Investigation steps:**

1. List spatial versus always-loaded Actors.
2. Find hard reference chains from persistent systems to region Actors.
3. Inspect Level Blueprint usage and replace direct references with class/data-driven spawn or stable IDs where appropriate.
4. Move durable state into save/quest/server systems.
5. Re-run traversal and confirm cells unload and memory drops.

**Fix pattern:**

Keep only stable region IDs and durable state in always-loaded systems. Spatial Actors reconstruct presentation when loaded.

**Interview debrief:**

> I would first prove why the content is retained. The fix is usually reference ownership, not only changing grid size.

**Sources:** [SRC-ASSET-001] [SRC-ASSET-010]

## Scenario 3: Quest Phase Visible In Editor, Missing In Package

**Symptom:** The restored village appears in editor testing but not in the packaged build.

**Likely phase:** Data Layer runtime/cook mismatch.

**First evidence:**

- Data Layer Asset and Instance names;
- Editor versus Runtime layer type;
- initial runtime state;
- packaged log of phase state and layer request;
- cook inclusion for assets only referenced by that layer;
- clean workspace reproduction.

**Hypotheses:**

1. The layer is Editor-only.
2. The Runtime Data Layer Instance initial state differs from the editor view.
3. The quest/save system never applies the restored phase in package.
4. Assets referenced only through editor state are not cooked.
5. Local unsaved editor state hid the error.

**Investigation steps:**

1. Confirm layer type and instance configuration.
2. Add a log line: durable phase, desired Runtime Data Layers, result state.
3. Test new game, loaded save and direct map launch in package.
4. Verify cook inclusion for assets used only by the restored phase.
5. Add a validation rule: gameplay phase layers must be Runtime and have owner/system documentation.

**Fix pattern:**

Durable quest/save state drives Runtime Data Layer presentation. Editor visibility is not accepted as gameplay evidence.

**Interview debrief:**

> I separate world truth from layer presentation. Then I prove the packaged runtime layer path, not only the editor view.

**Sources:** [SRC-WORLD-001] [SRC-ASSET-005] [SRC-ASSET-006]

## Scenario 4: Late Joiner Sees Wrong World Phase

**Symptom:** Player A burns the village. Player B joins and sees normal village content for several seconds or permanently.

**Likely phase:** authority and state reconstruction.

**First evidence:**

- server-owned world phase value;
- replication/save reconstruction path;
- client layer activation log;
- join timing relative to cell load and layer change;
- any client-only layer toggles.

**Hypotheses:**

1. Client toggles Runtime Data Layers locally from a UI/event without authoritative state.
2. Server state exists but late join does not replay the phase.
3. Layer activation happens before relevant cells are ready and never retries.
4. Save/load state uses Actor presence instead of stable world-state IDs.

**Investigation steps:**

1. Define server/save source of truth for the phase.
2. On client join, request or replicate durable phase state before presenting layers.
3. Make layer application idempotent: reapply same phase after cells load.
4. Log phase generation and layer readiness per local player.
5. Test join before, during and after the phase transition.

**Fix pattern:**

Use a world-state coordinator that can replay desired Runtime Data Layer state from durable truth.

**Interview debrief:**

> World Partition streaming and replication relevancy are different systems. A client loading a cell is not proof that it owns the correct world phase.

**Sources:** [SRC-WORLD-001] [SRC-NET-001] [SRC-NET-004]

## Scenario 5: Partial OFPA Submit Breaks A POI

**Symptom:** A teammate syncs the latest build and a POI has missing doors, references or Data Layer membership.

**Likely phase:** source control / OFPA collaboration.

**First evidence:**

- source-control changelist;
- editor View Changelist mapping of external actor files to Actor labels;
- missing external actor packages;
- reference/redirector validation;
- clean workspace cook/open result.

**Hypotheses:**

1. Actor external file omitted from submit.
2. Rename/move redirector not fixed or not submitted.
3. Level Instance/Data Layer membership changed in a companion file.
4. Binary conflict resolved by choosing one side.
5. Local workspace had unsaved actors not present in source control.

**Investigation steps:**

1. Do not review only encoded filenames.
2. Use editor/source-control changelist view to map files to Actors.
3. Compare expected Actor labels with submitted files.
4. Run reference validation and a clean workspace cook/open.
5. Add a pre-submit checklist or commandlet report for OFPA-heavy areas.

**Fix pattern:**

Every world-building changelist should be reviewable by Actor intent, not raw encoded external filenames.

**Interview debrief:**

> OFPA reduces conflict size. It does not remove the need for source-control discipline.

**Sources:** [SRC-WORLD-002]

## Scenario 6: Level Instance Content Missing After Embedding

**Symptom:** A repeated camp looks complete while editing the Level Instance but some Actors disappear in the World Partition map or package.

**Likely phase:** Level Instance mode / OFPA compatibility.

**First evidence:**

- Level Instance mode;
- list of contained Actors;
- OFPA support/status for contained Actors;
- package traversal result;
- warnings during conversion/embedding.

**Hypotheses:**

1. Embedded mode discards or cannot carry non-OFPA Actors as expected.
2. The team used Level Streaming mode for many high-density instances and created runtime overhead.
3. Gameplay references target Actors inside instances by unstable paths.
4. The POI should be a normal Actor/data-driven spawn rather than a reusable level arrangement.

**Investigation steps:**

1. Record Level Instance mode and exact branch behavior.
2. Validate contained Actors and source-control representation.
3. Package the map and check actual output.
4. Profile instance count and mode cost if Level Streaming mode is used.
5. If gameplay identity is needed, move identity to stable IDs and runtime systems.

**Fix pattern:**

Choose Level Instance/Packed Level Blueprint from mutability, density, runtime mode and identity needs.

**Interview debrief:**

> I would not choose the container only from editor convenience. I would prove the selected mode in a package.

**Sources:** [SRC-WORLD-004] [SRC-WORLD-002]

## Scenario 7: HLOD Proxy Looks Wrong

**Symptom:** Distant buildings show incorrect colours, missing trim or visible seams after HLOD build.

**Likely phase:** HLOD proxy/material settings.

**First evidence:**

- HLOD Layer type and settings;
- material merge/equivalent material settings;
- proxy mesh/material assets;
- before/after screenshots at fixed camera positions;
- source materials using world/actor position or complex shader logic.

**Hypotheses:**

1. Material merging is invalid for position-dependent materials.
2. Simplification or sampling settings are too aggressive.
3. Merge distance closes or opens gaps incorrectly.
4. Proxy asset is stale relative to source content.

**Investigation steps:**

1. Compare source and proxy at fixed distances.
2. Identify material features lost by merge.
3. Rebuild HLOD from clean content.
4. Adjust HLOD Layer settings or split problematic Actors into a different layer.
5. Capture visual delta and memory/performance cost.

**Fix pattern:**

HLOD settings are content policy. Some assets need separate layers or no material merge.

**Interview debrief:**

> I would make the artefact measurable and tie the HLOD setting to content characteristics, not random slider changes.

**Sources:** [SRC-WORLD-003] [SRC-RENDER-011]

## Scenario 8: HLOD Hides A Near-Field Hitch

**Symptom:** The city is cheap at a distance, then hitches badly when the player crosses the proxy handoff range.

**Likely phase:** runtime activation/performance.

**First evidence:**

- far, transition and near captures;
- Game/Draw/RHI/GPU time;
- cell activation timing;
- Actor/Component count;
- collision/nav/AI spawn timing;
- shader/texture streaming activity.

**Hypotheses:**

1. HLOD reduced far object cost but near cells spawn too many Actors.
2. Collision/nav/AI setup runs in the same frame as visual handoff.
3. Texture/shader/PSO work occurs at the transition.
4. HLOD transition range is too close for target hardware.

**Investigation steps:**

1. Capture a short Insights trace around the transition.
2. Record counts at far, transition and near positions.
3. Disable or change one suspected system at a time: AI spawn, collision, material, streaming range.
4. Move preload earlier or split activation phases.
5. Use HLOD only for the measured cost it actually reduces.

**Fix pattern:**

Separate distant representation optimisation from near-field activation budgeting.

**Interview debrief:**

> I would not call this an HLOD failure until I prove which thread/resource spikes at the handoff.

**Sources:** [SRC-WORLD-003] [SRC-PERF-003] [SRC-RENDER-011]

## Scenario 9: PCG Output Duplicates After Cell Reload

**Symptom:** Trees or rocks double after leaving and returning to a partitioned/generated area.

**Likely phase:** PCG output ownership and cleanup.

**First evidence:**

- graph version, seed and input identity;
- generation mode;
- output owner/component;
- cleanup/regeneration log;
- Actor/Component/instance counts before and after reload;
- Data Layer/HLOD assignment.

**Hypotheses:**

1. Regeneration does not clean previous output.
2. Output is owned by a transient/spatial object that is reconstructed incorrectly.
3. Runtime generation runs on every load without idempotent ownership.
4. Generated Actors are used for set dressing where instances would be cheaper.

**Investigation steps:**

1. Log generation request ID, seed and owner.
2. Count output before/after disable, reload and regenerate.
3. Verify cleanup path and ownership.
4. Test packaged runtime if runtime generation is enabled.
5. Move deterministic editor-generated output or instance-friendly representation where appropriate.

**Fix pattern:**

Generated output needs stable ownership and cleanup, just like hand-authored content.

**Interview debrief:**

> I separate graph execution from output lifetime. Duplicated trees are usually an ownership/cleanup bug, not a random-number bug.

**Sources:** [SRC-PCG-001] [SRC-PCG-004] [SRC-ASSET-010]

## Scenario 10: Package Missing A World Phase Asset

**Symptom:** A rare event layer works in editor but package logs missing meshes or classes when the event starts.

**Likely phase:** cook dependency closure.

**First evidence:**

- event layer assets and references;
- hard/soft reference paths;
- Primary Asset or cook rules;
- cook log/manifests where available;
- packaged log for load failure.

**Hypotheses:**

1. Asset exists only through editor-loaded state.
2. Soft reference path is not part of managed cook inclusion.
3. Data Layer content has assets excluded by platform/chunk rule.
4. Redirector hides a moved path in editor.

**Investigation steps:**

1. Reproduce from clean packaged launch.
2. Identify exact missing path/class and owning system.
3. Audit hard/soft reference closure and cook rules.
4. Repair through Asset Manager/Primary Asset/explicit cook policy, not "cook everything" by default.
5. Add a packaged event activation smoke test.

**Fix pattern:**

Cook inclusion must be explicit enough for rare world phases, optional content and soft-reference-heavy systems.

**Interview debrief:**

> Editor success is weak evidence for package correctness because editor discovery and already-loaded content hide missing cook roots.

**Sources:** [SRC-ASSET-001] [SRC-ASSET-003] [SRC-ASSET-005] [SRC-ASSET-006]

## Scenario 11: Multiple Runtime Grids Added For Organisation

**Symptom:** Team adds separate runtime grids for towns, roads and forests because it seems organised, then streaming performance gets worse.

**Likely phase:** grid policy/performance.

**First evidence:**

- number of runtime grids;
- cell counts and loading ranges per grid;
- streaming-source decisions;
- traversal memory/I/O/cell activation data;
- reason each grid exists.

**Hypotheses:**

1. Grids were added for authoring labels rather than runtime needs.
2. More grid evaluations/cell interactions increase overhead.
3. Data Layers or content tags would solve organisation without runtime-grid cost.
4. Loading ranges overlap and inflate residency.

**Investigation steps:**

1. Ask what runtime behavior each grid uniquely provides.
2. Compare one-grid and multi-grid traversal on the same path.
3. Move authoring organisation to Data Layers/folders/tags when runtime behavior is identical.
4. Keep multiple grids only where measured behavior justifies complexity.

**Fix pattern:**

Runtime grids are runtime policy, not merely content taxonomy.

**Interview debrief:**

> I would treat additional grids as a measured performance/streaming choice. The official docs warn that multiple runtime grids can affect performance.

**Sources:** [SRC-ASSET-010]

## Scenario 12: Stale HLOD Shipped After Content Change

**Symptom:** A building was removed near release, but the distant proxy still shows it in the packaged build.

**Likely phase:** builder automation/stale generated artefact.

**First evidence:**

- HLOD builder output timestamp/log;
- source Actor change;
- proxy asset change;
- package/cook log;
- fixed-camera far screenshot.

**Hypotheses:**

1. HLOD builder was not run after source change.
2. Generated proxy asset was not submitted.
3. Cook used stale derived/generated data.
4. CI only opened the map but did not validate HLOD build output.

**Investigation steps:**

1. Rebuild HLOD from clean workspace.
2. Compare source change to proxy output.
3. Confirm generated HLOD artifacts are included/submitted as project policy requires.
4. Add a release gate: builder run, changed-output check and fixed-camera visual smoke.

**Fix pattern:**

Treat HLOD output as generated release content with an owner, build step and stale-output detection.

**Interview debrief:**

> HLOD is part of the build pipeline. If stale proxies can ship, the validation gate is incomplete.

**Sources:** [SRC-WORLD-003] [SRC-BUILD-016] [SRC-BUILD-017]

## Commandlet Gate Pattern

Use commandlets/builders as evidence producers, not magic correctness boxes:

```text
Gate: LargeWorld_PreSubmit
  Input:
    map path
    changed actor list
    target branch
    target platform tier
  Steps:
    1. Validate external actor/changelist mapping.
    2. Run reference policy checks for spatial -> always-loaded violations.
    3. Validate Runtime Data Layer ownership/initial state metadata.
    4. Validate Level Instance mode and contained actor policy.
    5. Validate PCG/generated-output owner and cleanup metadata.
    6. Run or verify HLOD builder output.
    7. Cook/package representative map or run a cheaper pre-submit subset.
    8. Emit stable report lines and owner labels.
```

Sample report shape:

```json
{
  "map": "OpenWorld_Main",
  "build": "CL_123456",
  "checks": [
    {
      "id": "WP_REF_001",
      "severity": "error",
      "owner": "gameplay",
      "actor": "BP_QuestManager",
      "message": "Always-loaded manager hard-references spatial actor BP_TownDoor_17",
      "suggested_fix": "Use stable door ID and query loaded actor at interaction time"
    },
    {
      "id": "WP_HLOD_004",
      "severity": "warning",
      "owner": "world_art",
      "region": "Town_North",
      "message": "HLOD proxy older than source actor changes",
      "suggested_fix": "Rebuild HLOD and submit generated output"
    }
  ]
}
```

## Interview Drill Prompts

Answer each in five minutes with one diagram, one failure mode and one validation step:

1. A designer wants five quest states for one village. How do you use Runtime Data Layers without making them save authority?
2. A streaming source says complete, but player still falls. What else can be missing?
3. A map opens locally but fails for teammate after OFPA submit. What do you check first?
4. Distant HLOD is cheap, but entering the region hitches. How do you split representation cost from activation cost?
5. A Level Instance POI needs unique loot and quest state per copy. Where should identity live?
6. A generated biome is deterministic in editor but duplicates in package. What ownership data is missing?
7. A rare event asset is missing only in package. How do hard/soft references and cook rules enter the answer?
8. Why is a target traversal run better evidence than a static editor screenshot?

## Self-Grading Rubric

| Score | Answer quality |
|---|---|
| 1 | Names a UE feature only, no evidence path |
| 2 | Gives plausible fix but no phase classification |
| 3 | Identifies phase, gathers evidence and proposes a fix |
| 4 | Adds durable ownership/source-control/cook/performance implications |
| 5 | Produces a reusable gate, target proof and clear owner assignment |

An interview-ready answer is at least a 4 for common P2 prompts and a 5 for senior/tools/world-building roles.
