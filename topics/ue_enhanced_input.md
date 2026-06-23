# Unreal Enhanced Input Architecture

See also: [[ue_gameplay_ability_system]], [[ue_networking_and_replication]], [[ue_ui_systems]], [[ue_hands_on_projects]].

## Cluster scope

**Priority:** P1 for modern UE gameplay engineers; P2/P3 for UI, tools, networking and GAS follow-up depth.  
**Expected depth:** D3-D4.  
**Version scope:** UE5.3-UE5.6 preferred. Enhanced Input is a plugin-based UE5-era input stack; the official UE5.6 overview states that it is enabled by default for UE5 projects, but exact APIs, project settings, CommonUI integration and remapping workflows are target-branch sensitive. [SRC-INPUT-001] [SRC-INPUT-002] [SRC-INPUT-003] [SRC-INPUT-004] [SRC-INPUT-005]

Enhanced Input is not "the new place to put key binds." It is a semantic input architecture: devices produce raw values; mapping contexts bind physical controls to semantic actions; modifiers shape values; triggers decide when an action state should fire; gameplay code consumes the action as intent. This gives a project better tools for remapping, local multiplayer, contextual modes, gamepad/mouse/touch parity, ability activation and UI/gameplay input coordination. [SRC-INPUT-001]

## Why it exists

Legacy Action and Axis mappings are global and comparatively flat. They can work for simple projects, but they become awkward when a game needs:

- per-local-player bindings;
- runtime remapping;
- layered context such as on-foot, vehicle, menu, photo mode and spectating;
- 1D keyboard keys feeding a 2D movement action;
- dead zones, sensitivity, inversion and world-space transforms;
- hold, tap, double-tap, chorded and blocker semantics;
- UI and gameplay fighting less over the same button.

Enhanced Input addresses those problems with data assets and a local-player subsystem. Epic's UE5.6 overview calls out contextual mappings, prioritisation, chorded actions, dead zones and custom filtering/processing of raw input values. [SRC-INPUT-001]

The interview signal is judgement: a good answer separates physical input, semantic intent and authoritative gameplay. A weak answer talks only about "binding keys in `SetupPlayerInputComponent`."

## Core mental model

```text
Device / platform input
  -> Mapping Contexts on one Local Player
  -> per-binding Modifiers
  -> per-binding / action Triggers
  -> Input Action value + trigger event
  -> local intent adapter
  -> Pawn/component/ASC/server request/UI command
```

### Input Action

An Input Action is the semantic thing the user can do: Move, Look, Jump, Fire, Interact, OpenInventory, Confirm, Cancel, AbilityPrimary. The UE5.6 overview describes Input Actions as the communication link between Enhanced Input and project code, represented as data assets. They carry value type: Boolean, Axis1D, Axis2D or Axis3D. [SRC-INPUT-001] [SRC-INPUT-002]

Design rule: name actions after intent, not devices. `IA_Fire` is good. `IA_LeftMouseButton` is a hidden key bind.

Common action-value choices:

| Intent | Value type | Notes |
|---|---|---|
| Jump, Confirm, Reload | Boolean | treat press/release/hold through trigger event and trigger configuration |
| Move | Axis2D | keyboard can be converted from four 1D keys with modifiers |
| Look | Axis2D | mouse/gamepad sensitivity and inversion are modifier concerns |
| Trigger pressure | Axis1D | analogue triggers, throttle, charge pressure |
| VR/controller pose-like value | Axis3D | only if the action truly carries 3D value |

### Input Mapping Context

An Input Mapping Context maps physical controls to Input Actions for a specific mode or layer: common gameplay, infantry, vehicle, aircraft, build mode, menu, debug camera, radial wheel. The overview says contexts can be added, removed and prioritised for each user through the Enhanced Input Local Player Subsystem. [SRC-INPUT-001] [SRC-INPUT-003] [SRC-INPUT-004]

