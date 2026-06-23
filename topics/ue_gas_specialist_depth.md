# GAS Specialist Depth: Prediction, Effects, Tasks and Debugging

See also: [[ue_gameplay_ability_system]], [[ue_network_prediction_rollback_proof]], [[ue_networking_and_replication]], [[ue_gas_specialist_workbook]], [[ue_interview_question_bank]].

This is the second-pass GAS chapter. The baseline chapter explains the framework shape; this file focuses on the implementation questions that distinguish "I have used GAS" from "I can debug and design a production GAS slice."

Version target: **UE5.3-UE5.6**. GAS is plugin-dependent and API details are branch-sensitive. Code below is intentionally schematic where exact signatures differ; verify names and helper macros in the target engine branch before copying. [SRC-GAS-001] [SRC-GAS-002] [SRC-GAS-003]

## 1. Specialist mental model

Treat GAS as a distributed state machine with three overlapping ledgers:

| Ledger | Owns | Typical evidence | Failure smell |
|---|---|---|---|
| ASC gameplay state | granted specs, attributes, tags, active effects, prediction state | ASC dump, spec handle, effect handle, tag count, attribute value | wrong owner/avatar, leaked tag/effect, stale grant |
| Ability activation state | activation request, commit, tasks, target data, cancellation/end | activation trace, prediction key, task lifecycle, failure tags, end reason | cost charged twice, active ability never ends, callback after cancel |
| Presentation state | cues, montages, VFX/audio/UI feedback | cue trace, montage state, local prediction marker, UI model update | duplicate cue, stuck montage, UI shows predicted value forever |

Do not debug these as one undifferentiated "ability bug." A strong workflow says which ledger diverged first.

## 2. Minimum evidence contract

Every non-trivial ability should be able to emit a compact trace row in development builds:

```text
Time, NetMode, Role, Controller, OwnerActor, AvatarActor,
ASC, AbilityClass, SpecHandle, ActivationPredictionKey,
ActivationMode, InputEventOrGameplayEvent, TargetDataSummary,
CanActivateResult, FailureTags, CommitResult,
CostEffectHandle, CooldownEffectHandle, AppliedEffectHandles,
TaskName, TaskState, EndReason, bWasCancelled
```

Add the same correlation fields to cue, attribute-change and target-validation logs. This lets you line up a client-predicted activation, the server's validation, effect application, cue execution and final UI update. [SRC-GAS-011] [SRC-GAS-013]

### Useful invariants

- A grant has an authority source and a removable handle.
- An activation has exactly one terminal path: complete, cancel, reject or end-on-owner-destroy.
- A committed cost/cooldown has an intended refund/no-refund policy.
- Every loose tag has one documented owner and symmetric removal.
- Every target-data packet is evidence, not authority.
- Every non-cosmetic side effect is either server-owned or reversible under prediction.

## 3. ASC placement proof, not preference

"Put the ASC on PlayerState" and "put it on the Pawn" are both incomplete answers. The specialist answer proves the chosen lifetime.

| Requirement | Pawn-owned ASC pressure | PlayerState-owned ASC pressure |
|---|---|---|
| State dies on death/possession | simpler | may need explicit reset |
| Cooldowns/effects persist through respawn | awkward | natural |
| AI minions with short lifetimes | natural | usually unnecessary |
| Player attributes shown on scoreboard | extra projection | natural owner state |
| Fast possession swaps | refresh/regrant carefully | owner survives, avatar refresh required |
| Owner-only effect detail | owner connection still matters | PlayerState relevancy/update rate must be checked |

The acceptance test is a dedicated server with a remote owning client and simulated proxy:

1. spawn and possess the first avatar;
2. grant startup and equipment abilities once;
3. activate a predicted ability;
4. die and respawn with a new avatar;
5. confirm preserved or reset state matches design;
6. prove old-avatar delegates, input bindings and tasks are gone;
7. activate again under lag/loss. [SRC-GAS-002] [SRC-GAS-010]

## 4. Idempotent grants

Duplicate grants are common because startup code runs from possession, respawn, equipment replication, loadout rebuilds and PIE lifecycle. Build a small grant ledger rather than assuming one class equals one grant.

