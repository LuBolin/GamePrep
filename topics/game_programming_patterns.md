# Game Programming Patterns in Unreal

See also: [[game_architecture_patterns]], [[ue_cpp_idioms]], [[ue_massentity_ecs]], [[advanced_gameplay_patterns_specialist]], [[ue_hands_on_projects]].

Patterns are a vocabulary for recurring design forces. They are useful when they make ownership, change, lifetime, authority, data flow, or performance clearer. They are harmful when they add indirection in anticipation of requirements that never arrive.

This chapter targets **D3–D4** interview depth for a three-year Unreal engineer: recognise a pattern in production code, explain its cost, map it to Unreal mechanisms, debug its failure modes, and choose a simpler alternative when appropriate. [SRC-ARCH-001]

## 1. Start with forces, not pattern names

Before saying “use Observer” or “make it a Component”, state the pressure:

1. **Change axis:** what is expected to vary—content, algorithm, platform, game mode, input source, or rules?
2. **Lifetime and owner:** who creates, retains, resets, and destroys each object?
3. **Authority and audience:** who may mutate state, and who observes it locally or over the network?
4. **Time semantics:** immediate call, end-of-frame batch, queued event, replicated state, persisted record, or replayable command?
5. **Data volume and frequency:** how many instances/messages/updates, and on which thread?
6. **Failure policy:** what if a target disappears, an asset is not loaded, a message is duplicated, or a definition changes?
7. **Tooling:** how will designers author/validate it and engineers trace it?

A pattern is justified when its new concept pays for itself against those forces.

## 2. SOLID as practical design pressure

SOLID is better used as a set of questions than as a verdict on every class.

### Single responsibility / one reason to change

Ask whether unrelated stakeholders or change axes collide. A `UWeaponComponent` that owns equip state, performs attacks, renders menus, writes save files, awards quests, and uploads telemetry has several reasons to change. Splitting every five-line function into a class is not SRP; cohesive behaviour can remain together.

In Unreal, practical seams often follow:

- authored definition versus runtime mutable instance;
- authoritative rules versus replicated presentation state;
- gameplay model versus UI projection;
- reusable capability versus owning Actor orchestration;
- runtime module versus editor-only authoring/validation.

### Open/closed

Stable policy seams can support new content/strategies without editing central conditionals. Unreal tools include data assets, Blueprint subclasses, interfaces, components, tags, and policy UObjects. But an extensibility point creates API/lifecycle/versioning cost. If there are two fixed cases and no credible third, a clear `switch` may be superior.

### Liskov substitution

A subtype must preserve the behavioural contract expected by its callers—not merely compile. A weapon subtype that sometimes consumes negative ammo, changes authority rules, or requires a different hidden initialisation sequence is not safely substitutable. Composition/Strategy often fits variants whose invariants diverge.

### Interface segregation

Consumers should depend on the capabilities they need. `IDamageable`, `IInteractable`, or a narrow read-only inventory view is often clearer than exposing a giant `IGameplayEverything`. Tiny interfaces are not free: each adds discovery, wiring and versioning. Split around real consumers and authority boundaries.

### Dependency inversion

High-level policy should not be coupled unnecessarily to a volatile concrete provider. A quest evaluator can consume a narrow event/state interface rather than reaching into a specific Character, widget, and save singleton. Dependency injection may be constructor parameters for ordinary C++, explicit initialisation for UObjects, component owner contracts, or subsystem lookup at a composition boundary.

**Important distinction:** dependency inversion is a dependency-direction principle; a dependency-injection container is one possible mechanism. Unreal projects rarely need a general-purpose container merely to follow it.

## 3. Component pattern: compose coherent capabilities

The Component pattern allows one entity to delegate distinct domains to contained objects. UE's Actor/ActorComponent model is a direct engine-level expression, with registration, activation, Tick, replication, and owner lifecycle semantics. [SRC-ARCH-002] [SRC-EPIC-012]

### Good Component boundary

A health component can own:

- current/max health runtime state;
- validated authoritative damage/heal mutation;
- death threshold transition;
- compact change/death events;
- replication of its state when configured.