The local-player part matters. In split-screen or multi-local-player contexts, two users may have different devices, mappings and UI focus. Do not store a global "current mapping context" in GameInstance and assume it applies to every player.

Good mapping-context design:

- a small always-on common context for universal actions;
- mutually exclusive mode contexts for on-foot/vehicle/menu where the same key means different things;
- a temporary high-priority context for modal actions such as a radial wheel;
- clear add/remove ownership on possession, mode entry/exit, screen activation and teardown;
- deterministic priorities and debug logs.

Bad mapping-context design:

- one enormous context containing every possible action;
- context priority numbers copied without policy;
- adding contexts repeatedly without removing old ones;
- changing contexts from several systems without a coordinator;
- placing gameplay-only mappings in a UI activation path with no restoration rule.

### Modifiers

Input Modifiers alter raw input before triggers evaluate it. Built-in use cases include dead zones, axis swizzling, negation, sensitivity, smoothing, axis inversion and world-space conversion. Epic's overview also shows using modifiers for user settings such as aim inversion. [SRC-INPUT-001]

Modifiers should be deterministic, cheap and local to input interpretation. They should not silently perform gameplay validation or mutate authoritative state. A modifier can read local settings; it should not consume ammo, apply damage, start cooldowns or decide whether the server accepts an ability.

### Triggers

Input Triggers decide whether modified input activates the action. The overview describes action trigger events such as Started, Ongoing, Triggered, Completed and Cancelled, and trigger result states such as None, Ongoing and Triggered. It also distinguishes explicit, implicit and blocker trigger types. [SRC-INPUT-001]

Common trigger use:

| Need | Trigger shape | Architectural caution |
|---|---|---|
| press once | Press/Started or Triggered depending setup | avoid firing every tick by accident |
| hold to charge | Hold/Ongoing/Completed | define commit point: press, threshold, release or server accept |
| tap vs hold | Tap/Hold combination | avoid double execution when both patterns observe same key |
| chord | Chorded action | useful for modifier buttons; can confuse remapping if hidden |
| block in mode | blocker trigger or context removal | context removal is often clearer when mode disallows the action |

The most common trigger bug is binding the wrong event. `Triggered` can fire every tick for continuous axes or repeated triggers. `Started` can be the correct edge for a one-shot local request. `Completed` or `Cancelled` can be essential for release/charge/cancel semantics.

## Where code should live

Input architecture spans several objects:

| Layer | Responsibility |
|---|---|
| `ULocalPlayer` / LocalPlayer subsystem | per-user mappings, remaps, active contexts, user input settings |
| `APlayerController` | local player orchestration, UI/gameplay mode transitions, possession input rebinding |
| Pawn/Character | avatar movement and ability surfaces that AI/replay/network can also drive |
| Actor Components | reusable capability endpoints: movement mode, weapon, interaction, inventory |
| ASC/GAS adapter | map input actions/tags to ability specs/activation/release |
| UI/CommonUI | active screen focus/routing and high-level input mode, not gameplay authority |
| Server RPC path | validates intent and owns durable gameplay mutation |

Do not make the input system the owner of gameplay truth. Enhanced Input samples local user intent. The server, authoritative component, ASC or gameplay system decides whether that intent is valid.

### Controller versus Pawn binding

In Unreal, `SetupPlayerInputComponent` is commonly implemented on a Pawn/Character to bind actions to avatar behaviour. That is appropriate when the action is avatar-specific and possession-driven. But mapping contexts often belong to the local player or controller because they describe what the local user can currently do, not what the Pawn class universally supports.

Practical split:

- PlayerController or local-player input coordinator adds/removes contexts.
- Pawn/Character binds actions to capability methods when possessed.
- Components expose stable methods such as `StartSprint`, `StopSprint`, `RequestInteract`, `BeginFire`, `EndFire`.
- AI or replay can call the same component methods without pretending to be keyboard input.

This keeps input collection separate from gameplay capability.

## C++ binding pattern

The exact signatures vary by branch and project style, but the official overview shows binding through `UEnhancedInputComponent` and receiving an action instance/value. [SRC-INPUT-001] [SRC-INPUT-005]

