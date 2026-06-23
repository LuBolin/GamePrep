# Game Architecture Patterns

## Cluster 1 — Gameplay Framework, Lifecycle, and Responsibility Placement

**Priority:** P0  
**Expected depth:** D4 for gameplay/generalist roles  
**Version scope:** Stable UE4/UE5 concepts, confirmed against UE5.6 framework/API material. The detailed lifecycle sequence source is labelled **UE4-era** and used only where current APIs still corroborate it.  
**Prerequisites:** UObject/reflection/lifetime cluster in `ue_cpp_idioms.md`.

### Architecture begins with lifetime and audience

Most “which Unreal class?” questions become easier if you ask four things before naming a type:

1. Does the responsibility need a world transform or presence?
2. Whose lifetime should it share: process, game instance, world, match, player, pawn, or component owner?
3. Who must see it: server only, owning client, every client, local player only, or editor only?
4. Is this identity-bearing behaviour, reusable behaviour, or plain data?

This is more reliable than memorising class definitions. A score can fit in many classes syntactically, but “replicated state for one participant that survives pawn replacement” points to `PlayerState`; “server-only match admission rules” points to `GameMode`; “reusable health behaviour attached to several actors” points towards an Actor Component.

### Actor, Component, and plain UObject

An Actor is a world entity that can be placed or spawned, owns Components, participates naturally in world lifecycle, and is the primary gameplay replication unit. The Actor's transform comes from its root Scene Component rather than being stored directly on the Actor. Use an Actor when independent world identity, spawning/destruction, networking relevance, or level placement is central. [SRC-EPIC-011]

Components compose behaviour into Actors:

| Type | Transform? | Rendering/collision geometry? | Typical use |
|---|---:|---:|---|
| `UActorComponent` | No | No | Health, inventory, interaction capability, abstract movement/input policy |
| `USceneComponent` | Yes | Not necessarily | Attachment pivots, cameras, spring arms, audio positions |
| `UPrimitiveComponent` | Yes | Yes | Meshes, collision shapes, visible or collidable primitives |

Scene Components form an attachment tree; the root Scene Component supplies the Actor transform. Registration connects a Component to the world/scene and enables relevant update, render, and physics state. Constructor-created default Components register automatically during Actor spawning; runtime-created Components require the correct ownership and registration sequence. Registration can be non-trivial work, so dynamic component churn should be measured. [SRC-EPIC-012]

A plain UObject is appropriate for identity-bearing engine-integrated logic that does not need world presence or Actor replication/lifecycle: for example, an instanced policy object, dialogue node runtime, or editor model. A reflected `USTRUCT` is often better for copyable data. An ordinary C++ type remains best for deterministic RAII helpers or hot/value-oriented data that needs no reflection.

**Composition is not automatically good architecture.** A `UHealthComponent` is valuable when it defines a coherent capability with an owner contract. It becomes a dumping ground when it reaches into arbitrary sibling components, UI, GameMode, save files, and networking through casts. Give components narrow inputs/outputs, define authority for mutation, and communicate through explicit interfaces/delegates where decoupling is real rather than ceremonial.

### Pawn, Character, and Controller

A Pawn is a possessable Actor representing an agent in the world. A Character is a specialised Pawn with a capsule, skeletal mesh conventions, `CharacterMovementComponent`, humanoid-style movement, animation integration, and mature network movement support. Choose Character when those defaults fit; choose Pawn for vehicles, board pieces, cameras, creatures, or custom locomotion where Character assumptions fight the design. [SRC-EPIC-010]

A Controller represents decision-making or “will” and can possess a Pawn. `PlayerController` translates human/local-player intent and is a primary client–server interaction point. `AIController` drives non-human agents. Separating Controller from Pawn allows respawn, spectating, vehicle changes, or AI takeover without pretending the physical avatar and decision source are the same lifetime.

