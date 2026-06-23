# NetworkPrediction Plugin and Full Rollback Proof

See also: [[ue_networking_and_replication]], [[ue_networking_target_branch_proof]], [[ue_gameplay_ability_system]], [[ue_network_prediction_rollback_workbook]], [[ue_hands_on_projects]].

**Version scope:** UE5.3-UE5.6 target, but this chapter is deliberately labelled **plugin-dependent** and **source-sensitive**. The public Epic API page available during this run is a maintained page observed as UE5.8; it proves the existence and vocabulary of the runtime `NetworkPrediction` plugin module, services, state buffers, cue traits and debug CVars, but exact implementation signatures must be checked in the target engine source before writing production code. [SRC-NET-020]

**Audience:** gameplay/network engineer who already understands authority, ownership, RPCs, replicated properties, CharacterMovement prediction, server rewind and the difference between prediction, reconciliation, interpolation and rollback. [SRC-NET-001] [SRC-NET-005] [SRC-NET-006] [SRC-NET-009]

## What Problem This Chapter Solves

The earlier networking chapter explains baseline replication, CharacterMovement saved moves, lag compensation and terminology boundaries. This chapter deepens the remaining gap: how to discuss and prove a **full rollback/resimulation-style** gameplay simulation, especially when a project is evaluating Epic's `NetworkPrediction` plugin or a custom equivalent.

The key interview distinction:

> CharacterMovement prediction is a mature specialised pipeline for character movement. Server rewind evaluates historical state for validation. Full rollback restores old simulation state and resimulates forward. The NetworkPrediction plugin exposes rollback/interpolation/smoothing services and input/sync/aux/cue state vocabulary, but a production feature still needs target-source proof, determinism discipline and side-effect control. [SRC-NET-009] [SRC-NET-020]

Do not answer "we use rollback" unless you can name:

1. the simulation state that can be snapshotted;
2. the input stream that can be replayed;
3. the frame/time index used for comparison;
4. the correction authority;
5. which side effects are replay-safe;
6. how presentation is smoothed after correction;
7. how you force, log and verify reconciles.

## What The Public API Page Proves

The public API page is not a tutorial, but it is useful evidence. It shows:

- the plugin is a runtime plugin under `/Engine/Plugins/Runtime/NetworkPrediction/Source/NetworkPrediction/`; [SRC-NET-020]
- `UNetworkPredictionComponent` is described as the base component for running a `TNetworkedSimulationModel` through an Actor component; [SRC-NET-020]
- the API surface names fixed and independent tick paths, including `TFixedRollbackService`, `TIndependentRollbackService`, interpolation and smoothing services; [SRC-NET-020]
- the vocabulary includes input, sync, aux and cue state, for example comments around server/client replication payloads and `TNetSimOutput`; [SRC-NET-020]
- the API surface includes buffers such as `TNetworkPredictionBuffer` and `TNetworkSimAuxBuffer`; [SRC-NET-020]
- NetSimCue traits distinguish weak/local, replicated, authority-only and resimulated cue behaviours; [SRC-NET-020]
- debug CVars include force/skip reconcile, print reconciles, interpolation and smoothing controls. [SRC-NET-020]

That is enough to build a study proof model. It is **not** enough to copy function signatures or claim a target branch supports every service identically.

## What Still Requires Target-Branch Proof

Before committing to the plugin in a real UE5.3-UE5.6 branch, verify:

1. Is the `NetworkPrediction` plugin enabled, compiled and supported for the target platform/configuration?
2. Which headers define the model definition, driver registration and component binding APIs in that branch?
3. Are fixed tick, independent tick, interpolation, smoothing and rollback services all compiled and usable for the model shape?
4. Are NetSimCue traits and serialisation requirements unchanged from the maintained API page?
5. What CVars exist in the target build, and which are stripped or const in Test/Shipping?
6. How does the model interact with Actor replication, ownership, relevancy, dormancy and subobject replication?
7. What is the exact memory cost of input/sync/aux/cue buffers for the desired rollback window?
8. Which simulation operations are deterministic enough across client/server, platform/compiler and fixed-step settings?

Write the proof packet before writing production feature code.

## Specialist Mental Model

Think of rollback as a contract among four streams:

| Stream | Owns | Must Be Replay-Safe? | Common Bug |
|---|---|---:|---|
| Input command | Player intent for one frame/tick | yes | input contains world pointers, random results or presentation state |
| Sync state | authoritative simulation result that clients compare/correct | yes | missing one gameplay-affecting field causes false "correct" state |
| Aux state | slower-changing or conditionally-written state used by simulation | usually | stale tunable/ability state after correction |
| Cue stream | side effects emitted from simulation | depends on cue trait | sound/VFX/reward replays on each resimulate |