```cpp
// Header: values are usually asset references configured in Blueprint/defaults.
UPROPERTY(EditDefaultsOnly, Category = "Input")
TObjectPtr<UInputAction> MoveAction;

UPROPERTY(EditDefaultsOnly, Category = "Input")
TObjectPtr<UInputAction> JumpAction;

UPROPERTY(EditDefaultsOnly, Category = "Input")
TObjectPtr<UInputAction> FireAction;

virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

void HandleMove(const FInputActionValue& Value);
void HandleJumpStarted(const FInputActionValue& Value);
void HandleFireStarted(const FInputActionValue& Value);
void HandleFireCompleted(const FInputActionValue& Value);
```

```cpp
void AInterviewCharacter::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    Super::SetupPlayerInputComponent(PlayerInputComponent);

    UEnhancedInputComponent* EnhancedInput = Cast<UEnhancedInputComponent>(PlayerInputComponent);
    check(EnhancedInput);

    EnhancedInput->BindAction(MoveAction, ETriggerEvent::Triggered, this, &AInterviewCharacter::HandleMove);
    EnhancedInput->BindAction(JumpAction, ETriggerEvent::Started, this, &AInterviewCharacter::HandleJumpStarted);
    EnhancedInput->BindAction(FireAction, ETriggerEvent::Started, this, &AInterviewCharacter::HandleFireStarted);
    EnhancedInput->BindAction(FireAction, ETriggerEvent::Completed, this, &AInterviewCharacter::HandleFireCompleted);
    EnhancedInput->BindAction(FireAction, ETriggerEvent::Canceled, this, &AInterviewCharacter::HandleFireCompleted);
}

void AInterviewCharacter::HandleMove(const FInputActionValue& Value)
{
    const FVector2D Move = Value.Get<FVector2D>();
    if (Controller && !Move.IsNearlyZero())
    {
        const FRotator ControlRot(0.0, Controller->GetControlRotation().Yaw, 0.0);
        AddMovementInput(FRotationMatrix(ControlRot).GetUnitAxis(EAxis::X), Move.Y);
        AddMovementInput(FRotationMatrix(ControlRot).GetUnitAxis(EAxis::Y), Move.X);
    }
}

void AInterviewCharacter::HandleFireStarted(const FInputActionValue& Value)
{
    WeaponComponent->BeginFireIntent();
}

void AInterviewCharacter::HandleFireCompleted(const FInputActionValue& Value)
{
    WeaponComponent->EndFireIntent();
}
```

Notes:

- Use the action value type you authored. A Boolean action is not a 2D vector.
- Bind edge events intentionally. A trigger event that fires every tick can turn one shot into spam.
- Keep device names out of the handler. The handler receives semantic intent.
- For networked gameplay, the handler usually calls a local prediction path or owned Server RPC, not direct authoritative mutation.

## Mapping context lifecycle pattern

Mapping contexts are often added through `UEnhancedInputLocalPlayerSubsystem`. The UE5.6 overview shows adding a context by retrieving the local player's subsystem and calling `AddMappingContext` with a priority. [SRC-INPUT-001] [SRC-INPUT-004]

