# Networking Target-Branch Implementation Proof

**Version scope:** UE5.3-UE5.6 target. This chapter is deliberately **source-sensitive**. It does not claim that the curriculum repository contains a UE source checkout or a compilable game project. Its purpose is to define the implementation proof packet a candidate/team should produce inside the real target branch before claiming saved moves, Fast Array, Replication Graph, Iris or NetworkPrediction production readiness.

**Prerequisites:** baseline authority/RPC/property replication, replicated subobjects, Fast Array concepts, CharacterMovement prediction, lag compensation and NetworkPrediction rollback vocabulary. [SRC-NET-001] [SRC-NET-003] [SRC-NET-004] [SRC-NET-006] [SRC-NET-009] [SRC-NET-012] [SRC-NET-015] [SRC-NET-020]

## What "Target-Branch Proof" Means

Target-branch proof is not another explanation of networking concepts. It is an evidence packet that answers:

1. Which exact engine version, changelist and project config were tested?
2. Which headers/source files define the APIs used?
3. Which plugins/systems are enabled in the target build?
4. Which code path was compiled and run in a packaged or multi-process target?
5. Which deliberately broken case failed before the fix?
6. Which logs/profilers prove the fixed path behaves correctly under packet conditions?
7. Which assumptions remain version-sensitive or unavailable in this branch?

Interview shorthand:

> I separate conceptual knowledge from branch proof. For saved moves, Fast Array, RepGraph, Iris and NetworkPrediction, I would show the target headers/config, a minimal compiled feature, forced failure cases, dedicated-server packet tests and before/after profiler/log evidence. Without that, I would call the answer architectural awareness, not implementation proof.

## Proof Packet Manifest

Every networking proof packet should start with a manifest.

```yaml
feature: Project3_NetworkingProof
engine:
  version: "UE 5.x.y"
  changelist: "target branch CL if available"
  source_type: "Epic launcher / source checkout / studio fork"
project:
  net_mode: "dedicated server + remote clients"
  replication_system: "Generic / RepGraph / Iris"
  plugins_enabled:
    - "ReplicationGraph?"
    - "Iris?"
    - "NetworkPrediction?"
builds_tested:
  - "Editor multi-process"
  - "Development Server + Development Client"
  - "Packaged Development or Test where feasible"
packet_conditions:
  latency_ms: [0, 100, 200]
  loss_percent: [0, 2, 5]
  jitter_ms: [0, 30]
evidence:
  source_audit: "docs/networking_source_audit.md"
  logs: "SavedMove_*.log, FastArray_*.log, RepGraph_*.log"
  captures: "NetworkProfiler/Insights captures"
  failure_matrix: "docs/networking_failure_matrix.md"
```

The manifest is intentionally boring. It prevents a common failure: "I implemented it once somewhere" with no proof of branch, config, platform, packet conditions or artefact identity.

## Source Audit Table

Create this table before writing feature code.

| Area | Target files to pin | What to verify | Output |
|---|---|---|---|
| CharacterMovement saved moves | `CharacterMovementComponent.h/.cpp` and any packed move helpers in target branch | virtual signatures, compressed flag availability, move combining, allocation path, replay hooks | one compile note and one saved-move schematic adjusted to target signatures |
| Fast Array | `FastArraySerializer.h` / NetCore headers | item base type, dirty mark calls, callbacks, change/delete limits, push-model interaction | one minimal replicated inventory/status list |
| Replicated subobjects | Actor/channel/object replication headers and docs/source | registered list APIs, legacy override compatibility, stale-list removal path, subobject RPC routing if used | one static/default and one dynamic subobject test |
| RepGraph | ReplicationGraph plugin/module headers and project config | plugin availability, graph class setup, node APIs, connection manager data, platform/splitscreen caveats | adoption/rejection memo plus tiny graph if justified |
| Iris | Iris plugin/config/source | whether enabled, descriptors/fragments, registered subobject requirement, fragment registration for custom UObjects | config proof; no Iris code if disabled |
| NetworkPrediction | plugin headers/source | component/model definition APIs, fixed/independent tick, rollback services, cue traits, debug CVars | toy model compile proof or explicit rejection |

If the source audit cannot be completed, write an explicit "not proven" note. That is stronger than inventing signatures from memory.

## Saved Moves Proof

### Feature Target

Implement predicted sprint and one short predicted dash. Sprint proves a small continuous flag; dash proves an input edge and replay boundary.

### Required State Table