Full rollback does not mean "everything in the Actor rewinds." It means the feature has a bounded simulation model whose relevant state can be restored and replayed. Animation, UI, audio and many Actor components remain presentation systems that consume corrected state.

## Input Command Rules

An input command should be compact, deterministic and serialisable.

Good fields:

- movement axes or quantised aim;
- pressed/released action bits;
- desired ability/action ID;
- local sequence/frame;
- optional client timestamp if the authority validates bounds;
- small deterministic parameters such as charge amount or selected slot.

Bad fields:

- direct `UObject*` or `AActor*` authority decisions from the client;
- random numbers generated outside the model;
- world query results treated as truth;
- cosmetic animation state;
- data derived from current presentation rather than gameplay state;
- large arrays that make bandwidth and resimulation unpredictable.

### Schematic Input Type

This is intentionally schematic. Replace names and registration calls with target-branch API after source verification.

```cpp
struct FDashInputCmd
{
    uint8 bPressedDash : 1 = false;
    int8 MoveX = 0;      // quantised -127..127
    int8 MoveY = 0;
    uint16 LocalFrame = 0;
    uint8 RequestedSlot = 0;

    bool NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess)
    {
        Ar.SerializeBits(&bPressedDash, 1);
        Ar << MoveX;
        Ar << MoveY;
        Ar << LocalFrame;
        Ar << RequestedSlot;
        bOutSuccess = true;
        return true;
    }
};
```

Interview cue: this command still does **not** authorise dash. It only asks the simulation to evaluate dash under server-owned cooldown, resource, movement and anti-cheat rules.

## Sync State Rules

Sync state is the state the client/server compare for correction. If a field can affect future simulation, it usually belongs in sync state or aux state, not hidden in an Actor component.

For a dash/combat micro-simulation:

- position or local simulation transform;
- velocity;
- movement mode/state;
- dash phase and remaining time;
- cooldown/resource state if it affects future acceptance;
- authoritative revision/frame;
- stable interaction target if it affects future movement or collision.

Do not store only visual endpoints. If the next tick uses velocity, cooldown, dash phase or root-motion ownership, those must be reproduced.

### Schematic Sync State

```cpp
struct FDashSyncState
{
    FVector_NetQuantize10 Location;
    FVector_NetQuantize10 Velocity;
    uint8 MovementMode = 0;
    uint8 DashPhase = 0;
    int16 DashTicksRemaining = 0;
    uint16 StateFrame = 0;

    void NetSerialize(FArchive& Ar, UPackageMap* Map, bool& bOutSuccess);

    bool ShouldReconcile(const FDashSyncState& Authority) const
    {
        constexpr float LocationToleranceSq = 4.0f; // 2 cm squared, example only.
        return FVector::DistSquared(Location, Authority.Location) > LocationToleranceSq
            || Velocity != Authority.Velocity
            || MovementMode != Authority.MovementMode
            || DashPhase != Authority.DashPhase
            || DashTicksRemaining != Authority.DashTicksRemaining;
    }
};
```

The tolerance is a design decision. Too tight causes reconcile spam; too loose hides divergence and exploit windows.

## Aux State Rules

Aux state is useful for data that affects simulation but does not change every frame. Examples:

- movement tunables for the current stance/equipment;
- surface friction profile;
- ability tags relevant to the model;
- selected weapon movement modifier;
- server-approved dash table version.

The common mistake is reading these directly from mutable game objects during resimulation. If a resimulate frame reads "current weapon" after the player swapped weapons, old frames are replayed with new data. Snapshot or version the input-relevant aux state.

### Aux State Checklist

For each aux field, answer:

1. Who changes it?
2. Is it replicated before the input that depends on it?
3. Is the old value available during resimulation?
4. Does changing it invalidate pending predicted inputs?
5. Does it need a version number for logs and replay tests?

## Simulation Tick Contract

The model tick must behave like a pure-enough function:

```cpp
// Schematic only: exact NetworkPrediction signatures are branch-sensitive.
FDashSyncState SimulateDashTick(
    const FDashSyncState& Prev,
    const FDashAuxState& Aux,
    const FDashInputCmd& Input,
    const FNetSimTimeStep& Step,
    FDashCueWriter& Cues)
{
    FDashSyncState Next = Prev;

    const FVector DesiredMove = DecodeMove(Input.MoveX, Input.MoveY);
    ApplyAcceleration(Next, DesiredMove, Aux, Step);

    if (Input.bPressedDash && CanStartDash(Prev, Aux))
    {
        Next.DashPhase = 1;
        Next.DashTicksRemaining = Aux.DashDurationTicks;
        Cues.EmitDashStarted(/* replay-safe payload */);
    }

    if (Next.DashPhase != 0)
    {
        IntegrateDash(Next, Aux, Step);
    }

    ResolveModelCollision(Next, Aux, Step);
    ++Next.StateFrame;
    return Next;
}
```

This tick should avoid:

- reading wall-clock time;
- reading current Actor state outside the model;
- performing random queries without controlled seeds/results;
- spawning Actors directly;
- granting rewards directly;
- playing audio or VFX directly;
- mutating inventory, quests or analytics as a side effect of every replay.

## Side-Effect Ledger

Rollback proof fails most often on side effects. Classify every emitted effect.

| Side Effect | Allowed During Resim? | Safer Pattern |
|---|---:|---|
| cosmetic footstep | usually yes if cue trait is chosen correctly | weak/local cue or deduped presentation event |
| dash start VFX | yes if idempotent and replay-safe | cue key = sim instance + frame + cue type |
| damage | no as client-predicted irreversible side effect | authority commit after validation |
| item grant | no | server transaction outside rollback tick |
| cooldown visual | yes as projection | derive from corrected sync state |
| camera shake | maybe | local weak cue with generation/frame dedupe |
| analytics/achievement | no | emit from authoritative commit once |

If a side effect cannot tolerate undo/replay, do not put it inside the resimulated tick path.

## NetSimCue Questions

The public API page exposes cue trait vocabulary including weak local cues, replicated non-predicted cues, authority-only cues and non-replicated resimulated cues. [SRC-NET-020]

Use that vocabulary to ask the right design questions:

1. Who invokes the cue?
2. Who should receive it?
3. Is it safe to undo and replay during resimulate?
4. Does it need `NetIdentical`/dedupe semantics?
5. Is it local-only presentation, replicated presentation or authority-only gameplay?
6. Can it be late-joined or reconstructed from sync state instead?

Do not make a cue responsible for authoritative gameplay mutation. A cue is presentation/notification from the model, not a durable domain transaction.

## Reconcile Proof Harness

A rollback proof is not complete until you can force divergence and demonstrate recovery.

### Minimum Harness

Build a tiny model such as dash movement, charge movement or deterministic projectile steering.

Required toggles:

- artificial client-only acceleration bias;
- artificial server-only collision blocker;
- forced dropped input command;
- forced delayed aux-state change;
- forced reconcile CVar or equivalent target-branch hook;
- deterministic seed mismatch test if random-like behaviour exists;
- cue dedupe on/off switch for demonstration.

Required logs per frame:

```text
Sim=DashModel Instance=17 LocalRole=Autonomous Frame=120
InputHash=4A93 AuxVersion=9 SyncHashBefore=93CA SyncHashAfter=12EE
Cues=[DashStart#120] Reconcile=No
```

On correction:

```text
Reconcile Instance=17 AuthorityFrame=118
Reason=LocationErrorSq:121.0 ToleranceSq:4.0
RestoredFrame=118 ReplayFrames=119..123
PreCorrectionHash=77BE AuthorityHash=21AC PostReplayHash=6D10
CuesSuppressed=[DashStart#120 duplicate]
```

Required acceptance criteria:

1. client predicts and feels responsive under lag;
2. server rejects an invalid command;
3. client restores old sync/aux state and replays pending inputs;
4. final state matches the authoritative state within documented tolerances;
5. cue side effects are not duplicated;
6. irreversible effects occur once on authority;
7. smoothing/interpolation hides correction within the agreed UX envelope;
8. logs identify the exact frame and field that triggered reconcile.

## Debugging Workflows

### Bug: Reconcile Every Frame

Likely causes:

- tolerance too strict;
- client/server tick step differs;
- non-deterministic math or branch;
- aux state version mismatch;
- client reads current Actor state during replay;
- collision/query result is not reproduced;
- input compression loses a meaningful bit.

Workflow:

1. Disable smoothing/interpolation if target CVars support it, so correction is visible. [SRC-NET-020]
2. Print reconciles or equivalent target-branch trace.
3. Log sync-state field diffs, not only "mismatch".
4. Compare fixed step/index and input hash on both ends.
5. Compare aux version and gameplay tags/resources used by the tick.
6. Run a no-lag local client/server test before packet emulation.
7. Re-enable one feature at a time: collision, dash, cues, smoothing.