Do not place every input-derived behaviour in PlayerController. Input collection and player-level orchestration may belong there, but movement capabilities and avatar actions commonly live on Pawn/Character/components so AI, replay, prediction, and alternate controllers can reuse them. The clean boundary is project-specific; the interview signal is that you discuss possession, lifetime, authority, and reuse.

### GameMode, GameState, PlayerController, and PlayerState

| Class | Exists where in multiplayer? | Lifetime/identity | Good homes | Poor homes |
|---|---|---|---|---|
| `GameMode` | Server only | Rules for the current level/match | Login/spawn policy, match transitions, win rules | Data clients must directly read |
| `GameState` | Server and replicated to clients | Shared match/world state | Match phase, team scores, server-synchronised time, player list | Private per-player data; server-only policy |
| `PlayerController` | Server for each player; on a client, its own controller | Connection/local-player control viewpoint | Input orchestration, owning-client RPC/UI coordination, possession | Public state every client must see |
| `PlayerState` | Server and replicated to all clients | Participant identity across pawn changes | Name, score, team, public match stats | Pawn-local transient movement state |

GameMode defines authoritative rules and only exists on the server. GameState carries changing match information relevant to connected clients. PlayerState exists for each participant and is replicated broadly, unlike PlayerController visibility. [SRC-EPIC-010] [SRC-EPIC-013] [SRC-EPIC-014]

The table is a starting point, not permission to expose secrets. “Replicated to all clients” means public participant state belongs naturally in PlayerState; private inventory contents may require owner-only replication elsewhere. Similarly, single-player projects benefit from the same responsibility boundaries even when networking is absent.

### GameInstance, World, and Subsystems

`GameInstance` lives across map loads for the process instance and is not replicated; server and clients have independent instances. It can coordinate session-level data, but turning it into a global bucket creates hidden dependencies and makes multi-world/editor testing painful. [SRC-EPIC-018]

Subsystems are auto-instanced UObjects whose lifetime follows a host:

| Subsystem | Lifetime fit | Example responsibility |
|---|---|---|
| `UEngineSubsystem` | Engine/process | Process-wide service genuinely independent of a game instance |
| `UGameInstanceSubsystem` | Game instance, across world travel | Session orchestration, local profile/service facade |
| `UWorldSubsystem` | One world | World registry, spatial/gameplay service scoped to that world |
| `ULocalPlayerSubsystem` | One local player | Local input/UI/preferences, including split-screen separation |
| `UEditorSubsystem` | Editor | Editor-only workflow/tool service |

Choose the narrowest host lifetime that matches the responsibility. A subsystem removes manual discovery and ownership boilerplate; it does not justify global state, replace replicated Actors, or automatically solve dependencies. Explicitly declare subsystem dependencies where ordering matters, clean up delegates/tasks in `Deinitialize`, and account for multiple PIE worlds and local players. [SRC-EPIC-017]

### Lifecycle is a contract, not one event

Actors can be loaded from a level, duplicated for PIE, spawned immediately, or spawned deferred. Those paths differ before converging on component initialisation and play. The UE4-era lifecycle overview remains a useful map: loaded actors receive `PostLoad`, spawned actors receive `PostActorCreated`, construction runs, Components initialise, and then play begins. Current component API confirms that `BeginPlay` requires registration and initialisation and is normally called after `PostInitializeComponents`, though networking or child-actor conditions can delay it. [SRC-EPIC-015] [SRC-EPIC-016]

Practical placement:

- **Constructor:** defaults and `CreateDefaultSubobject`; no assumption of a gameplay world.
- **OnConstruction / Construction Script:** derive editor/runtime instance setup from exposed properties; expect reruns in editor workflows and avoid irreversible external side effects.
- **OnRegister:** component has an owning context and is entering registration; pair side effects with unregistration and expect reruns.
- **PostInitializeComponents:** Actor components are initialised; useful for instance wiring that must precede play.
- **BeginPlay:** gameplay begins for this world/Actor; bind gameplay events, start timers, and access runtime services when their contracts permit.
- **EndPlay:** stop timers/tasks, unbind delegates, release gameplay relationships for destruction, travel, streaming unload, or PIE end. Do not reserve cleanup only for explicit `Destroy`.
- **BeginDestroy/FinishDestroy:** low-level UObject resource teardown, not ordinary gameplay cleanup.