It should not need to know the concrete HUD, GameMode, quest implementation, save slot path, or every damage source class. Those consumers observe or orchestrate it.

### Component versus inheritance

Choose inheritance when the subtype genuinely preserves an “is-a” contract and uses a shared framework protocol. Choose composition when capabilities vary independently, need reuse across unrelated Actor classes, or create combinatorial subclasses.

`AFireFlyingArmouredEnemy` is a warning sign if fire, flight, and armour are independent features. Components are not automatically correct if they then discover each other via casts and form an implicit dependency web.

### Owner contract

Document:

- required owner interfaces/components;
- who calls initialisation and when;
- whether the component is server-authoritative;
- what it replicates and to whom;
- events it emits and their timing;
- teardown/unbinding/reset rules;
- whether runtime addition/removal is supported.

## 4. Observer: synchronous notification with lifetime cost

Observer lets a subject notify interested listeners without naming concrete consumer types. Native/dynamic multicast delegates are the usual UE mechanism. Delegate invocation is normally synchronous: observers run during the broadcast unless the implementation explicitly queues work. [SRC-ARCH-003] [SRC-ARCH-013]

### Appropriate uses

- health changed → UI projection, audio response, local analytics;
- inventory changed → refresh affected views;
- objective changed → presentation and dependent rule updates;
- subsystem connection state changed → flow coordinator reacts.

### Event payloads are contracts

Prefer a payload that describes the transition adequately: old/new value, delta/reason, stable source/instigator identifier, sequence/version if needed. If every listener immediately calls back into the subject for five fields, the event is under-specified and can observe inconsistent intermediate state.

### Failure modes

- stale raw/lambda captures after listener destruction;
- duplicate binding after BeginPlay/reinitialisation;
- forgotten unbinding from long-lived publishers;
- re-entrant mutation during broadcast;
- unspecified listener order treated as logic order;
- event cascade where one action synchronously triggers dozens of hidden actions;
- dynamic delegate selected in a hot native-only path without need.

Observers are best for notification, not for hiding a required dependency. If the caller needs an answer or success/failure from one collaborator, an explicit interface/function call is clearer.

## 5. Command: represent an intention as data/behaviour

Command encapsulates a request so it can be queued, mapped, logged, replayed, undone, validated, or executed by another receiver. [SRC-ARCH-004]

Examples:

- input action → `FMoveCommand`/`FActivateSlotCommand`;
- editor tool operation with undo/redo;
- tactical order queued for units;
- deterministic simulation input for replay/rollback;
- server request DTO validated into an authoritative action.

### A useful command contract

- stable type/identifier;
- compact parameters and target identifiers—not fragile raw pointers for persistence/networking;
- issuer and sequence/timestamp where ordering matters;
- validation separate from authoritative execution;
- explicit result/rejection reason;
- serialisation/versioning if persisted or transmitted;
- undo data only when a true reversible operation is required.

### Command is not automatically networking

A client command/RPC expresses intent. The server still validates ownership, state, rate, range, cooldown, target existence, and anti-cheat rules. Replaying commands requires deterministic enough state or recorded outcomes. Reliable transport does not make a command valid or idempotent.

### When a function call is enough

If an operation is immediate, local, not queued/logged/replayed, and has one caller/receiver, a function call is simpler. Do not allocate command UObjects for every button merely to claim decoupling.

## 6. State and Strategy: mode versus interchangeable policy

These patterns often look similar in class diagrams but answer different questions.

### State

State changes an object's behaviour according to its current mode and transitions. [SRC-ARCH-005]

Use it when enter/exit/transition semantics matter: weapon idle/firing/reloading, interaction idle/holding/completing, game flow loading/lobby/match/results. Define:

- allowed transitions and their guards;
- transition owner/authority;
- enter/exit side effects and cancellation;
- timers/tasks/delegates owned by each state;
- whether transitions are immediate, queued, or replicated;
- recovery from invalid/missing state.

A `bool bReloading`, `bFiring`, `bEquipping`, and `bStunned` cluster often permits impossible combinations. One state machine may help for mutually exclusive modes. Independent orthogonal modes may need concurrent state regions/components, not one cross-product enum.