```cpp
void UInputModeCoordinator::ApplyGameplayContext(APlayerController* PC)
{
    if (!PC)
    {
        return;
    }

    ULocalPlayer* LocalPlayer = PC->GetLocalPlayer();
    if (!LocalPlayer)
    {
        return;
    }

    UEnhancedInputLocalPlayerSubsystem* Subsystem =
        LocalPlayer->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>();
    if (!Subsystem)
    {
        return;
    }

    // Keep priorities named in one place. Magic numbers become bugs.
    constexpr int32 CommonPriority = 0;
    constexpr int32 GameplayPriority = 10;

    Subsystem->AddMappingContext(CommonContext, CommonPriority);
    Subsystem->AddMappingContext(OnFootContext, GameplayPriority);
}

void UInputModeCoordinator::EnterVehicle(APlayerController* PC)
{
    UEnhancedInputLocalPlayerSubsystem* Subsystem = GetEnhancedInputSubsystem(PC);
    if (!Subsystem)
    {
        return;
    }

    Subsystem->RemoveMappingContext(OnFootContext);
    Subsystem->AddMappingContext(VehicleContext, 20);
}

void UInputModeCoordinator::ExitVehicle(APlayerController* PC)
{
    UEnhancedInputLocalPlayerSubsystem* Subsystem = GetEnhancedInputSubsystem(PC);
    if (!Subsystem)
    {
        return;
    }

    Subsystem->RemoveMappingContext(VehicleContext);
    Subsystem->AddMappingContext(OnFootContext, 10);
}
```

Production code should be idempotent: repeated possession, respawn, local-player recreation, settings reload or screen activation should not stack duplicate contexts or leave old mode contexts alive.

## Remapping and settings

Runtime remapping is one of the strongest reasons to use Enhanced Input. Do not treat remapping as only a UI screen. It needs a data contract:

1. Which actions are player-mappable?
2. Which bindings are platform/device-specific?
3. Which conflicts are allowed, blocked or context-dependent?
4. Which mapping is persisted per user/local player?
5. Which defaults are restored when an action disappears or a plugin/DLC is disabled?
6. How are glyphs/UI prompts updated?
7. How are invalid or missing hardware bindings handled?

The mapping context is not necessarily the save format. A robust project stores user preferences separately, applies them through a local-player settings/subsystem path, validates conflicts, and then updates contexts. This is especially important with split-screen, shared machines and cloud saves.

## UI and CommonUI boundary

CommonUI can own activatable screen stacks, desired focus and input mode for menus. Enhanced Input owns semantic input actions and mapping contexts. They are adjacent, not replacements. The UI chapter already notes that CommonUI's Enhanced Input integration is version-sensitive and that direct controller input mode calls can fight CommonUI activation. [SRC-UI-010] [SRC-UI-011]

Good policy:

- one coordinator owns transitions between gameplay, menu and modal mappings;
- screen activation can request a UI context, but the request passes through that coordinator;
- closing a modal restores the previous context/focus, not a hard-coded "gameplay" assumption;
- local-player identity is explicit for split-screen;
- gameplay-critical input cannot be swallowed by an invisible widget path without logging/debug proof.

Example failure: pressing Gamepad Face Button Bottom triggers both "Jump" and "Confirm" because gameplay context remains active behind a menu. Fix by context removal, priority policy, trigger/blocker design or one coordinator that switches to UI mode.

## GAS and ability input boundary

For GAS, Enhanced Input should emit semantic input actions; an adapter maps those actions to ability spec handles, gameplay tags or explicit ability requests. The ability should not know that the key was `E` or Right Trigger. [SRC-GAS-001] [SRC-GAS-010] [SRC-INPUT-001]

Good ability-input adapter responsibilities:

- rebuild bindings when abilities/loadout/equipment grants change;
- bind press, release and hold semantics to ability activation/input release paths;
- keep handles/tags stable and log missing specs;
- prevent duplicate bindings after respawn or avatar change;
- keep UI prompts in sync with mapping/remap state;
- pass only validatable client intent to the server.

Common bug: the ASC lives on PlayerState, the Pawn is respawned, and input delegates are rebound on the new avatar while old bindings remain. Symptoms include duplicate ability activation, release never arriving, or abilities firing after death. The fix is idempotent binding ownership, explicit unbind/rebuild and actor-info refresh.

## Networking boundary

Input is local. Authority is server-side. In a networked game:

- local client samples input and may predict presentation or movement;
- owning client sends compact intent through an owned route;
- server validates timing, state, target, cooldown, resources and authority;
- replicated state or correction converges everyone else.