The crucial debugging question is not “did BeginPlay run?” but “which creation path am I on, which world is this, is the object registered/initialised, and can this hook run more than once?”

### Ownership, attachment, possession, and references are different relations

Unreal uses overlapping words that should not be collapsed:

- **GC reference:** whether an object is retained.
- **Outer:** containment/identity context.
- **Actor Owner:** gameplay/network ownership relationship used by several systems.
- **Attachment:** spatial parent–child relation between Scene Components (and Actors via roots).
- **Possession:** Controller directs a Pawn.
- **Replication authority:** server is authoritative for replicated gameplay state.

Attaching a weapon to a Character does not by itself define who may call its Server RPC, who retains every associated UObject, or where inventory state should live. In an interview, naming these relations separately is often more valuable than quoting APIs.

### Tick, timers, events, and explicit updates

Tick is appropriate for work that genuinely samples or integrates every frame, such as movement or camera smoothing. It is not inherently evil, but thousands of enabled tick functions, per-frame searches, allocations, casts, or Blueprint work form a budget problem. Components do not tick by default and must opt in. [SRC-EPIC-012]

Prefer events for state transitions, timers for coarse scheduled work, and explicit manager/batched updates when central scheduling or cache locality matters. Do not replace Tick with a timer at the same frequency and call it optimisation; profile invocation count, work per invocation, and latency requirements. Disable Tick when idle, choose sensible tick intervals only when semantics allow, and verify the limiting thread with Insights/stat tools.

### Common architecture failures

1. **God Character:** inventory, quests, UI, saves, combat, and networking all live on one subclass.
2. **Global GameInstance bucket:** world-specific objects survive travel accidentally and tests depend on hidden mutable state.
3. **GameMode read by clients:** works in standalone, becomes null on network clients.
4. **Public state in PlayerController:** remote clients cannot see other players' controllers.
5. **Pawn identity in Pawn:** score/team disappears or becomes awkward when respawning.
6. **Component spaghetti:** reusable components discover each other through casts and create circular startup order.
7. **Constructor gameplay:** CDO/editor instances trigger side effects or world access fails.
8. **Cleanup only in Destroy:** travel, streaming unload, and PIE leave delegates/tasks alive.
9. **Dynamic components without registration:** object exists but does not participate correctly in world/render/physics lifecycle.
10. **Tick substitution theatre:** high-frequency timers perform the same work with less clarity.

### Debugging workflow: wrong class or wrong lifecycle

1. Log world name/type, net mode/role where relevant, object path, owner, pawn/controller/player state, and lifecycle hook.
2. Reproduce in standalone, PIE with multiple clients, and after travel/respawn—the failing boundary usually reveals the misplaced lifetime.
3. Draw the state owner and audiences. Mark server-only, owning-client, and public replicated data.
4. Verify creation path: loaded, PIE duplicate, immediate spawn, deferred spawn, or runtime component creation.
5. Verify component owner, registration, activation, and tick state independently.
6. Check delegate/timer/task symmetry between setup and EndPlay/Deinitialize.
7. Move data only after identifying the lifetime/audience mismatch; do not “fix” nulls with global lookups.

### Strong interview answer pattern

For “where should this live?” answer in this order:

1. Define lifetime and world presence.
2. Define authority and replication audience.
3. Define whether identity or reusable capability matters.
4. Choose the framework class and name the nearest alternative.
5. Explain creation/cleanup hooks and one failure mode.