### Strategy

Strategy selects an interchangeable algorithm/policy behind a stable contract: targeting policy, damage falloff, item sorting, navigation cost policy, reward calculation, or firing pattern.

The owner usually chooses/injects a Strategy; a State usually participates in transitions based on runtime mode. A policy can be an ordinary C++ value/type, function/delegate, UObject/DataAsset-backed strategy, interface implementation, or template—choose from reflection, lifetime, hot-path and authoring needs.

### Branch or object?

Use a data enum/branch when the set is small, stable and transparent. Use Strategy objects when variants have substantial independent code/state, are authored/loaded independently, or must be tested/substituted without modifying the owner.

## 7. Factory and Prototype: centralise valid creation

A Factory owns creation choices and invariant setup. Unreal already supplies factories at several layers: `SpawnActor`, `NewObject`, class defaults, Data Assets, Asset Manager, Blueprint classes, and editor asset factories.

A gameplay factory is useful when callers should not know:

- concrete implementation class/platform variant;
- spawn pool versus fresh creation;
- required owner/instigator/team/definition setup;
- authority constraints;
- async asset load path;
- registration with managers or telemetry.

Avoid a giant `UGameObjectFactory` switch for every type. Put creation near the bounded domain and return the narrowest useful contract.

Prototype creates/configures instances from an exemplar. UE's CDO/defaults and Blueprint class hierarchy resemble prototype-style authored defaults, but duplicating a live UObject/Actor is not equivalent to safe runtime spawning. References, world registration, ownership, replication, transient state, and instanced subobjects require supported APIs and explicit contracts.

## 8. Object Pool: exchange churn for retained complexity

Pooling reuses allocated/constructed objects instead of creating/destroying them repeatedly. It is justified by measured churn or expensive registration—not by the noun “projectile”. [SRC-ARCH-008]

### Pool contract

- acquire/release ownership;
- maximum capacity and exhaustion policy;
- full reset of timers, delegates, movement, effects, collision, visibility, owner/instigator, replicated/transient state;
- stale-handle protection/generation IDs where relevant;
- no double release;
- warm-up and shrink policy;
- retained memory budget;
- world/travel teardown.

### Actor/replication cautions

Hiding/deactivating an Actor does not recreate spawn/destruction replication semantics. Network identity, relevancy, dormancy, client state, BeginPlay/EndPlay assumptions, and external references can leak across uses. Often a batched manager, Niagara system, data-oriented projectile simulation, or lightweight struct is better than pooling full Actors.

## 9. Service Locator and Subsystems

Service Locator provides a central way to retrieve services. It is convenient but hides dependencies at the call site, encourages global lifetime, makes tests depend on environment setup, and can turn access order into runtime failure. [SRC-ARCH-007]

UE Subsystems improve one aspect: they bind service instances to explicit hosts such as Engine, GameInstance, World, LocalPlayer, or Editor. They do not erase locator trade-offs. [SRC-ARCH-014]

### Appropriate subsystem use

- one service naturally exists per host lifetime;
- callers across that scope need coordinated access;
- state is not naturally owned by one Actor/component;
- initialisation/deinitialisation can be explicit;
- multi-world/client semantics are understood.

### Warning signs

- every gameplay class calls five global subsystems directly;
- WorldSubsystem stores process/session state or GameInstanceSubsystem stores level Actors across travel;
- tests cannot construct logic without a complete World;
- subsystem reaches back into arbitrary Actors/UI and becomes a god manager;
- service retrieval is used deep in pure calculations.

Use the subsystem as a **composition boundary**: retrieve it near orchestration, then pass narrow interfaces/data into core logic when that improves clarity/testability.

## 10. Event Queue and message bus

An Event Queue decouples send time from processing time. A message bus also decouples sender from receiver identity/topic routing. [SRC-ARCH-006]

This is different from a normal delegate:

| Mechanism | Typical timing | Best for | Main risk |
|---|---|---|---|
| direct call/interface | immediate | required collaboration/result | concrete coupling if boundary is broad |
| delegate/Observer | immediate broadcast | local state-change notification | lifetime, re-entrancy, hidden cascade |
| event queue | later ordered processing | smoothing/batching/cross-system handoff | latency, overflow, stale targets |
| message bus | direct or queued topic routing | bounded cross-domain communication | invisible dependencies, schema/version sprawl |
| replicated property/state | network convergence | durable current truth | latency and OnRep/order assumptions |
| RPC | network event/request | transient targeted intent/notification | ownership, reliability and validation |

### Delivery semantics must be designed

Specify:

- immediate, end-of-frame, fixed-step, or background processing;
- ordering scope and priority;
- bounded capacity/backpressure/drop/coalescing;
- target lifetime and stable identifiers;
- duplicate/idempotency behaviour;
- thread safety and payload ownership;
- observability: sequence, source, topic, queue depth and processing time;
- whether late subscribers need current state (often a state query, not event replay).

A single global `FGameplayMessage` bag of variants is not modularity. Use domain-owned schemas and explicit bridges.

## 11. Data-driven design: definitions are not runtime state

Data-driven design moves tunable/content variation into authored, validated definitions while stable code interprets them. UE supports Data Assets, Primary Data Assets, DataTables, CurveTables, configs, tags, Blueprint classes and asset management. [SRC-ARCH-009] [SRC-ARCH-010] [SRC-ARCH-011]

### The three-layer model

1. **Definition:** immutable-ish authored content: item ID/name, tags, icon soft reference, stack rules, base stats, ability class/reference.
2. **Runtime instance/state:** quantity, durability, random roll, cooldown, owner, equipped slot, quest progress.
3. **Presentation:** localised text/icon/widget/effects derived from definition + state.

Do not save or replicate an entire Data Asset repeatedly. Save/replicate a stable definition ID plus mutable state, then resolve the local definition. Define behaviour when an ID is missing or content version changes.

### Data Asset versus DataTable

- Data Asset: individually referenced rich object graph, inheritance/asset references/editor detail customisation; works naturally with Primary Asset identity/bundles.
- DataTable: homogeneous row schema, compact spreadsheet/CSV-oriented bulk balancing and lookup.
- CurveTable/Curve: numeric progression/interpolation.
- Blueprint class/defaults: authored type with behaviour/default components when spawning a class is genuinely required.

Do not select a DataTable merely because “designers like Excel”; schema migration, row identity, validation, references, merge conflicts and loading still need design.

### Validation

Editor-time Data Validation should reject impossible definitions: missing IDs, negative costs, invalid tag combinations, hard-reference budget violations, cycles, absent assets, impossible stack sizes, or unsupported strategy combinations. This moves failures from runtime and interviews into authoring feedback. [SRC-ARCH-012]

## 12. Dirty Flag, Double Buffer, Flyweight, and Type Object

### Dirty Flag

Cache derived data and mark it dirty when inputs change; recompute once when needed or at a controlled phase. Useful for aggregate stats, UI view models, expensive transforms or batched queries. Bugs arise from missed invalidation, repeated writes, thread visibility, and stale network/save state. Always define the authoritative inputs and recomputation point.

### Double Buffer

Maintain current/next buffers so producers do not expose partially updated state to consumers. Useful in fixed-step simulation, transforms/particles, and cross-thread snapshots. Costs include memory, copy/swap semantics, one-step latency, and ownership of references inside buffers.

### Flyweight

Share immutable intrinsic data while storing per-instance extrinsic state separately. Item definitions are flyweights; an inventory stack carries quantity/durability/owner. This aligns with Data Assets but only if runtime code does not mutate the shared definition.

### Type Object

Represent “type” variation as runtime data/object rather than a compiled subclass for each content case. A weapon definition plus selected strategies/tags can replace hundreds of shallow subclasses. If variations truly need distinct framework behaviour/components, classes remain valid.

## 13. Pattern interaction example: weapon system

A bounded weapon design might use:

- **Component:** equipment/weapon capability attached to a Pawn;
- **Data Asset / Flyweight:** immutable weapon definition and soft presentation assets;
- **runtime struct/UObject:** ammo, heat, random seed, attachments, current mode;
- **State:** idle/equipping/firing/reloading with explicit enter/exit cancellation;
- **Strategy:** hitscan/projectile/beam targeting or damage falloff policy;
- **Command:** input intent with sequence/parameters, server validation and prediction hooks;
- **Observer:** compact ammo/state changes to UI/audio/animation;
- **Factory:** authoritative spawn/setup or runtime instance creation;
- **Pool:** only a measured effect/projectile representation pool with strict reset;
- **Subsystem:** perhaps a world-scoped projectile simulation or definition registry, not every weapon's mutable state;
- **Dirty Flag:** aggregate modifiers recomputed once after attachment changes.

This is not a recommendation to implement every bullet. Begin with component + definition + explicit runtime state; add patterns only when their force appears.

## 14. Debugging architecture

### Trace data flow, not class diagrams

For one broken transition, record:

1. source action/event/command and stable correlation ID;
2. authority/world/net role and owner;
3. current state/version before mutation;
4. validation and rejection reason;
5. authoritative mutation;
6. emitted synchronous/queued/network notifications;
7. observer count, queue depth and processing phase;
8. save/replication/presentation projection.

### Common architecture smells

- bidirectional component dependencies and cast chains;
- getters returning mutable internal arrays/maps;
- UI directly mutating authoritative model;
- definition object mutated as per-instance state;
- SaveGame storing live UObject/Actor pointers;
- events used as durable truth;
- global bus topics with no owner/version;
- asynchronous callback after owner/world teardown;
- `GetSubsystem` in low-level calculations/tests;
- abstraction with one implementation and no credible seam pressure;
- central enum switch changed for every new content asset;
- pooling without reset/exhaustion diagnostics.

## 15. Profiling architecture

Patterns have measurable costs:

- Observer: listener count, broadcast frequency, payload copies, cascade duration;
- Command/queue: allocations, queue depth, latency, drops/coalescing, processing burst;
- Components: Actor/component count, registration, Tick, replication and cache locality;
- Strategies/UObjects: virtual/reflection calls and object/allocation count in hot loops;
- data definitions: hard-reference closure, load time, memory, lookup frequency;
- pools: peak retained memory, reset time, hit/miss/exhaustion and stale use;
- services: contention, hidden work, startup/travel ordering;
- dirty caches: invalidation rate, recompute count and stale-read incidence.

Architecture is not performance-neutral. First preserve a clear contract; then measure whether a simpler/batched/data-oriented representation is needed.

## 16. Interview answer framework

For “Which pattern would you use?” answer:

1. Restate the force and constraints.
2. Offer the simplest baseline.
3. Introduce the pattern and map it to UE mechanisms.
4. Define ownership, lifetime, authority, timing, loading and data contracts.
5. Name one failure mode and diagnostic.
6. Name the threshold at which you would simplify or escalate.

Example:

> Health changes have one authoritative writer but several local consumers. I would keep validated mutation and state in a reusable Health Component and expose a compact synchronous change delegate for presentation. I would not let UI mutate it or assume listener order. The server replicates durable health state; the delegate is local notification, not the network protocol. I would bind/unbind by lifecycle and trace duplicate/re-entrant broadcasts. If there is only one required collaborator needing a result, I would use a direct interface call instead of a multicast event.

## 17. Hands-on verification

Build Project 8 as a small sandbox where the same interaction can run through direct call, delegate, queued message and Command. Instrument order, latency, allocations, listener lifetime and failure behaviour. Then add a data-defined weapon with runtime state, State/Strategy variants, and one deliberately bad global subsystem. Refactor it at a measured seam and document what became easier—and what became more complex.

## 18. Source map

- Pattern vocabulary/trade-offs: [SRC-ARCH-001]–[SRC-ARCH-008]
- UE Components/delegates/subsystems: [SRC-EPIC-012] [SRC-ARCH-013] [SRC-ARCH-014]
- Data Assets/tables/asset identity: [SRC-ARCH-009] [SRC-ARCH-010] [SRC-ARCH-011]
- Authored-data validation: [SRC-ARCH-012]