```cpp
// Schematic. Verify exact GAS types/helpers in the target branch.
USTRUCT()
struct FAbilityGrantRecord
{
    GENERATED_BODY()

    UPROPERTY()
    FName SourceId;

    UPROPERTY()
    TSubclassOf<UGameplayAbility> AbilityClass;

    UPROPERTY()
    FGameplayAbilitySpecHandle Handle;

    UPROPERTY()
    int32 Level = 1;
};

void ULoadoutAbilityGranter::ApplyGrantSet(UAbilitySystemComponent* ASC, FName SourceId)
{
    if (!ASC || !ASC->GetOwner() || !ASC->GetOwner()->HasAuthority())
    {
        return;
    }

    for (const FAbilityGrantDef& Def : Grants)
    {
        if (ActiveGrantBySource.Contains(SourceId, Def.AbilityClass))
        {
            continue; // idempotent rebuild
        }

        FGameplayAbilitySpec Spec(Def.AbilityClass, Def.Level);
        FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(Spec);
        ActiveGrantBySource.Add({SourceId, Def.AbilityClass, Handle, Def.Level});
    }
}

void ULoadoutAbilityGranter::RemoveGrantSet(UAbilitySystemComponent* ASC, FName SourceId)
{
    for (const FAbilityGrantRecord& Record : ActiveGrantBySource.FindBySource(SourceId))
    {
        ASC->ClearAbility(Record.Handle);
    }
    ActiveGrantBySource.RemoveSource(SourceId);
}
```

Interview point: the handle belongs to a runtime grant. The class is not enough when equipment, buffs or progression can grant the same ability more than once. [SRC-GAS-003] [SRC-GAS-010]

## 5. Attribute invariants across all mutation paths

An attribute invariant is not "clamp in one callback." It is a policy proved through base changes, instant effects, duration modifiers, executions, replication and UI observation. [SRC-GAS-004]

### Health example

Policy:

- `Health` is always between `0` and `MaxHealth`.
- `MaxHealth` reduction clamps current health but does not generate extra damage attribution.
- Incoming damage is server-authoritative and may route through shield/armour before Health.
- UI observes final replicated/predicted value through a model, not direct polling.

Schematic pattern:

```cpp
// Schematic; exact macros, replication helpers and callback signatures are branch-sensitive.
void UCombatAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
    if (Attribute == GetMaxHealthAttribute())
    {
        NewValue = FMath::Max(NewValue, 1.0f);
    }
}

void UCombatAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    if (Data.EvaluatedData.Attribute == GetIncomingDamageAttribute())
    {
        const float Damage = GetIncomingDamage();
        SetIncomingDamage(0.0f);

        const float NewHealth = FMath::Clamp(GetHealth() - Damage, 0.0f, GetMaxHealth());
        SetHealth(NewHealth);

        if (NewHealth <= 0.0f)
        {
            // Notify authoritative death system once. Do not run durable death from UI/cue.
        }
    }
    else if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
        SetHealth(FMath::Clamp(GetHealth(), 0.0f, GetMaxHealth()));
    }
}
```

Validation matrix:

| Mutation route | Expected proof |
|---|---|
| Instant damage GE | server Health changes, client display converges |
| Healing over max | clamped without repeated delegate storm |
| MaxHealth buff removed | Health clamps once, no fake damage event |
| Predicted cost | client provisional value reconciles |
| Replication notify | UI receives final value and no stale old-avatar binding |

## 6. Effects, specs, captures and executions

The mistake is to say "use an ExecutionCalculation for damage" without saying why. Pick the lightest mechanism that expresses the rule and evidence needs.

| Mechanism | Use when | Avoid when |
|---|---|---|
| Direct modifier | simple scalar change on one attribute | multiple derived outputs, complex branch rules |
| Scalable float / curve | content-level magnitude varies by level/data | runtime context controls most of the value |
| Set-by-caller | application-time magnitude is supplied by the ability/projectile/charge | client can spoof final magnitude or missing values go unvalidated |
| Custom magnitude calculation | one derived magnitude from captured attributes/tags | you need to emit several output modifiers |
| Execution calculation | authoritative multi-attribute result such as armour, shield, crit and health | local prediction demands reversible client-side certainty |

