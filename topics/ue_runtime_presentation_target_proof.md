# Runtime Presentation Target-Proof: Animation, Physics, UI, Audio, and VFX

See also: [[ue_animation_systems]], [[ue_physics_collision]], [[ue_ui_systems]], [[ue_audio_systems]], [[ue_niagara_vfx]], [[ue_runtime_presentation_target_proof_workbook]].

Version target: **UE5.3-UE5.6**. This chapter is a specialist proof layer for systems that often look correct in editor demos but fail under target runtime conditions: animation authority, physics/collision, UI lifecycle, audio concurrency and Niagara/VFX. Exact console commands, profiler panels, notify/task APIs, Chaos settings, CommonUI/MVVM surfaces, MetaSound/Niagara features and platform support must be verified against the target branch and project plugins. [SRC-ANIM-001] [SRC-PHYS-001] [SRC-UI-001] [SRC-AUDIO-001] [SRC-FX-001] [SRC-PERF-003]

## Why This Exists

These systems are presentation-heavy, so teams often judge them by screenshots, PIE playback or a designer-visible happy path. A production engineer needs a stronger proof:

```text
gameplay truth
  -> animation / physics / UI / audio / VFX presentation
  -> target package
  -> multiplayer/authority and lifecycle events
  -> profiler evidence
  -> failure injection
```

The core question is not "does it look good once?" It is "does the system behave correctly when owners die, widgets recycle, abilities cancel, clients correct, packages load cold, platform budgets apply and hundreds of instances run?"

See also: [[ue_animation_systems]], [[ue_physics_collision]], [[ue_ui_systems]], [[ue_audio_systems]], [[ue_niagara_vfx]].

## Unified Presentation Proof Packet

Create one packet per feature or vertical slice:

```yaml
feature: CombatHit_ReactionStack
build:
  engine_revision: "target branch"
  project_revision: "commit/changelist"
  package: Win64 Development Game
scenario:
  map: /Game/Maps/CombatArena
  net_mode: dedicated_server_two_clients
  latency_ms: 100
  sample_window: 90s
systems:
  animation: montage_section, notify_window, root_motion_policy
  physics: hit query, collision profile, ragdoll_constraint_policy
  ui: local-player HUD, damage feed list recycling
  audio: predicted local hit, replicated impact, concurrency group
  vfx: Niagara impact, pooled component, bounds/significance policy
evidence:
  logs: structured event IDs and net mode
  profiler: CSV + one targeted Insights/GPU/audio/VFX capture
  screenshots_or_video: before_after_or_failure
  injected_failures: owner_destroyed, cancel, stale_widget, wrong_collision, duplicate_audio
result:
  pass_fail: pass
  residual_risks: platform-specific audio mix not checked
```

The evidence packet is deliberately cross-system because bugs cross boundaries. A hit effect may be caused by animation timing, collision query settings, network authority, UI stale data, duplicate audio, VFX pooling or memory/overdraw.

## Animation Proof

### What To Prove

Animation proof covers correctness, authority and runtime cost:

- locomotion state and action/montage layering do not fight;
- notifies are timing signals, not unvalidated gameplay authority;
- cancellation/death/travel cleans up montage, task and cue state;
- root motion or in-place movement policy works under dedicated-server latency;
- retargeting/IK/warping changes preserve gameplay contact points;
- Animation Insights/profiler evidence covers 1/20/100-character scale. [SRC-ANIM-003] [SRC-ANIM-006] [SRC-ANIM-010] [SRC-ANIM-018]

### Notify Window Authority Pattern

```cpp
// Schematic only. Verify montage, ability and network APIs in the target branch.
void UCombatHitWindowComponent::NotifyHitWindowOpened(FName WindowId)
{
    ActiveWindows.Add(WindowId, GetWorld()->GetTimeSeconds());
    UE_LOG(LogTemp, Display, TEXT("HIT_WINDOW_OPEN Window=%s NetMode=%d"),
           *WindowId.ToString(), (int32)GetWorld()->GetNetMode());
}

bool UCombatHitWindowComponent::ServerValidateHit(const FHitClaim& Claim) const
{
    if (!ActiveWindows.Contains(Claim.WindowId))
    {
        return false;
    }

    if (!IsMontageSectionStillAuthoritative(Claim.MontageSection))
    {
        return false;
    }

    return IsTargetWithinServerTrace(Claim.Target, Claim.Trace);
}
```