### Bug: Dash Starts Twice

Likely causes:

- cue fires on predicted frame and again after replay without dedupe;
- both Actor code and model cue play presentation;
- authority replicated cue arrives after local predicted cue and lacks key matching;
- side effect is in tick instead of cue/commit path.

Workflow:

1. Give every cue a deterministic key: instance, frame, cue type, payload hash.
2. Log predicted, replayed and replicated cue delivery separately.
3. Confirm cue trait choice matches desired behaviour.
4. Move gameplay mutation out of cue handlers.
5. Test forced reconcile at the frame where the cue is emitted.

### Bug: Server Accepts Impossible Client Action

Likely causes:

- command includes trusted world result;
- server validates cooldown but not positional plausibility;
- old aux state lets client use stale equipment/movement rules;
- target actor identity is accepted without range/LOS/state checks.

Workflow:

1. Treat input as intent, never proof.
2. Recompute authority checks from server-owned sync/aux/history state.
3. Bound timestamps/frame numbers.
4. Validate resource/cooldown/team/invulnerability.
5. Reject with structured reason and keep client rollback clean.

### Bug: Replay Uses New Weapon Tuning For Old Frames

Likely causes:

- simulation reads weapon data asset or component directly;
- aux state lacks versioned movement/ability parameters;
- equipment swap invalidation does not flush pending prediction.

Workflow:

1. Add aux version to every input log line.
2. Snapshot movement-affecting tunables into aux state.
3. On aux version change, define whether pending inputs are replayed with old aux or rejected.
4. Add a test that swaps equipment exactly before a predicted dash.

### Bug: Smoothing Hides Correctness Failure

Likely causes:

- visual smoothing masks frequent reconcile spam;
- final Actor transform is inspected instead of model sync state;
- QA reports "feels okay" while bandwidth/correction cost is high.

Workflow:

1. Disable smoothing/interpolation in debug where possible. [SRC-NET-020]
2. Count reconciles per minute and replay frames per reconcile.
3. Capture visible correction separately from state mismatch frequency.
4. Set fail thresholds for automated tests.

## Profiling Workflow

Measure rollback as both network and CPU/memory architecture.

### Network

- input command size and send rate;
- sync/aux correction payload size;
- cue replication payloads;
- reconcile frequency under packet lag/loss/jitter;
- bandwidth under idle, normal play and correction storm.

Use Unreal's networking/profiling tools available in the target build and keep Network Profiler/Insights captures as evidence. [SRC-NET-010]

### CPU

- simulation tick time per model instance;
- replay frames per correction;
- worst-case correction burst;
- cue dedupe table cost;
- collision/query cost during replay;
- smoothing/interpolation cost.

### Memory

- input buffer length * command size;
- sync buffer length * sync size;
- aux buffer length * aux size;
- cue history/dedupe memory;
- per-instance overhead;
- debug trace overhead.

### Example Budget Table

| Parameter | Example | Why It Matters |
|---|---:|---|
| input command | 8 bytes | client->server rate scales with players and tick rate |
| sync state | 64 bytes | correction/snapshot memory scales with rollback window |
| aux state | 32 bytes | old versions must be replayable |
| buffer window | 120 frames | 2 seconds at 60 Hz, maybe too much for many instances |
| max replay | 12 frames | visible correction and CPU burst threshold |
| cue history | 64 keys | prevents duplicated local presentation |

The exact values are feature-specific. The important part is proving the budget rather than assuming rollback is free.

## Feature Fit Matrix

| Feature | Good Rollback Candidate? | Reason |
|---|---:|---|
| deterministic dash micro-sim | yes | compact state/input and responsive UX benefit |
| projectile with simple deterministic steering | maybe | collision/history and server validation need care |
| full ragdoll combat | weak | physics determinism and state size are high risk |
| inventory transaction | no | better as server authoritative command/transaction |
| UI cooldown preview | projection only | derive from predicted/corrected state |
| hitscan lag compensation | not necessarily rollback | usually server rewind/query of history, not full resim |
| fighting-game-like local combat | maybe | requires strict deterministic state, inputs and side-effect discipline |

## Integration Boundary With Actors

The model can live under an Actor/component, but the Actor should not become an untracked state bag.

Define:

1. Model-owned state: replayed and corrected.
2. Actor-owned durable state: replicated through normal UE replication.
3. Presentation state: animation, audio, VFX, camera and UI.
4. Authority transactions: damage, inventory, score, quests, analytics.

### Actor Bridge Checklist