Do not send device events as reliable RPC spam. "Pressed key X" is often the wrong protocol. Send semantic intent with sequence/time where needed: fire request, interaction target, ability activation, movement saved move. For movement-affecting input, CharacterMovement already has a specialised prediction path; custom sprint/dash state must be represented there, not as an unrelated delayed replicated bool. [SRC-NET-009] [SRC-NET-018] [SRC-INPUT-001]

Input architecture should define which intents are:

| Intent type | Typical path |
|---|---|
| local-only UI | handled by local UI/controller |
| cosmetic-only | local prediction plus optional unreliable cosmetic event |
| authoritative gameplay request | owned Server RPC or ability activation path |
| movement-affecting | CharacterMovement/prediction saved-move path |
| replay/rollback deterministic input | command/input log with stable frame/sequence |

## Debugging workflow

Use this when an action does not fire, fires twice or fires in the wrong mode:

1. Confirm the correct local player, controller and possessed Pawn.
2. Confirm the Input Action asset is assigned and value type matches the handler.
3. Confirm the mapping context is added to the correct `UEnhancedInputLocalPlayerSubsystem`.
4. Log current active contexts, priorities and owning system.
5. Check whether another higher-priority context consumes or shadows the same physical key.
6. Inspect modifiers: dead zone, axis swizzle, negate, inversion, scalar and custom code.
7. Inspect triggers: event type, hold/tap/chord/blocker state and whether `Triggered` repeats.
8. Confirm UI/CommonUI focus/input mode is not intercepting the event.
9. Confirm bindings are not duplicated after possession, respawn or screen reopen.
10. For multiplayer, confirm the input path routes through an owning client and server validation, not a client-only mutation.

Useful log shape:

```text
InputTrace LocalPlayer=0 PC=BP_PlayerController_C_0 Pawn=BP_Hero_C_2
  Contexts=[Common:0, OnFoot:10, RadialMenu:50]
  Action=IA_Fire Value=Bool(true) Event=Started Trigger=Pressed
  Route=WeaponComponent.BeginFireIntent Prediction=LocalCosmetic ServerRPC=RequestFire
```

This makes most "input broken" bugs obvious: wrong player, wrong context, wrong priority, wrong event, wrong route.

## Performance workflow

Enhanced Input is rarely the largest runtime cost by itself, but input systems can create high-frequency downstream work:

- continuous `Triggered` handlers doing traces, allocation or UI rebuilds every frame;
- repeated context rebuilds on Tick or focus churn;
- custom modifiers/triggers performing object searches or expensive state queries;
- duplicate bindings multiplying handler calls;
- UI/gameplay context fights causing repeated mode switches;
- remapping screens rebuilding all prompts every input event.

Workflow:

1. Count handler invocations per action per frame.
2. Trace expensive handlers in Timing Insights, not just Enhanced Input internals.
3. Log context add/remove frequency and duplicate binding count.
4. Move expensive semantic work into scheduled/batched systems.
5. Keep modifiers/triggers cheap and side-effect-free.
6. Test low/high input rates, menu open/close loops and split-screen.

Interview answer: "I would profile the work triggered by input, not blame the input plugin first."

## Common bugs and misconceptions

1. **Device action naming:** `IA_LeftMouse` instead of `IA_Fire`, making remapping and gamepad support awkward.
2. **One giant mapping context:** every mode action is active and priority/collision bugs multiply.
3. **Wrong trigger event:** edge action bound to repeating `Triggered`.
4. **Action value mismatch:** handler reads `FVector2D` from a Boolean action.
5. **Duplicate binding:** possession/respawn/reopen binds again without clearing.
6. **Context leak:** vehicle/menu/radial context remains active after exit.
7. **UI/gameplay fight:** CommonUI or direct input-mode calls compete with mapping context changes.
8. **Input as authority:** client mutates game state directly because the key fired locally.
9. **Modifier overreach:** custom modifier reads or mutates gameplay state with hidden cost.
10. **Remap without conflict policy:** two actions become accidentally bound to the same key in the same context.
11. **Global local-player assumption:** split-screen users share settings or contexts incorrectly.
12. **GAS duplicate input:** ability bindings rebuild on respawn without unbinding previous avatar/spec paths.