| State | Path | Reason |
|---|---|---|
| `bWantsToSprint` | saved move compressed/custom data | changes movement speed for this move |
| dash pressed edge | saved move custom data or target branch equivalent | one-frame transition must not be merged away |
| dash direction | compact saved move data if not derivable | server/client simulation must reproduce same vector |
| stamina/cooldown | server-owned authoritative state; optionally predicted display | validation, not client authority |
| camera FOV/VFX/audio | presentation only | not movement simulation |
| damage/hit result | authority path outside saved move | saved moves are not hit authority |

### Schematic Target-Adjusted Code Shape

This shape belongs in the proof packet after signatures are adjusted against the target branch. Do not copy blindly.

```cpp
// Header names, virtual signatures and packed move APIs must be pinned
// against the target UE5.3-UE5.6 source branch.

class UProofMovementComponent : public UCharacterMovementComponent
{
    GENERATED_BODY()

public:
    uint8 bWantsToSprint : 1 = false;
    uint8 bDashPressedThisFrame : 1 = false;
    FVector_NetQuantizeNormal PendingDashDirection = FVector::ForwardVector;

    virtual void UpdateFromCompressedFlags(uint8 Flags) override;
    virtual float GetMaxSpeed() const override;
    virtual FNetworkPredictionData_Client* GetPredictionData_Client() const override;

    bool CanStartDash_ServerAuth(const ACharacter& Character) const;
};

class FSavedMove_Proof final : public FSavedMove_Character
{
public:
    uint8 bSavedWantsToSprint : 1 = false;
    uint8 bSavedDashPressed : 1 = false;
    FVector_NetQuantizeNormal SavedDashDirection = FVector::ForwardVector;

    virtual void Clear() override;
    virtual uint8 GetCompressedFlags() const override;
    virtual bool CanCombineWith(const FSavedMovePtr& NewMove, ACharacter* Character, float MaxDelta) const override;
    virtual void SetMoveFor(ACharacter* Character, float InDeltaTime, FVector const& NewAccel, FNetworkPredictionData_Client_Character& ClientData) override;
    virtual void PrepMoveFor(ACharacter* Character) override;
};
```

### Must-Fail Broken Cases

Run each broken branch first and keep the evidence.

| Broken case | Expected symptom | Proof after fix |
|---|---|---|
| sprint stored only in replicated bool | sprint toggles feel delayed; corrections under lag | correction count drops after saved-move integration |
| dash edge not saved | server sees no dash or inconsistent dash | one dash edge appears in client/server move logs |
| `CanCombineWith` ignores sprint/dash | transition disappears inside combined move | transition move is preserved |
| server uses different stamina/cooldown rule | valid-looking client prediction rejected | rejection reason is logged and expected |
| direct `SetActorLocation` dash | rubber-band/correction spike | movement owner path goes through movement simulation |

### Required Logs

```text
MoveProof Role=Autonomous Frame=1024 MoveId=817
Sprint=1 DashEdge=0 DashDir=(1,0,0) Flags=0x10 CanCombine=No
ServerReplay Sprint=1 DashEdge=0 MaxSpeed=840 Correction=No

MoveCorrection Feature=Dash MoveId=821
Reason=CooldownRejected ClientLoc=(...) ServerLoc=(...) Distance=128.4
PendingReplay=4 UX=RejectedCueShown
```

### Acceptance Criteria

- Dedicated server plus at least two clients.
- Sprint and dash run under 0/100/200 ms latency and at least one loss/jitter profile.
- Broken branches produce visible/logged failures before the fix.
- Correction count and distance are recorded before/after.
- Source audit lists target signatures and any differences from public docs.
- No irreversible gameplay side effects are attached to saved moves.

## Fast Array Proof

### Feature Target

Owner-only replicated inventory/status list with 10, 100 and 1000 items. Compare simple whole-array replication against Fast Array only after measuring.

### Item Schema

```cpp
USTRUCT()
struct FProofInventoryItem : public FFastArraySerializerItem
{
    GENERATED_BODY()

    UPROPERTY()
    FName StableItemId;

    UPROPERTY()
    int32 StackCount = 1;

    UPROPERTY()
    uint8 Revision = 0;
};

USTRUCT()
struct FProofInventoryList : public FFastArraySerializer
{
    GENERATED_BODY()

    UPROPERTY()
    TArray<FProofInventoryItem> Items;

    bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms);

    void AddOrChange_ServerOnly(FName StableItemId, int32 NewStack);
    void Remove_ServerOnly(FName StableItemId);
};
```

Use the target header to fill exact traits/macros/callback signatures.

### Mutation Contract

Every mutation must go through one server-owned API:

1. find by stable item ID;
2. mutate stack/revision;
3. mark item or array dirty using target API;
4. emit UI/view-model event only after authoritative state changed;
5. log operation ID, item ID, old revision, new revision and dirty mark path.

### Must-Fail Broken Cases

