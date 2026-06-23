# Unreal UI Systems for Interviews

Unreal UI is not merely a collection of Widget Blueprints. It is a local presentation system built over Slate, fed by gameplay state, driven by a particular local user, and constrained by CPU traversal, layout, paint, draw submission, texture memory and overdraw. Strong engineers define those boundaries before choosing a widget or optimisation switch.

Version target: **UE5.3–UE5.6**. UMG and Slate fundamentals are stable. CommonUI and Model-View-ViewModel (MVVM) are **plugin-dependent**; MVVM is documented as **Beta**, and CommonUI/Enhanced Input integration is version-sensitive. Maintained UE5.7 optimisation pages are used for concepts, not as unqualified UE5.3–UE5.6 API guarantees.

## 1. The end-to-end model

```text
authoritative gameplay state
  server / replicated owner state / local settings
                ↓ events or field notifications
local presentation model / view-model / presenter
  formatting-ready snapshot, pending state, audience rules
                ↓ explicit render/update
UUserWidget tree (UObject-facing UMG)
                ↓ wraps/rebuilds
SWidget tree (Slate layout, hit testing and paint)
                ↓ draw elements / batches
renderer, textures, materials and GPU composition
```

The arrows are contracts, not permission to duplicate ownership. Gameplay remains authoritative; the UI projects that state and emits intents. A health bar may interpolate a displayed value, but it must not become the source of health. A purchase button requests a transaction; hiding it locally is not server validation.

## 2. UMG, Slate, CommonUI and MVVM

### UMG and `UUserWidget`

UMG is the designer-friendly UObject/Blueprint layer. It supplies Widget Blueprints, animation, named child widgets and familiar engine lifetime/reference integration. Most game HUDs and menus should begin here.

`UWidget` wraps a Slate control; `UUserWidget` composes a reusable widget tree. The wrapper distinction matters: the UObject instance and its underlying Slate representation have related but non-identical lifetimes.

### Slate and `SWidget`

Slate is Unreal's lower-level cross-platform C++ UI framework. Its widgets are generally held through `TSharedRef<SWidget>` or `TSharedPtr<SWidget>`, configured with arguments/attributes and delegates, and arranged through slots. Panels have dynamic child slots, compound widgets have a fixed composition, and leaves draw content. Slate computes desired size bottom-up and arranges children top-down. [SRC-UI-001]

Choose Slate when building editor UI, low-level reusable controls, or a surface whose dynamic structure/performance/control justifies C++ ownership. Do not rewrite an ordinary game menu in Slate merely to sound advanced.

### CommonUI

CommonUI is a **plugin-dependent** layer for cross-platform, multilayer menus. It adds activatable-widget stacks, action routing, input glyphs, styles and controller-friendly navigation. Its useful architectural idea is a tree/stack of activatable screens: the leafmost active screen owns the current interaction while lower layers can remain present. [SRC-UI-009] [SRC-UI-010]

Activation is not construction. A `UCommonActivatableWidget` can activate and deactivate repeatedly while the UObject and Slate widget remain alive. Put “screen is now interactive” work in activation hooks and undo it in deactivation hooks; reserve construction/destruction for representation lifetime.

### MVVM

Unreal's Model-View-ViewModel plugin provides FieldNotify-driven bindings and view-model creation modes. Epic documents it as **Beta**. Push notification can avoid repeatedly evaluating raw property bindings, but MVVM does not remove the need for ownership, initialisation, authority, formatting and teardown design. [SRC-UI-012]

Use it when a project accepts the plugin/version risk and benefits from reusable declarative bindings. A small presenter with explicit events is often simpler. Never add a view-model that merely mirrors every gameplay property without defining a presentation boundary.

## 3. `UUserWidget` lifecycle: the interview trap

Think in three lifetimes:

1. **UObject instance:** created, initialised, retained by references, eventually garbage-collected.
2. **Slate representation:** built, attached, detached or rebuilt.
3. **screen/activity state:** visible/interactive, suspended, hidden or reactivated.