The point is the proof shape: notifies open a server-validated window; gameplay effect application checks authoritative montage/window/geometry state; cosmetic trails and hit sparks can be predicted and deduped separately.

### Animation Failure Matrix

| Failure | Evidence to capture | Likely owner |
|---|---|---|
| attack hits after cancel | montage section, notify window, ability/task state | gameplay/animation |
| foot slide on proxies | root motion, CharacterMovement correction, smoothing | animation/networking |
| retargeted foot contact wrong | source/target skeleton, IK/warping debug | animation |
| 100 NPC animation hitch | update/evaluate cost, budget allocator, LOD | animation/performance |
| notify fires twice | montage replay, prediction/correction, duplicate binding | animation/networking |

## Physics And Collision Proof

### What To Prove

Physics/collision proof should classify the bug before fixing it:

- query versus simulation versus CharacterMovement versus event notification;
- object type/channel/profile/response matrix;
- simple versus complex geometry and trace/sweep semantics;
- `FHitResult` interpretation, including initial overlap and blocking/touch distinction;
- force/impulse/mass/damping/sleep/CCD/substep choices;
- constraint/ragdoll stability and recovery;
- server authority for gameplay consequences. [SRC-PHYS-001] [SRC-PHYS-002] [SRC-PHYS-003] [SRC-PHYS-006]

### Collision Matrix Test

For any projectile/melee/interactable system, generate a table:

| Source | Target | Expected query | Expected sim | Event expected | Authority |
|---|---|---|---|---|---|
| melee sweep | pawn capsule | block/touch hit | none | server hit claim | server |
| projectile | shield | block | optional sim impulse | hit | server |
| trigger volume | player | overlap | none | begin/end overlap | server or local depending system |
| ragdoll body | world static | block | sim contact | physics contact only | server presentation policy |

Then automate or record at least one representative query per row. A profile existing in the editor is not proof; the target package should log the actual profile/object/response.

### Ragdoll Recovery Proof

Acceptance criteria:

1. Physics Asset bodies/constraints are validated in editor and package.
2. Authority policy is recorded: server-simulated, client cosmetic, or hybrid.
3. Transition into ragdoll stores enough state to recover.
4. Collision profile changes are reversible.
5. Recovery waits for sleep or bounded timeout.
6. Animation blend aligns capsule, mesh and root.
7. Dedicated-server test confirms no duplicate authority or stuck simulated state.

## UI Proof

### What To Prove

UI proof is mostly lifetime and audience proof:

- widgets do not own gameplay truth;
- local-player scope is correct for split screen and ownership;
- async asset loads reject stale widget/list-entry completions;
- virtualised list entries reset every visual/binding/delegate;
- input focus and CommonUI/Enhanced Input contexts transition cleanly;
- low scalability/localisation/accessibility still preserves gameplay readability;
- Slate/UMG invalidation/list/render-target costs are measured. [SRC-UI-001] [SRC-UI-005] [SRC-UI-009] [SRC-UI-012]

### Recycled Entry Contract

```cpp
// Schematic only. Verify ListView interfaces and async load wrappers in target branch.
void UInventoryEntryWidget::BindItem(const FInventoryRowViewModel& InModel)
{
    CurrentRequestId = FGuid::NewGuid();
    CurrentItemId = InModel.ItemId;

    NameText->SetText(InModel.DisplayName);
    QuantityText->SetText(FText::AsNumber(InModel.Quantity));
    Icon->SetBrush(DefaultBrush);

    const FGuid RequestId = CurrentRequestId;
    RequestIconAsync(InModel.Icon, [WeakThis = TWeakObjectPtr<UInventoryEntryWidget>(this), RequestId](UTexture2D* Texture)
    {
        UInventoryEntryWidget* Self = WeakThis.Get();
        if (!Self || Self->CurrentRequestId != RequestId)
        {
            return;
        }

        Self->SetIcon(Texture);
    });
}

void UInventoryEntryWidget::NativeOnEntryReleased()
{
    CurrentRequestId.Invalidate();
    CurrentItemId = FItemInstanceId();
    UnbindOldDelegates();
    ResetTransientVisuals();
}
```