| Broken case | Expected symptom | Fix proof |
|---|---|---|
| mutation without dirty mark | client misses update | debug validation or packet comparison shows update sent after fix |
| reused item ID | remove/change callback applies to wrong UI item | stable unique ID policy and stale-selection test pass |
| reorder used as identity | UI selection jumps | UI keys by item ID, not index |
| whole array used for large frequent changes | high bytes/frequency | Fast Array capture reduces bytes or is rejected with evidence |
| late join missing state | new client sees partial list | join-in-progress receives authoritative current list |

### Capture Table

| Scenario | Simple array bytes | Fast Array bytes | Server CPU | Client callbacks | Decision |
|---|---:|---:|---:|---|---|
| 10 items, one change/sec | | | | | likely simple array unless proof says otherwise |
| 100 items, ten changes/sec | | | | | measure |
| 1000 items, burst add/remove | | | | | likely Fast Array candidate |

Do not claim Fast Array is "better" without this table.

## RepGraph Proof

### Feature Target

1000 replicated pickup/AI/proxy Actors and 16-64 clients in a synthetic scenario, or the largest scale the target machine can run. The proof is valid if it rejects RepGraph with evidence.

### Actor Policy Classification

| Bucket | Examples | Graph policy | Failure test |
|---|---|---|---|
| global always relevant | match state, objective clock | small always list | add 100 fake globals and prove budget fails |
| owner-only/private | inventory, personal quest clues | owner policy, not spatial | wrong owner leak test |
| spatial | pickups, projectiles, NPCs | grid/zone node | teleport/reclassification test |
| team/faction | team pings, revealed enemies | team node or connection state | team switch/visibility expiry |
| dormant/static | placed interactables | dormant bucket and wake path | mutate while dormant before wake |
| carried/attached | equipped item, flag | follows carrier relevance | carrier change/drop test |

### Minimal Graph Node Schematic

```cpp
// Pseudocode. Use target ReplicationGraph headers for exact signatures.
class UProofSpatialNode final : public UReplicationGraphNode
{
    GENERATED_BODY()

public:
    void AddActor_Dynamic(AActor* Actor);
    void RemoveActor_Dynamic(AActor* Actor);
    void NotifyActorMoved(AActor* Actor, const FVector& OldLocation, const FVector& NewLocation);
    virtual void GatherActorListsForConnection(const FConnectionGatherActorListParameters& Params) override;

private:
    TMap<FIntPoint, FActorRepListRefView> CellLists;
};
```

### Proof Metrics

- relevant Actor count per connection;
- actors considered/gathered per node;
- stale bucket count after movement/teleport;
- always-relevant list size;
- per-connection server net time;
- traffic bytes before/after;
- CPU cost of graph updates;
- error cases for spectators, team swaps and carried Actors.

### Adoption Rule

Adopt RepGraph only if:

1. generic replication tuning is insufficient;
2. profiling shows relevance/list construction is a bottleneck;
3. your Actor/audience model maps cleanly to graph policies;
4. team can own graph-node correctness and debugging;
5. target platform/config supports the plugin and edge cases.

Otherwise write a rejection memo and keep simpler replication.

## Iris Proof

### First Question

Is Iris enabled in the target project/build?

Evidence:

- config key/value or project setting;
- runtime CVar/query output;
- source/module status;
- one log line from startup or replication system selection;
- target branch notes if the feature is experimental or fork-specific.

If Iris is not enabled, stop at awareness/proof-of-status. Do not write Iris-specific code.

### If Iris Is Enabled

Prove:

1. registered subobjects path is used for subobjects that need Iris compatibility; [SRC-NET-012]
2. custom replicated UObject fragment/descriptor registration is implemented only as target source requires;
3. Generic-only assumptions have been removed from copied code;
4. target debug tooling/CVars are available;
5. one dynamic subobject is created, replicated, removed and GC-tested under Iris;
6. migration risk from Generic/RepGraph is documented.

### Iris Misuse Tests

| Misuse | Expected evidence |
|---|---|
| Generic-only `ReplicateSubobjects` copied into Iris path | compile/runtime proof that target requires registered list/fragments |
| stale registered subobject | validation/crash prevention through removal-before-GC test |
| config assumption | startup proof shows Iris actually enabled or disabled |
| custom UObject lacks fragment registration | replication missing or validation failure in target branch |

## NetworkPrediction Proof

Use the existing rollback proof chapter for conceptual state design. The target-branch implementation proof adds compile/run gates.

### Minimal Compile Gate

Produce one of:

- toy counter model compiling and running through the target plugin;
- rollback dash model compiling and running;
- explicit rejection memo proving the plugin API/status is unavailable or unsuitable in this branch.

### Model Audit Table