Example: “Public team score is authoritative on the server but must be visible to every client for the current match. GameState matches that lifetime and audience, with server mutation and replication to clients. GameMode should enforce the scoring rules but cannot be the client-readable store because it is server-only. PlayerState would fit per-participant contribution, not the aggregate team score.” [SRC-EPIC-013]

### What a three-year engineer should know

Be able to draw the framework from memory, place common state by lifetime/audience, explain possession and respawn, distinguish the three component tiers, reason through normal lifecycle hooks, and debug a standalone-only design that fails with two clients. Specialist depth includes seamless travel internals, exact replication ownership paths, component instance data/rerun construction, world-context edge cases, and engine-source lifecycle ordering.

### Conflict and uncertainty notes

- `SRC-EPIC-015` is explicitly UE4.27-era. Its broad load/spawn/construction/initialisation/EndPlay model remains useful, but exact UE5.3–UE5.6 ordering must be checked against current API and engine source when correctness depends on a hook boundary.
- High-level framework examples sometimes place health/ammo/inventory in PlayerState. Treat these as audience/lifetime examples, not a universal rule; public participant state, private owner-only state, and pawn-local transient state have different homes. [SRC-EPIC-018]

## Part II: Gameplay System Design

The framework gives systems a home; design work defines their contracts. Interviewers are usually testing whether you can discover requirements, separate durable truth from presentation, preserve authority, and explain trade-offs—not whether your class names match their project.

## 1. The system-design canvas

Before drawing classes, ask:

| Dimension | Questions |
|---|---|
| Player experience | What action and feedback must feel immediate? What failure must be explained? |
| Domain invariants | What must always be true? Are quantities bounded? Can states overlap? |
| Ownership/lifetime | Match, participant, pawn, item instance, world object, session, or account? What survives respawn/travel? |
| Authority/audience | Who may mutate? Owner only, all clients, team, server only? What can be predicted? |
| Definition/state/presentation | Which values are authored/shared, mutable per instance, and local visual projections? |
| Identity | Stable ID, asset ID, gameplay tag, Actor network identity, GUID, slot key? |
| Operations | Commands/queries/events; synchronous, queued, async, replicated? |
| Persistence/versioning | What is saved? What happens after content/schema changes? |
| Loading | Which references are hard/soft? Can the system operate while presentation assets load? |
| Failure | Missing asset, stale target, capacity conflict, duplicate request, disconnect, travel? |
| Scale/performance | Maximum instances, update/message frequency, replication cost, lookup and memory shape? |
| Tooling/testing | How do designers author/validate it? What can run without a World? How is data flow traced? |

State assumptions out loud. “Inventory” could mean eight fixed combat slots or a persistent MMO warehouse; architecture follows the constraints.

### The five-contract summary

A strong design can usually be summarised as:

1. **Definition contract:** immutable authored content and stable identifier.
2. **State contract:** authoritative mutable runtime fields and invariants.
3. **Operation contract:** validated commands and read-only queries.
4. **Notification contract:** local events/replication and their ordering/version semantics.
5. **Persistence contract:** serialised IDs/state, schema version and migration/fallback.

## 2. Health and damage

### Requirements baseline

- reusable across players, AI, destructibles;
- server-authoritative in multiplayer;
- supports damage, healing, invulnerability and death once;
- presentation receives reason/source but cannot mutate truth;
- save/restore only where the owner lifetime requires it.

### Model

`UHealthComponent` owns `CurrentHealth`, `MaxHealth`, alive/dead transition and authoritative mutation. A damage request is structured data: amount/type/tags, instigator/causer identifiers, hit/context, prediction/request ID where needed. A pipeline can apply validation, immunity, armour/resistance, team rules and clamping before committing one result.

Emit a committed result containing old/new health, applied amount, reason/source and whether death occurred. Replicate durable state; use local delegates/OnRep for UI, animation and audio. A multicast cosmetic may supplement state but must not be the only record of death.