`OnInitialized` is the once-per-instance initialisation hook for ordinary runtime instances. `Construct` runs after the underlying Slate widget is constructed and may run multiple times when the widget is removed and added to a hierarchy. Epic explicitly recommends `OnInitialized` when a true once-on-creation event is required. [SRC-UI-002]

`PreConstruct` supports design-time/runtime preview. Treat it as presentation setup: it can execute in the editor, so gameplay lookups, network calls and destructive side effects are unsafe assumptions. `Destruct` accompanies Slate teardown, not necessarily immediate UObject collection; a surviving instance may later rebuild.

### Practical hook policy

| Concern | Typical home | Rule |
|---|---|---|
| cache stable child references; bind stable self events | initialisation | do once when dependency lifetime truly matches instance |
| preview labels/colours | pre-construct | tolerate design time and missing World/gameplay state |
| subscribe to current presented model | construct/explicit `BindModel` | make idempotent; bind only after valid dependency exists |
| set current focus/input context | screen activation | repeat on every activation |
| unsubscribe model, cancel timers/requests | deactivation/destruct/explicit `UnbindModel` | symmetric with the hook that acquired them |
| release long-lived service ownership | explicit owner shutdown / UObject lifetime | do not rely solely on viewport removal |

An idempotent binding can remove the previous delegate handle before adding a new one, or track exactly which source is bound. `AddUnique` helps only for compatible delegate types and does not repair binding to the wrong old source.

### Ownership and garbage collection

Creating a widget does not guarantee the desired product lifetime. Keep a reflected strong reference from the owning HUD/controller/subsystem while it should survive. Avoid accidental global roots and service-held strong references that keep whole screens alive after travel.

For callbacks crossing lifetime boundaries:

- use delegate handles or owner-aware dynamic delegates and remove them symmetrically;
- use weak UObject captures where the callback can outlive the widget;
- cancel timers, latent work and async handles when the screen no longer owns the result;
- validate the local player, World and presented object again when a callback fires;
- reject stale results with a request generation or identity token.

## 4. Model–presentation contracts

### Prefer snapshot plus deltas

A robust screen binding sequence is:

1. acquire the correct local-player-facing source;
2. subscribe to future changes;
3. read/render a complete initial snapshot;
4. apply subsequent deltas on the game thread;
5. unsubscribe when the binding lifetime ends.

Subscribe-before-snapshot avoids a gap, but can produce a delta during the snapshot. Solve that with a revision, serialised game-thread order, or a final refresh. The exact choice depends on concurrency and cost.

### Presentation model responsibilities

A useful view-model/presenter may own:

- display-ready `FText`, icons and sorting/grouping;
- permissions such as “can afford” or “can equip”, clearly derived from gameplay policy;
- pending/confirmed/rejected transaction state;
- selection, expansion and scroll restoration where these are genuinely UI state;
- request identity and async asset state;
- subscriptions translating coarse gameplay changes into bounded UI updates.

It should not own authoritative inventory, health, cooldown or quest completion.

### Event-driven versus binding/tick

UMG property bindings and Slate attributes are pull mechanisms. They can be convenient, but a large visible hierarchy may evaluate them frequently, and formatting/allocation inside a binding magnifies cost. A per-widget `Tick` that polls gameplay has the same scaling smell.

Prefer event/FieldNotify-driven updates for values that change discretely. Set the widget property when the model changes, then let invalidation do its job. Tick only for genuine continuous presentation—an animation, cursor tracking or interpolation—and disable it when inactive. Measure before rewriting small static screens.

## 5. Layout is a constraint system

Slate layout conceptually has two passes: children compute desired size bottom-up; parents arrange allotted geometry top-down. [SRC-UI-001] The child asks; the parent decides.

### Flow panels versus Canvas

Use horizontal/vertical/wrap/grid/overlay panels where content should reflow. Use Canvas for deliberate absolute layering or anchored HUD composition. A Canvas-heavy menu with fixed offsets tends to break under long localisation strings, font changes, aspect ratios and accessibility scaling.

