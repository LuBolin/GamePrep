# Unreal Engine Interview Question Bank

## UObject, Reflection, and Lifetime

### Question: Why does Unreal need reflection on top of C++?

- **Category:** UObject / Reflection
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Compiled C++ does not provide the structured runtime metadata Unreal needs for editor tooling, Blueprint exposure, serialisation, replication declarations, and managed object references.
- **Strong 3-Year-Engineer Answer:** Unreal opts selected declarations into an engine-visible model using macros and UHT-generated code. The ordinary C++ compiler still compiles the result. Reflection lets the engine enumerate supported classes, properties, and functions and attach specifiers that drive distinct systems. I would not make every type reflected: hot value data and RAII helpers can stay ordinary C++, while a `USTRUCT` is useful when value semantics plus property/serialisation support are enough. The design trade-off is engine integration versus extra constraints and lifecycle cost.
- **Common Weak Answer:** “So Blueprints can see C++.”
- **Follow-up Questions:** What does UHT generate? When is `USTRUCT` enough? What breaks if a type is unsupported by UHT?
- **Hands-on Verification Task:** Expose one C++ type as a reflected struct and another as a UObject; compare editor, Blueprint, serialisation, and lifetime behaviour.
- **Sources:** [SRC-EPIC-001], [SRC-EPIC-002], [SRC-EPIC-008]
- **Version Notes:** Stable UE4/UE5; verified for UE5.6.

### Question: What does `UPROPERTY` do besides exposing a field to the editor?

- **Category:** UObject / Reflection / UE C++
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Depending on type and specifiers, it can make a property visible to reflection-driven GC reference discovery, serialisation, replication declarations, Blueprint access, config/save systems, and editor tooling.
- **Strong 3-Year-Engineer Answer:** I treat those as separate capabilities. `EditAnywhere` is an editor-editing policy; `BlueprintReadOnly` is a scripting access policy; `Transient` is a persistence policy; `Replicated` declares networking intent; and a reflected strong UObject member gives GC a discoverable reference edge. I choose only the specifiers required by the contract and keep mutation constrained rather than exposing everything broadly.
- **Common Weak Answer:** “It prevents garbage collection and shows the variable in Blueprint.”
- **Follow-up Questions:** Does every UPROPERTY keep a UObject alive? What is `Transient`? Editor visibility versus Blueprint accessibility?
- **Hands-on Verification Task:** Make four fields with intentionally different editor, Blueprint, save, and lifetime contracts.
- **Sources:** [SRC-EPIC-001], [SRC-EPIC-007], [SRC-EPIC-009]
- **Version Notes:** Stable concept; prefer `UPROPERTY() TObjectPtr<T>` for persistent UE5 UObject fields.

### Question: Explain UObject garbage collection without saying “Unreal handles it”.

- **Category:** UObject / Memory
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Unreal periodically marks UObjects reachable from roots through discoverable strong references and reclaims unreachable objects in a later sweep/destruction process.
- **Strong 3-Year-Engineer Answer:** I reason about a reference graph. A reflected strong property normally forms an edge; weak and soft references do not retain their targets. C++ lexical scope is not the ownership mechanism, so a raw local disappearing does not immediately destroy a UObject and a raw member does not necessarily retain it. For a disappearing object I identify the intended root-to-target chain and force GC in a controlled test. For hitches or retention I profile collection, then inspect object count, churn, roots, strong fields, native reference bridges, and caches before changing GC timing.
- **Common Weak Answer:** “Anything without UPROPERTY gets deleted.”
- **Follow-up Questions:** What is a root? Can a cycle be collected? How is incremental reachability different?
- **Hands-on Verification Task:** Complete the five-case lifetime lab and explain each outcome.
- **Sources:** [SRC-EPIC-005], [SRC-EPIC-006], [SRC-EPIC-009]
- **Version Notes:** Mark-and-sweep is baseline; incremental reachability is Experimental in UE5.6.

### Question: Why should a persistent UObject member usually be `UPROPERTY() TObjectPtr<T>`?

- **Category:** UObject / Memory / UE C++
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** It expresses a reflection-visible strong UObject reference that the collector and configured engine systems can track.
- **Strong 3-Year-Engineer Answer:** The owner needs a persistent retaining edge, so I use a reflected `TObjectPtr`. A raw member does not express the same UE5 contract; a weak pointer intentionally allows collection; and a soft pointer represents a path/load dependency rather than an already-loaded retained object. `TObjectPtr` also supports UE5 cook-time dependency tracking and barriers used by experimental incremental GC. I still avoid worker-thread dereference unless lifetime and thread constraints are explicitly guaranteed.
- **Common Weak Answer:** “Because raw pointers are always unsafe.”
- **Follow-up Questions:** Is a local raw pointer acceptable? Does `TObjectPtr` without UPROPERTY retain? How do you reference from a non-UObject owner?
- **Hands-on Verification Task:** Remove the reflected edge in the lifetime lab and force collection.
- **Sources:** [SRC-EPIC-005], [SRC-EPIC-009]
- **Version Notes:** UE5-preferred idiom; recognise raw reflected pointer declarations in older code.

### Question: What is the difference between weak and soft UObject references?

- **Category:** UObject / Assets / Lifetime
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** A weak pointer observes a loaded object without retaining it; a soft pointer stores an object path and can represent an unloaded asset for explicit loading.
- **Strong 3-Year-Engineer Answer:** I choose weak for caches, observers, and delayed callbacks where the object may disappear. I resolve and validate at the point of use. I choose soft for asset or class dependencies that should not become hard load dependencies; the code handles null, pending, and valid states and owns the async loading flow. Neither keeps the target alive by itself, but only soft references provide the on-disk locator/loading role.
- **Common Weak Answer:** “Soft pointers are safer weak pointers.”
- **Follow-up Questions:** When would a hard asset reference be preferable? What owns the loaded asset after an async load? Why not store paths as strings?
- **Hands-on Verification Task:** Observe a weak actor after destruction and a soft asset before/during/after async loading.
- **Sources:** [SRC-EPIC-009]
- **Version Notes:** Stable UE5 guidance; `TLazyObjectPtr` is deprecated.

### Question: What does `Outer` mean, and does it keep an object alive?

- **Category:** UObject / Lifetime
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Outer establishes containment and identity context, including object paths; do not rely on it as a substitute for an explicit strong reference.
- **Strong 3-Year-Engineer Answer:** I pass the logical containing object to `NewObject` so names, paths, package/context relationships, and lookup make sense. Separately, I store the child through a discoverable strong edge if it must survive. This avoids the dangerous shorthand “Outer owns it”, which conflates containment with reachability. I verify exact subsystem-specific lifecycle assumptions rather than generalising from one object type.
- **Common Weak Answer:** “Outer is the parent and automatically owns/deletes the child.”
- **Follow-up Questions:** How is Outer different from Actor attachment? What Outer would a transient helper use? What creates the retaining edge?
- **Hands-on Verification Task:** Create two children with the same Outer but retain only one, then force collection.
- **Sources:** [SRC-EPIC-004], [SRC-EPIC-006]
- **Version Notes:** Stable UE4/UE5 mental model.

### Question: What is the CDO, and why should constructors be lightweight?

- **Category:** UObject / Lifecycle
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** The Class Default Object is the class template from which defaults are derived; constructors initialise defaults and default subobjects, so world-dependent gameplay work belongs later.
- **Strong 3-Year-Engineer Answer:** A UObject constructor is not merely a per-spawn gameplay hook. It participates in creating class defaults, including Blueprint-derived defaults, and can run when no valid gameplay world exists. I set deterministic defaults and create default subobjects there. I defer world lookups, dynamic gameplay state, and instance bindings to the correct lifecycle phase. When debugging constructor surprises, I log object flags and distinguish CDO, archetype, editor preview, and runtime instance.
- **Common Weak Answer:** “BeginPlay happens after the constructor.”
- **Follow-up Questions:** `CreateDefaultSubobject` versus `NewObject`? Why can changing a C++ default update placed instances? What about component initialisation?
- **Hands-on Verification Task:** Log constructor and BeginPlay calls for the CDO, editor instances, and runtime instances.
- **Sources:** [SRC-EPIC-003], [SRC-EPIC-006]
- **Version Notes:** Stable UE4/UE5.

### Question: Why is `TSharedPtr<UObject>` usually the wrong ownership model?

- **Category:** C++ / UObject Lifetime
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Reference-counted C++ smart pointers and Unreal's UObject reachability collector are separate lifetime systems; UObject references should use the UObject-aware mechanisms that express retention, observation, or loading intent.
- **Strong 3-Year-Engineer Answer:** `TSharedPtr` is useful for ordinary non-UObject objects with shared deterministic ownership. It does not make a normal GC-discoverable UObject property edge or replace Unreal's identity, serialisation, and weak/soft pointer semantics. For a retained UObject member I use reflected `TObjectPtr`; for observation, `TWeakObjectPtr`; for deferred asset loading, a soft pointer; and for a deliberate native-owner bridge, `TStrongObjectPtr` or `FGCObject` as appropriate. The important thing is not pointer fashion but keeping one coherent lifetime domain.
- **Common Weak Answer:** “Unreal forbids smart pointers.”
- **Follow-up Questions:** When is `TSharedPtr` correct in UE code? What does `TStrongObjectPtr` do? How can cycles behave in each model?
- **Hands-on Verification Task:** Implement an ordinary C++ RAII owner next to a UObject graph and document the destruction trigger for each.
- **Sources:** [SRC-EPIC-003], [SRC-EPIC-009]
- **Version Notes:** Stable principle; dedicated UObject pointer options have evolved in UE5.

## Gameplay Framework and Lifecycle

### Question: Actor or Actor Component—how do you decide?

- **Category:** Gameplay Framework / Architecture
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Use an Actor for independent world identity, spawning, placement, or replication relevance; use a Component for a coherent capability whose lifetime and context belong to an Actor.
- **Strong 3-Year-Engineer Answer:** I start from lifetime and world identity. A pickup or projectile is naturally an Actor because it exists independently in the world. Health is often a Component because it is reusable behaviour attached to many Actor types and does not need its own transform. I keep the component's owner contract narrow and expose events/interfaces rather than having it discover UI, save, and sibling systems through casts. If the data is just copyable configuration, I use a struct or DataAsset instead of either.
- **Common Weak Answer:** “Components are better than inheritance.”
- **Follow-up Questions:** Actor Component versus Scene Component? Can Components replicate? When is a UObject better?
- **Hands-on Verification Task:** Implement health as both Character code and a Component, then compare dependencies and reuse on a destructible prop.
- **Sources:** [SRC-EPIC-011], [SRC-EPIC-012]
- **Version Notes:** Stable UE4/UE5.

### Question: Pawn versus Character?

- **Category:** Gameplay Framework
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Character is a specialised Pawn with humanoid-oriented collision, movement, animation, and network movement support; Pawn is the more general possessable agent.
- **Strong 3-Year-Engineer Answer:** I choose Character when its capsule, skeletal conventions, CharacterMovement, and replication/prediction model fit the design. For a vehicle, board piece, spectator camera, or radically custom locomotion, Pawn avoids fighting assumptions I do not need. I would not choose Character solely because the entity is controlled; possession is provided by Pawn.
- **Common Weak Answer:** “Character is for players and Pawn is for AI.”
- **Follow-up Questions:** Can AI possess a Character? Where should movement input be translated? Why separate Controller?
- **Hands-on Verification Task:** Possess the same custom Pawn with PlayerController and AIController, then compare with Character movement.
- **Sources:** [SRC-EPIC-010]
- **Version Notes:** Stable UE4/UE5.

### Question: Why separate Controller from Pawn?

- **Category:** Gameplay Framework / Architecture
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Controller represents decision-making/player connection while Pawn is the world avatar, allowing possession changes, respawn, spectating, and reuse across human and AI control.
- **Strong 3-Year-Engineer Answer:** Their lifetimes and responsibilities differ. A player can keep a Controller and PlayerState while destroying and replacing a Pawn; an AIController can possess a different agent; input intent can be translated without embedding connection/UI concerns in the avatar. I still keep avatar capabilities on the Pawn/components so AI and alternate controllers can drive them through a common command boundary.
- **Common Weak Answer:** “Controller reads input and Pawn moves.”
- **Follow-up Questions:** What survives respawn? Where does owning-client UI coordination live? Possession versus ownership?
- **Hands-on Verification Task:** Implement death, spectating, and respawn without reconstructing player score or connection-level state.
- **Sources:** [SRC-EPIC-010], [SRC-EPIC-018]
- **Version Notes:** Stable UE4/UE5.

### Question: GameMode versus GameState?

- **Category:** Gameplay Framework / Networking
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** GameMode owns authoritative match rules and exists only on the server; GameState exposes changing match state that exists on server and clients and can replicate.
- **Strong 3-Year-Engineer Answer:** GameMode decides whether a score is valid, when the match transitions, and how players spawn. GameState stores the resulting public match phase, team scores, and synchronised information clients need. I mutate authoritative state on the server and replicate through GameState. A design that reads GameMode from client UI may work standalone and fail immediately in multiplayer because clients do not have it.
- **Common Weak Answer:** “GameMode is logic, GameState is variables.”
- **Follow-up Questions:** Where does server time come from? Per-player score? What exists during travel?
- **Hands-on Verification Task:** Break a score widget by reading GameMode on a client, then repair the boundary with GameState.
- **Sources:** [SRC-EPIC-010], [SRC-EPIC-013]
- **Version Notes:** Stable UE4/UE5; confirmed for UE5.6.

### Question: PlayerController versus PlayerState?

- **Category:** Gameplay Framework / Networking
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** PlayerController represents control/connection and exists on server plus its owning client; PlayerState represents public participant state and replicates to all clients.
- **Strong 3-Year-Engineer Answer:** Input orchestration, possession, owning-client RPCs, and local UI coordination naturally involve PlayerController. Name, team, public score, and participant state that others must see belong in PlayerState and can survive Pawn replacement. I do not place secrets in broadly replicated PlayerState merely because it survives respawn; owner-only data needs a different replication policy/home.
- **Common Weak Answer:** “Controller has logic; PlayerState has data.”
- **Follow-up Questions:** Does every client have every PlayerController? What survives seamless travel? Where should private inventory live?
- **Hands-on Verification Task:** Show a scoreboard listing remote PlayerStates without accessing remote PlayerControllers.
- **Sources:** [SRC-EPIC-010], [SRC-EPIC-014]
- **Version Notes:** Stable UE4/UE5; API confirmed for UE5.6.

### Question: Constructor, OnConstruction, or BeginPlay?

- **Category:** Gameplay Framework / Lifecycle
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Constructor establishes class defaults/default subobjects; OnConstruction derives instance setup and can rerun; BeginPlay starts runtime gameplay after initialisation.
- **Strong 3-Year-Engineer Answer:** I keep constructors deterministic because they initialise CDOs and may run without a gameplay world. Construction logic can run in editor workflows and more than once, so it must tolerate reruns and avoid external irreversible side effects. BeginPlay is for runtime bindings, timers, and gameplay service access after the Actor/components are initialised, while cleanup belongs in EndPlay because destruction, travel, streaming, and PIE shutdown all end play.
- **Common Weak Answer:** “Use BeginPlay if the constructor crashes.”
- **Follow-up Questions:** Where does PostInitializeComponents fit? Deferred spawn? Why clean up in EndPlay?
- **Hands-on Verification Task:** Log each hook for placed, spawned, deferred-spawned, and PIE-duplicated Actors.
- **Sources:** [SRC-EPIC-003], [SRC-EPIC-015], [SRC-EPIC-016]
- **Version Notes:** Broad model stable; detailed lifecycle source is UE4.27-era, so verify exact UE5 ordering where material.

### Question: When should logic live in a subsystem?

- **Category:** Gameplay Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Use a subsystem for an auto-instanced service whose responsibility naturally shares an Engine, Editor, GameInstance, World, or LocalPlayer lifetime and does not need Actor presence/replication.
- **Strong 3-Year-Engineer Answer:** I choose the narrowest host lifetime. A world-scoped registry fits a WorldSubsystem; local UI/input state fits a LocalPlayerSubsystem; session coordination across maps can fit a GameInstanceSubsystem. I would not use a subsystem to avoid dependency design, to hold arbitrary globals, or to replace replicated GameState. I make dependencies and teardown explicit and test multiple PIE worlds/local players.
- **Common Weak Answer:** “Subsystems are Unreal singletons.”
- **Follow-up Questions:** GameInstanceSubsystem versus WorldSubsystem? How is replication handled? What happens in PIE?
- **Hands-on Verification Task:** Implement a world registry and prove two PIE worlds do not share it.
- **Sources:** [SRC-EPIC-017]
- **Version Notes:** UE4.22+/UE5 stable concept; API confirmed for target-era UE5.

### Question: Tick, timer, or event?

- **Category:** Gameplay Framework / Performance
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Tick for true frame-dependent work, timers for scheduled/coarse work, and events for state transitions; optimise the work and frequency based on measurement.
- **Strong 3-Year-Engineer Answer:** I first define latency and cadence. Movement integration may need Tick; a poison pulse may use a timer; a health bar should react to a health-changed event. Replacing Tick with a timer at frame rate changes little. I inspect enabled tick count and work per call, eliminate per-frame searches/allocations, batch where useful, disable idle updates, and profile the relevant thread.
- **Common Weak Answer:** “Never use Tick; timers are faster.”
- **Follow-up Questions:** Tick intervals? Component tick defaults? How do you avoid stale timer callbacks?
- **Hands-on Verification Task:** Implement one behaviour three ways and compare invocation count, latency, and CPU trace.
- **Sources:** [SRC-EPIC-012]
- **Version Notes:** Stable UE4/UE5 performance reasoning.

## UE Core Types and C++/Blueprint Integration

### Question: `FName`, `FString`, or `FText`?

- **Category:** UE C++ Idioms / Strings
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Use FName for identifiers, FString for mutable/general character data, and FText for localisable user-facing display text.
- **Strong 3-Year-Engineer Answer:** I choose by semantics, not microbenchmark. FName uses name-table identity and is efficient for repeated identifier comparisons but is immutable and case-insensitive in its comparison semantics. FString owns mutable characters for parsing/building/logging. FText carries localisation and culture-aware formatting history. I minimise conversions: FText-to-FString can lose localisation history and FString-to-FName can lose case distinctions.
- **Common Weak Answer:** “FName is fastest, FString is normal, FText is for UI.”
- **Follow-up Questions:** Player-supplied display name? Asset identifier? What does `TEXT()` do? Why can conversion be lossy?
- **Hands-on Verification Task:** Audit twenty string fields in a small project and justify each type plus conversion boundary.
- **Sources:** [SRC-EPIC-022], [SRC-EPIC-023]
- **Version Notes:** Stable UE4/UE5; target UE5.3–UE5.6.

### Question: Compare `std::vector` and `TArray`.

- **Category:** Standard C++ / UE C++ / Containers
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Both are contiguous value-owning dynamic sequences; TArray integrates with UE reflection/serialisation APIs and uses UE's relocation/allocator model.
- **Strong 3-Year-Engineer Answer:** Both own elements, deep-copy as values, grow dynamically, and can invalidate element handles on reallocation/mutation. I use TArray at reflected and engine-facing boundaries because UPROPERTY, Blueprint, serialisation, and UE APIs understand it. A pure standard library may use vector internally. I do not claim either is universally faster, and I do not assume every std::vector invalidation or relocation detail maps identically to TArray.
- **Common Weak Answer:** “TArray is Unreal's faster vector.”
- **Follow-up Questions:** What is trivially relocatable in UE? When does TArray invalidate pointers? `RemoveAt` versus `RemoveAtSwap`?
- **Hands-on Verification Task:** Implement equivalent growth/removal workloads and inspect addresses, copies/moves, order, and engine integration.
- **Sources:** [SRC-CPP-009], [SRC-EPIC-024]
- **Version Notes:** UE5.3–UE5.6; exact relocation traits are version-sensitive specialist depth.

### Question: When would you choose `TSet` or `TMap` over `TArray`?

- **Category:** UE C++ Idioms / Containers
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Choose TSet for unique unordered values and TMap for unique-key lookup; choose TArray when sequence/order/contiguous iteration dominates.
- **Strong 3-Year-Engineer Answer:** I start from invariants and access patterns. TArray is excellent for ordered iteration and often cache-friendly linear scans. TSet/TMap use hashing for expected constant-time lookup but require correct equality/hash and do not promise stable iteration order or element addresses. Small collections can still favour a linear array due to locality, so I profile representative sizes rather than selecting by asymptotic notation alone.
- **Common Weak Answer:** “Use maps and sets whenever lookup must be fast.”
- **Follow-up Questions:** What makes a valid key? Is iteration deterministic? What happens to element pointers on growth?
- **Hands-on Verification Task:** Benchmark lookup and iteration for representative small and large data, including hashing cost.
- **Sources:** [SRC-EPIC-024], [SRC-EPIC-025], [SRC-EPIC-026]
- **Version Notes:** Stable UE4/UE5 concepts.

### Question: `Cast`, interface, or component lookup?

- **Category:** UE C++ Idioms / Architecture
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Cast for a legitimate runtime type boundary, interface for a shared capability contract, and component lookup for composed capability owned by an Actor.
- **Strong 3-Year-Engineer Answer:** One checked Cast at a boundary is clear and returns null on incompatibility. Repeated casts across many concrete types suggest the caller depends on implementation classes; I would consider a UInterface, Actor Component, tag/query, or registry. Interfaces express behaviour without promising shared state; Components package reusable state/behaviour. I avoid abstraction theatre when one concrete relationship is actually the invariant.
- **Common Weak Answer:** “Casting is bad; always use interfaces.”
- **Follow-up Questions:** `CastChecked`? `IsA`? ExactCast? Interface execution from C++?
- **Hands-on Verification Task:** Refactor a repeated cast chain into an interface and compare dependencies and call-site clarity.
- **Sources:** [SRC-EPIC-027]
- **Version Notes:** Stable UE4/UE5; target API checked for UE5.6.

### Question: `NewObject`, `SpawnActor`, or `CreateDefaultSubobject`?

- **Category:** UE C++ Idioms / Lifecycle
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** NewObject creates runtime UObjects, SpawnActor creates world Actors, and CreateDefaultSubobject defines constructor-time default subobjects.
- **Strong 3-Year-Engineer Answer:** The API selects a lifecycle contract, not just allocation. NewObject supplies Outer/identity and requires an appropriate retaining reference. SpawnActor routes Actor construction, world registration, spawn collision policy, construction scripts, and BeginPlay. CreateDefaultSubobject is constructor/CDO structure. For data required before construction finishes, I use deferred Actor spawn and always complete it with FinishSpawning.
- **Common Weak Answer:** “Use NewObject for classes starting with U and SpawnActor for A.”
- **Follow-up Questions:** What happens if deferred spawn is not finished? Runtime Components? Why not `new`?
- **Hands-on Verification Task:** Trace lifecycle logs for each creation route and one deferred Actor.
- **Sources:** [SRC-EPIC-003], [SRC-EPIC-004], [SRC-EPIC-015]
- **Version Notes:** Stable UE4/UE5; exact hook ordering is version-sensitive.

### Question: Native, multicast, or dynamic delegate?

- **Category:** UE C++ Idioms / Delegates
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Choose single versus multicast from listener cardinality and dynamic versus native from reflection/Blueprint/serialisation requirements.
- **Strong 3-Year-Engineer Answer:** Internal C++ callbacks use native delegates unless reflection is required. Multicast is for notifications to several listeners and normally has no return value. Dynamic multicast is appropriate for BlueprintAssignable events but uses reflection and is slower. Binding choice carries lifetime: raw bindings require explicit proof/unbinding; UObject/shared/weak-aware bindings guard the matching lifetime domain. I store delegate handles and unbind symmetrically.
- **Common Weak Answer:** “Dynamic multicast is best because it supports everything.”
- **Follow-up Questions:** `AddRaw` risk? `AddWeakLambda`? Why no multicast return value? Duplicate bindings?
- **Hands-on Verification Task:** Implement native and Blueprint-assignable health events, then test listener destruction and duplicate BeginPlay binding.
- **Sources:** [SRC-EPIC-028], [SRC-EPIC-029]
- **Version Notes:** Stable UE4/UE5 concepts; macro overload sets are version-sensitive.

### Question: What belongs in C++ versus Blueprint?

- **Category:** C++ / Blueprint Architecture
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Put stable invariants, authority, reusable algorithms, complex lifetime, and proven hotspots in C++; use Blueprint/data for designer-owned composition, tuning, presentation, and bounded extension.
- **Strong 3-Year-Engineer Answer:** I design a narrow C++ API rather than splitting by percentage. Damage validation, authority, replication, and state mutation are C++ invariants. Designers tune DataAssets and implement hit presentation through events/Blueprint children. Blueprint reads state and requests validated actions instead of writing fields freely. I keep event frequency visible, profile before migration, and test packaged asset behaviour.
- **Common Weak Answer:** “C++ is fast and Blueprint is for simple things.”
- **Follow-up Questions:** Can production gameplay live in Blueprint? How do you migrate a hotspot? What should designers own?
- **Hands-on Verification Task:** Document and defend every C++/Blueprint boundary in Project 1.
- **Sources:** [SRC-EPIC-030], [SRC-EPIC-031]
- **Version Notes:** UE5.3–UE5.6; performance conclusions require project profiling.

### Question: `BlueprintImplementableEvent` versus `BlueprintNativeEvent`?

- **Category:** C++ / Blueprint Architecture
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** ImplementableEvent has no native implementation; NativeEvent provides a native `_Implementation` fallback that Blueprint may override.
- **Strong 3-Year-Engineer Answer:** I use ImplementableEvent for optional extension/presentation where doing nothing is safe. I use NativeEvent when a sensible native default is required. Mandatory invariants should not depend on a Blueprint override calling parent; I keep them in a non-overridable public operation and call a bounded extension hook after validation/state mutation.
- **Common Weak Answer:** “NativeEvent is ImplementableEvent but faster.”
- **Follow-up Questions:** How does Blueprint call parent? Where is `_Implementation` declared? What if the override is absent?
- **Hands-on Verification Task:** Build a native damage wrapper with an optional Blueprint presentation event and a separate native-default policy hook.
- **Sources:** [SRC-EPIC-031]
- **Version Notes:** Stable UE4/UE5.

### Question: What does `BlueprintPure` promise, and what performance trap can it create?

- **Category:** C++ / Blueprint / Performance
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It promises no observable state mutation and removes exec pins; consumers may cause repeated evaluation, so expensive pure queries can run more often than expected.
- **Strong 3-Year-Engineer Answer:** Pure is semantic, not a performance annotation. I use it for cheap deterministic accessors/calculations. An expensive world search marked pure hides scheduling and can be reevaluated for each use. I cache at an intentional lifetime or expose an impure scheduled node when cost matters, then confirm frequency with Blueprint debugging and Insights.
- **Common Weak Answer:** “Pure functions are optimised and run once.”
- **Follow-up Questions:** Can a const function still be expensive? When should results be cached? Why is hidden work dangerous in graphs?
- **Hands-on Verification Task:** Count calls to a pure node connected to several consumers and compare an explicitly cached version.
- **Sources:** [SRC-EPIC-031]
- **Version Notes:** Stable Blueprint execution principle; verify compiler behaviour in target version for edge cases.

### Question: Why prefer Blueprint read-only state plus validated functions over `BlueprintReadWrite`?

- **Category:** C++ / Blueprint Architecture
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Controlled mutation preserves invariants, authority, notifications, replication, and debugging visibility.
- **Strong 3-Year-Engineer Answer:** A writable health field allows any graph to bypass clamping, authority, death transitions, analytics, and OnRep/event flow. I expose read-only state and a request/mutation function that validates context and performs all side effects. Editor editability, Blueprint readability, and runtime mutation are separate policies, so I choose specifiers independently.
- **Common Weak Answer:** “Private variables are cleaner.”
- **Follow-up Questions:** `EditDefaultsOnly` versus BlueprintReadOnly? How should UI update? Where does server authority live?
- **Hands-on Verification Task:** Replace a writable health property with validated server mutation and event-driven UI.
- **Sources:** [SRC-EPIC-007], [SRC-EPIC-031]
- **Version Notes:** Stable UE4/UE5 architecture guidance.

## Enhanced Input

### Question: How would you structure input in a modern UE5 project?

- **Category:** Enhanced Input / Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Use semantic Input Actions, small mode-specific Mapping Contexts on the correct LocalPlayer, cheap modifiers/triggers, and an adapter from local intent to gameplay/UI/ability/server routes.
- **Strong 3-Year-Engineer Answer:** I separate physical devices from semantic intent. Input Actions are things the user means to do, such as Move, Fire or Confirm. Mapping Contexts represent modes like common gameplay, on-foot, vehicle, menu or radial wheel and are added to the correct Enhanced Input Local Player Subsystem with explicit priorities and idempotent lifecycle. Modifiers handle local value shaping such as swizzle, dead zone, sensitivity and inversion; triggers express press, hold, tap, release and blockers. Handlers route intent to Pawn/components, UI or an ability adapter, while the server still validates durable gameplay mutation.
- **Common Weak Answer:** “Create Input Actions and bind them in the Character.”
- **Follow-up Questions:** Where do contexts live? What owns remapping? What changes for split-screen?
- **Hands-on Verification Task:** Build common/on-foot/menu/vehicle contexts and prove add/remove/priority logs across possession and menu open/close.
- **Sources:** [SRC-INPUT-001], [SRC-INPUT-004], [SRC-INPUT-005]
- **Version Notes:** UE5.3–UE5.6 target; Enhanced Input settings/API are branch/project sensitive.

### Question: Input Action versus Input Mapping Context?

- **Category:** Enhanced Input / Concepts
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** An Input Action is semantic intent/value; a Mapping Context binds physical controls to actions for one local-player mode or layer.
- **Strong 3-Year-Engineer Answer:** `IA_Fire` or `IA_Move` describes what gameplay/UI wants, including value type such as Boolean or Axis2D. A Mapping Context says which key, button, stick or touch input currently drives that action, plus modifiers/triggers. Contexts can be added, removed and prioritised per LocalPlayer, so the same physical button can mean Jump in gameplay and Confirm in UI without both firing if ownership is designed correctly.
- **Common Weak Answer:** “Actions are for buttons; contexts are groups of actions.”
- **Follow-up Questions:** Why not name actions after keys? How do priorities resolve collisions? Where do user remaps fit?
- **Hands-on Verification Task:** Rename one device-named action into semantic action names and support keyboard plus gamepad without changing gameplay code.
- **Sources:** [SRC-INPUT-001], [SRC-INPUT-002], [SRC-INPUT-003]
- **Version Notes:** Stable UE5 Enhanced Input concept; exact asset/API fields vary.

### Question: Input Modifier versus Input Trigger?

- **Category:** Enhanced Input / Concepts
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** A modifier transforms the input value; a trigger decides whether the modified value activates the action and which event state is emitted.
- **Strong 3-Year-Engineer Answer:** Modifiers are value processors: dead zone, axis swizzle, negate, sensitivity, smoothing, inversion or local/world conversion. Triggers are activation rules: press, release, hold, tap, chord, blocker or custom conditions. I keep both cheap and side-effect-free; gameplay validation belongs in gameplay/authority code. Bugs often come from using a trigger when context removal is clearer, or using a modifier to read/mutate heavy gameplay state.
- **Common Weak Answer:** “Modifiers change input; triggers are for special buttons.”
- **Follow-up Questions:** Where would you put aim inversion? Hold-to-charge? Chorded input? Server validation?
- **Hands-on Verification Task:** Implement WASD as one Axis2D action using negate/swizzle modifiers and add hold/release semantics for a charge action.
- **Sources:** [SRC-INPUT-001]
- **Version Notes:** Exact built-in modifier/trigger list is version-sensitive.

### Question: Started, Triggered, Completed and Canceled—how do you choose?

- **Category:** Enhanced Input / Trigger Events
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Choose the event that matches the semantic edge or continuous value: Started for begin edge, Triggered for satisfied/continuous actions, Completed for normal finish and Canceled for interrupted evaluation.
- **Strong 3-Year-Engineer Answer:** I do not bind every action to `Triggered` blindly. Continuous Move/Look usually uses Triggered because values update while active. A one-shot Jump or Fire press may use Started. Charge/release actions often need Started/Ongoing/Completed/Canceled so cancellation cleans UI/cosmetics and release commits at the right point. If a hold trigger fires repeatedly, the handler must be designed for that. The important question is what gameplay moment the input event represents.
- **Common Weak Answer:** “Triggered is the normal event.”
- **Follow-up Questions:** Why did fire spam every frame? How do you handle release? What if hold is cancelled?
- **Hands-on Verification Task:** Bind the same action to Started, Triggered, Completed and Canceled, log event order for press, hold, release and early cancel.
- **Sources:** [SRC-INPUT-001], [SRC-INPUT-005]
- **Version Notes:** Trigger behaviour depends on trigger configuration and branch.

### Question: How do you debug an Enhanced Input action that does not fire?

- **Category:** Enhanced Input / Debugging
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Check local player, active contexts/priorities, action asset/value type, modifiers, triggers, UI focus/input mode, binding lifecycle and network route.
- **Strong 3-Year-Engineer Answer:** I log the exact LocalPlayer, PlayerController, Pawn, active mapping contexts with priorities, action name, value, trigger event and route. Then I verify the context was added to the correct `UEnhancedInputLocalPlayerSubsystem`, the physical key is mapped, no higher-priority context or UI focus is consuming it, modifiers are not zeroing it, trigger conditions are met, the handler value type matches the action and bindings were not duplicated or skipped after possession. For multiplayer I also check that the input routes through the owning client and server validation.
- **Common Weak Answer:** “Check that the key is mapped.”
- **Follow-up Questions:** What if it works before opening a menu? What if split-screen player two fails? What if value is always zero?
- **Hands-on Verification Task:** Inject wrong LocalPlayer context, wrong trigger and wrong value type; diagnose each from logs.
- **Sources:** [SRC-INPUT-001], [SRC-INPUT-004], [SRC-UI-010]
- **Version Notes:** Debug UI/tooling differs by project; logging policy is stable.

### Question: How should remapping be designed?

- **Category:** Enhanced Input / Remapping
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Treat remapping as per-user/local-player settings with conflict policy, device/platform awareness, persistence, UI prompt updates and fallback defaults.
- **Strong 3-Year-Engineer Answer:** I decide which actions are player-mappable and which bindings are mode/platform-specific. The save format should be user preference data, not an accidental dump of gameplay assets. On apply, I validate conflicts within relevant contexts, preserve required Confirm/Cancel/accessibility paths, update glyphs/prompts and handle missing devices. In split-screen, settings are per LocalPlayer. I also need migration/default-restore behaviour when actions are renamed or DLC/plugins add/remove actions.
- **Common Weak Answer:** “Let players choose a new key for each action.”
- **Follow-up Questions:** How handle conflicts? How update button glyphs? What if two contexts use the same key intentionally?
- **Hands-on Verification Task:** Build a remap screen that rejects same-context conflicts but allows mode-specific reuse, then persist separately for two local players.
- **Sources:** [SRC-INPUT-001], [SRC-INPUT-004], [SRC-UI-011]
- **Version Notes:** Built-in user-settings/remapping APIs are branch/project sensitive.

### Question: How should Enhanced Input coordinate with UI or CommonUI?

- **Category:** Enhanced Input / UI
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Use one input coordinator so UI activation/focus and gameplay mapping contexts do not compete; menu contexts should block or replace gameplay contexts intentionally.
- **Strong 3-Year-Engineer Answer:** CommonUI can own activatable screen focus and input mode; Enhanced Input owns semantic action mappings. I avoid direct PlayerController input-mode calls, viewport calls and mapping-context changes from multiple systems fighting each other. Opening a modal should add or request the right UI context, focus the intended widget, prevent gameplay leakage and restore prior state on close. I test Confirm/Cancel, popup stacks, hidden focus targets, device changes and local-player identity.
- **Common Weak Answer:** “When UI opens, disable player input.”
- **Follow-up Questions:** What if Jump and Confirm share a button? Who restores focus? What happens when all screens deactivate?
- **Hands-on Verification Task:** Open a modal while holding movement/fire and prove only UI Confirm/Cancel routes until it closes.
- **Sources:** [SRC-INPUT-001], [SRC-UI-010], [SRC-UI-011]
- **Version Notes:** CommonUI is plugin-dependent; integration with Enhanced Input is target-version sensitive.

### Question: How does Enhanced Input fit with GAS?

- **Category:** Enhanced Input / GAS
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Enhanced Input emits semantic local intent; an adapter maps actions/tags to ability spec handles and press/release events while ASC/server validation owns gameplay truth.
- **Strong 3-Year-Engineer Answer:** Abilities should not know the physical key. I map input actions such as AbilityPrimary or AbilitySlot1 to granted specs, handles or tags. The adapter rebuilds idempotently when equipment, grants, Pawn/avatar or PlayerState-owned ASC changes. Press/release/hold semantics must match the ability design, and duplicate bindings after respawn must be prevented. GAS still checks tags, cost, cooldown, target, prediction policy and authority; input is just the request path.
- **Common Weak Answer:** “Bind each key directly to ActivateAbility.”
- **Follow-up Questions:** ASC on PlayerState? Release events? Loadout changes? Duplicate bindings after respawn?
- **Hands-on Verification Task:** Respawn a Pawn with PlayerState-owned ASC and prove one press creates exactly one activation and one release path.
- **Sources:** [SRC-INPUT-001], [SRC-GAS-002], [SRC-GAS-010], [SRC-GAS-013]
- **Version Notes:** GAS and Enhanced Input integration is project-specific and branch-sensitive.

### Question: How does input become network-safe gameplay?

- **Category:** Enhanced Input / Networking
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Convert local input into semantic intent, then route it through prediction or an owned server-validated request; never treat the local key event as authority.
- **Strong 3-Year-Engineer Answer:** The client can sample input and present immediate feedback, but durable state belongs to the server. For interaction/fire/ability requests, I send compact semantic intent with target/time/sequence as needed through an owning route and validate server-side. For movement-affecting input, I use the CharacterMovement prediction/saved-move path rather than delayed replicated bools. I avoid reliable RPC spam for high-frequency input; state replication, unreliable events, batching or specialised prediction are better paths depending on the system.
- **Common Weak Answer:** “Input happens on the client, so call a Server RPC from the input event.”
- **Follow-up Questions:** What validates fire? What belongs in saved moves? Why not reliable RPC every Tick?
- **Hands-on Verification Task:** Implement local fire feedback plus authoritative server validation and compare with a bad client-authoritative version.
- **Sources:** [SRC-INPUT-001], [SRC-NET-004], [SRC-NET-009], [SRC-NET-018]
- **Version Notes:** Generic authority model stable; movement prediction details are branch-sensitive.

### Question: Why can input fire twice after respawn or screen reopen?

- **Category:** Enhanced Input / Debugging
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Binding or context lifecycle is not idempotent; old bindings/contexts remain while new ones are added.
- **Strong 3-Year-Engineer Answer:** Respawn, possession change, widget reconstruction and ability grant changes are all rebinding hazards. I check whether `SetupPlayerInputComponent` ran again, whether a component or UI subscribed twice, whether a Mapping Context was added repeatedly and whether release/cancel paths were unbound. The fix is owned binding handles or a clear rebuild phase, idempotent context application, unbind/remove on teardown and logs that prove one active route per action/local player.
- **Common Weak Answer:** “The input event is buggy.”
- **Follow-up Questions:** How do widgets differ between Construct and activation? How would you log duplicate routes? What if ASC lives on PlayerState?
- **Hands-on Verification Task:** Deliberately bind fire twice after respawn, detect duplicate handler calls and repair with explicit rebuild ownership.
- **Sources:** [SRC-INPUT-001], [SRC-INPUT-005], [SRC-UI-002], [SRC-GAS-002]
- **Version Notes:** Exact binding handle APIs and widget lifecycle surfaces vary; idempotent ownership principle is stable.

## Networking and Replication

### Question: Authority versus ownership—what is the difference?

- **Category:** Networking / Replication
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** The server has authority over replicated truth; ownership links an Actor to a PlayerController/connection and controls RPC routing, conditions, and owner relevance.
- **Strong 3-Year-Engineer Answer:** Authority answers who decides replicated state: the server. Ownership answers which client connection is associated with the Actor. An owning client may have an autonomous proxy and submit valid Server RPC intent, but the server validates and commits the result. Ownership is not attachment, Outer, possession, or local role, though possession commonly establishes relevant ownership for a Pawn.
- **Common Weak Answer:** “The owning client has authority over its Pawn.”
- **Follow-up Questions:** Autonomous versus simulated proxy? Who can call Server RPCs? Does attachment transfer ownership?
- **Hands-on Verification Task:** Log authority, role, Owner, and owning PlayerController for server, owner client, and remote client.
- **Sources:** [SRC-NET-002], [SRC-NET-003]
- **Version Notes:** Stable UE5 client/server model; target UE5.3–UE5.6.

### Question: Why does a Server RPC fail to execute?

- **Category:** Networking / RPC
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Common causes are an unreplicated/unestablished Actor, invalid owning connection, wrong call side/timing, irrelevance/dormancy/channel state, or declaration/implementation error.
- **Strong 3-Year-Engineer Answer:** I first reproduce on a remote client and log net mode, role, Actor owner, PlayerController, and replication state. A client can normally call a Server RPC only through an Actor it owns via its connection; having a pointer to a world Actor is insufficient. I confirm the Actor was server-spawned, replicated, relevant/channel-established, and the RPC signature is correct. I route world-target intent through the owned Pawn/Controller and validate the target on server.
- **Common Weak Answer:** “Make the RPC Reliable.”
- **Follow-up Questions:** Why does it work on listen host? Can a Component call RPCs? When is BeginPlay too early?
- **Hands-on Verification Task:** Intentionally call a Server RPC on an unowned pickup, then repair the routing without changing it to always relevant.
- **Sources:** [SRC-NET-003], [SRC-NET-006]
- **Version Notes:** UE5.3–UE5.6; exact execution table should be checked for target version.

### Question: Replicated property or RPC?

- **Category:** Networking / Architecture
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Use properties for durable current state and RPCs for transient directed events/requests.
- **Strong 3-Year-Engineer Answer:** A late joiner needs the door's current open state, so that is replicated state with OnRep presentation. A replaceable impact spark can be an unreliable multicast. Client intent is a Server RPC; authoritative outcome becomes replicated state. Reliable RPCs are for infrequent essential events that cannot be reconstructed, not a stream of state snapshots.
- **Common Weak Answer:** “Properties are variables and RPCs are functions.”
- **Follow-up Questions:** Death event or death state? How do late joiners catch up? Can property changes be coalesced?
- **Hands-on Verification Task:** Implement a door first with multicast events only, join late, then repair it with replicated state.
- **Sources:** [SRC-NET-001], [SRC-NET-004], [SRC-NET-006]
- **Version Notes:** Stable replication design principle.

### Question: How should an OnRep function be designed?

- **Category:** Networking / Properties
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** As a local, order-tolerant, idempotent reaction to newly applied server state—not as authoritative mutation.
- **Strong 3-Year-Engineer Answer:** I use OnRep to update presentation, UI, derived caches, and local events. I do not assume another property's OnRep ran first; related invariant fields become one replicated struct or a coordinated state transition. I handle unresolved references and compare old/new values where needed. The server invokes shared reaction explicitly after its own mutation if required rather than relying on client notification semantics.
- **Common Weak Answer:** “OnRep runs whenever the variable changes.”
- **Follow-up Questions:** Does it run on server assignment? Is callback order deterministic? Why might an Actor reference be null temporarily?
- **Hands-on Verification Task:** Split a death transition across several OnReps, reproduce ordering fragility, then combine the state.
- **Sources:** [SRC-NET-004], [SRC-NET-005]
- **Version Notes:** Target UE5.3–UE5.6; exact notify conditions are version/API-specific.

### Question: Reliable versus unreliable RPC?

- **Category:** Networking / RPC
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Reliable RPCs retransmit until acknowledged and can block later reliable traffic; unreliable RPCs may drop and have weaker ordering, fitting frequent replaceable events.
- **Strong 3-Year-Engineer Answer:** I use reliability from consequence, frequency, and recoverability. An infrequent purchase confirmation may need reliable delivery; aim updates or cosmetic impacts should tolerate loss and supersession. I never send a reliable RPC every Tick because packet loss grows a backlog and latency. Durable truth should usually be replicated state so later updates converge.
- **Common Weak Answer:** “Reliable for important data, unreliable for unimportant data.”
- **Follow-up Questions:** Can reliable traffic arrive late? Ordering across Actors? What is head-of-line blocking?
- **Hands-on Verification Task:** Spam reliable and unreliable counters under simulated loss and compare backlog, order, and final convergence.
- **Sources:** [SRC-NET-005], [SRC-NET-006]
- **Version Notes:** Stable principle; exact channel/bunch behaviour is specialist/version-sensitive.

### Question: What are autonomous and simulated proxies?

- **Category:** Networking / Roles / Movement
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** The owning client has an autonomous proxy that predicts local control; other clients normally see simulated proxies driven by replicated/interpolated state.
- **Strong 3-Year-Engineer Answer:** The server Actor is authoritative. The autonomous proxy gathers input and predicts to hide round-trip latency, then reconciles server acknowledgements/corrections. Simulated proxies do not originate that player's intent and smooth received movement. Role is not ownership itself, and non-replicated client-local Actors can have local authority without representing server truth.
- **Common Weak Answer:** “Autonomous means client authority; simulated means NPC.”
- **Follow-up Questions:** What does CharacterMovement replay? Why do direct SetActorLocation calls cause correction? What is network smoothing?
- **Hands-on Verification Task:** Observe roles for two clients and trigger a movement correction under artificial latency.
- **Sources:** [SRC-NET-002], [SRC-NET-009]
- **Version Notes:** UE5.3–UE5.6; movement implementation details evolve.

### Question: Relevancy versus dormancy?

- **Category:** Networking / Optimisation
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Relevancy decides per connection whether an Actor should replicate; dormancy keeps an Actor present but skips normal consideration while its state is stable.
- **Strong 3-Year-Engineer Answer:** A dynamically spawned Actor that becomes irrelevant can be destroyed on that client and recreated later. A dormant Actor remains on both sides but is omitted from normal replication gathering. Relevancy is audience/spatial significance; dormancy is change frequency. I wake/flush a dormant Actor before changing replicated state and avoid sleep/wake churn.
- **Common Weak Answer:** “Both stop replication to save bandwidth.”
- **Follow-up Questions:** Owner relevancy? Always relevant cost? What happens to dormant component updates?
- **Hands-on Verification Task:** Compare one distance-irrelevant pickup with one dormant objective on a remote client.
- **Sources:** [SRC-NET-007], [SRC-NET-008]
- **Version Notes:** UE5.3–UE5.6; partial dormancy/deprecation detail is version-sensitive.

### Question: Why must a dormant Actor be woken before state mutation?

- **Category:** Networking / Dormancy
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Waking/flush reinitialises replication comparison state; mutating first can make the change invisible or implementation-dependent.
- **Strong 3-Year-Engineer Answer:** Dormant Actors are skipped. Before a one-off update I call FlushNetDormancy/ForceNetUpdate as appropriate, or set Awake for sustained changes, then mutate authoritative replicated state. Components/subobjects require waking their owning Actor. I use dormancy validation/logging and never rely on an after-the-fact flush that happened to work in one case.
- **Common Weak Answer:** “ForceNetUpdate after changing it.”
- **Follow-up Questions:** Flush versus Awake? DORM_Initial? Why can churn be slower?
- **Hands-on Verification Task:** Update a dormant Fast Array/property before and after flush and compare with dormancy validation enabled.
- **Sources:** [SRC-NET-008]
- **Version Notes:** Maintained documentation; verify exact UE5.6 behaviour/source.

### Question: How would you replicate a health/damage system securely?

- **Category:** Networking / System Design
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Client submits owned intent; server validates hit/state/rate/resources, mutates authoritative health, and replicates resulting state with OnRep presentation.
- **Strong 3-Year-Engineer Answer:** The client never sends trusted final damage. It sends compact action intent/evidence through an owned Actor. The server checks caller ownership, cooldown/ammo, target validity, distance/trace or game-specific lag compensation, then applies damage. Health replicates to the required audience; private combat details use conditions. Death is durable state for late joiners; cosmetics can be unreliable events. I test cheating parameters, packet loss, respawn, and two clients.
- **Common Weak Answer:** “Make ApplyDamage a reliable Server RPC and replicate Health.”
- **Follow-up Questions:** Hitscan lag compensation? Owner-only health? Prediction? Duplicate requests?
- **Hands-on Verification Task:** Attempt invalid damage values/ranges from a modified client and prove server rejection.
- **Sources:** [SRC-NET-001], [SRC-NET-003], [SRC-NET-004]
- **Version Notes:** Architecture stable; anti-cheat/lag compensation specifics are game-dependent.

### Question: Why is OnRep order not a safe protocol?

- **Category:** Networking / Ordering
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Different replicated variables' OnRep callbacks have no deterministic relative order, and RPC/property ordering has limited context-specific guarantees.
- **Strong 3-Year-Engineer Answer:** If fields form one invariant, I replicate them together in a struct/state object or delay reaction until all required generation/version data is present. I do not make `OnRep_Mode` depend on `OnRep_Target` having executed first. Across Actors, RPC call order is not globally preserved, so cross-Actor protocols use explicit IDs/sequences/state readiness.
- **Common Weak Answer:** “Declare the properties in the right order.”
- **Follow-up Questions:** Ordering on one Actor? Reliable RPC ordering? PostRepNotifies?
- **Hands-on Verification Task:** Build two dependent OnReps, introduce packet loss, then repair with one versioned struct.
- **Sources:** [SRC-NET-005]
- **Version Notes:** Exact execution details must be confirmed for target minor version.

### Question: When should you consider Replication Graph or Iris?

- **Category:** Networking / Scalability
- **Priority:** P2
- **Expected Depth:** D2
- **Short Answer:** Consider them when baseline profiling and project constraints show a need for scalable relevance routing or newer replication architecture; neither is automatic merely because the project uses UE5.
- **Strong 3-Year-Engineer Answer:** I first know which system the project uses. Replication Graph addresses large Actor/connection relevance gathering through graph nodes and spatial/policy routing. Iris provides a newer descriptors/filtering/serialisation framework with version/configuration implications. I profile baseline Actor consideration, bandwidth, and server time, then evaluate migration cost and feature compatibility instead of adopting by fashion.
- **Common Weak Answer:** “Iris replaces replication in UE5.”
- **Follow-up Questions:** What problem does RepGraph solve? Is dormancy still relevant? What would you benchmark?
- **Hands-on Verification Task:** Record baseline relevant Actor count/cost and propose a spatial graph partition; prototype only if justified.
- **Sources:** [SRC-NET-001], [SRC-NET-008]
- **Version Notes:** Version/configuration-sensitive; UE5.3–UE5.6 capabilities differ.

### Question: How do you profile network replication?

- **Category:** Networking / Profiling
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Capture representative multi-client traffic and rank Actors, properties, and RPCs by bytes/frequency while correlating server processing and player-visible latency.
- **Strong 3-Year-Engineer Answer:** I set target player/Actor counts and packet conditions, then use target-version Networking Insights/Network Profiler plus server timing. I find high-frequency reliable RPCs, always-relevant Actors, oversized properties, poor update rates, and Actors that should be dormant. I change one audience/frequency/representation policy, rerun the same scenario, and verify gameplay under loss—not just total bandwidth on localhost.
- **Common Weak Answer:** “Use stat net and reduce NetUpdateFrequency.”
- **Follow-up Questions:** Bandwidth versus server CPU? Why can lower frequency hurt feel? What does Fast Array optimise?
- **Hands-on Verification Task:** Capture Project 3 before/after dormancy and owner-only conditions with identical clients and route.
- **Sources:** [SRC-NET-008], [SRC-NET-010]
- **Version Notes:** Tool availability/UI varies by engine version and build configuration.

## AI and Navigation

### Question: What are the responsibilities of AIController and Pawn?

- **Category:** AI / Gameplay Framework
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** AIController owns decision/control policy; Pawn/Character owns the physical agent and reusable gameplay capabilities.
- **Strong 3-Year-Engineer Answer:** The Controller receives perception/world input, runs decision logic, and issues commands. The Pawn provides movement, combat, health, animation, and world representation through components. This separation supports possession changes, respawn, and shared capabilities between AI/human control. I avoid putting weapon mechanics directly inside BT Tasks; tasks request a stable Pawn/component action.
- **Common Weak Answer:** “AIController runs the Behaviour Tree and Pawn moves.”
- **Follow-up Questions:** Where should Perception live? What runs on clients? When does OnPossess start the tree?
- **Hands-on Verification Task:** Drive one compatible Pawn alternately with AIController and PlayerController using the same combat component.
- **Sources:** [SRC-AI-001], [SRC-EPIC-010]
- **Version Notes:** Stable UE4/UE5.

### Question: Task, Service, or Decorator in a Behaviour Tree?

- **Category:** AI / Behaviour Tree
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Tasks perform actions, Decorators gate/observe branches, and Services periodically update context while a branch is active.
- **Strong 3-Year-Engineer Answer:** Leaves should usually be actions. Conditions belong in Decorators so branch availability and observer aborts are visible. Services are for bounded periodic updates, not expensive every-frame searches or latent actions. A Running Task must finish exactly once and clean up on abort. Blackboard keys expose decision memory but should have clear writers and lifecycle.
- **Common Weak Answer:** “Tasks do things, Services run in background, Decorators are if statements.”
- **Follow-up Questions:** Can Services run off-thread? How does a latent Task finish? What state can a native Task store?
- **Hands-on Verification Task:** Implement one latent move/attack Task and prove success, failure, and abort cleanup.
- **Sources:** [SRC-AI-002]
- **Version Notes:** UE5.3–UE5.6; native node instancing/task-memory details are specialist depth.

### Question: What do Observer Aborts do?

- **Category:** AI / Behaviour Tree
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** They let observed condition changes abort the current branch and/or interrupt lower-priority branches, enabling event-driven replanning.
- **Strong 3-Year-Engineer Answer:** `Self` stops a branch when its condition becomes invalid. `Lower Priority` lets a newly valid higher-priority branch interrupt work to its right/lower priority. `Both` combines these semantics. This avoids polling the whole tree every frame, but the aborted task must cancel movement, delegates, or gameplay work correctly.
- **Common Weak Answer:** “They restart the tree when a Blackboard value changes.”
- **Follow-up Questions:** Why does left-to-right order matter? Which mode supports Patrol interrupted by Combat? What is event-driven about UE BTs?
- **Hands-on Verification Task:** Build Patrol/Combat branches and observe exact transitions for target acquired/lost.
- **Sources:** [SRC-AI-002]
- **Version Notes:** Stable UE Behaviour Tree concept.

### Question: Behaviour Tree versus StateTree versus FSM?

- **Category:** AI / Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** BTs excel at reactive priority/action selection, StateTree at hierarchical state/transition logic with selectors, and FSMs at small explicit modal systems.
- **Strong 3-Year-Engineer Answer:** I choose the smallest model matching semantics and team tooling. A three-mode door needs no BT. Reactive NPC action priority fits BT and its mature AI debugger. A system whose enter/exit state and hierarchical transitions dominate can fit StateTree. StateTree is UE5-era and configuration/version-sensitive, not a universal replacement. Mechanics remain behind stable components regardless of decision model.
- **Common Weak Answer:** “StateTree is the new faster Behaviour Tree.”
- **Follow-up Questions:** Can they compose? What causes FSM transition explosion? Where does Blackboard-like data live?
- **Hands-on Verification Task:** Model the same patrol/chase/attack policy in all three and compare transitions, interrupts, and debugging.
- **Sources:** [SRC-AI-002], [SRC-AI-003]
- **Version Notes:** StateTree UE5-specific/plugin-dependent; verify UE5.6 project configuration.

### Question: How should AI Perception memory be handled?

- **Category:** AI / Perception
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Separate current perception from known/not-forgotten stimuli and explicit last-known-location/alert memory with age/decay policy.
- **Strong 3-Year-Engineer Answer:** A failed sight stimulus means sight changed; it does not necessarily invalidate hearing, damage knowledge, or investigation memory. I store stimulus sense, success, location, time/age, and target identity, then apply game-specific forgetting/alert decay. I configure Max Age deliberately—zero can mean never forgotten—and inspect Perception debug data before changing decision logic.
- **Common Weak Answer:** “Set the Blackboard target on seen and clear it on lost.”
- **Follow-up Questions:** Currently perceived versus known? Stimuli source registration? Affiliation/team filtering?
- **Hands-on Verification Task:** Implement see, lose sight, investigate, hear target, reacquire, and timed forget transitions.
- **Sources:** [SRC-AI-004]
- **Version Notes:** UE5.6 source; team/Blueprint limitations are version-sensitive.

### Question: How does EQS work?

- **Category:** AI / EQS
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** A Generator creates candidate items, Contexts define reference frames, Tests filter and score candidates, and the run mode returns suitable result(s).
- **Strong 3-Year-Engineer Answer:** EQS answers a spatial/environmental query, not the whole decision. Hard requirements such as reachability filter; preferences such as ideal distance score. I control candidate count and run frequency, place cheap rejecting tests before traces/path tests where applicable, and inspect normalised scores/results in the EQS debugger. Results are valid for a time window, not eternal truth.
- **Common Weak Answer:** “EQS finds the best point.”
- **Follow-up Questions:** Generator versus Context? Filter versus score? Why can highest score still be wrong?
- **Hands-on Verification Task:** Build a cover query and deliberately make reachability score-only, then repair it as a filter.
- **Sources:** [SRC-AI-005], [SRC-AI-008]
- **Version Notes:** UE5.3–UE5.6; verify plugin/configuration status.

### Question: Why does MoveTo fail?

- **Category:** AI / Navigation / Debugging
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Possession, NavMesh projection, agent settings, goal/path validity, path-follow state, movement component, overlapping requests, or authority can each fail independently.
- **Strong 3-Year-Engineer Answer:** I use AI Debugger before rewriting the BT. I confirm AIController possession and active task, visualise NavMesh and project start/goal, compare agent radius/height with nav data, inspect request/path result and partial-path policy, then check path following, movement mode/speed, root motion/physics, and server authority. I distinguish no path from path not being followed.
- **Common Weak Answer:** “Rebuild the NavMesh and increase acceptance radius.”
- **Follow-up Questions:** Partial paths? Dynamic obstacles? Nav link? Request replacement?
- **Hands-on Verification Task:** Create and diagnose six distinct MoveTo failure cases.
- **Sources:** [SRC-AI-006], [SRC-AI-008]
- **Version Notes:** UE5.3–UE5.6; exact result enums/APIs target-sensitive.

### Question: Pathfinding versus avoidance?

- **Category:** AI / Navigation
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Pathfinding computes a route through navigable static space; avoidance locally adjusts velocity around moving agents/obstacles while following it.
- **Strong 3-Year-Engineer Answer:** NavMesh graph search produces a path corridor using polygon costs. RVO or Detour Crowd addresses local dynamic interactions without global replanning for every encounter. UE's RVO is CharacterMovement-based and can steer outside nav bounds; Detour Crowd adds corridor/topology optimisation and capacity settings. I enable one system, not both, then profile crowd scale and deadlocks.
- **Common Weak Answer:** “NavMesh handles obstacles and RVO smooths the path.”
- **Follow-up Questions:** Why not recalculate path every frame? RVO versus Detour? Doorway congestion?
- **Hands-on Verification Task:** Run crossing crowds with no avoidance, RVO, and Detour; record failure modes and cost.
- **Sources:** [SRC-AI-006], [SRC-AI-007]
- **Version Notes:** Maintained page may show later UE5; verify target settings.

### Question: Explain A* at game-interview depth.

- **Category:** AI / Algorithms
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** A* expands nodes by `f=g+h`, combining known path cost with a heuristic estimate to the goal.
- **Strong 3-Year-Engineer Answer:** With non-negative edges and an admissible/consistent heuristic under the relevant formulation, A* can preserve optimality while exploring fewer nodes than Dijkstra. Better informed heuristics reduce work; weighted/non-admissible variants trade optimality for speed. In UE navigation the graph is commonly NavMesh polygons/costs, after which path following and avoidance solve different problems.
- **Common Weak Answer:** “A* is Dijkstra plus straight-line distance and is always fastest.”
- **Follow-up Questions:** Admissible heuristic? Why squared distance may be problematic as h? BFS versus Dijkstra?
- **Hands-on Verification Task:** Implement A* on a weighted grid and compare zero, Manhattan, and weighted heuristics.
- **Sources:** [SRC-AI-006]
- **Version Notes:** Algorithmic synthesis; exact UE Recast/Detour implementation is specialist/source-dependent.

### Question: How do you profile and LOD AI?

- **Category:** AI / Performance
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Measure per-layer work—decision, perception, EQS, navigation, movement—and reduce frequency/fidelity by significance rather than disabling arbitrary code.
- **Strong 3-Year-Engineer Answer:** I count active agents, BT services/tasks, perception listeners/source pairs, EQS candidates/traces, path requests, nav rebuilds, and movement/animation cost. I replace known-state polling with events, stagger updates, shrink sensing/query scope, filter EQS cheaply, and assign distance/significance tiers. Distant agents may think/query less often or move to Mass/aggregate simulation while preserving important state.
- **Common Weak Answer:** “Increase BT service intervals and use simpler AI far away.”
- **Follow-up Questions:** How prevent update spikes? What would you measure first? When use MassEntity?
- **Hands-on Verification Task:** Scale Project 2 to increasing counts and produce per-layer before/after traces.
- **Sources:** [SRC-AI-002], [SRC-AI-004], [SRC-AI-005], [SRC-AI-008]
- **Version Notes:** Workload/platform dependent; no universal interval is prescribed.

## Smart Objects and StateTree

### Question: What problem do Smart Objects solve?

- **Category:** AI / Smart Objects / Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Smart Objects let world objects advertise usable affordances and slots so agents can query, claim, use and release interactions without hardcoding every object type into every agent.
- **Strong 3-Year-Engineer Answer:** I think of a Smart Object as a world affordance contract. A chair, cover point or repair terminal exposes slots, tags, requirements and behavior data. An agent decides it wants an activity, queries candidates, claims one slot, reaches it, executes behavior and releases it on every success/failure/abort path. Movement, animation, ability rules and authority remain separate systems.
- **Common Weak Answer:** “Smart Objects are interactable Actors for AI.”
- **Follow-up Questions:** What is a slot? Why is found not claimed? What owns gameplay effects?
- **Hands-on Verification Task:** Build rest, cover and terminal affordances and log query, claim, use and release.
- **Sources:** [SRC-AI-011], [SRC-AI-012], [SRC-AI-013], [SRC-AI-014], [SRC-AI-015]
- **Version Notes:** UE5 plugin/project-sensitive; verify exact API and feature status in target branch.

### Question: Describe the Smart Object lifecycle.

- **Category:** AI / Smart Objects / Lifecycle
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Query candidates, claim a specific slot, navigate/validate, use the behavior, then release the claim on completion, failure or abort.
- **Strong 3-Year-Engineer Answer:** I separate every phase because each can fail independently. Query returns possible candidates; claim reserves one slot under current rules; reaching validates nav and movement; use runs behavior/animation/ability work; release clears ownership and updates state. I release on MoveTo failure, StateTree failure, interruption, death, despawn, object disabled/destroyed and server rejection.
- **Common Weak Answer:** “Find a Smart Object and run the interaction.”
- **Follow-up Questions:** What if two agents pick the same object? What if the object is destroyed mid-use? Where does cooldown live?
- **Hands-on Verification Task:** Inject five failure paths and prove no slot remains permanently claimed.
- **Sources:** [SRC-AI-010], [SRC-AI-012], [SRC-AI-015]
- **Version Notes:** Claim/use/release API details are branch-sensitive.

### Question: How does StateTree execute and why is it useful for Smart Object behavior?

- **Category:** AI / StateTree
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** StateTree selects a root-to-leaf active state path, runs tasks for active states, and uses transitions to reselect states; this fits explicit Acquire/Reach/Use/Exit interaction sequences.
- **Strong 3-Year-Engineer Answer:** StateTree combines hierarchical state-machine structure with selector-like state selection. Enter Conditions select a leaf, all states from root to leaf become active, and tasks run for active states. Transitions can react to task completion/failure or monitored conditions. Smart Object interactions benefit because claim, reach, use, fail and cleanup can be named states with visible transitions and shared context.
- **Common Weak Answer:** “StateTree is a faster Behavior Tree.”
- **Follow-up Questions:** What runs concurrently? How are transitions evaluated? What does context data provide?
- **Hands-on Verification Task:** Build `Acquire -> Reach -> Use -> Release` and show active state path and transition reason for success and failure.
- **Sources:** [SRC-AI-003], [SRC-AI-010]
- **Version Notes:** StateTree is UE5-era and plugin/project-sensitive.

### Question: How do Parameters, Context Data, Evaluators and Global Tasks differ in StateTree?

- **Category:** AI / StateTree / Data Flow
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Parameters configure a tree instance, Context Data comes from the execution environment, Evaluators expose runtime-derived data, and Global Tasks run for the tree lifetime.
- **Strong 3-Year-Engineer Answer:** I use parameters for externally supplied configuration, context data for owner/user/object identity, evaluators for observed runtime facts, and global tasks for persistent tree-lifetime work. In a Smart Object StateTree, context may include the user Actor and Smart Object used. I avoid hiding ownership state in a task output if cleanup in another branch depends on it.
- **Common Weak Answer:** “They are all variables you can bind.”
- **Follow-up Questions:** Which data can Enter Conditions bind to? What can a task bind to? Why can wrong context break reuse?
- **Hands-on Verification Task:** Trace one slot identity from claim through Reach and Use without using a global singleton.
- **Sources:** [SRC-AI-010]
- **Version Notes:** Exact Blueprint/C++ extension APIs are version-sensitive.

### Question: How would you debug two NPCs using the same Smart Object slot?

- **Category:** AI / Smart Objects / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Prove whether both only found the same candidate or both actually claimed it, then inspect claim ownership, release timing, authority and slot alignment.
- **Strong 3-Year-Engineer Answer:** I log query result, claim success/failure, slot identity, claimant, claim handle lifetime, StateTree active state, release path and server/client role. If claim ownership is correct but visuals overlap, I inspect slot transforms, approach offsets, movement acceptance, animation alignment and avoidance. If both claimed, I look for bypassed subsystem calls, client-only claim, stale release or duplicate runtime registration.
- **Common Weak Answer:** “Increase distance between slots.”
- **Follow-up Questions:** What if claim is server-correct but clients overlap? What if object streaming duplicates registration? How do you test race conditions?
- **Hands-on Verification Task:** Run two agents racing for one slot and capture loser behavior and release logs.
- **Sources:** [SRC-AI-008], [SRC-AI-009], [SRC-AI-010], [SRC-AI-012]
- **Version Notes:** Debugger categories and handles are branch-sensitive.

### Question: How should Smart Object interaction work in multiplayer?

- **Category:** AI / Smart Objects / Networking
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Clients can propose/preview interactions, but the server should validate and own durable claim/use outcomes for gameplay-relevant objects.
- **Strong 3-Year-Engineer Answer:** I treat a Smart Object use as gameplay state. The client can display prompts from local discovery, but the server validates object identity, slot, distance, line of sight, enabled/cooldown state, permissions, tags, inventory/team and claim availability. The client sends intent, not trusted reward or final state. Replicated state and cosmetic prediction are designed according to correction tolerance.
- **Common Weak Answer:** “Replicate the interact event to everyone.”
- **Follow-up Questions:** What is safe to predict? What does the server validate? How do late joiners see occupied objects?
- **Hands-on Verification Task:** Implement a client prompt with server-authoritative claim and rejection feedback.
- **Sources:** [SRC-AI-012], [SRC-NET-004], [SRC-NET-007]
- **Version Notes:** Replication behavior is project-specific and target-branch sensitive.

### Question: When would you choose Smart Objects plus StateTree instead of a Behavior Tree task or simple code?

- **Category:** AI / Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Choose Smart Objects plus StateTree for designer-authored world affordances with explicit interaction lifecycle; choose BT for reactive action selection and simple code for tiny local state.
- **Strong 3-Year-Engineer Answer:** If the interaction is a reusable world opportunity with slots, tags, user requirements and abort cleanup, Smart Objects plus StateTree gives a clear data and lifecycle model. If the problem is combat priority selection, BT may fit better. If it is a three-state door or one component mode, simple code is cheaper. I choose from semantics, tooling, debugging and scale rather than novelty.
- **Common Weak Answer:** “Use Smart Objects whenever AI interacts with the world.”
- **Follow-up Questions:** Can they compose? Where should mechanics live? When does the asset overhead hurt?
- **Hands-on Verification Task:** Implement one chair interaction in simple code, BT task and Smart Object StateTree; compare maintainability and failure handling.
- **Sources:** [SRC-AI-002], [SRC-AI-003], [SRC-AI-010], [SRC-AI-011]
- **Version Notes:** StateTree and Smart Objects are plugin/project-sensitive.

### Question: How do you profile Smart Object and StateTree activity systems?

- **Category:** AI / Performance
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Count agents, query frequency/radius, candidates, path validations, active StateTrees/tasks/transitions and failed-query retries, then stagger/filter/back off before optimising internals.
- **Strong 3-Year-Engineer Answer:** I first separate query cost, path validation, StateTree tick/transition cost and behavior execution. I reduce candidate count through tags and radius, avoid every-frame requery, add failed-query backoff, stagger ambient decisions, use event-driven availability where possible and profile 1/20/100-agent scenarios. For crowds, I compare Actor AI with Mass/LOD approaches rather than forcing full StateTrees on every distant agent.
- **Common Weak Answer:** “StateTree is fast, so it should be fine.”
- **Follow-up Questions:** How can monitored transitions create cost? What should be cached? How do you avoid synchronized spikes?
- **Hands-on Verification Task:** Compare synchronous versus staggered Smart Object queries for 100 agents with trace evidence.
- **Sources:** [SRC-AI-010], [SRC-AI-011], [SRC-PERF-001], [SRC-PERF-003]
- **Version Notes:** Performance depends on authored content, world scale and target platform.

## PCG and Procedural Content

### Question: What is PCG in Unreal and what is its core data flow?

- **Category:** PCG / Architecture
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** PCG is a plugin framework where spatial input flows through a PCG Graph from a PCG Component, generating and transforming points/attributes into spawned or output content.
- **Strong 3-Year-Engineer Answer:** I treat PCG as a graph-based content pipeline. A component or volume provides level/spatial input, the graph samples or creates point data, nodes filter and transform points using density and metadata attributes, and spawner/output nodes create content. I define whether generation is editor-only or runtime, expose parameters, keep seed/input contracts reproducible, and profile graph execution separately from spawned output cost.
- **Common Weak Answer:** “PCG randomly scatters meshes.”
- **Follow-up Questions:** What is a point? What owns generation? What is density?
- **Hands-on Verification Task:** Build a small biome graph and record input points, filtered points and spawned output count.
- **Sources:** [SRC-PCG-001], [SRC-PCG-005], [SRC-PCG-006], [SRC-PCG-007]
- **Version Notes:** PCG is plugin/runtime-sensitive; exact APIs and node behavior must be target-verified.

### Question: What are PCG points, density and attributes?

- **Category:** PCG / Data Model
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Points are candidate spatial records with transform/bounds/density/seed and attributes; density influences probability/weighting, and attributes carry data through the graph.
- **Strong 3-Year-Engineer Answer:** I do not treat a point as a spawned object. It is data that can be filtered, scored, transformed, matched to tables/assets, or discarded. Density is not final mesh count; downstream filters/pruning/spawners can change output. Attributes need clear names, types and metadata domains so later nodes read the intended data.
- **Common Weak Answer:** “Points are where meshes spawn and density is how many.”
- **Follow-up Questions:** Static versus dynamic attributes? How can density change but output count not? What is a seed for?
- **Hands-on Verification Task:** Add `Biome`, `SlopeClass` and `SpawnWeight` attributes and inspect them before/after filtering.
- **Sources:** [SRC-PCG-001], [SRC-PCG-007]
- **Version Notes:** Exact point fields/accessors are API-sensitive.

### Question: How do PCG metadata domains cause bugs?

- **Category:** PCG / Metadata
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** An attribute can exist on the wrong domain, such as data-level, point-level or element-level metadata, so downstream nodes may not read it.
- **Strong 3-Year-Engineer Answer:** I check whether an attribute lives on `@Data`, `@Points` or `@Elements`, and whether the downstream selector expects that domain/type/name. I use explicit prefixes, the Attributes list and inspect/debug views. Domain mistakes create graphs that look correct because a name exists somewhere but logic silently filters or matches the wrong data.
- **Common Weak Answer:** “Rename the attribute and reconnect the node.”
- **Follow-up Questions:** What does `$Position` mean? What does `@Data` mean? Why delete temporary attributes?
- **Hands-on Verification Task:** Create a graph where a point filter fails due to wrong domain, then fix it with explicit selectors.
- **Sources:** [SRC-PCG-001], [SRC-PCG-003]
- **Version Notes:** Attribute selector UI/API evolves; verify target branch.

### Question: How do graph parameters and graph instances help PCG production workflows?

- **Category:** PCG / Authoring
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Parameters expose controlled variation, and graph instances reuse graph logic with different overrides, similar in spirit to material instances.
- **Strong 3-Year-Engineer Answer:** I expose high-level controls with units/ranges: seed, density multiplier, biome type, asset set, slope threshold, exclusion tag, output policy and debug mode. Graph instances let the team reuse the same rule set in different places without copying graphs. I avoid mystery scalar parameters and document which outputs can change when a parameter changes.
- **Common Weak Answer:** “Parameters let designers tweak the graph.”
- **Follow-up Questions:** What should not be exposed? How do overrides interact with subgraphs? How do you preserve reproducibility?
- **Hands-on Verification Task:** Create two graph instances with different seeds/assets and produce reproducible before/after output reports.
- **Sources:** [SRC-PCG-001], [SRC-PCG-006]
- **Version Notes:** Exact graph parameter APIs are branch-sensitive.

### Question: How do you debug a PCG graph that produces wrong output?

- **Category:** PCG / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Freeze seed/inputs/parameters, inspect points and attributes after each phase, check metadata domains, disable nodes to isolate the first contradiction, then inspect spawner/output ownership.
- **Strong 3-Year-Engineer Answer:** I identify whether the failure is count, transform, asset choice, layer, runtime state or performance. Then I freeze the graph version, component parameters and input actors, use debug rendering/inspect/attributes list, compare point count/density/attributes after sampling/filtering/metadata/spawning, disable nodes to bisect, verify mesh weights and transform rules, and check cleanup/regeneration/cooked behavior.
- **Common Weak Answer:** “Increase density or rebuild the graph.”
- **Follow-up Questions:** How do you inspect node output? What if a debug node is editor-only? What if output duplicates after regenerate?
- **Hands-on Verification Task:** Diagnose wrong output caused by attribute-domain mismatch and duplicate regeneration.
- **Sources:** [SRC-PCG-001], [SRC-PCG-003]
- **Version Notes:** Debug UI and shipping behavior are target-sensitive.

### Question: When should PCG run at runtime?

- **Category:** PCG / Runtime Generation
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Only when runtime variation is a product requirement and the graph meets deterministic, streaming, memory, hitch, save/load and authority budgets.
- **Strong 3-Year-Engineer Answer:** I prefer editor-baked or authoring-time PCG unless runtime generation is actually required. Runtime PCG needs bounded frequency/radius, stable seeds/inputs, asset residency, async/loading policy, collision/nav implications, generated-output ownership, save/load migration and multiplayer authority decisions. I profile packaged target hardware because editor interactivity does not prove runtime safety.
- **Common Weak Answer:** “PCG supports runtime, so generate when the player gets close.”
- **Follow-up Questions:** What changes in multiplayer? How do you avoid hitches? How do World Partition/Data Layers affect this?
- **Hands-on Verification Task:** Compare editor-generated versus runtime-generated variants and record hitch/memory/output differences.
- **Sources:** [SRC-PCG-001], [SRC-PCG-002], [SRC-PCG-004], [SRC-PERF-003]
- **Version Notes:** Runtime/partition behavior is plugin and branch sensitive.

### Question: How do you profile PCG?

- **Category:** PCG / Performance
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Separate graph execution from generated output cost, then measure points, attributes, spawned actors/instances, collision/nav/render/memory and runtime hitch behavior.
- **Strong 3-Year-Engineer Answer:** I collect input size, point count at major phases, attribute count, graph execution time, spawned Actor/Component/instance count, memory, collision/nav impact, draw/GPU cost and runtime hitch duration. I compare output representations and use the same seed/inputs for before/after. If the graph is cheap but it spawns 20,000 Actors, the fix is output architecture, not a faster filter node.
- **Common Weak Answer:** “Reduce density until FPS improves.”
- **Follow-up Questions:** Graph cost versus output cost? Actor versus HISM? How can temporary attributes matter?
- **Hands-on Verification Task:** Run 1x/10x/100x output scale and report graph time separately from render/collision/memory.
- **Sources:** [SRC-PCG-001], [SRC-PCG-003], [SRC-PERF-001], [SRC-RENDER-008]
- **Version Notes:** Target platform/build and generated content type dominate results.

### Question: How should PCG-generated gameplay objects be handled?

- **Category:** PCG / Gameplay Integration
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** PCG can place candidates, but gameplay systems must validate durable state, authority, save/load, replication and invariants.
- **Strong 3-Year-Engineer Answer:** I let PCG create spatial opportunities such as cover points, spawn candidates, loot containers or Smart Objects, but gameplay systems own durable truth. Generated objects need stable IDs or reconstruction rules, validation by authority, save/load policy, cleanup on regeneration, and tests for navigation/collision/streaming. I avoid hiding economy/combat/quest rules inside a generation graph.
- **Common Weak Answer:** “Spawn gameplay actors directly from PCG and let their Blueprints handle it.”
- **Follow-up Questions:** How do you save generated objects? How do clients agree on output? What happens when graph version changes?
- **Hands-on Verification Task:** Generate cover/loot candidates with PCG but validate final use through gameplay authority.
- **Sources:** [SRC-PCG-001], [SRC-PCG-003], [SRC-NET-004], [SRC-ASSET-003]
- **Version Notes:** Multiplayer/save behavior is project-specific; PCG APIs are branch-sensitive.

## Platform Constraints and Certification Readiness

### Question: How would you prepare a UE game for mobile?

- **Category:** Platform / Mobile
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Define device tiers and budgets, configure device profiles/scalability, package clean builds, test on target devices, and prove frame, memory, thermal, input and store/package behavior.
- **Strong 3-Year-Engineer Answer:** I start with target devices, OS/toolchain versions, frame/memory/package/startup budgets and product quality goals. I set device profiles and scalability tiers, then test packaged builds on real devices for cold/warm launch, several-minute thermal behavior, hitches, memory, texture pool, input/safe area and lifecycle events. I keep SDK/signing/package requirements explicit and archive logs/symbols/manifests for every build.
- **Common Weak Answer:** “Lower the graphics settings and test on a phone.”
- **Follow-up Questions:** Thermal throttling? Device profiles versus scalability? Mobile renderer feature support?
- **Hands-on Verification Task:** Run Project 4/6 Platform Readiness Extension on two device tiers.
- **Sources:** [SRC-PLAT-001], [SRC-PLAT-002], [SRC-PLAT-005], [SRC-PLAT-006], [SRC-PERF-007], [SRC-PERF-008]
- **Version Notes:** Mobile setup, renderer and store requirements are platform/version-sensitive.

### Question: Device Profiles versus Scalability?

- **Category:** Platform / Performance
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Device Profiles select platform/device-specific CVars/defaults, while Scalability controls quality groups and tiers that trade fidelity for performance.
- **Strong 3-Year-Engineer Answer:** I use Device Profiles to choose defaults based on platform, hardware and memory buckets; scalability groups expose controlled changes to resolution, view distance, shadows, textures, effects, foliage, material quality and related settings. I prove which profile and CVars are applied in a packaged target run, and I never let low scalability remove gameplay-significant objects or information.
- **Common Weak Answer:** “Both are graphics settings.”
- **Follow-up Questions:** How do profiles inherit? What should be a profile default versus user setting? How verify applied CVars?
- **Hands-on Verification Task:** Dump applied profile/scalability CVars in two packaged runs and compare visual/gameplay acceptance.
- **Sources:** [SRC-PERF-007], [SRC-PERF-008]
- **Version Notes:** CVar names and profile selectors are branch/platform sensitive.

### Question: What is certification readiness?

- **Category:** Platform / Certification
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** It is evidence that the build handles platform-required state transitions and failure cases according to platform-holder rules, not only that gameplay works.
- **Strong 3-Year-Engineer Answer:** I build a confidential checklist from platform-holder docs and convert it into tests for boot/loading, suspend/resume, input disconnect, account/profile changes, storage errors, save corruption, network loss, overlays, entitlements, localisation/safe areas, crashes and update/DLC states. Each case has expected behavior, evidence and an owner. I do not invent public TRC/TCR/XR details; I reference the actual platform docs in the project.
- **Common Weak Answer:** “QA runs the certification checklist near the end.”
- **Follow-up Questions:** Why is suspend/resume hard? Storage failure? Controller disconnect?
- **Hands-on Verification Task:** Build a certification-adjacent matrix with event, expected behavior, observed behavior and evidence path.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016], [SRC-BUILD-018]
- **Version Notes:** Platform-holder certification docs are confidential and change over time.

### Question: A packaged Android build crashes on launch. What do you do?

- **Category:** Platform / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Identify the exact artifact/device/toolchain, reproduce from clean install, collect target logs/crash data, classify the failing phase, inspect staged files and symbolicate the crash.
- **Strong 3-Year-Engineer Answer:** I record package, commit, engine, config, SDK/NDK, device model and OS. I install the archived package on a clean device, collect platform/UE logs and crash data, then classify whether it fails before engine init, plugin/native library load, asset load, shader/PSO, map load or gameplay startup. I inspect staged assets/config/native libraries, verify permissions and symbolicate against the exact archived symbols before changing code.
- **Common Weak Answer:** “Run it from the editor or reinstall the SDK.”
- **Follow-up Questions:** Native library missing? Wrong ABI? Cooked asset missing? Symbol mismatch?
- **Hands-on Verification Task:** Force one packaged launch failure and produce a phase-classified diagnosis with logs and symbols.
- **Sources:** [SRC-PLAT-001], [SRC-PLAT-004], [SRC-PLAT-007], [SRC-BUILD-018]
- **Version Notes:** Android toolchain/package behavior is version-sensitive.

### Question: How do you design a release artifact contract?

- **Category:** Platform / Build Release
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Archive the exact package, symbols, manifests, logs, metadata, command parameters and validation results needed to reproduce, diagnose and certify a build.
- **Strong 3-Year-Engineer Answer:** A release artifact set includes engine/project commits, target/platform/config, SDK/toolchain versions, package ID, maps, plugins, cook/stage/package logs, manifests/chunks, signing metadata where appropriate, symbols, crash reporter settings and smoke/performance results. The contract exists so a device-only crash, missing asset or certification issue can be diagnosed against the exact shipped bits.
- **Common Weak Answer:** “Keep the packaged build and logs.”
- **Follow-up Questions:** Why symbols must match? What manifests matter? How handle third-party binaries?
- **Hands-on Verification Task:** Archive a packaged build and prove a forced crash symbolicates against it.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016], [SRC-BUILD-017], [SRC-BUILD-018]
- **Version Notes:** Artifact fields vary by platform/store/studio policy.

### Question: How do you profile platform performance correctly?

- **Category:** Platform / Profiling
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use packaged builds on representative hardware, define budgets/scenarios, measure frame percentiles/hitches/memory/thermal over realistic durations, then compare device profiles/scalability.
- **Strong 3-Year-Engineer Answer:** I define target FPS, resolution, quality tier, workload, sample duration and thermal state. I capture cold/warm launch, several-minute gameplay and stress scenes on representative hardware. I record P50/P95/P99, hitches, Game/Draw/GPU, memory/LLM, texture pool, I/O and battery/thermal notes where possible. I only add gates after understanding noise and variance.
- **Common Weak Answer:** “Use stat unit on the target device and lower settings.”
- **Follow-up Questions:** Why averages hide hitches? Why editor is invalid? What is a thermal-length capture?
- **Hands-on Verification Task:** Produce two-tier packaged performance captures with applied profile proof.
- **Sources:** [SRC-PERF-001], [SRC-PERF-003], [SRC-PERF-007], [SRC-PERF-008], [SRC-PERF-009], [SRC-PERF-010]
- **Version Notes:** Tool support varies by platform/build configuration.

### Question: What platform lifecycle failures should a UE game handle?

- **Category:** Platform / Lifecycle
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Suspend/resume, background/foreground, input disconnect, account/profile changes, network loss, storage failure, save corruption and entitlement/overlay changes.
- **Strong 3-Year-Engineer Answer:** I treat each as a state transition with expected behavior and evidence. On resume, delegates/timers/audio/network/UI/input should not be stale. On storage/network/account failure, gameplay should block, retry or degrade gracefully. For platform overlays and controller changes, input/focus state must restore. I test these in packaged builds because editor rarely reproduces platform lifecycle correctly.
- **Common Weak Answer:** “Save on pause and reconnect if network drops.”
- **Follow-up Questions:** What happens during async load? How handle corrupted save? What about lost controller during gameplay?
- **Hands-on Verification Task:** Inject three lifecycle failures and write expected/observed/evidence rows.
- **Sources:** [SRC-PLAT-001], [SRC-PLAT-002], [SRC-BUILD-018]
- **Version Notes:** Exact events/APIs are platform-specific.

### Question: How do you discuss console certification without leaking or bluffing?

- **Category:** Platform / Interview Strategy
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** State that exact requirements are platform-holder confidential, then discuss the engineering categories, evidence workflow and need to use official platform docs.
- **Strong 3-Year-Engineer Answer:** I would not quote public guesses for TRC/TCR/XR. I would say the project builds a checklist from the platform-holder documentation and maps every requirement to tests, owners and evidence. The public engineering categories are boot/load, suspend/resume, input, storage, account, network, overlays, crashes, localisation/accessibility and package/entitlement handling.
- **Common Weak Answer:** “I know console cert means no crashes and proper button prompts.”
- **Follow-up Questions:** How do you turn a confidential requirement into automated or manual evidence? Who owns fixes? What belongs in release notes?
- **Hands-on Verification Task:** Convert five public category failures into a test matrix without inventing confidential details.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016], [SRC-BUILD-018]
- **Version Notes:** Source-sensitive; exact console requirements require authorised platform documentation.

## Profiling and Optimisation

### Question: How do you diagnose a frame-rate drop systematically?

- **Category:** Profiling / Optimisation
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Define target/scenario, classify the limiting lane with frame-time stats, capture detailed evidence, test one hypothesis, and verify before/after on target hardware.
- **Strong 3-Year-Engineer Answer:** I record build, hardware, resolution/quality, VSync/cap/dynamic resolution, camera/workload, and frame-time distribution. `stat unit` classifies Game, Draw, GPU, or another constraint. I capture Insights or GPU profile on slow representative frames, inspect top scopes/passes plus frequency/callers, make one causal change, and rerun the identical scenario. I verify correctness, visual quality, memory, percentiles/hitches, and other device tiers, then add regression coverage.
- **Common Weak Answer:** “Run stat unit, then optimise the biggest number.”
- **Follow-up Questions:** How do pipelines overlap? What if Game thread is waiting? How do you avoid benchmark noise?
- **Hands-on Verification Task:** Produce a before/after trace with one falsified alternative hypothesis.
- **Sources:** [SRC-PERF-001], [SRC-PERF-002], [SRC-PERF-003]
- **Version Notes:** UE5.3–UE5.6; tool UI evolves.

### Question: Why use frame time instead of only FPS?

- **Category:** Profiling / Frame Budget
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Milliseconds are additive and map directly to budgets; FPS is reciprocal and averages can hide hitches.
- **Strong 3-Year-Engineer Answer:** At 60 FPS I have 16.67 ms, but I budget below that for variability. I track median plus P95/P99 and worst representative hitches. A 50 ms spike matters even if average FPS remains near 60. Milliseconds let me reason about a 2 ms feature cost directly; the same FPS difference means different time at different baselines.
- **Common Weak Answer:** “Frame time is more accurate.”
- **Follow-up Questions:** Budgets for 30/90/120? Why leave headroom? What is display-bound?
- **Hands-on Verification Task:** Convert several FPS values to ms and compare equal 10 FPS changes at different baselines.
- **Sources:** [SRC-PERF-001]
- **Version Notes:** Platform-independent principle.

### Question: What does `stat unit` tell you?

- **Category:** Profiling / UE Tools
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** High-level Frame, Game, Draw, GPU, RHI, and dynamic-resolution timings for initial bottleneck classification.
- **Strong 3-Year-Engineer Answer:** If Frame tracks Game, I inspect gameplay/CPU traces; if Draw, render-thread CPU; if GPU, GPU passes. I account for overlapping/synchronised pipelines, frame caps, VSync and dynamic resolution, so I do not add the lanes. It is triage, not root-cause proof; Insights or GPU Visualiser provides detail.
- **Common Weak Answer:** “Whichever number is highest is the bottleneck.”
- **Follow-up Questions:** Why are several numbers near Frame? How can waits mislead? Which build should you use?
- **Hands-on Verification Task:** Create independent game-thread and GPU loads and classify both with identical settings.
- **Sources:** [SRC-PERF-002]
- **Version Notes:** UE5.3–UE5.6; stat labels/tool behaviour target-sensitive.

### Question: How do you use Unreal Insights effectively?

- **Category:** Profiling / UE Tools
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Capture a short reproducible window, compare slow/healthy frames, inspect relevant tracks/scopes/callers, instrument ambiguity, and recapture.
- **Strong 3-Year-Engineer Answer:** I enable only needed trace channels, include warm-up and the event marker, then align Game/Render/RHI/GPU/workers. I inspect inclusive/exclusive time, call count, callers/callees, task dependencies and waits. Aggregation separates steady cost from one hitch. If project code is opaque, I add one scoped event/counter around the suspected unit. Heavy traces can perturb the workload, so final verification uses a lighter capture/representative build.
- **Common Weak Answer:** “Sort functions by total time.”
- **Follow-up Questions:** Inclusive versus exclusive? How find duplicate calls? How identify waiting versus work?
- **Hands-on Verification Task:** Find one duplicate-scheduled project scope using callers/callees and a custom counter.
- **Sources:** [SRC-PERF-003], [SRC-PERF-004]
- **Version Notes:** UE5.3–UE5.6; channels/panels evolve.

### Question: Tick, timer, callback, or dirty batching?

- **Category:** Profiling / Gameplay Performance
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Match cadence and semantics: frame integration uses Tick, scheduled work uses timers, state changes use events, and many same-frame changes may need dirty batching.
- **Strong 3-Year-Engineer Answer:** I measure frequency and work. Replacing Tick with a frame-rate timer changes little. Polling a rare state change is wasteful, so a callback fits. But if dozens of events overwrite the same derived result in one frame, I mark dirty and recompute once. I disable idle updates, stagger intervals where latency permits, and profile total invocation count—not follow a ban on Tick.
- **Common Weak Answer:** “Timers are cheaper than Tick.”
- **Follow-up Questions:** When are events worse? Tick intervals? Manager batching?
- **Hands-on Verification Task:** Implement and trace one system using all four scheduling models under low/high change rates.
- **Sources:** [SRC-PERF-005]
- **Version Notes:** Workload-dependent; no universal cadence.

### Question: How do you investigate a memory leak or growth?

- **Category:** Profiling / Memory
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Compare marked timeline points across a repeatable lifecycle, query live/growth allocations, group by callstack/tag, and distinguish cache/warm-up/GC from unbounded retention.
- **Strong 3-Year-Engineer Answer:** I repeat the event such as level travel or opening/closing UI. In Memory Insights I mark before/after and query growth, long-lived, or leak-like allocations grouped by callstack/LLM tag. A one-time warm cache is not a leak; staircase growth across cycles is stronger evidence. I check UObject references/GC and streaming before assuming native allocation loss.
- **Common Weak Answer:** “Look for the largest allocator in Memory Insights.”
- **Follow-up Questions:** Churn versus leak? Short-lived allocation query? How can GC delay release?
- **Hands-on Verification Task:** Create intentional retained widget/native allocations across repeated open/close cycles and identify both.
- **Sources:** [SRC-PERF-006]
- **Version Notes:** UE5 Memory Insights; Android callstack support has version constraints.

### Question: How do you diagnose a GC hitch?

- **Category:** Profiling / UObject / Memory
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Correlate the spike with GC, object/reference counts and destruction bursts, then reduce unnecessary UObject churn/retention or schedule measured collection at tolerable boundaries.
- **Strong 3-Year-Engineer Answer:** I confirm GC timing in Insights and inspect object counts and the event causing destruction. I find short-lived Actors/UObjects that could be values, accidental strong graphs, or mass destruction in one frame. Pooling is evaluated with memory/reset costs. I do not simply lengthen the interval, which may create a larger later spike; manual GC is justified only at measured UX-safe boundaries such as loading.
- **Common Weak Answer:** “Increase time between garbage collections.”
- **Follow-up Questions:** Why can pooling hurt? What does incremental GC change? How do soft references affect memory?
- **Hands-on Verification Task:** Create a burst-destroy scenario and compare reduced UObject churn, staged destruction, and bounded pooling.
- **Sources:** [SRC-PERF-005], [SRC-EPIC-005]
- **Version Notes:** Incremental reachability is experimental in UE5.6.

### Question: CPU-bound versus GPU-bound—how do you prove it?

- **Category:** Profiling / Rendering
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Use frame/thread/GPU timings and controlled sensitivity tests, then inspect the limiting CPU scopes or GPU passes.
- **Strong 3-Year-Engineer Answer:** `stat unit` gives initial classification. I remove frame caps/confounders and test resolution scale: a strong GPU-time response suggests pixel/shader/fill cost, while little response may suggest fixed geometry/pass cost or CPU limitation. CPU cases go to Timing Insights; GPU cases to stat GPU/GPU Visualiser and possibly RenderDoc. I verify the same packaged target workload.
- **Common Weak Answer:** “If GPU utilisation is 100%, it is GPU-bound.”
- **Follow-up Questions:** Draw thread versus GPU? Dynamic resolution? Why can utilisation mislead?
- **Hands-on Verification Task:** Build one resolution-sensitive material load and one game-thread loop, then classify blindly from captures.
- **Sources:** [SRC-PERF-001], [SRC-PERF-002]
- **Version Notes:** Target hardware and measurement mode matter.

### Question: When is object pooling a good optimisation?

- **Category:** Profiling / Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** When measured allocation/construction/destruction churn is material and objects can be reset safely within acceptable retained memory.
- **Strong 3-Year-Engineer Answer:** I profile churn first. A pool trades allocator/registration/GC work for persistent peak capacity, reset code, stale handles and cache behaviour. I define acquire/release ownership, reset invariants, maximum size, shrink policy, and failure diagnostics. Lightweight values or batch representation may beat pooling heavy Actors entirely.
- **Common Weak Answer:** “Pool frequently spawned objects such as bullets.”
- **Follow-up Questions:** Pooling Actors? Network identity? How test stale state? When is allocation already cheap?
- **Hands-on Verification Task:** Compare spawn/destroy, pool, and batched/projectile-data designs including memory and reset failures.
- **Sources:** [SRC-PERF-005], [SRC-CPP-006]
- **Version Notes:** Workload/platform dependent.

### Question: How do scalability and device profiles differ from optimisation?

- **Category:** Profiling / Scalability
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Optimisation reduces cost for a given result; scalability/device profiles intentionally choose different quality/cost configurations across hardware.
- **Strong 3-Year-Engineer Answer:** Scalability groups expose user/quality tiers; device profiles apply platform/device-family overrides such as resolution, texture pools and feature CVars. I design scalable dimensions early and test low/high tiers on real devices. I keep gameplay authority and required readability independent of client visual quality. Profiles complement optimisation—they do not excuse an inefficient baseline.
- **Common Weak Answer:** “Scalability turns expensive settings down on low-end devices.”
- **Follow-up Questions:** Dynamic resolution? Memory buckets? How avoid profile inheritance mistakes?
- **Hands-on Verification Task:** Define three tiers and two device profiles, then verify applied CVars, memory and visual/gameplay correctness.
- **Sources:** [SRC-PERF-007], [SRC-PERF-008]
- **Version Notes:** Exact groups/CVars and device selectors vary by platform/version.

### Question: How do you design a reproducible packaged performance benchmark?

- **Category:** Profiling / Benchmark Design
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Pin build, hardware, quality/profile, scenario, warm-up, sample window, metrics and pass/fail rules before comparing results.
- **Strong 3-Year-Engineer Answer:** I define the benchmark contract first: engine/project revision, target/config/platform, device, driver/power/thermal state, resolution, VSync/cap/dynamic resolution, scalability/device profile, map, fixed camera/input path, seed, warm-up and sample length. I collect frame-time distribution, hitches, Game/Draw/GPU, memory and scenario counts. Then I compare identical packaged runs with tolerance bands and artifacts, not editor viewport averages.
- **Common Weak Answer:** “Run the level and record FPS before and after.”
- **Follow-up Questions:** How handle warm-up? How prove profile selection? Why packaged? What is noise?
- **Hands-on Verification Task:** Write a benchmark contract for one combat scenario and explain which fields make the result reproducible.
- **Sources:** [SRC-PERF-001], [SRC-PERF-003], [SRC-PERF-009]
- **Version Notes:** Tool channels and platform capture setup are target-sensitive.

### Question: CSV Profiler versus Unreal Insights?

- **Category:** Profiling / Tools
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** CSV is lightweight long-run/regression telemetry; Insights is heavier causal trace analysis for frames, tasks, allocations, loads and waits.
- **Strong 3-Year-Engineer Answer:** I use CSV to monitor stable metrics across automated scenarios: frame percentiles, hitches, high-level thread/GPU timings, memory tags and project counters. It is good for detecting drift over many runs. When a CSV gate fails, I use Insights or a GPU/platform profiler to find the causal frame, scope, wait, allocation or pass. CSV is a smoke alarm; Insights is the investigation.
- **Common Weak Answer:** “Insights is more detailed, so always use Insights.”
- **Follow-up Questions:** What makes CSV suitable for CI? What can CSV not prove? How can traces perturb results?
- **Hands-on Verification Task:** Add one CSV gate metric and one Insights follow-up capture for the same deliberate regression.
- **Sources:** [SRC-PERF-003], [SRC-PERF-009], [SRC-PERF-004]
- **Version Notes:** CSV categories and trace channels vary by branch.

### Question: Memory Insights versus Low-Level Memory Tracker?

- **Category:** Profiling / Memory
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Memory Insights explains allocation lifetime, growth, churn and callstacks; LLM assigns memory ownership to tags/budgets.
- **Strong 3-Year-Engineer Answer:** For an over-budget target, I use LLM to see which subsystem tags own memory at stable checkpoints: boot, map loaded, peak and return-to-menu. Then I use Memory Insights around the suspicious interval to inspect allocation lifetime, callstacks, growth and churn. LLM tells me where the budget went; Memory Insights helps explain why it was allocated, retained or repeatedly churned.
- **Common Weak Answer:** “Use Memory Insights to find the largest memory category.”
- **Follow-up Questions:** What is capacity versus churn? How can tag granularity mislead? What if growth is a cache?
- **Hands-on Verification Task:** Create a repeated UI open/close growth case and report both LLM-tag ownership and Memory Insights callstack evidence.
- **Sources:** [SRC-PERF-006], [SRC-PERF-010], [SRC-PERF-005]
- **Version Notes:** Memory callstacks and LLM tag coverage depend on platform/build configuration.

### Question: How do you prove device profiles and scalability settings actually applied?

- **Category:** Profiling / Scalability
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Capture applied CVars/profile selection on target hardware, then verify performance, memory and gameplay readability at low and high tiers.
- **Strong 3-Year-Engineer Answer:** I do not treat config files as proof. I run the packaged target, dump applied profile/CVars or equivalent evidence, and compare low/high tiers with the same scenario. I verify visual readability, UI affordances, gameplay cues, memory pool changes and network/authority independence. A low profile that hides enemy telegraphs or changes simulation is a product bug, even if frame time improves.
- **Common Weak Answer:** “Set lower scalability values for low-end devices.”
- **Follow-up Questions:** Profile inheritance? Memory buckets? Dynamic resolution? How keep gameplay independent?
- **Hands-on Verification Task:** Build two profiles with conflicting overrides, prove which one wins at runtime and document the fix.
- **Sources:** [SRC-PERF-007], [SRC-PERF-008], [SRC-PERF-001]
- **Version Notes:** Device selectors, inherited CVars and platform profiles are target-sensitive.

### Question: Packaged build has an intermittent hitch. What is your workflow?

- **Category:** Profiling / Hitch Diagnosis
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Reproduce in the exact target build, use CSV for long-run detection, then capture a short causal trace around the hitch and classify load, GC, PSO, task, memory or GPU source.
- **Strong 3-Year-Engineer Answer:** I first lock build, device profile, resolution and frame caps. If the hitch is intermittent, I run lightweight CSV telemetry to find when it happens. Then I capture a short Insights window around one reproduced hitch and align Game/Draw/GPU, file/loading, memory allocation, GC, shader/PSO and task waits. I make one causal change and rerun the identical packaged scenario, then add a gate so the hitch does not return.
- **Common Weak Answer:** “Profile with Insights until you see a spike.”
- **Follow-up Questions:** Why short traces? How distinguish wait from work? How detect PSO or async-load hitches?
- **Hands-on Verification Task:** Inject one first-use load or PSO hitch and produce CSV detection plus one short causal trace.
- **Sources:** [SRC-PERF-003], [SRC-PERF-009], [SRC-PERF-006], [SRC-RENDER-012]
- **Version Notes:** Hitch mechanisms and trace channel names are branch/RHI/platform sensitive.

### Question: How do you add performance CI without making it unusable?

- **Category:** Profiling / CI
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use tiered, scenario-specific gates with stable metrics, tolerance bands, artifacts and owners; escalate to detailed captures only on failure.
- **Strong 3-Year-Engineer Answer:** I keep pre-submit cheap and reserve packaged CSV scenarios, memory checkpoints and device-profile runs for nightly or release tiers. Each gate names scenario, warm-up/window, build, profile, metric, threshold, tolerance, artifacts and owner. It should fail on actionable repeated regressions, not one noisy run. On failure, automation captures the minimal causal evidence needed: Insights, Memory Insights, GPU capture or platform profiler.
- **Common Weak Answer:** “Fail CI if FPS drops below 60.”
- **Follow-up Questions:** Rolling baseline? P95/P99? False positives? Who owns a gate?
- **Hands-on Verification Task:** Design a four-scenario performance gate table with thresholds, tolerance and escalation artifacts.
- **Sources:** [SRC-PERF-009], [SRC-PERF-011], [SRC-BUILD-015]
- **Version Notes:** Automation hooks, hardware availability and noise policy are project-specific.

## Rendering and GPU Performance

### Question: Walk through a rendered frame in Unreal at interview depth.

- **Category:** Rendering / Pipeline
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** The game thread updates state, render-side code builds visible-view work, RHI command work is submitted, and the GPU executes geometry, depth/shadow, material, lighting, translucency and post passes that vary by renderer/features.
- **Strong 3-Year-Engineer Answer:** I separate overlapping Game, Draw/render, optional RHI, and GPU lanes rather than adding their times. For a deferred desktop view I expect visibility/draw preparation, depth/shadow work, a base pass writing GBuffer surface data, lighting, feature passes such as Lumen, translucency, post and composition. The exact graph depends on platform, renderer and settings, so I verify it in GPU Visualiser/Insights rather than claiming a fixed universal order.
- **Common Weak Answer:** “The CPU sends meshes to the GPU, which draws vertices and pixels.”
- **Follow-up Questions:** What is the RHI? Why can a property update appear a frame later? Which work can overlap?
- **Hands-on Verification Task:** Annotate one target capture with CPU lanes, major GPU passes, dependencies, and overlapping intervals.
- **Sources:** [SRC-PERF-003], [SRC-RENDER-003]
- **Version Notes:** UE5.3–UE5.6; pass shape and threading configuration are target-sensitive.

### Question: Deferred versus forward shading—how do you choose?

- **Category:** Rendering / Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Deferred offers the default broad feature/dynamic-light path; forward can offer a leaner baseline and MSAA for suitable VR/content, with a different feature and material/light cost model.
- **Strong 3-Year-Engineer Answer:** Deferred writes material attributes to GBuffer targets then lights later, buying flexibility at memory/bandwidth cost. Unreal's forward renderer culls lights/reflection captures into a grid and evaluates relevant ones during shading; it supports MSAA and can suit VR. I begin with required features, anti-aliasing and platform support, then benchmark the same representative scenes. Epic's example performance gain is not a project-independent promise.
- **Common Weak Answer:** “Forward is faster but supports fewer lights.”
- **Follow-up Questions:** Why is MSAA relevant to VR? What happens to translucent surfaces? How does light coverage affect cost?
- **Hands-on Verification Task:** Compare a controlled project in both renderers, listing feature differences and Game/Draw/GPU/pass results.
- **Sources:** [SRC-RENDER-002]
- **Version Notes:** Feature compatibility evolves; verify the exact target branch.

### Question: Draw calls versus shader, geometry, fill-rate and bandwidth cost?

- **Category:** Rendering / Performance
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Draw/state submission often pressures CPU render/RHI work; geometry pressures vertex/raster work; shader/fill/overdraw pressures GPU pixels; render targets/textures can pressure memory bandwidth.
- **Strong 3-Year-Engineer Answer:** I name a cost dimension before proposing a fix. Thousands of tiny objects can be Draw-bound even at low resolution. One full-screen post or translucent effect can be GPU-bound with few draws. Shadow/depth passes can be geometry-heavy without full material shading. I use object-count, resolution, material, caster and coverage sensitivity tests, then confirm the affected lane/pass. Instancing does not repair a full-screen shader, and lowering resolution does not repair render-thread object management.
- **Common Weak Answer:** “Reduce polygons and draw calls to make GPU rendering faster.”
- **Follow-up Questions:** Why can merging hurt? What are tiny triangles? How can a shader be bandwidth-bound?
- **Hands-on Verification Task:** Build four scenes isolating object count, triangle count, full-screen shading and translucent overdraw; classify blind captures.
- **Sources:** [SRC-RENDER-004], [SRC-RENDER-009], [SRC-RENDER-010]
- **Version Notes:** Hardware/content dependent; diagnostic views are not final timing evidence.

### Question: How do you investigate a GPU regression?

- **Category:** Rendering / Debugging
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Reproduce under controlled settings, prove GPU limitation, name the regressed pass, test a mechanism hypothesis, then verify a requirement-preserving change across targets.
- **Strong 3-Year-Engineer Answer:** I record build, API/driver/hardware, resolution/screen percentage, quality, VSync/cap/dynamic resolution and camera path. I use `stat unit`/trace for classification and `stat gpu`, GPU Visualiser or platform tools to compare named passes with a baseline. My hypothesis includes a mechanism and prediction—for example VSM invalidation from moving WPO casters. I isolate one variable, recapture distributions, verify visual correctness and check whether cost moved to Draw, Game or memory.
- **Common Weak Answer:** “Run ProfileGPU, disable the largest feature, and see if FPS improves.”
- **Follow-up Questions:** When use RenderDoc/PIX/Nsight? Why can feature-off tests mislead? How handle capture noise?
- **Hands-on Verification Task:** Produce a regression report containing one falsified hypothesis and one confirmed pass/mechanism.
- **Sources:** [SRC-PERF-001], [SRC-RENDER-009], [SRC-RENDER-010]
- **Version Notes:** GPU event names/tool support vary by RHI, platform and build.

### Question: Why can translucency be expensive?

- **Category:** Rendering / Materials
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Large layered translucent surfaces repeatedly shade/blend pixels, have weaker opaque depth rejection, sorting constraints and renderer-feature limitations.
- **Strong 3-Year-Engineer Answer:** Cost is coverage × layers × samples × shader work, not simply particle count. I inspect the Translucency pass and overdraw/complexity views, then test internal resolution, screen coverage, material simplification and layer count. Depending on the look, I reduce coverage/layers, simplify shading, use lower-resolution paths, masked/opaque/impostor alternatives, or author effects to avoid full-screen stacks. I recheck sorting and visual requirements.
- **Common Weak Answer:** “Transparency is expensive because the GPU must sort every pixel.”
- **Follow-up Questions:** Masked versus translucent? Why can a small particle system be expensive? What does early Z provide opaque surfaces?
- **Hands-on Verification Task:** Hold particle count constant while varying screen coverage and layers; chart Translucency pass scaling.
- **Sources:** [SRC-RENDER-004], [SRC-RENDER-010]
- **Version Notes:** Exact translucency paths and features are renderer/version-sensitive.

### Question: Nanite versus LOD, instancing and HLOD?

- **Category:** Rendering / Geometry
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Nanite virtualises supported geometry detail/visibility; LOD simplifies one asset, instancing represents repeats efficiently, and HLOD proxies distant spatial groups.
- **Strong 3-Year-Engineer Answer:** These are not interchangeable checkboxes. I choose from platform/renderer support, repetition, deformation/material needs, streaming and culling granularity. Nanite changes geometry and draw economics but does not remove material, pixel, shadow, lighting, object-management or memory budgets. Traditional LOD/ISM/HISM/HLOD remain useful for unsupported targets/content and different CPU/world-organisation costs. I compare the same camera path and record Draw/RHI, named GPU passes, memory and artefacts.
- **Common Weak Answer:** “Use Nanite for high-poly meshes and HISM for low-poly repeats.”
- **Follow-up Questions:** What does Nanite not solve? Why can one merged proxy hurt culling? What support must be checked?
- **Hands-on Verification Task:** Compare independent components, ISM/HISM, LOD, Nanite and HLOD representations of one population.
- **Sources:** [SRC-RENDER-005], [SRC-RENDER-008], [SRC-RENDER-011]
- **Version Notes:** Nanite's support matrix is highly version-sensitive; current docs may exceed UE5.6.

### Question: How do Virtual Shadow Maps work, and what invalidates their cache?

- **Category:** Rendering / Shadows
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** VSMs allocate/render high-resolution virtual shadow-map pages needed by visible pixels and cache them; light or caster changes, new visibility and deformation can require or invalidate pages.
- **Strong 3-Year-Engineer Answer:** I correlate shadow pass time with VSM page/cache/invalidation visualisations. Moving lights can affect broad regions; moving or WPO-deformed casters invalidate their pages; non-Nanite casters can add conventional draw pressure; oversized bounds and coarse pages can enlarge work. I freeze controlled lights/casters or WPO to test the mechanism, then narrow influence, fix bounds/deformation, reduce caster cost or change quality—not simply disable all shadows.
- **Common Weak Answer:** “VSMs stream shadow tiles like virtual textures, so only moving objects cost.”
- **Follow-up Questions:** Why do Nanite and VSM fit together? What are coarse pages? Why can stationary-looking foliage invalidate?
- **Hands-on Verification Task:** Create static, rigid-moving and WPO caster cases and compare page invalidation and shadow pass times.
- **Sources:** [SRC-RENDER-006]
- **Version Notes:** Actively developed across UE5; CVars and visualisations require target verification.

### Question: How would you debug a Lumen artefact or performance problem?

- **Category:** Rendering / Lighting
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Classify the visual/cost symptom, identify software versus hardware tracing and quality, inspect Lumen scene/surface representations and named passes, then isolate one representation or trace variable.
- **Strong 3-Year-Engineer Answer:** I distinguish leaking, missing bounce, noisy emissive, slow update, bad reflection and excessive time. I check scalability and active tracing mode, compare the main view with Lumen overview/scene/surface-cache visualisations, and determine whether screen traces hide a scene-representation problem. I isolate GI from reflections and change one geometry, distance-field, surface-cache, trace, material or quality variable. Published Epic budgets are platform/configuration targets, not guarantees.
- **Common Weak Answer:** “Increase Lumen quality or switch on hardware ray tracing.”
- **Follow-up Questions:** Screen traces versus reliable fallback? Why do tiny emissives create noise? What does software tracing require?
- **Hands-on Verification Task:** Deliberately create a scene-representation artefact and document the visualisation-led fix.
- **Sources:** [SRC-RENDER-001], [SRC-RENDER-007]
- **Version Notes:** UE5-specific and highly platform/scalability/version-sensitive.

### Question: What is RDG, and why does declaring resources matter?

- **Category:** Rendering / Engine Internals
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** RDG records passes and declared resource dependencies into a graph so Unreal can validate, schedule, transition, cull and manage transient resources safely.
- **Strong 3-Year-Engineer Answer:** `FRDGBuilder` creates descriptors and records passes; underlying resources may be allocated later. Pass parameter structs declare reads/writes, from which RDG derives dependencies and lifetimes. Execution lambdas run later and must not capture expired memory or access undeclared resources. The graph can then handle barriers, transient aliasing, pass culling, parallel command recording and async-compute fences. RDG Insights/validation help inspect the contract.
- **Common Weak Answer:** “RDG automatically orders render passes and saves memory.”
- **Follow-up Questions:** Transient versus external resource? Setup versus execute timeline? Why avoid immediate command lists?
- **Hands-on Verification Task:** Add one named RDG pass in a source branch and inspect its dependencies/lifetime in RDG Insights.
- **Sources:** [SRC-RENDER-003]
- **Version Notes:** Specialist, source-branch and exact-API sensitive.

### Question: Static switch, dynamic material parameter, or separate material?

- **Category:** Rendering / Materials / Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Dynamic parameters change runtime values in an existing variant; static switches compile variants; separate parents define genuinely different bounded feature families.
- **Strong 3-Year-Engineer Answer:** I use dynamic parameters for runtime value variation that should not multiply permutations. Static switches can compile away unused paths but combinations multiply shader compile time, library/memory/PSO diversity and maintenance. I split parent materials when feature families or shading models are genuinely different, while avoiding thousands of unique one-offs. I inspect platform stats, actual permutation counts and runtime passes rather than counting graph nodes.
- **Common Weak Answer:** “Static switches are faster because branches are removed.”
- **Follow-up Questions:** What creates shader permutations besides switches? Material instance versus dynamic instance? How can permutations cause hitches/build pain?
- **Hands-on Verification Task:** Build a small material family, enumerate static combinations, compare compile/library/runtime effects, and redesign it with a permutation budget.
- **Sources:** [SRC-RENDER-003], [SRC-RENDER-004]
- **Version Notes:** Shader/PSO pipelines and analysis tools vary by platform/version.

### Question: Shader permutation, runtime shader cost, or PSO hitch?

- **Category:** Rendering / Shaders
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Shader permutations are compiled variants, runtime shader cost is GPU work, and PSO hitches are expensive first-time pipeline-state creation/binding coverage problems.
- **Strong 3-Year-Engineer Answer:** I split the diagnosis. Static switches, vertex factories, platforms, passes and quality levels create compiled shader variants and increase compile/DDC/library pressure. Runtime shader cost is instructions, texture reads, bandwidth and occupancy in the hot pass. A PSO hitch can still happen when shader code exists because the exact pipeline state combination was not cached/prepared. I use material stats and pass timings for runtime cost, build/cook/DDC data for permutation pressure, and packaged PSO logging/cold-cache captures for first-use hitches.
- **Common Weak Answer:** “Compile the shaders earlier and the rendering hitch goes away.”
- **Follow-up Questions:** What creates permutations? What is in a PSO? Why can scene captures miss cache coverage?
- **Hands-on Verification Task:** Create a rare material path that has compiled shaders but misses PSO cache coverage, then prove the first-use hitch and fix.
- **Sources:** [SRC-RENDER-012], [SRC-RENDER-013], [SRC-RENDER-004]
- **Version Notes:** PSO cache workflow is RHI/platform/branch sensitive.

### Question: How do you build a safe custom global shader or RDG pass?

- **Category:** Rendering / Shader Programming
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Register/load shader code correctly, bound permutations, declare RDG parameters/resources, respect deferred execution lifetimes and validate output with RDG tools or GPU capture.
- **Strong 3-Year-Engineer Answer:** I use a global/plugin shader only when material graphs or existing passes are the wrong abstraction. The plugin/module must map shader source and load early enough for shader type registration. `ShouldCompilePermutation` limits platform/feature variants. Parameter structs declare constants and RDG resources; the pass lambda executes later, so it must not capture expired memory or read UObjects on the wrong thread. I start with a constant output, inspect RDG validation/Insights and take a GPU capture if output or bindings are wrong.
- **Common Weak Answer:** “Add a `.usf`, call it from C++, and the render thread will run it.”
- **Follow-up Questions:** module loading phase? shader directory mapping? RDG dependency? captured stack memory? packaged build?
- **Hands-on Verification Task:** Add a minimal compute/full-screen pass and deliberately break one shader path or RDG dependency before fixing it.
- **Sources:** [SRC-RENDER-013], [SRC-RENDER-014], [SRC-RENDER-003], [SRC-RENDER-019]
- **Version Notes:** Exact macros, includes and APIs require target source compile proof.

### Question: What is the PSO cache workflow for first-use hitches?

- **Category:** Rendering / PSO Caches
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Reproduce first-use hitches in packaged target builds, collect/merge/package/precache representative PSOs, and verify cold-cache runs across rare content paths.
- **Strong 3-Year-Engineer Answer:** I do not assume editor warm-cache smoothness predicts shipping. I reproduce in packaged builds with target RHI/settings, correlate hitch timing with first appearance of material/effect/view, and separate shader compilation, asset streaming and PSO creation. Then I collect stable PSOs from representative gameplay including VFX, UI, cutscenes, scene captures and scalability paths, merge/package/precache through the target workflow and verify cold-cache hitches disappear. I also reduce uncontrolled material/permutation diversity so caches remain tractable.
- **Common Weak Answer:** “Run around the map once so the PSOs are cached.”
- **Follow-up Questions:** rare cosmetics? VFX? scene captures? RHI differences? cache invalidation after material changes?
- **Hands-on Verification Task:** Build a rare VFX or scene-capture material, record first-use hitch, add it to PSO coverage and prove the fix in a fresh packaged run.
- **Sources:** [SRC-RENDER-012], [SRC-RENDER-019], [SRC-PERF-003]
- **Version Notes:** Collection commands, cache formats and precache systems vary by branch/platform.

### Question: What does the mesh drawing pipeline explain for gameplay engineers?

- **Category:** Rendering / Mesh Drawing
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Gameplay primitives become render-side scene proxies, mesh batches and pass-specific draw commands; material slots, dynamic state and render-state invalidation can create CPU render cost.
- **Strong 3-Year-Engineer Answer:** A Static Mesh Component does not equal one draw call. The renderer builds primitive scene data, relevance, mesh batches and pass processors for depth, base, shadow, velocity and other passes. Cached mesh draw commands help suitable static work, while per-frame material overrides, component recreation, `MarkRenderStateDirty`, custom depth/stencil changes and many material sections can force more dynamic setup. If Draw/RHI rises, I inspect primitive/draw counts, material diversity, render-state invalidation and independent Components versus ISM/HISM/Nanite/HLOD before proposing a fix.
- **Common Weak Answer:** “Each mesh is one draw call unless it has many materials.”
- **Follow-up Questions:** static versus dynamic draw command? material sections? render-state dirty? scene captures?
- **Hands-on Verification Task:** Compare 1000 independent Components, ISM/HISM and a per-frame render-state-dirty variant with Draw/RHI and primitive/draw counts.
- **Sources:** [SRC-RENDER-015], [SRC-RENDER-008], [SRC-PERF-003]
- **Version Notes:** Renderer internals and class names evolve; use branch source for implementation details.

### Question: How should you budget render targets and scene captures?

- **Category:** Rendering / Render Targets
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Budget resolution, format, update cadence, active count, view complexity, mips/copies/readbacks and consumers; assume scene captures are extra views until proven cheaper.
- **Strong 3-Year-Engineer Answer:** A render target is memory and bandwidth, and a SceneCapture2D often renders an additional camera view. SceneCaptureCube can multiply that further by six directions. For minimaps, thumbnails and security cameras I prefer manual or low-frequency updates, lower resolution/format, show-only lists, distance/show-flag trimming and shared/lifetime-managed targets. I include captures in PSO collection because they exercise material/pass combinations the main camera may not. I measure memory, GPU pass cost and UI/material sampling cost separately.
- **Common Weak Answer:** “Render targets are just textures, so reduce their resolution if they are slow.”
- **Follow-up Questions:** HDR format cost? `CaptureEveryFrame`? cube capture? readback? PSO coverage?
- **Hands-on Verification Task:** Build minimap, thumbnail and mirror variants and compare update cadence, resolution, format, show-only lists and active count.
- **Sources:** [SRC-RENDER-016], [SRC-RENDER-017], [SRC-RENDER-018], [SRC-RENDER-012]
- **Version Notes:** Exact SceneCapture settings and renderer paths are version/platform sensitive.

### Question: When should you escalate to RenderDoc or platform GPU tools?

- **Category:** Rendering / GPU Capture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** After Unreal pass timings name the problem area but you need exact pipeline state, resources, shaders, barriers, occupancy clues or platform-specific behaviour.
- **Strong 3-Year-Engineer Answer:** I use `stat gpu`, GPU Visualiser, RDG Insights and view modes to identify the pass and likely mechanism first. I escalate to RenderDoc, PIX, Nsight, Xcode or console tools when I need bound shader/resource/pipeline-state detail, draw/dispatch inspection, bandwidth/occupancy clues or vendor/RHI-specific behaviour. The capture must record build, RHI, GPU/driver, settings and event markers, and the finding must be translated back into an Unreal-level fix with normal before/after profiler evidence.
- **Common Weak Answer:** “Use RenderDoc whenever the GPU is slow.”
- **Follow-up Questions:** Why not only editor captures? What metadata must be recorded? Which tool for console/mobile?
- **Hands-on Verification Task:** Capture one custom pass or expensive material, identify a bound resource/pipeline fact, then verify the fix with GPU Visualiser/Insights.
- **Sources:** [SRC-RENDER-019], [SRC-RENDER-003], [SRC-RENDER-009]
- **Version Notes:** Tool support varies by RHI/platform/build; target vendor tools may supersede RenderDoc.

### Question: When does “works in editor” become weak rendering evidence?

- **Category:** Rendering / Production Proof
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** When shader/module startup, PSO coverage, scene-capture paths, cache state or packaged branch behaviour can differ from the editor path.
- **Strong 3-Year-Engineer Answer:** Editor success is only the first gate. Rendering features can still fail in packaged builds because the module load path differs, shader directories are not mapped in the packaged startup path, PSO coverage is missing on cold caches, or scene-capture/UI/material combinations were never exercised outside editor. I treat editor proof as correctness smoke, then package a deterministic target build, rerun cold, and archive pass/capture evidence plus fallback policy for lower-end targets.
- **Common Weak Answer:** “If it looks correct in PIE, the renderer part is done.”
- **Follow-up Questions:** shader-directory mapping? packaged module load? cold-cache hitch? scene-capture-only path?
- **Hands-on Verification Task:** Make one small rendering change, prove it in PIE, then produce a packaged evidence packet that either confirms or disproves parity.
- **Sources:** [SRC-RENDER-013], [SRC-RENDER-014], [SRC-RENDER-012], [SRC-RENDER-019]
- **Version Notes:** Packaged startup and PSO workflows are branch/platform sensitive.

### Question: How do you triage a custom shader pass that is black only in packaged build?

- **Category:** Rendering / Shader Programming
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Confirm the pass exists, then check shader registration/path mapping, permutation support, RDG resource declaration, thread/lifetime correctness and packaged build metadata before changing the algorithm.
- **Strong 3-Year-Engineer Answer:** I start by proving the pass is present with the expected event name in packaged evidence. If it is absent, I suspect module load order or shader-directory mapping. If it is present but black, I reduce it to a constant output, verify `ShouldCompilePermutation`, inspect RDG parameter declarations, resource formats/UAV flags and execution lifetime, then capture the frame in RenderDoc or the platform tool. I do not debug the math before proving the shader actually compiled, loaded and bound the expected resources in the packaged path.
- **Common Weak Answer:** “The shader code is wrong; tweak the HLSL until it looks right.”
- **Follow-up Questions:** event name? packaged path mapping? constant-colour output? UAV format? stack capture bug?
- **Hands-on Verification Task:** Deliberately break one packaged shader registration or RDG-resource detail, then restore it with packaged capture proof.
- **Sources:** [SRC-RENDER-014], [SRC-RENDER-003], [SRC-RENDER-019]
- **Version Notes:** Exact API names and validation output require target-branch confirmation.

### Question: How do you separate render-target memory problems from view-cost problems?

- **Category:** Rendering / Render Targets
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Measure target count/resolution/format/lifetime separately from capture/update cadence and pass timings; a render target can be expensive even when the scene view is cheap, and vice versa.
- **Strong 3-Year-Engineer Answer:** I split the diagnosis into allocation and rendering. Memory pressure comes from how many targets exist, their resolution, format, mip policy, cube faces, history buffers and lifetime. View cost comes from what a SceneCapture renders and how often it updates. I can disable capture updates while keeping the consumer alive to isolate view rendering from sampling/composition cost, and I can inventory target allocations even if GPU pass time is modest. This avoids the common mistake of lowering resolution when the real issue was six always-on cube views, or blaming captures when the memory spike came from too many long-lived HDR surfaces.
- **Common Weak Answer:** “If render targets are slow, lower the resolution.”
- **Follow-up Questions:** cube faces? history buffers? update cadence? consumer sampling? lifetime audit?
- **Hands-on Verification Task:** Build a minimap and a thumbnail path, then produce separate memory and GPU-view-cost tables for both.
- **Sources:** [SRC-RENDER-016], [SRC-RENDER-017], [SRC-RENDER-018]
- **Version Notes:** Memory accounting and renderer-path details vary by platform and target settings.

### Question: What metadata must travel with a GPU capture to make it useful?

- **Category:** Rendering / GPU Capture
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Build identity, platform/RHI/GPU/driver, quality settings, map/scenario path, event markers and the Unreal-side evidence that justified the capture.
- **Strong 3-Year-Engineer Answer:** A GPU capture without context is hard to compare or trust. I record engine/project commit, packaged build/config, platform/RHI, GPU and driver, resolution/scalability, content path, whether caches were warm or cold, and any named GPU/RDG events relevant to the feature. I also attach the Unreal-side evidence that selected this frame: `stat gpu`, GPU Visualiser, RDG Insights or a hitch timestamp. That lets another engineer reproduce the same frame and map the external capture back to an Unreal-level fix instead of debating whether the capture came from the same workload.
- **Common Weak Answer:** “The capture file already contains everything important.”
- **Follow-up Questions:** cold or warm cache? build ID? event names? same workload? why keep Unreal evidence too?
- **Hands-on Verification Task:** Capture one expensive frame and attach a repro note detailed enough for another engineer to retake the same capture.
- **Sources:** [SRC-RENDER-019], [SRC-RENDER-003], [SRC-PERF-003]
- **Version Notes:** Tool metadata and capture support differ by platform and vendor.

## Game Programming Patterns and System Design

### Question: Component versus inheritance?

- **Category:** Architecture / Composition
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Inheritance fits a stable substitutable “is-a” protocol; Components fit independently varying reusable capabilities with an explicit owner contract.
- **Strong 3-Year-Engineer Answer:** I use framework inheritance where Unreal expects it and the subtype preserves behaviour. I use Components to avoid combinatorial subclasses and share capabilities such as health across unrelated Actors. A Component still needs required-owner dependencies, authority, replication, initialisation and cleanup documented. If Components discover many siblings by casts, composition has only hidden the coupling; owner orchestration or narrow interfaces may be clearer.
- **Common Weak Answer:** “Composition is always better than inheritance.”
- **Follow-up Questions:** When use plain UObject/struct? How does Component replication work? What makes a Component too broad?
- **Hands-on Verification Task:** Refactor fire/flying/armour subclass combinations into Components and document new dependency/lifecycle costs.
- **Sources:** [SRC-ARCH-002], [SRC-EPIC-012]
- **Version Notes:** Stable design principle; UE Component APIs/lifecycle target-sensitive.

### Question: Direct call, delegate/Observer, or queued event?

- **Category:** Patterns / Communication
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Direct calls express required immediate collaboration; delegates synchronously notify local observers; queues decouple processing time and require delivery/backpressure semantics.
- **Strong 3-Year-Engineer Answer:** If one collaborator must return success, I use an explicit call/interface. If several local consumers react to committed state, a multicast delegate fits, with lifetime, re-entrancy and order treated explicitly. If work must batch, cross a phase/thread boundary or tolerate producer/consumer rate mismatch, I queue a domain message with bounded capacity, ownership, ordering, stale-target and observability rules. Replicated state/RPCs are separate network mechanisms.
- **Common Weak Answer:** “Use delegates to decouple systems and a message bus for global events.”
- **Follow-up Questions:** Are delegates asynchronous? What if a late subscriber needs current state? How handle queue overflow?
- **Hands-on Verification Task:** Route one transition through all three and measure call order, latency, allocations and teardown failures.
- **Sources:** [SRC-ARCH-003], [SRC-ARCH-006], [SRC-ARCH-013]
- **Version Notes:** Native/dynamic delegate choices and thread safety are target/API-sensitive.

### Question: State versus Strategy?

- **Category:** Patterns / Behaviour
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** State models runtime modes/transitions; Strategy substitutes an algorithm/policy behind a stable contract.
- **Strong 3-Year-Engineer Answer:** Weapon idle/firing/reloading has transition guards and enter/exit cancellation, so State is natural. Hitscan/projectile/beam targeting can be Strategies chosen by definition/equipment. Small stable cases may remain enums/branches. I define who owns transitions/policy selection, whether objects carry state, and how they are authored, loaded, tested and cleaned up.
- **Common Weak Answer:** “State changes automatically; Strategy is selected manually.”
- **Follow-up Questions:** Orthogonal states? Strategy as function versus UObject? When use StateTree?
- **Hands-on Verification Task:** Implement a weapon State machine plus two fire Strategies, then compare with one enum switch.
- **Sources:** [SRC-ARCH-005], [SRC-AI-003]
- **Version Notes:** Pattern stable; StateTree is separate UE5/version-sensitive framework.

### Question: What is the risk of using a Subsystem as a Service Locator?

- **Category:** Architecture / Services
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Subsystems scope service lifetime, but ubiquitous lookup still hides dependencies, couples tests to engine context and encourages god services.
- **Strong 3-Year-Engineer Answer:** I choose the host lifetime deliberately—World, GameInstance, LocalPlayer, Engine—and keep the domain bounded. Retrieval near a composition/orchestration boundary is reasonable. Deep core logic calling `GetSubsystem` makes dependencies invisible and may select the wrong world/client. Where it materially helps, I pass a narrow service interface/handle or data into testable logic. I avoid a general container unless actual complexity justifies it.
- **Common Weak Answer:** “Subsystems are Unreal's dependency injection.”
- **Follow-up Questions:** World versus GameInstance subsystem? How test? When is global access acceptable?
- **Hands-on Verification Task:** Implement both hidden lookup and explicit-boundary versions; test with two Worlds and without a full map.
- **Sources:** [SRC-ARCH-007], [SRC-ARCH-014]
- **Version Notes:** UE5.3–UE5.6; subsystem classes/lifetimes stable, exact initialisation details target-sensitive.

### Question: What makes a system data-driven in Unreal?

- **Category:** Architecture / Data-Driven Design
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Stable code interprets authored, validated definitions while mutable runtime state and presentation remain separate and definitions have stable loading identity.
- **Strong 3-Year-Engineer Answer:** I might put immutable item/weapon definitions in Data/Primary Data Assets, homogeneous balance rows in DataTables and curves in Curve assets. Runtime quantity, durability, owner and cooldown live in instances/values, not the shared definition. Saves and replication use stable definition IDs plus mutable state. I design soft/hard loading, missing-ID/content-version policy and editor validation; “put it in a DataTable” alone is not architecture.
- **Common Weak Answer:** “Expose variables to designers in DataTables instead of hard-coding.”
- **Follow-up Questions:** Data Asset versus DataTable? Primary Asset? How prevent runtime mutation? How validate?
- **Hands-on Verification Task:** Convert class-per-weapon content to definitions + runtime state and add invalid/missing-content tests.
- **Sources:** [SRC-ARCH-009], [SRC-ARCH-010], [SRC-ARCH-011], [SRC-ARCH-012]
- **Version Notes:** Asset Manager/configuration details are version/project-sensitive.

### Question: Design a multiplayer-ready health and damage system.

- **Category:** System Design / Health
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** A reusable authoritative capability validates structured damage, commits bounded state/death once, replicates durable state and emits local committed results for presentation.
- **Strong 3-Year-Engineer Answer:** A Health Component owns current/max health and alive generation. The server receives/triggers a damage request, validates source/rules, applies ordered resistance/armour/modifiers and clamps, then commits one result containing old/new/applied/reason/instigator and death transition. Health replicates with OnRep; UI/audio/animation observe locally. Cosmetics never replace durable state. I define reset/resurrection, friendly fire, prediction policy and save lifetime explicitly.
- **Common Weak Answer:** “Put replicated Health on a Component and multicast damage effects.”
- **Follow-up Questions:** Why not negative damage for healing? Modifier order? Client hit prediction? Death once?
- **Hands-on Verification Task:** Implement damage/heal/death/reset and inject duplicate damage, client mutation and lost cosmetic RPC.
- **Sources:** [SRC-EPIC-012], [SRC-NET-004], [SRC-NET-005]
- **Version Notes:** Generic replication baseline; damage framework/GAS alternatives are project-specific.

### Question: Design an inventory and equipment system.

- **Category:** System Design / Inventory
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Separate item definitions, mutable stack/instance state, container placement and public equipment projection behind authoritative transactional operations.
- **Strong 3-Year-Engineer Answer:** I clarify fungible versus unique items, capacity/slot rules, persistence, trading, audience and scale. Definitions use stable IDs; stacks store quantity, unique instances add instance ID/durability/rolls, and equipment references an inventory instance. The server validates add/move/split/merge/equip commands and commits deltas with operation ID/container revision. Owner-private contents and public equipped appearance are separate projections. Saves contain IDs/state/schema version with migration/missing-content policy.
- **Common Weak Answer:** “Use an inventory ActorComponent with a replicated array of item UObjects.”
- **Follow-up Questions:** Why not Actors per item? Fast Array? Atomic trade? Client drag/drop prediction?
- **Hands-on Verification Task:** Implement transactions/revisions and diagnose stale requests, duplicates, private-data leakage and renamed definitions.
- **Sources:** [SRC-ARCH-009], [SRC-ARCH-011], [SRC-NET-004]
- **Version Notes:** Fast Array and replicated subobject depth is a later specialist follow-on.

### Question: Design an interaction system.

- **Category:** System Design / Interaction
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Separate local discovery/selection and read-only prompt query from an owning-client request that the server revalidates and executes.
- **Strong 3-Year-Engineer Answer:** A trace/overlap gathers candidates and local policy selects focus. A narrow interface exposes action descriptors/prompts without mutating state. The owning Pawn/Controller sends target/action/context; server rechecks target, range/LOS, cooldown, required state/item, ownership and request rate. Committed state replicates; private rejection returns to the owner. Hold interactions define server completion, predicted progress and cancellation cleanup.
- **Common Weak Answer:** “Line trace, cast to IInteractable and call Interact through a Server RPC.”
- **Follow-up Questions:** Why route RPC through owner? Stale target? Hold prediction? Multiple actions?
- **Hands-on Verification Task:** Test moving target, spoofed distance, destroyed target, possession change and cancellation during hold.
- **Sources:** [SRC-NET-003], [SRC-NET-006], [SRC-ARCH-003]
- **Version Notes:** Generic UE networking; exact validation/prediction is game-specific.

### Question: Design a quest/objective system.

- **Category:** System Design / Quests
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Versioned quest definitions contain stable objective IDs/conditions; authoritative instances contain progress and consume committed domain events plus current-state queries.
- **Strong 3-Year-Engineer Answer:** Definitions are Data Assets with stable quest/objective IDs, graph/conditions, text references and rewards. Instances store progress, branch decisions and version. Evaluators subscribe to bounded committed events such as inventory delta or enemy defeated by tag, but also query durable state on activation/load so earlier events are not lost. Server grants rewards once and replicates relevant personal/party progress. Saves use IDs and migrations, never array index identity.
- **Common Weak Answer:** “Use an event bus and increment objectives when matching events arrive.”
- **Follow-up Questions:** Late subscription? Party membership? Definition update? Event sourcing?
- **Hands-on Verification Task:** Build “own ore then kill tagged enemy”, activate late, save mid-quest and migrate reordered objectives.
- **Sources:** [SRC-ARCH-006], [SRC-ARCH-009], [SRC-ARCH-012]
- **Version Notes:** Project architecture; Gameplay Tags/GAS integration is a separate cluster.

### Question: How do you make gameplay systems save/load friendly?

- **Category:** System Design / Persistence
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Save versioned authoritative records with stable identifiers, migrate before restore, resolve/create owners, apply state without normal side effects, rebuild derived data and notify once.
- **Strong 3-Year-Engineer Answer:** I do not serialise live pointers/widgets/effects/caches. Records include schema version, stable definition/instance/world IDs and mutable state. Load validates and migrates, ensures content/world, resolves owners, applies a controlled restore transaction, rebuilds relationships/caches, then emits ready. Missing/renamed content has explicit policy. Save writes are failure-aware/atomic as platform requires. In multiplayer, authority restores gameplay state; local client settings are a separate trust domain.
- **Common Weak Answer:** “Put serialisable UPROPERTY fields into a SaveGame object.”
- **Follow-up Questions:** How avoid re-awarding events? Async loading? Actor identity? Corrupt save? Cloud conflict?
- **Hands-on Verification Task:** Implement v1→v2 migration, missing definition and interrupted write simulations.
- **Sources:** [SRC-ARCH-009], [SRC-ARCH-011], [SRC-ARCH-012]
- **Version Notes:** Save platform APIs and packaging details require target-specific verification.

### Question: When is Object Pool useful, and when is it harmful?

- **Category:** Patterns / Performance
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** It helps measured expensive allocation/registration churn when objects reset safely; it harms through retained peak memory, stale state/handles and lifecycle/network complexity.
- **Strong 3-Year-Engineer Answer:** I profile first and compare lighter representations. A pool defines acquire/release ownership, cap/exhaustion, generation/stale-handle protection, complete reset, warm/shrink and teardown. Actor pools must account for timers, delegates, collision, effects, owner/instigator, replication identity/relevancy/dormancy and BeginPlay assumptions. A batched projectile/effect system may be better than pooled Actors.
- **Common Weak Answer:** “Pool bullets, particles and enemies because spawning is expensive.”
- **Follow-up Questions:** How detect double release? What stays resident? Networked pool? Why can Niagara be better?
- **Hands-on Verification Task:** Compare spawn/destroy, bounded pool and batched values under representative load and memory constraints.
- **Sources:** [SRC-ARCH-008], [SRC-PERF-005]
- **Version Notes:** Workload/platform/project dependent.

### Question: How do gameplay systems stay modular under production pressure?

- **Category:** Architecture / Production
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Preserve bounded ownership and contracts at real change axes, validate data, instrument transactions, refactor from observed coupling and resist speculative frameworks.
- **Strong 3-Year-Engineer Answer:** I keep definition/state/presentation and authority/audience explicit; expose commands/read-only queries/committed events; use stable IDs and versioned persistence; keep UI and content assets out of core mutation; and trace operation IDs/revisions. I accept local duplication before a premature universal abstraction. Refactoring signals include repeated central switches, sibling-cast webs, broad mutable getters, god subsystems, event cascades and representation scale mismatch. Tests sit at deterministic domain seams plus UE/network integration boundaries.
- **Common Weak Answer:** “Use SOLID, interfaces and event-driven architecture.”
- **Follow-up Questions:** When tolerate coupling? What should not be abstracted? How prevent overengineering?
- **Hands-on Verification Task:** Audit one feature's dependency/data-flow graph and justify one seam kept, one introduced and one removed.
- **Sources:** [SRC-ARCH-001], [SRC-ARCH-002], [SRC-ARCH-007], [SRC-ARCH-012]
- **Version Notes:** Stable principles; exact UE mechanisms target-sensitive.

## Assets, Loading, Cooking, and Streaming

### Question: Hard versus soft asset reference?

- **Category:** Assets / References
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** A hard reference joins eager load/dependency closure; a soft reference stores indirect identity for deferred resolution/loading and does not itself own residency.
- **Strong 3-Year-Engineer Answer:** I use hard references for genuinely co-resident required content and soft object/class references for optional or phase-loaded content. Soft is not weak: it can identify an unloaded asset by path, while weak only observes a current object. I also design cook inclusion, async request/cancellation, retained handle/strong owner, failure fallback and packaged testing. Then I audit transitive closure, not only the field type.
- **Common Weak Answer:** “Hard loads immediately; soft loads asynchronously and saves memory.”
- **Follow-up Questions:** `TSoftObjectPtr` versus `TWeakObjectPtr`? Soft class? Who keeps it resident? Cook inclusion?
- **Hands-on Verification Task:** Create hard and soft item definitions, inspect closure/startup memory, then release residency ownership after async load.
- **Sources:** [SRC-ASSET-001], [SRC-ASSET-002]
- **Version Notes:** Core semantics stable; exact async/handle APIs target-sensitive.

### Question: How do you make async asset loading lifetime-safe?

- **Category:** Assets / Async Loading
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Scope each request to an owner/generation, validate owner/world at completion, handle partial failure/cancellation and retain assets explicitly for required residency.
- **Strong 3-Year-Engineer Answer:** I store a request handle/token and state, capture a weak owner rather than an unsafe raw pointer, cancel or invalidate on replacement/travel/EndPlay, and compare generation before committing. Completion resolves all assets, handles partial failure, establishes a hard owner/managed handle, then publishes one state transition. I profile callback-side UObject/resource/spawn work because async I/O can still end in a game-thread hitch.
- **Common Weak Answer:** “Use `RequestAsyncLoad` with a weak lambda.”
- **Follow-up Questions:** Can cancellation stop all underlying I/O? What if request A completes after B? When synchronous loading is acceptable?
- **Hands-on Verification Task:** Close UI and travel during two reordered requests; prove stale completion cannot update the new world/view.
- **Sources:** [SRC-ASSET-002], [SRC-ASSET-012]
- **Version Notes:** Streamable/Asset Manager handle semantics require exact branch verification.

### Question: Asset Registry versus Asset Manager?

- **Category:** Assets / Discovery
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Registry queries unloaded asset metadata; Asset Manager adds Primary identity plus managed load, audit, cook and chunk policy.
- **Strong 3-Year-Engineer Answer:** Registry filters `FAssetData` by package/path/class/tags without loading candidate UObjects, but `GetAsset()` can turn the query into a load and discovery can be incomplete during async scan. Asset Manager addresses Primary Assets by `Type:Name`, manages Secondary dependencies/bundles and supports cook/chunk rules. A project can use Registry discovery within an Asset Manager-backed domain.
- **Common Weak Answer:** “The Asset Registry is editor-only; Asset Manager is for runtime.”
- **Follow-up Questions:** Searchable tags? Primary versus Secondary? Why not call `GetAsset` on every result?
- **Hands-on Verification Task:** Query definitions by metadata, confirm none load, then deliberately call `GetAsset` and capture the load.
- **Sources:** [SRC-ASSET-003], [SRC-ASSET-004]
- **Version Notes:** Runtime registry/cooked metadata and scan timing depend on target/configuration.

### Question: What are Primary Assets and bundles?

- **Category:** Assets / Asset Manager
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Primary Assets have stable managed IDs/rules; named bundles group soft dependencies for purpose-specific loading such as UI versus gameplay.
- **Strong 3-Year-Engineer Answer:** A Primary Asset ID is Type plus Name and lets Asset Manager discover/load/audit that definition directly. Secondary assets are reached/managed through primaries. I define bundle contracts so loading an item definition need not load all presentation/gameplay content. I configure scan paths/types, verify bundle contents/residency and audit cook/chunk ownership. IDs still need migration policy when content identity changes.
- **Common Weak Answer:** “Primary Data Assets are Data Assets that can load their references asynchronously.”
- **Follow-up Questions:** Primary Asset Label? Recursive bundles? What is management authority?
- **Hands-on Verification Task:** Create `UI`/`Game` bundles, load each separately and inspect loaded objects/memory and packaged inclusion.
- **Sources:** [SRC-ASSET-003], [SRC-ASSET-006]
- **Version Notes:** Bundle/rule details and audit UI vary across UE5.

### Question: Build, cook, stage, package, deploy, and run?

- **Category:** Assets / Packaging
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Build compiles code; cook creates target asset data; stage assembles outputs; package bundles distribution; deploy transfers to target; run executes it.
- **Strong 3-Year-Engineer Answer:** I keep these phases separate because failures have different evidence. Missing symbols/plugins are often build/configuration; missing target content is cook management; missing staged config/prerequisite is staging; archive/chunk/layout failure is packaging; SDK/device transfer is deploy; runtime crash/load is run. Packaging may orchestrate several phases but does not make the terms synonymous.
- **Common Weak Answer:** “Cooking converts assets and packaging makes the executable.”
- **Follow-up Questions:** Development versus Shipping? Pak versus IoStore? Why cook by platform?
- **Hands-on Verification Task:** Archive logs/artifacts per phase and classify injected code, cook, staging and runtime failures.
- **Sources:** [SRC-ASSET-005]
- **Version Notes:** Exact build operations/archive formats are platform/version-sensitive.

### Question: Asset works in editor but is missing in a packaged build—what do you do?

- **Category:** Assets / Debugging
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Verify exact identity and cook inclusion/management chain in packaged logs/manifests before changing timing or cooking everything.
- **Strong 3-Year-Engineer Answer:** I reproduce in clean packaged Development, capture the exact path/Primary ID and inspect cook logs, manifests/container and Asset Manager audit. I verify scan paths/rules/labels, reachable maps/references, arbitrary runtime strings, plugin/target/editor-only flags, case and redirectors. The editor sees loose content and may have assets already loaded. I repair explicit management/reference policy and add a validation/package test.
- **Common Weak Answer:** “Add the folder to Additional Asset Directories to Cook.”
- **Follow-up Questions:** Why might case matter? Soft path and cook? How find the owning chunk?
- **Hands-on Verification Task:** Exclude an unmanaged path asset deliberately, diagnose from artifacts, then fix via Primary Asset policy.
- **Sources:** [SRC-ASSET-003], [SRC-ASSET-005], [SRC-ASSET-006]
- **Version Notes:** Cook manifests/container inspection tools vary by target.

### Question: What are redirectors, and how should a team handle them?

- **Category:** Assets / Content Maintenance
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Rename/move leaves an old-path redirector; fix-up resaves referencers to the new asset and removes the redirector when all can be updated.
- **Strong 3-Year-Engineer Answer:** I move assets through the editor, include the move/redirector/reference resaves coherently in source control, and run Fix Up or source-control-aware ResavePackages commandlet. I do not delete redirector files manually. I check unloaded/read-only packages, arbitrary config strings, collisions and packaged/case-sensitive behaviour, then validate references/cook.
- **Common Weak Answer:** “Redirectors are temporary aliases; delete them after moving assets.”
- **Follow-up Questions:** Why fix-up can fail? Redirector chain? Cross-project migration?
- **Hands-on Verification Task:** Move an asset referenced by unloaded maps, inspect redirector, fix all packages and verify packaged load.
- **Sources:** [SRC-ASSET-007]
- **Version Notes:** Commandlet/source-control options require target/toolchain verification.

### Question: What is DDC, and should it be source controlled?

- **Category:** Assets / Build Performance
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** DDC caches regenerable platform/settings-derived asset data and normally should not be authoritative source-controlled content.
- **Strong 3-Year-Engineer Answer:** Shaders, textures, meshes and other assets generate derived results keyed by inputs/version/platform. Local/shared/remote cache layers avoid regeneration. Cold or misconfigured DDC increases startup/compile/cook time and can expose hitches; deletion can diagnose corruption but forces rebuild. I inspect backend/key/version/shared-cache health rather than treating cache as source data.
- **Common Weak Answer:** “DDC stores compiled shaders and can always be safely deleted.”
- **Follow-up Questions:** Shared DDC? Zen? What invalidates a key? Why not copy arbitrary DDC between projects?
- **Hands-on Verification Task:** Measure cold/warm derived-data workflows and verify source assets remain sufficient to regenerate.
- **Sources:** [SRC-ASSET-008]
- **Version Notes:** DDC/Zen backends and configuration evolve across UE5.

### Question: Level streaming versus World Partition?

- **Category:** Assets / World Streaming
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Level streaming explicitly loads/unloads level packages; World Partition spatially divides a persistent UE5 world into cells driven by streaming sources, with Data Layer/OFPA/HLOD integration.
- **Strong 3-Year-Engineer Answer:** I choose from world structure, collaboration, authoring and runtime control—not because one is newer. With either, durable state cannot live only in unloadable Actors. I define preload source/range/readiness, spatial versus always-loaded Actors, cross-cell references, async cleanup, server/client semantics, memory/I/O concurrency and teleport handling. In UE5, World Composition is legacy and World Partition is recommended for many large-world projects.
- **Common Weak Answer:** “World Partition replaces level streaming in UE5.”
- **Follow-up Questions:** `Is Spatially Loaded`? Streaming source? Cross-cell Actor reference? Data Layers?
- **Hands-on Verification Task:** Preload a teleport destination, wait for collision/gameplay readiness, then test unload callbacks and durable state.
- **Sources:** [SRC-ASSET-009], [SRC-ASSET-010]
- **Version Notes:** World Partition runtime hash/settings are UE5 version-sensitive.

### Question: How do you diagnose an asset-loading hitch?

- **Category:** Assets / Profiling
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Mark request-to-first-use stages, compare cold/warm packaged target captures, inspect I/O/package/object/resource/callback work and dependency closure.
- **Strong 3-Year-Engineer Answer:** I correlate request, queue/start, package/object ready, resource ready, callback commit and first visible use in Insights/platform I/O tools. I inspect whether the main thread stalls on sync fallback, serialisation/fixups, registration/spawn, shader/texture resource work or GC after unload. I audit transitive closure and start timing/priorities. I reproduce cold cache/DDC and target storage/memory because developer warm-cache results hide stalls.
- **Common Weak Answer:** “Use async loading and preload the asset earlier.”
- **Follow-up Questions:** Can async loading still hitch? How distinguish I/O from completion work? Why can first use be later than load complete?
- **Hands-on Verification Task:** Create one large dependency root and callback spawn burst, then separate and verify both causes in traces.
- **Sources:** [SRC-ASSET-002], [SRC-ASSET-012]
- **Version Notes:** Loading trace channels/tool panels vary by UE minor/platform.

## Build Systems, Modules, Plugins, and Editor Tools

### Question: What do UBT, UHT, the compiler, and linker each do?

- **Category:** Build / Toolchain
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** UBT constructs the target/module build graph, UHT generates reflection glue, the compiler builds translation units, and the linker resolves symbols into binaries.
- **Strong 3-Year-Engineer Answer:** UBT evaluates Target.cs, Build.cs and descriptors, then invokes tools with the correct target/configuration environment. UHT parses supported reflected declarations and emits generated code consumed by native compilation. Compiler errors concern syntax/types/includes; linker errors concern definitions/exports/linked binaries. Module loading and cooking/package are later stages. I start from the first causal diagnostic and classify its stage before changing code or deleting build products.
- **Common Weak Answer:** “UBT compiles C++ and UHT compiles Unreal macros.”
- **Follow-up Questions:** Who owns generated project files? What produces `.generated.h`? What causes unresolved externals?
- **Hands-on Verification Task:** Trigger one UHT, compile, link and module-load error and write the minimal evidence-led fix for each.
- **Sources:** [SRC-BUILD-001], [SRC-BUILD-007]
- **Version Notes:** Stable architecture; commands/generated output layout vary by UE version/platform.

### Question: Target versus module in Unreal?

- **Category:** Build / Architecture
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** A target describes an application/configuration composition; a module is a separately built functionality/API unit in its dependency graph.
- **Strong 3-Year-Engineer Answer:** TargetRules selects Game, Client, Server, Editor or Program and global build behaviour. ModuleRules defines one module's dependencies, includes, definitions and libraries. A target includes a set of modules; a plugin can contain multiple modules. “Development Editor” combines configuration and target. I use modules to enforce runtime/editor/domain boundaries, not merely folders.
- **Common Weak Answer:** “Targets are executable projects and modules are DLLs.”
- **Follow-up Questions:** Monolithic build? Module loading phase? Why build Server target?
- **Hands-on Verification Task:** Draw modules included by Game, Editor and Server targets and prove the differences with builds.
- **Sources:** [SRC-BUILD-002], [SRC-BUILD-003], [SRC-BUILD-004]
- **Version Notes:** Exact TargetRules/ModuleRules properties are version-sensitive.

### Question: Public versus Private module dependency?

- **Category:** Build / Modules
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Public dependencies are exposed by public headers/API to consumers; private dependencies are implementation-only and should not propagate.
- **Strong 3-Year-Engineer Answer:** If my public header uses a type whose definition consumers need, its module is normally a public dependency. If use is confined to cpp/private implementation, it is private. Public/private folders control cross-module header visibility, not C++ access. I make public headers self-contained, forward-declare where legal, avoid another module's Private path and verify non-unity consumer builds. Adding everything public hides dependency design and expands compile fan-out.
- **Common Weak Answer:** “Public means other modules can include it; Private means only this module.”
- **Follow-up Questions:** Forward declaration limits? Transitive includes? Include path versus dependency?
- **Hands-on Verification Task:** Move a Slate type from a public API to private implementation and compare consumer dependencies/rebuild fan-out.
- **Sources:** [SRC-BUILD-002], [SRC-BUILD-003]
- **Version Notes:** Stable module principle.

### Question: Why can code compile in unity builds but fail non-unity or on CI?

- **Category:** Build / C++ Dependencies
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Unity translation units can supply accidental neighbouring includes/definitions and hide missing IWYU or ODR problems.
- **Strong 3-Year-Engineer Answer:** Each file should include what it directly uses. Unity combines cpp files for speed, so an unrelated include or macro state may leak into another file. Different grouping/order, clean agents and non-unity expose the defect. I fix self-sufficient headers and Build.cs dependencies rather than adding a broad PCH or relying on generated IDE order. CI should include a non-unity/IWYU-oriented lane where cost permits.
- **Common Weak Answer:** “CI uses a different compiler or stale cache.”
- **Follow-up Questions:** PCH? ODR? Forward declaration? Why clean builds differ?
- **Hands-on Verification Task:** Remove a direct include, observe unity success/non-unity failure, and repair the exact dependency.
- **Sources:** [SRC-BUILD-001], [SRC-BUILD-002]
- **Version Notes:** Unity/IWYU defaults vary by target/engine branch.

### Question: Header is visible, but linker reports unresolved external—why?

- **Category:** Build / Linking
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Declaration compiled, but the exact definition/export or owning binary dependency is absent for that target/configuration.
- **Strong 3-Year-Engineer Answer:** I compare mangled-signature details: definition exists and matches const/static/template/signature; the owning module is linked; public crossing symbol has the module API macro; implementation is not excluded by platform/configuration; template/inline definition is visible; third-party library/runtime variant matches. Then I rule out stale binary. Header visibility alone does not link a symbol.
- **Common Weak Answer:** “Add the module to PublicDependencyModuleNames.”
- **Follow-up Questions:** API macro? Templates? Static library architecture? Monolithic versus modular?
- **Hands-on Verification Task:** Omit a definition, API export and module dependency separately; distinguish all three linker cases.
- **Sources:** [SRC-BUILD-002], [SRC-BUILD-003]
- **Version Notes:** Platform linker diagnostics differ.

### Question: Runtime module versus Editor module?

- **Category:** Build / Architecture
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Runtime contains code/types valid in packaged targets; Editor contains UnrealEd/tool UI/asset authoring extensions and may depend on Runtime, never vice versa.
- **Strong 3-Year-Engineer Answer:** Shared runtime asset schemas live in Runtime with no UnrealEd/PropertyEditor/AssetTools dependencies. Details customisations, validators and asset actions live in Editor. `WITH_EDITOR` can guard narrow code but does not erase a runtime Build.cs dependency on Editor. I prove the boundary by building Game/Server/Shipping and packaging, not only Editor. Runtime content must not reference UClasses defined only in Editor.
- **Common Weak Answer:** “Wrap editor includes in `#if WITH_EDITOR`.”
- **Follow-up Questions:** `WITH_EDITORONLY_DATA`? Developer module? Dedicated server? Editor plugin content?
- **Hands-on Verification Task:** Deliberately expose an Editor class from Runtime, then refactor and prove package exclusion.
- **Sources:** [SRC-BUILD-002], [SRC-BUILD-005]
- **Version Notes:** Host types/macros and stripping semantics require exact target verification.

### Question: What belongs in `.uplugin`, `.uproject`, Build.cs, and Target.cs?

- **Category:** Build / Configuration
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Descriptors declare discoverable project/plugin/modules/compatibility; Build.cs defines module build contracts; Target.cs defines application target composition/global rules.
- **Strong 3-Year-Engineer Answer:** The plugin descriptor states metadata, code/content capability, dependencies and module type/loading compatibility. Project descriptor lists project modules/plugins. Each module's Build.cs defines dependencies/includes/definitions/libraries/platform conditions. Target.cs defines Game/Editor/Client/Server/Program target rules. I avoid obsolete `AdditionalDependencies` descriptor workarounds when a Build.cs dependency is the real contract.
- **Common Weak Answer:** “Uplugin enables modules, Build.cs lists libraries, Target.cs chooses Development or Shipping.”
- **Follow-up Questions:** Loading phase? Platform allow/deny lists? Plugin content? Primary project module?
- **Hands-on Verification Task:** Add a two-module plugin manually and trace every descriptor/rule field into Game and Editor builds.
- **Sources:** [SRC-BUILD-002], [SRC-BUILD-004], [SRC-BUILD-005]
- **Version Notes:** Descriptor/rule field names evolve; validate UE5.6 schema/source.

### Question: When is Live Coding safe, and when should you restart?

- **Category:** Build / Iteration
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** It is best for implementation changes; reflected layout, constructors/defaults/subobjects, static caches/destructors and build graph changes require clean restart/rebuild validation.
- **Strong 3-Year-Engineer Answer:** Live Coding patches binaries and can reinstate UObjects. I keep reinstancing enabled as recommended, but invalidate cached pointers on reload and expect existing instances/defaults to differ from clean startup. Build.cs/Target.cs/descriptors cannot be proven by patching. For UPROPERTY/UFUNCTION/layout, constructor, CDO/subobject, static lifetime or shutdown crashes, I close the editor and do a normal clean-enough rebuild. I never treat Live Coding success as Shipping/package proof.
- **Common Weak Answer:** “Live Coding is safe for cpp changes but not header changes.”
- **Follow-up Questions:** Object reinstancing? Existing constructor defaults? Why shutdown crash? Hot Reload?
- **Hands-on Verification Task:** Patch function, constructor, reflected field and static cached pointer; compare existing/new/clean-start behaviour.
- **Sources:** [SRC-BUILD-006]
- **Version Notes:** UE5.6 target; availability differs on console/mobile and settings evolve.

### Question: Module startup/shutdown registration—what can go wrong?

- **Category:** Build / Editor Extensions
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Registrations can duplicate or outlive modules across reload; shutdown order may make dependencies unavailable, so startup/unregister ownership must be symmetric and guarded.
- **Strong 3-Year-Engineer Answer:** I record handles/registered names for tabs, menus, detail layouts, styles, commands, validators and delegates, and unregister them in ShutdownModule only if the owning service/module is still available. I avoid force-loading a dependency during shutdown. Static shared state and raw delegates are reload hazards. I test enable/disable, Live Coding/module reload and editor exit, and make registrations idempotent where APIs allow.
- **Common Weak Answer:** “Unregister everything in `ShutdownModule`.”
- **Follow-up Questions:** Loading phase? Static style set? Delegate handle? Why duplicate tabs after Live Coding?
- **Hands-on Verification Task:** Intentionally omit detail/delegate unregister, reload module and diagnose duplicate/stale callback behaviour.
- **Sources:** [SRC-BUILD-002], [SRC-BUILD-006], [SRC-BUILD-010]
- **Version Notes:** Extension registration APIs vary across UE5.

### Question: Editor Utility Widget, Python, C++ editor module, or commandlet?

- **Category:** Editor Tools / Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Choose by audience and execution: interactive designer UI, pipeline scripting, durable native integration, or headless CI batch processing.
- **Strong 3-Year-Engineer Answer:** Editor Utility Widgets offer rapid UMG-based tabs; Python is strong pipeline/DCC glue but editor-only and version-sensitive; C++ modules fit maintained integrated tools/custom types; commandlets fit repeatable headless work in a reduced environment. I put deterministic analyse/transform logic below the front end so UI and CI share it. I do not assume World, Slate or loaded assets in a commandlet.
- **Common Weak Answer:** “Use Blueprint for simple tools, Python for batch, and C++ for performance.”
- **Follow-up Questions:** Can Python run packaged? Commandlet environment? How share implementation? Beta/experimental caveats?
- **Hands-on Verification Task:** Expose one asset audit through Widget and commandlet front ends with identical results.
- **Sources:** [SRC-BUILD-008], [SRC-BUILD-011], [SRC-BUILD-012]
- **Version Notes:** Utility Widget/Python status and APIs are version/plugin-sensitive.

### Question: How do you design a safe batch asset modification tool?

- **Category:** Editor Tools / System Design
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Separate analysis from mutation and provide dry run, preflight, transactions, source-control handling, cancellation, idempotence, explicit saves and structured reports.
- **Strong 3-Year-Engineer Answer:** I query candidates without loading unnecessarily, analyse and preview exact changes, validate type/version/write/checkout/collision preconditions, then mutate through supported editor/property APIs. Interactive operations use transactions where supported, mark/save only intended packages, isolate per-asset failure and emit changed/skipped/failed reasons. Large jobs are cancellable/resumable/idempotent. The core operation also runs headless for CI, followed by side-effect-free validation.
- **Common Weak Answer:** “Loop selected assets, call Modify, change fields, mark dirty and save.”
- **Follow-up Questions:** Undo limits? Rename/move? Partial failure? Source control? Multi-user? Performance?
- **Hands-on Verification Task:** Process 1,000 fixtures with dry run, read-only failures, cancellation at half, rerun and audit zero unintended changes.
- **Sources:** [SRC-BUILD-009], [SRC-BUILD-011], [SRC-ASSET-004]
- **Version Notes:** Editor/source-control transaction APIs are version/provider-sensitive.

### Question: Validation versus automatic repair?

- **Category:** Editor Tools / Validation
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Validation deterministically reports invariant violations; repair is an explicit transactional operation with preview and recovery.
- **Strong 3-Year-Engineer Answer:** I use `IsDataValid` for project asset-owned invariants or an Editor Validator for cross-cutting classes. Rules are side-effect-free, actionable, warning/error consistent and runnable on save/manual/CI commandlet. They handle missing/unloaded dependencies and have valid/invalid fixtures. I never mutate source assets inside CI validation; a separate tool can offer controlled repair. Failures include asset/property, expected condition and remediation.
- **Common Weak Answer:** “Validators check assets and can fix simple naming errors automatically.”
- **Follow-up Questions:** `IsDataValid` versus validator? Blueprint/Python registration? Performance? Validate dependencies?
- **Hands-on Verification Task:** Add stable-ID/value/reference validators and a separate idempotent fix action, then run command-line validation.
- **Sources:** [SRC-BUILD-009], [SRC-BUILD-014]
- **Version Notes:** Python/Blueprint validator discovery and API are plugin/version-sensitive.

### Question: AutomationTool versus BuildGraph?

- **Category:** Build / Release Automation
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** AutomationTool is Unreal's command-line automation runner; BuildGraph is a declarative graph format for larger repeatable build-farm pipelines.
- **Strong 3-Year-Engineer Answer:** I use AutomationTool/RunUAT commands such as BuildCookRun for direct build, cook, stage, package, deploy and run operations. BuildGraph sits above that when a release needs named parameters, dependency nodes, agents, labels, artifact publication and multiple platform/configuration jobs. A one-off package command can be AutomationTool-only; a studio release pipeline should expose dependencies and artifacts as graph data so failures, retries and ownership are visible.
- **Common Weak Answer:** “BuildGraph is just a build script.”
- **Follow-up Questions:** What is BuildCookRun? What is an artifact? Why avoid one long shell command?
- **Hands-on Verification Task:** Convert a local package command into a two-node BuildGraph or documented RunUAT pipeline with explicit build and package artifacts.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016], [SRC-BUILD-017]
- **Version Notes:** BuildGraph tasks and RunUAT flags are branch/platform/build-farm sensitive.

### Question: How would you design release pipeline gates for an Unreal project?

- **Category:** Build / CI
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Gate progressively from compile and validation to clean cook/package, launch smoke, asset/runtime dependency proof, symbols and crash reporting.
- **Strong 3-Year-Engineer Answer:** I separate fast pre-submit gates from nightly clean and release-candidate gates. The release path should clean sync/build, run UHT/compile/link for the intended targets, run side-effect-free validation commandlets, cook intended content, stage/package/archive, launch on a clean machine, verify third-party runtime files, archive symbols and metadata, and force a controlled packaged crash to prove symbolication. Each gate should publish logs/manifests/artifacts and fail on first causal evidence rather than hiding errors behind broad workarounds.
- **Common Weak Answer:** “Run package in CI and check that it succeeds.”
- **Follow-up Questions:** Where do symbols belong? What should be per-submit versus nightly? What is clean-machine proof?
- **Hands-on Verification Task:** Write a tiered CI matrix for Development Editor, Development Game, Shipping Game and Server where relevant, then justify every expensive lane.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016], [SRC-BUILD-017], [SRC-BUILD-018], [SRC-BUILD-014]
- **Version Notes:** Platform signing, certification and symbol handling are project/platform sensitive.

### Question: Packaged build crashes but the editor works. What is your workflow?

- **Category:** Build / Packaged Runtime Debugging
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Reproduce the exact packaged target, preserve logs/symbols/build metadata, classify whether the failure is content, module, staged dependency, config or platform runtime, then fix and rerun the gate.
- **Strong 3-Year-Engineer Answer:** I do not debug this as an editor problem first. I capture the packaged log, callstack, build ID, target/config/platform, command line and staged manifest. Then I check common differences: uncooked assets, editor-only module or content dependency, missing runtime DLL/shared library, target-only config, stripped data, platform RHI/device profile and stale/wrong symbols. If the crash is unsymbolicated, fixing symbol/archive plumbing is part of the release bug. The final proof is a clean-machine packaged launch and the same crash/smoke gate passing.
- **Common Weak Answer:** “Delete Intermediate/Saved and repackage.”
- **Follow-up Questions:** How do you identify missing staged files? What if the callstack is wrong? What if it only happens in Shipping?
- **Hands-on Verification Task:** Inject one editor-only dependency and one missing staged runtime file, then diagnose both from package logs/manifests without opening the editor.
- **Sources:** [SRC-BUILD-017], [SRC-BUILD-018], [SRC-BUILD-005], [SRC-ASSET-005], [SRC-ASSET-006]
- **Version Notes:** Crash reporter, symbol format and package layout vary by platform and project policy.

### Question: How do you prove a third-party runtime dependency is release-ready?

- **Category:** Build / Third-Party Integration
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Prove compile/link/runtime staging for every target platform/configuration from a clean machine, including licensing, symbols and failure diagnostics.
- **Strong 3-Year-Engineer Answer:** I separate header visibility, link-time libraries and runtime-loaded files. Build.cs should declare include paths, libraries, delay-load/runtime dependencies and platform conditions explicitly. The proof is not “it works on my editor machine”; it is a clean agent package whose staged directory contains the exact binary variant and no global install assumptions. I verify architecture/configuration match, packaging logs, launch, failure message quality, license placement and symbol/debug artifact handling where applicable.
- **Common Weak Answer:** “Add the DLL next to the exe.”
- **Follow-up Questions:** Static versus dynamic library? Delay-load? Dedicated server? Mac/Linux rpath/install names?
- **Hands-on Verification Task:** Add a mock External module with a staged runtime file and make CI fail if the packaged clean-machine launch cannot find it.
- **Sources:** [SRC-BUILD-002], [SRC-BUILD-017], [SRC-BUILD-015]
- **Version Notes:** Platform dynamic-library and signing rules require target verification.

### Question: How should crash reporting and symbols fit into a release pipeline?

- **Category:** Build / Operations
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Treat symbols, build metadata, logs and forced-crash symbolication as release artifacts, not optional post-release cleanup.
- **Strong 3-Year-Engineer Answer:** A crash report is only useful when it can be tied to the exact build, configuration, platform, changelist and symbols. I archive symbols with package artifacts and validate the crash path before distribution by forcing a controlled packaged crash. The expected report should include a symbolicated stack, log tail and enough custom context to route triage. Endpoint/privacy/platform constraints are policy decisions, but the pipeline should prove whatever approved crash flow the project uses.
- **Common Weak Answer:** “Shipping crashes are handled by Crash Reporter.”
- **Follow-up Questions:** What makes a crash unsymbolicated? What metadata should be included? How do privacy/platform rules affect this?
- **Hands-on Verification Task:** Force a packaged crash, then verify symbolicated callstack, build ID, log tail and routing against the archived artifact set.
- **Sources:** [SRC-BUILD-018], [SRC-BUILD-015], [SRC-BUILD-017]
- **Version Notes:** Symbol upload, endpoints and crash client behaviour are studio/platform sensitive.

### Question: How would you design a practical Unreal CI matrix?

- **Category:** Build / CI
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use tiers: fast pre-submit confidence, nightly clean proof, release-candidate artifact proof and platform/certification jobs where needed.
- **Strong 3-Year-Engineer Answer:** I keep pre-submit cheap: compile key targets, run unit/automation/validation tests and maybe a limited cook. Nightly clean jobs prove no hidden local cache or unity/include assumptions, then cook/package/launch representative targets. Release-candidate jobs archive artifacts, symbols and manifests, run smoke tests, crash proof and platform packaging. Platform/certification lanes are narrower but more expensive. The matrix is justified by failure cost and feedback time; if a job is too slow, I split it rather than letting teams bypass it.
- **Common Weak Answer:** “Run all platforms on every commit.”
- **Follow-up Questions:** Which jobs need clean workspaces? Where do non-unity builds belong? What should block submit versus release?
- **Hands-on Verification Task:** Build a matrix with four tiers and assign each gate a maximum runtime, artifact output and owner.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016], [SRC-BUILD-017], [SRC-BUILD-014]
- **Version Notes:** CI capacity, platform hardware and build-farm tooling are studio-specific.

## ECS and MassEntity

### Question: Why use ECS or MassEntity at all?

- **Category:** MassEntity / Architecture
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** To process large populations with shared data shapes in contiguous archetype/chunk batches and decouple simulation fidelity from expensive object representation.
- **Strong 3-Year-Engineer Answer:** Mass pays when many entities share stable compositions and processors access a few fragment columns sequentially. It reduces per-object scheduling/indirection and enables simulation/representation LOD. The trade is specialised authoring/debugging, structural-change cost, less natural per-object polymorphism and plugin/version risk. I compare the same behaviour/fidelity against Actors and measure simulation, structure, bridges and rendering separately.
- **Common Weak Answer:** “ECS is faster because data is contiguous and avoids OOP overhead.”
- **Follow-up Questions:** When not use Mass? Does Mass guarantee multithreading? What about unique agents?
- **Hands-on Verification Task:** Compare 100/1,000/10,000 Actor and Mass agents with equivalent movement and representation.
- **Sources:** [SRC-MASS-001], [SRC-MASS-002]
- **Version Notes:** UE5-specific/plugin-dependent; exact performance is workload/branch-specific.

### Question: Actor/Component architecture versus MassEntity?

- **Category:** MassEntity / Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Actors are rich independent UObject world identities; Mass entities are lightweight compositions processed in batches, often with Actor/ISM/no representation by LOD.
- **Strong 3-Year-Engineer Answer:** I keep players, bosses and richly interactive physics/Blueprint objects as Actors unless evidence says otherwise. I use Mass for large homogeneous populations with bounded batchable behaviour. A hybrid can keep Mass as simulation identity and promote nearby/selected entities to Actors. I define one authoritative writer during promotion, stable ID mapping, pool reset and transition hysteresis.
- **Common Weak Answer:** “Actors are for important entities and Mass is for background crowds.”
- **Follow-up Questions:** Can Mass entity have an Actor? Who owns transform? How handle interaction/replication?
- **Hands-on Verification Task:** Promote/demote a selected Mass agent while preserving state and invalidating external Actor references safely.
- **Sources:** [SRC-MASS-001], [SRC-MASS-002], [SRC-MASS-005]
- **Version Notes:** Representation APIs/traits vary significantly across UE5.

### Question: Fragment, entity, archetype, and chunk?

- **Category:** MassEntity / Core Model
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Fragment is data; entity is manager-local identity plus composition; archetype groups identical composition; chunks store archetype entities with contiguous fragment columns.
- **Strong 3-Year-Engineer Answer:** Processors query archetypes by fragment/tag requirements and iterate chunk arrays. This turns many pointer-chasing object calls into dense batch access. Adding/removing a fragment changes composition and moves an entity to another archetype. Entity handles are not durable save/network IDs and fragment views cannot be cached beyond execution.
- **Common Weak Answer:** “An entity is an ID, Components are data, and archetypes are groups of Components.”
- **Follow-up Questions:** Tag? Shared fragment? Chunk capacity? Archetype explosion?
- **Hands-on Verification Task:** Inspect two compositions, their archetypes/chunks and migration after adding a tag.
- **Sources:** [SRC-MASS-001]
- **Version Notes:** Stable conceptual model; concrete APIs version-sensitive.

### Question: Mass tag versus Boolean fragment field?

- **Category:** MassEntity / Data Design
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** A tag changes composition and enables query grouping; a Boolean stays in data and avoids migration but requires filtering/branching.
- **Strong 3-Year-Engineer Answer:** I use a tag for relatively stable, query-selective categorical state that materially changes which processors run. I use a field for high-frequency toggles or values already processed in the same batch. Every optional tag combination can add archetypes, and toggling tags queues structural moves. I measure migrations, archetypes, chunk occupancy and branch/query cost.
- **Common Weak Answer:** “Tags are more efficient because they store no data.”
- **Follow-up Questions:** Structural command? `None` requirement? How avoid archetype explosion?
- **Hands-on Verification Task:** Compare per-frame moving-tag toggles with a Boolean field across 10,000 entities.
- **Sources:** [SRC-MASS-001], [SRC-MASS-003]
- **Version Notes:** Exact structural command/query APIs target-sensitive.

### Question: Shared fragment versus chunk fragment versus entity fragment?

- **Category:** MassEntity / Data Design
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Entity fragment is per entity; shared fragment is shared by a configured group; chunk fragment is one value per storage chunk.
- **Strong 3-Year-Engineer Answer:** Transform/health are per entity. Movement configuration or representation descriptor may be shared, but high-cardinality shared values fragment batches. Chunk fragments hold managerial batch data such as LOD grouping. I choose cardinality from access and mutation patterns and keep shared configuration effectively immutable where possible.
- **Common Weak Answer:** “Shared fragments are static config; chunk fragments are cached processor data.”
- **Follow-up Questions:** Const shared? How can shared values create archetypes? When use subsystem instead?
- **Hands-on Verification Task:** Add one unique shared value per entity and observe archetype/chunk degradation, then canonicalise values.
- **Sources:** [SRC-MASS-001]
- **Version Notes:** Exact shared/chunk APIs and grouping semantics are version-sensitive.

### Question: How do Mass queries and processors work?

- **Category:** MassEntity / Processing
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** A processor declares read/write/presence requirements in an EntityQuery, which caches matching archetypes and supplies chunk fragment views for batch execution.
- **Strong 3-Year-Engineer Answer:** Requirements include All/Any/None/Optional and ReadOnly/ReadWrite. Precise access enables dependency analysis and possible parallel scheduling. Inside chunk iteration I fetch views once and loop indices without per-entity UObject lookups/allocations. Processor phase/group/prerequisites define pipeline order. Query construction/registration changed across UE5, so I verify the branch.
- **Common Weak Answer:** “Processors run systems on every entity that has the required fragments.”
- **Follow-up Questions:** Optional fragments? Subsystem requirement? Parallel chunks? Why declare read-only accurately?
- **Hands-on Verification Task:** Build movement query, add an exclusion tag and inspect matched archetypes/entity counts.
- **Sources:** [SRC-MASS-003], [SRC-MASS-004]
- **Version Notes:** Highly API-sensitive between UE5 minor versions.

### Question: Why defer structural changes in Mass?

- **Category:** MassEntity / Processing
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Composition changes move storage/archetype and can invalidate active iteration; command buffers batch them at a safe flush point.
- **Strong 3-Year-Engineer Answer:** Adding/removing tags/fragments or destroying entities changes the set/storage being iterated. I enqueue the operation in the execution context/command buffer and treat it as visible after the documented flush. I avoid per-frame structural churn by using fields, batching or stable tiers where appropriate. Deferred does not mean free; I profile command count and migrations.
- **Common Weak Answer:** “Mass is multithreaded, so entity changes must be queued.”
- **Follow-up Questions:** When does flush occur? Can next processor see change? How initialise added fragment?
- **Hands-on Verification Task:** Implement death as deferred tag/fragment transition and compare a deliberately high-churn design.
- **Sources:** [SRC-MASS-001]
- **Version Notes:** Flush/command APIs and phase details require target source confirmation.

### Question: What is a Mass Trait?

- **Category:** MassEntity / Authoring
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** A Trait is an entity-template authoring recipe that adds/configures fragments/tags/shared data for a capability; processors provide runtime logic.
- **Strong 3-Year-Engineer Answer:** Traits compose entity configs/templates and should validate required/conflicting traits and defaults. They are not UActorComponents and do not own per-frame behaviour. A movement Trait can add velocity/config fragments and ensure the template's expected composition, while movement processors query those fragments globally.
- **Common Weak Answer:** “Traits are Mass Components that bundle fragments and processors.”
- **Follow-up Questions:** Entity Config/template? Spawner? Trait inheritance/conflicts?
- **Hands-on Verification Task:** Create movement and representation Traits with one conflict/missing-dependency validation error.
- **Sources:** [SRC-MASS-001], [SRC-MASS-002]
- **Version Notes:** Trait/template APIs/assets vary across UE5.

### Question: Visual LOD versus simulation LOD in Mass?

- **Category:** MassEntity / LOD
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Visual LOD selects Actor/ISM/none representation; simulation LOD changes calculation frequency/fidelity; they solve different budgets.
- **Strong 3-Year-Engineer Answer:** A nearby agent may use an Actor and full-rate steering, medium an ISM with reduced decisions, far no representation with coarse updates. LOD needs hysteresis, maximum counts, transition budget and gameplay invariants. Representation cost can dominate even when Mass simulation is cheap, so I profile the layers separately. Replication LOD can also vary per viewer where supported.
- **Common Weak Answer:** “Mass LOD replaces Actors with ISMs and updates far agents less often.”
- **Follow-up Questions:** Actor pooling? Significance? Replication LOD? Prevent transition thrash?
- **Hands-on Verification Task:** Implement four tiers and chart simulation/representation counts, transitions and frame cost over a camera path.
- **Sources:** [SRC-MASS-002], [SRC-MASS-005]
- **Version Notes:** MassGameplay representation/LOD is plugin- and version-sensitive.

### Question: A Mass entity is not processed—how do you debug it?

- **Category:** MassEntity / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Verify World/handle, exact composition, query requirements/registration, processor activation/order and deferred-command flush, then inspect matched archetypes in target tools.
- **Strong 3-Year-Engineer Answer:** I first confirm correct World entity manager and a valid handle. I dump composition/archetype and compare All/Any/None/Optional plus shared/tag/subsystem requirements. Then I verify the query is registered for the branch, processor active/auto-executed in the expected phase/group, and the structural command has flushed. Mass debugger capability differs by version, so I use available debugger/log/source rather than random fragment additions.
- **Common Weak Answer:** “Check whether the entity has all fragments required by the processor.”
- **Follow-up Questions:** Wrong shared fragment? Processor dependency? Cached archetypes? World subsystem?
- **Hands-on Verification Task:** Create six missing-processing causes and diagnose them from a blind trace/composition dump.
- **Sources:** [SRC-MASS-003], [SRC-MASS-004], [SRC-MASS-006], [SRC-MASS-007]
- **Version Notes:** Later Mass Debugger features may not exist in UE5.6.

### Question: How do you profile Mass fairly against Actors?

- **Category:** MassEntity / Performance
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Hold behaviour/visual fidelity constant and separate simulation, structural, bridge and representation costs across population scales.
- **Strong 3-Year-Engineer Answer:** I record per-processor total/time per entity, queries/chunks/entities, occupancy, archetypes/shared variants, deferred migrations, game-thread bridges, memory/entity, LOD populations, Actor/ISM counts, Game/Draw/GPU and network estimate. I compare the same camera/behaviour and include warm-up. Removing skeletal animation/collision from only the Mass case is a product fidelity change, not an engine-only win.
- **Common Weak Answer:** “Spawn the same number and compare FPS/CPU.”
- **Follow-up Questions:** What indicates archetype explosion? Structural churn? Representation bottleneck? When manager-of-structs wins?
- **Hands-on Verification Task:** Produce an Actor/Mass benchmark with one intentionally unfair comparison, identify it and correct it.
- **Sources:** [SRC-MASS-001], [SRC-MASS-002], [SRC-PERF-003]
- **Version Notes:** Hardware/content/branch dependent.

### Question: What should a broad engineer know about Mass replication?

- **Category:** MassEntity / Networking
- **Priority:** P3
- **Expected Depth:** D2
- **Short Answer:** It is a specialised, evolving MassGameplay path using server authority and viewer relevance/LOD; custom data and stable identity require target-specific C++ design.
- **Strong 3-Year-Engineer Answer:** I would not assume Actor replication applies automatically to fragments. I define authoritative server state, stable network identity/remapping, per-viewer relevance/update tiers and which fragment projections replicate. I avoid dual Actor/Mass authoritative replication. The docs retain historical experimental notes and maintained APIs move, so before production I inspect the UE5.6 plugin/source, platform support and sample implementation.
- **Common Weak Answer:** “Mass replication is experimental and uses replication LOD to send entities.”
- **Follow-up Questions:** Local handle versus network ID? Actor promotion? Client prediction? Iris relation?
- **Hands-on Verification Task:** Write a replication design and source-verification checklist; prototype only after confirming target support.
- **Sources:** [SRC-MASS-002], [SRC-NET-001]
- **Version Notes:** Experimental/status/API must be verified in the exact UE5.3–UE5.6 branch.

## Gameplay Ability System

### Question: When would you choose GAS over a bespoke ability Component?

- **Category:** GAS / Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Choose GAS when interacting abilities need shared tags, costs, cooldowns, effects, attributes, cancellation and prediction; keep a bespoke Component for a small bounded rule set.
- **Strong 3-Year-Engineer Answer:** I start from interaction density and networking, not genre. GAS earns its concepts when many abilities share lifecycle policy, modifiers stack from many sources, designers need data-driven effects and supported prediction matters. I price in plugin/version, authoring and debugging complexity. I prototype sprint, damage, stun and respawn in the smallest representative slice and compare duplicated policy and failure handling against a focused Component.
- **Common Weak Answer:** “GAS is for complex multiplayer RPG abilities; Components are for simple games.”
- **Follow-up Questions:** What is the adoption threshold? Can GAS coexist with ordinary Components? What state should stay outside GAS?
- **Hands-on Verification Task:** Implement sprint and stun both ways, then compare state interactions, multiplayer correction and authoring cost.
- **Sources:** [SRC-GAS-001]
- **Version Notes:** GAS is plugin-dependent; exact prediction capability is target-sensitive.

### Question: Explain ASC owner versus avatar and where you would put the ASC.

- **Category:** GAS / Lifetime
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Owner logically owns durable ASC state; avatar is the current physical actor. Pawn placement is simple, while PlayerState placement can preserve state across respawn.
- **Strong 3-Year-Engineer Answer:** Placement follows lifetime and ownership. For state that dies with a Pawn, Pawn-owned ASC is direct. For player attributes/effects/cooldowns that survive possession or respawn, PlayerState can own the ASC and the Pawn becomes avatar. I initialise actor info on server and owning client when their relationships are valid, refresh it on respawn, remove old-avatar bindings and test dedicated-server timing.
- **Common Weak Answer:** “Put the ASC on PlayerState for multiplayer and Character for single-player.”
- **Follow-up Questions:** What breaks on respawn? What about AI? Why can client activation fail while server works?
- **Hands-on Verification Task:** Respawn a PlayerState-owned ASC under latency and prove cooldown, attributes and input survive without duplicate delegates.
- **Sources:** [SRC-GAS-002]
- **Version Notes:** Exact initialisation hooks depend on framework and UE branch.

### Question: Gameplay Ability class, spec, instance and handle—what differs?

- **Category:** GAS / Abilities
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Class defines behaviour; a spec is one runtime grant in an ASC; an instance holds activation/per-owner state according to policy; a handle identifies the grant.
- **Strong 3-Year-Engineer Answer:** Authority grants an ability class as an `FGameplayAbilitySpec`, which can include level/input/source data and returns a spec handle. The instancing policy determines whether execution uses no mutable instance, one per owner/spec or one per activation. I remove by the intended handle/source rather than assuming one class means one grant, and make startup/equipment grants idempotent.
- **Common Weak Answer:** “The ability spec is an instance of the ability.”
- **Follow-up Questions:** Can one class be granted twice? Who may call GiveAbility? Which policy supports concurrent state?
- **Hands-on Verification Task:** Grant the same ability from two items and remove only one source safely.
- **Sources:** [SRC-GAS-003], [SRC-GAS-010]
- **Version Notes:** Instancing/network-policy combinations require target verification.

### Question: Walk through ability activation, commit, cancellation and ending.

- **Category:** GAS / Abilities
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Request, eligibility, activation, deliberate commit, asynchronous execution, cancel/complete and one explicit end/cleanup path.
- **Strong 3-Year-Engineer Answer:** I capture failure tags from actor-info, network, tag, cost, cooldown and custom checks. I commit cost/cooldown at the first intentional irreversible point, not automatically on input. Tasks/montages/targeting then run. Every success, rejection, interruption, death and cancellation converges on idempotent cleanup and `EndAbility`; otherwise the spec can remain active and block other abilities.
- **Common Weak Answer:** “Activate checks cost and cooldown, runs the montage, then ends.”
- **Follow-up Questions:** When commit a charged shot? Is cancellation failure? What if montage never sends its event?
- **Hands-on Verification Task:** Inject interruption before/after commit and assert one cost, one cooldown and one end callback.
- **Sources:** [SRC-GAS-003]
- **Version Notes:** Stable model; exact callbacks/task APIs are version-sensitive.

### Question: How do GAS Gameplay Tags differ from Boolean state?

- **Category:** GAS / Tags
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Tags are registered hierarchical predicates with source counts and query semantics; a Boolean is one value owned by one state path.
- **Strong 3-Year-Engineer Answer:** GAS tags form a shared vocabulary for activation, blocking, cancellation, cooldown and effect state. Multiple effects may grant the same tag, so removing one source must not clear another. Parent matching is deliberate. I use effect/ability-granted tags for scoped state, loose tags only with one symmetric owner, and ordinary data for magnitude, identity or timestamps.
- **Common Weak Answer:** “Tags are named Booleans that avoid hard references.”
- **Follow-up Questions:** Owned versus asset tag? Parent matching? What causes a leaked loose tag?
- **Hands-on Verification Task:** Apply two stun sources, remove one and prove `State.CrowdControl.Stunned` remains until both end.
- **Sources:** [SRC-GAS-006]
- **Version Notes:** Tag-query edge cases should be unit-tested in the target branch.

### Question: Explain attribute base/current values and invariant handling.

- **Category:** GAS / Attributes
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Base is the underlying value; current is the evaluated value after active modifiers. Attribute Sets declare values, but invariants must cover all mutation/replication paths.
- **Strong 3-Year-Engineer Answer:** I use attributes when effects, capture, modifiers, prediction or ASC delegates need them. For Health/MaxHealth I define clamping and what a maximum change does to current health, test instant, duration, execution, base-change and replication paths, and drive UI from snapshot plus change delegates. I distinguish authoritative value, replicated value and display projection before blaming the effect.
- **Common Weak Answer:** “Base is the maximum and current is the remaining amount.”
- **Follow-up Questions:** Attribute versus inventory count? Where clamp? Why can UI differ from server?
- **Hands-on Verification Task:** Apply/remove max-health modifiers at low/full health and verify invariants on server and client.
- **Sources:** [SRC-GAS-004]
- **Version Notes:** Rep-notify helpers and mutation hooks require UE5.3–UE5.6 source verification.

### Question: Gameplay Effect definition, spec and active effect—what differs?

- **Category:** GAS / Effects
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Definition is data; spec is mutable application data with level/context/captures; applying a non-instant spec creates retained active-effect state.
- **Strong 3-Year-Engineer Answer:** A `UGameplayEffect` defines duration policy, modifiers, executions, tags and stacking. An `FGameplayEffectSpec` binds application-specific context, level, captured source/target information, set-by-caller values and calculated details. Instant effects execute without entering the active container; duration/infinite effects do, and periodic effects are retained and execute on intervals. I log the spec/application handle when debugging.
- **Common Weak Answer:** “A Gameplay Effect asset is instantiated on the target.”
- **Follow-up Questions:** Snapshot versus live capture? Set-by-caller? Periodic versus Tick?
- **Hands-on Verification Task:** Inspect instant, duration and periodic applications and compare active-container/attribute behaviour.
- **Sources:** [SRC-GAS-005], [SRC-GAS-009]
- **Version Notes:** Gameplay Effect authoring components changed across UE5.

### Question: Design a stacking buff completely.

- **Category:** GAS / Effects
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Define aggregation owner, limit, overflow, duration refresh, expiration, period, tags/cues and UI—not only the stack limit.
- **Strong 3-Year-Engineer Answer:** I first decide aggregate-by-source or target. Then stack limit/overflow, whether reapplication refreshes/resets/extends duration, whether expiration removes one/all, period reset, magnitude per stack, granted tag/cue lifetime and how UI attributes sources. I test two instigators, max-stack overflow, refresh near expiry, dispel and replication to owner/non-owner.
- **Common Weak Answer:** “Set stack limit five and refresh duration on application.”
- **Follow-up Questions:** What if sources differ? How does immunity interact? What does UI replicate?
- **Hands-on Verification Task:** Build two-source poison and execute an acceptance matrix for all policies.
- **Sources:** [SRC-GAS-005]
- **Version Notes:** Exact stacking options/UI replication are branch/project-sensitive.

### Question: Ability Task versus Gameplay Cue versus Gameplay Effect?

- **Category:** GAS / Integration
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Task coordinates asynchronous ability work, cue presents cosmetics, and effect changes/queryable gameplay state.
- **Strong 3-Year-Engineer Answer:** I use a task for ability-scoped waiting such as target data, input, montage or movement and ensure it terminates with the ability. I use cues for visual/audio feedback that may arrive late without changing truth. I use effects for costs, attributes, duration/periodic state and tags. An ability orchestrates them; none should secretly duplicate another's authority.
- **Common Weak Answer:** “Tasks are latent nodes, cues are VFX and effects are buffs.”
- **Follow-up Questions:** Custom task lifetime? Persistent cue removal? Where does damage live?
- **Hands-on Verification Task:** Cancel an ability at every task phase and prove no gameplay/cue/task leak.
- **Sources:** [SRC-GAS-001], [SRC-GAS-005], [SRC-GAS-007], [SRC-GAS-008]
- **Version Notes:** Ability Task source is UE4.27-era; verify current task API.

### Question: What does Local Predicted mean in GAS?

- **Category:** GAS / Networking
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** The owning client performs supported provisional work under a prediction key while the server validates and confirms/rejects authoritative execution.
- **Strong 3-Year-Engineer Answer:** The client requests activation and can immediately run supported predicted effects/cues/movement, correlated by a prediction key/window. The server checks ownership, actor info, tags, cost, cooldown, cadence and target evidence. Confirmation converges state; rejection rolls back supported provisional changes. I keep irreversible custom side effects and final damage server-side and test dedicated-server lag/loss, rejection, cancellation and duplicates.
- **Common Weak Answer:** “It runs on client first and replicates to server to remove input lag.”
- **Follow-up Questions:** Prediction key? What cannot be rolled back? Why can a cue double-play?
- **Hands-on Verification Task:** Reject every third dash under latency and trace client/server convergence by prediction key.
- **Sources:** [SRC-GAS-001], [SRC-GAS-011], [SRC-GAS-013]
- **Version Notes:** Prediction support is operation- and branch-specific.

### Question: Full, Mixed and Minimal ASC effect replication modes?

- **Category:** GAS / Networking
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Full sends full GE information to all; Mixed sends full to owner/autonomous and minimal to simulated proxies; Minimal sends minimal GE information.
- **Strong 3-Year-Engineer Answer:** These modes govern active Gameplay Effect detail, not every ASC field. I commonly consider Mixed for player-owned ASCs that need owner detail and Minimal for server-controlled populations, but only after defining which remote UI/cues/tags require data and confirming owner setup. Attribute replication and other state have their own paths. I validate bandwidth and observable behaviour for every audience.
- **Common Weak Answer:** “Minimal only replicates tags, Mixed replicates effects to owner, Full is for single-player.”
- **Follow-up Questions:** Does Minimal stop attribute replication? Why does Mixed need correct owner? Which audience needs duration?
- **Hands-on Verification Task:** Compare owner/simulated-proxy active-effect visibility and bandwidth in all three modes.
- **Sources:** [SRC-GAS-012]
- **Version Notes:** Maintained API may display UE5.7; verify target semantics.

### Question: An ability will not activate on the owning client—how do you debug it?

- **Category:** GAS / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Trace ASC/actor info, grant/spec, ownership/net policy, failure tags, owned tags, cost/cooldown, triggers and blockers on the failing machine.
- **Strong 3-Year-Engineer Answer:** I first identify server, owning client or proxy and inspect the exact ASC owner/avatar and actor info. Then I prove the spec is granted, capture activation failure tags, inspect required/blocked tag counts and sources, cost/cooldown and custom checks. I verify input/event payload and net execution policy, then compare dedicated server with listen server. Logs include spec handle, role, prediction key and rejection/end reason.
- **Common Weak Answer:** “Check CanActivateAbility and whether the client owns the Character.”
- **Follow-up Questions:** Works until respawn? No failure tag? Server activates but client does not? Effect remains after cancel?
- **Hands-on Verification Task:** Diagnose five injected failures without adding arbitrary delays or bypassing checks.
- **Sources:** [SRC-GAS-002], [SRC-GAS-003], [SRC-GAS-006]
- **Version Notes:** Debug commands/UI vary; the evidence sequence is stable.

## Algorithms and Data Structures

### Question: How do you choose a data structure in an interview?

- **Category:** Algorithms / Problem Solving
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Derive it from dominant operations, bounds, ordering, stability, memory/layout and determinism after stating a correct baseline.
- **Strong 3-Year-Engineer Answer:** I ask maximum count, update/query ratio, lookup key, ordering/duplicates, mutation, stable-reference and deterministic-iteration requirements. I give the simplest correct baseline and cost, then identify repeated work that justifies an index or different representation. I usually prefer dense arrays for hot iteration and may add a hash ID→index view. I state expected/worst/amortised costs, invalidation and memory, then test adversarial distributions.
- **Common Weak Answer:** “Use a hash map for fast lookup, vector for iteration and tree when order matters.”
- **Follow-up Questions:** When can two structures coexist? Does Big-O decide? What if references must be stable?
- **Hands-on Verification Task:** Defend containers for inventory, 10,000 projectiles and a timer scheduler under changing constraints.
- **Sources:** [SRC-ALG-001], [SRC-ALG-002], [SRC-ALG-003]
- **Version Notes:** Standard container facts must be translated deliberately to UE containers.

### Question: Explain worst-case, average/expected and amortised complexity.

- **Category:** Algorithms / Complexity
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Worst bounds any input; expected averages under stated assumptions/distribution; amortised spreads occasional expensive operations over a sequence.
- **Strong 3-Year-Engineer Answer:** Hash lookup is expected constant with suitable hash/load but can be linear under collisions. Dynamic-array append is amortised constant because occasional growth moves all elements, though that single growth can still hitch a frame. I define `n`, include auxiliary space and split build/update/query. In production I then measure realistic and adversarial P95/P99 because asymptotics do not include constants/cache/allocation.
- **Common Weak Answer:** “Worst is slowest, average is normal and amortised means mostly O(1).”
- **Follow-up Questions:** Why can amortised O(1) hitch? Output-sensitive cost? What assumptions support hash O(1)?
- **Hands-on Verification Task:** Plot per-operation and cumulative cost of an expanding array and rehashing table.
- **Sources:** [SRC-ALG-001], [SRC-ALG-002], [SRC-ALG-003]
- **Version Notes:** Stable computer-science concept.

### Question: Dynamic array versus linked list?

- **Category:** Algorithms / Containers
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Prefer contiguous arrays for locality/index/iteration; a list wins only when stable nodes and frequent known-position splicing outweigh allocation/pointer costs.
- **Strong 3-Year-Engineer Answer:** A vector/TArray gives constant index, amortised append and dense cache-friendly iteration, but shifts on middle erase and invalidates references on relocation. A list has constant splice/erase only after the node is known; search remains linear and every node adds pointer/allocation overhead. For unordered removal I consider swap-remove; for stable identity I use handles/indirection rather than defaulting to a list.
- **Common Weak Answer:** “Vector has fast reads; list has fast inserts/deletes.”
- **Follow-up Questions:** Stable pointers? Small N? What does swap-remove break?
- **Hands-on Verification Task:** Benchmark scan and known/unknown-position erase for representative element sizes/counts.
- **Sources:** [SRC-ALG-002]
- **Version Notes:** UE `TArray` details differ from `std::vector`; conceptual trade-off remains.

### Question: What can go wrong with a hash map?

- **Category:** Algorithms / Hashing
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Poor hash/equality, collisions/load, rehash invalidation/spikes, mutable keys, memory slack and non-deterministic iteration.
- **Strong 3-Year-Engineer Answer:** Equal keys must hash equally and stored key identity must not mutate. Expected constant operations assume suitable distribution/load; worst case is linear. Rehash can allocate, move bucket state and invalidate according to the container. I reserve credible capacity, never use iteration order as gameplay protocol, and consider direct indexing/bitset for dense bounded IDs or sorted arrays for small static data.
- **Common Weak Answer:** “Hash collisions make lookup slower, so use a good hash.”
- **Follow-up Questions:** Hash combine? Deterministic replay? Why can a map be slower than array scan?
- **Hands-on Verification Task:** Inject a constant hash, measure bucket distribution and prove lookup correctness remains while cost degrades.
- **Sources:** [SRC-ALG-003]
- **Version Notes:** Exact invalidation/order differs by standard and UE container implementation.

### Question: How does a heap-backed priority queue work?

- **Category:** Algorithms / Heaps
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** A heap keeps parent priority over children, giving constant top and logarithmic push/pop without globally sorting all elements.
- **Strong 3-Year-Engineer Answer:** A binary heap is a compact tree in an array. Insertion bubbles up; top removal swaps/replaces root and bubbles down, both logarithmic. It supports A*/Dijkstra frontiers, timers and top-K. I check comparator direction, deterministic tie-breaks and whether decrease-key/arbitrary removal is required; otherwise I may push improved duplicates and skip stale entries when popped.
- **Common Weak Answer:** “A priority queue is a sorted queue where highest priority comes first.”
- **Follow-up Questions:** Why not keep a sorted vector? Min-heap with std::priority_queue? Stale entries?
- **Hands-on Verification Task:** Implement a min-heap with invariant checks and compare top-K against full sort.
- **Sources:** [SRC-ALG-004]
- **Version Notes:** Standard adaptor surface differs from UE heap helpers.

### Question: Binary search—what are its preconditions and common bugs?

- **Category:** Algorithms / Searching
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Search a sorted/partitioned range with a consistent comparator; use half-open bounds and test duplicates, empty and insertion endpoints.
- **Strong 3-Year-Engineer Answer:** I define whether I need any equal, first equal, first not-less (`lower_bound`) or insertion point. I maintain `[lo,hi)`, compute `lo+(hi-lo)/2`, and prove the answer remains in the interval each step. Logarithmic comparisons do not imply logarithmic iterator movement on a linked/forward range, and sorting cost must be justified by repeated queries.
- **Common Weak Answer:** “Repeatedly compare middle and halve until found; O(log n).”
- **Follow-up Questions:** Why half-open? Duplicates? One query on unsorted input? Ordered-map member lower_bound?
- **Hands-on Verification Task:** Differential-test all insertion points against `std::lower_bound` for generated sorted arrays.
- **Sources:** [SRC-ALG-006]
- **Version Notes:** Stable algorithm; standard overloads evolve.

### Question: BFS versus DFS?

- **Category:** Algorithms / Graphs
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** BFS uses FIFO and finds minimum edge-count paths in unweighted graphs; DFS uses a stack and suits reachability, components, cycles and backtracking.
- **Strong 3-Year-Engineer Answer:** Both are `O(V+E)` with adjacency lists and visited state. BFS explores by distance layer, so it finds shortest paths only when edges have equal cost. DFS follows depth and finds a path, not generally the shortest. I mark visited at enqueue/push discovery to prevent duplicates, store parent only when needed and restart from unvisited vertices to cover disconnected graphs.
- **Common Weak Answer:** “BFS is level order and DFS goes deep; use BFS for shortest path.”
- **Follow-up Questions:** Weighted graph? Recursive overflow? Grid flood fill? Disconnected components?
- **Hands-on Verification Task:** Implement both iteratively and visualise order/parent paths on cyclic disconnected graphs.
- **Sources:** [SRC-ALG-007]
- **Version Notes:** Stable algorithm concepts.

### Question: Explain Dijkstra and A* and their correctness constraints.

- **Category:** Algorithms / Pathfinding
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Dijkstra expands lowest known non-negative path cost; A* adds a remaining-cost heuristic and preserves optimality under suitable heuristic conditions.
- **Strong 3-Year-Engineer Answer:** Both relax edges and track best `g` plus parent. Dijkstra orders by `g`; negative edges break its settled-cost reasoning. A* orders by `f=g+h`; `h=0` is Dijkstra. An admissible/consistent heuristic matched to movement can reduce expansions while preserving optimality. Overweighting can deliberately trade path quality for speed. I separate global pathfinding from following/avoidance and measure expansions/frontier/time.
- **Common Weak Answer:** “A* is Dijkstra plus distance to goal, so it is faster.”
- **Follow-up Questions:** Manhattan versus Euclidean? Inconsistent heuristic? Duplicate frontier entries? Dynamic obstacles?
- **Hands-on Verification Task:** Compare BFS/Dijkstra/A* costs and expansions on weighted maps, then inject an invalid heuristic.
- **Sources:** [SRC-ALG-007], [SRC-ALG-008], [SRC-ALG-009]
- **Version Notes:** Stable algorithms; engine navigation implementation is separate.

### Question: How do you topologically sort dependencies and detect cycles?

- **Category:** Algorithms / Graphs
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** For a DAG, repeatedly emit zero-indegree nodes and remove their outgoing edges; fewer than V outputs proves a cycle.
- **Strong 3-Year-Engineer Answer:** Kahn's algorithm computes indegrees, queues all zero-indegree nodes, emits one and decrements neighbours, enqueueing newly zero nodes. It is `O(V+E)`. Multiple valid orders exist, so deterministic build/system scheduling needs a stable tie-break. If output is short, I report a useful cycle path/subgraph rather than only failing. A directed cycle means no topological order.
- **Common Weak Answer:** “DFS and reverse postorder; if there is a cycle it fails.”
- **Follow-up Questions:** Determinism? Undirected graph? Build dependency cycle diagnostics?
- **Hands-on Verification Task:** Sort a module graph with stable lexical tie-break and report one exact cycle.
- **Sources:** [SRC-ALG-010]
- **Version Notes:** Stable algorithm concept.

### Question: What problem does union–find solve?

- **Category:** Algorithms / Connectivity
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It maintains connected-component partitions under unions, with near-constant amortised Find/Union using path compression and union by size/rank.
- **Strong 3-Year-Engineer Answer:** Each item points toward a representative root. Find compresses paths; Union attaches the smaller/ranker tree to the other root. It is ideal for incremental/offline connectivity, Kruskal and procedural region linking. It does not recover shortest paths and ordinary DSU does not support arbitrary edge deletion well, so I would use graph traversal or a dynamic-connectivity design for those requirements.
- **Common Weak Answer:** “It groups nodes and can check whether two nodes connect in O(1).”
- **Follow-up Questions:** Why both optimisations? Component size? Edge deletion? MST relation?
- **Hands-on Verification Task:** Compare random DSU connectivity queries against BFS on a small graph.
- **Sources:** [SRC-ALG-011]
- **Version Notes:** Stable algorithm concept.

### Question: Design weighted random selection.

- **Category:** Algorithms / Sampling
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Sample uniformly over total weight and select the containing prefix interval; optimise with prefix search only when repeated draws justify preprocessing.
- **Strong 3-Year-Engineer Answer:** I reject negative/NaN weights and define zero-total policy. A linear prefix scan is `O(n)` and simplest for small/dynamic sets. Static repeated draws can precompute prefix sums and lower-bound in `O(log n)`; alias tables can go further. I use sufficient precision, specify `[0,total)` boundary and deterministic RNG stream. A shuffle bag is an anti-streak product design, not independent weighted sampling.
- **Common Weak Answer:** “Sum weights, roll random and subtract until zero.”
- **Follow-up Questions:** Dynamic updates? Floating precision? Fairness test? Deterministic replay?
- **Hands-on Verification Task:** Distribution-test linear/prefix variants with zero, skewed and huge weights.
- **Sources:** [SRC-ALG-006]
- **Version Notes:** RNG APIs/determinism are engine/platform-sensitive.

### Question: How would you accelerate neighbour queries for 10,000 moving agents?

- **Category:** Algorithms / Spatial Structures
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Keep all-pairs as oracle, then try a tuned uniform grid/spatial hash, exact-filter candidates and measure update/query plus clustered worst cases.
- **Strong 3-Year-Engineer Answer:** For similarly sized moving 2D agents, I map positions/AABBs to integer cells using floor, including negatives. Objects enter all overlapped cells or follow a documented centre policy; queries gather relevant cells, deduplicate and exact distance-filter. Cell size trades multi-cell coverage against bucket occupancy. I measure candidate/hit ratio, max bucket, updates and P95 under formations, not only uniform random placement.
- **Common Weak Answer:** “Use a quadtree or spatial hash to make queries O(n log n).”
- **Follow-up Questions:** Large objects? Threading? All in one cell? Cell update frequency?
- **Hands-on Verification Task:** Differential-test hash results against all-pairs while agents cross negative cell boundaries.
- **Sources:** [SRC-ALG-012], [SRC-ALG-014]
- **Version Notes:** Workload-specific; no universal complexity guarantee.

### Question: Grid/spatial hash versus quadtree, k-d tree or BVH?

- **Category:** Algorithms / Spatial Structures
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Select from dimension, extent, size distribution, motion and query/update types; no structure dominates every workload.
- **Strong 3-Year-Engineer Answer:** Uniform grids suit moving similarly sized objects with tunable density. Quad/octrees adapt bounded space but straddlers and churn matter. k-d trees fit relatively static point nearest-neighbour. BVHs fit bounded geometry/ray/collision traversal with refit/rebuild choices; R-trees index boxes/ranges. For small N I keep a flat scan. I benchmark actual clustered motion, build/update/query and memory.
- **Common Weak Answer:** “Use quadtree for 2D, octree for 3D and BVH for rendering.”
- **Follow-up Questions:** Loose octree? Refit versus rebuild? Candidate false positives? Mixed sizes?
- **Hands-on Verification Task:** Compare scan, grid and tree/library index for two controlled spatial distributions.
- **Sources:** [SRC-ALG-012], [SRC-ALG-013], [SRC-ALG-014]
- **Version Notes:** Engine subsystem implementations/version APIs are separate from these concepts.

### Question: Broadphase versus narrowphase?

- **Category:** Algorithms / Collision
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Broadphase cheaply returns conservative candidate pairs; narrowphase performs accurate shape tests on candidates.
- **Strong 3-Year-Engineer Answer:** Broadphase uses AABBs, sweep-and-prune or spatial structures to reject obvious non-overlap. It may return false positives but must avoid false negatives. Narrowphase uses actual sphere/box/convex/mesh tests and contact generation. I track update/build cost, candidate count, duplicate pairs and candidate-to-true-hit ratio; the same candidate/filter architecture appears in interest management and AI queries.
- **Common Weak Answer:** “Broadphase checks bounding boxes, narrowphase checks mesh collision.”
- **Follow-up Questions:** Why false positives okay? Moving objects? Continuous collision? Network interest relation?
- **Hands-on Verification Task:** Build sphere broadphase AABB candidates and compare exact sphere intersections against all-pairs.
- **Sources:** [SRC-ALG-012]
- **Version Notes:** Physics-engine internals are implementation/version-specific.

### Question: How do you debug and prove an optimised algorithm correct?

- **Category:** Algorithms / Testing
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** State invariant, minimise counterexamples and differential/property-test the optimised result against a simple oracle on generated adversarial inputs.
- **Strong 3-Year-Engineer Answer:** I retain a slow reference—Dijkstra for A*, all-pairs for spatial query, BFS for DSU connectivity—and generate small deterministic cases. I compare exact result/cost properties while checking invariants after each mutation. Failures shrink to minimal seed/input. Performance runs separately on realistic maximum and skewed cases, recording build/update/query, allocations and P95/P99, so correctness tests do not masquerade as benchmarks.
- **Common Weak Answer:** “Write unit tests for edge cases and profile the optimised version.”
- **Follow-up Questions:** Property examples? Nondeterministic failures? Float-tolerant comparison? Benchmark fairness?
- **Hands-on Verification Task:** Create one differential fuzzer that catches an injected spatial-hash negative-coordinate bug.
- **Sources:** [SRC-ALG-007], [SRC-ALG-012]
- **Version Notes:** Test framework choice is project-specific.

## Game Maths

### Question: What does the dot product tell you in gameplay code?

- **Category:** Game Maths / Vectors
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** It measures magnitude-scaled alignment; for unit vectors it is cosine of their angle.
- **Strong 3-Year-Engineer Answer:** I use dot for front/behind, FOV cones, projection and velocity-normal components. For a cone I normalise forward/to-target and compare their dot against cosine of the half-angle, avoiding `acos`. I clamp before inverse trig and handle zero direction. A projection onto arbitrary `b` is `b*(a·b)/(b·b)`; if `b` is unit, the denominator is one.
- **Common Weak Answer:** “Dot returns the angle between two vectors.”
- **Follow-up Questions:** Why normalise? Why compare cosine? Projection/rejection? Surface slide?
- **Hands-on Verification Task:** Draw an FOV cone and projection/rejection while varying vector lengths and zero input.
- **Sources:** [SRC-MATH-003]
- **Version Notes:** Stable maths; exact UE helper names/types vary.

### Question: What does the cross product tell you, and what can go wrong?

- **Category:** Game Maths / Vectors
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** It returns a perpendicular with sine/area-scaled magnitude; operand order and Unreal's handedness determine sign.
- **Strong 3-Year-Engineer Answer:** I use cross to build a basis, compute triangle normals and establish left/right around a chosen up axis. Swapping operands negates the result, and parallel inputs produce near zero, so safe normalisation needs a fallback. In Unreal I verify a known +X/+Y case against its left-handed Z-up convention instead of importing a right-hand mnemonic.
- **Common Weak Answer:** “Cross gives the normal/right vector.”
- **Follow-up Questions:** Signed angle? Triangle winding? What if forward parallels up?
- **Hands-on Verification Task:** Draw both `A×B` and `B×A` for Unreal basis axes and a near-parallel case.
- **Sources:** [SRC-MATH-001], [SRC-MATH-003]
- **Version Notes:** Unreal axis convention is essential.

### Question: Point versus vector transform, and local versus world space?

- **Category:** Game Maths / Transforms
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** A point includes translation; a vector/direction does not. Local coordinates are relative to a parent/object basis; world coordinates use the level basis/origin.
- **Strong 3-Year-Engineer Answer:** I name spaces in variables and use `TransformPosition` for locations, `TransformVector` when scale/rotation apply, and no-scale direction transforms where appropriate. Inverse operations move world back to local. I test identity, translation, rotation and non-uniform scale separately, draw axes, and round-trip local→world→local. Treating aim direction as a point adds actor location.
- **Common Weak Answer:** “Local is relative to the Actor and world is absolute; use the transform matrix.”
- **Follow-up Questions:** Component versus actor local? Normal transform? Parent scale? Why inverse transpose?
- **Hands-on Verification Task:** Transform a muzzle point and aim direction through nested non-uniform transforms and diagnose intentional misuse.
- **Sources:** [SRC-MATH-001], [SRC-MATH-004], [SRC-MATH-005]
- **Version Notes:** Exact composition behaviour should be tested in target UE branch.

### Question: Why does transform order matter?

- **Category:** Game Maths / Matrices
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Transform composition is non-commutative; rotate-then-translate generally differs from translate-then-rotate.
- **Strong 3-Year-Engineer Answer:** I refuse to quote a bare multiplication order without naming row/column and application convention. I derive it by applying transforms to a known point through the engine API. Translation affects points not directions; non-uniform scale and rotation can imply shear that a simple SRT representation may not preserve exactly. Inverse undoes composition in reverse order, and zero scale is singular.
- **Common Weak Answer:** “Matrices multiply right-to-left, so scale then rotate then translate.”
- **Follow-up Questions:** Parent-child composition? Non-uniform scale? Normal transform? Inverse order?
- **Hands-on Verification Task:** Compare two transform orders on one offset point and visualise the different orbit/pivot behaviour.
- **Sources:** [SRC-MATH-004], [SRC-MATH-005]
- **Version Notes:** Engine convention/API is authoritative.

### Question: Rotator versus quaternion, and what is gimbal lock?

- **Category:** Game Maths / Rotation
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Rotators are readable ordered Euler angles with singularities; unit quaternions compose/interpolate arbitrary orientation without that Euler parameterisation singularity.
- **Strong 3-Year-Engineer Answer:** Gimbal lock is two Euler rotation axes aligning at a singular orientation, not the object losing physical ability to rotate. I use Rotators/degrees for editor and constrained yaw/pitch, quaternions for composition and arbitrary 3D interpolation. Quaternion multiplication order matters, q and -q represent the same orientation, and normalisation/sign alignment matter for interpolation.
- **Common Weak Answer:** “Quaternions avoid gimbal lock and are four-dimensional rotations.”
- **Follow-up Questions:** q versus -q? Quaternion inverse? Why shortest-path Slerp? Can converting to Euler reintroduce issues?
- **Hands-on Verification Task:** Compare Euler component interpolation and quaternion Slerp near wrap and a pitch singularity.
- **Sources:** [SRC-MATH-006], [SRC-MATH-007]
- **Version Notes:** `TQuat::Slerp` source is pinned to UE5.5; verify target API.

### Question: Lerp, Slerp, constant-speed movement and smoothing—how do they differ?

- **Category:** Game Maths / Interpolation
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Lerp parametrises a straight segment; Slerp follows an orientation sphere arc; repeated current-target lerp eases, while move-towards enforces speed.
- **Strong 3-Year-Engineer Answer:** `lerp(a,b,t)=a+t(b-a)` is constant speed only when `t` advances linearly between fixed endpoints. `lerp(current,target,speed*dt)` each frame is exponential-like and frame-rate-sensitive unless alpha is derived, e.g. `1-exp(-λdt)`. For constant translational speed I clamp the step to `speed*dt`. For orientation I use shortest-aligned quaternion Slerp where constant angular interpolation matters.
- **Common Weak Answer:** “Lerp is linear movement and Slerp is smooth rotation.”
- **Follow-up Questions:** Overshoot? Exponential half-life? Euler wrap? Spring damping?
- **Hands-on Verification Task:** Plot positions at 30/60/120 Hz for fixed-alpha, exponential-alpha and move-towards variants.
- **Sources:** [SRC-MATH-007], [SRC-MATH-008]
- **Version Notes:** Interp helper semantics must be read exactly; similarly named helpers differ.

### Question: Derive closest point on a segment.

- **Category:** Game Maths / Geometry
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Project `P-A` onto `B-A`, divide by segment length squared, clamp parameter to `[0,1]`, then evaluate `A+t(B-A)`.
- **Strong 3-Year-Engineer Answer:** `t=dot(P-A,d)/dot(d,d)`. Clamping distinguishes segment from infinite line. If `d·d` is near zero, return A because the segment degenerates to a point. The result supports capsule distance, path following and aim assist. I compare squared distance afterward when no actual length is needed.
- **Common Weak Answer:** “Project the point onto the line and clamp it to the segment.”
- **Follow-up Questions:** Degenerate segment? Ray variant? Capsule use? Why squared distance?
- **Hands-on Verification Task:** Visualise t and closest point for before/inside/after/zero-length cases.
- **Sources:** [SRC-MATH-008]
- **Version Notes:** Stable geometry; verify helper argument order.

### Question: How do ray–plane and ray–sphere tests work?

- **Category:** Game Maths / Intersections
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Substitute the parametric ray into the primitive equation, solve for t, then filter by ray/segment interval and degeneracies.
- **Strong 3-Year-Engineer Answer:** For plane, `t=n·(p0-o)/(n·d)`; near-zero denominator is parallel and coplanarity is separate. For sphere, substitute `o+td` into squared radius, solve the quadratic and choose nearest acceptable non-negative root. I handle origin-inside, tangent, both-roots-behind, non-unit direction and zero direction. A line root is not automatically a ray hit.
- **Common Weak Answer:** “Ray-plane uses dot product and ray-sphere uses the quadratic formula.”
- **Follow-up Questions:** Segment range? One-sided plane? Surface normal? Numerical stability?
- **Hands-on Verification Task:** Draw roots/t for parallel, tangent, inside and behind cases and compare against engine traces where semantics match.
- **Sources:** [SRC-MATH-012]
- **Version Notes:** Engine collision filters/shapes add semantics beyond primitive maths.

### Question: Explain ray–AABB using slabs.

- **Category:** Game Maths / Intersections
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Compute each axis's ray interval inside its min/max slab; intersect intervals and accept when enter≤exit and exit is not behind.
- **Strong 3-Year-Engineer Answer:** Per axis I derive two t values, swap for direction sign, update global `tEnter=max(enters)` and `tExit=min(exits)`. Near-zero direction means origin must already lie within that slab; blindly dividing can create infinities/NaNs. I clamp to ray/segment range and track the entering axis for normal. For OBB I transform the ray into local box space carefully.
- **Common Weak Answer:** “Check intersection with six planes and take the closest.”
- **Follow-up Questions:** Origin inside? Zero direction? Segment? Non-uniform OBB transform?
- **Hands-on Verification Task:** Differential-test slab code across zero/negative directions and inside starts.
- **Sources:** [SRC-MATH-013]
- **Version Notes:** Stable maths; adapt coordinate and parameter conventions.

### Question: Walk from world position to screen pixel.

- **Category:** Game Maths / Rendering Spaces
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** World→view→clip, divide homogeneous coordinates by w to NDC, then map NDC to viewport pixels/depth convention.
- **Strong 3-Year-Engineer Answer:** View expresses position relative to camera; projection maps the view frustum to homogeneous clip coordinates. Perspective divide creates NDC and viewport transform maps to pixels. I check behind/near-camera and clipping before using UI coordinates. Depth is projected/non-linear and RHI convention-specific; I use UE helpers rather than assuming OpenGL/DirectX ranges. Translated world spaces help precision.
- **Common Weak Answer:** “Multiply by view-projection matrix and divide x/y by z.”
- **Follow-up Questions:** Why w? Orthographic difference? Depth reconstruction? Behind camera?
- **Hands-on Verification Task:** Display every space value for moving points crossing each frustum boundary.
- **Sources:** [SRC-MATH-001], [SRC-MATH-009]
- **Version Notes:** UE4.27 rendering-space terminology is conceptual; current renderer/RHI is authoritative.

### Question: Why do normals and colour textures need special treatment?

- **Category:** Game Maths / Rendering
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Normals are directions requiring special non-uniform-scale transform; lighting colours need linear arithmetic while normal/data textures must not undergo colour gamma decoding.
- **Strong 3-Year-Engineer Answer:** A normal must remain perpendicular, so under non-uniform scale it uses inverse-transpose-style treatment then normalisation, unlike an ordinary vector. Tangent-space normal maps encode a local direction through TBN and are data, not display colour. Base colour commonly arrives sRGB-decoded to linear for lighting; masks/roughness/normals should use their data settings. I verify tangent handedness/mirrored UVs and import flags.
- **Common Weak Answer:** “Normal maps are blue and should have sRGB disabled.”
- **Follow-up Questions:** TBN? Mirrored UVs? World-space normals? Why linear lighting?
- **Hands-on Verification Task:** Deliberately apply sRGB to a normal map and ordinary vector transform under non-uniform scale; visualise both failures.
- **Sources:** [SRC-MATH-005], [SRC-MATH-010], [SRC-MATH-011]
- **Version Notes:** Material/texture pipeline settings are renderer/content-version-sensitive.

### Question: How do you make vector/intersection maths numerically robust?

- **Category:** Game Maths / Numerical Robustness
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** State units/scale, use domain tolerances, guard degenerates/division, clamp inverse-trig inputs, reject non-finite values and test boundary cases.
- **Strong 3-Year-Engineer Answer:** I do not use exact equality for computed geometry or one magic epsilon for every domain. I avoid square roots for range comparisons, safe-normalise with defined zero fallback, clamp dot before acos, branch for parallel/zero segment/direction and filter roots by the correct interval. I test huge/small coordinates, tangent/parallel/inside cases and NaN. With LWC I still avoid accidental float narrowing and prefer local spaces.
- **Common Weak Answer:** “Use KINDA_SMALL_NUMBER and IsNearlyEqual for floats.”
- **Follow-up Questions:** Absolute versus relative tolerance? NaN comparison? Cancellation? LWC limitations?
- **Hands-on Verification Task:** Generate boundary/scale cases and compare custom primitives against double/reference/engine results.
- **Sources:** [SRC-MATH-002], [SRC-MATH-008], [SRC-MATH-012]
- **Version Notes:** Precision/type aliases changed in UE5; verify target overloads.

## Animation Systems

### Question: How should gameplay communicate with an Animation Blueprint?

- **Category:** Animation / Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Gameplay/movement expose a compact stable presentation snapshot and explicit action requests; animation derives pose and reports correlated cosmetic/timing events without owning durable truth.
- **Strong 3-Year-Engineer Answer:** Health, ammo, action legality and authority remain in gameplay. The AnimBP consumes velocity, movement mode, aim, stance/equipment and current action/revision through property access or a project snapshot suitable for its thread. Gameplay may request a montage/layer change through a narrow interface. Notifies return cosmetic or validated window events tagged with an action ID, and gameplay has cancellation/timeouts if presentation is skipped.
- **Common Weak Answer:** “Cast to the Character in AnimBP Update and read whatever variables animation needs.”
- **Follow-up Questions:** Push or pull? Thread safety? What may a notify change? Late join/relevancy?
- **Hands-on Verification Task:** Replace a casting/polling EventGraph with a documented snapshot/action interface and trace both threads.
- **Sources:** [SRC-ANIM-003], [SRC-ANIM-004], [SRC-ANIM-015]
- **Version Notes:** Property-access call sites and scheduling are target-version-sensitive.

### Question: EventGraph versus AnimGraph and animation update versus evaluation?

- **Category:** Animation / Runtime
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** EventGraph is ordinary game-thread Blueprint update logic; AnimGraph is relevance-driven pose dataflow that can update/evaluate on workers.
- **Strong 3-Year-Engineer Answer:** I separate initialisation, parameter update, node update and pose evaluation. The EventGraph runs on the game thread and is often the expensive polling layer. AnimGraph pose nodes and thread-safe functions can use workers, so arbitrary UObject/world access is unsafe. I use thread-safe update/property access or a native proxy/snapshot, and diagnose both the value timeline and pose-node relevance rather than treating the graph as Tick.
- **Common Weak Answer:** “EventGraph calculates variables and AnimGraph plays animations.”
- **Follow-up Questions:** Property Access? Fast Path? Post Evaluate? What forces game-thread work?
- **Hands-on Verification Task:** Capture one AnimBP before/after moving safe derivation off the EventGraph and compare trace threads/cost.
- **Sources:** [SRC-ANIM-004], [SRC-ANIM-015], [SRC-ANIM-019]
- **Version Notes:** Exact task scheduling and Fast Path eligibility require target verification.

### Question: Animation state machine versus montage?

- **Category:** Animation / Pose Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** State machines model persistent pose regimes with continuously evaluated transitions; montages model directed discrete actions with sections, slots and cancellation.
- **Strong 3-Year-Engineer Answer:** Grounded versus in-air belongs naturally in a locomotion state machine. Reload, melee and hit reaction often belong in montages layered through slots because gameplay starts/cancels them and may jump sections. Gameplay still owns legal action phase. I check whether locomotion continues underneath, root motion/network correlation, slot conflicts and interruption fallback before choosing.
- **Common Weak Answer:** “State machines are for locomotion; montages are for one-shot animations.”
- **Follow-up Questions:** Idle/walk/run states or Blend Space? Combo sections? Persistent death? One-shot node?
- **Hands-on Verification Task:** Implement reload both ways and document transition/action/network complexity.
- **Sources:** [SRC-ANIM-005], [SRC-ANIM-008]
- **Version Notes:** Stable design distinction; concrete node/API behaviour varies.

### Question: How do Blend Spaces and Sync Groups solve different problems?

- **Category:** Animation / Locomotion
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Blend Spaces select/interpolate poses from parameter axes; Sync Groups align playback phase of related animations during blending.
- **Strong 3-Year-Engineer Answer:** I feed a locomotion Blend Space calibrated actual speed and local direction, with compatible samples/ranges. Interpolation alone does not align left/right foot contacts across clips of different lengths, so sync markers/groups coordinate semantic gait phase and leader/follower timing. I inspect wrap, sample triangulation, smoothing, leader changes and notify priority when feet or notifies misbehave.
- **Common Weak Answer:** “Blend Spaces blend walk/run and Sync Groups keep animations at the same speed.”
- **Follow-up Questions:** Aim Offset? Local direction? Markers versus normalised time? Leader notify behaviour?
- **Hands-on Verification Task:** Blend walk/run with mismatched phase, then add markers and record contact/notify behaviour.
- **Sources:** [SRC-ANIM-006], [SRC-ANIM-011]
- **Version Notes:** Maintained sync documentation may show later UE behaviour.

### Question: Why can `Montage_Play` succeed but no montage is visible?

- **Category:** Animation / Debugging
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** The call can start an instance while the required Slot is absent/irrelevant, conflicted, overwritten or incompatible with the active pose graph.
- **Strong 3-Year-Engineer Answer:** I verify mesh/AnimInstance and Skeleton compatibility, then find the montage's Slot node in the currently relevant final graph branch. I inspect slot group conflicts, montage weight/section/play rate, linked layer swaps and later per-bone/aim layers that overwrite the affected bones. On network I identify the requesting machine and replicated action; I log interruption/blend-out/end separately.
- **Common Weak Answer:** “Check the montage Slot name and make sure it is DefaultSlot.”
- **Follow-up Questions:** Slot group concurrency? Upper-body placement? Linked layer? Server versus client?
- **Hands-on Verification Task:** Inject missing Slot, wrong group and late overwrite; diagnose each from graph weights rather than trial-and-error.
- **Sources:** [SRC-ANIM-008], [SRC-ANIM-016]
- **Version Notes:** Montage delegate/API details vary by branch.

### Question: Are Animation Notifies reliable gameplay events?

- **Category:** Animation / Gameplay Integration
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** No; they are timeline events affected by relevance, blending, looping, sync, throttling and network audience, so authoritative gameplay must validate and have durable fallback.
- **Strong 3-Year-Engineer Answer:** I use notifies for cosmetics or to report a timing window to an already authorised action. The receiver checks action/revision and deduplicates. I specify weight threshold, sync leader, queued/branching semantics, server/client audience, looping and cancellation fallback. Damage/ammo/death cannot exist only because a notify happened, especially under dedicated server or animation budgeting.
- **Common Weak Answer:** “Use notifies for footsteps and hit frames, but not important replicated logic.”
- **Follow-up Questions:** Notify State? Missing End? Dedicated server? Sync leader? Exactly once?
- **Hands-on Verification Task:** Skip/throttle/interupt a notify-state melee window and prove authoritative action closes safely.
- **Sources:** [SRC-ANIM-010], [SRC-ANIM-011], [SRC-ANIM-017]
- **Version Notes:** Delivery semantics depend on node/asset/component configuration.

### Question: In-place locomotion versus root motion?

- **Category:** Animation / Movement
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** In-place lets CharacterMovement own displacement; root motion extracts authored root displacement, trading control/network simplicity for grounded authored motion.
- **Strong 3-Year-Engineer Answer:** I use in-place for responsive locomotion and straightforward movement prediction, calibrating clip speed/phase to capsule motion. Root motion suits authored attacks/vaults where trajectory/contact matters, but server authority, collision, interruption, target alignment and correction must be designed. I never let CharacterMovement and root motion both apply the same displacement, and I test dedicated-server lag and blocking geometry.
- **Common Weak Answer:** “Root motion looks better but is harder to replicate.”
- **Follow-up Questions:** Root Motion from Montages? Capsule drift? Motion Warping? Predicting an attack?
- **Hands-on Verification Task:** Compare in-place and root-motion dash under collision and 150 ms latency; capture corrections.
- **Sources:** [SRC-ANIM-008], [SRC-ANIM-009], [SRC-ANIM-014]
- **Version Notes:** Root-motion modes/thread/network paths are highly version-sensitive.

### Question: How do you diagnose foot sliding?

- **Category:** Animation / Debugging
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Correlate capsule speed, clip stride/play rate, Blend Space axes, sync phase, retarget proportions, root-motion mode, network smoothing and IK.
- **Strong 3-Year-Engineer Answer:** I first measure actual movement speed against source clip displacement and calibrated samples. Then check local-direction/ranges, gait markers, play-rate policy and retargeted stride. I inspect mesh-versus-capsule movement from network smoothing and accidental root-motion mixing. Finally I disable foot IK; if sliding remains, IK was not the cause. I fix the responsible layer rather than pinning feet harder.
- **Common Weak Answer:** “Add foot IK and adjust animation play rate to character speed.”
- **Follow-up Questions:** Start/stop? Turning? Simulated proxy only? Unequal proportions?
- **Hands-on Verification Task:** Inject four sliding causes and produce a trace/video diagnosis matrix.
- **Sources:** [SRC-ANIM-006], [SRC-ANIM-009], [SRC-ANIM-011], [SRC-ANIM-013]
- **Version Notes:** CharacterMovement smoothing details are target/version-specific.

### Question: How do you debug a bad IK retarget?

- **Category:** Animation / Retargeting
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Verify source/target skeletons, retarget root/chains/mapping and matching retarget poses before tuning offsets or IK.
- **Strong 3-Year-Engineer Answer:** I reproduce with a simple reference pose/sequence, inspect hierarchy and source/target preview assets, then root/pelvis, chain boundaries/mapping and A/T retarget pose. I disable root, FK and IK passes independently and read the output log. Only after the systemic setup is correct do I tune end effectors/stride/offsets, with LOD and runtime versus export policy documented.
- **Common Weak Answer:** “Adjust the retarget pose and IK goals until hands and feet line up.”
- **Follow-up Questions:** Different bone counts? Root scale? Runtime versus baked? Retarget LOD?
- **Hands-on Verification Task:** Retarget to a differently proportioned mesh and diagnose root drift, arm twist and foot contact separately.
- **Sources:** [SRC-ANIM-013]
- **Version Notes:** UE5.6 Retargeter must not inherit maintained UE5.7 operation-stack assumptions.

### Question: What are Linked Anim Layers for?

- **Category:** Animation / Modularity
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** They define semantic pose extension points that runtime animation classes can override, supporting modular equipment/vehicle logic and asset lifetime boundaries.
- **Strong 3-Year-Engineer Answer:** The base AnimBP owns universal locomotion/final composition; an Animation Layer Interface defines contracts such as WeaponUpperBody. The equipment system selects a layer class whose assets can load with that equipment. I document input pose/curves/root-motion expectations, transition/cancellation on swap and avoid a hard reference from the base class that defeats streaming. I profile duplicate locomotion evaluation.
- **Common Weak Answer:** “Linked layers split a large AnimBP into reusable graphs.”
- **Follow-up Questions:** Linked Anim Graph versus layer? Interface? Asset unloading? Montage on swap?
- **Hands-on Verification Task:** Swap rifle/bow layers while a montage is active and audit references, callbacks and pose continuity.
- **Sources:** [SRC-ANIM-012]
- **Version Notes:** Linking/templates/lifetime behaviour evolves across UE5.

### Question: How do you profile and optimise animation for many characters?

- **Category:** Animation / Performance
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Separate game-thread data gathering, worker pose evaluation, skeletal finalisation, skinning, IK queries, assets and cosmetics; then reduce work by thread safety, relevance and significance/LOD.
- **Strong 3-Year-Engineer Answer:** I capture Insights/Animation Insights with character counts and LOD/significance. First remove EventGraph casts/polling and unsafe Blueprint work using thread-safe update/property access. Then identify expensive duplicate pose branches, IK/Control Rig/curves and bone counts. I apply skeletal LOD, lower update frequency/interpolation, Leader/Copy Pose where valid and budget traces/effects. The Budget Allocator is plugin-dependent and requires must-update policies for local/critical/ragdoll/event cases.
- **Common Weak Answer:** “Use Fast Path, Update Rate Optimisation and disable IK at distance.”
- **Follow-up Questions:** Game versus worker thread? Render skinning? Essential character? Notifies under throttling?
- **Hands-on Verification Task:** Scale 1/20/100 characters and produce a before/after trace with quality failure tests.
- **Sources:** [SRC-ANIM-015], [SRC-ANIM-016], [SRC-ANIM-017], [SRC-ANIM-019]
- **Version Notes:** Tools and allocator are plugin/version-sensitive.

### Question: Motion Warping and IK—what do they solve, and what do they not solve?

- **Category:** Animation / Procedural
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Motion Warping adapts root-motion windows to a runtime target; IK adjusts joints toward goals. Neither validates gameplay reachability or replaces collision/authority.
- **Strong 3-Year-Engineer Answer:** For a vault/finisher, gameplay validates a target and provides a warp target with lifetime/fallback; warping adjusts translation/rotation over an authored window. IK then handles local hand/foot contact in the correct space and graph order. I enforce solver limits, smooth contact, respect LOD and budget world traces. If the target becomes invalid or collision blocks motion, gameplay cancels/replans rather than forcing a visual solve.
- **Common Weak Answer:** “Motion Warping aligns the character to targets; IK prevents feet/hands clipping.”
- **Follow-up Questions:** Goal space? Solver order? Overextension? Network target? LOD?
- **Hands-on Verification Task:** Warp to valid/invalid ledges and feed one goal in the wrong space; demonstrate safe fallback.
- **Sources:** [SRC-ANIM-013], [SRC-ANIM-014], [SRC-ANIM-018]
- **Version Notes:** Plugin/tool APIs are highly version-sensitive.

## Physics and Collision

### Question: Explain Unreal collision filtering without saying “set the preset correctly”.

- **Category:** Physics / Collision Filtering
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** A PrimitiveComponent's collision-enabled mode, Object Type and response to the other object's or trace's channel jointly determine Ignore, Overlap or Block; a profile packages that policy.
- **Strong 3-Year-Engineer Answer:** I separate query and simulation participation. For object interaction I inspect both Components' Object Types/responses; for a trace by channel I inspect the target's response to that semantic trace channel; for object queries I select categories. I use named project profiles, audit runtime overrides and test the matrix automatically. Notification generation is separate from filtering.
- **Common Weak Answer:** “Each object has a collision channel and chooses Block, Overlap or Ignore.”
- **Follow-up Questions:** Channel versus object query? Query-only? Bilateral response? Profile override?
- **Hands-on Verification Task:** Build a six-profile matrix and assert every pair/query result in automation.
- **Sources:** [SRC-PHYS-001]
- **Version Notes:** UE5.6 source; some documentation prose retains UE4 wording.

### Question: Does a blocking collision guarantee an Event Hit?

- **Category:** Physics / Events
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** No; blocking response and movement/simulation notification settings/path are separate, and callbacks are not an exactly-once gameplay protocol.
- **Strong 3-Year-Engineer Answer:** Blocking can stop movement or create solver contacts without generating the Blueprint/C++ hit notification unless the relevant movement/simulation notification path is enabled. Resting contact, substeps and multiple bodies can repeat callbacks. I log component/body/action IDs and deduplicate product consequences; durable damage/score remains server-owned rather than “one callback equals one event”.
- **Common Weak Answer:** “Enable Simulation Generates Hit Events on both objects.”
- **Follow-up Questions:** Swept movement? Resting contact? Begin/End same frame? Overlap generation?
- **Hands-on Verification Task:** Compare kinematic sweep and two simulated bodies with notification settings and count callbacks under substeps.
- **Sources:** [SRC-PHYS-001], [SRC-PHYS-005]
- **Version Notes:** Callback details depend on movement/component/solver path.

### Question: Trace by channel versus query by object type?

- **Category:** Physics / Scene Queries
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Channel asks each target how it responds to a semantic query; object query selects targets by their intrinsic Object Types.
- **Strong 3-Year-Engineer Answer:** I use a semantic channel for Visibility/Camera/Interaction/Weapon-style questions because target profiles can independently block/overlap/ignore that query. I use object types when collecting categories such as Pawns or PhysicsBodies. I do not use Visibility simply because it returns a hit. I document single/multi and Block/Overlap stopping semantics and exact Components required.
- **Common Weak Answer:** “Channel checks one layer; object trace can check several object channels.”
- **Follow-up Questions:** Multi by channel stopping? Profile schema? Custom channel migration?
- **Hands-on Verification Task:** Implement one interaction query both ways and demonstrate a shield/component policy that needs semantic channel filtering.
- **Sources:** [SRC-PHYS-001], [SRC-PHYS-002]
- **Version Notes:** Exact multi-query ordering/semantics require target API verification.

### Question: Line trace, shape sweep or overlap?

- **Category:** Physics / Scene Queries
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Line samples a zero-volume segment; sweep moves a volume along a segment; overlap tests a volume at one pose.
- **Strong 3-Year-Engineer Answer:** Hitscan/visibility often starts with a line. Character clearance/melee/forgiving interaction uses the product shape swept across motion. Spawn/area occupancy uses overlap. I define channel/object filtering, simple/complex, ignored components, initial penetration and single/multi. A sphere trace is a swept sphere, not a stationary radius check.
- **Common Weak Answer:** “Use line for accuracy, sphere/capsule for wider detection and overlap for triggers.”
- **Follow-up Questions:** Continuous collision? Initial overlap? Shape orientation? Query cost?
- **Hands-on Verification Task:** Compare all three on thin/off-axis/starting-inside targets and visualise accepted volume.
- **Sources:** [SRC-PHYS-002]
- **Version Notes:** Stable distinction; exact helpers vary.

### Question: `Location` versus `ImpactPoint`, and `Normal` versus `ImpactNormal` in a swept hit?

- **Category:** Physics / Hit Results
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Location is the swept shape's safe location; ImpactPoint is contact on hit geometry. Normal relates to the swept shape/contact; ImpactNormal describes hit surface.
- **Strong 3-Year-Engineer Answer:** For a line they can coincide, hiding the distinction. For a sphere/capsule sweep the shape centre at contact differs from surface contact. I also inspect blocking/initial-penetration, normalised Time/Distance and actual hit Component/body/bone. Initial overlap can produce zero distance and depenetration-oriented data, so it is not an ordinary impact.
- **Common Weak Answer:** “ImpactPoint is more accurate; ImpactNormal is the surface normal.”
- **Follow-up Questions:** Initial penetration? Bone/face/physical material validity? Multi-components?
- **Hands-on Verification Task:** Draw all four values for line/sphere/capsule sweeps against a sloped wall and starting overlap.
- **Sources:** [SRC-PHYS-002]
- **Version Notes:** Read exact target `FHitResult` contract.

### Question: Simple versus complex collision?

- **Category:** Physics / Geometry
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Simple uses primitives/convex gameplay shapes suited to robust queries/simulation; complex uses render triangles for detailed queries with greater cost/restrictions.
- **Strong 3-Year-Engineer Answer:** I author simple collision to the movement/simulation/gameplay silhouette, and request complex only for precise static geometry, face/material/UV needs. Complex-as-simple can have simulation restrictions and triangle detail can snag or cost memory/query time. I inspect cooked runtime collision and benchmark representative candidates rather than equating visual accuracy with better physics.
- **Common Weak Answer:** “Simple is faster and complex is per-poly accurate.”
- **Follow-up Questions:** Convex decomposition? Moving trimesh? Physical material? QueryComplex flag?
- **Hands-on Verification Task:** Compare chair movement/projectile queries across simple, simple+complex and complex-as-simple variants.
- **Sources:** [SRC-PHYS-003], [SRC-PHYS-007]
- **Version Notes:** Cooker/platform/complex simulation behaviour is target-sensitive.

### Question: CharacterMovement versus rigid-body movement?

- **Category:** Physics / Movement Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** CharacterMovement is capsule/sweep-based authored movement with floor modes and mature prediction; rigid bodies are force/contact-driven solver state.
- **Strong 3-Year-Engineer Answer:** For a responsive humanoid I use CharacterMovement and let the capsule own transform, deriving animation from it. A physics Pawn fits mechanics where mass/contact/forces are the movement model and justifies custom control/network work. I transition explicitly for root motion/ragdoll and never teleport/set transforms every frame while expecting an independently simulated body to remain authoritative.
- **Common Weak Answer:** “CharacterMovement is kinematic; physics is for realistic movement.”
- **Follow-up Questions:** Physics interaction? Root motion? Ragdoll? Network prediction?
- **Hands-on Verification Task:** Implement one pushable movement interaction with Character, swept kinematic object and rigid body; document ownership.
- **Sources:** [SRC-PHYS-004], [SRC-NET-005], [SRC-ANIM-009]
- **Version Notes:** Movement and physics-network paths vary by branch.

### Question: Force versus impulse versus setting velocity?

- **Category:** Physics / Rigid Bodies
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Force accumulates over simulation time, impulse changes momentum immediately, and direct velocity imposes state while potentially overriding solver continuity.
- **Strong 3-Year-Engineer Answer:** I use continuous force/thrust through the supported timestep path, an impulse for discrete hits/explosions and force/impulse at location when torque is intended. I verify mass-scaling flags and which component/body receives it. Setting velocity is appropriate for initialisation or explicit control/correction, not as a universal movement loop. I test sleep/wake and substep behaviour.
- **Common Weak Answer:** “Force is per frame, impulse is one-shot and velocity skips mass.”
- **Follow-up Questions:** At location? Mass/inertia? Accel/velocity change? Physics callback?
- **Hands-on Verification Task:** Compare equal force/impulse on two masses and off-centre application with centre-of-mass visualisation.
- **Sources:** [SRC-PHYS-004], [SRC-MATH-012]
- **Version Notes:** Exact Chaos/Component API flags require target docs/source.

### Question: What does physics substepping solve, and what does it not solve?

- **Category:** Physics / Timestep
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** It splits long frame deltas into smaller solver steps for stability/accuracy at CPU/bookkeeping cost; it does not guarantee determinism or fix bad geometry/filtering/constraints.
- **Strong 3-Year-Engineer Answer:** I tune max substep delta/count from quality and worst frame time. I expect repeated/delayed contact callbacks because solver substeps queue results for the game thread. I still need CCD/sweeps for tunnelling, sane mass ratios/constraint frames and supported physics-thread data access. I compare solver and game-frame timelines rather than assuming one Tick equals one physics step.
- **Common Weak Answer:** “Substepping makes physics frame-rate independent and more deterministic.”
- **Follow-up Questions:** Callback duplicates? Max Physics Delta? Async physics? CPU cost?
- **Hands-on Verification Task:** Run a constrained stack/ragdoll at controlled hitches with different substep limits and record stability/callback count/cost.
- **Sources:** [SRC-PHYS-005], [SRC-PHYS-009]
- **Version Notes:** Maintained substep page has historical caveats; verify target/platform.

### Question: How do you debug an unstable physics constraint?

- **Category:** Physics / Constraints
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Inspect body/constraint local frames, mass/inertia, limits/drives/collision and timestep before increasing solver strength/iterations.
- **Strong 3-Year-Engineer Answer:** I record the minimal two-body case and draw both constraint frames/axes. I verify body assignment, free/limited/locked degrees, target, stiffness/damping/max force and self-collision. Then inspect penetrations, extreme mass ratio, tiny shapes, long chain and timestep/substeps. Contradictory hard limits or an axis mismatch cannot be cured reliably with enormous drive strength.
- **Common Weak Answer:** “Increase solver iterations, damping and constraint stiffness until jitter stops.”
- **Follow-up Questions:** Swing versus twist? Projection? Break threshold? Motor target?
- **Hands-on Verification Task:** Build a door hinge, rotate one frame intentionally and diagnose the resulting wrong degree of freedom from CVD.
- **Sources:** [SRC-PHYS-006], [SRC-PHYS-009]
- **Version Notes:** Constraint options/solver behaviour are version-sensitive.

### Question: Describe a safe animation-to-ragdoll-and-back transition.

- **Category:** Physics / Ragdoll
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Transfer ownership in an ordered state transition among gameplay, capsule/CharacterMovement, Skeletal Mesh Physics Asset and network presentation.
- **Strong 3-Year-Engineer Answer:** Server commits ragdoll/death, captures pose/velocity, disables/reconfigures movement and capsule collision, then enables Physics Asset simulation/profile and seeds momentum once. For recovery I wait for a valid state, choose pelvis orientation, find collision-safe capsule placement, align get-up, stop/blend simulation and restore movement in one order. Capsule and ragdoll never both push as transform owners.
- **Common Weak Answer:** “Disable CharacterMovement, enable physics below pelvis and later teleport capsule to pelvis.”
- **Follow-up Questions:** Attachments/camera? Simulated proxy? Rest detection? Get-up side?
- **Hands-on Verification Task:** Ragdoll on slope/stairs/wall under latency and recover without penetration or duplicate impulse.
- **Sources:** [SRC-PHYS-008], [SRC-ANIM-016]
- **Version Notes:** Physical-animation/network details are project/version-sensitive.

### Question: Why is multiplayer rigid-body physics difficult, and what modes should you know?

- **Category:** Physics / Networking
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Contacts diverge under latency; server authority must reconcile client simulation. Know default correction and awareness of Predictive Interpolation/Resimulation only with target-version verification.
- **Strong 3-Year-Engineer Answer:** The server owns final gameplay state. Default replicated physics corrects client body state but local interactions can be overwritten. Maintained docs describe Predictive Interpolation for anticipated local interactions and Resimulation with cached history/input replay, at CPU/memory/network complexity. I do not claim those later maintained paths for UE5.3–UE5.6 without source/status proof. I validate client intent, bound relevant bodies and use CVD/server-client recordings for divergence.
- **Common Weak Answer:** “Enable replicated movement; for prediction use Network Physics Component and resimulation.”
- **Follow-up Questions:** What history? Server/client tick offset? Cosmetic debris? Throw request validation?
- **Hands-on Verification Task:** Compare a pushed/thrown body under lag/loss and document correction, authority and target-supported replication mode.
- **Sources:** [SRC-PHYS-010], [SRC-PHYS-009], [SRC-NET-001]
- **Version Notes:** Highly version-sensitive specialist area; maintained docs are later UE5.

### Question: How do you profile a physics-heavy scene?

- **Category:** Physics / Performance
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Separate scene queries, active bodies/broadphase contacts, solver/constraints, callbacks, substeps, Physics Assets and network history/corrections.
- **Strong 3-Year-Engineer Answer:** I record body awake/sleep counts, candidate/contact/island/constraint work, query shape/count/candidates, callback volume, substeps and sync/wait time. CVD reconstructs shapes, filters, contacts, joints and queries; Insights gives frame/thread context. I remove unnecessary collision participation, simplify shapes/profiles, reduce query/callback frequency and reserve CCD/substeps/ragdoll detail/network history for significant objects, checking downstream VFX/Blueprint cost too.
- **Common Weak Answer:** “Use stat physics, Insights and simplify collision/put objects to sleep.”
- **Follow-up Questions:** Query or solver bound? Complex collision? Waking? Callback cascade? Network memory?
- **Hands-on Verification Task:** Scale 10/100/1000 props and produce a phase/count-led optimisation report.
- **Sources:** [SRC-PHYS-009], [SRC-PERF-003], [SRC-PHYS-005]
- **Version Notes:** Stat names/CVD features vary by target.

## UI Systems

### Question: Explain `OnInitialized`, `Construct`, activation and `Destruct` without calling them UI `BeginPlay`/`EndPlay`.

- **Category:** UI / Lifecycle
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** They correspond to different lifetimes: UObject instance, underlying Slate representation and screen activity; `Construct` can repeat.
- **Strong 3-Year-Engineer Answer:** I use `OnInitialized` for once-per-runtime-instance setup, repeatable `Construct` or explicit `BindModel` for the current Slate/model binding, and CommonUI activation for “this screen now owns interaction”. Every acquisition has a symmetric release at the same boundary. Removing from the viewport does not imply immediate UObject collection, so delegates, timers, async work and service references need explicit ownership.
- **Common Weak Answer:** “Construct is BeginPlay, Destruct is EndPlay, and CommonUI activation just changes visibility.”
- **Follow-up Questions:** PreConstruct design time? Rebuild? GC reference? Duplicate bindings?
- **Hands-on Verification Task:** Reopen/re-add one widget ten times, log all hooks and prove exactly one model callback per change.
- **Sources:** [SRC-UI-002], [SRC-UI-009]
- **Version Notes:** Core lifecycle concept is stable; maintained API page resolves UE5.7.

### Question: How should gameplay state reach a HUD?

- **Category:** UI / Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Through a local-player presentation projection: subscribe, render an initial snapshot, apply deltas, and keep gameplay authoritative.
- **Strong 3-Year-Engineer Answer:** A controller/local-player-owned presenter reads the correct replicated/local source, subscribes to changes, creates display-ready state and renders an initial snapshot. Widgets emit intents rather than mutating authoritative data. I rebind on possession/travel and represent pending/confirmed/rejected transactions explicitly. Revisions or serial game-thread ordering close the subscribe/snapshot race.
- **Common Weak Answer:** “Bind widget properties directly to Character variables or cast to the player every Tick.”
- **Follow-up Questions:** Initial snapshot race? PlayerState versus Pawn? Split-screen? Server rejection?
- **Hands-on Verification Task:** Build health/inventory projection that survives possession swap and late widget reconstruction.
- **Sources:** [SRC-UI-012], [SRC-NET-001]
- **Version Notes:** MVVM is Beta/plugin-dependent; the architectural contract does not require it.

### Question: Property binding, FieldNotify event or widget Tick—how do you choose?

- **Category:** UI / Update Performance
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Prefer bounded push updates for discrete changes, Tick for genuine continuous presentation, and measure pull bindings in their visible hierarchy.
- **Strong 3-Year-Engineer Answer:** Legacy property bindings/Slate attributes are pull paths and can repeatedly evaluate formatting/allocation. For health, inventory and labels I update on gameplay events or FieldNotify and set widget properties once per change. Interpolation/animation may Tick while active, then disable itself. I use Slate Insights to count updated/invalidated/painted widgets before replacing a harmless small binding.
- **Common Weak Answer:** “Bindings are slow, so always replace every binding with an event.”
- **Follow-up Questions:** Initial value? Event storm? Volatile widget? Hidden screen?
- **Hands-on Verification Task:** Compare 100 rows using Blueprint bindings, Tick and batched event updates in the same trace.
- **Sources:** [SRC-UI-004], [SRC-UI-005], [SRC-UI-012]
- **Version Notes:** MVVM Beta and UE5.7 optimisation guidance require target verification.

### Question: How does Unreal UI layout decide a widget's size and position?

- **Category:** UI / Layout
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Children compute Desired Size bottom-up; parents arrange allotted geometry top-down according to panel-slot constraints.
- **Strong 3-Year-Engineer Answer:** I choose flow panels for reflow and Canvas only for intentional anchored layers. To debug, I find the first parent where desired and allotted geometry diverge, then inspect slot fill/alignment, anchors/offsets, wrappers, DPI and safe zone. I test localisation expansion and aspect ratios instead of adding arbitrary SizeBoxes.
- **Common Weak Answer:** “Anchors and ScaleBox make the layout responsive.”
- **Follow-up Questions:** Point versus split anchor? Hidden versus Collapsed? DPI curve? Long text?
- **Hands-on Verification Task:** Make one panel survive narrow, ultrawide, safe-zone and pseudo-localised layouts without per-resolution branches.
- **Sources:** [SRC-UI-001], [SRC-UI-006], [SRC-UI-007], [SRC-UI-008]
- **Version Notes:** Stable architecture; editor controls vary.

### Question: Why can a visible enabled button fail to receive a click?

- **Category:** UI / Hit Testing
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Another widget/parent visibility mode may own the hit-test path, geometry may differ from appearance, or the wrong user/input layer may be active.
- **Strong 3-Year-Engineer Answer:** I use Widget Reflector's hit-testable picker/grid, inspect the exact geometry and walk overlays/parents for hit-test visibility. Then I check enabled/interactable state, clipping and which Slate user has focus. With CommonUI I log the active leaf, input config, cursor/capture and mapping transitions instead of randomly toggling visibility.
- **Common Weak Answer:** “Set the button and all its parents to Visible and make sure Z Order is high.”
- **Follow-up Questions:** Self Hit Test Invisible? Focus? Mouse capture? Modal layer?
- **Hands-on Verification Task:** Inject a transparent overlay intercept and diagnose it solely from the hit-test path.
- **Sources:** [SRC-UI-003], [SRC-UI-010]
- **Version Notes:** Widget Reflector UI varies by minor version.

### Question: Design controller focus/navigation for a modal screen.

- **Category:** UI / Input and Focus
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Define focus per local user, a deterministic initial target, navigation graph, and restoration when the modal closes or the target disappears.
- **Strong 3-Year-Engineer Answer:** On activation I select a valid desired focus target for the owning local player and define directional/escape behaviour. I handle disabled/removed focused entries and restore the previous screen's focus. CommonUI can coordinate the active screen and input config; I avoid competing direct controller/viewport mode calls. I test mouse↔gamepad transitions and split-screen separately.
- **Common Weak Answer:** “Set keyboard focus to the first button on Construct.”
- **Follow-up Questions:** Per-user focus? List recycling? Back action? No active CommonUI screen?
- **Hands-on Verification Task:** Navigate, disable and remove the focused row, open/close a modal, and prove focus never gets lost.
- **Sources:** [SRC-UI-010], [SRC-UI-017]
- **Version Notes:** CommonUI is plugin-dependent; Enhanced Input integration is version-sensitive.

### Question: What problem does CommonUI solve, and when would you not adopt it?

- **Category:** UI / CommonUI
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It structures complex multiplatform menu layers, activation, action routing/glyphs and navigation; small/simple UI may not justify its plugin conventions.
- **Strong 3-Year-Engineer Answer:** I adopt it when the product needs stacked screens/modals, controller-first navigation, platform action glyphs and one coherent input-routing policy. I distinguish construction from activation and define root/default input config. I may keep plain UMG for a bounded HUD or tool where the stack/style/input machinery adds migration and version risk. I verify Enhanced Input support in the target branch.
- **Common Weak Answer:** “CommonUI is Epic's newer replacement for UMG and Enhanced Input.”
- **Follow-up Questions:** Activatable leaf? Input restoration? Styles? Plugin/version risk?
- **Hands-on Verification Task:** Implement the same modal flow in UMG and CommonUI and compare ownership/transition complexity.
- **Sources:** [SRC-UI-009], [SRC-UI-010], [SRC-UI-011]
- **Version Notes:** Plugin-dependent and version-sensitive.

### Question: Why does a virtualised inventory row show the wrong icon or selection?

- **Category:** UI / Lists and Async
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Entry widgets are recycled; old state, subscriptions or async callbacks survived rebinding to a new item.
- **Strong 3-Year-Engineer Answer:** The item/model is durable identity and the row is temporary presentation. On assignment I fully reset and bind; on release I unsubscribe/cancel/invalidate. Async completion carries stable item ID and request generation plus a weak entry, and updates only if all still match. Selection is model/list state, not a colour retained in the row.
- **Common Weak Answer:** “Call RequestRefresh after every icon finishes loading.”
- **Follow-up Questions:** Entry release? Shared load? Scroll restoration? Duplicate item objects?
- **Hands-on Verification Task:** Scroll 1000 rows under delayed random icon loads and assert no stale image/selection appears.
- **Sources:** [SRC-UI-013], [SRC-UI-014], [SRC-ASSET-002]
- **Version Notes:** Exact ListView callbacks differ by surface/version; recycling contract is central.

### Question: When should UI assets be hard referenced versus loaded asynchronously?

- **Category:** UI / Assets
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Hard-reference small always-resident UI foundations; soft/async-load large or optional content when startup closure/residency matters, with placeholders and stale-result guards.
- **Strong 3-Year-Engineer Answer:** I audit the dependency closure. Core fonts/styles/icons can be resident; catalog portraits tied to optional items/DLC should usually be soft and loaded through a coalesced service. Requests are bounded/prioritised for visible rows, cancellable where possible and accepted only for the current item generation. I distinguish load failure from missing cook inclusion.
- **Common Weak Answer:** “Use soft references for all UI images so loading is faster.”
- **Follow-up Questions:** Residency owner? Cook rules? Shared handles? Placeholder flicker?
- **Hands-on Verification Task:** Compare startup/residency of hard-referenced and soft-loaded 1000-icon catalogues in a packaged build.
- **Sources:** [SRC-ASSET-001], [SRC-ASSET-002], [SRC-ASSET-006]
- **Version Notes:** Exact async handle and cook APIs are target-sensitive.

### Question: Screen-space marker or world-space `WidgetComponent`?

- **Category:** UI / World Widgets
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Choose from occlusion/diegesis/input/readability and scaling; world widgets add scene/render-target work and are not a free nameplate solution.
- **Strong 3-Year-Engineer Answer:** A world-space component fits a diegetic panel that should be transformed/occluded in 3D. A projected screen-space marker often scales better and remains readable for many actors. I control draw size/redraw cadence, create only by relevance/significance, and compare 1/20/100 actors including GPU overdraw/render-target memory. Input and local-player audience remain explicit.
- **Common Weak Answer:** “World space is immersive; screen space is always on top.”
- **Follow-up Questions:** Draw at Desired Size? Manual redraw? VR? Occlusion? Pooling?
- **Hands-on Verification Task:** Profile both implementations for 100 moving actors with unchanged and rapidly changing labels.
- **Sources:** [SRC-UI-015], [SRC-UI-004]
- **Version Notes:** Maintained pages; exact rendering behaviour must be measured on target.

### Question: How do you make a UI localisation- and accessibility-ready?

- **Category:** UI / Product Quality
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Preserve `FText`, use culture-aware formatting and flexible layouts; test expansion/RTL/fonts plus contrast, scaling, redundant cues, labels and all input paths.
- **Strong 3-Year-Engineer Answer:** User-facing text stays `FText`; I avoid lossy FString round-trips and use plural/number/date formatting. Flow layout is tested with pseudo-localisation, long/RTL strings and font fallback at supported DPI/text scales. State never relies only on colour; controls have meaningful labels, adequate targets and keyboard/controller/touch paths. Platform accessibility integration is verified on hardware, not inferred from API existence.
- **Common Weak Answer:** “Put strings in a string table and add subtitles/colour-blind mode.”
- **Follow-up Questions:** Text history? Glyph fallback? Focus order? Safe zones? Screen reader?
- **Hands-on Verification Task:** Run a pseudo-localised accessibility pass and record every overflow, missing glyph and unreachable control.
- **Sources:** [SRC-UI-016], [SRC-UI-018], [SRC-UI-006]
- **Version Notes:** Platform support and project requirements vary.

### Question: How do you profile and optimise an expensive UMG screen?

- **Category:** UI / Performance
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Classify model/binding, update, layout, paint, draw submission, asset and GPU/overdraw costs, then optimise the measured stage.
- **Strong 3-Year-Engineer Answer:** I reproduce a fixed interaction and correlate Slate Insights' updated/invalidated/painted widgets with Widget Reflector hierarchy/invalidation reasons, Game/Render timing and GPU capture. I count visible widgets, generated rows, bindings/ticks, draw elements/batches, render targets and texture/font memory. Event updates, virtualisation, shallower hierarchy and valid invalidation come before speculative Retainers; Retainer/volatility/redraw cadence are measured trade-offs.
- **Common Weak Answer:** “Add an Invalidation Box and Retainer Panel, remove bindings, and reduce widget count.”
- **Follow-up Questions:** Layout versus paint invalidation? Volatile? Overdraw? Hidden animation? Font atlas?
- **Hands-on Verification Task:** Produce before/after traces for a 1000-item inventory and attribute each gain to one changed mechanism.
- **Sources:** [SRC-UI-003], [SRC-UI-004], [SRC-UI-005]
- **Version Notes:** UE5.7 optimisation guidance is conceptual for target; tool/UI details vary.

## Audio Systems

### Question: Sound Wave, Sound Cue or MetaSound Source?

- **Category:** Audio / Source Design
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Wave is recorded content/properties, Cue is an authored Sound Node behaviour graph, and MetaSound is a procedural DSP rendering graph with sample-accurate internal control.
- **Strong 3-Year-Engineer Answer:** I use a Wave for a simple stable one-shot/stem, a Cue for sample-based randomisation/modulation/branching, and MetaSound when procedural synthesis or audio-buffer/sample timing justifies graph/runtime complexity. Gameplay decides semantic state and passes bounded parameters; the source graph shapes sound. I benchmark polyphony because each active procedural graph still costs DSP.
- **Common Weak Answer:** “Sound Cue is the old system and MetaSound is the modern replacement.”
- **Follow-up Questions:** Named parameters? Sample accuracy? Reuse? Polyphony?
- **Hands-on Verification Task:** Implement one weapon source all three ways and justify/measure each version.
- **Sources:** [SRC-AUDIO-001], [SRC-AUDIO-007]
- **Version Notes:** MetaSound node/runtime details are version-sensitive.

### Question: Fire-and-forget sound versus owned Audio Component?

- **Category:** Audio / Lifetime
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Use one-shot when no later control/follow is needed; own a Component for attachment, loop, fade, parameters, callbacks or explicit stop/reuse.
- **Strong 3-Year-Engineer Answer:** A persistent engine/charge/ambience loop has one owner and explicit start/update/fade/stop transitions. I do not restart it each Tick. Attached spawned sources define stop-on-owner-destroy and auto-destroy; retained Components handle travel/deactivation/pooling resets. Auto-destroyed pointers cannot be assumed valid after completion.
- **Common Weak Answer:** “Use PlaySound for short sounds and AudioComponent for long sounds.”
- **Follow-up Questions:** Attachment? Auto-destroy? Pool reset? Owner destruction?
- **Hands-on Verification Task:** Inject destroyed owner, pooled Actor and repeated-start failures into one loop and one one-shot.
- **Sources:** [SRC-AUDIO-002]
- **Version Notes:** Exact API signatures/flags require target check.

### Question: Explain attenuation beyond “sound gets quieter with distance”.

- **Category:** Audio / Spatialisation
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** It packages listener-relative volume shape/curve plus spatialisation, focus, occlusion, air absorption, priority and environment-send policy.
- **Strong 3-Year-Engineer Answer:** I validate listener, source transform, inner/outer shape and effective gain first. Spatialisation maps direction to target output/HRTF; occlusion is a scheduled geometry query driving interpolation/filtering, not physical acoustics. Focus/distance can also scale priority and sends. I budget queries and specialised spatialisation by significance and test multiple listeners/platform output.
- **Common Weak Answer:** “Set a radius, enable spatialisation and occlusion.”
- **Follow-up Questions:** Mono/stereo? Trace channel? Focus? Split-screen? Reverb send?
- **Hands-on Verification Task:** Visualise and profile a moving source through a doorway across three attenuation/occlusion cadences.
- **Sources:** [SRC-AUDIO-003]
- **Version Notes:** Spatialisation plugins/platform behaviour are target-sensitive.

### Question: What does Sound Concurrency solve?

- **Category:** Audio / Voice Management
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** It limits semantic groups of active instances and resolves new requests by owner scope, retrigger and reject/replace/scale/steal policy.
- **Strong 3-Year-Engineer Answer:** I define groups from perceptual/product constraints, such as footsteps per owner and globally or protected dialogue—not one group for everything. Owner-limited policy needs a meaningful playback owner. When full, priority/distance/age/quietness policy may reject or steal; release fades avoid clicks. I log requests, rejections, steals, active/virtual voices rather than calling every cut-out an attenuation bug.
- **Common Weak Answer:** “Concurrency caps how many copies of a sound can play.”
- **Follow-up Questions:** Owner missing? Retrigger? Priority? Virtualisation? Platform voice cap?
- **Hands-on Verification Task:** Force every configured resolution path with 100 impacts from several owners.
- **Sources:** [SRC-AUDIO-004], [SRC-AUDIO-003]
- **Version Notes:** UE5.6 Python API evidences settings; use target runtime asset/API.

### Question: Sound Class/Sound Mix versus Submix?

- **Category:** Audio / Mixing and Routing
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Classes categorise/control source properties and temporary mixes; Submixes route and process actual combined audio buffers through DSP.
- **Strong 3-Year-Engineer Answer:** Master/Music/SFX/Dialogue/UI classes provide hierarchical category/user control. Mixes temporarily adjust them with balanced lifetime. Sources then route dry output and sends through a documented Submix graph for shared reverb, radio, dynamics or endpoint work. Per-source effects multiply by voices; a compatible Submix effect processes the combined bus, though its graph may run continuously.
- **Common Weak Answer:** “Sound Classes are volume buses and Submixes are effects buses.”
- **Follow-up Questions:** Passive mix? Dry/send path? Source effect? Always-running graph? User slider?
- **Hands-on Verification Task:** Draw and verify a dialogue-duck/reverb route with overlapping duck requests.
- **Sources:** [SRC-AUDIO-005], [SRC-AUDIO-006]
- **Version Notes:** Legacy/plugin fields require target interpretation.

### Question: What problem does Quartz solve, and what does it not solve?

- **Category:** Audio / Timing
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** It schedules rendering to sample/musical time across game/audio buffer boundaries; it does not remove device latency or make gameplay/network state deterministic.
- **Strong 3-Year-Engineer Answer:** Normal game-thread commands are consumed around audio-buffer timing and jitter. Quartz schedules ahead to seconds/bars/beats for sample-accurate transitions/cadence. MetaSound handles source-internal sample timing. Network music still needs shared epoch, drift and late-join policy, and gameplay authority remains separate.
- **Common Weak Answer:** “Quartz gives zero-latency sample-accurate audio.”
- **Follow-up Questions:** Buffer size? Clock? Late command? Network synchronisation?
- **Hands-on Verification Task:** Compare game-Timer and Quartz repeated cadence under game-thread hitches.
- **Sources:** [SRC-AUDIO-008], [SRC-AUDIO-007]
- **Version Notes:** Maintained Quartz source is UE5.7; pin target API/status.

### Question: Why can a loaded Sound Wave still hitch or miss its first playback?

- **Category:** Audio / Streaming and Memory
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** The UObject can be loaded while required compressed chunks/decoder data are not resident or ready in the stream cache.
- **Strong 3-Year-Engineer Answer:** I distinguish asset load from compressed chunk residency and decoded/render buffers. Latency-critical short sounds may be retained/primed; long music/dialogue streams. I inspect cache size/chunk utilisation, evictions, concurrent streams, I/O and underruns in a cooked target build. Increasing cache trades memory for latency and can still fail with poor chunk distribution/policy.
- **Common Weak Answer:** “Preload the SoundWave or increase the audio cache.”
- **Follow-up Questions:** Retain/prime/on-demand? Chunk count? Decode CPU? Trim risk?
- **Hands-on Verification Task:** Create a cold-cache burst plus long streams and compare loading policies/memory/first-play latency.
- **Sources:** [SRC-AUDIO-009], [SRC-AUDIO-005]
- **Version Notes:** Maintained cache details/CVars are later-version and platform-sensitive.

### Question: How should audio work in multiplayer?

- **Category:** Audio / Networking
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Replicate authoritative gameplay state or bounded events; each client renders relevant local audio and deduplicates predicted versus confirmed feedback.
- **Strong 3-Year-Engineer Answer:** Durable engine/ability state replicates so late clients reconstruct loops. Important transients carry stable event/sequence identity and relevancy. Owners can predict responsive fire feedback, then correlate acceptance and suppress duplicate replicated playback. UI/local settings stay local. Dedicated-server gameplay never depends on an audio device or Component.
- **Common Weak Answer:** “Multicast PlaySoundAtLocation from the server.”
- **Follow-up Questions:** Late join? Relevancy? Prediction rejection? Loop stop? Dedicated server?
- **Hands-on Verification Task:** Test predicted fire and replicated engine loop with two clients, packet delay and late join.
- **Sources:** [SRC-NET-001], [SRC-AUDIO-002]
- **Version Notes:** Audio remains presentation; event transport policy is project-specific.

### Question: A sound request happened but nothing was heard. What is your workflow?

- **Category:** Audio / Debugging
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Trace source/component, listener/attenuation, admission, gain/routing, data/render and output device in order.
- **Strong 3-Year-Engineer Answer:** I log semantic event ID, source, owner, Component, World, transform and parameters. Then inspect active/stopped lifetime, listener distance and effective attenuation; concurrency rejection/steal/virtualisation and priority; Wave/Component/Class/Mix gain; Submix route/effects; stream chunk/cache/underrun; and platform device. Runtime audio CVars/logs support each hypothesis, with target help because names change.
- **Common Weak Answer:** “Check volume, attenuation and whether the Sound Cue plays in the editor.”
- **Follow-up Questions:** First-play only? Under battle load? Preview works? Dedicated server? Stuck mix?
- **Hands-on Verification Task:** Diagnose nine injected faults using the ordered workflow and record discriminating evidence.
- **Sources:** [SRC-AUDIO-003], [SRC-AUDIO-004], [SRC-AUDIO-006], [SRC-AUDIO-009], [SRC-AUDIO-010]
- **Version Notes:** Debug CVars/tools are version-sensitive.

### Question: How do you profile and optimise a busy audio scene?

- **Category:** Audio / Performance
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Separate request/update CPU, logical/real/virtual voices, source DSP, spatial/occlusion, decode/I/O/cache, per-source effects, submix DSP, audio callback and memory.
- **Strong 3-Year-Engineer Answer:** I stress representative combat plus music/dialogue and count requests, active Components, rejects, steals and virtual/rendered voices. I correlate audio render headroom with graph/effect cost, occlusion queries, decoder/I/O and cache churn. I remove duplicates first, then enforce perceptual concurrency/priority, lower insignificant update/query rates, simplify high-polyphony graphs, share suitable DSP and tune platform cache/voice/effect quality with listening tests.
- **Common Weak Answer:** “Reduce concurrent sounds, compression quality and MetaSound node count.”
- **Follow-up Questions:** Quiet source cost? Source versus Submix effect? Underrun? Memory forms? Low-end scaling?
- **Hands-on Verification Task:** Profile 1/20/100 emitters and 100-impact burst, making one-variable before/after captures.
- **Sources:** [SRC-AUDIO-006], [SRC-AUDIO-007], [SRC-AUDIO-009], [SRC-AUDIO-010]
- **Version Notes:** Counters/CVars and platform budgets vary.

## Niagara and VFX

### Question: Explain Niagara System, Emitter, Module and Parameter responsibilities.

- **Category:** VFX / Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** A System deploys/co-ordinates Emitters; each Emitter owns a particle population; ordered Modules transform typed parameter/attribute data; Parameters carry scoped values.
- **Strong 3-Year-Engineer Answer:** I group Emitters only when they form one effect/lifetime. System then Emitter then Particle groups execute in order, with Renderers consuming resulting attributes. A later Module can overwrite an earlier writer, and namespace/type distinguish variables. User parameters are the external typed contract; System, Emitter and Particle values have different cardinality/lifetime.
- **Common Weak Answer:** “A System contains Emitters, Emitters contain Modules, and Parameters expose controls.”
- **Follow-up Questions:** Stack order? Namespaces? Renderer? Inheritance overrides?
- **Hands-on Verification Task:** Trace every writer of Position/Velocity/Colour in a multi-Emitter impact and reorder one deliberately.
- **Sources:** [SRC-FX-001], [SRC-FX-002]
- **Version Notes:** Architecture stable; maintained overview may include later features.

### Question: What is the difference among Active, Inactive, InactiveClear and Complete?

- **Category:** VFX / Lifecycle
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Active spawns/simulates; Inactive stops spawning but simulates survivors; InactiveClear removes particles then becomes inactive; Complete neither simulates nor renders.
- **Strong 3-Year-Engineer Answer:** I choose the transition from product intent. A stopped trail can become Inactive to let particles finish, while an interrupted telegraph may need clear/fade. Completion is when pool/auto-destroy can reclaim the instance. I never use VFX completion as authoritative gameplay timing because pre-cull/scalability/dedicated server can skip it.
- **Common Weak Answer:** “Inactive pauses it, clear kills it and complete destroys it.”
- **Follow-up Questions:** Auto-destroy? Looping Emitter? Pool release? Deactivate immediate?
- **Hands-on Verification Task:** Log each state and visible particle count for deactivate, deactivate-immediate and natural completion.
- **Sources:** [SRC-FX-002], [SRC-FX-006]
- **Version Notes:** Spawn/Component API details require target verification.

### Question: CPU or GPU Niagara simulation?

- **Category:** VFX / Simulation
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Choose from population, module/Data Interface support, CPU interaction/readback, bounds and target CPU/GPU headroom; GPU moves particle scripts, not all effect work.
- **Strong 3-Year-Engineer Answer:** GPU is attractive for dense parallel cosmetic particles, but System/Emitter/Component and Render Thread work remain, GPU compute competes with graphics, and CPU readback/events add latency/synchronisation. CPU fits bounded particles needing supported CPU data/events. I measure instance Game Thread, particle simulation, GPU compute, submission and pixel cost independently.
- **Common Weak Answer:** “CPU for fewer interactive particles, GPU for thousands because it is faster.”
- **Follow-up Questions:** Which stages move? Data Interface support? Bounds? Readback? Dedicated server?
- **Hands-on Verification Task:** Build equivalent CPU/GPU sparks and compare 1/20/100 instances, readback and bounds.
- **Sources:** [SRC-FX-005], [SRC-FX-003]
- **Version Notes:** Module and Data Interface support is branch/platform-sensitive.

### Question: How should gameplay pass data into Niagara?

- **Category:** VFX / Data Flow
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Through a small typed User-parameter contract, suitable Data Interfaces/collections, or Data Channels for high-volume records; gameplay remains authoritative.
- **Strong 3-Year-Engineer Answer:** For one instance I define User parameter name/type/units/space/default and set spawn-critical values before activation. Shared global wind can use a collection; specialised external data uses a supported Data Interface. Many impact records can feed a UE5.6 Data Channel when its transient schema/query semantics fit. I do not use Niagara data as replicated durable truth.
- **Common Weak Answer:** “Expose User variables and set them by name from Blueprint.”
- **Follow-up Questions:** Live update? Reinitialise? Per-instance versus global? GPU DI? Data Channel persistence?
- **Hands-on Verification Task:** Implement typed impact position/normal/surface/intensity through Components and a Data Channel comparison.
- **Sources:** [SRC-FX-001], [SRC-FX-007]
- **Version Notes:** Data Channel and DI APIs are version-sensitive.

### Question: How do Niagara Component pooling and pre-cull interact with lifetime?

- **Category:** VFX / Lifetime and Pooling
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Pooling reuses Components to avoid allocation/GC; pre-cull can skip an already-insignificant spawn. Neither reduces simulation/render cost or guarantees a returned live Component.
- **Strong 3-Year-Engineer Answer:** I pool frequent equivalent one-shots after measuring allocation pressure and completely reset transform, attachment, parameters, age/random state and callbacks. Auto-destroy/pool release invalidates exclusive pointer assumptions. Pre-cull is safe only for disposable presentation; code tolerates no Component and gameplay never waits on FX completion.
- **Common Weak Answer:** “Enable pooling and pre-cull on all Niagara spawns for performance.”
- **Follow-up Questions:** Pool priming hitch? Persistent Component? Reuse callback? Memory retention?
- **Hands-on Verification Task:** Stress 1000 pooled impacts and assert clean reset/no retained-pointer use after pre-cull or completion.
- **Sources:** [SRC-FX-005], [SRC-FX-006]
- **Version Notes:** Exact pool methods/API overloads require target check.

### Question: Why does a Niagara effect disappear at camera edges or when its origin leaves view?

- **Category:** VFX / Bounds and Culling
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Its System/Emitter bounds may not contain visible particles/ribbon, or scalability/visibility culling may reject the instance.
- **Strong 3-Year-Engineer Answer:** I freeze and draw bounds, inspect Debugger execution/cull reason, then compare peak motion, simulation space and fixed local bounds. I isolate distance, instance, budget, visibility and renderer/material causes. Too-small bounds pop; huge bounds defeat frustum/occlusion culling, so I derive conservative valid bounds and test owner movement/ribbon length/collision deflection.
- **Common Weak Answer:** “Increase fixed bounds until it stops disappearing.”
- **Follow-up Questions:** Local or world fixed box? GPU dynamic bounds? Ribbon? Occlusion? LWC?
- **Hands-on Verification Task:** Reproduce origin-offscreen popping and repair it with measured tight bounds rather than a world-sized box.
- **Sources:** [SRC-FX-003], [SRC-FX-005]
- **Version Notes:** Exact bounds options differ by target/version.

### Question: What is a Niagara Effect Type and how would you use significance?

- **Category:** VFX / Scalability
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Effect Type groups related Systems under shared per-platform scalability, significance, budgets and validation; significance chooses which instances retain quality/existence.
- **Strong 3-Year-Engineer Answer:** I separate Combat, Environment and UI/first-person families because readable priority differs. Significance can combine distance, local ownership, screen relevance and age, with hysteresis to avoid thrash. I preserve core telegraphs while scaling secondary sparks/lights/distortion, configure instance/budget reactions per platform and validate every System has bounds/Effect Type/budget-safe renderers.
- **Common Weak Answer:** “Effect Type is an LOD profile that culls far particles by distance.”
- **Follow-up Questions:** Local-player culling? Global budget? Spawn scaling? Baseline? Validation?
- **Hands-on Verification Task:** Saturate three Effect Types and document cull/degrade order on two quality levels.
- **Sources:** [SRC-FX-004], [SRC-FX-005]
- **Version Notes:** UE5.6 API supports core policy; exact controls evolve.

### Question: Can Niagara collision or events drive gameplay damage?

- **Category:** VFX / Gameplay Boundary
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** No; Niagara collision/events are cosmetic simulation evidence with target/view/readback limitations, while gameplay/physics authority decides damage.
- **Strong 3-Year-Engineer Answer:** CPU/GPU collision paths use different scene representations and costs. GPU depth collision is camera-dependent; readback is delayed; maintained ray-tracing collision is experimental/asynchronous. Gameplay performs authoritative trace/projectile/overlap, then passes impact result to Niagara. I bound particle event fan-out so one collision cannot cascade unlimited Systems/audio/decals.
- **Common Weak Answer:** “CPU particles can trigger damage, but GPU particles are visual only.”
- **Follow-up Questions:** Depth off-screen? Distance fields? Readback latency? Event budget? Server?
- **Hands-on Verification Task:** Compare visual collision paths while proving one gameplay trace remains the sole damage owner.
- **Sources:** [SRC-FX-011], [SRC-PHYS-002]
- **Version Notes:** GPU ray-tracing collision is experimental in maintained later docs.

### Question: Sprite, Mesh, Ribbon, Light or Component Renderer—how do their costs differ?

- **Category:** VFX / Rendering
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Sprites often cost translucent pixels; Meshes geometry/sections; Ribbons tessellation/history/sorting; Lights lighting/shadows; Components per-particle object/Tick overhead.
- **Strong 3-Year-Engineer Answer:** I choose from the visual need and profile the relevant dimension. A few full-screen smoke quads can dominate overdraw; many mesh particles add triangle/material/shadow work; long ribbons add geometry/translucency; particle lights multiply lighting; Component Renderer creates/ticks Components and stays tightly bounded. Material, coverage, overlap, sorting and views can outweigh particle count.
- **Common Weak Answer:** “Sprites are cheapest, then ribbons, meshes, lights and Components.”
- **Follow-up Questions:** Lit translucency? Sort? VR? Sections? Component count?
- **Hands-on Verification Task:** Produce equivalent silhouettes with each viable renderer and capture geometry, submission and GPU pixel cost.
- **Sources:** [SRC-FX-009], [SRC-FX-008]
- **Version Notes:** Maintained UE5.7 renderer/performance pages require target-feature checks.

### Question: How do you diagnose and optimise translucent Niagara overdraw?

- **Category:** VFX / GPU Performance
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Measure screen coverage, overlapping layers and material cost with Shader Complexity/Quad Overdraw plus GPU timings, then reduce coverage/overlap/shader work or scale quality.
- **Strong 3-Year-Engineer Answer:** Particle count alone is misleading. I reproduce the close-camera worst case, inspect Quad Overdraw and GPU passes, then vary spawn/lifetime/size, opacity cut-out, material samples/noise/distortion/lighting, sorting and secondary Emitters one at a time. Effect Type spawn/renderer scaling provides platform policy while preserving the readable core.
- **Common Weak Answer:** “Reduce particle count and use unlit materials.”
- **Follow-up Questions:** Large soft quads? Distortion? VR? Masked alternative? Early-out?
- **Hands-on Verification Task:** Optimise close smoke/explosion with visual acceptance screenshots and one-variable GPU captures.
- **Sources:** [SRC-FX-008], [SRC-RENDER-004]
- **Version Notes:** View modes/tool UI may vary; GPU principle is stable.

### Question: What are lightweight/stateless Niagara Emitters?

- **Category:** VFX / Modern UE5
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** UE5.4+ constrained Emitters designed to reduce memory/CPU/tick and authoring overhead for compatible simple effects.
- **Strong 3-Year-Engineer Answer:** I treat them as a feature/cost trade, not “better GPU Emitters”. A pure stateless System can avoid much traditional stateful simulation overhead, but module/Data Interface/lifetime/renderer requirements may exclude it. I verify exact UE5.4–UE5.6 support, reproduce the look and profile before migrating high-volume simple FX.
- **Common Weak Answer:** “Stateless Emitters are the new faster replacement for normal Niagara Emitters.”
- **Follow-up Questions:** Introduced when? Mixed System? Unsupported feature? Compile/tick difference?
- **Hands-on Verification Task:** Port one compatible burst and one incompatible stateful trail; document feature/performance result.
- **Sources:** [SRC-FX-010]
- **Version Notes:** Introduced UE5.4; maintained page is UE5.7 and must not define target feature set.

### Question: How do you profile a Niagara-heavy scene?

- **Category:** VFX / Profiling
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Separate instance/Game Thread, CPU particles, GPU compute, Render Thread/submission, GPU graphics/overdraw and memory.
- **Strong 3-Year-Engineer Answer:** Niagara Debugger shows counts, states, bounds, memory and cull reasons; FX Outliner captures System/instance Game/Render timing; Insights supplies frame/thread context; GPU capture plus Shader Complexity/Quad Overdraw finds compute/render/pixel cost. I count Systems/Emitters/particles/renderers/lights/Components/events/readbacks and pool churn, then alter lifetime, culling, simulation or rendering one mechanism at a time on packaged target hardware.
- **Common Weak Answer:** “Use stat Niagara, Niagara Debugger and ProfileGPU, then reduce particles/modules.”
- **Follow-up Questions:** GPU readback overhead? Average versus max? Culled count? System versus renderer bound?
- **Hands-on Verification Task:** Profile 1/20/100 combat effects and attribute each optimisation to one cost domain.
- **Sources:** [SRC-FX-003], [SRC-FX-005], [SRC-FX-008]
- **Version Notes:** Debugger performance panels include experimental/version-sensitive portions.

## Lua and C# Integration

### Question: Where do Lua and C# fit in an Unreal project?

- **Category:** Scripting / Adoption
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Through third-party/studio runtimes or adjacent tools/services, not as Epic's baseline gameplay languages; adopt for a concrete iteration, content, modding or tooling need.
- **Strong 3-Year-Engineer Answer:** C++/Blueprint keep native foundations and authority. Lua can support high-level content/gameplay through a pinned plugin; C# often fits editor/build/backend tools and can run in-process only through a specific CLR integration. I score engine/API coverage, lifetime, calls/GC, platforms, package/debug, security, maintenance and exit strategy before adoption.
- **Common Weak Answer:** “Lua is for hotfix gameplay and C# is an easier alternative to C++.”
- **Follow-up Questions:** Why in-process? Platform matrix? Who maintains plugin? Which logic remains native?
- **Hands-on Verification Task:** Write an evidence-based adopt/reject matrix for one Lua and one C# path.
- **Sources:** [SRC-SCRIPT-002], [SRC-SCRIPT-003], [SRC-SCRIPT-005]
- **Version Notes:** Every runtime path is third-party and commit/platform-sensitive.

### Question: Explain Lua tables, metatables and closures in an engine-integration context.

- **Category:** Lua / Language Model
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Tables are associative storage used for arrays/objects/modules; metatables customise lookup/operations; closures capture upvalues and can retain state/native wrappers.
- **Strong 3-Year-Engineer Answer:** I avoid assuming sequence length/order for holed tables and use explicit modules. `__index` can implement prototype lookup but may hide native calls. Closures stored in delegates/timers can retain whole tables and wrappers after owner death, so registrations have explicit handles/unbind and runtime generation. Only nil/false are false; assigning nil removes a key.
- **Common Weak Answer:** “Tables are dictionaries, metatables are classes, and closures are lambdas.”
- **Follow-up Questions:** `__index`? Truthiness? `require` cache? Cross-runtime cycle?
- **Hands-on Verification Task:** Create and diagnose a closure/delegate cycle retaining a native wrapper across travel.
- **Sources:** [SRC-SCRIPT-001]
- **Version Notes:** Pin plugin Lua/LuaJIT version; this answer uses Lua 5.4 language reference.

### Question: What should an Unreal engineer know about the Lua C API stack?

- **Category:** Lua / Native Interop
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Native functions read/push typed values on a virtual stack; indices, stack balance, borrowed pointers, protected errors and registry references require strict lifetime discipline.
- **Strong 3-Year-Engineer Answer:** I validate args, use stable/absolute indices as the stack changes, restore/check top on all paths, copy or retain strings only within documented lifetime, and use/release registry refs for Lua values held natively. Calls enter through protected gateways with traceback. Yielding through C requires continuation-aware APIs; a raw pointer in light userdata is not safe ownership.
- **Common Weak Answer:** “Push arguments, call the function, then pop the result.”
- **Follow-up Questions:** Negative index? Registry? `pcall`? Yield across C? Full versus light userdata?
- **Hands-on Verification Task:** Implement a tiny protected C binding with stack-top assertions and injected type/error/yield cases.
- **Sources:** [SRC-SCRIPT-001]
- **Version Notes:** Exact plugin abstraction may hide the C API but inherits its VM rules.

### Question: Lua GC versus Unreal GC—what can go wrong?

- **Category:** Lua / Lifetime
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** They trace different heaps, so wrappers can outlive destroyed UObjects, bindings can retain UObjects, and callbacks can form cycles neither product lifecycle breaks promptly.
- **Strong 3-Year-Engineer Answer:** Native ownership is independent of Lua reachability. The binding invalidates handles on UObject destruction; every call validates handle/World/thread. Explicit shutdown unbinds delegates/timers/coroutines/registry refs; `__gc` is fallback only. I test travel/reload and forced GC on both sides, logging wrapper/runtime generation and native identity.
- **Common Weak Answer:** “Use weak tables and `IsValid`; both garbage collectors will eventually clean it.”
- **Follow-up Questions:** Registry root? Delegate closure? AddToRoot? Finaliser timing? VM shutdown order?
- **Hands-on Verification Task:** Destroy a wrapped UObject under delegate/coroutine/registry references and prove safe invalidation/no retention.
- **Sources:** [SRC-SCRIPT-001], [SRC-SCRIPT-002]
- **Version Notes:** Binding implementation is plugin-specific.

### Question: Is a Lua coroutine asynchronous or multithreaded?

- **Category:** Lua / Async
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** It is cooperatively suspended execution, not an OS thread; UObject access is legal only on the host thread that resumes it under the engine's rules.
- **Strong 3-Year-Engineer Answer:** I integrate coroutines with a scheduler that records owner/World/runtime generation and cancellation. Resume occurs on Game Thread for gameplay. A coroutine cannot yield through every C call, and reload/travel must close/cancel suspended work. Worker computation uses copied immutable data and posts results back.
- **Common Weak Answer:** “Coroutines are lightweight threads for non-blocking gameplay.”
- **Follow-up Questions:** Yield across C? Cancellation? Error propagation? Reload?
- **Hands-on Verification Task:** Delay a coroutine across owner destruction/travel and reject the stale resume safely.
- **Sources:** [SRC-SCRIPT-001]
- **Version Notes:** Plugin scheduler/yield wrappers vary.

### Question: How would you design a safe script-facing Unreal API?

- **Category:** Scripting / API Design
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Expose coarse semantic commands, snapshots and events with stable IDs, typed units, validation and explicit lifetime/thread/error contracts.
- **Strong 3-Year-Engineer Answer:** Native code owns invariants/authority and presents a narrow façade rather than raw mutable Components. I batch state, correlate async requests, invalidate handles, marshal callbacks to Game Thread and protect errors/exceptions. Reflected baseline may be augmented with generated hot-path and manual semantic bindings. Versioned schemas make reload/network/package behaviour explicit.
- **Common Weak Answer:** “Expose UFUNCTIONs/UPROPERTYs and keep performance-critical functions in C++.”
- **Follow-up Questions:** Call granularity? Stable IDs? Reflection gap? Async cancellation? Security allow-list?
- **Hands-on Verification Task:** Specify an inventory/objective façade and reject five unsafe raw-engine methods.
- **Sources:** [SRC-SCRIPT-002], [SRC-SCRIPT-004], [SRC-SCRIPT-007]
- **Version Notes:** Binding capability is plugin-specific.

### Question: How do .NET GC and native Unreal object lifetime interact?

- **Category:** C# / Lifetime
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Managed GC moves/reclaims managed objects; Unreal owns UObjects. Handles/wrappers must root callbacks, observe native invalidation and release registrations deterministically.
- **Strong 3-Year-Engineer Answer:** A managed wrapper stores a binding handle, never ownership by raw pointer. Native destruction invalidates it. Managed delegates retained natively are rooted until unregister, and GCHandles are freed. `IDisposable`/SafeHandle-style teardown releases native registrations; finalisation is fallback. Pinning is short because it impairs moving GC. I force both GCs in lifecycle tests.
- **Common Weak Answer:** “Use weak references to UObjects and IDisposable for deterministic GC.”
- **Follow-up Questions:** GCHandle? Pinning? SafeHandle? Finalizer? Cross-runtime cycle?
- **Hands-on Verification Task:** Implement/test wrapper invalidation and callback unregistration across forced managed/UE GC.
- **Sources:** [SRC-SCRIPT-006], [SRC-SCRIPT-007], [SRC-SCRIPT-008]
- **Version Notes:** Plugin wrapper semantics may differ; principles are runtime/native interop facts.

### Question: What are the main C# native-marshalling traps?

- **Category:** C# / Native Interop
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** ABI layout, bool/char/string encoding, enum width, arrays/ownership, delegate lifetime/calling convention, pinning and non-blittable copies.
- **Strong 3-Year-Engineer Answer:** I define fixed layout/packing and prefer small blittable records. C# bool does not automatically equal C++ bool ABI. Strings specify encoding and allocator/owner; arrays pass pointer/count and lifetime. A retained callback uses an exact signature and rooted delegate/handle, catches exceptions and unregisters before freeing. I benchmark empty, marshalled and batched calls.
- **Common Weak Answer:** “P/Invoke handles conversion automatically; use IntPtr for Unreal pointers.”
- **Follow-up Questions:** Blittable? Struct alignment? FString/FText? Delegate GC? Call frequency?
- **Hands-on Verification Task:** Round-trip a struct with an intentionally wrong bool/layout/string and repair from byte-level evidence.
- **Sources:** [SRC-SCRIPT-007]
- **Version Notes:** ABI is compiler/platform/plugin specific.

### Question: What threading trap does C# `async`/`Task` create in Unreal?

- **Category:** C# / Async
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Continuations may run on thread-pool threads; managed async does not grant thread-safe UObject access.
- **Strong 3-Year-Engineer Answer:** Worker Tasks operate only on copied/immutable data. Results post through an explicit Game Thread dispatcher and revalidate owner, World and runtime generation. I propagate cancellation on travel/reload/shutdown and never synchronously block Game Thread on a Task waiting to return there. Exceptions are observed/caught at boundary.
- **Common Weak Answer:** “Use `await` so the engine thread is not blocked.”
- **Follow-up Questions:** SynchronisationContext? Cancellation? Late callback? Deadlock? Exception?
- **Hands-on Verification Task:** Run an off-thread calculation, then inject travel and prove stale result cancellation/Game Thread application.
- **Sources:** [SRC-SCRIPT-005], [SRC-SCRIPT-006]
- **Version Notes:** A plugin may provide its own dispatcher/context; verify rather than assume.

### Question: Why does Native AOT not automatically make C# safe to ship on every Unreal platform?

- **Category:** C# / Packaging
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** AOT is platform-specific and restricts dynamic loading, runtime codegen and reflection/trimming patterns; the Unreal plugin still needs native/runtime integration and platform approval.
- **Strong 3-Year-Engineer Answer:** I start from the integration's supported platform matrix. Native AOT removes JIT but current .NET docs list no dynamic assembly loading/Reflection.Emit, trimming and diagnostics constraints. Those can conflict with reflection-generated bindings/hot reload. I build per architecture, inspect warnings, stage native/managed dependencies/symbols and test clean packaged/server builds under store/signing policy.
- **Common Weak Answer:** “Use IL2CPP/Native AOT for consoles and mobile.”
- **Follow-up Questions:** Trimming? Reflection? Code signing? Runtimeconfig? Debug symbols?
- **Hands-on Verification Task:** Produce a deployment matrix showing which integration features fail under a hypothetical AOT-only target.
- **Sources:** [SRC-SCRIPT-009], [SRC-SCRIPT-010]
- **Version Notes:** .NET AOT capability is not evidence that a specific Unreal plugin supports it.

### Question: What does safe script or assembly hot reload require?

- **Category:** Scripting / Reload
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** A transaction: quiesce, cancel/unbind, serialise versioned state, invalidate old handles/code, load/validate, migrate/rebind, resume or rollback.
- **Strong 3-Year-Engineer Answer:** Reloading a Lua module does not update old closures/instances; managed assemblies/load contexts may retain callbacks/types. I stamp runtime generations, reject old callbacks, persist stable IDs rather than wrappers and test schema migration/failure rollback. Multiplayer authoritative changes require coordinated protocol/content versions and platform policy; “hotfix” is not just replacing files.
- **Common Weak Answer:** “Clear `package.loaded` or reload the DLL, then recreate affected objects.”
- **Follow-up Questions:** Old delegates? Coroutine/Task? State schema? Rollback? Client/server version?
- **Hands-on Verification Task:** Reload with a state-schema change and an old delayed callback; migrate/reject deterministically.
- **Sources:** [SRC-SCRIPT-001], [SRC-SCRIPT-005]
- **Version Notes:** Exact reload support differs sharply by plugin/runtime.

### Question: How do you profile and debug a mixed native/script runtime?

- **Category:** Scripting / Debugging and Performance
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Correlate native and script/managed stacks by boundary call/runtime generation while measuring call/marshalling/allocation/GC/wrapper/root/reload/package costs.
- **Strong 3-Year-Engineer Answer:** Each boundary call logs ID, direction, façade method, owner/World/runtime generation and timing. A crash report includes native stack plus Lua traceback or managed exception/stack. I measure empty/value/batched calls, converted bytes/allocations, Lua/.NET GC, wrappers/handles/delegates and UObjects retained by bindings. I compare editor and packaged/server and optimise boundary architecture before VM micro-tuning.
- **Common Weak Answer:** “Use the Lua/Visual Studio debugger and Unreal Insights; move slow code to C++.”
- **Follow-up Questions:** Cross-runtime leak? Forced GC? Package-only symbol issue? Reflection cache? Call batching?
- **Hands-on Verification Task:** Diagnose stale handle, wrong thread, ABI mismatch and old-generation callback with mixed evidence.
- **Sources:** [SRC-SCRIPT-001], [SRC-SCRIPT-005], [SRC-SCRIPT-006], [SRC-SCRIPT-007]
- **Version Notes:** Debugger/profiler integration is plugin/platform-sensitive.

### Question: How do you choose UPROPERTY specifiers without cargo-culting `EditAnywhere BlueprintReadWrite`?

- **Category:** UE C++ / Property Specifiers
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Name the consuming system and intended lifecycle: editor authoring, Blueprint access, serialisation, config, save, duplication, replication or GC reference discovery.
- **Strong 3-Year-Engineer Answer:** I start from the state contract. Defaults tuned by designers are often `EditDefaultsOnly`; runtime state is usually mutated through functions and may only be visible or not exposed at all; Blueprint write access is separate from editor editability. `Config`, `SaveGame`, `Transient`, `DuplicateTransient`, `Instanced` and replication flags are not general "persistence" labels - each is consumed by a specific engine path. I avoid writable public state when it bypasses authority, validation or invariants.
- **Common Weak Answer:** “Use `UPROPERTY` for editor/Blueprint and GC, then add whatever specifiers are needed.”
- **Follow-up Questions:** `EditDefaultsOnly` versus `EditInstanceOnly`? `BlueprintReadOnly` versus `VisibleAnywhere`? What consumes `SaveGame`? When does `Instanced` matter?
- **Hands-on Verification Task:** Build one Actor with tuning, runtime, cache, saved and replicated fields; predict and observe editor visibility, Blueprint access, duplication, save and replication effects.
- **Sources:** [SRC-EPIC-007], [SRC-EPIC-031], [SRC-EPIC-035]
- **Version Notes:** UE5.3-UE5.6 target; rare specifiers should be compiled in the target branch.

### Question: What is the difference between specifiers and metadata?

- **Category:** UE C++ / Reflection Metadata
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Specifiers usually express engine/reflection flags or generated behaviour, while metadata primarily guides editor and Blueprint node/tooling presentation.
- **Strong 3-Year-Engineer Answer:** I treat metadata as authoring/node information unless documentation for a subsystem says otherwise. `DisplayName`, `ToolTip`, category metadata and world-context metadata improve the editor or generated Blueprint node shape; they are not game authority, security, replication or save policy. If a runtime rule matters, enforce it in code, validation, data schema or server authority, and use metadata only to make the authoring surface clearer.
- **Common Weak Answer:** “Metadata is just extra specifiers in `meta=(...)`.”
- **Follow-up Questions:** Why not depend on metadata in Shipping logic? What does `WorldContext` change? What should validation use instead?
- **Hands-on Verification Task:** Add metadata that changes Blueprint node display and prove the native validation path still enforces the rule without reading metadata.
- **Sources:** [SRC-EPIC-032], [SRC-EPIC-036]
- **Version Notes:** Metadata keys and node behaviour are version-sensitive; branch-check unusual keys.

### Question: How would you explain `Config`, `SaveGame`, `Transient`, and `DuplicateTransient`?

- **Category:** UE C++ / Persistence
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** They describe different persistence and copy paths: config `.ini`, save-game archive selection, normal serialisation exclusion, and reset-on-duplication behaviour.
- **Strong 3-Year-Engineer Answer:** I do not call them all "saved state". `Config` needs class-level config setup and load/save policy; it is good for settings/defaults, not authoritative match state. `SaveGame` is a flag for archives that honour it, so the save system still needs stable IDs, schema versioning and reference policy. `Transient` prevents normal persistence for temporary/cache data. `DuplicateTransient` is useful when copy/PIE/editor duplication should reset runtime-only values. I test editor duplication, PIE, clean package and load migration.
- **Common Weak Answer:** “Config is for settings, SaveGame is for saves, Transient is not saved.”
- **Follow-up Questions:** What happens with a live Actor pointer? How do defaults interact with config? Why can PIE duplication expose bugs?
- **Hands-on Verification Task:** Create five fields with these flags, duplicate the Actor, save/load through a custom archive and log which fields survive each path.
- **Sources:** [SRC-EPIC-032], [SRC-EPIC-035]
- **Version Notes:** Archive/save behaviour is project-specific even when flags are stable.

### Question: When should you use a reflected `UINTERFACE` instead of a native abstract C++ interface?

- **Category:** UE C++ / Interfaces
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Use `UINTERFACE` when UObject/reflection/Blueprint participation is required; use native abstract C++ interfaces for pure C++ boundaries that do not need reflection.
- **Strong 3-Year-Engineer Answer:** A reflected interface has a UObject-facing `UINTERFACE` type and an `I...` implementation type. It is useful for cross-UObject capabilities, Blueprint implementation or Blueprint calls. It does not store state, replicate calls or solve ownership. For reusable state/behaviour I consider Components. For pure internal library polymorphism, a native abstract interface is simpler and cheaper. If Blueprint implementers are possible, I call through the generated/interface execution path rather than assuming a native virtual call covers every implementation.
- **Common Weak Answer:** “Use interfaces to avoid casts.”
- **Follow-up Questions:** Interface versus Component? Blueprint implementer? Network authority? Where is state stored?
- **Hands-on Verification Task:** Implement one native class and one Blueprint class through the same interface, then call both safely from C++ and Blueprint.
- **Sources:** [SRC-EPIC-037], [SRC-EPIC-031]
- **Version Notes:** Interface syntax is stable, but generated helper details should be compiled in the target branch.

### Question: What does `Outer` imply, and what does it not imply?

- **Category:** UObject / Identity
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** `Outer` provides containment and identity/path context; it is not by itself a normal C++ ownership or GC-retention guarantee.
- **Strong 3-Year-Engineer Answer:** I choose Outer to place an object in the right identity/package/context relationship. It affects names and paths and some APIs require a suitable Outer. If the child must be retained, I still need a reflected strong reference, `TStrongObjectPtr`, `FGCObject` reporting or another explicit retention mechanism. `Within=...` can constrain Outer class, but it also does not replace lifetime design. I log name, class, outer, package and flags when debugging identity bugs.
- **Common Weak Answer:** “The Outer owns the UObject.”
- **Follow-up Questions:** What creates the GC edge? What about package paths? What about `Within`? What changes in PIE?
- **Hands-on Verification Task:** Create two UObjects with the same Outer but only one retained by a reflected property; force GC and explain the result.
- **Sources:** [SRC-EPIC-004], [SRC-EPIC-032], [SRC-EPIC-043]
- **Version Notes:** Stable concept; exact flags/path formatting can vary across editor/package contexts.

### Question: How do `FindObject`, `LoadObject`, `NewObject`, `DuplicateObject`, and `SpawnActor` differ?

- **Category:** UE C++ / Object APIs
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** They express different intents: create UObject, create Actor in a World, find an already-loaded object, synchronously load an object, or duplicate an object graph.
- **Strong 3-Year-Engineer Answer:** `NewObject` is for UObjects, with Outer/name/flags/template and an explicit retention policy. Actors go through `SpawnActor` or deferred spawn because World registration, construction, collision and Actor lifecycle matter. Lookup is not loading; `FindObject` style APIs should not be used as asset streaming policy. `LoadObject` blocks and needs cook/path correctness. `DuplicateObject` copies an object graph subject to flags/instancing/duplication rules, so transient caches and source references must be reviewed.
- **Common Weak Answer:** “Use `NewObject` for UObjects, `SpawnActor` for Actors, and `LoadObject` when you need an asset.”
- **Follow-up Questions:** Deferred spawn? Cook failure? Duplicate transient? Same name/Outer? Retention?
- **Hands-on Verification Task:** Create, find, synchronously load and duplicate a small UObject/subobject graph; record path, flags, Outer and which references point to source versus duplicate.
- **Sources:** [SRC-EPIC-003], [SRC-EPIC-004], [SRC-ASSET-001], [SRC-ASSET-002]
- **Version Notes:** Exact overloads and duplicate semantics require target header/source confirmation.

### Question: How do you choose between `UE_LOG`, `UE_LOGFMT`, `check`, `verify`, and `ensure`?

- **Category:** UE C++ / Diagnostics
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Logs record diagnostic events; checks assert impossible programmer invariants; verify preserves expression execution; ensure reports unexpected survivable state.
- **Strong 3-Year-Engineer Answer:** I create subsystem log categories and use verbosity filters rather than `LogTemp` spam. `UE_LOGFMT` is good for structured fields and correlation IDs. `check` is for states where continuing is unsafe; normal content/network/user failures need return paths. `verify` is for expressions whose side effects must still execute when check-style assertions are disabled. `ensure` captures unexpected but survivable conditions, usually with a normal fallback and enough context to reproduce.
- **Common Weak Answer:** “Use `check` in debug and `ensure` if you do not want a crash.”
- **Follow-up Questions:** What happens in Shipping? Why not use check for validation? What fields belong in a useful log?
- **Hands-on Verification Task:** Implement a validated inventory move with a normal rejection path, an ensure for broken owner state and a check for an impossible normalised-request invariant.
- **Sources:** [SRC-EPIC-038], [SRC-EPIC-039]
- **Version Notes:** Build configuration and platform reporting settings affect assertion behaviour.

### Question: What is the danger of storing `TFunctionRef` or a view type for later?

- **Category:** UE C++ / Core Vocabulary
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** They are non-owning views over a callable or storage; storing them beyond the call can dangle.
- **Strong 3-Year-Engineer Answer:** I use `TFunctionRef` for synchronous algorithms that invoke a callback before returning. If the callback escapes, I switch to an owning callable or delegate with explicit lifetime and weak captures. Array/span-style views are similar: they avoid copying but depend on the underlying storage staying alive and unmutated for the view's use. They are excellent API vocabulary when the lifetime is obvious, and dangerous in timers, tasks, async proxies and cached members.
- **Common Weak Answer:** “`TFunctionRef` is a faster delegate.”
- **Follow-up Questions:** `TFunctionRef` versus `TFunction`? View to temporary? Worker thread? Weak capture?
- **Hands-on Verification Task:** Write one synchronous filter using a function ref, then deliberately store the function ref in a delayed callback and catch the lifetime bug in review/tests.
- **Sources:** [SRC-EPIC-042], [SRC-CPP-018], [SRC-CPP-025]
- **Version Notes:** Exact UE Core type names/APIs require target header confirmation.

### Question: When is `TOptional` better than a sentinel value?

- **Category:** UE C++ / Core Vocabulary
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** When absence is a valid separate state and no normal value should be overloaded to mean "not present".
- **Strong 3-Year-Engineer Answer:** I use optional vocabulary for query results, parsed values or optional configuration where default value and missing value have different semantics. A sentinel like `-1`, zero vector or empty name can collide with real data or hide validation errors. I avoid using optional as an ownership model for UObjects; object lifetime still needs weak/strong/soft/reference semantics. When serialising or exposing to Blueprint, I check whether the optional type itself is supported or map it to explicit fields.
- **Common Weak Answer:** “Use optional when a value can be null.”
- **Follow-up Questions:** Optional versus pointer? Blueprint exposure? Default value? Saved schema?
- **Hands-on Verification Task:** Replace a `-1` "not found" index with an optional result and update all callers to handle absence explicitly.
- **Sources:** [SRC-EPIC-040], [SRC-CPP-025]
- **Version Notes:** UE API/reflection support is branch-sensitive; pin headers before exposing optional fields.

### Question: What is a safe design for an async Blueprint node?

- **Category:** Blueprint / Async
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** A proxy UObject with explicit in-flight retention, activation, cancellation, Game Thread completion, stale-generation checks and delegate cleanup.
- **Strong 3-Year-Engineer Answer:** I define who retains the proxy, which World/player/owner it belongs to, and what cancels it. Native callbacks store handles and clear them exactly once. Results from tasks, network or async loading are marshalled to the Game Thread before touching UObjects or broadcasting Blueprint delegates. A request generation rejects stale completions after travel, destroy or manual cancel. Success and failure paths both release references and broadcast at most once.
- **Common Weak Answer:** “Create a `UBlueprintAsyncActionBase`, expose success/failure delegates, and call them when the async work finishes.”
- **Follow-up Questions:** What if the proxy is GC'd? What if the owner is destroyed? What thread completes? What about duplicate activation?
- **Hands-on Verification Task:** Implement a proxy around a timer or async load and test owner destroy, travel, manual cancel, failure, success and success-after-cancel.
- **Sources:** [SRC-EPIC-036], [SRC-EPIC-044], [SRC-EPIC-028]
- **Version Notes:** Async proxy APIs and latent/Blueprint integration details are target-version sensitive.

### Question: When would you use `FGCObject`?

- **Category:** UObject / GC Bridge
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** When a non-UObject native owner must report UObject references to the garbage collector and cannot reasonably store reflected UObject properties.
- **Strong 3-Year-Engineer Answer:** My default is a UObject owner with `UPROPERTY` references. I consider `FGCObject` for native managers, Slate/tool objects or helper caches that have a clear native lifetime and must retain UObjects. The implementation documents what it reports, when references change, how teardown happens and how PIE/travel is handled. I avoid it for casual gameplay because missed references collect too early and hidden strong references leak or retain whole graphs.
- **Common Weak Answer:** “Use `FGCObject` if a normal C++ class needs UObjects.”
- **Follow-up Questions:** Why not `UPROPERTY`? Why not weak? What about `AddToRoot`? How do you find leaks?
- **Hands-on Verification Task:** Add an `FGCObject` native helper to Project 7A and prove both retained and released cases under forced GC and travel.
- **Sources:** [SRC-EPIC-043], [SRC-EPIC-005], [SRC-EPIC-009]
- **Version Notes:** Specialist API; verify branch signatures and collector behaviour.

### Question: How should off-thread work interact with UObjects?

- **Category:** UE C++ / Thread Boundaries
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Snapshot plain data on the Game Thread, process off-thread without UObject access, then apply results back on the Game Thread after weak/generation validation.
- **Strong 3-Year-Engineer Answer:** I assume gameplay UObjects are Game Thread owned unless a subsystem explicitly documents otherwise. Worker tasks receive copied immutable data, IDs or pure value arrays. They do not dereference Actors, mutate Components or broadcast Blueprint delegates. Completion posts to the Game Thread, resolves weak owners, checks World/request generation/cancellation and then applies state. Releasing a task handle is not cancellation; cancellation is a protocol I design.
- **Common Weak Answer:** “Use `AsyncTask` or Tasks and avoid heavy work on the Game Thread.”
- **Follow-up Questions:** Raw `this` capture? World travel? Shared arrays? Task cancellation? Thread assertions?
- **Hands-on Verification Task:** Build a snapshot-partition-merge targeting task and inject owner destruction/travel before completion.
- **Sources:** [SRC-CPP-026], [SRC-CPP-025], [SRC-EPIC-028]
- **Version Notes:** UE task APIs and thread checks are version/toolchain sensitive.

### Question: How do you debug a generated-code or specifier problem?

- **Category:** UE C++ / UHT Debugging
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Read the first real UHT/compiler error, inspect the reflected declaration before the macro, reduce the declaration, then identify which system consumes the specifier.
- **Strong 3-Year-Engineer Answer:** I separate UHT parse errors, C++ compile errors, linker/export errors and Blueprint compiler errors. Generated-file cascades usually point back to an unsupported type, missing include, wrong generated header order, bad prefix, missing module dependency or invalid specifier combination. If it compiles but behaviour is wrong, I ask which consumer failed: editor details, Blueprint node, config/save archive, duplication, replication, GC, cook or package. Then I build a minimal target-branch example.
- **Common Weak Answer:** “Clean intermediates and regenerate project files.”
- **Follow-up Questions:** Generated header order? Public dependency? API macro? Blueprint compiler error? Dynamic metadata?
- **Hands-on Verification Task:** Create five intentional UHT/specifier failures and classify each as UHT, C++ compile, link, Blueprint compile or runtime-system mismatch.
- **Sources:** [SRC-EPIC-001], [SRC-EPIC-002], [SRC-EPIC-032], [SRC-CPP-025], [SRC-BUILD-002]
- **Version Notes:** Exact diagnostics and generated code differ by engine version and build mode.

### Question: How does Actor Component replication differ from Actor replication?

- **Category:** Networking / Components
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Components replicate through their owning replicated Actor and must be explicitly set to replicate; their properties, RPCs and subobjects have their own contracts and overhead.
- **Strong 3-Year-Engineer Answer:** I first make sure the owning Actor replicates, then enable component replication through constructor/default settings or runtime calls depending on whether the component is static or dynamic. Component properties replicate like Actor properties through the component's lifetime props. Component RPCs follow normal RPC authority/ownership ideas but can have more overhead than routing through the Actor, so I choose the surface deliberately. Component subobjects only matter after the owning Actor and component replicate to that connection.
- **Common Weak Answer:** “Components replicate automatically if the Actor replicates.”
- **Follow-up Questions:** Static versus dynamic component? Component RPC overhead? Subobject condition? Bandwidth cost?
- **Hands-on Verification Task:** Add a static and dynamic replicated component to Project 3 and prove property/RPC/component creation behaviour on two remote clients.
- **Sources:** [SRC-NET-011], [SRC-NET-004], [SRC-NET-006]
- **Version Notes:** Exact component replication APIs and overhead details require target branch verification.

### Question: When would you use a replicated UObject subobject?

- **Category:** Networking / Subobjects
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** When nested state belongs to an owning Actor or Component, needs replicated fields, but does not deserve full Actor identity.
- **Strong 3-Year-Engineer Answer:** I use a replicated UObject subobject for authoritative nested state with clear ownership through an Actor or Actor Component. The object supports networking, registers its replicated fields, and is registered through the target replication path. Default subobjects and dynamic subobjects have different mapping behaviour: dynamic references can be null on clients until the subobject is created/mapped. I remove registered subobjects before deletion/GC and test join-in-progress, dormancy, owner change and teardown.
- **Common Weak Answer:** “Use UObjects instead of Actors because they are lighter.”
- **Follow-up Questions:** Default versus dynamic subobject? Registered list versus `ReplicateSubobjects`? Iris? GC crash?
- **Hands-on Verification Task:** Add a dynamic replicated UObject subobject with one replicated field and pointer OnRep; force GC after removal.
- **Sources:** [SRC-NET-012], [SRC-EPIC-004], [SRC-EPIC-009]
- **Version Notes:** The Object Replication source is maintained beyond the target docs; confirm exact API in UE5.3-UE5.6.

### Question: Why can a replicated subobject pointer be null on a client?

- **Category:** Networking / Subobject Mapping
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** A dynamic subobject created on the server must be replicated and mapped before the client reference becomes valid.
- **Strong 3-Year-Engineer Answer:** A replicated pointer to a dynamic subobject is not enough by itself. The server creates the object and registers or replicates it as a subobject; until the client receives the subobject data and maps the network reference, the pointer can remain null or unresolved. Client code must tolerate that window, react through OnRep or validity checks, and avoid using the pointer as if it was constructed locally. I inspect NetGUID/unmapped reference logs when debugging this.
- **Common Weak Answer:** “Replication is delayed; just add a delay node.”
- **Follow-up Questions:** Default subobject? Stably named? OnRep? Unmapped RPC parameter? Async loading?
- **Hands-on Verification Task:** Log pointer validity before and after subobject mapping under packet lag and join-in-progress.
- **Sources:** [SRC-NET-012], [SRC-NET-016], [SRC-NET-005]
- **Version Notes:** Exact mapping/debug CVars vary by target branch.

### Question: What is the registered subobjects list and why does it matter?

- **Category:** Networking / Subobjects
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** It is a maintained list of subobjects to replicate from an Actor or Component, reducing per-subobject virtual calls and required for Iris subobject replication in the maintained docs.
- **Strong 3-Year-Engineer Answer:** Instead of overriding `ReplicateSubobjects` and manually calling through the Actor channel each update, the owner maintains a registered list using target-branch add/remove APIs. The list must be kept in sync with object lifetime; if a subobject is deleted or collected while still registered, a raw stale pointer can crash later. It also changes debugging and push-model expectations. I use it for modern subobject designs, especially if Iris is in scope, but verify exact branch APIs.
- **Common Weak Answer:** “It is just a cleaner way to replicate subobjects.”
- **Follow-up Questions:** Legacy path? Iris? Removal before GC? Client-side lists/replays? Push model?
- **Hands-on Verification Task:** Implement registered-list subobject replication, then deliberately skip removal before GC in a throwaway branch and capture the failure.
- **Sources:** [SRC-NET-012], [SRC-NET-014], [SRC-NET-016]
- **Version Notes:** Branch-sensitive; do not copy maintained-page code into UE5.3-UE5.6 without source confirmation.

### Question: When should you use `FFastArraySerializer`?

- **Category:** Networking / Fast Array
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** When an authoritative replicated collection has stable item identity and benefits from item-level add/change/remove deltas.
- **Strong 3-Year-Engineer Answer:** I consider Fast Array for inventories, status lists or replicated item collections where full-array replication is wasteful and item changes matter. I design stable item IDs, centralise server mutation paths, mark item/array dirty correctly, and make client callbacks idempotent. I test join-in-progress, removal, reorder, reused IDs, packet loss and coalesced changes. If the array is tiny or rarely changes, simple replication can be better.
- **Common Weak Answer:** “Fast Array is always faster for replicated arrays.”
- **Follow-up Questions:** Dirty marking? Push model? Stable ID? Reorder? Removal callback?
- **Hands-on Verification Task:** Compare a 50-item inventory using whole-array replication and Fast Array under add/remove/change workloads.
- **Sources:** [SRC-NET-015], [SRC-NET-016], [SRC-NET-004]
- **Version Notes:** API details, callbacks and dirty marking require target header/source verification.

### Question: What problem does Replication Graph solve?

- **Category:** Networking / Replication Graph
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It reduces server CPU spent building per-client replication lists for large Actor/player counts by using persistent graph nodes and shared data.
- **Strong 3-Year-Engineer Answer:** Generic replication can become expensive when many Actors consider many connections. Replication Graph lets the project build persistent nodes such as spatial grids, always-relevant lists, team/owner lists or dormant/static buckets, then produce per-connection Actor lists from shared cached data. I only adopt it after profiling shows replication list construction/relevancy is the bottleneck; otherwise normal relevancy, update rates and dormancy are simpler.
- **Common Weak Answer:** “Replication Graph makes networking faster.”
- **Follow-up Questions:** Spatial node? Always relevant node? Splitscreen caveat? Stale buckets? Profiling proof?
- **Hands-on Verification Task:** Classify Project 3 Actors into candidate graph buckets and write a rejection/adoption memo based on measured scale.
- **Sources:** [SRC-NET-013], [SRC-NET-010], [SRC-PERF-003]
- **Version Notes:** Plugin/platform limitations and exact node APIs require target branch confirmation.

### Question: How should you talk about Iris in a UE5 interview?

- **Category:** Networking / Iris
- **Priority:** P2
- **Expected Depth:** D2
- **Short Answer:** Iris is an opt-in newer replication system, experimental in the docs, and not automatically used by every UE5 project.
- **Strong 3-Year-Engineer Answer:** I would say I know Iris exists and is motivated by larger worlds, higher player counts and lower server costs, but I would not claim it is the baseline unless project config/source shows it is enabled. I would verify `net.Iris.UseIrisReplication`, registered subobject requirements, fragment registration for custom UObjects and tooling/CVar support in the target branch. For a general gameplay role, strong generic replication fundamentals matter first.
- **Common Weak Answer:** “UE5 uses Iris now.”
- **Follow-up Questions:** Experimental? Registered subobjects? Fragment registration? Migration risk? Generic versus Iris?
- **Hands-on Verification Task:** Inspect a target project config/source and produce a one-page Iris enabled/not-enabled evidence note.
- **Sources:** [SRC-NET-014], [SRC-NET-012], [SRC-NET-001]
- **Version Notes:** Iris status/API is highly version and project-config sensitive.

### Question: What does push-model replication change?

- **Category:** Networking / Push Model
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It lets game code report dirtiness instead of relying only on polling, but it does not change authority, relevance or replication protocol design.
- **Strong 3-Year-Engineer Answer:** Push model can reduce unnecessary work when mutation paths mark dirty correctly. The risk is missing a dirty mark and believing state replicated when networking safely skipped it. I use validation CVars/tests in development, keep server authority unchanged, and still design relevancy, dormancy, conditions and late-join state normally. It is a performance/reporting tool, not a gameplay ownership model.
- **Common Weak Answer:** “Push model makes replication instant.”
- **Follow-up Questions:** Dirty mark? Fast Array? Dormancy? Validation CVar? Client mutation?
- **Hands-on Verification Task:** Enable target-branch push-model validation if supported and intentionally miss a dirty mark.
- **Sources:** [SRC-NET-016], [SRC-NET-004], [SRC-NET-008]
- **Version Notes:** CVar names and push model availability differ by branch/config.

### Question: How do network debug CVars fit into a diagnosis workflow?

- **Category:** Networking / Debugging
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** They are targeted evidence tools for dormancy, NetGUIDs, push model, subobjects, packet behaviour and validation, not production fixes.
- **Strong 3-Year-Engineer Answer:** I start with a hypothesis from logs/profiler and then use the narrowest target-branch CVar or command to confirm it. For example, dormancy validation for lost updates, PackageMap/NetGUID logs for unmapped references, push-model validation for missed dirty marks, and subobject comparison/debug flags for registered-list issues. I keep captures short because logging can alter timing and memory, and I never rely on a maintained-doc command until I confirm it exists in the target build.
- **Common Weak Answer:** “Turn on net debug commands until you see something.”
- **Follow-up Questions:** Shipping availability? Log volume? NetGUID? Dormancy? Packet lag/loss?
- **Hands-on Verification Task:** For one Project 3 bug, record the before/after with exactly one diagnostic CVar and explain why that CVar was chosen.
- **Sources:** [SRC-NET-016], [SRC-NET-010], [SRC-PERF-003]
- **Version Notes:** Maintained CVar pages can exceed UE5.3-UE5.6; query the target runtime.

### Question: How would you replicate a modular inventory with high item churn?

- **Category:** Networking / System Design
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Server owns item instances; choose simple owner-only state, Fast Array or subobjects based on item count, nested state and client audience.
- **Strong 3-Year-Engineer Answer:** I start with durable authoritative state on the server, stable item IDs and owner-only/public audience decisions. If the inventory is small, a simple replicated struct/array can be enough. If item-level deltas matter, I use Fast Array with dirty-marked authoritative mutations and idempotent client callbacks. If items have nested replicated state that does not need Actor identity, I consider replicated subobjects; if they need world identity/relevancy, they may be Actors. I test late join, reorder, drop/equip/remove, stale UI selection and packet loss.
- **Common Weak Answer:** “Use a reliable RPC whenever inventory changes.”
- **Follow-up Questions:** Owner-only? Stable IDs? Equip state? Subobject versus Actor? UI stale pointer?
- **Hands-on Verification Task:** Implement two versions of a 50-item inventory and compare bytes, correctness and complexity.
- **Sources:** [SRC-NET-015], [SRC-NET-012], [SRC-NET-003], [SRC-NET-004]
- **Version Notes:** Fast Array and subobject APIs require target branch confirmation.

### Question: How does CharacterMovement client prediction work?

- **Category:** Networking / CharacterMovement
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** The owning client saves and simulates moves immediately; the server reproduces them authoritatively; corrections cause the client to apply server state and replay unacknowledged moves.
- **Strong 3-Year-Engineer Answer:** CharacterMovement is not simple transform replication. The autonomous proxy samples input, stores a move, simulates it locally and sends compact move data to the server. The server reconstructs and validates movement, then acknowledges or corrects. When corrected, the client returns to the authoritative state and replays pending moves; simulated proxies smooth remote updates. Custom movement state must enter that same saved-move path or prediction will diverge.
- **Common Weak Answer:** “The client moves and the server replicates the movement back.”
- **Follow-up Questions:** Autonomous proxy? Saved move? Correction? Replay? Simulated proxy smoothing?
- **Hands-on Verification Task:** Trigger one movement correction under packet lag and explain the saved move, server reproduction, correction and replay sequence from logs.
- **Sources:** [SRC-NET-009], [SRC-NET-017], [SRC-NET-019]
- **Version Notes:** High-level pipeline is stable; exact packed move APIs are branch-sensitive.

### Question: When do you extend `FSavedMove_Character`?

- **Category:** Networking / CharacterMovement
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** When small custom state affects CharacterMovement simulation and must be captured, compressed, restored and replayed with the client move.
- **Strong 3-Year-Engineer Answer:** I extend saved moves for movement-affecting state such as predicted sprint, dash input edge, custom movement mode flags or compact surface evidence. I avoid putting cosmetics or final gameplay outcomes in the move. The implementation captures state in `SetMoveFor`, encodes compact flags/data, restores state for replay, updates server simulation from compressed flags and prevents unsafe move combining across state transitions. I then validate server authority and measure correction rate.
- **Common Weak Answer:** “Whenever I add a replicated movement bool.”
- **Follow-up Questions:** `CanCombineWith`? compressed flags? replay restore? what not to save? server validation?
- **Hands-on Verification Task:** Add predicted sprint with a custom saved-move flag and prove correction counts improve under lag/loss.
- **Sources:** [SRC-NET-009], [SRC-NET-018], [SRC-NET-019]
- **Version Notes:** Treat code examples as schematic until compiled against target headers.

### Question: Why does a predicted sprint or dash jitter even though the bool replicates?

- **Category:** Networking / Debugging
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** The replicated bool arrives too late for movement prediction; the movement-affecting state must be part of the saved move/server reproduction path.
- **Strong 3-Year-Engineer Answer:** A replicated `bIsSprinting` can eventually converge presentation, but CharacterMovement prediction needs the sprint state at the exact input sample that generated a move. If the client predicts with sprint speed while the server reproduces the same move without sprint, the server correction is valid. I would log compressed flags, max speed, movement mode and correction distance on client/server, check `CanCombineWith`, and verify stamina/tag validation is consistent or intentionally corrective.
- **Common Weak Answer:** “Increase `NetUpdateFrequency` or make the RPC reliable.”
- **Follow-up Questions:** delayed state? saved move? correction distance? stamina mismatch? move combining?
- **Hands-on Verification Task:** Implement sprint first as a replicated bool, then as saved-move state, and compare correction metrics.
- **Sources:** [SRC-NET-009], [SRC-NET-017], [SRC-NET-018], [SRC-NET-010]
- **Version Notes:** Exact debug commands/CVars differ by target branch.

### Question: How would you design a predicted dash?

- **Category:** Networking / CharacterMovement
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Save the dash input edge and compact movement state, validate it on the server, integrate movement through CharacterMovement or a supported prediction path, and instrument corrections.
- **Strong 3-Year-Engineer Answer:** I would start by deciding whether the dash is CharacterMovement-driven, root-motion-driven or a root-motion-source style movement. The owning client can predict the input edge, direction and custom mode; the saved move must preserve that edge so replay produces exactly one dash. The server validates cooldown, resources, tags, direction, floor/wall context and collision. I would reject impossible dashes cleanly, avoid direct transform writes, and test packet lag/loss with correction logs and replay traces.
- **Common Weak Answer:** “On button press, call a Server RPC that launches the character.”
- **Follow-up Questions:** input edge? double dash bug? root motion? direct transform? rejection UX?
- **Hands-on Verification Task:** Add a dash to Project 3, deliberately let moves combine across the dash edge, then fix and document the difference.
- **Sources:** [SRC-NET-009], [SRC-NET-018], [SRC-ANIM-014]
- **Version Notes:** Animation/root-motion integration is target and implementation sensitive.

### Question: What is lag compensation or server rewind?

- **Category:** Networking / Lag Compensation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** The server evaluates an action against bounded historical hit-relevant state so it can judge what the client plausibly saw without trusting the client's hit result.
- **Strong 3-Year-Engineer Answer:** For a hitscan weapon, the client can predict presentation and send fire intent with sequence, timing and aim evidence. The server validates ownership, weapon state, fire rate, ammo, timestamp window and aim plausibility. It then samples server-stored target snapshots near the query time, performs its own trace against rewind hitboxes, applies authoritative damage and replicates the result. The hard parts are bounded windows, clock mapping, occlusion/respawn policy, memory budget and abuse detection.
- **Common Weak Answer:** “The client tells the server who it hit because the client saw it.”
- **Follow-up Questions:** timestamp trust? max rewind? hitbox snapshots? current blockers? abuse detection?
- **Hands-on Verification Task:** Build a bounded server rewind buffer for hitscan targets and compare no rewind, bounded rewind and excessive rewind cases.
- **Sources:** [SRC-NET-001], [SRC-NET-006], [SRC-PERF-006]
- **Version Notes:** This is a design pattern synthesis, not a single generic UE subsystem guarantee.

### Question: What should a lag-compensation history buffer store?

- **Category:** Networking / Lag Compensation
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Store compact value snapshots of hit-relevant state: server time/sequence, root transform, hitboxes or pose-derived boxes, movement/target state and enough metadata to validate transitions.
- **Strong 3-Year-Engineer Answer:** I store the minimum data required for authoritative hit tests: timestamp, sequence, root transform, compact hitbox transforms/bounds, velocity if interpolation/extrapolation is allowed, and state such as alive/team/invulnerable or respawn generation. I avoid storing mutable pointers to current transforms. The buffer is bounded, allocation-free in steady state and sized from player count, snapshot rate, retained window and hitbox count. Query complexity is chosen from measured workload.
- **Common Weak Answer:** “Store every Actor transform for a few seconds.”
- **Follow-up Questions:** value snapshot? ring buffer? generation ID? memory budget? interpolation?
- **Hands-on Verification Task:** Estimate memory for 64 players, 16 snapshots each and 32 compact hitboxes, then verify with Memory Insights or custom counters.
- **Sources:** [SRC-ALG-002], [SRC-PERF-006], [SRC-NET-010]
- **Version Notes:** Exact hitbox representation is game-specific.

### Question: How do prediction, reconciliation, interpolation, rewind and rollback differ?

- **Category:** Networking / Concepts
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Prediction simulates locally, reconciliation applies server correction and replays, interpolation smooths remote state, rewind evaluates past state, and rollback restores old simulation state then resimulates.
- **Strong 3-Year-Engineer Answer:** I use the terms narrowly. CharacterMovement prediction is local speculation for the owner. Reconciliation is accepting a server correction and replaying unacknowledged moves. Interpolation/smoothing presents remote proxies between updates. Server rewind is a lag-compensation hit test against historical snapshots. Rollback is a broader deterministic or controlled simulation architecture with input logs, state snapshots, resimulation and side-effect control. Calling all of these “rollback” hides important engineering constraints.
- **Common Weak Answer:** “Rollback means the client corrects its movement.”
- **Follow-up Questions:** simulated proxy? replay? hit rewind? deterministic state? side effects?
- **Hands-on Verification Task:** Draw a timeline for one movement correction and one lag-compensated shot, labelling each term correctly.
- **Sources:** [SRC-NET-009], [SRC-NET-001], [SRC-ALG-002]
- **Version Notes:** Network Prediction plugin or custom rollback systems should be labelled separately.

### Question: How do you prevent lag compensation from becoming an exploit?

- **Category:** Networking / Security
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Clamp rewind windows, map timestamps through server-known timing, validate weapon/player state and aim evidence, define occlusion/respawn policy, and log suspicious extremes.
- **Strong 3-Year-Engineer Answer:** I never trust a client hit result or arbitrary timestamp. The server owns fire rate, ammo, cooldowns, alive/team state and damage. Client timing is converted into a bounded server query time, with impossible values rejected. Aim evidence must be plausible for the player view/muzzle, and the rewind policy must handle blockers, doors, death, respawn and invulnerability. I also monitor repeated max-window hits, sequence gaps and impossible aim deltas because fairness and anti-cheat are part of the design.
- **Common Weak Answer:** “The server is authoritative, so lag compensation is safe.”
- **Follow-up Questions:** clock mapping? maximum window? closed doors? respawn generation? suspicious patterns?
- **Hands-on Verification Task:** Add three cheat-style requests to the Project 3 hitscan test: old timestamp, impossible aim delta and stale target generation.
- **Sources:** [SRC-NET-001], [SRC-NET-006], [SRC-NET-010]
- **Version Notes:** Anti-cheat thresholds are project and genre dependent.

### Question: How would you profile custom movement prediction and lag compensation?

- **Category:** Networking / Profiling
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Measure correction counts/distances, saved move allocation/combine/replay, custom move bytes, history memory/allocation, rewind query cost and traffic under representative lag/loss.
- **Strong 3-Year-Engineer Answer:** I split feel, CPU, memory and bandwidth. For movement, I track corrections per feature, correction distance, move allocation, move combining and replay count under packet conditions. For lag compensation, I track snapshot bytes, allocation rate, retained window, hitbox count, query/trace cost and false local hit confirmations/rejections. Network Profiler/Insights identifies traffic; Unreal Insights and counters identify CPU/memory. LAN smoothness is not evidence.
- **Common Weak Answer:** “Use Network Profiler and see if it is high.”
- **Follow-up Questions:** correction metric? memory formula? query cost? packet loss? false hit marker?
- **Hands-on Verification Task:** Produce a before/after table for saved-move sprint and a memory/trace-cost table for rewind buffers.
- **Sources:** [SRC-NET-010], [SRC-PERF-003], [SRC-PERF-004], [SRC-PERF-006]
- **Version Notes:** Tool availability and trace channels vary by branch/build.

### Question: How would you answer a custom CharacterMovement question when you have not shipped it?

- **Category:** Networking / Interview Strategy
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Be explicit about the known pipeline, the extension points you would verify, and the tests/profiling you would use; do not invent exact signatures.
- **Strong 3-Year-Engineer Answer:** I would say I understand the contract: client saved moves, server reproduction, correction and replay. For implementation, I would inspect the target branch's `UCharacterMovementComponent`, `FSavedMove_Character` and client prediction data APIs before copying signatures. I can still design the state boundary: save only movement-affecting compact state, prevent unsafe combine, validate on the server and measure corrections under lag/loss. That is stronger than pretending every UE minor version exposes the same hooks.
- **Common Weak Answer:** “I would Google a tutorial and add `FLAG_Custom_0`.”
- **Follow-up Questions:** target branch? schematic code? validation? correction proof? what not to claim?
- **Hands-on Verification Task:** Write a branch-verification note listing exact saved-move virtuals used by your project and one compile-tested extension.
- **Sources:** [SRC-NET-017], [SRC-NET-018], [SRC-NET-019], [SRC-NET-009]
- **Version Notes:** Deliberately version-sensitive; honesty is part of the answer.

## Device Lab Automation And Target Evidence

### Question: How would you design a device-lab gate for an Unreal project?

- **Category:** Profiling / Build / Platform Automation
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Package reproducibly, run deterministic target scenarios on a small device matrix, capture logs/CSV/traces/crashes, attach every artifact to build/device/scenario identity, and gate only stable actionable signals.
- **Strong 3-Year-Engineer Answer:** I would start with a small matrix: one fast smoke lane, one low supported device/tier, one target device/tier and a reporting-only long soak. BuildCookRun or BuildGraph produces the package, symbols, manifests and build metadata. A runner such as Gauntlet/RunUnreal, AutomationTool or a platform script installs and launches deterministic scenarios with warm-up/sample windows and explicit exit markers. Every run archives logs, CSV, active Device Profile/CVars, screenshots where useful, selected traces and crash data. I would keep noisy metrics reporting-only until variance is understood, then promote only actionable thresholds to blocking gates.
- **Common Weak Answer:** “Just run the packaged game on a phone in CI.”
- **Follow-up Questions:** Which devices? Which scenarios? What artifacts? What is blocking versus reporting? How do you handle flakes?
- **Hands-on Verification Task:** Build the Project 4/6 Device Lab Automation Extension and produce one run manifest plus CSV report.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016], [SRC-PERF-009], [SRC-PERF-011], [SRC-PLAT-009]
- **Version Notes:** Device deployment, runner commands and telemetry flags are branch/platform sensitive.

### Question: Where does Gauntlet fit in Unreal automation?

- **Category:** Build / Automation / Platform
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Gauntlet is a UE-aware automation framework used through RunUnreal-style workflows to launch packaged/editor targets, run tests and monitor results; it is not a substitute for project scenario design or platform validation.
- **Strong 3-Year-Engineer Answer:** I treat Gauntlet as an orchestration layer when it fits the project. Public Epic docs show `RunUnreal` workflows for boot tests, editor automation, target automation, networking tests and target platform/device command shapes. The project still needs deterministic scenario controllers, readiness/exit markers, telemetry setup and artifact policy. I would verify the exact C# APIs, command classes, device adapters and platform deployment support in the target engine branch before promising a production lane.
- **Common Weak Answer:** “Gauntlet automatically tests the whole game.”
- **Follow-up Questions:** RunUAT? `UE.TargetAutomation`? packaged build path? target device adapters? project controller?
- **Hands-on Verification Task:** Create a minimal boot or target-automation runner note listing exact command line, target build path and artifacts pulled.
- **Sources:** [SRC-PLAT-009], [SRC-BUILD-016], [SRC-PERF-011]
- **Version Notes:** Gauntlet APIs and target-device support are implementation and platform sensitive.

### Question: What belongs in a device-lab run manifest?

- **Category:** Build / Release Evidence
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Build ID, engine/project revision, platform/configuration, package and symbols paths, device ID/state, scenario ID, command line, install mode and output artifact paths.
- **Strong 3-Year-Engineer Answer:** A useful manifest lets another engineer reproduce or triage without searching CI history. I include build/package identity, symbols, cook/stage manifests, exact command parameters, device identity and state, scenario contract, install mode, telemetry settings and output paths for log, CSV, trace, screenshot/video and crash dump. The manifest should travel with the artifact set because dashboards and URLs expire.
- **Common Weak Answer:** “The CI job URL is enough.”
- **Follow-up Questions:** symbols? stage manifest? device profile? install mode? command line?
- **Hands-on Verification Task:** Write a JSON manifest for one packaged smoke run and verify every referenced file exists.
- **Sources:** [SRC-BUILD-017], [SRC-BUILD-018], [SRC-PLAT-007], [SRC-PLAT-008]
- **Version Notes:** Exact package/symbol formats vary by platform and configuration.

### Question: How do CSV and Unreal Insights differ in a device lab?

- **Category:** Profiling / Automation
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** CSV is light, repeatable telemetry for gates and trend aggregation; Insights is a heavier causal trace used selectively around a failed or sampled window.
- **Strong 3-Year-Engineer Answer:** I run CSV frequently because it can capture frame percentiles, hitches, counters and memory-related tags across repeated packaged scenarios with manageable overhead. If a CSV gate fails, I use the worst repeatable window to capture a short Unreal Insights, Memory Insights, GPU or platform-profiler trace. Always-on full tracing can change timing and produce too much data, so I separate broad detection from causal diagnosis.
- **Common Weak Answer:** “Insights is better, so always trace everything.”
- **Follow-up Questions:** overhead? aggregation? hitch window? trace channels? Memory Insights versus LLM?
- **Hands-on Verification Task:** Run one scenario with CSV for three repetitions, then capture a short trace only for the worst repeatable hitch.
- **Sources:** [SRC-PERF-003], [SRC-PERF-006], [SRC-PERF-009], [SRC-PERF-010]
- **Version Notes:** CSV categories, trace channels and overhead vary by branch/build/platform.

### Question: How do you debug a device-lab failure that only happens on one device?

- **Category:** Platform / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Classify phase and identity first, compare nearest passing runs, verify device state/profile/install mode, rerun only for recognised infrastructure classes, then capture the smallest causal evidence.
- **Strong 3-Year-Engineer Answer:** I would not immediately change gameplay code from one outlier. I check build ID, device ID, scenario ID, command line, install mode, active profile/CVars, thermal/power state and the first causal log. Then I compare same build/different device and same device/different build. If it looks like install, pull, device-health or thermal noise, I do one controlled rerun or quarantine the device after repeated health failures. If it reproduces with a stable crash, hitch or memory pattern, I capture a focused trace or platform profiler and assign product ownership.
- **Common Weak Answer:** “Rerun until it passes.”
- **Follow-up Questions:** nearest passing run? device quarantine? thermal? profile proof? product versus infrastructure?
- **Hands-on Verification Task:** Inject one lab-infrastructure failure and one deterministic product failure and classify them differently in the report.
- **Sources:** [SRC-PERF-001], [SRC-PERF-009], [SRC-PLAT-005]
- **Version Notes:** Platform health signals and profiler access are platform sensitive.

### Question: When should an automated metric become a release-blocking gate?

- **Category:** Profiling / Release Management
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** After the team has measured variance, confirmed the signal is stable on target devices, defined tolerance and owner, and proved failures are actionable.
- **Strong 3-Year-Engineer Answer:** I start new metrics as reporting lanes. I collect enough history across repeated runs and device states to understand natural variance. Then I define thresholds with tolerance bands, acceptable rerun policy and ownership. A good blocking gate fails for a reason the team can investigate and fix, not because one device was hot, a profiler changed timing or a network dependency flickered. Release-blocking gates should produce evidence, not just a red build.
- **Common Weak Answer:** “Set a target FPS and fail the build below it.”
- **Follow-up Questions:** variance? tolerance band? owner? reporting lane? rerun policy?
- **Hands-on Verification Task:** Convert one CSV metric from reporting to proposed blocking status with variance data and an owner.
- **Sources:** [SRC-PERF-001], [SRC-PERF-009], [SRC-PERF-011]
- **Version Notes:** Thresholds are product, device and genre dependent.

### Question: Why is forced packaged-crash proof part of a device-lab packet?

- **Category:** Build / Crash Debugging
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It proves the archived package, symbols, crash metadata and symbolication path match the exact build that users or testers will run.
- **Strong 3-Year-Engineer Answer:** A crash report without matching symbols is often a dead end. I force a known packaged crash in the lab, archive the package and symbols, then verify the callstack symbolicates to the expected function and build ID. This catches wrong symbols, missing uploads, privacy/endpoint mistakes and build-artifact mismatch before a real device-only crash appears late in release.
- **Common Weak Answer:** “Crashes will show the line number automatically.”
- **Follow-up Questions:** build ID? symbols? crash endpoint? log tail? privacy?
- **Hands-on Verification Task:** Force a packaged crash and attach package ID, symbols path, crash context and symbolicated callstack to the evidence packet.
- **Sources:** [SRC-BUILD-018], [SRC-BUILD-017]
- **Version Notes:** Symbol formats and crash reporting endpoints are platform/studio specific.

### Question: What is the difference between a product failure, lab infrastructure failure and flaky device?

- **Category:** Automation / Triage
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** A product failure reproduces against the build/scenario, a lab infrastructure failure belongs to the runner/install/artifact path, and a flaky device repeatedly fails health or environment checks outside product logic.
- **Strong 3-Year-Engineer Answer:** A deterministic crash with a stable callstack is product evidence even if it appears in automation. A failed artifact pull, missing runner permission or broken install service is lab infrastructure. A device that intermittently fails install, overheats, loses connection or cannot provide logs across different builds should be health-checked and possibly quarantined. The gate output should make this classification visible so ownership is not guessed from a red CI job.
- **Common Weak Answer:** “Any red device run means the game is broken.”
- **Follow-up Questions:** health check? quarantine? rerun? deterministic crash? artifact pull?
- **Hands-on Verification Task:** Add a triage column to the device-lab report and classify five synthetic failures.
- **Sources:** [SRC-PERF-001], [SRC-PERF-011], [SRC-PLAT-009]
- **Version Notes:** Studio CI policy controls rerun and quarantine rules.

## World Partition And Large-World Pipeline

### Question: How would you design a UE5 open-world production pipeline?

- **Category:** World Partition / System Design
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Define traversal and platform budgets, World Partition grid/source policy, spatial versus always-loaded Actors, Data Layers, reusable POIs, HLOD, cook/package gates and target traversal evidence.
- **Strong 3-Year-Engineer Answer:** I would start from traversal speed, memory/I/O budgets, target platforms and failure tolerance. Then I would define World Partition cell/source policy, which Actors are spatial versus always loaded, how Runtime Data Layers present world phases, when to use Level Instances or Packed Level Blueprints, how HLOD proxies are built, and how generated content is assigned/cleaned. Durable state lives in save/server/quest systems, not only in unloadable Actors. The release proof is a clean cook/package plus traversal and fast-travel runs with loading, memory, hitch and visual-transition evidence.
- **Common Weak Answer:** “Use World Partition and HLOD.”
- **Follow-up Questions:** grid size? streaming source? always-loaded reference? Data Layer authority? packaged proof?
- **Hands-on Verification Task:** Build the Project 6 Large-World Production Pipeline Extension and present the five-minute memo.
- **Sources:** [SRC-ASSET-010], [SRC-WORLD-001], [SRC-WORLD-003], [SRC-WORLD-004], [SRC-BUILD-017]
- **Version Notes:** World Partition runtime hash, builders and Data Layer APIs are UE5 minor-version sensitive.

### Question: World Partition versus Level Streaming?

- **Category:** World Partition / Streaming
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Level Streaming explicitly loads/unloads level packages; World Partition divides a persistent UE5 world into grid cells driven by streaming sources and integrates with OFPA, Data Layers and HLOD.
- **Strong 3-Year-Engineer Answer:** I would choose from world shape, authoring collaboration, runtime control and validation needs. World Partition is strong for large continuous UE5 worlds because it automates spatial cell streaming and ties into OFPA/Data Layers/HLOD. Explicit Level Streaming can still be valid for bounded interiors, arenas, menus, authored scenarios or cases where level package control is clearer. With either system, durable gameplay state must not live only in unloadable Actors, and teleport/readiness/cook behavior needs packaged proof.
- **Common Weak Answer:** “World Partition replaced level streaming.”
- **Follow-up Questions:** streaming source? `Is Spatially Loaded`? Data Layers? sublevels? fast travel?
- **Hands-on Verification Task:** Implement one WP region and one explicit streamed level scenario, then compare control, package proof and debugging complexity.
- **Sources:** [SRC-ASSET-009], [SRC-ASSET-010], [SRC-WORLD-001]
- **Version Notes:** World Partition is UE5-specific; legacy World Composition and project streaming setups vary.

### Question: How should Runtime Data Layers be used for progression?

- **Category:** World Partition / Data Layers
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Keep durable progression in quest/save/server state, then use Runtime Data Layers to present the correct world phase through load/activation policy.
- **Strong 3-Year-Engineer Answer:** I treat Data Layers as a presentation and streaming state mechanism, not the source of truth. A quest/save/server model owns stable world phase such as `VillageRestored`; a coordinator applies the corresponding Runtime Data Layer state, waits for readiness and handles rollback/load. This makes save/load, late join, replication and content migration testable. I also distinguish Data Layer Assets from world-specific Instances and never assume an editor-visible layer is runtime-active.
- **Common Weak Answer:** “Put each quest phase in a Data Layer.”
- **Follow-up Questions:** Asset versus Instance? Editor versus Runtime? save authority? late join? initial runtime state?
- **Hands-on Verification Task:** Add two Runtime Data Layers to the large-world extension and drive them from a saveable world-state coordinator.
- **Sources:** [SRC-WORLD-001], [SRC-ASSET-010]
- **Version Notes:** Exact Data Layer subsystem APIs require target-branch verification.

### Question: How do you prevent fast travel into unloaded or collisionless space?

- **Category:** World Partition / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Move or create a destination streaming source, wait for streaming completion and project-specific collision/nav/gameplay readiness, then teleport or show a loading/fallback path.
- **Strong 3-Year-Engineer Answer:** I would never use a blind delay as the real fix. The travel system should request streaming around the destination, poll a readiness concept such as the streaming source completion state, and add project checks for collision, nav, key gameplay actors and Data Layer state. If readiness times out, it should fail gracefully. I would validate the worst fast-travel path in a packaged build on target hardware and capture the loading/hitch trace.
- **Common Weak Answer:** “Wait two seconds before teleporting.”
- **Follow-up Questions:** source range? target state? collision proof? timeout? packaged trace?
- **Hands-on Verification Task:** Inject a no-readiness teleport and repair it with a streaming-source readiness gate.
- **Sources:** [SRC-ASSET-010], [SRC-PERF-003], [SRC-PERF-009]
- **Version Notes:** API signatures and debug views vary by UE5 branch.

### Question: What can go wrong with One File Per Actor?

- **Category:** World Partition / Source Control
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** OFPA reduces level-file conflicts, but encoded external actor files and partial submits can create missing or dangling references.
- **Strong 3-Year-Engineer Answer:** OFPA is an editor/source-control storage workflow, not runtime streaming. It stores Actor instances externally during editing and World Partition uses it by default, but cooked levels embed the Actor data. Because file names are encoded, I validate through the editor/source-control changelist view and run reference/redirector checks. A partial submit can pass on one artist's machine because local unsaved or already-loaded files hide the missing external actor.
- **Common Weak Answer:** “OFPA means map conflicts are solved.”
- **Follow-up Questions:** cooked behavior? encoded filenames? changelist view? dangling references? clean workspace?
- **Hands-on Verification Task:** Omit one external actor file from a test changelist and reproduce the failure in a clean workspace.
- **Sources:** [SRC-WORLD-002], [SRC-ASSET-010], [SRC-ASSET-007]
- **Version Notes:** Source-control integration and external actor paths are project/toolchain sensitive.

### Question: Level Instance or Packed Level Blueprint?

- **Category:** World Building / Authoring
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Use Level Instances for reusable authored Actor arrangements and Packed Level Blueprints for static dense visual arrangements; avoid treating either as a generic gameplay-state container.
- **Strong 3-Year-Engineer Answer:** I would choose from mutability, density, runtime mode and identity. A reusable POI with normal Actor composition can fit a Level Instance. A dense static kitbash often fits Packed Level Blueprint because it is optimised for static visual content. If the object is dynamic gameplay with unique save/replication identity, I would use normal Actor/data architecture instead. I would also verify embedded versus Level Streaming mode and OFPA compatibility because high-density streaming-mode instances can create runtime cost.
- **Common Weak Answer:** “Packed Level Blueprint is just a faster Level Instance.”
- **Follow-up Questions:** static or dynamic? OFPA? embedded mode? Level Streaming mode cost? stable IDs?
- **Hands-on Verification Task:** Build the same POI both ways and document authoring, package and runtime trade-offs.
- **Sources:** [SRC-WORLD-004], [SRC-WORLD-002]
- **Version Notes:** Level Instance runtime modes and editor behavior are UE5 branch-sensitive.

### Question: How do you evaluate HLOD for a large world?

- **Category:** Rendering / World Partition
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Name the cost HLOD should reduce, choose HLOD layers/proxy settings, build them in automation, and compare far/transition/near evidence for performance, memory and artefacts.
- **Strong 3-Year-Engineer Answer:** I do not enable HLOD just because the world is large. I first identify whether the problem is draw/RHI, primitive count, material cost, shadows, streaming or memory. Then I choose instancing/merged/simplified layers, check material merge and proxy settings, build HLOD in CI, and measure far, transition and near positions. HLOD can reduce distant representation cost but it will not fix near-field Actor activation, collision/nav setup or shader/texture hitches during handoff.
- **Common Weak Answer:** “HLOD reduces draw calls, so it is always good.”
- **Follow-up Questions:** proxy memory? material artefacts? collision? Nanite? handoff hitch?
- **Hands-on Verification Task:** Capture far/transition/near traces for one HLOD region and write a before/after table.
- **Sources:** [SRC-WORLD-003], [SRC-RENDER-011], [SRC-RENDER-008], [SRC-PERF-003]
- **Version Notes:** HLOD settings, debug views and builder output vary by target branch and content.

### Question: What belongs in a packaged large-world validation gate?

- **Category:** Build / World Partition
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Validate OFPA/source-control, references, Data Layers, Level Instances, generated output, HLOD build, cook/package and target traversal telemetry.
- **Strong 3-Year-Engineer Answer:** A useful gate checks the whole world pipeline, not only whether the editor opens the map. I would run source-control validation for OFPA, detect prohibited hard references to spatial Actors, validate Runtime Data Layer policy, verify Level Instance mode/OFPA compatibility, check PCG/generated output assignment, build HLODs, cook/package from a clean workspace and run packaged traversal/fast-travel scenarios. The output should include logs, manifests where available, loading/memory/hitch telemetry and an owner for each failure category.
- **Common Weak Answer:** “If the map cooks, it is fine.”
- **Follow-up Questions:** stale HLOD? clean workspace? traversal path? active Data Layers? failure owner?
- **Hands-on Verification Task:** Add a large-world CI checklist to Project 6 and deliberately fail three categories.
- **Sources:** [SRC-ASSET-010], [SRC-WORLD-002], [SRC-WORLD-003], [SRC-PCG-001], [SRC-BUILD-016], [SRC-BUILD-017]
- **Version Notes:** Commandlets, builder names and package artifacts are branch/platform sensitive.

## Android Platform Profiling

### Question: How do you profile Android GPU performance for a UE game?

- **Category:** Platform / Android Profiling
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use a packaged build, Unreal stats/CSV for scenario classification, then Android system/frame profiling to identify GPU scheduling, render passes, counters and device behavior.
- **Strong 3-Year-Engineer Answer:** I start with a deterministic packaged scenario and Unreal CSV/stat data to identify the failing window. If the problem is render/GPU, I capture Android system profiling through APA or AGI to see CPU/GPU scheduling, present waits and GPU queue/utilization. For a representative GPU-bound frame, I use AGI Frame Profiler to identify expensive render passes and classify binning, rendering or GMEM load/store cost, then map that back to Unreal passes, materials, render targets, shadows, UI or scene captures. Before/after must use the same device, graphics API, Device Profile, scalability and sample window.
- **Common Weak Answer:** “Open AGI and look for red bars.”
- **Follow-up Questions:** packaged build? active profile? system profile versus frame profile? render pass owner? repeatability?
- **Hands-on Verification Task:** Complete the Project 4 Android Platform Profiler Extension and produce a one-page before/after report.
- **Sources:** [SRC-PLAT-010], [SRC-PLAT-011], [SRC-PLAT-012], [SRC-PLAT-013], [SRC-PERF-009]
- **Version Notes:** AGI/APA support depends on Android version, GPU, driver, graphics API and tool release.

### Question: Unreal Insights versus Android GPU Inspector?

- **Category:** Platform / Profiler Selection
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Unreal Insights shows engine-side scopes/tasks/loading/memory; AGI/APA shows Android device, OS, GPU queue, render-pass and counter evidence.
- **Strong 3-Year-Engineer Answer:** I use Unreal Insights when I need Unreal ownership: game-thread scopes, Render/RHI work, tasks, loading, memory and named markers. I use Android profiling when I need device truth: CPU scheduling, present/swap waits, GPU utilization/queue, render-pass execution, GMEM load/store, counters or thermal/device behavior. The useful answer correlates them. Android tools do not know my gameplay ownership, and Unreal traces alone can miss driver/GPU/OS behavior.
- **Common Weak Answer:** “AGI is better because it is lower level.”
- **Follow-up Questions:** active CPU? GPU wait? Unreal marker? render pass? artifact correlation?
- **Hands-on Verification Task:** Capture the same Android scenario with Unreal Insights and AGI/APA, then explain one thing each tool revealed that the other did not.
- **Sources:** [SRC-PERF-003], [SRC-PLAT-011], [SRC-PLAT-012], [SRC-PLAT-013]
- **Version Notes:** Trace channel and Android profiler availability vary by build/device.

### Question: Android shows high CPU frame time. What do you check before optimising gameplay?

- **Category:** Platform / Android CPU-GPU Diagnosis
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Distinguish active CPU work from total CPU time that includes waiting on GPU/present, then correlate with Unreal Game/Render/RHI scopes.
- **Strong 3-Year-Engineer Answer:** On Android, high total CPU frame time can include waiting around `eglSwapBuffers`, `dequeueBuffer` or `vkQueuePresentKHR` when the GPU is the bottleneck. I check active CPU time, GPU utilization/queue and present waits in Android profiling, then map engine-side work with Unreal stats/Insights. If active CPU is low and GPU wait is high, rewriting gameplay Tick is the wrong first fix; I would investigate render passes, materials, resolution, overdraw, bandwidth or sync points.
- **Common Weak Answer:** “CPU is high, so reduce Tick.”
- **Follow-up Questions:** total versus active CPU? present wait? GPU queue? Render/RHI? exact frame window?
- **Hands-on Verification Task:** Create one GPU-bound Android scene where CPU total appears high and explain the wait from Android profiling.
- **Sources:** [SRC-PLAT-012], [SRC-PERF-002], [SRC-PERF-003]
- **Version Notes:** Track names and visibility depend on graphics API/device/tool.

### Question: How do you diagnose GMEM or render-pass bandwidth on mobile?

- **Category:** Platform / Mobile GPU
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use AGI Frame Profiler to find the expensive pass, inspect binning/rendering/GMEM load-store signals, then map the pass to Unreal render targets, depth/stencil, shadows, post, UI or scene captures.
- **Strong 3-Year-Engineer Answer:** I first identify the representative frame and longest render pass. If rendering dominates, I look at shader complexity, texture fetches, overdraw and framebuffer resolution. If binning dominates, I inspect geometry, vertex format and draw structure. If GMEM load/store dominates, I look for render-target/depth/stencil stores and pass transitions that can be removed, reduced or downsampled. The fix has to preserve visual correctness and be verified on the same device/frame.
- **Common Weak Answer:** “Reduce triangles.”
- **Follow-up Questions:** longest pass? binning versus rendering? GMEM store? render target? visual proof?
- **Hands-on Verification Task:** Capture one AGI frame profile and write a pass-level diagnosis table.
- **Sources:** [SRC-PLAT-013], [SRC-RENDER-016], [SRC-RENDER-017], [SRC-PERF-003]
- **Version Notes:** Mobile GPU architecture and profiler panes vary by vendor and driver.

### Question: How do you interpret Android GPU counters safely?

- **Category:** Platform / GPU Counters
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Use counters to support a concrete hypothesis, and treat names/definitions as GPU-vendor-specific rather than universal metrics.
- **Strong 3-Year-Engineer Answer:** I do not compare raw counter names across Mali, PowerVR and Adreno as if they were the same. I record device/GPU/driver/tool version, use counters to test a hypothesis such as texture bandwidth, vertex pressure, shader pressure or idle/wait, then check the vendor documentation where necessary. Counters are supporting evidence; the fix still needs a content/engine owner and before/after frame-time proof.
- **Common Weak Answer:** “GPU counter X is high, so the fix is obvious.”
- **Follow-up Questions:** GPU vendor? counter definition? matching Unreal content? before/after? driver support?
- **Hands-on Verification Task:** Compare two Android devices and document why their raw GPU counter names are not directly interchangeable.
- **Sources:** [SRC-PLAT-014], [SRC-PLAT-010]
- **Version Notes:** Counter support and definitions are vendor/driver/tool sensitive.

### Question: How do you handle Android LMK memory reports?

- **Category:** Platform / Memory
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Treat LMK as lifecycle memory pressure, correlate Android termination/vitals with Unreal LLM/Memory Insights, and reproduce foreground/background warm-return behavior.
- **Strong 3-Year-Engineer Answer:** I confirm whether the app was killed, crashed or restarted, then correlate Android LMK/vitals/log evidence with Unreal memory checkpoints, LLM tags and Memory Insights. A foreground playtest is not enough; I test heavy map, background, other memory pressure and foreground return. Android docs call out user-perceived LMK rate as a vitals metric and warn that a lower rate does not automatically prove memory health. The fix may be content residency, texture/mesh/audio pools, lifecycle unload policy or streaming budget.
- **Common Weak Answer:** “Vitals is green, so memory is fine.”
- **Follow-up Questions:** foreground peak? background retained memory? warm start? LLM tags? user-perceived LMK?
- **Hands-on Verification Task:** Run a background/foreground LMK reproduction and correlate Android evidence with Unreal memory checkpoints.
- **Sources:** [SRC-PLAT-015], [SRC-PERF-006], [SRC-PERF-010]
- **Version Notes:** Android vitals data is aggregate/delayed; local reproduction still matters.

### Question: Should you enable the Android ADPF Unreal plugin?

- **Category:** Platform / Thermal Performance
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Evaluate it with a no-adaptation baseline, supported-device proof, thermal/scalability logs and quality acceptance; do not use it to hide underlying regressions.
- **Strong 3-Year-Engineer Answer:** ADPF can help Android games respond to thermal and CPU-management behavior, and the Unreal plugin can use thermal state, scalability and performance hint sessions. I would first establish baseline performance/thermal behavior, then enable the plugin on supported devices and log CVars, thermal headroom/status, scalability changes and frame stability. If ADPF keeps FPS stable by silently dropping quality from high to low, that may be acceptable or not, but it is a product decision and not proof that the content cost was fixed.
- **Common Weak Answer:** “ADPF fixes Android performance.”
- **Follow-up Questions:** baseline? supported device? quality level? performance hints? thermal headroom?
- **Hands-on Verification Task:** Run the Android profiler extension with and without ADPF and report frame stability versus quality changes.
- **Sources:** [SRC-PLAT-016], [SRC-PLAT-017], [SRC-PLAT-005], [SRC-PERF-009]
- **Version Notes:** ADPF plugin version, OS/API support and device behavior are target-sensitive.

### Question: When is Android Fixed Performance Mode useful?

- **Category:** Platform / Benchmarking
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Use it for repeatable benchmark comparisons where available, but do not treat it as representative sustained user behavior.
- **Strong 3-Year-Engineer Answer:** Fixed Performance Mode can reduce dynamic CPU clock variation during benchmarking, which is useful for A/B comparisons. I would record that it was enabled and keep those results separate from normal thermal/device-lab runs. Users do not normally play under lab-fixed conditions, so release confidence still needs sustained normal-mode captures with battery, thermal and ADPF/scalability behavior visible.
- **Common Weak Answer:** “Always enable fixed mode for performance tests.”
- **Follow-up Questions:** availability? benchmark or release run? thermal realism? artifact label? variance?
- **Hands-on Verification Task:** Run the same Android scenario in normal mode and fixed-performance mode where supported, then explain which result answers which question.
- **Sources:** [SRC-PLAT-017], [SRC-PERF-009]
- **Version Notes:** Availability and behavior are device/API dependent.

## Build Farm And Release Failure Diagnosis

### Question: RunUAT reports a generic failure. How do you debug it?

- **Category:** Build / Automation
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Find the first causal error and failed phase before acting on the final AutomationTool wrapper message.
- **Strong 3-Year-Engineer Answer:** I classify the phase first: UBT/UHT build, cook, stage, package, deploy, launch, smoke, archive or symbol upload. Then I search the full log upward from the final wrapper failure to the first causal error, capture the exact command line/environment and reproduce the failing phase alone if possible. The final report should name the phase, first error, artifact/log path, owner and recurrence guard. A red CI badge is not a diagnosis.
- **Common Weak Answer:** “Rerun CI or clean the workspace.”
- **Follow-up Questions:** first causal line? cook log? failed phase? owner? recurrence guard?
- **Hands-on Verification Task:** Take one failed BuildCookRun log and write a one-page phase/owner/fix report.
- **Sources:** [SRC-BUILD-016], [SRC-BUILD-017]
- **Version Notes:** RunUAT commands and log layout vary by engine/platform.

### Question: Local package passes but clean CI fails. What is your model?

- **Category:** Build / CI
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Assume the clean agent exposed an undeclared dependency: generated files, unity includes, local binaries, global third-party files, warm DDC or loose editor content.
- **Strong 3-Year-Engineer Answer:** I would compare clean and local build inputs: Binaries/Intermediate reliance, generated headers, SDK/toolchain, plugin binaries, DDC/cache, staged runtime dependencies and cook roots. Then I would reproduce on a clean workspace and fix the declaration: Build.cs dependency, include, staging rule, cook rule, plugin descriptor or cache key. Copying local outputs into CI is a workaround, not a fix.
- **Common Weak Answer:** “The build farm is flaky.”
- **Follow-up Questions:** non-unity? DDC? RuntimeDependency? plugin binary? clean package?
- **Hands-on Verification Task:** Remove local Binaries/Intermediate/DDC and reproduce one hidden dependency failure.
- **Sources:** [SRC-BUILD-001], [SRC-BUILD-002], [SRC-BUILD-017], [SRC-ASSET-008]
- **Version Notes:** CI cache and unity-build policy are project-specific.

### Question: How does an editor-only module leak into Shipping?

- **Category:** Build / Modules
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** A runtime module or asset depends on Editor-only classes/modules, often hidden by Editor builds or misplaced `WITH_EDITOR` assumptions.
- **Strong 3-Year-Engineer Answer:** I check Runtime module Build.cs dependencies, plugin module types, runtime assets referencing Editor classes and target differences between Editor and Game/Server/Shipping. `WITH_EDITOR` can guard code paths but it does not make a runtime module safe to depend on UnrealEd or PropertyEditor. The fix is usually separating Runtime schema from Editor tooling and proving Game/Server/Shipping package builds.
- **Common Weak Answer:** “Wrap it in `#if WITH_EDITOR`.”
- **Follow-up Questions:** Build.cs dependency? plugin module type? asset class reference? Shipping target? package proof?
- **Hands-on Verification Task:** Inject an editor module dependency into a runtime plugin, observe Shipping failure and repair the boundary.
- **Sources:** [SRC-BUILD-002], [SRC-BUILD-005], [SRC-BUILD-009]
- **Version Notes:** Module host types and descriptor fields are branch-sensitive.

### Question: How do you prove a third-party runtime dependency is staged correctly?

- **Category:** Build / Release
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Launch the package on a clean machine/device and verify staged runtime files, platform paths, symbols and Build.cs/plugin descriptor rules.
- **Strong 3-Year-Engineer Answer:** Headers and link success are not enough. I verify the correct static/import/runtime library for platform/config/architecture, then inspect staged files and launch on a clean machine without global installs. The staging rule should live in Build.cs/plugin configuration, not a manual copy step. I archive the manifest and include a smoke test so missing DLL/shared object/framework failures appear before release.
- **Common Weak Answer:** “It works on my machine.”
- **Follow-up Questions:** RuntimeDependency? delay load? platform architecture? staged manifest? clean machine?
- **Hands-on Verification Task:** Add a mock third-party runtime file to Project 6 and prove clean-machine launch fails before the staging rule and passes after it.
- **Sources:** [SRC-BUILD-003], [SRC-BUILD-005], [SRC-BUILD-017]
- **Version Notes:** Dynamic library handling is platform/configuration sensitive.

### Question: Why force a packaged crash in the build pipeline?

- **Category:** Build / Crash Proof
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** To prove package, build ID, crash metadata, symbol archive and symbolication route match before real users hit crashes.
- **Strong 3-Year-Engineer Answer:** A crash report without matching symbols wastes triage time. I force a known crash in a packaged non-editor build, verify the report arrives with build metadata/log tail and symbolicates to the expected function using the archived symbols. This catches wrong symbols, missing uploads, endpoint/privacy mistakes and binary/package mismatch as release-pipeline bugs.
- **Common Weak Answer:** “Crash reporting will work when needed.”
- **Follow-up Questions:** build ID? symbol path? endpoint? privacy? forced crash function?
- **Hands-on Verification Task:** Add a forced crash smoke to release automation and fail the build if symbolication does not match.
- **Sources:** [SRC-BUILD-018], [SRC-BUILD-017]
- **Version Notes:** Crash reporting endpoints, privacy and symbol formats are project/platform sensitive.

### Question: What does a good BuildGraph dependency model express?

- **Category:** Build / BuildGraph
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** It expresses named inputs, outputs, dependencies, agents, labels and artifact flow, not just a long sequence of shell commands.
- **Strong 3-Year-Engineer Answer:** I model the release as nodes with explicit inputs and outputs: build, cook, validate, stage, package, symbols, smoke, archive and publish. If a package consumes HLOD output, validation reports or cooked artifacts, that dependency should be in the graph. Agent-local files should not be assumed to exist across nodes. Good BuildGraph design makes retries, ownership and missing artifacts visible.
- **Common Weak Answer:** “BuildGraph is just a big XML script.”
- **Follow-up Questions:** node dependency? artifact path? agent boundary? label? stale output?
- **Hands-on Verification Task:** Draw a BuildGraph for Project 6 and intentionally omit one dependency, then explain the failure.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016]
- **Version Notes:** Studio build-farm wrappers may layer additional conventions over BuildGraph.

### Question: What must a release manifest contain to make a package reproducible?

- **Category:** Build / Release
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Artifact identity, command parameters, content scope, symbol/crash routes, logs/manifests and validation results needed to rebuild, launch and diagnose the same package.
- **Strong 3-Year-Engineer Answer:** I treat the release manifest as the contract tying binaries to evidence. At minimum I want engine/project commit, platform/config, package/build ID, build/cook/stage/package commands or graph parameters, SDK/toolchain versions, cooked map/content scope, symbol archive location, crash-reporting route, log/manifests paths and smoke/validation outcomes. Without that, a “good package” is not reproducible and late crash or missing-content bugs become archaeology instead of engineering.
- **Common Weak Answer:** “Keep the package and the build log somewhere.”
- **Follow-up Questions:** build ID? symbol path? cooked scope? command line? smoke result?
- **Hands-on Verification Task:** Archive one packaged build with a manifest, then hand it to another engineer and see if they can launch and symbolicate it without asking you questions.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016], [SRC-BUILD-018]
- **Version Notes:** Exact fields vary by platform/store/studio policy, but artifact identity and symbol linkage are always required.

### Question: How do you decide what belongs in pre-submit, nightly and release-candidate build lanes?

- **Category:** Build / CI
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Put fast ownership-local regressions in pre-submit, cache/clean-machine and target-matrix proof in nightly, and distributable artifact plus crash/symbol proof in release-candidate lanes.
- **Strong 3-Year-Engineer Answer:** I design the matrix around feedback speed and defect class. Pre-submit should catch source, UHT, validators and small smoke failures quickly enough that engineers still care. Nightly should catch hidden clean-agent, non-unity, cook/package and target-matrix issues that are too expensive per change. Release-candidate lanes prove the actual shipped artifact: package, symbols, runtime dependencies, smoke coverage and forced-crash symbolication. If an expensive lane runs too often, people bypass it; if it runs too rarely, release-only failures accumulate.
- **Common Weak Answer:** “Run everything on every change so quality is higher.”
- **Follow-up Questions:** who owns nightly failures? non-unity? package smoke? forced crash? device tiers?
- **Hands-on Verification Task:** Propose a three-tier CI matrix for one Unreal project and justify why each job sits in that tier.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-017], [SRC-BUILD-018]
- **Version Notes:** Exact lane composition depends on project scale, platforms and build-farm cost.

### Question: What is the right cache-busting order for a build or package failure?

- **Category:** Build / Debugging
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Identify the failing phase, rerun the narrowest step, compare local versus clean inputs, invalidate the smallest plausible state, and only then escalate to a full clean rebuild.
- **Strong 3-Year-Engineer Answer:** I do not begin with “delete everything”. First I classify the failing phase and first causal log line. Then I rerun that phase alone if possible and compare local versus clean-agent inputs: generated files, plugin binaries, staging rules, cook roots, DDC, environment variables and SDK/toolchain. If state contamination is still the strongest hypothesis, I invalidate the smallest output or cache that matches the evidence. A full clean rebuild is the expensive last step, not the first line of thought.
- **Common Weak Answer:** “Delete Intermediate, Binaries and DerivedDataCache immediately.”
- **Follow-up Questions:** first causal error? cook-only rerun? DDC versus staged files? toolchain drift? full clean when?
- **Hands-on Verification Task:** Take one package failure and document the smallest cache/output reset that actually fixes it.
- **Sources:** [SRC-BUILD-016], [SRC-BUILD-017]
- **Version Notes:** Cache layers and available rerun granularity vary by branch and CI wrapper.

### Question: What should a build-farm failure report include to be actionable?

- **Category:** Build / Automation
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Failed phase, first causal error, artifact identity, evidence paths, root-cause classification, clean-rerun status and the recurrence guard.
- **Strong 3-Year-Engineer Answer:** I want a report that another engineer can act on without replaying my full investigation. That means naming the failed phase, quoting the first causal error, attaching artifact identity (branch/commit, package/build ID, platform/config), linking logs/manifests/symbols, stating whether it reproduced on a clean rerun, classifying the root cause and ending with the smallest recurrence guard such as a Build.cs fix, staging rule, cook rule, graph dependency or smoke gate. “CI failed” is a dashboard state, not a report.
- **Common Weak Answer:** “Paste the red job link and say packaging failed.”
- **Follow-up Questions:** artifact ID? symbols? reproducible or flaky? owner? recurrence guard?
- **Hands-on Verification Task:** Rewrite one historical build failure into a phase/evidence/root-cause/guard report.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016], [SRC-BUILD-017]
- **Version Notes:** Evidence storage and report format differ by studio, but the information needs are stable.

## AI Custom Senses And MassAI

### Question: When should you implement a custom AI sense?

- **Category:** AI / Perception / Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use a custom sense when recurring game-specific evidence needs perception-style listener, stimulus, memory, range/team and debugging behaviour.
- **Strong 3-Year-Engineer Answer:** I would not create a custom sense for a one-off trigger or ordinary gameplay state. I use it when the input is genuinely sensory evidence: spatial, aged, strength/tagged, listener-filtered and merged with other perception memory. The sense should batch/filter/report stimuli; the AI memory layer decides how evidence affects target, last-known location and confidence. Exact `UAISense` APIs are branch-sensitive, so I verify target headers before implementing.
- **Common Weak Answer:** "Use a custom sense whenever sight/hearing are not enough."
- **Follow-up Questions:** What is evidence versus truth? How do you expire stimuli? What belongs outside the sense?
- **Hands-on Verification Task:** Design a scent sense payload and prove it updates memory without directly choosing a BT branch.
- **Sources:** [SRC-AI-004], [SRC-AI-016], [SRC-AI-017], [SRC-AI-018], [SRC-AI-019]
- **Version Notes:** Custom sense implementation is UE5 branch/API-sensitive.

### Question: What data belongs in a custom stimulus?

- **Category:** AI / Perception / Data Design
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Source identity, location, strength/confidence, tag/type, time/age, success/loss state where relevant, and enough stable identity to handle destroyed/reused Actors.
- **Strong 3-Year-Engineer Answer:** I include the minimum fields needed to interpret evidence later: source or target, stable ID/generation if durability matters, stimulus and receiver locations, strength, tag/category, team/affiliation if intentional, world/server time and expiry policy. I avoid stuffing decision state into the stimulus. The memory projection can then merge sight, hearing, damage and custom evidence with clear confidence/age rules.
- **Common Weak Answer:** "Actor and location."
- **Follow-up Questions:** What if the source is destroyed? Should team be stored or re-evaluated? How does strength differ from priority?
- **Hands-on Verification Task:** Emit a stimulus, destroy the source before processing and handle it without a stale Actor crash.
- **Sources:** [SRC-AI-004], [SRC-AI-018]
- **Version Notes:** `FAIStimulus` constructors/helpers must be verified in target headers.

### Question: Why should a custom sense not write Blackboard keys directly?

- **Category:** AI / Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** The sense should report evidence; a memory/decision layer should interpret and project that evidence into Blackboard or StateTree state.
- **Strong 3-Year-Engineer Answer:** If the sense writes `CurrentTarget` directly, collection, memory and decision become tangled. A failed sight update, weak scent and direct damage all have different confidence and age. I want a separate projection layer that merges sources, decays confidence, writes named keys and is testable. That also lets the same sense feed BT, StateTree, squad memory or Mass aggregation.
- **Common Weak Answer:** "It is easier for the sense to set the target."
- **Follow-up Questions:** How do you merge two senses? What owns decay? How do you unit test memory?
- **Hands-on Verification Task:** Implement the same scent evidence feeding a BT Blackboard and a StateTree context through one memory component.
- **Sources:** [SRC-AI-004], [SRC-AI-019]
- **Version Notes:** Stable architecture; API calls target-sensitive.

### Question: How do you debug a custom sense that never reports stimuli?

- **Category:** AI / Perception / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Prove each layer: listener config, source event, correct World, sense update, filter acceptance, stimulus report and memory projection.
- **Strong 3-Year-Engineer Answer:** I reduce to one listener and one source. Then I log event emission with World/time/source/location, confirm the custom sense has pending events, count listeners after range/team filters, prove a stimulus is reported, and only then inspect AI memory and decisions. I also check module dependencies and packaged target availability because editor-only success can hide missing runtime setup.
- **Common Weak Answer:** "Put logs in the Behaviour Tree."
- **Follow-up Questions:** Wrong World? Missing sense config? Source destroyed? Module dependency?
- **Hands-on Verification Task:** Create six no-stimulus failures and diagnose them from a blind trace.
- **Sources:** [SRC-AI-008], [SRC-AI-009], [SRC-AI-016], [SRC-AI-019]
- **Version Notes:** Debug UI and API names vary by UE5 minor version.

### Question: How do you profile a custom AI sense?

- **Category:** AI / Performance
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Count events, discarded/merged events, listeners considered, expensive checks, stimuli reported, callbacks, memory writes, allocations and game-thread time.
- **Strong 3-Year-Engineer Answer:** I start at the producer. If 100 agents emit 50 events/sec, callback optimisation is not the first fix. I measure events emitted, accepted, merged, expired, listeners after coarse filters, traces/path queries, stimuli reported, memory updates and timing. Then I reduce event production, batch similar events, prefilter by range/team/tag, cap/stagger traces and add significance tiers.
- **Common Weak Answer:** "Move it to C++ and lower update frequency."
- **Follow-up Questions:** What if traces dominate? What if callbacks dominate? What if memory writes cause BT churn?
- **Hands-on Verification Task:** Run 1/20/100 agents with increasing event rates and compare three filtering strategies.
- **Sources:** [SRC-AI-004], [SRC-PERF-003], [SRC-PERF-004]
- **Version Notes:** Workload and platform dependent.

### Question: What does MassAI mean in a practical Unreal architecture?

- **Category:** MassEntity / AI Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** It means using MassEntity data/processors for crowd or agent simulation and bridging to classic AI/gameplay only where rich behaviour requires it.
- **Strong 3-Year-Engineer Answer:** I do not treat MassAI as "BT but faster". Mass stores durable agent state in fragments, processors run batch movement/activity/LOD, and representation may be Actor, ISM or none. StateTree, ZoneGraph and Smart Objects can participate, but exact integration is plugin/version-sensitive. I keep rich unique agents as Actors unless scale/access patterns justify Mass or a hybrid promotion path.
- **Common Weak Answer:** "MassAI is Unreal's ECS AI system for crowds."
- **Follow-up Questions:** What remains Actor-based? How do fragments feed StateTree? What should be profiled?
- **Hands-on Verification Task:** Explain which parts of Project 2 should stay Actor AI and which parts could move to Project 5 Mass.
- **Sources:** [SRC-MASS-001], [SRC-MASS-002], [SRC-MASS-009], [SRC-MASS-010]
- **Version Notes:** UE5 plugin/version-sensitive; verify exact modules/source.

### Question: ZoneGraph versus NavMesh for AI crowds?

- **Category:** MassEntity / AI Navigation
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** NavMesh supports general traversability/pathfinding; ZoneGraph-style lane data supports structured crowd/traffic movement through authored lanes and zones.
- **Strong 3-Year-Engineer Answer:** For combat AI, arbitrary NavMesh pathing and EQS may be appropriate. For pedestrians or traffic, agents often need lane following, next segment, local avoidance and bottleneck handling, not a fresh full path query every decision tick. ZoneGraph/lane data can provide structured movement substrate for Mass crowds, while avoidance and Smart Object reservations solve different local problems.
- **Common Weak Answer:** "ZoneGraph is a faster NavMesh for Mass."
- **Follow-up Questions:** Doorway congestion? Lane closure? When still need NavMesh? How does avoidance fit?
- **Hands-on Verification Task:** Build one pedestrian route with a blocked lane and compare lane failure to local avoidance failure.
- **Sources:** [SRC-AI-006], [SRC-MASS-008], [SRC-MASS-009], [SRC-MASS-010]
- **Version Notes:** ZoneGraph plugin/tooling and APIs are target-sensitive.

### Question: How should StateTree be used with Mass entities?

- **Category:** MassEntity / StateTree
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use it for shared state/activity flow over entity context/fragments, not arbitrary per-entity Actor-style calls that erase Mass batching.
- **Strong 3-Year-Engineer Answer:** A Mass StateTree-like flow can choose activities, wait for movement, react to signals and set fragments. Durable activity, target and claim state should live in fragments or equivalent Mass data. If every task calls Actor components and UObjects per entity each tick, the architecture is no longer data-oriented. I verify target plugin/source because Mass StateTree integration changes across UE5 versions.
- **Common Weak Answer:** "Mass StateTree lets you run StateTrees on thousands of entities."
- **Follow-up Questions:** Where store claim handle? What is a signal? What makes it batch-friendly?
- **Hands-on Verification Task:** Build `Find -> Reserve -> Reach -> Use -> Exit` with claim state stored outside task locals.
- **Sources:** [SRC-AI-010], [SRC-MASS-001], [SRC-MASS-002]
- **Version Notes:** Mass StateTree integration is plugin/branch-sensitive.

### Question: How do Smart Objects fit into Mass crowds?

- **Category:** MassEntity / Smart Objects
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** They provide a claim/use/release concurrency boundary for specific world affordances, while Mass owns scalable activity and representation state.
- **Strong 3-Year-Engineer Answer:** A Mass entity can choose an activity and seek a compatible affordance, but "candidate found" is not "slot owned". The authoritative Smart Object claim reserves the slot, and the entity stores claim/activity state durably. Nearby agents may promote to Actors for animation/collision, while distant agents may use lightweight representation. Every abort, despawn and demotion path must release the claim.
- **Common Weak Answer:** "Mass entities use Smart Objects like normal AI does."
- **Follow-up Questions:** What if representation tier has no Actor? What if claim succeeds but route fails? How prevent leaked slots?
- **Hands-on Verification Task:** Run 20 Mass entities competing for five slots and prove no leaked or duplicate claims.
- **Sources:** [SRC-AI-011], [SRC-AI-012], [SRC-MASS-002], [SRC-MASS-010]
- **Version Notes:** Smart Object/Mass integration is target-branch sensitive.

### Question: What is the Actor promotion contract in a MassAI system?

- **Category:** MassEntity / Hybrid Actors
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Define stable mapping, state transfer, authoritative writer, reset/invalidation, network audience and hysteresis when an entity gains or loses Actor representation.
- **Strong 3-Year-Engineer Answer:** Promotion is not just spawning an Actor. I define entity ID to Actor mapping, copy state into the Actor, choose whether Mass or Actor owns transform, suspend conflicting processors if needed, copy final state back, reset pooled Actors and invalidate external references on demotion. Without this, CharacterMovement, Mass movement and replication can all authoritatively fight.
- **Common Weak Answer:** "Promote important Mass entities to Actors near the player."
- **Follow-up Questions:** Who owns transform? What happens to references on demotion? How prevent LOD thrash?
- **Hands-on Verification Task:** Inject a dual-writer transform bug and fix it by explicit promotion policy.
- **Sources:** [SRC-MASS-002], [SRC-MASS-005]
- **Version Notes:** Representation APIs and pooling differ across UE5 versions.

### Question: How do you debug a Mass crowd jam?

- **Category:** MassEntity / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Separate route/lane choice, local avoidance, movement integration, Smart Object reservation, collision/animation and representation.
- **Strong 3-Year-Engineer Answer:** I inspect lane/route fragments, desired velocity, avoidance neighbour data and processor execution first. Then I check whether a Smart Object/door/slot reservation blocks exit, and whether promoted Actor collision fights the Mass movement result. I compare no avoidance, avoidance and reduced-density scenarios, and use available Mass/Visual Logger tooling before changing lane data or processor code.
- **Common Weak Answer:** "Increase avoidance radius or reduce crowd density."
- **Follow-up Questions:** Lane graph versus avoidance? Actor collision? Processor order? Door reservation?
- **Hands-on Verification Task:** Create a bottleneck jam and write a layer-by-layer diagnosis table.
- **Sources:** [SRC-MASS-007], [SRC-MASS-008], [SRC-MASS-009], [SRC-MASS-010]
- **Version Notes:** Mass debugger and crowd tooling vary by branch.

### Question: How do you profile MassAI fairly?

- **Category:** MassEntity / Performance
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Compare equivalent behaviour/visual fidelity and split simulation, structural, bridge, representation, StateTree/Smart Object and rendering costs.
- **Strong 3-Year-Engineer Answer:** I record entity counts, processors, per-entity time, archetypes/chunk occupancy, structural command volume, lane density, StateTree tasks/transitions/signals, Smart Object query/claim attempts, Actor promotions, Actor/ISM/no representation counts, Game/Draw/GPU and memory. I avoid claiming Mass is faster if the Mass version removed skeletal animation, collision or replication.
- **Common Weak Answer:** "Spawn the same number of Actors and Mass entities and compare FPS."
- **Follow-up Questions:** What if representation dominates? What if structural churn dominates? What if StateTree dominates?
- **Hands-on Verification Task:** Produce a fair and an intentionally unfair Actor/MassAI comparison and explain the difference.
- **Sources:** [SRC-MASS-001], [SRC-MASS-002], [SRC-MASS-007], [SRC-PERF-003]
- **Version Notes:** Workload, target hardware and branch dependent.

## First-Pass Specialist Expansion Set

### Question: Why can an Animation Notify be a bad authority boundary?

- **Category:** Animation / Networking / Gameplay
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Notifies are presentation/timing signals from animation evaluation; durable gameplay authority should validate state, timing, ownership and network role separately.
- **Strong 3-Year-Engineer Answer:** I use notifies to open windows, trigger cosmetics or report timing, but I do not let a client-side notify directly apply damage or inventory state. Montages can be interrupted, skipped by relevance/LOD, differ by client, or fail to play. For melee, the server owns the attack state and validation window; the notify can request/check a trace only if the authority path confirms ability state, montage section/window, target validity and anti-duplicate policy.
- **Common Weak Answer:** "Use an Anim Notify at the hit frame to apply damage."
- **Follow-up Questions:** What if montage is interrupted? What if notify runs only on owning client? How do dedicated servers handle animation?
- **Hands-on Verification Task:** Implement a melee window where client notifies only play cosmetics and the server validates the hit window separately.
- **Sources:** [SRC-ANIM-008], [SRC-ANIM-010], [SRC-NET-001]
- **Version Notes:** Montage/notify dispatch and animation relevance are content/branch-sensitive.

### Question: How do you debug a montage that never exits an attack state?

- **Category:** Animation / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Inspect montage play/section/slot/group, blend-out/interruption, notify/state callbacks, gameplay task cancellation and the state variable owner.
- **Strong 3-Year-Engineer Answer:** I first prove whether the montage actually started on the expected skeletal mesh/AnimInstance/slot. Then I inspect section transitions, blend out, interruption, montage end delegates, notify state end, ability/task cancellation and any replicated state that should clear attack. I add one idempotent cleanup path for normal end, interruption, death, stun, unequip and mesh destruction so gameplay is not waiting for one notify that may never arrive.
- **Common Weak Answer:** "The notify did not fire; add another notify."
- **Follow-up Questions:** Slot mismatch? Montage group? Interrupted blend out? Mesh swap? Death mid-attack?
- **Hands-on Verification Task:** Inject interrupted montage, mesh destroy and stun cases and prove attack state clears exactly once.
- **Sources:** [SRC-ANIM-008], [SRC-ANIM-010], [SRC-ANIM-016]
- **Version Notes:** Exact AnimInstance delegate names and debugger views vary by branch.

### Question: Why can root motion create movement and network bugs?

- **Category:** Animation / Movement / Networking
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Root motion moves the character from animation data, so it must be coordinated with CharacterMovement, collision, authority, prediction/correction and montage state.
- **Strong 3-Year-Engineer Answer:** I identify who owns capsule movement each frame. If CharacterMovement and root motion both move the capsule, foot sliding or correction can appear. In multiplayer I check whether root motion is authoritative, replicated/predicted appropriately, interruptible, and compatible with movement mode, floor/collision and montage section. I would build a latency/loss test before choosing root-motion attacks for competitive movement.
- **Common Weak Answer:** "Root motion makes animation look better but is harder online."
- **Follow-up Questions:** In-place alternative? Server authority? Correction symptoms? Motion Warping?
- **Hands-on Verification Task:** Compare in-place, root-motion and scripted movement dash under packet lag and record corrections.
- **Sources:** [SRC-ANIM-009], [SRC-NET-009], [SRC-PHYS-002]
- **Version Notes:** Root motion modes/network handling are branch and project-policy sensitive.

### Question: Why might a blocking collision not produce a Hit event?

- **Category:** Physics / Collision / Debugging
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Blocking response, physics simulation, sweep path, movement API, notification flags and component setup are separate requirements.
- **Strong 3-Year-Engineer Answer:** I inspect collision enabled mode, object/trace responses, both components' profiles, whether movement used sweep or physics simulation, whether hit generation/notification is enabled, whether simple/complex geometry participates and whether the event should be on the moved component, hit component or actor. A Block response means the query/simulation can block; it does not guarantee an exactly-once gameplay event.
- **Common Weak Answer:** "Turn on Generate Hit Events."
- **Follow-up Questions:** Sweep versus teleport? Query-only versus physics? Simple versus complex? Moved component?
- **Hands-on Verification Task:** Build a collision matrix where Block succeeds with and without Hit events, then explain each row.
- **Sources:** [SRC-PHYS-001], [SRC-PHYS-002], [SRC-PHYS-003]
- **Version Notes:** Component notification details need target tests.

### Question: Sweep movement versus physics simulation?

- **Category:** Physics / Movement
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** A sweep is an explicit scene query/move test; physics simulation is solver-driven rigid-body integration with contacts, forces and constraints.
- **Strong 3-Year-Engineer Answer:** `SetActorLocation` with sweep is a kinematic movement/query path. Chaos simulation integrates velocity/forces and resolves contacts over solver steps. They can both interact with collision, but they have different ownership, event timing and networking assumptions. I avoid mixing kinematic teleports, CharacterMovement and simulated physics on one object without one clear authoritative movement owner.
- **Common Weak Answer:** "Sweep is physics movement with collision."
- **Follow-up Questions:** Teleport? CCD? character capsule? server authority? substeps?
- **Hands-on Verification Task:** Move one crate by sweep, one by force and one by teleport; compare Hit/Overlap/contact behaviour.
- **Sources:** [SRC-PHYS-002], [SRC-PHYS-004], [SRC-PHYS-005]
- **Version Notes:** Chaos solver details and callbacks vary by version/settings.

### Question: Why can UUserWidget Construct be the wrong place for one-time setup?

- **Category:** UI / UMG / Lifecycle
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Construct can run when the underlying Slate widget is rebuilt; true once-per-widget-object setup belongs in an appropriate one-time lifecycle hook.
- **Strong 3-Year-Engineer Answer:** I separate UObject widget lifetime from Slate tree lifetime. Construct is useful for binding to child widgets after construction, but if I bind external delegates, start async requests or register global callbacks there without idempotent cleanup, rebuilds can duplicate work. I use the correct once-created hook for durable setup and release subscriptions at the matching teardown/removal boundary.
- **Common Weak Answer:** "Construct is BeginPlay for widgets."
- **Follow-up Questions:** OnInitialized? NativeDestruct? CommonUI activation? ListView recycled entries?
- **Hands-on Verification Task:** Force widget removal/re-add/rebuild and prove delegate subscription count stays one.
- **Sources:** [SRC-UI-002], [SRC-UI-005]
- **Version Notes:** Exact lifecycle hooks and CommonUI/MVVM integration are version/plugin sensitive.

### Question: How do virtualised ListView entries cause stale UI bugs?

- **Category:** UI / Lists / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Entry widgets are recycled presentation objects; item data identity must not be stored as permanent widget-local state without reset/rebind policy.
- **Strong 3-Year-Engineer Answer:** I distinguish item object/data from generated entry widget. A row widget can be released and reused for a different item, so selection, async thumbnail handles, delegates and cached text/images must bind and unbind per item assignment/release. Stale UI usually means an entry retained previous item state or an async callback updated a recycled widget.
- **Common Weak Answer:** "ListView is bugged because rows reuse widgets."
- **Follow-up Questions:** Item identity? OnEntryReleased? Async thumbnail? selection state?
- **Hands-on Verification Task:** Build a virtualised inventory list with async icons and prove recycled rows do not show stale icons.
- **Sources:** [SRC-UI-013], [SRC-UI-014], [SRC-ASSET-004]
- **Version Notes:** Python API source is conceptual; use target C++/Blueprint APIs for implementation.

### Question: Retainer Panel versus Invalidation Box?

- **Category:** UI / Performance
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Invalidation reduces widget layout/paint work when UI is unchanged; Retainer renders UI to a texture at controlled cadence but adds render-target/GPU/memory trade-offs.
- **Strong 3-Year-Engineer Answer:** I first classify whether the cost is update, layout, paint, draw submission, render target or overdraw. Invalidation helps stable UI avoid repeated work, but volatile/dynamic widgets can invalidate constantly. Retainers can lower update frequency or composite UI, but they add texture memory, latency, capture cost and overdraw. I use Slate Insights and target GPU data before adding panels.
- **Common Weak Answer:** "Use Retainer Panels to optimise UMG."
- **Follow-up Questions:** What is volatility? What invalidates? Render target memory? Input latency?
- **Hands-on Verification Task:** Profile a dynamic HUD with no optimisation, invalidation and Retainer; identify which lane changed.
- **Sources:** [SRC-UI-004], [SRC-UI-005], [SRC-RENDER-016]
- **Version Notes:** UMG optimisation docs may target later UE5; verify project branch.

### Question: How does audio concurrency stealing become a gameplay bug?

- **Category:** Audio / Gameplay / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** If gameplay relies on hearing a sound that can be culled/stolen/virtualised, audio voice policy can hide required feedback.
- **Strong 3-Year-Engineer Answer:** Audio concurrency is a presentation resource policy. If a reload cue, warning beep or parry signal is gameplay-critical, I ensure the gameplay state is visible through UI/animation/haptics or reserves priority appropriately. I inspect concurrency group, owner scoping, resolution rule, priority, retrigger time, attenuation and platform voice limits. I do not replicate or gate gameplay success on whether an audio component actually rendered a voice.
- **Common Weak Answer:** "Raise the sound's priority."
- **Follow-up Questions:** Owner scope? voice steal rule? UI fallback? network duplication?
- **Hands-on Verification Task:** Create 40 overlapping sounds and prove critical warnings remain perceivable or have non-audio fallback.
- **Sources:** [SRC-AUDIO-003], [SRC-AUDIO-004], [SRC-AUDIO-010]
- **Version Notes:** Audio settings and platform voice limits are target-sensitive.

### Question: Why can MetaSounds be performance-sensitive?

- **Category:** Audio / MetaSounds / Performance
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Procedural/sample-accurate graphs still consume CPU/DSP/voice resources and can multiply cost through polyphony and parameter churn.
- **Strong 3-Year-Engineer Answer:** I treat a MetaSound graph like runtime audio code. I profile voice count, graph complexity, parameter update frequency, source lifetime, concurrency, submix routing and platform buffers. Procedural generation avoids some asset constraints but does not make unlimited polyphony free. I bound spawn rates, reuse components where appropriate and compare the result on target hardware.
- **Common Weak Answer:** "MetaSounds are generated audio so memory is lower and performance is fine."
- **Follow-up Questions:** Polyphony? parameter churn? submix effects? stream cache?
- **Hands-on Verification Task:** Build a procedural weapon loop and compare 1/20/100 simultaneous voices.
- **Sources:** [SRC-AUDIO-006], [SRC-AUDIO-007], [SRC-PERF-003]
- **Version Notes:** MetaSound nodes and performance characteristics evolve across UE5.

### Question: Why should Niagara events not drive authoritative damage?

- **Category:** Niagara / Gameplay / Networking
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Niagara is presentation/effects simulation; authoritative gameplay collision, damage and replication should live in gameplay systems.
- **Strong 3-Year-Engineer Answer:** Niagara can visualise impacts, consume gameplay data and sometimes produce useful local effect events, but GPU simulation/readback/culling/scalability and client presentation make it a poor source of durable authority. I use gameplay collision/traces/abilities to decide damage, replicate bounded gameplay events, then feed Niagara through parameters/data channels for visuals.
- **Common Weak Answer:** "Use Niagara collision events for projectile damage."
- **Follow-up Questions:** GPU readback? culling? client authority? replay? pooling?
- **Hands-on Verification Task:** Implement projectile damage through gameplay collision and use Niagara only for impact visuals; then disable VFX and prove damage still works.
- **Sources:** [SRC-FX-003], [SRC-FX-005], [SRC-FX-007], [SRC-NET-001]
- **Version Notes:** Niagara event/readback features are version and platform sensitive.

### Question: What state must be reset when pooling Niagara or audio components?

- **Category:** VFX / Audio / Lifetime
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Reset parameters, attachments, transforms, owner/context, delegates/callbacks, activation state, pooling handles and any gameplay-facing references.
- **Strong 3-Year-Engineer Answer:** Pooling trades allocation/spawn cost for reset complexity and retained memory. A pooled effect or sound can leak previous owner, attach parent, parameter values, gameplay tag, callback, concurrency owner, transform or active state. I define a reset contract and test repeated acquire/release with different owners. Gameplay should not depend on the pooled component staying alive unless ownership is explicit.
- **Common Weak Answer:** "Deactivate and reuse the component."
- **Follow-up Questions:** Attached parent? delegates? auto-destroy? owner scoped concurrency? parameters?
- **Hands-on Verification Task:** Pool an impact effect/sound and prove no previous owner's colour, callback or attachment leaks.
- **Sources:** [SRC-FX-006], [SRC-FX-005], [SRC-AUDIO-002]
- **Version Notes:** Pooling APIs and auto-destroy behaviour are version-sensitive.

### Question: Why is transforming normals under non-uniform scale tricky?

- **Category:** Game Maths / Transforms
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Normals are directions perpendicular to surfaces; under non-uniform scale they generally require inverse-transpose-style handling, not the same transform as points.
- **Strong 3-Year-Engineer Answer:** Points, vectors and normals have different transform semantics. Translation affects points but not directions. Non-uniform scale can make a simply transformed normal no longer perpendicular to the transformed surface. In Unreal I use the correct API/helper for the data type and test with known non-uniform-scale cases rather than assuming `TransformVector` works for normals.
- **Common Weak Answer:** "Normalise the transformed vector."
- **Follow-up Questions:** Point versus vector? inverse transform? tangent basis? normal maps?
- **Hands-on Verification Task:** Transform a tilted triangle under non-uniform scale and compare geometric face normal against naive transformed normal.
- **Sources:** [SRC-MATH-004], [SRC-MATH-005], [SRC-MATH-010]
- **Version Notes:** Exact helper names/types must be verified in target branch.

### Question: When can a spatial hash perform badly?

- **Category:** Algorithms / Spatial Structures
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Bad cell size, clustered distributions, high update churn, duplicate boundary insertion and poor query radius assumptions can erase expected benefits.
- **Strong 3-Year-Engineer Answer:** I choose cell size from object size and query radius, then test clustered and adversarial distributions, not only uniform random data. I track cell occupancy, candidates per query, duplicate insertions, update/move cost and memory. A spatial hash is excellent when locality assumptions hold; with extreme clustering or huge query radii it can approach all-pairs plus overhead.
- **Common Weak Answer:** "Spatial hash makes neighbour queries O(1)."
- **Follow-up Questions:** Cell size? boundary overlap? moving objects? worst case? validation oracle?
- **Hands-on Verification Task:** Compare all-pairs and spatial hash across uniform, clustered and line-distributed points.
- **Sources:** [SRC-ALG-012], [SRC-ALG-013], [SRC-PERF-003]
- **Version Notes:** Algorithmic; implementation depends on distribution and workload.

### Question: How do Lua or C# wrappers become stale around UObjects?

- **Category:** Scripting / Lifetime
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Embedded/managed runtimes can hold wrappers after the underlying UObject is destroyed, collected, unloaded or otherwise invalid for that call.
- **Strong 3-Year-Engineer Answer:** Lua/.NET GC and Unreal GC trace different roots. A wrapper must not imply the UObject is alive or safe. I use weak/validated object handles, explicit invalidation on destruction/world teardown, deterministic callback unregistration and tests for level unload, hot reload, PIE restart and async callbacks. The wrapper releases binding resources; it does not arbitrarily delete UObject memory.
- **Common Weak Answer:** "Use weak references."
- **Follow-up Questions:** Who owns native object? callback after unload? GCHandle? plugin support?
- **Hands-on Verification Task:** Hold a script reference to an Actor, unload/destroy it and prove calls fail safely.
- **Sources:** [SRC-SCRIPT-001], [SRC-SCRIPT-006], [SRC-SCRIPT-007], [SRC-SCRIPT-008]
- **Version Notes:** Plugin/runtime specific; Unreal baseline does not include Lua/C# gameplay runtime.

## Core Systems Specialist Expansion Set

### Question: When is a raw UObject pointer unsafe even if it currently works?

- **Category:** UObject / Lifetime
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** It is unsafe when the reference is not discoverable/validated across GC, async work, destruction, level unload, delayed callbacks or ownership changes.
- **Strong 3-Year-Engineer Answer:** A raw pointer can be a local non-owning observation, but I need to know what keeps the object alive and what invalidates the pointer. For member references I usually need reflected strong/weak/soft/handle semantics. For delayed callbacks, timers, async tasks and delegates, I validate on use and unbind/clear at the right lifetime boundary. "It was valid when assigned" is not a lifetime policy.
- **Common Weak Answer:** "Use UPROPERTY for anything you want to keep alive."
- **Follow-up Questions:** TObjectPtr versus TWeakObjectPtr? soft pointer? lambda capture? world unload? automatic nulling?
- **Hands-on Verification Task:** Store a raw pointer in a delayed lambda, destroy/unload the object, then repair with weak validation and explicit unbinding.
- **Sources:** [SRC-EPIC-006], [SRC-EPIC-009], [SRC-EPIC-005]
- **Version Notes:** UE5 pointer recommendations and incremental GC barriers are target-version sensitive.

### Question: How do you choose between `TObjectPtr`, `TWeakObjectPtr` and `TSoftObjectPtr`?

- **Category:** UObject / Pointer Selection
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Choose from ownership/reference strength, load state, lifetime domain and whether the pointer should keep an object reachable.
- **Strong 3-Year-Engineer Answer:** `TObjectPtr` is for reflected UObject references that should participate in the reachable object graph and UE5 pointer tracking. `TWeakObjectPtr` observes an object without keeping it alive and must be validity-checked. `TSoftObjectPtr` references an asset/object path that may be unloaded and needs load/cook policy. I do not use soft pointers as a general null-safe pointer, and I do not use weak pointers where the system needs ownership.
- **Common Weak Answer:** "`TObjectPtr` replaces raw UObject pointers and soft pointers are lazy weak pointers."
- **Follow-up Questions:** What keeps object alive? cook inclusion? async load? GC nulling? editor-only reference?
- **Hands-on Verification Task:** Implement one field of each type and predict behaviour after GC, unload and packaged cook.
- **Sources:** [SRC-EPIC-009], [SRC-ASSET-002], [SRC-ASSET-006]
- **Version Notes:** Exact UE5 pointer guidance should be checked against target branch.

### Question: Constructor, OnConstruction, PostInitializeComponents or BeginPlay?

- **Category:** Gameplay Framework / Lifecycle
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Use constructor for defaults/default subobjects, construction for editor/instance setup, component initialisation hooks for registered components and BeginPlay for gameplay start.
- **Strong 3-Year-Engineer Answer:** I avoid world-dependent gameplay in constructors because they also run for CDO/defaults and may run before a valid World or registered components. Construction can rerun in editor and must be idempotent. Component initialisation hooks fit setup requiring components to exist/register. BeginPlay starts gameplay once the Actor participates in play. Exact ordering differs by load/spawn/deferred paths, so I test the target branch for hook-sensitive systems.
- **Common Weak Answer:** "Constructor creates components; BeginPlay does runtime stuff."
- **Follow-up Questions:** CDO? placed versus spawned? deferred spawn? replicated initial state? editor rerun?
- **Hands-on Verification Task:** Log placed, duplicated, spawned and deferred-spawned Actors through each hook and identify one bad side effect.
- **Sources:** [SRC-EPIC-015], [SRC-EPIC-006], [SRC-EPIC-010]
- **Version Notes:** `SRC-EPIC-015` is UE4-era; exact UE5 ordering requires branch/source verification.

### Question: How can a Subsystem become a hidden Service Locator problem?

- **Category:** Gameplay Architecture / Subsystems
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** A scoped Subsystem still hides dependencies when arbitrary code pulls global services instead of receiving explicit interfaces or context.
- **Strong 3-Year-Engineer Answer:** Subsystems are useful lifetime hosts: World, GameInstance, LocalPlayer, Engine or Editor. They become a problem when every gameplay object calls `GetSubsystem` for unrelated services, making dependencies invisible, hard to test and unsafe across multiple Worlds/PIE/local players. I keep host scope narrow, expose small interfaces and pass dependencies into core logic where explicit ownership matters.
- **Common Weak Answer:** "Subsystems are Unreal's dependency injection."
- **Follow-up Questions:** World versus GameInstance? local player? replication? testability? hidden order dependency?
- **Hands-on Verification Task:** Refactor a globally pulled inventory service into an explicit interface used by a component and unit-style test.
- **Sources:** [SRC-EPIC-017], [SRC-ARCH-007]
- **Version Notes:** Subsystem host classes are stable; project architecture determines dependency policy.

### Question: Why is a `BlueprintPure` function with hidden work dangerous?

- **Category:** C++ / Blueprint / API Design
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Pure nodes look side-effect-free and may be evaluated more often than expected, so hidden loads, allocations, mutation or expensive queries cause invisible bugs and performance cost.
- **Strong 3-Year-Engineer Answer:** A pure Blueprint API should behave like a cheap query from the user's point of view. If it loads assets, changes state, allocates heavily, performs traces or depends on call order, graph authors will accidentally create repeated work or nondeterministic behaviour. I make side effects explicit with callable commands, cache results deliberately and profile Blueprint call sites rather than assuming purity is free.
- **Common Weak Answer:** "Pure just means no execution pin."
- **Follow-up Questions:** Hidden asset load? const C++? caching? expensive getter? Blueprint VM cost?
- **Hands-on Verification Task:** Convert an expensive pure getter into an explicit cached command/query pair and compare call counts.
- **Sources:** [SRC-EPIC-031], [SRC-EPIC-034], [SRC-EPIC-036]
- **Version Notes:** Blueprint compiler/evaluation details are target-sensitive; API contract principle is stable.

### Question: When should a delegate be dynamic rather than native?

- **Category:** UE C++ / Delegates / Blueprint
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Use dynamic delegates when reflection/Blueprint/serialisation needs them; prefer native delegates for pure C++ hot paths and stronger type/performance needs.
- **Strong 3-Year-Engineer Answer:** Dynamic delegates are reflected and useful for Blueprint assignment, serialisation and editor-facing events, but they cost more and have reflection constraints. Native delegates fit C++-only events and hot paths. The bigger issue is lifetime: bind/unbind ownership, weak captures, duplicate bindings and re-entrancy. I choose delegate type from audience and lifetime, not habit.
- **Common Weak Answer:** "Dynamic delegates are for Blueprint and native delegates are faster."
- **Follow-up Questions:** multicast? sparse? weak lambda? duplicate binding? thread? serialization?
- **Hands-on Verification Task:** Implement the same health event as native C++ and BlueprintAssignable dynamic multicast, then document lifetime and cost trade-offs.
- **Sources:** [SRC-EPIC-028], [SRC-EPIC-029], [SRC-EPIC-031]
- **Version Notes:** Exact delegate helpers and binding APIs are branch-sensitive.

### Question: How do async Blueprint action nodes leak or call back into dead objects?

- **Category:** C++ / Blueprint / Async
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Proxy objects, delegates, timers, latent work and captured UObjects can outlive the caller/world unless cancellation and validation are explicit.
- **Strong 3-Year-Engineer Answer:** An async action proxy needs a clear owner/world context, activation path, completion/cancel path and cleanup. I store weak references to callers/targets unless the proxy intentionally owns them, unbind external delegates, cancel timers/requests, handle world teardown and guarantee exactly-once completion/cancellation. I test level unload, PIE stop and caller destruction, not just happy-path completion.
- **Common Weak Answer:** "Make the proxy UPROPERTY so GC keeps it alive."
- **Follow-up Questions:** Who owns proxy? world context? cancellation? duplicate completion? latent request handle?
- **Hands-on Verification Task:** Build an async load/progress Blueprint node, destroy the caller mid-request and prove no stale callback occurs.
- **Sources:** [SRC-EPIC-044], [SRC-EPIC-009], [SRC-ASSET-002]
- **Version Notes:** Proxy API and metadata are target-sensitive.

### Question: Why should Data Assets usually be treated as definitions rather than mutable runtime state?

- **Category:** Gameplay Architecture / Data Assets
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Data Assets are shared authored definitions; mutating them as per-instance runtime state leaks changes across users, saves, PIE/editor and asset references.
- **Strong 3-Year-Engineer Answer:** I use Data Assets for item/ability/enemy definitions and copy needed mutable state into runtime instances, components or save records. If I mutate the definition, every user of that asset can see the change, and editor/PIE/package behaviour can become confusing. I validate definitions, keep runtime state separate and define migration/versioning for asset identity changes.
- **Common Weak Answer:** "Use Data Assets instead of hardcoded values."
- **Follow-up Questions:** PrimaryDataAsset? runtime instance? save/load? designer edits? validation?
- **Hands-on Verification Task:** Build an item definition asset and an inventory instance; prove durability survives editing the definition after save.
- **Sources:** [SRC-ARCH-009], [SRC-ARCH-012], [SRC-ASSET-006]
- **Version Notes:** Asset Manager rules and validation setup are project-sensitive.

### Question: How does `FAssetData::GetAsset` accidentally turn discovery into loading?

- **Category:** Assets / Asset Registry / Performance
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Asset Registry metadata queries can stay unloaded, but calling `GetAsset` resolves the actual UObject and may synchronously load it.
- **Strong 3-Year-Engineer Answer:** I use Asset Registry to discover unloaded asset metadata by path/class/tags. If I then call `GetAsset` inside a scan or UI list, I may force asset loads, hitches and dependency closure. For large inventories or tools, I display metadata from `FAssetData`, use soft references/Asset Manager for explicit async loads, and profile discovery separately from loading.
- **Common Weak Answer:** "Asset Registry lets you find assets without loading them."
- **Follow-up Questions:** async scan complete? tags? thumbnail? soft path? editor versus packaged?
- **Hands-on Verification Task:** Query 100 item assets and compare metadata-only list versus `GetAsset` loading with Insights/asset load logs.
- **Sources:** [SRC-ASSET-004], [SRC-ASSET-002], [SRC-PERF-003]
- **Version Notes:** Registry fields and package discovery are branch/build sensitive.

### Question: How do Primary Asset bundles become accidental memory bombs?

- **Category:** Assets / Asset Manager
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Bad bundle boundaries can load UI, gameplay, audio, VFX and large dependencies together when only one purpose-specific subset is needed.
- **Strong 3-Year-Engineer Answer:** Bundles should express why content is needed: UI icon, gameplay mesh, preview, audio, cinematic, etc. If every soft dependency sits in one default bundle, opening an inventory can load combat meshes and VFX. I audit bundle contents, dependency chains, cook/chunk rules and runtime residency. A Primary Asset ID is only useful if its management rules match product loading boundaries.
- **Common Weak Answer:** "Put soft references into bundles so Asset Manager can load them."
- **Follow-up Questions:** Secondary assets? recursive load? chunking? unload policy? editor loaded state?
- **Hands-on Verification Task:** Create `UI` and `Gameplay` bundles for item assets and prove inventory UI does not load gameplay meshes.
- **Sources:** [SRC-ASSET-001], [SRC-ASSET-002], [SRC-ASSET-006]
- **Version Notes:** Asset Manager configuration is project and branch sensitive.

### Question: Why can Live Coding hide structural bugs?

- **Category:** Build / Iteration / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Live Coding patches code into a running editor; it does not prove clean startup, reflection/layout migration, serialisation, constructor defaults or packaged correctness.
- **Strong 3-Year-Engineer Answer:** Live Coding is excellent for iteration, but after changing reflected layout, constructors, defaults, module dependencies, serialised fields or lifecycle-sensitive code, I restart/rebuild and run a clean test. Reinstancing can preserve old state or mask missing constructor/default propagation. I do not accept "works after Live Coding" as release evidence.
- **Common Weak Answer:** "If Live Coding compiles, the change is fine."
- **Follow-up Questions:** UHT? CDO defaults? serialized assets? module dependencies? hot reload difference?
- **Hands-on Verification Task:** Change a reflected property/default and compare Live Coding behaviour to clean editor restart and packaged build.
- **Sources:** [SRC-BUILD-006], [SRC-BUILD-001], [SRC-EPIC-006]
- **Version Notes:** Live Coding behaviour and reinstancing support are target-version sensitive.

### Question: What makes save/load a transaction rather than a dump?

- **Category:** Gameplay Architecture / Save Systems
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Save/load needs stable identity, schema/versioning, dependency ordering, validation, partial failure policy and explicit reconstruction rules.
- **Strong 3-Year-Engineer Answer:** I do not just serialise current Actor fields. I define stable IDs for durable objects, save only authored/runtime state that should persist, version the schema, handle missing/renamed assets, rebuild transient references, validate after load and apply state in an order that respects dependencies. For multiplayer or streamed worlds, authority, loaded cells and late-join visibility also matter.
- **Common Weak Answer:** "Put SaveGame on variables and serialise the object."
- **Follow-up Questions:** stable IDs? asset rename? level streaming? transient fields? version migration?
- **Hands-on Verification Task:** Save an inventory/quest/world-object state, rename one asset, change schema version and load with a migration path.
- **Sources:** [SRC-EPIC-035], [SRC-ASSET-006], [SRC-WORLD-002]
- **Version Notes:** Save architecture is project-specific; specifiers only mark fields for consuming systems.

## GAS Specialist Expansion Set

### Question: How would you prove the correct ASC owner/avatar placement for a respawning character?

- **Category:** GAS / Lifetime / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Run a dedicated-server respawn test that logs owner, avatar, actor info, grants, old bindings and predicted activation before and after respawn.
- **Strong 3-Year-Engineer Answer:** I choose Pawn-owned or PlayerState-owned ASC from state lifetime, replication and ownership requirements, then prove it. The test must spawn, possess, grant abilities, activate a predicted ability, die, respawn with a new avatar and activate again under lag. I inspect whether attributes/cooldowns persist or reset by design, whether actor info is refreshed on authority and owning client, and whether old-avatar input/tasks/delegates are gone.
- **Common Weak Answer:** "Use PlayerState for players because it survives respawn."
- **Follow-up Questions:** What breaks with Mixed replication? What happens to old tasks? What state should reset on death?
- **Hands-on Verification Task:** Compare Pawn-owned and PlayerState-owned ASC in Project 9 and produce a lifetime decision table.
- **Sources:** [SRC-GAS-002], [SRC-GAS-010], [SRC-GAS-013]
- **Version Notes:** Actor-info hooks and replication details are branch/project-sensitive.

### Question: Why is a grant-source ledger safer than removing abilities by class?

- **Category:** GAS / Grants / Equipment
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** A class can be granted by multiple sources; removing by source/spec handle preserves exact ownership and avoids deleting the wrong grant.
- **Strong 3-Year-Engineer Answer:** Startup, equipment, buffs and progression can grant the same ability class with different levels or source objects. I record source ID, class, level and returned spec handle, then remove by handle when that source disappears. The ledger makes grant rebuild idempotent across possession, replication and loadout changes. Removing by class is fragile because it assumes a one-class-one-grant world that many real loadouts violate.
- **Common Weak Answer:** "Clear the ability class when the item is unequipped."
- **Follow-up Questions:** What if removal happens during activation? Can client grant? How avoid duplicate startup grants?
- **Hands-on Verification Task:** Grant the same ability from weapon and buff, then remove only the weapon grant.
- **Sources:** [SRC-GAS-003], [SRC-GAS-010]
- **Version Notes:** Exact spec and handle APIs require target-branch verification.

### Question: What should a GAS activation trace contain?

- **Category:** GAS / Observability
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Role, owner/avatar, ASC, ability class, spec handle, prediction key, target data, failure tags, commit/effect handles, task state and end reason.
- **Strong 3-Year-Engineer Answer:** I want one row that can correlate owner-client prediction with server authority. It should include net mode/role, owner/avatar, controller, ASC, ability class, spec handle, activation prediction key, activation mode, input/event source, target-data summary, CanActivate result/failure tags, commit result, cost/cooldown/applied effect handles, task state, cue path and end reason. Without these fields, teams guess across logs.
- **Common Weak Answer:** "Log when ActivateAbility starts and ends."
- **Follow-up Questions:** Why include prediction key? Why failure tags? How correlate cues and UI?
- **Hands-on Verification Task:** Add a shared trace schema to a predicted dash, effect application and UI update.
- **Sources:** [SRC-GAS-011], [SRC-GAS-013], [SRC-PERF-003]
- **Version Notes:** Logging implementation is project-specific; field concepts are stable.

### Question: How do you prove a Health attribute invariant is complete?

- **Category:** GAS / Attributes
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Test every mutation path: instant effects, max changes, periodic effects, predictions, replication and UI observation.
- **Strong 3-Year-Engineer Answer:** I define the invariant first, such as Health always in `[0, MaxHealth]` and MaxHealth reduction clamping Health without fake damage attribution. Then I test instant damage, healing, MaxHealth buffs/debuffs, periodic regeneration, predicted costs and replication. I compare server, owner client, simulated proxy and UI model. Clamping in one hook is not proof because base/current changes and executions can enter through different paths.
- **Common Weak Answer:** "Clamp Health in PostGameplayEffectExecute."
- **Follow-up Questions:** Base versus current? MaxHealth decrease? UI binding? predicted values?
- **Hands-on Verification Task:** Run an invariant matrix for Health/MaxHealth/Shield and document every mutation route.
- **Sources:** [SRC-GAS-004], [SRC-GAS-005]
- **Version Notes:** Attribute callback and replication helper details are branch-sensitive.

### Question: Direct modifier, magnitude calculation or execution calculation?

- **Category:** GAS / Effects / Calculations
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use the lightest mechanism that expresses the rule: direct for simple scalar changes, calculation for derived magnitude, execution for authoritative multi-attribute outcomes.
- **Strong 3-Year-Engineer Answer:** A direct modifier fits simple additive/multiplicative/override changes. A custom magnitude calculation fits one derived magnitude from captured attributes/tags. An execution calculation fits combat outcomes that emit several modifiers or route shield, armour, crit and health. I also state capture timing and validation of set-by-caller values. "Use execution for damage" is incomplete unless the rule actually needs that authority and complexity.
- **Common Weak Answer:** "Executions are for damage and modifiers are for buffs."
- **Follow-up Questions:** Snapshot versus live capture? set-by-caller? multiple outputs? prediction?
- **Hands-on Verification Task:** Implement the same simple damage as a modifier, then a shield/armour case as an execution-style calculation.
- **Sources:** [SRC-GAS-005], [SRC-GAS-009]
- **Version Notes:** GE authoring details changed across UE5; verify target branch.

### Question: What does capture timing mean in a Gameplay Effect design?

- **Category:** GAS / Effects / Captures
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Captured values may represent state at spec creation/application or later evaluation; the design must say which timing is intended.
- **Strong 3-Year-Engineer Answer:** If a projectile is fired while buffed and lands after the buff expires, attack power could intentionally use launch-time or impact-time state. GAS captures/specs make that timing a real design choice. I document the intended moment, test buff add/remove between launch and impact, and log captured values in the execution. Unspecified capture timing creates combat bugs that look random to designers.
- **Common Weak Answer:** "GAS captures the attribute when the effect runs."
- **Follow-up Questions:** Projectile travel? periodic effect? source tags? target tags?
- **Hands-on Verification Task:** Fire a projectile, add/remove attack and defence tags before impact, and record which values apply.
- **Sources:** [SRC-GAS-005], [SRC-GAS-009]
- **Version Notes:** Exact capture definitions and APIs require target-source verification.

### Question: What makes a stacking Gameplay Effect policy complete?

- **Category:** GAS / Stacking
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Source aggregation, max count, refresh, expiration, period reset, overflow, granted tags, UI attribution and persistence must all be specified.
- **Strong 3-Year-Engineer Answer:** "Stacks to five" is not a policy. I define whether stacks aggregate by source or target, the limit, duration refresh, whether expiration removes one or all, period reset, overflow behaviour, tag/cue state while stacks remain, UI source display and save/travel persistence. Debugging then follows active effect handle, source key, stack count, duration, period and tag count.
- **Common Weak Answer:** "Set stack count to five and refresh duration."
- **Follow-up Questions:** Remove one stack or all? source attribution? overflow? tag counts?
- **Hands-on Verification Task:** Implement poison and haste with different stacking policies and inject expiry/source bugs.
- **Sources:** [SRC-GAS-005], [SRC-GAS-006]
- **Version Notes:** GE stacking authoring UI/details are version-sensitive.

### Question: How do you debug a predicted GAS ability that snaps back?

- **Category:** GAS / Prediction / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Correlate client and server by prediction key, then inspect validation, commit, target data and non-reversible side effects.
- **Strong 3-Year-Engineer Answer:** I first verify the ability is truly Local Predicted and running on the owning client with a valid ASC/actor info. Then I correlate client and server activation by spec handle and prediction key. I inspect server validation, failure tags, cost/cooldown commit, target data and whether custom side effects happened outside supported GAS prediction. Snaps are acceptable on rejection only if resources, animation, cue and UI converge cleanly.
- **Common Weak Answer:** "Prediction is failing because the server corrected the client."
- **Follow-up Questions:** What side effects are reversible? What about movement? Why not trust listen server?
- **Hands-on Verification Task:** Force four rejection reasons for predicted dash and prove coherent cleanup.
- **Sources:** [SRC-GAS-011], [SRC-GAS-013], [SRC-GAS-003]
- **Version Notes:** Prediction support is operation-specific and branch-sensitive.

### Question: Why can cost or cooldown be applied twice in GAS?

- **Category:** GAS / Costs / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Duplicate commit paths, manual plus framework application, repeated input triggers, duplicate grants or prediction/server mismatch can apply the same policy twice.
- **Strong 3-Year-Engineer Answer:** I correlate activation by spec handle and prediction key, then inspect whether `CommitAbility` and manual cost/cooldown application both run, whether input pressed/held fires repeated activations, whether duplicate grants bind the same input, and whether predicted/server confirmation paths both create presentation or gameplay effects. The fix is usually one authoritative commit point plus idempotent input/grant policy, not a cooldown-specific hack.
- **Common Weak Answer:** "The input event is firing twice."
- **Follow-up Questions:** Manual GE application? duplicate spec? rejection refund? held trigger?
- **Hands-on Verification Task:** Inject manual cost application and duplicate grants, then repair with one commit path and grant ledger.
- **Sources:** [SRC-GAS-003], [SRC-GAS-005], [SRC-GAS-010], [SRC-GAS-013]
- **Version Notes:** Exact commit helpers and input integration are target-sensitive.

### Question: How should server target-data validation work for a predicted attack?

- **Category:** GAS / Target Data / Security
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Treat client target data as evidence; the server validates ownership, timing, target, range, line of sight, cadence, tags and derives final result.
- **Strong 3-Year-Engineer Answer:** The client can propose hit/target/location for responsiveness, but the server checks the spec is activatable, the request timing is plausible, target exists and is valid, range/LOS/faction/dead-state rules pass, resources/cooldown/cadence are valid and set-by-caller values are bounded or derived. The client should not send "deal 500 damage." If using server rewind, I name it separately from full rollback.
- **Common Weak Answer:** "Validate target data by checking if the Actor is valid."
- **Follow-up Questions:** LOS? stale actor? fire cadence? lag compensation? set-by-caller spoofing?
- **Hands-on Verification Task:** Send invalid target data under lag and record each rejection reason.
- **Sources:** [SRC-GAS-011], [SRC-GAS-005], [SRC-NET-004]
- **Version Notes:** Target-data structures and rewind integration are project-specific.

### Question: What is the acceptance test for a custom Ability Task?

- **Category:** GAS / Ability Tasks
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Cancel the ability at every wait phase and prove delegates/timers are unbound and no late callback mutates gameplay.
- **Strong 3-Year-Engineer Answer:** A task owns asynchronous ability-scoped work. I test event-before-binding, cancel-before-event, owner destroyed, task ending, ability ending, duplicate event and late callback. The task should end exactly once, unbind external handles in destruction, check whether it should broadcast delegates, and ignore or safely fail late events. The real product is cancellation correctness.
- **Common Weak Answer:** "The task ends when the ability ends."
- **Follow-up Questions:** External delegates? timers? montage interrupts? exactly-once broadcast?
- **Hands-on Verification Task:** Build a wait-release task and cancel during every phase with trace proof.
- **Sources:** [SRC-GAS-007], [SRC-GAS-003]
- **Version Notes:** Ability Task source is UE4.27-era; verify current APIs.

### Question: How do you prevent Gameplay Cues from leaking authority?

- **Category:** GAS / Gameplay Cues / Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Cues present gameplay state; they must not calculate durable damage, tags, inventory, objectives or win conditions.
- **Strong 3-Year-Engineer Answer:** I keep gameplay truth in abilities, effects, attributes or ordinary server systems. Cue handlers use context for VFX/audio/camera/UI presentation and tolerate prediction, duplication, relevancy loss and missing assets. I test by disabling cue assets and proving health/tags/effects still work. Persistent cues also need cleanup on effect removal, owner destroy and pooling reset.
- **Common Weak Answer:** "GameplayCue is where the gameplay effect happens visually and logically."
- **Follow-up Questions:** predicted duplicate? persistent cue removal? missing package asset? pooling?
- **Hands-on Verification Task:** Disable poison and dash cue assets, then prove gameplay outcomes still occur.
- **Sources:** [SRC-GAS-008], [SRC-FX-005], [SRC-AUDIO-002]
- **Version Notes:** Cue routing and notify classes are branch/project-sensitive.

### Question: What does Full, Mixed and Minimal GE replication actually decide?

- **Category:** GAS / Replication
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** It controls active Gameplay Effect information detail by audience; it does not mean every ASC attribute/tag field follows that same rule.
- **Strong 3-Year-Engineer Answer:** Full sends full active-effect information to relevant clients. Mixed is commonly used when owners need detailed effect information while simulated proxies receive less. Minimal sends minimal effect information. I choose from product observability and bandwidth evidence, verify owner setup, and separately confirm attribute/tag/cue visibility. Saying "Minimal means no attributes" confuses active GE detail with other replicated state.
- **Common Weak Answer:** "Mixed is for players and Minimal is for AI."
- **Follow-up Questions:** owner setup? UI needs? spectator needs? bandwidth? tags?
- **Hands-on Verification Task:** Compare owner and simulated proxy UI/cues/tags/effects under all three modes.
- **Sources:** [SRC-GAS-012], [SRC-GAS-002]
- **Version Notes:** Exact mode behaviour must be validated in target branch.

### Question: How do you debug a cooldown tag that never clears?

- **Category:** GAS / Tags / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Inspect active effects and loose-tag sources/counts, then trace duration, stack, removal and respawn/equipment lifecycle.
- **Strong 3-Year-Engineer Answer:** I do not just remove the tag. I identify every active effect and loose owner that grants the cooldown tag, then inspect effect duration, stack/refresh, inhibition, removal handle, source object and old-avatar/equipment lifetime. Tags are counted, so one source ending must not clear another, and one leaked source can keep the tag forever. The final fix is at the missing lifecycle edge.
- **Common Weak Answer:** "Clear the cooldown tag when the ability ends."
- **Follow-up Questions:** effect handle? loose tag? stack source? respawn? equipment removal?
- **Hands-on Verification Task:** Leak a cooldown tag through equipment removal, then fix by removing the correct effect/source.
- **Sources:** [SRC-GAS-005], [SRC-GAS-006]
- **Version Notes:** Tag-count inspection tooling is branch/project-sensitive.

### Question: How should GAS integrate with UI without UI owning truth?

- **Category:** GAS / UI / Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** UI should observe ASC-derived model state and request actions through input/commands; it should not own ability availability, health or cooldown truth.
- **Strong 3-Year-Engineer Answer:** I project attributes, tags, cooldowns and ability availability into a local UI model from ASC delegates/queries. UI can request activation through Enhanced Input or a command adapter, but GAS still validates tags, cost, cooldown, target and authority. I bind/unbind per local player/avatar and handle predicted values separately from confirmed values. A widget should not be the source of health or cooldown state.
- **Common Weak Answer:** "Bind progress bars directly to GAS attributes."
- **Follow-up Questions:** predicted UI? respawn rebind? split-screen? simulated proxy?
- **Hands-on Verification Task:** Destroy/respawn the avatar and prove UI rebinds to the current ASC/model without stale values.
- **Sources:** [SRC-GAS-002], [SRC-GAS-004], [SRC-UI-002], [SRC-INPUT-001]
- **Version Notes:** UI architecture and plugin choices are project-sensitive.

### Question: What GAS state should remain outside GAS?

- **Category:** GAS / Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Durable inventory, quest state, save identity, economy, matchmaking and many authoritative transactions usually remain ordinary gameplay systems that may grant/apply GAS state.
- **Strong 3-Year-Engineer Answer:** GAS is excellent for abilities, tags, attributes, effects, tasks, cues and selected prediction. It is not a universal domain model. Inventory can grant abilities/effects, quests can listen to events, UI can observe attributes, but durable ownership, item identity, quest progress, save migration and cross-container transactions usually belong in explicit systems. This keeps GAS from becoming a god object and makes persistence/networking clearer.
- **Common Weak Answer:** "Put all combat and player state in the ASC."
- **Follow-up Questions:** item identity? save migration? quest objective? trade transaction?
- **Hands-on Verification Task:** Draw Project 9 boundaries between inventory, save, UI, abilities and effects.
- **Sources:** [SRC-GAS-001], [SRC-ARCH-009], [SRC-EPIC-035]
- **Version Notes:** Architecture decision; GAS plugin details remain branch-sensitive.

### Question: How do you compare GAS against a bespoke ability Component?

- **Category:** GAS / System Design
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Compare interaction density, prediction, tags/effects/stacking, content authoring, debugging, team cost and failure handling in a representative slice.
- **Strong 3-Year-Engineer Answer:** I implement or design one vertical slice: sprint/dash, damage, cooldown, stun, respawn and one equipment grant. GAS earns itself if it removes duplicated policy around tags, costs, effects, prediction and authoring. A bespoke Component wins if the rule set is small, server-only and easier to test. The comparison includes debugging evidence and team maintainability, not just whether both versions can trigger an action.
- **Common Weak Answer:** "Use GAS for multiplayer or complex games."
- **Follow-up Questions:** What is the first GAS benefit? What is the first GAS cost? Can both coexist?
- **Hands-on Verification Task:** Build a small bespoke branch and GAS branch for Project 9 and write a decision memo.
- **Sources:** [SRC-GAS-001], [SRC-ARCH-001], [SRC-NET-001]
- **Version Notes:** Project-specific; avoid genre-only decisions.

### Question: What is the difference between GAS prediction and full rollback networking?

- **Category:** GAS / Networking / Rollback
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** GAS prediction correlates supported provisional owner actions with server confirmation; full rollback restores broader simulation state and resimulates from input/state history.
- **Strong 3-Year-Engineer Answer:** GAS prediction is scoped to supported ability/effect/cue/target-data paths and uses prediction keys to reconcile owner-side provisional work. It does not automatically make inventory, arbitrary Actors, AI or physics roll back. Full rollback is a larger architecture with deterministic or controlled state snapshots, input logs, side-effect suppression and resimulation windows. I keep the terms separate so debugging and acceptance tests match the system actually built.
- **Common Weak Answer:** "GAS predicted abilities are rollback."
- **Follow-up Questions:** What rolls back? custom movement? inventory? server rewind?
- **Hands-on Verification Task:** List every side effect in predicted Dash and mark GAS-predicted, server-only, cosmetic or custom rollback-required.
- **Sources:** [SRC-GAS-011], [SRC-GAS-013], [SRC-NET-018], [SRC-NET-019]
- **Version Notes:** Network Prediction plugin and full rollback systems require separate target-source proof.

### Question: How do you make GAS profiling actionable?

- **Category:** GAS / Profiling
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Separate activations, active effects, periodic ticks, tag delegates, execution calculations, tasks, cues, Blueprint work and replication by audience.
- **Strong 3-Year-Engineer Answer:** I do not optimise "GAS" as a blob. I count ability activations, active effects per ASC, periodic execution rate, tag-query/delegate churn, execution/magnitude cost, Blueprint task/cue time, cue spawn/pool pressure, replication bytes and allocation growth after activate/cancel/respawn. Then I fix the measured layer: leaked tasks, excessive periodic effects, Blueprint hot paths, cue detail, tag delegate storms or wrong replication detail.
- **Common Weak Answer:** "GAS is heavy, so use fewer Gameplay Effects."
- **Follow-up Questions:** periodic effects? tag queries? cues? replication mode? UI polling?
- **Hands-on Verification Task:** Profile 50 bots with periodic regen and cues, then reduce the measured bottleneck with evidence.
- **Sources:** [SRC-GAS-005], [SRC-GAS-012], [SRC-PERF-003], [SRC-PERF-004]
- **Version Notes:** Trace channels and tooling vary by target branch.

## Apple and Console Profiling Expansion Set

### Question: How do you correlate Unreal Insights with an Xcode Metal GPU capture?

- **Category:** Platform Profiling / Apple / Rendering
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Capture the same packaged scenario/frame, align markers/timestamps, map Metal passes/pipelines/counters back to UE passes/content, then validate with normal UE before/after timing.
- **Strong 3-Year-Engineer Answer:** I first use UE stats, CSV, GPU Visualizer/RDG/Insights to identify the failing window and pass. Then I capture the same frame with Xcode GPU Frame Capture, recording device, OS, Xcode, UE build, RHI, package and thermal/performance state. In Metal I inspect performance timeline, counters, pipeline states and shader hot spots, but I translate the finding back to a UE pass/material/render target/scene capture/content pattern. The fix is accepted only when the normal UE gate improves too.
- **Common Weak Answer:** "Use Xcode to see which Metal shader is slow."
- **Follow-up Questions:** What metadata? What if replay is on a different device? What if serial mode changes timing?
- **Hands-on Verification Task:** Capture one GPU-bound UE frame with Unreal and Xcode and write a map-back table.
- **Sources:** [SRC-PERF-003], [SRC-RENDER-019], [SRC-PLAT-021], [SRC-PLAT-022], [SRC-PLAT-023]
- **Version Notes:** Metal debugger features and counter availability are device/GPU/OS/Xcode sensitive.

### Question: Why is simulator memory not enough for iOS memory proof?

- **Category:** Platform Profiling / Apple / Memory
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** The simulator does not model iOS device memory warnings/terminations faithfully, so safe simulator memory does not prove safe target memory.
- **Strong 3-Year-Engineer Answer:** I use simulator for convenient debugging, but final memory proof must run on the target device. Apple notes simulator memory can stay green because macOS does not issue the same memory warnings or out-of-memory terminations. For UE, I compare device Xcode memory evidence with UE Memory Insights/LLM, high-water and retained memory, asset/render target pools and native allocations. I separate simulator-only findings from ship evidence.
- **Common Weak Answer:** "Simulator uses more memory, so device will be fine."
- **Follow-up Questions:** high-water versus retained? LLM categories? native allocations? background/foreground?
- **Hands-on Verification Task:** Run the same load scenario in simulator and on device, then explain the differences.
- **Sources:** [SRC-PLAT-019], [SRC-PERF-006], [SRC-PERF-010]
- **Version Notes:** iOS memory behaviour varies by device, OS and app state.

### Question: How would you use MetricKit for a UE game?

- **Category:** Platform Profiling / Apple / Telemetry
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use MetricKit as field telemetry for trends and diagnostics, then reproduce locally with UE/platform captures.
- **Strong 3-Year-Engineer Answer:** MetricKit gives daily real-device metric and diagnostic reports such as launch, responsiveness, CPU/memory/GPU/display, disk, termination, crashes/hangs and signpost-related data. I aggregate by app version, device/OS class and scenario marker where possible. It tells me what to prioritise in the field, not the exact root cause of a single frame. I convert a spike into a device-lab reproduction and add signposts/UE markers if attribution is weak.
- **Common Weak Answer:** "MetricKit tells you the performance problem."
- **Follow-up Questions:** reports frequency? signposts? app version? local reproduction?
- **Hands-on Verification Task:** Convert a hypothetical MetricKit hitch spike into a local profiling plan.
- **Sources:** [SRC-PLAT-024], [SRC-PLAT-018], [SRC-PERF-009]
- **Version Notes:** MetricKit availability and metrics differ by OS/platform.

### Question: What metadata must accompany an Apple Metal capture?

- **Category:** Platform Profiling / Evidence
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Device/GPU/OS/Xcode, UE branch/build/RHI, package/scenario, capture scope/frame, thermal/performance state and matching UE evidence.
- **Strong 3-Year-Engineer Answer:** Without metadata, a Metal capture is hard to reproduce or compare. I record device model/GPU family, OS, Xcode, UE branch, build config, RHI/Metal settings, package/cook/shader/PSO state, map/scenario, battery/thermal/performance state, capture option/scope/count and the Unreal evidence that led to the capture. I also note whether replay/profiling/capture settings could perturb timing.
- **Common Weak Answer:** "Save the .gputrace file."
- **Follow-up Questions:** capture overhead? replay compatibility? shader sources? package ID?
- **Hands-on Verification Task:** Create a capture checklist and reject one incomplete capture.
- **Sources:** [SRC-PLAT-021], [SRC-PLAT-022], [SRC-PLAT-023]
- **Version Notes:** Exact capture and profiling options change with Xcode and target device.

### Question: How do you interpret Metal serial versus concurrent profiling modes?

- **Category:** Platform Profiling / Apple / GPU
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Concurrent mode reflects overlapped runtime execution; serial mode isolates pass costs more precisely but does not represent normal runtime performance.
- **Strong 3-Year-Engineer Answer:** Apple GPUs can overlap vertex, fragment and compute work. The Metal debugger can profile in concurrent mode, which reflects that overlap, or serial mode, which forces passes to run after one another for clearer per-pass data. Serial mode is useful for attribution, but I do not treat its total time as normal runtime. I use it to understand a pass, then validate the fix in a representative concurrent/device run and UE timing.
- **Common Weak Answer:** "Serial is more accurate, so use it for final numbers."
- **Follow-up Questions:** overlap? counters? performance state? UE pass timing?
- **Hands-on Verification Task:** Compare one Metal trace in concurrent and serial modes and explain the difference.
- **Sources:** [SRC-PLAT-022]
- **Version Notes:** GPU execution mode options are Xcode/device sensitive.

### Question: How does Xcode Thread Performance Checker apply to Unreal?

- **Category:** Platform Profiling / Apple / CPU
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It can flag native priority inversions and main-thread non-UI work, especially in platform wrappers/plugins, but it is not a replacement for UE GameThread profiling.
- **Strong 3-Year-Engineer Answer:** I use it when a native iOS/macOS wrapper, platform service, Objective-C/Swift bridge, file/network call or main-thread wait may be causing responsiveness issues. It detects classes of problems such as priority inversions and non-UI work on the main thread. For normal UE gameplay cost, I still use Unreal Insights, stats and CSV. The tool answers an OS/native responsiveness question, not every UE frame-time question.
- **Common Weak Answer:** "Use Thread Performance Checker to optimise the GameThread."
- **Follow-up Questions:** priority inversion? QoS? platform plugin? tests as failures?
- **Hands-on Verification Task:** Inject a blocking native wrapper call and catch it with Thread Performance Checker.
- **Sources:** [SRC-PLAT-020], [SRC-PERF-003]
- **Version Notes:** Tool behaviour depends on Xcode/runtime and native code path.

### Question: How do you answer console profiling questions without leaking confidential details?

- **Category:** Platform Profiling / Console / Interview Safety
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Discuss source-safe workflow and categories, and state that exact tools, counters and cert rules require authorised platform-holder docs.
- **Strong 3-Year-Engineer Answer:** I would not quote confidential TRC/TCR/XR-style requirements or platform-tool counters publicly. I can still describe the engineering workflow: reproduce in a packaged devkit build, archive symbols/build ID, capture Unreal and platform traces, record SDK/firmware/tool versions where allowed, map CPU/GPU/memory/I/O/network/lifecycle findings back to UE/content owners, and validate through the release gate. Exact requirements live in authorised docs.
- **Common Weak Answer:** "Console cert mostly checks crashes, save data and controller disconnect."
- **Follow-up Questions:** what can be public? where store artifacts? how map back to UE?
- **Hands-on Verification Task:** Write a source-safe console profiler memo with redacted platform evidence category.
- **Sources:** [SRC-PLAT-018], [SRC-BUILD-018], [SRC-PERF-003]
- **Version Notes:** Console details are source-sensitive and often confidential.

### Question: What makes console profiling different from PC packaged profiling?

- **Category:** Platform Profiling / Console
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Fixed platform constraints, SDK/toolchain, memory layout, GPU/driver behavior, storage, OS lifecycle and certification-adjacent states can differ materially from PC.
- **Strong 3-Year-Engineer Answer:** A PC packaged profile is useful, but console proof must use the target package, SDK/devkit, symbols and platform tools. I watch CPU scheduling, GPU counters/captures, memory pools, I/O/storage, online services, suspend/resume, account/controller/network/storage failures and crash symbolication. I keep confidential details private, but the public engineering answer is that target hardware and platform state transitions decide readiness.
- **Common Weak Answer:** "Console is just a fixed-spec PC."
- **Follow-up Questions:** memory? storage? lifecycle? platform services? symbols?
- **Hands-on Verification Task:** Compare a PC profiling packet against a source-safe console evidence packet template.
- **Sources:** [SRC-PLAT-005], [SRC-PLAT-018], [SRC-BUILD-018], [SRC-PERF-009]
- **Version Notes:** Exact console behavior requires platform-holder docs and devkit evidence.

### Question: How do you handle a GPU hitch that appears only on one Apple or console target?

- **Category:** Platform Profiling / GPU Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Verify package/config, reproduce deterministically, capture UE and platform GPU evidence, map the pass/pipeline/counter back to content, then rerun the same scenario.
- **Strong 3-Year-Engineer Answer:** I first rule out package/content/config differences: shader/PSO state, scalability, RHI, device profile, OS/firmware and thermal/performance state. Then I use UE GPU/RDG/CSV evidence to pick the failing frame and platform GPU capture to inspect pass/pipeline/shader/bandwidth-like detail. I change one UE/content variable at a time and rerun the same deterministic scenario. I do not generalise one device capture across platforms.
- **Common Weak Answer:** "It is probably a driver issue."
- **Follow-up Questions:** PSO? shader permutation? scene capture? thermal? replay device?
- **Hands-on Verification Task:** Build a GPU hitch triage checklist for Apple and a source-safe console path.
- **Sources:** [SRC-RENDER-019], [SRC-PLAT-021], [SRC-PLAT-022], [SRC-PLAT-023]
- **Version Notes:** Platform GPU tools/counters are device/vendor/version sensitive.

### Question: How do field metrics become a device-lab task?

- **Category:** Platform Profiling / Telemetry / Device Lab
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Break the metric down by version/device/OS/scenario, form a reproduction hypothesis, then run a controlled target-device capture.
- **Strong 3-Year-Engineer Answer:** Field metrics tell me priority and affected populations. I group by app version, content version, OS, device class and scenario markers/signposts. Then I create a device-lab task with package ID, target devices, deterministic scenario, UE CSV/Insights and platform tool capture. If the field metric lacks attribution, I add markers or telemetry in the next build. I do not treat the aggregate metric as the root cause.
- **Common Weak Answer:** "If users report hitches, profile the slowest device."
- **Follow-up Questions:** signposts? cohorts? app version? local reproduction? no repro?
- **Hands-on Verification Task:** Turn a MetricKit/Organizer-style hang spike into a lab run manifest.
- **Sources:** [SRC-PLAT-024], [SRC-PLAT-018], [SRC-PLAT-009]
- **Version Notes:** Field telemetry latency and availability differ by platform.

### Question: What should a platform profiling memo include?

- **Category:** Platform Profiling / Communication
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Repro metadata, UE evidence, platform evidence, hypothesis, change, before/after result, residual risk and what the evidence does not prove.
- **Strong 3-Year-Engineer Answer:** A senior memo makes the work reproducible. It lists build/package ID, device/OS/tool versions, scenario, settings, UE trace/CSV, platform artifact, exact finding, mapped UE/content owner, one change, before/after comparison and residual risk. It explicitly says what the evidence does not prove, such as other devices, other OS versions, normal runtime after capture overhead, or confidential console requirements outside the tested path.
- **Common Weak Answer:** "Attach screenshots and summarize the bottleneck."
- **Follow-up Questions:** metadata? artifact identity? access control? overgeneralisation?
- **Hands-on Verification Task:** Write a one-page memo for a Metal capture or source-safe console issue.
- **Sources:** [SRC-PLAT-018], [SRC-PLAT-021], [SRC-PERF-009]
- **Version Notes:** Evidence requirements are project/platform specific.

### Question: How do you avoid overclaiming from one platform capture?

- **Category:** Platform Profiling / Evidence Quality
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Label device/tool/build scope, repeat the scenario, cross-check with UE metrics and state what remains unproven.
- **Strong 3-Year-Engineer Answer:** Platform captures are powerful but narrow. I record exact metadata, repeat enough to understand variance, correlate with UE evidence, validate after the change and avoid applying one device/GPU/OS result to all SKUs. For Apple, replay/capture compatibility and performance state matter. For consoles, confidential counters and tool versions must stay inside authorised workflows. The report should state residual risk clearly.
- **Common Weak Answer:** "The profiler proves this shader is the bottleneck."
- **Follow-up Questions:** variance? other SKUs? capture overhead? field metrics? source-sensitive evidence?
- **Hands-on Verification Task:** Add a "what this does not prove" section to an existing profiling report.
- **Sources:** [SRC-PLAT-018], [SRC-PLAT-022], [SRC-PLAT-023], [SRC-PERF-001]
- **Version Notes:** Tool/counter semantics and target hardware differ.

## Advanced Gameplay Patterns Specialist Set

### Question: What makes a gameplay operation transactional?

- **Category:** Gameplay Architecture / Transactions
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** It validates against a known state revision, commits all required changes together, emits an auditable result and controls failure/duplicate/retry behavior.
- **Strong 3-Year-Engineer Answer:** A gameplay transaction is a contract, not necessarily a database feature. For an inventory move, I validate ownership, capacity, locks, item existence and expected revisions before mutating. The commit produces deltas, new revisions and a result tied to an operation ID. If validation fails, no partial remove/add remains. This makes server authority, UI prediction, save/load and bug reconstruction tractable.
- **Common Weak Answer:** "A transaction means undo if something fails."
- **Follow-up Questions:** operation ID? stale revision? duplicate request? UI prediction?
- **Hands-on Verification Task:** Implement item move as validate-then-commit deltas and inject add failure after remove.
- **Sources:** [SRC-ARCH-004], [SRC-NET-001], [SRC-EPIC-035]
- **Version Notes:** Architecture pattern; implementation is project-specific.

### Question: Why are operation IDs useful in gameplay systems?

- **Category:** Gameplay Architecture / Observability
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** They correlate request, validation, commit, replication, UI prediction, telemetry and duplicate/retry handling.
- **Strong 3-Year-Engineer Answer:** Without an operation ID, a drag/drop, reward claim or trade looks like unrelated logs from UI, server and replication. With one ID, I can reconstruct the path and make duplicate requests idempotent. The ID is not the state version; it names the intention. Revisions name the state the intention expected. Together they solve stale UI and duplicate/replay bugs.
- **Common Weak Answer:** "Use timestamps for logs."
- **Follow-up Questions:** operation ID versus revision? server retry? UI pending state?
- **Hands-on Verification Task:** Add operation IDs to inventory UI prediction and server commit logs.
- **Sources:** [SRC-ARCH-004], [SRC-ARCH-006]
- **Version Notes:** Stable design principle.

### Question: What is the difference between an event and durable truth?

- **Category:** Gameplay Architecture / Events
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** An event says something happened; durable state answers what is true now for late subscribers, save/load and replication.
- **Strong 3-Year-Engineer Answer:** I use events for incremental notifications, not as the only source of truth unless event sourcing is explicitly designed. A quest that requires owning 10 ore must query current inventory on activation/load, then observe committed deltas. If it only listens for future pickup events, it misses progress that happened before subscription. Durable state, revisions and snapshots make late join and save/load reliable.
- **Common Weak Answer:** "Just replay the item pickup events."
- **Follow-up Questions:** late subscription? save/load? event payload? current snapshot?
- **Hands-on Verification Task:** Build a quest objective that works when the player already owns the required item.
- **Sources:** [SRC-ARCH-003], [SRC-ARCH-006], [SRC-EPIC-035]
- **Version Notes:** Event sourcing is a separate architecture, not the default.

### Question: What makes an event payload strong enough?

- **Category:** Gameplay Architecture / Events
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** It carries the committed transition: IDs, old/new values or delta, reason/source and revision/operation context where needed.
- **Strong 3-Year-Engineer Answer:** A weak `OnInventoryChanged` forces listeners to query mutable state and can observe inconsistent timing. A strong payload includes operation ID, container/item IDs, before/after revisions, deltas, old/new quantities and reason/source. The payload should let most listeners update without re-entering the publisher. If payloads become huge, provide a snapshot/query API plus compact deltas deliberately.
- **Common Weak Answer:** "Listeners can query what they need."
- **Follow-up Questions:** old/new value? revision? source? UI list update?
- **Hands-on Verification Task:** Replace a broad changed event with a delta payload and measure UI refresh work.
- **Sources:** [SRC-ARCH-003], [SRC-EPIC-028]
- **Version Notes:** Delegate choice and payload type are project/branch sensitive.

### Question: When is an event queue better than a delegate?

- **Category:** Gameplay Architecture / Queues
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use a queue when delayed, batched, ordered or throttled processing is part of the desired semantics.
- **Strong 3-Year-Engineer Answer:** Delegates are usually synchronous notifications. Queues are useful for end-of-frame UI deltas, perception batching, telemetry, editor validation streams or other work where latency/batching/backpressure is intended. I define delivery phase, ordering scope, capacity, drop/coalesce policy, stale target handling, duplicate policy, thread rules and tracing. If the caller needs an immediate result, I do not hide it behind a bus.
- **Common Weak Answer:** "Use a message bus to decouple systems."
- **Follow-up Questions:** capacity? stale target? order? immediate answer? observability?
- **Hands-on Verification Task:** Build a bounded UI delta queue and record drops/coalesces under burst updates.
- **Sources:** [SRC-ARCH-006], [SRC-ARCH-003]
- **Version Notes:** Pattern-level design; implementation varies.

### Question: How do revisions prevent stale UI operations?

- **Category:** Gameplay Architecture / UI Reconciliation
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** The client request states the revision it was based on; the server rejects or reconciles if authoritative state has changed.
- **Strong 3-Year-Engineer Answer:** If a player drags an item from slot A to B while the server already consumed that item, the UI acted on stale state. The request should include operation ID and expected container revision. The server validates against current revision and returns commit/rejection with new revision. The UI can discard stale responses, clear pending prediction and rebuild from authoritative state.
- **Common Weak Answer:** "Just refresh the inventory after the server replies."
- **Follow-up Questions:** operation ordering? pending state? duplicate response? server conflict?
- **Hands-on Verification Task:** Send two drag/drop operations out of order and prove UI resolves correctly.
- **Sources:** [SRC-NET-001], [SRC-ARCH-004]
- **Version Notes:** Applies to both simple replication and Fast Array-style deltas.

### Question: How do you make a reward grant idempotent?

- **Category:** Gameplay Architecture / Idempotency
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Store a durable claim/operation record so repeated requests or load/retry paths cannot grant twice.
- **Strong 3-Year-Engineer Answer:** I give the reward claim a stable operation or entitlement ID and store the committed claim in authoritative state/save data. If the request repeats, the server returns already-claimed or the same result instead of granting again. During save/load restore, I apply claimed state without firing normal reward events. This protects against RPC retry, UI double-click, disconnect/reconnect and migration replay.
- **Common Weak Answer:** "Disable the button after click."
- **Follow-up Questions:** save/load? quest completion replay? server retry? migration?
- **Hands-on Verification Task:** Force the same quest reward request twice and after load, then prove no duplicate currency/item.
- **Sources:** [SRC-ARCH-004], [SRC-EPIC-035], [SRC-NET-004]
- **Version Notes:** Persistence and server authority policy are project-specific.

### Question: What is a restore path in save/load architecture?

- **Category:** Save Systems / Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** A controlled load path that applies versioned state without replaying normal gameplay side effects, then rebuilds and emits a ready snapshot.
- **Strong 3-Year-Engineer Answer:** Normal setters may fire achievements, quests, damage, audio and analytics. Restore mode loads header/version, migrates records, resolves definitions, creates runtime instances, applies state with notifications suppressed or tagged as restore, rebuilds caches and emits one ready snapshot. I test with historical fixture saves. The system stores stable IDs and versioned state, not live UObjects.
- **Common Weak Answer:** "Load variables marked SaveGame."
- **Follow-up Questions:** migration? missing definition? notifications? derived cache?
- **Hands-on Verification Task:** Load a save that would otherwise fire reward events and repair with restore mode.
- **Sources:** [SRC-EPIC-035], [SRC-ARCH-009], [SRC-ASSET-006]
- **Version Notes:** `SaveGame` is only a flag for consuming archives; system design is project-owned.

### Question: Why are historical save fixtures valuable?

- **Category:** Save Systems / Testing
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** They prove old player data still migrates after schema/content changes.
- **Strong 3-Year-Engineer Answer:** Designers rename item IDs, split objectives and remove definitions. A few small old saves with expected post-load state catch broken migrations early. Each fixture records old schema/content state and expected current result, including missing-content policy. This turns save compatibility from "hope" into repeatable tests and helps interviewers see production thinking.
- **Common Weak Answer:** "Backwards compatibility is handled by version numbers."
- **Follow-up Questions:** renamed asset? removed item? objective reordered? platform save?
- **Hands-on Verification Task:** Add fixtures for a renamed item and split quest objective.
- **Sources:** [SRC-EPIC-035], [SRC-ARCH-012]
- **Version Notes:** Fixture format is project-specific.

### Question: How do you design UI prediction without UI owning truth?

- **Category:** Gameplay Architecture / UI Prediction
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** UI creates pending operation IDs and visual state, then reconciles from authoritative commit/rejection and revisions.
- **Strong 3-Year-Engineer Answer:** UI can show immediate feedback for drag/drop, equip or purchase, but the server/domain model owns truth. The view model marks a pending operation with expected revision. The authoritative result returns operation ID, new revisions and deltas or rejection. Widgets are disposable; they can rebuild from current authoritative state. This supports out-of-order replies, stale targets, disconnects and rejected predictions.
- **Common Weak Answer:** "Let UI update locally, then server corrects later."
- **Follow-up Questions:** pending marker? widget recycling? stale response? full rebuild?
- **Hands-on Verification Task:** Implement predicted inventory drag/drop with server rejection and out-of-order response.
- **Sources:** [SRC-UI-002], [SRC-ARCH-004], [SRC-NET-001]
- **Version Notes:** UI framework choices are project/plugin sensitive.

### Question: What should a production state machine own besides state names?

- **Category:** Gameplay Architecture / State Machines
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Enter/exit side effects, timers/tasks/delegates, transition guards, authority, replication, save/restore and cancellation cleanup.
- **Strong 3-Year-Engineer Answer:** A reload state is not just an enum. It may own montage/task delegates, ammo reservation, input buffering, interrupt rules, predicted presentation, authoritative ammo commit and cleanup. Each enter path has a matching exit/cancel path. If side effects are scattered outside the state contract, you get stuck montages, duplicate ammo changes and reload after death.
- **Common Weak Answer:** "Use an enum or State pattern for idle/firing/reloading."
- **Follow-up Questions:** cancellation? death? save? prediction? side-effect ownership?
- **Hands-on Verification Task:** Interrupt reload by stun, weapon swap and death, then prove all side effects clean up.
- **Sources:** [SRC-ARCH-005], [SRC-ARCH-004]
- **Version Notes:** StateTree/GAS/Anim state machines are separate implementations with their own contracts.

### Question: How can a global message bus become an architecture problem?

- **Category:** Gameplay Architecture / Messaging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** It hides dependencies, spreads unversioned schemas, weakens ownership and can let arbitrary subscribers mutate authoritative state.
- **Strong 3-Year-Engineer Answer:** A bus is useful at bounded domain bridges, but a global bag of messages becomes invisible coupling. I want topic ownership, schema/version, delivery phase, capacity, duplicate policy, observability and authority boundaries. Gameplay truth should not be committed by arbitrary listeners. For required collaborations, direct commands/queries are clearer.
- **Common Weak Answer:** "A bus decouples everything."
- **Follow-up Questions:** schema owner? order? replay? mutation authority? queue depth?
- **Hands-on Verification Task:** Trace one global message topic and replace it with a domain-owned event or direct command.
- **Sources:** [SRC-ARCH-006], [SRC-ARCH-007]
- **Version Notes:** Pattern-level; implementation varies.

### Question: How do Data Asset definitions support migration-safe runtime state?

- **Category:** Data-Driven Architecture / Persistence
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Runtime state stores stable definition IDs and mutable fields, while definitions stay authored/shared and migrations map old IDs/schema to current content.
- **Strong 3-Year-Engineer Answer:** Data Assets are definitions, not per-instance state. Runtime records store a stable definition ID plus quantity/durability/rolls/owner. Saves store IDs and schema version, then resolve definitions during restore with missing-content policy. Validation catches impossible definitions before runtime. If designers rename or split content IDs, migration fixtures prove old saves still load correctly.
- **Common Weak Answer:** "Save a reference to the Data Asset."
- **Follow-up Questions:** soft reference? missing asset? Primary Asset ID? validation? content version?
- **Hands-on Verification Task:** Rename an item definition and migrate an old save to the new ID.
- **Sources:** [SRC-ARCH-009], [SRC-ARCH-011], [SRC-ARCH-012], [SRC-EPIC-035]
- **Version Notes:** Asset identity and Asset Manager configuration are project-specific.

### Question: How do you debug a duplicated inventory item?

- **Category:** Gameplay Architecture / Debugging
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Reconstruct the operation log: operation ID, revisions, validation, commit deltas, replication/UI prediction and save/load path.
- **Strong 3-Year-Engineer Answer:** I do not start from the UI screenshot. I find the operation ID, source/target container revisions, validation result, commit delta and server timestamp. Then I check duplicate request/retry, partial mutation before failure, replicated delta, client prediction applying twice and save/load replay. The fix usually adds validate-before-mutate, idempotency or revision rejection, then a regression test with duplicate operation ID.
- **Common Weak Answer:** "The UI probably added the item twice."
- **Follow-up Questions:** server commit? replicated delta? save replay? stale revision?
- **Hands-on Verification Task:** Inject duplicate request and partial failure bugs, then repair and reconstruct from logs.
- **Sources:** [SRC-ARCH-004], [SRC-NET-001], [SRC-EPIC-035]
- **Version Notes:** Inventory implementation is project-specific.

### Question: What architecture costs should you profile in advanced patterns?

- **Category:** Gameplay Architecture / Profiling
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Operation rate, validation cost, queue depth, delegate cascades, dirty-cache recompute, save migration, UI rebuilds and replicated delta size.
- **Strong 3-Year-Engineer Answer:** Patterns have runtime cost. I measure transaction count/latency, validation allocations, queue depth/drop/coalesce, delegate listener count and cascade time, dirty invalidations and recompute cost, UI pending/rebuild cost, save/load migration time, validation tooling time and replicated delta bytes/frequency. Then I optimise the measured mechanism while preserving IDs/revisions/evidence.
- **Common Weak Answer:** "Patterns are architecture; performance comes later."
- **Follow-up Questions:** delegate cascade? queue capacity? replication delta? save migration?
- **Hands-on Verification Task:** Stress 1000 inventory operations and report validation, event and UI update cost separately.
- **Sources:** [SRC-PERF-003], [SRC-ARCH-003], [SRC-ARCH-006]
- **Version Notes:** Tool counters and tracing hooks are project-specific.

## Broad Specialist Scenario Expansion Set

### Question: Why should animation notifies usually not be the sole authority for damage?

- **Category:** Animation / Gameplay Authority
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Notifies are timing/presentation signals; authoritative gameplay must validate state, target, montage/ability context and server timing.
- **Strong 3-Year-Engineer Answer:** A notify can mark an intended hit window, spawn effects or request a server-validated action, but it should not be the only durable damage rule. Montages can blend, skip, be interrupted, run differently on proxies or not exist on a dedicated server. I tie damage to authoritative gameplay state, ability/weapon context, target validation and one committed result, with notify timing as evidence or synchronisation.
- **Common Weak Answer:** "Put damage in the attack notify."
- **Follow-up Questions:** dedicated server? montage interrupt? prediction? replay?
- **Hands-on Verification Task:** Disable animation playback on server and prove authoritative melee damage still follows validated windows.
- **Sources:** [SRC-ANIM-008], [SRC-ANIM-010], [SRC-NET-001]
- **Version Notes:** Notify delivery and montage networking are target-branch sensitive.

### Question: How do you debug foot sliding in a networked character?

- **Category:** Animation / Movement / Networking
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Compare authoritative movement velocity/root motion, AnimBP inputs, Blend Space thresholds, sync markers and smoothing on server/owner/proxy.
- **Strong 3-Year-Engineer Answer:** I separate movement truth from pose selection. I log CharacterMovement velocity, acceleration, movement mode, root-motion mode, replicated smoothing and AnimBP exposed variables on owner and simulated proxy. Then I inspect Blend Space samples, sync groups, play rates, stride warping/IK and retarget scale. If root motion is involved, I verify who owns movement and how corrections are handled.
- **Common Weak Answer:** "Adjust the Blend Space."
- **Follow-up Questions:** root motion? simulated proxy smoothing? sync groups? retarget scale?
- **Hands-on Verification Task:** Record owner/proxy animation and movement variables while inducing packet loss.
- **Sources:** [SRC-ANIM-004], [SRC-ANIM-006], [SRC-ANIM-009], [SRC-NET-018]
- **Version Notes:** Root-motion networking and smoothing are version/project sensitive.

### Question: Why can a blocking collision still miss gameplay response?

- **Category:** Physics / Collision / Events
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Collision response, movement path, notification settings, geometry mode, component ownership and authority all affect whether gameplay receives an event.
- **Strong 3-Year-Engineer Answer:** A Block response means a query/simulation can block; it does not guarantee exactly-one Hit event. I inspect collision enabled, object/trace channels, both profiles, simple/complex geometry, sweep versus physics simulation, hit/overlap notification flags, moved component, initial overlap and network authority. Gameplay consequence should come from the authoritative path, not any local contact.
- **Common Weak Answer:** "Enable Generate Hit Events."
- **Follow-up Questions:** sweep? physics simulation? initial overlap? simple/complex? authority?
- **Hands-on Verification Task:** Build a matrix where a Block occurs with and without a Hit event and explain each case.
- **Sources:** [SRC-PHYS-001], [SRC-PHYS-002], [SRC-PHYS-003]
- **Version Notes:** Event timing and Chaos behavior need target tests.

### Question: How do you choose between sweep movement and simulated physics?

- **Category:** Physics / Movement Ownership
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Choose from ownership of integration: kinematic sweeps query/move deliberately, while simulation lets the physics solver integrate bodies/contacts.
- **Strong 3-Year-Engineer Answer:** `SetActorLocation` with sweep or CharacterMovement is a kinematic/query-style path with explicit gameplay ownership. Simulated physics integrates velocity, forces, masses, constraints and contacts over solver steps. Mixing kinematic teleports, CharacterMovement and rigid-body simulation on the same object often creates conflicting writers and network ambiguity. I choose one authority and bridge presentation carefully.
- **Common Weak Answer:** "Use physics for realistic objects and sweeps for simple ones."
- **Follow-up Questions:** networking? forces? constraints? hit events? prediction?
- **Hands-on Verification Task:** Implement a pushable crate with both approaches and compare correction/events under multiplayer.
- **Sources:** [SRC-PHYS-004], [SRC-PHYS-005], [SRC-NET-001]
- **Version Notes:** Networked physics features are highly version-sensitive.

### Question: Why can CommonUI focus break in split-screen?

- **Category:** UI / CommonUI / Local Player
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Focus and input context must be scoped to the correct local player, activatable layer and desired focus target.
- **Strong 3-Year-Engineer Answer:** Split-screen exposes global-focus assumptions. I verify which LocalPlayer owns the widget, which CommonUI activatable stack/layer is active, which input config is applied, what desired focus target returns and whether Enhanced Input contexts were added per local player. I use user-specific focus APIs and test two controllers opening/closing menus independently.
- **Common Weak Answer:** "Set keyboard focus on the widget."
- **Follow-up Questions:** LocalPlayer subsystem? desired focus target? input mode? controller glyphs?
- **Hands-on Verification Task:** Two local players open independent menus and restore gameplay focus without stealing from each other.
- **Sources:** [SRC-UI-010], [SRC-UI-017], [SRC-INPUT-004]
- **Version Notes:** CommonUI and Enhanced Input integration is plugin/version sensitive.

### Question: When can UMG invalidation or Retainer Panels make performance worse?

- **Category:** UI / Performance
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** When content changes frequently, volatility is mislabelled, render targets add GPU/memory cost, or latency/overdraw outweigh saved layout work.
- **Strong 3-Year-Engineer Answer:** Invalidation helps unchanged UI avoid repeated layout/paint. Retainers trade update frequency for render-target memory, capture/composite cost and latency. If the widget changes every frame, contains volatile children, has large render targets or adds overdraw, these tools can increase cost. I use Slate Insights/Widget Reflector and GPU/memory evidence before and after.
- **Common Weak Answer:** "Wrap expensive widgets in an Invalidation Box or Retainer."
- **Follow-up Questions:** volatility? render target size? latency? Slate Insights?
- **Hands-on Verification Task:** Compare a changing combat HUD with no invalidation, invalidation and retainer variants.
- **Sources:** [SRC-UI-004], [SRC-UI-005], [SRC-UI-003]
- **Version Notes:** UMG optimisation guidance is maintained beyond target; verify branch behavior.

### Question: How do you debug a sound that should be playing but is inaudible?

- **Category:** Audio / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Trace request, component lifetime, asset readiness, attenuation/listener, concurrency/priority, routing/mix/submix and platform device.
- **Strong 3-Year-Engineer Answer:** I log semantic event ID, owner, world, transform and audio asset. Then I check whether an Audio Component exists or auto-destroyed, whether the Wave/Cue/MetaSound is loaded, listener distance/focus/occlusion, concurrency admission/steal/virtualisation, Sound Class/Mix gain, Submix routing/effects, stream-cache underrun and output device. Gameplay warnings should not rely solely on an audible sound.
- **Common Weak Answer:** "Check volume and attenuation."
- **Follow-up Questions:** concurrency? virtualisation? routing? stream cache? platform device?
- **Hands-on Verification Task:** Force the same sound to fail through attenuation, concurrency and submix mute, then distinguish logs.
- **Sources:** [SRC-AUDIO-002], [SRC-AUDIO-003], [SRC-AUDIO-004], [SRC-AUDIO-006], [SRC-AUDIO-010]
- **Version Notes:** Audio console commands and stream cache details are target-sensitive.

### Question: Why can audio concurrency become a gameplay bug?

- **Category:** Audio / Gameplay Boundary
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Concurrency can reject, steal or virtualise sounds, so critical gameplay information cannot exist only as audio playback.
- **Strong 3-Year-Engineer Answer:** Audio concurrency is a presentation resource policy. If a low-health warning or parry cue is only an audible event, it can be stolen, culled, virtualised or inaudible. Durable gameplay state should update UI/haptics/replicated state as needed, while sound is one presentation channel. I test forced concurrency exhaustion and confirm gameplay remains understandable.
- **Common Weak Answer:** "Raise priority on important sounds."
- **Follow-up Questions:** owner-scoped concurrency? virtualization? accessibility? replication?
- **Hands-on Verification Task:** Set max concurrency low and prove critical warnings still surface through non-audio channels.
- **Sources:** [SRC-AUDIO-004], [SRC-AUDIO-003], [SRC-UI-018]
- **Version Notes:** Concurrency setting surfaces differ by target.

### Question: How do Niagara bounds cause disappearing effects?

- **Category:** Niagara / Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Incorrect fixed/dynamic bounds can cause frustum/cull decisions to remove an effect even though particles should be visible.
- **Strong 3-Year-Engineer Answer:** Niagara culling depends on System/Component state, bounds, significance and Effect Type policy. If bounds are too small or not updated for GPU particles/large motion, the renderer may cull the effect. I inspect bounds in Niagara Debugger/FX Outliner, force visible bounds tests, compare CPU/GPU sim path and verify Effect Type scalability before changing spawn logic.
- **Common Weak Answer:** "Increase spawn rate or lifetime."
- **Follow-up Questions:** GPU particles? fixed bounds? significance? Effect Type?
- **Hands-on Verification Task:** Build a projectile trail with intentionally tiny bounds and fix it with debugger evidence.
- **Sources:** [SRC-FX-003], [SRC-FX-005], [SRC-FX-004]
- **Version Notes:** Niagara debugger panels and GPU readback features vary.

### Question: Why are Niagara Data Channels not authoritative gameplay storage?

- **Category:** Niagara / Gameplay Boundary
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** They are useful game-to-VFX/System data paths, not durable replicated gameplay state or validation authority.
- **Strong 3-Year-Engineer Answer:** Niagara Data Channels can efficiently feed repeated impacts or gameplay data into VFX, but they are presentation-facing and feature/version-sensitive. Gameplay collision, damage, quest and inventory state should remain in authoritative gameplay systems. I treat channel records as visual inputs; disabling Niagara must not break gameplay truth.
- **Common Weak Answer:** "Use a data channel to store impact events for gameplay and VFX."
- **Follow-up Questions:** replication? persistence? readback? culling? authority?
- **Hands-on Verification Task:** Disable the Niagara System consuming channel data and prove damage/quest logic still works.
- **Sources:** [SRC-FX-007], [SRC-FX-005], [SRC-NET-001]
- **Version Notes:** Data Channel APIs and behavior are version-sensitive.

### Question: How do you compute a signed angle safely in Unreal?

- **Category:** Game Maths / Vectors
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Use dot for magnitude, cross/axis for sign, clamp dot for `acos`, and state coordinate/axis conventions.
- **Strong 3-Year-Engineer Answer:** I normalise valid input vectors, clamp dot to `[-1,1]` before inverse cosine, and use the sign of `Dot(Cross(A,B), ReferenceAxis)` to orient the result. In Unreal I state left-handed, Z-up and which axis defines "positive." For near-zero vectors or nearly parallel cases, I use tolerances and fallbacks rather than letting NaNs propagate.
- **Common Weak Answer:** "Use `acos(dot)`."
- **Follow-up Questions:** sign axis? zero vector? clamp? degrees/radians?
- **Hands-on Verification Task:** Test signed yaw from forward to target in all quadrants and near zero-length input.
- **Sources:** [SRC-MATH-001], [SRC-MATH-003], [SRC-MATH-008]
- **Version Notes:** Exact helper names vary; maths convention is stable.

### Question: What does "shortest path" mean for quaternion interpolation?

- **Category:** Game Maths / Rotation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Because `q` and `-q` represent the same orientation, interpolation should choose the hemisphere that avoids the long rotation path when desired.
- **Strong 3-Year-Engineer Answer:** Quaternions have a double-cover property: opposite signs can represent the same orientation. If interpolation uses the wrong hemisphere, it can rotate the long way. I choose the correct Slerp/normalized Slerp helper or flip sign based on dot product where appropriate, then test 179/181-degree and wrap-around cases. I also distinguish constant-speed interpolation from exponential smoothing.
- **Common Weak Answer:** "Use Slerp to avoid gimbal lock."
- **Follow-up Questions:** `q` vs `-q`? normalisation? constant speed? Rotator conversion?
- **Hands-on Verification Task:** Interpolate camera orientation across a 180-degree boundary and inspect path.
- **Sources:** [SRC-MATH-006], [SRC-MATH-007], [SRC-MATH-008]
- **Version Notes:** `TQuat` source is UE5.5-pinned; verify target helpers.

### Question: What makes an A* heuristic safe?

- **Category:** Algorithms / Pathfinding
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It should not overestimate remaining cost for optimality; consistency further preserves efficient closed-set behavior.
- **Strong 3-Year-Engineer Answer:** A* prioritises `g + h`. If `h` overestimates, A* may find a path quickly but not the optimal path. On grids, Manhattan, octile or Euclidean heuristics must match movement rules and costs. I test against Dijkstra on generated cases and keep the heuristic admissible/consistent unless the design intentionally accepts weighted/suboptimal A* for speed.
- **Common Weak Answer:** "Use distance to target."
- **Follow-up Questions:** diagonal movement? weighted terrain? consistency? tie-break?
- **Hands-on Verification Task:** Compare A* to Dijkstra across random weighted grids and detect counterexamples.
- **Sources:** [SRC-ALG-007], [SRC-ALG-008], [SRC-AI-007]
- **Version Notes:** Algorithmic; Unreal NavMesh path cost uses engine-specific data.

### Question: Why does union-find not solve dynamic connectivity with deletions?

- **Category:** Algorithms / Data Structures
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Union-find efficiently merges sets and queries connectivity, but it does not support splitting components after edge deletion.
- **Strong 3-Year-Engineer Answer:** Disjoint-set union is excellent for incremental connectivity: union components and find representatives with path compression/weighted union. Once merged, it has no cheap "un-union" operation. If my game needs doors/bridges closing and dynamic obstacle deletion, I need another structure, rebuild strategy, graph search, dynamic connectivity algorithm or domain-specific partition update.
- **Common Weak Answer:** "Use union-find for all connectivity."
- **Follow-up Questions:** deletion? path compression? offline queries? rebuild threshold?
- **Hands-on Verification Task:** Add/removing doors in a grid and compare DSU rebuild against BFS queries.
- **Sources:** [SRC-ALG-011], [SRC-ALG-010]
- **Version Notes:** Algorithmic; workload decides implementation.

### Question: What is false sharing and why can it matter in games?

- **Category:** C++ / Performance / Concurrency
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Independent variables on the same cache line are written by different cores, causing cache-line bouncing despite no logical sharing.
- **Strong 3-Year-Engineer Answer:** CPU caches operate at cache-line granularity. If worker threads repeatedly write adjacent counters or component data in the same line, cores invalidate each other's cache lines even though the variables are logically independent. In games this can appear in job counters, per-thread accumulators or hot SoA writes. I measure before padding/alignment and verify on target hardware because constants and effects vary.
- **Common Weak Answer:** "False sharing is when threads share data accidentally."
- **Follow-up Questions:** cache line? padding? per-thread buffers? target benchmark?
- **Hands-on Verification Task:** Benchmark adjacent atomic counters versus padded per-thread counters.
- **Sources:** [SRC-CPP-020], [SRC-CPP-023], [SRC-PERF-003]
- **Version Notes:** Hardware/cache-line details are platform-specific.

### Question: What is the trap with `std::pmr::monotonic_buffer_resource`?

- **Category:** C++ / Allocators
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It can make repeated allocations cheap but releases memory in bulk; it is not a general lifetime or destructor policy.
- **Strong 3-Year-Engineer Answer:** Monotonic resources are good for phase-scoped temporary allocations where you can discard the whole arena. They are poor for long-lived mixed lifetimes because individual frees do not reclaim space. Object destructors and semantic cleanup still matter, and thread safety must be checked. In Unreal I also compare against engine allocators, `TArray` reserve patterns and frame/operation scratch buffers.
- **Common Weak Answer:** "Use PMR to avoid allocations."
- **Follow-up Questions:** bulk lifetime? destructors? thread safety? Unreal allocator fit?
- **Hands-on Verification Task:** Build a per-frame parser using monotonic storage and show why it fails for retained objects.
- **Sources:** [SRC-CPP-024], [SRC-CPP-001]
- **Version Notes:** C++17+ and project toolchain dependent.

### Question: Why is accessing UObjects from arbitrary async tasks dangerous?

- **Category:** UE C++ / Threading / UObject
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** UObject lifetime, GC, World ownership and many engine APIs are game-thread-bound unless explicitly documented otherwise.
- **Strong 3-Year-Engineer Answer:** An async task can outlive the UObject, run during world teardown or touch APIs that are not thread-safe. I copy plain data for worker work, use weak object handles validated back on the game thread, cancel on owner teardown and route UObject mutation through documented game-thread paths. Task handle release is not cancellation, and GC visibility/lifetime must be explicit.
- **Common Weak Answer:** "Use `AsyncTask` and check `IsValid`."
- **Follow-up Questions:** GC? world teardown? task cancellation? data snapshot?
- **Hands-on Verification Task:** Start async work from an Actor, destroy/unload it, and repair stale callback access.
- **Sources:** [SRC-CPP-026], [SRC-EPIC-009], [SRC-EPIC-005]
- **Version Notes:** Task API and thread-safety rules are branch-sensitive.

### Question: What makes Lua hotfix reload risky?

- **Category:** Lua / Runtime Integration
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Old closures, wrappers, callbacks and schema assumptions can survive reload unless generation, invalidation and migration are explicit.
- **Strong 3-Year-Engineer Answer:** Replacing a Lua file does not automatically update old closures, object wrappers or registered callbacks. I stamp runtime generation, reject stale callbacks, unregister delegates/timers, migrate state by stable IDs and coordinate multiplayer/server versions. Hotfix is a transaction: quiesce, load, validate, migrate, resume or rollback. It is plugin/studio-specific, not baseline Unreal.
- **Common Weak Answer:** "Reload the module and scripts update."
- **Follow-up Questions:** old closures? UObject wrappers? multiplayer version? rollback?
- **Hands-on Verification Task:** Reload a Lua ability while callbacks are pending and prove old generation is rejected.
- **Sources:** [SRC-SCRIPT-001], [SRC-SCRIPT-002], [SRC-SCRIPT-003]
- **Version Notes:** Plugin, Lua version and platform support are exact-commit sensitive.

### Question: Why can C# Native AOT or trimming be a problem for Unreal scripting plugins?

- **Category:** C# / Interop / Deployment
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** AOT/trimming can remove or restrict dynamic reflection/loading patterns that many managed binding systems rely on.
- **Strong 3-Year-Engineer Answer:** Managed Unreal integrations often depend on reflection metadata, generated bindings, dynamic loading, callbacks and runtime hosting. Native AOT/trimming can improve deployment characteristics but may break dynamic code paths unless the plugin explicitly supports it. Console/mobile support also depends on platform policy and plugin implementation. I would prototype the exact plugin/commit/platform rather than assuming AOT solves shipping.
- **Common Weak Answer:** "Use Native AOT for consoles."
- **Follow-up Questions:** reflection? hostfxr? callbacks? platform policy? trimming annotations?
- **Hands-on Verification Task:** Identify one managed binding path that relies on reflection and explain its AOT risk.
- **Sources:** [SRC-SCRIPT-005], [SRC-SCRIPT-007], [SRC-SCRIPT-009], [SRC-SCRIPT-010]
- **Version Notes:** Third-party plugin and platform specific.

### Question: Why should Runtime Data Layers not be the sole quest source of truth?

- **Category:** World Partition / Gameplay State
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Data Layers control world streaming/visibility state; durable quest/save/server state should own the gameplay phase and drive layer state.
- **Strong 3-Year-Engineer Answer:** A Runtime Data Layer can represent "village restored" content visibility, but the quest/save/server model owns whether the village is restored. The coordinator applies Data Layer state, waits for readiness and handles load/late join/rollback. If the Data Layer is treated as truth, save migration, replication and content editing become fragile. I distinguish Data Layer Assets, Instances and durable gameplay state.
- **Common Weak Answer:** "Put quest phases in Data Layers."
- **Follow-up Questions:** save/load? late join? editor instance? streaming readiness?
- **Hands-on Verification Task:** Save a world phase, reload, and apply Data Layer state from durable quest state.
- **Sources:** [SRC-WORLD-001], [SRC-ASSET-010], [SRC-NET-001]
- **Version Notes:** Data Layer APIs and runtime behavior are target-sensitive.

### Question: What owns PCG-generated output at runtime?

- **Category:** PCG / Runtime Ownership
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** The project must define whether output is editor-baked, generated by a PCG Component, owned by spawned Actors/components, or converted into durable gameplay state.
- **Strong 3-Year-Engineer Answer:** PCG graph execution is not the same as gameplay ownership. Generated meshes/instances/Actors can affect navigation, collision, lighting, streaming and saves. If runtime generation is used, the system needs cleanup, determinism/seed policy, replication/save policy and authority. Often PCG should create authored/baked content, while gameplay systems own durable runtime state.
- **Common Weak Answer:** "The PCG graph owns what it spawns."
- **Follow-up Questions:** cleanup? save? replication? World Partition? nav/collision?
- **Hands-on Verification Task:** Generate interactable props and define which are saved, replicated, cleaned up or editor-only.
- **Sources:** [SRC-PCG-001], [SRC-PCG-004], [SRC-PCG-005]
- **Version Notes:** PCG runtime behavior is plugin/version sensitive.

### Question: How do you distinguish device-lab infrastructure failure from product failure?

- **Category:** Device Lab / Triage
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use run manifests, artifact identity, first-cause logs and rerun/quarantine policy to classify lab versus build/game regressions.
- **Strong 3-Year-Engineer Answer:** A failed automated target run is not automatically a product bug. I inspect build/package ID, device ID, install/deploy logs, launch permissions, scenario markers, app logs, CSV flush, crash symbols and lab runner health. Gate states should include pass, product fail, lab infrastructure fail, flaky quarantine, reporting-only metric and missing evidence. Reruns must not hide deterministic failures.
- **Common Weak Answer:** "Rerun the failed device."
- **Follow-up Questions:** artifact identity? first causal log? quarantine? missing CSV?
- **Hands-on Verification Task:** Classify three failures: install failure, deterministic crash and thermal flaky run.
- **Sources:** [SRC-PLAT-009], [SRC-PERF-009], [SRC-BUILD-018]
- **Version Notes:** Automation tooling is branch/platform sensitive.

### Question: Why do clean-agent build failures matter?

- **Category:** Build / Release Automation
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** They reveal undeclared dependencies, stale caches, local environment assumptions and missing staged artifacts before release.
- **Strong 3-Year-Engineer Answer:** A developer machine can hide missing SDKs, generated files, DDC/cache state, local plugins, environment variables, editor-only modules or unstaged runtime DLLs. Clean-agent failures prove the release recipe is incomplete. I compare local and clean logs by first cause, archive command line/manifests/artifacts, and fix the dependency declaration or staging rule rather than copying files manually.
- **Common Weak Answer:** "CI machine is missing setup."
- **Follow-up Questions:** BuildGraph? RunUAT? staged runtime dependency? cache masking?
- **Hands-on Verification Task:** Remove a locally available runtime DLL from staging and catch it on clean package.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016], [SRC-BUILD-018]
- **Version Notes:** BuildGraph/RunUAT behavior is branch/toolchain sensitive.

## NetworkPrediction Rollback Proof Expansion Set

### Question: Does the NetworkPrediction plugin mean Unreal has automatic full-game rollback?

- **Category:** Networking / NetworkPrediction
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** No. It exposes networked simulation/rollback vocabulary and services, but each feature still needs bounded state, replay-safe inputs, authority validation, side-effect control and target-branch proof.
- **Strong 3-Year-Engineer Answer:** The public API page shows a runtime NetworkPrediction plugin with model/component vocabulary, rollback/interpolation/smoothing services, input/sync/aux/cue state concepts, buffers and reconcile CVars. That does not make the whole Actor or game automatically rollback-capable. I would prove exact target APIs in source, define a small simulation model, serialise input/sync/aux state, validate on the server, dedupe cues and measure buffer/replay cost before choosing it.
- **Common Weak Answer:** "UE has rollback if you enable the plugin."
- **Follow-up Questions:** What state rewinds? What does a cue do? What needs source proof?
- **Hands-on Verification Task:** Produce a target-branch plugin evidence packet before writing model code.
- **Sources:** [SRC-NET-020], [SRC-NET-001]
- **Version Notes:** Plugin API is maintained-doc/source-sensitive for UE5.3-UE5.6 targets.

### Question: What is the difference between CharacterMovement prediction and full rollback?

- **Category:** Networking / Prediction Terminology
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** CharacterMovement prediction is a specialised movement pipeline; full rollback restores a bounded simulation state and resimulates forward.
- **Strong 3-Year-Engineer Answer:** CharacterMovement already supports client prediction, server reproduction, correction, replay and smoothing for character movement. Full rollback is a broader architecture: state snapshots, input logs, replay windows, side-effect dedupe and deterministic-enough simulation for a bounded model. I would not call saved-move correction full-game rollback, and I would not use a rollback plugin to replace basic CharacterMovement unless the feature requires it.
- **Common Weak Answer:** "They are the same because both replay."
- **Follow-up Questions:** What does saved move store? What does sync state store? What about VFX?
- **Hands-on Verification Task:** Compare a predicted sprint saved move with a rollback dash model.
- **Sources:** [SRC-NET-009], [SRC-NET-018], [SRC-NET-020]
- **Version Notes:** Saved-move and plugin APIs must be checked in target source.

### Question: What belongs in a rollback input command?

- **Category:** Networking / Rollback State Design
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Compact player intent for one frame/tick: quantised axes, action bits, sequence/frame and small validated parameters.
- **Strong 3-Year-Engineer Answer:** Input commands should be replayable intent, not client authority. I include pressed/released bits, quantised movement/aim, selected action/slot and local frame. I avoid world pointers, client hit results as truth, random outputs, current animation state and large payloads. The server still validates cooldown, resource, range, target state and timestamp bounds from authority state.
- **Common Weak Answer:** "Put whatever the client did in the command."
- **Follow-up Questions:** Can it contain an Actor pointer? What about a hit result? How compress?
- **Hands-on Verification Task:** Design a dash input command and reject one impossible client request.
- **Sources:** [SRC-NET-003], [SRC-NET-006], [SRC-NET-020]
- **Version Notes:** Exact serialisation helpers are plugin/branch-sensitive.

### Question: What belongs in rollback sync state?

- **Category:** Networking / Rollback State Design
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Frame-to-frame simulation truth that clients compare against authority and replay from.
- **Strong 3-Year-Engineer Answer:** Sync state should include fields that affect future simulation and correction: transform or local sim position, velocity, phase/mode, remaining dash ticks, authoritative frame and any compact state needed for deterministic continuation. If a field affects the next tick but is hidden in an Actor component, resimulation can look correct in one frame and diverge later.
- **Common Weak Answer:** "Only replicate the final transform."
- **Follow-up Questions:** Where does cooldown go? What about velocity? What tolerance?
- **Hands-on Verification Task:** Remove one movement-affecting sync field and show false convergence or repeated reconcile.
- **Sources:** [SRC-NET-020], [SRC-NET-009]
- **Version Notes:** Sync comparison APIs are target-branch details.

### Question: What belongs in rollback aux state?

- **Category:** Networking / Rollback State Design
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Slower-changing simulation inputs such as approved tunables, equipment modifiers, tag/resource snapshots or table versions.
- **Strong 3-Year-Engineer Answer:** Aux state captures the data the simulation needs but does not produce every tick. The trap is reading current equipment/tunables during replay of old frames. I version aux state, log it with each input, and define whether pending inputs replay with old aux values or get rejected on aux changes.
- **Common Weak Answer:** "Read the current data asset during replay."
- **Follow-up Questions:** Equipment swap during prediction? Data table change? Old aux missing?
- **Hands-on Verification Task:** Swap weapon movement modifiers during a predicted dash and prove replay policy.
- **Sources:** [SRC-NET-020], [SRC-NET-005]
- **Version Notes:** Aux buffer implementation details are source-sensitive.

### Question: Why are rollback side effects dangerous?

- **Category:** Networking / Rollback Side Effects
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Prediction and replay can execute the same tick multiple times, duplicating effects unless they are cue-managed, deduped or moved to authority commit.
- **Strong 3-Year-Engineer Answer:** The simulation tick may run locally, get corrected, then replay pending frames. If it directly spawns VFX, grants rewards, applies damage or writes analytics, those effects can happen more than once. I classify side effects: replay-safe cues with deterministic keys, local weak presentation, replicated presentation or one-time authority transactions outside the resimulated path.
- **Common Weak Answer:** "Just check if server."
- **Follow-up Questions:** What about local VFX? Damage? Analytics? Camera shake?
- **Hands-on Verification Task:** Force reconcile on a dash-start frame and prove no duplicated VFX/audio/rewards.
- **Sources:** [SRC-NET-020], [SRC-NET-001]
- **Version Notes:** Cue traits and exact dedupe hooks are target-branch details.

### Question: How would you prove a rollback dash converges?

- **Category:** Networking / Rollback Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Force divergence, log restored frame and field diff, replay pending inputs and prove final state matches authority within tolerance.
- **Strong 3-Year-Engineer Answer:** I build a compact dash model, run a dedicated server with lag/loss, add client-only acceleration bias and a server-only blocker, then force/print reconciles. Logs should show input hash, aux version, authority frame, field diffs, restored frame, replay window and post-replay sync hash. I also verify cue dedupe, rejection UX and smoothing on/off so correctness is not hidden by presentation.
- **Common Weak Answer:** "Try it online and see if it feels smooth."
- **Follow-up Questions:** What fields log? What tolerance? What about smoothing?
- **Hands-on Verification Task:** Create a reconcile log packet for a forced location error.
- **Sources:** [SRC-NET-020], [SRC-NET-010]
- **Version Notes:** CVar names and trace output vary by target build.

### Question: What causes reconcile every frame in a rollback model?

- **Category:** Networking / Rollback Debugging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Tick mismatch, nondeterministic simulation, missing state, overly strict tolerance, aux mismatch or unreproduced collision/query results.
- **Strong 3-Year-Engineer Answer:** I disable smoothing, print reconcile details, compare field diffs, input hashes, aux versions and time step. Then I simplify: no collision, no cues, no random, fixed input. Reconcile every frame usually means client and server are not actually simulating the same model, or the comparison tolerances/payload omit a field that changes future state.
- **Common Weak Answer:** "Increase tolerance."
- **Follow-up Questions:** Which field diverges first? Does smoothing hide it? Are inputs identical?
- **Hands-on Verification Task:** Add a client-only friction value and identify it through sync/aux logs.
- **Sources:** [SRC-NET-020], [SRC-NET-009]
- **Version Notes:** Debug CVar availability is target-sensitive.

### Question: Why should a rollback tick avoid reading current Actor state directly?

- **Category:** Networking / Rollback Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Replay of old frames can accidentally use new Actor state, causing nondeterministic divergence.
- **Strong 3-Year-Engineer Answer:** A rollback tick must replay frame N using frame-N-relevant input, sync and aux state. If it reads current weapon component, current tag set, current animation state or current world query output, replay is contaminated by later state. Actor state should be bridged into versioned model inputs or aux state, and durable mutations should happen at explicit authority boundaries.
- **Common Weak Answer:** "The Actor has the latest data, so use it."
- **Follow-up Questions:** Equipment swap? Respawn? changing tags? streaming actor?
- **Hands-on Verification Task:** Replay old dash frames after equipment swap and compare direct Actor read versus aux snapshot.
- **Sources:** [SRC-NET-020], [SRC-NET-005]
- **Version Notes:** Actor/model bridge APIs are project-specific.

### Question: How do NetSimCue-style events differ from gameplay transactions?

- **Category:** Networking / Cues
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Cues are simulation notifications/presentation events; durable gameplay transactions must be authority-owned and replay-safe.
- **Strong 3-Year-Engineer Answer:** The plugin API exposes cue traits around invocation, replication and resimulation behaviour. I use cues for VFX, sound, camera or UI notification after deciding replay/dedupe rules. I do not grant items, score, damage or achievements from cue handlers, because replay/correction and late delivery can duplicate or desynchronise durable state.
- **Common Weak Answer:** "Use a cue for every event."
- **Follow-up Questions:** Weak cue? replicated cue? reward? late join?
- **Hands-on Verification Task:** Convert dash VFX to a keyed cue and move damage to an authority commit.
- **Sources:** [SRC-NET-020], [SRC-NET-001]
- **Version Notes:** Cue trait names and requirements require branch source proof.

### Question: How do you decide rollback buffer length?

- **Category:** Networking / Performance
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** From maximum correction window, RTT/jitter/loss policy, replay cost, memory budget and gameplay tolerance.
- **Strong 3-Year-Engineer Answer:** I calculate input/sync/aux/cue memory per frame per instance, then multiply by desired history window and model count. The window must cover realistic correction delay without enabling old exploit requests or excessive replay bursts. I measure under packet lag/loss and define fail thresholds for replay frames per correction, correction storm CPU and memory.
- **Common Weak Answer:** "Keep a few seconds just in case."
- **Follow-up Questions:** How many instances? What state size? What max replay burst?
- **Hands-on Verification Task:** Compare 30/60/120-frame buffers at 64 model instances.
- **Sources:** [SRC-NET-020], [SRC-NET-010]
- **Version Notes:** Buffer implementations and defaults are target-sensitive.

### Question: When should you reject rollback and use a simpler networking model?

- **Category:** Networking / Architecture Choice
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** When state is large/nondeterministic, responsiveness gain is low, side effects are irreversible, or tooling/source proof is too expensive.
- **Strong 3-Year-Engineer Answer:** Rollback has a cost: model isolation, serialisation, deterministic-enough simulation, buffers, cue dedupe, correction UX and debugging. I would prefer server-authoritative commands for inventory/quests, CharacterMovement saved moves for movement-affecting character flags, server rewind for hitscan validation, and interpolation for remote presentation when full resimulation is unnecessary.
- **Common Weak Answer:** "Rollback is always better."
- **Follow-up Questions:** Inventory? ragdoll physics? hitscan? dash?
- **Hands-on Verification Task:** Write an adopt/reject memo comparing rollback dash against CharacterMovement saved moves.
- **Sources:** [SRC-NET-009], [SRC-NET-020]
- **Version Notes:** Project needs and branch plugin status drive the decision.

### Question: How does rollback interact with server rewind?

- **Category:** Networking / Lag Compensation
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Rewind queries historical server state for validation; rollback restores model state and resimulates. They can coexist but are not the same.
- **Strong 3-Year-Engineer Answer:** For hitscan, the server may rewind target hitboxes to evaluate a shot timestamp. That does not require restoring and resimulating the shooter's whole gameplay model. A rollback movement model might also keep history, but its purpose is correction and replay of predicted simulation. I keep the terms separate so fairness, security and state ownership are clear.
- **Common Weak Answer:** "Lag compensation is rollback."
- **Follow-up Questions:** What is restored? What is queried? Who validates timestamp?
- **Hands-on Verification Task:** Document a dash rollback path and a hitscan rewind path side by side.
- **Sources:** [SRC-NET-009], [SRC-NET-006], [SRC-NET-020]
- **Version Notes:** Rewind implementation is game-specific unless using a target subsystem.

### Question: What must be logged in a rollback correction?

- **Category:** Networking / Observability
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Instance, role, frame, input hash, aux version, authority sync diff, restored frame, replay window, post-replay hash and cue suppression.
- **Strong 3-Year-Engineer Answer:** A useful correction log says why correction happened and whether recovery worked. I want field-level diffs rather than "mismatch", the input/aux context that produced the mismatch, replay cost, final convergence and side-effect handling. Without those fields, rollback bugs become invisible snapbacks or vague "feels bad" reports.
- **Common Weak Answer:** "Log when correction happens."
- **Follow-up Questions:** Which field diverged? Did cues duplicate? How many frames replayed?
- **Hands-on Verification Task:** Implement a reconcile log format and use it on a forced dash divergence.
- **Sources:** [SRC-NET-020], [SRC-NET-010]
- **Version Notes:** Trace/log APIs are project-specific.

### Question: Why can smoothing be dangerous during rollback debugging?

- **Category:** Networking / Debugging
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It can hide frequent corrections, making a broken simulation look acceptable until CPU/bandwidth/edge cases fail.
- **Strong 3-Year-Engineer Answer:** Smoothing is a UX layer after correction, not proof of correctness. I disable smoothing/interpolation during debugging where possible, inspect model sync diffs and reconcile counts, then re-enable smoothing to evaluate feel. A feature can look smooth while reconciling every frame and burning replay CPU.
- **Common Weak Answer:** "If it looks smooth, networking is correct."
- **Follow-up Questions:** Which CVars? What metrics? What threshold fails?
- **Hands-on Verification Task:** Show the same correction storm with smoothing off and on.
- **Sources:** [SRC-NET-020], [SRC-NET-009]
- **Version Notes:** CVar names and runtime availability vary.

### Question: How should rollback handle respawn or destruction?

- **Category:** Networking / Lifetime
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Invalidate model instance IDs, pending inputs, cue histories and stale target identities across lifetime boundaries.
- **Strong 3-Year-Engineer Answer:** Respawn is a generation boundary. Old predicted inputs, target references and cue dedupe keys must not apply to the new life. I include generation/stable IDs in commands and cues, flush or reject pending frames at death/respawn, and reconstruct durable state for late joiners instead of replaying historical predicted presentation.
- **Common Weak Answer:** "Reset the Actor."
- **Follow-up Questions:** stale target ID? old cue? pending dash? late join?
- **Hands-on Verification Task:** Kill and respawn during a pending dash prediction and prove stale replay is rejected.
- **Sources:** [SRC-NET-001], [SRC-NET-007], [SRC-NET-020]
- **Version Notes:** Model lifetime management is project/branch-specific.

### Question: How do you prevent Actor replication from fighting rollback sync state?

- **Category:** Networking / Actor Bridge
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Assign one writer for each field and separate model-owned state, durable replicated state and presentation state.
- **Strong 3-Year-Engineer Answer:** If normal Actor replication writes transform or phase while the rollback model also corrects it, the client can oscillate or hide true authority. I document model-owned fields, Actor durable fields, presentation-only fields and authority transactions. Relevancy/dormancy/lifetime must be compatible with correction data and late-join reconstruction.
- **Common Weak Answer:** "Replicate the Actor as usual and also run rollback."
- **Follow-up Questions:** Transform owner? dormancy? late join? presentation?
- **Hands-on Verification Task:** Add a competing replicated transform and capture the resulting correction conflict.
- **Sources:** [SRC-NET-004], [SRC-NET-007], [SRC-NET-008], [SRC-NET-020]
- **Version Notes:** Exact bridge design is project-specific.

### Question: What makes a rollback simulation deterministic enough?

- **Category:** Networking / Determinism
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Controlled time step, identical inputs/aux state, stable query results, bounded numeric tolerance and no hidden mutable state.
- **Strong 3-Year-Engineer Answer:** It does not have to be mathematically perfect for every platform if reconciliation tolerances and authority correction are acceptable, but it must be reproducible enough across the replay window. I remove wall-clock reads, uncontrolled randomness, current Actor reads, unordered query dependence and direct side effects. Then I prove divergence rate under packet conditions and platform builds.
- **Common Weak Answer:** "Use fixed tick and it is deterministic."
- **Follow-up Questions:** randomness? collision? floating point? data versions?
- **Hands-on Verification Task:** Introduce nondeterministic query ordering and show reconcile frequency rising.
- **Sources:** [SRC-NET-020], [SRC-NET-009]
- **Version Notes:** Determinism tolerance is feature/platform-specific.

### Question: What is a good first rollback proof feature?

- **Category:** Networking / Scope Control
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** A compact deterministic movement/action micro-simulation such as dash, not ragdoll combat or inventory.
- **Strong 3-Year-Engineer Answer:** A good first proof has small input, small sync state, obvious responsiveness benefit, controlled side effects and easy forced divergence. Dash or charge movement is better than full physics ragdoll, inventory transactions or complex AI. The aim is to prove the tooling and failure model before expanding scope.
- **Common Weak Answer:** "Start with the hardest combat system."
- **Follow-up Questions:** Why not inventory? Why not ragdoll? What side effects?
- **Hands-on Verification Task:** Build a toy counter model before dash.
- **Sources:** [SRC-NET-020], [SRC-NET-009]
- **Version Notes:** Feature fit depends on product and branch plugin support.

### Question: How should client-provided target data work in rollback-style actions?

- **Category:** Networking / Security
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Treat it as intent/evidence; validate range, line of sight, timing, state and plausibility on the server.
- **Strong 3-Year-Engineer Answer:** Prediction can show local feedback, but the server decides. I bound the timestamp/frame, validate ownership, cooldown, resources, target generation, range/LOS and movement plausibility, and ignore client hit results as authoritative truth. If the action is rejected, rollback/reconcile returns presentation to a valid state.
- **Common Weak Answer:** "The client predicted it, so accept it."
- **Follow-up Questions:** stale target? old timestamp? occlusion? respawn?
- **Hands-on Verification Task:** Send a stale target ID after respawn and prove server rejection plus client cleanup.
- **Sources:** [SRC-NET-003], [SRC-NET-006], [SRC-NET-020]
- **Version Notes:** Validation policy is game-specific.

### Question: How do you profile rollback cost?

- **Category:** Networking / Performance
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Measure command bandwidth, correction payloads, replay CPU, buffer memory, cue history and correction frequency under packet conditions.
- **Strong 3-Year-Engineer Answer:** I treat rollback as network plus CPU/memory architecture. I measure input send rate/size, sync/aux correction size, cue payloads, reconciles per minute, replay frames per correction, model tick/replay CPU, buffer memory per instance and worst-case correction storm. Then I compare against simpler alternatives before adopting the plugin.
- **Common Weak Answer:** "Profile frame rate only."
- **Follow-up Questions:** memory per instance? replay burst? cue history? network capture?
- **Hands-on Verification Task:** Profile 1/16/64/128 rollback dash instances with forced reconcile every N frames.
- **Sources:** [SRC-NET-010], [SRC-NET-020]
- **Version Notes:** Tooling differs by target build and platform.

### Question: How do you answer "what proof would convince you to ship rollback?"

- **Category:** Networking / Production Readiness
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Source-backed APIs, deterministic-enough model, forced reconcile proof, side-effect safety, measured budgets, packet-condition tests and a simpler-alternative comparison.
- **Strong 3-Year-Engineer Answer:** I need a branch evidence packet, compile proof, automated lag/loss tests, forced divergence/reconcile logs, field-level diffing, cue dedupe proof, invalid-command rejection, no irreversible side effects in replay, memory/CPU/bandwidth budgets, smoothing UX checks and an adopt/reject memo versus saved moves, server-authoritative commands and server rewind. Without those, rollback remains an interesting prototype.
- **Common Weak Answer:** "A demo that feels good."
- **Follow-up Questions:** What metrics? What failure injection? What alternative?
- **Hands-on Verification Task:** Produce a one-page ship/no-ship rollback evidence memo after the workbook labs.
- **Sources:** [SRC-NET-020], [SRC-NET-010], [SRC-NET-009]
- **Version Notes:** Production readiness is target/platform/team dependent.

## Core Systems and Production Debugging Expansion Set

### Question: Why can a reflected C++ property still fail to show up or replicate as expected?

- **Category:** UE C++ / Reflection
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Reflection, editor exposure, Blueprint access, serialisation and replication are separate systems controlled by different specifiers and registration paths.
- **Strong 3-Year-Engineer Answer:** `UPROPERTY` makes a field visible to Unreal's reflection system, but editor details, Blueprint read/write, config/save behaviour and replication each need the correct specifier and owning class setup. For replication I still need lifetime registration and authority-owned mutation; for editor visibility I need details/category/edit flags. I debug by checking generated header/UHT output, class defaults, details panel, lifetime props and network traces separately.
- **Common Weak Answer:** "Add UPROPERTY and Unreal handles it."
- **Follow-up Questions:** UHT? `DOREPLIFETIME`? BlueprintReadOnly? SaveGame?
- **Hands-on Verification Task:** Create one reflected field that is editor-visible but not replicated, then fix replication explicitly.
- **Sources:** [SRC-EPIC-001], [SRC-EPIC-002], [SRC-EPIC-007], [SRC-NET-004]
- **Version Notes:** Exact specifier combinations and validation warnings are branch-sensitive.

### Question: What is the difference between `Outer`, owner and attachment?

- **Category:** UObject / Lifetime
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** `Outer` is containment/name/lifetime context, owner is gameplay/network ownership, and attachment is scene hierarchy.
- **Strong 3-Year-Engineer Answer:** A UObject's Outer participates in object path/name context and often lifetime containment, but it is not the same as `AActor::Owner` or a component attachment parent. Network owning connection, GC reachability and transform hierarchy are separate. I avoid using one relationship to imply the others and log object path, owner, attach parent and reference chain when debugging lifetime bugs.
- **Common Weak Answer:** "Outer means owner."
- **Follow-up Questions:** GC reachability? RPC routing? scene transform? package path?
- **Hands-on Verification Task:** Create an Actor, Component and subobject with different Outer/Owner/Attach relationships and log all three.
- **Sources:** [SRC-EPIC-003], [SRC-EPIC-006], [SRC-EPIC-009], [SRC-NET-003]
- **Version Notes:** UObject lifetime and networking ownership details are source-sensitive.

### Question: Why is `UPROPERTY` not a magic fix for all dangling references?

- **Category:** UObject / GC
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It helps GC discover references, but lifetime, validity, weak references, pending kill/destruction and thread access still require design.
- **Strong 3-Year-Engineer Answer:** A reflected strong pointer can keep an object reachable; a weak pointer can null when the object is destroyed; raw pointers can become stale unless managed by a specific owner. But none of this makes an object semantically valid after level unload, actor destroy, pending kill or async completion. I design owner/lifetime contracts and validate on use, especially across timers, delegates and tasks.
- **Common Weak Answer:** "Use UPROPERTY and it cannot crash."
- **Follow-up Questions:** weak pointer? delegate? async task? level unload?
- **Hands-on Verification Task:** Fire an async callback after Actor destroy and fix it with weak pointer plus generation validation.
- **Sources:** [SRC-EPIC-005], [SRC-EPIC-006], [SRC-EPIC-009], [SRC-CPP-002]
- **Version Notes:** GC incrementality and validity helpers vary by branch.

### Question: How should you choose between `NewObject`, `CreateDefaultSubobject` and spawning an Actor?

- **Category:** UObject / Creation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Use `CreateDefaultSubobject` for class default subobjects in constructors, `NewObject` for UObjects/components with explicit ownership, and spawning for world Actors.
- **Strong 3-Year-Engineer Answer:** Default subobjects define class/component composition and should be created in constructors. Runtime UObjects need an appropriate Outer, reference path and initialisation contract. Actors are world entities with lifecycle, networking, transform and ticking semantics and should be spawned through the world. I choose based on identity, lifetime, transform/world presence and replication needs.
- **Common Weak Answer:** "They all create Unreal objects."
- **Follow-up Questions:** constructor? CDO? BeginPlay? replication? Outer?
- **Hands-on Verification Task:** Implement the same ability helper as a UObject, component and Actor and compare lifecycle/logs.
- **Sources:** [SRC-EPIC-004], [SRC-EPIC-011], [SRC-EPIC-012], [SRC-EPIC-015]
- **Version Notes:** Exact creation flags and deferred spawning APIs are branch-sensitive.

### Question: Why is GameMode not a good place for client HUD state?

- **Category:** Gameplay Framework / Authority
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** GameMode exists only on the server; client-visible match/player state belongs in replicated GameState/PlayerState or local UI models.
- **Strong 3-Year-Engineer Answer:** GameMode owns authoritative rules on the server. GameState replicates shared match state; PlayerState replicates per-player public persistent state; PlayerController/HUD/widgets own local presentation. Putting client HUD data in GameMode fails on clients and blurs authority/presentation boundaries.
- **Common Weak Answer:** "GameMode is global, so use it."
- **Follow-up Questions:** dedicated server? GameState? PlayerState? local-only private state?
- **Hands-on Verification Task:** Try to read GameMode from a remote client and replace the design with GameState/PlayerState.
- **Sources:** [SRC-EPIC-010], [SRC-EPIC-013], [SRC-EPIC-014], [SRC-NET-001]
- **Version Notes:** Gameplay framework roles are stable; exact call order is branch-sensitive.

### Question: What makes BeginPlay ordering a fragile dependency?

- **Category:** Gameplay Framework / Lifecycle
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Different Actors/components/world systems can begin at different times, and replication/possession/async loading may not be ready.
- **Strong 3-Year-Engineer Answer:** `BeginPlay` is useful, but it should not assume every other Actor, component, replicated pointer, subsystem or async asset is initialised. I use explicit readiness events, dependency registration, idempotent binding and late arrival handling. When debugging, I log lifecycle events with world, role, net mode and object identity.
- **Common Weak Answer:** "Everything is valid in BeginPlay."
- **Follow-up Questions:** replicated references? possession? streamed level? subsystem?
- **Hands-on Verification Task:** Reproduce a null dependency in BeginPlay with streamed Actors and fix it using readiness handshake.
- **Sources:** [SRC-EPIC-015], [SRC-EPIC-016], [SRC-EPIC-017], [SRC-ASSET-009]
- **Version Notes:** Lifecycle ordering needs target-branch verification for exact cases.

### Question: When is a subsystem the wrong abstraction?

- **Category:** Gameplay Architecture / Subsystems
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** When it hides ownership, authority, world/player scope or lifecycle rather than clarifying a service boundary.
- **Strong 3-Year-Engineer Answer:** Subsystems are good for scoped services with clear lifetime, but a subsystem is not a dumping ground for global state. I choose GameInstance/World/LocalPlayer/Engine scope based on data lifetime and audience, then expose narrow APIs. If state is per-match, per-player, replicated or save-specific, an Actor/Component/PlayerState/service object may be clearer.
- **Common Weak Answer:** "Put managers in subsystems."
- **Follow-up Questions:** world scope? local player? multiplayer? save state?
- **Hands-on Verification Task:** Move a global inventory manager into a scoped per-player service and explain lifetime changes.
- **Sources:** [SRC-EPIC-017], [SRC-ARCH-007], [SRC-ARCH-014]
- **Version Notes:** Subsystem classes and creation order are branch-sensitive.

### Question: How do BlueprintNativeEvent and BlueprintImplementableEvent differ in API design?

- **Category:** C++ / Blueprint Integration
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Native events provide a C++ default implementation; implementable events require Blueprint implementation and should not be mandatory for core invariants unless guarded.
- **Strong 3-Year-Engineer Answer:** I use native events when C++ can provide a safe baseline and Blueprint customises behaviour. Implementable events are useful for presentation hooks or designer-defined extension points, but core gameplay invariants need C++ authority checks and safe defaults. I avoid making critical server validation depend only on Blueprint override.
- **Common Weak Answer:** "They are the same but one has C++."
- **Follow-up Questions:** `_Implementation`? authority? designer extension? missing override?
- **Hands-on Verification Task:** Convert a required damage validation event into a native event with C++ fallback.
- **Sources:** [SRC-EPIC-001], [SRC-ARCH-013], [SRC-ARCH-014]
- **Version Notes:** Generated function details are UHT/branch-sensitive.

### Question: Why can Blueprint pure nodes cause performance or correctness issues?

- **Category:** C++ / Blueprint Integration
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** They may be evaluated multiple times and should be side-effect-free and cheap.
- **Strong 3-Year-Engineer Answer:** A pure node is an expression dependency, not a cached variable. If it does expensive work, queries mutable state or has hidden side effects, Blueprint graph evaluation can surprise you. I keep pure nodes deterministic, cheap and side-effect-free, and expose explicit impure commands for mutation or costly operations.
- **Common Weak Answer:** "Pure means faster."
- **Follow-up Questions:** repeated evaluation? side effects? caching? profiling?
- **Hands-on Verification Task:** Instrument an expensive pure node called from multiple graph pins and replace it with a cached impure call.
- **Sources:** [SRC-EPIC-001], [SRC-ARCH-013], [SRC-PERF-001]
- **Version Notes:** Blueprint compiler behaviour can vary; profile target graphs.

### Question: What is a safe pattern for C++ exposing mutable collections to Blueprint?

- **Category:** C++ / Blueprint API
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Expose snapshots, validated commands and change events rather than direct mutable internal arrays/maps.
- **Strong 3-Year-Engineer Answer:** Direct mutable collection exposure makes invariants, replication dirty marking, save identity and UI stale selection harder to control. I expose read-only snapshots or view models, centralise mutation through validated functions and emit events with stable IDs/revisions. For replicated collections, mutations must preserve identity and dirty-marking semantics.
- **Common Weak Answer:** "Make the array BlueprintReadWrite."
- **Follow-up Questions:** invariant? Fast Array? UI recycling? stable IDs?
- **Hands-on Verification Task:** Replace a BlueprintReadWrite inventory array with command/query APIs and revisioned events.
- **Sources:** [SRC-EPIC-007], [SRC-NET-015], [SRC-ARCH-012], [SRC-UI-013]
- **Version Notes:** Blueprint container exposure and replication helpers are branch-sensitive.

### Question: How do you debug a `stat unit` bottleneck result?

- **Category:** Profiling / CPU-GPU Triage
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Use it as a top-level split, then drill into game, draw/render, GPU, memory and subsystem-specific traces.
- **Strong 3-Year-Engineer Answer:** `stat unit` tells me whether frame time is dominated by Game, Draw/Render thread or GPU, but it does not identify root cause alone. I reproduce a stable scenario, capture Unreal Insights/stat groups/GPU captures as appropriate, and avoid optimising the wrong thread. I also check hitch timing, memory/IO and scalability/device profile settings.
- **Common Weak Answer:** "GPU is high, reduce textures."
- **Follow-up Questions:** game thread? render thread? GPU capture? hitch? scenario control?
- **Hands-on Verification Task:** Capture one CPU-bound and one GPU-bound frame and explain the next tool for each.
- **Sources:** [SRC-PERF-001], [SRC-PERF-002], [SRC-PERF-003], [SRC-RENDER-009]
- **Version Notes:** Stat names and Insights tracks vary by branch.

### Question: When do you use LLM versus Memory Insights?

- **Category:** Profiling / Memory
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** LLM gives tagged runtime memory budgets; Memory Insights helps inspect allocation behaviour and callstacks when instrumented.
- **Strong 3-Year-Engineer Answer:** I use LLM to track high-level memory categories over scenarios and enforce budgets, then Memory Insights to investigate allocation churn, callstacks and lifetime patterns. Memory debugging needs controlled maps, warm/cold runs, streaming/cook parity and platform target data, not only editor observations.
- **Common Weak Answer:** "They both show memory."
- **Follow-up Questions:** tags? callstacks? allocation churn? platform? streaming?
- **Hands-on Verification Task:** Use LLM to find a growing category, then Memory Insights to identify allocation source.
- **Sources:** [SRC-PERF-006], [SRC-PERF-010], [SRC-PERF-005], [SRC-ASSET-012]
- **Version Notes:** Instrumentation availability varies by platform/build.

### Question: Why can PSO compilation cause packaged-game hitches?

- **Category:** Rendering / PSO
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** New graphics/compute pipeline state combinations may compile or initialise during gameplay if not collected/prepared.
- **Strong 3-Year-Engineer Answer:** Materials, vertex factories, render states and shader permutations can produce PSOs. If a packaged build encounters uncached PSOs during play, it can hitch. I reproduce in packaged target, collect PSO data, build/ship caches according to target workflow, and verify with hitch traces and render captures rather than assuming editor shader compile equals runtime readiness.
- **Common Weak Answer:** "Shaders compiled, so PSOs are fine."
- **Follow-up Questions:** packaged build? material permutation? cache collection? hitch trace?
- **Hands-on Verification Task:** Trigger a new material/effect in a packaged run and compare with/without PSO cache workflow.
- **Sources:** [SRC-RENDER-012], [SRC-RENDER-013], [SRC-RENDER-014], [SRC-PERF-003]
- **Version Notes:** PSO cache workflows differ by platform/RHI/UE branch.

### Question: What is an RDG lifetime bug?

- **Category:** Rendering / RDG
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Using render graph resources outside their declared graph lifetime or missing dependencies between passes.
- **Strong 3-Year-Engineer Answer:** RDG tracks resources and pass dependencies. If I hold a graph texture/buffer beyond the graph, mutate external resources without registration, or omit dependency declarations, behaviour can be invalid or race-prone. I debug by simplifying passes, naming resources, using validation where available and checking extraction/external resource rules in the target branch.
- **Common Weak Answer:** "RDG is just a render command list wrapper."
- **Follow-up Questions:** extraction? external texture? pass dependency? validation?
- **Hands-on Verification Task:** Create a two-pass RDG effect with a deliberate missing dependency and fix it.
- **Sources:** [SRC-RENDER-003], [SRC-RENDER-013], [SRC-RENDER-014]
- **Version Notes:** RDG helper APIs are branch-sensitive.

### Question: When is Nanite the wrong answer?

- **Category:** Rendering / Geometry
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** When geometry/material/platform/animation/overdraw constraints do not match Nanite's strengths or support matrix.
- **Strong 3-Year-Engineer Answer:** Nanite helps with high-detail static-like geometry, but it is not a universal optimisation. I check platform/RHI support, material features, foliage/masked/translucent cost, deformation needs, instance count, fallback meshes, LOD/HLOD interaction and GPU captures. The right answer may be simpler LODs, HLOD, material reduction or culling.
- **Common Weak Answer:** "Turn on Nanite for all meshes."
- **Follow-up Questions:** masked foliage? skeletal mesh? platform? material cost? fallback?
- **Hands-on Verification Task:** Compare a high-poly opaque rock and masked foliage with Nanite on/off using GPU evidence.
- **Sources:** [SRC-RENDER-005], [SRC-RENDER-008], [SRC-RENDER-011], [SRC-RENDER-004]
- **Version Notes:** Nanite support evolves across UE5 versions and platforms.

### Question: How do soft references affect cooking?

- **Category:** Assets / Cooking
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Soft references avoid hard load dependencies, but assets still need a cook/staging discovery path through Asset Manager, labels, maps, rules or explicit loads.
- **Strong 3-Year-Engineer Answer:** A soft reference is a path that can be loaded later; it does not automatically guarantee the asset is cooked into every package. I define primary assets/rules or labels, validate references, test packaged builds and watch for editor-only paths. Debugging starts with asset registry, cook logs, reference viewer and package contents.
- **Common Weak Answer:** "Soft references always cook the asset."
- **Follow-up Questions:** Asset Manager? primary asset? chunk? editor-only? package test?
- **Hands-on Verification Task:** Soft-reference an icon that works in editor but fails in package, then fix cook rules.
- **Sources:** [SRC-ASSET-001], [SRC-ASSET-002], [SRC-ASSET-003], [SRC-ASSET-006]
- **Version Notes:** Cook rule details are project/branch-specific.

### Question: Why are redirectors dangerous if left unmanaged?

- **Category:** Assets / Source Control
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** They hide moved asset paths in editor but can create broken references, cook surprises and source-control churn if not fixed.
- **Strong 3-Year-Engineer Answer:** Redirectors help asset moves, but production needs fix-up and validation so references point at real assets. I run fix-up, validate references, review source control changes and test cook/package. Hidden redirectors can make local editor use succeed while clean cook or another branch fails.
- **Common Weak Answer:** "Redirectors are harmless."
- **Follow-up Questions:** fix-up? source control? package? asset registry?
- **Hands-on Verification Task:** Move a referenced asset, leave redirector, clean cook, then fix references and compare.
- **Sources:** [SRC-ASSET-004], [SRC-ASSET-007], [SRC-ASSET-011], [SRC-BUILD-017]
- **Version Notes:** Editor workflows and validation differ by branch.

### Question: Why can DDC hide production build problems?

- **Category:** Assets / Derived Data
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Warm local/cache state can mask missing generation steps, bad shaders/assets or slow first-run costs.
- **Strong 3-Year-Engineer Answer:** Derived Data Cache improves iteration, but a developer with warm DDC is not representative of clean machines, CI or players. I test cold/warm cache behaviour, shared DDC configuration, cook from clean agent and first-run performance. DDC misses can appear as build time, package time or runtime hitching depending on pipeline.
- **Common Weak Answer:** "DDC only affects editor speed."
- **Follow-up Questions:** shared DDC? clean agent? PSO/shaders? cook time?
- **Hands-on Verification Task:** Run a clean-agent cook with empty DDC and compare time/failures with warm cache.
- **Sources:** [SRC-ASSET-008], [SRC-BUILD-017], [SRC-PERF-003]
- **Version Notes:** DDC backend/configuration is studio-specific.

### Question: What is the first thing to check when an editor plugin fails in packaged builds?

- **Category:** Build / Plugins
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Whether runtime modules, dependencies, staging rules and platform allow the plugin outside editor.
- **Strong 3-Year-Engineer Answer:** Many plugins have editor-only modules or dependencies. I inspect `.uplugin`, module type/loading phase, target rules, runtime third-party libraries, platform allow/deny lists and staging. Then I reproduce in a packaged target, not PIE. The fix is usually module separation or staging/dependency declaration, not copying files manually.
- **Common Weak Answer:** "It works in editor, so packaging is broken."
- **Follow-up Questions:** module type? staged DLL? platform allow list? target.cs?
- **Hands-on Verification Task:** Create an editor-only dependency in a runtime module and catch it during package.
- **Sources:** [SRC-BUILD-001], [SRC-BUILD-002], [SRC-BUILD-003], [SRC-BUILD-005], [SRC-BUILD-017]
- **Version Notes:** Build.cs/Target.cs fields and staging behaviour are branch-sensitive.

### Question: What makes BuildGraph useful beyond running a build command?

- **Category:** Build / Automation
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It encodes repeatable build/cook/package/test/artifact graphs with declared dependencies and machine-usable steps.
- **Strong 3-Year-Engineer Answer:** BuildGraph is useful when release work needs reproducible stages, artefacts, parallelism, labels and CI integration. I use it to make clean-agent builds explicit: restore dependencies, build targets, cook/package, run automation, collect symbols/logs/crashes and publish artefacts. The graph should expose first-cause failure and avoid local-machine assumptions.
- **Common Weak Answer:** "It is just a script format."
- **Follow-up Questions:** artifacts? clean agent? RunUAT? symbols? test gate?
- **Hands-on Verification Task:** Build a graph that packages and runs a smoke test, then breaks a staged dependency.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016], [SRC-BUILD-017], [SRC-BUILD-018]
- **Version Notes:** BuildGraph nodes/options are engine-version sensitive.

### Question: What should an AI Perception memory model store?

- **Category:** AI / Perception
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Stimulus source, sense, location, age, strength, success/loss, confidence and gameplay interpretation.
- **Strong 3-Year-Engineer Answer:** AI Perception events are not the whole memory model. I convert stimuli into blackboard/world-state facts with timestamps, expiry, confidence and last-known positions. I distinguish currently perceived target from last known target, and clear or degrade facts intentionally. Debugging uses AI Debugger, Visual Logger and event logs.
- **Common Weak Answer:** "Use OnPerceptionUpdated to set TargetActor."
- **Follow-up Questions:** stimulus age? last known location? lost sight? hearing? confidence?
- **Hands-on Verification Task:** Implement sight/hearing memory where lost sight causes investigation before forgetting.
- **Sources:** [SRC-AI-004], [SRC-AI-008], [SRC-AI-009], [SRC-AI-018]
- **Version Notes:** Sense config APIs are branch-sensitive.

### Question: Why can Behaviour Tree Observer Aborts be better than polling services?

- **Category:** AI / Behaviour Trees
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** They let relevant condition changes interrupt branches without high-frequency polling everywhere.
- **Strong 3-Year-Engineer Answer:** Services are useful, but a tree full of frequent polling can waste CPU and hide state ownership. Observer aborts let blackboard changes trigger branch reevaluation and cancellation. I design keys with clear writers, use abort modes intentionally, and debug with AI Debugger/Visual Logger when a branch sticks or thrashes.
- **Common Weak Answer:** "Put a service on every branch."
- **Follow-up Questions:** abort self/lower priority? key writers? latent task abort?
- **Hands-on Verification Task:** Replace a 0.1s target-check service with a blackboard observer abort and compare behaviour.
- **Sources:** [SRC-AI-002], [SRC-AI-008], [SRC-AI-009]
- **Version Notes:** BT editor/runtime details are branch-sensitive.

### Question: What is the performance trap with EQS?

- **Category:** AI / EQS
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Generating and testing many candidate points with expensive traces/contexts can scale badly across many AI.
- **Strong 3-Year-Engineer Answer:** EQS is powerful for tactical queries, but cost depends on generator size, test count, trace/nav tests, context complexity, frequency and number of agents. I profile query time, limit frequency, cache or share results where valid, use cheaper filters first and fallback behaviours for failed/expensive queries.
- **Common Weak Answer:** "EQS is just data, so it is cheap."
- **Follow-up Questions:** candidate count? traces? frequency? shared results?
- **Hands-on Verification Task:** Run an attack-position EQS for 1/20/100 AI and reduce cost without changing outcome.
- **Sources:** [SRC-AI-005], [SRC-AI-008], [SRC-PERF-003]
- **Version Notes:** EQS tooling and tracing vary by branch.

### Question: Why can Mass archetype count explode?

- **Category:** MassEntity / ECS
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Too many fragment/tag/shared-fragment combinations split entities into many archetypes and reduce chunk efficiency.
- **Strong 3-Year-Engineer Answer:** Mass performance depends on data layout and chunk iteration. If designers create high-cardinality shared fragments, toggle tags frequently or over-specialise composition, entities scatter across archetypes and structural changes increase. I inspect archetype/chunk counts, query matches and migrations, then consolidate stable composition and move high-cardinality data out of shared-fragment splits where appropriate.
- **Common Weak Answer:** "Mass is always faster than Actors."
- **Follow-up Questions:** shared fragment? structural change? chunk occupancy? tag churn?
- **Hands-on Verification Task:** Add per-entity shared variants until archetype count explodes, then redesign.
- **Sources:** [SRC-MASS-001], [SRC-MASS-003], [SRC-MASS-004], [SRC-MASS-007]
- **Version Notes:** Mass APIs and debugger views are branch/plugin-sensitive.

### Question: Why are structural changes during Mass iteration risky?

- **Category:** MassEntity / ECS
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Composition changes can invalidate iteration assumptions and require deferred commands/migrations.
- **Strong 3-Year-Engineer Answer:** ECS iteration expects stable chunk/archetype data while a query runs. Adding/removing fragments/tags directly during iteration can be invalid or expensive; Mass uses deferred command patterns for structural changes. I design processors so hot per-frame state changes are data values when possible, not constant composition churn.
- **Common Weak Answer:** "Just add/remove tags whenever."
- **Follow-up Questions:** deferred commands? migration cost? query invalidation? high-frequency state?
- **Hands-on Verification Task:** Toggle a tag every frame for 10k entities and measure migrations versus a state enum fragment.
- **Sources:** [SRC-MASS-001], [SRC-MASS-003], [SRC-MASS-004], [SRC-MASS-007]
- **Version Notes:** Exact Mass command APIs are branch-sensitive.

### Question: Why does GAS ASC placement matter?

- **Category:** GAS / Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** ASC lifetime and owner/avatar choice determine ability persistence, respawn behaviour, replication audience and init order.
- **Strong 3-Year-Engineer Answer:** Putting ASC on PlayerState can preserve long-lived player abilities across Pawn respawn; putting it on Pawn can be simpler for pawn-local enemies. The important part is consistent owner/avatar assignment, attribute initialisation, ability grants/removal and replication mode. I test death/respawn/possession/late join rather than choosing by habit.
- **Common Weak Answer:** "Always put ASC on PlayerState."
- **Follow-up Questions:** owner/avatar? AI? respawn? Mixed replication mode?
- **Hands-on Verification Task:** Compare PlayerState ASC and Pawn ASC through respawn and late join.
- **Sources:** [SRC-GAS-001], [SRC-GAS-002], [SRC-GAS-010], [SRC-GAS-012]
- **Version Notes:** GAS setup and replication details are branch-sensitive.

### Question: What does `FScopedPredictionWindow` not make safe?

- **Category:** GAS / Prediction
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** It does not make arbitrary gameplay side effects reversible or authoritative.
- **Strong 3-Year-Engineer Answer:** Prediction windows help correlate predicted GAS operations, but server authority still validates activation, cost, cooldown and target data. Irreversible side effects such as rewards, durable inventory, authoritative damage or analytics need authority commit and rollback/reconcile design. I distinguish GAS prediction from full rollback.
- **Common Weak Answer:** "Wrap it in a prediction window."
- **Follow-up Questions:** target data? cue dedupe? irreversible side effect? server reject?
- **Hands-on Verification Task:** Predict a local cue under lag and reject the ability on the server without duplicating side effects.
- **Sources:** [SRC-GAS-011], [SRC-GAS-013], [SRC-GAS-003], [SRC-NET-020]
- **Version Notes:** Supported prediction paths are exact-version sensitive.

### Question: Why does Enhanced Input mapping context priority matter?

- **Category:** Input / Enhanced Input
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Multiple contexts can bind overlapping actions; priority and local-player ownership determine which mappings win.
- **Strong 3-Year-Engineer Answer:** Enhanced Input is local-player scoped. Gameplay, vehicle, menu and modal contexts may overlap, so I add/remove contexts explicitly with priorities and test device changes/split-screen. Bugs often come from stale contexts after possession/UI close or adding contexts to the wrong local player.
- **Common Weak Answer:** "Bind all actions once."
- **Follow-up Questions:** local player subsystem? modal UI? split-screen? stale context?
- **Hands-on Verification Task:** Add gameplay and menu contexts with overlapping confirm/cancel and prove correct priority through UI open/close.
- **Sources:** [SRC-INPUT-001], [SRC-INPUT-003], [SRC-INPUT-004], [SRC-UI-010]
- **Version Notes:** Enhanced Input APIs are plugin/version sensitive.

### Question: Why can animation state machines miss important gameplay cancellation?

- **Category:** Animation / Gameplay Boundary
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Animation transitions can be interrupted, blended or skipped; gameplay authority needs its own cancellation/timeout path.
- **Strong 3-Year-Engineer Answer:** State machines are presentation control. Gameplay windows such as melee damage or interact locks should listen to authoritative state, montage/notify evidence and cancellation/timeout rules, not only "animation reached exit." I log montage state, notify begin/end, authority window and cancellation reason separately.
- **Common Weak Answer:** "When the animation exits, gameplay ends."
- **Follow-up Questions:** interrupted montage? blend out? dedicated server? notify state?
- **Hands-on Verification Task:** Interrupt a melee montage and prove the damage window closes exactly once.
- **Sources:** [SRC-ANIM-005], [SRC-ANIM-008], [SRC-ANIM-010], [SRC-NET-001]
- **Version Notes:** Animation event timing can vary by graph and branch.

### Question: Why are linked animation layers useful but risky?

- **Category:** Animation / Architecture
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** They modularise animation logic, but missing layers, stale interfaces or mismatched skeleton assumptions can break pose output.
- **Strong 3-Year-Engineer Answer:** Linked layers help equipment/locomotion modularity and reduce monolithic AnimBPs. I still need clear layer interfaces, fallback poses, skeleton compatibility, sync/slot policy and thread-safe data snapshots. Debugging checks active linked instance, layer binding, slot routing and pose watches.
- **Common Weak Answer:** "Use linked layers for all animation reuse."
- **Follow-up Questions:** layer interface? fallback? sync groups? equipment swap?
- **Hands-on Verification Task:** Hot-swap an equipment linked layer and prove missing layer uses a safe fallback.
- **Sources:** [SRC-ANIM-003], [SRC-ANIM-012], [SRC-ANIM-011], [SRC-ANIM-016]
- **Version Notes:** Linked layer behaviour and debugging tools are branch-sensitive.

### Question: What does physics substepping solve and not solve?

- **Category:** Physics / Simulation
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It can improve simulation stability under variable frame time, but it does not fix bad ownership, constraints, collision setup or networking.
- **Strong 3-Year-Engineer Answer:** Substepping lets physics integrate in smaller steps when frame time is large, which can help stability. It will not fix contradictory transform writers, wrong collision modes, unrealistic mass ratios, unstable constraints or unvalidated client physics. I compare with/without substep using Chaos Visual Debugger, contact/constraint evidence and network correction logs.
- **Common Weak Answer:** "Turn on substepping for stable physics."
- **Follow-up Questions:** constraints? mass ratio? transform owner? network physics?
- **Hands-on Verification Task:** Create an unstable hinged door and compare substep versus corrected constraint frames/mass.
- **Sources:** [SRC-PHYS-004], [SRC-PHYS-005], [SRC-PHYS-006], [SRC-PHYS-009]
- **Version Notes:** Chaos settings and networking modes are branch-sensitive.

### Question: Why do constraint frames matter?

- **Category:** Physics / Constraints
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** The constraint solves relative motion around defined frames; wrong frames make limits and motors behave incorrectly.
- **Strong 3-Year-Engineer Answer:** A constraint is not just "connect these bodies." Its local frames define axes, pivots and limits. If frames are offset or rotated incorrectly, doors twist, joints explode or motors fight. I debug by visualising frames, checking mass/inertia, collision between constrained bodies and solver settings.
- **Common Weak Answer:** "Increase stiffness."
- **Follow-up Questions:** local frames? mass ratio? collision? solver iteration?
- **Hands-on Verification Task:** Build a door with a wrong hinge frame, capture CVD evidence and fix the frame.
- **Sources:** [SRC-PHYS-006], [SRC-PHYS-008], [SRC-PHYS-009]
- **Version Notes:** Constraint APIs/settings vary by branch.

### Question: Why should ListView entries clean up in row release?

- **Category:** UI / Virtualisation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Virtualised rows are recycled; stale bindings, async icon loads or selection state can leak into a different item.
- **Strong 3-Year-Engineer Answer:** A row widget is presentation, not item identity. When released/reused, it must unbind delegates, cancel or generation-check async icon loads and reset visual state. I key all updates by stable item ID and request generation so late async completion cannot update the wrong row.
- **Common Weak Answer:** "Each item has its own widget."
- **Follow-up Questions:** async icon? selection? row reuse? stable ID?
- **Hands-on Verification Task:** Scroll a 1000-item inventory with delayed icon loads and reject stale row updates.
- **Sources:** [SRC-UI-013], [SRC-UI-014], [SRC-ASSET-002], [SRC-UI-005]
- **Version Notes:** UMG/ListView APIs differ across branches.

### Question: Why can CommonUI and Enhanced Input conflict in modal screens?

- **Category:** UI / Input Routing
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Both manage input context/routing; stale contexts or wrong local-player focus can route actions to gameplay behind the modal.
- **Strong 3-Year-Engineer Answer:** CommonUI manages activatable widget stacks and action routing, while Enhanced Input owns local-player mapping contexts. For modals I set focus and input mode for the correct local player, push/remove UI contexts with priority and test controller/mouse/touch/split-screen. I log focused widget, active contexts and local player when debugging.
- **Common Weak Answer:** "Set input mode UI only."
- **Follow-up Questions:** local player? mapping context priority? focus restore? split-screen?
- **Hands-on Verification Task:** Open a pause modal in split-screen and prove only the correct player navigates it.
- **Sources:** [SRC-UI-009], [SRC-UI-010], [SRC-UI-011], [SRC-INPUT-004]
- **Version Notes:** CommonUI and Enhanced Input are plugin/version sensitive.

### Question: When is Quartz useful for game audio?

- **Category:** Audio / Timing
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** For sample-accurate or musically scheduled audio events that need better timing than ordinary game-thread timers.
- **Strong 3-Year-Engineer Answer:** Quartz schedules audio against a musical/time clock in the audio system, useful for rhythm, music transitions and tightly timed loops. It is not a replacement for gameplay authority. I test under game-thread hitches and compare scheduled audio timing with ordinary timers while keeping server gameplay decisions separate.
- **Common Weak Answer:** "Use Quartz for all timers."
- **Follow-up Questions:** game-thread hitch? audio thread? multiplayer? music transitions?
- **Hands-on Verification Task:** Schedule a beat-aligned loop under a forced game-thread hitch and compare with timer-based playback.
- **Sources:** [SRC-AUDIO-008], [SRC-AUDIO-006], [SRC-AUDIO-010]
- **Version Notes:** Audio timing behaviour depends on platform/audio backend.

### Question: Why can MetaSounds still be expensive?

- **Category:** Audio / Performance
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Procedural audio graphs consume CPU/voices/resources and can scale badly if spawned or parameterised without budget.
- **Strong 3-Year-Engineer Answer:** MetaSounds are powerful procedural sources, not free. I profile voice count, graph complexity, parameter update rate, concurrency, submix/DSP cost and stream/cache behaviour. For repeated impacts or loops I pool/reuse where appropriate and degrade by platform/importance.
- **Common Weak Answer:** "MetaSounds are newer, so they are faster."
- **Follow-up Questions:** voice count? parameter rate? concurrency? submix cost?
- **Hands-on Verification Task:** Spawn 100 procedural impacts and compare CPU/voice/concurrency metrics with a simpler Sound Cue.
- **Sources:** [SRC-AUDIO-007], [SRC-AUDIO-004], [SRC-AUDIO-006], [SRC-AUDIO-010]
- **Version Notes:** MetaSound features and performance vary by branch/platform.

### Question: Why can Niagara GPU collisions be misleading for gameplay?

- **Category:** VFX / Gameplay Boundary
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Particle collision can be approximate, presentation-oriented and GPU-delayed; gameplay collision should use authoritative gameplay/physics queries.
- **Strong 3-Year-Engineer Answer:** Niagara collision is for effects. GPU simulation/readback and scalability can make it unsuitable for authoritative damage or interaction. Gameplay should own collision/traces on the server or authoritative system, then feed Niagara presentation data. I debug visual effect collision separately from gameplay hit validation.
- **Common Weak Answer:** "Use particle collision for damage."
- **Follow-up Questions:** GPU readback? server? scalability? bounds?
- **Hands-on Verification Task:** Compare a gameplay trace hit with a Niagara collision event and explain authority boundary.
- **Sources:** [SRC-FX-003], [SRC-FX-005], [SRC-FX-011], [SRC-PHYS-002]
- **Version Notes:** Niagara collision modules differ by simulation target/branch.

### Question: How should runtime PCG interact with navigation and collision?

- **Category:** PCG / Runtime Systems
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Generated output needs explicit policy for collision, navmesh updates, authority, cleanup, save and replication.
- **Strong 3-Year-Engineer Answer:** Runtime PCG is not only visual placement. If it spawns collidable or interactable content, I define deterministic seeds, ownership, nav/collision update timing, cleanup on regeneration, save/replication policy and World Partition interaction. For many games, baking or editor generation is safer for gameplay-critical geometry.
- **Common Weak Answer:** "Generate it and nav will update."
- **Follow-up Questions:** runtime nav cost? replication? cleanup? save identity?
- **Hands-on Verification Task:** Generate a blocking obstacle at runtime and prove nav, save, replication and cleanup policy.
- **Sources:** [SRC-PCG-001], [SRC-PCG-004], [SRC-PCG-005], [SRC-AI-006]
- **Version Notes:** Runtime PCG and nav update support are branch/project sensitive.

### Question: Why is World Partition streaming readiness not gameplay readiness?

- **Category:** World Partition / Gameplay
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Loaded cells/actors may not have completed gameplay initialisation, replication, async assets or dependency binding.
- **Strong 3-Year-Engineer Answer:** World Partition controls spatial content loading, but gameplay systems still need readiness handshakes. A streamed Actor can exist before its dependencies, replicated state, async assets or AI/navigation are ready. I separate streaming state from quest/system state and use stable IDs plus readiness events for gameplay transitions.
- **Common Weak Answer:** "If the cell is loaded, gameplay can run."
- **Follow-up Questions:** BeginPlay? async assets? Data Layers? nav? replication?
- **Hands-on Verification Task:** Stream in an objective cell and delay activation until dependencies report ready.
- **Sources:** [SRC-ASSET-010], [SRC-WORLD-001], [SRC-ASSET-009], [SRC-EPIC-015]
- **Version Notes:** World Partition APIs and runtime hash behaviour are branch-sensitive.

### Question: Why should Lua or C# scripting expose semantic APIs instead of raw UObject ownership?

- **Category:** Scripting / Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Semantic APIs preserve lifetime, authority, thread and version boundaries across language runtimes.
- **Strong 3-Year-Engineer Answer:** Script runtimes have separate GC, reload, wrapper and threading concerns. Exposing raw mutable UObject ownership invites stale handles, wrong-thread access, authority bypass and reload bugs. I expose stable IDs, read-only snapshots, validated commands, events and explicit cancellation/lifetime semantics.
- **Common Weak Answer:** "Expose all UObject methods to scripts."
- **Follow-up Questions:** wrapper invalidation? reload? authority? GC? thread?
- **Hands-on Verification Task:** Replace raw inventory mutation from Lua/C# with a validated C++ command facade.
- **Sources:** [SRC-SCRIPT-001], [SRC-SCRIPT-002], [SRC-SCRIPT-005], [SRC-EPIC-009]
- **Version Notes:** Plugin runtime behaviour is exact-version/platform sensitive.

## Targeted Interview Expansion Set to 600+

### Question: When should you prefer value semantics over UObject references?

- **Category:** C++ / UE Architecture
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Prefer values for small deterministic data without identity, reflection lifetime, polymorphic UObject behaviour or shared ownership needs.
- **Strong 3-Year-Engineer Answer:** Value types are simpler to copy, compare, serialise and test. I use UObject references when I need engine identity, reflection, GC, editor asset identity or polymorphism. For hot simulation, values often reduce pointer chasing and lifetime bugs, but I still watch copy cost and alignment.
- **Common Weak Answer:** "Everything in Unreal should be a UObject."
- **Follow-up Questions:** identity? GC? cache locality? replication?
- **Hands-on Verification Task:** Convert a per-frame combat stat object from UObject to struct and compare lifetime/profiling.
- **Sources:** [SRC-CPP-001], [SRC-CPP-019], [SRC-EPIC-003], [SRC-EPIC-009]
- **Version Notes:** UObject/reflection requirements are project-specific.

### Question: What does move semantics not guarantee?

- **Category:** Standard C++ / Move Semantics
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Moving does not guarantee no work, no allocation, or a useful value remaining in the moved-from object.
- **Strong 3-Year-Engineer Answer:** `std::move` is a cast to an rvalue; the type's move operation decides what happens. Moved-from objects must be valid but not necessarily meaningful. In Unreal code I still check container invalidation, UObject ownership rules and whether moving large values actually reduces cost.
- **Common Weak Answer:** "`std::move` always makes it fast."
- **Follow-up Questions:** moved-from state? move_if_noexcept? container reallocation?
- **Hands-on Verification Task:** Instrument copy/move constructors for a value used inside `TArray`/`std::vector` growth.
- **Sources:** [SRC-CPP-004], [SRC-CPP-005], [SRC-CPP-009], [SRC-CPP-010]
- **Version Notes:** Container behaviour depends on type traits and allocator.

### Question: Why can `std::function` be costly in hot code?

- **Category:** C++ / Callables
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It type-erases calls and may allocate/copy captures, adding overhead versus templates, function refs or direct delegates.
- **Strong 3-Year-Engineer Answer:** `std::function` is useful for storage and generic callbacks, but hot per-entity loops often benefit from templates, function pointers, `TFunctionRef`-style non-owning call views or explicit strategy objects. I measure call count, capture size, allocation and lifetime before using it broadly.
- **Common Weak Answer:** "`std::function` is just a callback."
- **Follow-up Questions:** capture lifetime? allocation? type erasure? delegate alternative?
- **Hands-on Verification Task:** Benchmark direct call, template callable, function pointer, `std::function` and Unreal delegate in a tight loop.
- **Sources:** [SRC-CPP-018], [SRC-ARCH-013], [SRC-PERF-003]
- **Version Notes:** Unreal callable helpers are branch-sensitive.

### Question: What does release/acquire ordering solve?

- **Category:** C++ / Atomics
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It publishes writes before a release store to a thread that observes them through an acquire load.
- **Strong 3-Year-Engineer Answer:** Release/acquire is for visibility and ordering, not mutual exclusion. It can safely publish immutable data or state transitions when designed carefully, but wrong lifetime, ABA, multiple writers or non-atomic side state still break it. I use mutexes when invariants span multiple fields.
- **Common Weak Answer:** "Atomics make code thread-safe."
- **Follow-up Questions:** data race? happens-before? multiple fields? lifetime?
- **Hands-on Verification Task:** Publish a snapshot pointer with release/acquire and explain why the pointed data must outlive readers.
- **Sources:** [SRC-CPP-020], [SRC-CPP-021], [SRC-CPP-022]
- **Version Notes:** Platform memory model is C++ standard plus compiler/runtime behaviour.

### Question: Why does ODR matter in Unreal modules?

- **Category:** C++ / Build
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Duplicate or inconsistent definitions across modules can cause link failures or undefined behaviour.
- **Strong 3-Year-Engineer Answer:** Unreal modules make boundaries explicit, but normal C++ ODR still applies. Template definitions, inline functions, static data, generated headers and exported symbols must be handled consistently. I debug with link errors, module dependencies, export macros and minimal two-module repros.
- **Common Weak Answer:** "UBT handles all C++ linkage."
- **Follow-up Questions:** export macro? inline? template visibility? static init?
- **Hands-on Verification Task:** Create a helper defined in two modules and fix the duplicate symbol/export problem.
- **Sources:** [SRC-BUILD-001], [SRC-BUILD-002], [SRC-BUILD-003], [SRC-CPP-001]
- **Version Notes:** Linker diagnostics vary by toolchain/platform.

### Question: How do you choose between `TArray`, `TSet` and `TMap`?

- **Category:** UE Containers / Algorithms
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Choose by access pattern, ordering, mutation frequency, memory/cache cost and identity requirements.
- **Strong 3-Year-Engineer Answer:** `TArray` is compact and cache-friendly for iteration and small linear search. Sets/maps help key lookup but cost hashing, memory and iteration locality. I profile realistic counts, know invalidation rules and avoid picking hash containers for tiny hot sets.
- **Common Weak Answer:** "Use map for lookup, array for lists."
- **Follow-up Questions:** small N? order? stable indices? memory? hashing?
- **Hands-on Verification Task:** Compare inventory lookup with 16, 256 and 10k items using array, set and map.
- **Sources:** [SRC-ALG-002], [SRC-ALG-003], [SRC-CPP-009]
- **Version Notes:** Unreal container internals differ from STL; verify target behaviour.

### Question: Why can `FText` be the wrong type for gameplay identity?

- **Category:** UE Types / Localisation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** `FText` is for localised display text, not stable IDs or logic keys.
- **Strong 3-Year-Engineer Answer:** Display text can change by culture, source string, namespace/key or designer editing. Gameplay identity should use stable names/IDs/tags/assets, while UI presents `FText`. I avoid comparing localised text for rules, saves, analytics or networking.
- **Common Weak Answer:** "Compare the displayed name."
- **Follow-up Questions:** `FName`? gameplay tags? save migration? localisation?
- **Hands-on Verification Task:** Localise an item name and prove gameplay ID remains stable.
- **Sources:** [SRC-UI-016], [SRC-EPIC-001], [SRC-ARCH-009]
- **Version Notes:** Localisation pipeline is project-specific.

### Question: What makes Data Assets useful for gameplay tuning?

- **Category:** Data-Driven Gameplay
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** They separate designer-authored definitions from runtime instance state.
- **Strong 3-Year-Engineer Answer:** Data Assets are good for stable definitions: item archetypes, ability configs, enemy tunings and effect tables. Runtime state such as durability, stack count or cooldown belongs in instances/save/replication. I validate assets, version IDs and avoid mutating shared definitions at runtime.
- **Common Weak Answer:** "Store item instances in Data Assets."
- **Follow-up Questions:** definition vs instance? validation? save ID? soft references?
- **Hands-on Verification Task:** Build item definition assets plus runtime item instances with stable IDs.
- **Sources:** [SRC-ARCH-009], [SRC-ARCH-010], [SRC-ARCH-012], [SRC-ASSET-003]
- **Version Notes:** Asset Manager integration is project-specific.

### Question: Why can delegates leak or crash after object destruction?

- **Category:** UE C++ / Delegates
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Bound callbacks can outlive the object or capture stale state unless unbound or weak/lifetime-checked.
- **Strong 3-Year-Engineer Answer:** Delegates connect lifetimes. For UObject targets I use the appropriate binding helper and still unbind during teardown when ownership is not obvious. For lambdas I avoid raw `this` captures across async/timer boundaries unless a weak pointer/generation check protects use.
- **Common Weak Answer:** "Delegates clean themselves up."
- **Follow-up Questions:** lambda capture? timer? multicast? async completion?
- **Hands-on Verification Task:** Bind a lambda to a timer, destroy the Actor, then fix the stale capture.
- **Sources:** [SRC-ARCH-013], [SRC-EPIC-009], [SRC-CPP-002]
- **Version Notes:** Delegate helpers vary by engine branch.

### Question: Why can RPC reliability harm multiplayer responsiveness?

- **Category:** Networking / RPCs
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Reliable RPCs queue and must be delivered in order, so overuse can block newer important traffic.
- **Strong 3-Year-Engineer Answer:** Reliability is for important infrequent events, not replaceable high-rate state. Spamming reliable RPCs for input, cosmetic updates or tick state can create backlog under loss. I use properties for durable state, unreliable for replaceable events, and reliable only when loss is unacceptable and rate is bounded.
- **Common Weak Answer:** "Reliable is safer."
- **Follow-up Questions:** queue? packet loss? state vs event? input commands?
- **Hands-on Verification Task:** Send reliable cosmetic RPCs under packet loss and observe backlog versus unreliable/state replication.
- **Sources:** [SRC-NET-006], [SRC-NET-005], [SRC-NET-010]
- **Version Notes:** Exact channel behaviour is engine-source sensitive.

### Question: What does OnRep ordering not guarantee?

- **Category:** Networking / Replication
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It does not guarantee separate replicated properties arrive or execute in the semantic order your gameplay assumes.
- **Strong 3-Year-Engineer Answer:** Replication ordering has constraints, but protocol design should not depend on multiple independent properties forming an ordered transaction unless grouped or versioned. I replicate coherent structs/revisions or derive state idempotently. Debugging logs property versions and OnRep execution rather than assuming order.
- **Common Weak Answer:** "Replicate A then B in code order."
- **Follow-up Questions:** coherent struct? revision? RPC ordering? late join?
- **Hands-on Verification Task:** Split match phase/time into two props, reproduce inconsistent UI, then merge into one replicated struct.
- **Sources:** [SRC-NET-004], [SRC-NET-005], [SRC-NET-001]
- **Version Notes:** Exact ordering details are target-version sensitive.

### Question: Why can dormancy hide state changes?

- **Category:** Networking / Dormancy
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Dormant Actors may not send updates until properly woken/flushed before mutation.
- **Strong 3-Year-Engineer Answer:** Dormancy is a bandwidth optimisation for stable Actors. If code mutates replicated state while the Actor is dormant or wakes after mutation incorrectly, clients can miss or delay updates. I wake/flush before changes according to target rules, then validate with network debug logs and packet loss tests.
- **Common Weak Answer:** "Dormant Actors still replicate when changed."
- **Follow-up Questions:** wake before change? initial dormancy? RepGraph? debug CVar?
- **Hands-on Verification Task:** Mutate a dormant objective before and after waking and compare client state.
- **Sources:** [SRC-NET-008], [SRC-NET-016], [SRC-NET-004]
- **Version Notes:** Dormancy behaviour differs with RepGraph/Iris/version.

### Question: What makes Fast Array identity important?

- **Category:** Networking / Fast Array
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Delta replication needs stable item identity to distinguish add, change, remove and reorder.
- **Strong 3-Year-Engineer Answer:** Fast Array is not just "replicate arrays faster." Items need stable IDs and centralised authoritative mutation. Reusing IDs or mutating without dirty marks can make clients miss changes or apply them to the wrong UI row. I test add/remove/reorder/reuse/late join/loss.
- **Common Weak Answer:** "Fast Array handles all array changes automatically."
- **Follow-up Questions:** dirty mark? stable ID? reorder? UI stale selection?
- **Hands-on Verification Task:** Reuse a Fast Array item ID and catch the stale client behaviour.
- **Sources:** [SRC-NET-015], [SRC-NET-016], [SRC-ALG-002]
- **Version Notes:** Exact Fast Array hooks are source-sensitive.

### Question: What problem does Replication Graph solve?

- **Category:** Networking / Scalability
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It scales per-connection relevance/list building for many Actors using graph nodes and persistent policy data.
- **Strong 3-Year-Engineer Answer:** RepGraph is for replication scalability when baseline relevancy/list building is measured as a bottleneck. It lets the project organise Actors by spatial and policy nodes. I would not adopt it by fashion; I first profile relevant Actor counts, bandwidth and server net time, then prototype buckets and validate edge cases.
- **Common Weak Answer:** "RepGraph makes networking faster."
- **Follow-up Questions:** spatial node? always relevant? team relevance? splitscreen caveat?
- **Hands-on Verification Task:** Prototype one spatial bucket node and compare relevant Actor iteration before/after.
- **Sources:** [SRC-NET-013], [SRC-NET-010], [SRC-NET-007]
- **Version Notes:** RepGraph support and caveats are project/platform sensitive.

### Question: Why should Iris be discussed carefully in UE5 interviews?

- **Category:** Networking / Iris
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It is a newer opt-in/experimental replication system in the docs, not automatically every UE5 project's baseline.
- **Strong 3-Year-Engineer Answer:** I describe Iris as a specialist replication framework with descriptors/fragments/filtering and different requirements, then ask whether the project enables it. I avoid copying Iris-specific subobject assumptions into Generic replication and verify target configuration/source before implementation claims.
- **Common Weak Answer:** "UE5 uses Iris."
- **Follow-up Questions:** config? registered subobjects? fragments? migration?
- **Hands-on Verification Task:** Inspect a target project config/source and document whether Iris is actually enabled.
- **Sources:** [SRC-NET-014], [SRC-NET-012], [SRC-NET-001]
- **Version Notes:** Iris status and APIs are version/project sensitive.

### Question: How do you debug a disappearing replicated subobject?

- **Category:** Networking / Subobjects
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Check owner replication, subobject registration/lifetime, mapping timing, GC references, conditions and client OnRep validity.
- **Strong 3-Year-Engineer Answer:** Dynamic subobjects need stable ownership and lifetime. I log server creation, registration, pointer replication, client mapping, conditions, deletion and GC. Clients may see null before mapping, and stale registered-list entries can crash if not removed before destruction.
- **Common Weak Answer:** "Replicate the pointer."
- **Follow-up Questions:** registered list? GC? mapping? condition? Iris?
- **Hands-on Verification Task:** Delete a registered dynamic subobject without removal in a safe branch and capture the failure/validation.
- **Sources:** [SRC-NET-012], [SRC-NET-011], [SRC-NET-016], [SRC-EPIC-009]
- **Version Notes:** Subobject APIs are branch and Iris/Generic sensitive.

### Question: Why should gameplay tags not replace all state?

- **Category:** GAS / Tags
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Tags are excellent semantic labels/queries, but numeric values, timers, ownership and durable state still need proper models.
- **Strong 3-Year-Engineer Answer:** Tags answer "has this condition/state category?" They are not a replacement for attributes, cooldown specs, save identity or complex state machines. I use tags for gating, immunity, categories and event semantics, then keep quantitative state in attributes/effects/runtime systems.
- **Common Weak Answer:** "Represent every state as a tag."
- **Follow-up Questions:** stacks? duration? numeric magnitude? save? prediction?
- **Hands-on Verification Task:** Replace a numeric armour value encoded in tags with an attribute/effect model.
- **Sources:** [SRC-GAS-006], [SRC-GAS-004], [SRC-GAS-005]
- **Version Notes:** GAS tag replication and prediction are branch-sensitive.

### Question: How do you design an AttributeSet invariant?

- **Category:** GAS / Attributes
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Define canonical bounds and repair points for base/current values, max changes and replication/prediction paths.
- **Strong 3-Year-Engineer Answer:** For Health/MaxHealth, I define clamp rules, what happens when MaxHealth decreases, where damage/heal modifies current value, and which callbacks enforce bounds. I test GameplayEffect execution, direct setter misuse, replication and prediction. Invariants need tests, not just comments.
- **Common Weak Answer:** "Clamp in the UI."
- **Follow-up Questions:** max decrease? PreAttributeChange? PostGameplayEffectExecute? prediction?
- **Hands-on Verification Task:** Apply a MaxHealth debuff below current Health and verify server/client invariant.
- **Sources:** [SRC-GAS-004], [SRC-GAS-005], [SRC-GAS-009]
- **Version Notes:** Callback semantics need target-source verification.

### Question: Why should Ability Tasks clean up symmetrically?

- **Category:** GAS / Ability Tasks
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Tasks bind delegates/timers/targets and may end by completion, cancel, ability end or avatar destruction.
- **Strong 3-Year-Engineer Answer:** Ability Tasks create async lifetimes inside abilities. Every bind, timer, target reference and prediction path needs cleanup in all exit paths. I test cancellation, interruption, avatar destroy, network reject and task timeout so stale callbacks cannot fire into ended abilities.
- **Common Weak Answer:** "The task object will be GC'd."
- **Follow-up Questions:** `OnDestroy`? delegate unbind? prediction reject? avatar destroyed?
- **Hands-on Verification Task:** Build a custom wait task, cancel it mid-event and prove no callback fires after ability end.
- **Sources:** [SRC-GAS-007], [SRC-GAS-003], [SRC-EPIC-009]
- **Version Notes:** Task APIs are branch-sensitive.

### Question: Why should input buffering record commands rather than raw key state only?

- **Category:** Input / Gameplay
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Gameplay needs semantic intent with timing, context and validation, not just current hardware state.
- **Strong 3-Year-Engineer Answer:** Raw key state is device-level. A fighting/combat buffer usually records semantic commands with frame/time, context, consumed state and action IDs. That supports replay, rollback, accessibility remap and deterministic tests. Enhanced Input can map devices into actions, but gameplay still owns buffer semantics.
- **Common Weak Answer:** "Check if the key is down."
- **Follow-up Questions:** remapping? action context? rollback? consume window?
- **Hands-on Verification Task:** Implement a 6-frame dodge buffer across keyboard and gamepad actions.
- **Sources:** [SRC-INPUT-001], [SRC-INPUT-002], [SRC-INPUT-005], [SRC-NET-020]
- **Version Notes:** Enhanced Input action event semantics are plugin-version sensitive.

### Question: How do you debug animation-thread unsafe data access?

- **Category:** Animation / Threading
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Snapshot game-thread state into anim-safe variables and avoid arbitrary UObject/gameplay reads during threaded evaluation.
- **Strong 3-Year-Engineer Answer:** Animation can evaluate on worker threads. Anim graphs should consume prepared proxy/snapshot data, not chase mutable gameplay objects. I log update/evaluate phases, use Pose Watch/Animation Insights and move gameplay queries into game-thread update paths with versioned snapshots.
- **Common Weak Answer:** "Read the character from the AnimBP whenever."
- **Follow-up Questions:** update vs evaluate? proxy? stale owner? threaded evaluation?
- **Hands-on Verification Task:** Replace an AnimGraph gameplay object read with a movement/presentation snapshot.
- **Sources:** [SRC-ANIM-003], [SRC-ANIM-004], [SRC-ANIM-016], [SRC-ANIM-019]
- **Version Notes:** Animation threading details are branch-sensitive.

### Question: Why does Motion Warping still need gameplay validation?

- **Category:** Animation / Motion Warping
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It adjusts animation motion toward targets, but gameplay authority must validate target, timing and collision.
- **Strong 3-Year-Engineer Answer:** Motion Warping is presentation/movement assistance for interactions like vaults or attacks. The target point and action validity still require gameplay checks: range, occupancy, nav/collision, authority and cancellation. I keep animation alignment separate from permission to perform the action.
- **Common Weak Answer:** "Warp to whatever target the montage says."
- **Follow-up Questions:** server? collision? target changed? cancel?
- **Hands-on Verification Task:** Warp an interaction to a blocked target and reject it with server validation.
- **Sources:** [SRC-ANIM-008], [SRC-ANIM-009], [SRC-PHYS-002], [SRC-NET-001]
- **Version Notes:** Motion Warping plugin/API support must be checked in target branch.

### Question: What is the difference between query collision and physics simulation?

- **Category:** Physics / Collision
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Queries ask spatial questions such as traces/overlaps; simulation resolves bodies, contacts, impulses and constraints.
- **Strong 3-Year-Engineer Answer:** Gameplay often needs query collision for traces/sweeps and physics simulation for dynamic bodies. A component can block a trace but not simulate, or simulate but not generate the query/event expected. I debug collision responses, object channels, enabled modes, movement path and event flags separately.
- **Common Weak Answer:** "Collision is collision."
- **Follow-up Questions:** trace channel? overlap event? simulate physics? sweep?
- **Hands-on Verification Task:** Build one object that blocks traces but does not simulate, and one that simulates but does not answer a custom trace.
- **Sources:** [SRC-PHYS-001], [SRC-PHYS-002], [SRC-PHYS-004]
- **Version Notes:** Collision profiles are project/version sensitive.

### Question: Why can CCD be the wrong fix for tunnelling?

- **Category:** Physics / Collision
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** CCD has cost/limitations and may not address wrong movement method, collision shape, timestep or authority path.
- **Strong 3-Year-Engineer Answer:** Continuous collision can help fast simulated bodies, but for gameplay projectiles a sweep, hitscan, fixed movement substep or authoritative trace may be simpler and more deterministic. I first identify whether the object moves by physics, kinematic transform, projectile movement or teleport, then choose the collision strategy.
- **Common Weak Answer:** "Turn on CCD for fast things."
- **Follow-up Questions:** sweep? projectile movement? server authority? substep? shape?
- **Hands-on Verification Task:** Compare fast projectile sweep versus simulated body CCD under lag and high velocity.
- **Sources:** [SRC-PHYS-001], [SRC-PHYS-002], [SRC-PHYS-005], [SRC-NET-001]
- **Version Notes:** CCD/Chaos behaviour is branch and platform sensitive.

### Question: Why is world-space UI expensive at scale?

- **Category:** UI / World-Space Widgets
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Many widget components add update, layout, render target/batch and occlusion costs.
- **Strong 3-Year-Engineer Answer:** World-space widgets are convenient, but 100 nameplates can stress widget ticking, layout, draw calls, render targets and visibility logic. I compare screen-space projection, instanced/material labels or relevance-throttled widgets, and capture Slate/Insights/Game/Render/GPU evidence.
- **Common Weak Answer:** "Use WidgetComponent for every label."
- **Follow-up Questions:** screen-space? occlusion? redraw? retainer? relevance?
- **Hands-on Verification Task:** Compare 20/100 actor nameplates using WidgetComponent versus projected Slate/UMG list.
- **Sources:** [SRC-UI-015], [SRC-UI-004], [SRC-UI-005], [SRC-PERF-003]
- **Version Notes:** WidgetComponent rendering behaviour varies by branch/platform.

### Question: How do you make UI text robust for localisation?

- **Category:** UI / Localisation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Use `FText`, flexible layout, pseudo-localisation, truncation/wrapping policy and non-text-dependent affordances.
- **Strong 3-Year-Engineer Answer:** Localised text can expand, change direction and require different fonts. I avoid fixed pixel text boxes, test pseudo-localisation/long strings/RTL where needed, use scalable text and design icons/non-colour cues carefully. Gameplay IDs stay separate from display text.
- **Common Weak Answer:** "Leave enough space for English."
- **Follow-up Questions:** pseudo-localisation? wrapping? RTL? accessibility? stable IDs?
- **Hands-on Verification Task:** Pseudo-localise an inventory and fix clipped buttons without shrinking text below readability.
- **Sources:** [SRC-UI-016], [SRC-UI-006], [SRC-UI-018]
- **Version Notes:** Localisation and accessibility requirements vary by platform/product.

### Question: Why should audio not be the only critical feedback channel?

- **Category:** Audio / Accessibility
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Audio can be muted, unavailable, stolen by concurrency or inaccessible to some players.
- **Strong 3-Year-Engineer Answer:** Critical warnings need redundant feedback: UI, animation, haptics or gameplay affordances. Audio systems can reject/virtualise voices through concurrency, lose routing or be disabled by user/device settings. I design critical feedback with non-audio confirmation and test concurrency rejection.
- **Common Weak Answer:** "Make the sound louder."
- **Follow-up Questions:** concurrency? accessibility? routing? haptics? UI cue?
- **Hands-on Verification Task:** Force warning sound rejection through concurrency and prove another feedback channel remains.
- **Sources:** [SRC-AUDIO-004], [SRC-AUDIO-005], [SRC-AUDIO-010], [SRC-UI-018]
- **Version Notes:** Accessibility requirements are platform/product sensitive.

### Question: Why should long music streams be tested in cooked builds?

- **Category:** Audio / Streaming
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Cooked streaming/cache/chunk behaviour can differ from editor and reveal stalls or missing staged data.
- **Strong 3-Year-Engineer Answer:** Editor playback may hide package/chunk/residency problems. In target cooked builds I test stream cache settings, seek/loop transitions, missing chunks, I/O pressure and memory. I capture audio logs, stream-cache metrics and first-play behaviour on target hardware.
- **Common Weak Answer:** "It plays in editor."
- **Follow-up Questions:** stream cache? chunk? first play? platform I/O?
- **Hands-on Verification Task:** Package a long music track into a separate chunk and test first playback plus missing chunk failure.
- **Sources:** [SRC-AUDIO-009], [SRC-ASSET-006], [SRC-BUILD-017], [SRC-AUDIO-010]
- **Version Notes:** Audio streaming backend is platform/branch sensitive.

### Question: Why can Niagara fixed bounds be both too small and too large?

- **Category:** VFX / Niagara Bounds
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Too small culls visible particles; too large keeps invisible work alive and hurts culling efficiency.
- **Strong 3-Year-Engineer Answer:** Bounds are a performance/correctness contract. I visualise bounds, test camera angles/distance, compare dynamic/fixed bounds and check Effect Type scalability. I avoid solving disappearance by making huge bounds everywhere; instead I author realistic bounds per effect and platform budget.
- **Common Weak Answer:** "Set bounds very large."
- **Follow-up Questions:** culling? scalability? GPU sim? significance? overdraw?
- **Hands-on Verification Task:** Deliberately shrink and over-expand an impact effect's bounds and capture visual/performance impact.
- **Sources:** [SRC-FX-003], [SRC-FX-005], [SRC-FX-004], [SRC-FX-008]
- **Version Notes:** Niagara bounds/scalability behaviour varies by branch.

### Question: What is the role of Niagara Effect Types?

- **Category:** VFX / Scalability
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** They group systems for budget, significance and scalability policy.
- **Strong 3-Year-Engineer Answer:** Effect Types help define how classes of effects scale or cull under platform/performance pressure. I use them to prioritise combat-critical, environment and cosmetic effects differently. Debugging checks if an effect was culled by bounds, significance, distance, budget or instance count.
- **Common Weak Answer:** "Effect Types are just labels."
- **Follow-up Questions:** significance? budget? local player? platform? debug?
- **Hands-on Verification Task:** Create Combat and Ambient Effect Types and show different culling under load.
- **Sources:** [SRC-FX-004], [SRC-FX-005], [SRC-FX-003], [SRC-FX-008]
- **Version Notes:** Scalability settings are project/platform specific.

### Question: Why can A* return bad paths in games even with correct code?

- **Category:** Algorithms / Pathfinding
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** The graph, costs, heuristic, movement constraints or dynamic obstacles may not model the actual game problem.
- **Strong 3-Year-Engineer Answer:** A* correctness is relative to the graph and cost function. If diagonal costs, traversal penalties, agent size, one-way links, dynamic blockers or heuristic assumptions are wrong, code can be correct but paths feel wrong. I debug by visualising graph nodes, edge costs, open/closed sets and comparing expected movement rules.
- **Common Weak Answer:** "A* is optimal."
- **Follow-up Questions:** admissible heuristic? graph model? dynamic obstacles? agent radius?
- **Hands-on Verification Task:** Add terrain penalties and show how heuristic/cost mismatch changes path quality.
- **Sources:** [SRC-ALG-007], [SRC-ALG-008], [SRC-AI-006]
- **Version Notes:** Engine navigation has its own constraints beyond generic A*.

### Question: When is a spatial hash a poor fit?

- **Category:** Algorithms / Spatial Structures
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** When object sizes/density vary widely, queries are irregular, or cell size creates too many collisions/empty checks.
- **Strong 3-Year-Engineer Answer:** Spatial hash is simple and fast for roughly uniform objects and local queries. It degrades with huge objects, clustered density, bad cell size or frequent rebuilds. I compare against grids, quad/octrees, BVHs or engine broadphase based on query shape and update pattern.
- **Common Weak Answer:** "Spatial hash is always O(1)."
- **Follow-up Questions:** cell size? clustered objects? moving objects? broadphase?
- **Hands-on Verification Task:** Compare uniform and clustered object distributions with different cell sizes.
- **Sources:** [SRC-ALG-012], [SRC-ALG-013], [SRC-PHYS-001]
- **Version Notes:** Engine physics/navigation broadphases may already solve the problem.

### Question: Why should matrix/vector convention be stated in gameplay math answers?

- **Category:** Game Math / Transformations
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Row/column, handedness and multiplication order affect transform composition and bug diagnosis.
- **Strong 3-Year-Engineer Answer:** Transform math bugs often come from unstated conventions. Unreal's coordinate system, `FTransform` semantics and local/world-space conversions must be handled deliberately. I name source/destination spaces, multiplication order and whether normals require inverse-transpose handling under non-uniform scale.
- **Common Weak Answer:** "Multiply transforms until it works."
- **Follow-up Questions:** local vs world? normal transform? handedness? non-uniform scale?
- **Hands-on Verification Task:** Convert a hit normal through non-uniform scaled component space correctly.
- **Sources:** [SRC-MATH-001], [SRC-MATH-003], [SRC-MATH-004], [SRC-MATH-009]
- **Version Notes:** APIs are stable but convention misunderstandings are common.

### Question: Why can Euler angles fail in camera or aiming code?

- **Category:** Game Math / Rotation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** They can suffer order dependence, wrapping and gimbal-like singularities; quaternions/vectors may be better for interpolation/composition.
- **Strong 3-Year-Engineer Answer:** Rotators are convenient, but I avoid naive component interpolation for arbitrary 3D orientation. For smooth rotation I choose quaternions or vector-based construction where appropriate, handle shortest path and clamp/normalise. I still use rotators when UI/control semantics are yaw/pitch/roll.
- **Common Weak Answer:** "Lerp pitch/yaw/roll."
- **Follow-up Questions:** shortest path? wrap at 180? quaternion sign? aim constraints?
- **Hands-on Verification Task:** Compare rotator component lerp and quaternion slerp across a 180-degree crossing.
- **Sources:** [SRC-MATH-006], [SRC-MATH-007], [SRC-MATH-008]
- **Version Notes:** Exact helper functions should be checked in target API.

### Question: Why is custom AI sense registration a lifecycle problem?

- **Category:** AI / Custom Senses
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Sense config, stimuli sources, perception components, ageing and listener registration all have lifecycle and world-scope concerns.
- **Strong 3-Year-Engineer Answer:** A custom sense is not just an event. I verify sense class/config, source registration, listener updates, stimulus age/strength, world ownership and teardown. Bugs often look like "AI never reacts" but come from missing source registration, wrong sense config, expired stimulus or not updating listeners after possession/spawn.
- **Common Weak Answer:** "Subclass UAISense and broadcast."
- **Follow-up Questions:** source registration? stimulus age? listener update? world?
- **Hands-on Verification Task:** Implement a noise-like custom stimulus and prove perception memory ages out correctly.
- **Sources:** [SRC-AI-016], [SRC-AI-017], [SRC-AI-018], [SRC-AI-019]
- **Version Notes:** AI Perception internals are branch-sensitive.

### Question: Why can ZoneGraph/Mass crowd routes fail at runtime?

- **Category:** MassAI / ZoneGraph
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Lane data, tags, fragments, processor order, spawn position and runtime availability must all align.
- **Strong 3-Year-Engineer Answer:** ZoneGraph route following depends on authored lanes, query filters/tags, Mass fragments, processor order and runtime subsystem readiness. I debug with ZoneGraph/Mass debugger views, entity fragment inspection, lane validity and fallback behaviour when no route exists. It is plugin/version-sensitive and should be proven on target branch.
- **Common Weak Answer:** "Place ZoneGraph and Mass will follow it."
- **Follow-up Questions:** lane tags? fragments? processor phase? fallback? debugger?
- **Hands-on Verification Task:** Spawn Mass agents with a wrong lane tag and diagnose empty route assignment.
- **Sources:** [SRC-MASS-009], [SRC-MASS-010], [SRC-MASS-007], [SRC-AI-006]
- **Version Notes:** ZoneGraph/MassAI APIs are plugin/branch sensitive.

### Question: Why do StateTree tasks need explicit data-flow design?

- **Category:** StateTree / Gameplay AI
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Inputs, outputs, instance data and external data determine whether tasks observe current state or stale assumptions.
- **Strong 3-Year-Engineer Answer:** StateTree is not magic state management. I define which data each evaluator/task reads/writes, what is per-instance, what is external, and how transitions react to changes. Debugging checks active state, data view, task completion, event flow and authority boundaries.
- **Common Weak Answer:** "StateTree replaces all AI logic."
- **Follow-up Questions:** evaluators? external data? instance data? transition event?
- **Hands-on Verification Task:** Build an interact state whose target becomes invalid mid-task and prove transition cleanup.
- **Sources:** [SRC-AI-003], [SRC-AI-010], [SRC-AI-011]
- **Version Notes:** StateTree APIs evolve across UE5.

### Question: What makes Smart Object claims fail in multiplayer?

- **Category:** Smart Objects / Multiplayer
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Reservation/claim authority, slot state, actor lifetime and replicated visibility must be coordinated.
- **Strong 3-Year-Engineer Answer:** A Smart Object slot claim should be authority-owned if it affects gameplay. I validate availability on the server, handle user abort/death/despawn, release slots symmetrically and replicate only necessary state. Client-only claims can create double use or stuck reservations.
- **Common Weak Answer:** "Client claims the slot it sees."
- **Follow-up Questions:** server claim? release? slot invalid? user dies?
- **Hands-on Verification Task:** Two clients race to claim one slot; prove one server-authoritative winner and clean loser UI.
- **Sources:** [SRC-AI-011], [SRC-AI-012], [SRC-AI-013], [SRC-NET-001]
- **Version Notes:** Smart Object APIs are branch/plugin sensitive.

### Question: Why can packaged mobile performance differ radically from PIE?

- **Category:** Platform / Mobile Profiling
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** RHI, shader formats, device profiles, thermal state, packaging, IO, resolution and platform services differ from editor.
- **Strong 3-Year-Engineer Answer:** PIE is not a target performance proof. I profile packaged target builds on representative devices with stable scenario scripts, device profile/scalability settings, thermal/battery notes, CPU/GPU/memory captures and platform tools. Then I correlate Unreal timings with platform counters rather than trusting editor results.
- **Common Weak Answer:** "It is fast in editor."
- **Follow-up Questions:** device profile? thermal? RHI? packaged assets? platform profiler?
- **Hands-on Verification Task:** Compare one scene in PIE, standalone and packaged Android/iOS target with matching camera path.
- **Sources:** [SRC-PLAT-005], [SRC-PLAT-006], [SRC-PERF-009], [SRC-PERF-003]
- **Version Notes:** Platform tools and device support are target-specific.

### Question: What should a crash report contain to be actionable?

- **Category:** Build / Crash Triage
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Symbols/build ID, callstack, logs, platform, scenario markers, repro steps and artifact identity.
- **Strong 3-Year-Engineer Answer:** A crash without symbols or artifact identity is often noise. I collect exact build, changelist, platform, config, map/scenario, logs before crash, callstack with symbols, minidump, user/session metadata and recent events. For automation, I make crash upload and symbol publishing part of the release graph.
- **Common Weak Answer:** "Send the callstack."
- **Follow-up Questions:** symbols? build ID? logs? repro? automation?
- **Hands-on Verification Task:** Trigger a controlled crash in a packaged build and prove symbolicated report links to artifact.
- **Sources:** [SRC-BUILD-018], [SRC-BUILD-015], [SRC-BUILD-017]
- **Version Notes:** Crash reporting pipeline is platform/studio specific.

### Question: Why should console and Apple profiling be source-safe in public notes?

- **Category:** Platform / Profiling
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Methodology can be discussed publicly, but proprietary platform counters/devkit details may be NDA-controlled.
- **Strong 3-Year-Engineer Answer:** I separate source-safe evidence categories from confidential platform details. Public notes can discuss Unreal timings, captures, memory trends, CPU/GPU split, frame pacing and official public Apple/Xcode workflows. Console-specific devkit counters, thresholds and internal tool details should stay in authorised documentation.
- **Common Weak Answer:** "List every counter you saw."
- **Follow-up Questions:** public docs? NDA? methodology? report redaction?
- **Hands-on Verification Task:** Redact a platform profiling report into public-safe method, evidence categories and private appendix.
- **Sources:** [SRC-PLAT-018], [SRC-PLAT-021], [SRC-PLAT-022], [SRC-PERF-003]
- **Version Notes:** Platform documentation access and policies vary.

### Question: What is a good acceptance criterion for a hands-on Unreal project?

- **Category:** Interview Projects / Evidence
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It proves behaviour, failure injection, profiling/debug evidence and source/version caveats, not just feature completion.
- **Strong 3-Year-Engineer Answer:** A strong project defines target engine/platform, minimum feature, deliberate bugs, debug/profiling captures, multiplayer or packaged proof where relevant, and a decision memo. Interviewers value evidence that the candidate can diagnose and defend trade-offs, not only a demo video.
- **Common Weak Answer:** "It works in editor."
- **Follow-up Questions:** packaged? profiler? failure injection? source IDs? acceptance tests?
- **Hands-on Verification Task:** Add failure injection and evidence requirements to one existing portfolio project.
- **Sources:** [SRC-PERF-003], [SRC-BUILD-017], [SRC-NET-010], [SRC-ARCH-012]
- **Version Notes:** Evidence depends on target role and platform.

### Question: How should you answer "tell me about a hard bug" in an Unreal interview?

- **Category:** Interviewing / Behavioural Technical
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Describe symptom, scope, hypotheses, evidence, root cause, fix, verification and prevention.
- **Strong 3-Year-Engineer Answer:** I structure it as a debugging case study: production symptom, constrained reproduction, subsystem ownership, tools used, hypotheses eliminated, root cause, minimal fix, tests/profiling after fix and what changed to prevent recurrence. I avoid vague hero stories and include one trade-off or mistake.
- **Common Weak Answer:** "I tried things until it worked."
- **Follow-up Questions:** what evidence? what hypothesis was wrong? what test prevents regression?
- **Hands-on Verification Task:** Write one bug postmortem from a project using this structure.
- **Sources:** [SRC-PERF-003], [SRC-BUILD-018], [SRC-ARCH-012]
- **Version Notes:** Interview format varies; evidence-first structure is stable.

### Question: How should you handle uncertainty in a version-sensitive Unreal interview answer?

- **Category:** Interviewing / Version Sensitivity
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** State the stable concept, label the version-sensitive part and explain how you would verify the target branch.
- **Strong 3-Year-Engineer Answer:** Unreal evolves quickly. A strong answer distinguishes concepts from exact APIs: "The architecture is X; in UE5.3-5.6 I would verify header Y/config Z before copying signatures." That is better than hallucinating specifics. I mention official docs, source headers, small compile proof and target-platform test.
- **Common Weak Answer:** "I think the API is probably..."
- **Follow-up Questions:** where verify? what if docs are maintained newer? how prove?
- **Hands-on Verification Task:** Take one branch-sensitive API from this curriculum and produce a source/header verification note.
- **Sources:** [SRC-BUILD-001], [SRC-EPIC-002], [SRC-NET-020], [SRC-RENDER-003]
- **Version Notes:** This is a meta-pattern for all version-sensitive UE answers.

## Final Question Target Supplement

### Question: Why can editor-only data accidentally enter runtime code?

- **Category:** Build / Runtime Boundary
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Editor modules, metadata, validation helpers and asset tooling can be referenced from runtime modules unless boundaries are explicit.
- **Strong 3-Year-Engineer Answer:** Runtime modules should not depend on editor-only APIs. I inspect module dependencies, preprocessor guards, plugin module types and packaged build errors. The fix is to separate editor tooling from runtime code and move shared schema into a runtime-safe module.
- **Common Weak Answer:** "It compiles in editor, so it is fine."
- **Follow-up Questions:** module type? `WITH_EDITOR`? packaged build? dependency graph?
- **Hands-on Verification Task:** Add an editor utility dependency to runtime code and catch it in a packaged build.
- **Sources:** [SRC-BUILD-002], [SRC-BUILD-003], [SRC-BUILD-005], [SRC-BUILD-017]
- **Version Notes:** Module rules are branch/toolchain sensitive.

### Question: What makes a gameplay tag hierarchy maintainable?

- **Category:** Gameplay Tags / Data Design
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Clear naming, ownership, validation, limited ambiguity and documented semantics for broad versus specific tags.
- **Strong 3-Year-Engineer Answer:** Tags are a shared semantic contract. I keep hierarchy meaning consistent, avoid duplicate synonyms, document who owns tag changes and add validation for required/forbidden combinations. Tags used in networking, GAS and data assets need migration care because renames affect content and saves.
- **Common Weak Answer:** "Just add tags when needed."
- **Follow-up Questions:** hierarchy? migration? validation? broad query?
- **Hands-on Verification Task:** Create a tag validation rule that catches duplicate damage category semantics.
- **Sources:** [SRC-GAS-006], [SRC-ARCH-012], [SRC-ARCH-010]
- **Version Notes:** Tag loading/redirect rules are project-sensitive.

### Question: How should you design a cooldown system without GAS?

- **Category:** Gameplay Systems / Cooldowns
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Centralise authority, time source, identifiers, prediction policy, UI projection and save/replication semantics.
- **Strong 3-Year-Engineer Answer:** Cooldowns need stable action IDs, authority-owned start/end times or remaining durations, server validation, optional client prediction and UI reconciliation. I avoid scattered timers on UI or abilities. If multiplayer, clients request actions and server commits or rejects cooldown state.
- **Common Weak Answer:** "Set a timer on the button."
- **Follow-up Questions:** server time? save/load? prediction? shared cooldown groups?
- **Hands-on Verification Task:** Implement a server-authoritative cooldown with local UI prediction and rejection cleanup.
- **Sources:** [SRC-NET-001], [SRC-ARCH-005], [SRC-ARCH-010]
- **Version Notes:** Time source and replication policy are project-specific.

### Question: Why can save/load break event-driven quest systems?

- **Category:** Save Systems / Quests
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Events that already happened are gone after load unless durable quest state is restored and reconciled.
- **Strong 3-Year-Engineer Answer:** Event subscriptions are for future changes; saves need durable objective state, content IDs, versions and restore logic. On load I rebuild quest state, query relevant world state and emit a restore-ready snapshot rather than replaying reward events. Missed event bugs usually come from no initial query.
- **Common Weak Answer:** "Subscribe to the event again."
- **Follow-up Questions:** old event? restore mode? reward replay? migration?
- **Hands-on Verification Task:** Save after collecting two of three items, reload and prove quest progress without replaying rewards.
- **Sources:** [SRC-ARCH-003], [SRC-ARCH-006], [SRC-ARCH-012], [SRC-WORLD-001]
- **Version Notes:** Save schema and event bus design are project-specific.

### Question: What is the difference between command and event in gameplay architecture?

- **Category:** Gameplay Architecture / Messaging
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** A command asks a system to do something; an event reports that something happened.
- **Strong 3-Year-Engineer Answer:** Commands have an owner, validation path and result. Events are observations after a state change and should not usually be treated as requests. Mixing them causes duplicate effects and unclear authority. I use operation IDs/revisions for commands and idempotent event subscribers for projections.
- **Common Weak Answer:** "Both are messages."
- **Follow-up Questions:** validation? retry? idempotency? authority?
- **Hands-on Verification Task:** Refactor an inventory pickup event into a command plus committed event.
- **Sources:** [SRC-ARCH-004], [SRC-ARCH-003], [SRC-ARCH-006]
- **Version Notes:** Messaging implementation is project-specific.

### Question: Why can object pooling create stale-state bugs?

- **Category:** Gameplay Architecture / Object Pool
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Reused objects can retain bindings, timers, parameters, owner references or replicated/presentation state if reset is incomplete.
- **Strong 3-Year-Engineer Answer:** Pooling trades allocation cost for lifecycle complexity. Every pooled object needs an acquire/reset/release contract covering delegates, timers, components, materials, audio/VFX parameters and network authority. I add poison/debug generations to catch stale callbacks.
- **Common Weak Answer:** "Pooling just reuses actors."
- **Follow-up Questions:** reset contract? delegates? owner? generation? replication?
- **Hands-on Verification Task:** Pool projectile Actors, intentionally leave an old owner/damage value, then catch with generation logging.
- **Sources:** [SRC-ARCH-008], [SRC-EPIC-015], [SRC-NET-001]
- **Version Notes:** Pooling design depends on Actor/component lifecycle.

### Question: What makes a service locator dangerous?

- **Category:** Gameplay Architecture / Dependencies
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It can hide dependencies, scope, lifetime and authority behind global lookups.
- **Strong 3-Year-Engineer Answer:** Service location can be pragmatic for engine/world services, but overuse makes systems hard to test and reason about. I prefer explicit dependencies at composition boundaries and narrow interfaces. If using a locator/subsystem, I document scope, lifetime, threading and fallback behaviour.
- **Common Weak Answer:** "Global access is convenient."
- **Follow-up Questions:** testability? world scope? multiplayer? teardown?
- **Hands-on Verification Task:** Replace a global service lookup in combat code with an injected interface.
- **Sources:** [SRC-ARCH-007], [SRC-EPIC-017], [SRC-ARCH-014]
- **Version Notes:** Subsystem and service lifetimes are branch/project sensitive.

### Question: Why should validation run in editor and CI?

- **Category:** Tools / Data Validation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Editor-only validation catches authoring mistakes early, while CI prevents invalid content from reaching packages.
- **Strong 3-Year-Engineer Answer:** Data validation should check IDs, references, platform constraints, cookability, naming and gameplay invariants. Running it only manually is unreliable. I integrate editor validation with commandlet/CI paths and make failures actionable with asset path, rule and fix guidance.
- **Common Weak Answer:** "Designers will notice broken assets."
- **Follow-up Questions:** commandlet? CI gate? asset path? false positives?
- **Hands-on Verification Task:** Add a validation rule for duplicate item IDs and run it in a commandlet/CI step.
- **Sources:** [SRC-BUILD-009], [SRC-BUILD-014], [SRC-ARCH-012], [SRC-ASSET-011]
- **Version Notes:** Validation APIs are branch-sensitive.

### Question: When should you write a commandlet?

- **Category:** Tools / Commandlets
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** For repeatable headless editor/asset/build operations that should run in automation.
- **Strong 3-Year-Engineer Answer:** Commandlets are useful for validation, asset migration, data export/import, cook checks or batch processing where UI is unnecessary. I design them idempotent, source-control aware, log-rich and safe for CI. For interactive workflows, editor utility widgets or details customisations may fit better.
- **Common Weak Answer:** "Use commandlets for every tool."
- **Follow-up Questions:** headless? idempotent? source control? logs? editor utility alternative?
- **Hands-on Verification Task:** Build a commandlet that validates and optionally fixes asset naming.
- **Sources:** [SRC-BUILD-012], [SRC-BUILD-011], [SRC-BUILD-008], [SRC-BUILD-010]
- **Version Notes:** Commandlet APIs and editor scripting vary by branch.

### Question: Why can Python editor scripts be unsafe for production data migration?

- **Category:** Tools / Editor Scripting
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Without transaction, validation, source-control and restart recovery, batch edits can corrupt or partially migrate assets.
- **Strong 3-Year-Engineer Answer:** Python is powerful for editor automation, but migrations need dry run, backups/source-control checkout, asset load/save policy, validation before/after and crash recovery. I avoid one-off scripts that mutate thousands of assets without an audit log and rollback plan.
- **Common Weak Answer:** "Run a Python loop over all assets."
- **Follow-up Questions:** dry run? transactions? source control? partial failure?
- **Hands-on Verification Task:** Write a dry-run asset migration report before enabling save mode.
- **Sources:** [SRC-BUILD-011], [SRC-BUILD-014], [SRC-ASSET-004], [SRC-ARCH-012]
- **Version Notes:** Editor scripting APIs are branch-sensitive.

### Question: What does a rendering optimisation decision memo include?

- **Category:** Rendering / Performance Process
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Scenario, bottleneck evidence, changed content/settings, before/after captures, visual impact and platform risk.
- **Strong 3-Year-Engineer Answer:** I do not present "reduced shadows" as an optimisation without context. A memo names target hardware, camera path, resolution, scalability/device profile, CPU/GPU split, GPU capture findings, before/after frame time and visual trade-offs. It also notes whether the change shifts cost elsewhere.
- **Common Weak Answer:** "Lower quality until FPS improves."
- **Follow-up Questions:** target platform? GPU capture? visual delta? scalability?
- **Hands-on Verification Task:** Optimise one GPU-bound scene and write a before/after decision memo.
- **Sources:** [SRC-RENDER-004], [SRC-RENDER-009], [SRC-RENDER-019], [SRC-PERF-003]
- **Version Notes:** RHI/platform tooling differs.

### Question: Why can render targets become hidden memory/performance costs?

- **Category:** Rendering / Render Targets
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** They consume GPU memory/bandwidth and may add extra passes, resolves and update work.
- **Strong 3-Year-Engineer Answer:** Scene captures, UI retainers and custom render targets can silently add expensive offscreen rendering. I check resolution, format, update frequency, capture show flags, mips, lifetime and platform memory. Then I profile with GPU captures and Memory/LLM where supported.
- **Common Weak Answer:** "Render target is just a texture."
- **Follow-up Questions:** format? resolution? update frequency? scene capture? retainer?
- **Hands-on Verification Task:** Create 10 scene captures, reduce update rate/resolution and compare GPU/memory.
- **Sources:** [SRC-RENDER-016], [SRC-RENDER-017], [SRC-RENDER-018], [SRC-PERF-006]
- **Version Notes:** Render target formats/support are platform-sensitive.

### Question: What is shader permutation explosion?

- **Category:** Rendering / Shaders
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Many static switches/features/material variants generate large numbers of shader combinations to compile and ship.
- **Strong 3-Year-Engineer Answer:** Permutations can increase compile time, package size, memory and PSO complexity. I reduce unnecessary static switches, split materials carefully, use runtime parameters where appropriate and track shader/PSO evidence. Optimisation needs a material feature matrix, not blind consolidation.
- **Common Weak Answer:** "One master material solves everything."
- **Follow-up Questions:** static switch? compile time? package size? PSO? runtime parameter?
- **Hands-on Verification Task:** Add three static switches to a master material and inspect permutation/compile impact.
- **Sources:** [SRC-RENDER-013], [SRC-RENDER-014], [SRC-RENDER-012], [SRC-RENDER-004]
- **Version Notes:** Shader pipeline differs by RHI/branch.

### Question: Why can HLOD be a build-system problem, not just a world-building feature?

- **Category:** World Partition / Build Pipeline
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** HLOD generation creates derived assets that must be built, validated, versioned and packaged reproducibly.
- **Strong 3-Year-Engineer Answer:** HLOD affects runtime performance, content workflow and CI. I need deterministic builder commandlets or documented editor steps, source-control policy for generated assets, cook/package validation and visual/performance checks. Broken HLODs often surface as package size, missing proxy meshes or wrong streaming behaviour.
- **Common Weak Answer:** "Click Build HLODs before release."
- **Follow-up Questions:** commandlet? source control? package? visual diff? CI?
- **Hands-on Verification Task:** Build HLODs on a clean machine and verify generated assets package and stream correctly.
- **Sources:** [SRC-WORLD-003], [SRC-ASSET-010], [SRC-BUILD-017], [SRC-BUILD-015]
- **Version Notes:** World Partition builder APIs are branch-sensitive.

### Question: Why can OFPA create merge noise?

- **Category:** World Building / Source Control
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** One File Per Actor improves parallel editing but creates many actor asset files whose churn needs naming, ownership and review discipline.
- **Strong 3-Year-Engineer Answer:** OFPA reduces map-file conflicts, but teams still need actor naming, folder ownership, validation and review tooling. Large moves or procedural/editor scripts can touch many files. I pair OFPA with changelist hygiene, validation and content ownership rules.
- **Common Weak Answer:** "OFPA removes merge conflicts."
- **Follow-up Questions:** actor files? ownership? generated changes? validation?
- **Hands-on Verification Task:** Move a level section under OFPA and inspect source-control file churn.
- **Sources:** [SRC-WORLD-002], [SRC-ASSET-010], [SRC-BUILD-009]
- **Version Notes:** OFPA workflows depend on project/source-control tooling.

### Question: Why can Android thermal throttling invalidate a benchmark?

- **Category:** Platform / Mobile Profiling
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Performance changes over time as the device heats and changes CPU/GPU frequency or scheduling policy.
- **Strong 3-Year-Engineer Answer:** Mobile profiling needs warmup, thermal/battery state, repeated runs and platform counters. A 30-second cold run can misrepresent sustained gameplay. I record device model, OS, temperature/thermal state where possible, run duration, scenario path and Unreal/platform timings.
- **Common Weak Answer:** "Average one short run."
- **Follow-up Questions:** sustained FPS? ADPF? CPU/GPU frequency? battery? repeatability?
- **Hands-on Verification Task:** Run a 10-minute mobile performance loop and compare first minute versus final minute.
- **Sources:** [SRC-PLAT-015], [SRC-PLAT-016], [SRC-PLAT-017], [SRC-PERF-009]
- **Version Notes:** Thermal APIs/device behaviour vary widely.

### Question: What should a platform device profile change prove?

- **Category:** Platform / Scalability
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It improves the target scenario on target hardware without unacceptable visual, memory or compatibility regressions.
- **Strong 3-Year-Engineer Answer:** Device profiles are production configuration, not guesses. I document target devices, inherited profile chain, changed CVars/settings, before/after performance, memory and visual captures. I test edge devices and make sure the change applies in packaged builds.
- **Common Weak Answer:** "Set lower scalability for mobile."
- **Follow-up Questions:** profile inheritance? packaged? visual proof? memory? device list?
- **Hands-on Verification Task:** Change one mobile shadow/resolution profile setting and verify on two device tiers.
- **Sources:** [SRC-PERF-007], [SRC-PERF-008], [SRC-PLAT-005], [SRC-PLAT-006]
- **Version Notes:** Device profile inheritance and CVars vary by branch.

### Question: Why should automation distinguish flaky from deterministic failures?

- **Category:** Automation / CI
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Deterministic failures should block; flaky infrastructure or tests need quarantine/triage without hiding real regressions.
- **Strong 3-Year-Engineer Answer:** A good gate records artifact identity, device, first-cause log, retry policy and classification. Blind reruns hide product failures; blocking on lab flakes destroys trust. I use quarantine with owners, reproduction attempts, trend data and promotion rules.
- **Common Weak Answer:** "Rerun until green."
- **Follow-up Questions:** first cause? quarantine? ownership? artifact identity?
- **Hands-on Verification Task:** Classify three failing automation runs as product, lab infrastructure and flaky test.
- **Sources:** [SRC-PERF-011], [SRC-PLAT-009], [SRC-BUILD-018]
- **Version Notes:** Automation frameworks are branch/platform sensitive.

### Question: Why can multiplayer tests pass in PIE but fail in multi-process dedicated server?

- **Category:** Networking / Testing
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** PIE can share process/editor assumptions; dedicated server exposes real net mode, ownership, timing, packaging and authority boundaries.
- **Strong 3-Year-Engineer Answer:** PIE is useful for iteration, but real networking bugs often require standalone or dedicated multi-process tests with packet emulation. I check authority, owning connections, asset availability, RPC permissions, replication timing and build configuration. A feature is not multiplayer-proven until tested outside happy-path PIE.
- **Common Weak Answer:** "It works with two PIE windows."
- **Follow-up Questions:** dedicated server? packet lag? packaged? ownership? net mode?
- **Hands-on Verification Task:** Reproduce a Server RPC ownership failure in multi-process dedicated server.
- **Sources:** [SRC-NET-001], [SRC-NET-003], [SRC-NET-006], [SRC-NET-010]
- **Version Notes:** Test setup differs by branch/project.

### Question: How do you know whether an optimisation moved the bottleneck?

- **Category:** Profiling / Method
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Compare before/after full frame breakdown, not just one improved metric.
- **Strong 3-Year-Engineer Answer:** Optimisation can shift cost from GPU to CPU, memory, IO or hitches. I keep the scenario constant, capture Game/Render/GPU/memory/hitch data before and after, and inspect visual/UX changes. A good report says what improved, what got worse and whether the product goal was met.
- **Common Weak Answer:** "FPS went up once."
- **Follow-up Questions:** scenario control? variance? memory? hitches? visual quality?
- **Hands-on Verification Task:** Optimise draw calls and show whether game thread, render thread or GPU became next bottleneck.
- **Sources:** [SRC-PERF-001], [SRC-PERF-002], [SRC-PERF-003], [SRC-PERF-009]
- **Version Notes:** Profiling tool availability is build/platform sensitive.

### Question: Why is "just make it async" not a performance strategy?

- **Category:** Performance / Async
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Async work still costs CPU/memory/synchronisation and may add latency, contention or unsafe UObject access.
- **Strong 3-Year-Engineer Answer:** I use async when work can be safely partitioned and merged, not to hide design problems. I snapshot data, avoid off-thread UObject mutation, limit task fan-out, measure scheduling/merge cost and validate cancellation/lifetime. Sometimes batching, caching or reducing work is better.
- **Common Weak Answer:** "Move it to another thread."
- **Follow-up Questions:** data race? merge? cancellation? task overhead? UObject thread safety?
- **Hands-on Verification Task:** Parallelise a query with snapshot/merge and compare against a simpler cached single-thread version.
- **Sources:** [SRC-CPP-020], [SRC-PERF-003], [SRC-EPIC-009]
- **Version Notes:** Unreal task APIs and UObject thread rules are branch-sensitive.

### Question: Why should generated code be part of source control policy?

- **Category:** Build / Generated Artifacts
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Teams must decide which generated artifacts are reproducible build outputs and which are authored assets that must be reviewed/versioned.
- **Strong 3-Year-Engineer Answer:** Some generated files should never be committed; others, like generated HLODs, baked data or platform configs, may be intentional artefacts. The policy must support clean-agent builds, reviewability, determinism and rollback. Ambiguity causes missing files, huge diffs and local-only builds.
- **Common Weak Answer:** "Commit generated files if build needs them."
- **Follow-up Questions:** reproducible? reviewed? clean agent? source control churn?
- **Hands-on Verification Task:** Classify DDC, generated headers, HLODs, cooked packages and validation reports into commit/ignore/archive buckets.
- **Sources:** [SRC-BUILD-001], [SRC-BUILD-015], [SRC-ASSET-008], [SRC-WORLD-003]
- **Version Notes:** Policy depends on project pipeline and branch.

### Question: What is the simplest useful portfolio evidence packet?

- **Category:** Interview Projects / Portfolio
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** A short problem statement, source/version scope, before/after evidence, failure injection, profiler/debug captures and decision memo.
- **Strong 3-Year-Engineer Answer:** I would rather show one well-evidenced system than many shallow demos. The packet should prove the candidate can design, diagnose, measure and communicate trade-offs. Include exact UE version, target platform, reproducible steps, screenshots/captures/logs and what was deliberately not built.
- **Common Weak Answer:** "Show a gameplay video."
- **Follow-up Questions:** source IDs? profiler capture? failure case? trade-off?
- **Hands-on Verification Task:** Create a one-page evidence packet for Project 3 or Project 9.
- **Sources:** [SRC-PERF-003], [SRC-NET-010], [SRC-BUILD-017], [SRC-ARCH-012]
- **Version Notes:** Portfolio expectations vary by role.

## Target-Proof, Networking, Profiling, Build, and Specialist Expansion

### Question: What belongs in a packaged performance proof manifest?

- **Category:** Profiling / Release Evidence
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Build identity, target environment, scenario, metrics, thresholds, command lines, artifacts and owners.
- **Strong 3-Year-Engineer Answer:** A manifest should make the package reproducible: engine/project revision, target/config/platform, build ID, device/profile, map, command line, warm-up, sample window, CSV/trace paths, symbols and pass/fail thresholds. Without identity, CSV files and traces become anecdotes. I also record unsupported branch features instead of pretending all tools exist everywhere.
- **Common Weak Answer:** "Attach the CSV."
- **Follow-up Questions:** build ID? active profile? symbols? threshold owner?
- **Hands-on Verification Task:** Fill a manifest for one packaged scenario and ask another engineer to reproduce it.
- **Sources:** [SRC-PERF-009], [SRC-BUILD-017], [SRC-BUILD-018]
- **Version Notes:** Command fields and artifact paths are project-specific.

### Question: Why is an active Device Profile dump part of performance evidence?

- **Category:** Profiling / Scalability
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Source files do not prove runtime selection; the packaged build must show which profile and CVars were actually active.
- **Strong 3-Year-Engineer Answer:** Device profiles and scalability can completely change render cost, resolution, streaming pools and feature quality. I capture applied CVars/profile inheritance in logs or a dump before comparing results. Otherwise a regression can be hidden by lower quality or a fix can be credited to the wrong change.
- **Common Weak Answer:** "The profile is configured in DefaultDeviceProfiles."
- **Follow-up Questions:** inherited profile? packaged? dynamic resolution? low-tier readability?
- **Hands-on Verification Task:** Run the same package with two profiles and compare CVar dumps.
- **Sources:** [SRC-PERF-007], [SRC-PERF-008], [SRC-PLAT-005]
- **Version Notes:** Profile selectors and CVars are branch/platform sensitive.

### Question: When should a CSV performance gate block a change?

- **Category:** Profiling / Automation
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** When repeated, controlled runs exceed actionable thresholds tied to an owner and artifact evidence.
- **Strong 3-Year-Engineer Answer:** I do not block on one noisy FPS average. A blocking gate needs fixed scenario, warm-up, repeated runs, active profile proof and stable metrics like P95/P99 frame time, hitch count, first-interactive time or memory growth. Failing output should point to an owner and request causal capture only for that failing window.
- **Common Weak Answer:** "Fail if FPS drops below 60 once."
- **Follow-up Questions:** variance? rolling baseline? reporting-only metrics? causal escalation?
- **Hands-on Verification Task:** Design thresholds for three runs of a UI, combat and traversal scenario.
- **Sources:** [SRC-PERF-009], [SRC-PERF-011]
- **Version Notes:** CSV categories and automation hooks vary by branch.

### Question: How do you separate a CSV signal from causal profiling?

- **Category:** Profiling / Method
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** CSV detects repeatable regressions cheaply; Insights/GPU/platform captures explain the failing window.
- **Strong 3-Year-Engineer Answer:** I use CSV for long-run and automation metrics because it is cheap enough for repeated packaged scenarios. If CSV fails, I take a small targeted trace around the scenario window and classify CPU, Draw/RHI, GPU, memory, I/O, streaming or shader/PSO cost. Treating CSV alone as root cause usually leads to guesswork.
- **Common Weak Answer:** "CSV tells you what caused the hitch."
- **Follow-up Questions:** marker alignment? trace overhead? causal owner?
- **Hands-on Verification Task:** Trigger a CSV hitch and capture a 10-second Insights window around it.
- **Sources:** [SRC-PERF-003], [SRC-PERF-009]
- **Version Notes:** Trace channels and marker APIs are target-sensitive.

### Question: What is a good packaged hitch triage order?

- **Category:** Profiling / Packaged Builds
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Confirm package/settings, locate the window, classify thread/resource ownership, make one causal change, rerun same scenario.
- **Strong 3-Year-Engineer Answer:** I first verify build ID, platform, active profile, VSync/dynamic resolution and scenario reproducibility. Then I use CSV/logs to locate the hitch and capture a bounded trace. I check Game, Draw/RHI, GPU, loading/file I/O, memory/GC, shader/PSO and streaming before changing code or content.
- **Common Weak Answer:** "Packaged builds are slower."
- **Follow-up Questions:** cold cache? PSO? streaming? memory? Draw/RHI?
- **Hands-on Verification Task:** Reproduce one packaged-only hitch and classify it using a trace.
- **Sources:** [SRC-PERF-003], [SRC-RENDER-014], [SRC-ASSET-006]
- **Version Notes:** Tooling and shader/PSO workflows vary by platform.

### Question: How would you prove a memory budget regression?

- **Category:** Profiling / Memory
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Combine stable checkpoint budgets with LLM/tag ownership and Memory Insights or equivalent allocation evidence.
- **Strong 3-Year-Engineer Answer:** I define checkpoints such as boot, menu, map loaded, peak combat, travel and return-to-menu. LLM or tag totals identify subsystem ownership; Memory Insights explains allocation lifetime, callstacks, growth and churn. Then I separate capacity, leak-like retention, churn, streaming pressure and GC/object count issues.
- **Common Weak Answer:** "Increase the pool size."
- **Follow-up Questions:** checkpoint? tag ownership? churn? long session?
- **Hands-on Verification Task:** Run five menu open/close cycles and report memory growth by tag/callstack.
- **Sources:** [SRC-PERF-006], [SRC-PERF-010]
- **Version Notes:** LLM tags and memory tracing depend on build/platform.

### Question: What makes a BuildGraph file better than a shell blob?

- **Category:** Build / BuildGraph
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** It exposes parameters, dependencies, agents, labels and artifacts as a graph rather than implicit command order.
- **Strong 3-Year-Engineer Answer:** BuildGraph is useful when release work has build, cook, package, tests, symbols, crash proof and publish steps. I want dependencies such as `PerfGate` requiring `Package`, and `CrashProof` requiring package plus symbols. A shell blob can work locally, but it hides skipped work, stale outputs and failure ownership.
- **Common Weak Answer:** "BuildGraph is just XML for running commands."
- **Follow-up Questions:** artifacts? agents? labels? stale package?
- **Hands-on Verification Task:** Draw a graph where performance cannot run before packaging.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-016]
- **Version Notes:** Exact BuildGraph tasks are branch/tooling sensitive.

### Question: When is direct RunUAT enough instead of BuildGraph?

- **Category:** Build / AutomationTool
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** For a simple local or single-lane package proof where dependency graph/artifact publishing is not yet needed.
- **Strong 3-Year-Engineer Answer:** RunUAT/BuildCookRun is appropriate for a documented build-cook-stage-package command, especially early in a project or for one-off validation. Once the pipeline needs multiple targets, agents, symbols, smoke/performance gates and reusable artifacts, BuildGraph or equivalent CI graph becomes more defensible.
- **Common Weak Answer:** "Always use BuildGraph."
- **Follow-up Questions:** artifact flow? cost? release candidate? wrapper?
- **Hands-on Verification Task:** Document a RunUAT package command and list when it should graduate to graph automation.
- **Sources:** [SRC-BUILD-016], [SRC-BUILD-017], [SRC-BUILD-015]
- **Version Notes:** RunUAT flags must be checked in the target branch.

### Question: How do you find the first useful error in a RunUAT failure?

- **Category:** Build / Debugging
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Identify the failed phase and search upward from the final AutomationTool wrapper to the first causal build/cook/stage/runtime error.
- **Strong 3-Year-Engineer Answer:** The final UAT exception often says the command failed, not why. I classify phase, inspect the earliest concrete error, then check command line, target/config/platform, cook scope, staged files and environment. Fixing the wrapper message or deleting caches does not prevent recurrence.
- **Common Weak Answer:** "Clean Intermediate and rerun."
- **Follow-up Questions:** build vs cook vs stage? first cause? artifact path?
- **Hands-on Verification Task:** Annotate a failing UAT log with phase and first causal line.
- **Sources:** [SRC-BUILD-016], [SRC-BUILD-017]
- **Version Notes:** Logs and wrappers vary by studio.

### Question: What is the difference between cook, stage, package and archive failures?

- **Category:** Build / Packaging
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Cook transforms content for platform; stage assembles runtime layout; package produces distributable containers/layout; archive retains outputs.
- **Strong 3-Year-Engineer Answer:** Missing asset rules usually surface in cook, missing runtime DLLs or plugin files in stage, container/signing/layout issues in package, and missing reproducibility evidence in archive. I classify the phase before fixing because "package failed" is too broad to assign ownership.
- **Common Weak Answer:** "They are all packaging."
- **Follow-up Questions:** staged dependency? cooked asset? archive manifest?
- **Hands-on Verification Task:** Create one artificial failure in each phase and document the log signature.
- **Sources:** [SRC-BUILD-017], [SRC-ASSET-006]
- **Version Notes:** Platform packaging details vary.

### Question: Why is a forced packaged crash part of release proof?

- **Category:** Build / Crash Reporting
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It proves crash reports, build IDs, symbols and routing work before players hit real crashes.
- **Strong 3-Year-Engineer Answer:** A crash pipeline is only useful if the report matches the exact package and symbols. I force a controlled crash in a non-editor package, verify callstack symbolication, log tail, build ID and privacy-safe context, then archive the evidence with the release. Unsymbolicated production crashes are a release-process failure.
- **Common Weak Answer:** "Crash reports show up automatically."
- **Follow-up Questions:** symbol archive? build ID? endpoint? privacy?
- **Hands-on Verification Task:** Force a packaged crash and intentionally test wrong-symbol rejection.
- **Sources:** [SRC-BUILD-018], [SRC-BUILD-017]
- **Version Notes:** Crash routing is project/platform policy.

### Question: How do you prove staged third-party dependencies are correct?

- **Category:** Build / Third-Party Integration
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Build/link and launch on a clean machine without global installs, then inspect staged runtime files.
- **Strong 3-Year-Engineer Answer:** Headers and link libraries prove compile-time integration only. Runtime proof needs platform/config library selection, staged DLL/shared object/framework, clean-machine launch and symbol/debug policy. Developer-machine PATH or editor-adjacent files can hide broken staging.
- **Common Weak Answer:** "It links locally."
- **Follow-up Questions:** delay-load? clean machine? server target? licence?
- **Hands-on Verification Task:** Remove the globally installed dependency and launch the package.
- **Sources:** [SRC-BUILD-004], [SRC-BUILD-017]
- **Version Notes:** Staging rules vary by platform and build system.

### Question: Why is "cook all content" a weak missing-asset fix?

- **Category:** Assets / Packaging
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It hides dependency/cook-rule bugs, bloats package size and can break chunking or platform budgets.
- **Strong 3-Year-Engineer Answer:** If a packaged asset is missing, I inspect map list, Asset Manager rules, hard/soft references, Primary Assets, labels and stage manifests. Cooking everything may pass locally, but it destroys intent and makes releases larger and less predictable. The fix should express the ownership rule that includes the asset.
- **Common Weak Answer:** "Add the Content folder to always cook."
- **Follow-up Questions:** Primary Asset? soft ref? chunk? package size?
- **Hands-on Verification Task:** Fix a missing soft-referenced item through Asset Manager rules rather than broad cook.
- **Sources:** [SRC-ASSET-005], [SRC-ASSET-006], [SRC-BUILD-017]
- **Version Notes:** Cook rules depend on project setup.

### Question: What should a World Partition HLOD builder proof include?

- **Category:** World Partition / HLOD
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Builder command/log, target map, output freshness, packaged inclusion, visual quality and performance impact.
- **Strong 3-Year-Engineer Answer:** A builder log alone is not enough. I prove it ran for the correct map, generated or validated expected output, shipped in the package, improved the intended cost and did not introduce unacceptable visual or transition issues. Stale HLOD should be caught by automation, not by manual release memory.
- **Common Weak Answer:** "HLOD build completed."
- **Follow-up Questions:** stale output? packaged? transition hitch? memory?
- **Hands-on Verification Task:** Change one HLOD-relevant actor and prove the builder/gate detects freshness.
- **Sources:** [SRC-WORLD-003], [SRC-ASSET-010], [SRC-BUILD-017]
- **Version Notes:** Builder commandlets and requirements are branch-sensitive.

### Question: Why can HLOD hide but not fix near-field activation cost?

- **Category:** World Partition / Performance
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** HLOD reduces distant representation cost, but near actors still need loading, registration, collision, nav, materials and gameplay startup.
- **Strong 3-Year-Engineer Answer:** If a hitch occurs at HLOD handoff, I separate proxy cost from near-field cell activation. The fix might be streaming readiness, preloading, actor/component reduction, construction script cleanup, shader/PSO coverage or content budgets. Raising proxy quality cannot fix a BeginPlay or collision setup spike.
- **Common Weak Answer:** "Use better HLOD settings."
- **Follow-up Questions:** transition range? actor activation? shader first use? collision?
- **Hands-on Verification Task:** Capture far, transition and near-field ranges separately and identify the hitch owner.
- **Sources:** [SRC-WORLD-003], [SRC-PERF-003], [SRC-RENDER-011]
- **Version Notes:** HLOD costs are content and platform dependent.

### Question: How do you prove fast travel readiness in a World Partition map?

- **Category:** World Partition / Streaming
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Use a streaming source/readiness check plus project-specific collision, nav and gameplay readiness before teleporting.
- **Strong 3-Year-Engineer Answer:** Cell load is not the whole contract. I log streaming source state, requested cells, Runtime Data Layers, collision/nav readiness and key gameplay actors. A packaged traversal or teleport scenario should commit movement only after readiness or fail gracefully with a loading/fallback state.
- **Common Weak Answer:** "Add a delay before teleport."
- **Follow-up Questions:** collision? Data Layer? timeout? packaged proof?
- **Hands-on Verification Task:** Implement a preloader that logs readiness and rejects premature teleport.
- **Sources:** [SRC-ASSET-010], [SRC-WORLD-001], [SRC-PERF-009]
- **Version Notes:** Exact streaming-source APIs are branch-sensitive.

### Question: What does Runtime Data Layer proof need beyond toggling visibility?

- **Category:** World Partition / Data Layers
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Runtime layer state must be tied to durable gameplay truth, save/load and packaged behaviour.
- **Strong 3-Year-Engineer Answer:** Runtime Data Layers present world phases; they should not be the only authority for quest/save/server state. I prove initial state, transitions, package inclusion, late-load behaviour and reconstruction from durable state. Editor-only layers or unsaved editor state can hide packaged failure.
- **Common Weak Answer:** "The layer is visible in editor."
- **Follow-up Questions:** save authority? late join? initial state? packaged?
- **Hands-on Verification Task:** Save/load a phase change and verify packaged Data Layer state.
- **Sources:** [SRC-WORLD-001], [SRC-ASSET-010]
- **Version Notes:** Data Layer APIs and replication policy are project-sensitive.

### Question: How do you stop manager actors from keeping spatial actors loaded?

- **Category:** World Partition / References
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Avoid hard references from always-loaded systems to unloadable actors; use stable IDs, soft refs, events or registries.
- **Strong 3-Year-Engineer Answer:** In World Partition, reference edges can affect loading bundles. I keep durable truth in managers/save/server systems and represent spatial actors by stable identifiers or soft lookup paths. Then I validate always-loaded actor lists and packaged memory/cell unloading rather than trusting author intent.
- **Common Weak Answer:** "Set Is Spatially Loaded."
- **Follow-up Questions:** hard refs? Level Blueprint? soft ref? durable state?
- **Hands-on Verification Task:** Add a hard manager reference to a spatial actor and observe loaded cell/memory behaviour.
- **Sources:** [SRC-ASSET-010], [SRC-EPIC-009], [SRC-ASSET-003]
- **Version Notes:** Reference loading behaviour is version/content sensitive.

### Question: Why does OFPA need source-control validation?

- **Category:** World Building / OFPA
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** External actor files reduce map-file conflicts but make partial submits and review mapping real risks.
- **Strong 3-Year-Engineer Answer:** OFPA improves collaboration, but encoded files and mass moves can make review hard. I use editor/source-control tooling, actor labels, validation and clean-workspace package tests to catch omitted external actor files or unintended generated churn.
- **Common Weak Answer:** "OFPA solves source control."
- **Follow-up Questions:** partial submit? encoded files? generated actors? clean workspace?
- **Hands-on Verification Task:** Omit one external actor file and document the packaged/editor failure.
- **Sources:** [SRC-WORLD-002], [SRC-ASSET-010]
- **Version Notes:** Workflow depends on source-control integration.

### Question: What makes a RepGraph adoption defensible?

- **Category:** Networking / Replication Graph
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** A profile shows per-connection relevancy/list-building cost or scale needs that RepGraph policy can reduce.
- **Strong 3-Year-Engineer Answer:** I would not adopt RepGraph just because the game is multiplayer. I classify actors by spatial, global, owner-only, team, dormancy and frequency policy, then compare server CPU, bandwidth and correctness under representative actor/client counts. Adoption is justified when graph policy beats default replication complexity without hiding ownership bugs.
- **Common Weak Answer:** "Use RepGraph for big maps."
- **Follow-up Questions:** actor policy? bandwidth? dormancy? splitscreen caveat?
- **Hands-on Verification Task:** Compare default replication with a simple spatial/global RepGraph setup.
- **Sources:** [SRC-NET-007], [SRC-NET-008], [SRC-NET-013], [SRC-NET-010]
- **Version Notes:** RepGraph setup and platform caveats are branch/project sensitive.

### Question: How do you prove a Fast Array implementation is correct?

- **Category:** Networking / Fast Array
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Mutations mark items/array dirty, IDs/keys remain stable, clients receive add/change/remove hooks, and bandwidth improves for representative changes.
- **Strong 3-Year-Engineer Answer:** I test add, modify, remove, reorder, burst and stale-client cases. Correctness includes server authority, stable identity, dirty marking, client prediction/cosmetic policy and late join. Performance proof compares simple replicated arrays with Fast Array under realistic list size and change frequency.
- **Common Weak Answer:** "Fast Array only sends deltas."
- **Follow-up Questions:** dirty marking? item identity? late join? removal hook?
- **Hands-on Verification Task:** Build a 100-item inventory test and compare one-item mutation bandwidth.
- **Sources:** [SRC-NET-015], [SRC-NET-004], [SRC-NET-016]
- **Version Notes:** Exact hooks and debug CVars require target-source confirmation.

### Question: What is a safe saved-move extension boundary?

- **Category:** Networking / Character Movement
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Only include deterministic movement-affecting input/state needed for client prediction and server replay.
- **Strong 3-Year-Engineer Answer:** Saved moves are for reproducing movement, not arbitrary gameplay state. I store compact sprint/dash input flags or vectors, implement clear/set/combine contracts, and prove correction/replay under lag/loss. Damage, inventory and non-deterministic effects remain server authority or separate prediction systems.
- **Common Weak Answer:** "Put anything the client needs in saved moves."
- **Follow-up Questions:** compressed flags? combine? replay? server authority?
- **Hands-on Verification Task:** Add predicted sprint and show server correction when client lies.
- **Sources:** [SRC-NET-009], [SRC-NET-017], [SRC-NET-018], [SRC-NET-019]
- **Version Notes:** Virtuals and packed move details are branch-sensitive.

### Question: Why is saved-move combining a correctness risk?

- **Category:** Networking / Prediction
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Combining moves with different movement-affecting state can lose edges or replay the wrong input.
- **Strong 3-Year-Engineer Answer:** If sprint state, dash edge, custom acceleration or movement mode differs, combining may erase the exact command sequence needed for server reproduction. I define `CanCombineWith` around deterministic equivalence and test one-frame transitions under packet loss and correction.
- **Common Weak Answer:** "Combining just saves bandwidth."
- **Follow-up Questions:** edge events? dash? acceleration? correction?
- **Hands-on Verification Task:** Create a one-frame dash bug caused by over-aggressive move combining.
- **Sources:** [SRC-NET-009], [SRC-NET-018]
- **Version Notes:** Saved-move internals are target-branch sensitive.

### Question: How do you separate movement prediction from ability authority?

- **Category:** Networking / Gameplay Architecture
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Let client predict movement presentation where safe, but keep gameplay effects, damage and cooldown authority on the server or prediction framework.
- **Strong 3-Year-Engineer Answer:** A sprint or dash movement input can be predicted via CharacterMovement saved moves, but damage, invulnerability, stamina spend and cooldowns need server validation. I design one authoritative ledger and make client prediction cosmetic or replay-safe, then test correction under latency.
- **Common Weak Answer:** "The client feels responsive, so accept it."
- **Follow-up Questions:** cooldown? stamina? rollback? correction? cheating?
- **Hands-on Verification Task:** Implement predicted dash movement with server-authoritative cooldown.
- **Sources:** [SRC-NET-009], [SRC-GAS-010], [SRC-NET-020]
- **Version Notes:** NetworkPrediction/GAS behaviour is plugin/project sensitive.

### Question: Why can dynamic replicated subobjects crash or desynchronise?

- **Category:** Networking / Subobjects
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Registration, lifetime, ownership and stale references must align with replication and GC.
- **Strong 3-Year-Engineer Answer:** A dynamic UObject subobject needs server ownership, correct outer/lifetime, replication registration, stable removal and GC-safe references. I test create, mutate, remove, late join and owner disconnect. With Iris, registered subobject/fragments requirements become especially branch-sensitive.
- **Common Weak Answer:** "Set the object to replicate."
- **Follow-up Questions:** registered list? GC? late join? Iris?
- **Hands-on Verification Task:** Add/remove a replicated inventory subobject and test client stale reference handling.
- **Sources:** [SRC-NET-012], [SRC-NET-011], [SRC-NET-014], [SRC-EPIC-009]
- **Version Notes:** Exact subobject APIs vary by UE version and Iris status.

### Question: What first question should you ask before using Iris details in an answer?

- **Category:** Networking / Iris
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Is Iris enabled and supported in the target project/branch for this feature?
- **Strong 3-Year-Engineer Answer:** Iris is specialist and version-sensitive. I first check project settings, branch source and platform status. If enabled, I verify registered subobject/fragments requirements, filtering and debug tools in that branch. If not enabled, I keep the answer at awareness level and use default replication proof.
- **Common Weak Answer:** "Iris replaces old replication."
- **Follow-up Questions:** opt-in? fragments? subobjects? fallback?
- **Hands-on Verification Task:** Capture target project Iris status and one branch-specific API proof.
- **Sources:** [SRC-NET-014], [SRC-NET-012], [SRC-NET-016]
- **Version Notes:** Iris is experimental/opt-in in target scope.

### Question: How do you explain NetworkPrediction without overclaiming?

- **Category:** Networking / NetworkPrediction Plugin
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** It is a plugin/API surface for fixed/rollback-style prediction models, but exact integration must be proven in target source.
- **Strong 3-Year-Engineer Answer:** I describe the model vocabulary: input/cmd, sync state, aux state, simulation tick, reconcile/rollback, cues and smoothing. Then I state that public API docs are not a copyable UE5.3-UE5.6 recipe. A target proof needs plugin enablement, compile gate, forced reconcile and side-effect safety tests.
- **Common Weak Answer:** "It gives rollback networking."
- **Follow-up Questions:** sync vs aux? cue dedupe? compile proof? branch?
- **Hands-on Verification Task:** Produce a minimal compile or rejection memo for the plugin in a target branch.
- **Sources:** [SRC-NET-020], [SRC-NET-009]
- **Version Notes:** Plugin APIs are highly branch-sensitive.

### Question: What state belongs in a rollback sync state?

- **Category:** Networking / Rollback Design
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Deterministic authoritative simulation state required to rewind and resimulate.
- **Strong 3-Year-Engineer Answer:** Sync state should hold compact values such as position-like state, velocity-like state or simulation counters that define outcome. Input commands are per-tick inputs; aux state holds slower parameters/config. Non-deterministic side effects, spawned actors and audio/VFX cues need separate dedupe or authoritative handling.
- **Common Weak Answer:** "Put all actor state in sync."
- **Follow-up Questions:** input? aux? cues? determinism?
- **Hands-on Verification Task:** Classify dash model fields into input, sync, aux and cue state.
- **Sources:** [SRC-NET-020], [SRC-NET-009]
- **Version Notes:** Exact NetworkPrediction model definitions require target source.

### Question: Why are side effects dangerous in rollback simulation?

- **Category:** Networking / Rollback
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Resimulation may execute the same tick multiple times, duplicating non-idempotent effects.
- **Strong 3-Year-Engineer Answer:** Rollback-safe simulation should update deterministic state, while irreversible effects such as sound, particles, damage events, inventory grants or analytics are gated through cues or authoritative ledgers. I add operation IDs or cue keys so replay/reconcile does not double-spawn or double-apply.
- **Common Weak Answer:** "Just do everything in the sim tick."
- **Follow-up Questions:** cue key? damage? analytics? replay?
- **Hands-on Verification Task:** Force a correction and prove a dash cue fires once.
- **Sources:** [SRC-NET-020], [SRC-ARCH-014]
- **Version Notes:** Cue APIs and services are plugin/branch sensitive.

### Question: How do you debug an owning-connection RPC failure?

- **Category:** Networking / RPCs
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Check actor ownership, net mode, role, call site, connection, relevancy and whether the RPC direction is legal.
- **Strong 3-Year-Engineer Answer:** Server RPCs require an owning client connection. I inspect owner chain, Pawn/Controller possession, actor spawn side, replicated state timing and logs. A common fix is moving the request through the PlayerController or an owned component, not making everything reliable or multicast.
- **Common Weak Answer:** "RPCs are unreliable."
- **Follow-up Questions:** PlayerController? owner? possession? dedicated server?
- **Hands-on Verification Task:** Call a Server RPC from an unowned pickup and repair the route.
- **Sources:** [SRC-NET-003], [SRC-NET-006], [SRC-NET-001]
- **Version Notes:** Ownership principles stable; debug tooling varies.

### Question: Why is reliable RPC not a state synchronisation strategy?

- **Category:** Networking / RPCs
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Reliable RPC guarantees delivery ordering for calls on a channel, not durable reconstruction for late join, relevancy or lost state context.
- **Strong 3-Year-Engineer Answer:** Persistent state belongs in replicated properties, Fast Arrays, save/server state or explicit snapshots. Reliable RPCs fit important events, but can backlog and still require a valid actor/channel and receiver context. Late joiners will not receive old RPC history unless the state is represented elsewhere.
- **Common Weak Answer:** "Make important state reliable."
- **Follow-up Questions:** late join? backlog? channel? OnRep?
- **Hands-on Verification Task:** Convert a cosmetic event plus durable inventory state into RPC plus replicated property design.
- **Sources:** [SRC-NET-004], [SRC-NET-006], [SRC-NET-015]
- **Version Notes:** RPC order/channel behaviour is engine implementation sensitive.

### Question: How do dormancy and push-style updates fail in practice?

- **Category:** Networking / Dormancy
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Mutating dormant actors or unmarked changed state can leave clients stale until explicitly woken or dirtied.
- **Strong 3-Year-Engineer Answer:** Dormancy reduces replication work, but the server must wake or flush relevant actors before changing replicated state that clients need. I test state changes while dormant, late join, relevancy return and debug CVars. For push-style/delta systems, dirty marking and validation are part of correctness.
- **Common Weak Answer:** "Dormancy just saves bandwidth."
- **Follow-up Questions:** wake before change? late join? debug CVar? Fast Array?
- **Hands-on Verification Task:** Change a dormant door state and prove the client receives or misses it.
- **Sources:** [SRC-NET-008], [SRC-NET-016], [SRC-NET-015]
- **Version Notes:** Dormancy/Iris interactions are target-version sensitive.

### Question: What is a strong answer to "server authority"?

- **Category:** Networking / Architecture
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** The server owns truth for gameplay outcomes; clients may predict, request or display but must be validated/reconciled.
- **Strong 3-Year-Engineer Answer:** I separate authority, ownership, relevancy and prediction. The server decides damage, inventory, objectives and world state. Clients can send intent and locally predict safe movement/presentation, but the server validates and replicates truth. I test cheating paths, late join and packet loss rather than only happy-path responsiveness.
- **Common Weak Answer:** "Server does everything."
- **Follow-up Questions:** ownership? autonomous proxy? prediction? cosmetic multicast?
- **Hands-on Verification Task:** Build a pickup request that rejects impossible client claims.
- **Sources:** [SRC-NET-001], [SRC-NET-003], [SRC-NET-006]
- **Version Notes:** Architecture principle stable.

### Question: How do you decide between replicated component, subobject and actor?

- **Category:** Networking / Object Model
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Choose by lifetime, transform/world identity, ownership, replication audience, RPC needs and object count.
- **Strong 3-Year-Engineer Answer:** Actors fit world identity/relevancy/transform/lifecycle. Components fit actor-owned behaviour with component lifecycle and optional RPC/property replication. UObject subobjects fit actor-owned data or polymorphic state when actor overhead is unnecessary, but require careful registration/lifetime proof. I test late join and removal for whichever path I choose.
- **Common Weak Answer:** "Use UObject because it is lighter."
- **Follow-up Questions:** transform? owner? relevancy? Iris? GC?
- **Hands-on Verification Task:** Model inventory entry as actor, component and subobject; compare overhead and correctness.
- **Sources:** [SRC-NET-011], [SRC-NET-012], [SRC-EPIC-012]
- **Version Notes:** Subobject/Iris details are branch-sensitive.

### Question: What would you measure before lowering NetUpdateFrequency?

- **Category:** Networking / Bandwidth
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Actor importance, change frequency, relevancy, bandwidth, visual smoothness and correction/latency impact.
- **Strong 3-Year-Engineer Answer:** Lower frequency can reduce traffic but worsen perceived staleness or increase correction artifacts. I first classify actor type, ownership, relevancy and update value. I measure with network profiler/stat tools under representative clients and use dormancy/conditions/RepGraph/Fast Array where more appropriate.
- **Common Weak Answer:** "Lower it for all static actors."
- **Follow-up Questions:** dormancy? relevancy? movement smoothing? owner-only?
- **Hands-on Verification Task:** Compare traffic and visual quality for three replicated actor classes.
- **Sources:** [SRC-NET-007], [SRC-NET-008], [SRC-NET-010]
- **Version Notes:** Metrics/tooling vary by branch.

### Question: How do you prove a GAS prediction bug is not just a visual bug?

- **Category:** GAS / Prediction
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Compare local prediction, server authoritative state, GameplayEffect/cooldown/cost state and correction outcome.
- **Strong 3-Year-Engineer Answer:** I log ability activation, prediction key/window, commit, cost/cooldown, target data, GE application and cue firing on client and server. A visual-only mismatch is different from server rejecting an ability or duplicating a GameplayCue. I prove under latency and packet loss.
- **Common Weak Answer:** "The animation desynced."
- **Follow-up Questions:** prediction key? commit? target data? cue duplicate?
- **Hands-on Verification Task:** Inject server rejection for a predicted dash ability and trace client correction.
- **Sources:** [SRC-GAS-001], [SRC-GAS-010], [SRC-NET-001]
- **Version Notes:** GAS prediction details are project and branch sensitive.

### Question: What should an ASC placement decision consider?

- **Category:** GAS / Architecture
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Lifetime, possession, replication audience, respawn, attribute ownership and ability grants.
- **Strong 3-Year-Engineer Answer:** ASC on PlayerState can survive pawn respawn and fit player-owned abilities; ASC on Pawn can fit creature/body-specific abilities. I decide based on owner/avatar separation, replication mode, save/respawn and grant ledger. Then I test possession changes, death/respawn and dedicated server.
- **Common Weak Answer:** "Put ASC on PlayerState for multiplayer."
- **Follow-up Questions:** NPC? avatar? replication mode? respawn?
- **Hands-on Verification Task:** Move ASC placement and document which state survives respawn.
- **Sources:** [SRC-GAS-001], [SRC-GAS-002], [SRC-EPIC-014]
- **Version Notes:** Common patterns are project-dependent.

### Question: Why is an ability grant ledger useful?

- **Category:** GAS / Grants
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It records why abilities/effects were granted so cleanup, reload, respawn and debugging are deterministic.
- **Strong 3-Year-Engineer Answer:** Ability specs and handles are not enough context by themselves. A ledger ties grants to item, class, quest, loadout or temporary buff source. It prevents orphaned abilities after equipment removal, respawn or save/load and makes bug reports explainable.
- **Common Weak Answer:** "Remove the ability handle."
- **Follow-up Questions:** duplicate grant? source item? save/load? cleanup?
- **Hands-on Verification Task:** Equip/unequip an item repeatedly and prove no orphan ability remains.
- **Sources:** [SRC-GAS-003], [SRC-GAS-013], [SRC-ARCH-014]
- **Version Notes:** Grant policies are project-specific.

### Question: How do Attribute base and current values shape bugs?

- **Category:** GAS / Attributes
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Confusing permanent base changes with temporary/current modifiers causes stacking, restore and UI bugs.
- **Strong 3-Year-Engineer Answer:** I treat base value as durable baseline and current value as the result after active effects/modifiers where the project pattern supports that distinction. Damage, healing, max-health changes and buffs need explicit invariants and clamps. UI should observe the authoritative attribute flow rather than caching stale derived values.
- **Common Weak Answer:** "Health is just a float."
- **Follow-up Questions:** max health change? clamp? stacking? UI?
- **Hands-on Verification Task:** Apply max-health buff, damage, expiry and verify final health invariant.
- **Sources:** [SRC-GAS-004], [SRC-GAS-005], [SRC-GAS-006]
- **Version Notes:** Exact attribute callbacks are branch-sensitive.

### Question: Why can GameplayEffect execution calculations become hard to debug?

- **Category:** GAS / Gameplay Effects
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Captures, snapshots, tags, stacking and source/target context interact across time.
- **Strong 3-Year-Engineer Answer:** I log source/target ASC, captured attributes, snapshot policy, granted/blocked tags, set-by-caller values and stacking rules. A wrong damage result may come from capture timing, missing tag, wrong source object or stale spec. I create deterministic tests for each calculation path.
- **Common Weak Answer:** "The formula is wrong."
- **Follow-up Questions:** snapshot? captured attr? tags? spec context?
- **Hands-on Verification Task:** Build two damage calculations with snapshot and live capture and compare.
- **Sources:** [SRC-GAS-005], [SRC-GAS-006], [SRC-GAS-007]
- **Version Notes:** GAS internals require branch/source confirmation.

### Question: What is a safe GameplayCue policy?

- **Category:** GAS / Gameplay Cues
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Cues present gameplay state/events without becoming authority or duplicating under prediction/correction.
- **Strong 3-Year-Engineer Answer:** GameplayCues should be keyed, audience-aware and resilient to prediction replay. I separate cue visuals/audio from authoritative effect application, handle removal/end events and test under latency. Duplicate cue spam often points to missing dedupe or incorrect prediction handling.
- **Common Weak Answer:** "Cues are just particles."
- **Follow-up Questions:** prediction? remove? authority? cosmetic audience?
- **Hands-on Verification Task:** Force a predicted ability correction and prove the cue does not duplicate.
- **Sources:** [SRC-GAS-008], [SRC-GAS-010], [SRC-NET-020]
- **Version Notes:** Cue systems are project/plugin sensitive.

### Question: How do you test AbilityTask cleanup?

- **Category:** GAS / Ability Tasks
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Cancel/end abilities across success, failure, owner death, travel and prediction rejection while checking delegates/timers/listeners are removed.
- **Strong 3-Year-Engineer Answer:** AbilityTasks often bind delegates, wait on gameplay events or manage timers. I test every exit path and assert no duplicate callbacks after reactivation. Cleanup must be robust on server and client because leaked tasks create ghost activation, cue duplication or crashes.
- **Common Weak Answer:** "EndAbility cleans it."
- **Follow-up Questions:** cancellation? owner destroyed? prediction reject? delegates?
- **Hands-on Verification Task:** Build a wait-target task and cancel it from five paths.
- **Sources:** [SRC-GAS-009], [SRC-GAS-013]
- **Version Notes:** AbilityTask APIs can vary by branch.

### Question: When should you not adopt GAS?

- **Category:** GAS / Architecture
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** When the game's ability/state needs are small enough that GAS complexity, replication and authoring overhead exceed its benefits.
- **Strong 3-Year-Engineer Answer:** GAS is powerful for networked abilities, tags, effects, attributes and prediction, but it imposes mental model and tooling cost. For a simple single-player interaction system, a custom component may be clearer. I decide from feature roadmap, team familiarity, multiplayer needs and designer authoring requirements.
- **Common Weak Answer:** "Always use GAS for abilities."
- **Follow-up Questions:** prediction? designer workflow? complexity? migration?
- **Hands-on Verification Task:** Write a decision memo comparing GAS and a bespoke ability component.
- **Sources:** [SRC-GAS-001], [SRC-ARCH-014]
- **Version Notes:** GAS value is highly project-dependent.

### Question: How do Smart Objects prevent AI hardcoding?

- **Category:** AI / Smart Objects
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** They move affordance data and claimable interaction slots into world-authored objects rather than bespoke AI code.
- **Strong 3-Year-Engineer Answer:** Smart Objects let AI query, claim, use and release authored affordances. The AI does not need a hardcoded list of "chairs" or "cover"; it asks for suitable objects by tags/filters/context. Correctness depends on claim lifecycle, failure handling, nav/animation integration and server authority if multiplayer.
- **Common Weak Answer:** "Smart Objects are interactable actors."
- **Follow-up Questions:** claim? slot? tags? release?
- **Hands-on Verification Task:** Build two AI agents racing for one Smart Object slot and prove only one claims it.
- **Sources:** [SRC-AI-011], [SRC-AI-012], [SRC-AI-015]
- **Version Notes:** Smart Objects are plugin/project sensitive.

### Question: StateTree or Behaviour Tree?

- **Category:** AI / Decision Systems
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Choose based on decision shape, data flow, authoring needs, team expertise and debugging support rather than trend.
- **Strong 3-Year-Engineer Answer:** Behaviour Trees are strong for hierarchical AI decisions with Blackboard and event-driven decorators; StateTree combines hierarchical states, selectors, tasks and data flow and can fit smart-object/activity style logic. I compare transition semantics, reuse, debugging, designer workflow and performance before choosing.
- **Common Weak Answer:** "StateTree is the new Behaviour Tree."
- **Follow-up Questions:** transitions? observers? task lifetime? debugging?
- **Hands-on Verification Task:** Implement patrol/interrupt/use-object in both and compare clarity.
- **Sources:** [SRC-AI-002], [SRC-AI-010], [SRC-AI-008]
- **Version Notes:** StateTree feature/API status is branch/plugin sensitive.

### Question: How do you debug a custom AI sense that remembers forever?

- **Category:** AI / Perception
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Inspect stimulus age, success state, forgetting policy, listener updates and Blackboard projection.
- **Strong 3-Year-Engineer Answer:** A sense should report evidence, not permanent truth. I check FAIStimulus age/expiration, source registration, listener configuration and whether AI code projects stale perception into Blackboard without clearing it. Visual Logger and AI debugger help separate sense data from decision memory.
- **Common Weak Answer:** "Clear the Blackboard key sometimes."
- **Follow-up Questions:** stimulus age? listener? source? decision memory?
- **Hands-on Verification Task:** Make a target leave range and prove perception and Blackboard clear independently.
- **Sources:** [SRC-AI-004], [SRC-AI-016], [SRC-AI-018], [SRC-AI-009]
- **Version Notes:** Custom sense APIs are source-sensitive.

### Question: Why should custom senses avoid gameplay decisions inside the sense?

- **Category:** AI / Architecture
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Senses should provide evidence; controllers/BT/StateTree decide behaviour.
- **Strong 3-Year-Engineer Answer:** Putting "attack now" logic in a sense couples perception sampling, AI policy, difficulty and gameplay side effects. I make the sense produce stimuli with strength/location/tag, then let decision systems reason over memory, Blackboard, EQS or StateTree. This keeps debugging and scaling manageable.
- **Common Weak Answer:** "Sense knows what the AI should do."
- **Follow-up Questions:** evidence vs truth? difficulty? testing? scaling?
- **Hands-on Verification Task:** Move a "flee if smell" decision from sense code into StateTree/BT.
- **Sources:** [SRC-AI-016], [SRC-AI-019], [SRC-AI-002], [SRC-AI-010]
- **Version Notes:** API signatures are branch-sensitive.

### Question: What does MassEntity buy over actors?

- **Category:** MassEntity / ECS
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Data-oriented batches over fragments/archetypes for large populations, with explicit trade-offs in tooling, representation and gameplay integration.
- **Strong 3-Year-Engineer Answer:** Mass can improve scale by processing homogeneous data in chunks instead of thousands of ticking actors. It is not a drop-in Actor replacement: you need representation, promotion/demotion, debugging, replication awareness and authoring workflow. I adopt it when measured actor scale fails and Mass fits the simulation shape.
- **Common Weak Answer:** "Mass is faster ECS."
- **Follow-up Questions:** fragments? processors? representation? hybrid actor?
- **Hands-on Verification Task:** Compare 1000 Actor agents with Mass agents under equal movement/representation requirements.
- **Sources:** [SRC-MASS-001], [SRC-MASS-002], [SRC-MASS-006]
- **Version Notes:** Mass APIs are branch/plugin sensitive.

### Question: Why is structural mutation expensive in ECS/Mass systems?

- **Category:** MassEntity / Performance
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Adding/removing fragments or tags can move entities between archetypes/chunks and invalidate assumptions.
- **Strong 3-Year-Engineer Answer:** ECS performance depends on stable archetypes and cache-friendly chunk iteration. Frequent structural changes can create command-buffer work, chunk moves and synchronisation points. I prefer tags/fragments that change at controlled phase boundaries and profile mutation bursts separately from steady-state processing.
- **Common Weak Answer:** "Tags are free."
- **Follow-up Questions:** archetype move? command buffer? observer? phase?
- **Hands-on Verification Task:** Toggle a fragment on thousands of entities every frame and measure versus state field.
- **Sources:** [SRC-MASS-002], [SRC-MASS-004], [SRC-MASS-006]
- **Version Notes:** Exact processor/command APIs are version-sensitive.

### Question: How should Mass actors be promoted and demoted?

- **Category:** MassEntity / Hybrid Architecture
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Promote only entities needing rich Actor behaviour and keep one authoritative state writer during transitions.
- **Strong 3-Year-Engineer Answer:** A common hybrid pattern uses Mass for far/large-scale simulation and Actors for nearby, interactive or cinematic entities. Promotion/demotion needs identity mapping, state copy, ownership, replication policy and rollback for failed creation. Dual writers are the main bug.
- **Common Weak Answer:** "Spawn actors near the player."
- **Follow-up Questions:** identity? state handoff? replication? dual writer?
- **Hands-on Verification Task:** Promote one crowd entity to Actor near the player and demote it without state divergence.
- **Sources:** [SRC-MASS-007], [SRC-MASS-008], [SRC-MASS-010]
- **Version Notes:** Representation and crowd plugins are branch-sensitive.

### Question: What is ZoneGraph's role in crowd AI?

- **Category:** MassAI / ZoneGraph
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** It provides lane/zone-style spatial data for movement/crowd flow rather than per-agent freeform pathfinding.
- **Strong 3-Year-Engineer Answer:** ZoneGraph can support structured pedestrian/traffic-like flow where agents follow authored lanes/zones. It complements Mass crowd and StateTree-style activities. I still need avoidance, representation, transitions to NavMesh/Actors and debugging for blocked or disconnected lanes.
- **Common Weak Answer:** "It replaces NavMesh."
- **Follow-up Questions:** lanes? avoidance? Mass? transitions?
- **Hands-on Verification Task:** Build a simple two-lane route and debug a disconnected lane.
- **Sources:** [SRC-MASS-009], [SRC-MASS-010], [SRC-AI-006]
- **Version Notes:** ZoneGraph/MassCrowd are plugin/version sensitive.

### Question: How do you profile a PCG graph versus its generated output?

- **Category:** PCG / Performance
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Separate graph execution cost from spawned Actors/components/instances, collision, nav, memory and rendering cost.
- **Strong 3-Year-Engineer Answer:** PCG graph time is only one part. Output may create thousands of components, collision bodies, nav modifiers, materials or draw calls. I record input point count, filtered count, output count, generated representation and packaged/runtime policy before optimising graph nodes.
- **Common Weak Answer:** "The PCG graph is slow."
- **Follow-up Questions:** output count? Actors vs ISM? collision? runtime generation?
- **Hands-on Verification Task:** Generate the same points as Actors and instances and compare cost.
- **Sources:** [SRC-PCG-001], [SRC-PCG-003], [SRC-PERF-003]
- **Version Notes:** PCG runtime behaviour is plugin/branch sensitive.

### Question: What should PCG output ownership record?

- **Category:** PCG / Pipeline
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Seed, graph version, input identity, owner, cleanup policy, Data Layer/HLOD policy and runtime/cook behaviour.
- **Strong 3-Year-Engineer Answer:** Generated output needs provenance or the team cannot safely regenerate, diff, cook or debug it. I store enough metadata to know why output exists and whether stale duplicates should be removed. In World Partition, I also verify Data Layer and HLOD assignment.
- **Common Weak Answer:** "Regenerate if it looks wrong."
- **Follow-up Questions:** seed? cleanup? package? HLOD?
- **Hands-on Verification Task:** Change a PCG seed and prove old output is removed or versioned.
- **Sources:** [SRC-PCG-001], [SRC-PCG-004], [SRC-WORLD-003]
- **Version Notes:** Generated output policy is project-specific.

### Question: Why are metadata domains a PCG debugging issue?

- **Category:** PCG / Debugging
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Attribute values may live on different data/point/element domains, so filters or mutations can read the wrong assumption.
- **Strong 3-Year-Engineer Answer:** A PCG graph can appear logically correct while a filter sees missing/default values because the attribute is not on the domain the node expects. I inspect debug points and metadata at intermediate nodes, then assert assumptions with sanity checkpoints.
- **Common Weak Answer:** "The filter node is broken."
- **Follow-up Questions:** point attribute? data attribute? debug node? defaults?
- **Hands-on Verification Task:** Put an attribute on the wrong domain and diagnose a dead branch.
- **Sources:** [SRC-PCG-001], [SRC-PCG-003], [SRC-PCG-007]
- **Version Notes:** Node behaviour evolves across plugin versions.

### Question: How do you choose between Level Instance and Packed Level Blueprint?

- **Category:** World Building / Reusable Content
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Choose by editability, reuse, static density, runtime behaviour, OFPA support and gameplay identity.
- **Strong 3-Year-Engineer Answer:** Level Instances help reuse authored level chunks; Packed Level Blueprints can suit dense static visual clusters. Mutable gameplay actors, unique save identity or high-density runtime streaming need careful treatment. I prove packaged behaviour and source-control workflow rather than assuming editor composition is enough.
- **Common Weak Answer:** "Use Packed Level Blueprint for reusable places."
- **Follow-up Questions:** static? mutable? OFPA? runtime mode?
- **Hands-on Verification Task:** Build a POI both ways and compare packaged edit/runtime constraints.
- **Sources:** [SRC-WORLD-004], [SRC-ASSET-010]
- **Version Notes:** Level Instance modes are target-version sensitive.

### Question: How do you prove a shader plugin works in a package?

- **Category:** Rendering / Shaders
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Compile/cook/package the shader path, render a visible target output and capture failure logs for missing mapping/library issues.
- **Strong 3-Year-Engineer Answer:** Editor success can hide shader directory mapping, load phase, platform permutation or cook issues. I verify plugin module load phase, shader registration, supported platforms, cooked shader library and a packaged smoke view that would be obviously wrong if the shader failed.
- **Common Weak Answer:** "It compiles in editor."
- **Follow-up Questions:** load phase? cook? platform? black output?
- **Hands-on Verification Task:** Package a material/global shader path and intentionally break directory mapping.
- **Sources:** [SRC-RENDER-012], [SRC-RENDER-013], [SRC-BUILD-017]
- **Version Notes:** Shader plugin APIs are branch/RHI sensitive.

### Question: How do you diagnose a first-use PSO hitch?

- **Category:** Rendering / PSO
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Correlate hitch timing with first material/effect/view use, PSO logging and packaged cold-cache runs.
- **Strong 3-Year-Engineer Answer:** I separate shader compilation, shader library availability, PSO creation, texture streaming and render-state setup. Then I collect representative PSOs from packaged scenarios or reduce state diversity. Warm developer caches are not proof that shipped users avoid first-use hitches.
- **Common Weak Answer:** "Precompile shaders."
- **Follow-up Questions:** shader library? PSO cache? cold cache? scene capture?
- **Hands-on Verification Task:** Trigger a first-use effect hitch and verify PSO/cache coverage.
- **Sources:** [SRC-RENDER-014], [SRC-PERF-003], [SRC-BUILD-017]
- **Version Notes:** PSO workflows vary by RHI/platform.

### Question: Why can scene captures dominate cost?

- **Category:** Rendering / Scene Capture
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** They can render additional views with their own visibility, passes, render targets and update cadence.
- **Strong 3-Year-Engineer Answer:** I treat a SceneCapture2D as another camera until evidence says otherwise; cube captures can be much worse. I reduce resolution/format/update rate/show flags, avoid per-frame captures unless necessary, and profile GPU, Draw/RHI and render-target memory separately.
- **Common Weak Answer:** "It is just a texture preview."
- **Follow-up Questions:** capture every frame? cube? show flags? UI retainer?
- **Hands-on Verification Task:** Compare 1, 10 and 50 capture components with manual update.
- **Sources:** [SRC-RENDER-016], [SRC-RENDER-017], [SRC-RENDER-018]
- **Version Notes:** Renderer path/platform support changes cost.

### Question: What does RDG help you reason about?

- **Category:** Rendering / RDG
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Render pass/resource lifetime, dependencies, aliasing and validation in the render graph.
- **Strong 3-Year-Engineer Answer:** RDG makes render work explicit as passes and resources with declared dependencies. It helps catch lifetime and ordering bugs that ad hoc render code can hide. Specialist answers should still be source-checked because exact APIs and validation tools change.
- **Common Weak Answer:** "RDG is a faster way to render."
- **Follow-up Questions:** resource lifetime? pass dependency? extraction? validation?
- **Hands-on Verification Task:** Write a small pass that intentionally uses a resource outside lifetime and fix it.
- **Sources:** [SRC-RENDER-019], [SRC-RENDER-012]
- **Version Notes:** RDG APIs are source/RHI sensitive.

### Question: How do you choose Nanite, LOD, ISM/HISM or HLOD?

- **Category:** Rendering / Visibility
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Match the technique to geometry, repetition, distance grouping, material/deformation needs, platform support and measured bottleneck.
- **Strong 3-Year-Engineer Answer:** Nanite helps supported complex geometry, LOD simplifies individual assets, instancing helps repeated compatible meshes, and HLOD replaces distant spatial groups. I measure CPU submission, GPU passes, shadows, memory and streaming rather than choosing by name.
- **Common Weak Answer:** "Use Nanite for everything."
- **Follow-up Questions:** material? WPO? repeated mesh? platform? shadows?
- **Hands-on Verification Task:** Render one population four ways and compare Draw/GPU/memory.
- **Sources:** [SRC-RENDER-008], [SRC-RENDER-010], [SRC-RENDER-011]
- **Version Notes:** Feature support varies by platform and UE version.

### Question: Why can material slots be more expensive than triangle count?

- **Category:** Rendering / Materials
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Slots can split mesh sections and create more pass/material/draw work across multiple rendering passes.
- **Strong 3-Year-Engineer Answer:** Triangle count is only one axis. Many slots, unique material instances and pass participation can increase CPU draw setup and GPU state/pixel cost. I inspect section counts, material complexity, shadow/depth/base participation and render-thread/RHI metrics.
- **Common Weak Answer:** "Optimise polygons first."
- **Follow-up Questions:** sections? shadow pass? material instance? overdraw?
- **Hands-on Verification Task:** Compare one mesh with 1 versus 12 material slots at equal triangle count.
- **Sources:** [SRC-RENDER-004], [SRC-RENDER-015], [SRC-PERF-003]
- **Version Notes:** Renderer internals are branch/RHI sensitive.

### Question: How do you decide whether a VFX regression is CPU or GPU?

- **Category:** VFX / Profiling
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Separate system simulation, spawning, component count, renderer submission, material overdraw and GPU pass time.
- **Strong 3-Year-Engineer Answer:** Niagara can cost CPU in spawn/update, memory from buffers, render-thread work from components/renderers and GPU from materials/overdraw/particles. I profile with stats/Insights/GPU capture and scale emitter count, particle count and material complexity independently.
- **Common Weak Answer:** "Particles are GPU-heavy."
- **Follow-up Questions:** CPU sim? GPU sim? overdraw? pooling?
- **Hands-on Verification Task:** Build a CPU-sim and GPU-sim effect and compare under 100 emitters.
- **Sources:** [SRC-FX-001], [SRC-FX-006], [SRC-PERF-003]
- **Version Notes:** Niagara feature support is platform/branch sensitive.

### Question: Why can pooling Niagara components be wrong?

- **Category:** VFX / Lifetime
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Reused components may retain parameters, bindings, transforms or lifecycle assumptions unless reset explicitly.
- **Strong 3-Year-Engineer Answer:** Pooling can reduce spawn/destruction churn, but pooled VFX must reset user parameters, attachment, owner, scalability state, completion callbacks and audio/gameplay coupling. I test cancellation, replay and owner destruction, not only the happy path.
- **Common Weak Answer:** "Pool all effects for performance."
- **Follow-up Questions:** reset? owner destroyed? scalability? completion?
- **Hands-on Verification Task:** Pool a status effect and prove no stale colour/target remains.
- **Sources:** [SRC-FX-001], [SRC-FX-007], [SRC-ARCH-014]
- **Version Notes:** Pooling APIs/policies vary by project.

### Question: How do you debug audio that plays twice in multiplayer?

- **Category:** Audio / Multiplayer
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Determine whether audio is predicted locally, replicated as an event, attached to state, or triggered by both client and server.
- **Strong 3-Year-Engineer Answer:** I separate local feedback from authoritative replicated events. Some sounds should be local-only, some multicast/cue-based, and persistent loops should follow replicated state with dedupe. Logs should include instigator, net mode, owner and event ID so double fire is visible.
- **Common Weak Answer:** "Only play it on server."
- **Follow-up Questions:** local feedback? multicast? persistent loop? event ID?
- **Hands-on Verification Task:** Trigger a weapon sound under prediction and remove duplicate server echo.
- **Sources:** [SRC-AUDIO-001], [SRC-AUDIO-004], [SRC-NET-006]
- **Version Notes:** Audio replication policy is project-specific.

### Question: What makes a MetaSound performance issue different from a SoundCue issue?

- **Category:** Audio / MetaSounds
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** MetaSounds are procedural graphs with graph execution/parameter complexity, not only asset playback routing.
- **Strong 3-Year-Engineer Answer:** I inspect graph complexity, update rate, parameter driving, voice count, concurrency, submix routing and platform CPU. A MetaSound can be powerful, but procedural authoring should still follow budgets and profiler evidence. I avoid assuming it is cheaper or more expensive without measuring.
- **Common Weak Answer:** "MetaSounds are always better."
- **Follow-up Questions:** graph cost? voices? parameters? platform?
- **Hands-on Verification Task:** Compare a looping SoundCue and MetaSound variant under 100 voices.
- **Sources:** [SRC-AUDIO-002], [SRC-AUDIO-009], [SRC-PERF-003]
- **Version Notes:** MetaSounds are version/platform sensitive.

### Question: How do you decide between animation montage and state machine logic?

- **Category:** Animation / Gameplay
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Use state machines for continuous locomotion/state selection and montages for discrete authored actions layered through slots/sections.
- **Strong 3-Year-Engineer Answer:** Montages are strong for attacks, reloads and reactions with sections/notifies/slots; locomotion state machines/blend spaces are better for continuous pose selection. I define authority and cancellation policy, especially for multiplayer notifies and root motion.
- **Common Weak Answer:** "Use montages for actions."
- **Follow-up Questions:** slots? sync? notify authority? root motion?
- **Hands-on Verification Task:** Implement a melee attack montage layered over locomotion and test cancellation.
- **Sources:** [SRC-ANIM-003], [SRC-ANIM-006], [SRC-ANIM-009]
- **Version Notes:** Animation APIs and tooling vary by branch.

### Question: Why are animation notifies not reliable gameplay authority by themselves?

- **Category:** Animation / Networking
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** They are tied to animation playback/timing and can differ by client, LOD, montage state, prediction and cancellation.
- **Strong 3-Year-Engineer Answer:** Notifies are useful timing signals, but authoritative damage or inventory changes should be server-validated. I design notify windows with server checks, montage state verification and fallback for interrupted animations. Cosmetic notifies can remain client-side.
- **Common Weak Answer:** "Put damage in the notify."
- **Follow-up Questions:** montage cancel? server? LOD? prediction?
- **Hands-on Verification Task:** Trigger damage from a notify window and reject it when montage state is invalid.
- **Sources:** [SRC-ANIM-006], [SRC-NET-001], [SRC-GAS-009]
- **Version Notes:** Notify behaviour depends on animation setup.

### Question: How do you diagnose root-motion foot sliding in multiplayer?

- **Category:** Animation / Movement
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Compare animation root motion, CharacterMovement authority, network smoothing, montage state and correction.
- **Strong 3-Year-Engineer Answer:** I check whether movement is in-place or root-motion-driven, who owns authority, whether root motion is replicated/consumed correctly, and how smoothing/correction affects simulated proxies. I use a dedicated-server latency test, not only local PIE.
- **Common Weak Answer:** "Fix the animation."
- **Follow-up Questions:** server authority? montage? smoothing? simulated proxy?
- **Hands-on Verification Task:** Compare in-place and root-motion dodge under 100 ms latency.
- **Sources:** [SRC-ANIM-010], [SRC-NET-009], [SRC-ANIM-015]
- **Version Notes:** Root-motion networking is setup/branch sensitive.

### Question: How do you classify a collision bug before fixing it?

- **Category:** Physics / Collision
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Determine query versus physics simulation, channel/object response, simple/complex geometry and event notification path.
- **Strong 3-Year-Engineer Answer:** I first ask whether this is a trace/sweep/overlap query, rigid-body contact, CharacterMovement movement, or hit/overlap event bug. Then I inspect collision enabled, object type, responses, profile, simple/complex geometry, movement flags and notification settings.
- **Common Weak Answer:** "Collision is set wrong."
- **Follow-up Questions:** query vs physics? simple/complex? hit vs overlap? channel?
- **Hands-on Verification Task:** Build a response matrix for projectile, pawn, trigger and physics prop.
- **Sources:** [SRC-PHYS-001], [SRC-PHYS-002], [SRC-PHYS-003]
- **Version Notes:** Chaos details and debug tools vary.

### Question: Why can sweep results be misunderstood?

- **Category:** Physics / Queries
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Initial overlap, blocking hit, touch hits, normal direction, impact point and trace complexity have specific meanings.
- **Strong 3-Year-Engineer Answer:** I read the FHitResult fields carefully: blocking hit versus overlap/touch, start penetrating, time, impact point, location, normals and actor/component. Multi traces may include non-blocking hits before a block. Misreading these creates teleport, projectile and ledge bugs.
- **Common Weak Answer:** "Use Hit Actor and Impact Point."
- **Follow-up Questions:** start penetrating? trace complex? normal? multi?
- **Hands-on Verification Task:** Log and visualise a sphere sweep starting inside geometry.
- **Sources:** [SRC-PHYS-003], [SRC-MATH-011]
- **Version Notes:** Query APIs are stable, details branch-sensitive.

### Question: When should you use physics substepping?

- **Category:** Physics / Simulation
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** When simulation stability needs smaller physics timesteps, after confirming cost and determinism/authority implications.
- **Strong 3-Year-Engineer Answer:** Substepping can improve fast-moving or constrained simulation stability, but it increases CPU cost and does not fix bad collision setup or unrealistic forces. I profile physics time, test network authority and tune damping/mass/constraints first.
- **Common Weak Answer:** "Turn on substepping for better physics."
- **Follow-up Questions:** cost? CCD? constraints? network?
- **Hands-on Verification Task:** Compare a high-speed projectile/constraint with and without substepping.
- **Sources:** [SRC-PHYS-005], [SRC-PHYS-006], [SRC-PERF-003]
- **Version Notes:** Chaos settings differ by branch/platform.

### Question: How do you design UI model projection for multiplayer?

- **Category:** UI / Architecture
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Project replicated/local model state into view models or widgets with clear audience and lifetime boundaries.
- **Strong 3-Year-Engineer Answer:** UI should not own gameplay truth. I define whether data is server-public, owner-only, local-only or cosmetic, then project it into widgets via model/view-model/event bindings. I test pawn death, possession change, split-screen/local player and stale async assets.
- **Common Weak Answer:** "Bind widgets directly to actors."
- **Follow-up Questions:** owner-only? PlayerState? async icon? possession?
- **Hands-on Verification Task:** Build an inventory UI that survives pawn respawn without stale actor pointers.
- **Sources:** [SRC-UI-001], [SRC-UI-011], [SRC-NET-004]
- **Version Notes:** MVVM/CommonUI are plugin/version sensitive.

### Question: Why can UMG bindings hurt performance?

- **Category:** UI / Performance
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Frequent polling/binding evaluation can cause repeated work and hidden object traversal every frame.
- **Strong 3-Year-Engineer Answer:** I prefer event-driven or batched updates for stable UI state, while recognising very high-frequency changes may need explicit efficient update paths. I profile Slate/UMG invalidation, list recycling and widget counts rather than guessing.
- **Common Weak Answer:** "Never use bindings."
- **Follow-up Questions:** invalidation? list view? event frequency? retainer?
- **Hands-on Verification Task:** Replace polling health bindings with event updates and profile 100 widgets.
- **Sources:** [SRC-UI-009], [SRC-PERF-005], [SRC-PERF-003]
- **Version Notes:** UMG optimisation guidance changes across versions.

### Question: How do ListView recycled entries fail?

- **Category:** UI / ListView
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Recycled widgets can keep stale delegates, async icons, selection state or model references if not rebound/reset.
- **Strong 3-Year-Engineer Answer:** Entry widgets are reused, so `OnListItemObjectSet` must reset every visual and binding, cancel or validate async loads, and unbind old delegates. I stress with scroll, filter, sort and destroyed data items.
- **Common Weak Answer:** "Create a widget per item."
- **Follow-up Questions:** async image? delegate? selection? sort?
- **Hands-on Verification Task:** Create a 1000-row inventory and prove no stale icon after rapid scrolling.
- **Sources:** [SRC-UI-005], [SRC-UI-009], [SRC-ASSET-004]
- **Version Notes:** UMG/ListView APIs vary by branch.

### Question: How do you handle UI focus across gamepad, mouse and CommonUI?

- **Category:** UI / Input
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Define input mode, focus target, navigation rules, activation stack and game/UI ownership transitions.
- **Strong 3-Year-Engineer Answer:** I test focus entry/exit, modal layers, gamepad navigation, pointer interaction, pause/menu transitions and local-player scope. CommonUI can help with activation, but the project still needs clear ownership and fallback for lost focus.
- **Common Weak Answer:** "Set keyboard focus."
- **Follow-up Questions:** local player? modal? back action? Enhanced Input?
- **Hands-on Verification Task:** Build a modal inventory and verify gamepad back/focus restoration.
- **Sources:** [SRC-UI-010], [SRC-UI-012], [SRC-INPUT-001]
- **Version Notes:** CommonUI/Enhanced Input are plugin/branch sensitive.

### Question: What belongs in a save-game schema version?

- **Category:** Gameplay Architecture / Save Systems
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Stable IDs, version number, migration rules and separation between durable truth and transient runtime pointers.
- **Strong 3-Year-Engineer Answer:** Saves should not serialise arbitrary actor pointers from streamed cells. I store stable content IDs, state values, schema version and migration code. For World Partition, I reconstruct presentation from durable state and handle missing/renamed content gracefully.
- **Common Weak Answer:** "Mark fields SaveGame."
- **Follow-up Questions:** migration? stable ID? streamed actor? missing asset?
- **Hands-on Verification Task:** Rename/remove a saved pickup definition and migrate the save.
- **Sources:** [SRC-ARCH-010], [SRC-EPIC-007], [SRC-ASSET-010]
- **Version Notes:** Save format is project-specific.

### Question: How do you design an inventory item identity model?

- **Category:** Gameplay Architecture / Inventory
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Separate item definition, instance ID, stack/count, runtime state, ownership and replication/persistence policy.
- **Strong 3-Year-Engineer Answer:** A definition asset describes item type; an instance or stack carries player-owned mutable state. I avoid using actor pointers as identity and define how item state replicates, saves, stacks, trades and survives asset renames. Fast Array may fit replicated collections.
- **Common Weak Answer:** "Use Data Assets for items."
- **Follow-up Questions:** instance state? stack? Fast Array? save ID?
- **Hands-on Verification Task:** Model stackable potions and unique weapons through definitions plus instances.
- **Sources:** [SRC-ARCH-008], [SRC-ASSET-005], [SRC-NET-015]
- **Version Notes:** Inventory architecture is project-dependent.

### Question: What is a transactional gameplay operation?

- **Category:** Gameplay Architecture / Transactions
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** A multi-step gameplay change with explicit validate/apply/commit/rollback or compensation semantics.
- **Strong 3-Year-Engineer Answer:** Operations like trade, craft, equip or quest reward often touch inventory, UI, save, analytics and abilities. I give them operation IDs, precondition checks, authoritative commit order and idempotent retries where needed. This prevents duplicate rewards and half-applied state after disconnect or crash.
- **Common Weak Answer:** "Call all the functions in order."
- **Follow-up Questions:** idempotency? rollback? server authority? save?
- **Hands-on Verification Task:** Implement craft operation with failure after inventory removal and repair it.
- **Sources:** [SRC-ARCH-014], [SRC-NET-001], [SRC-GAS-013]
- **Version Notes:** Pattern is engine-agnostic; UE implementation varies.

### Question: How do gameplay tags help without becoming string soup?

- **Category:** Gameplay Architecture / Tags
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** They provide hierarchical semantic labels, but require taxonomy ownership, validation and clear query semantics.
- **Strong 3-Year-Engineer Answer:** Gameplay tags are great for state, requirements, filtering and cross-system communication. They become unmaintainable if every feature invents overlapping tags. I define naming rules, authority, documentation, validation and avoid using tags as a hidden replacement for structured data.
- **Common Weak Answer:** "Use tags instead of enums."
- **Follow-up Questions:** hierarchy? validation? tag queries? ownership?
- **Hands-on Verification Task:** Design a tag taxonomy for stun, silence, elemental damage and item rarity.
- **Sources:** [SRC-ARCH-013], [SRC-GAS-011], [SRC-BUILD-009]
- **Version Notes:** Gameplay tag setup is project-specific.

### Question: How should Blueprint pure functions be treated?

- **Category:** Blueprint / API Design
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Pure functions should be cheap, side-effect-free queries; expensive or mutating work should be explicit.
- **Strong 3-Year-Engineer Answer:** Blueprint pure nodes can be reevaluated in ways authors do not expect. I avoid hidden loads, allocations, random state, network calls or mutation in pure functions. Expensive queries should be cached, event-driven or explicit so graphs show cost and side effects.
- **Common Weak Answer:** "Pure means no exec pin."
- **Follow-up Questions:** hidden load? random? allocation? binding?
- **Hands-on Verification Task:** Replace an expensive pure inventory query in UI with cached/event update.
- **Sources:** [SRC-EPIC-031], [SRC-PERF-005], [SRC-ASSET-004]
- **Version Notes:** Blueprint VM/editor behaviour can vary.

### Question: How do you expose C++ safely to Blueprint?

- **Category:** C++ / Blueprint API
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Expose a narrow, intention-revealing surface with correct specifiers, validation, lifetime and authority rules.
- **Strong 3-Year-Engineer Answer:** I decide editability, Blueprint access, categories, authority checks and object lifetime deliberately. Blueprint-callable APIs should validate inputs, avoid raw dangerous ownership, communicate latent/async behaviour and preserve invariants. Broad `BlueprintReadWrite` on internal state is a maintenance hazard.
- **Common Weak Answer:** "Add BlueprintCallable."
- **Follow-up Questions:** EditDefaultsOnly? authority? latent? UObject lifetime?
- **Hands-on Verification Task:** Refactor a public mutable health field into safe Blueprint functions/events.
- **Sources:** [SRC-EPIC-007], [SRC-EPIC-030], [SRC-EPIC-031]
- **Version Notes:** Specifiers are UE-version sensitive.

### Question: How do you choose between UObject, struct and plain C++ type?

- **Category:** UE C++ / Type Design
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Choose based on reflection, identity/lifetime, value semantics, editor/Blueprint/serialisation needs and performance.
- **Strong 3-Year-Engineer Answer:** Plain C++ fits implementation detail and RAII. USTRUCT fits reflected value data. UObject fits identity, reflection, polymorphism, GC and engine integration. I avoid making everything UObject because lifetime, allocation and GC/debug complexity matter.
- **Common Weak Answer:** "Use UObject if Blueprint needs it."
- **Follow-up Questions:** value copy? GC? editor? replication?
- **Hands-on Verification Task:** Implement the same config as struct, UObject and plain C++ helper and compare trade-offs.
- **Sources:** [SRC-EPIC-003], [SRC-EPIC-008], [SRC-CPP-001]
- **Version Notes:** Core concepts stable UE4/UE5.

### Question: When is TStrongObjectPtr appropriate?

- **Category:** UE C++ / UObject Pointers
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** For explicit strong ownership of a UObject from non-reflected/native contexts where GC must see the reference.
- **Strong 3-Year-Engineer Answer:** Most UObject fields should be reflected `TObjectPtr`/UPROPERTY. `TStrongObjectPtr` is a special native owner for cases like temporary tools or non-UObject managers where a reflected property is unavailable. I use it sparingly and document lifetime because it can unintentionally retain objects.
- **Common Weak Answer:** "Use it instead of UPROPERTY."
- **Follow-up Questions:** non-UObject owner? leak risk? weak/soft alternative?
- **Hands-on Verification Task:** Retain a UObject from a non-UObject helper and then release it under forced GC.
- **Sources:** [SRC-EPIC-009], [SRC-EPIC-003]
- **Version Notes:** UE5 pointer guidance is version-sensitive.

### Question: Why can raw UObject members be dangerous even if they are never null in testing?

- **Category:** UE C++ / Lifetime
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Reflection/GC may not see the reference, and destruction/editor reload/async timing can invalidate assumptions.
- **Strong 3-Year-Engineer Answer:** A raw pointer can work accidentally while some other root retains the object. I ask what owns the object and whether the reference edge is visible to GC, serialisation, editor undo or replication. If it is only a temporary non-owning observation, weak or raw may be fine with validity checks; persistent ownership needs a tracked path.
- **Common Weak Answer:** "It works because another object owns it."
- **Follow-up Questions:** GC edge? weak? async? editor reload?
- **Hands-on Verification Task:** Force GC after removing the hidden owner and observe the raw pointer.
- **Sources:** [SRC-EPIC-005], [SRC-EPIC-009], [SRC-EPIC-006]
- **Version Notes:** Pointer wrappers changed in UE5.

### Question: How do you diagnose a CDO mutation bug?

- **Category:** UObject / Construction
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Check whether code changed class default object/default subobject state instead of per-instance runtime state.
- **Strong 3-Year-Engineer Answer:** Constructors and default subobjects configure defaults through the CDO. Runtime mutation should happen on instances at appropriate lifecycle points. If all future instances inherit unexpected state or editor defaults change, I inspect constructor, PostInit, construction script and asset default mutation paths.
- **Common Weak Answer:** "The constructor ran twice."
- **Follow-up Questions:** CDO? default subobject? construction script? instance?
- **Hands-on Verification Task:** Mutate a default subobject at runtime incorrectly and observe new instance defaults.
- **Sources:** [SRC-EPIC-003], [SRC-EPIC-006], [SRC-EPIC-012]
- **Version Notes:** Lifecycle details are branch-sensitive.

### Question: Why can `ConstructorHelpers` create asset-coupling problems?

- **Category:** UE C++ / Assets
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** It creates hard constructor-time references that affect load/cook dependencies and can be inflexible for data-driven content.
- **Strong 3-Year-Engineer Answer:** ConstructorHelpers can be valid for fixed native defaults, but widespread use hardcodes asset paths and loads dependencies early. I prefer editable soft references, Data Assets or Asset Manager rules where designers need variation or streaming control.
- **Common Weak Answer:** "Use it to load assets in C++."
- **Follow-up Questions:** hard ref? cook? streaming? designer override?
- **Hands-on Verification Task:** Replace a hard constructor asset path with a soft/Data Asset-driven reference and inspect cook deps.
- **Sources:** [SRC-EPIC-004], [SRC-ASSET-001], [SRC-ASSET-003]
- **Version Notes:** Asset loading rules are project/version sensitive.

### Question: How do you choose between delegate, interface and direct call?

- **Category:** Gameplay Architecture / Communication
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Direct calls fit known required collaborators; interfaces fit polymorphic requests; delegates/events fit observation and fan-out.
- **Strong 3-Year-Engineer Answer:** I consider ownership, lifetime, cardinality, authority and testability. Delegates need unbinding/lifetime care; interfaces should express required capability; direct references are clearest when dependency is real and scoped. Overusing events hides flow.
- **Common Weak Answer:** "Delegates make it decoupled."
- **Follow-up Questions:** lifetime? one-to-many? authority? test?
- **Hands-on Verification Task:** Refactor an interaction system three ways and compare debugging clarity.
- **Sources:** [SRC-EPIC-029], [SRC-ARCH-001], [SRC-ARCH-004]
- **Version Notes:** Delegate binding APIs vary.

### Question: How do you prevent subsystem abuse?

- **Category:** Gameplay Architecture / Subsystems
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Define scope, lifetime, ownership, authority, dependencies and test seams instead of using global access casually.
- **Strong 3-Year-Engineer Answer:** Subsystems are useful for engine/world/game-instance/local-player services, but they can become service locators. I document which world/player they belong to, teardown order, net mode, threading and dependency interface. Gameplay state that belongs to actors, saves or server authority should not be hidden in the wrong subsystem.
- **Common Weak Answer:** "Put shared things in GameInstanceSubsystem."
- **Follow-up Questions:** world scope? local player? multiplayer? teardown?
- **Hands-on Verification Task:** Move a global inventory service to the correct owner and explain why.
- **Sources:** [SRC-EPIC-017], [SRC-ARCH-007], [SRC-EPIC-018]
- **Version Notes:** Subsystem lifetimes are stable conceptually; APIs vary.

### Question: How do you decide whether to tick?

- **Category:** Gameplay Framework / Performance
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Tick only when continuous per-frame work is required and cheaper event/timer/batch/significance alternatives do not fit.
- **Strong 3-Year-Engineer Answer:** I classify update frequency, owner count, data dependencies and visibility/importance. Timers/events are better for sparse changes, but high-frequency batched work can beat many delegate events. I measure cost, disable unused ticks and consider manager batching or Mass for large populations.
- **Common Weak Answer:** "Never tick."
- **Follow-up Questions:** timers? batching? significance? event storm?
- **Hands-on Verification Task:** Replace 1000 actor ticks with a batched manager and compare.
- **Sources:** [SRC-EPIC-011], [SRC-PERF-004], [SRC-PERF-005]
- **Version Notes:** Tick configuration details are branch-sensitive.

### Question: What does Actor ownership not mean?

- **Category:** Gameplay Framework / Networking
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Ownership is not attachment, not authority, not lifetime ownership and not necessarily relevance by itself.
- **Strong 3-Year-Engineer Answer:** In Unreal networking, owner/owning connection affects RPC permission and owner-based replication conditions. Authority is about which machine owns truth. Attachment is transform hierarchy. Lifetime may be GC/outer/world ownership. Mixing these meanings causes RPC and replication bugs.
- **Common Weak Answer:** "Owner means who controls it."
- **Follow-up Questions:** attachment? outer? server authority? owner-only replication?
- **Hands-on Verification Task:** Build an attached weapon that is not owned by the right controller and fix RPC routing.
- **Sources:** [SRC-NET-003], [SRC-EPIC-011], [SRC-EPIC-012]
- **Version Notes:** Ownership semantics stable; implementation details vary.

### Question: How do you debug a BeginPlay ordering assumption?

- **Category:** Gameplay Framework / Lifecycle
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Log lifecycle and dependencies, then move setup to explicit readiness/registration instead of relying on incidental order.
- **Strong 3-Year-Engineer Answer:** BeginPlay order across actors/components/streamed levels/network spawn can surprise. I avoid "actor B exists because A began first" assumptions. Systems should register, wait for required dependencies or use authoritative initialization points such as GameMode/GameState/subsystem flows.
- **Common Weak Answer:** "Delay by one frame."
- **Follow-up Questions:** streamed level? replication? component registration? dependency?
- **Hands-on Verification Task:** Create two actors with BeginPlay ordering dependency and replace it with registration/readiness.
- **Sources:** [SRC-EPIC-015], [SRC-EPIC-016], [SRC-EPIC-011]
- **Version Notes:** Exact ordering requires target confirmation.

### Question: How do you use Data Assets without putting runtime state in them?

- **Category:** Gameplay Architecture / Data
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Treat Data Assets as definitions/config, and store mutable per-player/per-instance state elsewhere.
- **Strong 3-Year-Engineer Answer:** Data Assets are shared content definitions. If runtime code mutates them, every user of that definition can see unintended changes, and save/replication semantics become unclear. I store instance state in components, structs, save data, ASC specs or replicated collections with stable IDs back to the definition.
- **Common Weak Answer:** "Data Assets are item instances."
- **Follow-up Questions:** shared asset? save? replication? instance ID?
- **Hands-on Verification Task:** Build item definitions and per-player inventory instances.
- **Sources:** [SRC-ARCH-008], [SRC-ASSET-005], [SRC-EPIC-007]
- **Version Notes:** Asset/editor behaviour is branch-sensitive.

### Question: How do you design a quest objective system for networking and saves?

- **Category:** Gameplay Architecture / Quests
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Store durable objective state on server/save with stable IDs, project presentation separately, and make updates idempotent.
- **Strong 3-Year-Engineer Answer:** Objective definitions live in data; player/world progress lives in authoritative state. Events should carry stable IDs and operation IDs where duplication is possible. UI, markers and world actors present that state and can be rebuilt after streaming, late join or load.
- **Common Weak Answer:** "Each quest actor tracks itself."
- **Follow-up Questions:** stable IDs? late join? duplicate event? streaming?
- **Hands-on Verification Task:** Complete an objective in a streamed cell, unload it and reload/save/late-join.
- **Sources:** [SRC-ARCH-010], [SRC-ASSET-010], [SRC-NET-004]
- **Version Notes:** Save/quest patterns are project-specific.

### Question: How do you evaluate a spatial hash implementation?

- **Category:** Algorithms / Spatial Structures
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Test distribution, cell size, query radius, worst-case clustering, update cost and correctness against a slow oracle.
- **Strong 3-Year-Engineer Answer:** A spatial hash can make neighbour queries fast on average, but bad cell size or clustering can degrade heavily. I compare against all-pairs on generated/adversarial cases, track false negatives, update cost and memory, and choose structure based on workload distribution.
- **Common Weak Answer:** "Use spatial hash for nearby objects."
- **Follow-up Questions:** cell size? clustering? dynamic updates? oracle?
- **Hands-on Verification Task:** Build random and clustered tests and compare against brute force.
- **Sources:** [SRC-ALG-010], [SRC-ALG-011], [SRC-MATH-011]
- **Version Notes:** Algorithm is engine-independent.

### Question: Why is A* admissibility not the whole pathfinding story?

- **Category:** Algorithms / AI Navigation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Heuristic admissibility helps optimality, but real games also need graph quality, dynamic obstacles, smoothing, costs, queries and budget.
- **Strong 3-Year-Engineer Answer:** In UE, Recast/NavMesh handles much of the path graph, but understanding A* helps reason about costs, heuristics and failure. I also care about path following, avoidance, links, partial paths, dynamic generation, EQS query cost and debug visualization.
- **Common Weak Answer:** "A* finds shortest path."
- **Follow-up Questions:** costs? heuristic? avoidance? partial path?
- **Hands-on Verification Task:** Implement grid A* and compare with UE NavMesh behaviour on costs/links.
- **Sources:** [SRC-ALG-007], [SRC-AI-006], [SRC-AI-007]
- **Version Notes:** UE navigation behaviour is branch/settings sensitive.

### Question: How do you debug non-uniform scale transforming normals?

- **Category:** Maths / Rendering
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Normals require inverse-transpose style handling, not the same transform as positions under non-uniform scale.
- **Strong 3-Year-Engineer Answer:** Points, vectors and normals transform differently. Non-uniform scale can make a visually plausible mesh shade incorrectly if normals are transformed like positions. I build a small test with known normal/light direction and compare engine/material output to expected math.
- **Common Weak Answer:** "Normalize after transform."
- **Follow-up Questions:** inverse transpose? tangent space? negative scale?
- **Hands-on Verification Task:** Visualise normals on a non-uniformly scaled mesh and explain the lighting error.
- **Sources:** [SRC-MATH-005], [SRC-MATH-010], [SRC-RENDER-004]
- **Version Notes:** Maths stable; engine spaces/settings vary.

### Question: Why do quaternions not automatically solve rotation design?

- **Category:** Maths / Rotation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** They avoid many Euler/gimbal issues, but interpolation, shortest path, normalisation and designer-facing controls still matter.
- **Strong 3-Year-Engineer Answer:** Quaternions are a representation/tool, not a policy. I decide whether rotation should be constant-speed, shortest-arc, constrained, network-smoothed or designer-authored. I also normalise and handle conversions carefully because exposing raw quaternions to designers is rarely ergonomic.
- **Common Weak Answer:** "Use quaternions to avoid gimbal lock."
- **Follow-up Questions:** Slerp? shortest path? constant speed? network smoothing?
- **Hands-on Verification Task:** Compare Rotator interpolation, Slerp and constant angular speed for a turret.
- **Sources:** [SRC-MATH-006], [SRC-MATH-007], [SRC-NET-009]
- **Version Notes:** API names/version availability vary.

### Question: How do you reason about floating-point determinism in gameplay?

- **Category:** Maths / Determinism
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Floating point is approximate and platform/order sensitive; deterministic gameplay needs constrained operations, fixed steps, quantisation or authoritative correction.
- **Strong 3-Year-Engineer Answer:** I avoid assuming bit-identical floats across platforms, compilers and execution order. For networked gameplay I use server authority, quantised replicated state or rollback models with strict determinism proofs. For local gameplay I define tolerances and avoid exact equality.
- **Common Weak Answer:** "Use doubles."
- **Follow-up Questions:** fixed step? quantisation? tolerance? platform?
- **Hands-on Verification Task:** Create a simulation that diverges with different update order and repair it.
- **Sources:** [SRC-MATH-002], [SRC-NET-009], [SRC-NET-020]
- **Version Notes:** Determinism requirements are system-specific.

### Question: What makes Lua integration unsafe around UObject lifetime?

- **Category:** Scripting / Lua
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Script references can outlive UObjects or hide references from GC unless the bridge defines ownership and validity clearly.
- **Strong 3-Year-Engineer Answer:** Lua is plugin-dependent in Unreal projects, so bridge semantics matter. I need to know whether wrappers hold weak, strong or reflected references; how destruction invalidates script objects; and how hot reload/reload handles delegates/timers. I avoid presenting plugin behaviour as core UE.
- **Common Weak Answer:** "Lua has garbage collection too."
- **Follow-up Questions:** UObject GC? weak wrapper? delegate unbind? plugin?
- **Hands-on Verification Task:** Destroy a UObject referenced by Lua and test wrapper validity/error behaviour.
- **Sources:** [SRC-SCRIPT-001], [SRC-SCRIPT-002], [SRC-EPIC-009]
- **Version Notes:** Lua integration is plugin-dependent.

### Question: How should C# tooling integration be scoped in Unreal workflows?

- **Category:** Scripting / C#
- **Priority:** P3
- **Expected Depth:** D2
- **Short Answer:** Treat it as editor/tool/build ecosystem integration unless the project has a specific runtime C# plugin with proven constraints.
- **Strong 3-Year-Engineer Answer:** C# may appear in AutomationTool/Gauntlet/build tooling or plugins. I separate official tooling usage from runtime scripting ecosystems. For runtime C#, I ask about plugin support, hot reload, GC/lifetime bridge, platform support, packaging and performance before making claims.
- **Common Weak Answer:** "Unreal supports C# like Unity."
- **Follow-up Questions:** runtime plugin? packaging? UObject bridge? platform?
- **Hands-on Verification Task:** Classify three C# uses as AutomationTool, editor tooling or runtime plugin.
- **Sources:** [SRC-SCRIPT-006], [SRC-SCRIPT-007], [SRC-BUILD-016]
- **Version Notes:** Runtime C# is plugin-dependent.

### Question: How do you avoid managed/native lifetime bugs in scripting bridges?

- **Category:** Scripting / Interop
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Define ownership, weak/strong handles, invalidation, thread affinity and teardown paths explicitly.
- **Strong 3-Year-Engineer Answer:** Managed objects and UObjects have separate lifetime systems. A bridge must decide whether native references are tracked by Unreal GC, script GC or explicit handles. I test object destruction, world teardown, script reload, delegate callbacks and async calls after owner death.
- **Common Weak Answer:** "The wrapper keeps it alive."
- **Follow-up Questions:** weak handle? world teardown? delegate? async?
- **Hands-on Verification Task:** Trigger callback after UObject destruction and verify safe failure.
- **Sources:** [SRC-SCRIPT-002], [SRC-SCRIPT-008], [SRC-EPIC-009]
- **Version Notes:** Bridge behaviour is plugin-specific.

### Question: How do you answer console profiling questions without leaking confidential details?

- **Category:** Platform / Console
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Discuss source-safe methodology: target package identity, authorised tools, scenario markers, before/after evidence and confidentiality boundaries.
- **Strong 3-Year-Engineer Answer:** I do not invent or disclose platform-holder counters, TRC/TCR details or tool screenshots. I explain how I would use authorised devkit tooling, symbols, package identity and UE markers, then report source-safe conclusions such as CPU/GPU/memory ownership and residual risk.
- **Common Weak Answer:** "Share exact console certification rules."
- **Follow-up Questions:** confidentiality? authorised docs? symbol match? source-safe report?
- **Hands-on Verification Task:** Rewrite a platform-profiler report with confidential details redacted but useful ownership retained.
- **Sources:** [SRC-PLAT-024], [SRC-PERF-003], [SRC-BUILD-018]
- **Version Notes:** Console details require authorised platform docs.

### Question: How do Apple MetricKit and Xcode captures complement Unreal evidence?

- **Category:** Platform / Apple Profiling
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Unreal traces show engine-side ownership; Apple tools show platform/GPU/memory/responsiveness evidence on real devices.
- **Strong 3-Year-Engineer Answer:** I align UE markers, build ID and scenario with Xcode/Instruments/Metal/MetricKit artifacts. Platform evidence can reveal GPU pass, memory high-water, responsiveness or field trends that engine-only data misses. I still map findings back to UE systems/content before fixing.
- **Common Weak Answer:** "Use Xcode instead of Unreal Insights."
- **Follow-up Questions:** markers? device vs simulator? field metric? Metal capture?
- **Hands-on Verification Task:** Correlate a UE CSV spike with one Xcode/Instruments finding.
- **Sources:** [SRC-PLAT-018], [SRC-PLAT-019], [SRC-PLAT-021], [SRC-PERF-003]
- **Version Notes:** Apple tooling depends on Xcode/OS/device.

### Question: Why is simulator memory not device memory proof?

- **Category:** Platform / Apple
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Simulators differ in hardware, OS services, graphics stack, memory pressure and packaging from physical devices.
- **Strong 3-Year-Engineer Answer:** Simulators are useful for functional iteration, but memory/performance release proof needs physical device or target-equivalent hardware. I record device model, OS, build config, scenario and memory high-water; then compare with field or device profiling where available.
- **Common Weak Answer:** "It passes on simulator."
- **Follow-up Questions:** GPU? memory pressure? OS? package?
- **Hands-on Verification Task:** Run the same scene on simulator and device and compare memory/performance.
- **Sources:** [SRC-PLAT-002], [SRC-PLAT-020], [SRC-PERF-003]
- **Version Notes:** Platform support changes with SDK/device.

### Question: How do Android AGI frame profiles complement Unreal GPU stats?

- **Category:** Platform / Android Profiling
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Unreal shows engine pass/timing context; AGI can expose device/GPU render-pass and counter details where supported.
- **Strong 3-Year-Engineer Answer:** I use UE markers/CSV to locate the failing frame, then AGI to inspect render pass, tiler/binning/rendering or GMEM load/store style costs where available. I do not compare raw counters across vendors without documentation, and I map the finding back to UE content/pass ownership.
- **Common Weak Answer:** "AGI tells the exact Unreal fix."
- **Follow-up Questions:** frame match? vendor counters? GMEM? UE pass?
- **Hands-on Verification Task:** Capture a GPU-bound Android frame and map longest pass to a UE feature.
- **Sources:** [SRC-PLAT-010], [SRC-PLAT-013], [SRC-PLAT-014], [SRC-PERF-003]
- **Version Notes:** AGI support depends on device/driver/API.

### Question: How do you use Android LMK evidence in a UE memory investigation?

- **Category:** Platform / Android Memory
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Correlate OS low-memory kill/process evidence with UE memory checkpoints and scenario state.
- **Strong 3-Year-Engineer Answer:** I record UE LLM/Memory Insights checkpoints around map load, peak gameplay and background/foreground. If Android LMK kills the app, I correlate process logs, memory pressure, lifecycle state and UE resident allocations. The fix might be content budget, streaming, cache lifetime or platform lifecycle handling.
- **Common Weak Answer:** "The OS killed us randomly."
- **Follow-up Questions:** background? peak? LLM tag? lifecycle?
- **Hands-on Verification Task:** Run background/foreground memory pressure and correlate UE checkpoints with OS logs.
- **Sources:** [SRC-PLAT-015], [SRC-PERF-006], [SRC-PERF-010]
- **Version Notes:** Android memory policies vary by OS/device.

### Question: How can ADPF hide a regression?

- **Category:** Platform / Android Performance
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Adaptive performance can reduce quality or workload, making frame time look better without fixing the underlying cost.
- **Strong 3-Year-Engineer Answer:** ADPF can be valuable for thermal/sustained performance, but I compare baseline and adapted runs. If quality drops to mask a GPU or CPU regression, the report must say so. I treat adaptation as product policy, not a substitute for optimisation.
- **Common Weak Answer:** "ADPF fixes mobile performance."
- **Follow-up Questions:** quality changed? thermal? baseline? user experience?
- **Hands-on Verification Task:** Run a scenario with and without adaptation and record quality/performance changes.
- **Sources:** [SRC-PLAT-016], [SRC-PLAT-017], [SRC-PERF-009]
- **Version Notes:** ADPF support is device/plugin/version sensitive.

### Question: What belongs in a device-lab run manifest?

- **Category:** Automation / Device Lab
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Build ID, device ID, scenario ID, command line, profile/settings, artifact paths, timestamps and gate policy.
- **Strong 3-Year-Engineer Answer:** A device-lab result must bind exact package to exact device and scenario. I include install/launch command, OS/toolchain, power/thermal state where relevant, CSV/log paths, screenshots/video if useful, rerun policy and owner. Otherwise failures cannot be compared or reproduced.
- **Common Weak Answer:** "The lab output says failed."
- **Follow-up Questions:** artifact identity? device state? rerun? owner?
- **Hands-on Verification Task:** Write a run manifest for two devices and two scenarios.
- **Sources:** [SRC-PLAT-009], [SRC-BUILD-017], [SRC-PERF-011]
- **Version Notes:** Gauntlet/device adapters are branch/platform sensitive.

### Question: How should automation classify failures?

- **Category:** Automation / Triage
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Separate product failure, infrastructure failure, flaky device/test, blocked by missing evidence and reporting-only metric drift.
- **Strong 3-Year-Engineer Answer:** Blind reruns erode trust. I capture first causal log, artifact identity and device state, then classify. Product failures get owners and blockers; lab infrastructure gets infra owner; flaky devices enter quarantine; missing evidence fails the proof process if the gate is meant to block.
- **Common Weak Answer:** "Rerun failures."
- **Follow-up Questions:** quarantine? deterministic? first-cause? missing evidence?
- **Hands-on Verification Task:** Classify ten historical lab failures and define recurrence guards.
- **Sources:** [SRC-PERF-011], [SRC-PLAT-009], [SRC-BUILD-018]
- **Version Notes:** Automation policy is project-specific.

### Question: How do you design a smoke scenario for packaging?

- **Category:** Build / Smoke Testing
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It should launch packaged build, load representative content, execute a small deterministic path and exit with machine-readable pass/fail.
- **Strong 3-Year-Engineer Answer:** A smoke scenario is not a full test suite. It catches missing modules/assets/config, startup crashes and basic map readiness quickly. I include log markers, timeout, screenshot optional, artifact pull and stable exit code so CI can distinguish product failure from runner failure.
- **Common Weak Answer:** "Open the game manually."
- **Follow-up Questions:** timeout? pass marker? missing asset? exit code?
- **Hands-on Verification Task:** Create a front-end-to-map smoke path with pass/fail log markers.
- **Sources:** [SRC-BUILD-017], [SRC-PERF-011], [SRC-PLAT-009]
- **Version Notes:** Runner APIs are branch/platform sensitive.

### Question: How do you validate a plugin's runtime/editor module split?

- **Category:** Build / Plugins
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Build/package non-editor targets and audit module dependencies, descriptors and runtime assets.
- **Strong 3-Year-Engineer Answer:** Editor code must not leak into runtime modules or packaged assets. I check Build.cs dependencies, module type/loading phase, descriptor filters, UObject class references in assets and Shipping/Game builds. The proof is a package, not only an Editor compile.
- **Common Weak Answer:** "It compiles in the editor."
- **Follow-up Questions:** module type? asset class? server target? Shipping?
- **Hands-on Verification Task:** Put an editor-only class into a runtime asset and catch it in package.
- **Sources:** [SRC-BUILD-005], [SRC-BUILD-006], [SRC-BUILD-017]
- **Version Notes:** Module descriptors vary by UE version.

### Question: How do unity builds hide include and ODR problems?

- **Category:** C++ / Build
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Unity compilation merges translation units, masking missing includes and changing duplicate/static initialization visibility.
- **Strong 3-Year-Engineer Answer:** Code that compiles only in unity may rely on incidental includes from neighbouring files. I run non-unity or targeted clean jobs for important modules, maintain IWYU discipline and treat ODR/static-init failures separately from ordinary syntax errors.
- **Common Weak Answer:** "Unity builds are faster."
- **Follow-up Questions:** missing include? ODR? generated header? clean build?
- **Hands-on Verification Task:** Remove an include masked by unity and catch it in non-unity build.
- **Sources:** [SRC-BUILD-001], [SRC-CPP-027], [SRC-CPP-029]
- **Version Notes:** Build settings are project-specific.

### Question: How do you debug Unreal unresolved externals across modules?

- **Category:** C++ / Unreal Modules
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Compare declaration/definition/export macro/module dependency/target visibility and template instantiation.
- **Strong 3-Year-Engineer Answer:** I read the mangled or demangled symbol, verify namespace/class/signature, ensure the definition is linked into the module, add proper Public/Private dependency, and check API export macros across module boundaries. Unity or stale binaries can mask the real issue.
- **Common Weak Answer:** "Add the module to Build.cs."
- **Follow-up Questions:** export macro? Private dependency? template? target?
- **Hands-on Verification Task:** Create an exported class used by another module and break/fix the API macro.
- **Sources:** [SRC-BUILD-001], [SRC-BUILD-002], [SRC-CPP-027]
- **Version Notes:** Toolchain/linker details vary.

### Question: Why can static initialisation be especially risky in Unreal modules?

- **Category:** C++ / Startup
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Cross-translation-unit and module load order can run before engine systems, reflection, configs or plugin dependencies are ready.
- **Strong 3-Year-Engineer Answer:** I avoid non-local dynamic initialisation that depends on engine state. Prefer function-local statics for pure constants, explicit module startup for engine-dependent setup and clear shutdown order. Hot reload/live coding and plugin unload make hidden globals even riskier.
- **Common Weak Answer:** "Static variables are fine if they compile."
- **Follow-up Questions:** module startup? shutdown? hot reload? config?
- **Hands-on Verification Task:** Create a global that touches UObject system too early and move it to module startup.
- **Sources:** [SRC-CPP-029], [SRC-BUILD-005], [SRC-EPIC-002]
- **Version Notes:** Module load order is project/branch sensitive.

### Question: When should you use `TFunctionRef` instead of `TFunction`?

- **Category:** UE C++ / Callables
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Use `TFunctionRef` for non-owning synchronous callbacks; use owning wrappers when callback must outlive the call.
- **Strong 3-Year-Engineer Answer:** A function ref should not escape. It avoids ownership/allocation overhead but relies on caller lifetime. If I store, dispatch async or bind later, I need an owning callable/delegate and safe captured object lifetime. This mirrors standard callable ownership reasoning.
- **Common Weak Answer:** "TFunctionRef is faster."
- **Follow-up Questions:** stored? async? captured UObject? delegate?
- **Hands-on Verification Task:** Write a function that accidentally stores a function ref and repair it.
- **Sources:** [SRC-CPP-018], [SRC-EPIC-029]
- **Version Notes:** Unreal callable APIs vary.

### Question: How do lambda captures create async lifetime bugs?

- **Category:** C++ / Async
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Captured references or raw pointers may outlive their owners when the task/callback runs later.
- **Strong 3-Year-Engineer Answer:** I capture values deliberately, use weak pointers for UObjects, check validity on the correct thread and cancel/unbind on teardown. Capturing `this` into latent/async work is a lifetime contract, not a convenience.
- **Common Weak Answer:** "Capture by reference to avoid copies."
- **Follow-up Questions:** UObject validity? thread affinity? cancellation? shared ownership?
- **Hands-on Verification Task:** Trigger a callback after owner destruction and fix with weak capture/unbind.
- **Sources:** [SRC-CPP-002], [SRC-CPP-026], [SRC-EPIC-009]
- **Version Notes:** Async APIs and thread rules are branch-sensitive.

### Question: How do you approach a standard C++ move bug in Unreal container code?

- **Category:** C++ / Move Semantics
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Check moved-from validity, ownership transfer, container invalidation and Unreal relocation/copy expectations.
- **Strong 3-Year-Engineer Answer:** `MoveTemp`/`std::move` casts; it does not by itself transfer correctly. I inspect special members, noexcept, resource ownership and container operation. For UObjects I avoid moving ownership the way I would with unique C++ objects and respect reflection/GC references.
- **Common Weak Answer:** "Move makes it faster."
- **Follow-up Questions:** moved-from state? noexcept? UObject? invalidation?
- **Hands-on Verification Task:** Move a struct holding arrays/pointers through a container and verify invariants.
- **Sources:** [SRC-CPP-003], [SRC-CPP-004], [SRC-CPP-005], [SRC-EPIC-024]
- **Version Notes:** Standard C++ plus Unreal container details.

### Question: How do you decide between `TArray`, `TSet` and `TMap`?

- **Category:** UE C++ / Containers
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Choose by access pattern, ordering, uniqueness, key lookup, mutation frequency, memory and iteration cost.
- **Strong 3-Year-Engineer Answer:** `TArray` is strong for dense ordered iteration and cache locality. `TSet` and `TMap` trade memory and order for hash lookup/uniqueness. I consider invalidation, stable handles, replication/serialisation, deterministic order and hot-path iteration before choosing.
- **Common Weak Answer:** "Map for lookup, array for lists."
- **Follow-up Questions:** order? invalidation? memory? determinism?
- **Hands-on Verification Task:** Implement an inventory lookup three ways and profile iteration/lookup/memory.
- **Sources:** [SRC-EPIC-024], [SRC-EPIC-025], [SRC-EPIC-026]
- **Version Notes:** Container APIs stable conceptually.

### Question: Why is `FText` not just a string?

- **Category:** UE C++ / Localisation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** `FText` carries localisation/display semantics, while `FString` is mutable text data and `FName` is an identity/name key.
- **Strong 3-Year-Engineer Answer:** I use `FText` for user-facing display, culture formatting and localisation; `FString` for string manipulation/storage; `FName` for case-insensitive-ish identifiers/tags/keys where appropriate. Converting casually can lose localisation history or create identity bugs.
- **Common Weak Answer:** "FText is for UI."
- **Follow-up Questions:** culture? invariant? FName? conversion?
- **Hands-on Verification Task:** Localise an item name and show how bad conversion breaks display/history.
- **Sources:** [SRC-EPIC-022], [SRC-EPIC-023]
- **Version Notes:** Core semantics stable.

### Question: How do you debug a soft reference that loads in editor but not package?

- **Category:** Assets / Soft References
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Check cook inclusion, Asset Manager rules, Primary Assets, labels, redirectors and packaged path logs.
- **Strong 3-Year-Engineer Answer:** A soft reference is an address, not a guarantee the asset is cooked or resident. Editor may find loose content that the package excludes. I prove Asset Manager/cook rules include the asset, then test async load in packaged runtime and handle failure gracefully.
- **Common Weak Answer:** "Soft references are broken."
- **Follow-up Questions:** Primary Asset? label? redirector? async failure?
- **Hands-on Verification Task:** Package a soft-referenced cosmetic and fix missing cook inclusion.
- **Sources:** [SRC-ASSET-001], [SRC-ASSET-003], [SRC-ASSET-005], [SRC-ASSET-006]
- **Version Notes:** Cook settings are project-specific.

### Question: What is the difference between Asset Registry and Asset Manager?

- **Category:** Assets / Systems
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Asset Registry indexes asset metadata; Asset Manager adds project-level Primary Asset identity, loading, bundles and cook/chunk policy.
- **Strong 3-Year-Engineer Answer:** The registry helps discover assets and metadata without fully loading everything. Asset Manager gives higher-level rules for game content identity and lifecycle. I do not use registry discovery as a substitute for cook/ownership rules.
- **Common Weak Answer:** "Both find assets."
- **Follow-up Questions:** Primary Asset? bundles? cook? metadata?
- **Hands-on Verification Task:** Discover items through registry, then manage loading/cook through Asset Manager.
- **Sources:** [SRC-ASSET-004], [SRC-ASSET-005]
- **Version Notes:** Asset Manager rules are project-config sensitive.

### Question: Why can async loading still hitch?

- **Category:** Assets / Loading
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Async loading can still trigger game-thread registration, UObject creation, shader/PSO use, texture streaming, callbacks and dependency loads.
- **Strong 3-Year-Engineer Answer:** Async request completion is not the same as fully hitch-free presentation. I profile load time, game-thread object registration, component creation, shader/material first use and memory pressure. The fix can be preload, staged activation, asset simplification or better dependency boundaries.
- **Common Weak Answer:** "Async load avoids hitches."
- **Follow-up Questions:** callback work? dependencies? shader? component registration?
- **Hands-on Verification Task:** Async-load a complex actor class and measure completion versus spawn/registration hitch.
- **Sources:** [SRC-ASSET-002], [SRC-ASSET-006], [SRC-PERF-003]
- **Version Notes:** Async loading internals are branch-sensitive.

### Question: How do you design a reproducible algorithm benchmark?

- **Category:** Algorithms / Performance
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Fix input distribution, seed, workload, warm-up, measurement, correctness oracle and percentile reporting.
- **Strong 3-Year-Engineer Answer:** Benchmarks lie when they use one random input or ignore correctness. I generate typical and adversarial cases, compare with a slow oracle, report P50/P95/worst and separate allocation/setup from query/update cost. For games, frame budget and distribution matter more than big-O alone.
- **Common Weak Answer:** "Measure one large input."
- **Follow-up Questions:** oracle? seed? distribution? allocation?
- **Hands-on Verification Task:** Benchmark spatial hash queries against brute force across three distributions.
- **Sources:** [SRC-ALG-001], [SRC-ALG-010], [SRC-PERF-003]
- **Version Notes:** Method is engine-independent.

### Question: What is an interview-safe way to discuss lock-free code?

- **Category:** C++ / Concurrency
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Start with correctness and ownership; use lock-free only with a written memory-order/progress proof and measurement.
- **Strong 3-Year-Engineer Answer:** Most gameplay systems do not need custom lock-free structures. I prefer message queues, task partitioning or mutex-protected invariants unless contention proves otherwise. If using atomics, I define publication, lifetime, ABA/destruction issues, memory order and tests on target platforms.
- **Common Weak Answer:** "Lock-free is faster."
- **Follow-up Questions:** ABA? reclamation? memory order? progress?
- **Hands-on Verification Task:** Replace a broken atomic multi-field invariant with mutex or proven single-producer queue.
- **Sources:** [SRC-CPP-020], [SRC-CPP-021], [SRC-CPP-022]
- **Version Notes:** Standard C++11+; platform performance varies.

### Question: How do you reason about false sharing in game code?

- **Category:** C++ / Performance
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Multiple threads writing different data on the same cache line can fight over ownership and slow down.
- **Strong 3-Year-Engineer Answer:** False sharing appears when worker outputs, counters or component data place hot writes adjacent in memory. I prove it with profiling/counters or controlled benchmarks, then use per-thread buffers, padding/alignment or data layout changes. I do not blindly pad every struct.
- **Common Weak Answer:** "Cache lines only matter in engine code."
- **Follow-up Questions:** per-thread output? padding? SoA? measurement?
- **Hands-on Verification Task:** Benchmark per-thread counters adjacent versus padded/per-thread vectors.
- **Sources:** [SRC-CPP-023], [SRC-CPP-020], [SRC-PERF-003]
- **Version Notes:** Cache line constants and effects are platform-specific.

### Question: How do you answer "inheritance or composition" in Unreal?

- **Category:** Architecture / C++ Design
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Use inheritance for true substitutable type relationships; use components/composition for mixable behaviours, data and feature variation.
- **Strong 3-Year-Engineer Answer:** Unreal encourages Actor/Component composition for gameplay features because it avoids deep hierarchies and improves reuse. Inheritance still fits engine framework types and stable specialisations. I also consider Blueprint authoring, replication, lifetime and performance.
- **Common Weak Answer:** "Composition is always better."
- **Follow-up Questions:** substitutability? component overhead? Blueprint? replication?
- **Hands-on Verification Task:** Refactor a deep enemy class hierarchy into components and note trade-offs.
- **Sources:** [SRC-ARCH-001], [SRC-EPIC-011], [SRC-EPIC-012], [SRC-CPP-001]
- **Version Notes:** Principle stable; UE implementation varies.

### Question: How do you design an interaction system that works with AI, player and multiplayer?

- **Category:** Gameplay Architecture / Interaction
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Separate discoverability, eligibility, authority, execution, prediction/presentation and ownership of state.
- **Strong 3-Year-Engineer Answer:** A player trace, AI Smart Object query and scripted use should converge on a common interaction contract. The server validates eligibility, state changes are durable/replicated, and clients show prediction/cosmetic feedback. I test contention, cancellation, stale interactables and streamed actors.
- **Common Weak Answer:** "Line trace and call Use."
- **Follow-up Questions:** AI? server? contention? streamed? UI prompt?
- **Hands-on Verification Task:** Build a door/terminal interaction used by player and AI with server validation.
- **Sources:** [SRC-ARCH-004], [SRC-AI-011], [SRC-NET-006]
- **Version Notes:** Architecture is project-specific.

### Question: How do you design equipment that grants abilities and UI state?

- **Category:** Gameplay Architecture / Equipment
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use definitions plus instance state, a grant ledger, authority validation and projection to UI/animation/audio.
- **Strong 3-Year-Engineer Answer:** Equipment is not just attaching a mesh. It can change stats, abilities, input mapping, UI, animation layers and save state. I track source item instance, granted ability/effect handles, attachment state and cleanup on unequip/drop/death.
- **Common Weak Answer:** "Attach weapon and add ability."
- **Follow-up Questions:** cleanup? save? ASC? animation? prediction?
- **Hands-on Verification Task:** Equip/unequip a weapon that grants an ability and prove cleanup after death.
- **Sources:** [SRC-ARCH-009], [SRC-GAS-003], [SRC-ANIM-006]
- **Version Notes:** Equipment architecture is project-specific.

### Question: How do you prevent duplicate rewards?

- **Category:** Gameplay Architecture / Idempotency
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Use authoritative operation IDs, durable completion state and idempotent application.
- **Strong 3-Year-Engineer Answer:** Rewards can duplicate through retries, disconnects, rollback, save/load or repeated events. I store completion/operation IDs on the authoritative side and make reward application check prior commit. UI and audio can replay, but inventory/currency truth should not.
- **Common Weak Answer:** "Disable the button."
- **Follow-up Questions:** retry? save? network? analytics?
- **Hands-on Verification Task:** Simulate reward RPC retry and prove inventory increments once.
- **Sources:** [SRC-ARCH-014], [SRC-NET-001], [SRC-ASSET-006]
- **Version Notes:** Pattern project-specific.

### Question: What is a strong answer to "what did you optimise?"

- **Category:** Interview Meta / Evidence
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** State scenario, target, bottleneck evidence, change, before/after result, trade-off and recurrence guard.
- **Strong 3-Year-Engineer Answer:** I avoid generic "I improved performance" claims. I say: on target X in scenario Y, P95 frame time was Z due to traced cause A. I changed B, verified correctness/visual quality, measured result C over repeated runs and added guard D. I also state what the evidence does not prove.
- **Common Weak Answer:** "I reduced draw calls."
- **Follow-up Questions:** target? metric? causal proof? trade-off?
- **Hands-on Verification Task:** Rewrite a vague portfolio bullet into an evidence-based optimisation story.
- **Sources:** [SRC-PERF-003], [SRC-PERF-009], [SRC-RENDER-019]
- **Version Notes:** Interview framing role-dependent.

### Question: How do you answer "tell me about a hard bug"?

- **Category:** Interview Meta / Debugging
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Use a structured narrative: symptom, scope, hypotheses, evidence, root cause, fix, guard and lesson.
- **Strong 3-Year-Engineer Answer:** A strong bug story is not a war story. I describe how I narrowed the system boundary, found first reliable evidence, rejected false leads, fixed root cause and added a test/tool/log/gate. The interviewer should hear judgement, not just persistence.
- **Common Weak Answer:** "It was random but I eventually fixed it."
- **Follow-up Questions:** false lead? reproduction? guard? team impact?
- **Hands-on Verification Task:** Write a STAR-style debugging answer for one workbook scenario.
- **Sources:** [SRC-PERF-003], [SRC-BUILD-017], [SRC-NET-010]
- **Version Notes:** General interview skill.

### Question: How do you evaluate whether a feature is production-ready?

- **Category:** Interview Meta / Production Readiness
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Define requirements, target constraints, correctness tests, failure modes, performance evidence, tooling, ownership and rollback/guard strategy.
- **Strong 3-Year-Engineer Answer:** Production readiness means the feature works under expected platforms, content scale, networking modes, save/load, packaging and failure conditions. I want evidence, not only a demo. I also define owner, metrics, debug tools and known risks.
- **Common Weak Answer:** "QA passed it."
- **Follow-up Questions:** scale? failure injection? package? ownership?
- **Hands-on Verification Task:** Create a readiness checklist for one hands-on project.
- **Sources:** [SRC-ARCH-014], [SRC-PERF-009], [SRC-BUILD-017]
- **Version Notes:** Readiness bar depends on product stage.

### Question: How do you decide what not to build?

- **Category:** System Design / Scope
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Tie scope to product requirements, risk, implementation cost, testability, maintenance and evidence.
- **Strong 3-Year-Engineer Answer:** A good system design answer includes exclusions. I state what is out of scope for the first version, what assumptions would trigger redesign and what instrumentation will tell us. This prevents vague mega-systems and shows production judgement.
- **Common Weak Answer:** "Build it flexible for future features."
- **Follow-up Questions:** milestone? extension point? risk? evidence?
- **Hands-on Verification Task:** Add an "out of scope and trigger to revisit" section to a design memo.
- **Sources:** [SRC-ARCH-014], [SRC-ARCH-012]
- **Version Notes:** General architecture skill.

### Question: How do you use failure injection in an Unreal project?

- **Category:** Debugging / Verification
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Deliberately break expected dependencies or states to prove diagnostics and guards catch them.
- **Strong 3-Year-Engineer Answer:** I inject missing cook assets, wrong device profile, latency/loss, stale HLOD, cancelled abilities, destroyed owners, partial OFPA submit or invalid save version. The point is not chaos for its own sake; it proves the system fails clearly and the recurrence guard works.
- **Common Weak Answer:** "Test the happy path thoroughly."
- **Follow-up Questions:** diagnostic clarity? owner? guard? rollback?
- **Hands-on Verification Task:** Add three injected failures to one project extension and document expected evidence.
- **Sources:** [SRC-BUILD-017], [SRC-PERF-011], [SRC-NET-016]
- **Version Notes:** Failure hooks are project-specific.

### Question: How do you choose a profiling tool under time pressure?

- **Category:** Profiling / Workflow
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Pick the cheapest tool that answers the current question, then escalate only when the question requires more detail.
- **Strong 3-Year-Engineer Answer:** Stats classify quickly, CSV tracks repeated automation, Insights explains CPU/task/loading/memory windows, GPU capture explains passes/resources and platform tools explain device/driver/thermal truth. Tool choice follows hypothesis and artifact cost.
- **Common Weak Answer:** "Start with Unreal Insights for everything."
- **Follow-up Questions:** overhead? automation? GPU? platform?
- **Hands-on Verification Task:** Map five symptoms to the first and second profiler you would use.
- **Sources:** [SRC-PERF-001], [SRC-PERF-003], [SRC-PERF-009]
- **Version Notes:** Tool availability varies by branch/platform.

### Question: How do you avoid overfitting an optimisation to one scene?

- **Category:** Profiling / Validation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Test representative scenarios, device tiers, content variations and quality settings, then state the evidence boundary.
- **Strong 3-Year-Engineer Answer:** Optimising one capture can regress another scene. I validate against low/high device profiles, at least one typical and one stress scenario, and a visual/UX check. I record where the result applies and when the system should be reprofiled.
- **Common Weak Answer:** "It got faster in the test map."
- **Follow-up Questions:** other tier? visual quality? stress case? residual risk?
- **Hands-on Verification Task:** Apply one render optimisation to two different maps and compare.
- **Sources:** [SRC-PERF-007], [SRC-PERF-008], [SRC-RENDER-019]
- **Version Notes:** Scenario coverage is product-specific.

### Question: What is a good source-control policy for generated HLOD and PCG output?

- **Category:** Pipeline / Generated Content
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Decide whether output is committed, archived or regenerated, then enforce freshness and reviewability.
- **Strong 3-Year-Engineer Answer:** Generated output can be authored asset, derived build artifact or runtime-generated state. HLOD may need committed/reviewed proxies in some pipelines; PCG may regenerate from seeds. The policy must support clean builds, diffs, rollback, package inclusion and stale-output detection.
- **Common Weak Answer:** "Generated files should not be committed."
- **Follow-up Questions:** clean agent? visual review? archive? stale?
- **Hands-on Verification Task:** Classify HLOD, PCG actors, DDC and cooked packages into commit/archive/ignore buckets.
- **Sources:** [SRC-WORLD-003], [SRC-PCG-001], [SRC-BUILD-015]
- **Version Notes:** Pipeline policy is studio-specific.

### Question: How do you debug "works after opening editor once"?

- **Category:** Build / Packaged Debugging
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Suspect generated files, local caches, cooked content, editor-loaded assets, source-control omissions or environment dependencies.
- **Strong 3-Year-Engineer Answer:** The editor may generate assets, warm DDC, load loose content, fix redirectors or provide environment paths. I reproduce from a clean workspace/agent without editor side effects, inspect source-control changes and compare cook/stage logs. The fix becomes a clean-build or validation gate.
- **Common Weak Answer:** "Open editor before building."
- **Follow-up Questions:** DDC? generated asset? redirector? local DLL?
- **Hands-on Verification Task:** Create an editor-generated asset dependency and catch it on clean package.
- **Sources:** [SRC-BUILD-017], [SRC-ASSET-008], [SRC-WORLD-002]
- **Version Notes:** Local environment issues are project-specific.

### Question: How do you treat redirectors in a release pipeline?

- **Category:** Assets / Source Control
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Fix and validate redirectors deliberately so moved assets do not hide stale references or cook/package surprises.
- **Strong 3-Year-Engineer Answer:** Redirectors are useful during editing, but a release pipeline should validate/fix them under source control and confirm references resolve in clean cook/package. Bulk moves require review because redirector cleanup can touch many assets.
- **Common Weak Answer:** "Ignore redirectors unless package fails."
- **Follow-up Questions:** clean workspace? source control? soft paths? package?
- **Hands-on Verification Task:** Move an asset, inspect redirector, fix it and verify clean package.
- **Sources:** [SRC-ASSET-008], [SRC-BUILD-009], [SRC-BUILD-017]
- **Version Notes:** Editor tooling may vary.

### Question: How do you know a system design answer is too abstract?

- **Category:** System Design / Interview
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It lacks ownership, lifecycle, failure modes, data flow, authority, performance and verification.
- **Strong 3-Year-Engineer Answer:** I make design answers concrete by naming data structures, owner objects, network audience, save strategy, update path, debug/profiling hooks and first tests. Patterns are useful only if they reduce real complexity and can be operated by a team.
- **Common Weak Answer:** "Use components, events and data-driven design."
- **Follow-up Questions:** who owns state? how debug? how profile? what fails?
- **Hands-on Verification Task:** Rewrite a vague inventory architecture into ownership/data-flow/testing sections.
- **Sources:** [SRC-ARCH-014], [SRC-ARCH-001], [SRC-PERF-003]
- **Version Notes:** General interview skill.

### Question: What is a good "3-year engineer" depth for Unreal internals?

- **Category:** Interview Meta / Depth
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Know the mental model, common failures, debugging route and when exact engine-source confirmation is needed.
- **Strong 3-Year-Engineer Answer:** I do not need to recite every engine private function, but I should know enough to avoid wrong architecture and diagnose common failures. For branch-sensitive systems like Iris, NetworkPrediction, RDG or World Partition builders, I state the concept and then verify exact API/source before implementation.
- **Common Weak Answer:** "I know the docs."
- **Follow-up Questions:** source-sensitive? debug workflow? common bug? proof?
- **Hands-on Verification Task:** Pick one branch-sensitive topic and write concept versus target-source proof notes.
- **Sources:** [SRC-NET-020], [SRC-RENDER-019], [SRC-WORLD-003]
- **Version Notes:** Depth expectations depend on role.

### Question: How do you answer when you are uncertain about an Unreal API?

- **Category:** Interview Meta / Accuracy
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** State the concept confidently, mark exact names/version as uncertain, and describe how you would verify.
- **Strong 3-Year-Engineer Answer:** For version-sensitive APIs I avoid inventing function names. I say the design shape, the source/document I would check, the compile or runtime proof I would run, and the risk if wrong. That is better than bluffing and shows production judgement.
- **Common Weak Answer:** "I think the function is called X."
- **Follow-up Questions:** source? compile proof? version? fallback?
- **Hands-on Verification Task:** Take one schematic code sample and annotate which signatures need target verification.
- **Sources:** [SRC-BUILD-017], [SRC-NET-020], [SRC-AI-016]
- **Version Notes:** Particularly important for UE5.3-UE5.6 differences.

### Question: How do you create a useful debugging log without spamming?

- **Category:** Debugging / Observability
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Log structured scenario/state transitions, IDs, authority, timing and errors at boundaries, with categories/verbosity and artifact retention.
- **Strong 3-Year-Engineer Answer:** Logs should answer "what state changed, for whom, when and why". I include build/scenario IDs, object IDs, net mode, owner, relevant state and error codes. High-frequency data belongs in counters/CSV/traces, not unbounded log spam.
- **Common Weak Answer:** "Add more UE_LOGs."
- **Follow-up Questions:** category? verbosity? ID? artifact?
- **Hands-on Verification Task:** Add structured logs to a fast-travel readiness scenario.
- **Sources:** [SRC-PERF-003], [SRC-NET-016], [SRC-BUILD-017]
- **Version Notes:** Logging categories are project-specific.

### Question: How do you build a regression guard from a bug fix?

- **Category:** Debugging / Regression
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Convert the root cause into the smallest reliable test, validation rule, automation scenario or performance threshold.
- **Strong 3-Year-Engineer Answer:** After a fix I ask what would have caught this earlier: unit test, automation map, commandlet validator, cook gate, CSV threshold, network latency test or content rule. The guard should fail for the original bug and avoid broad false positives.
- **Common Weak Answer:** "Tell the team to be careful."
- **Follow-up Questions:** smallest guard? false positive? owner? artifact?
- **Hands-on Verification Task:** Turn one workbook injected failure into a CI/pre-submit check.
- **Sources:** [SRC-PERF-011], [SRC-BUILD-009], [SRC-BUILD-017]
- **Version Notes:** Automation support varies.

### Question: How do you evaluate if an editor tool is safe?

- **Category:** Editor Tools / Safety
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It supports dry-run, source-control awareness, validation, transactions/undo where appropriate, logging and recovery from partial failure.
- **Strong 3-Year-Engineer Answer:** Batch tools can damage many assets quickly. I design them with preview, ownership filters, backups/source-control checkout, per-asset result logs, validation before/after and idempotent retry. UI polish is secondary to safe mutation.
- **Common Weak Answer:** "It worked on my selected assets."
- **Follow-up Questions:** dry run? undo? partial failure? CI?
- **Hands-on Verification Task:** Add dry-run and per-asset report to a batch rename/migration tool.
- **Sources:** [SRC-BUILD-008], [SRC-BUILD-009], [SRC-BUILD-011]
- **Version Notes:** Editor APIs are branch-sensitive.

### Question: How do you choose commandlet versus Editor Utility Widget?

- **Category:** Editor Tools / Automation
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Commandlets fit headless repeatable automation; EUWs fit interactive editor workflows.
- **Strong 3-Year-Engineer Answer:** If the task must run in CI, validate assets or process batches without UI, I prefer a commandlet or automation command. If a designer needs guided selection, preview and manual choices, an Editor Utility Widget may fit. I often share core logic underneath both.
- **Common Weak Answer:** "Commandlets are for programmers, EUWs for designers."
- **Follow-up Questions:** headless? dry-run? source control? shared core?
- **Hands-on Verification Task:** Build a validation core called by both an EUW and commandlet.
- **Sources:** [SRC-BUILD-008], [SRC-BUILD-011], [SRC-BUILD-012]
- **Version Notes:** Tool APIs vary.

### Question: How do you explain UHT versus UBT?

- **Category:** Build / Foundations
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** UHT parses reflected declarations and generates metadata/glue; UBT configures/builds targets/modules and invokes toolchains.
- **Strong 3-Year-Engineer Answer:** UHT is part of the reflection/build pipeline for UCLASS/USTRUCT/UFUNCTION/UPROPERTY. UBT resolves targets/modules, dependencies, generated code, compile and link. Errors can come from UHT parsing/specifiers or C++ compiler/linker stages; I classify the stage before fixing.
- **Common Weak Answer:** "Both are Unreal's build tools."
- **Follow-up Questions:** generated header? Build.cs? target? reflection?
- **Hands-on Verification Task:** Create one UHT specifier error and one linker error and classify both.
- **Sources:** [SRC-EPIC-002], [SRC-BUILD-001], [SRC-BUILD-002]
- **Version Notes:** Build pipeline stable; details vary.

### Question: How do you diagnose Blueprint performance without blaming Blueprints broadly?

- **Category:** Blueprint / Performance
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Profile specific Blueprint VM work, tick/bind frequency, allocations, object access and native boundary cost in context.
- **Strong 3-Year-Engineer Answer:** Blueprints are not automatically too slow. I look for high-frequency tick/bind loops, expensive pure functions, dynamic casts, asset loads, repeated allocations and unbatched work. Then I move hot inner loops or stable services to C++ only when measurement supports it.
- **Common Weak Answer:** "Rewrite Blueprints in C++."
- **Follow-up Questions:** profiler? tick? pure binding? native boundary?
- **Hands-on Verification Task:** Profile a Blueprint-heavy UI and move only the measured hot path.
- **Sources:** [SRC-EPIC-031], [SRC-PERF-003], [SRC-PERF-005]
- **Version Notes:** Blueprint tooling/performance varies by version.

### Question: How do you design a deterministic replayable test scenario?

- **Category:** Testing / Automation
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Fix seed, map, initial state, input path, timing boundaries, metrics and expected outputs.
- **Strong 3-Year-Engineer Answer:** Reproducible tests need controlled random seeds, stable scenario data, readiness markers and output comparison. For gameplay/performance, I log scenario start/end, entity counts and target settings. For network tests, I add latency/loss settings and deterministic command scripts.
- **Common Weak Answer:** "Record a video of the bug."
- **Follow-up Questions:** seed? readiness? input? net emulation?
- **Hands-on Verification Task:** Create a replayable combat peak scenario with fixed spawn seed.
- **Sources:** [SRC-PERF-011], [SRC-PERF-009], [SRC-NET-016]
- **Version Notes:** Automation features are branch-sensitive.

### Question: What is the difference between correctness and presentation in prediction?

- **Category:** Networking / Prediction
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Correctness is authoritative gameplay state; presentation is local responsiveness that may be corrected.
- **Strong 3-Year-Engineer Answer:** Prediction improves feel by showing likely outcome early, but the server still owns gameplay truth. I separate predicted animation/VFX/movement from authoritative damage, inventory and cooldown. Reconciliation should repair state without duplicating side effects.
- **Common Weak Answer:** "Prediction means client does it first."
- **Follow-up Questions:** correction? side effects? server rejection? cues?
- **Hands-on Verification Task:** Predict a dash visual locally and reject server-side cooldown.
- **Sources:** [SRC-NET-009], [SRC-GAS-010], [SRC-NET-020]
- **Version Notes:** Prediction mechanisms differ by system.

### Question: How do you handle late join in a stateful multiplayer system?

- **Category:** Networking / State
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Represent durable current state in replicated/snapshot systems, not only in transient RPC history.
- **Strong 3-Year-Engineer Answer:** Late joiners need world, player, inventory, objective and ability state reconstructed from authoritative server state. I avoid relying on old multicast events. Replicated properties, Fast Arrays, initial state RPCs or custom snapshots should establish current truth when relevance/connection begins.
- **Common Weak Answer:** "Replay the events."
- **Follow-up Questions:** dormancy? Fast Array? GameState? snapshot?
- **Hands-on Verification Task:** Join late after a door opened and inventory changed; prove correct state.
- **Sources:** [SRC-NET-004], [SRC-NET-006], [SRC-NET-015]
- **Version Notes:** Initial replication behaviour is branch-sensitive.

### Question: How do you decide if a replicated property should be owner-only?

- **Category:** Networking / Privacy and Bandwidth
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Owner-only fits private client state; public gameplay state belongs on broader replication channels.
- **Strong 3-Year-Engineer Answer:** I classify state by audience: public, team, owner, server-only or cosmetic. Owner-only can save bandwidth and protect hidden information, but other clients and late joiners will not see it. UI convenience is not a reason to expose private data globally.
- **Common Weak Answer:** "Owner-only is for inventory."
- **Follow-up Questions:** spectators? team? cheating? UI?
- **Hands-on Verification Task:** Split ammo reserve owner-only from visible weapon state public.
- **Sources:** [SRC-NET-003], [SRC-NET-004], [SRC-NET-007]
- **Version Notes:** Conditions and audience policy are project-sensitive.

### Question: How do you debug replicated references that are unresolved on clients?

- **Category:** Networking / Object References
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Check spawn/replication order, relevancy, ownership, NetGUID mapping, dormancy and whether the referenced object is replicated/cooked/loaded.
- **Strong 3-Year-Engineer Answer:** A replicated pointer is only useful if the client can resolve the referenced object. I inspect actor/subobject replication, creation timing, relevancy, dormancy, unloaded streaming cells, and network debug CVars for unmapped references. Sometimes stable IDs are better than direct pointers.
- **Common Weak Answer:** "Replicate the pointer."
- **Follow-up Questions:** NetGUID? subobject? streaming? late join?
- **Hands-on Verification Task:** Replicate a reference to a dynamically spawned actor before it is relevant and fix it.
- **Sources:** [SRC-NET-004], [SRC-NET-012], [SRC-NET-016]
- **Version Notes:** Debug CVars and replication internals are version-sensitive.

### Question: How do you design combat hit validation?

- **Category:** Gameplay / Networking
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Validate client intent on the server using timing, range, geometry, ability state and anti-cheat tolerance.
- **Strong 3-Year-Engineer Answer:** A responsive client can predict swing traces or feedback, but the server validates montage/ability state, target, range, collision/history and cooldown. I include latency tolerance, replay windows if needed, and rejection feedback. Damage application remains authoritative and idempotent.
- **Common Weak Answer:** "Client sends hit target to server."
- **Follow-up Questions:** lag compensation? montage? cooldown? duplicate hit?
- **Hands-on Verification Task:** Implement melee hit request with server-side range/timing validation.
- **Sources:** [SRC-NET-001], [SRC-PHYS-003], [SRC-GAS-010]
- **Version Notes:** Lag compensation design is project-specific.

### Question: How do you explain deferred spawning?

- **Category:** Gameplay Framework / Spawning
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It lets code configure spawned actor properties/components before construction completion/BeginPlay-style runtime use.
- **Strong 3-Year-Engineer Answer:** Deferred spawning is useful when initial data affects construction or component setup. I use it to avoid "spawn then immediately mutate" races, especially for replicated actors or construction-sensitive setup. I still respect authority, ownership and lifecycle ordering.
- **Common Weak Answer:** "It delays spawn."
- **Follow-up Questions:** construction? replication? exposed-on-spawn? BeginPlay?
- **Hands-on Verification Task:** Spawn a projectile with owner/team/damage configured before activation.
- **Sources:** [SRC-EPIC-011], [SRC-EPIC-015]
- **Version Notes:** Exact lifecycle ordering is branch-sensitive.

### Question: How do you select a subsystem host?

- **Category:** UE Architecture / Subsystems
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Match lifetime and scope: engine, editor, game instance, world or local player.
- **Strong 3-Year-Engineer Answer:** World-specific gameplay services belong in WorldSubsystem or actors, player-specific UI/input services in LocalPlayerSubsystem, persistent session services in GameInstanceSubsystem and editor-only tooling in EditorSubsystem. The wrong host creates multiplayer, travel and teardown bugs.
- **Common Weak Answer:** "GameInstanceSubsystem lasts longest."
- **Follow-up Questions:** PIE worlds? split screen? travel? editor?
- **Hands-on Verification Task:** Move a per-local-player input coordinator out of GameInstanceSubsystem.
- **Sources:** [SRC-EPIC-017], [SRC-INPUT-004], [SRC-EPIC-018]
- **Version Notes:** Subsystem API stable conceptually.

### Question: How do Enhanced Input mapping contexts fail across UI/gameplay?

- **Category:** Input / Enhanced Input
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Context priority/lifetime can leave stale gameplay actions active or block UI actions after mode changes.
- **Strong 3-Year-Engineer Answer:** Mapping contexts should be owned by a local-player input coordinator with clear push/pop rules for gameplay, vehicle, menu and modal UI. I test possession, respawn, split-screen/local player and CommonUI transitions. Server receives intent, not raw UI state.
- **Common Weak Answer:** "Add all contexts on BeginPlay."
- **Follow-up Questions:** priority? local player? modal? respawn?
- **Hands-on Verification Task:** Open a menu and prove fire/jump actions are blocked while UI navigation works.
- **Sources:** [SRC-INPUT-001], [SRC-INPUT-003], [SRC-INPUT-004], [SRC-UI-012]
- **Version Notes:** Enhanced Input is plugin/project-settings sensitive.

### Question: How do Enhanced Input triggers and modifiers affect network design?

- **Category:** Input / Networking
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** They shape local semantic intent before networking; the server should validate gameplay actions rather than trust raw input meaning.
- **Strong 3-Year-Engineer Answer:** Modifiers/triggers are great for local interpretation like hold, chord, dead zone or tap. Network code should send validated intent such as "request dash" with timestamp/context, not arbitrary client authority. I test remapping and device differences so action semantics remain consistent.
- **Common Weak Answer:** "Replicate Input Actions."
- **Follow-up Questions:** hold? remap? validation? cooldown?
- **Hands-on Verification Task:** Implement hold-to-charge locally and server-validated release.
- **Sources:** [SRC-INPUT-001], [SRC-INPUT-002], [SRC-NET-006]
- **Version Notes:** Input plugin APIs vary.

### Question: How do you design accessibility without treating it as polish only?

- **Category:** UI / Accessibility
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Preserve readability, navigation, feedback and control options across device profiles, localisation and input modes.
- **Strong 3-Year-Engineer Answer:** Accessibility intersects UI layout, input, audio/visual feedback, colour contrast, safe zones, font scaling and gameplay readability. I test pseudo-localisation, controller navigation, low settings and platform constraints. Performance settings must not remove critical feedback.
- **Common Weak Answer:** "Add subtitles later."
- **Follow-up Questions:** safe zones? contrast? input remap? low tier?
- **Hands-on Verification Task:** Run pseudo-localised UI on low scalability and gamepad-only input.
- **Sources:** [SRC-UI-013], [SRC-UI-014], [SRC-PERF-008]
- **Version Notes:** Platform accessibility requirements vary.

### Question: How do you classify "too many actors"?

- **Category:** Performance / Gameplay Framework
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Separate ticking, components, rendering primitives, physics/collision, replication, GC and construction/registration costs.
- **Strong 3-Year-Engineer Answer:** Actor count alone is not the root cause. I measure tick cost, component counts, render primitives/materials, collision bodies, replication channels, UObject count/GC and spawn/destruction churn. The fix could be pooling, batching, instancing, Mass, HLOD or simpler data.
- **Common Weak Answer:** "Actors are expensive."
- **Follow-up Questions:** tick? components? render? replication? GC?
- **Hands-on Verification Task:** Spawn 10k simple actors and add costs one axis at a time.
- **Sources:** [SRC-EPIC-011], [SRC-PERF-003], [SRC-MASS-001], [SRC-RENDER-015]
- **Version Notes:** Costs are workload/branch/platform dependent.

### Question: How do you debug GC hitches?

- **Category:** UObject / Profiling
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Inspect object count, reference graph, churn, roots, destruction work and GC timing rather than only changing GC settings.
- **Strong 3-Year-Engineer Answer:** GC hitches usually come from too many objects, too much churn, large reference graphs, destruction callbacks or bad lifetime policy. I profile GC timing/object counts, identify churn sources and reduce UObject allocation or retention before tuning collection intervals.
- **Common Weak Answer:** "Run GC less often."
- **Follow-up Questions:** object count? churn? roots? incremental GC?
- **Hands-on Verification Task:** Create UObject churn and compare pooling/value-type redesign.
- **Sources:** [SRC-EPIC-005], [SRC-PERF-003], [SRC-EPIC-009]
- **Version Notes:** Incremental GC is version/experimental sensitive.

### Question: How do you use Unreal Insights for task contention?

- **Category:** Profiling / Tasks
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Inspect task launch, prerequisites, worker occupancy, waits, locks and game-thread merge points.
- **Strong 3-Year-Engineer Answer:** A wait scope may show where the game thread blocked, not the root work source. I follow the dependency chain, worker task duration, scheduling granularity, lock contention and merge/copy-back cost. Then I adjust partitioning or ownership.
- **Common Weak Answer:** "The wait is the expensive code."
- **Follow-up Questions:** prerequisite? worker idle? lock? merge?
- **Hands-on Verification Task:** Profile a ParallelFor workload with too-small chunks and a locked output vector.
- **Sources:** [SRC-CPP-026], [SRC-PERF-003], [SRC-CPP-021]
- **Version Notes:** Insights task tracks vary by branch.

### Question: How do you decide between pooling and allocation reduction?

- **Category:** Performance / Memory
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Pool only when reuse is safe and measured allocation/destruction churn is meaningful; otherwise reduce work or use value/batch data.
- **Strong 3-Year-Engineer Answer:** Pooling adds lifetime, reset and stale-state risk. I first prove allocation/destruction churn or spawn cost is the bottleneck. Then I define reset invariants, owner teardown, memory cap and profiling evidence. Sometimes SoA/value arrays or fewer objects are better.
- **Common Weak Answer:** "Pool expensive objects."
- **Follow-up Questions:** reset? memory cap? owner destroyed? stale state?
- **Hands-on Verification Task:** Pool projectiles and deliberately test stale damage/team/owner fields.
- **Sources:** [SRC-PERF-003], [SRC-ARCH-014], [SRC-CPP-024]
- **Version Notes:** Allocation cost and pooling APIs vary.

### Question: How do you design a performance budget for a feature?

- **Category:** Performance / Budgeting
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Allocate CPU/GPU/memory/loading/network budgets per scenario/tier and define measurement and owner.
- **Strong 3-Year-Engineer Answer:** A budget needs target platform, scenario, peak counts, quality tier and metric thresholds. I decide whether the feature owns Game, Draw/RHI, GPU pass, memory, package size, bandwidth or load time. Then I add instrumentation so the budget can be enforced over time.
- **Common Weak Answer:** "Keep it under 60 FPS."
- **Follow-up Questions:** tier? memory? hitches? owner? tool?
- **Hands-on Verification Task:** Budget an inventory UI, VFX burst or AI crowd feature.
- **Sources:** [SRC-PERF-001], [SRC-PERF-009], [SRC-PERF-010]
- **Version Notes:** Budgets are product/platform specific.

### Question: How do you handle platform-specific feature fallback?

- **Category:** Platform / Scalability
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Define capability detection, fallback content/settings, gameplay readability and target proof per platform tier.
- **Strong 3-Year-Engineer Answer:** Platform fallback is not "turn it off". I decide what visual/gameplay signal must remain, then provide alternative materials, lighting, input, UI or simulation scale. I prove active fallback in packaged builds with screenshots/metrics and avoid changing gameplay rules by graphics tier.
- **Common Weak Answer:** "Lower quality on weak platforms."
- **Follow-up Questions:** readability? feature support? content variant? gameplay parity?
- **Hands-on Verification Task:** Design fallback for expensive shadows or VFX on low mobile tier.
- **Sources:** [SRC-PLAT-005], [SRC-PLAT-006], [SRC-PERF-008]
- **Version Notes:** Platform feature support changes.

### Question: How do you debug package-size growth?

- **Category:** Build / Assets
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Compare cooked/staged manifests, asset dependencies, chunks/containers, maps, shader/PSO data and platform files by build ID.
- **Strong 3-Year-Engineer Answer:** Package size growth is an artifact problem, not a guess. I diff manifests, find new roots or dependency chains, inspect textures/audio/shaders/plugins and check accidental cook-all or duplicate content. The fix should be an ownership/cook rule change, content budget or asset optimisation.
- **Common Weak Answer:** "Compress it more."
- **Follow-up Questions:** manifest diff? dependency root? plugin? shader library?
- **Hands-on Verification Task:** Add one large soft-referenced asset and trace why it entered the package.
- **Sources:** [SRC-ASSET-006], [SRC-BUILD-017], [SRC-RENDER-013]
- **Version Notes:** Container formats/platform packaging vary.

### Question: How do you prove a loading-screen improvement?

- **Category:** Loading / Performance
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Measure cold and warm first-interactive time, loading phases, I/O, asset dependency changes, memory and user-visible readiness.
- **Strong 3-Year-Engineer Answer:** I define first interactive, not just map-open complete. Then I capture cold/warm package runs, load traces, asset lists, async loading, shader/PSO first-use and memory peaks. A loading screen that ends before collision/UI/gameplay readiness is a bug.
- **Common Weak Answer:** "The bar finishes faster."
- **Follow-up Questions:** cold cache? readiness? async? first interactive?
- **Hands-on Verification Task:** Add markers for load start, map loaded, playable ready and compare before/after.
- **Sources:** [SRC-ASSET-006], [SRC-PERF-003], [SRC-PERF-009]
- **Version Notes:** Loading tool channels vary by branch.

### Question: What is a good "source-sensitive" caveat?

- **Category:** Interview Meta / Source Discipline
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It names which part is stable concept, which part is branch/API sensitive, and how to verify implementation.
- **Strong 3-Year-Engineer Answer:** I might say: "The design shape is stable, but exact `UAISense` signatures and NetworkPrediction model hooks need target branch source confirmation." That is useful because it preserves the concept without pretending public docs are a drop-in implementation guide.
- **Common Weak Answer:** "It depends."
- **Follow-up Questions:** what depends? source? compile proof? fallback?
- **Hands-on Verification Task:** Add caveats to three schematic code snippets in the curriculum.
- **Sources:** [SRC-AI-016], [SRC-NET-020], [SRC-WORLD-003]
- **Version Notes:** Applies heavily to UE5.3-UE5.6 specialist systems.

### Question: How do you make a handoff report actionable?

- **Category:** Team Workflow / Debugging
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Include symptom, reproduction, artifact identity, evidence, root-cause classification, owner, fix/guard and residual risk.
- **Strong 3-Year-Engineer Answer:** A report should let the next engineer act without replaying your investigation. I link logs/traces/CSV/package, identify first failing phase or scenario window, explain what was ruled out and assign an owner with the smallest recurrence guard.
- **Common Weak Answer:** "It crashes sometimes; see logs."
- **Follow-up Questions:** first cause? artifact? owner? guard?
- **Hands-on Verification Task:** Rewrite a vague CI failure note into an actionable handoff.
- **Sources:** [SRC-BUILD-017], [SRC-PERF-003], [SRC-PERF-011]
- **Version Notes:** Team conventions vary.

### Question: How do you design a rollback-safe gameplay cue key?

- **Category:** Networking / Rollback
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Use a deterministic event identity tied to simulation tick, source, predicted action and effect type so replay does not duplicate presentation.
- **Strong 3-Year-Engineer Answer:** Rollback/resimulation can execute the same logical event multiple times. A cue key lets presentation decide whether it is new, replayed or corrected. I include enough state to dedupe without suppressing legitimate repeated events, and I keep gameplay authority outside the cue.
- **Common Weak Answer:** "Do not play cues during rollback."
- **Follow-up Questions:** tick? source ID? repeated dash? correction?
- **Hands-on Verification Task:** Force two corrections and prove one dash trail cue per logical dash.
- **Sources:** [SRC-NET-020], [SRC-GAS-008], [SRC-ARCH-014]
- **Version Notes:** Cue systems are plugin/branch sensitive.

### Question: How would you test NetworkPrediction smoothing?

- **Category:** Networking / NetworkPrediction
- **Priority:** P3
- **Expected Depth:** D4
- **Short Answer:** Compare visual interpolation/smoothing against authoritative state under forced corrections, packet loss and latency.
- **Strong 3-Year-Engineer Answer:** I run a deterministic movement scenario with controlled net emulation, force reconcile events and record raw authoritative state versus smoothed presentation. Good smoothing hides correction visually without masking gameplay truth or adding unacceptable input delay.
- **Common Weak Answer:** "If it looks smooth, it works."
- **Follow-up Questions:** correction? input latency? cue timing? authority?
- **Hands-on Verification Task:** Toggle smoothing on/off for a rollback dash and capture before/after video/logs.
- **Sources:** [SRC-NET-020], [SRC-NET-009], [SRC-NET-016]
- **Version Notes:** Exact smoothing services require target source.

### Question: What makes an automation threshold maintainable?

- **Category:** Automation / Performance Gates
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** It is tied to scenario, metric, owner, tolerance, baseline policy and artifact evidence.
- **Strong 3-Year-Engineer Answer:** A maintainable threshold is not a magic number. It has a reason, target device/profile, enough repetitions, false-positive policy and owner. It should catch meaningful regressions without punishing unrelated noise or encouraging teams to bypass the gate.
- **Common Weak Answer:** "Set a hard FPS threshold."
- **Follow-up Questions:** rolling baseline? tolerance? owner? reporting-only?
- **Hands-on Verification Task:** Convert a noisy CSV metric into warning/blocking levels.
- **Sources:** [SRC-PERF-009], [SRC-PERF-011]
- **Version Notes:** CI policy is project-specific.

### Question: How do you prove a build graph did not use stale artifacts?

- **Category:** Build / BuildGraph
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Bind every artifact to build ID/input revision and make downstream nodes consume declared outputs, not ambient paths.
- **Strong 3-Year-Engineer Answer:** I include build ID in artifact paths, clean outputs or validate freshness, and record producer node metadata. Downstream package/performance/crash nodes must depend on the producing node. Otherwise a green graph may test yesterday's package or symbols.
- **Common Weak Answer:** "The node ran successfully."
- **Follow-up Questions:** output path? build ID? dependency? clean agent?
- **Hands-on Verification Task:** Intentionally point a perf node at an old package and add a build-ID guard.
- **Sources:** [SRC-BUILD-015], [SRC-BUILD-017], [SRC-BUILD-018]
- **Version Notes:** Artifact systems vary by CI.

### Question: How do you handle shader cache and DDC in performance proof?

- **Category:** Rendering / Build Artifacts
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Declare cache state, test cold/warm paths where relevant, and distinguish build-time cache from shipped first-use behaviour.
- **Strong 3-Year-Engineer Answer:** Warm DDC can hide local shader/asset processing, but shipped users care about packaged shader libraries, PSO coverage and first-use state creation. I record cache policy and run at least one cold-cache or clean-package path for first-use hitch features.
- **Common Weak Answer:** "Warm the cache before benchmarking."
- **Follow-up Questions:** DDC? PSO? shader library? shipped user?
- **Hands-on Verification Task:** Compare first-run effect hitch with warm and cold local caches.
- **Sources:** [SRC-ASSET-008], [SRC-RENDER-014], [SRC-BUILD-017]
- **Version Notes:** Cache systems are branch/platform sensitive.

### Question: How do you check World Partition package inclusion?

- **Category:** World Partition / Packaging
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Verify representative maps, external actors, Data Layers, HLOD artifacts and generated content through cook/stage logs and packaged traversal.
- **Strong 3-Year-Engineer Answer:** Editor visibility is not package inclusion. I check cook roots, external actor files, Runtime Data Layer content, HLOD proxies, Level Instances and PCG/generated assets. Then I run a packaged route that exercises those cells and logs readiness.
- **Common Weak Answer:** "The map opens in editor."
- **Follow-up Questions:** external actors? HLOD? Runtime Data Layer? traversal?
- **Hands-on Verification Task:** Omit a generated/HLOD artifact and show how package traversal detects it.
- **Sources:** [SRC-ASSET-010], [SRC-WORLD-003], [SRC-BUILD-017]
- **Version Notes:** Package layout varies by platform.

### Question: How would you interview-answer "we have a one-frame network correction pop"?

- **Category:** Networking / Debugging
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Capture movement prediction data, input/saved moves, server correction, smoothing and gameplay state around the pop.
- **Strong 3-Year-Engineer Answer:** I would reproduce with fixed latency/loss, log client input/saved moves, server authoritative movement, correction size/time and smoothing. Then I check custom movement flags, move combining, non-deterministic client logic and whether a gameplay system changes movement outside prediction.
- **Common Weak Answer:** "Increase smoothing."
- **Follow-up Questions:** saved move? combine? non-determinism? server correction?
- **Hands-on Verification Task:** Inject a client-only speed multiplier and diagnose the correction.
- **Sources:** [SRC-NET-009], [SRC-NET-018], [SRC-NET-019]
- **Version Notes:** Movement internals are branch-sensitive.

### Question: How do you avoid double-applying costs in predicted abilities?

- **Category:** GAS / Prediction
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Use prediction-aware commit paths, authoritative server validation and reconciliation-safe UI/cue handling.
- **Strong 3-Year-Engineer Answer:** Costs/cooldowns should not be blindly applied once locally and again from server without prediction reconciliation. I track prediction keys, ability commit, rejected activation and UI rollback. The server owns truth; the client presents provisional state.
- **Common Weak Answer:** "Apply cost on both sides."
- **Follow-up Questions:** prediction key? rollback? UI? rejection?
- **Hands-on Verification Task:** Force server rejection after local predicted cost and repair UI/state.
- **Sources:** [SRC-GAS-010], [SRC-GAS-006], [SRC-NET-001]
- **Version Notes:** GAS prediction setup is project-sensitive.

### Question: How do you decide if an ability should be instanced?

- **Category:** GAS / Ability Instances
- **Priority:** P3
- **Expected Depth:** D3
- **Short Answer:** Choose based on per-activation state, concurrency, tasks, memory, cancellation and shared/default data.
- **Strong 3-Year-Engineer Answer:** Instancing policy affects where mutable ability state can live and how concurrent activations behave. If an ability uses tasks or per-activation state, instancing may be needed. Stateless abilities can avoid instance overhead, but only if they truly keep no activation-specific state.
- **Common Weak Answer:** "Instance abilities that are complex."
- **Follow-up Questions:** concurrent activation? tasks? mutable fields? memory?
- **Hands-on Verification Task:** Build a charge ability with mutable state and test two simultaneous activations.
- **Sources:** [SRC-GAS-003], [SRC-GAS-009]
- **Version Notes:** Exact policies are GAS/branch sensitive.

### Question: How do you debug a Behaviour Tree that thrashes between branches?

- **Category:** AI / Behaviour Tree
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Inspect Blackboard changes, decorator observers/aborts, service frequency, task result timing and cooldown/hysteresis.
- **Strong 3-Year-Engineer Answer:** Thrashing usually means decision inputs change too often or aborts are too broad. I use BT debugger/Visual Logger, log Blackboard key transitions and add hysteresis/cooldowns or better state ownership. The fix is not always "lower service tick"; it is stabilising decision conditions.
- **Common Weak Answer:** "Observer aborts are bugged."
- **Follow-up Questions:** Blackboard? service frequency? hysteresis? task finish?
- **Hands-on Verification Task:** Create a chase/flee BT that oscillates at range threshold and add hysteresis.
- **Sources:** [SRC-AI-002], [SRC-AI-008], [SRC-AI-009]
- **Version Notes:** BT debugging UI varies by branch.

### Question: How do you budget EQS?

- **Category:** AI / EQS
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Control query frequency, candidate count, test cost, run mode, agent count and caching.
- **Strong 3-Year-Engineer Answer:** EQS cost is multiplicative: agents times candidates times tests times frequency. I profile real query load, use cheap filters early, reduce frequency, cache context where valid and avoid using EQS as a per-frame reflex for every NPC.
- **Common Weak Answer:** "EQS is expensive."
- **Follow-up Questions:** candidate count? test order? run mode? frequency?
- **Hands-on Verification Task:** Run the same query with 10, 100 and 1000 candidates and profile.
- **Sources:** [SRC-AI-005], [SRC-AI-008], [SRC-PERF-003]
- **Version Notes:** EQS tooling/behaviour is project-sensitive.

### Question: How do you decide between RVO and Detour Crowd?

- **Category:** AI / Avoidance
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Compare avoidance needs, NavMesh/path integration, agent count, movement component support and debugging requirements.
- **Strong 3-Year-Engineer Answer:** Avoidance is different from pathfinding. RVO and Detour Crowd have different integration and trade-offs, and Epic guidance warns against using both at once. I test with representative density, bottlenecks and movement constraints.
- **Common Weak Answer:** "Use both for better avoidance."
- **Follow-up Questions:** path following? density? debug? movement component?
- **Hands-on Verification Task:** Build a corridor crowd test with RVO, Detour Crowd and neither.
- **Sources:** [SRC-AI-007], [SRC-AI-006], [SRC-AI-008]
- **Version Notes:** Avoidance setup varies.

### Question: How do you validate a reusable POI in a large world?

- **Category:** World Building / POI
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Test authoring mode, source-control files, runtime identity, Data Layer/HLOD/cook inclusion and packaged traversal.
- **Strong 3-Year-Engineer Answer:** A reusable POI is not just a visual prefab. I decide Level Instance/Packed Level Blueprint/actors, verify OFPA/source-control behaviour, assign Data Layer/HLOD policy, preserve unique gameplay IDs and package it. Then I stream into and out of it in a target build.
- **Common Weak Answer:** "Make a Level Instance."
- **Follow-up Questions:** mutable state? OFPA? HLOD? package?
- **Hands-on Verification Task:** Place one POI twice and verify save IDs and HLOD/cook behaviour.
- **Sources:** [SRC-WORLD-004], [SRC-WORLD-002], [SRC-WORLD-003]
- **Version Notes:** POI workflow is project-specific.

### Question: How do you keep UI async asset loads safe?

- **Category:** UI / Async Assets
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Track request identity, widget/model lifetime and cancellation or stale-result rejection.
- **Strong 3-Year-Engineer Answer:** Async icons/previews can return after a list entry is recycled or widget is destroyed. I store request tokens or expected item IDs, use weak widget/model references, cancel where possible and ignore stale completion. This is especially important in virtualised lists.
- **Common Weak Answer:** "Check IsValid in callback."
- **Follow-up Questions:** recycled entry? request token? cancellation? soft ref?
- **Hands-on Verification Task:** Rapid-scroll a list with async icons and prove no stale icon appears.
- **Sources:** [SRC-UI-005], [SRC-ASSET-002], [SRC-ASSET-004]
- **Version Notes:** Async APIs vary by branch.

### Question: How do you design local-player scoped UI/input in split-screen?

- **Category:** UI / Local Player
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Scope widgets, input contexts and view models to the correct LocalPlayer/PlayerController.
- **Strong 3-Year-Engineer Answer:** Split-screen breaks global UI assumptions. I store local UI/input services in LocalPlayer scope, avoid global singleton focus state, and test each player's menus, prompts, remapping and owner-only replicated data independently.
- **Common Weak Answer:** "Use GetPlayerController(0)."
- **Follow-up Questions:** LocalPlayerSubsystem? focus? owner-only? input context?
- **Hands-on Verification Task:** Open inventory for player 2 without changing player 1 focus/input.
- **Sources:** [SRC-UI-001], [SRC-INPUT-004], [SRC-NET-003]
- **Version Notes:** Local multiplayer setup is project-specific.

### Question: How do you debug a Physics Asset ragdoll recovery issue?

- **Category:** Physics / Animation
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Inspect physics bodies/constraints, collision profiles, pose handoff, authority, sleep state and animation blend recovery.
- **Strong 3-Year-Engineer Answer:** Ragdoll recovery sits across animation, physics and gameplay. I verify Physics Asset bodies, constraint frames/drives, collision responses, server authority, component transform, blend-to-animation and gameplay state. Networked ragdoll needs explicit audience/authority policy.
- **Common Weak Answer:** "Tune the ragdoll asset."
- **Follow-up Questions:** constraint frame? collision? server? blend?
- **Hands-on Verification Task:** Knock down a character, recover to animation and test under dedicated server.
- **Sources:** [SRC-PHYS-007], [SRC-ANIM-010], [SRC-NET-001]
- **Version Notes:** Chaos/animation behaviour varies by branch.

### Question: How do you analyse a constraint instability bug?

- **Category:** Physics / Constraints
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Check mass ratios, frames, limits, drives, damping, timestep/substeps, collision and solver cost.
- **Strong 3-Year-Engineer Answer:** Constraints fail when physical setup is unrealistic or under-resolved. I visualise frames, test simple shapes, tune mass/damping/limits, consider substepping/CCD where appropriate and profile physics time. The fix is rarely one magic stiffness number.
- **Common Weak Answer:** "Increase solver iterations."
- **Follow-up Questions:** frame? mass ratio? substep? collision?
- **Hands-on Verification Task:** Build a swinging door/chain and tune from unstable to stable with evidence.
- **Sources:** [SRC-PHYS-006], [SRC-PHYS-005], [SRC-PERF-003]
- **Version Notes:** Chaos solver details vary.

### Question: How do you decide whether a render target should update every frame?

- **Category:** Rendering / Render Targets
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Tie cadence to visual requirement and measure memory/bandwidth/view cost against alternatives.
- **Strong 3-Year-Engineer Answer:** Every-frame render targets are expensive when they imply extra views, resolves, mips or UI copies. I ask whether the content changes, whether event/manual cadence suffices and whether lower resolution/format/show flags preserve quality. Then I profile Draw/GPU and memory.
- **Common Weak Answer:** "It looks smoother every frame."
- **Follow-up Questions:** manual update? format? scene capture? mips?
- **Hands-on Verification Task:** Change an inventory preview from every-frame to on-demand and compare.
- **Sources:** [SRC-RENDER-016], [SRC-RENDER-017], [SRC-PERF-003]
- **Version Notes:** RHI/platform costs vary.

### Question: How do you debug a material that is cheap in shader complexity but expensive in game?

- **Category:** Rendering / Materials
- **Priority:** P2
- **Expected Depth:** D4
- **Short Answer:** Check pass participation, overdraw, shadows, WPO, texture bandwidth, permutations, render targets and scene coverage.
- **Strong 3-Year-Engineer Answer:** Shader complexity view is one clue, not full truth. A material may affect depth, base, shadow, velocity, translucency or custom depth and be used across large screen area or many instances. I use GPU capture/pass timing and material usage context.
- **Common Weak Answer:** "Shader complexity is green."
- **Follow-up Questions:** overdraw? shadow pass? WPO? screen coverage?
- **Hands-on Verification Task:** Compare an opaque and translucent material with equal node count but different overdraw.
- **Sources:** [SRC-RENDER-004], [SRC-RENDER-005], [SRC-RENDER-019]
- **Version Notes:** Visualisers and pass costs are RHI-specific.

### Question: How do you prove a content LOD change did not break gameplay readability?

- **Category:** Rendering / Scalability
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Compare visual/gameplay cues, target device profiles, performance metrics and accessibility/readability constraints.
- **Strong 3-Year-Engineer Answer:** Performance gains are invalid if players lose enemy silhouettes, interactable cues or UI readability. I capture before/after on low/high tiers, include gameplay-critical views and record frame/memory savings. Quality sign-off is part of the proof.
- **Common Weak Answer:** "LOD saves performance."
- **Follow-up Questions:** silhouette? cues? low tier? accessibility?
- **Hands-on Verification Task:** Reduce enemy material/LOD quality and run a readability checklist plus CSV.
- **Sources:** [SRC-PERF-008], [SRC-RENDER-008], [SRC-UI-013]
- **Version Notes:** Product readability standards vary.

### Question: How do you design a debug overlay without harming performance?

- **Category:** Tools / Runtime Debugging
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Gate it by build/config/CVar, batch data, limit drawing/text, and measure overhead.
- **Strong 3-Year-Engineer Answer:** Debug overlays can add text layout, line drawing, UObject lookups and allocations. I expose targeted categories, sample rates and distance filters, and make sure Shipping policy is respected. For high-volume data, logs/CSV/Insights may be better.
- **Common Weak Answer:** "Only developers enable it."
- **Follow-up Questions:** CVar? allocation? shipping? distance filter?
- **Hands-on Verification Task:** Add an AI/perception overlay and measure with 100 agents.
- **Sources:** [SRC-AI-008], [SRC-PERF-003], [SRC-BUILD-007]
- **Version Notes:** Debug draw APIs vary.

### Question: How do you handle privacy in crash/performance artifacts?

- **Category:** Build / Operations
- **Priority:** P2
- **Expected Depth:** D3
- **Short Answer:** Retain enough diagnostic context while redacting or avoiding personal/sensitive data according to platform/studio policy.
- **Strong 3-Year-Engineer Answer:** Crash and telemetry artifacts can include user paths, account IDs, chat, device data or internal filenames. I define approved context keys, retention, access control and redaction. Diagnostic value does not override privacy and platform requirements.
- **Common Weak Answer:** "More logs are better."
- **Follow-up Questions:** PII? retention? platform policy? access?
- **Hands-on Verification Task:** Review a crash context payload and remove unsafe fields while preserving triage value.
- **Sources:** [SRC-BUILD-018], [SRC-PLAT-024]
- **Version Notes:** Policies vary by studio/platform.

### Question: How do you explain a trade-off you chose under production pressure?

- **Category:** Interview Meta / Judgement
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Name the constraint, options, evidence, chosen compromise, risk and follow-up guard.
- **Strong 3-Year-Engineer Answer:** I avoid heroic narratives. I explain the deadline/platform/quality constraint, what options I evaluated, what evidence supported the decision, what risk remained and what monitoring/test was added. This shows judgement and accountability.
- **Common Weak Answer:** "We picked the faster fix."
- **Follow-up Questions:** alternatives? evidence? risk? follow-up?
- **Hands-on Verification Task:** Write a production decision memo for a performance or networking compromise.
- **Sources:** [SRC-ARCH-014], [SRC-PERF-003], [SRC-BUILD-017]
- **Version Notes:** General interview skill.

### Question: How do you prepare a five-minute specialist deep-dive answer?

- **Category:** Interview Meta / Communication
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Use problem, constraints, design, implementation, debugging, measurement, failure case and lesson.
- **Strong 3-Year-Engineer Answer:** I choose one system and tell it with evidence. I include why the system existed, the architecture, one hard bug, one measurement or proof packet, and what I would improve. That beats listing many technologies without depth.
- **Common Weak Answer:** "Describe every feature I touched."
- **Follow-up Questions:** evidence? failure? trade-off? lesson?
- **Hands-on Verification Task:** Prepare a five-minute answer from the unified target-proof packet.
- **Sources:** [SRC-PERF-009], [SRC-BUILD-015], [SRC-NET-020]
- **Version Notes:** Tailor depth to role.

### Question: How do you answer "what would you check in engine source?"

- **Category:** Interview Meta / Engine Source
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Identify the branch-sensitive seam: virtual hooks, replication path, builder commandlet, plugin API or renderer/task internals.
- **Strong 3-Year-Engineer Answer:** I would not browse source aimlessly. For saved moves I check CharacterMovement headers/cpp; for subobjects/Iris I check replication registration/fragments; for HLOD I check builder commandlet behaviour; for RDG I check pass/resource APIs. Then I make a compile or runtime proof.
- **Common Weak Answer:** "Read the engine source."
- **Follow-up Questions:** which file? why? compile proof? risk?
- **Hands-on Verification Task:** Pick a schematic sample and list exact engine files/classes to verify.
- **Sources:** [SRC-NET-017], [SRC-NET-018], [SRC-WORLD-003], [SRC-RENDER-019]
- **Version Notes:** Source layout differs by branch.

### Question: How do you handle a feature that cannot be proven in the current branch?

- **Category:** Production Readiness / Scope
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Document rejection or unsupported status with evidence, fallback design and future verification trigger.
- **Strong 3-Year-Engineer Answer:** If Iris, NetworkPrediction, a platform profiler or a builder command is unavailable, I do not leave a silent gap. I record target branch evidence, explain fallback, mark risk and define what would need to change before adoption. This preserves scope without pretending support exists.
- **Common Weak Answer:** "Skip it."
- **Follow-up Questions:** fallback? risk? future trigger? evidence?
- **Hands-on Verification Task:** Write a rejection memo for one unsupported target-branch feature.
- **Sources:** [SRC-NET-014], [SRC-NET-020], [SRC-BUILD-017]
- **Version Notes:** Version-sensitive systems need explicit support checks.