### Invariants

- `0 <= CurrentHealth <= MaxHealth` unless an explicitly documented over-health model exists;
- death transition fires at most once per life/generation;
- client requests never supply trusted final damage;
- modifiers have a deterministic documented order;
- resurrection/reset creates a new valid lifecycle and clears owned timers/effects.

### Common design failures

- every damage source edits Health directly;
- UI binds to Tick and calculates death itself;
- armour, quests, audio and save logic accumulate in the Component;
- damage events arrive before committed state and listeners observe old values;
- death exists only as an unreliable multicast;
- healing implemented as negative damage without clarifying rule interactions.

## 3. Inventory and equipment

### Separate definition, stack/instance and container

An item definition Data Asset/Flyweight can contain stable ID, tags, display data, stack limit, weight, ability/equipment references and soft presentation assets. [SRC-ARCH-009]

Runtime representations depend on requirements:

- **stack value:** definition ID + quantity for fungible items;
- **item instance:** stable instance ID + definition ID + durability/rolls/attachments for unique mutable items;
- **container entry:** slot/grid location and instance/stack reference;
- **equipment state:** which inventory instance is equipped in which slot, plus spawned visual/gameplay representation if needed.

Do not spawn an Actor for every backpack item. Actors represent world presence; inventory records usually do not need transforms, world registration, Tick, or Actor replication.

### Operation boundary

Expose intentions such as:

- `TryAdd(definition, quantity, source)`;
- `TryMove(instance/stack, from, to, amount)`;
- `TrySplit`, `TryMerge`, `TryRemove`;
- `TryEquip(instance, slot)`;
- read-only queries/views.

The authoritative container validates capacity, stack compatibility, slot rules, locks/trades, ownership and current version. Return structured success/failure and the committed delta. Avoid returning a mutable `TArray&` that lets callers bypass invariants.

### Replication

Replicate compact owner-relevant entries/deltas, not UI assets or redundant derived totals. At small scale, a replicated array/struct can be adequate. At larger/churn-heavy scale, Fast Array-style delta replication may matter as a specialist follow-on. Public equipped appearance can be a separate broadly visible projection; private backpack contents should not automatically live in broadly replicated PlayerState.

Client drag/drop may predict visual intent, but server response resolves conflicts. Use operation IDs/container revisions to discard stale replies and rebuild the view from authoritative state.

### Save/load

Persist stable definition/instance IDs and mutable fields, not live Actor/UObject pointers. Include schema/content version. On load, resolve definitions, validate quantities/slots, migrate renamed IDs, and choose explicit missing-content policy: reject, substitute placeholder, refund, or preserve opaque record.

### Debugging

Log operation ID, player/container, before revision, request, validation result, committed delta and after revision. For duplication/loss bugs, reconstruct the transaction sequence; screenshots of the UI are insufficient.

## 4. Interaction system

### Separate discovery, eligibility, execution and presentation

1. **Discovery:** trace/overlap/query creates candidates.
2. **Selection:** local policy chooses focus by distance, angle, priority and occlusion.
3. **Query:** candidate exposes prompt/available actions through a narrow interface or interaction component.
4. **Request:** owning Pawn/Controller sends stable target/action and context to authority.
5. **Validation/execution:** server rechecks target existence, range, line of sight, ownership, cooldown, required item/state and rate.
6. **Feedback:** committed state replicates; targeted rejection/cosmetics inform the player.

An `IInteractable` interface is good for heterogeneous world types when the caller needs a small stable capability. Do not put all implementation in one interface method with side effects during prompt querying. Queries should be safe/read-only; execution is a separate validated operation.

### Hold interactions

Define who owns timing, cancellation reasons, tolerance, prediction and progress correction. The client can show immediate predicted progress; the server remains authoritative for completion. Cancellation must clean timers, input state, delegates and UI on target destruction, movement, damage, possession change and EndPlay.