Capture timing must be part of the design. A projectile launched while buffed and landing after the buff expires may intentionally use launch-time or impact-time attack power. Both are valid; unspecified timing is the bug. [SRC-GAS-005] [SRC-GAS-009]

### Execution debug log

For a damage execution, log:

- source ASC, target ASC and effect spec handle/context;
- source tags and target tags relevant to the calculation;
- captured attack, defence, shield and resistance values;
- set-by-caller values and defaults;
- random/crit seed policy if any;
- every output modifier and final death/reaction decision.

Do not leave designers with only "damage was zero."

## 7. Complete stacking policy

A stacking design is complete only when this table has answers:

| Question | Example answer |
|---|---|
| Aggregate by source, target, ability class or item? | poison stacks by source item, haste by target |
| Max count? | poison 5, haste 1 |
| Refresh duration on new stack? | poison refreshes, haste replaces only if stronger |
| Expiration removes one stack or all? | poison removes one per expiry |
| Period reset on stack? | poison period does not reset |
| Overflow behaviour? | at 5, new stack deals burst and refreshes |
| Granted tag while any stack remains? | `State.Poisoned` counted while stack count > 0 |
| UI source attribution? | show strongest source and total count |
| Save/travel persistence? | non-persistent unless explicitly serialised |

Debug a stacking bug by inspecting the active effect handle, stack source key, duration/period policy, granted tag count and UI model. Do not fix a leaked stack by clearing the whole tag namespace. [SRC-GAS-005] [SRC-GAS-006]

## 8. Prediction: two executions and one authority

Local prediction means the owning client may run a supported provisional path before the server result arrives. It does not mean the server trusts final state. [SRC-GAS-011] [SRC-GAS-013]

For a predicted ability, draw two timelines:

```text
Owner client: input -> local CanActivate -> prediction key -> provisional commit/cue/task -> waits
Server:       RPC/request -> authority CanActivate -> validate target/cost/tags -> commit/effects -> confirm/reject
Owner client: confirmation keeps state OR rejection rolls back/reconciles supported predicted work
```

### Side-effect ledger

| Side effect | Prediction policy |
|---|---|
| cost/cooldown GE | only through supported predicted GAS path; test rejection |
| camera shake, local sound | cosmetic; tolerate cancel/duplicate prevention |
| montage | clear/interrupt on rejection/cancel |
| movement impulse/dash | server validates final movement; custom movement may need CharacterMovement or separate proof |
| damage/kill/objective reward | server only |
| inventory mutation | server transaction, not local prediction |
| spawned projectile | use a deliberate predicted-projectile architecture; do not improvise |

### Schematic activation flow

```cpp
// Schematic only. Real code must use target-branch GAS and movement APIs.
void UGP_DashAbility::ActivateAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    const FGameplayEventData* TriggerEventData)
{
    UAbilitySystemComponent* ASC = ActorInfo ? ActorInfo->AbilitySystemComponent.Get() : nullptr;
    if (!ASC)
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    const FVector RequestedDirection = ReadAndClampDirection(TriggerEventData);
    if (!IsDirectionPlausible(RequestedDirection))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // Predicted cosmetic/presentation side effects must be paired with rejection cleanup.
    StartDashCueAndMontage(RequestedDirection);

    if (HasAuthority(&ActivationInfo))
    {
        ApplyAuthoritativeDashState(RequestedDirection);
    }

    StartDashTaskWithExactlyOnceEnd(Handle, ActorInfo, ActivationInfo);
}
```

The point is not the exact helper names. The point is the order: validate enough to avoid nonsense, commit deliberately, separate provisional cosmetic/movement from authority, and guarantee a terminal cleanup path.

## 9. Target data validation

Client target data is a proposal. The server should derive or bound the final result.

Validation checklist:

1. owner/client is allowed to activate this spec now;
2. request has a valid prediction/activation context;
3. input timestamp or sequence is plausible;
4. target actor/component still exists and is targetable;
5. range, angle and line of sight pass with server state or accepted rewind policy;
6. fire cadence, ammo, resources and cooldown are valid;
7. team/faction/immunity/dead-state rules pass;
8. hit surface/bone data is optional evidence, not final damage;
9. set-by-caller magnitudes are recomputed or bounded server-side;
10. rejection has a visible failure path for owner UX.

For hitscan, this may combine GAS target data with a server rewind/history system. Keep terms precise: server rewind evaluates historical state; it is not automatically full rollback. [SRC-GAS-011] [SRC-NET-004]

## 10. Ability Tasks: cancellation is the product

Ability Tasks are powerful because they wrap asynchronous work inside ability lifetime. They are dangerous when they outlive the ability's truth. [SRC-GAS-007]

Every custom task should specify:

- owning ability and ASC;
- activation preconditions;
- external delegates/timers/latent handles it binds;
- completion, cancellation and owner-destroy behaviour;
- whether it broadcasts after cancellation;
- thread/game-thread assumptions;
- whether late events are ignored or converted into failure.

### Schematic custom task cleanup

```cpp
// Schematic; verify inheritance, macros and delegate APIs in target GAS branch.
void UAbilityTask_WaitValidatedRelease::Activate()
{
    if (!Ability || !AbilitySystemComponent.IsValid())
    {
        EndTask();
        return;
    }

    BoundHandle = InputRouter->OnRelease.AddUObject(this, &ThisClass::HandleRelease);
}

void UAbilityTask_WaitValidatedRelease::HandleRelease(const FReleasePayload& Payload)
{
    if (bHasEnded || !Ability || !AbilitySystemComponent.IsValid())
    {
        return;
    }

    if (ShouldBroadcastAbilityTaskDelegates())
    {
        OnRelease.Broadcast(Payload);
    }

    EndTask();
}

void UAbilityTask_WaitValidatedRelease::OnDestroy(bool bInOwnerFinished)
{
    bHasEnded = true;
    if (InputRouter && BoundHandle.IsValid())
    {
        InputRouter->OnRelease.Remove(BoundHandle);
    }

    Super::OnDestroy(bInOwnerFinished);
}
```

The acceptance test is to cancel the ability during every wait phase and prove no late gameplay mutation occurs.

## 11. Gameplay Cues without authority leakage

Gameplay Cues should survive loss, duplication, prediction and removal timing without changing truth. [SRC-GAS-008]

Rules:

- cue notify receives gameplay context; it does not calculate damage;
- predicted cue paths must prevent duplicate local/server presentation where the project requires it;
- persistent cues must clean up on effect removal, owner destroy and relevancy loss;
- cue pooling resets owner, parameters, attachments and callbacks;
- missing cue assets in a package are cosmetic failures, not gameplay failures;
- remote spectators may need different cue detail than owners.

Debug duplicate cues by logging cue tag, prediction key/activation mode, owner/avatar, local role, effect handle and whether the cue came from local prediction, effect replication or explicit execution.

## 12. Failure workflows

### Cost or cooldown applied twice

1. Correlate owner-client prediction key and server activation.
2. Confirm the ability commits once per activation path.
3. Check whether cost/cooldown is applied manually and by `CommitAbility`.
4. Check repeated input trigger versus held input.
5. Check duplicate grants/specs bound to the same input.
6. Confirm rejection rollback/refund policy.
7. Compare standalone, listen server and dedicated remote client.

### Cooldown tag remains forever

1. Inspect active effects that grant the cooldown tag.
2. Check stack/duration/period policy and inhibition.
3. Verify the effect removal handle and source.
4. Look for loose tag additions in input/UI code.
5. Test death/respawn/travel/equipment removal.
6. Compare tag count, not only tag presence.

### Ability works on server but not owning client

1. Validate ASC owner and owning connection.
2. Confirm actor info is initialised on the client after possession/replication.
3. Confirm input adapter targets the current spec handle/avatar.
4. Confirm net execution policy permits local request/prediction.
5. Dump required/blocked tags and failure tags on the client and server.
6. Confirm ability was granted on authority and replicated/known as intended.