Extra `SizeBox`/`ScaleBox`/Canvas layers are not free fixes. Every wrapper adds hierarchy and may create conflicting constraints. Debug the first parent whose allotted geometry differs from intent.

### Anchors

Canvas anchors define a minimum/maximum normalised region; offsets then act relative to that region. A point anchor positions from a point; split anchors can stretch with the parent. Anchors adapt placement, not content semantics. [SRC-UI-008]

### DPI scaling

UMG applies a project DPI curve according to a selected viewport dimension rule, such as shortest side. Design against multiple resolutions/aspect ratios and the project curve, not one editor preview. [SRC-UI-006]

### Safe zones

Safe Zones account for platform display areas, TV title-safe regions and mobile cut-outs. Test through Device Profiles/device emulation and on target hardware. Keep critical controls/text inside safe bounds; backgrounds may deliberately bleed beyond them. [SRC-UI-007]

### Localisation pressure

Use `FText` for user-facing localisable text. Converting it to `FString` and back can discard text history needed for culture-aware reconstruction. Use culture-aware number, date, plural and gender formatting, string tables where they improve reuse, and run the gather/export/import/compile pipeline in packaged tests. [SRC-UI-016]

Test pseudo-localisation, long strings, missing glyphs, line wrapping, truncation, right-to-left layout where required, and font fallback. “Fits English at 1080p” is not acceptance evidence.

## 6. Visibility, hit testing and focus

Visibility combines drawing, layout participation and hit-test behaviour. A transparent widget can still intercept input; a decorative parent can block descendants depending on hit-test visibility; `Hidden` and `Collapsed` differ in layout participation. When a button cannot be clicked, inspect the hit-test path rather than only its `IsEnabled` value.

### Focus is per user

Keyboard/controller focus belongs to a Slate user/local player, not globally to “the UI”. `SetUserFocus` targets a specific player controller; split-screen makes accidental global assumptions obvious. [SRC-UI-017]

Every modal/screen should define:

- initial/default focus;
- directional navigation and wrap/stop/explicit overrides;
- focus restoration after a popup closes;
- behaviour when the focused entry disappears or becomes disabled;
- mouse-to-gamepad and gamepad-to-mouse transitions;
- escape/back handling and a guaranteed exit path.

### CommonUI input configuration

CommonUI routes input through the active widget tree. Activatable widgets can supply desired input mode, mouse-capture behaviour and desired focus target; Epic strongly recommends implementing the desired focus target. Deactivation restores the previous configuration, but deactivating every screen can leave the last configuration, so define a safe root/default. [SRC-UI-010]

Do not mix competing ownership casually: direct `PlayerController` input-mode/cursor calls, viewport calls, Enhanced Input mapping changes and CommonUI activation can fight. Choose one coordinator and log transitions.

CommonUI's Enhanced Input support is explicitly **version-sensitive** and has been described as limited in maintained documentation. Confirm target branch/settings before standardising the architecture. [SRC-UI-011]

### Touch and accessibility

Touch needs adequate target size, gesture conflict policy, safe zones and no hover-only affordances. Accessibility also includes readable contrast, scalable text, redundant non-colour cues, remappable input and meaningful accessible labels/behaviours. Unreal exposes accessibility interfaces/events, but platform integration and project support must be tested on targets. [SRC-UI-018]

## 7. Virtualised lists and entry recycling

`ListView` virtualises entries: items are the data; entry widgets are temporary visualisations for the currently generated rows. Entry widgets can be released and reused as the list scrolls. [SRC-UI-013] [SRC-UI-014]

Therefore:

- store durable state in the item/model, not only in the entry widget;
- treat every item assignment as a full rebind;
- clear old delegates, brushes, selection/hover state and async requests;
- reset placeholders before beginning a new icon request;
- on release, unsubscribe and invalidate the entry generation;
- do not retain an entry pointer as the identity of an item;
- use stable item identity when restoring selection/scroll state.