## 5. Weapon and ability system

### Decide scope before choosing GAS

For a few weapons/abilities, a Component + Data Assets + explicit state/strategies can be clear. Gameplay Ability System (GAS) becomes attractive when many networked abilities need tags, costs, cooldowns, attributes, effects, prediction and scalable content composition. GAS is not a requirement for calling a function “ability”, and its dedicated cluster remains to be written.

### Weapon model

- definition: ID, tags, base stats, fire/targeting policy, presentation refs;
- runtime instance: ammo/heat/durability/attachments/random seed;
- owner capability: equip/unequip, current weapon, input/command routing;
- state machine: idle/equipping/firing/reloading/cooling with cancellation;
- authority: server validates fire cadence, ammo, target/origin tolerance and state;
- prediction: local responsiveness for suitable actions, reconciled to server result;
- presentation: animation/effects/audio consume state/results rather than own rules.

Do not let animation notify timing be the sole authoritative damage rule. Notifies are useful synchronisation/presentation signals; authority still needs robust state and validation.

### Modifier order

Define whether modifiers are additive, multiplicative, override, clamped and ordered by layer/tags. Store authored modifiers as data; compile/cache aggregate runtime stats and mark dirty when equipment/effects change. Floating-point and random determinism matter if prediction/replay relies on reproducing results.

## 6. Quest and objective system

### Definitions and instances

Quest definitions describe objective graph, conditions, text references, rewards and version. A quest instance stores stable quest ID, objective progress/state, activation/completion time, branch decisions and schema version.

Objective evaluators subscribe to **domain events** such as item committed, enemy defeated with tags, area entered, dialogue choice committed—not UI clicks or concrete Actor classes. They initialise from current durable state where necessary because events before subscription may be missed.

### Event versus query

Events are efficient for incremental progress, but durable state answers “what is true now”. A quest “own 10 ore” should query inventory on activation/load and then observe committed inventory deltas. Replaying every historical item event is unnecessary unless event sourcing is an explicit product architecture.

### Multiplayer/audience

Clarify personal, party, team and world quests. Server owns progression/rewards. Replicate only relevant progress; local UI uses definitions for text. Party membership changes require an explicit policy: snapshot contributors, dynamic sharing, retroactive credit, or no transfer.

### Content evolution

Renaming objective IDs can corrupt saves. Use stable IDs, definition versions, migration tables/functions and validation for duplicate/missing/cyclic objective links. Avoid serialising array indices as identity if designers can reorder objectives.

## 7. Save/load architecture

Save data is a versioned snapshot/record, not a memory dump.

### Save boundary

- stable identifiers instead of pointers;
- only authoritative gameplay state, not transient caches/widgets/effects;
- transform/world identity with explicit level/streaming semantics;
- schema version and per-system migration;
- content version or missing-definition policy;
- deterministic ordering/checksums where useful for testing;
- async operation and failure/atomic-replace policy;
- platform/user ownership and security requirements.

### Restore phases

1. read and validate file/header/version;
2. migrate serialised records;
3. ensure required world/content is available;
4. create/resolve owning objects;
5. apply core state without firing normal gameplay rewards repeatedly;
6. rebuild derived caches and relationships;
7. emit one restore/ready notification for presentation;
8. verify invariants and record recoverable omissions.

Applying setters naively during load can trigger achievements, damage, quests or replication side effects. Provide an explicit restore transaction/path with controlled notifications.

### Multiplayer

Usually the authority loads authoritative state and replicates projections. A client's local settings/profile are a separate trust/lifetime domain. Never accept a client save file as authoritative economy/combat state without a designed trust model.

## 8. UI as a projection, not an owner

Widgets should consume read-only state/view models and send intentions to controllers/components/services. They should not own authoritative inventory, health, quest or cooldown truth.

Event-driven updates avoid per-frame binding cost, but UI must handle:

- initial snapshot before subscribing;
- missed/reordered async responses using revisions;
- widget destruction/recreation and delegate cleanup;
- localised/display assets loading after model data;
- list virtualisation for scale;
- prediction pending/rejected/confirmed states;
- split-screen LocalPlayer ownership.

A view model can aggregate/format domain state for UI, preventing gameplay systems from depending on UMG classes.

## 9. Testability and seams

Extract deterministic operations where they pay:

- pure/value inventory transaction over an input snapshot;
- damage modifier evaluation;
- quest condition evaluation;
- stat aggregation;
- target scoring/selection;
- save migration.

Then test edge tables: zero/negative/max values, duplicates, stale revisions, missing IDs, transition cancellation, authority rejection and migration from every supported schema. Integration tests cover UObject lifecycle, replication, assets and world behaviour.

Do not abstract the entire engine behind interfaces. Put seams at volatile providers and domain rules; keep straightforward Unreal integration straightforward.

## 10. Refactoring signals under production pressure

| Signal | Likely pressure | Candidate response |
|---|---|---|
| Same condition switch changed for each new content type | variation axis stabilised | data definition, Strategy or Type Object |
| Character grows unrelated systems | independent lifetimes/change | coherent Components/services |
| Components cast to many siblings | hidden dependency graph | owner orchestration, explicit interface/contracts |
| Every listener polls after a broad event | weak payload | committed delta/snapshot with version |
| Event storms recompute same aggregate | temporal batching | Dirty Flag/end-of-frame coalescing |
| UI and server disagree after rejection | missing transaction identity | operation IDs/revisions and authoritative rebuild |
| Save breaks when content moves | unstable identity | asset/stable IDs and migrations |
| One subsystem knows every Actor/widget | service-locator/god manager | bounded domain ownership and narrow composition boundary |
| Thousands of similar Actors/Components | representation mismatch | manager/batch/ECS/Mass/data-oriented simulation |
| Pool “fix” grows memory/stale state | wrong optimisation | profile churn; simplify/batch or enforce bounded reset |

## 11. Interview answer: design an inventory system

> I would first clarify whether items are fungible stacks or unique instances, slot/grid/weight rules, persistence, multiplayer audience, trading and maximum size. Definitions would be immutable Data Assets identified by stable asset/item ID; runtime entries store quantity or unique mutable state. An authoritative inventory component/service exposes validated transaction commands and read-only queries, not its mutable array. Each commit increments a revision and emits a compact delta. Private contents replicate owner-only; public equipped appearance is a separate projection. UI predicts only presentation and reconciles by operation ID/revision. Saves contain definition/instance IDs plus mutable state and schema version, with migration/missing-content policy. I would start with a simple array/map and only adopt delta replication or spatial indexing when measured scale requires it.

## 12. Design exercises

For Project 9, produce one-page canvases for health/damage, inventory/equipment, interaction, weapon/ability, quest/objective and save/load. Each must include:

- assumptions and explicit non-goals;
- definition/state/presentation schemas;
- lifetime/authority/audience map;
- command/query/event API;
- failure/rejection matrix;
- replication and persistence plan;
- loading/reference policy;
- debug log/trace fields;
- performance estimate and escalation threshold;
- three unit tests and three integration/network tests;
- one deliberately simpler rejected/accepted alternative.

## 13. Additional source map

- Components/framework/lifetime: [SRC-EPIC-010] [SRC-EPIC-012] [SRC-EPIC-013] [SRC-ARCH-014]
- Data definitions, identity/loading and validation: [SRC-ARCH-009] [SRC-ARCH-010] [SRC-ARCH-011] [SRC-ARCH-012]
- Observer/Command/State/queues/services/pooling: [SRC-ARCH-003]–[SRC-ARCH-008]
- Replication authority and durable state: [SRC-NET-001] [SRC-NET-003] [SRC-NET-004] [SRC-NET-005]