- Actor ownership and owning connection are correct before accepting local input. [SRC-NET-003]
- Server RPC/input path is permissioned and rate-limited. [SRC-NET-006]
- Actor replicated properties do not fight the model's sync state.
- Dormancy/relevancy does not hide needed correction state. [SRC-NET-007] [SRC-NET-008]
- Destruction/respawn invalidates model instance IDs and cue histories.
- Late joiners reconstruct from authoritative durable state, not old predicted cues.

## Strong Interview Answer Patterns

### "Does NetworkPrediction give Unreal full rollback?"

> It gives a plugin architecture and services around networked simulation, including rollback, interpolation, smoothing, input/sync/aux state and cue concepts in the public API page. I would not describe that as automatic full-game rollback. A feature still needs a bounded model, serialisable replay-safe state, deterministic-enough ticks, side-effect dedupe, authority validation, buffer budgets and target-branch source proof before production use.

### "How would you prove a predicted dash rollback works?"

> I would build a small model with compact input, sync and aux state; predict locally; validate on the server; force divergence with lag and artificial bias; use reconcile logs to show the restored frame, authoritative field diff and replay window; prove cues dedupe; and measure replay CPU, buffer memory and bandwidth. I would also show that damage/rewards are authority transactions outside the replayed tick.

### "What belongs in sync state versus aux state?"

> Sync state is frame-to-frame simulation truth that clients compare against authority, such as position, velocity, movement phase and state frame. Aux state is slower-changing simulation input such as approved tunables, equipment modifiers or tag/resource snapshots. If replaying old frames with the new value changes results, the old aux value must be available or pending prediction must be rejected.

### "Why do rollback systems duplicate effects?"

> Because the simulation tick is run once during prediction and again during replay. Any side effect emitted directly from the tick can happen twice unless it is represented as a cue with replay-aware traits and deterministic keys, or moved to a one-time authoritative commit path.

### "When would you reject rollback?"

> I would reject it for systems with huge or non-deterministic state, low responsiveness benefit, irreversible side effects, high collision/physics dependence or no tooling budget. A server-authoritative command, CharacterMovement saved move, server rewind or interpolation-only approach may be simpler and more robust.

## Hands-On Verification Task

Build `RollbackDashLab` in a target UE branch.

Minimum evidence:

1. target engine version, plugin status, headers and exact APIs used;
2. one bounded model with input/sync/aux/cue state;
3. local prediction under simulated 100 ms RTT and 2 percent packet loss;
4. forced reconcile from client-only acceleration bias;
5. forced invalid command rejection;
6. cue dedupe proof during replay;
7. no damage/reward in the resimulated path;
8. smoothing on/off comparison;
9. buffer CPU/memory/bandwidth report;
10. adopt/reject memo for the plugin versus custom saved-move/server-authoritative design.

Acceptance criteria:

- final authoritative state converges after correction;
- repeated forced reconcile does not duplicate VFX/audio/rewards;
- logs identify frame, input hash, aux version and field diffs;
- the proof runs in dedicated-server multi-process mode;
- exact target-branch source/API caveats are documented.

## Common Misconceptions

1. **"Rollback means the whole Actor rewinds."** Usually only a bounded simulation model rewinds.
2. **"NetworkPrediction removes server authority."** It does not; server validation still owns truth.
3. **"A cue can apply gameplay damage."** Cues are the wrong boundary for durable authority mutations.
4. **"If it works at zero latency, it works with rollback."** Packet delay, loss and correction expose different bugs.
5. **"Smoothing success means correctness."** Smoothing can hide frequent reconciles.
6. **"Server rewind is rollback."** Rewind queries historical state; rollback restores state and resimulates.
7. **"Public API docs are enough to implement the plugin."** Exact templates and service wiring are target-source details.
8. **"Determinism is only a physics problem."** Randomness, data versions, time, query ordering, floating-point and side effects all matter.

## Source Map

- `SRC-NET-020` anchors the NetworkPrediction plugin API vocabulary: runtime plugin, component/model surface, rollback/interpolation/smoothing services, state/cue/buffer/CVar names.
- `SRC-NET-009` anchors CharacterMovement prediction/correction/replay/smoothing as a separate mature movement pipeline.
- `SRC-NET-001`, `SRC-NET-003`, `SRC-NET-005`, `SRC-NET-006`, `SRC-NET-007` and `SRC-NET-008` anchor baseline authority, ownership, RPC, ordering, relevancy and dormancy constraints around any model.
- `SRC-NET-010` anchors network profiling evidence expectations.