## Strong interview answer patterns

### "How would you structure input in a modern UE5 project?"

> I would define semantic Input Actions and small Mapping Contexts per mode: common gameplay, on-foot, vehicle, UI/modal and debug where appropriate. Contexts are added to the correct LocalPlayer subsystem with explicit priorities and idempotent lifecycle. Modifiers handle local value shaping such as dead zones, swizzles, sensitivity and inversion. Triggers express tap/hold/chord/press semantics. Gameplay handlers turn action values into intent against Pawn/components or an ability adapter; the server still validates durable state. UI/CommonUI and gameplay mapping changes go through one coordinator so focus, input mode and mapping context do not fight.

### "Why did this input fire twice?"

> I would check duplicate bindings first, then active contexts and trigger event semantics. A respawn or reopen path might have rebound the same action without clearing it, or both gameplay and menu contexts may be active. If it is a hold/axis action, `Triggered` may be firing every tick by design. I would log LocalPlayer, active contexts, priority, action, trigger event and route before changing gameplay code.

### "How does Enhanced Input fit with GAS?"

> Enhanced Input should produce semantic local intent. A separate adapter maps input actions or tags to ability spec handles and press/release events. The ability does not know the physical key. The adapter rebuilds idempotently when grants, equipment, Pawn/avatar or local-player settings change. The ASC/server validates activation, cost, cooldown, tags and target; input is not authority.

## What a three-year engineer should know

A solid three-year UE engineer should be able to:

- explain Input Actions, Mapping Contexts, Modifiers and Triggers;
- choose action value types correctly;
- bind actions in C++/Blueprint and understand trigger events;
- add/remove contexts for the correct LocalPlayer;
- debug context priority, UI focus and duplicate binding issues;
- separate physical input from semantic intent and server authority;
- design simple remapping/conflict policy;
- integrate input with Pawn/components, UI and GAS without hard coupling.

Specialist depth begins with custom modifiers/triggers, platform-specific remapping/glyph systems, CommonUI input routing internals, split-screen edge cases, accessibility input requirements, replay/rollback input logs and full input automation.

## Hands-on verification task

Extend Project 1 and Project 9:

1. Create `IA_Move`, `IA_Look`, `IA_Jump`, `IA_Interact`, `IA_Fire`, `IA_AbilityPrimary`, `IA_Confirm`, `IA_Cancel`.
2. Create `IMC_Common`, `IMC_OnFoot`, `IMC_Menu`, `IMC_Vehicle` and one temporary high-priority `IMC_Radial`.
3. Add a LocalPlayer input coordinator that logs context add/remove, priorities and owning requester.
4. Bind on-foot actions to Character/components, not directly to UI.
5. Add one remappable key screen with conflict detection and per-local-player persistence.
6. Add a CommonUI or equivalent modal and prove gameplay actions do not leak through while UI confirm/cancel works.
7. Add a GAS/ability adapter or mock ability adapter with press/release semantics and idempotent rebuild after respawn.
8. Test keyboard/mouse, gamepad and two local players if possible.
9. Inject failures: duplicate binding, wrong trigger event, leaked context, value-type mismatch and UI focus intercept.
10. Produce an input routing diagram plus logs showing correct routes in gameplay, menu, vehicle and respawn.

## Source caveats

- `SRC-INPUT-001` is the official UE5.6 overview and contains the main conceptual contract, but exact class signatures and project settings should be verified in the target branch.
- `SRC-INPUT-002` through `SRC-INPUT-005` are API anchors. Treat maintained API navigation as branch-sensitive.
- `SRC-UI-011` notes CommonUI/Enhanced Input integration as version-sensitive; keep UI and gameplay input ownership explicit.
- Network and GAS integration guidance here is architectural synthesis over official input, networking and GAS sources; it is not a hidden engine subsystem that makes local input authoritative.