### Predicted montage or cue gets stuck

1. Record activation, rejection/cancel and end reason.
2. Confirm montage/cue is started inside the intended predicted/confirmed path.
3. Ensure cancel/reject calls the same cleanup path as interrupt/death.
4. Remove stale old-avatar bindings.
5. Check task `OnDestroy` and external delegate removal.
6. Test with packet loss and forced server rejection.

### Attribute UI diverges from gameplay

1. Compare server attribute, owner client attribute, simulated proxy attribute and UI model.
2. Distinguish base/current, predicted/final and old/new avatar.
3. Confirm Attribute Set registration and replication notify/delegate binding.
4. Check UI bound to owner ASC, not a destroyed Pawn or copied value.
5. Validate that cues/widgets are presentation, not the source of the value.

## 13. Profiling specialist checklist

Do not optimise GAS by removing concepts blindly. First measure:

- ability activations per second by class;
- active effects per ASC, per population and per frame;
- periodic effect execution rates;
- tag query/change/broadcast hot spots;
- custom magnitude/execution calculation time;
- Blueprint task/cue VM time;
- number of concurrent ability tasks and leaked tasks after cancellation;
- cue spawn/pool pressure;
- replication bytes and effect detail per audience;
- allocations during repeated activate/cancel/respawn;
- UI updates caused by attribute/tag delegates.

Typical fixes:

- replace high-frequency periodic effects with lower-frequency aggregate rules where semantics allow;
- move hot execution calculations to C++;
- cache tag queries or simplify taxonomies only after tracing;
- avoid Blueprint polling from widgets;
- reduce cue detail through significance or owner/spectator policy;
- choose GE replication mode from audience requirements, not habit;
- fix leaked tasks/effects before tuning.

## 14. Strong answer patterns

### "How do you debug a predicted GAS ability?"

> I correlate owner-client and server activation by spec handle and prediction key. Then I separate ledgers: activation/commit, target-data validation, effect application, task cleanup and cosmetic cues. I test dedicated-server lag, rejection, cancellation and out-of-order conditions. Anything outside GAS prediction, such as custom movement, inventory or rewards, needs its own reversible or server-only policy.

### "How do you design a damage Gameplay Effect?"

> I decide whether damage is a direct modifier, set-by-caller magnitude, custom magnitude calculation or execution. For multi-attribute combat I usually use an authoritative execution that logs captured source/target attributes, relevant tags, set-by-caller inputs, shield/armour routing and output modifiers. I state snapshot versus live capture timing and keep final damage server-derived.

### "How do you know a GAS task is safe?"

> I can cancel the owning ability at every wait phase and prove all timers, delegates and callbacks are unbound or ignored. The task broadcasts through ability-safe delegates, ends exactly once and never mutates gameplay after cancellation or owner destruction.

## 15. Hands-on verification bundle

Build a GAS evidence map for one predicted dash and one aimed damage ability:

1. trace grant, activation, prediction key, target data, commit, effects, cues, task end and UI update;
2. inject rejection reasons: blocked tag, insufficient stamina, invalid target, stale avatar and missing set-by-caller;
3. force cancellation during every task phase;
4. compare Pawn-owned and PlayerState-owned ASC for one respawn scenario;
5. compare Full, Mixed and Minimal GE replication for owner and simulated proxy;
6. run with dedicated server lag/loss and export log/Insights/network evidence;
7. write a one-page decision: what is GAS-owned, what remains ordinary gameplay architecture and what is cosmetic only.

## 16. Source map

- GAS overview and ASC/owner/avatar model: [SRC-GAS-001] [SRC-GAS-002]
- abilities, grants, specs and lifecycle: [SRC-GAS-003] [SRC-GAS-010]
- attributes and invariants: [SRC-GAS-004]
- effects, specs, captures, stacking and tags: [SRC-GAS-005] [SRC-GAS-006] [SRC-GAS-009]
- tasks, target data, cues and prediction records: [SRC-GAS-007] [SRC-GAS-008] [SRC-GAS-011] [SRC-GAS-013]
- replication modes: [SRC-GAS-012]