Target proof: rapid scroll, filter, sort, item deletion and async icon delay should never show stale icon/text/selection.

### UI Failure Matrix

| Failure | Evidence to capture | Likely owner |
|---|---|---|
| stale icon in list | request ID, item ID, entry reuse logs | UI/assets |
| player 2 menu controls player 1 | LocalPlayer, focus, input context logs | UI/input |
| health UI stale after pawn death | model owner, possession, replicated source | gameplay/UI |
| UI hitch with 1000 rows | Slate/UMG/Insights, entry count, invalidation | UI/performance |
| low tier hides critical prompt | profile/CVar, screenshot, accessibility check | UI/platform |

## Audio Proof

### What To Prove

Audio proof covers event dedupe, voice budget and routing:

- local-predicted sounds do not double with server replicated events;
- persistent loops start/stop from durable state;
- concurrency/priority/virtualisation policy is intentional;
- Sound Class/Mix/Submix routing is observable;
- MetaSound/procedural graph cost is profiled where used;
- audio memory/streaming/cache behaviour fits platform budget. [SRC-AUDIO-001] [SRC-AUDIO-004] [SRC-AUDIO-007] [SRC-AUDIO-009]

### Multiplayer Audio Event Key

```text
AudioEventKey:
  SourceActorNetId
  LogicalActionId
  SurfaceOrCueType
  PredictionKeyOrServerEventId
  LocalAudience
```

Use an event key for dedupe when the same weapon hit can be predicted locally and later confirmed or corrected by server state. Persistent loops should usually follow replicated state transitions rather than fire-and-forget events.

### Audio Failure Matrix

| Failure | Evidence to capture | Likely owner |
|---|---|---|
| hit sound plays twice | event key, net mode, prediction/server source | audio/networking |
| important VO stolen | concurrency group, priority, virtualisation | audio/design |
| loop never stops | owning state, component lifetime, stop event | audio/gameplay |
| packaged audio missing | cook/stage logs, stream cache, asset ref | audio/build |
| MetaSound CPU spike | voice count, graph complexity, profiler | audio/performance |

## Niagara And VFX Proof

### What To Prove

Niagara proof covers lifetime, bounds, authority, scalability and GPU/CPU cost:

- VFX are presentation, not authoritative gameplay state;
- predicted/local VFX dedupe against replicated/server cues;
- pooled components reset all parameters, owner, attachment and callbacks;
- bounds are correct so effects are not culled early or always visible;
- Effect Type/significance/cull policy preserves critical feedback;
- CPU/GPU simulation, renderer, material/overdraw and memory costs are measured;
- packaged platform supports the chosen features. [SRC-FX-001] [SRC-FX-006] [SRC-FX-007] [SRC-FX-009] [SRC-PERF-003]

### Pool Reset Checklist

Before reusing a pooled VFX component, reset:

- owner/source object;
- transform/attachment/socket;
- user parameters and dynamic material parameters;
- activation/completion delegates;
- auto-destroy/auto-activate flags;
- scalability/significance state;
- event IDs/dedupe keys;
- audio/light/collision side effects;
- hidden/visible state and time dilation assumptions.

### VFX Failure Matrix

| Failure | Evidence to capture | Likely owner |
|---|---|---|
| stale colour/target in pooled effect | user params, owner ID, reset log | VFX/gameplay |
| effect disappears near edge | bounds, camera, culling debug | VFX |
| duplicate hit spark | cue/event key, prediction/server source | VFX/networking |
| GPU overdraw spike | GPU capture, material, screen coverage | VFX/rendering |
| CPU emitter spike | system/emitter stats, spawn/update counts | VFX/performance |

## Cross-System Scenario: Server-Validated Hit Reaction

Build one scenario that touches all five systems:

1. Client starts melee montage with predicted swing trail and local whoosh audio.
2. Notify opens a hit window.
3. Client sends hit claim with operation ID and trace payload.
4. Server validates montage/window/range/collision and applies damage.
5. Replicated damage state updates UI and hit reaction.
6. Audio/VFX use event keys so prediction and confirmation do not duplicate.
7. Physics impulse or ragdoll only applies under authority policy.
8. Widgets update from model state, not direct destroyed target pointers.
9. Target package runs with latency/loss, cancellation, owner death and 20/100-character scale.

### Evidence Table

| System | Required evidence |
|---|---|
| Animation | montage section, notify window, cancellation path |
| Physics | server query, collision profile, `FHitResult` classification |
| UI | health/model update, local-player scope, stale async rejection |
| Audio | event key, concurrency group, duplicate suppression |
| VFX | cue key, pooled reset, bounds/scalability |
| Networking | operation ID, server validation, correction/rejection |
| Profiling | CSV plus targeted trace for worst frame |

See also: [[ue_networking_and_replication]], [[advanced_gameplay_patterns_specialist]], [[ue_profiling_optimisation]], [[ue_runtime_presentation_target_proof_workbook]].

## Interview Answer Patterns

### "How do you prove a hit effect stack is production-ready?"

> I would not judge it only from a PIE video. I would define the authoritative hit operation, montage/notify window, server collision validation, UI model update, audio/VFX event keys and physics/ragdoll policy. Then I would run a packaged dedicated-server scenario with latency, cancellation and owner death, capture logs and CSV, and inject duplicate audio, stale widget and wrong collision failures to prove the guards work.

### "How do you debug duplicate presentation in multiplayer?"

> I first identify whether the event came from local prediction, server confirmation, multicast, replicated state or a recycled component/widget. Then I add a logical event key or operation ID, log net mode/owner/source, and separate durable state from presentation. The fix might be cue dedupe, local-only sound, server-only gameplay effect or stale async rejection.

### "How do you prove UI is correct, not just pretty?"

> I prove ownership and lifecycle. The widget reads a model or view model, not gameplay truth; it handles local-player scope, possession changes, async asset completion, list recycling and package/runtime input focus. I stress it with 1000 rows, fast scroll, pseudo-localisation, gamepad-only input and low device profile.

## Hands-On Verification Task

Create a "presentation proof arena":

1. Implement one attack/action that uses montage or animation timing.
2. Validate hit/collision on the server.
3. Show health/damage feed in UI with recycled list entries and async icon/name loading.
4. Play one predicted/local sound and one replicated/confirmed sound with dedupe.
5. Spawn one Niagara effect through a pool with reset checks.
6. Add one physics reaction or ragdoll variant.
7. Run packaged standalone and dedicated-server two-client scenarios.
8. Inject failures: cancelled montage still damages, wrong collision profile, stale UI entry, duplicate sound, stale pooled VFX, bad bounds.
9. Capture CSV and one targeted trace for the worst frame.
10. Write a report with artifacts, owners and residual risk.

## Source Map

- Animation authority, montages, root motion and profiling: `SRC-ANIM-003`, `SRC-ANIM-006`, `SRC-ANIM-010`, `SRC-ANIM-018`.
- Collision, physics, ragdoll and constraints: `SRC-PHYS-001`, `SRC-PHYS-003`, `SRC-PHYS-006`, `SRC-PHYS-007`.
- UI lifetime, list recycling, focus and performance: `SRC-UI-001`, `SRC-UI-005`, `SRC-UI-009`, `SRC-UI-012`.
- Audio event/routing/concurrency/profiling: `SRC-AUDIO-001`, `SRC-AUDIO-004`, `SRC-AUDIO-007`, `SRC-AUDIO-009`.
- Niagara/VFX lifetime, bounds, scalability and profiling: `SRC-FX-001`, `SRC-FX-006`, `SRC-FX-007`, `SRC-FX-009`.
- Target profiling and networking connections: `SRC-PERF-003`, `SRC-PERF-009`, `SRC-NET-001`, `SRC-NET-006`.