Classic bug: entry A starts loading icon A, is recycled for item B, then callback A paints icon A into row B. Capture a weak entry plus item ID/request generation, and accept only if both still match.

For large datasets, also avoid rebuilding/sorting/filtering the complete list on each small change. Batch model changes, preserve stable identities and measure item count, generated entry count, regeneration rate and per-update work.

## 8. Asynchronous icons and data

Hard references from a common HUD to every item portrait can inflate the startup dependency closure. Soft references plus asynchronous loading allow placeholders and bounded residency, but introduce lifetime and stale-result problems. [SRC-ASSET-001] [SRC-ASSET-002]

A safe request path records:

- requested stable item/asset identity;
- monotonically increasing request generation;
- weak destination widget or presenter;
- cancellable/shared load handle when supported;
- ownership policy for loaded residency.

On completion, verify widget validity, current item identity, generation and screen activity. Then apply the brush. A successful load is not permission to update an unrelated recycled row.

Coalesce duplicate loads through an asset/presentation service, bound concurrent requests, prioritise visible entries, and distinguish “not loaded”, “missing”, “failed” and “cancelled” in diagnostics.

## 9. World-space widgets

`WidgetComponent` can render a widget in World space, where it behaves like scene geometry and can be occluded, or Screen space, where it is composited to the screen and remains unoccluded. Draw Size, manual redraw and redraw interval affect quality and work; continuously drawing at desired size can be expensive. [SRC-UI-015]

Use world widgets intentionally for diegetic panels/nameplates. At scale:

- create only for relevant/near/significant actors;
- reduce update/redraw rate for distant or unchanged widgets;
- avoid one continuously updating render target per actor;
- choose occlusion semantics explicitly;
- define ownership and input routing for multiplayer/local players;
- compare against projected screen-space markers or a batched custom solution.

Test resolution, material translucency, depth, motion, VR/stereo and readability. A world WidgetComponent transfers costs into both UI and scene rendering domains.

## 10. Multiplayer and audience boundaries

Widgets are local presentation objects and are not replicated. Replicate gameplay state to an appropriate gameplay owner, then let each local player build its own projection.

Examples:

- public scoreboard state may come from `GameState`/`PlayerState`;
- private inventory belongs in owner-only replicated state, not public `GameState`;
- a Pawn HUD must survive possession changes by rebinding through PlayerController/local-player ownership;
- split-screen needs one UI root, focus user and settings context per local player;
- dedicated servers should not require viewport/UI classes for gameplay correctness.

For an action button:

1. UI derives local affordance from current replicated state;
2. click emits an intent with stable item/action identity;
3. server validates authority, ownership, cost and current state;
4. UI displays bounded pending feedback;
5. replicated confirmation or explicit rejection resolves it;
6. timeout/reconnect/travel cancels or reconstructs the pending presentation.

Do not make a client-side disabled button the security boundary. Do not multicast UI instructions when replicating durable state would let late joiners and rebuilt screens reconstruct correctly.

## 11. Invalidation and rendering performance

A UI frame can spend time in model/binding logic, widget update, desired-size/layout, hit testing, paint traversal, draw-element generation/batching, render-thread submission and GPU composition. Optimise the measured stage.

Invalidation caches work until a relevant change occurs. A hierarchy/layout invalidation implies more work than paint-only invalidation. Widgets that change every frame may be marked volatile so they do not repeatedly invalidate a cached parent, but volatility deliberately forfeits paint caching for that subtree. [SRC-UI-004]

Retainer panels render a subtree to a texture and can phase/lower redraw frequency. They can reduce draw frequency/calls, but add render-target memory, update latency, compositing and potentially expensive repaint. Use after measuring; an Invalidation Box is the usual first CPU-oriented caching experiment. **The cited optimisation guidance is maintained UE5.7; verify exact UE5.3–UE5.6 behaviour/settings.** [SRC-UI-004]

### Cost multipliers