| Model part | Target source/API | Proof |
|---|---|---|
| component binding | target `UNetworkPredictionComponent`/driver APIs | compile note |
| model definition | target template/definition type | compile note |
| input state | serialisation and frame indexing | packet/log proof |
| sync state | reconcile comparison | forced correction proof |
| aux state | versioning and replay | equipment/tuning swap test |
| cues | trait/dedupe semantics | forced replay no duplicate |
| services | fixed/independent tick, rollback, interpolation, smoothing | CVar/log proof |

### Forced Reconcile Script

Run a deterministic forced mismatch:

```text
1. Client predicts dash with acceleration bias +10%.
2. Server runs normal acceleration.
3. Authority sends correction at frame N.
4. Client restores frame N, replays N+1..current.
5. Dash cue key emitted once.
6. Final sync hash converges.
```

Acceptance:

- logs show input hash, sync hash, aux version and replay window;
- VFX/audio cue does not duplicate;
- damage/rewards/analytics are outside replayed tick;
- smoothing off shows correction clearly; smoothing on improves presentation only;
- CPU/memory/bandwidth table is captured for rollback window.

## Unified Networking Proof Checklist

Before claiming the networking specialist cluster complete in a real project, the packet should contain:

- `networking_source_audit.md`;
- saved-move sprint/dash compile and packet-condition proof;
- Fast Array simple-vs-delta byte/callback comparison;
- replicated subobject lifecycle and stale-list failure proof;
- RepGraph adoption or rejection memo with measured scale data;
- Iris enabled/disabled proof and, if enabled, fragment/subobject tests;
- NetworkPrediction compile proof or rejection memo;
- dedicated server multi-client logs;
- packet lag/loss/jitter matrix;
- Network Profiler/Insights captures;
- failure matrix with broken-before/fixed-after evidence;
- branch-sensitive caveats.

## Interview Answer Patterns

### "Have you implemented custom saved moves?"

> I would answer with a branch-pinned example: I checked the target `CharacterMovementComponent` and `FSavedMove_Character` signatures, captured sprint/dash state in saved moves, prevented combining across transitions, restored state before replay and validated server-side stamina/cooldown. The proof was correction count under packet lag/loss before and after. I would avoid claiming exact signatures without the target source.

### "How do you prove Fast Array was worth it?"

> I compare simple replicated array versus Fast Array under realistic item counts and mutation rates. The proof needs stable item IDs, centralised server mutation, dirty marks, idempotent callbacks, late-join and packet-loss tests, and byte/callback captures. If the list is tiny or changes rarely, I might reject Fast Array and keep simpler replication.

### "Would you use Replication Graph?"

> Only after profiling shows generic relevance/list construction is the bottleneck. I classify Actors by audience, prototype the simplest graph policy if scale justifies it, test stale buckets and edge cases, and compare against tuned generic replication. No profiling, no RepGraph adoption.

### "Does the project use Iris?"

> I would not infer that from UE5. I would show config/source/runtime proof. If Iris is disabled, I keep the answer at awareness. If enabled, I prove registered subobjects and any custom UObject fragment requirements against the target branch.

### "Does NetworkPrediction solve rollback?"

> It can provide a plugin architecture for networked simulation, but the project still needs compile proof, bounded input/sync/aux/cue state, forced reconcile logs, side-effect safety, buffer budgets and target-branch support. I would show a toy or dash model before using it in production.

## Common Red Flags

1. "It compiles in editor" offered as networking proof.
2. No dedicated-server run.
3. No packet lag/loss.
4. No target header/source audit.
5. Fast Array without stable IDs.
6. RepGraph adopted without a scale profile.
7. Iris assumed because the engine is UE5.
8. NetworkPrediction assumed from public API docs without plugin compile proof.
9. Corrections treated as visual noise rather than protocol evidence.
10. Stale subobject pointer removal not tested.

## Source Map

- `SRC-NET-001`, `SRC-NET-003`, `SRC-NET-004`, `SRC-NET-005`, `SRC-NET-006`, `SRC-NET-007`, `SRC-NET-008`: baseline authority, ownership, replication, ordering, RPC, relevancy and dormancy boundaries.
- `SRC-NET-009`, `SRC-NET-017`, `SRC-NET-018`, `SRC-NET-019`: CharacterMovement and saved-move source/API anchors.
- `SRC-NET-011`, `SRC-NET-012`, `SRC-NET-015`, `SRC-NET-016`: component/subobject/Fast Array/debug-CVar proof anchors.
- `SRC-NET-013`, `SRC-NET-014`: RepGraph and Iris proof anchors.
- `SRC-NET-020`: NetworkPrediction public API vocabulary anchor.
- `SRC-NET-010` and `SRC-PERF-003`: capture/profiling proof anchors.