- deep/wide hierarchy and repeated layout invalidation;
- bound functions, Blueprint tick and allocation-heavy text/brush formatting;
- rebuilding whole panels rather than updating changed rows;
- dynamic materials, clipping, masks and effects that inhibit batching;
- many unique textures/materials and font-atlas churn;
- large translucent areas and multiple full-screen layers causing overdraw;
- frequent world-widget render-target redraws;
- animation continuing while hidden/covered;
- focus/navigation or hit-test work over unnecessary widgets.

### Measurement set

- visible widget and generated-list-entry counts;
- update/tick/binding calls and time;
- invalidation reason/count and layout/paint work;
- draw elements/batches and render-thread UI time;
- GPU pass time, render-target memory and overdraw;
- texture/font atlas memory and load hitches;
- screen activation/rebuild frequency;
- 1/10/100/1000-item and low-end-device scaling.

## 12. Debugging workflows

### Wrong layout

1. Reproduce at exact resolution, aspect ratio, DPI scale, culture and safe-zone profile.
2. Pick the widget in Widget Reflector.
3. Walk upward from desired size/allotted geometry to find the first wrong constraint.
4. Inspect anchors, offsets, slot fill/alignment, visibility and wrapper panels.
5. Test long/localised text and font fallback before changing fixed sizes.

### Click/focus failure

1. Use Widget Reflector's hit-testable picking/hit-test grid and focus information. [SRC-UI-003]
2. Inspect overlays and parent visibility/hit-test modes.
3. Confirm enabled/interactable state and local Slate user.
4. Log screen activation, input config, mapping contexts, cursor/capture and desired focus target.
5. Test keyboard, gamepad, mouse, touch, popup close and split-screen separately.

### Duplicate or stale updates

1. Log widget instance, source identity, bind/unbind hook and delegate handle.
2. Count `Construct`, activation and model-binding calls.
3. Remove/re-add, possess/unpossess, travel and reopen repeatedly.
4. Check service-held references, timers and async callbacks.
5. Make teardown symmetric and stale-result acceptance explicit.

### UI hitch or sustained cost

1. Classify startup/load hitch versus every-frame cost.
2. Use Slate Insights' frame view to see widgets painted, invalidated and updated. [SRC-UI-005]
3. Use Widget Reflector invalidation/update/paint debugging to identify the responsible subtree. [SRC-UI-003]
4. Correlate Game/Render/GPU traces and texture/asset loads.
5. Change one mechanism—event update, hierarchy, invalidation, virtualisation, retainer/redraw cadence—and recapture the same scenario.

## 13. What a three-year engineer should know

Expected D3–D4:

- UMG versus Slate responsibility and UObject/Slate lifetime distinction;
- `OnInitialized` versus repeatable `Construct`, plus symmetric teardown;
- gameplay truth versus local presentation projection;
- event-driven updates versus property bindings/tick;
- desired size, panel constraints, anchors, DPI and safe zones;
- visibility/hit testing, per-user focus and controller navigation;
- CommonUI activatable-screen/input concepts with plugin caveats;
- virtualised-list item versus recycled entry lifetime;
- soft/async icon loading with stale-result rejection;
- world-space widget redraw/scaling choices;
- localisation/accessibility basics;
- Widget Reflector and Slate Insights-led diagnosis;
- invalidation/layout/paint/draw/GPU cost classification.

Specialist D4–D5:

- custom Slate widgets, attributes and invalidation internals;
- custom navigation/input processors and platform accessibility bridges;
- large-screen architecture, custom batching/drawing and font shaping;
- CommonUI/MVVM source-level/version migration;
- VR/stereo/diegetic UI and renderer integration;
- automated localisation/layout/accessibility validation.

## 14. Common misconceptions

| Misconception | Better model |
|---|---|
| “Construct is UI BeginPlay.” | It follows Slate construction and can repeat; once-per-instance initialisation is separate. |
| “Remove From Parent destroys the widget.” | It detaches presentation; references may keep the UObject alive and it may rebuild. |
| “Binding is event-driven.” | Many legacy bindings/Slate attributes are pulled; know when and how often they evaluate. |
| “Hidden UI is free.” | Visibility mode and activity matter; timers, animations, bindings or retained references may still cost. |
| “Canvas makes UI responsive.” | Anchors help placement, but flow layout, DPI, safe zones and content expansion still need design. |
| “List entry equals list item.” | The item is data identity; the generated entry is recyclable presentation. |
| “CommonUI replaces Enhanced Input.” | They coordinate different concerns and integration is version-sensitive. |
| “Retainer Panel is an automatic optimisation.” | It trades repaint/draw frequency for render-target memory, latency and composition. |
| “Widgets replicate.” | Gameplay state replicates; each local player constructs presentation. |
| “A disabled purchase button prevents cheating.” | The server must validate the transaction. |

## 15. Strong interview answer patterns

### Design a durable HUD

> I keep gameplay state in authoritative gameplay objects and expose a local presentation snapshot plus change events. The PlayerController/local-player UI root owns screens and rebinds on possession/travel. Widgets subscribe idempotently, render an initial snapshot, and unsubscribe/cancel async work symmetrically. UI emits intents; the server validates them. I profile model work, invalidation/layout/paint, draw submission and GPU overdraw separately.

### Debug a controller focus soft-lock

> I reproduce with the exact local user and input device, inspect the focus path and hit-test grid in Widget Reflector, and log activation, desired focus target, input config, cursor/capture and mapping-context transitions. I test popup close, disabled/removed focused entries and restoration. With CommonUI I let one coordinator own input config instead of mixing direct controller calls with activatable-screen state.

### Optimise a large inventory

> I measure update, layout/paint, draw and asset-load cost first. I keep durable item state outside rows, use a virtualised ListView, rebind/release entries completely, and guard async icons by stable item ID and request generation. Model changes are batched rather than rebuilding the list. Then I use Slate Insights/Widget Reflector to decide whether event updates, invalidation, hierarchy reduction or redraw cadence addresses the actual bottleneck.

## 16. Hands-on verification project

Extend Project 1/9 with a Local-Player UI Lab:

1. build a HUD root plus inventory, settings, modal confirmation and world-nameplate screens;
2. define gameplay-state → presenter → widget ownership and lifetime diagrams;
3. implement initial snapshot plus event updates without widget polling;
4. repeatedly add/remove/reopen, possess/unpossess and travel while proving one active subscription;
5. add a 10/100/1000-item virtualised inventory and inject recycled-row stale icon/selection bugs;
6. asynchronously load soft icon references with placeholders, cancellation and generation checks;
7. support mouse, keyboard, gamepad and touch with deterministic initial/restored focus;
8. optionally implement the screen stack in CommonUI and document target-version/plugin assumptions;
9. test 16:9, ultrawide, narrow, DPI presets, safe zones, pseudo-localisation and long/RTL text where required;
10. add accessible labels, non-colour state cues, scalable text and controller-remappable actions;
11. compare screen-space and world-space nameplates at 1/20/100 actors with redraw/significance policies;
12. capture Widget Reflector and Slate Insights evidence, then compare binding/tick versus event-driven, invalidated versus uncached, and Retainer experiments;
13. run dedicated server plus two clients/split-screen and prove private/public audience boundaries and transaction rejection;
14. deliver lifecycle logs, focus/navigation matrix, async stale-result tests, layout screenshots and CPU/GPU/memory scaling report.

## 17. Source map

- architecture/lifecycle/layout: [SRC-UI-001] [SRC-UI-002] [SRC-UI-006] [SRC-UI-007] [SRC-UI-008]
- CommonUI/MVVM/input: [SRC-UI-009] [SRC-UI-010] [SRC-UI-011] [SRC-UI-012] [SRC-UI-017]
- virtualisation/async/world widgets: [SRC-UI-013] [SRC-UI-014] [SRC-ASSET-001] [SRC-ASSET-002] [SRC-UI-015]
- localisation/accessibility: [SRC-UI-016] [SRC-UI-018]
- debugging/performance: [SRC-UI-003] [SRC-UI-004] [SRC-UI-005]
