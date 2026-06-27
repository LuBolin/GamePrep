# Unreal Engine Interview Flashcards

See also: [[ue_interview_question_bank]], [[cpp_interview_question_bank]], [[ue_hands_on_projects]].

## UObject, Reflection, and Lifetime

Q: What runs before the ordinary C++ compiler to process Unreal-aware declarations?  
A: Unreal Header Tool (UHT), invoked by UBT, parses supported metadata and generates UObject support code.  
Category: UObject / Build  
Priority: P0

Q: Is UHT a complete C++ compiler?  
A: No. It parses a supported Unreal-focused subset and generates code that is subsequently compiled by the configured C++ compiler.  
Category: UObject / Build  
Priority: P1

Q: Why must a `.generated.h` include normally be the last include in a reflected header?  
A: Generated declarations depend on the completed preceding include context, and Unreal's expected header structure enforces that placement.  
Category: UObject / Build  
Priority: P1

Q: Name four systems enabled by Unreal reflection.  
A: Any four of editor integration, Blueprint exposure, serialisation, replication metadata, config/save support, runtime type information, and UObject reference discovery.  
Category: Reflection  
Priority: P0

Q: Does `UPROPERTY` mean “editable in the editor”?  
A: No. Editor editing is controlled by specific specifiers; UPROPERTY can participate in several independent engine systems.  
Category: Reflection  
Priority: P0

Q: What is a CDO?  
A: The Class Default Object, the default template maintained for each UClass.  
Category: UObject Lifecycle  
Priority: P0

Q: What belongs in a UObject constructor?  
A: Deterministic defaults and default subobject creation, not world-dependent gameplay work.  
Category: UObject Lifecycle  
Priority: P0

Q: How should a runtime UObject normally be created?  
A: With `NewObject`, supplying an appropriate Outer.  
Category: UObject Lifecycle  
Priority: P0

Q: How should an Actor normally be created at runtime?  
A: Through the world's `SpawnActor` APIs, not `NewObject` or ordinary `new`.  
Category: Gameplay Framework  
Priority: P0

Q: What does `CreateDefaultSubobject` create?  
A: A constructor-time default subobject that becomes part of the class/default object structure.  
Category: UObject Lifecycle  
Priority: P0

Q: Should UObjects be destroyed with `delete`?  
A: No. They participate in Unreal's managed object lifecycle and garbage collection.  
Category: UObject Lifetime  
Priority: P0

Q: What high-level algorithm manages UObject reclamation?  
A: Mark-and-sweep garbage collection based on reachability.  
Category: UObject Lifetime  
Priority: P0

Q: Does a raw UObject pointer keep its target alive merely by being non-null?  
A: No. Retention requires an engine-visible strong reference or another explicit lifetime mechanism.  
Category: UObject Lifetime  
Priority: P0

Q: What is the normal UE5 persistent strong UObject member form?  
A: `UPROPERTY() TObjectPtr<T>`.  
Category: UObject Pointers  
Priority: P0

Q: Does an unreflected `TObjectPtr` member automatically provide the normal GC property edge?  
A: No; for persistent UObject/USTRUCT members, pair it with UPROPERTY so the edge is visible to the relevant engine systems.  
Category: UObject Pointers  
Priority: P0

Q: When is a raw UObject pointer commonly appropriate?  
A: As a short-lived local or parameter while some other mechanism guarantees the target's lifetime.  
Category: UObject Pointers  
Priority: P1

Q: What is `TWeakObjectPtr` for?  
A: Observing or caching a UObject without preventing its collection; resolve and validate it at use time.  
Category: UObject Pointers  
Priority: P0

Q: What is `TSoftObjectPtr` for?  
A: Referencing an object or asset by path without forcing a hard load dependency, allowing explicit deferred loading.  
Category: Assets / UObject Pointers  
Priority: P0

Q: What is the key difference between weak and soft references?  
A: Weak observes a loaded object; soft can locate an unloaded object by path and support explicit loading.  
Category: UObject Pointers  
Priority: P0

Q: Which UObject pointer type is deprecated in favour of soft references?  
A: `TLazyObjectPtr`.  
Category: UObject Pointers  
Priority: P2

Q: What is `TStrongObjectPtr` mainly for?  
A: Deliberately keeping a UObject alive from a non-UObject owner when a reflected property is unavailable.  
Category: UObject Pointers  
Priority: P2

Q: Why is `TSharedPtr<UObject>` normally the wrong model?  
A: It belongs to a separate reference-counted lifetime system and does not express the normal UObject GC/reference contract.  
Category: UObject Pointers  
Priority: P0

Q: Does `Outer` universally mean GC ownership?  
A: No. It primarily supplies containment and identity context; use an explicit strong reference when retention is required.  
Category: UObject Lifetime  
Priority: P0

Q: What is the first step when a UObject disappears unexpectedly?  
A: State the intended root-to-target strong-reference chain, then test it under controlled forced GC.  
Category: Debugging  
Priority: P0

Q: What should be checked when a UObject never disappears?  
A: Reflected strong fields, root additions, native strong holders/reference reporters, closures/delegates, and caches that should be weak.  
Category: Debugging  
Priority: P1

Q: Why is increasing the GC interval not a first-line hitch fix?  
A: It can merely accumulate more work and produce rarer but larger pauses; profile object churn and reference graphs first.  
Category: Profiling  
Priority: P1

Q: What is the UE5.6 status of incremental reachability analysis?  
A: Experimental; do not present it as universally enabled baseline behaviour.  
Category: UObject Lifetime  
Priority: P2

Q: Why does incremental reachability favour `TObjectPtr` properties?  
A: Assignments can provide GC write barriers that preserve newly created reference edges across incremental marking.  
Category: UObject Internals  
Priority: P3

Q: Is a USTRUCT value independently garbage-collected?  
A: No. It has value lifetime, though reflected fields inside it can participate in engine systems including UObject reference tracking.  
Category: Reflection  
Priority: P1

Q: What four verbs simplify pointer selection?  
A: Retain, observe, locate/load, and deterministically own.  
Category: UObject Pointers  
Priority: P0

## Gameplay Framework and Lifecycle

Q: When is an Actor preferable to a Component?  
A: When the concept needs independent world identity, spawning/placement, or its own replication relevance.  
Category: Gameplay Framework  
Priority: P0

Q: When is an Actor Component preferable to an Actor?  
A: For a coherent reusable capability whose lifetime and context belong to an owning Actor.  
Category: Gameplay Architecture  
Priority: P0

Q: Does `UActorComponent` have a transform?  
A: No. Use `USceneComponent` when the component needs a transform.  
Category: Components  
Priority: P0

Q: What distinguishes `UPrimitiveComponent` from `USceneComponent`?  
A: A Primitive Component has geometry used for rendering and/or collision in addition to a transform.  
Category: Components  
Priority: P0

Q: Where does an Actor get its transform?  
A: From its root Scene Component.  
Category: Actor  
Priority: P0

Q: What must a runtime-created Component normally do before participating in the world?  
A: Be associated with an Actor and registered correctly.  
Category: Components  
Priority: P1

Q: Do Actor Components tick by default?  
A: No; ticking must be enabled where needed.  
Category: Components / Performance  
Priority: P1

Q: What is a Pawn?  
A: A possessable Actor representing an agent in the world.  
Category: Gameplay Framework  
Priority: P0

Q: What does Character add to Pawn?  
A: Humanoid-oriented capsule, mesh/movement conventions, CharacterMovement, animation integration, and mature network movement support.  
Category: Gameplay Framework  
Priority: P0

Q: Are Characters only for human players?  
A: No. PlayerControllers or AIControllers can possess them.  
Category: Gameplay Framework  
Priority: P0

Q: Why are Controller and Pawn separate?  
A: Decision source/connection and physical avatar have different lifetimes, enabling respawn, spectating, possession changes, and human/AI reuse.  
Category: Gameplay Architecture  
Priority: P0

Q: Where does GameMode exist in multiplayer?  
A: Only on the server.  
Category: Gameplay Framework / Networking  
Priority: P0

Q: What belongs in GameMode?  
A: Authoritative match rules, admission/spawn policy, and match transitions.  
Category: Gameplay Framework  
Priority: P0

Q: What belongs in GameState?  
A: Changing match-wide state that clients need, such as match phase, team scores, and public objectives.  
Category: Gameplay Framework / Networking  
Priority: P0

Q: Why does client code fail when it reads GameMode?  
A: Network clients do not have a GameMode instance.  
Category: Debugging / Networking  
Priority: P0

Q: Which class is a natural home for public participant score that survives respawn?  
A: PlayerState.  
Category: Gameplay Framework  
Priority: P0

Q: Do clients have every player's PlayerController?  
A: No. A client normally has its own PlayerController; the server has one for every connected player.  
Category: Gameplay Framework / Networking  
Priority: P0

Q: Which class is broadly replicated for every participant: PlayerController or PlayerState?  
A: PlayerState.  
Category: Gameplay Framework / Networking  
Priority: P0

Q: Should secret owner-only state automatically go in PlayerState?  
A: No. PlayerState is broadly visible; choose a home and replication condition matching the intended audience.  
Category: Gameplay Architecture / Networking  
Priority: P1

Q: What should normally happen in an Actor constructor?  
A: Deterministic defaults and default-subobject creation, not world-dependent gameplay work.  
Category: Lifecycle  
Priority: P0

Q: Why must OnConstruction logic tolerate reruns?  
A: Construction can execute repeatedly in editor and instance workflows as properties or construction state change.  
Category: Lifecycle  
Priority: P1

Q: When is component BeginPlay called?  
A: After the component is registered and initialised when its owning Actor begins play, or when created after that point under the documented lifecycle.  
Category: Lifecycle  
Priority: P1

Q: Why clean gameplay resources in EndPlay rather than only on explicit Destroy?  
A: Travel, streaming unload, PIE end, shutdown, and other paths can end play without the gameplay path assuming a manual Destroy call.  
Category: Lifecycle / Debugging  
Priority: P0

Q: What lifetime does a GameInstanceSubsystem share?  
A: The GameInstance, typically across map loads for that process instance.  
Category: Subsystems  
Priority: P1

Q: What lifetime does a WorldSubsystem share?  
A: One UWorld.  
Category: Subsystems  
Priority: P1

Q: What lifetime does a LocalPlayerSubsystem share?  
A: One local player, which matters for split-screen and per-user local state.  
Category: Subsystems  
Priority: P1

Q: Are subsystems replicated automatically?  
A: No. They are lifetime/service containers, not automatic network replication units.  
Category: Subsystems / Networking  
Priority: P1

Q: What is the best default alternative to per-frame UI polling?  
A: Event-driven updates when the underlying state changes.  
Category: Performance / UI  
Priority: P0

Q: Is a frame-rate timer inherently cheaper than Tick?  
A: No. Similar frequency and work usually mean similar cost; choose semantics first and measure.  
Category: Performance  
Priority: P1

Q: Name four distinct Unreal relationships often confused as ownership.  
A: Any four of GC retention, Outer containment, Actor Owner/network ownership, Scene attachment, Controller possession, and server authority.  
Category: Gameplay Architecture  
Priority: P0

## Standard C++ Lifetime and Ownership

Q: What is the difference between storage and object lifetime?  
A: Storage is suitably sized/aligned bytes; lifetime is the interval in which an object of a type exists in those bytes.  
Category: Standard C++ / Lifetime  
Priority: P0

Q: Can a non-null pointer dangle?  
A: Yes. The address can remain non-null after the pointed-to object's lifetime ends.  
Category: Standard C++ / Lifetime  
Priority: P0

Q: Does a reference guarantee its target remains alive?  
A: No. A reference must initially bind to an object, but can dangle if that object's lifetime ends first.  
Category: Standard C++ / Lifetime  
Priority: P0

Q: What four questions clarify lifetime design?  
A: Where is storage, when is the object alive, who owns/destructs it, and which handles may access it?  
Category: Standard C++ / Lifetime  
Priority: P0

Q: What is value semantics?  
A: The object owns its state and has deliberate copy, move, assignment, and destruction behaviour representing values.  
Category: Standard C++ / Values  
Priority: P0

Q: Can a value type allocate dynamically internally?  
A: Yes; strings and vectors are value-like while managing dynamic storage internally.  
Category: Standard C++ / Values  
Priority: P1

Q: What does RAII bind together?  
A: Resource acquisition/release and an owning object's construction/destruction.  
Category: Standard C++ / Resource Management  
Priority: P0

Q: Is RAII only for memory?  
A: No; it applies to locks, files, sockets, GPU handles, registrations, transactions, and other resources.  
Category: Standard C++ / Resource Management  
Priority: P0

Q: What is the Rule of Zero?  
A: Compose self-managing members so the class needs no custom destructor, copy, or move operations.  
Category: Standard C++ / Class Design  
Priority: P0

Q: What is the useful interpretation of the Rule of Five?  
A: If one copy/move/destructor operation is declared, deliberately define, default, or delete the coherent set.  
Category: Standard C++ / Class Design  
Priority: P0

Q: Can declaring a destructor suppress implicit moves?  
A: Yes, which is why all special-member semantics must be considered together.  
Category: Standard C++ / Class Design  
Priority: P1

Q: When should copying a type be deleted?  
A: When duplicating the logical resource/identity is invalid or would create multiple apparent unique owners.  
Category: Standard C++ / Class Design  
Priority: P0

Q: What does a copy operation represent?  
A: Creation or assignment of another object with the source's logical value according to the type's copy contract.  
Category: Standard C++ / Values  
Priority: P0

Q: What does `std::move` itself do?  
A: It casts to an xvalue so move-aware overloads may be selected; it transfers nothing by itself.  
Category: Standard C++ / Move Semantics  
Priority: P0

Q: What state is a moved-from standard-library object usually in?  
A: Valid but unspecified, unless that type documents a stronger postcondition.  
Category: Standard C++ / Move Semantics  
Priority: P0

Q: What may safely be done with a valid-but-unspecified moved-from object?  
A: Destroy it, assign a new value, or call operations whose preconditions you establish.  
Category: Standard C++ / Move Semantics  
Priority: P0

Q: Why can `std::move` on a const object copy?  
A: Typical move operations require a non-const rvalue because they modify the source; a const rvalue may match the copy overload instead.  
Category: Standard C++ / Move Semantics  
Priority: P1

Q: Why should a genuinely non-throwing move be marked `noexcept`?  
A: Containers can use it during relocation while preserving strong exception guarantees rather than copying.  
Category: Standard C++ / Move Semantics  
Priority: P1

Q: What does `std::move_if_noexcept` select?  
A: Move when it is non-throwing or copying is unavailable; otherwise a const lvalue reference enabling copy.  
Category: Standard C++ / Move Semantics  
Priority: P2

Q: Should a function usually write `return std::move(Local);`?  
A: No; returning the local directly preserves NRVO eligibility and supports implicit move fallback.  
Category: Standard C++ / Value Semantics  
Priority: P0

Q: What is NRVO?  
A: Permitted construction of a named local return object directly in the caller's result storage.  
Category: Standard C++ / Value Semantics  
Priority: P1

Q: What return construction is guaranteed in C++17?  
A: Same-type prvalue results are constructed directly in their final destination under the prvalue rules.  
Category: Standard C++ / Value Semantics  
Priority: P1

Q: What ownership does `unique_ptr` express?  
A: Exclusive dynamic ownership with destruction on owner destruction/reset and transfer through move.  
Category: Standard C++ / Ownership  
Priority: P0

Q: Is `unique_ptr` copyable?  
A: No; it is movable so ownership can be transferred.  
Category: Standard C++ / Ownership  
Priority: P0

Q: What is the difference between `unique_ptr::get()` and `release()`?  
A: `get()` borrows the pointer while ownership remains; `release()` relinquishes ownership without deleting.  
Category: Standard C++ / Ownership  
Priority: P1

Q: What ownership does `shared_ptr` express?  
A: Reference-counted co-ownership; the managed object is destroyed when the final strong owner releases.  
Category: Standard C++ / Ownership  
Priority: P0

Q: Why is `shared_ptr` not the safest universal default?  
A: It adds count/control-block costs, obscures ownership and destruction, and allows strong cycles.  
Category: Standard C++ / Ownership  
Priority: P0

Q: Does shared_ptr's thread-safe control block make the pointee thread-safe?  
A: No; pointee state requires its own synchronisation.  
Category: Standard C++ / Concurrency  
Priority: P0

Q: What problem does `weak_ptr` solve?  
A: Non-retaining observation of shared-owned objects and breaking strong reference cycles.  
Category: Standard C++ / Ownership  
Priority: P0

Q: How should a weak_ptr target be accessed?  
A: Call `lock()` and use the resulting local shared_ptr if non-null.  
Category: Standard C++ / Ownership  
Priority: P0

Q: Why not call `expired()` and then access separately?  
A: Lifetime can change between the check and access; `lock()` combines the attempt with acquiring strong lifetime.  
Category: Standard C++ / Concurrency  
Priority: P1

Q: What should `const T&` usually communicate?  
A: Required non-owning read-only access for the duration of the call.  
Category: Standard C++ / API Design  
Priority: P0

Q: What should `T*` commonly communicate in a modern API?  
A: Nullable non-owning access, unless an explicitly documented legacy/interop contract says otherwise.  
Category: Standard C++ / API Design  
Priority: P0

Q: When should a smart pointer be passed by value?  
A: When the function intentionally acquires/transfers unique ownership or participates in shared ownership.  
Category: Standard C++ / API Design  
Priority: P0

Q: What does vector reallocation invalidate?  
A: All iterators, pointers, and references to its elements.  
Category: Standard C++ / Containers  
Priority: P0

Q: Without reallocation, what does vector insertion invalidate?  
A: Handles at or after the insertion point, including the old end iterator.  
Category: Standard C++ / Containers  
Priority: P1

Q: What does vector erase invalidate?  
A: Handles to erased elements and every element after them.  
Category: Standard C++ / Containers  
Priority: P0

Q: Does reserve make vector element handles permanently stable?  
A: No; it can prevent selected reallocations, but erase, insertion position, reorder, and later capacity growth still matter.  
Category: Standard C++ / Containers  
Priority: P0

Q: What is the relationship between TSharedPtr and UObject GC?  
A: TSharedPtr is reference counting for ordinary non-UObjects; it is not a normal UObject ownership mechanism.  
Category: UE C++ / Ownership  
Priority: P0

Q: What is the difference between TWeakPtr and TWeakObjectPtr?  
A: TWeakPtr observes TSharedPtr-managed objects; TWeakObjectPtr observes UObjects in Unreal's object system.  
Category: UE C++ / Ownership  
Priority: P0

## UE Core Types, Containers, Delegates, and Blueprint

Q: What is FName primarily for?  
A: Immutable identifier/name-table semantics and frequent comparisons, not mutable or localisable text.  
Category: UE C++ / Strings  
Priority: P0

Q: What is FString primarily for?  
A: Mutable character data used for building, parsing, searching, paths, logs, and external text.  
Category: UE C++ / Strings  
Priority: P0

Q: What is FText primarily for?  
A: Localisable and culture-aware user-facing display text.  
Category: UE C++ / Strings  
Priority: P0

Q: Why can FText-to-FString conversion be lossy?  
A: It can discard localisation history and context, retaining only a display string.  
Category: UE C++ / Strings  
Priority: P1

Q: Why can FString-to-FName conversion be lossy?  
A: FName's identifier semantics are case-insensitive, so case distinctions can be lost.  
Category: UE C++ / Strings  
Priority: P1

Q: Which type should normally hold a localisable menu label?  
A: FText.  
Category: UE C++ / Strings  
Priority: P0

Q: Which type should normally hold a socket or bone identifier?  
A: FName.  
Category: UE C++ / Strings  
Priority: P0

Q: Which type should normally hold a mutable path under construction?  
A: FString.  
Category: UE C++ / Strings  
Priority: P0

Q: What does the TEXT macro provide?  
A: A TCHAR-compatible string literal for Unreal's native character representation.  
Category: UE C++ / Strings  
Priority: P1

Q: What semantics does TArray have?  
A: An ordered contiguous value-owning dynamic sequence with deep-copy value behaviour.  
Category: UE C++ / Containers  
Priority: P0

Q: When is TSet a natural choice?  
A: When values must be unique, hash lookup matters, and iteration order is not meaningful.  
Category: UE C++ / Containers  
Priority: P1

Q: When is TMap a natural choice?  
A: When unique keys map to associated values and hash lookup fits the access pattern.  
Category: UE C++ / Containers  
Priority: P1

Q: Does TSet iteration preserve insertion order?  
A: No; its iteration order is not a stable semantic contract.  
Category: UE C++ / Containers  
Priority: P0

Q: What must a normal TSet/TMap key provide?  
A: Equality semantics and a matching hash, commonly operator== and GetTypeHash.  
Category: UE C++ / Containers  
Priority: P1

Q: Why can hash-container element pointers become invalid?  
A: Backing storage can reallocate as the container grows, regardless of sparse gaps.  
Category: UE C++ / Containers  
Priority: P1

Q: What does RemoveAt preserve that RemoveAtSwap does not?  
A: Relative order of remaining TArray elements.  
Category: UE C++ / Containers  
Priority: P0

Q: When is RemoveAtSwap appropriate?  
A: When element order is irrelevant and replacing the removed slot from the end is acceptable.  
Category: UE C++ / Containers  
Priority: P1

Q: Why is TArray usually preferred over std::vector in a UPROPERTY?  
A: Unreal's reflection, serialisation, Blueprint, and replication property systems understand supported TArray properties.  
Category: UE C++ / Containers  
Priority: P0

Q: Is TArray always faster than std::vector?  
A: No; choose by ecosystem and semantics, then measure representative workloads.  
Category: UE C++ / Containers  
Priority: P0

Q: What does Cast<T> return when a UObject is incompatible?  
A: Null.  
Category: UE C++ / Casting  
Priority: P0

Q: When is CastChecked appropriate?  
A: When incompatibility is a programming invariant violation and a hard diagnostic is intended.  
Category: UE C++ / Casting  
Priority: P1

Q: What can repeated casts across many concrete Actor types indicate?  
A: A missing capability boundary such as an interface, component, registry, or explicit relationship.  
Category: UE Architecture  
Priority: P0

Q: Which API creates a runtime UObject?  
A: NewObject.  
Category: UE C++ / Creation  
Priority: P0

Q: Which API creates a runtime Actor?  
A: UWorld::SpawnActor or its appropriate variant.  
Category: UE C++ / Creation  
Priority: P0

Q: Which API defines a constructor-time default subobject?  
A: CreateDefaultSubobject.  
Category: UE C++ / Creation  
Priority: P0

Q: Why use deferred Actor spawning?  
A: To set required data before construction scripts/final spawn lifecycle completes.  
Category: UE C++ / Creation  
Priority: P1

Q: What must complete a deferred Actor spawn?  
A: FinishSpawning/FinishSpawningActor through the appropriate API.  
Category: UE C++ / Creation  
Priority: P0

Q: What is a native single-cast delegate for?  
A: One native callback, potentially with a return value.  
Category: UE C++ / Delegates  
Priority: P1

Q: What is a multicast delegate for?  
A: Broadcasting a notification to multiple listeners, normally without a return value.  
Category: UE C++ / Delegates  
Priority: P0

Q: Why use a dynamic delegate?  
A: When reflection, serialisation, or Blueprint binding is required.  
Category: UE C++ / Delegates  
Priority: P0

Q: Are dynamic delegates faster than native delegates?  
A: No; their reflection/serialisation capability adds overhead.  
Category: UE C++ / Delegates  
Priority: P1

Q: What lifetime risk does AddRaw introduce?  
A: The delegate does not track the raw target; it may invoke a destroyed object unless explicitly unbound/lifetime-proven.  
Category: UE C++ / Delegates  
Priority: P0

Q: What does AddWeakLambda protect?  
A: It invokes the lambda only while the associated UObject weak reference remains valid.  
Category: UE C++ / Delegates  
Priority: P1

Q: Why store an FDelegateHandle?  
A: To remove the exact binding deterministically during cleanup or reconfiguration.  
Category: UE C++ / Delegates  
Priority: P0

Q: What does Blueprintable allow?  
A: Blueprint classes may derive from the native class.  
Category: C++ / Blueprint  
Priority: P1

Q: What does BlueprintType allow?  
A: The type can be used for Blueprint variables and pins.  
Category: C++ / Blueprint  
Priority: P1

Q: What does BlueprintPure promise?  
A: No observable state mutation; it does not promise low cost or single evaluation.  
Category: C++ / Blueprint  
Priority: P0

Q: When should BlueprintImplementableEvent be used?  
A: For a Blueprint-provided extension where no native implementation is required and absence is safe.  
Category: C++ / Blueprint  
Priority: P1

Q: When should BlueprintNativeEvent be used?  
A: When a native default implementation is required but Blueprint may override it.  
Category: C++ / Blueprint  
Priority: P1

Q: Why prefer BlueprintReadOnly plus mutation functions for critical state?  
A: Validated functions preserve authority, clamping, replication, events, and other invariants.  
Category: C++ / Blueprint  
Priority: P0

## Networking and Replication

Q: Which machine owns authoritative replicated game state?  
A: The server.  
Category: Networking / Authority  
Priority: P0

Q: Does an autonomous proxy own authoritative game truth?  
A: No; it predicts locally and submits intent, while the server remains authoritative.  
Category: Networking / Roles  
Priority: P0

Q: What does an Actor's owning connection determine?  
A: Client RPC destination, valid client-to-server RPC routing, owner conditions, and owner relevancy/priority behaviour.  
Category: Networking / Ownership  
Priority: P0

Q: Is Actor ownership the same as attachment?  
A: No; network ownership and spatial attachment are separate relationships.  
Category: Networking / Ownership  
Priority: P0

Q: Why can listen-host testing hide networking bugs?  
A: The host is both local player and authoritative server, bypassing remote ownership, delay, and separate-instance boundaries.  
Category: Networking / Testing  
Priority: P0

Q: Where must replicated gameplay Actors normally be spawned?  
A: On the authoritative server.  
Category: Networking / Replication  
Priority: P0

Q: Does setting bReplicates on a client-spawned Actor make it server-authoritative?  
A: No; it remains a local client Actor.  
Category: Networking / Replication  
Priority: P0

Q: What two C++ pieces normally register a replicated property?  
A: A Replicated/ReplicatedUsing UPROPERTY and DOREPLIFETIME-style registration in GetLifetimeReplicatedProps.  
Category: Networking / Properties  
Priority: P0

Q: What is OnRep for?  
A: Reacting locally when replicated server state is applied, often updating presentation, UI, events, or derived caches.  
Category: Networking / Properties  
Priority: P0

Q: Should OnRep perform authoritative state mutation?  
A: No; authoritative mutation belongs on the server.  
Category: Networking / Properties  
Priority: P0

Q: Is the order of different OnRep callbacks deterministic?  
A: No.  
Category: Networking / Ordering  
Priority: P0

Q: How should several replicated fields forming one invariant be sent?  
A: Prefer one coherent struct/state transition or explicit coordination/versioning.  
Category: Networking / Architecture  
Priority: P0

Q: What is COND_OwnerOnly for?  
A: Replicating a property only to the Actor's owning connection.  
Category: Networking / Conditions  
Priority: P1

Q: What is COND_SkipOwner for?  
A: Replicating to everyone except the owner, often when the owner predicts locally.  
Category: Networking / Conditions  
Priority: P1

Q: What is COND_InitialOnly for?  
A: Sending a property's initial replicated value without continuous updates.  
Category: Networking / Conditions  
Priority: P1

Q: What does a Server RPC communicate?  
A: A call from an owning client/server context to execute authoritative logic on the server.  
Category: Networking / RPC  
Priority: P0

Q: What does a Client RPC target?  
A: The owning client connection of the Actor on which the server invokes it.  
Category: Networking / RPC  
Priority: P0

Q: From where must a NetMulticast normally be called to fan out over the network?  
A: The authoritative server.  
Category: Networking / RPC  
Priority: P0

Q: Why can a client not call a Server RPC on any Actor it can see?  
A: Valid routing normally requires that client's owning connection to own the replicated Actor.  
Category: Networking / RPC  
Priority: P0

Q: What should a client send: intent or trusted outcome?  
A: Intent/evidence; the server validates and derives the authoritative outcome.  
Category: Networking / Security  
Priority: P0

Q: What does Reliable mean for an RPC?  
A: It is retransmitted until acknowledged, with associated bandwidth and blocking/backlog costs.  
Category: Networking / RPC  
Priority: P0

Q: Why is a reliable RPC every Tick dangerous?  
A: Packet loss can build a retransmission backlog and delay later reliable traffic.  
Category: Networking / Performance  
Priority: P0

Q: What type of data fits an unreliable RPC?  
A: Frequent transient or superseding data whose occasional loss is acceptable.  
Category: Networking / RPC  
Priority: P1

Q: Why are properties better than RPC-only state for late joiners?  
A: Replication converges them to current truth; past RPC events are not automatically replayed.  
Category: Networking / Architecture  
Priority: P0

Q: What is an autonomous proxy?  
A: The owning client's local replicated Actor instance that predicts/control-simulates its own actions.  
Category: Networking / Roles  
Priority: P0

Q: What is a simulated proxy?  
A: A non-owning client instance that simulates/interpolates replicated authoritative updates.  
Category: Networking / Roles  
Priority: P0

Q: What does CharacterMovement client prediction record?  
A: Saved moves/input and movement state that can be sent, acknowledged/corrected, and replayed.  
Category: Networking / Movement  
Priority: P1

Q: Why can direct transform changes fight CharacterMovement networking?  
A: They may bypass its saved-move prediction protocol, causing server discrepancy and correction.  
Category: Networking / Movement  
Priority: P1

Q: What is Actor relevancy?  
A: A per-connection decision about whether an Actor should currently replicate to that client.  
Category: Networking / Relevancy  
Priority: P0

Q: What can happen to a dynamic replicated Actor when it becomes irrelevant?  
A: Its client proxy can be destroyed and later recreated if relevant again.  
Category: Networking / Relevancy  
Priority: P1

Q: What does bAlwaysRelevant trade?  
A: Simpler guaranteed relevance for increased server consideration and bandwidth to every applicable connection.  
Category: Networking / Performance  
Priority: P1

Q: What is network dormancy?  
A: Keeping an Actor present while excluding stable state from normal replication consideration.  
Category: Networking / Dormancy  
Priority: P0

Q: Should replicated state change while an Actor is dormant?  
A: Not before waking/flushing it; changes may be missed or implementation-dependent.  
Category: Networking / Dormancy  
Priority: P0

Q: When should a dormant Actor be flushed or awakened?  
A: Before changing replicated state.  
Category: Networking / Dormancy  
Priority: P0

Q: Is dormancy the same as irrelevancy?  
A: No; dormant Actors remain present, while irrelevant dynamic Actors may be removed from a client.  
Category: Networking / Dormancy  
Priority: P0

Q: Why can frequent dormancy changes hurt performance?  
A: Sleep/wake/flush processing can cost more than keeping a frequently changing Actor awake.  
Category: Networking / Performance  
Priority: P1

Q: What problem does Replication Graph address?  
A: Scalable routing/filtering of many replicated Actors across many connections.  
Category: Networking / Scalability  
Priority: P2

Q: Is Iris automatically the replication system for every UE5 project?  
A: No; system selection is project, engine-version, plugin, and configuration dependent.  
Category: Networking / Iris  
Priority: P2

Q: What should a network profile rank?  
A: Actor, property, and RPC bytes/frequency plus server processing and player-visible latency under representative conditions.  
Category: Networking / Profiling  
Priority: P1

Q: What is the first diagnostic for a multiplayer-only failure?  
A: Reproduce on a remote client and log net mode, role, owner/connection, relevance, dormancy, and replication setup.  
Category: Networking / Debugging  
Priority: P0

## AI and Navigation

Q: What should normally own AI decision policy?  
A: AIController and its decision components/assets, while the Pawn owns physical capabilities.  
Category: AI / Framework  
Priority: P1

Q: What is a Blackboard?  
A: Shared decision-relevant memory/keys used by a Behaviour Tree, not the decision policy itself.  
Category: AI / Behaviour Tree  
Priority: P1

Q: What does a Behaviour Tree Selector do?  
A: Tries children by priority/order until one succeeds or remains running.  
Category: AI / Behaviour Tree  
Priority: P1

Q: What does a Behaviour Tree Sequence do?  
A: Runs children in order until one fails/runs; succeeds if all succeed.  
Category: AI / Behaviour Tree  
Priority: P1

Q: What belongs in a Behaviour Tree Task?  
A: A bounded action with explicit success, failure, running, and abort lifecycle.  
Category: AI / Behaviour Tree  
Priority: P1

Q: What belongs in a Behaviour Tree Decorator?  
A: A branch condition/gate, optionally observing changes to abort/replan.  
Category: AI / Behaviour Tree  
Priority: P1

Q: What is a Behaviour Tree Service?  
A: A periodic context update active while execution remains in its associated subtree.  
Category: AI / Behaviour Tree  
Priority: P1

Q: Does a Behaviour Tree Service run on a background thread?  
A: No; “service” does not imply asynchronous or off-thread execution.  
Category: AI / Behaviour Tree  
Priority: P0

Q: What does Observer Aborts Self do?  
A: Aborts the branch when its observed condition becomes invalid.  
Category: AI / Behaviour Tree  
Priority: P1

Q: What does Observer Aborts Lower Priority do?  
A: Lets a newly valid higher-priority branch interrupt lower-priority execution.  
Category: AI / Behaviour Tree  
Priority: P1

Q: Why are UE Behaviour Trees called event-driven?  
A: Observed changes can trigger replanning/aborts without continuously reevaluating the whole tree every frame.  
Category: AI / Behaviour Tree  
Priority: P1

Q: What must a latent Behaviour Tree Task do?  
A: Signal completion exactly once and clean up/cancel work when aborted.  
Category: AI / Behaviour Tree  
Priority: P0

Q: Is Simple Parallel necessarily multithreaded?  
A: No; it models concurrent behaviours, not CPU parallelism.  
Category: AI / Behaviour Tree  
Priority: P1

Q: What is StateTree?  
A: A UE5 hierarchical state machine combining states/transitions with selector-style state selection.  
Category: AI / StateTree  
Priority: P1

Q: When is a simple FSM preferable to BT or StateTree?  
A: When there are few explicit stable states/transitions and larger tooling adds no value.  
Category: AI / Architecture  
Priority: P1

Q: What does AI Perception collect?  
A: Stimuli from configured senses and registered sources, producing perception update events/data.  
Category: AI / Perception  
Priority: P1

Q: Currently perceived versus known perceived?  
A: Currently perceived is sensed now; known includes not-yet-forgotten prior stimuli.  
Category: AI / Perception  
Priority: P1

Q: What does a failed sight stimulus mean?  
A: Sight is no longer successfully sensing that stimulus; other senses/memory may remain valid.  
Category: AI / Perception  
Priority: P0

Q: What can Max Age zero mean in AI Perception?  
A: The stimulus is never forgotten for that sense configuration.  
Category: AI / Perception  
Priority: P1

Q: What is an EQS Generator?  
A: It creates candidate Actors or locations to evaluate.  
Category: AI / EQS  
Priority: P1

Q: What is an EQS Context?  
A: A reference frame/provider such as querier, target, or objective used by generators/tests.  
Category: AI / EQS  
Priority: P1

Q: What is an EQS Test?  
A: A filter and/or scorer applied to generated candidates.  
Category: AI / EQS  
Priority: P1

Q: Should a mandatory EQS constraint only be a score?  
A: No; filter invalid candidates so a high-scoring invalid item cannot win.  
Category: AI / EQS  
Priority: P0

Q: What does UE NavMesh represent?  
A: Tiled navigable polygons generated from collision, connected as a costed graph for agent pathfinding.  
Category: AI / Navigation  
Priority: P1

Q: What can a Nav Modifier change?  
A: Navigable area classification/cost or exclusion, affecting path choice.  
Category: AI / Navigation  
Priority: P1

Q: What does a Nav Link represent?  
A: A traversable connection between otherwise disconnected navigation regions, such as a jump or door transition.  
Category: AI / Navigation  
Priority: P1

Q: Does visible green NavMesh guarantee MoveTo succeeds?  
A: No; agent settings, goal projection, path policy, path following, movement, requests, and authority still matter.  
Category: AI / Navigation  
Priority: P0

Q: Pathfinding versus path following?  
A: Pathfinding computes a route; path following drives the agent along it.  
Category: AI / Navigation  
Priority: P0

Q: Pathfinding versus avoidance?  
A: Pathfinding handles global static-space routes; avoidance locally steers around moving agents/obstacles.  
Category: AI / Navigation  
Priority: P0

Q: Where is UE RVO avoidance integrated?  
A: CharacterMovement for Character agents.  
Category: AI / Avoidance  
Priority: P1

Q: What is one RVO limitation in UE?  
A: It does not use NavMesh for avoidance and may steer an agent outside navigable bounds.  
Category: AI / Avoidance  
Priority: P1

Q: Should RVO and Detour Crowd both be enabled for one agent?  
A: No; Epic's guidance treats them as exclusive alternatives.  
Category: AI / Avoidance  
Priority: P1

Q: What is A*'s priority function?  
A: `f(n) = g(n) + h(n)`, known path cost plus heuristic estimate.  
Category: AI / Algorithms  
Priority: P1

Q: What does an admissible A* heuristic do?  
A: It never overestimates remaining cost, supporting optimality under the relevant conditions.  
Category: AI / Algorithms  
Priority: P1

Q: Which tools show BT/Blackboard, EQS, Perception, and NavMesh state?  
A: The built-in AI Debugger; Visual Logger adds timeline/spatial recording.  
Category: AI / Debugging  
Priority: P0

## Profiling and Optimisation

Q: What is the frame budget at 30 FPS?  
A: Approximately 33.33 ms.  
Category: Profiling / Frame Budget  
Priority: P0

Q: What is the frame budget at 60 FPS?  
A: Approximately 16.67 ms.  
Category: Profiling / Frame Budget  
Priority: P0

Q: What is the frame budget at 120 FPS?  
A: Approximately 8.33 ms.  
Category: Profiling / Frame Budget  
Priority: P0

Q: Why budget below the theoretical frame limit?  
A: To absorb content variance, OS work, streaming, GC, networking, and thermal/device variability.  
Category: Profiling / Frame Budget  
Priority: P1

Q: Why are milliseconds better than FPS for budgeting?  
A: Milliseconds are additive and directly show feature cost; FPS is reciprocal.  
Category: Profiling / Frame Budget  
Priority: P0

Q: What does P99 frame time reveal?  
A: Slow-tail behaviour/hitches hidden by averages or medians.  
Category: Profiling / Statistics  
Priority: P1

Q: What does the Game time in stat unit represent?  
A: High-level game-thread time.  
Category: Profiling / Stats  
Priority: P0

Q: What does Draw in stat unit represent?  
A: High-level render-thread CPU time.  
Category: Profiling / Stats  
Priority: P0

Q: What does GPU in stat unit represent?  
A: Time the GPU takes to execute the frame's rendering work under the measurement setup.  
Category: Profiling / Stats  
Priority: P0

Q: Should Game, Draw and GPU times be added to get Frame?  
A: No; pipeline stages overlap and synchronise, so Frame tends toward the limiting lane.  
Category: Profiling / Frame Pipeline  
Priority: P0

Q: What is stat unit best used for?  
A: Initial bottleneck classification, not final root-cause proof.  
Category: Profiling / Stats  
Priority: P0

Q: What does Timing Insights show?  
A: Frame and CPU/GPU tracks, scopes/timers, counters, callers/callees and related trace data.  
Category: Profiling / Insights  
Priority: P0

Q: Inclusive versus exclusive scope time?  
A: Inclusive includes child calls; exclusive is time in the scope excluding children.  
Category: Profiling / Insights  
Priority: P1

Q: Why inspect call count as well as duration?  
A: Tiny work repeated many times can dominate total cost and reveal duplicate scheduling.  
Category: Profiling / Insights  
Priority: P0

Q: What do callers and callees help answer?  
A: Who triggered an expensive scope and where its time is delegated.  
Category: Profiling / Insights  
Priority: P1

Q: Why capture only needed trace channels/windows?  
A: Tracing consumes CPU, memory and disk and can perturb the workload.  
Category: Profiling / Measurement  
Priority: P1

Q: What is QUICK_SCOPE_CYCLE_COUNTER for?  
A: Adding a scoped cycle timing stat quickly around project code.  
Category: Profiling / Instrumentation  
Priority: P1

Q: What does Memory Insights Growth query investigate?  
A: Allocations created between marked times that remain live beyond the later marker.  
Category: Profiling / Memory  
Priority: P1

Q: What does a Short Living Allocations query reveal?  
A: Allocation churn created and freed within a selected interval.  
Category: Profiling / Memory  
Priority: P1

Q: Why repeat a lifecycle when investigating leaks?  
A: Unbounded staircase growth across cycles distinguishes retention from one-time warm-up/cache growth.  
Category: Profiling / Memory  
Priority: P0

Q: Why can a memory increase be legitimate?  
A: Caches, streaming, warm-up or delayed GC may retain memory intentionally.  
Category: Profiling / Memory  
Priority: P1

Q: What should be confirmed before changing GC settings?  
A: A measured GC-correlated spike plus object/reference/churn evidence.  
Category: Profiling / GC  
Priority: P0

Q: Why can increasing the GC interval worsen hitches?  
A: More objects/reference work may accumulate for a larger later collection.  
Category: Profiling / GC  
Priority: P1

Q: When can manual GC be reasonable?  
A: At a measured UX-safe boundary such as loading, when it prevents worse gameplay-time spikes or memory pressure.  
Category: Profiling / GC  
Priority: P1

Q: Is replacing Tick with a same-frequency timer an optimisation?  
A: Usually not; invocation frequency and work remain similar.  
Category: Profiling / CPU  
Priority: P0

Q: When can event-driven updates be worse than batching?  
A: When many events fire in one frame and repeatedly recompute results that are overwritten.  
Category: Profiling / CPU  
Priority: P1

Q: What is dirty batching?  
A: Marking data changed and recomputing once at a controlled point instead of on every change event.  
Category: Profiling / CPU  
Priority: P1

Q: What quick test suggests pixel/shader GPU cost?  
A: Lower screen percentage/resolution and observe whether GPU time falls substantially.  
Category: Profiling / GPU  
Priority: P0

Q: What tool ranks broad GPU passes?  
A: stat GPU and GPU Visualiser/profile GPU, with Insights GPU tracks as available.  
Category: Profiling / GPU  
Priority: P0

Q: What is RenderDoc best suited for?  
A: Detailed single-frame graphics pipeline, resource, draw-event and shader debugging.  
Category: Profiling / GPU  
Priority: P1

Q: Why is object pooling not automatically faster?  
A: It retains memory and adds reset, stale-handle, capacity and lifecycle complexity.  
Category: Profiling / Architecture  
Priority: P0

Q: What evidence justifies pooling?  
A: Measured material allocation/construction/destruction churn and a safe bounded reset/ownership contract.  
Category: Profiling / Architecture  
Priority: P1

Q: What do scalability groups provide?  
A: Quality/performance tiers controlling groups of rendering/features.  
Category: Profiling / Scalability  
Priority: P1

Q: What do device profiles provide?  
A: Platform/device-family-specific configuration and CVar overrides, including quality and memory settings.  
Category: Profiling / Scalability  
Priority: P1

Q: Why profile packaged builds on target hardware?  
A: Editor/debug overhead and desktop drivers/hardware do not reproduce target CPU, GPU, memory, I/O, thermal or build behaviour.  
Category: Profiling / Method  
Priority: P0

## Rendering and GPU Performance

Q: What is the RHI?  
A: Unreal's abstraction over platform graphics interfaces, resources, pipelines and command submission.  
Category: Rendering / Pipeline  
Priority: P1

Q: Why are Game, Draw/RHI and GPU times not additive?  
A: They are overlapping pipeline lanes with dependencies and synchronisation rather than one serial stack.  
Category: Rendering / Pipeline  
Priority: P0

Q: What does a deferred base pass commonly write?  
A: Depth and surface/material attributes into GBuffer render targets for later lighting.  
Category: Rendering / Deferred  
Priority: P1

Q: What is a key cost of deferred shading?  
A: GBuffer render-target memory and bandwidth, alongside its feature-specific trade-offs.  
Category: Rendering / Deferred  
Priority: P1

Q: Why can forward shading suit VR?  
A: It can provide a leaner supported path and hardware MSAA, which can preserve clarity in head-mounted displays.  
Category: Rendering / Forward  
Priority: P1

Q: Is forward shading always faster than deferred?  
A: No; required features, materials, lights, platform and content determine the measured result.  
Category: Rendering / Architecture  
Priority: P0

Q: What does a depth prepass trade?  
A: Extra geometry/draw work for the chance to reject hidden pixels before expensive shading.  
Category: Rendering / Depth  
Priority: P1

Q: What usually pays draw-call setup cost?  
A: CPU render/RHI command preparation and state submission, though draws also trigger GPU work.  
Category: Rendering / Submission  
Priority: P0

Q: Why can merging meshes hurt performance?  
A: It can make culling, transforms, materials, streaming and updates too coarse.  
Category: Rendering / Submission  
Priority: P1

Q: What is fill-rate pressure?  
A: Cost from shading/writing many pixel samples, often amplified by resolution, coverage and overdraw.  
Category: Rendering / Pixels  
Priority: P0

Q: What is overdraw?  
A: Shading or processing the same screen area multiple times because several surfaces/layers cover it.  
Category: Rendering / Pixels  
Priority: P0

Q: Why is translucency often overdraw-heavy?  
A: Visible layers must blend and cannot benefit from opaque depth rejection in the same way.  
Category: Rendering / Translucency  
Priority: P0

Q: What does strong resolution sensitivity suggest?  
A: A pixel-scaled pass such as lighting, translucency, post-processing or bandwidth is likely important.  
Category: Rendering / Diagnosis  
Priority: P0

Q: Does Shader Complexity equal GPU frame time?  
A: No; it is a comparative diagnostic view that must be confirmed with pass timing on target.  
Category: Rendering / Materials  
Priority: P0

Q: Dynamic parameter versus static switch?  
A: A dynamic parameter changes a runtime value; a static switch selects separately compiled shader variants.  
Category: Rendering / Materials  
Priority: P1

Q: What is permutation explosion?  
A: Multiplicative compiled shader variants from feature, switch, pass, platform and vertex-factory combinations.  
Category: Rendering / Shaders  
Priority: P1

Q: Why is a material graph node count not a GPU cost?  
A: Compilation, optimisation, execution frequency, textures, control flow, registers and pass context determine actual work.  
Category: Rendering / Shaders  
Priority: P1

Q: What can incorrect WPO bounds cause?  
A: Culling or shadow popping; oversized compensation can harm visibility and shadow invalidation efficiency.  
Category: Rendering / Materials  
Priority: P1

Q: LOD versus HLOD?  
A: LOD simplifies one asset; HLOD replaces a distant spatial group with a proxy representation.  
Category: Rendering / Geometry  
Priority: P0

Q: ISM versus many independent mesh components?  
A: ISM represents compatible repeated mesh instances with lower submission/scene overhead but less per-object independence.  
Category: Rendering / Geometry  
Priority: P1

Q: What does HISM add conceptually?  
A: A hierarchy intended to improve culling and LOD handling for large instance populations.  
Category: Rendering / Geometry  
Priority: P1

Q: What is Nanite's core mechanism?  
A: Hierarchical geometry clusters, view-dependent detail selection, demand streaming and a GPU-driven rendering path.  
Category: Rendering / Nanite  
Priority: P1

Q: What does Nanite primarily solve?  
A: Supported geometry detail, LOD selection, visibility and traditional draw-submission pressure.  
Category: Rendering / Nanite  
Priority: P0

Q: Name three costs Nanite does not erase.  
A: Material/pixel shading, lighting/shadows and scene/memory/streaming work; translucency and deformation also remain constrained.  
Category: Rendering / Nanite  
Priority: P0

Q: Why must Nanite support claims be version-labelled?  
A: Supported geometry, materials, renderers, platforms, VR, deformation and ray tracing evolve substantially across UE5 releases.  
Category: Rendering / Nanite  
Priority: P1

Q: What does Lumen provide?  
A: Fully dynamic diffuse global illumination and reflections using screen traces plus software or hardware scene tracing.  
Category: Rendering / Lumen  
Priority: P1

Q: What does Lumen software ray tracing depend on?  
A: Signed distance-field-based scene representations, with target-version configuration and limits.  
Category: Rendering / Lumen  
Priority: P1

Q: Why inspect Lumen visualisations before raising quality?  
A: A missing/bad scene or surface-cache representation may be the cause, and more samples need not repair it.  
Category: Rendering / Lumen  
Priority: P1

Q: What is the VSM caching model?  
A: Allocate/render virtual shadow pages needed by visible pixels and reuse unchanged pages across frames.  
Category: Rendering / VSM  
Priority: P1

Q: What commonly invalidates VSM pages?  
A: Moving/changing lights, moving or WPO-deformed casters, visibility changes and affected bounds.  
Category: Rendering / VSM  
Priority: P0

Q: Why can non-Nanite VSM casters be costly?  
A: They can retain conventional shadow draw-call/geometry pressure, especially over broad coarse-page regions.  
Category: Rendering / VSM  
Priority: P1

Q: What is RDG?  
A: Unreal's Render Dependency Graph for recording passes/resources and compiling their dependencies into executable render work.  
Category: Rendering / RDG  
Priority: P1

Q: Why do RDG pass parameters declare resources?  
A: RDG derives dependencies, transitions and resource lifetimes from declared reads and writes.  
Category: Rendering / RDG  
Priority: P1

Q: Transient versus external RDG resource?  
A: A transient resource belongs to graph lifetime and may alias memory; an external resource crosses the graph boundary.  
Category: Rendering / RDG  
Priority: P2

Q: What is a classic RDG lifetime bug?  
A: Capturing stack or other short-lived data that expires before the deferred execution lambda uses it.  
Category: Rendering / RDG  
Priority: P2

Q: What makes a good GPU hypothesis?  
A: It names the pass, causal mechanism, controlled variable and predicted timing response.  
Category: Rendering / Diagnosis  
Priority: P0

Q: Why is disabling a rendering feature not automatically a fix?  
A: It locates a cost contributor but may violate visual requirements and does not identify the efficient shipping intervention.  
Category: Rendering / Diagnosis  
Priority: P0

Q: What should a rendering before/after report preserve?  
A: Target setup, camera/workload, quality/correctness, repeated timing distribution and checks for cost shifting to CPU or memory.  
Category: Rendering / Method  
Priority: P0

Q: What should a packaged rendering proof packet contain?  
A: Build identity, platform/RHI/settings, deterministic map or camera path, cold/warm-cache note, pass timings and any RDG/GPU capture links.  
Category: Rendering / Production Proof  
Priority: P1

Q: Why is editor-only rendering proof weak?  
A: Module startup, shader registration, PSO coverage, scene-capture paths and cache state can differ in packaged builds.  
Category: Rendering / Production Proof  
Priority: P1

Q: Custom shader pass black in packaged build first asks what?  
A: Did the pass compile/load and appear with the expected event name in packaged evidence?  
Category: Rendering / Shader Programming  
Priority: P2

Q: Constant-colour output test is useful for what custom-pass bug class?  
A: It separates binding/registration/resource issues from algorithmic shader math.  
Category: Rendering / Shader Programming  
Priority: P2

Q: Shader-directory mapping bug often appears where?  
A: In packaged or branch-specific startup paths even when editor development looks fine.  
Category: Rendering / Shader Programming  
Priority: P2

Q: `ShouldCompilePermutation` guards what?  
A: Which platform/feature/permutation variants compile rather than letting unsupported paths fail later.  
Category: Rendering / Shader Programming  
Priority: P2

Q: Render-target memory cost depends on what beyond resolution?  
A: Format, mips, cube faces, history buffers, active count, lifetime and buffering.  
Category: Rendering / Render Targets  
Priority: P1

Q: Scene-capture view cost depends mainly on what?  
A: What the capture renders, how often it updates and how many capture views are active.  
Category: Rendering / Scene Capture  
Priority: P1

Q: How do you isolate capture rendering from UI/material sampling cost?  
A: Disable or slow capture updates while keeping the consuming UI/material path alive.  
Category: Rendering / Scene Capture  
Priority: P2

Q: SceneCaptureCube worst-case cost multiplier versus one 2D capture?  
A: Roughly six directions/views before extra format or mip costs.  
Category: Rendering / Scene Capture  
Priority: P1

Q: Memory spike but modest GPU pass time suggests what?  
A: Too many or too-large live render targets, history buffers or cube faces rather than only expensive shading.  
Category: Rendering / Render Targets  
Priority: P2

Q: GPU capture metadata must include what workload facts?  
A: Build/config, platform/RHI/GPU/driver, map/scenario path, settings and whether caches were warm or cold.  
Category: Rendering / GPU Capture  
Priority: P2

Q: Why keep Unreal-side evidence with an external GPU capture?  
A: It explains why that frame was chosen and maps the capture back to an Unreal-level fix.  
Category: Rendering / GPU Capture  
Priority: P2

Q: A RenderDoc frame alone does not prove what?  
A: That the same conclusion generalises to console/mobile/vendor-specific tooling and targets.  
Category: Rendering / GPU Capture  
Priority: P2

Q: Good rendering interview habit after naming a pass?  
A: Classify whether the issue is correctness, first-use hitch, steady-state GPU cost, render-thread churn or memory lifetime.  
Category: Rendering / Interview Strategy  
Priority: P1

## Patterns and System Design

Q: What should precede choosing a design pattern?  
A: The concrete change, lifetime, authority, timing, scale, failure and tooling forces.  
Category: Patterns / Method  
Priority: P0

Q: What does Single Responsibility mean usefully?  
A: Cohesion around one bounded reason/change axis, not one tiny class per function.  
Category: Architecture / SOLID  
Priority: P1

Q: When is Open/Closed harmful?  
A: When speculative extension points cost more than editing a small stable branch.  
Category: Architecture / SOLID  
Priority: P1

Q: What does Liskov substitution test?  
A: Whether a subtype preserves the behavioural contract/invariants expected through its base type.  
Category: Architecture / SOLID  
Priority: P1

Q: Dependency inversion versus dependency injection?  
A: Inversion concerns dependency direction toward stable abstractions; injection is one wiring mechanism.  
Category: Architecture / SOLID  
Priority: P1

Q: When does Component beat inheritance?  
A: When capabilities vary independently or must be reused across unrelated Actor types.  
Category: Patterns / Component  
Priority: P0

Q: What must a UE Component owner contract state?  
A: Required owner capabilities, lifecycle, authority, replication, events and teardown/reset rules.  
Category: Patterns / Component  
Priority: P0

Q: What signals a bad Component graph?  
A: Many sibling casts, bidirectional dependencies and hidden UI/save/global-service reach-through.  
Category: Patterns / Component  
Priority: P1

Q: Are normal UE delegate broadcasts asynchronous?  
A: No; they normally invoke bound listeners synchronously unless work is explicitly queued.  
Category: Patterns / Observer  
Priority: P0

Q: Why not rely on multicast delegate listener order?  
A: Notification order is not a robust business-logic dependency contract.  
Category: Patterns / Observer  
Priority: P1

Q: What is a re-entrant Observer bug?  
A: A listener mutates/broadcasts the subject during its original broadcast, violating transition assumptions.  
Category: Patterns / Observer  
Priority: P1

Q: When is a direct call clearer than Observer?  
A: When one required collaborator must produce an immediate result or success/failure.  
Category: Patterns / Communication  
Priority: P0

Q: What does Command represent?  
A: An intention/request as a value or object that can be queued, logged, replayed, validated or undone.  
Category: Patterns / Command  
Priority: P1

Q: Why use stable IDs in persisted/network Commands?  
A: Raw object pointers are process/world-lifetime references and cannot safely identify later targets.  
Category: Patterns / Command  
Priority: P1

Q: Does a client Command make execution authoritative?  
A: No; the server still validates ownership, state, target, rate and rules.  
Category: Patterns / Command  
Priority: P0

Q: State versus Strategy in one line?  
A: State models current mode/transitions; Strategy substitutes an algorithm/policy.  
Category: Patterns / Behaviour  
Priority: P0

Q: What do enter/exit methods need to own?  
A: State-specific timers, tasks, delegates and cancellation/cleanup.  
Category: Patterns / State  
Priority: P1

Q: What do many overlapping state booleans risk?  
A: Impossible combinations and unclear transition ownership.  
Category: Patterns / State  
Priority: P1

Q: When is an enum branch better than Strategy objects?  
A: When variants are few, stable, simple and do not need independent authoring/testing/state.  
Category: Patterns / Strategy  
Priority: P1

Q: What should a Factory centralise?  
A: Concrete selection plus invariant creation/setup, authority, loading and registration.  
Category: Patterns / Factory  
Priority: P1

Q: Why is duplicating a live Actor not a safe Prototype by default?  
A: World registration, ownership, subobjects, references, replication and transient state need supported creation contracts.  
Category: Patterns / Prototype  
Priority: P2

Q: What does Object Pool trade?  
A: Allocation/construction churn for retained peak memory, reset/lifetime and stale-handle complexity.  
Category: Patterns / Pool  
Priority: P0

Q: Name three Actor pool reset concerns.  
A: Timers/delegates, collision/movement/effects, and owner/instigator/replication state.  
Category: Patterns / Pool  
Priority: P1

Q: What does a Subsystem solve better than an unscoped singleton?  
A: It binds service instance lifetime to an Engine, GameInstance, World, LocalPlayer or Editor host.  
Category: Architecture / Services  
Priority: P1

Q: What does a Subsystem not solve?  
A: Hidden dependencies, god-service scope, wrong-world lookup and test coupling.  
Category: Architecture / Services  
Priority: P0

Q: Delegate versus Event Queue timing?  
A: A delegate normally notifies now; a queue deliberately processes later.  
Category: Patterns / Event Queue  
Priority: P0

Q: What must a bounded queue define at capacity?  
A: Backpressure, drop, replace/coalesce or overflow policy plus metrics.  
Category: Patterns / Event Queue  
Priority: P1

Q: Why are events not durable state?  
A: Late/missed listeners cannot infer current truth reliably from transient notifications.  
Category: Architecture / Events  
Priority: P0

Q: What are the three data-driven layers?  
A: Shared authored definition, mutable runtime instance/state and local presentation.  
Category: Architecture / Data-Driven  
Priority: P0

Q: Data Asset versus DataTable?  
A: Data Assets suit individual rich referenced definitions; DataTables suit homogeneous row schemas/bulk tabular balance data.  
Category: Architecture / Data-Driven  
Priority: P1

Q: What should save/replication store for data-defined items?  
A: Stable definition ID plus mutable instance state, not a repeated definition or live pointer.  
Category: Architecture / Data-Driven  
Priority: P0

Q: What is Flyweight in an item system?  
A: Shared immutable item definition plus separate per-stack/instance quantity, durability and owner state.  
Category: Patterns / Flyweight  
Priority: P1

Q: What is Dirty Flag?  
A: Mark cached derived data stale when inputs change and recompute once at a controlled read/phase.  
Category: Patterns / Dirty Flag  
Priority: P1

Q: What is Dirty Flag's classic bug?  
A: Missing invalidation leaves a stale cache that appears valid.  
Category: Patterns / Dirty Flag  
Priority: P1

Q: What five contracts summarise a gameplay design?  
A: Definition, state, operations, notifications and persistence.  
Category: System Design / Method  
Priority: P0

Q: Why should UI not own gameplay truth?  
A: Widgets are local/recreatable presentation and cannot enforce authoritative lifecycle, replication or persistence.  
Category: System Design / UI  
Priority: P0

Q: What is an inventory container revision for?  
A: Detecting stale operations/responses and rebuilding UI from authoritative state.  
Category: System Design / Inventory  
Priority: P1

Q: Why separate backpack contents from equipped appearance?  
A: Private owner data and public world-visible projection have different replication audiences.  
Category: System Design / Inventory  
Priority: P0

Q: What should a committed damage result contain?  
A: Old/new value, applied amount, reason/source and transition such as death.  
Category: System Design / Health  
Priority: P1

Q: What are interaction's four core stages?  
A: Discovery/selection, read-only eligibility/query, authoritative request validation/execution, and feedback.  
Category: System Design / Interaction  
Priority: P0

Q: Why query quest-relevant current state on activation?  
A: Events before subscription may be missed, while durable state still answers what is true now.  
Category: System Design / Quests  
Priority: P1

Q: Why not identify quest objectives by array index?  
A: Designer reordering changes indices and corrupts persisted progress identity.  
Category: System Design / Quests  
Priority: P1

Q: Why is save/load not a memory dump?  
A: Runtime pointers, caches, worlds and content versions do not survive; records need stable IDs, schema and migration.  
Category: System Design / Persistence  
Priority: P0

Q: Why use a controlled restore transaction?  
A: Normal setters/events may re-award rewards or trigger transient side effects while dependencies are incomplete.  
Category: System Design / Persistence  
Priority: P1

Q: What makes an architecture debug log useful?  
A: Correlation/operation ID, authority/world, prior revision, validation result, committed delta and resulting revision.  
Category: Architecture / Debugging  
Priority: P0

Q: What is a good refactoring signal for Strategy/data definitions?  
A: A central conditional repeatedly changes for each new content variant along a stable variation axis.  
Category: Architecture / Refactoring  
Priority: P1

Q: What is a representation-mismatch signal?  
A: Thousands of similar Actors/Components where batch, manager, data-oriented or Mass representation fits better.  
Category: Architecture / Performance  
Priority: P1

## Assets, Loading, and Cooking

Q: What is an asset dependency closure?  
A: The transitive set of packages/objects loaded or managed because a root references them.  
Category: Assets / Dependencies  
Priority: P0

Q: Are hard asset references inherently bad?  
A: No; they are appropriate for small required co-resident dependencies with understood closure.  
Category: Assets / References  
Priority: P0

Q: What does `TSoftObjectPtr` hold conceptually?  
A: Typed indirect object identity/path that can resolve loaded content or be loaded on demand.  
Category: Assets / References  
Priority: P0

Q: `TSoftObjectPtr` versus `TWeakObjectPtr`?  
A: Soft can identify unloaded asset content; weak only observes a current UObject instance lifetime.  
Category: Assets / References  
Priority: P0

Q: `TSoftClassPtr` versus `TSoftObjectPtr`?  
A: Soft class identifies a loadable class constrained by base type; soft object identifies an object asset.  
Category: Assets / References  
Priority: P1

Q: Does a soft reference automatically load its target?  
A: No; it supplies identity for explicit resolution/loading.  
Category: Assets / References  
Priority: P0

Q: Does a soft reference guarantee cook inclusion?  
A: No; inclusion must follow supported reference/Asset Manager/cook rules and be verified.  
Category: Assets / Cooking  
Priority: P0

Q: Does a soft reference keep a loaded asset resident?  
A: Not by itself; a strong reference or managed load handle must own required residency.  
Category: Assets / Residency  
Priority: P0

Q: When is synchronous asset loading reasonable?  
A: In controlled loading/startup/tools phases with explicit latency budget, not surprise hot gameplay paths.  
Category: Assets / Loading  
Priority: P1

Q: What guards reordered async requests?  
A: A request generation/token checked before completion commits state.  
Category: Assets / Async Loading  
Priority: P0

Q: Why capture a weak owner in async completion?  
A: The owner or World can be destroyed/unloaded before the request finishes.  
Category: Assets / Async Loading  
Priority: P0

Q: Why can an async load still hitch?  
A: Serialisation/fixups, resource creation and callback-side spawn/registration may still block critical threads.  
Category: Assets / Profiling  
Priority: P0

Q: What does the Asset Registry return?  
A: `FAssetData` metadata records that can be filtered without loading asset UObjects.  
Category: Assets / Registry  
Priority: P1

Q: What is dangerous about `FAssetData::GetAsset()`?  
A: It loads the represented asset if not already loaded.  
Category: Assets / Registry  
Priority: P0

Q: Why may editor Asset Registry results be incomplete initially?  
A: Asset discovery/gathering can occur asynchronously.  
Category: Assets / Registry  
Priority: P1

Q: What identifies a Primary Asset?  
A: `FPrimaryAssetId`, composed of Primary Asset Type and Name.  
Category: Assets / Asset Manager  
Priority: P1

Q: Primary versus Secondary Asset?  
A: Primary is directly managed by ID/rules; Secondary is reached/managed through references from primaries.  
Category: Assets / Asset Manager  
Priority: P1

Q: What is an Asset Bundle?  
A: A named group of soft dependencies on a Primary Asset loaded for a purpose/phase.  
Category: Assets / Asset Manager  
Priority: P1

Q: What can a Primary Asset Label control?  
A: Management/cook/chunk rules for explicit, collection or directory-selected assets.  
Category: Assets / Cooking  
Priority: P1

Q: What is a cook chunk?  
A: A numbered asset collection that can be built/distributed as a separate content archive/container group.  
Category: Assets / Cooking  
Priority: P1

Q: What does Build do in packaging workflow?  
A: Compiles target code, plugins and binaries.  
Category: Assets / Packaging  
Priority: P1

Q: What does Cook do?  
A: Converts/selects assets into target-platform runtime formats.  
Category: Assets / Cooking  
Priority: P0

Q: What does Stage do?  
A: Copies compiled and cooked outputs/configuration into a staging layout.  
Category: Assets / Packaging  
Priority: P1

Q: Why can an asset exist in Content Browser but not a package?  
A: Editor loose discovery does not prove that cook inclusion rules selected it.  
Category: Assets / Cooking  
Priority: P0

Q: Why is “cook everything” a weak missing-asset fix?  
A: It hides management defects and inflates install size, memory risk and patch/chunk content.  
Category: Assets / Cooking  
Priority: P0

Q: What is an asset redirector?  
A: An old-path object left after move/rename that redirects references to the new asset.  
Category: Assets / Redirectors  
Priority: P1

Q: What does Fix Up Redirectors do?  
A: Resaves referencer packages to the new path and removes resolvable redirectors.  
Category: Assets / Redirectors  
Priority: P1

Q: Should redirectors be deleted manually from disk?  
A: No; use editor/commandlet fix-up with source-control-aware package resaves.  
Category: Assets / Redirectors  
Priority: P0

Q: What is DDC?  
A: A cache hierarchy for regenerable platform/settings-derived asset data.  
Category: Assets / DDC  
Priority: P1

Q: Is DDC authoritative source content?  
A: No; source inputs should regenerate it.  
Category: Assets / DDC  
Priority: P0

Q: What does World Partition stream around?  
A: Runtime grid cells around active streaming sources, subject to grid/loading/Data Layer rules.  
Category: Assets / World Partition  
Priority: P1

Q: What does `Is Spatially Loaded` govern conceptually?  
A: Whether an Actor's load depends on spatial streaming-source range versus non-spatial rules.  
Category: Assets / World Partition  
Priority: P1

Q: Why are hard cross-cell Actor references risky?  
A: They can bundle/co-load Actors and defeat expected streaming granularity.  
Category: Assets / World Partition  
Priority: P1

Q: Where should durable quest/inventory state live in a streamed world?  
A: In the correct persistent authoritative lifetime, not only on spatial Actors that unload.  
Category: Assets / Architecture  
Priority: P0

Q: What six markers help profile loading?  
A: Request, I/O/start, package/object ready, resource ready, callback commit and first visible use.  
Category: Assets / Profiling  
Priority: P1

Q: Why test cold and warm loading?  
A: Warm DDC/OS/storage/object caches can hide real first-run and target-device stalls.  
Category: Assets / Profiling  
Priority: P0

## Build Systems, Modules, Plugins, and Editor Tools

Q: What is UBT responsible for?  
A: Evaluating targets/modules/descriptors and constructing/invoking the Unreal compile/link build graph.  
Category: Build / UBT  
Priority: P0

Q: What is UHT responsible for?  
A: Parsing supported reflected declarations and generating reflection glue/metadata code for native compilation.  
Category: Build / UHT  
Priority: P0

Q: Is UHT Unreal's C++ compiler?  
A: No; the platform compiler still compiles C++, after UHT generates required code.  
Category: Build / UHT  
Priority: P0

Q: What does the linker prove?  
A: That referenced compiled symbols can be resolved from linked objects/libraries with compatible definitions/exports.  
Category: Build / Linking  
Priority: P0

Q: What is authoritative: `.sln` or UBT rules?  
A: UBT's Target.cs/Build.cs/descriptors; IDE projects are generated editing/invocation metadata.  
Category: Build / UBT  
Priority: P0

Q: What does `.Target.cs` describe?  
A: An application target such as Game, Client, Server, Editor or Program and its global rules.  
Category: Build / Targets  
Priority: P1

Q: What does `.Build.cs` describe?  
A: One module's dependencies, include/definition/library and platform build contract.  
Category: Build / Modules  
Priority: P0

Q: Are Build.cs files runtime C# scripting?  
A: No; they are C# rules evaluated by UBT while constructing the native build.  
Category: Build / Modules  
Priority: P0

Q: Target versus configuration?  
A: Target selects application composition; configuration selects Debug/Development/Test/Shipping-style optimisation/debug behaviour.  
Category: Build / Targets  
Priority: P1

Q: What is an Unreal module?  
A: A separately built functionality/API unit participating in a directed dependency and load graph.  
Category: Build / Modules  
Priority: P0

Q: Are Unreal modules C++20 modules?  
A: No.  
Category: Build / Modules  
Priority: P1

Q: When is a module dependency public?  
A: When the dependency's types/includes are exposed by this module's public headers/API to consumers.  
Category: Build / Dependencies  
Priority: P0

Q: When is a module dependency private?  
A: When used only in private headers/cpp implementation and not exposed to consumers.  
Category: Build / Dependencies  
Priority: P0

Q: Public/Private folders versus C++ access specifiers?  
A: Folders control cross-module header visibility; access specifiers control class member access.  
Category: Build / Modules  
Priority: P0

Q: Why prefer private dependencies where possible?  
A: They reduce transitive API coupling and compile/rebuild fan-out.  
Category: Build / Dependencies  
Priority: P1

Q: Why is another module's Private include path a smell?  
A: It bypasses encapsulation and creates brittle unsupported implementation coupling.  
Category: Build / Dependencies  
Priority: P0

Q: Include path versus module dependency?  
A: Include path locates headers; dependency supplies the module's compile/link environment and graph relation.  
Category: Build / Dependencies  
Priority: P1

Q: What does a module API macro do?  
A: Exports/imports public binary symbols across modular build boundaries.  
Category: Build / Linking  
Priority: P1

Q: Header compiles but unresolved external—first model?  
A: Declaration is visible, but matching definition/export/owning linked binary is absent.  
Category: Build / Linking  
Priority: P0

Q: What can unity builds hide?  
A: Missing direct includes, macro/order dependence and some ODR problems through combined translation units.  
Category: Build / IWYU  
Priority: P1

Q: When is a forward declaration insufficient?  
A: When code needs complete size/layout, inheritance, inline member use or template/destructor semantics requiring the definition.  
Category: Build / C++  
Priority: P1

Q: What belongs in a Runtime module?  
A: Code and reflected types valid for packaged game/client/server targets without editor-only dependencies.  
Category: Build / Architecture  
Priority: P0

Q: What belongs in an Editor module?  
A: Asset authoring, validators, customisations and Unreal Editor UI/integration code.  
Category: Build / Architecture  
Priority: P0

Q: Does `WITH_EDITOR` make an Editor module dependency Shipping-safe?  
A: No; the module dependency graph must itself keep Runtime independent of Editor modules.  
Category: Build / Architecture  
Priority: P0

Q: Which way should editor/runtime dependency point?  
A: Editor may depend on Runtime; Runtime must not depend on Editor.  
Category: Build / Architecture  
Priority: P0

Q: What does a `.uplugin` descriptor provide?  
A: Plugin discovery/metadata, module host/load declarations, dependencies, compatibility and content capability.  
Category: Build / Plugins  
Priority: P1

Q: What controls whether a plugin can contain content?  
A: Its descriptor's content capability plus correct content/cook management.  
Category: Build / Plugins  
Priority: P1

Q: What does module type control?  
A: Which hosts/targets may compile/load the module, such as Runtime versus Editor.  
Category: Build / Modules  
Priority: P1

Q: What does loading phase control?  
A: When a compatible module is loaded during application startup.  
Category: Build / Modules  
Priority: P1

Q: Why not set every plugin module to PreDefault?  
A: Earlier loading costs startup and can mask dependency/registration defects rather than solve them.  
Category: Build / Modules  
Priority: P1

Q: What must editor module shutdown mirror?  
A: Startup registrations: delegates, tabs, menus, customisations, styles, commands, validators and resources.  
Category: Editor Tools / Lifecycle  
Priority: P0

Q: Why avoid force-loading dependencies during ShutdownModule?  
A: Shutdown order may have unloaded them; loading new modules while tearing down creates lifecycle hazards.  
Category: Editor Tools / Lifecycle  
Priority: P1

Q: What is Live Coding?  
A: Runtime native binary recompilation/patching with optional UObject reinstancing.  
Category: Build / Live Coding  
Priority: P0

Q: What is object reinstancing?  
A: Replacing existing reflected object instances so structural code changes can be represented after reload.  
Category: Build / Live Coding  
Priority: P1

Q: Why can Live Coding stale a pointer cache?  
A: Reinstancing replaces objects while raw/static cached addresses may still reference old instances.  
Category: Build / Live Coding  
Priority: P0

Q: Which changes strongly favour clean restart?  
A: Reflected layout, constructors/default subobjects, build/descriptors, static lifetime/destructors and shutdown-sensitive state.  
Category: Build / Live Coding  
Priority: P0

Q: Editor Utility Widget best fit?  
A: Rapid designer-facing interactive editor tabs/workflows using UMG-style authoring.  
Category: Editor Tools / Utility  
Priority: P1

Q: Unreal editor Python best fit?  
A: Editor/pipeline/DCC automation and batch glue—not packaged gameplay scripting.  
Category: Editor Tools / Python  
Priority: P1

Q: Commandlet best fit?  
A: Repeatable headless batch or CI work in a reduced environment without assumed World/UI.  
Category: Editor Tools / Commandlets  
Priority: P1

Q: What should a safe batch tool do before mutation?  
A: Analyse, dry-run/preview and validate type, write/source-control, collision and dependency preconditions.  
Category: Editor Tools / Safety  
Priority: P0

Q: Why should batch repair be idempotent?  
A: Rerunning after interruption/partial failure should not compound changes or corrupt state.  
Category: Editor Tools / Safety  
Priority: P1

Q: Validation versus repair?  
A: Validation reports side-effect-free invariant failures; repair is an explicit previewed transactional operation.  
Category: Editor Tools / Validation  
Priority: P0

Q: `IsDataValid` versus Editor Validator?  
A: The asset class owns its invariants via `IsDataValid`; validators handle broader/cross-cutting asset rules.  
Category: Editor Tools / Validation  
Priority: P1

Q: Why use property handles in details customisations?  
A: They preserve multi-edit, transactions, notifications, defaults and editor property semantics.  
Category: Editor Tools / Details  
Priority: P1

Q: What proves runtime/editor separation?  
A: Successful clean Game/Server/Shipping builds and package/run with Editor modules/content excluded.  
Category: Build / Verification  
Priority: P0

## ECS and MassEntity

Q: What is MassEntity?  
A: UE5's plugin/module-based data-oriented entity framework for batch processing fragment compositions.  
Category: MassEntity / Core  
Priority: P2

Q: What workload best suits Mass?  
A: Large populations with few stable data shapes and batchable update/access patterns.  
Category: MassEntity / Architecture  
Priority: P2

Q: Does Mass replace Actors?  
A: No; Actors remain rich world objects and can serve as selected Mass representations.  
Category: MassEntity / Architecture  
Priority: P2

Q: What is a Mass entity?  
A: Manager-local identity/handle associated with a fragment/tag composition.  
Category: MassEntity / Core  
Priority: P2

Q: Is a Mass entity a UObject?  
A: No.  
Category: MassEntity / Core  
Priority: P2

Q: What is a Mass fragment?  
A: An atomic data aspect consumed in batch calculations; logic belongs in processors.  
Category: MassEntity / Fragments  
Priority: P2

Q: Should fragments contain behaviour?  
A: No; keep them data-oriented and process them through systems/processors.  
Category: MassEntity / Fragments  
Priority: P2

Q: What is a Mass tag?  
A: A zero-payload composition element whose presence/absence is queryable state.  
Category: MassEntity / Tags  
Priority: P2

Q: Why are Mass tags not free booleans?  
A: Adding/removing them changes composition and migrates entities between archetypes.  
Category: MassEntity / Tags  
Priority: P2

Q: What is an archetype?  
A: The group/storage definition for entities with exactly the same composition.  
Category: MassEntity / Storage  
Priority: P2

Q: What is a Mass chunk?  
A: Storage for archetype entities with contiguous arrays/columns for each fragment type.  
Category: MassEntity / Storage  
Priority: P2

Q: What creates archetype explosion?  
A: Many independent optional composition/tag/shared-value combinations.  
Category: MassEntity / Storage  
Priority: P2

Q: Why does chunk occupancy matter?  
A: Sparse/small chunks reduce contiguous batch efficiency and increase iteration/metadata overhead.  
Category: MassEntity / Performance  
Priority: P2

Q: What is a shared fragment?  
A: Configuration/data shared by a group rather than repeated for every entity.  
Category: MassEntity / Fragments  
Priority: P2

Q: What is a chunk fragment?  
A: One data value associated with a chunk, often for batch/managerial metadata.  
Category: MassEntity / Fragments  
Priority: P2

Q: Why avoid high-cardinality shared fragment values?  
A: They split entities into many groups/archetypes and reduce batching.  
Category: MassEntity / Performance  
Priority: P2

Q: What is a Mass EntityQuery?  
A: Declared fragment/tag/subsystem presence and access requirements over cached matching archetypes.  
Category: MassEntity / Queries  
Priority: P2

Q: What do `All`, `Any`, `None`, and `Optional` express?  
A: Composition-presence requirements used to match archetypes/entities.  
Category: MassEntity / Queries  
Priority: P2

Q: Why declare ReadOnly versus ReadWrite accurately?  
A: It supports correctness, conflict analysis and potential parallel scheduling.  
Category: MassEntity / Queries  
Priority: P2

Q: What is a Mass processor?  
A: A typically stateless UObject that executes query-defined logic over matching entity chunks.  
Category: MassEntity / Processors  
Priority: P2

Q: Where should per-entity mutable state live?  
A: In fragments, not implicit processor-member arrays or iteration order.  
Category: MassEntity / Processors  
Priority: P2

Q: Does Mass automatically run processors off-thread?  
A: No; access requirements, processor settings, subsystems and UObject/World work constrain scheduling.  
Category: MassEntity / Threading  
Priority: P2

Q: Why avoid UObject calls in a hot chunk loop?  
A: They reintroduce indirection, game-thread constraints and per-entity overhead that defeat batching.  
Category: MassEntity / Performance  
Priority: P2

Q: Why defer Mass structural changes?  
A: Composition migration can invalidate active iteration/storage, so commands batch changes at safe points.  
Category: MassEntity / Structural Change  
Priority: P2

Q: Is deferred structural change free?  
A: No; command volume and archetype migrations can dominate when composition churns.  
Category: MassEntity / Structural Change  
Priority: P2

Q: What is a Mass Trait?  
A: A template-authoring recipe that adds/configures fragments, tags and shared data for a capability.  
Category: MassEntity / Authoring  
Priority: P2

Q: Does a Trait execute per-frame logic?  
A: No; processors execute runtime logic.  
Category: MassEntity / Authoring  
Priority: P2

Q: What creates a Mass entity template?  
A: A configured composition assembled from Traits/entity config data for spawning.  
Category: MassEntity / Authoring  
Priority: P2

Q: What are Mass Observers for?  
A: Reacting to supported entity composition add/remove operations, often for initialisation/cleanup.  
Category: MassEntity / Observers  
Priority: P3

Q: What is a Mass Signal conceptually?  
A: A lightweight named wake-up indication, event-like but without durable payload state.  
Category: MassEntity / Signals  
Priority: P3

Q: What representation choices can MassGameplay LOD use?  
A: High/low Actor, ISM, or no representation, subject to target-version support.  
Category: MassEntity / Representation  
Priority: P2

Q: Visual LOD versus simulation LOD?  
A: Visual LOD changes representation; simulation LOD changes update frequency/fidelity.  
Category: MassEntity / LOD  
Priority: P2

Q: What prevents LOD transition thrash?  
A: Hysteresis, maximum tier counts, transition budgets and stable significance rules.  
Category: MassEntity / LOD  
Priority: P2

Q: What must Actor promotion define?  
A: Stable entity mapping, state/transform ownership, transfer direction, pool reset and demotion cleanup.  
Category: MassEntity / Hybrid  
Priority: P2

Q: What is a dual-writer hybrid bug?  
A: Actor Tick and Mass processor both authoritatively update the same transform/state.  
Category: MassEntity / Hybrid  
Priority: P2

Q: Is `FMassEntityHandle` a save/network ID?  
A: No; use stable domain/network identity and validate manager-local handles.  
Category: MassEntity / Identity  
Priority: P2

Q: Can fragment views be cached after chunk execution?  
A: No; storage can move/change and views are scoped to execution.  
Category: MassEntity / Lifetime  
Priority: P2

Q: First checks when a processor misses an entity?  
A: Correct World/manager, valid handle, exact composition/query, registration/activation/phase, and command flush.  
Category: MassEntity / Debugging  
Priority: P2

Q: Four layers to profile in a hybrid Mass system?  
A: Simulation processors, structural changes, Actor/UObject bridges, and visual representation.  
Category: MassEntity / Performance  
Priority: P2

Q: What makes an Actor-versus-Mass benchmark fair?  
A: Equivalent behaviour/visual fidelity, camera/workload and separate simulation/representation measurements.  
Category: MassEntity / Performance  
Priority: P2

Q: When should Mass be avoided?  
A: Few unique UObject/Blueprint/physics-rich entities, high structural churn or no batchable data access/ownership capacity.  
Category: MassEntity / Architecture  
Priority: P2

Q: What must be verified before relying on Mass replication?  
A: Exact UE branch plugin status/APIs, stable identity, supported custom data, viewer LOD and platform needs.  
Category: MassEntity / Networking  
Priority: P3

## Gameplay Ability System

Q: When does GAS earn its complexity?  
A: Dense shared ability/effect/tag/attribute rules, designer authoring, cancellation and multiplayer prediction.  
Category: GAS / Architecture  
Priority: P2

Q: When is a bespoke ability Component preferable?  
A: A small bounded rule set with little stacking, cross-ability interaction or prediction need.  
Category: GAS / Architecture  
Priority: P2

Q: What does an ASC coordinate?  
A: Granted abilities, attributes, owned tags, active effects, cues and GAS networking.  
Category: GAS / Core  
Priority: P2

Q: ASC owner versus avatar?  
A: Owner logically owns durable state; avatar is the current physical actor abilities use.  
Category: GAS / Lifetime  
Priority: P2

Q: Why put an ASC on PlayerState?  
A: To preserve player ability state, attributes/effects/cooldowns across Pawn replacement when desired.  
Category: GAS / Lifetime  
Priority: P2

Q: Main PlayerState-owned ASC risk?  
A: Incorrect owner/avatar initialisation, replication/ownership timing, stale Pawn bindings and update-frequency assumptions.  
Category: GAS / Lifetime  
Priority: P2

Q: Who normally grants abilities?  
A: Authority; `GiveAbility` on a non-authoritative actor is ignored.  
Category: GAS / Abilities  
Priority: P2

Q: What is an ability spec?  
A: One runtime grant record in an ASC containing class plus grant/runtime data and a handle.  
Category: GAS / Abilities  
Priority: P2

Q: Why retain a spec handle?  
A: To identify/remove the intended grant when a class can have multiple sources/specs.  
Category: GAS / Abilities  
Priority: P2

Q: Three ability instancing policies?  
A: Non-Instanced, Instanced Per Actor and Instanced Per Execution.  
Category: GAS / Abilities  
Priority: P2

Q: Biggest Non-Instanced constraint?  
A: No per-activation mutable instance state; execution behaves through shared definition/CDO constraints.  
Category: GAS / Abilities  
Priority: P2

Q: Why must an activated ability explicitly end?  
A: Otherwise it may remain active, leak tasks/state and block other abilities.  
Category: GAS / Lifecycle  
Priority: P2

Q: What is commit timing?  
A: The deliberate irreversible point at which cost/cooldown is applied after sufficient validation.  
Category: GAS / Lifecycle  
Priority: P2

Q: Is cancellation the same as destruction?  
A: No; it is a termination path that must run explicit policy and cleanup.  
Category: GAS / Lifecycle  
Priority: P2

Q: Why are GAS tags not Booleans?  
A: They are hierarchical predicates and the same owned tag may have multiple counted sources.  
Category: GAS / Tags  
Priority: P2

Q: When is a loose tag safe?  
A: When one authoritative owner has explicit symmetric add/remove lifecycle and observability.  
Category: GAS / Tags  
Priority: P2

Q: What should tags not store?  
A: Magnitude, timestamps, ordered state, item identity or arbitrary runtime strings.  
Category: GAS / Tags  
Priority: P2

Q: Attribute base versus current value?  
A: Base is underlying value; current is evaluated after active modifiers.  
Category: GAS / Attributes  
Priority: P2

Q: When should a value be a GAS attribute?  
A: When effects, modifiers, capture, prediction or standard ASC observation should operate on it.  
Category: GAS / Attributes  
Priority: P2

Q: Why is clamping in one attribute hook risky?  
A: Base changes, executions, periodic effects and replication can follow different mutation paths.  
Category: GAS / Attributes  
Priority: P2

Q: Gameplay Effect definition versus spec?  
A: Definition is reusable data; spec binds level, context, captures and application-time values.  
Category: GAS / Effects  
Priority: P2

Q: Does an instant GE enter the active-effect container?  
A: No; it executes immediately rather than remaining active.  
Category: GAS / Effects  
Priority: P2

Q: What is special about a periodic GE?  
A: It remains active and executes its effects repeatedly at its period.  
Category: GAS / Effects  
Priority: P2

Q: What does set-by-caller provide?  
A: An application-time magnitude stored on a spec, which must be validated and server-authoritative.  
Category: GAS / Effects  
Priority: P2

Q: What does effect context answer?  
A: Who/what/where caused an application for attribution and calculation.  
Category: GAS / Effects  
Priority: P2

Q: Complete stacking policy dimensions?  
A: Aggregation owner, limit/overflow, duration/period refresh, expiration, magnitude, tags/cues and UI.  
Category: GAS / Effects  
Priority: P2

Q: Modifier versus execution calculation?  
A: Modifier handles direct value operations; execution handles multi-attribute authoritative outcomes.  
Category: GAS / Effects  
Priority: P2

Q: Typical GAS representation of a cooldown?  
A: A Gameplay Effect that grants a cooldown tag for a defined duration.  
Category: GAS / Costs and Cooldowns  
Priority: P2

Q: What is an Ability Task?  
A: Ability-scoped asynchronous work that reports through delegates and ends with work/ability lifecycle.  
Category: GAS / Tasks  
Priority: P2

Q: What is target data?  
A: Structured proposed target/hit/location evidence passed through ability targeting and networking.  
Category: GAS / Targeting  
Priority: P2

Q: Why must server validate client target data?  
A: Client proposal improves responsiveness but cannot authoritatively determine range, hit, cadence or damage.  
Category: GAS / Targeting  
Priority: P2

Q: What owns truth: Gameplay Cue or Gameplay Effect?  
A: Gameplay Effect/attribute/tag state; a cue is cosmetic presentation.  
Category: GAS / Cues  
Priority: P2

Q: What does Local Predicted mean?  
A: Owning client performs supported provisional work; server validates and confirms or rejects it.  
Category: GAS / Networking  
Priority: P2

Q: What is a prediction key for?  
A: Correlating provisional client actions/effects with server confirmation or rejection.  
Category: GAS / Networking  
Priority: P2

Q: Full GE replication mode?  
A: Full Gameplay Effect information goes to all relevant clients.  
Category: GAS / Networking  
Priority: P2

Q: Mixed GE replication mode?  
A: Full detail to owner/autonomous proxies and minimal detail to simulated proxies.  
Category: GAS / Networking  
Priority: P2

Q: Does Minimal GE replication disable attribute replication?  
A: No; it describes minimal GE information, not every ASC state path.  
Category: GAS / Networking  
Priority: P2

Q: First evidence for activation failure?  
A: Failing machine, ASC owner/avatar/actor info, spec handle and activation failure tags.  
Category: GAS / Debugging  
Priority: P2

Q: Why can GAS work on listen server but fail remotely?  
A: Listen server hides owning-client actor-info, ownership, RPC and prediction timing mistakes.  
Category: GAS / Debugging  
Priority: P2

Q: What should a GAS performance trace separate?  
A: Effects/tags/calculations/tasks, Blueprint work, cues/presentation and replication audiences.  
Category: GAS / Performance  
Priority: P2

## Algorithms and Data Structures

Q: First step before optimising an interview solution?  
A: Clarify constraints and give a correct baseline with time/space cost.  
Category: Algorithms / Problem Solving  
Priority: P0

Q: What must accompany a Big-O claim?  
A: Variable meaning, case/assumptions, time/space and build/update/query phase.  
Category: Algorithms / Complexity  
Priority: P0

Q: Amortised O(1) append means what?  
A: Total sequence averages constant per append though one growth may be linear and hitch.  
Category: Algorithms / Complexity  
Priority: P0

Q: Why can O(n) array scan beat O(log n) tree lookup?  
A: Small N, contiguous cache-friendly access and lower allocation/indirection constants.  
Category: Algorithms / Complexity  
Priority: P0

Q: Default container for hot iteration?  
A: Usually a contiguous dynamic array unless another requirement dominates.  
Category: Algorithms / Containers  
Priority: P0

Q: Swap-remove trade-off?  
A: Constant removal after index but destroys element order and moves the last element.  
Category: Algorithms / Containers  
Priority: P0

Q: Linked-list O(1) erase precondition?  
A: The target node/iterator is already known; finding it can still be linear.  
Category: Algorithms / Containers  
Priority: P0

Q: Queue versus stack?  
A: FIFO supports BFS/order; LIFO supports DFS/backtracking/undo.  
Category: Algorithms / Containers  
Priority: P0

Q: Ring-buffer essential policy?  
A: Define capacity-full behaviour: overwrite, reject, grow or back-pressure.  
Category: Algorithms / Containers  
Priority: P1

Q: Hash-table expected and worst lookup?  
A: Expected O(1) under good assumptions; worst O(n) under pathological collisions.  
Category: Algorithms / Hashing  
Priority: P0

Q: Required hash/equality relationship?  
A: Equal keys must produce equal hashes.  
Category: Algorithms / Hashing  
Priority: P0

Q: Why not mutate a stored hash key?  
A: Changing equality/hash can make it unreachable in its current bucket.  
Category: Algorithms / Hashing  
Priority: P0

Q: When use an ordered map/tree?  
A: When sorted iteration, ranges, predecessor/successor or worst-case logarithmic operations matter.  
Category: Algorithms / Trees  
Priority: P1

Q: Heap invariant?  
A: Every parent outranks its children according to the comparator.  
Category: Algorithms / Heaps  
Priority: P0

Q: Priority-queue top/push/pop costs?  
A: Top O(1); push and pop O(log n).  
Category: Algorithms / Heaps  
Priority: P0

Q: Simpler alternative to decrease-key in A*?  
A: Push improved duplicate entries and discard stale ones when popped.  
Category: Algorithms / Heaps  
Priority: P1

Q: Comparator strict-order bug example?  
A: Using `<=` so equal elements each claim to precede the other.  
Category: Algorithms / Sorting  
Priority: P0

Q: When not fully sort?  
A: For min/max, top-K, partition or kth selection where less work answers the requirement.  
Category: Algorithms / Sorting  
Priority: P0

Q: Binary-search core precondition?  
A: Range is sorted/partitioned consistently with the comparator.  
Category: Algorithms / Searching  
Priority: P0

Q: What does lower_bound return?  
A: First position whose element is not ordered before the target—its insertion point.  
Category: Algorithms / Searching  
Priority: P0

Q: Why use half-open binary-search bounds?  
A: Empty ranges and insertion at n are natural, with one consistent invariant.  
Category: Algorithms / Searching  
Priority: P0

Q: BFS frontier and guarantee?  
A: FIFO queue; shortest path by edge count in unweighted/equal-cost graph.  
Category: Algorithms / Graphs  
Priority: P0

Q: DFS frontier and typical uses?  
A: Stack; reachability, components, cycles and backtracking.  
Category: Algorithms / Graphs  
Priority: P0

Q: BFS/DFS adjacency-list complexity?  
A: O(V+E) time and O(V) auxiliary state.  
Category: Algorithms / Graphs  
Priority: P0

Q: When mark a BFS node visited?  
A: When enqueued/discovered, preventing duplicate frontier insertion.  
Category: Algorithms / Graphs  
Priority: P0

Q: How cover disconnected graph components?  
A: Start traversal from every still-unvisited vertex.  
Category: Algorithms / Graphs  
Priority: P0

Q: Dijkstra edge-weight requirement?  
A: Non-negative weights for its settled-cost reasoning.  
Category: Algorithms / Pathfinding  
Priority: P0

Q: A* priority formula?  
A: `f(n)=g(n)+h(n)`: known start cost plus estimated remaining cost.  
Category: Algorithms / Pathfinding  
Priority: P0

Q: A* with h=0 becomes what?  
A: Dijkstra's algorithm.  
Category: Algorithms / Pathfinding  
Priority: P0

Q: Admissible heuristic?  
A: It never overestimates true remaining optimal cost.  
Category: Algorithms / Pathfinding  
Priority: P0

Q: Pathfinding versus path following?  
A: Search finds a route; following/steering/avoidance executes it in a dynamic world.  
Category: Algorithms / Pathfinding  
Priority: P0

Q: Topological order exists for what graph?  
A: A directed acyclic graph.  
Category: Algorithms / Graphs  
Priority: P1

Q: Kahn cycle test?  
A: If emitted vertices are fewer than V after zero-indegree processing, a cycle exists.  
Category: Algorithms / Graphs  
Priority: P1

Q: Why deterministic topological tie-break?  
A: Multiple valid orders exist; stable scheduling/build output may require one repeatable choice.  
Category: Algorithms / Graphs  
Priority: P1

Q: Union–find operations?  
A: Find component representative and Union two components.  
Category: Algorithms / Connectivity  
Priority: P1

Q: Union–find optimisations?  
A: Path compression plus union by rank/size.  
Category: Algorithms / Connectivity  
Priority: P1

Q: What does ordinary DSU not answer?  
A: Shortest path and arbitrary dynamic edge deletion.  
Category: Algorithms / Connectivity  
Priority: P1

Q: Weighted-random prefix method?  
A: Sample in [0,total) and select first cumulative weight above the sample.  
Category: Algorithms / Sampling  
Priority: P1

Q: Prefix sums improve repeated weighted draws to what?  
A: O(n) build and O(log n) draw with binary search for static weights.  
Category: Algorithms / Sampling  
Priority: P1

Q: Why is a shuffle bag not independent weighted random?  
A: Removing consumed outcomes deliberately changes future probabilities to reduce streaks.  
Category: Algorithms / Sampling  
Priority: P1

Q: Spatial hash negative-coordinate trap?  
A: Truncation toward zero; use mathematical floor for cell coordinates.  
Category: Algorithms / Spatial  
Priority: P1

Q: Spatial-hash cell too small?  
A: Objects overlap many cells, increasing insertion/query/dedup work.  
Category: Algorithms / Spatial  
Priority: P1

Q: Spatial-hash cell too large?  
A: Buckets contain many false candidates, increasing narrowphase work.  
Category: Algorithms / Spatial  
Priority: P1

Q: Spatial partition worst-case?  
A: Severe clustering can put all objects together and restore quadratic pair tests.  
Category: Algorithms / Spatial  
Priority: P1

Q: Broadphase false-positive/negative policy?  
A: Conservative false positives are acceptable; false negatives are not.  
Category: Algorithms / Collision  
Priority: P1

Q: What does narrowphase do?  
A: Accurate shape/geometry tests on broadphase candidate pairs.  
Category: Algorithms / Collision  
Priority: P1

Q: Best correctness oracle for optimised neighbour search?  
A: All-pairs exact filtering on small generated inputs.  
Category: Algorithms / Testing  
Priority: P0

Q: Best correctness oracle for A* path cost?  
A: Dijkstra on the same non-negative graph.  
Category: Algorithms / Testing  
Priority: P0

Q: What should a spatial benchmark record?  
A: Build/update/query, candidates/hits, max occupancy, memory and P95/P99 across realistic distributions.  
Category: Algorithms / Performance  
Priority: P1

Q: Prefix-sum range-query trade-off?  
A: O(n) preprocessing enables O(1) static range sums, but updates need rebuilding or another structure.  
Category: Algorithms / Patterns  
Priority: P1

## Game Maths

Q: Unreal editor coordinate convention?  
A: Left-handed, Z-up; +X forward, +Y right, +Z up.  
Category: Game Maths / Conventions  
Priority: P0

Q: `FRotator` angle unit?  
A: Degrees.  
Category: Game Maths / Conventions  
Priority: P0

Q: Point versus direction transform?  
A: A point receives translation; a direction/vector does not.  
Category: Game Maths / Spaces  
Priority: P0

Q: Squared-distance optimisation rule?  
A: Compare distance squared with radius squared, never an unsquared threshold.  
Category: Game Maths / Vectors  
Priority: P0

Q: Unit-vector dot product range/meaning?  
A: [-1,1], from opposite through perpendicular to same direction.  
Category: Game Maths / Vectors  
Priority: P0

Q: FOV dot test?  
A: `dot(forward,toTargetUnit) >= cos(halfAngle)`.  
Category: Game Maths / Vectors  
Priority: P0

Q: Projection onto arbitrary b?  
A: `b * dot(a,b)/dot(b,b)`, for non-zero b.  
Category: Game Maths / Vectors  
Priority: P0

Q: Reflection about unit normal n?  
A: `v - 2*dot(v,n)*n`.  
Category: Game Maths / Vectors  
Priority: P0

Q: Cross-product operand swap?  
A: It negates the result: `a×b = -(b×a)`.  
Category: Game Maths / Vectors  
Priority: P0

Q: Cross magnitude meaning?  
A: `|a||b|sinθ`, the parallelogram area.  
Category: Game Maths / Vectors  
Priority: P0

Q: Robust signed angle around axis n?  
A: `atan2(n·(a×b), a·b)` after suitable projection/normalisation.  
Category: Game Maths / Angles  
Priority: P1

Q: Why clamp before acos?  
A: Floating error can push an intended cosine slightly outside [-1,1].  
Category: Game Maths / Robustness  
Priority: P0

Q: Orthonormal basis inverse operation?  
A: Dot the vector with each basis axis; transpose equals inverse for orthonormal rotation.  
Category: Game Maths / Spaces  
Priority: P0

Q: Why transform normals specially under non-uniform scale?  
A: Ordinary transform would not preserve perpendicularity; use inverse-transpose-style scaling then normalise.  
Category: Game Maths / Transforms  
Priority: P1

Q: Why does transform composition order matter?  
A: Rotation, scale and translation are non-commutative.  
Category: Game Maths / Transforms  
Priority: P0

Q: Zero-scale consequence?  
A: Transform is singular; ordinary inverse is undefined/unrepresentable.  
Category: Game Maths / Transforms  
Priority: P1

Q: What is gimbal lock?  
A: Euler parameterisation singularity where two rotation axes align and one independent degree is lost.  
Category: Game Maths / Rotation  
Priority: P0

Q: Quaternion q versus -q?  
A: They represent the same orientation.  
Category: Game Maths / Rotation  
Priority: P0

Q: Quaternion multiplication property?  
A: It composes rotations and is order-dependent/non-commutative.  
Category: Game Maths / Rotation  
Priority: P0

Q: Unit quaternion inverse?  
A: Its conjugate, reversing the represented rotation.  
Category: Game Maths / Rotation  
Priority: P1

Q: Slerp main property?  
A: Spherical interpolation with constant angular-rate parameterisation along an arc.  
Category: Game Maths / Interpolation  
Priority: P0

Q: Lerp formula?  
A: `a + t*(b-a)`.  
Category: Game Maths / Interpolation  
Priority: P0

Q: Why repeated `lerp(current,target,speed*dt)` is not constant speed?  
A: Each step is a fraction of remaining distance, producing easing.  
Category: Game Maths / Interpolation  
Priority: P0

Q: Frame-independent exponential alpha?  
A: `1 - exp(-lambda*dt)`.  
Category: Game Maths / Interpolation  
Priority: P1

Q: Constant-speed move step?  
A: Move at most `speed*dt` toward target and clamp at target.  
Category: Game Maths / Interpolation  
Priority: P0

Q: Velocity and acceleration dimensions?  
A: Velocity L/T; acceleration L/T².  
Category: Game Maths / Kinematics  
Priority: P0

Q: Impulse effect in simple linear model?  
A: `deltaVelocity = impulse / mass`.  
Category: Game Maths / Physics  
Priority: P1

Q: Closest-segment parameter?  
A: `clamp(dot(P-A,B-A)/|B-A|²,0,1)`, with zero-length fallback.  
Category: Game Maths / Geometry  
Priority: P0

Q: Ray equation?  
A: `p(t)=origin+t*direction`, with t≥0 for a ray.  
Category: Game Maths / Intersections  
Priority: P0

Q: Ray-plane denominator?  
A: `dot(planeNormal, rayDirection)`; near zero means parallel.  
Category: Game Maths / Intersections  
Priority: P0

Q: Ray-sphere roots both negative mean?  
A: Infinite line intersects, but sphere lies behind ray origin.  
Category: Game Maths / Intersections  
Priority: P0

Q: Ray starts inside sphere: desired hit?  
A: Usually the nearest non-negative exit root, according to query policy.  
Category: Game Maths / Intersections  
Priority: P1

Q: Ray–AABB slab hit condition?  
A: Intersect per-axis t intervals; accept when enter≤exit and exit≥0.  
Category: Game Maths / Intersections  
Priority: P0

Q: Zero ray direction on one AABB axis?  
A: Origin must already lie inside that axis slab.  
Category: Game Maths / Intersections  
Priority: P0

Q: Sweep versus overlap?  
A: Sweep tests motion over an interval; overlap tests one configuration.  
Category: Game Maths / Collision  
Priority: P1

Q: Rendering-space pipeline?  
A: Local→world→view→clip→perspective divide/NDC→viewport.  
Category: Game Maths / Rendering Spaces  
Priority: P1

Q: Why divide clip coordinates by w?  
A: Perspective divide converts homogeneous clip coordinates to NDC.  
Category: Game Maths / Rendering Spaces  
Priority: P1

Q: Perspective depth versus world distance?  
A: Depth is projected and generally non-linear with view distance.  
Category: Game Maths / Rendering Spaces  
Priority: P1

Q: What does UV represent?  
A: A 2D parameterisation of a surface for texture/data lookup.  
Category: Game Maths / Rendering  
Priority: P1

Q: Why are tangent normal maps mostly blue?  
A: Their local +Z component usually points outward from the surface.  
Category: Game Maths / Rendering  
Priority: P1

Q: Why disable sRGB for normal/roughness/mask textures?  
A: They encode linear data, not display colour requiring gamma decoding.  
Category: Game Maths / Rendering  
Priority: P1

Q: Where should lighting arithmetic occur?  
A: In linear-light space, not gamma/sRGB encoded values.  
Category: Game Maths / Rendering  
Priority: P1

Q: LWC does not eliminate what?  
A: Float narrowing, cancellation, rendering precision and need for local/translated spaces.  
Category: Game Maths / Robustness  
Priority: P1

Q: First debug step for wrong-space maths?  
A: Name/draw input and output spaces, origins and basis axes.  
Category: Game Maths / Debugging  
Priority: P0

## Animation Systems

Q: Who owns durable gameplay truth: AnimBP or gameplay?  
A: Gameplay; AnimBP owns pose/presentation derived from a bounded snapshot.  
Category: Animation / Architecture  
Priority: P1

Q: Default owner of Character displacement?  
A: CharacterMovement, unless an explicit root-motion contract supplies movement.  
Category: Animation / Movement  
Priority: P1

Q: Animation Sequence contains what?  
A: Time-based Skeleton bone transforms plus optional curves, notifies and root-motion data.  
Category: Animation / Assets  
Priority: P1

Q: Additive pose represents what?  
A: A pose delta relative to a defined base/reference pose.  
Category: Animation / Blending  
Priority: P1

Q: Aim Offset sample type?  
A: Typically mesh-space additive poses blended over an input pose.  
Category: Animation / Blending  
Priority: P1

Q: EventGraph thread?  
A: Game thread.  
Category: Animation / Threads  
Priority: P1

Q: AnimGraph can evaluate where?  
A: On worker threads when its path is eligible/safe.  
Category: Animation / Threads  
Priority: P1

Q: Safe source for worker AnimBP data?  
A: Supported Property Access or a project-owned stable snapshot/proxy.  
Category: Animation / Threads  
Priority: P1

Q: Why minimise AnimBP EventGraph polling?  
A: It runs sequentially on the game thread and repeated casts/queries scale poorly.  
Category: Animation / Performance  
Priority: P1

Q: AnimGraph relevance means what?  
A: Whether a node currently contributes/evaluates toward the final output pose.  
Category: Animation / Runtime  
Priority: P1

Q: State-machine best fit?  
A: Persistent pose regimes with continuously evaluated transitions.  
Category: Animation / State Machines  
Priority: P1

Q: Montage best fit?  
A: Directed discrete actions needing trigger/cancel, sections, slots, notifies or root motion.  
Category: Animation / Montages  
Priority: P1

Q: Why keep idle/walk/run in one Blend Space?  
A: They are a continuous parameterised locomotion regime, avoiding transition-state explosion.  
Category: Animation / Locomotion  
Priority: P1

Q: Conduit purpose?  
A: Centralise a shared transition branch decision among multiple animation states.  
Category: Animation / State Machines  
Priority: P2

Q: Blend Space solves what?  
A: Interpolates animation samples over one/two semantic input axes.  
Category: Animation / Locomotion  
Priority: P1

Q: Common strafing Blend Space input?  
A: Speed plus velocity direction expressed in actor/local space.  
Category: Animation / Locomotion  
Priority: P1

Q: Sync Group solves what?  
A: Aligns timing/semantic phase of related animations during blending.  
Category: Animation / Sync  
Priority: P1

Q: Sync leader affects what besides timing?  
A: It has priority for triggering animation notifies.  
Category: Animation / Sync  
Priority: P2

Q: Montage Slot purpose?  
A: An AnimGraph insertion point where montage pose affects the final graph.  
Category: Animation / Montages  
Priority: P1

Q: Same Slot Group montages commonly do what?  
A: Compete/override according to group playback rules.  
Category: Animation / Montages  
Priority: P1

Q: Notify versus Notify State?  
A: Instant timeline event versus begin/tick/end window.  
Category: Animation / Notifies  
Priority: P1

Q: Why not make damage depend only on a notify?  
A: Evaluation/blending/network/throttling can skip/duplicate timing; server action must validate and own outcome.  
Category: Animation / Notifies  
Priority: P1

Q: Root motion does what?  
A: Extracts root-bone displacement so it can drive character/movement rather than only mesh drift.  
Category: Animation / Root Motion  
Priority: P1

Q: Root-motion double-movement cause?  
A: CharacterMovement/code and extracted animation both apply the same displacement.  
Category: Animation / Root Motion  
Priority: P1

Q: Replicating montage/root motion proves what?  
A: Presentation/movement support, not replicated gameplay permission, hit or outcome.  
Category: Animation / Networking  
Priority: P1

Q: Motion Warping purpose?  
A: Adjust an authored root-motion window toward a validated runtime warp target.  
Category: Animation / Procedural  
Priority: P2

Q: IK input must state what?  
A: Goal space, chain/limits, solver order and LOD/contact policy.  
Category: Animation / IK  
Priority: P2

Q: First retarget checks?  
A: Source/target Skeletons, retarget root, chains/mapping and matching retarget poses.  
Category: Animation / Retargeting  
Priority: P2

Q: Why A-pose/T-pose mismatch matters?  
A: Retarget deltas start from different references, producing systematic limb deformation.  
Category: Animation / Retargeting  
Priority: P2

Q: Linked Anim Layer purpose?  
A: Semantic pose extension point overridden by modular runtime animation classes/assets.  
Category: Animation / Modularity  
Priority: P2

Q: Linked-layer asset benefit?  
A: Equipment/vehicle-specific animation assets can load/unload with their feature.  
Category: Animation / Modularity  
Priority: P2

Q: First montage-visible check after successful play?  
A: Required Slot node exists and is relevant in the final active graph path.  
Category: Animation / Debugging  
Priority: P1

Q: First foot-sliding measurement?  
A: Actual capsule speed versus source clip stride/play rate and Blend Space calibration.  
Category: Animation / Debugging  
Priority: P1

Q: Why IK may not fix foot sliding?  
A: Cause may be speed/phase/retarget/root-motion/network mismatch upstream.  
Category: Animation / Debugging  
Priority: P1

Q: Pose Watch helps with what?  
A: Isolating each selected AnimGraph node/layer's influence on output pose.  
Category: Animation / Debugging  
Priority: P1

Q: Rewind Debugger animation value?  
A: Records and scrubs states, variables, asset times, blends and pose behaviour around a failure.  
Category: Animation / Debugging  
Priority: P1

Q: Animation Insights helps with what?  
A: Trace-based animation state/value observation and performance profiling over time.  
Category: Animation / Performance  
Priority: P1

Q: Animation Budget Allocator is what?  
A: Plugin that significance-throttles skeletal animation updates/interpolation to fit a budget.  
Category: Animation / Performance  
Priority: P2

Q: Budget allocator key risk?  
A: Timing/pose quality or gameplay-adjacent events may break when updates are reduced/skipped.  
Category: Animation / Performance  
Priority: P2

Q: Animation performance layers to separate?  
A: Game-thread gathering, worker evaluate, bone finalisation, skinning, IK queries, assets and cosmetics.  
Category: Animation / Performance  
Priority: P1

Q: Motion Matching changes what?  
A: Locomotion pose selection uses a feature/database query rather than only hand-authored transitions.  
Category: Animation / Modern UE5  
Priority: P3

## Physics and Collision

Q: Four collision-enabled concepts?  
A: None, query-only, physics-only, or query-and-physics participation.  
Category: Physics / Filtering  
Priority: P1

Q: Object Type describes what?  
A: The intrinsic collision category of a PrimitiveComponent.  
Category: Physics / Filtering  
Priority: P1

Q: Trace channel describes what?  
A: A semantic query that each target profile responds to.  
Category: Physics / Filtering  
Priority: P1

Q: Collision profile packages what?  
A: Collision enabled, Object Type and response matrix as named policy.  
Category: Physics / Filtering  
Priority: P1

Q: Does Block imply Hit event?  
A: No; filtering and notification generation/path are separate.  
Category: Physics / Events  
Priority: P1

Q: Overlap event basic requirements?  
A: Compatible responses/query participation and overlap-event generation on the relevant components.  
Category: Physics / Events  
Priority: P1

Q: Why deduplicate physics callbacks?  
A: Multiple bodies, resting contacts, substeps and network paths can report repeated logical contact.  
Category: Physics / Events  
Priority: P1

Q: Trace by channel asks what?  
A: Which targets respond to this semantic trace channel.  
Category: Physics / Queries  
Priority: P1

Q: Object query asks what?  
A: Which colliders belong to selected Object Types.  
Category: Physics / Queries  
Priority: P1

Q: Line trace tests what?  
A: A zero-volume line segment.  
Category: Physics / Queries  
Priority: P1

Q: Sphere/capsule trace is what?  
A: A shape sweep from start to end.  
Category: Physics / Queries  
Priority: P1

Q: Overlap query tests what?  
A: Shape occupancy/intersection at one pose, not motion time of impact.  
Category: Physics / Queries  
Priority: P1

Q: Multi trace by channel typically stops where?  
A: It returns overlaps up to and including the first blocking hit.  
Category: Physics / Queries  
Priority: P1

Q: `FHitResult::Time` means what?  
A: Normalised impact position along the trace/sweep segment.  
Category: Physics / Hit Results  
Priority: P1

Q: Swept `Location` versus `ImpactPoint`?  
A: Safe shape location at contact versus point on hit geometry.  
Category: Physics / Hit Results  
Priority: P1

Q: `ImpactNormal` means what?  
A: Normal of the hit surface at contact.  
Category: Physics / Hit Results  
Priority: P1

Q: Initial penetration implication?  
A: Query starts overlapped; zero time/distance and depenetration semantics differ from ordinary impact.  
Category: Physics / Hit Results  
Priority: P1

Q: Simple collision is what?  
A: Primitive/convex approximation designed for robust cheaper gameplay queries/simulation.  
Category: Physics / Geometry  
Priority: P1

Q: Complex collision is what?  
A: Triangle-mesh geometry for detailed scene queries, with greater cost/restrictions.  
Category: Physics / Geometry  
Priority: P1

Q: Why not default every mesh to complex collision?  
A: Memory/query cost, snagging/degenerate triangles and simulation restrictions.  
Category: Physics / Geometry  
Priority: P1

Q: SetActorLocation with sweep is what kind of movement?  
A: Kinematic scene-query movement, not free rigid-body simulation.  
Category: Physics / Movement  
Priority: P1

Q: CharacterMovement's usual collision identity?  
A: Upright capsule with movement sweeps/floor/step/slide logic.  
Category: Physics / Movement  
Priority: P1

Q: One-transform-owner rule?  
A: CharacterMovement, root motion, solver, correction or cinematic owns a body at a time.  
Category: Physics / Architecture  
Priority: P1

Q: Force versus impulse?  
A: Force integrates over time; impulse changes momentum immediately.  
Category: Physics / Rigid Bodies  
Priority: P1

Q: Force at location adds what effect?  
A: Torque from the lever arm as well as linear force.  
Category: Physics / Rigid Bodies  
Priority: P1

Q: Mass versus inertia?  
A: Mass governs linear response; inertia tensor governs angular response.  
Category: Physics / Rigid Bodies  
Priority: P1

Q: Damping versus friction?  
A: Damping attenuates velocity; friction is contact tangential response.  
Category: Physics / Rigid Bodies  
Priority: P1

Q: Sleeping body purpose?  
A: Remove sufficiently quiet bodies from active simulation until a wake condition.  
Category: Physics / Performance  
Priority: P1

Q: Tunnelling cause?  
A: Fast/small body crosses thin geometry between discrete simulation samples.  
Category: Physics / CCD  
Priority: P1

Q: Tunnelling mitigations?  
A: Sweep/CCD, smaller steps, thicker/simple geometry or analytical projectile/hitscan.  
Category: Physics / CCD  
Priority: P1

Q: Substepping benefit/cost?  
A: Smaller solver steps improve stability/accuracy at added CPU/bookkeeping cost.  
Category: Physics / Timestep  
Priority: P1

Q: Does substepping guarantee determinism?  
A: No.  
Category: Physics / Timestep  
Priority: P1

Q: Why multiple callbacks in one game frame with substeps?  
A: Solver-step contacts are queued and delivered after substep processing.  
Category: Physics / Events  
Priority: P1

Q: First unstable-constraint check?  
A: Draw/verify both body-local constraint frames and intended axes/degrees.  
Category: Physics / Constraints  
Priority: P1

Q: Constraint stability stressors?  
A: Extreme mass ratios, tiny shapes, long chains, stiff/conflicting limits and large timestep.  
Category: Physics / Constraints  
Priority: P1

Q: Physical Material controls what broad domains?  
A: Contact properties such as friction/restitution and Surface Type classification.  
Category: Physics / Materials  
Priority: P1

Q: Physics Asset contains what?  
A: Skeletal Mesh physics bodies and constraints for collision/ragdoll/physical animation.  
Category: Physics / Ragdoll  
Priority: P1

Q: Ragdoll transition must transfer what?  
A: Transform/movement ownership and collision state from capsule/animation to Physics Asset simulation.  
Category: Physics / Ragdoll  
Priority: P1

Q: Safe get-up first requires what?  
A: Stable reference body/orientation and collision-safe capsule placement.  
Category: Physics / Ragdoll  
Priority: P1

Q: Server-authoritative throw request sends what?  
A: Validatable intent/target/input; server applies final force/state.  
Category: Physics / Networking  
Priority: P1

Q: Resimulation physics costs what?  
A: Cached state/input history, CPU resimulation, memory and correction presentation.  
Category: Physics / Networking  
Priority: P2

Q: Status of Predictive Interpolation/Resimulation for this curriculum?  
A: Highly version-sensitive specialist awareness until UE5.3–UE5.6 source support is verified.  
Category: Physics / Networking  
Priority: P2

Q: Chaos Visual Debugger can inspect what?  
A: Recorded bodies, shapes/channels, contacts, constraints and scene queries, subject to version.  
Category: Physics / Debugging  
Priority: P1

Q: First failed-trace workflow?  
A: Draw query, inspect exact component query enable/type/response/geometry/ignores, then recorded candidates.  
Category: Physics / Debugging  
Priority: P1

Q: Physics performance layers?  
A: Queries, awake bodies/broadphase/contacts, solver/constraints, callbacks, substeps, assets and network history.  
Category: Physics / Performance  
Priority: P1

## UI Systems

Q: `UUserWidget` lifetime has which three useful layers?  
A: UObject instance, underlying Slate representation, and screen/activity lifetime.  
Category: UI / Lifecycle  
Priority: P1

Q: True once-per-runtime-instance widget hook?  
A: `OnInitialized`.  
Category: UI / Lifecycle  
Priority: P1

Q: Why can `Construct` repeat?  
A: It follows construction of the underlying Slate widget, including hierarchy remove/re-add/rebuild use.  
Category: UI / Lifecycle  
Priority: P1

Q: Safe `PreConstruct` assumption?  
A: It may run at design time, so gameplay World/state and side effects cannot be assumed.  
Category: UI / Lifecycle  
Priority: P1

Q: Does Remove From Parent guarantee widget GC?  
A: No; surviving strong references can retain the UObject and it may rebuild.  
Category: UI / Lifetime  
Priority: P1

Q: Delegate-binding lifetime rule?  
A: Bind and unbind symmetrically at the boundary matching the source and screen lifetime.  
Category: UI / Lifetime  
Priority: P1

Q: Robust initial UI synchronisation pattern?  
A: Subscribe, render a complete snapshot, then apply deltas with revision/order protection.  
Category: UI / Architecture  
Priority: P1

Q: UI's relationship to gameplay truth?  
A: Local projection and intent source, never authoritative durable gameplay state.  
Category: UI / Architecture  
Priority: P1

Q: Legacy property bindings are usually what update model?  
A: Pull/evaluation paths, not inherently change-driven events.  
Category: UI / Performance  
Priority: P1

Q: Tick belongs in UI when?  
A: Genuine continuous active presentation needs it; disable it outside that lifetime.  
Category: UI / Performance  
Priority: P1

Q: Slate layout's first conceptual pass?  
A: Children compute/cache Desired Size bottom-up.  
Category: UI / Layout  
Priority: P1

Q: Slate layout's second conceptual pass?  
A: Parents arrange allotted geometry top-down.  
Category: UI / Layout  
Priority: P1

Q: Child asks, parent does what?  
A: Decides the child's final allotted geometry through its panel slot constraints.  
Category: UI / Layout  
Priority: P1

Q: Best default for content that must reflow?  
A: Flow panels such as horizontal/vertical/wrap/grid rather than fixed Canvas offsets.  
Category: UI / Layout  
Priority: P1

Q: What do split Canvas anchors enable?  
A: Stretching/offsets relative to a normalised parent region.  
Category: UI / Layout  
Priority: P1

Q: DPI curve solves what?  
A: Project-defined UI scaling across viewport dimensions; not all accessibility/layout concerns.  
Category: UI / Layout  
Priority: P1

Q: Safe Zone protects what?  
A: Critical content from platform/device unsafe or title-safe display regions.  
Category: UI / Layout  
Priority: P1

Q: Hidden versus Collapsed key difference?  
A: Hidden retains layout space; Collapsed does not.  
Category: UI / Visibility  
Priority: P1

Q: Why can a transparent overlay block a button?  
A: Visual transparency does not imply hit-test invisibility.  
Category: UI / Hit Testing  
Priority: P1

Q: Best tool for exact widget/hit-test/focus hierarchy?  
A: Widget Reflector.  
Category: UI / Debugging  
Priority: P1

Q: Focus belongs to what in local multiplayer?  
A: A specific Slate user/local player, not one global UI.  
Category: UI / Input  
Priority: P1

Q: Modal focus contract?  
A: Deterministic initial target, navigation/back policy and restoration on close/removal.  
Category: UI / Input  
Priority: P1

Q: CommonUI activation differs from construction how?  
A: A surviving widget can activate/deactivate repeatedly as interaction ownership changes.  
Category: UI / CommonUI  
Priority: P2

Q: CommonUI desired focus recommendation?  
A: Implement a valid desired focus target for activatable screens.  
Category: UI / CommonUI  
Priority: P2

Q: CommonUI integration danger?  
A: Competing direct controller/viewport input modes and mapping changes can fight activation input config.  
Category: UI / CommonUI  
Priority: P2

Q: MVVM status in this curriculum?  
A: Beta and plugin-dependent; useful FieldNotify push binding, not a mandatory architecture.  
Category: UI / MVVM  
Priority: P2

Q: Virtualised ListView durable identity?  
A: The item/model, not the generated entry widget.  
Category: UI / Lists  
Priority: P1

Q: Entry recycling requires what on every assignment?  
A: Full reset and rebind of visual state, subscriptions and requests.  
Category: UI / Lists  
Priority: P1

Q: Stale async icon guard?  
A: Weak destination plus stable item ID and matching request generation.  
Category: UI / Async  
Priority: P1

Q: Hard UI asset reference consequence?  
A: It can pull the referenced asset and transitive dependency closure into residency/load paths.  
Category: UI / Assets  
Priority: P1

Q: World-space WidgetComponent occlusion?  
A: It renders as scene geometry and can be occluded.  
Category: UI / World Widgets  
Priority: P2

Q: `Draw at Desired Size` scaling caution?  
A: Continuously resizing/redrawing world widgets can be expensive.  
Category: UI / World Widgets  
Priority: P2

Q: User-facing localisable Unreal text type?  
A: `FText`.  
Category: UI / Localisation  
Priority: P1

Q: Why avoid `FText`→`FString`→`FText` round-trips?  
A: They can lose text history needed for culture-aware reconstruction.  
Category: UI / Localisation  
Priority: P1

Q: Colour-only status is what accessibility issue?  
A: It lacks a redundant cue for users who cannot distinguish the colour.  
Category: UI / Accessibility  
Priority: P1

Q: Widgets replicate?  
A: No; replicate gameplay state and construct a projection for each local player.  
Category: UI / Networking  
Priority: P1

Q: Disabled client button is a security boundary?  
A: No; the server validates action intent and current state.  
Category: UI / Networking  
Priority: P1

Q: Invalidation purpose?  
A: Reuse cached UI work until a relevant hierarchy/layout/paint change invalidates it.  
Category: UI / Performance  
Priority: P1

Q: Volatile widget trade-off?  
A: Avoid repeated parent-cache invalidation by forfeiting paint caching for a frequently changing widget/subtree.  
Category: UI / Performance  
Priority: P2

Q: Retainer Panel trade-off?  
A: Lower/phased subtree redraw or draw work versus render-target memory, latency and composition/repaint cost.  
Category: UI / Performance  
Priority: P2

Q: Slate Insights frame view identifies what?  
A: Widgets painted, invalidated and updated in each traced Slate frame.  
Category: UI / Profiling  
Priority: P1

Q: UI performance layers?  
A: Model/binding, update, layout, hit test, paint/draw generation, render submission, assets and GPU composition/overdraw.  
Category: UI / Performance  
Priority: P1

## Audio Systems

Q: Sound Wave responsibility?  
A: Imported/sample audio content plus base playback/compression/loading/routing properties.  
Category: Audio / Sources  
Priority: P1

Q: Sound Cue responsibility?  
A: Sample-based Sound Node graph for authored variation/modulation/branching.  
Category: Audio / Sources  
Priority: P1

Q: MetaSound Source responsibility?  
A: Procedural DSP rendering graph with parameters and sample-accurate internal control.  
Category: Audio / Sources  
Priority: P1

Q: Who owns whether a gameplay sound event exists?  
A: Gameplay/presentation event policy; the source graph owns how it sounds.  
Category: Audio / Architecture  
Priority: P1

Q: Fire-and-forget fits when?  
A: A one-shot needs no later stop, fade, parameter, callback or attachment control.  
Category: Audio / Lifetime  
Priority: P1

Q: Persistent loop rule?  
A: One explicit owner and start/update/fade/stop state; never restart it each Tick.  
Category: Audio / Lifetime  
Priority: P1

Q: Auto-destroy caution?  
A: The Audio Component may be cleaned after finish/stop, so retained pointers need validity/lifetime discipline.  
Category: Audio / Lifetime  
Priority: P1

Q: Audio pool reset includes what?  
A: Sound, parameters, delegates, transform/attachment and playback/fade state.  
Category: Audio / Lifetime  
Priority: P2

Q: Attenuation includes more than volume—name four.  
A: Spatialisation, focus, occlusion/air absorption, priority and submix/reverb sends.  
Category: Audio / Spatialisation  
Priority: P1

Q: Occlusion is what model?  
A: A scheduled geometry-query-driven volume/filter approximation, not full acoustic simulation.  
Category: Audio / Spatialisation  
Priority: P1

Q: Distance/focus can modify what admission input?  
A: Effective sound priority.  
Category: Audio / Spatialisation  
Priority: P1

Q: Concurrency group models what?  
A: A semantic limit and resolution policy across related sound instances.  
Category: Audio / Concurrency  
Priority: P1

Q: Per-owner concurrency requires what?  
A: A meaningful owner on the playback request; absent ownership may fall back globally.  
Category: Audio / Concurrency  
Priority: P1

Q: Voice stealing means what?  
A: A playing/admitted source is replaced when a higher-policy request arrives under a full limit.  
Category: Audio / Concurrency  
Priority: P1

Q: Why add voice-steal release time?  
A: To fade an evicted source instead of cutting abruptly/clicking.  
Category: Audio / Concurrency  
Priority: P2

Q: Virtual source meaning?  
A: Logical playback/timeline can continue without consuming an audible rendered voice.  
Category: Audio / Concurrency  
Priority: P1

Q: Sound Class role?  
A: Hierarchical semantic grouping/control of source properties, user categories and mix policy.  
Category: Audio / Mixing  
Priority: P1

Q: Sound Mix role?  
A: Temporary/passive adjustment of Sound Class parameters during gameplay.  
Category: Audio / Mixing  
Priority: P1

Q: Submix role?  
A: Always-running signal mixing/routing graph with shared DSP and one downstream output per node.  
Category: Audio / Mixing  
Priority: P1

Q: Source effect cost scales mainly with what?  
A: Number of active source instances/voices using it.  
Category: Audio / DSP  
Priority: P1

Q: Submix effect advantage?  
A: Compatible shared processing occurs once on the combined bus instead of once per source.  
Category: Audio / DSP  
Priority: P1

Q: Submix silence guarantees zero cost?  
A: No; the graph is always running and effect behaviour must be profiled.  
Category: Audio / DSP  
Priority: P2

Q: Robust dialogue duck lifetime?  
A: Balanced/reference-counted overlapping requests with deliberate attack/release.  
Category: Audio / Mixing  
Priority: P1

Q: Normal game-thread play call guarantees sample timing?  
A: No; it crosses audio buffer/command timing and output latency.  
Category: Audio / Timing  
Priority: P1

Q: Quartz solves what?  
A: Ahead-of-time sample/musical scheduling across game and audio-buffer boundaries.  
Category: Audio / Timing  
Priority: P2

Q: Quartz removes device latency?  
A: No.  
Category: Audio / Timing  
Priority: P2

Q: Loaded SoundWave guarantees chunks ready?  
A: No; streamed compressed chunks/decoder state may still be unavailable.  
Category: Audio / Streaming  
Priority: P1

Q: Retain versus stream broad trade-off?  
A: Lower first-play/chunk latency versus higher resident memory.  
Category: Audio / Streaming  
Priority: P1

Q: Audio cache too small symptom?  
A: Aggressive eviction, chunk request pressure, first-play delay or underruns.  
Category: Audio / Streaming  
Priority: P1

Q: Best environment for audio memory/latency proof?  
A: Cooked target-platform build under representative concurrent streams/bursts.  
Category: Audio / Streaming  
Priority: P1

Q: Should Audio Components replicate?  
A: No; replicate semantic state/events and render audio locally.  
Category: Audio / Networking  
Priority: P1

Q: Prevent predicted plus replicated sound duplicate how?  
A: Correlate event/prediction identity and suppress or reconcile the owner confirmation.  
Category: Audio / Networking  
Priority: P1

Q: Late-join persistent loop source?  
A: Replicated durable gameplay state from which the client reconstructs local playback.  
Category: Audio / Networking  
Priority: P1

Q: Dedicated server audio rule?  
A: Gameplay correctness must not depend on an audio device or rendered sound.  
Category: Audio / Networking  
Priority: P1

Q: Missing-sound ordered layers?  
A: Request, asset/component, listener/attenuation, admission, gain/routing, stream/render, device.  
Category: Audio / Debugging  
Priority: P1

Q: Cut-outs only under battle load suggest what first?  
A: Concurrency, priority, voice stealing/virtualisation or stream-cache pressure.  
Category: Audio / Debugging  
Priority: P1

Q: First-play-only hitch suggests what?  
A: Cold chunk/decoder/setup/cache readiness rather than sustained DSP alone.  
Category: Audio / Debugging  
Priority: P1

Q: Quiet source can still cost what?  
A: Decode, spatialisation, source DSP, effects and a real voice unless culled/virtualised.  
Category: Audio / Performance  
Priority: P1

Q: Audio profiler population counters?  
A: Requests, active Components/logical sources, rejects, steals, virtual and rendered voices.  
Category: Audio / Performance  
Priority: P1

Q: Audio optimisation first move?  
A: Remove duplicate/restarted/unnecessary sources before tuning DSP or compression.  
Category: Audio / Performance  
Priority: P1

## Niagara and VFX

Q: Niagara System responsibility?  
A: Deploy/co-ordinate one effect containing Emitters, shared lifetime, parameters and scalability.  
Category: VFX / Architecture  
Priority: P1

Q: Niagara Emitter responsibility?  
A: Spawn, update and render one particle population within a System.  
Category: VFX / Architecture  
Priority: P1

Q: Why Module order matters?  
A: Modules read/write attributes sequentially; later writers consume or overwrite earlier results.  
Category: VFX / Architecture  
Priority: P1

Q: User namespace meaning?  
A: Typed parameter values intended to be set from outside the Niagara simulation.  
Category: VFX / Parameters  
Priority: P1

Q: System versus Particle parameter cardinality?  
A: One per System instance versus one value per particle.  
Category: VFX / Parameters  
Priority: P1

Q: Niagara group order broadly?  
A: System, Emitter, Particle, then Render consumption.  
Category: VFX / Execution  
Priority: P1

Q: Active execution state?  
A: Simulates and allows spawning.  
Category: VFX / Lifecycle  
Priority: P1

Q: Inactive execution state?  
A: Continues simulation but disallows new spawning.  
Category: VFX / Lifecycle  
Priority: P1

Q: InactiveClear execution state?  
A: Clears owned particles then transitions inactive.  
Category: VFX / Lifecycle  
Priority: P1

Q: Complete execution state?  
A: No simulation and no rendering.  
Category: VFX / Lifecycle  
Priority: P1

Q: GPU Sim Target moves which main scripts?  
A: Particle Spawn, Particle Update and supported Simulation Stages.  
Category: VFX / Simulation  
Priority: P1

Q: GPU simulation removes all Niagara CPU work?  
A: No; Components, System/Emitter work, instance management and submission remain.  
Category: VFX / Simulation  
Priority: P1

Q: GPU readback trade-off?  
A: CPU access arrives with latency, synchronisation and performance/debug overhead.  
Category: VFX / Simulation  
Priority: P1

Q: Data Interface purpose?  
A: Expose specialised external data/functions to supported Niagara execution targets.  
Category: VFX / Data Flow  
Priority: P1

Q: Data Channel fit?  
A: High-volume transient records between game/Niagara Systems, such as many impacts.  
Category: VFX / Data Flow  
Priority: P2

Q: Data Channel is authoritative replicated storage?  
A: No.  
Category: VFX / Data Flow  
Priority: P1

Q: Set spawn-critical User parameters when?  
A: Before activation/initial Spawn scripts consume them.  
Category: VFX / Parameters  
Priority: P1

Q: Spawn-attached exposes what lifecycle controls?  
A: Auto-destroy, auto-activate, pooling method and pre-cull, plus attachment transform.  
Category: VFX / Lifetime  
Priority: P1

Q: Niagara pooling reduces what?  
A: Repeated Component allocation and garbage collection.  
Category: VFX / Pooling  
Priority: P1

Q: Niagara pooling does not reduce what?  
A: Particle simulation and renderer/GPU work.  
Category: VFX / Pooling  
Priority: P1

Q: Pre-cull spawn caution?  
A: It may skip disposable FX, so callers cannot require a live Component or gameplay completion signal.  
Category: VFX / Scalability  
Priority: P1

Q: Too-small Niagara bounds symptom?  
A: Visible particles/ribbons pop when their origin/bounds leave visibility.  
Category: VFX / Bounds  
Priority: P1

Q: Too-large Niagara bounds cost?  
A: Weak frustum/occlusion culling and unnecessary render work.  
Category: VFX / Bounds  
Priority: P1

Q: Fixed Emitter bounds coordinate space?  
A: Local space.  
Category: VFX / Bounds  
Priority: P1

Q: Niagara Effect Type purpose?  
A: Shared significance, culling, budget, per-platform scalability and validation policy.  
Category: VFX / Scalability  
Priority: P1

Q: Significance answers what?  
A: Which effect instances retain existence/quality when budgets are constrained.  
Category: VFX / Scalability  
Priority: P1

Q: VFX degradation order principle?  
A: Preserve readable core; remove secondary particles, lights, distortion or renderers first.  
Category: VFX / Scalability  
Priority: P1

Q: Niagara collision can authorise gameplay damage?  
A: No; authoritative gameplay/physics owns hit and damage.  
Category: VFX / Gameplay Boundary  
Priority: P1

Q: Camera-depth collision limitation?  
A: It represents visible camera depth, not complete off-screen world geometry.  
Category: VFX / Collision  
Priority: P1

Q: Particle event fan-out risk?  
A: One population can cascade unbounded new Systems, audio, decals or callbacks.  
Category: VFX / Events  
Priority: P1

Q: Sprite Renderer common bottleneck?  
A: Translucent screen coverage, overlap, sorting and material pixel cost.  
Category: VFX / Rendering  
Priority: P1

Q: Mesh Renderer common bottleneck?  
A: Geometry, sections/materials, transforms and shadows.  
Category: VFX / Rendering  
Priority: P1

Q: Ribbon Renderer common costs?  
A: Tessellation/history/order, sorting and translucent pixels.  
Category: VFX / Rendering  
Priority: P1

Q: Particle Light danger?  
A: Multiplying dynamic lighting/radius/shadow cost across particles.  
Category: VFX / Rendering  
Priority: P1

Q: Component Renderer danger?  
A: Creates/updates Components with per-Component lifecycle/Tick overhead.  
Category: VFX / Rendering  
Priority: P1

Q: Translucent pixel cost model?  
A: Covered pixels × overlapping layers × shader cost × views.  
Category: VFX / GPU  
Priority: P1

Q: Best overdraw views?  
A: Shader Complexity and Quad Overdraw, corroborated with GPU timings.  
Category: VFX / GPU  
Priority: P1

Q: Lightweight Emitters introduced when?  
A: UE5.4.  
Category: VFX / Modern UE5  
Priority: P2

Q: Lightweight Emitter trade-off?  
A: Reduced state/tick/memory/authoring overhead for a constrained compatible feature set.  
Category: VFX / Modern UE5  
Priority: P2

Q: Niagara performance domains?  
A: Instance/Game Thread, CPU particles, GPU compute, Render Thread/submission, GPU graphics/overdraw and memory.  
Category: VFX / Performance  
Priority: P1

## Lua and C# Integration

Q: Unreal baseline gameplay languages?  
A: C++ and Blueprint; Lua/C# runtime gameplay requires a specific third-party or studio integration.  
Category: Scripting / Adoption  
Priority: P3

Q: Best reason to add another runtime?  
A: A measured iteration/content/modding/tooling need that outweighs interop/platform/maintenance cost.  
Category: Scripting / Adoption  
Priority: P3

Q: Safe script API shape?  
A: Coarse semantic commands, snapshots and events with stable IDs and explicit lifetime/thread/error rules.  
Category: Scripting / API  
Priority: P3

Q: Reflection binding limitation?  
A: It exposes supported reflected surface, not every C++-only engine API.  
Category: Scripting / Binding  
Priority: P3

Q: Generated binding trade-off?  
A: Better types/performance/IDE support versus generation, build, code-size and version maintenance.  
Category: Scripting / Binding  
Priority: P3

Q: Lua central container?  
A: Table, used conventionally for arrays, maps, objects and modules.  
Category: Lua / Language  
Priority: P3

Q: Lua false values?  
A: Only `nil` and `false`; zero and empty string are true.  
Category: Lua / Language  
Priority: P3

Q: Assigning nil to a table key does what?  
A: Removes that key.  
Category: Lua / Language  
Priority: P3

Q: `__index` common role?  
A: Delegated lookup enabling prototype/default/native-wrapper behaviour.  
Category: Lua / Metatables  
Priority: P3

Q: Closure lifetime risk?  
A: Captured tables/wrappers remain reachable through delegates/timers/module state.  
Category: Lua / Lifetime  
Priority: P3

Q: `require` normally caches where?  
A: `package.loaded`.  
Category: Lua / Modules  
Priority: P3

Q: Lua coroutine is a thread?  
A: A cooperative execution stack, not an OS worker thread.  
Category: Lua / Async  
Priority: P3

Q: Lua C API passes values how?  
A: Through a virtual stack on `lua_State`.  
Category: Lua / C API  
Priority: P3

Q: Native retaining a Lua value uses what concept?  
A: A registry reference that must later be released.  
Category: Lua / C API  
Priority: P3

Q: Full versus light userdata?  
A: Lua-managed userdata block/metatable versus bare pointer identity; neither guarantees UObject validity.  
Category: Lua / C API  
Priority: P3

Q: Protected Lua call purpose?  
A: Catch errors at a controlled host boundary and preserve traceback/product recovery.  
Category: Lua / Errors  
Priority: P3

Q: Lua GC sees UObject ownership graph?  
A: No; binding/native mechanisms must root or invalidate across heaps explicitly.  
Category: Lua / GC  
Priority: P3

Q: `__gc` should own product teardown?  
A: No; use explicit unbind/dispose/shutdown, with finalisation as fallback.  
Category: Lua / GC  
Priority: P3

Q: Cross-runtime cycle example?  
A: Native delegate → Lua closure → wrapper → binding/native object root.  
Category: Lua / GC  
Priority: P3

Q: UnLua and sluaunreal status?  
A: Distinct third-party Lua plugin ecosystems whose exact commit/platform support must be verified.  
Category: Lua / Plugins  
Priority: P3

Q: Puerts Unreal language?  
A: JavaScript/TypeScript, not Lua or C#.  
Category: Scripting / Plugins  
Priority: P3

Q: C# reference-type examples?  
A: Classes, arrays, strings and delegates.  
Category: C# / Language  
Priority: P3

Q: Managed GC roots include what interop root?  
A: GC handles.  
Category: C# / GC  
Priority: P3

Q: Why pin briefly?  
A: Preventing managed-object movement can impair/fragment compacting GC.  
Category: C# / GC  
Priority: P3

Q: Deterministic native-resource release in C#?  
A: `IDisposable`/`using`, preferably wrapping handles with SafeHandle.  
Category: C# / Lifetime  
Priority: P3

Q: Finalizer role?  
A: Nondeterministic safety net, not normal lifecycle control.  
Category: C# / Lifetime  
Priority: P3

Q: Blittable type meaning?  
A: Compatible managed/native bit representation requiring no conversion copy.  
Category: C# / Interop  
Priority: P3

Q: C# bool/C++ bool trap?  
A: Default marshalling widths/representations may differ; ABI must be explicit.  
Category: C# / Interop  
Priority: P3

Q: Native-retained managed callback requirement?  
A: Exact signature/calling convention, rooted delegate, Game Thread policy and symmetric unregister/free.  
Category: C# / Interop  
Priority: P3

Q: Managed exception crossing native callback?  
A: Catch at the boundary, log managed context and return explicit native failure.  
Category: C# / Errors  
Priority: P3

Q: C# Task permits UObject access on continuation?  
A: Not unless explicitly dispatched to Game Thread and revalidated.  
Category: C# / Async  
Priority: P3

Q: Game Thread deadlock pattern?  
A: Blocking on a Task that needs a continuation dispatched back to Game Thread.  
Category: C# / Async  
Priority: P3

Q: UnrealCLR status?  
A: Third-party .NET integration snapshot; exact UE/platform/activity/API support requires verification.  
Category: C# / Plugins  
Priority: P3

Q: Native AOT removes all platform risk?  
A: No; plugin support, native integration, trimming/reflection and platform policy remain.  
Category: C# / Packaging  
Priority: P3

Q: Native AOT dynamic-loading limitation?  
A: No dynamic assembly loading or runtime code generation such as Reflection.Emit.  
Category: C# / Packaging  
Priority: P3

Q: Hot reload is what transaction?  
A: Quiesce, cancel/unbind, serialise, invalidate, load/validate, migrate/rebind, resume or rollback.  
Category: Scripting / Reload  
Priority: P3

Q: Old callback rejection key?  
A: Runtime/VM/assembly generation plus owner/World identity.  
Category: Scripting / Reload  
Priority: P3

Q: Safe untrusted-script exposure?  
A: Allow-list façade plus resource/API quotas; full reflection is not a sandbox.  
Category: Scripting / Security  
Priority: P3

Q: Replicate script VM heap?  
A: No; replicate versioned semantic state/IDs through native authority contracts.  
Category: Scripting / Networking  
Priority: P3

Q: Mixed-runtime profile layers?  
A: Boundary calls/marshalling, VM/managed CPU, allocations/GC, wrappers/roots, reload/startup and package/platform cost.  
Category: Scripting / Performance  
Priority: P3

## Standard C++ Specialist Depth

Q: Forwarding-reference exact form?  
A: `T&&` where cv-unqualified `T` is deduced for that function call.  
Category: C++ / Templates  
Priority: P0

Q: Lvalue passed to forwarding reference deduces T as what?  
A: An lvalue reference type; reference collapsing preserves lvalueness.  
Category: C++ / Templates  
Priority: P0

Q: `std::forward<T>` purpose?  
A: Preserve the originally deduced value category for a downstream call.  
Category: C++ / Templates  
Priority: P0

Q: Double-forwarding risk?  
A: The first consumer may move from the argument, so the second sees consumed state.  
Category: C++ / Templates  
Priority: P1

Q: SFINAE meaning?  
A: Substitution failure removes a candidate instead of being a hard error in its immediate context.  
Category: C++ / Templates  
Priority: P1

Q: Concept meaning?  
A: Named compile-time requirement used to constrain templates and order candidates.  
Category: C++ / Templates  
Priority: P1

Q: Concepts cannot prove what?  
A: Runtime semantics, complexity, thread safety or laws such as strict weak ordering.  
Category: C++ / Templates  
Priority: P1

Q: `if constexpr` role?  
A: Compile-time branch selection with non-instantiation of a dependent discarded branch.  
Category: C++ / Templates  
Priority: P1

Q: Lambda `[this]` captures what?  
A: The pointer, not ownership or lifetime of the object.  
Category: C++ / Lambdas  
Priority: P0

Q: Async capture safest default?  
A: Explicit copied immutable data or weak/stable handles with execution-time validation.  
Category: C++ / Lambdas  
Priority: P0

Q: Init-capture use?  
A: Rename/copy/construct or move ownership into the closure explicitly.  
Category: C++ / Lambdas  
Priority: P1

Q: `std::function` provides what?  
A: Owning copyable type erasure for compatible callable targets.  
Category: C++ / Callables  
Priority: P1

Q: Empty `std::function` invocation?  
A: Throws `std::bad_function_call`; check policy/emptiness before call.  
Category: C++ / Callables  
Priority: P1

Q: Function pointer versus template callable?  
A: Simple stateless runtime target versus compile-time typed/inlinable callable.  
Category: C++ / Callables  
Priority: P1

Q: Object representation may contain what beyond value bytes?  
A: Padding bytes.  
Category: C++ / Object Model  
Priority: P0

Q: Alignment requirement means what?  
A: Object address must satisfy a type-specific byte boundary.  
Category: C++ / Object Model  
Priority: P0

Q: Why not raw-serialise a struct?  
A: Padding, layout, endianness, pointers/vptr and schema differ by ABI/version.  
Category: C++ / Object Model  
Priority: P0

Q: AoS benefit?  
A: Cohesive access when operations consume most fields of each record.  
Category: C++ / Data Layout  
Priority: P1

Q: SoA benefit?  
A: Contiguous hot fields for bandwidth, cache and SIMD-friendly population loops.  
Category: C++ / Data Layout  
Priority: P1

Q: False sharing?  
A: Independent cross-thread writes contend because values occupy the same cache line.  
Category: C++ / Cache  
Priority: P1

Q: Monotonic arena lifetime?  
A: Allocations are reclaimed together when resource releases/dies; per-allocation deallocate is a no-op.  
Category: C++ / Allocation  
Priority: P1

Q: Arena escape bug?  
A: A pointer/reference survives after the arena storage is bulk released.  
Category: C++ / Allocation  
Priority: P1

Q: Pool main additional correctness need?  
A: Complete reset plus generation/stale-handle protection.  
Category: C++ / Allocation  
Priority: P1

Q: `monotonic_buffer_resource` thread-safe?  
A: No.  
Category: C++ / Allocation  
Priority: P1

Q: Data race consequence in C++?  
A: Undefined behaviour.  
Category: C++ / Concurrency  
Priority: P0

Q: `volatile` provides inter-thread synchronisation?  
A: No; it is neither atomic nor a memory-order edge.  
Category: C++ / Concurrency  
Priority: P0

Q: Mutex protects what broader concept?  
A: A compound invariant/critical section, not merely one variable.  
Category: C++ / Concurrency  
Priority: P0

Q: Unnamed scoped lock bug?  
A: Temporary is destroyed immediately, so the intended scope is not protected.  
Category: C++ / Concurrency  
Priority: P1

Q: Condition-variable truth lives where?  
A: In mutex-protected predicate state, not in the notification.  
Category: C++ / Concurrency  
Priority: P1

Q: Predicate wait handles what?  
A: Spurious wake-ups and notification timing by rechecking state under lock.  
Category: C++ / Concurrency  
Priority: P1

Q: Relaxed atomic guarantees?  
A: Atomicity and modification order for that atomic, without publishing unrelated memory.  
Category: C++ / Atomics  
Priority: P1

Q: Release/acquire publication?  
A: Acquire observing release makes preceding producer writes visible to the consumer.  
Category: C++ / Atomics  
Priority: P1

Q: Several atomics create transactional invariant?  
A: No.  
Category: C++ / Atomics  
Priority: P0

Q: Lock-free means wait-free?  
A: No; operations may retry indefinitely under contention.  
Category: C++ / Atomics  
Priority: P1

Q: Good task input?  
A: Immutable/copied snapshot or partition with exclusive ownership.  
Category: C++ / Tasks  
Priority: P1

Q: Good parallel output pattern?  
A: Per-task/chunk local outputs followed by bounded deterministic merge.  
Category: C++ / Tasks  
Priority: P1

Q: Releasing UE task handle cancels execution?  
A: No; scheduler can retain the task through completion.  
Category: C++ / Tasks  
Priority: P1

Q: Task prerequisite versus nested task?  
A: Prerequisite controls execution readiness; nested work delays parent completion.  
Category: C++ / Tasks  
Priority: P2

Q: ODR broad rule?  
A: Entities need the permitted number/equivalence of definitions across the program.  
Category: C++ / Linking  
Priority: P0

Q: `inline` guarantees machine-code inlining?  
A: No; it chiefly permits ODR-compliant multiple definitions.  
Category: C++ / Linking  
Priority: P0

Q: Template definition often belongs in header why?  
A: Compiler needs definition visible when instantiating, absent explicit instantiation strategy.  
Category: C++ / Linking  
Priority: P0

Q: Header namespace-scope `static` state risk?  
A: Creates a separate internal-linkage object in each translation unit.  
Category: C++ / Linking  
Priority: P1

Q: Static initialisation order fiasco?  
A: Cross-translation-unit dynamic globals depend on unspecified/indeterminate initialisation order.  
Category: C++ / Linking  
Priority: P0

Q: ASan primary domain?  
A: Use-after-free and out-of-bounds/related memory access faults.  
Category: C++ / Tooling  
Priority: P1

Q: TSan primary domain?  
A: Data-race detection in instrumented executed code.  
Category: C++ / Tooling  
Priority: P1

Q: UBSan examples?  
A: Signed overflow, invalid shift, null/misalignment, bounds and vptr misuse checks.  
Category: C++ / Tooling  
Priority: P1

Q: Sanitizer-clean proves correctness?  
A: No; only executed instrumented supported paths and selected fault classes were checked.  
Category: C++ / Tooling  
Priority: P1

Q: Unreal maintained C++ baseline currently says what?  
A: C++20, but pin UE5.3–UE5.6 branch/project/toolchain rather than assuming maintained UE5.7 policy.  
Category: C++ / Unreal Policy  
Priority: P1

Q: Exceptions/RTTI in Unreal modules?  
A: Commonly disabled by default, but verify exact module/toolchain settings and never infer language guarantees from policy.  
Category: C++ / Unreal Policy  
Priority: P1

Q: Best first question for any UE specifier?  
A: Which engine system consumes this specifier, and at what lifecycle phase?  
Category: UE C++ / Specifiers  
Priority: P0

Q: `BlueprintReadOnly` guarantees C++ immutability?  
A: No; it only controls Blueprint write access.  
Category: UE C++ / Blueprint API  
Priority: P0

Q: `VisibleAnywhere` controls Blueprint access?  
A: No; it controls editor details visibility, not Blueprint read/write permission.  
Category: UE C++ / Property Specifiers  
Priority: P0

Q: `EditDefaultsOnly` best fits what?  
A: Designer-authored class/default tuning rather than per-instance runtime mutation.  
Category: UE C++ / Property Specifiers  
Priority: P0

Q: `EditInstanceOnly` best fits what?  
A: Per-placed-instance authoring where instance overrides are intentional.  
Category: UE C++ / Property Specifiers  
Priority: P1

Q: `Config` needs what class-level support?  
A: A class config specifier such as `Config=Game` or relevant inherited config setup.  
Category: UE C++ / Config  
Priority: P0

Q: `Config` is not what?  
A: A replicated runtime settings or authoritative gameplay-state system.  
Category: UE C++ / Config  
Priority: P0

Q: `SaveGame` alone does not solve what?  
A: Stable IDs, schema migration, live Actor references, asset availability or archive policy.  
Category: UE C++ / SaveGame  
Priority: P0

Q: `Transient` means what?  
A: Excluded from normal persistence/serialisation paths, not "short C++ lifetime".  
Category: UE C++ / Persistence  
Priority: P0

Q: `DuplicateTransient` protects against what class of bug?  
A: Runtime/cache state being copied into duplicates or PIE/editor copies.  
Category: UE C++ / Duplication  
Priority: P1

Q: Metadata primary role?  
A: Editor, tooling and Blueprint node guidance, not runtime gameplay authority.  
Category: UE C++ / Metadata  
Priority: P0

Q: Metadata runtime rule?  
A: Do not depend on metadata for gameplay authority, security, networking or save rules.  
Category: UE C++ / Metadata  
Priority: P0

Q: `UINTERFACE` has which two sides?  
A: A reflected `U...` interface type and an `I...` C++ implementation interface.  
Category: UE C++ / Interfaces  
Priority: P0

Q: Interface versus Component?  
A: Interface expresses capability; Component can own reusable state and behaviour.  
Category: UE C++ / Interfaces  
Priority: P0

Q: Interface call implies network authority?  
A: No; it is still a local call unless wrapped in proper RPC/authority flow.  
Category: UE C++ / Interfaces  
Priority: P0

Q: Reflected interface good use?  
A: Cross-UObject capability that may need Blueprint/reflection participation.  
Category: UE C++ / Interfaces  
Priority: P0

Q: Native abstract interface good use?  
A: Pure C++ boundary that does not need UObject/reflection/Blueprint participation.  
Category: UE C++ / Interfaces  
Priority: P1

Q: `Outer` primary meaning?  
A: Containment and identity/path context.  
Category: UObject / Identity  
Priority: P0

Q: `Outer` does not guarantee what?  
A: A GC-retaining strong reference edge.  
Category: UObject / Identity  
Priority: P0

Q: `Within=...` means what?  
A: Objects of that class require an Outer of the specified class.  
Category: UObject / Identity  
Priority: P1

Q: Object identity logging should include what?  
A: Name, class, Outer, package and flags, plus stable gameplay ID when present.  
Category: UObject / Debugging  
Priority: P1

Q: `NewObject` is for what?  
A: Creating runtime UObjects with explicit Outer/name/flags/template and retention policy.  
Category: UE C++ / Object APIs  
Priority: P0

Q: Actor creation path?  
A: `SpawnActor` or deferred spawn through a World, not `NewObject`.  
Category: UE C++ / Object APIs  
Priority: P0

Q: `FindObject` is not what?  
A: A load or asset streaming policy.  
Category: UE C++ / Object APIs  
Priority: P1

Q: `LoadObject` main risk?  
A: Blocking load plus path/cook correctness problems.  
Category: UE C++ / Object APIs  
Priority: P1

Q: `DuplicateObject` review focus?  
A: Instanced subobjects, transient fields, source references and duplicate-specific flags.  
Category: UE C++ / Object APIs  
Priority: P1

Q: Good log category practice?  
A: Use subsystem categories and verbosity filters instead of broad `LogTemp` spam.  
Category: UE C++ / Logging  
Priority: P0

Q: `UE_LOGFMT` advantage?  
A: Structured named or positional fields for clearer correlated diagnostics.  
Category: UE C++ / Logging  
Priority: P1

Q: `UE_LOGFMT` named and positional fields can mix?  
A: No.  
Category: UE C++ / Logging  
Priority: P1

Q: `check` is for what?  
A: Impossible programmer invariants where continuing is unsafe.  
Category: UE C++ / Assertions  
Priority: P0

Q: `ensure` is for what?  
A: Unexpected but survivable state that should report diagnostic evidence.  
Category: UE C++ / Assertions  
Priority: P0

Q: `verify` differs from `check` how?  
A: Its expression still executes where check-style assertions may be disabled.  
Category: UE C++ / Assertions  
Priority: P0

Q: Assertions should not replace what?  
A: Normal validation paths for content, network, input, save or user-driven failure.  
Category: UE C++ / Assertions  
Priority: P0

Q: `TOptional` expresses what?  
A: Presence or absence of a value without overloading a sentinel.  
Category: UE C++ / Core Vocabulary  
Priority: P1

Q: `TOptional` is not what?  
A: A UObject ownership or lifetime model.  
Category: UE C++ / Core Vocabulary  
Priority: P1

Q: `TVariant` expresses what?  
A: One value from a closed set of alternatives.  
Category: UE C++ / Core Vocabulary  
Priority: P1

Q: `TFunctionRef` ownership?  
A: Non-owning callable view, suitable for synchronous non-escaping callbacks.  
Category: UE C++ / Core Vocabulary  
Priority: P1

Q: Storing `TFunctionRef` for async use?  
A: A likely dangling-lifetime bug.  
Category: UE C++ / Core Vocabulary  
Priority: P1

Q: Array/span view main hazard?  
A: The underlying storage may die, reallocate or mutate before the view is used.  
Category: UE C++ / Core Vocabulary  
Priority: P1

Q: Async Blueprint proxy must define what?  
A: Retention, activation, cancellation, Game Thread completion, stale-generation checks and cleanup.  
Category: Blueprint / Async  
Priority: P1

Q: Async proxy early-GC cause?  
A: No retained reference keeps the proxy alive while the operation is in flight.  
Category: Blueprint / Async  
Priority: P1

Q: Async proxy leak cause?  
A: Delegate, timer, handle or subsystem array retains the proxy after completion/cancel.  
Category: Blueprint / Async  
Priority: P1

Q: Blueprint delegate broadcast thread?  
A: Marshal to the Game Thread before touching UObjects or broadcasting to Blueprint.  
Category: Blueprint / Async  
Priority: P0

Q: Stale async completion guard?  
A: Weak owner plus World/request generation/cancellation checks.  
Category: Blueprint / Async  
Priority: P1

Q: `FGCObject` is for what?  
A: Non-UObject native owners reporting UObject references to the garbage collector.  
Category: UObject / GC Bridge  
Priority: P1

Q: Default before `FGCObject` in gameplay?  
A: Prefer a UObject owner with reflected `UPROPERTY` references when possible.  
Category: UObject / GC Bridge  
Priority: P1

Q: `AddToRoot` main danger?  
A: Global retention until explicit removal, causing leaks or shutdown/travel retention.  
Category: UObject / GC Bridge  
Priority: P1

Q: Safe off-thread UObject workflow?  
A: Snapshot plain data on Game Thread, compute off-thread, revalidate and apply on Game Thread.  
Category: UE C++ / Threading  
Priority: P0

Q: Worker task touching gameplay UObjects?  
A: Unsafe unless a specific subsystem documents that access pattern.  
Category: UE C++ / Threading  
Priority: P0

Q: Releasing a UE task handle cancels execution?  
A: No; cancellation is a separate protocol.  
Category: UE C++ / Threading  
Priority: P1

Q: Deferred lambda raw `this` capture risk?  
A: Callback can run after owner destruction or travel.  
Category: UE C++ / Lambda Lifetime  
Priority: P0

Q: UHT cascade first response?  
A: Read the first real error and inspect the reflected declaration before the generated-code failure.  
Category: UE C++ / UHT Debugging  
Priority: P0

Q: Generated-code bug categories to separate?  
A: UHT parse, C++ compile, link/export, Blueprint compile and runtime consuming-system mismatch.  
Category: UE C++ / UHT Debugging  
Priority: P0

Q: UHT prefix sensitivity source?  
A: Epic coding standard notes UHT requires correct prefixes in many cases.  
Category: UE C++ / UHT Debugging  
Priority: P1

Q: Specifier compiles but behaviour is wrong, ask what?  
A: Which consumer failed: details panel, Blueprint node, config/save archive, duplication, replication, GC, cook or package.  
Category: UE C++ / Specifiers  
Priority: P0

Q: Actor Component replicates automatically when owner Actor replicates?  
A: No; the owning Actor and the component must both be configured to replicate.  
Category: Networking / Components  
Priority: P1

Q: Static replicated component is created where?  
A: In the Actor constructor/default component tree on both server and clients.  
Category: Networking / Components  
Priority: P1

Q: Dynamic replicated component should be created where for convergence?  
A: On the authoritative server, then marked/configured to replicate.  
Category: Networking / Components  
Priority: P1

Q: Component subobject conditions are checked only after what?  
A: The owning Actor and owning component replicate to that connection.  
Category: Networking / Subobjects  
Priority: P2

Q: Replicated UObject subobject owns network context?  
A: No; it replicates through an owning Actor or Actor Component.  
Category: Networking / Subobjects  
Priority: P2

Q: Default subobject replication reference advantage?  
A: The object exists on server and clients with stable naming relative to the owner.  
Category: Networking / Subobjects  
Priority: P2

Q: Dynamic subobject pointer can be null on client because?  
A: The client reference is not valid until the subobject is replicated and mapped.  
Category: Networking / Subobject Mapping  
Priority: P2

Q: Registered subobjects list is required for what maintained-doc system?  
A: Iris subobject replication.  
Category: Networking / Iris  
Priority: P2

Q: Registered subobject deletion rule?  
A: Remove it from the registered list before deletion or GC can leave a stale raw pointer.  
Category: Networking / Subobjects  
Priority: P2

Q: Legacy subobject replication hook?  
A: Override `ReplicateSubobjects` and explicitly replicate subobjects through the Actor channel.  
Category: Networking / Subobjects  
Priority: P2

Q: Subobject RPCs work by default like Actor RPCs?  
A: No; actor-owned UObject RPCs need explicit callspace/remote-function routing or actor/component routing.  
Category: Networking / Subobjects  
Priority: P2

Q: Fast Array is for what?  
A: Item-level delta replication of authoritative arrays with stable item identity.  
Category: Networking / Fast Array  
Priority: P2

Q: Fast Array first design question?  
A: What is the stable item identity?  
Category: Networking / Fast Array  
Priority: P2

Q: Fast Array mutation path must do what?  
A: Centralise server mutations and mark changed items/array dirty according to target API.  
Category: Networking / Fast Array  
Priority: P2

Q: Fast Array is unnecessary when?  
A: The array is tiny, rarely changes, or full replacement is simpler and cheap.  
Category: Networking / Fast Array  
Priority: P2

Q: Push model changes what?  
A: How game code reports property dirtiness, not authority/relevancy/protocol design.  
Category: Networking / Push Model  
Priority: P2

Q: Push model common bug?  
A: A mutation path forgets to mark dirty, so networking can skip the change.  
Category: Networking / Push Model  
Priority: P2

Q: Replication Graph solves what bottleneck?  
A: Server CPU cost of building per-client replication lists for many Actors/connections.  
Category: Networking / Replication Graph  
Priority: P2

Q: Replication Graph node stores what kind of useful data?  
A: Persistent shared Actor lists/buckets used to build per-connection replication lists.  
Category: Networking / Replication Graph  
Priority: P2

Q: Bad reason to adopt Replication Graph?  
A: "The project is multiplayer" without profiling evidence of relevance/list-construction scale issues.  
Category: Networking / Replication Graph  
Priority: P2

Q: Iris is automatic in UE5?  
A: No; it is opt-in/project-config dependent and documented as experimental in the checked source.  
Category: Networking / Iris  
Priority: P2

Q: Iris subobject replication requires what path in maintained docs?  
A: Registered subobjects list.  
Category: Networking / Iris  
Priority: P2

Q: Custom UObject with Iris may need what specialist function?  
A: Target-branch replication-fragment registration.  
Category: Networking / Iris  
Priority: P3

Q: Network debug CVars are what?  
A: Targeted evidence tools, not production fixes.  
Category: Networking / Debugging  
Priority: P1

Q: NetGUID/PackageMap logs help diagnose what?  
A: Unmapped references, stale object mapping and async-loaded network references.  
Category: Networking / Debugging  
Priority: P2

Q: Dormancy validation helps catch what?  
A: State changes while an Actor is dormant or not correctly woken.  
Category: Networking / Debugging  
Priority: P1

Q: Fast Array versus simple array must be justified by what?  
A: Correctness and measured byte/CPU benefit for the target workload.  
Category: Networking / Fast Array  
Priority: P2

Q: Modular inventory replication starts with what owner?  
A: Server-owned durable item state with stable item IDs and explicit audience.  
Category: Networking / System Design  
Priority: P1

Q: Reliable RPC inventory log problem?  
A: It sends events instead of durable state and can fail late-join/recovery semantics.  
Category: Networking / System Design  
Priority: P1

Q: CharacterMovement prediction owner?  
A: The autonomous proxy predicts locally, but the server remains authoritative.  
Category: Networking / CharacterMovement  
Priority: P1

Q: CharacterMovement saved move records what?  
A: Input and movement-affecting state needed for server reproduction and client replay.  
Category: Networking / CharacterMovement  
Priority: P1

Q: CharacterMovement correction triggers what client step?  
A: Apply authoritative server state, then replay unacknowledged saved moves.  
Category: Networking / CharacterMovement  
Priority: P1

Q: Simulated proxies use prediction or smoothing?  
A: Usually smoothing/interpolation of received movement, not owner-style input prediction.  
Category: Networking / CharacterMovement  
Priority: P1

Q: Custom sprint saved-move state should go where?  
A: Into compact saved-move data/flags, not only a delayed replicated bool.  
Category: Networking / CharacterMovement  
Priority: P2

Q: Saved-move combine bug?  
A: Moves with different custom movement state combine and erase a transition.  
Category: Networking / CharacterMovement  
Priority: P2

Q: `CanCombineWith` protects what?  
A: It prevents move compression from merging incompatible movement states.  
Category: Networking / CharacterMovement  
Priority: P2

Q: `SetMoveFor` custom extension captures what?  
A: Movement-affecting state at the input sample that created the saved move.  
Category: Networking / CharacterMovement  
Priority: P2

Q: `PrepMoveFor` custom extension restores what?  
A: Saved custom state before client replay simulation.  
Category: Networking / CharacterMovement  
Priority: P2

Q: `UpdateFromCompressedFlags` custom extension does what?  
A: Reconstructs compact movement flags for simulation, including server reproduction paths.  
Category: Networking / CharacterMovement  
Priority: P2

Q: Movement-affecting state examples?  
A: Sprint flag, dash edge, custom movement mode, compact surface evidence.  
Category: Networking / CharacterMovement  
Priority: P2

Q: State that does not belong in saved moves?  
A: Cosmetics, UI-only state, final hit results and authoritative damage outcomes.  
Category: Networking / CharacterMovement  
Priority: P2

Q: Direct transform writes can cause what in CharacterMovement?  
A: Valid server corrections because movement bypassed the prediction protocol.  
Category: Networking / CharacterMovement  
Priority: P1

Q: Root motion plus CharacterMovement common bug?  
A: Both systems move the capsule, so prediction/server reproduction disagree.  
Category: Networking / Animation Networking  
Priority: P2

Q: Predicted dash key risk?  
A: The dash input edge is lost, doubled or rejected if not saved/validated carefully.  
Category: Networking / CharacterMovement  
Priority: P2

Q: Dash server validation checks?  
A: Cooldown, resources, tags/state, direction, movement mode, surface and collision legality.  
Category: Networking / CharacterMovement  
Priority: P2

Q: Lag compensation solves what?  
A: It lets the server evaluate an action against bounded historical state the client plausibly saw.  
Category: Networking / Lag Compensation  
Priority: P1

Q: Lag compensation must not trust what?  
A: Client-reported hit result, damage, or arbitrary timestamp.  
Category: Networking / Lag Compensation  
Priority: P1

Q: Hitscan rewind server stores what?  
A: Compact value snapshots of hit-relevant target transforms/bounds and state.  
Category: Networking / Lag Compensation  
Priority: P2

Q: Rewind history should be bounded why?  
A: To cap memory, CPU, fairness window and abuse surface.  
Category: Networking / Lag Compensation  
Priority: P1

Q: Rewind timestamp policy?  
A: Convert client timing to bounded server query time and reject impossible values.  
Category: Networking / Lag Compensation  
Priority: P2

Q: Lag compensation fairness tradeoff?  
A: It helps the shooter but can feel unfair to targets if the window is too generous.  
Category: Networking / Lag Compensation  
Priority: P1

Q: Server rewind current-blocker question?  
A: Define whether past target state, current blockers, or timestamped blockers determine hits.  
Category: Networking / Lag Compensation  
Priority: P2

Q: Rewind respawn bug?  
A: A snapshot from an old life is hit after death/respawn without generation/state checks.  
Category: Networking / Lag Compensation  
Priority: P2

Q: Prediction versus reconciliation?  
A: Prediction simulates locally; reconciliation applies authoritative correction and replays inputs.  
Category: Networking / Concepts  
Priority: P1

Q: Rewind versus rollback?  
A: Rewind queries historical state; rollback restores old simulation state and resimulates forward.  
Category: Networking / Concepts  
Priority: P1

Q: Interpolation versus prediction?  
A: Interpolation smooths remote received state; prediction simulates local owner input immediately.  
Category: Networking / Concepts  
Priority: P1

Q: Full rollback requires what?  
A: Controlled/deterministic state, input logs, snapshots, resimulation and side-effect handling.  
Category: Networking / Concepts  
Priority: P2

Q: Movement prediction profiling first metric?  
A: Correction count and distance per feature under representative lag/loss.  
Category: Networking / Profiling  
Priority: P2

Q: Saved-move profiling includes what?  
A: Allocations, combine rate, replay count and custom payload size.  
Category: Networking / Profiling  
Priority: P2

Q: Rewind profiling includes what?  
A: Snapshot memory, allocation rate, query/trace cost and hit confirmation/rejection rate.  
Category: Networking / Profiling  
Priority: P2

Q: Strong branch-sensitive answer admits what?  
A: Exact CharacterMovement saved-move signatures must be verified in target headers/source.  
Category: Networking / Interview Strategy  
Priority: P2

Q: Shader permutation means what?  
A: One compiled shader variant for a platform/pass/feature/material/static-switch combination.  
Category: Rendering / Shaders  
Priority: P1

Q: Runtime shader cost means what?  
A: GPU work such as instructions, texture reads, bandwidth, occupancy and execution frequency.  
Category: Rendering / Shaders  
Priority: P1

Q: PSO hitch means what?  
A: Expensive first-time creation/preparation of an exact pipeline state during gameplay.  
Category: Rendering / PSO Caches  
Priority: P1

Q: Shader library coverage is not what?  
A: Proof that every exact PSO combination is cached/prepared.  
Category: Rendering / PSO Caches  
Priority: P1

Q: Static switches trade what?  
A: Lower runtime branches for more compiled permutations and PSO diversity.  
Category: Rendering / Materials  
Priority: P1

Q: PSO collection must include what rare paths?  
A: VFX, UI, cosmetics, cutscenes, scene captures and scalability-specific views.  
Category: Rendering / PSO Caches  
Priority: P1

Q: PSO hitch must be verified where?  
A: In packaged target builds with cold-cache or representative first-use conditions.  
Category: Rendering / PSO Caches  
Priority: P1

Q: Plugin shader first setup risk?  
A: Shader directory mapping or module loading happens too late/wrong for shader registration.  
Category: Rendering / Shader Programming  
Priority: P2

Q: Global shader permutation predicate does what?  
A: Limits compilation to supported/needed platforms and feature levels.  
Category: Rendering / Shader Programming  
Priority: P2

Q: RDG shader pass parameter structs declare what?  
A: Constants and GPU resource reads/writes so RDG can validate dependencies/lifetimes.  
Category: Rendering / RDG  
Priority: P2

Q: RDG pass lambda lifetime trap?  
A: It executes later, so captured stack data or invalid UObject access can be unsafe.  
Category: Rendering / RDG  
Priority: P2

Q: Mesh drawing pipeline starts from what render-side representation?  
A: Primitive scene proxy/data rather than arbitrary live gameplay UObject access.  
Category: Rendering / Mesh Drawing  
Priority: P2

Q: One mesh can produce many draws why?  
A: Material sections, passes, shadows, velocity/custom depth and state variants split work.  
Category: Rendering / Mesh Drawing  
Priority: P1

Q: Cached mesh draw commands dislike what?  
A: Frequent dynamic render-state changes, component churn and material override thrash.  
Category: Rendering / Mesh Drawing  
Priority: P2

Q: Draw/RHI regression first rendering checks?  
A: Primitive counts, material diversity, render-state invalidation and component representation.  
Category: Rendering / Mesh Drawing  
Priority: P1

Q: Render target cost dimensions?  
A: Resolution, format, update cadence, mips, clears/copies/readbacks, lifetime and consumers.  
Category: Rendering / Render Targets  
Priority: P1

Q: SceneCapture2D should be assumed to be what?  
A: An additional camera/view until target captures prove a cheaper path.  
Category: Rendering / Scene Capture  
Priority: P1

Q: SceneCaptureCube worst-case multiplier?  
A: Six directional views before considering mips, format or consumers.  
Category: Rendering / Scene Capture  
Priority: P2

Q: `CaptureEveryFrame` risk?  
A: It turns a preview/minimap/security feed into continuous extra render work.  
Category: Rendering / Scene Capture  
Priority: P1

Q: Scene capture PSO pitfall?  
A: Capture-only material/pass combinations may be absent from main-camera PSO coverage.  
Category: Rendering / PSO Caches  
Priority: P2

Q: Render target memory quick formula?  
A: Width * height * bytes per pixel * slices/faces * mip/buffering factors.  
Category: Rendering / Render Targets  
Priority: P2

Q: GPU capture escalation starts after what?  
A: Unreal pass timings identify the pass but not exact pipeline/resource/shader cause.  
Category: Rendering / GPU Capture  
Priority: P2

Q: RenderDoc capture metadata?  
A: Engine build, RHI/API, GPU/driver, resolution, scalability, camera path and event markers.  
Category: Rendering / GPU Capture  
Priority: P2

Q: External GPU capture must end with what?  
A: An Unreal-level fix verified by normal before/after profiling.  
Category: Rendering / GPU Capture  
Priority: P2

Q: Strong PSO answer separates what?  
A: Shader availability, exact PSO coverage and runtime hot-path cost.  
Category: Rendering / Interview Strategy  
Priority: P1

Q: AutomationTool is what?  
A: Unreal's command-line automation runner used for operations such as build, cook, stage, package, deploy and run.  
Category: Build / AutomationTool  
Priority: P2

Q: BuildGraph is what?  
A: A declarative graph for repeatable build-farm pipelines with parameters, nodes, dependencies, agents, labels and artifacts.  
Category: Build / BuildGraph  
Priority: P2

Q: BuildCookRun belongs to which layer?  
A: AutomationTool/RunUAT, often invoked directly or from a larger BuildGraph pipeline.  
Category: Build / AutomationTool  
Priority: P2

Q: A BuildGraph node should express what?  
A: One meaningful operation with explicit inputs, dependencies and outputs rather than hidden shell-script ordering.  
Category: Build / BuildGraph  
Priority: P2

Q: Release pipeline proof starts with what environment?  
A: A clean workspace or build agent with no reliance on local Binaries, Intermediate, caches or global installs.  
Category: Build / Release Automation  
Priority: P1

Q: Release artifact set should include what?  
A: Packaged output, logs, manifests, metadata, symbols and enough IDs to tie crashes back to the exact build.  
Category: Build / Release Artifacts  
Priority: P1

Q: Packaged-only crash first evidence?  
A: Packaged log, callstack, build ID, target/config/platform, command line and staged manifest.  
Category: Build / Packaged Debugging  
Priority: P1

Q: Editor success does not prove what?  
A: Cooked content inclusion, runtime module boundaries, staged dependencies or Shipping/platform behaviour.  
Category: Build / Packaged Debugging  
Priority: P1

Q: Third-party runtime dependency proof requires what?  
A: Clean packaged launch with the exact platform/config binary staged by Build.cs/pipeline rules, not a global install.  
Category: Build / Third-Party  
Priority: P2

Q: Header visibility proves what for third-party code?  
A: Only compilation visibility, not link library presence or runtime binary staging.  
Category: Build / Third-Party  
Priority: P1

Q: Dynamic library packaging trap?  
A: The editor finds a local/global DLL, but the packaged clean machine cannot because staging rules are missing.  
Category: Build / Third-Party  
Priority: P1

Q: Symbol archive must match what?  
A: The exact packaged executable/binary build identified by changelist/build ID, target, configuration and platform.  
Category: Build / Crash Reporting  
Priority: P1

Q: Forced packaged crash smoke test proves what?  
A: Crash capture, metadata, log tail, symbolication and routing work before the build reaches players.  
Category: Build / Crash Reporting  
Priority: P2

Q: Unsymbolicated crash is also what kind of bug?  
A: A release pipeline/artifact bug, because the team cannot diagnose the shipped build reliably.  
Category: Build / Crash Reporting  
Priority: P1

Q: Pre-submit CI tier optimises for what?  
A: Fast feedback on likely correctness failures: compile, focused tests, validation and narrow cooks.  
Category: Build / CI  
Priority: P2

Q: Nightly clean CI tier optimises for what?  
A: Detecting hidden local-cache, unity/include, cook/package and clean-machine assumptions.  
Category: Build / CI  
Priority: P2

Q: Release-candidate CI tier optimises for what?  
A: Producing distributable artifacts with symbols, manifests, smoke tests and crash proof.  
Category: Build / CI  
Priority: P2

Q: CI matrix should be justified by what?  
A: Failure cost and feedback time, with expensive gates moved to appropriate tiers instead of bypassed.  
Category: Build / CI  
Priority: P2

Q: Build stage versus cook stage?  
A: Build produces binaries; cook converts/selects target-platform content and dependencies.  
Category: Build / Packaging  
Priority: P1

Q: Stage versus package?  
A: Stage lays out the runtime files; package/archive creates the distributable/container/artifact set.  
Category: Build / Packaging  
Priority: P1

Q: Broad "cook everything" workaround hides what?  
A: Missing ownership rules, Primary Asset management mistakes and release size/load regressions.  
Category: Build / Cooking  
Priority: P1

Q: Strong release automation answer includes what final proof?  
A: Clean-machine launch plus symbolicated forced-crash proof from the archived artifact set.  
Category: Build / Interview Strategy  
Priority: P1

Q: Release manifest proves what beyond file existence?  
A: That another clean machine can identify, launch, symbolicate and diagnose the same exact packaged build.  
Category: Build / Release Artifacts  
Priority: P1

Q: Minimum reproducible package identity includes what?  
A: Commit or changelist, platform/config, package/build ID and the commands or graph parameters that produced it.  
Category: Build / Release Artifacts  
Priority: P1

Q: Why archive symbol location in the release manifest?  
A: Crash reports are weak evidence unless they can be matched to the exact shipped binary and symbol set.  
Category: Build / Crash Proof  
Priority: P1

Q: Pre-submit lane should catch what class of issues?  
A: Fast source, UHT, validation and smoke failures close to the author's change.  
Category: Build / CI  
Priority: P1

Q: Nightly build lane is good for what?  
A: Clean-agent, non-unity, cook/package and target-matrix problems that are too expensive to run on every change.  
Category: Build / CI  
Priority: P1

Q: Release-candidate lane must prove what?  
A: The distributable package, symbols, runtime dependencies, smoke path and crash proof all work together.  
Category: Build / CI  
Priority: P1

Q: Why is “run everything on every change” a weak CI answer?  
A: Excessive cost makes engineers bypass CI while still failing to model ownership and feedback speed.  
Category: Build / CI  
Priority: P2

Q: Correct cache-busting order starts with what?  
A: The failing phase and first causal log line, not blanket deletion of every output/cache.  
Category: Build / Debugging  
Priority: P1

Q: What should be rerun before a full clean rebuild when possible?  
A: The narrowest failing phase such as build, cook, stage or package.  
Category: Build / Debugging  
Priority: P1

Q: Smallest-plausible reset means what?  
A: Invalidate only the cache/output set the evidence actually implicates before escalating further.  
Category: Build / Debugging  
Priority: P2

Q: “Delete Intermediate and try again” is what kind of answer?  
A: A temporary symptom reset, not a root-cause debugging method.  
Category: Build / Interview Strategy  
Priority: P1

Q: Actionable build-farm report starts with what?  
A: Failed phase and first causal error rather than the final wrapper failure message.  
Category: Build / Automation  
Priority: P1

Q: Build-farm report should name what artifact facts?  
A: Branch/commit, package/build ID, platform/config and evidence paths for logs/manifests/symbols.  
Category: Build / Automation  
Priority: P1

Q: Why include clean-rerun status in a failure report?  
A: It distinguishes deterministic product/setup issues from flaky infrastructure or transient lab noise.  
Category: Build / Automation  
Priority: P2

Q: Best ending for a build-farm failure report?  
A: The smallest recurrence guard such as a Build.cs fix, staging rule, cook rule, graph dependency or smoke gate.  
Category: Build / Automation  
Priority: P1

Q: BuildGraph artifact flow should be what?  
A: Explicit in graph dependencies rather than hidden in an engineer's memory or local machine state.  
Category: Build / BuildGraph  
Priority: P2

Q: CrashProof graph node must depend on what?  
A: Packaged binaries plus the matching archived symbols and metadata.  
Category: Build / BuildGraph  
Priority: P2

Q: Missing feature in packaged build first asks what content question?  
A: Was it cooked/staged and discoverable, or only present as loose editor data?  
Category: Build / Packaged Debugging  
Priority: P1

Q: Reproducible package without a manifest is really what?  
A: A pile of files that someone hopes are the right release artifact.  
Category: Build / Release Artifacts  
Priority: P1

Q: Strong release answer ends with what proof?  
A: Another clean environment can launch the archived package and symbolicate a forced crash against it.  
Category: Build / Interview Strategy  
Priority: P1

Q: Performance capture contract starts with what?  
A: Build identity, target environment, quality/profile settings, scenario, capture tools, metrics and decision rule.  
Category: Profiling / Benchmark Design  
Priority: P1

Q: Editor viewport benchmark misses what?  
A: Packaged target config, cooked content, device profile, target RHI/driver, cold caches, thermals and platform memory limits.  
Category: Profiling / Benchmark Design  
Priority: P1

Q: CSV Profiler is best for what?  
A: Lightweight long-run or automated regression metrics rather than deep causal frame analysis.  
Category: Profiling / CSV  
Priority: P2

Q: Unreal Insights is best for what?  
A: Causal analysis of frames, scopes, tasks, waits, allocations, loads, network events and aligned timelines.  
Category: Profiling / Insights  
Priority: P1

Q: CSV gate failure should trigger what?  
A: A targeted Insights, GPU, Memory Insights or platform-profiler capture for the same scenario.  
Category: Profiling / CI  
Priority: P2

Q: Memory Insights answers what memory question?  
A: Why allocations exist, grow, churn, live long or leak-like, often with callstack and timeline evidence.  
Category: Profiling / Memory Insights  
Priority: P1

Q: LLM answers what memory question?  
A: Which subsystem/tag owns memory budget at stable checkpoints.  
Category: Profiling / LLM  
Priority: P2

Q: Capacity memory issue means what?  
A: Stable resident/tag total exceeds the target budget, even if no leak exists.  
Category: Profiling / Memory  
Priority: P1

Q: Memory churn issue means what?  
A: Short-lived allocation/reallocation/free rate creates CPU, fragmentation or hitch cost.  
Category: Profiling / Memory  
Priority: P1

Q: Growth memory issue means what?  
A: Repeated cycles do not return near baseline, suggesting retained state, cache policy or leak-like behaviour.  
Category: Profiling / Memory  
Priority: P1

Q: Device profile source file proves what?  
A: Nothing by itself; runtime applied profile/CVar evidence is required.  
Category: Profiling / Device Profiles  
Priority: P1

Q: Low scalability tier must still preserve what?  
A: Gameplay readability, UI affordances, critical feedback and simulation/authority correctness.  
Category: Profiling / Scalability  
Priority: P1

Q: Packaged hitch first classification domains?  
A: Load/I/O, shader/PSO, UObject/GC, task wait/lock, memory pressure, GPU pass or device-profile setting.  
Category: Profiling / Hitch Diagnosis  
Priority: P1

Q: Why use warm-up in benchmarks?  
A: To separate first-use cache/setup costs from the steady measurement window, unless first-use is the target metric.  
Category: Profiling / Benchmark Design  
Priority: P1

Q: Performance CI threshold should include what?  
A: Scenario, metric, window, tolerance, artifact, owner and escalation capture.  
Category: Profiling / CI  
Priority: P2

Q: One-run 1 percent performance delta is usually what?  
A: Noise unless repeated under a controlled scenario with clear variance bounds.  
Category: Profiling / Benchmark Design  
Priority: P2

Q: P95/P99 frame time protects against what?  
A: Average-FPS claims that hide frequent or severe hitches.  
Category: Profiling / Frame Budget  
Priority: P1

Q: Trace overhead means what for final proof?  
A: Heavy instrumentation can perturb workload, so final verification should use lighter/representative capture when possible.  
Category: Profiling / Insights  
Priority: P1

Q: Applied CVar dump helps prove what?  
A: The intended scalability/device profile values actually took effect in the target run.  
Category: Profiling / Device Profiles  
Priority: P1

Q: Strong optimisation proof includes what beyond faster frame time?  
A: Correctness, quality, memory, target tiers/devices and a regression signal.  
Category: Profiling / Interview Strategy  
Priority: P1

Q: Enhanced Input primarily separates what?  
A: Physical device input from semantic player intent and gameplay authority.  
Category: Enhanced Input / Architecture  
Priority: P1

Q: Input Action represents what?  
A: A semantic action/value the user can perform, such as Move, Fire, Confirm or AbilityPrimary.  
Category: Enhanced Input / Input Actions  
Priority: P1

Q: Input Mapping Context represents what?  
A: A mode/layer of physical bindings from keys/buttons/axes to Input Actions for a local player.  
Category: Enhanced Input / Mapping Contexts  
Priority: P1

Q: Mapping contexts are applied to what owner?  
A: The Enhanced Input Local Player Subsystem for the relevant local player.  
Category: Enhanced Input / Local Player  
Priority: P1

Q: Why avoid `IA_LeftMouseButton`?  
A: It names the device instead of semantic intent, making remapping and gamepad/touch support worse.  
Category: Enhanced Input / Input Actions  
Priority: P1

Q: Boolean Input Action is best for what?  
A: On/off intents such as Jump, Confirm, Reload or Interact.  
Category: Enhanced Input / Input Actions  
Priority: P1

Q: Axis2D Input Action is best for what?  
A: Two-dimensional values such as movement or look input.  
Category: Enhanced Input / Input Actions  
Priority: P1

Q: Input Modifier does what?  
A: Transforms raw input values before trigger evaluation.  
Category: Enhanced Input / Modifiers  
Priority: P1

Q: Input Trigger does what?  
A: Decides whether modified input activates an action and which trigger state/event is emitted.  
Category: Enhanced Input / Triggers  
Priority: P1

Q: Common modifier examples?  
A: Dead zone, negate, axis swizzle, sensitivity, smoothing, inversion and space conversion.  
Category: Enhanced Input / Modifiers  
Priority: P1

Q: Triggered event danger?  
A: It can fire repeatedly for continuous or repeated-trigger actions, causing one-shot spam if misused.  
Category: Enhanced Input / Triggers  
Priority: P1

Q: Started event often means what?  
A: The beginning edge of action evaluation, useful for one-shot press semantics.  
Category: Enhanced Input / Triggers  
Priority: P1

Q: Completed/Canceled are important for what?  
A: Release, charge, cancellation and cleanup semantics.  
Category: Enhanced Input / Triggers  
Priority: P1

Q: Context priority solves what?  
A: Which mapping wins when active contexts could use the same physical input.  
Category: Enhanced Input / Mapping Contexts  
Priority: P1

Q: One giant mapping context creates what risk?  
A: Mode collisions, inappropriate actions firing and unclear priority/remapping policy.  
Category: Enhanced Input / Architecture  
Priority: P1

Q: Input coordinator should log what?  
A: Local player, contexts, priorities, requester, action, trigger event and route.  
Category: Enhanced Input / Debugging  
Priority: P2

Q: Action does not fire first Enhanced Input checks?  
A: Correct local player, active context, priority, action asset, value type, modifiers, triggers and focus.  
Category: Enhanced Input / Debugging  
Priority: P1

Q: Input fires twice after respawn likely why?  
A: Bindings or contexts were added again without removing or rebuilding the old route.  
Category: Enhanced Input / Debugging  
Priority: P1

Q: Remapping needs what policy?  
A: Per-user/local-player persistence, conflict rules, device/platform awareness, glyph updates and safe defaults.  
Category: Enhanced Input / Remapping  
Priority: P2

Q: Context source file proves runtime activation?  
A: No; log or inspect the active LocalPlayer contexts and priorities.  
Category: Enhanced Input / Debugging  
Priority: P1

Q: Enhanced Input plus CommonUI risk?  
A: UI focus/input mode and gameplay mapping contexts can fight unless one coordinator owns transitions.  
Category: Enhanced Input / UI  
Priority: P1

Q: Gameplay action leaking through menu means what likely?  
A: Gameplay context remains active or higher/equal priority while UI confirm/cancel is active.  
Category: Enhanced Input / UI  
Priority: P1

Q: Enhanced Input with GAS should map to what?  
A: Ability specs, handles, tags or activation requests through an adapter, not physical key names.  
Category: Enhanced Input / GAS  
Priority: P2

Q: ASC on PlayerState input hazard?  
A: Respawn/avatar changes can duplicate or stale input bindings unless rebuilt idempotently.  
Category: Enhanced Input / GAS  
Priority: P2

Q: Network-safe input sends what?  
A: Validatable semantic intent, not authoritative client state or every raw device tick.  
Category: Enhanced Input / Networking  
Priority: P1

Q: Movement-affecting input should use what path?  
A: CharacterMovement prediction/saved moves when it changes movement simulation.  
Category: Enhanced Input / Networking  
Priority: P1

Q: Custom modifier should avoid what?  
A: Heavy gameplay searches, authoritative mutations and hidden side effects.  
Category: Enhanced Input / Modifiers  
Priority: P2

Q: Split-screen input requires what assumption change?  
A: Input settings and mapping contexts are per local player, not global.  
Category: Enhanced Input / Local Player  
Priority: P1

Q: Strong Enhanced Input answer mentions what four core concepts?  
A: Input Actions, Mapping Contexts, Modifiers and Triggers.  
Category: Enhanced Input / Interview Strategy  
Priority: P1

Q: Strong input architecture keeps server authority where?  
A: In validated gameplay systems, RPC/ability paths and replicated state, not local input handlers.  
Category: Enhanced Input / Interview Strategy  
Priority: P1

Q: Smart Objects primarily model what?  
A: World affordances that agents can query, claim, use and release.  
Category: Smart Objects / Architecture  
Priority: P2

Q: Smart Object "found" differs from "claimed" how?  
A: Found is only a candidate; claimed is a reserved slot ownership state.  
Category: Smart Objects / Lifecycle  
Priority: P2

Q: Smart Object slot represents what?  
A: A specific usable opportunity such as a seat, cover edge, terminal point or queue position.  
Category: Smart Objects / Slots  
Priority: P2

Q: Smart Object definition versus component?  
A: The definition describes authored slots/behavior; the component gives an instance world presence.  
Category: Smart Objects / Data Model  
Priority: P2

Q: Minimum Smart Object lifecycle?  
A: Query, claim, reach/validate, use and release.  
Category: Smart Objects / Lifecycle  
Priority: P2

Q: Smart Object release must happen on what paths?  
A: Success, failure, abort, movement fail, user destruction, object disable/destruction and authority rejection.  
Category: Smart Objects / Lifecycle  
Priority: P1

Q: Claimed slot still needs what before use?  
A: Navigation, reachability, alignment and object validity checks.  
Category: Smart Objects / Navigation  
Priority: P2

Q: Smart Object should not own what?  
A: All gameplay truth, animation correctness, ability rules or network authority.  
Category: Smart Objects / Architecture  
Priority: P2

Q: Two NPCs same slot first debug question?  
A: Did both merely find the candidate, or did both successfully claim it?  
Category: Smart Objects / Debugging  
Priority: P2

Q: Smart Object multiplayer policy?  
A: Client may propose/preview; server validates and owns durable claim/use outcomes.  
Category: Smart Objects / Networking  
Priority: P2

Q: StateTree combines what two models?  
A: Behavior Tree-style selectors with hierarchical state-machine states and transitions.  
Category: StateTree / Architecture  
Priority: P2

Q: StateTree selected path activates what?  
A: The selected leaf state and all parent states from root to leaf.  
Category: StateTree / Execution  
Priority: P2

Q: StateTree tasks run for which states?  
A: All active states, starting from root down to the selected state.  
Category: StateTree / Execution  
Priority: P2

Q: StateTree transition evaluation order?  
A: From the leaf state upward toward the root.  
Category: StateTree / Transitions  
Priority: P2

Q: StateTree task concurrency hazard?  
A: A helper task can complete first and trigger a transition before the main action is done.  
Category: StateTree / Debugging  
Priority: P2

Q: StateTree parameters are for what?  
A: External configuration of a tree instance.  
Category: StateTree / Data Flow  
Priority: P2

Q: StateTree context data is what?  
A: Execution-environment data such as actor/user/object identity.  
Category: StateTree / Data Flow  
Priority: P2

Q: StateTree evaluators expose what?  
A: Runtime-derived data to conditions and tasks.  
Category: StateTree / Data Flow  
Priority: P2

Q: StateTree global tasks are useful for what?  
A: Tree-lifetime work or data needed before root state selection.  
Category: StateTree / Data Flow  
Priority: P2

Q: Smart Object StateTree common sequence?  
A: Acquire claim, reach slot, use behavior, release and exit.  
Category: Smart Objects / StateTree  
Priority: P2

Q: Smart Object query performance starts with what counts?  
A: Agents, query frequency, radius, candidates, path validations and active StateTree tasks.  
Category: Smart Objects / Performance  
Priority: P2

Q: Failed Smart Object query should avoid what?  
A: Immediate every-frame retries by many agents.  
Category: Smart Objects / Performance  
Priority: P2

Q: Activity tags in Smart Objects serve what purpose?  
A: They express semantic use such as Rest, Cover or Repair for query/filter contracts.  
Category: Smart Objects / Data Design  
Priority: P2

Q: Strong Smart Object answer emphasizes what boundary?  
A: Affordance coordination is separate from movement, animation, abilities and authority.  
Category: Smart Objects / Interview Strategy  
Priority: P2

Q: When is simple code better than Smart Object plus StateTree?  
A: For tiny local modes where asset/tool overhead outweighs reuse, authoring and lifecycle benefits.  
Category: Smart Objects / Architecture  
Priority: P2

Q: PCG in Unreal primarily models what workflow?  
A: Graph-based procedural content generation from spatial input, points, attributes and output nodes.  
Category: PCG / Architecture  
Priority: P2

Q: PCG Component does what?  
A: Bridges level/actor context to a PCG Graph and manages generation.  
Category: PCG / Component  
Priority: P2

Q: PCG Graph is what?  
A: The reusable rule set that transforms input spatial data and points into generated output.  
Category: PCG / Graph  
Priority: P2

Q: PCG point represents what?  
A: A candidate spatial data record with transform, bounds, density, seed and attributes.  
Category: PCG / Points  
Priority: P2

Q: PCG density is not what?  
A: It is not automatically the final mesh count.  
Category: PCG / Density  
Priority: P2

Q: PCG density debug view represents what?  
A: Probability/weight-like point existence value from 0 to 1.  
Category: PCG / Density  
Priority: P2

Q: Static PCG attributes often start with what?  
A: A `$` prefix.  
Category: PCG / Attributes  
Priority: P2

Q: PCG metadata domain bug means what?  
A: The attribute exists, but not on the domain the downstream node reads.  
Category: PCG / Debugging  
Priority: P2

Q: `@Points` domain carries what?  
A: Per-point metadata.  
Category: PCG / Metadata  
Priority: P2

Q: `@Data` domain is useful for what?  
A: Data-level single-value attributes.  
Category: PCG / Metadata  
Priority: P2

Q: Graph parameters should expose what kind of controls?  
A: Named, bounded, designer-safe controls such as seed, density, asset set and debug mode.  
Category: PCG / Authoring  
Priority: P2

Q: PCG graph instances are useful for what?  
A: Reusing graph logic with different parameter overrides.  
Category: PCG / Authoring  
Priority: P2

Q: First PCG debugging step?  
A: Freeze seed, graph version, component parameters and input sources.  
Category: PCG / Debugging  
Priority: P2

Q: PCG wrong-output debug inspects what per phase?  
A: Point count, density, transforms, attributes and output count.  
Category: PCG / Debugging  
Priority: P2

Q: PCG node disable is useful for what?  
A: Isolating the graph phase where output first contradicts expectations.  
Category: PCG / Debugging  
Priority: P2

Q: PCG debug nodes caveat?  
A: Some debug output is editor-only or non-shipping behavior.  
Category: PCG / Debugging  
Priority: P2

Q: Runtime PCG needs what before adoption?  
A: Product requirement plus target proof for hitches, memory, streaming, determinism and authority.  
Category: PCG / Runtime  
Priority: P2

Q: Editor-only PCG can be what?  
A: An authoring/content-generation tool whose output ships as normal content.  
Category: PCG / Workflow  
Priority: P2

Q: PCG performance separates what two costs?  
A: Graph execution cost and generated output cost.  
Category: PCG / Performance  
Priority: P2

Q: PCG output cost can appear where?  
A: Actors/components, instances, rendering, collision, nav, streaming, memory and save/load.  
Category: PCG / Performance  
Priority: P2

Q: PCG generated gameplay objects need what?  
A: Authority, validation, stable identity or reconstruction, save/load and cleanup policy.  
Category: PCG / Gameplay Integration  
Priority: P2

Q: PCG should not hide what?  
A: Durable economy, combat, quest or replication rules inside graph logic.  
Category: PCG / Gameplay Integration  
Priority: P2

Q: `Apply On Actor` style PCG mutation risk?  
A: External actor changes may not be revertible by PCG and need explicit policy.  
Category: PCG / Safety  
Priority: P2

Q: Strong PCG answer includes what reproducibility fields?  
A: Seed, inputs, graph version, parameters and output evidence.  
Category: PCG / Interview Strategy  
Priority: P2

Q: PCG plus World Partition requires checking what?  
A: Data Layer, HLOD, streaming, generated ownership and target runtime behavior.  
Category: PCG / World Building  
Priority: P2

Q: Platform readiness starts from what?  
A: Target device/SKU, OS/toolchain, budgets, package path and evidence requirements.  
Category: Platform / Release  
Priority: P2

Q: Editor success proves what for platform readiness?  
A: Very little; packaged target-device proof is required.  
Category: Platform / Debugging  
Priority: P2

Q: Mobile performance differs from desktop because of what?  
A: Device variance, thermal throttling, battery, memory limits, lifecycle and input/store constraints.  
Category: Platform / Mobile  
Priority: P2

Q: Device Profiles choose what?  
A: Platform/device-specific defaults and CVars.  
Category: Platform / Device Profiles  
Priority: P2

Q: Scalability groups control what?  
A: Quality tiers for resolution, shadows, textures, effects, view distance, foliage and similar settings.  
Category: Platform / Scalability  
Priority: P2

Q: Low scalability must not remove what?  
A: Gameplay-significant objects or readable critical information.  
Category: Platform / Scalability  
Priority: P2

Q: Platform performance capture should use what build?  
A: A packaged build on representative target hardware.  
Category: Platform / Profiling  
Priority: P2

Q: Thermal-length capture checks what?  
A: Whether performance holds after the device heats up and throttles.  
Category: Platform / Mobile  
Priority: P2

Q: First device-only crash fact to record?  
A: Exact package/artifact, commit, config, device model, OS and SDK/toolchain.  
Category: Platform / Debugging  
Priority: P2

Q: Device-only crash workflow classifies what?  
A: Install, launch, native/plugin load, asset load, shader/PSO, map load or gameplay startup phase.  
Category: Platform / Debugging  
Priority: P2

Q: Release artifact contract includes what?  
A: Package, symbols, manifests, logs, metadata, command parameters and validation results.  
Category: Platform / Build Release  
Priority: P2

Q: Crash symbolication requires what?  
A: Symbols matching the exact archived build.  
Category: Platform / Crash Debugging  
Priority: P2

Q: Certification readiness means what?  
A: Evidence that platform-required state transitions and failures are handled according to platform-holder docs.  
Category: Platform / Certification  
Priority: P2

Q: Console TRC/TCR details should be handled how?  
A: As confidential platform-holder requirements, not public guesses.  
Category: Platform / Certification  
Priority: P2

Q: Cert-adjacent input failure example?  
A: Controller disconnect/reconnect or input device change mid-flow.  
Category: Platform / Lifecycle  
Priority: P2

Q: Cert-adjacent storage failure example?  
A: Save write failure, storage full, corrupted save or cloud conflict.  
Category: Platform / Lifecycle  
Priority: P2

Q: Suspend/resume tests catch what?  
A: Stale delegates, timers, audio, UI focus, network and async-loading state.  
Category: Platform / Lifecycle  
Priority: P2

Q: Platform UI checks include what?  
A: Safe area, readability, localisation/text overflow, input prompts and accessibility-relevant cues.  
Category: Platform / UI  
Priority: P2

Q: Mobile setup can fail before runtime due to what?  
A: SDK/NDK/Xcode/provisioning/signing/package requirements.  
Category: Platform / Toolchain  
Priority: P2

Q: Device Profiles and Scalability need what proof?  
A: Applied CVar/profile dumps or equivalent evidence from packaged target runs.  
Category: Platform / Evidence  
Priority: P2

Q: Platform performance gate tracks what beyond FPS?  
A: Percentiles, hitches, memory, texture pool, I/O, startup, thermal/power and variance.  
Category: Platform / Profiling  
Priority: P2

Q: Storage/network/account failures should be modeled as what?  
A: Explicit state transitions with expected behavior, evidence and owner.  
Category: Platform / Certification  
Priority: P2

Q: Clean-machine package proof prevents what?  
A: Reliance on editor-adjacent files, local installs, cached assets or unstaged binaries.  
Category: Platform / Build Release  
Priority: P2

Q: Strong platform answer avoids what?  
A: Inventing confidential certification numbers or treating editor/device behavior as equivalent.  
Category: Platform / Interview Strategy  
Priority: P2

Q: Platform readiness packet final decision options?  
A: Ship, hold, reduce scope, exclude tier or require authorised platform checklist proof.  
Category: Platform / Release  
Priority: P2

Q: Device-lab automation is mainly what?  
A: A repeatable target-evidence pipeline for packaged builds, devices, scenarios, telemetry, artifacts and triage.  
Category: Device Lab / Automation  
Priority: P2

Q: Device lab is not just what?  
A: It is not just owning devices or manually clicking through a build.  
Category: Device Lab / Automation  
Priority: P2

Q: Smallest useful device matrix includes what lanes?  
A: Fast smoke, low supported device/tier, target/common device/tier and optional reporting-only soak or high-tier lane.  
Category: Device Lab / Matrix  
Priority: P2

Q: Device-lab run manifest binds what?  
A: Build ID, device ID, scenario ID, command line, install mode and output artifacts.  
Category: Device Lab / Artifacts  
Priority: P2

Q: A packaged run manifest should include symbols why?  
A: To prove crash callstacks match the exact archived build.  
Category: Device Lab / Crash Proof  
Priority: P2

Q: Scenario contract needs what timing fields?  
A: Warm-up window, sample window and explicit readiness/exit markers.  
Category: Device Lab / Scenario Design  
Priority: P2

Q: Why add readiness markers to target scenarios?  
A: They distinguish launch/map-load failures from test-body failures and make time-to-ready measurable.  
Category: Device Lab / Scenario Design  
Priority: P2

Q: Why use CSV frequently in a device lab?  
A: It is light enough for repeated packaged runs and trend/gate aggregation.  
Category: Device Lab / Profiling  
Priority: P2

Q: When use Unreal Insights in a lab?  
A: For short causal captures around a failed or sampled window, not always-on exhaustive tracing.  
Category: Device Lab / Profiling  
Priority: P2

Q: Heavy traces can cause what lab problem?  
A: They can perturb timing, slow runs and produce too much data for routine gates.  
Category: Device Lab / Profiling  
Priority: P2

Q: Active Device Profile proof means what?  
A: A target-run dump or artifact showing which profile, scalability tier and CVars actually applied.  
Category: Device Lab / Profile Proof  
Priority: P2

Q: One good performance run proves what?  
A: Almost nothing by itself; variance, thermals, install state and scenario coverage still matter.  
Category: Device Lab / Profiling  
Priority: P2

Q: Metric should become blocking only after what?  
A: Variance is understood, tolerance is defined and failures are actionable with an owner.  
Category: Device Lab / Gates  
Priority: P2

Q: Reporting lane is useful before what?  
A: Before promoting a new metric to a release-blocking gate.  
Category: Device Lab / Gates  
Priority: P2

Q: Product failure in lab means what?  
A: A reproducible build/scenario issue such as a stable crash, assert, missing asset or performance regression.  
Category: Device Lab / Triage  
Priority: P2

Q: Lab infrastructure failure means what?  
A: Runner, install, artifact pull, permission, network, service or device-management failure outside product logic.  
Category: Device Lab / Triage  
Priority: P2

Q: Flaky device policy should avoid what?  
A: Both ignoring device health and rerunning deterministic product crashes until they disappear.  
Category: Device Lab / Triage  
Priority: P2

Q: Nearest passing run comparison asks what?  
A: Same build/different device, same device/different build or same scenario/different install/profile.  
Category: Device Lab / Debugging  
Priority: P2

Q: First causal log line matters why?  
A: The final runner/UAT wrapper error often hides the real cook, install, launch or fatal error.  
Category: Device Lab / Debugging  
Priority: P2

Q: Missing telemetry debugging first checks what?  
A: Command line, output path, scenario start marker, crash-before-flush and artifact pull timing.  
Category: Device Lab / Debugging  
Priority: P2

Q: Gauntlet fits device labs as what?  
A: A UE-aware orchestration option for RunUnreal tests, boot tests, target automation and monitored packaged runs.  
Category: Device Lab / Gauntlet  
Priority: P2

Q: Gauntlet does not replace what?  
A: Project-owned scenario design, assertions, telemetry, artifact policy and target-branch verification.  
Category: Device Lab / Gauntlet  
Priority: P2

Q: Target-device adapter caveat?  
A: Console/mobile deployment can require target-device integration support beyond a generic command line.  
Category: Device Lab / Gauntlet  
Priority: P2

Q: Forced crash proof catches what?  
A: Missing or wrong symbols, build ID mismatch, crash endpoint errors and unusable crash context.  
Category: Device Lab / Crash Proof  
Priority: P2

Q: Device-lab failure should assign what?  
A: An evidence-based owner such as product, build, content, platform, performance, test or infrastructure.  
Category: Device Lab / Triage  
Priority: P2

Q: Device-lab automation strengthens interviews because it shows what?  
A: The candidate can connect build, platform, profiling, debugging and release evidence into a defensible workflow.  
Category: Device Lab / Interview Strategy  
Priority: P2

Q: World Partition streams what around what?  
A: Runtime grid cells around active streaming sources.  
Category: World Partition / Streaming  
Priority: P1

Q: `Is Spatially Loaded` is mainly what kind of decision?  
A: An ownership/lifetime decision about whether an Actor belongs to spatial streaming or should remain always loaded.  
Category: World Partition / Actor Policy  
Priority: P2

Q: Durable large-world state should not live only where?  
A: In unloadable spatial Actors or currently active Data Layers.  
Category: World Partition / State  
Priority: P2

Q: Runtime Data Layers are best treated as what?  
A: Streaming/presentation state driven by durable gameplay state.  
Category: World Partition / Data Layers  
Priority: P2

Q: Data Layer Asset versus Instance?  
A: The Asset is reusable identity/configuration; the Instance is world-specific state/configuration referencing it.  
Category: World Partition / Data Layers  
Priority: P2

Q: Editor Data Layer mistake?  
A: Assuming an editor organisation/visibility layer is runtime-active gameplay state.  
Category: World Partition / Data Layers  
Priority: P2

Q: Fast travel readiness needs more than what?  
A: More than a destination transform or blind delay; it needs streaming and project-specific collision/gameplay readiness.  
Category: World Partition / Fast Travel  
Priority: P2

Q: A streaming source readiness gate should usually check what after cell loading?  
A: Collision, nav, key gameplay representatives and Runtime Data Layer state.  
Category: World Partition / Fast Travel  
Priority: P2

Q: OFPA is runtime streaming or editor storage?  
A: Editor/source-control storage; cooked Actor data is embedded in level data.  
Category: World Partition / OFPA  
Priority: P2

Q: OFPA reduces what source-control problem?  
A: Multiple people dirtying one monolithic level file for separate Actor edits.  
Category: World Partition / OFPA  
Priority: P2

Q: OFPA still needs changelist validation because why?  
A: External actor filenames are encoded and partial submits can leave dangling references.  
Category: World Partition / OFPA  
Priority: P2

Q: Level Instance is strongest for what?  
A: Reusable authored Actor arrangements such as POIs or building clusters.  
Category: World Building / Level Instances  
Priority: P2

Q: Packed Level Blueprint is strongest for what?  
A: Static dense visual arrangements where optimised representation matters.  
Category: World Building / Level Instances  
Priority: P2

Q: High-density Level Streaming-mode Level Instances require what?  
A: Skepticism and profiling because runtime cost can be significant.  
Category: World Building / Level Instances  
Priority: P2

Q: HLOD replaces what?  
A: Distant spatial groups with proxy representations.  
Category: World Partition / HLOD  
Priority: P2

Q: HLOD does not fix what by itself?  
A: Near-field Actor activation, collision/nav setup, shader/texture hitches or gameplay state problems.  
Category: World Partition / HLOD  
Priority: P2

Q: HLOD evaluation should compare which ranges?  
A: Far, transition and near-field positions.  
Category: World Partition / HLOD  
Priority: P2

Q: Material merging in HLOD can cause what?  
A: Visual artefacts, especially with world-position-dependent or actor-position-dependent materials.  
Category: World Partition / HLOD  
Priority: P2

Q: PCG output in World Partition needs checking for what?  
A: Ownership, cleanup, Data Layer assignment, HLOD assignment, cook inclusion and runtime behavior.  
Category: PCG / World Partition  
Priority: P2

Q: A visual HLOD proxy proves what about gameplay Actors?  
A: Nothing by itself; gameplay Actors, collision and nav still need readiness proof.  
Category: World Partition / HLOD  
Priority: P2

Q: Too much always-loaded world content often comes from what?  
A: Hard references, Level Blueprint references, manager references or cross-cell actor references.  
Category: World Partition / Debugging  
Priority: P2

Q: World Partition commandlet `-ReportOnly` is useful for what?  
A: Reviewing conversion impact before modifying/migrating content.  
Category: World Partition / Automation  
Priority: P2

Q: `-SkipStableGUIDValidation` should be treated how?  
A: As a diagnostic/emergency option, not a normal migration policy.  
Category: World Partition / Automation  
Priority: P2

Q: Packaged large-world proof should include what scenario types?  
A: Traversal, fast travel, Data Layer state changes and HLOD transition checks.  
Category: World Partition / Validation  
Priority: P2

Q: Strong open-world interview answers connect which areas?  
A: World Partition, Data Layers, OFPA, Level Instances, HLOD, generated content, cook/package and target profiling evidence.  
Category: World Partition / Interview Strategy  
Priority: P2

Q: Large-world bug evidence packet should first classify what?  
A: The failing phase: authoring, source control, streaming, runtime state, rendering, cook/package, target device or automation.  
Category: World Partition / Debugging  
Priority: P2

Q: Fast-travel readiness state machine starts with what?  
A: Request, destination source enabled, streaming ready, gameplay ready, commit teleport and cleanup source.  
Category: World Partition / Fast Travel  
Priority: P2

Q: Region that never unloads is usually debugged through what first?  
A: Always-loaded actors and hard reference chains to spatial actors.  
Category: World Partition / Debugging  
Priority: P2

Q: Late joiner seeing wrong world phase suggests what boundary issue?  
A: Runtime Data Layer presentation is not being reconstructed from authoritative durable world state.  
Category: World Partition / Multiplayer  
Priority: P2

Q: OFPA review should map encoded files to what?  
A: Meaningful Actor labels and intent through editor/source-control changelist views.  
Category: World Partition / OFPA  
Priority: P2

Q: Level Instance content missing after embedding first asks what?  
A: Which Level Instance mode is used and whether contained Actors are OFPA-compatible.  
Category: World Building / Level Instances  
Priority: P2

Q: HLOD proxy visual artefact first evidence includes what?  
A: HLOD Layer settings, proxy assets, source materials and fixed-camera before/after screenshots.  
Category: World Partition / HLOD  
Priority: P2

Q: Stale HLOD shipped after content change indicates what missing gate?  
A: Builder output and stale generated-proxy validation in the release pipeline.  
Category: World Partition / Automation  
Priority: P2

Q: Generated biome duplicates after reload usually means what?  
A: Output ownership and cleanup are not idempotent across cell reload/regeneration.  
Category: PCG / World Partition  
Priority: P2

Q: Rare event asset missing only in package points to what?  
A: Cook dependency closure, soft-reference management, Primary Asset rules or redirectors.  
Category: World Partition / Cook  
Priority: P2

Q: Multiple runtime grids should be justified by what?  
A: Measured runtime behavior, not authoring organisation alone.  
Category: World Partition / Grid Policy  
Priority: P2

Q: Large-world pre-submit gate should emit failures with what?  
A: Stable IDs, severity, owner, affected actor/region and suggested fix.  
Category: World Partition / Automation  
Priority: P2

Q: Static editor screenshot is weak large-world evidence because why?  
A: It does not prove streaming, layer state, cook inclusion, HLOD transitions or target-device hitches.  
Category: World Partition / Validation  
Priority: P2

Q: Large-world interview answer score 5 includes what beyond a fix?  
A: Reusable gate, target proof and clear owner assignment.  
Category: World Partition / Interview Strategy  
Priority: P2

Q: A visual proxy showing a removed building means what likely happened?  
A: HLOD output is stale or generated proxy assets were not rebuilt/submitted/cooked correctly.  
Category: World Partition / HLOD  
Priority: P2

Q: Android profiling should start from what build type?  
A: A packaged target build with known Device Profile, scalability and scenario metadata.  
Category: Android Profiling / Evidence  
Priority: P2

Q: Unreal Insights tells you what that AGI may not?  
A: Engine-side scopes, tasks, loading, memory and project markers/ownership.  
Category: Android Profiling / Tool Selection  
Priority: P2

Q: AGI or APA tells you what that Unreal Insights may not?  
A: Android device, OS, GPU queue, present wait, render-pass and counter behavior.  
Category: Android Profiling / Tool Selection  
Priority: P2

Q: Android total CPU frame time can include what misleading cost?  
A: CPU waiting for GPU completion or present/swap rather than active app code.  
Category: Android Profiling / CPU-GPU  
Priority: P2

Q: Active CPU time means what in Android frame diagnosis?  
A: Time the CPU is actually running app code rather than idle/waiting.  
Category: Android Profiling / CPU-GPU  
Priority: P2

Q: `eglSwapBuffers` or `vkQueuePresentKHR` waits can indicate what?  
A: The CPU may be waiting for GPU/present work, so the frame may be GPU-bound.  
Category: Android Profiling / CPU-GPU  
Priority: P2

Q: AGI Frame Profiler is best for what scope?  
A: In-depth analysis of a representative individual GPU frame and its render passes.  
Category: Android Profiling / AGI  
Priority: P2

Q: Expensive AGI binning usually points toward what?  
A: Vertex data, vertex count, geometry or related tiler/geometry pressure.  
Category: Android Profiling / Render Passes  
Priority: P2

Q: Expensive AGI rendering usually points toward what?  
A: Fragment shading, texture fetches, overdraw or high-resolution framebuffer cost.  
Category: Android Profiling / Render Passes  
Priority: P2

Q: Expensive GMEM load/store points toward what?  
A: Render-target, depth/stencil or framebuffer load/store bandwidth.  
Category: Android Profiling / Render Passes  
Priority: P2

Q: GPU counters are safe only when paired with what?  
A: GPU vendor/driver/tool context and a concrete hypothesis.  
Category: Android Profiling / Counters  
Priority: P2

Q: Do not compare Android GPU counters how?  
A: As raw universal metrics across Mali, PowerVR and Adreno.  
Category: Android Profiling / Counters  
Priority: P2

Q: Android LMK debugging should include what Unreal evidence?  
A: LLM tags, Memory Insights or memory checkpoints around lifecycle transitions.  
Category: Android Profiling / Memory  
Priority: P2

Q: User-perceived LMK rate above 1% means what in Android vitals docs?  
A: A critical need for immediate action.  
Category: Android Profiling / Memory  
Priority: P2

Q: Low LMK rate proves what?  
A: Not necessarily memory health; background kills may still hurt warm start and multitasking.  
Category: Android Profiling / Memory  
Priority: P2

Q: ADPF should be evaluated against what first?  
A: A no-adaptation baseline with visible frame, thermal and quality behavior.  
Category: Android Profiling / ADPF  
Priority: P2

Q: Unreal ADPF plugin can adjust what based on thermal state?  
A: Unreal Scalability quality levels according to thermal headroom/status.  
Category: Android Profiling / ADPF  
Priority: P2

Q: ADPF performance hints report what?  
A: Target and actual frame duration for Game, Render and RHI thread sessions where supported.  
Category: Android Profiling / ADPF  
Priority: P2

Q: Fixed Performance Mode is useful for what?  
A: Repeatable benchmark comparisons where available, not normal sustained user behavior.  
Category: Android Profiling / Benchmarking  
Priority: P3

Q: Android profiler report must name what owner?  
A: Whether the causal owner is gameplay, rendering, content, memory, platform, build or test infrastructure.  
Category: Android Profiling / Evidence  
Priority: P2

Q: Final RunUAT wrapper error is not what?  
A: The diagnosis; find the first causal error and failed phase.  
Category: Build Farm / Debugging  
Priority: P2

Q: Build failure phase classification should name what?  
A: Build, cook, stage, package, deploy, launch, smoke, archive, symbol upload or runtime.  
Category: Build Farm / Debugging  
Priority: P2

Q: Clean CI failure after local success usually means what?  
A: An undeclared dependency hidden by local binaries, generated files, cache, global installs or editor-loaded content.  
Category: Build Farm / CI  
Priority: P2

Q: Unity build can hide what?  
A: Missing includes, missing dependencies and include-order problems.  
Category: Build Farm / CI  
Priority: P2

Q: Runtime module safe from editor dependency requires what proof?  
A: Game/Server/Shipping build and package proof, not only Editor build success.  
Category: Build Farm / Modules  
Priority: P1

Q: `WITH_EDITOR` does not fix what by itself?  
A: A Runtime module Build.cs dependency on Editor-only modules.  
Category: Build Farm / Modules  
Priority: P1

Q: Third-party runtime dependency is release-ready only after what?  
A: It is staged and launched on a clean machine/device without global installs.  
Category: Build Farm / Staging  
Priority: P2

Q: Package missing asset should be fixed by what first?  
A: Correct cook/Asset Manager/dependency rule, not broad "cook everything" by default.  
Category: Build Farm / Cook  
Priority: P2

Q: Forced packaged crash proves what?  
A: Package, build ID, symbols, crash metadata and symbolication path match.  
Category: Build Farm / Crash Proof  
Priority: P2

Q: Symbols from a similar build are what?  
A: Weak evidence; symbols must match the exact shipped binary/build ID.  
Category: Build Farm / Crash Proof  
Priority: P2

Q: BuildGraph node dependency should express what?  
A: Artifact input/output flow between producer and consumer nodes.  
Category: Build Farm / BuildGraph  
Priority: P3

Q: Agent-local file assumption breaks what?  
A: Multi-agent BuildGraph or CI jobs where artifacts must be explicitly passed/archive.  
Category: Build Farm / BuildGraph  
Priority: P3

Q: Smoke test passing frontend proves what about gameplay maps?  
A: Little; release-critical maps need explicit launch/readiness coverage.  
Category: Build Farm / Smoke Tests  
Priority: P2

Q: Telemetry artifact missing should do what to the job?  
A: Fail or mark blocked by missing evidence instead of silently passing.  
Category: Build Farm / Telemetry  
Priority: P2

Q: Rerun CI is acceptable only for what?  
A: Defined infrastructure/flaky classes, not deterministic product failures.  
Category: Build Farm / Triage  
Priority: P2

Q: Custom AI sense should model what?  
A: Recurring game-specific sensory evidence, not durable gameplay truth or one-off triggers.  
Category: AI / Custom Senses  
Priority: P2

Q: Custom sense should write Blackboard keys directly?  
A: Usually no; it should report evidence, and a memory/projection layer should write decision state.  
Category: AI / Custom Senses  
Priority: P2

Q: Custom stimulus minimum fields?  
A: Source/target identity, location, strength, tag/type, time/age, success/loss where relevant and stable identity if needed.  
Category: AI / Custom Senses  
Priority: P2

Q: Perception evidence differs from gameplay truth how?  
A: Evidence informs memory/decisions; authoritative gameplay state still validates outcomes separately.  
Category: AI / Perception  
Priority: P1

Q: Failed sight event should not automatically do what?  
A: Erase all hearing, damage, scent or investigation memory.  
Category: AI / Perception  
Priority: P1

Q: AI remembers forever bug usually starts with what check?  
A: Inspect per-sense age/decay/Max Age and whether another sense is refreshing memory.  
Category: AI / Debugging  
Priority: P2

Q: Custom sense never fires first reduction?  
A: One listener, one source, known World and known event location.  
Category: AI / Debugging  
Priority: P2

Q: Wrong World in sense events causes what?  
A: The event can be emitted but never reach the listener/sense instance being debugged.  
Category: AI / Custom Senses  
Priority: P2

Q: Custom sense hot loop should filter before what?  
A: Before expensive traces, path checks or per-listener UObject work.  
Category: AI / Performance  
Priority: P2

Q: Custom sense scale profile counts what first?  
A: Events emitted, merged/discarded, listeners considered, traces, stimuli, callbacks and memory updates.  
Category: AI / Performance  
Priority: P2

Q: Event batching in a custom sense prevents what?  
A: Repeating the same expensive listener/filter work for many similar evidence events.  
Category: AI / Performance  
Priority: P2

Q: Source destroyed before custom sense update requires what?  
A: Weak/object validity checks and stable ID/generation if durable identity matters.  
Category: AI / Custom Senses  
Priority: P2

Q: Custom AI sense API examples should be treated how?  
A: Schematic until verified against the exact UE5 target branch headers.  
Category: AI / Version Sensitivity  
Priority: P2

Q: MassAI in this package means what?  
A: Mass-based data/processor AI or crowd workflows, not a single stable monolithic UE subsystem.  
Category: MassEntity / AI  
Priority: P2

Q: MassAI is not what?  
A: Behaviour Tree or StateTree made automatically faster for every NPC.  
Category: MassEntity / AI  
Priority: P2

Q: Good MassAI candidate?  
A: Large populations with stable data shapes, batchable movement/activity and representation LOD.  
Category: MassEntity / AI  
Priority: P2

Q: Poor MassAI candidate?  
A: A unique rich boss/player-style Actor with bespoke physics, animation, replication and logic.  
Category: MassEntity / AI  
Priority: P2

Q: ZoneGraph is useful for what?  
A: Structured lane/zone movement such as pedestrians or traffic, especially with Mass crowds.  
Category: MassEntity / ZoneGraph  
Priority: P2

Q: ZoneGraph is not simply what?  
A: A universally faster NavMesh replacement.  
Category: MassEntity / ZoneGraph  
Priority: P2

Q: NavMesh versus ZoneGraph mental model?  
A: NavMesh answers general traversability; ZoneGraph/lane data answers structured movement routes.  
Category: AI / Navigation  
Priority: P2

Q: Mass crowd route failure differs from avoidance failure how?  
A: Route/lane state may be wrong even if local steering works, or avoidance may jam despite valid route.  
Category: MassEntity / Debugging  
Priority: P2

Q: Mass StateTree should operate on what?  
A: Entity context/fragments/signals, not arbitrary per-entity Actor calls in hot loops.  
Category: MassEntity / StateTree  
Priority: P2

Q: Durable Mass activity state belongs where?  
A: In fragments or equivalent Mass data, not only task-local variables.  
Category: MassEntity / StateTree  
Priority: P2

Q: Smart Object candidate found differs from claimed how in Mass?  
A: Candidate is possible; claim reserves a specific slot under authoritative rules.  
Category: MassEntity / Smart Objects  
Priority: P2

Q: Mass Smart Object claim handle should survive what?  
A: StateTree/task transitions long enough to release on success, failure, abort, despawn or demotion.  
Category: MassEntity / Smart Objects  
Priority: P2

Q: Mass Smart Object claim leak test?  
A: Destroy/despawn all users and prove no slot remains claimed.  
Category: MassEntity / Smart Objects  
Priority: P2

Q: Actor promotion in Mass is not just what?  
A: Spawning an Actor near the player.  
Category: MassEntity / Hybrid Actors  
Priority: P2

Q: Actor promotion contract must name what writer?  
A: The authoritative transform/state writer during promoted and demoted states.  
Category: MassEntity / Hybrid Actors  
Priority: P2

Q: Dual-writer Mass promotion bug symptom?  
A: Mass movement and Actor movement both write transform, causing jitter or state correction.  
Category: MassEntity / Hybrid Actors  
Priority: P2

Q: Actor demotion must invalidate what?  
A: External Actor references and pooled Actor state that no longer represents the entity.  
Category: MassEntity / Hybrid Actors  
Priority: P2

Q: MassAI profiling must split what layers?  
A: Simulation, structure, bridge, StateTree/Smart Object, representation, rendering and networking where relevant.  
Category: MassEntity / Performance  
Priority: P2

Q: Fair Actor versus MassAI benchmark requires what?  
A: Equivalent behaviour and visual fidelity before comparing performance.  
Category: MassEntity / Performance  
Priority: P2

Q: MassAI representation win can be false if what changed?  
A: The Mass version removed skeletal animation, collision, replication or interaction fidelity.  
Category: MassEntity / Performance  
Priority: P2

Q: Mass crowd jam debugging starts by separating what?  
A: Route/lane choice, local avoidance, movement integration, reservations, collision and representation.  
Category: MassEntity / Debugging  
Priority: P2

Q: Mass lane bottleneck first metrics?  
A: Lane/route state, desired velocity, avoidance data, density and processor execution.  
Category: MassEntity / Debugging  
Priority: P2

Q: Smart Object use from Mass may need Actor promotion when?  
A: The interaction requires rich animation, collision, ability logic or network presentation unavailable in the current representation tier.  
Category: MassEntity / Smart Objects  
Priority: P2

Q: Mass signal without durable fragment state risks what?  
A: Waking logic that cannot reconstruct the actual activity/claim/movement state.  
Category: MassEntity / StateTree  
Priority: P2

Q: Perception aggregation pattern does what?  
A: Collects shared evidence for a group/crowd instead of every agent performing full sensing.  
Category: AI / Scaling  
Priority: P2

Q: Evidence tiering means what?  
A: Full perception near important gameplay and coarse/less frequent evidence for distant or ambient agents.  
Category: AI / Scaling  
Priority: P2

Q: Custom sense source-side throttling fixes what class of problem?  
A: Too many evidence events before perception filtering even begins.  
Category: AI / Performance  
Priority: P2

Q: MassAI exact plugin behaviour should be verified where?  
A: In target UE5.3-UE5.6 source, enabled plugins and project sample/prototype, not inferred from maintained docs alone.  
Category: MassEntity / Version Sensitivity  
Priority: P2

Q: Mass Crowd and ZoneGraph docs are what kind of source here?  
A: Official conceptual anchors; exact runtime APIs and tooling remain branch-sensitive.  
Category: MassEntity / Sources  
Priority: P2

Q: Strong custom sense interview answer emphasises what boundary?  
A: Sense reports stimuli; memory and decision layers interpret them; authority validates gameplay outcomes.  
Category: AI / Interview Strategy  
Priority: P2

Q: Strong MassAI interview answer emphasises what boundary?  
A: Mass simulation, representation, Actor promotion and rich gameplay authority are separate layers.  
Category: MassEntity / Interview Strategy  
Priority: P2

Q: Animation Notify should not be the sole authority for what?  
A: Durable gameplay outcomes such as damage, inventory changes or authoritative hit validation.  
Category: Animation / Authority  
Priority: P2

Q: Montage cleanup must handle which paths?  
A: Normal end, blend-out, interruption, death, stun, unequip, mesh destruction and task cancellation.  
Category: Animation / Debugging  
Priority: P2

Q: Root motion movement bug starts with what question?  
A: Who owns capsule movement this frame: animation/root motion, CharacterMovement, script or physics?  
Category: Animation / Movement  
Priority: P2

Q: Blocking collision guarantees Hit event?  
A: No; collision response and event notification/movement path are separate.  
Category: Physics / Collision  
Priority: P1

Q: Sweep movement is what?  
A: An explicit kinematic movement/scene-query path, not the same as solver-driven physics simulation.  
Category: Physics / Movement  
Priority: P2

Q: Generate Hit Events alone may not fix what?  
A: Wrong collision mode, profile, movement API, sweep path, component, simple/complex setup or simulation state.  
Category: Physics / Debugging  
Priority: P2

Q: UUserWidget Construct can run more than once because what can rebuild?  
A: The underlying Slate widget tree.  
Category: UI / Lifecycle  
Priority: P2

Q: Widget one-time external delegate binding should be what?  
A: Idempotent and released at the matching teardown boundary.  
Category: UI / Lifecycle  
Priority: P2

Q: ListView entry widget identity differs from what?  
A: The item/data identity it is currently presenting.  
Category: UI / Lists  
Priority: P2

Q: Recycled ListView row stale icon usually means what?  
A: An async callback or cached state updated a widget after it represented a different item.  
Category: UI / Lists  
Priority: P2

Q: Invalidation Box primarily reduces what?  
A: Repeated widget layout/paint work when unchanged UI remains valid.  
Category: UI / Performance  
Priority: P2

Q: Retainer Panel adds what trade-offs?  
A: Render-target memory, capture/composite cost, latency and GPU/overdraw considerations.  
Category: UI / Performance  
Priority: P2

Q: Audio concurrency is what kind of policy?  
A: Presentation resource/admission policy, not gameplay authority.  
Category: Audio / Concurrency  
Priority: P2

Q: Critical gameplay warning should not rely only on what?  
A: A sound that can be stolen, culled, virtualised or inaudible.  
Category: Audio / Gameplay  
Priority: P2

Q: MetaSounds are still limited by what?  
A: Voice count, DSP graph cost, parameter churn, routing and target platform resources.  
Category: Audio / MetaSounds  
Priority: P2

Q: Procedural audio does not mean what?  
A: Unlimited polyphony or free runtime cost.  
Category: Audio / Performance  
Priority: P2

Q: Niagara events should not authoritatively drive what?  
A: Damage, inventory, quest state or other durable gameplay outcomes.  
Category: Niagara / Gameplay  
Priority: P2

Q: Gameplay projectile damage should survive what VFX test?  
A: Disabling Niagara should not disable authoritative damage.  
Category: Niagara / Authority  
Priority: P2

Q: Pooling an effect or sound requires resetting what?  
A: Parameters, attachment, transform, owner/context, callbacks, activation state and gameplay references.  
Category: VFX / Audio / Pooling  
Priority: P2

Q: Pooling trades spawn cost for what?  
A: Reset complexity and retained memory.  
Category: Performance / Pooling  
Priority: P2

Q: Non-uniform scale makes normal transforms tricky because normals must remain what?  
A: Perpendicular to the transformed surface.  
Category: Maths / Transforms  
Priority: P1

Q: Points, vectors and normals share the same transform semantics?  
A: No; translation affects points, and normals need special handling under non-uniform scale.  
Category: Maths / Transforms  
Priority: P1

Q: Spatial hash expected speed depends on what?  
A: Cell size, query radius, object distribution, update churn and duplicate boundary handling.  
Category: Algorithms / Spatial Hash  
Priority: P2

Q: Spatial hash worst case can approach what?  
A: All-pairs search plus overhead when distribution/query assumptions collapse.  
Category: Algorithms / Spatial Hash  
Priority: P2

Q: Spatial structure benchmark must include what distributions?  
A: Uniform, clustered and adversarial/line-like distributions, not only random uniform cases.  
Category: Algorithms / Verification  
Priority: P2

Q: Lua/.NET GC and Unreal GC trace what?  
A: Different root sets and lifetimes.  
Category: Scripting / Lifetime  
Priority: P2

Q: Script UObject wrapper must not imply what?  
A: That the underlying UObject is alive or safe to call.  
Category: Scripting / Lifetime  
Priority: P2

Q: Script callback after world unload requires what?  
A: Explicit unregistration or validity checks that safely reject stale native objects.  
Category: Scripting / Debugging  
Priority: P2

Q: Managed wrapper releases binding resources but should not do what?  
A: Arbitrarily delete UObject memory.  
Category: C# / Interop  
Priority: P2

Q: Lua/C# runtime support in Unreal is what?  
A: Plugin- or studio-owned, not baseline Unreal gameplay runtime.  
Category: Scripting / Caveats  
Priority: P2

Q: Raw UObject pointer is unsafe when what is unclear?  
A: What keeps the object alive and what invalidates the pointer before use.  
Category: UObject / Lifetime  
Priority: P0

Q: `TObjectPtr` is mainly for what?  
A: Reflected UObject references that participate in UE object tracking/reachability semantics.  
Category: UObject / Pointers  
Priority: P0

Q: `TWeakObjectPtr` does what?  
A: Observes a UObject without keeping it alive and must be validity-checked on use.  
Category: UObject / Pointers  
Priority: P0

Q: `TSoftObjectPtr` represents what?  
A: A path/reference to an object or asset that may be unloaded and needs load/cook policy.  
Category: UObject / Soft References  
Priority: P0

Q: Constructor should not usually do what?  
A: World-dependent gameplay side effects.  
Category: Gameplay Framework / Lifecycle  
Priority: P0

Q: Construction script or OnConstruction must be what?  
A: Idempotent, because it can rerun during editor/instance changes.  
Category: Gameplay Framework / Lifecycle  
Priority: P1

Q: Subsystem scope should be what?  
A: The narrowest correct host lifetime: Engine, Editor, GameInstance, World or LocalPlayer.  
Category: Gameplay Architecture / Subsystems  
Priority: P1

Q: Subsystem can still become what anti-pattern?  
A: A hidden Service Locator/global bucket.  
Category: Gameplay Architecture / Subsystems  
Priority: P1

Q: BlueprintPure should not hide what?  
A: Mutation, asset loads, heavy allocation, traces or order-dependent side effects.  
Category: Blueprint / API Design  
Priority: P1

Q: Pure node does not mean what?  
A: Free, cached, side-effect-safe or evaluated only once.  
Category: Blueprint / API Design  
Priority: P1

Q: Dynamic delegates are justified by what need?  
A: Reflection, Blueprint assignment, serialisation or editor-facing event exposure.  
Category: UE C++ / Delegates  
Priority: P1

Q: Native delegates are often better for what?  
A: C++-only and hot-path event flows where reflection is unnecessary.  
Category: UE C++ / Delegates  
Priority: P1

Q: Async Blueprint proxy must define what lifecycle?  
A: Owner/world context, activation, completion, cancellation and cleanup.  
Category: Blueprint / Async  
Priority: P2

Q: Async proxy stale callback prevention needs what?  
A: Weak validation, delegate unbinding and world/caller teardown handling.  
Category: Blueprint / Async  
Priority: P2

Q: Data Asset should usually hold what?  
A: Shared authored definition data, not mutable per-instance runtime state.  
Category: Gameplay Architecture / Data Assets  
Priority: P1

Q: Runtime mutable item state belongs where?  
A: In an instance/component/save record, not the shared definition asset.  
Category: Gameplay Architecture / Data Assets  
Priority: P1

Q: Asset Registry query can become a load when what is called?  
A: `FAssetData::GetAsset` or equivalent object resolution.  
Category: Assets / Registry  
Priority: P2

Q: Metadata-only asset discovery avoids what?  
A: Synchronous UObject loading and dependency hitches during scans/lists.  
Category: Assets / Registry  
Priority: P2

Q: Primary Asset bundle should express what?  
A: Purpose-specific dependency groups such as UI, gameplay, preview, audio or cinematic.  
Category: Assets / Asset Manager  
Priority: P2

Q: Bad bundle boundaries can load what accidentally?  
A: UI, gameplay meshes, audio, VFX and large dependencies together.  
Category: Assets / Memory  
Priority: P2

Q: Live Coding proves clean packaged correctness?  
A: No; it patches a running editor and can hide startup, reflection, layout and serialisation issues.  
Category: Build / Live Coding  
Priority: P2

Q: After reflected layout/default changes, what proof is stronger than Live Coding?  
A: Full restart/rebuild and target package or clean editor validation.  
Category: Build / Live Coding  
Priority: P2

Q: Save/load is not just what?  
A: Dumping current Actor fields.  
Category: Gameplay Architecture / Save  
Priority: P1

Q: Save systems require stable what?  
A: Stable durable object IDs plus schema/versioning and reconstruction rules.  
Category: Gameplay Architecture / Save  
Priority: P1

Q: SaveGame specifier is incomplete until what is defined?  
A: The consuming save system, identity policy, versioning and load order.  
Category: Gameplay Architecture / Save  
Priority: P1

Q: Loaded save should rebuild what separately?  
A: Transient references, runtime caches and derived state.  
Category: Gameplay Architecture / Save  
Priority: P1

Q: Soft references do not guarantee what by themselves?  
A: Cook inclusion, load timing, residency or memory budget correctness.  
Category: Assets / Soft References  
Priority: P1

Q: Data validation catches what class of issue early?  
A: Authored content invariants before runtime, cook or packaged failure.  
Category: Tools / Data Validation  
Priority: P2

Q: Delegate lifetime bugs often come from what?  
A: Duplicate binding, missing unbind, stale captures or re-entrant broadcast assumptions.  
Category: UE C++ / Delegates  
Priority: P1

Q: CDO/default propagation bugs often appear after what?  
A: Constructor/default changes tested only through Live Coding or already-loaded instances.  
Category: UObject / CDO  
Priority: P1

## GAS Specialist Expansion

Q: GAS specialist debugging should separate which three ledgers?  
A: ASC gameplay state, ability activation state and presentation state.  
Category: GAS / Debugging  
Priority: P2

Q: A useful GAS activation trace should include what correlation key?  
A: The ability spec handle and activation/prediction key.  
Category: GAS / Observability  
Priority: P2

Q: Why log owner and avatar in GAS traces?  
A: Wrong owner/avatar or stale avatar actor info commonly breaks activation, respawn and prediction.  
Category: GAS / Lifetime  
Priority: P2

Q: Why is listen-server proof weak for GAS prediction?  
A: It can hide remote owning-client timing, ownership and prediction/reconciliation failures.  
Category: GAS / Networking  
Priority: P2

Q: Pawn-owned ASC is simplest when what is true?  
A: Ability/attribute/effect state should die with the Pawn.  
Category: GAS / Lifetime  
Priority: P2

Q: PlayerState-owned ASC is attractive when what must persist?  
A: Attributes, cooldowns, effects or abilities across possession/respawn.  
Category: GAS / Lifetime  
Priority: P2

Q: Respawn with PlayerState-owned ASC must refresh what?  
A: The avatar actor info and old-avatar input/task/delegate bindings.  
Category: GAS / Lifetime  
Priority: P2

Q: Ability grant source ledger records what?  
A: Source ID, ability class, level/source data and spec handle.  
Category: GAS / Grants  
Priority: P2

Q: Removing an ability by class is unsafe when what can happen?  
A: Multiple sources can grant the same ability class.  
Category: GAS / Grants  
Priority: P2

Q: Authority-only GiveAbility means what for client code?  
A: Client-side grant attempts are ignored or invalid for normal authoritative grants.  
Category: GAS / Grants  
Priority: P2

Q: Duplicate startup grants often come from what lifecycle events?  
A: Possession, respawn, loadout replication, equipment rebuilds and PIE startup paths.  
Category: GAS / Grants  
Priority: P2

Q: Attribute invariant proof requires testing what?  
A: Every mutation path, not just one callback.  
Category: GAS / Attributes  
Priority: P2

Q: Health and MaxHealth invariant usually states what?  
A: Health remains within `[0, MaxHealth]`, including after MaxHealth changes.  
Category: GAS / Attributes  
Priority: P2

Q: MaxHealth reduction should not automatically imply what?  
A: Damage attribution unless the design explicitly says so.  
Category: GAS / Attributes  
Priority: P2

Q: IncomingDamage meta-style attributes must be reset why?  
A: To prevent repeated consumption of the same transient damage value.  
Category: GAS / Attributes  
Priority: P2

Q: Attribute UI should observe what rather than own truth?  
A: ASC-derived attribute/tag/effect model state.  
Category: GAS / UI  
Priority: P2

Q: Direct Gameplay Effect modifier fits what case?  
A: A simple scalar change to one attribute.  
Category: GAS / Effects  
Priority: P2

Q: Custom magnitude calculation fits what case?  
A: One derived magnitude from captured attributes/tags.  
Category: GAS / Effects  
Priority: P2

Q: Execution calculation fits what case?  
A: Authoritative multi-attribute outcomes such as shield, armour, crit and health routing.  
Category: GAS / Effects  
Priority: P2

Q: Set-by-caller values must be what?  
A: Present, validated and bounded or server-derived.  
Category: GAS / Effects  
Priority: P2

Q: Capture timing answers what design question?  
A: Whether a calculation uses state from spec creation/application or later evaluation.  
Category: GAS / Effects  
Priority: P2

Q: Projectile buff timing test catches what?  
A: Whether launch-time or impact-time captured values drive final damage.  
Category: GAS / Effects  
Priority: P2

Q: Stacking policy is incomplete if it only says what?  
A: The maximum stack count.  
Category: GAS / Stacking  
Priority: P2

Q: Complete stacking policy includes what expiry question?  
A: Whether expiration removes one stack or all stacks.  
Category: GAS / Stacking  
Priority: P2

Q: Stacking source attribution matters because what?  
A: Different source aggregation/removal rules can change gameplay and UI.  
Category: GAS / Stacking  
Priority: P2

Q: A stack overflow policy answers what?  
A: What happens when a new stack arrives at the limit.  
Category: GAS / Stacking  
Priority: P2

Q: GAS tags from effects are counted, so removal must do what?  
A: Remove the source without clearing tags still granted by others.  
Category: GAS / Tags  
Priority: P2

Q: Loose tags are safe only with what?  
A: One documented owner and symmetric lifecycle.  
Category: GAS / Tags  
Priority: P2

Q: Local Predicted ability means the owner client does what?  
A: Runs supported provisional work before server confirmation.  
Category: GAS / Prediction  
Priority: P2

Q: Local Predicted does not mean what?  
A: The client owns authoritative gameplay truth.  
Category: GAS / Prediction  
Priority: P2

Q: Prediction key is used to correlate what?  
A: Client-predicted work with the server's confirm/reject decision.  
Category: GAS / Prediction  
Priority: P2

Q: Predicted custom side effects outside GAS need what?  
A: A reversible, cosmetic or server-only policy.  
Category: GAS / Prediction  
Priority: P2

Q: Damage/kill/objective reward should usually be what under prediction?  
A: Server-only authoritative state.  
Category: GAS / Prediction  
Priority: P2

Q: Predicted montage rejection requires what?  
A: Cleanup/interruption that returns animation and UI to coherent state.  
Category: GAS / Prediction  
Priority: P2

Q: Cost applied twice can indicate what?  
A: Manual application plus CommitAbility, duplicate grants, repeated input or prediction path duplication.  
Category: GAS / Debugging  
Priority: P2

Q: Cooldown tag stuck forever should be debugged by inspecting what?  
A: Active effects, loose tag owners, stack/duration policy and removal handles.  
Category: GAS / Debugging  
Priority: P2

Q: Server target data validation treats client hit data as what?  
A: Evidence or intent, not final authority.  
Category: GAS / Target Data  
Priority: P2

Q: Client should not send what in target data?  
A: Final unbounded damage such as "deal 500 damage."  
Category: GAS / Target Data  
Priority: P2

Q: Server target validation should check range and what else?  
A: Ownership, timing, line of sight, target state, cadence, resources, tags and faction/immunity rules.  
Category: GAS / Target Data  
Priority: P2

Q: Server rewind differs from full rollback how?  
A: Rewind evaluates past state for validation; rollback restores simulation state and resimulates.  
Category: GAS / Networking  
Priority: P2

Q: Ability Task safety is proven by what test?  
A: Cancel during every wait phase and confirm no late gameplay mutation.  
Category: GAS / Ability Tasks  
Priority: P2

Q: Custom Ability Task cleanup must release what?  
A: External delegates, timers, latent handles and stale owner references.  
Category: GAS / Ability Tasks  
Priority: P2

Q: Ability Task should broadcast after cancellation?  
A: No, unless explicitly designed and safe; late callbacks should be ignored or fail safely.  
Category: GAS / Ability Tasks  
Priority: P2

Q: Gameplay Cue should own what kind of result?  
A: Cosmetic/presentation results, not durable gameplay truth.  
Category: GAS / Gameplay Cues  
Priority: P2

Q: Disabling cue assets should not affect what?  
A: Health, tags, effects, inventory, objectives or win conditions.  
Category: GAS / Gameplay Cues  
Priority: P2

Q: Persistent cue cleanup should happen when what occurs?  
A: Effect removal, owner destruction, relevancy loss or pooled-effect reset.  
Category: GAS / Gameplay Cues  
Priority: P2

Q: Cue duplicate debugging should log cue tag and what?  
A: Prediction key, owner/avatar, role and whether it came from prediction, replication or explicit execution.  
Category: GAS / Gameplay Cues  
Priority: P2

Q: Full GE replication mode sends what?  
A: Full active Gameplay Effect information to relevant clients.  
Category: GAS / Replication  
Priority: P2

Q: Mixed GE replication is commonly useful when who needs detail?  
A: The owning client/autonomous proxy needs detail while simulated proxies need less.  
Category: GAS / Replication  
Priority: P2

Q: Minimal GE replication does not imply what?  
A: That attributes, tags or cues automatically stop replicating through their own paths.  
Category: GAS / Replication  
Priority: P2

Q: GE replication mode should be chosen from what?  
A: Product visibility requirements and bandwidth evidence.  
Category: GAS / Replication  
Priority: P2

Q: UI should request ability activation through what?  
A: Input/command adapter into GAS, not by owning ability truth itself.  
Category: GAS / UI  
Priority: P2

Q: GAS should usually not own what durable systems?  
A: Inventory identity, quest progress, save migration, economy or cross-container transactions.  
Category: GAS / Architecture  
Priority: P2

Q: GAS earns complexity when it removes duplicated what?  
A: Tag, cost, cooldown, effect, stacking, attribute and prediction policy.  
Category: GAS / Architecture  
Priority: P2

Q: Bespoke ability Component is often better when what is true?  
A: The rule set is small, server-only and easy to test without GAS concepts.  
Category: GAS / Architecture  
Priority: P2

Q: GAS prediction differs from full rollback because it is what?  
A: Scoped to supported provisional ability/effect paths rather than broad simulation resimulation.  
Category: GAS / Rollback  
Priority: P2

Q: Full rollback architecture needs what beyond GAS prediction keys?  
A: State snapshots, input logs, deterministic/controlled resimulation and side-effect control.  
Category: GAS / Rollback  
Priority: P2

Q: GAS profiling should count active effects and what else?  
A: Activations, periodic ticks, tag delegates, calculations, tasks, cues, Blueprint work and replication bytes.  
Category: GAS / Profiling  
Priority: P2

Q: High-frequency periodic effects should be changed only after what?  
A: Measuring their semantic need and actual execution cost.  
Category: GAS / Profiling  
Priority: P2

Q: GAS optimisation should not start by doing what?  
A: Removing semantic tags/effects blindly without tracing the measured bottleneck.  
Category: GAS / Profiling  
Priority: P2

Q: The first GAS-versus-bespoke comparison should use what?  
A: A representative vertical slice with prediction, tags, costs, stacking, respawn and debugging evidence.  
Category: GAS / System Design  
Priority: P2

## Apple and Console Profiling

Q: Apple/console profiling should correlate rather than replace what?  
A: Unreal stats/CSV/Insights and platform-specific evidence.  
Category: Platform Profiling / Evidence  
Priority: P2

Q: Xcode Metal capture is strongest for what?  
A: Metal workload details such as passes, pipeline states, counters and shader hot spots.  
Category: Apple / Metal  
Priority: P2

Q: Xcode Metal capture alone is weak at explaining what?  
A: Gameplay intent or UE system ownership without markers and map-back.  
Category: Apple / Metal  
Priority: P2

Q: Metal capture metadata must include what device details?  
A: Device model, GPU family, OS and Xcode version.  
Category: Apple / Evidence  
Priority: P2

Q: Metal capture metadata must include what UE details?  
A: UE branch/version, build config, RHI/settings, package/cook/shader/PSO state and scenario.  
Category: Apple / Evidence  
Priority: P2

Q: GPU Frame Capture can have what performance effect?  
A: A small but measurable CPU processing impact, so record whether it was enabled.  
Category: Apple / Metal  
Priority: P2

Q: Metal concurrent profiling represents what?  
A: Overlapped runtime-style GPU execution.  
Category: Apple / GPU  
Priority: P2

Q: Metal serial profiling is useful for what?  
A: Isolating pass costs, while not representing normal runtime performance.  
Category: Apple / GPU  
Priority: P2

Q: Metal trace replay requires what compatibility?  
A: Same device type, same GPU and same OS for reliable behavior.  
Category: Apple / Metal  
Priority: P2

Q: Simulator memory does not prove what?  
A: Safe iOS device memory under real memory warnings or termination behavior.  
Category: Apple / Memory  
Priority: P2

Q: Xcode memory graph shows what?  
A: Memory regions/objects/allocations and references between them.  
Category: Apple / Memory  
Priority: P2

Q: Xcode allocation stack traces help connect memory to what?  
A: The code path responsible for allocations.  
Category: Apple / Memory  
Priority: P2

Q: Instruments Allocations generation marks help isolate what?  
A: Allocations occurring during a specific feature or scenario window.  
Category: Apple / Memory  
Priority: P2

Q: UE iOS memory proof should combine Apple tools with what?  
A: UE Memory Insights, LLM and asset/render-target pool evidence.  
Category: Apple / Unreal  
Priority: P2

Q: Thread Performance Checker detects what classes of issue?  
A: Priority inversions and non-UI work on the main thread.  
Category: Apple / CPU  
Priority: P2

Q: Thread Performance Checker is not a replacement for what?  
A: Unreal GameThread profiling with stats/Insights/CSV.  
Category: Apple / CPU  
Priority: P2

Q: Priority inversion means what broadly?  
A: A lower-priority thread blocks higher-priority work, risking responsiveness.  
Category: Apple / CPU  
Priority: P2

Q: MetricKit provides what kind of reports?  
A: Daily real-device metric reports and diagnostics.  
Category: Apple / MetricKit  
Priority: P2

Q: MetricKit is useful for what decision?  
A: Prioritising field issues by device, OS, app version and metric category.  
Category: Apple / MetricKit  
Priority: P2

Q: MetricKit does not by itself provide what?  
A: Frame-level root cause for a single UE bottleneck.  
Category: Apple / MetricKit  
Priority: P2

Q: MetricKit signposts can help connect field data to what?  
A: Scenario or app-state markers.  
Category: Apple / MetricKit  
Priority: P2

Q: Xcode Organizer metrics should be turned into what?  
A: A device-lab reproduction task with UE/platform captures.  
Category: Apple / Organizer  
Priority: P2

Q: Apple GPU counter/capture findings must be mapped back to what?  
A: UE passes, content, materials, render targets, scene captures or shaders.  
Category: Apple / Rendering  
Priority: P2

Q: Expensive fragment pipeline on Metal may suggest what UE issues?  
A: Overdraw, material complexity, full-screen passes, translucency or resolution cost.  
Category: Apple / Rendering  
Priority: P2

Q: High Metal bandwidth pressure may suggest what UE issues?  
A: Render target format/resolution, GBuffer/post, scene captures or texture sampling.  
Category: Apple / Rendering  
Priority: P2

Q: Expensive Metal pipeline state can point to what UE concept?  
A: Material permutation, shader variant, PSO warm-up or content pattern.  
Category: Apple / Rendering  
Priority: P2

Q: Console profiling exact tools/counters are usually what?  
A: Confidential/platform-holder controlled.  
Category: Console / Source Safety  
Priority: P2

Q: Public console profiling answers should discuss what?  
A: Source-safe workflow and evidence categories, not confidential requirements.  
Category: Console / Source Safety  
Priority: P2

Q: Console profiling packet must include what identity data?  
A: Package/build ID, symbols, scenario, platform/tool versions where allowed and UE evidence.  
Category: Console / Evidence  
Priority: P2

Q: Console GPU issue workflow starts by verifying what?  
A: Package, content version, shader/PSO state, platform config and performance mode.  
Category: Console / GPU  
Priority: P2

Q: Console memory proof should separate what?  
A: Resident/committed pools, graphics memory, transient peaks, fragmentation and UE LLM categories.  
Category: Console / Memory  
Priority: P2

Q: Console certification-adjacent matrix should list what?  
A: Event, expected behavior, observed result, evidence path, owner and remaining risk.  
Category: Console / Certification  
Priority: P2

Q: Exact console certification requirements should come from where?  
A: Authorised platform-holder documentation.  
Category: Console / Certification  
Priority: P2

Q: Platform profiling memo must include what warning section?  
A: What this evidence does not prove.  
Category: Platform Profiling / Communication  
Priority: P2

Q: One device capture should not be generalised across what?  
A: Other devices, GPUs, OS versions, tool versions or content branches.  
Category: Platform Profiling / Evidence  
Priority: P2

Q: Field metrics become actionable when converted into what?  
A: A controlled device-lab run manifest and reproduction scenario.  
Category: Platform Profiling / Device Lab  
Priority: P2

Q: Apple performance guidance uses what improvement loop?  
A: Gather data, measure cause, plan one change, implement it and observe the result.  
Category: Apple / Process  
Priority: P2

Q: Apple device profiling usually has higher fidelity than what?  
A: Simulator profiling.  
Category: Apple / Process  
Priority: P2

Q: Metal shader edits inside the debugger must be copied where?  
A: Back to the real shader/source/content path if the change is accepted.  
Category: Apple / Metal  
Priority: P2

Q: Console raw confidential artifacts should be stored how?  
A: In access-controlled platform-approved locations outside public curriculum files.  
Category: Console / Source Safety  
Priority: P2

## Advanced Gameplay Patterns Specialist

Q: Specialist pattern design starts with what production question?  
A: What the operation guarantees under failure, duplication, stale state and recovery.  
Category: Gameplay Architecture / Patterns  
Priority: P1

Q: Gameplay transaction is not necessarily what?  
A: A database transaction; it is a controlled validate-and-commit gameplay operation.  
Category: Gameplay Architecture / Transactions  
Priority: P1

Q: Transactional inventory move should validate before what?  
A: Mutating source or target containers.  
Category: Gameplay Architecture / Transactions  
Priority: P1

Q: Transaction result should usually carry what?  
A: Operation ID, commit/rejection state, new revisions and deltas.  
Category: Gameplay Architecture / Transactions  
Priority: P1

Q: Operation ID identifies what?  
A: One user or system intention.  
Category: Gameplay Architecture / Observability  
Priority: P1

Q: Revision identifies what?  
A: The state version an operation expected or produced.  
Category: Gameplay Architecture / Revisions  
Priority: P1

Q: Reliable RPC is not the same as what?  
A: Idempotent gameplay operation.  
Category: Networking / Idempotency  
Priority: P1

Q: Idempotent reward grant needs what durable record?  
A: A claim/operation/entitlement record proving it already committed.  
Category: Gameplay Architecture / Idempotency  
Priority: P1

Q: Events are poor durable truth because what?  
A: Late subscribers, save/load and join-in-progress need current state.  
Category: Gameplay Architecture / Events  
Priority: P1

Q: Quest objective "own 10 ore" should query what on activation?  
A: Current inventory state before subscribing to future deltas.  
Category: Gameplay Architecture / Quests  
Priority: P1

Q: Weak event payload often forces listeners to do what?  
A: Re-query mutable publisher state and risk timing/cascade bugs.  
Category: Gameplay Architecture / Events  
Priority: P1

Q: Strong inventory delta event includes what context?  
A: Operation ID, container/item IDs, before/after revisions and old/new quantities.  
Category: Gameplay Architecture / Events  
Priority: P1

Q: Event queue delivery semantics must define what?  
A: Phase, order scope, capacity, stale-target handling, duplicates, threading and tracing.  
Category: Gameplay Architecture / Queues  
Priority: P2

Q: Event queue is a bad fit when the caller needs what?  
A: An immediate authoritative answer or result.  
Category: Gameplay Architecture / Queues  
Priority: P2

Q: Global message bus risk?  
A: Hidden dependencies, unversioned schemas and arbitrary authority boundaries.  
Category: Gameplay Architecture / Messaging  
Priority: P2

Q: Production state machine owns more than names; it owns what?  
A: Enter/exit side effects, tasks/delegates/timers, guards, cancellation and restoration policy.  
Category: Gameplay Architecture / State Machines  
Priority: P1

Q: Reload state might own what side effects?  
A: Montage/task delegates, ammo reservation, interruption, prediction and ammo commit cleanup.  
Category: Gameplay Architecture / State Machines  
Priority: P1

Q: Data Asset should represent what layer?  
A: Shared authored definition data.  
Category: Data-Driven Architecture / Definitions  
Priority: P1

Q: Runtime item state should store what instead of mutating the Data Asset?  
A: Definition ID plus mutable fields like quantity, durability, owner or rolls.  
Category: Data-Driven Architecture / Runtime State  
Priority: P1

Q: Save files should use stable IDs instead of what?  
A: Live UObject or Actor pointers.  
Category: Save Systems / Identity  
Priority: P1

Q: Missing content policy should be what?  
A: Explicit: fail, substitute, refund, hide, preserve opaque record or migrate.  
Category: Save Systems / Content Migration  
Priority: P1

Q: Historical save fixtures prove what?  
A: Old schema/content versions still migrate to valid current state.  
Category: Save Systems / Testing  
Priority: P2

Q: Restore mode avoids what bug?  
A: Replaying normal gameplay side effects such as rewards or achievements during load.  
Category: Save Systems / Restore  
Priority: P1

Q: Save/load restore path should emit what after rebuilding state?  
A: One restore-ready snapshot/event, not every normal gameplay event.  
Category: Save Systems / Restore  
Priority: P1

Q: UI prediction should create what for each pending operation?  
A: Operation ID and expected state revision.  
Category: UI / Prediction  
Priority: P1

Q: UI should reconcile pending prediction from what?  
A: Authoritative commit/rejection deltas and revisions.  
Category: UI / Prediction  
Priority: P1

Q: Widget state should be rebuildable from what?  
A: Authoritative model/view-model state.  
Category: UI / Projection  
Priority: P1

Q: Stale UI response is detected using what?  
A: Operation ID and revision comparison.  
Category: UI / Reconciliation  
Priority: P1

Q: Duplicate item debugging starts from what?  
A: Operation log, revisions, validation and committed deltas, not screenshots.  
Category: Gameplay Architecture / Debugging  
Priority: P1

Q: Partial mutation bug often violates what transaction rule?  
A: Validate everything before mutating committed state.  
Category: Gameplay Architecture / Transactions  
Priority: P1

Q: Quest missed progress often means what was missing?  
A: Initial durable-state query before event subscription.  
Category: Gameplay Architecture / Quests  
Priority: P1

Q: Save replayed reward often means what path was used incorrectly?  
A: Normal gameplay completion path instead of restore path.  
Category: Save Systems / Debugging  
Priority: P1

Q: Pattern profiling should measure delegate what?  
A: Listener count, broadcast frequency and cascade duration.  
Category: Gameplay Architecture / Profiling  
Priority: P2

Q: Pattern profiling should measure queue what?  
A: Depth, dropped/coalesced messages and processing time.  
Category: Gameplay Architecture / Profiling  
Priority: P2

Q: Pattern profiling should measure transaction what?  
A: Operation count, validation cost, commit latency and allocation churn.  
Category: Gameplay Architecture / Profiling  
Priority: P2

Q: Dirty-cache optimisation risk?  
A: Missed invalidation or repeated recomputation at the wrong phase.  
Category: Gameplay Architecture / Dirty Flag  
Priority: P2

Q: Double buffer costs include what?  
A: Extra memory, copy/swap semantics, one-step latency and reference ownership concerns.  
Category: Gameplay Architecture / Double Buffer  
Priority: P2

Q: Flyweight in inventory usually means what?  
A: Shared item definition plus per-instance/stack extrinsic state.  
Category: Gameplay Architecture / Flyweight  
Priority: P1

Q: Type Object can replace what when variations are data-driven?  
A: Large numbers of shallow subclasses.  
Category: Gameplay Architecture / Type Object  
Priority: P2

Q: Service retrieval should often happen where?  
A: At composition/orchestration boundaries, passing narrow interfaces into core logic.  
Category: Gameplay Architecture / Subsystems  
Priority: P1

Q: UI should not own authoritative inventory because why?  
A: Widgets are disposable projections and can be stale, recycled or local-only.  
Category: UI / Architecture  
Priority: P1

Q: Transaction logs should include before and after what?  
A: State revisions.  
Category: Gameplay Architecture / Observability  
Priority: P1

Q: Operation reject result should include what?  
A: A structured rejection reason.  
Category: Gameplay Architecture / Transactions  
Priority: P1

Q: Event sourcing should be used only when what is true?  
A: Replaying historical events is an explicit product architecture.  
Category: Gameplay Architecture / Events  
Priority: P2

Q: Save migration fixtures should be added when what changes?  
A: Designer-facing identity, objective IDs, schema or content meaning changes.  
Category: Save Systems / Migration  
Priority: P2

Q: Advanced architecture debugging traces what first?  
A: Data flow and operation state, not class diagrams.  
Category: Gameplay Architecture / Debugging  
Priority: P1

## Broad Specialist Scenario Flashcards

Q: Why is an animation notify a weak sole authority for melee damage?  
A: It depends on animation playback, montage section timing and local pose state; authoritative damage should be validated by gameplay state on the server.  
Category: Animation / Gameplay Authority  
Priority: P1

Q: What should a melee damage notify usually trigger instead of directly applying final damage?  
A: A gameplay request or window that the authority validates against attack state, timing, target data and cooldown rules.  
Category: Animation / Gameplay Authority  
Priority: P1

Q: What dedicated-server problem exposes animation-driven damage bugs?  
A: Servers may not evaluate the same animation graph or notify timing as clients.  
Category: Animation / Networking  
Priority: P1

Q: What evidence separates animation timing bugs from authority bugs in melee combat?  
A: Logs for attack state transitions, montage sections, notify receipt, server validation and damage commit.  
Category: Animation / Debugging  
Priority: P1

Q: Foot sliding in a networked character should be debugged by comparing which truths?  
A: Movement velocity/root motion authority, replicated movement corrections, pose selection and ground contact assumptions.  
Category: Animation / Locomotion  
Priority: P1

Q: Why can root motion create networked foot sliding?  
A: It changes who owns displacement and can conflict with movement component correction if authority and prediction are not aligned.  
Category: Animation / Root Motion  
Priority: P1

Q: What should locomotion debug overlays show for foot sliding?  
A: Desired velocity, actual velocity, acceleration, gait state, root-motion contribution and animation sample/phase.  
Category: Animation / Debugging  
Priority: P1

Q: Why does Block collision not guarantee gameplay hit logic ran?  
A: Blocking controls movement resolution; hit/overlap events also depend on sweep/simulation path, event flags, components and channel setup.  
Category: Physics / Collision  
Priority: P1

Q: What is the first collision question when a projectile passes through a target?  
A: Which component moved, how it moved, and whether that move produced a sweep or physics contact event.  
Category: Physics / Debugging  
Priority: P1

Q: What is a common mismatch between visual collision and gameplay collision?  
A: The visible mesh is not the component generating query or physics events.  
Category: Physics / Collision  
Priority: P1

Q: Why are kinematic movement and simulated physics dangerous on the same object?  
A: They create competing writers for transform and velocity, producing jitter, missed contacts or unstable authority.  
Category: Physics / Movement  
Priority: P1

Q: What should a physics-heavy gameplay feature define explicitly?  
A: Whether gameplay listens to query results, sweep hits, physics contacts or manually validated spatial tests.  
Category: Physics / System Design  
Priority: P1

Q: In split-screen, UI focus should be scoped to what?  
A: The relevant local player and input routing context.  
Category: UI / CommonUI  
Priority: P1

Q: Why can one player's controller navigate another player's menu?  
A: Focus or action routing is shared globally instead of being bound to the correct local player.  
Category: UI / Split Screen  
Priority: P1

Q: What should a CommonUI focus debug packet include?  
A: Local player, active input mode, activatable widget stack, focused widget and triggering input action.  
Category: UI / Debugging  
Priority: P1

Q: Why can Retainer Panels worsen UI performance?  
A: They add render target memory, capture/composite work and latency, so they only help when redraw reduction outweighs those costs.  
Category: UI / Performance  
Priority: P1

Q: When does invalidation help UMG performance most?  
A: When large unchanged subtrees can reuse cached layout/paint results across frames.  
Category: UI / Performance  
Priority: P1

Q: What invalidates a cached UI subtree unexpectedly?  
A: Volatile properties, frequent bindings, layout changes, dynamic materials or animation touching cached widgets.  
Category: UI / Performance  
Priority: P1

Q: What is the first audio debug split for an inaudible sound?  
A: Whether the sound request happened, a component/voice was created, and playback survived attenuation, concurrency and routing.  
Category: Audio / Debugging  
Priority: P1

Q: Why is audio concurrency a gameplay risk?  
A: A critical cue can be silently denied or evicted if it competes with less important sounds under the same policy.  
Category: Audio / Concurrency  
Priority: P1

Q: What should a critical gameplay warning use in addition to sound?  
A: Redundant visual, haptic or UI feedback that does not depend on audio voice allocation.  
Category: Audio / UX  
Priority: P1

Q: What audio debug evidence distinguishes request failure from routing failure?  
A: Request logs, audio component state, active voice data, concurrency result, sound class/submix routing and output device.  
Category: Audio / Debugging  
Priority: P1

Q: Why can a Niagara effect disappear even though the emitter still exists?  
A: Bounds, scalability, significance or culling policy can stop rendering while the system object remains alive.  
Category: VFX / Niagara  
Priority: P1

Q: What should be checked first when Niagara disappears only off-center or at distance?  
A: Fixed/dynamic bounds, camera culling, effect type scalability and significance settings.  
Category: VFX / Debugging  
Priority: P1

Q: Why should Niagara Data Channels not be durable gameplay truth?  
A: They are a high-throughput data path for effects, not an authoritative replicated persistence model.  
Category: VFX / Data Channels  
Priority: P1

Q: What is a safe pattern for gameplay-driven Niagara?  
A: Gameplay owns truth and publishes a bounded presentation stream to Niagara.  
Category: VFX / Architecture  
Priority: P1

Q: How do you compute a signed angle robustly?  
A: Use clamped dot for magnitude and cross product projected onto the chosen axis for sign.  
Category: Game Math / Vectors  
Priority: P1

Q: Why should dot-product inputs be clamped before `acos`?  
A: Floating-point error can push values slightly outside [-1, 1], producing NaN.  
Category: Game Math / Numerical Safety  
Priority: P1

Q: Why do quaternions have a shortest-path interpolation trap?  
A: `q` and `-q` represent the same rotation; interpolation should usually flip one quaternion when the dot product is negative.  
Category: Game Math / Quaternions  
Priority: P1

Q: What problem does normalizing quaternions after accumulation prevent?  
A: Drift away from unit length, which can introduce scaling-like artifacts and unstable interpolation.  
Category: Game Math / Quaternions  
Priority: P1

Q: What property must an A* heuristic have for optimality?  
A: It must not overestimate the remaining path cost.  
Category: Algorithms / Pathfinding  
Priority: P1

Q: What does Weighted A* trade for speed?  
A: It may return suboptimal paths because the heuristic is intentionally overweighted.  
Category: Algorithms / Pathfinding  
Priority: P2

Q: Why is union-find a poor fit for dynamic connectivity with deletions?  
A: It efficiently merges sets but does not naturally split them when edges are removed.  
Category: Algorithms / Data Structures  
Priority: P1

Q: What game system often tempts misuse of union-find?  
A: Territory, island, graph or room connectivity that needs frequent edge deletion.  
Category: Algorithms / Data Structures  
Priority: P2

Q: What is false sharing?  
A: Independent data on the same cache line being written by different cores, causing cache-line ping-pong.  
Category: C++ / Performance  
Priority: P1

Q: Why can padding improve multithreaded performance?  
A: It separates hot per-thread counters or queues onto different cache lines.  
Category: C++ / Cache  
Priority: P1

Q: What is the main lifetime rule for `std::pmr::monotonic_buffer_resource`?  
A: Individual allocations are not freed; memory is released in bulk when the resource is released or destroyed.  
Category: C++ / Memory Resources  
Priority: P1

Q: Why is a monotonic resource risky for a long-lived world subsystem?  
A: Repeated transient allocations can accumulate until the whole resource is reset.  
Category: C++ / Memory Resources  
Priority: P1

Q: What is the safe rule for arbitrary async tasks and UObjects?  
A: Do not freely touch UObjects off the game thread; snapshot data and return to the game thread for validation and mutation.  
Category: UE C++ / Async  
Priority: P1

Q: Why is a weak object pointer not enough for off-thread UObject use?  
A: Validity can change and many UObject APIs are not thread-safe even if the pointer is non-null.  
Category: UE C++ / Threading  
Priority: P1

Q: What should async task completion validate before mutating gameplay state?  
A: Object lifetime, world/session identity, request ID and current state revision.  
Category: UE C++ / Async  
Priority: P1

Q: Why can Lua hot reload create stale behavior?  
A: Existing closures, tables, bound delegates or userdata wrappers may still reference old code or assumptions.  
Category: Lua / Hot Reload  
Priority: P1

Q: What should a Lua hotfix boundary define?  
A: Which state is migrated, which delegates are rebound, and which objects require restart or reconstruction.  
Category: Lua / Tooling  
Priority: P1

Q: What makes C# Native AOT/trimming risky for scripting plugins?  
A: Reflection-discovered types or generated bindings may be removed or unavailable unless rooted explicitly.  
Category: C# / Runtime  
Priority: P2

Q: What should an Unreal C# bridge generate rather than discover dynamically in shipping builds?  
A: Explicit binding metadata for reflected Unreal types, methods and marshalled value shapes.  
Category: C# / Unreal Integration  
Priority: P2

Q: Why should Runtime Data Layers not be the sole source of quest truth?  
A: They control world/content streaming state, while quest truth must persist independently across save/load and streaming changes.  
Category: World Building / Data Layers  
Priority: P1

Q: What should quest logic store instead of only relying on streamed actor presence?  
A: Durable objective state and stable content identifiers.  
Category: World Building / Quests  
Priority: P1

Q: What PCG runtime ownership question must be answered first?  
A: Whether generated content is deterministic presentation, saved state, replicated gameplay state or editor-authored content.  
Category: PCG / Architecture  
Priority: P1

Q: Why can PCG-generated gameplay actors cause save bugs?  
A: Their identity, lifetime and regeneration policy may be unclear across reloads or world partition streaming.  
Category: PCG / Save Systems  
Priority: P1

Q: What should PCG-generated replicated content define?  
A: Authority, stable identity, replication relevance, cleanup and conflict behavior when generation changes.  
Category: PCG / Networking  
Priority: P1

Q: What is a device-lab failure-state test?  
A: A test that deliberately exercises cable disconnects, provisioning expiry, lost symbols, stale packages and capture tool failure.  
Category: Platform / Test Infrastructure  
Priority: P2

Q: Why is clean-agent build failure valuable?  
A: It exposes hidden local dependencies, untracked files, undeclared tools and environment assumptions.  
Category: Build Systems / CI  
Priority: P1

Q: What should a clean build machine prove?  
A: The repository, declared dependencies and documented setup are sufficient to build, cook/package and run tests.  
Category: Build Systems / Reliability  
Priority: P1

Q: What build evidence is stronger than "works on my machine"?  
A: A clean-agent log with exact tool versions, dependency restoration, build/cook/test commands and artifacts.  
Category: Build Systems / Debugging  
Priority: P1

Q: What separates a gameplay event from durable state?  
A: Events describe something that happened; durable state describes what remains true after loading, streaming and reconnection.  
Category: Gameplay Architecture / State  
Priority: P1

Q: Why should UI commands include operation IDs?  
A: They let the system reconcile accept/reject results and avoid applying stale responses to the wrong local prediction.  
Category: UI / Prediction  
Priority: P1

Q: What is the common bug when gameplay listens directly to presentation callbacks?  
A: Gameplay becomes dependent on optional, skipped, culled or locally different presentation behavior.  
Category: Gameplay Architecture / Presentation Boundary  
Priority: P1

Q: What should a specialist answer include after naming a likely root cause?  
A: Evidence needed to prove it and a minimal fix that preserves authority and ownership boundaries.  
Category: Interviewing / Answer Pattern  
Priority: P1

Q: What is the fastest way to weaken a debugging interview answer?  
A: Jumping to a fix before defining which subsystem owns truth and which evidence would distinguish hypotheses.  
Category: Interviewing / Debugging  
Priority: P1

Q: What should a performance answer always separate?  
A: CPU, GPU, memory, IO, threading and tool overhead before recommending changes.  
Category: Interviewing / Performance  
Priority: P1

Q: What is the specialist pattern for cross-system bugs?  
A: Trace authoritative state, presentation state, async boundaries, replication/persistence boundaries and tooling evidence separately.  
Category: Interviewing / Systems  
Priority: P1

Q: Why should source control ignore generated local artifacts?  
A: They make clean builds non-reproducible and hide missing generation steps.  
Category: Build Systems / Source Control  
Priority: P2

Q: What should content cook failures record?  
A: Asset path, platform, target configuration, cooker log excerpt, dependency chain and whether the asset is editor-only.  
Category: Asset Pipeline / Cook Debugging  
Priority: P1

Q: What does "editor works, packaged build fails" usually suggest?  
A: Editor-only dependency, missing cook reference, platform-specific asset setting, plugin packaging issue or config difference.  
Category: Build Systems / Packaging  
Priority: P1

Q: What makes an interview sample code answer stronger?  
A: Naming authority, lifetime, threading and failure semantics around the code, not just writing the happy path.  
Category: Interviewing / Code  
Priority: P1

Q: What should a gameplay subsystem expose to UI?  
A: Narrow query/command interfaces or view models, not mutable internal collections.  
Category: Gameplay Architecture / UI Boundary  
Priority: P1

Q: Why is a "manager" class suspicious in gameplay architecture?  
A: It often hides ownership, lifecycle, authority and dependency boundaries unless its contract is explicit.  
Category: Gameplay Architecture / Design  
Priority: P2

Q: What is a good replacement question for "where should this code go?"  
A: Which system owns the truth, lifetime, authority, persistence and observation contract?  
Category: Gameplay Architecture / Design  
Priority: P1

Q: Why should gameplay systems emit structured rejection reasons?  
A: They make UI reconciliation, telemetry, QA reproduction and exploit analysis possible.  
Category: Gameplay Architecture / Debugging  
Priority: P1

Q: What should a networked feature log for each player-facing command?  
A: Command ID, input time, client prediction state, server receipt, validation result and authoritative revision.  
Category: Networking / Observability  
Priority: P1

Q: Why is "replicate everything" not a design?  
A: Replication must choose owner, relevance, frequency, prediction needs and security validation per state category.  
Category: Networking / System Design  
Priority: P1

Q: What should be validated for client-provided target data?  
A: Range, line of sight, timing, team rules, cooldown/resource state and plausible movement history.  
Category: Gameplay / Validation  
Priority: P1

Q: Why can replay or demo systems expose hidden architecture bugs?  
A: They require deterministic enough event/state reconstruction and make local-only side effects visible.  
Category: Gameplay Architecture / Replay  
Priority: P2

Q: What should an ability-like custom system borrow from GAS design?  
A: Explicit activation state, effect ownership, prediction/rejection paths, tag-like state queries and cue separation.  
Category: Gameplay Systems / Design  
Priority: P1

Q: What should a save-game schema change include?  
A: Versioning, migration fixtures, old-save tests and a rollback plan for bad content identifiers.  
Category: Save Systems / Migration  
Priority: P1

Q: What is the risk of using display names as persistent IDs?  
A: Renames, localization and duplicate labels can break save/load, analytics or quest references.  
Category: Data Design / Identity  
Priority: P1

Q: What should designer-authored data validation catch before runtime?  
A: Missing IDs, duplicate IDs, invalid references, impossible requirements and platform-incompatible asset choices.  
Category: Tools / Validation  
Priority: P1

Q: Why should debugging tools expose stable IDs?  
A: Stable IDs let engineers connect logs, saves, replicated messages, data assets and editor selections.  
Category: Tools / Debugging  
Priority: P1

Q: What is a source-safe console profiling answer allowed to discuss?  
A: General evidence categories and methodology, not confidential devkit counters or platform NDA details.  
Category: Platform / Profiling  
Priority: P1

## NetworkPrediction Rollback Proof Flashcards

Q: Does NetworkPrediction mean automatic full-game rollback?  
A: No. It provides plugin/runtime vocabulary and services; each feature still needs a bounded replay-safe model and target-source proof.  
Category: Networking / NetworkPrediction  
Priority: P1

Q: What does the public NetworkPrediction API page prove?  
A: Plugin/module vocabulary, component/model surface, rollback/interpolation/smoothing services, state/cue/buffer names and debug CVar vocabulary.  
Category: Networking / NetworkPrediction  
Priority: P1

Q: What does the public NetworkPrediction API page not prove?  
A: A ready-to-copy UE5.3-UE5.6 implementation recipe or exact target-branch signatures.  
Category: Networking / Versioning  
Priority: P1

Q: Full rollback restores what?  
A: A bounded simulation state and resimulates forward from stored inputs/aux state.  
Category: Networking / Rollback  
Priority: P1

Q: Server rewind differs from rollback how?  
A: Rewind queries historical state for validation; rollback restores simulation state and resimulates.  
Category: Networking / Lag Compensation  
Priority: P1

Q: CharacterMovement prediction differs from full rollback how?  
A: It is a specialised movement prediction/correction/replay pipeline, not generic full-game rollback.  
Category: Networking / CharacterMovement  
Priority: P1

Q: Rollback input command should contain what?  
A: Compact replayable intent such as quantised axes, action bits, frame/sequence and small validated parameters.  
Category: Networking / Input Commands  
Priority: P1

Q: Rollback input command should not contain what?  
A: Trusted hit results, direct world pointers, random outputs, animation state or large authority decisions.  
Category: Networking / Input Commands  
Priority: P1

Q: Sync state is what?  
A: Frame-to-frame simulation truth that clients compare against authority and replay from.  
Category: Networking / Sync State  
Priority: P1

Q: Example sync fields for rollback dash?  
A: Location, velocity, movement mode/phase, dash ticks remaining and state frame.  
Category: Networking / Sync State  
Priority: P1

Q: Aux state is what?  
A: Slower-changing simulation input such as approved tunables, equipment modifiers, tag/resource snapshots or table versions.  
Category: Networking / Aux State  
Priority: P1

Q: Why version aux state?  
A: Old frames must replay with the correct old tunings, or pending prediction must be rejected.  
Category: Networking / Aux State  
Priority: P1

Q: Rollback tick should behave like what?  
A: A pure-enough function of previous sync state, aux state, input command and time step.  
Category: Networking / Simulation  
Priority: P1

Q: Rollback tick should avoid reading what?  
A: Wall-clock time, current Actor state, uncontrolled random values and mutable presentation state.  
Category: Networking / Simulation  
Priority: P1

Q: Why are direct side effects dangerous in rollback tick?  
A: Prediction and replay can execute the tick multiple times, duplicating effects.  
Category: Networking / Side Effects  
Priority: P1

Q: Replay-safe presentation should use what?  
A: Cue/event keys and dedupe semantics matched to the desired prediction/replication behaviour.  
Category: Networking / Cues  
Priority: P1

Q: Irreversible rollback side effects should live where?  
A: Outside the resimulated tick, usually in one-time authoritative commit paths.  
Category: Networking / Authority  
Priority: P1

Q: Damage inside a replayed tick risks what?  
A: Duplicate or divergent damage under prediction and correction.  
Category: Networking / Side Effects  
Priority: P1

Q: Item rewards inside a replayed tick risk what?  
A: Duplicate grants or inconsistent durable state.  
Category: Networking / Side Effects  
Priority: P1

Q: Cue key should usually include what?  
A: Model instance, simulation frame, cue type and relevant payload identity/hash.  
Category: Networking / Cues  
Priority: P1

Q: Reconcile proof starts by forcing what?  
A: Controlled divergence between client prediction and server authority.  
Category: Networking / Debugging  
Priority: P1

Q: Useful reconcile log includes what frame data?  
A: authority frame, restored frame, replay window and post-replay state hash.  
Category: Networking / Observability  
Priority: P1

Q: Useful reconcile log includes what state data?  
A: field diffs, input hash, aux version, sync hashes and cue suppression list.  
Category: Networking / Observability  
Priority: P1

Q: Reconcile every frame usually means what?  
A: Client and server are not simulating the same model or tolerance/state comparison is wrong.  
Category: Networking / Debugging  
Priority: P1

Q: Common causes of reconcile every frame?  
A: tick mismatch, nondeterministic branch, missing sync field, aux mismatch, strict tolerance or unreproduced collision.  
Category: Networking / Debugging  
Priority: P1

Q: Why disable smoothing during rollback debugging?  
A: Smoothing can hide frequent corrections and make a broken model feel acceptable.  
Category: Networking / Debugging  
Priority: P1

Q: Rollback smoothing proves what?  
A: UX quality after correction, not simulation correctness.  
Category: Networking / Smoothing  
Priority: P1

Q: Rollback buffer length depends on what?  
A: RTT/jitter/loss window, max correction delay, replay cost, memory budget and exploit tolerance.  
Category: Networking / Performance  
Priority: P1

Q: Rollback memory budget includes what buffers?  
A: input, sync, aux, cue history/dedupe and per-instance overhead.  
Category: Networking / Performance  
Priority: P1

Q: Rollback CPU budget includes what?  
A: model tick cost, replay frames per correction, collision/query cost and correction-storm bursts.  
Category: Networking / Performance  
Priority: P1

Q: Rollback network budget includes what?  
A: input command size/rate, correction payloads and cue replication payloads.  
Category: Networking / Performance  
Priority: P1

Q: A good first rollback proof feature is what?  
A: A compact deterministic micro-simulation such as dash or charge movement.  
Category: Networking / Scope  
Priority: P1

Q: A poor first rollback proof feature is what?  
A: Large nondeterministic systems such as ragdoll combat, inventory transactions or whole-game state.  
Category: Networking / Scope  
Priority: P1

Q: Rollback is weak for inventory because why?  
A: Inventory is a durable authority transaction, not a responsiveness-critical replayed simulation.  
Category: Networking / Architecture Choice  
Priority: P1

Q: Rollback is risky for ragdoll physics because why?  
A: Physics state is large and difficult to reproduce deterministically across replay windows.  
Category: Networking / Architecture Choice  
Priority: P2

Q: Actor bridge should separate what three state categories?  
A: model-owned replay state, Actor durable replicated state and presentation state.  
Category: Networking / Actor Bridge  
Priority: P1

Q: One writer per field prevents what?  
A: Actor replication and rollback sync state fighting each other.  
Category: Networking / Actor Bridge  
Priority: P1

Q: Respawn is what kind of rollback boundary?  
A: A generation/lifetime boundary that invalidates old inputs, model IDs, targets and cue histories.  
Category: Networking / Lifetime  
Priority: P1

Q: Late joiner should reconstruct rollback feature from what?  
A: Durable authoritative state, not historical predicted cues.  
Category: Networking / Late Join  
Priority: P1

Q: Client-provided target data is what?  
A: Intent/evidence that authority validates, not final truth.  
Category: Networking / Security  
Priority: P1

Q: Server validates rollback action target data against what?  
A: ownership, cooldown, resource, range, line of sight, timing, target generation and plausible movement.  
Category: Networking / Security  
Priority: P1

Q: Target generation prevents what exploit/bug?  
A: Applying stale predicted input to a respawned or different target.  
Category: Networking / Security  
Priority: P1

Q: Aux-state bug during equipment swap?  
A: Old predicted frames replay using new equipment tunings.  
Category: Networking / Aux State  
Priority: P1

Q: Fix for equipment-swap replay bug?  
A: Version/snapshot aux state or reject pending inputs across the version boundary.  
Category: Networking / Aux State  
Priority: P1

Q: A cue should not own what?  
A: Durable gameplay mutation such as damage, rewards, score or achievements.  
Category: Networking / Cues  
Priority: P1

Q: Rollback proof acceptance criterion for cues?  
A: Forced reconcile does not duplicate replay-safe VFX/audio and irreversible effects happen once.  
Category: Networking / Cues  
Priority: P1

Q: Rollback proof acceptance criterion for state?  
A: Client converges to authoritative state within documented tolerances after correction.  
Category: Networking / Rollback  
Priority: P1

Q: Rollback proof acceptance criterion for logs?  
A: Logs identify frame, input hash, aux version, field diffs and replay window.  
Category: Networking / Observability  
Priority: P1

Q: Rollback proof acceptance criterion for networking?  
A: Runs under dedicated-server packet lag/loss with measured correction and traffic behaviour.  
Category: Networking / Testing  
Priority: P1

Q: Rollback proof acceptance criterion for production choice?  
A: Compares plugin rollback against simpler alternatives and justifies the smallest sufficient architecture.  
Category: Networking / Architecture Choice  
Priority: P1

Q: What should target-branch NetworkPrediction proof record first?  
A: Engine version, plugin status, headers/source locations, exact APIs and available debug CVars.  
Category: Networking / Version Proof  
Priority: P1

Q: Why are maintained API pages not enough for NetworkPrediction code?  
A: The public page can resolve to later versions and does not prove target UE5.3-UE5.6 signatures or service wiring.  
Category: Networking / Version Proof  
Priority: P1

Q: Fixed-step alone gives what?  
A: A better time base, but not full determinism or side-effect safety by itself.  
Category: Networking / Determinism  
Priority: P1

Q: Determinism hazards beyond physics?  
A: randomness, data versions, current Actor reads, query order, floating point tolerances and side effects.  
Category: Networking / Determinism  
Priority: P1

Q: Rollback fit question before implementation?  
A: Is the responsiveness benefit worth the state isolation, buffers, replay cost and tooling burden?  
Category: Networking / Architecture Choice  
Priority: P1

Q: Server-authoritative commands are often better for what?  
A: inventory, quests, rewards, score and other durable transactions.  
Category: Networking / Architecture Choice  
Priority: P1

Q: CharacterMovement saved moves are often better for what?  
A: movement-affecting character flags that fit the existing movement prediction pipeline.  
Category: Networking / Architecture Choice  
Priority: P1

Q: Server rewind is often better for what?  
A: validating timestamped hitscan or hitbox queries without resimulating the shooter model.  
Category: Networking / Architecture Choice  
Priority: P1

Q: Rollback compare packet should include what alternatives?  
A: server command, CharacterMovement saved move, server rewind, plugin rollback and lightweight custom prediction.  
Category: Networking / Architecture Choice  
Priority: P2

Q: Correction storm metric?  
A: Replay frames and CPU time per correction multiplied by correction frequency.  
Category: Networking / Performance  
Priority: P1

Q: Field-level sync diffs are better than what?  
A: A generic "state mismatch" log.  
Category: Networking / Debugging  
Priority: P1

Q: What proves rollback rejected an invalid action cleanly?  
A: Structured server reject reason, client state convergence and no permanent ghost presentation.  
Category: Networking / Rejection  
Priority: P1

Q: Why should local hit markers be keyed?  
A: They may need dedupe or cancellation when authority later rejects the predicted action.  
Category: Networking / Presentation  
Priority: P1

Q: What is rollback's strongest interview answer pattern?  
A: Define state/input/authority/side effects, then describe forced reconcile evidence and measured costs.  
Category: Interviewing / Networking  
Priority: P1

Q: What is rollback's weakest interview answer pattern?  
A: "Enable the plugin and it handles it."  
Category: Interviewing / Networking  
Priority: P1

Q: What does "bounded model" mean for rollback?  
A: A clearly scoped subset of gameplay state with defined inputs, outputs, replay window and ownership.  
Category: Networking / Rollback  
Priority: P1

Q: What is the difference between prediction and authority?  
A: Prediction is local speculation for responsiveness; authority is the server-owned decision that confirms or corrects it.  
Category: Networking / Authority  
Priority: P1

Q: What should rollback do after authority rejection?  
A: Restore/correct state, replay valid pending input and clean or reconcile presentation side effects.  
Category: Networking / Reconciliation  
Priority: P1

Q: Why should rollback tests use packet emulation?  
A: Zero-latency tests do not expose delayed corrections, replay windows, cue duplication or stale prediction.  
Category: Networking / Testing  
Priority: P1

Q: What does a rollback ship/no-ship memo include?  
A: source proof, forced reconcile logs, side-effect proof, packet tests, budgets, platform risk and alternative comparison.  
Category: Networking / Production Readiness  
Priority: P1

## Core Systems Rapid Review Expansion

Q: Does `UPROPERTY` automatically replicate a field?  
A: No. Replication also needs replicated property setup, lifetime registration and authority-owned mutation.  
Category: UE C++ / Reflection  
Priority: P1

Q: Reflection exposure and editor visibility are what kind of systems?  
A: Related but separate systems controlled by different specifiers and tooling.  
Category: UE C++ / Reflection  
Priority: P1

Q: UHT primarily generates what?  
A: Reflection/glue metadata needed by Unreal's object, Blueprint and generated-code systems.  
Category: UE C++ / UHT  
Priority: P1

Q: Why can generated headers fail after a C++ rename?  
A: UHT, include order, stale generated code or reflected declaration mismatch can break the build.  
Category: UE C++ / UHT  
Priority: P2

Q: `Outer` means what?  
A: UObject containment/path/lifetime context, not necessarily gameplay owner or scene parent.  
Category: UObject / Lifetime  
Priority: P1

Q: Actor owner controls what important network concept?  
A: Owning connection and RPC routing/ownership semantics.  
Category: Networking / Ownership  
Priority: P1

Q: Attachment controls what?  
A: Scene transform hierarchy, not UObject containment or network ownership by itself.  
Category: Actor / Attachment  
Priority: P1

Q: Strong reflected UObject pointer helps GC by doing what?  
A: Making the reference visible to Unreal's garbage collector.  
Category: UObject / GC  
Priority: P1

Q: `TWeakObjectPtr` is useful for what?  
A: Referencing UObjects without keeping them alive and detecting invalidation.  
Category: UObject / Pointers  
Priority: P1

Q: Why can a valid weak pointer still be unsafe off-thread?  
A: UObject APIs and lifetime semantics are generally game-thread constrained.  
Category: UObject / Threading  
Priority: P1

Q: Pending async callback should validate what before using an Actor?  
A: Weak pointer, generation/request ID, world/session and current state.  
Category: UE C++ / Async  
Priority: P1

Q: `CreateDefaultSubobject` is normally used where?  
A: In constructors for class default subobjects/component composition.  
Category: UObject / Creation  
Priority: P1

Q: `NewObject` is used for what?  
A: Runtime UObject creation with explicit Outer/lifetime/reference ownership.  
Category: UObject / Creation  
Priority: P1

Q: `SpawnActor` is used when the object needs what?  
A: World Actor lifecycle, transform, ticking, networking or level ownership.  
Category: Actor / Creation  
Priority: P1

Q: The CDO represents what?  
A: The class default object holding default property values for a UObject class.  
Category: UObject / CDO  
Priority: P1

Q: Mutating CDO-like defaults at runtime risks what?  
A: Unexpected shared defaults, editor/runtime contamination or instance initialisation bugs.  
Category: UObject / CDO  
Priority: P2

Q: GameMode exists where?  
A: On the server only.  
Category: Gameplay Framework / GameMode  
Priority: P1

Q: Shared match state for clients belongs where?  
A: GameState or other replicated state, not GameMode.  
Category: Gameplay Framework / GameState  
Priority: P1

Q: Per-player public persistent state belongs where?  
A: PlayerState.  
Category: Gameplay Framework / PlayerState  
Priority: P1

Q: Local private input/UI state usually belongs near what?  
A: Local PlayerController, LocalPlayer subsystem, HUD/widget model or local presenter.  
Category: Gameplay Framework / Local State  
Priority: P1

Q: BeginPlay should not assume what?  
A: Every dependency, replicated reference, streamed Actor or async asset is ready.  
Category: Actor Lifecycle / BeginPlay  
Priority: P1

Q: Readiness handshake solves what lifecycle problem?  
A: Dependencies arriving after BeginPlay or in different streaming/replication order.  
Category: Actor Lifecycle / Readiness  
Priority: P1

Q: Subsystem scope should match what?  
A: Data lifetime and audience: engine, game instance, world or local player.  
Category: Gameplay Architecture / Subsystems  
Priority: P1

Q: A subsystem is suspicious when it hides what?  
A: Ownership, authority, lifetime, scope or dependencies.  
Category: Gameplay Architecture / Subsystems  
Priority: P2

Q: BlueprintNativeEvent provides what advantage?  
A: A C++ default implementation with Blueprint customisation.  
Category: C++ / Blueprint  
Priority: P1

Q: BlueprintImplementableEvent is safest for what?  
A: Optional extension/presentation hooks rather than mandatory core invariants.  
Category: C++ / Blueprint  
Priority: P1

Q: Blueprint pure nodes must be what?  
A: Cheap, deterministic and side-effect-free.  
Category: Blueprint / API Design  
Priority: P1

Q: Why can pure Blueprint nodes be expensive?  
A: They can be evaluated repeatedly by graph dependencies.  
Category: Blueprint / Performance  
Priority: P1

Q: Direct BlueprintReadWrite arrays risk what?  
A: Broken invariants, stale UI state, replication dirty-mark bugs and uncontrolled mutation.  
Category: Blueprint / API Design  
Priority: P1

Q: Safer Blueprint collection API exposes what?  
A: Snapshots, validated commands, stable IDs, revisions and change events.  
Category: Blueprint / API Design  
Priority: P1

Q: Value semantics are best for what kind of gameplay data?  
A: Small deterministic data without engine identity or UObject lifetime needs.  
Category: C++ / Design  
Priority: P1

Q: UObject references are justified by what needs?  
A: Engine identity, reflection, GC, editor assets, polymorphism or UObject integration.  
Category: UObject / Design  
Priority: P1

Q: `std::move` does what exactly?  
A: Casts an expression to an rvalue; the type's move operation decides the effect.  
Category: Standard C++ / Move  
Priority: P1

Q: Moved-from objects are guaranteed to be what?  
A: Valid but not necessarily meaningful.  
Category: Standard C++ / Move  
Priority: P1

Q: `std::function` can cost what?  
A: Type-erasure overhead, capture copies and possible allocation.  
Category: C++ / Callables  
Priority: P2

Q: Hot callbacks often prefer what over owning type erasure?  
A: Templates, direct calls, function pointers or non-owning function refs when lifetime is clear.  
Category: C++ / Callables  
Priority: P2

Q: Release/acquire atomics establish what?  
A: A happens-before relationship for publishing writes to an acquiring reader.  
Category: C++ / Atomics  
Priority: P2

Q: Atomics do not replace what?  
A: Mutexes/invariant protection when multiple fields must change consistently.  
Category: C++ / Concurrency  
Priority: P1

Q: False sharing happens when what?  
A: Different threads write independent data that shares one cache line.  
Category: C++ / Cache  
Priority: P1

Q: `std::pmr::monotonic_buffer_resource` frees memory how?  
A: In bulk, not per allocation.  
Category: C++ / Memory  
Priority: P1

Q: ODR bugs often appear as what?  
A: Duplicate symbol link errors, unresolved externals or undefined behaviour from inconsistent definitions.  
Category: C++ / Linkage  
Priority: P2

Q: Unreal module dependency bugs often need what evidence?  
A: Build.cs dependencies, include ownership, export macros and linker output.  
Category: Build / Modules  
Priority: P1

Q: `TArray` is strongest for what?  
A: Compact storage, iteration and small-N linear operations.  
Category: UE Containers / TArray  
Priority: P1

Q: Hash containers trade lookup for what costs?  
A: Memory, hashing, poorer locality and more complex invalidation behaviour.  
Category: Containers / Hashing  
Priority: P1

Q: `FText` is for what?  
A: Localised display text.  
Category: UE Types / FText  
Priority: P1

Q: Gameplay identity should not use what?  
A: Localised display text.  
Category: Data Design / Identity  
Priority: P1

Q: Data Assets should usually store what?  
A: Shared designer-authored definitions, not mutable per-instance runtime state.  
Category: Data-Driven Gameplay  
Priority: P1

Q: Runtime item state should store what separately from definition?  
A: stack count, durability, generated rolls, owner, save ID and instance-specific state.  
Category: Data-Driven Gameplay  
Priority: P1

Q: Delegates can crash after destruction because why?  
A: Bound callbacks or lambda captures can outlive the object they reference.  
Category: UE C++ / Delegates  
Priority: P1

Q: Delegate cleanup should happen where?  
A: In every relevant teardown/cancel/end path, not only happy-path completion.  
Category: UE C++ / Delegates  
Priority: P1

Q: Reliable RPC overuse causes what under packet loss?  
A: Queue/backlog and delayed newer traffic.  
Category: Networking / RPC  
Priority: P1

Q: Reliable RPC is best for what?  
A: Important infrequent events where loss is unacceptable and rate is bounded.  
Category: Networking / RPC  
Priority: P1

Q: Replaceable high-rate updates should usually avoid what?  
A: Reliable RPC spam.  
Category: Networking / RPC  
Priority: P1

Q: OnRep order should not encode what?  
A: Multi-property semantic transactions.  
Category: Networking / OnRep  
Priority: P1

Q: Coherent replicated state often uses what?  
A: One struct/revision or idempotent state reconstruction.  
Category: Networking / Replication  
Priority: P1

Q: Dormant Actor mutation must usually do what first?  
A: Wake or flush according to target rules before changing replicated state.  
Category: Networking / Dormancy  
Priority: P1

Q: Fast Array needs stable item identity to distinguish what?  
A: Add, change, remove, reorder and late-join state.  
Category: Networking / Fast Array  
Priority: P1

Q: Fast Array mutation must be centralised because why?  
A: Dirty marking, identity and replication semantics must stay consistent.  
Category: Networking / Fast Array  
Priority: P1

Q: Replication Graph solves what class of problem?  
A: Scalable relevant Actor list building and per-connection replication policy.  
Category: Networking / RepGraph  
Priority: P2

Q: RepGraph should be adopted after what?  
A: Profiling proves baseline relevancy/list building is a bottleneck.  
Category: Networking / RepGraph  
Priority: P2

Q: Iris is automatic in all UE5 projects?  
A: No. Treat it as project/version/config dependent and verify target status.  
Category: Networking / Iris  
Priority: P1

Q: Replicated subobject disappearance starts debugging from what?  
A: owner replication, registration, lifetime, GC, conditions and client mapping timing.  
Category: Networking / Subobjects  
Priority: P2

Q: Client may see replicated subobject pointer null before what?  
A: Network mapping/creation has resolved on the client.  
Category: Networking / Subobjects  
Priority: P2

Q: `stat unit` is useful as what?  
A: A top-level split between Game, Draw/Render and GPU bottlenecks.  
Category: Profiling / Triage  
Priority: P1

Q: `stat unit` is not enough because why?  
A: It identifies broad bottleneck area, not root cause.  
Category: Profiling / Triage  
Priority: P1

Q: LLM is best for what memory view?  
A: Tagged high-level runtime memory budgets.  
Category: Profiling / Memory  
Priority: P1

Q: Memory Insights is best for what memory view?  
A: Allocation behaviour, callstacks and lifetime patterns when instrumented.  
Category: Profiling / Memory  
Priority: P1

Q: Optimisation can move bottleneck from where to where?  
A: Any direction among game thread, render thread, GPU, memory, IO or hitches.  
Category: Profiling / Method  
Priority: P1

Q: A profiling scenario must keep what stable?  
A: map, camera path, build, settings, hardware, warmup and workload.  
Category: Profiling / Method  
Priority: P1

Q: "Just make it async" ignores what?  
A: task overhead, synchronisation, lifetime, cancellation, merge cost and thread safety.  
Category: Performance / Async  
Priority: P1

Q: Async UObject mutation off-thread risks what?  
A: Data races, lifetime violations and engine API thread-safety bugs.  
Category: Performance / Async  
Priority: P1

Q: PSO hitch happens when what occurs during gameplay?  
A: A new graphics/compute pipeline state is compiled or initialised.  
Category: Rendering / PSO  
Priority: P2

Q: PSO proof must happen in what build type?  
A: Packaged target build, not only editor.  
Category: Rendering / PSO  
Priority: P2

Q: RDG resources should not be used outside what?  
A: Their declared render graph lifetime unless properly extracted/managed.  
Category: Rendering / RDG  
Priority: P2

Q: RDG pass bugs often involve missing what?  
A: Resource lifetime or dependency declarations.  
Category: Rendering / RDG  
Priority: P2

Q: Nanite is not universal because why?  
A: Platform, material, deformation, foliage, overdraw, fallback and feature constraints still matter.  
Category: Rendering / Nanite  
Priority: P2

Q: Lumen optimisation starts with what?  
A: Target-platform GPU evidence, scene content and quality/scalability trade-offs.  
Category: Rendering / Lumen  
Priority: P2

Q: Render targets hide what costs?  
A: GPU memory, bandwidth, extra passes, resolves and update frequency.  
Category: Rendering / Render Targets  
Priority: P2

Q: Scene captures become expensive mainly through what?  
A: resolution, format, update rate, show flags and number of captured views.  
Category: Rendering / Scene Capture  
Priority: P2

Q: Shader permutation explosion is caused by what?  
A: Many static feature/material variants generating many shader combinations.  
Category: Rendering / Shaders  
Priority: P2

Q: One master material can be bad because why?  
A: It can create excessive permutations and complex runtime/material cost.  
Category: Rendering / Materials  
Priority: P2

Q: Soft references do not guarantee what?  
A: The asset will be cooked/staged into every target package.  
Category: Assets / Cooking  
Priority: P1

Q: Soft-referenced assets need what discovery path?  
A: Asset Manager, labels, maps, explicit rules or other cook/stage references.  
Category: Assets / Cooking  
Priority: P1

Q: Redirectors hide what problem?  
A: Old asset paths still resolving locally while clean cook/other branches may fail.  
Category: Assets / Redirectors  
Priority: P2

Q: DDC can mask what?  
A: Missing generation steps, cold-build cost and local-only derived data assumptions.  
Category: Assets / DDC  
Priority: P2

Q: Editor-only module dependency in runtime code fails when?  
A: In packaged/runtime targets where editor modules are unavailable.  
Category: Build / Modules  
Priority: P1

Q: Plugin packaged-build failure starts with checking what?  
A: module type, dependencies, platform lists, runtime libraries and staging.  
Category: Build / Plugins  
Priority: P1

Q: BuildGraph is useful for what?  
A: Reproducible build/cook/package/test/artifact automation graphs.  
Category: Build / Automation  
Priority: P2

Q: Clean-agent builds expose what?  
A: undeclared dependencies, missing generated files, stale caches and local environment assumptions.  
Category: Build / CI  
Priority: P1

Q: Crash reports need what to be actionable?  
A: symbols, build ID, callstack, logs, platform, scenario and artifact identity.  
Category: Build / Crash Triage  
Priority: P1

Q: Generated artifact policy distinguishes what?  
A: reproducible build outputs versus authored/generated assets that must be versioned.  
Category: Build / Source Control  
Priority: P2

Q: Data validation should run where?  
A: In editor and CI/commandlet automation.  
Category: Tools / Validation  
Priority: P1

Q: Validation failure should include what?  
A: asset path, broken rule, reason and fix guidance.  
Category: Tools / Validation  
Priority: P1

Q: Commandlets are best for what?  
A: Repeatable headless editor/asset/build operations in automation.  
Category: Tools / Commandlets  
Priority: P2

Q: Python asset migration needs what safety mode?  
A: dry run/reporting before saving mutated assets.  
Category: Tools / Python  
Priority: P2

Q: Editor script batch edits need what recovery evidence?  
A: source-control checkout, logs, validation before/after and partial-failure plan.  
Category: Tools / Python  
Priority: P2

Q: AI Perception event is not the same as what?  
A: A complete long-term memory model.  
Category: AI / Perception  
Priority: P1

Q: AI memory should track stimulus what?  
A: source, sense, location, age, strength, success/loss and confidence.  
Category: AI / Perception  
Priority: P1

Q: Behaviour Tree Observer Aborts reduce what?  
A: High-frequency polling services for state changes that can be observed.  
Category: AI / Behaviour Trees  
Priority: P1

Q: EQS cost scales with what?  
A: candidate count, test cost, trace/nav queries, context complexity and AI count.  
Category: AI / EQS  
Priority: P1

Q: EQS optimisation often starts with what?  
A: cheaper filters first, lower frequency, bounded candidates and query profiling.  
Category: AI / EQS  
Priority: P1

Q: Custom AI sense bugs often start with missing what?  
A: sense config, source registration, listener update or stimulus ageing policy.  
Category: AI / Custom Senses  
Priority: P2

Q: StateTree task correctness depends on explicit what?  
A: data flow, instance data, external data and transition/completion semantics.  
Category: AI / StateTree  
Priority: P2

Q: Smart Object claims should be owned by whom for gameplay?  
A: The authoritative server/system when gameplay state is affected.  
Category: AI / Smart Objects  
Priority: P2

Q: Smart Object slot release must happen when?  
A: On success, abort, user death/despawn, timeout and teardown.  
Category: AI / Smart Objects  
Priority: P2

Q: Mass archetype explosion comes from what?  
A: Too many fragment/tag/shared-fragment combinations and structural churn.  
Category: MassEntity / Archetypes  
Priority: P2

Q: Mass hot state changes should often be represented as what?  
A: Fragment values rather than high-frequency tag/composition changes.  
Category: MassEntity / Structure  
Priority: P2

Q: Mass structural changes during iteration should use what?  
A: Deferred command patterns supported by the target branch.  
Category: MassEntity / Commands  
Priority: P2

Q: ZoneGraph route failure starts debugging with what?  
A: lane data/tags, required fragments, processor order and subsystem readiness.  
Category: MassAI / ZoneGraph  
Priority: P2

Q: GAS ASC placement controls what?  
A: ability lifetime, owner/avatar relationship, respawn behaviour and replication audience.  
Category: GAS / ASC  
Priority: P2

Q: ASC on PlayerState is useful when what is needed?  
A: Player ability/state persistence across Pawn respawn.  
Category: GAS / ASC  
Priority: P2

Q: ASC on Pawn is often simpler for what?  
A: Pawn-local enemies or abilities that do not persist beyond the Pawn.  
Category: GAS / ASC  
Priority: P2

Q: `FScopedPredictionWindow` does not make what safe?  
A: arbitrary irreversible gameplay side effects.  
Category: GAS / Prediction  
Priority: P2

Q: Gameplay tags are best for what?  
A: semantic labels, gating, categories, immunity and state queries.  
Category: GAS / Tags  
Priority: P2

Q: Gameplay tags are weak for what by themselves?  
A: numeric values, timers, durable identity and complex state machines.  
Category: GAS / Tags  
Priority: P2

Q: Attribute invariant should be enforced where?  
A: In authoritative attribute/effect paths, not only UI presentation.  
Category: GAS / Attributes  
Priority: P2

Q: Ability Tasks need cleanup for what exit paths?  
A: completion, cancellation, ability end, reject, timeout and avatar destruction.  
Category: GAS / Ability Tasks  
Priority: P2

Q: Enhanced Input mapping contexts are scoped to what?  
A: The LocalPlayer subsystem.  
Category: Enhanced Input / Contexts  
Priority: P1

Q: Mapping context priority matters because why?  
A: Multiple contexts can bind overlapping actions and one must win deterministically.  
Category: Enhanced Input / Contexts  
Priority: P1

Q: Input buffering should store semantic what?  
A: Commands with timing/context, not only raw key state.  
Category: Input / Buffering  
Priority: P2

Q: Animation graphs should read gameplay through what?  
A: Thread-safe snapshots prepared on the appropriate update path.  
Category: Animation / Threading  
Priority: P2

Q: Anim Notifies should not be sole authority for what?  
A: gameplay damage, rewards or irreversible combat state.  
Category: Animation / Notifies  
Priority: P1

Q: Animation state machine exit should not alone drive what?  
A: gameplay cancellation/cleanup for authority-critical actions.  
Category: Animation / State Machines  
Priority: P2

Q: Linked animation layers need what safe fallback?  
A: a default layer/pose and validated interface binding.  
Category: Animation / Linked Layers  
Priority: P2

Q: Motion Warping still needs what validation?  
A: target, range, collision, authority and cancellation rules.  
Category: Animation / Motion Warping  
Priority: P2

## Runtime Systems Rapid Review Expansion

Q: Query collision answers what kind of question?  
A: Spatial tests such as trace, sweep or overlap.  
Category: Physics / Collision  
Priority: P1

Q: Physics simulation handles what?  
A: bodies, contacts, impulses, constraints and solver integration.  
Category: Physics / Simulation  
Priority: P1

Q: Blocking response does not guarantee what?  
A: That gameplay hit/overlap logic executed.  
Category: Physics / Collision  
Priority: P1

Q: Sweep movement differs from teleport movement how?  
A: Sweep tests the path for blocking hits; teleport changes transform without path collision.  
Category: Physics / Movement  
Priority: P1

Q: Initial penetration in a sweep should trigger what investigation?  
A: start location overlap, collision shape, ignored actors and depenetration policy.  
Category: Physics / Debugging  
Priority: P1

Q: Simple collision is often used for what?  
A: fast gameplay collision and simulation primitives.  
Category: Physics / Collision  
Priority: P1

Q: Complex collision is risky for what?  
A: dynamic simulation or expensive gameplay queries when simple collision would suffice.  
Category: Physics / Collision  
Priority: P2

Q: Physical Material gives gameplay what?  
A: surface type/friction/restitution-style data for feedback or movement rules.  
Category: Physics / Materials  
Priority: P2

Q: Substepping improves what?  
A: physics stability under large/variable frame times.  
Category: Physics / Substepping  
Priority: P2

Q: Substepping does not fix what?  
A: bad ownership, wrong collision, unstable constraints or network authority bugs.  
Category: Physics / Substepping  
Priority: P2

Q: Constraint frames define what?  
A: pivot, axes and limits for the constrained bodies.  
Category: Physics / Constraints  
Priority: P2

Q: Constraint explosion often starts with checking what?  
A: local frames, mass ratio, collision between constrained bodies and solver settings.  
Category: Physics / Constraints  
Priority: P2

Q: CCD is not always right because why?  
A: sweep/hitscan/fixed-step movement may be cheaper and more deterministic for gameplay projectiles.  
Category: Physics / CCD  
Priority: P2

Q: Networked physics needs what first?  
A: target-branch support proof and clear authority/prediction ownership.  
Category: Physics / Networking  
Priority: P2

Q: CVD helps inspect what?  
A: Chaos scene queries, contacts, constraints and physics timeline evidence.  
Category: Physics / Debugging  
Priority: P2

Q: Projectile gameplay authority should usually validate what?  
A: fire request, owner, weapon state, timestamp, collision path and damage commit.  
Category: Physics / Projectiles  
Priority: P1

Q: CharacterMovement and physics both moving a capsule cause what?  
A: competing transform/velocity ownership and correction instability.  
Category: Physics / Movement  
Priority: P1

Q: Wrong collision profile bug evidence includes what?  
A: object channel, response matrix, enabled mode, component identity and query params.  
Category: Physics / Debugging  
Priority: P1

Q: Trace tags are useful for what?  
A: labelling and finding specific scene queries in debug tooling/logs.  
Category: Physics / Debugging  
Priority: P2

Q: Ragdoll recovery should restore what safely?  
A: collision mode, pose/capsule alignment, movement owner and network state.  
Category: Physics / Ragdoll  
Priority: P2

Q: Slate is primarily what kind of UI layer?  
A: Unreal's lower-level immediate/declarative UI framework underlying UMG.  
Category: UI / Slate  
Priority: P2

Q: UMG widgets should not own what?  
A: authoritative gameplay state.  
Category: UI / Architecture  
Priority: P1

Q: Widget `Construct` can run how many times?  
A: Potentially multiple times across widget lifetime/rebuild patterns.  
Category: UI / Lifecycle  
Priority: P1

Q: Widget teardown should unbind what?  
A: delegates, timers, async callbacks and gameplay subscriptions.  
Category: UI / Lifecycle  
Priority: P1

Q: ListView rows are what?  
A: recycled presentation widgets, not durable item identity.  
Category: UI / ListView  
Priority: P1

Q: Async icon loads in list rows need what guard?  
A: item ID/request generation check before applying result.  
Category: UI / Async Loading  
Priority: P1

Q: Invalidation helps when what remains stable?  
A: large unchanged UI subtrees.  
Category: UI / Performance  
Priority: P1

Q: Invalidation is harmed by what?  
A: volatile bindings, frequent layout changes or animations touching cached widgets.  
Category: UI / Performance  
Priority: P1

Q: Retainer Panel trades what for what?  
A: extra render target/capture cost for reduced redraw/composite cost.  
Category: UI / Performance  
Priority: P2

Q: World-space widgets scale poorly due to what?  
A: widget updates, layout, render targets, draw cost and visibility/occlusion overhead.  
Category: UI / World Space  
Priority: P2

Q: Screen-space nameplates often need what policy?  
A: projection, relevance, occlusion, sorting and throttled updates.  
Category: UI / Nameplates  
Priority: P2

Q: CommonUI active screen bugs often involve what?  
A: activatable stack state, focus, input action routing or wrong local player.  
Category: UI / CommonUI  
Priority: P1

Q: SetUserFocus should target what?  
A: the correct local player/user context.  
Category: UI / Focus  
Priority: P1

Q: Safe Zones matter because why?  
A: screens may have notches, overscan or platform UI-reserved regions.  
Category: UI / Layout  
Priority: P1

Q: DPI scaling should be tested with what?  
A: multiple resolutions, aspect ratios, font scales and platform profiles.  
Category: UI / Layout  
Priority: P1

Q: Accessibility UI should avoid relying only on what?  
A: colour, audio or tiny fixed-size text.  
Category: UI / Accessibility  
Priority: P1

Q: Localised text can break UI by doing what?  
A: expanding, changing direction, using different glyphs or requiring wrapping.  
Category: UI / Localisation  
Priority: P1

Q: Widget Reflector is useful for what?  
A: inspecting widget hierarchy, hit testing, focus and layout.  
Category: UI / Debugging  
Priority: P1

Q: Slate Insights is useful for what?  
A: tracing UI invalidation, painting, update cost and widget performance.  
Category: UI / Debugging  
Priority: P1

Q: Sound Cue is generally what?  
A: an authored sound graph/asset for playback logic.  
Category: Audio / Sound Cue  
Priority: P2

Q: MetaSound is generally what?  
A: a procedural audio graph/source system.  
Category: Audio / MetaSounds  
Priority: P2

Q: MetaSounds are not automatically faster because why?  
A: graph complexity, voices, DSP and parameter updates still cost CPU.  
Category: Audio / Performance  
Priority: P2

Q: Attenuation controls what?  
A: distance/spatialisation/falloff-style behaviour for audible sources.  
Category: Audio / Attenuation  
Priority: P1

Q: Concurrency controls what?  
A: voice limits, rejection, stealing, virtualisation and priority among sounds.  
Category: Audio / Concurrency  
Priority: P1

Q: Audio class/mix bugs often look like what?  
A: sound plays but is inaudible due to gain/routing/mix state.  
Category: Audio / Routing  
Priority: P1

Q: Submixes are useful for what?  
A: routing, effects, recording/analysis and mix processing.  
Category: Audio / Submix  
Priority: P2

Q: Quartz is useful for what timing?  
A: sample-accurate or musical scheduling.  
Category: Audio / Quartz  
Priority: P2

Q: Quartz should not own what?  
A: authoritative gameplay rules in multiplayer.  
Category: Audio / Quartz  
Priority: P2

Q: Audio stream cache should be tested where?  
A: in cooked target builds under realistic IO/memory conditions.  
Category: Audio / Streaming  
Priority: P2

Q: Missing audio in package often starts with checking what?  
A: cook/staging, chunk, stream cache, routing and concurrency.  
Category: Audio / Debugging  
Priority: P1

Q: Attached sound bugs often involve what?  
A: destroyed attach parent, wrong transform, stale pooled component or owner lifetime.  
Category: Audio / Attached Sounds  
Priority: P1

Q: Critical gameplay warning needs what redundancy?  
A: non-audio feedback such as UI, visual, haptic or gameplay affordance.  
Category: Audio / Accessibility  
Priority: P1

Q: Networked audio should usually replicate what?  
A: durable gameplay state/events, not raw local audio component state.  
Category: Audio / Networking  
Priority: P1

Q: Late-join looping audio should reconstruct from what?  
A: authoritative durable state, not historical one-shot playback.  
Category: Audio / Networking  
Priority: P2

Q: Niagara system contains what?  
A: emitters, modules, parameters, renderers and execution state.  
Category: VFX / Niagara  
Priority: P1

Q: Niagara parameters should be set when?  
A: before activation when initial spawn behaviour depends on them.  
Category: VFX / Parameters  
Priority: P1

Q: Niagara fixed bounds too small cause what?  
A: visible particles/effects being culled.  
Category: VFX / Bounds  
Priority: P1

Q: Niagara fixed bounds too large cause what?  
A: poor culling efficiency and unnecessary work.  
Category: VFX / Bounds  
Priority: P1

Q: Effect Types group systems for what?  
A: budget, significance and scalability policy.  
Category: VFX / Scalability  
Priority: P2

Q: Niagara Data Channels should not hold what?  
A: durable authoritative gameplay truth.  
Category: VFX / Data Channels  
Priority: P1

Q: Gameplay should feed Niagara what?  
A: presentation data derived from authoritative gameplay state.  
Category: VFX / Architecture  
Priority: P1

Q: GPU Niagara simulation risks what for gameplay collision?  
A: delay, approximation and authority mismatch.  
Category: VFX / GPU Simulation  
Priority: P2

Q: Niagara component pooling needs what reset?  
A: parameters, attachment, activation state, delegates and owner/lifetime state.  
Category: VFX / Pooling  
Priority: P1

Q: VFX overdraw is worst with what materials?  
A: large translucent layers and particles covering many pixels.  
Category: VFX / Overdraw  
Priority: P1

Q: Lightweight Emitters should be treated how?  
A: as version/platform-specific optimisation candidates requiring target proof.  
Category: VFX / Niagara  
Priority: P2

Q: Niagara Debugger helps inspect what?  
A: system instances, parameters, culling/scalability and runtime state.  
Category: VFX / Debugging  
Priority: P1

Q: FX Outliner is useful for what?  
A: finding active systems and their runtime state/cost clues.  
Category: VFX / Debugging  
Priority: P1

Q: SpawnSystemAttached lifetime bug often involves what?  
A: destroyed attach parent, stale pooled component or wrong auto-destroy policy.  
Category: VFX / Lifetime  
Priority: P1

Q: Gameplay hit detection should not rely on what VFX system?  
A: particle collision events as final authority.  
Category: VFX / Gameplay Boundary  
Priority: P1

Q: A* optimality needs what heuristic property?  
A: it must not overestimate remaining cost.  
Category: Algorithms / A*  
Priority: P1

Q: A* can be correct but feel wrong if what is wrong?  
A: graph model, costs, movement constraints or heuristic.  
Category: Algorithms / A*  
Priority: P1

Q: Priority queue supports A* by doing what?  
A: efficiently selecting the lowest-priority frontier node.  
Category: Algorithms / Pathfinding  
Priority: P1

Q: Union-find is strong for what?  
A: incremental connectivity by merging sets.  
Category: Algorithms / Union-Find  
Priority: P2

Q: Union-find is weak for what?  
A: dynamic connectivity with deletions/splits.  
Category: Algorithms / Union-Find  
Priority: P2

Q: Spatial hash works best when what is roughly uniform?  
A: object sizes, density and local query scale.  
Category: Algorithms / Spatial Hash  
Priority: P2

Q: Spatial hash cell size controls what trade-off?  
A: too many cells versus too many candidates per cell.  
Category: Algorithms / Spatial Hash  
Priority: P2

Q: BVH is often useful for what?  
A: hierarchical broadphase queries over spatial objects.  
Category: Algorithms / BVH  
Priority: P2

Q: Topological sort detects what in dependency graphs?  
A: ordering and cycles among directed dependencies.  
Category: Algorithms / Graphs  
Priority: P2

Q: Binary search requires what precondition?  
A: sorted/searchable monotonic ordering.  
Category: Algorithms / Search  
Priority: P1

Q: Lower_bound is useful for what gameplay pattern?  
A: selecting threshold ranges such as XP levels or weighted cumulative tables.  
Category: Algorithms / Search  
Priority: P1

Q: Weighted random using cumulative weights needs what guard?  
A: total weight validity and stable handling of zero/invalid weights.  
Category: Algorithms / Random  
Priority: P2

Q: Big-O hides what practical costs?  
A: constants, cache locality, allocation, branch prediction and real data size.  
Category: Algorithms / Complexity  
Priority: P1

Q: Ring buffer is useful for what network/debug systems?  
A: recent input, history snapshots, rollback windows and telemetry.  
Category: Algorithms / Ring Buffer  
Priority: P1

Q: Ring buffer policy must define what?  
A: overwrite, reject, grow or back-pressure behaviour when full.  
Category: Algorithms / Ring Buffer  
Priority: P1

Q: Unreal uses what coordinate convention for vertical axis?  
A: Z-up.  
Category: Game Math / Coordinates  
Priority: P1

Q: Local space means what?  
A: coordinates relative to an object's parent/frame.  
Category: Game Math / Spaces  
Priority: P1

Q: World space means what?  
A: coordinates in the world's global frame.  
Category: Game Math / Spaces  
Priority: P1

Q: Dot product answers what common gameplay question?  
A: alignment/projection/angle relationship between directions.  
Category: Game Math / Vectors  
Priority: P1

Q: Cross product gives what?  
A: perpendicular vector and orientation/sign information relative to an axis.  
Category: Game Math / Vectors  
Priority: P1

Q: Clamp dot before acos because why?  
A: floating-point error can produce values slightly outside [-1, 1].  
Category: Game Math / Numerical Safety  
Priority: P1

Q: Quaternion `q` and `-q` represent what?  
A: the same rotation.  
Category: Game Math / Quaternions  
Priority: P1

Q: Quaternion interpolation should usually check what?  
A: dot sign to choose shortest path.  
Category: Game Math / Quaternions  
Priority: P1

Q: Non-uniform scale changes normal transformation how?  
A: normals need inverse-transpose-style handling, not ordinary position transform.  
Category: Game Math / Transforms  
Priority: P2

Q: LWC solves what pressure?  
A: precision problems in very large worlds by using larger coordinate precision internally.  
Category: Game Math / LWC  
Priority: P2

Q: Ray-box intersection must handle what edge case?  
A: near-zero direction components/slab division stability.  
Category: Game Math / Intersections  
Priority: P2

Q: Interpolation alpha should usually be what?  
A: clamped or deliberately allowed to extrapolate based on design.  
Category: Game Math / Interpolation  
Priority: P1

Q: Smooth damping differs from simple lerp by considering what?  
A: velocity/time behaviour rather than fixed fraction only.  
Category: Game Math / Smoothing  
Priority: P2

Q: Material roughness affects what?  
A: specular reflection spread/appearance.  
Category: Rendering / Materials  
Priority: P2

Q: Texture memory depends heavily on what?  
A: resolution, format, mipmaps and streaming/residency policy.  
Category: Rendering / Textures  
Priority: P1

Q: World Partition controls what?  
A: large-world spatial loading/streaming organisation.  
Category: World Partition  
Priority: P1

Q: Data Layers control what?  
A: content grouping/visibility/loading state, not durable quest truth by themselves.  
Category: World Partition / Data Layers  
Priority: P1

Q: Runtime Data Layer state should be driven by what?  
A: durable gameplay/save/server state when gameplay meaning matters.  
Category: World Partition / Data Layers  
Priority: P1

Q: OFPA improves what workflow?  
A: parallel level editing by storing Actors in separate files.  
Category: World Building / OFPA  
Priority: P2

Q: OFPA does not eliminate what?  
A: source-control churn, ownership policy or validation needs.  
Category: World Building / OFPA  
Priority: P2

Q: HLOD generates what kind of output?  
A: proxy/simplified representations for distant large-world content.  
Category: World Partition / HLOD  
Priority: P2

Q: HLOD is a build pipeline issue because why?  
A: generated assets must be reproducible, validated, packaged and source-controlled appropriately.  
Category: World Partition / HLOD  
Priority: P2

Q: Level Instance is useful for what?  
A: reusable authored level chunks or repeated environment assemblies.  
Category: World Building / Level Instances  
Priority: P2

Q: Streaming readiness is not gameplay readiness because why?  
A: dependencies, replication, async assets and system init may still be incomplete.  
Category: World Partition / Readiness  
Priority: P1

Q: PCG is strongest for what?  
A: procedural content authoring/generation where ownership and output policy are clear.  
Category: PCG / Overview  
Priority: P2

Q: PCG runtime output needs what policy?  
A: save, replication, cleanup, determinism, nav/collision and authority.  
Category: PCG / Runtime  
Priority: P2

Q: PCG graph should not own what by default?  
A: durable gameplay truth unless explicitly designed for it.  
Category: PCG / Architecture  
Priority: P2

Q: PCG generated blockers need what extra proof?  
A: navigation, collision, replication/save and cleanup behaviour.  
Category: PCG / Runtime  
Priority: P2

Q: Lua hot reload leaves what stale?  
A: old closures, tables, callbacks and userdata wrappers.  
Category: Lua / Hot Reload  
Priority: P2

Q: Lua coroutine cancellation needs what?  
A: explicit cancellation protocol and cleanup of callbacks/resources.  
Category: Lua / Coroutines  
Priority: P2

Q: Lua/Unreal GC boundary needs what design?  
A: clear ownership/rooting/wrapper invalidation and no stale UObject access.  
Category: Lua / Lifetime  
Priority: P2

Q: C# interop with Unreal needs what lifetime pattern?  
A: handles/wrappers with explicit invalidation and disposal/cleanup.  
Category: C# / Interop  
Priority: P2

Q: C# callbacks into Unreal need what thread policy?  
A: marshal results back to the game thread before UObject mutation.  
Category: C# / Threading  
Priority: P2

Q: Native AOT/trimming can remove what?  
A: reflection-discovered types or binding metadata unless rooted/configured.  
Category: C# / AOT  
Priority: P2

Q: Script API facade should expose what?  
A: stable IDs, snapshots, validated commands, events and cancellation.  
Category: Scripting / API Design  
Priority: P2

Q: Script API facade should hide what?  
A: raw mutable UObject ownership and authority-critical internals.  
Category: Scripting / API Design  
Priority: P2

Q: Android performance must be tested where?  
A: packaged target builds on representative devices.  
Category: Platform / Android  
Priority: P1

Q: Mobile thermal throttling invalidates what?  
A: short cold-run benchmark conclusions.  
Category: Platform / Mobile  
Priority: P1

Q: Device profile changes must prove what?  
A: target-hardware performance gain without unacceptable visual/memory regressions.  
Category: Platform / Device Profiles  
Priority: P1

Q: ADPF is relevant to what?  
A: Android dynamic performance/thermal/game mode style platform adaptation.  
Category: Platform / Android  
Priority: P2

Q: AGI helps inspect what?  
A: Android GPU/frame processing and render pass/counter evidence.  
Category: Platform / Android Profiling  
Priority: P2

Q: iOS/Apple profiling should correlate what?  
A: Xcode/Metal/MetricKit signals with Unreal frame and content evidence.  
Category: Platform / Apple  
Priority: P2

Q: Metal capture helps inspect what?  
A: GPU workload, passes, resources and shader/rendering cost on Apple targets.  
Category: Platform / Apple GPU  
Priority: P2

Q: MetricKit is useful for what kind of signal?  
A: field performance/diagnostic metrics from real app usage.  
Category: Platform / Apple  
Priority: P2

Q: Console profiling notes should avoid what?  
A: NDA-controlled counters, thresholds and proprietary devkit details in public-safe material.  
Category: Platform / Console  
Priority: P1

Q: Device-lab manifest should record what?  
A: build ID, device ID, OS, scenario, settings, timestamps, logs and artifacts.  
Category: Platform / Device Lab  
Priority: P1

Q: Automation flake triage needs what classification?  
A: product fail, lab infrastructure fail, flaky/quarantine, pass or missing evidence.  
Category: Automation / Triage  
Priority: P1

Q: Rerun-until-green hides what?  
A: deterministic failures and weak test/lab ownership.  
Category: Automation / Triage  
Priority: P1

## Debug Workflow Flashcard Expansion

Q: First step for any Unreal bug triage?  
A: Reproduce with exact build, map, role/net mode, platform and steps.  
Category: Debugging / Method  
Priority: P1

Q: A good bug log starts with what?  
A: stable object identity, world/net mode, role, timestamp/frame and scenario marker.  
Category: Debugging / Logging  
Priority: P1

Q: Why log IDs instead of display names?  
A: display names/localised labels can change or duplicate; IDs connect systems reliably.  
Category: Debugging / Logging  
Priority: P1

Q: A breakpoint is weak without what?  
A: a hypothesis about which invariant or ownership boundary failed.  
Category: Debugging / Method  
Priority: P1

Q: Good repro minimisation removes what?  
A: unrelated content, timing noise and systems not needed to trigger the bug.  
Category: Debugging / Reproduction  
Priority: P1

Q: Regression proof needs what?  
A: a failing case before the fix and a repeatable passing case after.  
Category: Debugging / Verification  
Priority: P1

Q: Debugging multiplayer begins with what question?  
A: Which machine owns the authoritative truth?  
Category: Networking / Debugging  
Priority: P1

Q: Server RPC failure first checks what?  
A: owning connection and whether the caller owns the Actor.  
Category: Networking / RPC Debugging  
Priority: P1

Q: Client RPC not arriving first checks what?  
A: server call site, owning connection, target Actor and connection relevance/lifetime.  
Category: Networking / RPC Debugging  
Priority: P1

Q: Multicast called from a client does what?  
A: it does not become a server-authoritative multicast to everyone.  
Category: Networking / RPC Debugging  
Priority: P1

Q: Missing replicated property first checks what?  
A: `bReplicates`, lifetime registration, authority mutation and relevance/dormancy.  
Category: Networking / Property Debugging  
Priority: P1

Q: OnRep not firing might mean what?  
A: value did not change, property not replicated, Actor not relevant, or update not sent.  
Category: Networking / OnRep Debugging  
Priority: P1

Q: Wrong client sees private state often means what?  
A: incorrect replication condition or ownership/audience model.  
Category: Networking / Privacy  
Priority: P1

Q: Bandwidth spike debugging ranks what first?  
A: Actors, properties and RPCs by bytes/frequency in network capture.  
Category: Networking / Profiling  
Priority: P1

Q: Reliable RPC backlog debugging checks what?  
A: reliable queue frequency, packet loss, call rate and replaceable traffic.  
Category: Networking / RPC Debugging  
Priority: P1

Q: Fast Array missed update first checks what?  
A: stable ID, mutation path and dirty mark.  
Category: Networking / Fast Array Debugging  
Priority: P1

Q: Fast Array wrong row update often means what?  
A: reused/missing item identity or stale UI selection.  
Category: Networking / Fast Array Debugging  
Priority: P1

Q: Subobject null on client may be normal until what?  
A: the subobject has been created/mapped by replication.  
Category: Networking / Subobject Debugging  
Priority: P2

Q: Stale subobject crash checks what?  
A: registered list removal before deletion/GC.  
Category: Networking / Subobject Debugging  
Priority: P2

Q: RepGraph scaling proof needs what before adoption?  
A: baseline relevant Actor and server net processing measurements.  
Category: Networking / RepGraph Debugging  
Priority: P2

Q: Iris debugging starts with what?  
A: target project config/source proving Iris is enabled and used.  
Category: Networking / Iris Debugging  
Priority: P2

Q: CharacterMovement sprint jitter first checks what?  
A: whether sprint intent is saved/restored in predicted moves.  
Category: Networking / Movement Debugging  
Priority: P1

Q: Saved move transition lost often means what?  
A: `CanCombineWith` merged across a meaningful state change.  
Category: Networking / Movement Debugging  
Priority: P1

Q: Direct transform dash causes what symptom?  
A: correction snaps or disagreement with CharacterMovement simulation.  
Category: Networking / Movement Debugging  
Priority: P1

Q: Lag-compensation exploit debugging checks what?  
A: timestamp bounds, target generation, aim plausibility and authority trace.  
Category: Networking / Rewind Debugging  
Priority: P2

Q: Rewind history buffer should store what kind of data?  
A: value snapshots, not mutable references to current transforms.  
Category: Networking / Rewind Debugging  
Priority: P2

Q: Rollback duplicated VFX first checks what?  
A: cue key/dedupe, cue trait and whether tick directly spawned effects.  
Category: Networking / Rollback Debugging  
Priority: P2

Q: Rollback replay with wrong tunings first checks what?  
A: aux state versioning and current Actor/data reads during replay.  
Category: Networking / Rollback Debugging  
Priority: P2

Q: Rollback correction storm first checks what?  
A: first divergent sync field, input hash, aux version and tick step.  
Category: Networking / Rollback Debugging  
Priority: P2

Q: GAS ability fails to activate first checks what?  
A: ASC init, avatar/owner, tags, cost, cooldown, input binding and authority.  
Category: GAS / Debugging  
Priority: P2

Q: GAS predicted ability snaps back first checks what?  
A: server rejection reason, prediction key, side effects and client/server rule mismatch.  
Category: GAS / Prediction Debugging  
Priority: P2

Q: GAS cue double-play first checks what?  
A: predicted cue, confirmed cue and dedupe/key policy.  
Category: GAS / Cue Debugging  
Priority: P2

Q: GAS missing attribute update first checks what?  
A: AttributeSet registration, modifier path, replication mode and UI subscription.  
Category: GAS / Attribute Debugging  
Priority: P2

Q: GAS grant leak first checks what?  
A: grant source ledger, handles and removal on unequip/death/class change.  
Category: GAS / Spec Debugging  
Priority: P2

Q: GameplayEffect stacking bug first checks what?  
A: stacking policy, source/target aggregation, expiration and refresh rules.  
Category: GAS / Effect Debugging  
Priority: P2

Q: AbilityTask callback after cancel checks what?  
A: delegate unbind, timer cleanup, `OnDestroy` path and weak avatar validation.  
Category: GAS / Task Debugging  
Priority: P2

Q: Enhanced Input action missing checks what?  
A: mapping context added to correct LocalPlayer, priority, trigger/modifier and component binding.  
Category: Input / Debugging  
Priority: P1

Q: Input fires in menu behind modal checks what?  
A: UI stack, focus, input mode and stale gameplay mapping context.  
Category: Input / UI Debugging  
Priority: P1

Q: Split-screen wrong player input checks what?  
A: local player ownership, focus target and subsystem/context routing.  
Category: Input / Split Screen  
Priority: P1

Q: Animation pose wrong after equipment swap checks what?  
A: linked layer binding, Anim Class, skeleton compatibility and fallback pose.  
Category: Animation / Debugging  
Priority: P2

Q: Anim notify missed checks what?  
A: montage section, blend/interruption, notify state lifetime and dedicated-server assumptions.  
Category: Animation / Notify Debugging  
Priority: P1

Q: Montage slot ignored checks what?  
A: slot node exists in graph, correct slot/group and montage plays on expected instance.  
Category: Animation / Montage Debugging  
Priority: P1

Q: Foot sliding first compares what?  
A: movement velocity/root motion, pose speed/direction and ground contact.  
Category: Animation / Locomotion Debugging  
Priority: P1

Q: Retargeted animation looks wrong checks what?  
A: retarget pose, skeleton proportions, root settings and IK chain mapping.  
Category: Animation / Retarget Debugging  
Priority: P2

Q: Root motion networking bug checks what?  
A: who owns displacement, movement mode, collision and correction path.  
Category: Animation / Root Motion Debugging  
Priority: P2

Q: Physics hit missing first checks what?  
A: moving component, sweep/simulation path, collision enabled, channels and event flags.  
Category: Physics / Debugging  
Priority: P1

Q: Overlap missing first checks what?  
A: collision responses, generate overlap events, component identity and initial overlap state.  
Category: Physics / Debugging  
Priority: P1

Q: Trace hits self first checks what?  
A: ignored actors/components and query params.  
Category: Physics / Trace Debugging  
Priority: P1

Q: Constraint jitter first checks what?  
A: frames, mass ratio, collision, solver settings and substep conditions.  
Category: Physics / Constraint Debugging  
Priority: P2

Q: Tunnelling bug first checks what?  
A: movement method, speed/timestep, shape size, sweep/CCD and authority path.  
Category: Physics / Collision Debugging  
Priority: P2

Q: Ragdoll never recovers checks what?  
A: physics blend, collision mode, capsule alignment, movement re-enable and network state.  
Category: Physics / Ragdoll Debugging  
Priority: P2

Q: UI row shows wrong icon first checks what?  
A: recycled row identity and async request generation.  
Category: UI / List Debugging  
Priority: P1

Q: UI button cannot be clicked first checks what?  
A: hit-test visibility, hierarchy, focus, input mode and overlay blockers.  
Category: UI / Interaction Debugging  
Priority: P1

Q: UI jank first checks what?  
A: bindings/tick, invalidation, layout rebuilds, list virtualisation and Slate traces.  
Category: UI / Performance Debugging  
Priority: P1

Q: Retainer experiment must compare what?  
A: redraw saved versus render-target/composite/memory cost added.  
Category: UI / Performance Debugging  
Priority: P2

Q: Text clipped after localisation checks what?  
A: layout constraints, wrapping, font scale, pseudo-localisation and longest strings.  
Category: UI / Localisation Debugging  
Priority: P1

Q: Controller focus lost checks what?  
A: focused widget, navigation rules, active stack, local player and input mode.  
Category: UI / Focus Debugging  
Priority: P1

Q: Inaudible sound first checks what?  
A: request fired, component/voice created, asset valid, attenuation, concurrency, class/mix and output.  
Category: Audio / Debugging  
Priority: P1

Q: Sound stolen unexpectedly checks what?  
A: concurrency group, priority, max count and resolution rule.  
Category: Audio / Concurrency Debugging  
Priority: P1

Q: Attached sound at wrong place checks what?  
A: attach target, socket, transform rules, parent lifetime and component reuse.  
Category: Audio / Attached Debugging  
Priority: P1

Q: MetaSound performance bug checks what?  
A: graph complexity, voice count, parameter update rate and DSP/submix cost.  
Category: Audio / MetaSound Debugging  
Priority: P2

Q: Beat timing drift checks what?  
A: game-thread timer versus Quartz/audio-clock scheduling under hitches.  
Category: Audio / Timing Debugging  
Priority: P2

Q: Missing packaged audio checks what?  
A: cook/stage/chunk, stream cache, platform format and runtime logs.  
Category: Audio / Packaging Debugging  
Priority: P2

Q: Niagara effect not visible first checks what?  
A: activation, asset validity, bounds, scalability, significance and renderer visibility.  
Category: VFX / Debugging  
Priority: P1

Q: Niagara effect disappears at distance checks what?  
A: bounds, Effect Type distance rules, significance and scalability budget.  
Category: VFX / Culling Debugging  
Priority: P1

Q: Pooled Niagara keeps old colour checks what?  
A: parameter reset before reuse and component activation state.  
Category: VFX / Pooling Debugging  
Priority: P1

Q: Niagara Data Channel bug checks what?  
A: writer/reader lifetime, data schema, frame timing and authority boundary.  
Category: VFX / Data Channel Debugging  
Priority: P2

Q: GPU particle gameplay mismatch checks what?  
A: GPU delay/readback, server authority and whether gameplay used VFX data incorrectly.  
Category: VFX / Gameplay Debugging  
Priority: P2

Q: VFX overdraw problem checks what?  
A: translucent area, particle count, material complexity and screen coverage.  
Category: VFX / Performance Debugging  
Priority: P1

Q: AI never sees target first checks what?  
A: controller/perception component, sense config, stimuli source, team/filter and debug view.  
Category: AI / Perception Debugging  
Priority: P1

Q: AI chases old location forever checks what?  
A: stimulus age, memory expiry, blackboard clearing and state transitions.  
Category: AI / Memory Debugging  
Priority: P1

Q: Behaviour Tree task stuck checks what?  
A: finish/abort callback, latent task state and blackboard aborts.  
Category: AI / BT Debugging  
Priority: P1

Q: EQS returns no result checks what?  
A: generator, context, filters, navmesh, trace tests and query debug output.  
Category: AI / EQS Debugging  
Priority: P1

Q: MoveTo fails first checks what?  
A: navmesh, agent radius, destination projection, path following result and movement component.  
Category: AI / Navigation Debugging  
Priority: P1

Q: Detour/RVO avoidance jitter checks what?  
A: avoidance mode, agent radii, crowd settings, desired velocity and path following ownership.  
Category: AI / Avoidance Debugging  
Priority: P2

Q: Mass processor not running checks what?  
A: processor registration, execution flags, phase/order and world/subsystem setup.  
Category: MassEntity / Debugging  
Priority: P2

Q: Mass query matches zero checks what?  
A: fragment/tag requirements, archetype composition and entity config/trait output.  
Category: MassEntity / Query Debugging  
Priority: P2

Q: Mass representation missing checks what?  
A: representation fragments, LOD/significance, spawner/config and visual subsystem readiness.  
Category: MassEntity / Representation Debugging  
Priority: P2

Q: Mass Actor promotion stale pointer checks what?  
A: stable entity ID mapping, Actor pool reset and demotion cleanup.  
Category: MassEntity / Hybrid Debugging  
Priority: P2

Q: PCG output stale after graph change checks what?  
A: generation mode, cleanup policy, seed/graph version and component regeneration.  
Category: PCG / Debugging  
Priority: P2

Q: PCG interactable missing after load checks what?  
A: save/replication identity, generation determinism and runtime ownership policy.  
Category: PCG / Save Debugging  
Priority: P2

Q: World Partition actor missing checks what?  
A: cell loading, Data Layer state, streaming source, actor descriptor and cook/package inclusion.  
Category: World Partition / Debugging  
Priority: P1

Q: Data Layer phase wrong after load checks what?  
A: durable gameplay state driving layer state, not layer state as sole truth.  
Category: World Partition / Data Layer Debugging  
Priority: P1

Q: HLOD wrong in package checks what?  
A: generated assets, source-control status, builder output, cook inclusion and visual validation.  
Category: World Partition / HLOD Debugging  
Priority: P2

Q: Soft asset fails only in package checks what?  
A: cook rule, Asset Manager registration, chunk/stage path and editor-only reference.  
Category: Assets / Packaging Debugging  
Priority: P1

Q: Async asset callback stale checks what?  
A: request handle, generation, weak owner and cancellation path.  
Category: Assets / Async Debugging  
Priority: P1

Q: Asset redirector bug checks what?  
A: fix-up status, reference viewer, source-control churn and clean cook result.  
Category: Assets / Redirector Debugging  
Priority: P2

Q: DDC issue checks what?  
A: clean/warm cache comparison, shared DDC config and generated data validity.  
Category: Assets / DDC Debugging  
Priority: P2

Q: Packaged plugin missing DLL checks what?  
A: runtime dependency declaration, staging rule, platform path and module loading.  
Category: Build / Packaging Debugging  
Priority: P1

Q: UHT error after include change checks what?  
A: reflected declaration syntax, generated header include order and module dependencies.  
Category: Build / UHT Debugging  
Priority: P1

Q: Clean CI fail but local pass checks what?  
A: undeclared dependency, untracked file, cache state, environment variable or SDK install.  
Category: Build / CI Debugging  
Priority: P1

Q: BuildGraph failure first checks what?  
A: first failed node, inputs/outputs/artifacts, environment and command line.  
Category: Build / Automation Debugging  
Priority: P2

Q: Crash without symbols first fixes what?  
A: symbol publishing/build artifact identity pipeline.  
Category: Build / Crash Debugging  
Priority: P1

Q: Mobile frame regression first checks what?  
A: packaged build, device profile, thermal state, CPU/GPU split and platform counters.  
Category: Platform / Mobile Debugging  
Priority: P1

Q: Android GPU issue checks what with AGI?  
A: frame time, render passes, counters and expensive GPU workload.  
Category: Platform / Android Debugging  
Priority: P2

Q: iOS memory issue checks what first?  
A: Xcode memory tools, Unreal memory tags, allocation pattern and target-device reproduction.  
Category: Platform / Apple Debugging  
Priority: P2

Q: Apple GPU issue checks what?  
A: Metal workload capture, pass/resource cost and Unreal render feature mapping.  
Category: Platform / Apple Debugging  
Priority: P2

Q: Device-lab install failure should be classified as what until proven otherwise?  
A: lab/infrastructure or packaging/deployment issue, not gameplay regression.  
Category: Device Lab / Triage  
Priority: P1

Q: Automation missing evidence should result in what gate state?  
A: missing-evidence/inconclusive, not pass.  
Category: Automation / Triage  
Priority: P1

## Question Bank Derived Flashcards

Q: Why does Unreal need reflection on top of C++?
A: Compiled C++ does not provide the structured runtime metadata Unreal needs for editor tooling, Blueprint exposure, serialisation, replication declarations, and managed object references.
Category: Derived Interview Bank / UObject / Reflection
Priority: P0

Q: What does `UPROPERTY` do besides exposing a field to the editor?
A: Depending on type and specifiers, it can make a property visible to reflection-driven GC reference discovery, serialisation, replication declarations, Blueprint access, config/save systems, and editor tooling.
Category: Derived Interview Bank / UObject / Reflection / UE C++
Priority: P0

Q: Explain UObject garbage collection without saying “Unreal handles it”.
A: Unreal periodically marks UObjects reachable from roots through discoverable strong references and reclaims unreachable objects in a later sweep/destruction process.
Category: Derived Interview Bank / UObject / Memory
Priority: P0

Q: Why should a persistent UObject member usually be `UPROPERTY() TObjectPtr<T>`?
A: It expresses a reflection-visible strong UObject reference that the collector and configured engine systems can track.
Category: Derived Interview Bank / UObject / Memory / UE C++
Priority: P0

Q: What is the difference between weak and soft UObject references?
A: A weak pointer observes a loaded object without retaining it; a soft pointer stores an object path and can represent an unloaded asset for explicit loading.
Category: Derived Interview Bank / UObject / Assets / Lifetime
Priority: P0

Q: What does `Outer` mean, and does it keep an object alive?
A: Outer establishes containment and identity context, including object paths; do not rely on it as a substitute for an explicit strong reference.
Category: Derived Interview Bank / UObject / Lifetime
Priority: P1

Q: What is the CDO, and why should constructors be lightweight?
A: The Class Default Object is the class template from which defaults are derived; constructors initialise defaults and default subobjects, so world-dependent gameplay work belongs later.
Category: Derived Interview Bank / UObject / Lifecycle
Priority: P0

Q: Why is `TSharedPtr<UObject>` usually the wrong ownership model?
A: Reference-counted C++ smart pointers and Unreal's UObject reachability collector are separate lifetime systems; UObject references should use the UObject-aware mechanisms that express retention, observation, or loading intent.
Category: Derived Interview Bank / C++ / UObject Lifetime
Priority: P0

Q: Actor or Actor Component—how do you decide?
A: Use an Actor for independent world identity, spawning, placement, or replication relevance; use a Component for a coherent capability whose lifetime and context belong to an Actor.
Category: Derived Interview Bank / Gameplay Framework / Architecture
Priority: P0

Q: Pawn versus Character?
A: Character is a specialised Pawn with humanoid-oriented collision, movement, animation, and network movement support; Pawn is the more general possessable agent.
Category: Derived Interview Bank / Gameplay Framework
Priority: P0

Q: Why separate Controller from Pawn?
A: Controller represents decision-making/player connection while Pawn is the world avatar, allowing possession changes, respawn, spectating, and reuse across human and AI control.
Category: Derived Interview Bank / Gameplay Framework / Architecture
Priority: P0

Q: GameMode versus GameState?
A: GameMode owns authoritative match rules and exists only on the server; GameState exposes changing match state that exists on server and clients and can replicate.
Category: Derived Interview Bank / Gameplay Framework / Networking
Priority: P0

Q: PlayerController versus PlayerState?
A: PlayerController represents control/connection and exists on server plus its owning client; PlayerState represents public participant state and replicates to all clients.
Category: Derived Interview Bank / Gameplay Framework / Networking
Priority: P0

Q: Constructor, OnConstruction, or BeginPlay?
A: Constructor establishes class defaults/default subobjects; OnConstruction derives instance setup and can rerun; BeginPlay starts runtime gameplay after initialisation.
Category: Derived Interview Bank / Gameplay Framework / Lifecycle
Priority: P0

Q: When should logic live in a subsystem?
A: Use a subsystem for an auto-instanced service whose responsibility naturally shares an Engine, Editor, GameInstance, World, or LocalPlayer lifetime and does not need Actor presence/replication.
Category: Derived Interview Bank / Gameplay Architecture
Priority: P1

Q: Tick, timer, or event?
A: Tick for true frame-dependent work, timers for scheduled/coarse work, and events for state transitions; optimise the work and frequency based on measurement.
Category: Derived Interview Bank / Gameplay Framework / Performance
Priority: P1

Q: `FName`, `FString`, or `FText`?
A: Use FName for identifiers, FString for mutable/general character data, and FText for localisable user-facing display text.
Category: Derived Interview Bank / UE C++ Idioms / Strings
Priority: P0

Q: Compare `std::vector` and `TArray`.
A: Both are contiguous value-owning dynamic sequences; TArray integrates with UE reflection/serialisation APIs and uses UE's relocation/allocator model.
Category: Derived Interview Bank / Standard C++ / UE C++ / Containers
Priority: P0

Q: When would you choose `TSet` or `TMap` over `TArray`?
A: Choose TSet for unique unordered values and TMap for unique-key lookup; choose TArray when sequence/order/contiguous iteration dominates.
Category: Derived Interview Bank / UE C++ Idioms / Containers
Priority: P1

Q: `Cast`, interface, or component lookup?
A: Cast for a legitimate runtime type boundary, interface for a shared capability contract, and component lookup for composed capability owned by an Actor.
Category: Derived Interview Bank / UE C++ Idioms / Architecture
Priority: P0

Q: `NewObject`, `SpawnActor`, or `CreateDefaultSubobject`?
A: NewObject creates runtime UObjects, SpawnActor creates world Actors, and CreateDefaultSubobject defines constructor-time default subobjects.
Category: Derived Interview Bank / UE C++ Idioms / Lifecycle
Priority: P0

Q: Native, multicast, or dynamic delegate?
A: Choose single versus multicast from listener cardinality and dynamic versus native from reflection/Blueprint/serialisation requirements.
Category: Derived Interview Bank / UE C++ Idioms / Delegates
Priority: P0

Q: What belongs in C++ versus Blueprint?
A: Put stable invariants, authority, reusable algorithms, complex lifetime, and proven hotspots in C++; use Blueprint/data for designer-owned composition, tuning, presentation, and bounded extension.
Category: Derived Interview Bank / C++ / Blueprint Architecture
Priority: P0

Q: `BlueprintImplementableEvent` versus `BlueprintNativeEvent`?
A: ImplementableEvent has no native implementation; NativeEvent provides a native `_Implementation` fallback that Blueprint may override.
Category: Derived Interview Bank / C++ / Blueprint Architecture
Priority: P1

Q: What does `BlueprintPure` promise, and what performance trap can it create?
A: It promises no observable state mutation and removes exec pins; consumers may cause repeated evaluation, so expensive pure queries can run more often than expected.
Category: Derived Interview Bank / C++ / Blueprint / Performance
Priority: P1

Q: Why prefer Blueprint read-only state plus validated functions over `BlueprintReadWrite`?
A: Controlled mutation preserves invariants, authority, notifications, replication, and debugging visibility.
Category: Derived Interview Bank / C++ / Blueprint Architecture
Priority: P0

Q: How would you structure input in a modern UE5 project?
A: Use semantic Input Actions, small mode-specific Mapping Contexts on the correct LocalPlayer, cheap modifiers/triggers, and an adapter from local intent to gameplay/UI/ability/server routes.
Category: Derived Interview Bank / Enhanced Input / Architecture
Priority: P1

Q: Input Action versus Input Mapping Context?
A: An Input Action is semantic intent/value; a Mapping Context binds physical controls to actions for one local-player mode or layer.
Category: Derived Interview Bank / Enhanced Input / Concepts
Priority: P1

Q: Input Modifier versus Input Trigger?
A: A modifier transforms the input value; a trigger decides whether the modified value activates the action and which event state is emitted.
Category: Derived Interview Bank / Enhanced Input / Concepts
Priority: P1

Q: Started, Triggered, Completed and Canceled—how do you choose?
A: Choose the event that matches the semantic edge or continuous value: Started for begin edge, Triggered for satisfied/continuous actions, Completed for normal finish and Canceled for interrupted evaluation.
Category: Derived Interview Bank / Enhanced Input / Trigger Events
Priority: P1

Q: How do you debug an Enhanced Input action that does not fire?
A: Check local player, active contexts/priorities, action asset/value type, modifiers, triggers, UI focus/input mode, binding lifecycle and network route.
Category: Derived Interview Bank / Enhanced Input / Debugging
Priority: P1

Q: How should remapping be designed?
A: Treat remapping as per-user/local-player settings with conflict policy, device/platform awareness, persistence, UI prompt updates and fallback defaults.
Category: Derived Interview Bank / Enhanced Input / Remapping
Priority: P2

Q: How should Enhanced Input coordinate with UI or CommonUI?
A: Use one input coordinator so UI activation/focus and gameplay mapping contexts do not compete; menu contexts should block or replace gameplay contexts intentionally.
Category: Derived Interview Bank / Enhanced Input / UI
Priority: P1

Q: How does Enhanced Input fit with GAS?
A: Enhanced Input emits semantic local intent; an adapter maps actions/tags to ability spec handles and press/release events while ASC/server validation owns gameplay truth.
Category: Derived Interview Bank / Enhanced Input / GAS
Priority: P2

Q: How does input become network-safe gameplay?
A: Convert local input into semantic intent, then route it through prediction or an owned server-validated request; never treat the local key event as authority.
Category: Derived Interview Bank / Enhanced Input / Networking
Priority: P1

Q: Why can input fire twice after respawn or screen reopen?
A: Binding or context lifecycle is not idempotent; old bindings/contexts remain while new ones are added.
Category: Derived Interview Bank / Enhanced Input / Debugging
Priority: P1

Q: Authority versus ownership—what is the difference?
A: The server has authority over replicated truth; ownership links an Actor to a PlayerController/connection and controls RPC routing, conditions, and owner relevance.
Category: Derived Interview Bank / Networking / Replication
Priority: P0

Q: Why does a Server RPC fail to execute?
A: Common causes are an unreplicated/unestablished Actor, invalid owning connection, wrong call side/timing, irrelevance/dormancy/channel state, or declaration/implementation error.
Category: Derived Interview Bank / Networking / RPC
Priority: P0

Q: Replicated property or RPC?
A: Use properties for durable current state and RPCs for transient directed events/requests.
Category: Derived Interview Bank / Networking / Architecture
Priority: P0

Q: How should an OnRep function be designed?
A: As a local, order-tolerant, idempotent reaction to newly applied server state—not as authoritative mutation.
Category: Derived Interview Bank / Networking / Properties
Priority: P0

Q: Reliable versus unreliable RPC?
A: Reliable RPCs retransmit until acknowledged and can block later reliable traffic; unreliable RPCs may drop and have weaker ordering, fitting frequent replaceable events.
Category: Derived Interview Bank / Networking / RPC
Priority: P0

Q: What are autonomous and simulated proxies?
A: The owning client has an autonomous proxy that predicts local control; other clients normally see simulated proxies driven by replicated/interpolated state.
Category: Derived Interview Bank / Networking / Roles / Movement
Priority: P0

Q: Relevancy versus dormancy?
A: Relevancy decides per connection whether an Actor should replicate; dormancy keeps an Actor present but skips normal consideration while its state is stable.
Category: Derived Interview Bank / Networking / Optimisation
Priority: P0

Q: Why must a dormant Actor be woken before state mutation?
A: Waking/flush reinitialises replication comparison state; mutating first can make the change invisible or implementation-dependent.
Category: Derived Interview Bank / Networking / Dormancy
Priority: P1

Q: How would you replicate a health/damage system securely?
A: Client submits owned intent; server validates hit/state/rate/resources, mutates authoritative health, and replicates resulting state with OnRep presentation.
Category: Derived Interview Bank / Networking / System Design
Priority: P0

Q: Why is OnRep order not a safe protocol?
A: Different replicated variables' OnRep callbacks have no deterministic relative order, and RPC/property ordering has limited context-specific guarantees.
Category: Derived Interview Bank / Networking / Ordering
Priority: P1

Q: When should you consider Replication Graph or Iris?
A: Consider them when baseline profiling and project constraints show a need for scalable relevance routing or newer replication architecture; neither is automatic merely because the project uses UE5.
Category: Derived Interview Bank / Networking / Scalability
Priority: P2

Q: How do you profile network replication?
A: Capture representative multi-client traffic and rank Actors, properties, and RPCs by bytes/frequency while correlating server processing and player-visible latency.
Category: Derived Interview Bank / Networking / Profiling
Priority: P1

Q: What are the responsibilities of AIController and Pawn?
A: AIController owns decision/control policy; Pawn/Character owns the physical agent and reusable gameplay capabilities.
Category: Derived Interview Bank / AI / Gameplay Framework
Priority: P1

Q: Task, Service, or Decorator in a Behaviour Tree?
A: Tasks perform actions, Decorators gate/observe branches, and Services periodically update context while a branch is active.
Category: Derived Interview Bank / AI / Behaviour Tree
Priority: P1

Q: What do Observer Aborts do?
A: They let observed condition changes abort the current branch and/or interrupt lower-priority branches, enabling event-driven replanning.
Category: Derived Interview Bank / AI / Behaviour Tree
Priority: P1

Q: Behaviour Tree versus StateTree versus FSM?
A: BTs excel at reactive priority/action selection, StateTree at hierarchical state/transition logic with selectors, and FSMs at small explicit modal systems.
Category: Derived Interview Bank / AI / Architecture
Priority: P1

Q: How should AI Perception memory be handled?
A: Separate current perception from known/not-forgotten stimuli and explicit last-known-location/alert memory with age/decay policy.
Category: Derived Interview Bank / AI / Perception
Priority: P1

Q: How does EQS work?
A: A Generator creates candidate items, Contexts define reference frames, Tests filter and score candidates, and the run mode returns suitable result(s).
Category: Derived Interview Bank / AI / EQS
Priority: P1

Q: Why does MoveTo fail?
A: Possession, NavMesh projection, agent settings, goal/path validity, path-follow state, movement component, overlapping requests, or authority can each fail independently.
Category: Derived Interview Bank / AI / Navigation / Debugging
Priority: P0

Q: Pathfinding versus avoidance?
A: Pathfinding computes a route through navigable static space; avoidance locally adjusts velocity around moving agents/obstacles while following it.
Category: Derived Interview Bank / AI / Navigation
Priority: P0

Q: Explain A* at game-interview depth.
A: A* expands nodes by `f=g+h`, combining known path cost with a heuristic estimate to the goal.
Category: Derived Interview Bank / AI / Algorithms
Priority: P1

Q: How do you profile and LOD AI?
A: Measure per-layer work—decision, perception, EQS, navigation, movement—and reduce frequency/fidelity by significance rather than disabling arbitrary code.
Category: Derived Interview Bank / AI / Performance
Priority: P1

Q: What problem do Smart Objects solve?
A: Smart Objects let world objects advertise usable affordances and slots so agents can query, claim, use and release interactions without hardcoding every object type into every agent.
Category: Derived Interview Bank / AI / Smart Objects / Architecture
Priority: P2

Q: Describe the Smart Object lifecycle.
A: Query candidates, claim a specific slot, navigate/validate, use the behavior, then release the claim on completion, failure or abort.
Category: Derived Interview Bank / AI / Smart Objects / Lifecycle
Priority: P2

Q: How does StateTree execute and why is it useful for Smart Object behavior?
A: StateTree selects a root-to-leaf active state path, runs tasks for active states, and uses transitions to reselect states; this fits explicit Acquire/Reach/Use/Exit interaction sequences.
Category: Derived Interview Bank / AI / StateTree
Priority: P2

Q: How do Parameters, Context Data, Evaluators and Global Tasks differ in StateTree?
A: Parameters configure a tree instance, Context Data comes from the execution environment, Evaluators expose runtime-derived data, and Global Tasks run for the tree lifetime.
Category: Derived Interview Bank / AI / StateTree / Data Flow
Priority: P2

Q: How would you debug two NPCs using the same Smart Object slot?
A: Prove whether both only found the same candidate or both actually claimed it, then inspect claim ownership, release timing, authority and slot alignment.
Category: Derived Interview Bank / AI / Smart Objects / Debugging
Priority: P2

Q: How should Smart Object interaction work in multiplayer?
A: Clients can propose/preview interactions, but the server should validate and own durable claim/use outcomes for gameplay-relevant objects.
Category: Derived Interview Bank / AI / Smart Objects / Networking
Priority: P2

Q: When would you choose Smart Objects plus StateTree instead of a Behavior Tree task or simple code?
A: Choose Smart Objects plus StateTree for designer-authored world affordances with explicit interaction lifecycle; choose BT for reactive action selection and simple code for tiny local state.
Category: Derived Interview Bank / AI / Architecture
Priority: P2

Q: How do you profile Smart Object and StateTree activity systems?
A: Count agents, query frequency/radius, candidates, path validations, active StateTrees/tasks/transitions and failed-query retries, then stagger/filter/back off before optimising internals.
Category: Derived Interview Bank / AI / Performance
Priority: P2

Q: What is PCG in Unreal and what is its core data flow?
A: PCG is a plugin framework where spatial input flows through a PCG Graph from a PCG Component, generating and transforming points/attributes into spawned or output content.
Category: Derived Interview Bank / PCG / Architecture
Priority: P2

Q: What are PCG points, density and attributes?
A: Points are candidate spatial records with transform/bounds/density/seed and attributes; density influences probability/weighting, and attributes carry data through the graph.
Category: Derived Interview Bank / PCG / Data Model
Priority: P2

Q: How do PCG metadata domains cause bugs?
A: An attribute can exist on the wrong domain, such as data-level, point-level or element-level metadata, so downstream nodes may not read it.
Category: Derived Interview Bank / PCG / Metadata
Priority: P2

Q: How do graph parameters and graph instances help PCG production workflows?
A: Parameters expose controlled variation, and graph instances reuse graph logic with different overrides, similar in spirit to material instances.
Category: Derived Interview Bank / PCG / Authoring
Priority: P2

Q: How do you debug a PCG graph that produces wrong output?
A: Freeze seed/inputs/parameters, inspect points and attributes after each phase, check metadata domains, disable nodes to isolate the first contradiction, then inspect spawner/output ownership.
Category: Derived Interview Bank / PCG / Debugging
Priority: P2

Q: When should PCG run at runtime?
A: Only when runtime variation is a product requirement and the graph meets deterministic, streaming, memory, hitch, save/load and authority budgets.
Category: Derived Interview Bank / PCG / Runtime Generation
Priority: P2

Q: How do you profile PCG?
A: Separate graph execution from generated output cost, then measure points, attributes, spawned actors/instances, collision/nav/render/memory and runtime hitch behavior.
Category: Derived Interview Bank / PCG / Performance
Priority: P2

Q: How should PCG-generated gameplay objects be handled?
A: PCG can place candidates, but gameplay systems must validate durable state, authority, save/load, replication and invariants.
Category: Derived Interview Bank / PCG / Gameplay Integration
Priority: P2

Q: How would you prepare a UE game for mobile?
A: Define device tiers and budgets, configure device profiles/scalability, package clean builds, test on target devices, and prove frame, memory, thermal, input and store/package behavior.
Category: Derived Interview Bank / Platform / Mobile
Priority: P2

Q: Device Profiles versus Scalability?
A: Device Profiles select platform/device-specific CVars/defaults, while Scalability controls quality groups and tiers that trade fidelity for performance.
Category: Derived Interview Bank / Platform / Performance
Priority: P2

Q: What is certification readiness?
A: It is evidence that the build handles platform-required state transitions and failure cases according to platform-holder rules, not only that gameplay works.
Category: Derived Interview Bank / Platform / Certification
Priority: P2

Q: A packaged Android build crashes on launch. What do you do?
A: Identify the exact artifact/device/toolchain, reproduce from clean install, collect target logs/crash data, classify the failing phase, inspect staged files and symbolicate the crash.
Category: Derived Interview Bank / Platform / Debugging
Priority: P2

Q: How do you design a release artifact contract?
A: Archive the exact package, symbols, manifests, logs, metadata, command parameters and validation results needed to reproduce, diagnose and certify a build.
Category: Derived Interview Bank / Platform / Build Release
Priority: P2

Q: How do you profile platform performance correctly?
A: Use packaged builds on representative hardware, define budgets/scenarios, measure frame percentiles/hitches/memory/thermal over realistic durations, then compare device profiles/scalability.
Category: Derived Interview Bank / Platform / Profiling
Priority: P2

Q: What platform lifecycle failures should a UE game handle?
A: Suspend/resume, background/foreground, input disconnect, account/profile changes, network loss, storage failure, save corruption and entitlement/overlay changes.
Category: Derived Interview Bank / Platform / Lifecycle
Priority: P2

Q: How do you discuss console certification without leaking or bluffing?
A: State that exact requirements are platform-holder confidential, then discuss the engineering categories, evidence workflow and need to use official platform docs.
Category: Derived Interview Bank / Platform / Interview Strategy
Priority: P2

Q: How do you diagnose a frame-rate drop systematically?
A: Define target/scenario, classify the limiting lane with frame-time stats, capture detailed evidence, test one hypothesis, and verify before/after on target hardware.
Category: Derived Interview Bank / Profiling / Optimisation
Priority: P0

Q: Why use frame time instead of only FPS?
A: Milliseconds are additive and map directly to budgets; FPS is reciprocal and averages can hide hitches.
Category: Derived Interview Bank / Profiling / Frame Budget
Priority: P0

Q: What does `stat unit` tell you?
A: High-level Frame, Game, Draw, GPU, RHI, and dynamic-resolution timings for initial bottleneck classification.
Category: Derived Interview Bank / Profiling / UE Tools
Priority: P0

Q: How do you use Unreal Insights effectively?
A: Capture a short reproducible window, compare slow/healthy frames, inspect relevant tracks/scopes/callers, instrument ambiguity, and recapture.
Category: Derived Interview Bank / Profiling / UE Tools
Priority: P0

Q: Tick, timer, callback, or dirty batching?
A: Match cadence and semantics: frame integration uses Tick, scheduled work uses timers, state changes use events, and many same-frame changes may need dirty batching.
Category: Derived Interview Bank / Profiling / Gameplay Performance
Priority: P0

Q: How do you investigate a memory leak or growth?
A: Compare marked timeline points across a repeatable lifecycle, query live/growth allocations, group by callstack/tag, and distinguish cache/warm-up/GC from unbounded retention.
Category: Derived Interview Bank / Profiling / Memory
Priority: P1

Q: How do you diagnose a GC hitch?
A: Correlate the spike with GC, object/reference counts and destruction bursts, then reduce unnecessary UObject churn/retention or schedule measured collection at tolerable boundaries.
Category: Derived Interview Bank / Profiling / UObject / Memory
Priority: P1

Q: CPU-bound versus GPU-bound—how do you prove it?
A: Use frame/thread/GPU timings and controlled sensitivity tests, then inspect the limiting CPU scopes or GPU passes.
Category: Derived Interview Bank / Profiling / Rendering
Priority: P0

Q: When is object pooling a good optimisation?
A: When measured allocation/construction/destruction churn is material and objects can be reset safely within acceptable retained memory.
Category: Derived Interview Bank / Profiling / Architecture
Priority: P1

Q: How do scalability and device profiles differ from optimisation?
A: Optimisation reduces cost for a given result; scalability/device profiles intentionally choose different quality/cost configurations across hardware.
Category: Derived Interview Bank / Profiling / Scalability
Priority: P1

Q: How do you design a reproducible packaged performance benchmark?
A: Pin build, hardware, quality/profile, scenario, warm-up, sample window, metrics and pass/fail rules before comparing results.
Category: Derived Interview Bank / Profiling / Benchmark Design
Priority: P1

Q: CSV Profiler versus Unreal Insights?
A: CSV is lightweight long-run/regression telemetry; Insights is heavier causal trace analysis for frames, tasks, allocations, loads and waits.
Category: Derived Interview Bank / Profiling / Tools
Priority: P2

Q: Memory Insights versus Low-Level Memory Tracker?
A: Memory Insights explains allocation lifetime, growth, churn and callstacks; LLM assigns memory ownership to tags/budgets.
Category: Derived Interview Bank / Profiling / Memory
Priority: P2

Q: How do you prove device profiles and scalability settings actually applied?
A: Capture applied CVars/profile selection on target hardware, then verify performance, memory and gameplay readability at low and high tiers.
Category: Derived Interview Bank / Profiling / Scalability
Priority: P1

Q: Packaged build has an intermittent hitch. What is your workflow?
A: Reproduce in the exact target build, use CSV for long-run detection, then capture a short causal trace around the hitch and classify load, GC, PSO, task, memory or GPU source.
Category: Derived Interview Bank / Profiling / Hitch Diagnosis
Priority: P1

Q: How do you add performance CI without making it unusable?
A: Use tiered, scenario-specific gates with stable metrics, tolerance bands, artifacts and owners; escalate to detailed captures only on failure.
Category: Derived Interview Bank / Profiling / CI
Priority: P2

Q: Walk through a rendered frame in Unreal at interview depth.
A: The game thread updates state, render-side code builds visible-view work, RHI command work is submitted, and the GPU executes geometry, depth/shadow, material, lighting, translucency and post passes that vary by renderer/features.
Category: Derived Interview Bank / Rendering / Pipeline
Priority: P1

Q: Deferred versus forward shading—how do you choose?
A: Deferred offers the default broad feature/dynamic-light path; forward can offer a leaner baseline and MSAA for suitable VR/content, with a different feature and material/light cost model.
Category: Derived Interview Bank / Rendering / Architecture
Priority: P1

Q: Draw calls versus shader, geometry, fill-rate and bandwidth cost?
A: Draw/state submission often pressures CPU render/RHI work; geometry pressures vertex/raster work; shader/fill/overdraw pressures GPU pixels; render targets/textures can pressure memory bandwidth.
Category: Derived Interview Bank / Rendering / Performance
Priority: P0

Q: How do you investigate a GPU regression?
A: Reproduce under controlled settings, prove GPU limitation, name the regressed pass, test a mechanism hypothesis, then verify a requirement-preserving change across targets.
Category: Derived Interview Bank / Rendering / Debugging
Priority: P0

Q: Why can translucency be expensive?
A: Large layered translucent surfaces repeatedly shade/blend pixels, have weaker opaque depth rejection, sorting constraints and renderer-feature limitations.
Category: Derived Interview Bank / Rendering / Materials
Priority: P1

Q: Nanite versus LOD, instancing and HLOD?
A: Nanite virtualises supported geometry detail/visibility; LOD simplifies one asset, instancing represents repeats efficiently, and HLOD proxies distant spatial groups.
Category: Derived Interview Bank / Rendering / Geometry
Priority: P1

Q: How do Virtual Shadow Maps work, and what invalidates their cache?
A: VSMs allocate/render high-resolution virtual shadow-map pages needed by visible pixels and cache them; light or caster changes, new visibility and deformation can require or invalidate pages.
Category: Derived Interview Bank / Rendering / Shadows
Priority: P1

Q: How would you debug a Lumen artefact or performance problem?
A: Classify the visual/cost symptom, identify software versus hardware tracing and quality, inspect Lumen scene/surface representations and named passes, then isolate one representation or trace variable.
Category: Derived Interview Bank / Rendering / Lighting
Priority: P1

Q: What is RDG, and why does declaring resources matter?
A: RDG records passes and declared resource dependencies into a graph so Unreal can validate, schedule, transition, cull and manage transient resources safely.
Category: Derived Interview Bank / Rendering / Engine Internals
Priority: P2

Q: Static switch, dynamic material parameter, or separate material?
A: Dynamic parameters change runtime values in an existing variant; static switches compile variants; separate parents define genuinely different bounded feature families.
Category: Derived Interview Bank / Rendering / Materials / Architecture
Priority: P1

Q: Shader permutation, runtime shader cost, or PSO hitch?
A: Shader permutations are compiled variants, runtime shader cost is GPU work, and PSO hitches are expensive first-time pipeline-state creation/binding coverage problems.
Category: Derived Interview Bank / Rendering / Shaders
Priority: P1

Q: How do you build a safe custom global shader or RDG pass?
A: Register/load shader code correctly, bound permutations, declare RDG parameters/resources, respect deferred execution lifetimes and validate output with RDG tools or GPU capture.
Category: Derived Interview Bank / Rendering / Shader Programming
Priority: P2

Q: What is the PSO cache workflow for first-use hitches?
A: Reproduce first-use hitches in packaged target builds, collect/merge/package/precache representative PSOs, and verify cold-cache runs across rare content paths.
Category: Derived Interview Bank / Rendering / PSO Caches
Priority: P1

Q: What does the mesh drawing pipeline explain for gameplay engineers?
A: Gameplay primitives become render-side scene proxies, mesh batches and pass-specific draw commands; material slots, dynamic state and render-state invalidation can create CPU render cost.
Category: Derived Interview Bank / Rendering / Mesh Drawing
Priority: P2

Q: How should you budget render targets and scene captures?
A: Budget resolution, format, update cadence, active count, view complexity, mips/copies/readbacks and consumers; assume scene captures are extra views until proven cheaper.
Category: Derived Interview Bank / Rendering / Render Targets
Priority: P1

Q: When should you escalate to RenderDoc or platform GPU tools?
A: After Unreal pass timings name the problem area but you need exact pipeline state, resources, shaders, barriers, occupancy clues or platform-specific behaviour.
Category: Derived Interview Bank / Rendering / GPU Capture
Priority: P2

Q: Component versus inheritance?
A: Inheritance fits a stable substitutable “is-a” protocol; Components fit independently varying reusable capabilities with an explicit owner contract.
Category: Derived Interview Bank / Architecture / Composition
Priority: P0

Q: Direct call, delegate/Observer, or queued event?
A: Direct calls express required immediate collaboration; delegates synchronously notify local observers; queues decouple processing time and require delivery/backpressure semantics.
Category: Derived Interview Bank / Patterns / Communication
Priority: P0

Q: State versus Strategy?
A: State models runtime modes/transitions; Strategy substitutes an algorithm/policy behind a stable contract.
Category: Derived Interview Bank / Patterns / Behaviour
Priority: P1

Q: What is the risk of using a Subsystem as a Service Locator?
A: Subsystems scope service lifetime, but ubiquitous lookup still hides dependencies, couples tests to engine context and encourages god services.
Category: Derived Interview Bank / Architecture / Services
Priority: P1

Q: What makes a system data-driven in Unreal?
A: Stable code interprets authored, validated definitions while mutable runtime state and presentation remain separate and definitions have stable loading identity.
Category: Derived Interview Bank / Architecture / Data-Driven Design
Priority: P0

Q: Design a multiplayer-ready health and damage system.
A: A reusable authoritative capability validates structured damage, commits bounded state/death once, replicates durable state and emits local committed results for presentation.
Category: Derived Interview Bank / System Design / Health
Priority: P0

Q: Design an inventory and equipment system.
A: Separate item definitions, mutable stack/instance state, container placement and public equipment projection behind authoritative transactional operations.
Category: Derived Interview Bank / System Design / Inventory
Priority: P0

Q: Design an interaction system.
A: Separate local discovery/selection and read-only prompt query from an owning-client request that the server revalidates and executes.
Category: Derived Interview Bank / System Design / Interaction
Priority: P0

Q: Design a quest/objective system.
A: Versioned quest definitions contain stable objective IDs/conditions; authoritative instances contain progress and consume committed domain events plus current-state queries.
Category: Derived Interview Bank / System Design / Quests
Priority: P1

Q: How do you make gameplay systems save/load friendly?
A: Save versioned authoritative records with stable identifiers, migrate before restore, resolve/create owners, apply state without normal side effects, rebuild derived data and notify once.
Category: Derived Interview Bank / System Design / Persistence
Priority: P0

Q: When is Object Pool useful, and when is it harmful?
A: It helps measured expensive allocation/registration churn when objects reset safely; it harms through retained peak memory, stale state/handles and lifecycle/network complexity.
Category: Derived Interview Bank / Patterns / Performance
Priority: P1

Q: How do gameplay systems stay modular under production pressure?
A: Preserve bounded ownership and contracts at real change axes, validate data, instrument transactions, refactor from observed coupling and resist speculative frameworks.
Category: Derived Interview Bank / Architecture / Production
Priority: P0

Q: Hard versus soft asset reference?
A: A hard reference joins eager load/dependency closure; a soft reference stores indirect identity for deferred resolution/loading and does not itself own residency.
Category: Derived Interview Bank / Assets / References
Priority: P0

Q: How do you make async asset loading lifetime-safe?
A: Scope each request to an owner/generation, validate owner/world at completion, handle partial failure/cancellation and retain assets explicitly for required residency.
Category: Derived Interview Bank / Assets / Async Loading
Priority: P0

Q: Asset Registry versus Asset Manager?
A: Registry queries unloaded asset metadata; Asset Manager adds Primary identity plus managed load, audit, cook and chunk policy.
Category: Derived Interview Bank / Assets / Discovery
Priority: P1

Q: What are Primary Assets and bundles?
A: Primary Assets have stable managed IDs/rules; named bundles group soft dependencies for purpose-specific loading such as UI versus gameplay.
Category: Derived Interview Bank / Assets / Asset Manager
Priority: P1

Q: Build, cook, stage, package, deploy, and run?
A: Build compiles code; cook creates target asset data; stage assembles outputs; package bundles distribution; deploy transfers to target; run executes it.
Category: Derived Interview Bank / Assets / Packaging
Priority: P1

Q: Asset works in editor but is missing in a packaged build—what do you do?
A: Verify exact identity and cook inclusion/management chain in packaged logs/manifests before changing timing or cooking everything.
Category: Derived Interview Bank / Assets / Debugging
Priority: P0

Q: What are redirectors, and how should a team handle them?
A: Rename/move leaves an old-path redirector; fix-up resaves referencers to the new asset and removes the redirector when all can be updated.
Category: Derived Interview Bank / Assets / Content Maintenance
Priority: P1

Q: What is DDC, and should it be source controlled?
A: DDC caches regenerable platform/settings-derived asset data and normally should not be authoritative source-controlled content.
Category: Derived Interview Bank / Assets / Build Performance
Priority: P1

Q: Level streaming versus World Partition?
A: Level streaming explicitly loads/unloads level packages; World Partition spatially divides a persistent UE5 world into cells driven by streaming sources, with Data Layer/OFPA/HLOD integration.
Category: Derived Interview Bank / Assets / World Streaming
Priority: P1

Q: How do you diagnose an asset-loading hitch?
A: Mark request-to-first-use stages, compare cold/warm packaged target captures, inspect I/O/package/object/resource/callback work and dependency closure.
Category: Derived Interview Bank / Assets / Profiling
Priority: P0

Q: What do UBT, UHT, the compiler, and linker each do?
A: UBT constructs the target/module build graph, UHT generates reflection glue, the compiler builds translation units, and the linker resolves symbols into binaries.
Category: Derived Interview Bank / Build / Toolchain
Priority: P0

Q: Target versus module in Unreal?
A: A target describes an application/configuration composition; a module is a separately built functionality/API unit in its dependency graph.
Category: Derived Interview Bank / Build / Architecture
Priority: P1

Q: Public versus Private module dependency?
A: Public dependencies are exposed by public headers/API to consumers; private dependencies are implementation-only and should not propagate.
Category: Derived Interview Bank / Build / Modules
Priority: P0

Q: Why can code compile in unity builds but fail non-unity or on CI?
A: Unity translation units can supply accidental neighbouring includes/definitions and hide missing IWYU or ODR problems.
Category: Derived Interview Bank / Build / C++ Dependencies
Priority: P1

Q: Header is visible, but linker reports unresolved external—why?
A: Declaration compiled, but the exact definition/export or owning binary dependency is absent for that target/configuration.
Category: Derived Interview Bank / Build / Linking
Priority: P0

Q: Runtime module versus Editor module?
A: Runtime contains code/types valid in packaged targets; Editor contains UnrealEd/tool UI/asset authoring extensions and may depend on Runtime, never vice versa.
Category: Derived Interview Bank / Build / Architecture
Priority: P0

Q: What belongs in `.uplugin`, `.uproject`, Build.cs, and Target.cs?
A: Descriptors declare discoverable project/plugin/modules/compatibility; Build.cs defines module build contracts; Target.cs defines application target composition/global rules.
Category: Derived Interview Bank / Build / Configuration
Priority: P1

Q: When is Live Coding safe, and when should you restart?
A: It is best for implementation changes; reflected layout, constructors/defaults/subobjects, static caches/destructors and build graph changes require clean restart/rebuild validation.
Category: Derived Interview Bank / Build / Iteration
Priority: P0

Q: Module startup/shutdown registration—what can go wrong?
A: Registrations can duplicate or outlive modules across reload; shutdown order may make dependencies unavailable, so startup/unregister ownership must be symmetric and guarded.
Category: Derived Interview Bank / Build / Editor Extensions
Priority: P1

Q: Editor Utility Widget, Python, C++ editor module, or commandlet?
A: Choose by audience and execution: interactive designer UI, pipeline scripting, durable native integration, or headless CI batch processing.
Category: Derived Interview Bank / Editor Tools / Architecture
Priority: P1

Q: How do you design a safe batch asset modification tool?
A: Separate analysis from mutation and provide dry run, preflight, transactions, source-control handling, cancellation, idempotence, explicit saves and structured reports.
Category: Derived Interview Bank / Editor Tools / System Design
Priority: P0

Q: Validation versus automatic repair?
A: Validation deterministically reports invariant violations; repair is an explicit transactional operation with preview and recovery.
Category: Derived Interview Bank / Editor Tools / Validation
Priority: P1

Q: AutomationTool versus BuildGraph?
A: AutomationTool is Unreal's command-line automation runner; BuildGraph is a declarative graph format for larger repeatable build-farm pipelines.
Category: Derived Interview Bank / Build / Release Automation
Priority: P2

Q: How would you design release pipeline gates for an Unreal project?
A: Gate progressively from compile and validation to clean cook/package, launch smoke, asset/runtime dependency proof, symbols and crash reporting.
Category: Derived Interview Bank / Build / CI
Priority: P2

Q: Packaged build crashes but the editor works. What is your workflow?
A: Reproduce the exact packaged target, preserve logs/symbols/build metadata, classify whether the failure is content, module, staged dependency, config or platform runtime, then fix and rerun the gate.
Category: Derived Interview Bank / Build / Packaged Runtime Debugging
Priority: P1

Q: How do you prove a third-party runtime dependency is release-ready?
A: Prove compile/link/runtime staging for every target platform/configuration from a clean machine, including licensing, symbols and failure diagnostics.
Category: Derived Interview Bank / Build / Third-Party Integration
Priority: P2

Q: How should crash reporting and symbols fit into a release pipeline?
A: Treat symbols, build metadata, logs and forced-crash symbolication as release artifacts, not optional post-release cleanup.
Category: Derived Interview Bank / Build / Operations
Priority: P2

Q: How would you design a practical Unreal CI matrix?
A: Use tiers: fast pre-submit confidence, nightly clean proof, release-candidate artifact proof and platform/certification jobs where needed.
Category: Derived Interview Bank / Build / CI
Priority: P2

Q: Why use ECS or MassEntity at all?
A: To process large populations with shared data shapes in contiguous archetype/chunk batches and decouple simulation fidelity from expensive object representation.
Category: Derived Interview Bank / MassEntity / Architecture
Priority: P2

Q: Actor/Component architecture versus MassEntity?
A: Actors are rich independent UObject world identities; Mass entities are lightweight compositions processed in batches, often with Actor/ISM/no representation by LOD.
Category: Derived Interview Bank / MassEntity / Architecture
Priority: P2

Q: Fragment, entity, archetype, and chunk?
A: Fragment is data; entity is manager-local identity plus composition; archetype groups identical composition; chunks store archetype entities with contiguous fragment columns.
Category: Derived Interview Bank / MassEntity / Core Model
Priority: P2

Q: Mass tag versus Boolean fragment field?
A: A tag changes composition and enables query grouping; a Boolean stays in data and avoids migration but requires filtering/branching.
Category: Derived Interview Bank / MassEntity / Data Design
Priority: P2

Q: Shared fragment versus chunk fragment versus entity fragment?
A: Entity fragment is per entity; shared fragment is shared by a configured group; chunk fragment is one value per storage chunk.
Category: Derived Interview Bank / MassEntity / Data Design
Priority: P2

Q: How do Mass queries and processors work?
A: A processor declares read/write/presence requirements in an EntityQuery, which caches matching archetypes and supplies chunk fragment views for batch execution.
Category: Derived Interview Bank / MassEntity / Processing
Priority: P2

Q: Why defer structural changes in Mass?
A: Composition changes move storage/archetype and can invalidate active iteration; command buffers batch them at a safe flush point.
Category: Derived Interview Bank / MassEntity / Processing
Priority: P2

Q: What is a Mass Trait?
A: A Trait is an entity-template authoring recipe that adds/configures fragments/tags/shared data for a capability; processors provide runtime logic.
Category: Derived Interview Bank / MassEntity / Authoring
Priority: P2

Q: Visual LOD versus simulation LOD in Mass?
A: Visual LOD selects Actor/ISM/none representation; simulation LOD changes calculation frequency/fidelity; they solve different budgets.
Category: Derived Interview Bank / MassEntity / LOD
Priority: P2

Q: A Mass entity is not processed—how do you debug it?
A: Verify World/handle, exact composition, query requirements/registration, processor activation/order and deferred-command flush, then inspect matched archetypes in target tools.
Category: Derived Interview Bank / MassEntity / Debugging
Priority: P2

Q: How do you profile Mass fairly against Actors?
A: Hold behaviour/visual fidelity constant and separate simulation, structural, bridge and representation costs across population scales.
Category: Derived Interview Bank / MassEntity / Performance
Priority: P2

Q: What should a broad engineer know about Mass replication?
A: It is a specialised, evolving MassGameplay path using server authority and viewer relevance/LOD; custom data and stable identity require target-specific C++ design.
Category: Derived Interview Bank / MassEntity / Networking
Priority: P3

Q: When would you choose GAS over a bespoke ability Component?
A: Choose GAS when interacting abilities need shared tags, costs, cooldowns, effects, attributes, cancellation and prediction; keep a bespoke Component for a small bounded rule set.
Category: Derived Interview Bank / GAS / Architecture
Priority: P2

Q: Explain ASC owner versus avatar and where you would put the ASC.
A: Owner logically owns durable ASC state; avatar is the current physical actor. Pawn placement is simple, while PlayerState placement can preserve state across respawn.
Category: Derived Interview Bank / GAS / Lifetime
Priority: P2

Q: Gameplay Ability class, spec, instance and handle—what differs?
A: Class defines behaviour; a spec is one runtime grant in an ASC; an instance holds activation/per-owner state according to policy; a handle identifies the grant.
Category: Derived Interview Bank / GAS / Abilities
Priority: P2

Q: Walk through ability activation, commit, cancellation and ending.
A: Request, eligibility, activation, deliberate commit, asynchronous execution, cancel/complete and one explicit end/cleanup path.
Category: Derived Interview Bank / GAS / Abilities
Priority: P2

Q: How do GAS Gameplay Tags differ from Boolean state?
A: Tags are registered hierarchical predicates with source counts and query semantics; a Boolean is one value owned by one state path.
Category: Derived Interview Bank / GAS / Tags
Priority: P2

Q: Explain attribute base/current values and invariant handling.
A: Base is the underlying value; current is the evaluated value after active modifiers. Attribute Sets declare values, but invariants must cover all mutation/replication paths.
Category: Derived Interview Bank / GAS / Attributes
Priority: P2

Q: Gameplay Effect definition, spec and active effect—what differs?
A: Definition is data; spec is mutable application data with level/context/captures; applying a non-instant spec creates retained active-effect state.
Category: Derived Interview Bank / GAS / Effects
Priority: P2

Q: Design a stacking buff completely.
A: Define aggregation owner, limit, overflow, duration refresh, expiration, period, tags/cues and UI—not only the stack limit.
Category: Derived Interview Bank / GAS / Effects
Priority: P2

Q: Ability Task versus Gameplay Cue versus Gameplay Effect?
A: Task coordinates asynchronous ability work, cue presents cosmetics, and effect changes/queryable gameplay state.
Category: Derived Interview Bank / GAS / Integration
Priority: P2

Q: What does Local Predicted mean in GAS?
A: The owning client performs supported provisional work under a prediction key while the server validates and confirms/rejects authoritative execution.
Category: Derived Interview Bank / GAS / Networking
Priority: P2

Q: Full, Mixed and Minimal ASC effect replication modes?
A: Full sends full GE information to all; Mixed sends full to owner/autonomous and minimal to simulated proxies; Minimal sends minimal GE information.
Category: Derived Interview Bank / GAS / Networking
Priority: P2

Q: An ability will not activate on the owning client—how do you debug it?
A: Trace ASC/actor info, grant/spec, ownership/net policy, failure tags, owned tags, cost/cooldown, triggers and blockers on the failing machine.
Category: Derived Interview Bank / GAS / Debugging
Priority: P2

Q: How do you choose a data structure in an interview?
A: Derive it from dominant operations, bounds, ordering, stability, memory/layout and determinism after stating a correct baseline.
Category: Derived Interview Bank / Algorithms / Problem Solving
Priority: P0

Q: Explain worst-case, average/expected and amortised complexity.
A: Worst bounds any input; expected averages under stated assumptions/distribution; amortised spreads occasional expensive operations over a sequence.
Category: Derived Interview Bank / Algorithms / Complexity
Priority: P0

Q: Dynamic array versus linked list?
A: Prefer contiguous arrays for locality/index/iteration; a list wins only when stable nodes and frequent known-position splicing outweigh allocation/pointer costs.
Category: Derived Interview Bank / Algorithms / Containers
Priority: P0

Q: What can go wrong with a hash map?
A: Poor hash/equality, collisions/load, rehash invalidation/spikes, mutable keys, memory slack and non-deterministic iteration.
Category: Derived Interview Bank / Algorithms / Hashing
Priority: P0

Q: How does a heap-backed priority queue work?
A: A heap keeps parent priority over children, giving constant top and logarithmic push/pop without globally sorting all elements.
Category: Derived Interview Bank / Algorithms / Heaps
Priority: P0

Q: Binary search—what are its preconditions and common bugs?
A: Search a sorted/partitioned range with a consistent comparator; use half-open bounds and test duplicates, empty and insertion endpoints.
Category: Derived Interview Bank / Algorithms / Searching
Priority: P0

Q: BFS versus DFS?
A: BFS uses FIFO and finds minimum edge-count paths in unweighted graphs; DFS uses a stack and suits reachability, components, cycles and backtracking.
Category: Derived Interview Bank / Algorithms / Graphs
Priority: P0

Q: Explain Dijkstra and A* and their correctness constraints.
A: Dijkstra expands lowest known non-negative path cost; A* adds a remaining-cost heuristic and preserves optimality under suitable heuristic conditions.
Category: Derived Interview Bank / Algorithms / Pathfinding
Priority: P0

Q: How do you topologically sort dependencies and detect cycles?
A: For a DAG, repeatedly emit zero-indegree nodes and remove their outgoing edges; fewer than V outputs proves a cycle.
Category: Derived Interview Bank / Algorithms / Graphs
Priority: P1

Q: What problem does union–find solve?
A: It maintains connected-component partitions under unions, with near-constant amortised Find/Union using path compression and union by size/rank.
Category: Derived Interview Bank / Algorithms / Connectivity
Priority: P1

Q: Design weighted random selection.
A: Sample uniformly over total weight and select the containing prefix interval; optimise with prefix search only when repeated draws justify preprocessing.
Category: Derived Interview Bank / Algorithms / Sampling
Priority: P1

Q: How would you accelerate neighbour queries for 10,000 moving agents?
A: Keep all-pairs as oracle, then try a tuned uniform grid/spatial hash, exact-filter candidates and measure update/query plus clustered worst cases.
Category: Derived Interview Bank / Algorithms / Spatial Structures
Priority: P1

Q: Grid/spatial hash versus quadtree, k-d tree or BVH?
A: Select from dimension, extent, size distribution, motion and query/update types; no structure dominates every workload.
Category: Derived Interview Bank / Algorithms / Spatial Structures
Priority: P1

Q: Broadphase versus narrowphase?
A: Broadphase cheaply returns conservative candidate pairs; narrowphase performs accurate shape tests on candidates.
Category: Derived Interview Bank / Algorithms / Collision
Priority: P1

Q: How do you debug and prove an optimised algorithm correct?
A: State invariant, minimise counterexamples and differential/property-test the optimised result against a simple oracle on generated adversarial inputs.
Category: Derived Interview Bank / Algorithms / Testing
Priority: P0

Q: What does the dot product tell you in gameplay code?
A: It measures magnitude-scaled alignment; for unit vectors it is cosine of their angle.
Category: Derived Interview Bank / Game Maths / Vectors
Priority: P0

Q: What does the cross product tell you, and what can go wrong?
A: It returns a perpendicular with sine/area-scaled magnitude; operand order and Unreal's handedness determine sign.
Category: Derived Interview Bank / Game Maths / Vectors
Priority: P0

Q: Point versus vector transform, and local versus world space?
A: A point includes translation; a vector/direction does not. Local coordinates are relative to a parent/object basis; world coordinates use the level basis/origin.
Category: Derived Interview Bank / Game Maths / Transforms
Priority: P0

Q: Why does transform order matter?
A: Transform composition is non-commutative; rotate-then-translate generally differs from translate-then-rotate.
Category: Derived Interview Bank / Game Maths / Matrices
Priority: P0

Q: Rotator versus quaternion, and what is gimbal lock?
A: Rotators are readable ordered Euler angles with singularities; unit quaternions compose/interpolate arbitrary orientation without that Euler parameterisation singularity.
Category: Derived Interview Bank / Game Maths / Rotation
Priority: P0

Q: Lerp, Slerp, constant-speed movement and smoothing—how do they differ?
A: Lerp parametrises a straight segment; Slerp follows an orientation sphere arc; repeated current-target lerp eases, while move-towards enforces speed.
Category: Derived Interview Bank / Game Maths / Interpolation
Priority: P0

Q: Derive closest point on a segment.
A: Project `P-A` onto `B-A`, divide by segment length squared, clamp parameter to `[0,1]`, then evaluate `A+t(B-A)`.
Category: Derived Interview Bank / Game Maths / Geometry
Priority: P0

Q: How do ray–plane and ray–sphere tests work?
A: Substitute the parametric ray into the primitive equation, solve for t, then filter by ray/segment interval and degeneracies.
Category: Derived Interview Bank / Game Maths / Intersections
Priority: P0

Q: Explain ray–AABB using slabs.
A: Compute each axis's ray interval inside its min/max slab; intersect intervals and accept when enter≤exit and exit is not behind.
Category: Derived Interview Bank / Game Maths / Intersections
Priority: P1

Q: Walk from world position to screen pixel.
A: World→view→clip, divide homogeneous coordinates by w to NDC, then map NDC to viewport pixels/depth convention.
Category: Derived Interview Bank / Game Maths / Rendering Spaces
Priority: P1

Q: Why do normals and colour textures need special treatment?
A: Normals are directions requiring special non-uniform-scale transform; lighting colours need linear arithmetic while normal/data textures must not undergo colour gamma decoding.
Category: Derived Interview Bank / Game Maths / Rendering
Priority: P1

Q: How do you make vector/intersection maths numerically robust?
A: State units/scale, use domain tolerances, guard degenerates/division, clamp inverse-trig inputs, reject non-finite values and test boundary cases.
Category: Derived Interview Bank / Game Maths / Numerical Robustness
Priority: P0

Q: How should gameplay communicate with an Animation Blueprint?
A: Gameplay/movement expose a compact stable presentation snapshot and explicit action requests; animation derives pose and reports correlated cosmetic/timing events without owning durable truth.
Category: Derived Interview Bank / Animation / Architecture
Priority: P1

Q: EventGraph versus AnimGraph and animation update versus evaluation?
A: EventGraph is ordinary game-thread Blueprint update logic; AnimGraph is relevance-driven pose dataflow that can update/evaluate on workers.
Category: Derived Interview Bank / Animation / Runtime
Priority: P1

Q: Animation state machine versus montage?
A: State machines model persistent pose regimes with continuously evaluated transitions; montages model directed discrete actions with sections, slots and cancellation.
Category: Derived Interview Bank / Animation / Pose Architecture
Priority: P1

Q: How do Blend Spaces and Sync Groups solve different problems?
A: Blend Spaces select/interpolate poses from parameter axes; Sync Groups align playback phase of related animations during blending.
Category: Derived Interview Bank / Animation / Locomotion
Priority: P1

Q: Why can `Montage_Play` succeed but no montage is visible?
A: The call can start an instance while the required Slot is absent/irrelevant, conflicted, overwritten or incompatible with the active pose graph.
Category: Derived Interview Bank / Animation / Debugging
Priority: P1

Q: Are Animation Notifies reliable gameplay events?
A: No; they are timeline events affected by relevance, blending, looping, sync, throttling and network audience, so authoritative gameplay must validate and have durable fallback.
Category: Derived Interview Bank / Animation / Gameplay Integration
Priority: P1

Q: In-place locomotion versus root motion?
A: In-place lets CharacterMovement own displacement; root motion extracts authored root displacement, trading control/network simplicity for grounded authored motion.
Category: Derived Interview Bank / Animation / Movement
Priority: P1

Q: How do you diagnose foot sliding?
A: Correlate capsule speed, clip stride/play rate, Blend Space axes, sync phase, retarget proportions, root-motion mode, network smoothing and IK.
Category: Derived Interview Bank / Animation / Debugging
Priority: P1

Q: How do you debug a bad IK retarget?
A: Verify source/target skeletons, retarget root/chains/mapping and matching retarget poses before tuning offsets or IK.
Category: Derived Interview Bank / Animation / Retargeting
Priority: P2

Q: What are Linked Anim Layers for?
A: They define semantic pose extension points that runtime animation classes can override, supporting modular equipment/vehicle logic and asset lifetime boundaries.
Category: Derived Interview Bank / Animation / Modularity
Priority: P2

Q: How do you profile and optimise animation for many characters?
A: Separate game-thread data gathering, worker pose evaluation, skeletal finalisation, skinning, IK queries, assets and cosmetics; then reduce work by thread safety, relevance and significance/LOD.
Category: Derived Interview Bank / Animation / Performance
Priority: P1

Q: Motion Warping and IK—what do they solve, and what do they not solve?
A: Motion Warping adapts root-motion windows to a runtime target; IK adjusts joints toward goals. Neither validates gameplay reachability or replaces collision/authority.
Category: Derived Interview Bank / Animation / Procedural
Priority: P2

Q: Explain Unreal collision filtering without saying “set the preset correctly”.
A: A PrimitiveComponent's collision-enabled mode, Object Type and response to the other object's or trace's channel jointly determine Ignore, Overlap or Block; a profile packages that policy.
Category: Derived Interview Bank / Physics / Collision Filtering
Priority: P1

Q: Does a blocking collision guarantee an Event Hit?
A: No; blocking response and movement/simulation notification settings/path are separate, and callbacks are not an exactly-once gameplay protocol.
Category: Derived Interview Bank / Physics / Events
Priority: P1

Q: Trace by channel versus query by object type?
A: Channel asks each target how it responds to a semantic query; object query selects targets by their intrinsic Object Types.
Category: Derived Interview Bank / Physics / Scene Queries
Priority: P1

Q: Line trace, shape sweep or overlap?
A: Line samples a zero-volume segment; sweep moves a volume along a segment; overlap tests a volume at one pose.
Category: Derived Interview Bank / Physics / Scene Queries
Priority: P1

Q: `Location` versus `ImpactPoint`, and `Normal` versus `ImpactNormal` in a swept hit?
A: Location is the swept shape's safe location; ImpactPoint is contact on hit geometry. Normal relates to the swept shape/contact; ImpactNormal describes hit surface.
Category: Derived Interview Bank / Physics / Hit Results
Priority: P1

Q: Simple versus complex collision?
A: Simple uses primitives/convex gameplay shapes suited to robust queries/simulation; complex uses render triangles for detailed queries with greater cost/restrictions.
Category: Derived Interview Bank / Physics / Geometry
Priority: P1

Q: CharacterMovement versus rigid-body movement?
A: CharacterMovement is capsule/sweep-based authored movement with floor modes and mature prediction; rigid bodies are force/contact-driven solver state.
Category: Derived Interview Bank / Physics / Movement Architecture
Priority: P1

Q: Force versus impulse versus setting velocity?
A: Force accumulates over simulation time, impulse changes momentum immediately, and direct velocity imposes state while potentially overriding solver continuity.
Category: Derived Interview Bank / Physics / Rigid Bodies
Priority: P1

Q: What does physics substepping solve, and what does it not solve?
A: It splits long frame deltas into smaller solver steps for stability/accuracy at CPU/bookkeeping cost; it does not guarantee determinism or fix bad geometry/filtering/constraints.
Category: Derived Interview Bank / Physics / Timestep
Priority: P1

Q: How do you debug an unstable physics constraint?
A: Inspect body/constraint local frames, mass/inertia, limits/drives/collision and timestep before increasing solver strength/iterations.
Category: Derived Interview Bank / Physics / Constraints
Priority: P1

Q: Describe a safe animation-to-ragdoll-and-back transition.
A: Transfer ownership in an ordered state transition among gameplay, capsule/CharacterMovement, Skeletal Mesh Physics Asset and network presentation.
Category: Derived Interview Bank / Physics / Ragdoll
Priority: P1

Q: Why is multiplayer rigid-body physics difficult, and what modes should you know?
A: Contacts diverge under latency; server authority must reconcile client simulation. Know default correction and awareness of Predictive Interpolation/Resimulation only with target-version verification.
Category: Derived Interview Bank / Physics / Networking
Priority: P2

Q: How do you profile a physics-heavy scene?
A: Separate scene queries, active bodies/broadphase contacts, solver/constraints, callbacks, substeps, Physics Assets and network history/corrections.
Category: Derived Interview Bank / Physics / Performance
Priority: P1

Q: Explain `OnInitialized`, `Construct`, activation and `Destruct` without calling them UI `BeginPlay`/`EndPlay`.
A: They correspond to different lifetimes: UObject instance, underlying Slate representation and screen activity; `Construct` can repeat.
Category: Derived Interview Bank / UI / Lifecycle
Priority: P1

Q: How should gameplay state reach a HUD?
A: Through a local-player presentation projection: subscribe, render an initial snapshot, apply deltas, and keep gameplay authoritative.
Category: Derived Interview Bank / UI / Architecture
Priority: P1

Q: Property binding, FieldNotify event or widget Tick—how do you choose?
A: Prefer bounded push updates for discrete changes, Tick for genuine continuous presentation, and measure pull bindings in their visible hierarchy.
Category: Derived Interview Bank / UI / Update Performance
Priority: P1

Q: How does Unreal UI layout decide a widget's size and position?
A: Children compute Desired Size bottom-up; parents arrange allotted geometry top-down according to panel-slot constraints.
Category: Derived Interview Bank / UI / Layout
Priority: P1

Q: Why can a visible enabled button fail to receive a click?
A: Another widget/parent visibility mode may own the hit-test path, geometry may differ from appearance, or the wrong user/input layer may be active.
Category: Derived Interview Bank / UI / Hit Testing
Priority: P1

Q: Design controller focus/navigation for a modal screen.
A: Define focus per local user, a deterministic initial target, navigation graph, and restoration when the modal closes or the target disappears.
Category: Derived Interview Bank / UI / Input and Focus
Priority: P1

Q: What problem does CommonUI solve, and when would you not adopt it?
A: It structures complex multiplatform menu layers, activation, action routing/glyphs and navigation; small/simple UI may not justify its plugin conventions.
Category: Derived Interview Bank / UI / CommonUI
Priority: P2

Q: Why does a virtualised inventory row show the wrong icon or selection?
A: Entry widgets are recycled; old state, subscriptions or async callbacks survived rebinding to a new item.
Category: Derived Interview Bank / UI / Lists and Async
Priority: P1

Q: When should UI assets be hard referenced versus loaded asynchronously?
A: Hard-reference small always-resident UI foundations; soft/async-load large or optional content when startup closure/residency matters, with placeholders and stale-result guards.
Category: Derived Interview Bank / UI / Assets
Priority: P1

Q: Screen-space marker or world-space `WidgetComponent`?
A: Choose from occlusion/diegesis/input/readability and scaling; world widgets add scene/render-target work and are not a free nameplate solution.
Category: Derived Interview Bank / UI / World Widgets
Priority: P2

Q: How do you make a UI localisation- and accessibility-ready?
A: Preserve `FText`, use culture-aware formatting and flexible layouts; test expansion/RTL/fonts plus contrast, scaling, redundant cues, labels and all input paths.
Category: Derived Interview Bank / UI / Product Quality
Priority: P1

Q: How do you profile and optimise an expensive UMG screen?
A: Classify model/binding, update, layout, paint, draw submission, asset and GPU/overdraw costs, then optimise the measured stage.
Category: Derived Interview Bank / UI / Performance
Priority: P1

Q: Sound Wave, Sound Cue or MetaSound Source?
A: Wave is recorded content/properties, Cue is an authored Sound Node behaviour graph, and MetaSound is a procedural DSP rendering graph with sample-accurate internal control.
Category: Derived Interview Bank / Audio / Source Design
Priority: P1

Q: Fire-and-forget sound versus owned Audio Component?
A: Use one-shot when no later control/follow is needed; own a Component for attachment, loop, fade, parameters, callbacks or explicit stop/reuse.
Category: Derived Interview Bank / Audio / Lifetime
Priority: P1

Q: Explain attenuation beyond “sound gets quieter with distance”.
A: It packages listener-relative volume shape/curve plus spatialisation, focus, occlusion, air absorption, priority and environment-send policy.
Category: Derived Interview Bank / Audio / Spatialisation
Priority: P1

Q: What does Sound Concurrency solve?
A: It limits semantic groups of active instances and resolves new requests by owner scope, retrigger and reject/replace/scale/steal policy.
Category: Derived Interview Bank / Audio / Voice Management
Priority: P1

Q: Sound Class/Sound Mix versus Submix?
A: Classes categorise/control source properties and temporary mixes; Submixes route and process actual combined audio buffers through DSP.
Category: Derived Interview Bank / Audio / Mixing and Routing
Priority: P1

Q: What problem does Quartz solve, and what does it not solve?
A: It schedules rendering to sample/musical time across game/audio buffer boundaries; it does not remove device latency or make gameplay/network state deterministic.
Category: Derived Interview Bank / Audio / Timing
Priority: P2

Q: Why can a loaded Sound Wave still hitch or miss its first playback?
A: The UObject can be loaded while required compressed chunks/decoder data are not resident or ready in the stream cache.
Category: Derived Interview Bank / Audio / Streaming and Memory
Priority: P1

Q: How should audio work in multiplayer?
A: Replicate authoritative gameplay state or bounded events; each client renders relevant local audio and deduplicates predicted versus confirmed feedback.
Category: Derived Interview Bank / Audio / Networking
Priority: P1

Q: A sound request happened but nothing was heard. What is your workflow?
A: Trace source/component, listener/attenuation, admission, gain/routing, data/render and output device in order.
Category: Derived Interview Bank / Audio / Debugging
Priority: P1

Q: How do you profile and optimise a busy audio scene?
A: Separate request/update CPU, logical/real/virtual voices, source DSP, spatial/occlusion, decode/I/O/cache, per-source effects, submix DSP, audio callback and memory.
Category: Derived Interview Bank / Audio / Performance
Priority: P1

Q: Explain Niagara System, Emitter, Module and Parameter responsibilities.
A: A System deploys/co-ordinates Emitters; each Emitter owns a particle population; ordered Modules transform typed parameter/attribute data; Parameters carry scoped values.
Category: Derived Interview Bank / VFX / Architecture
Priority: P1

Q: What is the difference among Active, Inactive, InactiveClear and Complete?
A: Active spawns/simulates; Inactive stops spawning but simulates survivors; InactiveClear removes particles then becomes inactive; Complete neither simulates nor renders.
Category: Derived Interview Bank / VFX / Lifecycle
Priority: P1

Q: CPU or GPU Niagara simulation?
A: Choose from population, module/Data Interface support, CPU interaction/readback, bounds and target CPU/GPU headroom; GPU moves particle scripts, not all effect work.
Category: Derived Interview Bank / VFX / Simulation
Priority: P1

Q: How should gameplay pass data into Niagara?
A: Through a small typed User-parameter contract, suitable Data Interfaces/collections, or Data Channels for high-volume records; gameplay remains authoritative.
Category: Derived Interview Bank / VFX / Data Flow
Priority: P1

Q: How do Niagara Component pooling and pre-cull interact with lifetime?
A: Pooling reuses Components to avoid allocation/GC; pre-cull can skip an already-insignificant spawn. Neither reduces simulation/render cost or guarantees a returned live Component.
Category: Derived Interview Bank / VFX / Lifetime and Pooling
Priority: P1

Q: Why does a Niagara effect disappear at camera edges or when its origin leaves view?
A: Its System/Emitter bounds may not contain visible particles/ribbon, or scalability/visibility culling may reject the instance.
Category: Derived Interview Bank / VFX / Bounds and Culling
Priority: P1

Q: What is a Niagara Effect Type and how would you use significance?
A: Effect Type groups related Systems under shared per-platform scalability, significance, budgets and validation; significance chooses which instances retain quality/existence.
Category: Derived Interview Bank / VFX / Scalability
Priority: P1

Q: Can Niagara collision or events drive gameplay damage?
A: No; Niagara collision/events are cosmetic simulation evidence with target/view/readback limitations, while gameplay/physics authority decides damage.
Category: Derived Interview Bank / VFX / Gameplay Boundary
Priority: P1

Q: Sprite, Mesh, Ribbon, Light or Component Renderer—how do their costs differ?
A: Sprites often cost translucent pixels; Meshes geometry/sections; Ribbons tessellation/history/sorting; Lights lighting/shadows; Components per-particle object/Tick overhead.
Category: Derived Interview Bank / VFX / Rendering
Priority: P1

Q: How do you diagnose and optimise translucent Niagara overdraw?
A: Measure screen coverage, overlapping layers and material cost with Shader Complexity/Quad Overdraw plus GPU timings, then reduce coverage/overlap/shader work or scale quality.
Category: Derived Interview Bank / VFX / GPU Performance
Priority: P1

Q: What are lightweight/stateless Niagara Emitters?
A: UE5.4+ constrained Emitters designed to reduce memory/CPU/tick and authoring overhead for compatible simple effects.
Category: Derived Interview Bank / VFX / Modern UE5
Priority: P2

Q: How do you profile a Niagara-heavy scene?
A: Separate instance/Game Thread, CPU particles, GPU compute, Render Thread/submission, GPU graphics/overdraw and memory.
Category: Derived Interview Bank / VFX / Profiling
Priority: P1

Q: Where do Lua and C# fit in an Unreal project?
A: Through third-party/studio runtimes or adjacent tools/services, not as Epic's baseline gameplay languages; adopt for a concrete iteration, content, modding or tooling need.
Category: Derived Interview Bank / Scripting / Adoption
Priority: P3

Q: Explain Lua tables, metatables and closures in an engine-integration context.
A: Tables are associative storage used for arrays/objects/modules; metatables customise lookup/operations; closures capture upvalues and can retain state/native wrappers.
Category: Derived Interview Bank / Lua / Language Model
Priority: P3

Q: What should an Unreal engineer know about the Lua C API stack?
A: Native functions read/push typed values on a virtual stack; indices, stack balance, borrowed pointers, protected errors and registry references require strict lifetime discipline.
Category: Derived Interview Bank / Lua / Native Interop
Priority: P3

Q: Lua GC versus Unreal GC—what can go wrong?
A: They trace different heaps, so wrappers can outlive destroyed UObjects, bindings can retain UObjects, and callbacks can form cycles neither product lifecycle breaks promptly.
Category: Derived Interview Bank / Lua / Lifetime
Priority: P3

Q: Is a Lua coroutine asynchronous or multithreaded?
A: It is cooperatively suspended execution, not an OS thread; UObject access is legal only on the host thread that resumes it under the engine's rules.
Category: Derived Interview Bank / Lua / Async
Priority: P3

Q: How would you design a safe script-facing Unreal API?
A: Expose coarse semantic commands, snapshots and events with stable IDs, typed units, validation and explicit lifetime/thread/error contracts.
Category: Derived Interview Bank / Scripting / API Design
Priority: P3

Q: How do .NET GC and native Unreal object lifetime interact?
A: Managed GC moves/reclaims managed objects; Unreal owns UObjects. Handles/wrappers must root callbacks, observe native invalidation and release registrations deterministically.
Category: Derived Interview Bank / C# / Lifetime
Priority: P3

Q: What are the main C# native-marshalling traps?
A: ABI layout, bool/char/string encoding, enum width, arrays/ownership, delegate lifetime/calling convention, pinning and non-blittable copies.
Category: Derived Interview Bank / C# / Native Interop
Priority: P3

Q: What threading trap does C# `async`/`Task` create in Unreal?
A: Continuations may run on thread-pool threads; managed async does not grant thread-safe UObject access.
Category: Derived Interview Bank / C# / Async
Priority: P3

Q: Why does Native AOT not automatically make C# safe to ship on every Unreal platform?
A: AOT is platform-specific and restricts dynamic loading, runtime codegen and reflection/trimming patterns; the Unreal plugin still needs native/runtime integration and platform approval.
Category: Derived Interview Bank / C# / Packaging
Priority: P3

Q: What does safe script or assembly hot reload require?
A: A transaction: quiesce, cancel/unbind, serialise versioned state, invalidate old handles/code, load/validate, migrate/rebind, resume or rollback.
Category: Derived Interview Bank / Scripting / Reload
Priority: P3

Q: How do you profile and debug a mixed native/script runtime?
A: Correlate native and script/managed stacks by boundary call/runtime generation while measuring call/marshalling/allocation/GC/wrapper/root/reload/package costs.
Category: Derived Interview Bank / Scripting / Debugging and Performance
Priority: P3

Q: How do you choose UPROPERTY specifiers without cargo-culting `EditAnywhere BlueprintReadWrite`?
A: Name the consuming system and intended lifecycle: editor authoring, Blueprint access, serialisation, config, save, duplication, replication or GC reference discovery.
Category: Derived Interview Bank / UE C++ / Property Specifiers
Priority: P0

Q: What is the difference between specifiers and metadata?
A: Specifiers usually express engine/reflection flags or generated behaviour, while metadata primarily guides editor and Blueprint node/tooling presentation.
Category: Derived Interview Bank / UE C++ / Reflection Metadata
Priority: P0

Q: How would you explain `Config`, `SaveGame`, `Transient`, and `DuplicateTransient`?
A: They describe different persistence and copy paths: config `.ini`, save-game archive selection, normal serialisation exclusion, and reset-on-duplication behaviour.
Category: Derived Interview Bank / UE C++ / Persistence
Priority: P0

Q: When should you use a reflected `UINTERFACE` instead of a native abstract C++ interface?
A: Use `UINTERFACE` when UObject/reflection/Blueprint participation is required; use native abstract C++ interfaces for pure C++ boundaries that do not need reflection.
Category: Derived Interview Bank / UE C++ / Interfaces
Priority: P0

Q: What does `Outer` imply, and what does it not imply?
A: `Outer` provides containment and identity/path context; it is not by itself a normal C++ ownership or GC-retention guarantee.
Category: Derived Interview Bank / UObject / Identity
Priority: P0

Q: How do `FindObject`, `LoadObject`, `NewObject`, `DuplicateObject`, and `SpawnActor` differ?
A: They express different intents: create UObject, create Actor in a World, find an already-loaded object, synchronously load an object, or duplicate an object graph.
Category: Derived Interview Bank / UE C++ / Object APIs
Priority: P0

Q: How do you choose between `UE_LOG`, `UE_LOGFMT`, `check`, `verify`, and `ensure`?
A: Logs record diagnostic events; checks assert impossible programmer invariants; verify preserves expression execution; ensure reports unexpected survivable state.
Category: Derived Interview Bank / UE C++ / Diagnostics
Priority: P0

Q: What is the danger of storing `TFunctionRef` or a view type for later?
A: They are non-owning views over a callable or storage; storing them beyond the call can dangle.
Category: Derived Interview Bank / UE C++ / Core Vocabulary
Priority: P1

Q: When is `TOptional` better than a sentinel value?
A: When absence is a valid separate state and no normal value should be overloaded to mean "not present".
Category: Derived Interview Bank / UE C++ / Core Vocabulary
Priority: P1

Q: What is a safe design for an async Blueprint node?
A: A proxy UObject with explicit in-flight retention, activation, cancellation, Game Thread completion, stale-generation checks and delegate cleanup.
Category: Derived Interview Bank / Blueprint / Async
Priority: P1

Q: When would you use `FGCObject`?
A: When a non-UObject native owner must report UObject references to the garbage collector and cannot reasonably store reflected UObject properties.
Category: Derived Interview Bank / UObject / GC Bridge
Priority: P1

Q: How should off-thread work interact with UObjects?
A: Snapshot plain data on the Game Thread, process off-thread without UObject access, then apply results back on the Game Thread after weak/generation validation.
Category: Derived Interview Bank / UE C++ / Thread Boundaries
Priority: P0

Q: How do you debug a generated-code or specifier problem?
A: Read the first real UHT/compiler error, inspect the reflected declaration before the macro, reduce the declaration, then identify which system consumes the specifier.
Category: Derived Interview Bank / UE C++ / UHT Debugging
Priority: P0

Q: How does Actor Component replication differ from Actor replication?
A: Components replicate through their owning replicated Actor and must be explicitly set to replicate; their properties, RPCs and subobjects have their own contracts and overhead.
Category: Derived Interview Bank / Networking / Components
Priority: P1

Q: When would you use a replicated UObject subobject?
A: When nested state belongs to an owning Actor or Component, needs replicated fields, but does not deserve full Actor identity.
Category: Derived Interview Bank / Networking / Subobjects
Priority: P2

Q: Why can a replicated subobject pointer be null on a client?
A: A dynamic subobject created on the server must be replicated and mapped before the client reference becomes valid.
Category: Derived Interview Bank / Networking / Subobject Mapping
Priority: P2

Q: What is the registered subobjects list and why does it matter?
A: It is a maintained list of subobjects to replicate from an Actor or Component, reducing per-subobject virtual calls and required for Iris subobject replication in the maintained docs.
Category: Derived Interview Bank / Networking / Subobjects
Priority: P2

Q: When should you use `FFastArraySerializer`?
A: When an authoritative replicated collection has stable item identity and benefits from item-level add/change/remove deltas.
Category: Derived Interview Bank / Networking / Fast Array
Priority: P2

Q: What problem does Replication Graph solve?
A: It reduces server CPU spent building per-client replication lists for large Actor/player counts by using persistent graph nodes and shared data.
Category: Derived Interview Bank / Networking / Replication Graph
Priority: P2

Q: How should you talk about Iris in a UE5 interview?
A: Iris is an opt-in newer replication system, experimental in the docs, and not automatically used by every UE5 project.
Category: Derived Interview Bank / Networking / Iris
Priority: P2

Q: What does push-model replication change?
A: It lets game code report dirtiness instead of relying only on polling, but it does not change authority, relevance or replication protocol design.
Category: Derived Interview Bank / Networking / Push Model
Priority: P2

Q: How do network debug CVars fit into a diagnosis workflow?
A: They are targeted evidence tools for dormancy, NetGUIDs, push model, subobjects, packet behaviour and validation, not production fixes.
Category: Derived Interview Bank / Networking / Debugging
Priority: P1

Q: How would you replicate a modular inventory with high item churn?
A: Server owns item instances; choose simple owner-only state, Fast Array or subobjects based on item count, nested state and client audience.
Category: Derived Interview Bank / Networking / System Design
Priority: P1

Q: How does CharacterMovement client prediction work?
A: The owning client saves and simulates moves immediately; the server reproduces them authoritatively; corrections cause the client to apply server state and replay unacknowledged moves.
Category: Derived Interview Bank / Networking / CharacterMovement
Priority: P1

Q: When do you extend `FSavedMove_Character`?
A: When small custom state affects CharacterMovement simulation and must be captured, compressed, restored and replayed with the client move.
Category: Derived Interview Bank / Networking / CharacterMovement
Priority: P2

Q: Why does a predicted sprint or dash jitter even though the bool replicates?
A: The replicated bool arrives too late for movement prediction; the movement-affecting state must be part of the saved move/server reproduction path.
Category: Derived Interview Bank / Networking / Debugging
Priority: P1

Q: How would you design a predicted dash?
A: Save the dash input edge and compact movement state, validate it on the server, integrate movement through CharacterMovement or a supported prediction path, and instrument corrections.
Category: Derived Interview Bank / Networking / CharacterMovement
Priority: P2

Q: What is lag compensation or server rewind?
A: The server evaluates an action against bounded historical hit-relevant state so it can judge what the client plausibly saw without trusting the client's hit result.
Category: Derived Interview Bank / Networking / Lag Compensation
Priority: P1

Q: What should a lag-compensation history buffer store?
A: Store compact value snapshots of hit-relevant state: server time/sequence, root transform, hitboxes or pose-derived boxes, movement/target state and enough metadata to validate transitions.
Category: Derived Interview Bank / Networking / Lag Compensation
Priority: P2

Q: How do prediction, reconciliation, interpolation, rewind and rollback differ?
A: Prediction simulates locally, reconciliation applies server correction and replays, interpolation smooths remote state, rewind evaluates past state, and rollback restores old simulation state then resimulates.
Category: Derived Interview Bank / Networking / Concepts
Priority: P1

Q: How do you prevent lag compensation from becoming an exploit?
A: Clamp rewind windows, map timestamps through server-known timing, validate weapon/player state and aim evidence, define occlusion/respawn policy, and log suspicious extremes.
Category: Derived Interview Bank / Networking / Security
Priority: P1

Q: How would you profile custom movement prediction and lag compensation?
A: Measure correction counts/distances, saved move allocation/combine/replay, custom move bytes, history memory/allocation, rewind query cost and traffic under representative lag/loss.
Category: Derived Interview Bank / Networking / Profiling
Priority: P2

Q: How would you answer a custom CharacterMovement question when you have not shipped it?
A: Be explicit about the known pipeline, the extension points you would verify, and the tests/profiling you would use; do not invent exact signatures.
Category: Derived Interview Bank / Networking / Interview Strategy
Priority: P2

Q: How would you design a device-lab gate for an Unreal project?
A: Package reproducibly, run deterministic target scenarios on a small device matrix, capture logs/CSV/traces/crashes, attach every artifact to build/device/scenario identity, and gate only stable actionable signals.
Category: Derived Interview Bank / Profiling / Build / Platform Automation
Priority: P2

Q: Where does Gauntlet fit in Unreal automation?
A: Gauntlet is a UE-aware automation framework used through RunUnreal-style workflows to launch packaged/editor targets, run tests and monitor results; it is not a substitute for project scenario design or platform validation.
Category: Derived Interview Bank / Build / Automation / Platform
Priority: P2

Q: What belongs in a device-lab run manifest?
A: Build ID, engine/project revision, platform/configuration, package and symbols paths, device ID/state, scenario ID, command line, install mode and output artifact paths.
Category: Derived Interview Bank / Build / Release Evidence
Priority: P2

Q: How do CSV and Unreal Insights differ in a device lab?
A: CSV is light, repeatable telemetry for gates and trend aggregation; Insights is a heavier causal trace used selectively around a failed or sampled window.
Category: Derived Interview Bank / Profiling / Automation
Priority: P2

Q: How do you debug a device-lab failure that only happens on one device?
A: Classify phase and identity first, compare nearest passing runs, verify device state/profile/install mode, rerun only for recognised infrastructure classes, then capture the smallest causal evidence.
Category: Derived Interview Bank / Platform / Debugging
Priority: P2

Q: When should an automated metric become a release-blocking gate?
A: After the team has measured variance, confirmed the signal is stable on target devices, defined tolerance and owner, and proved failures are actionable.
Category: Derived Interview Bank / Profiling / Release Management
Priority: P2

Q: Why is forced packaged-crash proof part of a device-lab packet?
A: It proves the archived package, symbols, crash metadata and symbolication path match the exact build that users or testers will run.
Category: Derived Interview Bank / Build / Crash Debugging
Priority: P2

Q: What is the difference between a product failure, lab infrastructure failure and flaky device?
A: A product failure reproduces against the build/scenario, a lab infrastructure failure belongs to the runner/install/artifact path, and a flaky device repeatedly fails health or environment checks outside product logic.
Category: Derived Interview Bank / Automation / Triage
Priority: P2

Q: How would you design a UE5 open-world production pipeline?
A: Define traversal and platform budgets, World Partition grid/source policy, spatial versus always-loaded Actors, Data Layers, reusable POIs, HLOD, cook/package gates and target traversal evidence.
Category: Derived Interview Bank / World Partition / System Design
Priority: P2

Q: World Partition versus Level Streaming?
A: Level Streaming explicitly loads/unloads level packages; World Partition divides a persistent UE5 world into grid cells driven by streaming sources and integrates with OFPA, Data Layers and HLOD.
Category: Derived Interview Bank / World Partition / Streaming
Priority: P1

Q: How should Runtime Data Layers be used for progression?
A: Keep durable progression in quest/save/server state, then use Runtime Data Layers to present the correct world phase through load/activation policy.
Category: Derived Interview Bank / World Partition / Data Layers
Priority: P2

Q: How do you prevent fast travel into unloaded or collisionless space?
A: Move or create a destination streaming source, wait for streaming completion and project-specific collision/nav/gameplay readiness, then teleport or show a loading/fallback path.
Category: Derived Interview Bank / World Partition / Debugging
Priority: P2

Q: What can go wrong with One File Per Actor?
A: OFPA reduces level-file conflicts, but encoded external actor files and partial submits can create missing or dangling references.
Category: Derived Interview Bank / World Partition / Source Control
Priority: P2

Q: Level Instance or Packed Level Blueprint?
A: Use Level Instances for reusable authored Actor arrangements and Packed Level Blueprints for static dense visual arrangements; avoid treating either as a generic gameplay-state container.
Category: Derived Interview Bank / World Building / Authoring
Priority: P2

Q: How do you evaluate HLOD for a large world?
A: Name the cost HLOD should reduce, choose HLOD layers/proxy settings, build them in automation, and compare far/transition/near evidence for performance, memory and artefacts.
Category: Derived Interview Bank / Rendering / World Partition
Priority: P2

Q: What belongs in a packaged large-world validation gate?
A: Validate OFPA/source-control, references, Data Layers, Level Instances, generated output, HLOD build, cook/package and target traversal telemetry.
Category: Derived Interview Bank / Build / World Partition
Priority: P2

Q: How do you profile Android GPU performance for a UE game?
A: Use a packaged build, Unreal stats/CSV for scenario classification, then Android system/frame profiling to identify GPU scheduling, render passes, counters and device behavior.
Category: Derived Interview Bank / Platform / Android Profiling
Priority: P2

Q: Unreal Insights versus Android GPU Inspector?
A: Unreal Insights shows engine-side scopes/tasks/loading/memory; AGI/APA shows Android device, OS, GPU queue, render-pass and counter evidence.
Category: Derived Interview Bank / Platform / Profiler Selection
Priority: P2

Q: Android shows high CPU frame time. What do you check before optimising gameplay?
A: Distinguish active CPU work from total CPU time that includes waiting on GPU/present, then correlate with Unreal Game/Render/RHI scopes.
Category: Derived Interview Bank / Platform / Android CPU-GPU Diagnosis
Priority: P2

Q: How do you diagnose GMEM or render-pass bandwidth on mobile?
A: Use AGI Frame Profiler to find the expensive pass, inspect binning/rendering/GMEM load-store signals, then map the pass to Unreal render targets, depth/stencil, shadows, post, UI or scene captures.
Category: Derived Interview Bank / Platform / Mobile GPU
Priority: P2

Q: How do you interpret Android GPU counters safely?
A: Use counters to support a concrete hypothesis, and treat names/definitions as GPU-vendor-specific rather than universal metrics.
Category: Derived Interview Bank / Platform / GPU Counters
Priority: P2

Q: How do you handle Android LMK memory reports?
A: Treat LMK as lifecycle memory pressure, correlate Android termination/vitals with Unreal LLM/Memory Insights, and reproduce foreground/background warm-return behavior.
Category: Derived Interview Bank / Platform / Memory
Priority: P2

Q: Should you enable the Android ADPF Unreal plugin?
A: Evaluate it with a no-adaptation baseline, supported-device proof, thermal/scalability logs and quality acceptance; do not use it to hide underlying regressions.
Category: Derived Interview Bank / Platform / Thermal Performance
Priority: P2

Q: When is Android Fixed Performance Mode useful?
A: Use it for repeatable benchmark comparisons where available, but do not treat it as representative sustained user behavior.
Category: Derived Interview Bank / Platform / Benchmarking
Priority: P3

Q: RunUAT reports a generic failure. How do you debug it?
A: Find the first causal error and failed phase before acting on the final AutomationTool wrapper message.
Category: Derived Interview Bank / Build / Automation
Priority: P2

Q: Local package passes but clean CI fails. What is your model?
A: Assume the clean agent exposed an undeclared dependency: generated files, unity includes, local binaries, global third-party files, warm DDC or loose editor content.
Category: Derived Interview Bank / Build / CI
Priority: P2

Q: How does an editor-only module leak into Shipping?
A: A runtime module or asset depends on Editor-only classes/modules, often hidden by Editor builds or misplaced `WITH_EDITOR` assumptions.
Category: Derived Interview Bank / Build / Modules
Priority: P1

Q: How do you prove a third-party runtime dependency is staged correctly?
A: Launch the package on a clean machine/device and verify staged runtime files, platform paths, symbols and Build.cs/plugin descriptor rules.
Category: Derived Interview Bank / Build / Release
Priority: P2

Q: Why force a packaged crash in the build pipeline?
A: To prove package, build ID, crash metadata, symbol archive and symbolication route match before real users hit crashes.
Category: Derived Interview Bank / Build / Crash Proof
Priority: P2

Q: What does a good BuildGraph dependency model express?
A: It expresses named inputs, outputs, dependencies, agents, labels and artifact flow, not just a long sequence of shell commands.
Category: Derived Interview Bank / Build / BuildGraph
Priority: P3

Q: When should you implement a custom AI sense?
A: Use a custom sense when recurring game-specific evidence needs perception-style listener, stimulus, memory, range/team and debugging behaviour.
Category: Derived Interview Bank / AI / Perception / Architecture
Priority: P2

Q: What data belongs in a custom stimulus?
A: Source identity, location, strength/confidence, tag/type, time/age, success/loss state where relevant, and enough stable identity to handle destroyed/reused Actors.
Category: Derived Interview Bank / AI / Perception / Data Design
Priority: P2

Q: Why should a custom sense not write Blackboard keys directly?
A: The sense should report evidence; a memory/decision layer should interpret and project that evidence into Blackboard or StateTree state.
Category: Derived Interview Bank / AI / Architecture
Priority: P2

Q: How do you debug a custom sense that never reports stimuli?
A: Prove each layer: listener config, source event, correct World, sense update, filter acceptance, stimulus report and memory projection.
Category: Derived Interview Bank / AI / Perception / Debugging
Priority: P2

Q: How do you profile a custom AI sense?
A: Count events, discarded/merged events, listeners considered, expensive checks, stimuli reported, callbacks, memory writes, allocations and game-thread time.
Category: Derived Interview Bank / AI / Performance
Priority: P2

Q: What does MassAI mean in a practical Unreal architecture?
A: It means using MassEntity data/processors for crowd or agent simulation and bridging to classic AI/gameplay only where rich behaviour requires it.
Category: Derived Interview Bank / MassEntity / AI Architecture
Priority: P2

Q: ZoneGraph versus NavMesh for AI crowds?
A: NavMesh supports general traversability/pathfinding; ZoneGraph-style lane data supports structured crowd/traffic movement through authored lanes and zones.
Category: Derived Interview Bank / MassEntity / AI Navigation
Priority: P2

Q: How should StateTree be used with Mass entities?
A: Use it for shared state/activity flow over entity context/fragments, not arbitrary per-entity Actor-style calls that erase Mass batching.
Category: Derived Interview Bank / MassEntity / StateTree
Priority: P2

Q: How do Smart Objects fit into Mass crowds?
A: They provide a claim/use/release concurrency boundary for specific world affordances, while Mass owns scalable activity and representation state.
Category: Derived Interview Bank / MassEntity / Smart Objects
Priority: P2

Q: What is the Actor promotion contract in a MassAI system?
A: Define stable mapping, state transfer, authoritative writer, reset/invalidation, network audience and hysteresis when an entity gains or loses Actor representation.
Category: Derived Interview Bank / MassEntity / Hybrid Actors
Priority: P2

Q: How do you debug a Mass crowd jam?
A: Separate route/lane choice, local avoidance, movement integration, Smart Object reservation, collision/animation and representation.
Category: Derived Interview Bank / MassEntity / Debugging
Priority: P2

Q: How do you profile MassAI fairly?
A: Compare equivalent behaviour/visual fidelity and split simulation, structural, bridge, representation, StateTree/Smart Object and rendering costs.
Category: Derived Interview Bank / MassEntity / Performance
Priority: P2

Q: Why can an Animation Notify be a bad authority boundary?
A: Notifies are presentation/timing signals from animation evaluation; durable gameplay authority should validate state, timing, ownership and network role separately.
Category: Derived Interview Bank / Animation / Networking / Gameplay
Priority: P2

Q: How do you debug a montage that never exits an attack state?
A: Inspect montage play/section/slot/group, blend-out/interruption, notify/state callbacks, gameplay task cancellation and the state variable owner.
Category: Derived Interview Bank / Animation / Debugging
Priority: P2

Q: Why can root motion create movement and network bugs?
A: Root motion moves the character from animation data, so it must be coordinated with CharacterMovement, collision, authority, prediction/correction and montage state.
Category: Derived Interview Bank / Animation / Movement / Networking
Priority: P2

Q: Why might a blocking collision not produce a Hit event?
A: Blocking response, physics simulation, sweep path, movement API, notification flags and component setup are separate requirements.
Category: Derived Interview Bank / Physics / Collision / Debugging
Priority: P1

Q: Sweep movement versus physics simulation?
A: A sweep is an explicit scene query/move test; physics simulation is solver-driven rigid-body integration with contacts, forces and constraints.
Category: Derived Interview Bank / Physics / Movement
Priority: P2

Q: Why can UUserWidget Construct be the wrong place for one-time setup?
A: Construct can run when the underlying Slate widget is rebuilt; true once-per-widget-object setup belongs in an appropriate one-time lifecycle hook.
Category: Derived Interview Bank / UI / UMG / Lifecycle
Priority: P2

Q: How do virtualised ListView entries cause stale UI bugs?
A: Entry widgets are recycled presentation objects; item data identity must not be stored as permanent widget-local state without reset/rebind policy.
Category: Derived Interview Bank / UI / Lists / Debugging
Priority: P2

Q: Retainer Panel versus Invalidation Box?
A: Invalidation reduces widget layout/paint work when UI is unchanged; Retainer renders UI to a texture at controlled cadence but adds render-target/GPU/memory trade-offs.
Category: Derived Interview Bank / UI / Performance
Priority: P2

Q: How does audio concurrency stealing become a gameplay bug?
A: If gameplay relies on hearing a sound that can be culled/stolen/virtualised, audio voice policy can hide required feedback.
Category: Derived Interview Bank / Audio / Gameplay / Debugging
Priority: P2

Q: Why can MetaSounds be performance-sensitive?
A: Procedural/sample-accurate graphs still consume CPU/DSP/voice resources and can multiply cost through polyphony and parameter churn.
Category: Derived Interview Bank / Audio / MetaSounds / Performance
Priority: P2

Q: Why should Niagara events not drive authoritative damage?
A: Niagara is presentation/effects simulation; authoritative gameplay collision, damage and replication should live in gameplay systems.
Category: Derived Interview Bank / Niagara / Gameplay / Networking
Priority: P2

Q: What state must be reset when pooling Niagara or audio components?
A: Reset parameters, attachments, transforms, owner/context, delegates/callbacks, activation state, pooling handles and any gameplay-facing references.
Category: Derived Interview Bank / VFX / Audio / Lifetime
Priority: P2

Q: Why is transforming normals under non-uniform scale tricky?
A: Normals are directions perpendicular to surfaces; under non-uniform scale they generally require inverse-transpose-style handling, not the same transform as points.
Category: Derived Interview Bank / Game Maths / Transforms
Priority: P1

Q: When can a spatial hash perform badly?
A: Bad cell size, clustered distributions, high update churn, duplicate boundary insertion and poor query radius assumptions can erase expected benefits.
Category: Derived Interview Bank / Algorithms / Spatial Structures
Priority: P2

Q: How do Lua or C# wrappers become stale around UObjects?
A: Embedded/managed runtimes can hold wrappers after the underlying UObject is destroyed, collected, unloaded or otherwise invalid for that call.
Category: Derived Interview Bank / Scripting / Lifetime
Priority: P2

Q: When is a raw UObject pointer unsafe even if it currently works?
A: It is unsafe when the reference is not discoverable/validated across GC, async work, destruction, level unload, delayed callbacks or ownership changes.
Category: Derived Interview Bank / UObject / Lifetime
Priority: P0

Q: How do you choose between `TObjectPtr`, `TWeakObjectPtr` and `TSoftObjectPtr`?
A: Choose from ownership/reference strength, load state, lifetime domain and whether the pointer should keep an object reachable.
Category: Derived Interview Bank / UObject / Pointer Selection
Priority: P0

Q: Constructor, OnConstruction, PostInitializeComponents or BeginPlay?
A: Use constructor for defaults/default subobjects, construction for editor/instance setup, component initialisation hooks for registered components and BeginPlay for gameplay start.
Category: Derived Interview Bank / Gameplay Framework / Lifecycle
Priority: P0

Q: How can a Subsystem become a hidden Service Locator problem?
A: A scoped Subsystem still hides dependencies when arbitrary code pulls global services instead of receiving explicit interfaces or context.
Category: Derived Interview Bank / Gameplay Architecture / Subsystems
Priority: P1

Q: Why is a `BlueprintPure` function with hidden work dangerous?
A: Pure nodes look side-effect-free and may be evaluated more often than expected, so hidden loads, allocations, mutation or expensive queries cause invisible bugs and performance cost.
Category: Derived Interview Bank / C++ / Blueprint / API Design
Priority: P1

Q: When should a delegate be dynamic rather than native?
A: Use dynamic delegates when reflection/Blueprint/serialisation needs them; prefer native delegates for pure C++ hot paths and stronger type/performance needs.
Category: Derived Interview Bank / UE C++ / Delegates / Blueprint
Priority: P1

Q: How do async Blueprint action nodes leak or call back into dead objects?
A: Proxy objects, delegates, timers, latent work and captured UObjects can outlive the caller/world unless cancellation and validation are explicit.
Category: Derived Interview Bank / C++ / Blueprint / Async
Priority: P2

Q: Why should Data Assets usually be treated as definitions rather than mutable runtime state?
A: Data Assets are shared authored definitions; mutating them as per-instance runtime state leaks changes across users, saves, PIE/editor and asset references.
Category: Derived Interview Bank / Gameplay Architecture / Data Assets
Priority: P1

Q: How does `FAssetData::GetAsset` accidentally turn discovery into loading?
A: Asset Registry metadata queries can stay unloaded, but calling `GetAsset` resolves the actual UObject and may synchronously load it.
Category: Derived Interview Bank / Assets / Asset Registry / Performance
Priority: P2

Q: How do Primary Asset bundles become accidental memory bombs?
A: Bad bundle boundaries can load UI, gameplay, audio, VFX and large dependencies together when only one purpose-specific subset is needed.
Category: Derived Interview Bank / Assets / Asset Manager
Priority: P2

Q: Why can Live Coding hide structural bugs?
A: Live Coding patches code into a running editor; it does not prove clean startup, reflection/layout migration, serialisation, constructor defaults or packaged correctness.
Category: Derived Interview Bank / Build / Iteration / Debugging
Priority: P2

Q: What makes save/load a transaction rather than a dump?
A: Save/load needs stable identity, schema/versioning, dependency ordering, validation, partial failure policy and explicit reconstruction rules.
Category: Derived Interview Bank / Gameplay Architecture / Save Systems
Priority: P1

Q: How would you prove the correct ASC owner/avatar placement for a respawning character?
A: Run a dedicated-server respawn test that logs owner, avatar, actor info, grants, old bindings and predicted activation before and after respawn.
Category: Derived Interview Bank / GAS / Lifetime / Debugging
Priority: P2

Q: Why is a grant-source ledger safer than removing abilities by class?
A: A class can be granted by multiple sources; removing by source/spec handle preserves exact ownership and avoids deleting the wrong grant.
Category: Derived Interview Bank / GAS / Grants / Equipment
Priority: P2

Q: What should a GAS activation trace contain?
A: Role, owner/avatar, ASC, ability class, spec handle, prediction key, target data, failure tags, commit/effect handles, task state and end reason.
Category: Derived Interview Bank / GAS / Observability
Priority: P2

Q: How do you prove a Health attribute invariant is complete?
A: Test every mutation path: instant effects, max changes, periodic effects, predictions, replication and UI observation.
Category: Derived Interview Bank / GAS / Attributes
Priority: P2

Q: Direct modifier, magnitude calculation or execution calculation?
A: Use the lightest mechanism that expresses the rule: direct for simple scalar changes, calculation for derived magnitude, execution for authoritative multi-attribute outcomes.
Category: Derived Interview Bank / GAS / Effects / Calculations
Priority: P2

Q: What does capture timing mean in a Gameplay Effect design?
A: Captured values may represent state at spec creation/application or later evaluation; the design must say which timing is intended.
Category: Derived Interview Bank / GAS / Effects / Captures
Priority: P2

Q: What makes a stacking Gameplay Effect policy complete?
A: Source aggregation, max count, refresh, expiration, period reset, overflow, granted tags, UI attribution and persistence must all be specified.
Category: Derived Interview Bank / GAS / Stacking
Priority: P2

Q: How do you debug a predicted GAS ability that snaps back?
A: Correlate client and server by prediction key, then inspect validation, commit, target data and non-reversible side effects.
Category: Derived Interview Bank / GAS / Prediction / Debugging
Priority: P2

Q: Why can cost or cooldown be applied twice in GAS?
A: Duplicate commit paths, manual plus framework application, repeated input triggers, duplicate grants or prediction/server mismatch can apply the same policy twice.
Category: Derived Interview Bank / GAS / Costs / Debugging
Priority: P2

Q: How should server target-data validation work for a predicted attack?
A: Treat client target data as evidence; the server validates ownership, timing, target, range, line of sight, cadence, tags and derives final result.
Category: Derived Interview Bank / GAS / Target Data / Security
Priority: P2

Q: What is the acceptance test for a custom Ability Task?
A: Cancel the ability at every wait phase and prove delegates/timers are unbound and no late callback mutates gameplay.
Category: Derived Interview Bank / GAS / Ability Tasks
Priority: P2

Q: How do you prevent Gameplay Cues from leaking authority?
A: Cues present gameplay state; they must not calculate durable damage, tags, inventory, objectives or win conditions.
Category: Derived Interview Bank / GAS / Gameplay Cues / Architecture
Priority: P2

Q: What does Full, Mixed and Minimal GE replication actually decide?
A: It controls active Gameplay Effect information detail by audience; it does not mean every ASC attribute/tag field follows that same rule.
Category: Derived Interview Bank / GAS / Replication
Priority: P2

Q: How do you debug a cooldown tag that never clears?
A: Inspect active effects and loose-tag sources/counts, then trace duration, stack, removal and respawn/equipment lifecycle.
Category: Derived Interview Bank / GAS / Tags / Debugging
Priority: P2

Q: How should GAS integrate with UI without UI owning truth?
A: UI should observe ASC-derived model state and request actions through input/commands; it should not own ability availability, health or cooldown truth.
Category: Derived Interview Bank / GAS / UI / Architecture
Priority: P2

Q: What GAS state should remain outside GAS?
A: Durable inventory, quest state, save identity, economy, matchmaking and many authoritative transactions usually remain ordinary gameplay systems that may grant/apply GAS state.
Category: Derived Interview Bank / GAS / Architecture
Priority: P2

Q: How do you compare GAS against a bespoke ability Component?
A: Compare interaction density, prediction, tags/effects/stacking, content authoring, debugging, team cost and failure handling in a representative slice.
Category: Derived Interview Bank / GAS / System Design
Priority: P2

Q: What is the difference between GAS prediction and full rollback networking?
A: GAS prediction correlates supported provisional owner actions with server confirmation; full rollback restores broader simulation state and resimulates from input/state history.
Category: Derived Interview Bank / GAS / Networking / Rollback
Priority: P2

Q: How do you make GAS profiling actionable?
A: Separate activations, active effects, periodic ticks, tag delegates, execution calculations, tasks, cues, Blueprint work and replication by audience.
Category: Derived Interview Bank / GAS / Profiling
Priority: P2

Q: How do you correlate Unreal Insights with an Xcode Metal GPU capture?
A: Capture the same packaged scenario/frame, align markers/timestamps, map Metal passes/pipelines/counters back to UE passes/content, then validate with normal UE before/after timing.
Category: Derived Interview Bank / Platform Profiling / Apple / Rendering
Priority: P2

Q: Why is simulator memory not enough for iOS memory proof?
A: The simulator does not model iOS device memory warnings/terminations faithfully, so safe simulator memory does not prove safe target memory.
Category: Derived Interview Bank / Platform Profiling / Apple / Memory
Priority: P2

Q: How would you use MetricKit for a UE game?
A: Use MetricKit as field telemetry for trends and diagnostics, then reproduce locally with UE/platform captures.
Category: Derived Interview Bank / Platform Profiling / Apple / Telemetry
Priority: P2

Q: What metadata must accompany an Apple Metal capture?
A: Device/GPU/OS/Xcode, UE branch/build/RHI, package/scenario, capture scope/frame, thermal/performance state and matching UE evidence.
Category: Derived Interview Bank / Platform Profiling / Evidence
Priority: P2

Q: How do you interpret Metal serial versus concurrent profiling modes?
A: Concurrent mode reflects overlapped runtime execution; serial mode isolates pass costs more precisely but does not represent normal runtime performance.
Category: Derived Interview Bank / Platform Profiling / Apple / GPU
Priority: P2

Q: How does Xcode Thread Performance Checker apply to Unreal?
A: It can flag native priority inversions and main-thread non-UI work, especially in platform wrappers/plugins, but it is not a replacement for UE GameThread profiling.
Category: Derived Interview Bank / Platform Profiling / Apple / CPU
Priority: P2

Q: How do you answer console profiling questions without leaking confidential details?
A: Discuss source-safe workflow and categories, and state that exact tools, counters and cert rules require authorised platform-holder docs.
Category: Derived Interview Bank / Platform Profiling / Console / Interview Safety
Priority: P2

Q: What makes console profiling different from PC packaged profiling?
A: Fixed platform constraints, SDK/toolchain, memory layout, GPU/driver behavior, storage, OS lifecycle and certification-adjacent states can differ materially from PC.
Category: Derived Interview Bank / Platform Profiling / Console
Priority: P2

Q: How do you handle a GPU hitch that appears only on one Apple or console target?
A: Verify package/config, reproduce deterministically, capture UE and platform GPU evidence, map the pass/pipeline/counter back to content, then rerun the same scenario.
Category: Derived Interview Bank / Platform Profiling / GPU Debugging
Priority: P2

Q: How do field metrics become a device-lab task?
A: Break the metric down by version/device/OS/scenario, form a reproduction hypothesis, then run a controlled target-device capture.
Category: Derived Interview Bank / Platform Profiling / Telemetry / Device Lab
Priority: P2

Q: What should a platform profiling memo include?
A: Repro metadata, UE evidence, platform evidence, hypothesis, change, before/after result, residual risk and what the evidence does not prove.
Category: Derived Interview Bank / Platform Profiling / Communication
Priority: P2

Q: How do you avoid overclaiming from one platform capture?
A: Label device/tool/build scope, repeat the scenario, cross-check with UE metrics and state what remains unproven.
Category: Derived Interview Bank / Platform Profiling / Evidence Quality
Priority: P2

Q: What makes a gameplay operation transactional?
A: It validates against a known state revision, commits all required changes together, emits an auditable result and controls failure/duplicate/retry behavior.
Category: Derived Interview Bank / Gameplay Architecture / Transactions
Priority: P1

Q: Why are operation IDs useful in gameplay systems?
A: They correlate request, validation, commit, replication, UI prediction, telemetry and duplicate/retry handling.
Category: Derived Interview Bank / Gameplay Architecture / Observability
Priority: P1

Q: What is the difference between an event and durable truth?
A: An event says something happened; durable state answers what is true now for late subscribers, save/load and replication.
Category: Derived Interview Bank / Gameplay Architecture / Events
Priority: P1

Q: What makes an event payload strong enough?
A: It carries the committed transition: IDs, old/new values or delta, reason/source and revision/operation context where needed.
Category: Derived Interview Bank / Gameplay Architecture / Events
Priority: P1

Q: When is an event queue better than a delegate?
A: Use a queue when delayed, batched, ordered or throttled processing is part of the desired semantics.
Category: Derived Interview Bank / Gameplay Architecture / Queues
Priority: P2

Q: How do revisions prevent stale UI operations?
A: The client request states the revision it was based on; the server rejects or reconciles if authoritative state has changed.
Category: Derived Interview Bank / Gameplay Architecture / UI Reconciliation
Priority: P1

Q: How do you make a reward grant idempotent?
A: Store a durable claim/operation record so repeated requests or load/retry paths cannot grant twice.
Category: Derived Interview Bank / Gameplay Architecture / Idempotency
Priority: P1

Q: What is a restore path in save/load architecture?
A: A controlled load path that applies versioned state without replaying normal gameplay side effects, then rebuilds and emits a ready snapshot.
Category: Derived Interview Bank / Save Systems / Architecture
Priority: P1

Q: Why are historical save fixtures valuable?
A: They prove old player data still migrates after schema/content changes.
Category: Derived Interview Bank / Save Systems / Testing
Priority: P2

Q: How do you design UI prediction without UI owning truth?
A: UI creates pending operation IDs and visual state, then reconciles from authoritative commit/rejection and revisions.
Category: Derived Interview Bank / Gameplay Architecture / UI Prediction
Priority: P1

Q: What should a production state machine own besides state names?
A: Enter/exit side effects, timers/tasks/delegates, transition guards, authority, replication, save/restore and cancellation cleanup.
Category: Derived Interview Bank / Gameplay Architecture / State Machines
Priority: P1

Q: How can a global message bus become an architecture problem?
A: It hides dependencies, spreads unversioned schemas, weakens ownership and can let arbitrary subscribers mutate authoritative state.
Category: Derived Interview Bank / Gameplay Architecture / Messaging
Priority: P2

Q: How do Data Asset definitions support migration-safe runtime state?
A: Runtime state stores stable definition IDs and mutable fields, while definitions stay authored/shared and migrations map old IDs/schema to current content.
Category: Derived Interview Bank / Data-Driven Architecture / Persistence
Priority: P1

Q: How do you debug a duplicated inventory item?
A: Reconstruct the operation log: operation ID, revisions, validation, commit deltas, replication/UI prediction and save/load path.
Category: Derived Interview Bank / Gameplay Architecture / Debugging
Priority: P1

Q: What architecture costs should you profile in advanced patterns?
A: Operation rate, validation cost, queue depth, delegate cascades, dirty-cache recompute, save migration, UI rebuilds and replicated delta size.
Category: Derived Interview Bank / Gameplay Architecture / Profiling
Priority: P2

Q: Why should animation notifies usually not be the sole authority for damage?
A: Notifies are timing/presentation signals; authoritative gameplay must validate state, target, montage/ability context and server timing.
Category: Derived Interview Bank / Animation / Gameplay Authority
Priority: P1

Q: How do you debug foot sliding in a networked character?
A: Compare authoritative movement velocity/root motion, AnimBP inputs, Blend Space thresholds, sync markers and smoothing on server/owner/proxy.
Category: Derived Interview Bank / Animation / Movement / Networking
Priority: P1

Q: Why can a blocking collision still miss gameplay response?
A: Collision response, movement path, notification settings, geometry mode, component ownership and authority all affect whether gameplay receives an event.
Category: Derived Interview Bank / Physics / Collision / Events
Priority: P1

Q: How do you choose between sweep movement and simulated physics?
A: Choose from ownership of integration: kinematic sweeps query/move deliberately, while simulation lets the physics solver integrate bodies/contacts.
Category: Derived Interview Bank / Physics / Movement Ownership
Priority: P1

Q: Why can CommonUI focus break in split-screen?
A: Focus and input context must be scoped to the correct local player, activatable layer and desired focus target.
Category: Derived Interview Bank / UI / CommonUI / Local Player
Priority: P2

Q: When can UMG invalidation or Retainer Panels make performance worse?
A: When content changes frequently, volatility is mislabelled, render targets add GPU/memory cost, or latency/overdraw outweigh saved layout work.
Category: Derived Interview Bank / UI / Performance
Priority: P2

Q: How do you debug a sound that should be playing but is inaudible?
A: Trace request, component lifetime, asset readiness, attenuation/listener, concurrency/priority, routing/mix/submix and platform device.
Category: Derived Interview Bank / Audio / Debugging
Priority: P2

Q: Why can audio concurrency become a gameplay bug?
A: Concurrency can reject, steal or virtualise sounds, so critical gameplay information cannot exist only as audio playback.
Category: Derived Interview Bank / Audio / Gameplay Boundary
Priority: P2

Q: How do Niagara bounds cause disappearing effects?
A: Incorrect fixed/dynamic bounds can cause frustum/cull decisions to remove an effect even though particles should be visible.
Category: Derived Interview Bank / Niagara / Debugging
Priority: P2

Q: Why are Niagara Data Channels not authoritative gameplay storage?
A: They are useful game-to-VFX/System data paths, not durable replicated gameplay state or validation authority.
Category: Derived Interview Bank / Niagara / Gameplay Boundary
Priority: P2

Q: How do you compute a signed angle safely in Unreal?
A: Use dot for magnitude, cross/axis for sign, clamp dot for `acos`, and state coordinate/axis conventions.
Category: Derived Interview Bank / Game Maths / Vectors
Priority: P1

Q: What does "shortest path" mean for quaternion interpolation?
A: Because `q` and `-q` represent the same orientation, interpolation should choose the hemisphere that avoids the long rotation path when desired.
Category: Derived Interview Bank / Game Maths / Rotation
Priority: P1

Q: What makes an A* heuristic safe?
A: It should not overestimate remaining cost for optimality; consistency further preserves efficient closed-set behavior.
Category: Derived Interview Bank / Algorithms / Pathfinding
Priority: P1

Q: Why does union-find not solve dynamic connectivity with deletions?
A: Union-find efficiently merges sets and queries connectivity, but it does not support splitting components after edge deletion.
Category: Derived Interview Bank / Algorithms / Data Structures
Priority: P2

Q: What is false sharing and why can it matter in games?
A: Independent variables on the same cache line are written by different cores, causing cache-line bouncing despite no logical sharing.
Category: Derived Interview Bank / C++ / Performance / Concurrency
Priority: P1

Q: What is the trap with `std::pmr::monotonic_buffer_resource`?
A: It can make repeated allocations cheap but releases memory in bulk; it is not a general lifetime or destructor policy.
Category: Derived Interview Bank / C++ / Allocators
Priority: P2

Q: Why is accessing UObjects from arbitrary async tasks dangerous?
A: UObject lifetime, GC, World ownership and many engine APIs are game-thread-bound unless explicitly documented otherwise.
Category: Derived Interview Bank / UE C++ / Threading / UObject
Priority: P1

Q: What makes Lua hotfix reload risky?
A: Old closures, wrappers, callbacks and schema assumptions can survive reload unless generation, invalidation and migration are explicit.
Category: Derived Interview Bank / Lua / Runtime Integration
Priority: P3

Q: Why can C# Native AOT or trimming be a problem for Unreal scripting plugins?
A: AOT/trimming can remove or restrict dynamic reflection/loading patterns that many managed binding systems rely on.
Category: Derived Interview Bank / C# / Interop / Deployment
Priority: P3

Q: Why should Runtime Data Layers not be the sole quest source of truth?
A: Data Layers control world streaming/visibility state; durable quest/save/server state should own the gameplay phase and drive layer state.
Category: Derived Interview Bank / World Partition / Gameplay State
Priority: P2

Q: What owns PCG-generated output at runtime?
A: The project must define whether output is editor-baked, generated by a PCG Component, owned by spawned Actors/components, or converted into durable gameplay state.
Category: Derived Interview Bank / PCG / Runtime Ownership
Priority: P2

Q: How do you distinguish device-lab infrastructure failure from product failure?
A: Use run manifests, artifact identity, first-cause logs and rerun/quarantine policy to classify lab versus build/game regressions.
Category: Derived Interview Bank / Device Lab / Triage
Priority: P2

Q: Why do clean-agent build failures matter?
A: They reveal undeclared dependencies, stale caches, local environment assumptions and missing staged artifacts before release.
Category: Derived Interview Bank / Build / Release Automation
Priority: P2

Q: Does the NetworkPrediction plugin mean Unreal has automatic full-game rollback?
A: No. It exposes networked simulation/rollback vocabulary and services, but each feature still needs bounded state, replay-safe inputs, authority validation, side-effect control and target-branch proof.
Category: Derived Interview Bank / Networking / NetworkPrediction
Priority: P2

Q: What is the difference between CharacterMovement prediction and full rollback?
A: CharacterMovement prediction is a specialised movement pipeline; full rollback restores a bounded simulation state and resimulates forward.
Category: Derived Interview Bank / Networking / Prediction Terminology
Priority: P1

Q: What belongs in a rollback input command?
A: Compact player intent for one frame/tick: quantised axes, action bits, sequence/frame and small validated parameters.
Category: Derived Interview Bank / Networking / Rollback State Design
Priority: P2

Q: What belongs in rollback sync state?
A: Frame-to-frame simulation truth that clients compare against authority and replay from.
Category: Derived Interview Bank / Networking / Rollback State Design
Priority: P2

Q: What belongs in rollback aux state?
A: Slower-changing simulation inputs such as approved tunables, equipment modifiers, tag/resource snapshots or table versions.
Category: Derived Interview Bank / Networking / Rollback State Design
Priority: P2

Q: Why are rollback side effects dangerous?
A: Prediction and replay can execute the same tick multiple times, duplicating effects unless they are cue-managed, deduped or moved to authority commit.
Category: Derived Interview Bank / Networking / Rollback Side Effects
Priority: P1

Q: How would you prove a rollback dash converges?
A: Force divergence, log restored frame and field diff, replay pending inputs and prove final state matches authority within tolerance.
Category: Derived Interview Bank / Networking / Rollback Debugging
Priority: P2

Q: What causes reconcile every frame in a rollback model?
A: Tick mismatch, nondeterministic simulation, missing state, overly strict tolerance, aux mismatch or unreproduced collision/query results.
Category: Derived Interview Bank / Networking / Rollback Debugging
Priority: P2

Q: Why should a rollback tick avoid reading current Actor state directly?
A: Replay of old frames can accidentally use new Actor state, causing nondeterministic divergence.
Category: Derived Interview Bank / Networking / Rollback Architecture
Priority: P2

Q: How do NetSimCue-style events differ from gameplay transactions?
A: Cues are simulation notifications/presentation events; durable gameplay transactions must be authority-owned and replay-safe.
Category: Derived Interview Bank / Networking / Cues
Priority: P2

Q: How do you decide rollback buffer length?
A: From maximum correction window, RTT/jitter/loss policy, replay cost, memory budget and gameplay tolerance.
Category: Derived Interview Bank / Networking / Performance
Priority: P2

Q: When should you reject rollback and use a simpler networking model?
A: When state is large/nondeterministic, responsiveness gain is low, side effects are irreversible, or tooling/source proof is too expensive.
Category: Derived Interview Bank / Networking / Architecture Choice
Priority: P2

Q: How does rollback interact with server rewind?
A: Rewind queries historical server state for validation; rollback restores model state and resimulates. They can coexist but are not the same.
Category: Derived Interview Bank / Networking / Lag Compensation
Priority: P2

Q: What must be logged in a rollback correction?
A: Instance, role, frame, input hash, aux version, authority sync diff, restored frame, replay window, post-replay hash and cue suppression.
Category: Derived Interview Bank / Networking / Observability
Priority: P2

Q: Why can smoothing be dangerous during rollback debugging?
A: It can hide frequent corrections, making a broken simulation look acceptable until CPU/bandwidth/edge cases fail.
Category: Derived Interview Bank / Networking / Debugging
Priority: P2

Q: How should rollback handle respawn or destruction?
A: Invalidate model instance IDs, pending inputs, cue histories and stale target identities across lifetime boundaries.
Category: Derived Interview Bank / Networking / Lifetime
Priority: P2

Q: How do you prevent Actor replication from fighting rollback sync state?
A: Assign one writer for each field and separate model-owned state, durable replicated state and presentation state.
Category: Derived Interview Bank / Networking / Actor Bridge
Priority: P2

Q: What makes a rollback simulation deterministic enough?
A: Controlled time step, identical inputs/aux state, stable query results, bounded numeric tolerance and no hidden mutable state.
Category: Derived Interview Bank / Networking / Determinism
Priority: P2

Q: What is a good first rollback proof feature?
A: A compact deterministic movement/action micro-simulation such as dash, not ragdoll combat or inventory.
Category: Derived Interview Bank / Networking / Scope Control
Priority: P2

Q: How should client-provided target data work in rollback-style actions?
A: Treat it as intent/evidence; validate range, line of sight, timing, state and plausibility on the server.
Category: Derived Interview Bank / Networking / Security
Priority: P2

Q: How do you profile rollback cost?
A: Measure command bandwidth, correction payloads, replay CPU, buffer memory, cue history and correction frequency under packet conditions.
Category: Derived Interview Bank / Networking / Performance
Priority: P2

Q: How do you answer "what proof would convince you to ship rollback?"
A: Source-backed APIs, deterministic-enough model, forced reconcile proof, side-effect safety, measured budgets, packet-condition tests and a simpler-alternative comparison.
Category: Derived Interview Bank / Networking / Production Readiness
Priority: P2

Q: Why can a reflected C++ property still fail to show up or replicate as expected?
A: Reflection, editor exposure, Blueprint access, serialisation and replication are separate systems controlled by different specifiers and registration paths.
Category: Derived Interview Bank / UE C++ / Reflection
Priority: P1

Q: What is the difference between `Outer`, owner and attachment?
A: `Outer` is containment/name/lifetime context, owner is gameplay/network ownership, and attachment is scene hierarchy.
Category: Derived Interview Bank / UObject / Lifetime
Priority: P1

Q: Why is `UPROPERTY` not a magic fix for all dangling references?
A: It helps GC discover references, but lifetime, validity, weak references, pending kill/destruction and thread access still require design.
Category: Derived Interview Bank / UObject / GC
Priority: P1

Q: How should you choose between `NewObject`, `CreateDefaultSubobject` and spawning an Actor?
A: Use `CreateDefaultSubobject` for class default subobjects in constructors, `NewObject` for UObjects/components with explicit ownership, and spawning for world Actors.
Category: Derived Interview Bank / UObject / Creation
Priority: P1

Q: Why is GameMode not a good place for client HUD state?
A: GameMode exists only on the server; client-visible match/player state belongs in replicated GameState/PlayerState or local UI models.
Category: Derived Interview Bank / Gameplay Framework / Authority
Priority: P1

Q: What makes BeginPlay ordering a fragile dependency?
A: Different Actors/components/world systems can begin at different times, and replication/possession/async loading may not be ready.
Category: Derived Interview Bank / Gameplay Framework / Lifecycle
Priority: P1

Q: When is a subsystem the wrong abstraction?
A: When it hides ownership, authority, world/player scope or lifecycle rather than clarifying a service boundary.
Category: Derived Interview Bank / Gameplay Architecture / Subsystems
Priority: P2

Q: How do BlueprintNativeEvent and BlueprintImplementableEvent differ in API design?
A: Native events provide a C++ default implementation; implementable events require Blueprint implementation and should not be mandatory for core invariants unless guarded.
Category: Derived Interview Bank / C++ / Blueprint Integration
Priority: P1

Q: Why can Blueprint pure nodes cause performance or correctness issues?
A: They may be evaluated multiple times and should be side-effect-free and cheap.
Category: Derived Interview Bank / C++ / Blueprint Integration
Priority: P1

Q: What is a safe pattern for C++ exposing mutable collections to Blueprint?
A: Expose snapshots, validated commands and change events rather than direct mutable internal arrays/maps.
Category: Derived Interview Bank / C++ / Blueprint API
Priority: P1

Q: How do you debug a `stat unit` bottleneck result?
A: Use it as a top-level split, then drill into game, draw/render, GPU, memory and subsystem-specific traces.
Category: Derived Interview Bank / Profiling / CPU-GPU Triage
Priority: P1

Q: When do you use LLM versus Memory Insights?
A: LLM gives tagged runtime memory budgets; Memory Insights helps inspect allocation behaviour and callstacks when instrumented.
Category: Derived Interview Bank / Profiling / Memory
Priority: P1

Q: Why can PSO compilation cause packaged-game hitches?
A: New graphics/compute pipeline state combinations may compile or initialise during gameplay if not collected/prepared.
Category: Derived Interview Bank / Rendering / PSO
Priority: P2

Q: What is an RDG lifetime bug?
A: Using render graph resources outside their declared graph lifetime or missing dependencies between passes.
Category: Derived Interview Bank / Rendering / RDG
Priority: P2

Q: When is Nanite the wrong answer?
A: When geometry/material/platform/animation/overdraw constraints do not match Nanite's strengths or support matrix.
Category: Derived Interview Bank / Rendering / Geometry
Priority: P2

Q: How do soft references affect cooking?
A: Soft references avoid hard load dependencies, but assets still need a cook/staging discovery path through Asset Manager, labels, maps, rules or explicit loads.
Category: Derived Interview Bank / Assets / Cooking
Priority: P1

Q: Why are redirectors dangerous if left unmanaged?
A: They hide moved asset paths in editor but can create broken references, cook surprises and source-control churn if not fixed.
Category: Derived Interview Bank / Assets / Source Control
Priority: P2

Q: Why can DDC hide production build problems?
A: Warm local/cache state can mask missing generation steps, bad shaders/assets or slow first-run costs.
Category: Derived Interview Bank / Assets / Derived Data
Priority: P2

Q: What is the first thing to check when an editor plugin fails in packaged builds?
A: Whether runtime modules, dependencies, staging rules and platform allow the plugin outside editor.
Category: Derived Interview Bank / Build / Plugins
Priority: P1

Q: What makes BuildGraph useful beyond running a build command?
A: It encodes repeatable build/cook/package/test/artifact graphs with declared dependencies and machine-usable steps.
Category: Derived Interview Bank / Build / Automation
Priority: P2

Q: What should an AI Perception memory model store?
A: Stimulus source, sense, location, age, strength, success/loss, confidence and gameplay interpretation.
Category: Derived Interview Bank / AI / Perception
Priority: P1

Q: Why can Behaviour Tree Observer Aborts be better than polling services?
A: They let relevant condition changes interrupt branches without high-frequency polling everywhere.
Category: Derived Interview Bank / AI / Behaviour Trees
Priority: P1

Q: What is the performance trap with EQS?
A: Generating and testing many candidate points with expensive traces/contexts can scale badly across many AI.
Category: Derived Interview Bank / AI / EQS
Priority: P1

Q: Why can Mass archetype count explode?
A: Too many fragment/tag/shared-fragment combinations split entities into many archetypes and reduce chunk efficiency.
Category: Derived Interview Bank / MassEntity / ECS
Priority: P2

Q: Why are structural changes during Mass iteration risky?
A: Composition changes can invalidate iteration assumptions and require deferred commands/migrations.
Category: Derived Interview Bank / MassEntity / ECS
Priority: P2

Q: Why does GAS ASC placement matter?
A: ASC lifetime and owner/avatar choice determine ability persistence, respawn behaviour, replication audience and init order.
Category: Derived Interview Bank / GAS / Architecture
Priority: P2

Q: What does `FScopedPredictionWindow` not make safe?
A: It does not make arbitrary gameplay side effects reversible or authoritative.
Category: Derived Interview Bank / GAS / Prediction
Priority: P2

Q: Why does Enhanced Input mapping context priority matter?
A: Multiple contexts can bind overlapping actions; priority and local-player ownership determine which mappings win.
Category: Derived Interview Bank / Input / Enhanced Input
Priority: P1

Q: Why can animation state machines miss important gameplay cancellation?
A: Animation transitions can be interrupted, blended or skipped; gameplay authority needs its own cancellation/timeout path.
Category: Derived Interview Bank / Animation / Gameplay Boundary
Priority: P2

Q: Why are linked animation layers useful but risky?
A: They modularise animation logic, but missing layers, stale interfaces or mismatched skeleton assumptions can break pose output.
Category: Derived Interview Bank / Animation / Architecture
Priority: P2

Q: What does physics substepping solve and not solve?
A: It can improve simulation stability under variable frame time, but it does not fix bad ownership, constraints, collision setup or networking.
Category: Derived Interview Bank / Physics / Simulation
Priority: P2

Q: Why do constraint frames matter?
A: The constraint solves relative motion around defined frames; wrong frames make limits and motors behave incorrectly.
Category: Derived Interview Bank / Physics / Constraints
Priority: P2

Q: Why should ListView entries clean up in row release?
A: Virtualised rows are recycled; stale bindings, async icon loads or selection state can leak into a different item.
Category: Derived Interview Bank / UI / Virtualisation
Priority: P1

Q: Why can CommonUI and Enhanced Input conflict in modal screens?
A: Both manage input context/routing; stale contexts or wrong local-player focus can route actions to gameplay behind the modal.
Category: Derived Interview Bank / UI / Input Routing
Priority: P1

Q: When is Quartz useful for game audio?
A: For sample-accurate or musically scheduled audio events that need better timing than ordinary game-thread timers.
Category: Derived Interview Bank / Audio / Timing
Priority: P2

Q: Why can MetaSounds still be expensive?
A: Procedural audio graphs consume CPU/voices/resources and can scale badly if spawned or parameterised without budget.
Category: Derived Interview Bank / Audio / Performance
Priority: P2

Q: Why can Niagara GPU collisions be misleading for gameplay?
A: Particle collision can be approximate, presentation-oriented and GPU-delayed; gameplay collision should use authoritative gameplay/physics queries.
Category: Derived Interview Bank / VFX / Gameplay Boundary
Priority: P2

Q: How should runtime PCG interact with navigation and collision?
A: Generated output needs explicit policy for collision, navmesh updates, authority, cleanup, save and replication.
Category: Derived Interview Bank / PCG / Runtime Systems
Priority: P2

Q: Why is World Partition streaming readiness not gameplay readiness?
A: Loaded cells/actors may not have completed gameplay initialisation, replication, async assets or dependency binding.
Category: Derived Interview Bank / World Partition / Gameplay
Priority: P2

Q: Why should Lua or C# scripting expose semantic APIs instead of raw UObject ownership?
A: Semantic APIs preserve lifetime, authority, thread and version boundaries across language runtimes.
Category: Derived Interview Bank / Scripting / Architecture
Priority: P2

Q: When should you prefer value semantics over UObject references?
A: Prefer values for small deterministic data without identity, reflection lifetime, polymorphic UObject behaviour or shared ownership needs.
Category: Derived Interview Bank / C++ / UE Architecture
Priority: P1

Q: What does move semantics not guarantee?
A: Moving does not guarantee no work, no allocation, or a useful value remaining in the moved-from object.
Category: Derived Interview Bank / Standard C++ / Move Semantics
Priority: P1

Q: Why can `std::function` be costly in hot code?
A: It type-erases calls and may allocate/copy captures, adding overhead versus templates, function refs or direct delegates.
Category: Derived Interview Bank / C++ / Callables
Priority: P2

Q: What does release/acquire ordering solve?
A: It publishes writes before a release store to a thread that observes them through an acquire load.
Category: Derived Interview Bank / C++ / Atomics
Priority: P2

Q: Why does ODR matter in Unreal modules?
A: Duplicate or inconsistent definitions across modules can cause link failures or undefined behaviour.
Category: Derived Interview Bank / C++ / Build
Priority: P2

Q: How do you choose between `TArray`, `TSet` and `TMap`?
A: Choose by access pattern, ordering, mutation frequency, memory/cache cost and identity requirements.
Category: Derived Interview Bank / UE Containers / Algorithms
Priority: P1

Q: Why can `FText` be the wrong type for gameplay identity?
A: `FText` is for localised display text, not stable IDs or logic keys.
Category: Derived Interview Bank / UE Types / Localisation
Priority: P1

Q: What makes Data Assets useful for gameplay tuning?
A: They separate designer-authored definitions from runtime instance state.
Category: Derived Interview Bank / Data-Driven Gameplay
Priority: P1

Q: Why can delegates leak or crash after object destruction?
A: Bound callbacks can outlive the object or capture stale state unless unbound or weak/lifetime-checked.
Category: Derived Interview Bank / UE C++ / Delegates
Priority: P1

Q: Why can RPC reliability harm multiplayer responsiveness?
A: Reliable RPCs queue and must be delivered in order, so overuse can block newer important traffic.
Category: Derived Interview Bank / Networking / RPCs
Priority: P1

Q: What does OnRep ordering not guarantee?
A: It does not guarantee separate replicated properties arrive or execute in the semantic order your gameplay assumes.
Category: Derived Interview Bank / Networking / Replication
Priority: P1

Q: Why can dormancy hide state changes?
A: Dormant Actors may not send updates until properly woken/flushed before mutation.
Category: Derived Interview Bank / Networking / Dormancy
Priority: P1

Q: What makes Fast Array identity important?
A: Delta replication needs stable item identity to distinguish add, change, remove and reorder.
Category: Derived Interview Bank / Networking / Fast Array
Priority: P2

Q: What problem does Replication Graph solve?
A: It scales per-connection relevance/list building for many Actors using graph nodes and persistent policy data.
Category: Derived Interview Bank / Networking / Scalability
Priority: P2

Q: Why should Iris be discussed carefully in UE5 interviews?
A: It is a newer opt-in/experimental replication system in the docs, not automatically every UE5 project's baseline.
Category: Derived Interview Bank / Networking / Iris
Priority: P2

Q: How do you debug a disappearing replicated subobject?
A: Check owner replication, subobject registration/lifetime, mapping timing, GC references, conditions and client OnRep validity.
Category: Derived Interview Bank / Networking / Subobjects
Priority: P2

Q: Why should gameplay tags not replace all state?
A: Tags are excellent semantic labels/queries, but numeric values, timers, ownership and durable state still need proper models.
Category: Derived Interview Bank / GAS / Tags
Priority: P2

Q: How do you design an AttributeSet invariant?
A: Define canonical bounds and repair points for base/current values, max changes and replication/prediction paths.
Category: Derived Interview Bank / GAS / Attributes
Priority: P2

Q: Why should Ability Tasks clean up symmetrically?
A: Tasks bind delegates/timers/targets and may end by completion, cancel, ability end or avatar destruction.
Category: Derived Interview Bank / GAS / Ability Tasks
Priority: P2

Q: Why should input buffering record commands rather than raw key state only?
A: Gameplay needs semantic intent with timing, context and validation, not just current hardware state.
Category: Derived Interview Bank / Input / Gameplay
Priority: P2

Q: How do you debug animation-thread unsafe data access?
A: Snapshot game-thread state into anim-safe variables and avoid arbitrary UObject/gameplay reads during threaded evaluation.
Category: Derived Interview Bank / Animation / Threading
Priority: P2

Q: Why does Motion Warping still need gameplay validation?
A: It adjusts animation motion toward targets, but gameplay authority must validate target, timing and collision.
Category: Derived Interview Bank / Animation / Motion Warping
Priority: P2

Q: What is the difference between query collision and physics simulation?
A: Queries ask spatial questions such as traces/overlaps; simulation resolves bodies, contacts, impulses and constraints.
Category: Derived Interview Bank / Physics / Collision
Priority: P1

Q: Why can CCD be the wrong fix for tunnelling?
A: CCD has cost/limitations and may not address wrong movement method, collision shape, timestep or authority path.
Category: Derived Interview Bank / Physics / Collision
Priority: P2

Q: Why is world-space UI expensive at scale?
A: Many widget components add update, layout, render target/batch and occlusion costs.
Category: Derived Interview Bank / UI / World-Space Widgets
Priority: P2

Q: How do you make UI text robust for localisation?
A: Use `FText`, flexible layout, pseudo-localisation, truncation/wrapping policy and non-text-dependent affordances.
Category: Derived Interview Bank / UI / Localisation
Priority: P1

Q: Why should audio not be the only critical feedback channel?
A: Audio can be muted, unavailable, stolen by concurrency or inaccessible to some players.
Category: Derived Interview Bank / Audio / Accessibility
Priority: P1

Q: Why should long music streams be tested in cooked builds?
A: Cooked streaming/cache/chunk behaviour can differ from editor and reveal stalls or missing staged data.
Category: Derived Interview Bank / Audio / Streaming
Priority: P2

Q: Why can Niagara fixed bounds be both too small and too large?
A: Too small culls visible particles; too large keeps invisible work alive and hurts culling efficiency.
Category: Derived Interview Bank / VFX / Niagara Bounds
Priority: P1

Q: What is the role of Niagara Effect Types?
A: They group systems for budget, significance and scalability policy.
Category: Derived Interview Bank / VFX / Scalability
Priority: P2

Q: Why can A* return bad paths in games even with correct code?
A: The graph, costs, heuristic, movement constraints or dynamic obstacles may not model the actual game problem.
Category: Derived Interview Bank / Algorithms / Pathfinding
Priority: P1

Q: When is a spatial hash a poor fit?
A: When object sizes/density vary widely, queries are irregular, or cell size creates too many collisions/empty checks.
Category: Derived Interview Bank / Algorithms / Spatial Structures
Priority: P2

Q: Why should matrix/vector convention be stated in gameplay math answers?
A: Row/column, handedness and multiplication order affect transform composition and bug diagnosis.
Category: Derived Interview Bank / Game Math / Transformations
Priority: P1

Q: Why can Euler angles fail in camera or aiming code?
A: They can suffer order dependence, wrapping and gimbal-like singularities; quaternions/vectors may be better for interpolation/composition.
Category: Derived Interview Bank / Game Math / Rotation
Priority: P1

Q: Why is custom AI sense registration a lifecycle problem?
A: Sense config, stimuli sources, perception components, ageing and listener registration all have lifecycle and world-scope concerns.
Category: Derived Interview Bank / AI / Custom Senses
Priority: P2

Q: Why can ZoneGraph/Mass crowd routes fail at runtime?
A: Lane data, tags, fragments, processor order, spawn position and runtime availability must all align.
Category: Derived Interview Bank / MassAI / ZoneGraph
Priority: P2

Q: Why do StateTree tasks need explicit data-flow design?
A: Inputs, outputs, instance data and external data determine whether tasks observe current state or stale assumptions.
Category: Derived Interview Bank / StateTree / Gameplay AI
Priority: P2

Q: What makes Smart Object claims fail in multiplayer?
A: Reservation/claim authority, slot state, actor lifetime and replicated visibility must be coordinated.
Category: Derived Interview Bank / Smart Objects / Multiplayer
Priority: P2

Q: Why can packaged mobile performance differ radically from PIE?
A: RHI, shader formats, device profiles, thermal state, packaging, IO, resolution and platform services differ from editor.
Category: Derived Interview Bank / Platform / Mobile Profiling
Priority: P1

Q: What should a crash report contain to be actionable?
A: Symbols/build ID, callstack, logs, platform, scenario markers, repro steps and artifact identity.
Category: Derived Interview Bank / Build / Crash Triage
Priority: P1

Q: Why should console and Apple profiling be source-safe in public notes?
A: Methodology can be discussed publicly, but proprietary platform counters/devkit details may be NDA-controlled.
Category: Derived Interview Bank / Platform / Profiling
Priority: P2

Q: What is a good acceptance criterion for a hands-on Unreal project?
A: It proves behaviour, failure injection, profiling/debug evidence and source/version caveats, not just feature completion.
Category: Derived Interview Bank / Interview Projects / Evidence
Priority: P1

Q: How should you answer "tell me about a hard bug" in an Unreal interview?
A: Describe symptom, scope, hypotheses, evidence, root cause, fix, verification and prevention.
Category: Derived Interview Bank / Interviewing / Behavioural Technical
Priority: P1

Q: How should you handle uncertainty in a version-sensitive Unreal interview answer?
A: State the stable concept, label the version-sensitive part and explain how you would verify the target branch.
Category: Derived Interview Bank / Interviewing / Version Sensitivity
Priority: P1

Q: Why can editor-only data accidentally enter runtime code?
A: Editor modules, metadata, validation helpers and asset tooling can be referenced from runtime modules unless boundaries are explicit.
Category: Derived Interview Bank / Build / Runtime Boundary
Priority: P1

Q: What makes a gameplay tag hierarchy maintainable?
A: Clear naming, ownership, validation, limited ambiguity and documented semantics for broad versus specific tags.
Category: Derived Interview Bank / Gameplay Tags / Data Design
Priority: P2

Q: How should you design a cooldown system without GAS?
A: Centralise authority, time source, identifiers, prediction policy, UI projection and save/replication semantics.
Category: Derived Interview Bank / Gameplay Systems / Cooldowns
Priority: P2

Q: Why can save/load break event-driven quest systems?
A: Events that already happened are gone after load unless durable quest state is restored and reconciled.
Category: Derived Interview Bank / Save Systems / Quests
Priority: P1

Q: What is the difference between command and event in gameplay architecture?
A: A command asks a system to do something; an event reports that something happened.
Category: Derived Interview Bank / Gameplay Architecture / Messaging
Priority: P1

Q: Why can object pooling create stale-state bugs?
A: Reused objects can retain bindings, timers, parameters, owner references or replicated/presentation state if reset is incomplete.
Category: Derived Interview Bank / Gameplay Architecture / Object Pool
Priority: P2

Q: What makes a service locator dangerous?
A: It can hide dependencies, scope, lifetime and authority behind global lookups.
Category: Derived Interview Bank / Gameplay Architecture / Dependencies
Priority: P2

Q: Why should validation run in editor and CI?
A: Editor-only validation catches authoring mistakes early, while CI prevents invalid content from reaching packages.
Category: Derived Interview Bank / Tools / Data Validation
Priority: P1

Q: When should you write a commandlet?
A: For repeatable headless editor/asset/build operations that should run in automation.
Category: Derived Interview Bank / Tools / Commandlets
Priority: P2

Q: Why can Python editor scripts be unsafe for production data migration?
A: Without transaction, validation, source-control and restart recovery, batch edits can corrupt or partially migrate assets.
Category: Derived Interview Bank / Tools / Editor Scripting
Priority: P2

Q: What does a rendering optimisation decision memo include?
A: Scenario, bottleneck evidence, changed content/settings, before/after captures, visual impact and platform risk.
Category: Derived Interview Bank / Rendering / Performance Process
Priority: P1

Q: Why can render targets become hidden memory/performance costs?
A: They consume GPU memory/bandwidth and may add extra passes, resolves and update work.
Category: Derived Interview Bank / Rendering / Render Targets
Priority: P2

Q: What is shader permutation explosion?
A: Many static switches/features/material variants generate large numbers of shader combinations to compile and ship.
Category: Derived Interview Bank / Rendering / Shaders
Priority: P2

Q: Why can HLOD be a build-system problem, not just a world-building feature?
A: HLOD generation creates derived assets that must be built, validated, versioned and packaged reproducibly.
Category: Derived Interview Bank / World Partition / Build Pipeline
Priority: P2

Q: Why can OFPA create merge noise?
A: One File Per Actor improves parallel editing but creates many actor asset files whose churn needs naming, ownership and review discipline.
Category: Derived Interview Bank / World Building / Source Control
Priority: P2

Q: Why can Android thermal throttling invalidate a benchmark?
A: Performance changes over time as the device heats and changes CPU/GPU frequency or scheduling policy.
Category: Derived Interview Bank / Platform / Mobile Profiling
Priority: P1

Q: What should a platform device profile change prove?
A: It improves the target scenario on target hardware without unacceptable visual, memory or compatibility regressions.
Category: Derived Interview Bank / Platform / Scalability
Priority: P1

Q: Why should automation distinguish flaky from deterministic failures?
A: Deterministic failures should block; flaky infrastructure or tests need quarantine/triage without hiding real regressions.
Category: Derived Interview Bank / Automation / CI
Priority: P1

Q: Why can multiplayer tests pass in PIE but fail in multi-process dedicated server?
A: PIE can share process/editor assumptions; dedicated server exposes real net mode, ownership, timing, packaging and authority boundaries.
Category: Derived Interview Bank / Networking / Testing
Priority: P1

Q: How do you know whether an optimisation moved the bottleneck?
A: Compare before/after full frame breakdown, not just one improved metric.
Category: Derived Interview Bank / Profiling / Method
Priority: P1

Q: Why is "just make it async" not a performance strategy?
A: Async work still costs CPU/memory/synchronisation and may add latency, contention or unsafe UObject access.
Category: Derived Interview Bank / Performance / Async
Priority: P1

Q: Why should generated code be part of source control policy?
A: Teams must decide which generated artifacts are reproducible build outputs and which are authored assets that must be reviewed/versioned.
Category: Derived Interview Bank / Build / Generated Artifacts
Priority: P2

Q: What is the simplest useful portfolio evidence packet?
A: A short problem statement, source/version scope, before/after evidence, failure injection, profiler/debug captures and decision memo.
Category: Derived Interview Bank / Interview Projects / Portfolio
Priority: P1

## Target-Proof, Build, World Partition, and Networking Expansion

Q: What is the main purpose of a packaged performance manifest?
A: To bind metrics and artifacts to an exact build, target, scenario, settings, thresholds and owner.
Category: Packaged Proof / Performance
Priority: P1

Q: Why is a CSV file without build identity weak evidence?
A: It cannot prove which package, platform, profile or scenario produced the numbers.
Category: Packaged Proof / Evidence
Priority: P1

Q: What is the first thing to verify when packaged performance differs from editor performance?
A: The exact package identity, active profile/CVars, scenario and target environment.
Category: Packaged Proof / Debugging
Priority: P1

Q: What does CSV profiling prove best?
A: Repeatable metric drift or regression signals across controlled runs.
Category: Profiling / CSV
Priority: P1

Q: What does CSV profiling not prove by itself?
A: The exact causal root of a failing frame or memory regression.
Category: Profiling / CSV
Priority: P1

Q: What should happen after a CSV gate fails?
A: Capture the smallest causal trace or platform/GPU/memory artifact for the failing window.
Category: Profiling / Workflow
Priority: P1

Q: Why should a performance gate include warm-up?
A: To separate startup/transient effects from the intended measurement window.
Category: Profiling / Gates
Priority: P1

Q: Why should a gate include readiness markers?
A: To avoid measuring loading or setup before the scenario has actually started.
Category: Profiling / Gates
Priority: P1

Q: What is a better blocking metric than average FPS?
A: Repeated P95/P99 frame time, hitch count, first-interactive time or memory growth tied to a scenario.
Category: Profiling / Gates
Priority: P1

Q: Why must active Device Profile be logged in a packaged run?
A: Source profile files do not prove which profile and CVars were selected at runtime.
Category: Profiling / Device Profiles
Priority: P1

Q: What can dynamic resolution hide in a performance test?
A: A GPU regression by lowering internal resolution rather than fixing the workload.
Category: Profiling / Rendering
Priority: P2

Q: What can VSync hide in a performance test?
A: Actual frame-time variation and GPU/CPU headroom beyond the cap.
Category: Profiling / Method
Priority: P2

Q: Why should packaged tests record power or thermal state on mobile?
A: Sustained performance can change as devices heat and throttle.
Category: Platform / Mobile Profiling
Priority: P1

Q: What does LLM help identify?
A: Memory ownership by subsystem/tag at budget checkpoints.
Category: Profiling / Memory
Priority: P1

Q: What does Memory Insights help identify?
A: Allocation lifetime, churn, growth, callstacks and leak-like patterns.
Category: Profiling / Memory
Priority: P1

Q: What is a capacity memory problem?
A: Stable resident memory exceeds the budget for a checkpoint or tier.
Category: Profiling / Memory
Priority: P1

Q: What is a growth memory problem?
A: Repeated cycles do not return near baseline memory.
Category: Profiling / Memory
Priority: P1

Q: What is a churn memory problem?
A: Short-lived allocation/free activity causes overhead or hitches even if live memory is acceptable.
Category: Profiling / Memory
Priority: P2

Q: Why is "increase the pool" often a weak memory fix?
A: It may hide ownership, retention or content-budget problems rather than solving them.
Category: Profiling / Memory
Priority: P1

Q: What is a good first-interactive definition?
A: The moment the player can safely see, control and interact with the intended state, not merely map load completion.
Category: Loading / Performance
Priority: P2

Q: Why can async loading still hitch?
A: Completion may trigger game-thread registration, callbacks, component creation, shader/PSO use or dependent loads.
Category: Assets / Async Loading
Priority: P1

Q: What does BuildGraph add beyond a long command line?
A: Explicit parameters, nodes, dependencies, agents, labels and artifact flow.
Category: Build / BuildGraph
Priority: P2

Q: When is direct RunUAT usually enough?
A: For a simple documented local or single-lane package proof without complex artifact dependencies.
Category: Build / RunUAT
Priority: P2

Q: When does BuildGraph become more valuable?
A: When builds need multiple targets, agents, artifacts, symbols, tests, publish steps and release visibility.
Category: Build / BuildGraph
Priority: P2

Q: What should a BuildGraph dependency prevent?
A: Downstream nodes such as performance or crash proof using stale or missing package/symbol artifacts.
Category: Build / BuildGraph
Priority: P2

Q: Why is a green BuildGraph not automatically a valid release?
A: It may have tested stale artifacts, skipped expected content or lacked launch/perf/crash evidence.
Category: Build / Release Proof
Priority: P1

Q: What should a release artifact manifest include?
A: Build ID, engine/project revision, target/config/platform, command lines, package path, symbols and validation results.
Category: Build / Artifacts
Priority: P1

Q: Why are symbols from a similar build not enough?
A: Crash diagnosis requires symbols that match the exact shipped binary/build ID.
Category: Build / Crash Proof
Priority: P1

Q: What does a forced packaged crash prove?
A: Crash reporting, build ID, logs and symbols work before real production crashes occur.
Category: Build / Crash Proof
Priority: P1

Q: Why should a forced crash run outside the editor?
A: Editor crash paths and symbols do not prove packaged client/server crash handling.
Category: Build / Crash Proof
Priority: P1

Q: What is the first useful question in a RunUAT failure?
A: Which phase failed first: build, cook, stage, package, deploy, run, test, symbols or crash route?
Category: Build / Debugging
Priority: P1

Q: Why is the final AutomationTool error often not the root cause?
A: It commonly wraps an earlier build, cook, stage or runtime error.
Category: Build / Debugging
Priority: P1

Q: What is a cook failure usually about?
A: Platform-specific content transformation, inclusion rules, dependencies or unsupported assets.
Category: Build / Cooking
Priority: P1

Q: What is a stage failure usually about?
A: Assembling runtime files, cooked content, plugin files and dependencies into the staged layout.
Category: Build / Staging
Priority: P1

Q: What is a package/archive failure usually about?
A: Producing and retaining the distributable container/layout and related release artifacts.
Category: Build / Packaging
Priority: P1

Q: Why is "cook all content" a weak fix?
A: It hides dependency rules, bloats packages and can break chunking/platform budgets.
Category: Assets / Cooking
Priority: P1

Q: What should missing packaged assets make you inspect?
A: Asset Manager rules, map lists, labels, soft references, redirectors, cook/stage manifests and plugin content.
Category: Assets / Packaging
Priority: P1

Q: What proves a third-party DLL or shared library is staged correctly?
A: A clean-machine/package launch without relying on globally installed or editor-adjacent files.
Category: Build / Third-Party
Priority: P2

Q: Why should plugin runtime/editor module boundaries be package-tested?
A: Editor targets can compile while Game/Shipping packages fail due to editor-only dependencies or asset classes.
Category: Build / Plugins
Priority: P1

Q: What can unity builds hide?
A: Missing includes, ODR problems and source-order assumptions that non-unity clean builds expose.
Category: C++ / Build
Priority: P1

Q: What is UHT's job?
A: Parse supported Unreal-reflected declarations and generate metadata/glue before C++ compilation.
Category: Build / UHT
Priority: P0

Q: What is UBT's job?
A: Resolve targets/modules/dependencies and drive compilation/linking with generated code.
Category: Build / UBT
Priority: P0

Q: What is the main risk of non-local static initialisation in Unreal modules?
A: It can run before engine systems, reflection, config or plugin dependencies are ready.
Category: C++ / Startup
Priority: P1

Q: What should World Partition HLOD proof contain?
A: Builder command/log, correct map, output freshness, packaged inclusion, visual acceptance and performance evidence.
Category: World Partition / HLOD
Priority: P2

Q: Why is HLOD builder success not complete proof?
A: It does not prove output freshness, packaged inclusion, visual quality or runtime performance.
Category: World Partition / HLOD
Priority: P2

Q: What can stale HLOD output cause?
A: Wrong visuals, missing geometry, incorrect package content or misleading performance results.
Category: World Partition / HLOD
Priority: P2

Q: What does HLOD primarily reduce?
A: Distant representation cost such as primitives, draws, material/shadow work or streaming complexity.
Category: World Partition / HLOD
Priority: P2

Q: What does HLOD not automatically fix?
A: Near-field actor activation, collision/nav setup, gameplay authority or first-use shader/PSO hitches.
Category: World Partition / HLOD
Priority: P2

Q: What should a fast-travel readiness check include?
A: Streaming completion plus project-specific collision, nav, Data Layer and gameplay actor readiness.
Category: World Partition / Streaming
Priority: P1

Q: Why is a blind delay a weak fast-travel fix?
A: It guesses timing instead of checking actual streaming and gameplay readiness.
Category: World Partition / Streaming
Priority: P1

Q: What should packaged traversal logs show?
A: Scenario boundaries, cell/layer/readiness state, HLOD transition and fast-travel commit/timeout.
Category: World Partition / Proof
Priority: P2

Q: What is the role of Runtime Data Layers?
A: Present and load/activate world phases, not serve as the only durable authority.
Category: World Partition / Data Layers
Priority: P2

Q: Where should durable world-state truth live?
A: In save/server/quest/state systems with stable IDs, not only in unloadable actors or Data Layers.
Category: World Partition / Architecture
Priority: P2

Q: Why can manager hard references break spatial loading?
A: They can keep spatial actors or bundles loaded through reference edges.
Category: World Partition / References
Priority: P2

Q: How do you represent unloadable actor state safely?
A: Store stable IDs and durable state outside the actor, then reconstruct presentation when loaded.
Category: World Partition / Persistence
Priority: P2

Q: What is OFPA good for?
A: Reducing level-file conflicts by storing actors externally during editor work.
Category: World Partition / OFPA
Priority: P2

Q: What is OFPA not a substitute for?
A: Source-control validation, reviewability, clean package tests or runtime streaming policy.
Category: World Partition / OFPA
Priority: P2

Q: Why can a Level Blueprint reference be dangerous in World Partition?
A: It can create hard references that affect loading or always-loaded behaviour.
Category: World Partition / References
Priority: P2

Q: What should PCG generated output record?
A: Seed, graph version, input identity, owner, cleanup policy, Data Layer/HLOD policy and runtime/cook behaviour.
Category: PCG / Pipeline
Priority: P2

Q: Why separate PCG graph cost from output cost?
A: Spawned actors, components, instances, collision, nav and rendering can dominate over graph execution.
Category: PCG / Performance
Priority: P2

Q: Why are PCG metadata domains a debugging issue?
A: Nodes may read attributes from a different domain than the one where values were authored.
Category: PCG / Debugging
Priority: P3

Q: What makes a reusable POI production-ready?
A: Authored structure, source-control policy, packaged behaviour, stable gameplay identity, Data Layer/HLOD/cook validation.
Category: World Building / POI
Priority: P2

Q: When does Packed Level Blueprint fit best?
A: Dense, mostly static visual arrangements where packed representation is acceptable.
Category: World Building / Level Instances
Priority: P2

Q: What should mutable gameplay POIs preserve?
A: Stable identity, save/replication state and runtime authority independent of reusable visual composition.
Category: World Building / POI
Priority: P2

Q: What is the first networking authority principle?
A: Server owns gameplay truth; clients request, predict or present subject to validation/reconciliation.
Category: Networking / Authority
Priority: P0

Q: What is actor ownership not the same as?
A: Authority, attachment, outer/lifetime ownership or guaranteed relevance.
Category: Networking / Ownership
Priority: P0

Q: What does owning connection affect?
A: RPC permissions and owner-based replication conditions/audience.
Category: Networking / Ownership
Priority: P0

Q: Why is reliable RPC not durable state?
A: It delivers calls, but late join/relevancy/dormancy require current state representation.
Category: Networking / RPCs
Priority: P1

Q: What belongs in replicated properties instead of old RPC history?
A: Durable current state needed by late joiners or newly relevant clients.
Category: Networking / State
Priority: P1

Q: What should a Server RPC from a pickup make you check?
A: Whether the actor is owned by the caller or request must route through an owned actor/controller.
Category: Networking / RPCs
Priority: P1

Q: What does NetUpdateFrequency trade?
A: Bandwidth/server update cost versus staleness, smoothness and correction behaviour.
Category: Networking / Bandwidth
Priority: P2

Q: What does dormancy require before changing replicated state?
A: Wake or flush appropriate actors before mutating state clients must observe.
Category: Networking / Dormancy
Priority: P2

Q: What happens if you mutate dormant state without waking?
A: Clients may remain stale until dormancy/relevancy changes or another update occurs.
Category: Networking / Dormancy
Priority: P2

Q: What makes Fast Array useful?
A: Delta-oriented replication for changing collections with stable item identity and dirty marking.
Category: Networking / Fast Array
Priority: P1

Q: What are common Fast Array bugs?
A: Missing dirty marks, unstable item IDs, bad removal hooks, late-join gaps and authority confusion.
Category: Networking / Fast Array
Priority: P1

Q: What must Fast Array proof compare?
A: Correct add/change/remove behaviour and bandwidth versus simple array replication under representative mutations.
Category: Networking / Fast Array
Priority: P1

Q: What is the correct boundary for CharacterMovement saved moves?
A: Deterministic movement-affecting client input/state needed for server reproduction and replay.
Category: Networking / Saved Moves
Priority: P1

Q: What should not be hidden inside saved moves?
A: Arbitrary gameplay truth such as damage, inventory, rewards or non-deterministic side effects.
Category: Networking / Saved Moves
Priority: P1

Q: Why can saved move combining break custom movement?
A: Combining moves with different state can erase one-frame edges or replay wrong input.
Category: Networking / Saved Moves
Priority: P2

Q: What should `CanCombineWith` preserve?
A: Deterministic equivalence for all movement-affecting custom state.
Category: Networking / Saved Moves
Priority: P2

Q: What is a good saved-move sprint proof?
A: Client predicts sprint, server reproduces/validates it, false client speed is corrected under latency.
Category: Networking / Movement Prediction
Priority: P1

Q: What is a good saved-move dash proof?
A: A short dash edge survives compressed flags/combine/replay and does not double-trigger side effects.
Category: Networking / Movement Prediction
Priority: P2

Q: What is RepGraph best justified by?
A: Measured scale/relevancy/list-building costs that actor policy can reduce.
Category: Networking / RepGraph
Priority: P2

Q: What actor policies should a RepGraph design classify?
A: Spatial, global, owner-only, team, always-relevant, dormant, frequency and temporary actors.
Category: Networking / RepGraph
Priority: P2

Q: Why is RepGraph not a generic performance switch?
A: It adds architecture and policy complexity that must match measured replication workload.
Category: Networking / RepGraph
Priority: P2

Q: What first question should Iris answers ask?
A: Is Iris enabled and supported for this target branch/project feature?
Category: Networking / Iris
Priority: P3

Q: Why is Iris branch-sensitive?
A: Status, APIs, fragments, filtering and subobject requirements can change by UE version.
Category: Networking / Iris
Priority: P3

Q: What must dynamic replicated subobjects prove?
A: Correct server ownership, registration, lifetime, mutation, removal, late join and GC safety.
Category: Networking / Subobjects
Priority: P2

Q: Why can replicated subobjects go stale after removal?
A: Clients or registered lists may retain references unless lifetime/removal is handled safely.
Category: Networking / Subobjects
Priority: P2

Q: What does NetworkPrediction plugin public docs prove?
A: Public module/API vocabulary exists, not a ready-to-copy target-branch integration recipe.
Category: Networking / NetworkPrediction
Priority: P3

Q: What are core NetworkPrediction model concepts?
A: Input/cmd, sync state, aux state, simulation tick, reconcile/rollback, cues and smoothing.
Category: Networking / NetworkPrediction
Priority: P3

Q: What belongs in rollback sync state?
A: Deterministic authoritative simulation state needed for rewind and resimulation.
Category: Networking / Rollback
Priority: P3

Q: What belongs in rollback input/cmd state?
A: Per-tick player or system commands driving the simulation.
Category: Networking / Rollback
Priority: P3

Q: What belongs in rollback aux state?
A: Slower-changing configuration or parameters that affect simulation but are not per-tick commands.
Category: Networking / Rollback
Priority: P3

Q: Why are side effects dangerous in rollback?
A: Resimulation can execute the same logical tick multiple times and duplicate irreversible effects.
Category: Networking / Rollback
Priority: P3

Q: How do cue keys help rollback?
A: They deduplicate presentation for the same logical event across replay/correction.
Category: Networking / Rollback
Priority: P3

Q: What is a rollback compile proof?
A: A minimal target-branch build that verifies plugin enablement, model definitions and required integration points.
Category: Networking / NetworkPrediction
Priority: P3

Q: What is a valid rejection memo for NetworkPrediction?
A: Evidence that target branch/project cannot support it, plus fallback design and future verification trigger.
Category: Networking / NetworkPrediction
Priority: P3

Q: Why is owner-only replication useful?
A: It limits private state to the owning connection and can reduce bandwidth/exposure.
Category: Networking / Conditions
Priority: P2

Q: What is the risk of owner-only replication?
A: Other clients, spectators or late joiners will not receive that state unless represented elsewhere.
Category: Networking / Conditions
Priority: P2

Q: What causes unresolved replicated object references?
A: Timing, relevancy, dormancy, unloaded content, unreplicated targets, subobject registration or NetGUID mapping issues.
Category: Networking / References
Priority: P2

Q: When are stable IDs better than replicated pointers?
A: When referenced actors may be unloaded, not yet relevant, destroyed or represented by durable save/world state.
Category: Networking / References
Priority: P2

Q: What should late join never rely on alone?
A: Past multicast/RPC events that were not represented as durable current state.
Category: Networking / Late Join
Priority: P1

Q: What is a good late-join test?
A: Join after world/inventory/door/objective state changed and verify current state reconstruction.
Category: Networking / Late Join
Priority: P1

Q: How do you debug a network correction pop?
A: Log input/saved moves, server authoritative state, correction size/timing, smoothing and custom movement flags.
Category: Networking / Debugging
Priority: P1

Q: Why is increasing smoothing not always the fix for correction pops?
A: It can hide symptoms while non-deterministic client/server movement remains broken.
Category: Networking / Debugging
Priority: P1

Q: What should a hit validation server check include?
A: Ability/montage state, timing, range, geometry/history, cooldown and duplicate-hit prevention.
Category: Networking / Combat
Priority: P1

## Specialist Systems Flashcard Expansion

Q: What should ASC placement be based on?
A: Lifetime, owner/avatar relationship, respawn, replication audience, attributes and grant cleanup.
Category: GAS / Architecture
Priority: P2

Q: When does ASC-on-PlayerState often fit?
A: Player-owned abilities or attributes that should survive pawn respawn.
Category: GAS / ASC Placement
Priority: P2

Q: When does ASC-on-Pawn often fit?
A: Body-specific or NPC abilities whose state should die with the pawn/avatar.
Category: GAS / ASC Placement
Priority: P2

Q: What is an ability grant ledger?
A: A record of why each ability/effect was granted so cleanup and debugging are deterministic.
Category: GAS / Grants
Priority: P2

Q: What bug does a grant ledger prevent?
A: Orphaned or duplicate abilities after equipment removal, respawn, save/load or loadout changes.
Category: GAS / Grants
Priority: P2

Q: What does ability commit usually represent?
A: The point where cost/cooldown/activation commitments become part of the ability transaction.
Category: GAS / Activation
Priority: P2

Q: Why can predicted ability cost double-apply?
A: Client provisional cost and server authoritative cost can both affect UI/state without reconciliation.
Category: GAS / Prediction
Priority: P2

Q: What should happen when a predicted ability is server-rejected?
A: Client presentation and provisional cost/cooldown/UI state must reconcile to server truth.
Category: GAS / Prediction
Priority: P2

Q: What is a prediction key used for?
A: Associating predicted client-side actions with server confirmation or rejection.
Category: GAS / Prediction
Priority: P2

Q: Why are GameplayCues not authority?
A: They present effects; they should not own durable damage, inventory or gameplay truth.
Category: GAS / Gameplay Cues
Priority: P2

Q: What common cue bug appears under prediction?
A: Duplicate VFX/audio when prediction and server correction both fire presentation.
Category: GAS / Gameplay Cues
Priority: P2

Q: What should a safe GameplayCue policy define?
A: Audience, authority, dedupe keys, start/end semantics and correction behaviour.
Category: GAS / Gameplay Cues
Priority: P2

Q: What is an AttributeSet invariant?
A: A rule such as current health must stay between zero and max health after modifiers.
Category: GAS / Attributes
Priority: P2

Q: Why is max-health buff ordering tricky?
A: Changing the maximum can require deliberate clamping or proportional adjustment of current value.
Category: GAS / Attributes
Priority: P2

Q: What does a GameplayEffect spec carry?
A: Context, source/target data, captured values, tags, magnitudes and configuration for application.
Category: GAS / Gameplay Effects
Priority: P2

Q: Why are snapshot captures important?
A: They decide whether a calculation uses values from effect creation time or later current values.
Category: GAS / Gameplay Effects
Priority: P3

Q: What can make GameplayEffect calculations hard to debug?
A: Captures, tags, stacking, context, set-by-caller values and timing interactions.
Category: GAS / Gameplay Effects
Priority: P3

Q: What should an AbilityTask cleanup test cover?
A: Success, cancel, owner death, ability end, prediction rejection, travel and repeated activation.
Category: GAS / Ability Tasks
Priority: P2

Q: Why can leaked AbilityTasks create ghost behaviour?
A: Old delegates/timers/listeners can fire after the ability should be inactive.
Category: GAS / Ability Tasks
Priority: P2

Q: What is target data in GAS used for?
A: Carrying selected targets, hit results or targeting intent from client/prediction to validation.
Category: GAS / Target Data
Priority: P2

Q: Why must target data be server-validated?
A: Clients can send impossible or stale targets unless authority checks range/timing/visibility/state.
Category: GAS / Target Data
Priority: P2

Q: When should a custom ability component beat GAS?
A: When requirements are small and GAS complexity exceeds the benefit.
Category: GAS / Adoption
Priority: P2

Q: What makes GAS adoption worthwhile?
A: Networked abilities, attributes, effects, tags, prediction, designer authoring and reusable systems.
Category: GAS / Adoption
Priority: P2

Q: What is a GameplayTag counted-state bug?
A: Failing to add/remove counts symmetrically, leaving blocked/stunned states stuck or prematurely cleared.
Category: GAS / Tags
Priority: P2

Q: What should a cooldown UI observe?
A: Authoritative or reconciled cooldown state, not only local visual timers.
Category: GAS / UI
Priority: P2

Q: What is a good GAS latency test?
A: Run predicted activation, rejection, cancellation and cue playback under packet lag/loss.
Category: GAS / Testing
Priority: P2

Q: What should a GAS debug log include?
A: ASC owner/avatar, ability, prediction key, commit, cost, cooldown, target data, effects and cues.
Category: GAS / Debugging
Priority: P2

Q: What is the main role of Behaviour Tree observer aborts?
A: Reactively abort lower-priority or current branches when observed Blackboard/decorator conditions change.
Category: AI / Behaviour Tree
Priority: P1

Q: Why do Behaviour Trees sometimes thrash?
A: Conditions or Blackboard values change near thresholds without hysteresis/cooldowns or stable ownership.
Category: AI / Behaviour Tree
Priority: P2

Q: What does StateTree combine?
A: Hierarchical states, selection, transitions, tasks, evaluators/data and state-machine-like flow.
Category: AI / StateTree
Priority: P2

Q: When can StateTree fit better than Behaviour Tree?
A: Activity/state workflows with explicit transitions, Smart Object use or data-flow style orchestration.
Category: AI / StateTree
Priority: P2

Q: What is the key Smart Object lifecycle?
A: Query, claim, use, complete or abort, then release.
Category: AI / Smart Objects
Priority: P2

Q: What does Smart Object claiming prevent?
A: Multiple agents using the same exclusive slot or affordance simultaneously.
Category: AI / Smart Objects
Priority: P2

Q: Why should Smart Objects not hide all interaction authority?
A: Multiplayer/server validation and durable gameplay state still need explicit ownership.
Category: AI / Smart Objects
Priority: P2

Q: What should custom AI senses produce?
A: Evidence/stimuli with location, strength, tag and age, not final behaviour decisions.
Category: AI / Perception
Priority: P2

Q: Why avoid gameplay policy inside `UAISense`?
A: It couples sensing cost/timing with decision logic, difficulty and side effects.
Category: AI / Custom Senses
Priority: P2

Q: What causes a custom sense to "remember forever"?
A: Stimulus aging/expiration or Blackboard memory projection is not cleared correctly.
Category: AI / Perception
Priority: P2

Q: What separates perception memory from Blackboard truth?
A: Perception stores sensed evidence; Blackboard stores decision inputs chosen by AI logic.
Category: AI / Perception
Priority: P2

Q: What does Visual Logger help debug?
A: Spatial and timeline AI/path/perception decisions that are hard to see from logs alone.
Category: AI / Debugging
Priority: P2

Q: What makes EQS cost grow?
A: Agents times candidates times tests times frequency and test complexity.
Category: AI / EQS
Priority: P2

Q: How do you budget EQS?
A: Reduce frequency/candidates, run cheap filters early, cache where valid and profile real agent counts.
Category: AI / EQS
Priority: P2

Q: What is a partial path bug?
A: AI accepts or fails to handle a route that reaches only part of the intended destination.
Category: AI / Navigation
Priority: P2

Q: What is the difference between pathfinding and avoidance?
A: Pathfinding plans a route; avoidance adjusts local movement to avoid collisions/crowding.
Category: AI / Navigation
Priority: P1

Q: Why should RVO and Detour Crowd not be blindly combined?
A: They are different avoidance systems and can conflict or duplicate movement decisions.
Category: AI / Avoidance
Priority: P2

Q: What does MassEntity improve when it fits?
A: Large-population data processing through fragments/archetypes/chunks instead of many ticking actors.
Category: MassEntity / ECS
Priority: P2

Q: What is a Mass fragment?
A: Data attached to entities and processed in batches by queries/processors.
Category: MassEntity / Fragments
Priority: P2

Q: What is a Mass tag?
A: Marker data with no payload used for classification/query/state.
Category: MassEntity / Tags
Priority: P2

Q: Why are frequent structural changes expensive in ECS?
A: They can move entities between archetypes/chunks and create command-buffer/sync work.
Category: MassEntity / Performance
Priority: P3

Q: What is actor promotion in Mass hybrid architecture?
A: Creating or activating a richer Actor representation for a nearby/interactive entity.
Category: MassEntity / Hybrid
Priority: P3

Q: What is the main bug in Mass actor promotion/demotion?
A: Dual writers causing entity and actor state to diverge during handoff.
Category: MassEntity / Hybrid
Priority: P3

Q: What does ZoneGraph provide?
A: Structured lanes/zones for crowd or traffic-like movement, not a universal NavMesh replacement.
Category: MassAI / ZoneGraph
Priority: P3

Q: What should a Mass crowd proof compare?
A: Actor baseline versus Mass simulation/representation under equivalent movement and visual requirements.
Category: MassEntity / Proof
Priority: P2

Q: What must Mass replication answers avoid?
A: Claiming ordinary Actor replication patterns apply automatically to Mass entities.
Category: MassEntity / Networking
Priority: P3

Q: What makes a PCG graph deterministic enough for production?
A: Controlled seeds, inputs, graph version, parameters, output ownership and cleanup/regeneration policy.
Category: PCG / Determinism
Priority: P2

Q: What is PCG density?
A: A value used to influence point/output selection, not a guarantee of final object count by itself.
Category: PCG / Concepts
Priority: P2

Q: What should a PCG debug checkpoint validate?
A: Key assumptions such as point count, attribute existence, domain, seed or output ownership.
Category: PCG / Debugging
Priority: P2

Q: Why can runtime PCG hitch?
A: Graph work and generated output creation can happen during gameplay on the target device.
Category: PCG / Runtime
Priority: P2

Q: What is the safest assumption about SceneCapture2D cost?
A: It is another rendered view until profiling proves a cheaper path.
Category: Rendering / Scene Capture
Priority: P2

Q: Why are cube captures especially costly?
A: They can render six directions/views per update.
Category: Rendering / Scene Capture
Priority: P2

Q: What render-target dimensions affect cost?
A: Resolution, format, mips, update cadence, clears/resolves/readback and consumers.
Category: Rendering / Render Targets
Priority: P2

Q: Why can render target readback be dangerous?
A: It can introduce CPU/GPU synchronisation and stalls.
Category: Rendering / Render Targets
Priority: P2

Q: What is shader permutation explosion?
A: A large number of shader variants from static switches/features/platform/material combinations.
Category: Rendering / Shaders
Priority: P2

Q: Why is one master material not always better?
A: Too many switches/features can increase permutation count, compile time and PSO complexity.
Category: Rendering / Materials
Priority: P2

Q: What is a PSO hitch?
A: A runtime stall when a required pipeline state is created or encountered without adequate coverage/warmup.
Category: Rendering / PSO
Priority: P2

Q: What should PSO proof use?
A: Packaged cold-cache scenarios, PSO logging/collection where supported and first-use hitch verification.
Category: Rendering / PSO
Priority: P2

Q: What does RDG make explicit?
A: Render passes, resources, dependencies and lifetimes in the render graph.
Category: Rendering / RDG
Priority: P3

Q: What is a common RDG bug?
A: Using a resource outside its declared lifetime or without the needed dependency.
Category: Rendering / RDG
Priority: P3

Q: What can material slots increase?
A: Mesh sections, pass participation, draw/state changes and material/shader cost.
Category: Rendering / Materials
Priority: P2

Q: Why is triangle count not the only render cost?
A: Materials, overdraw, shadows, passes, draw setup, bandwidth and screen coverage matter.
Category: Rendering / Performance
Priority: P1

Q: What does Nanite not solve by itself?
A: Material, pixel, translucency, WPO, shadow, platform and gameplay representation costs.
Category: Rendering / Nanite
Priority: P2

Q: What does instancing help most?
A: Repeated compatible mesh data with shared rendering paths and manageable per-instance updates.
Category: Rendering / Instancing
Priority: P2

Q: What can per-frame `MarkRenderStateDirty` cause?
A: Extra render-state updates and loss of cached rendering work.
Category: Rendering / Mesh Draw
Priority: P3

Q: What is overdraw?
A: Multiple fragments/pixels shading the same screen area, common with translucent/masked effects.
Category: Rendering / GPU
Priority: P1

Q: What is a good GPU-bound first question?
A: Which pass/resource/shader/screen area dominates in a target GPU capture?
Category: Rendering / GPU Profiling
Priority: P1

Q: What can VSM/Lumen/Nanite answers overclaim?
A: That enabling a UE5 feature automatically improves performance or quality on every platform.
Category: Rendering / UE5 Features
Priority: P2

Q: Why should visual quality be part of performance proof?
A: A faster frame is not acceptable if gameplay readability or art requirements are broken.
Category: Rendering / Quality
Priority: P1

Q: What can Niagara cost on CPU?
A: Spawning, system update, component management, parameter work and renderer submission.
Category: VFX / Niagara
Priority: P2

Q: What can Niagara cost on GPU?
A: Particle simulation, material overdraw, render passes, sorting and bandwidth.
Category: VFX / Niagara
Priority: P2

Q: Why can pooled VFX be stale?
A: Parameters, owners, attachments, delegates or completion state may not reset.
Category: VFX / Pooling
Priority: P3

Q: What should an audio concurrency rule define?
A: Voice limits, priority, virtualization/stealing behaviour and user-perceived importance.
Category: Audio / Concurrency
Priority: P2

Q: Why can multiplayer audio double-play?
A: Local prediction and replicated/server events may both trigger the same sound.
Category: Audio / Multiplayer
Priority: P2

Q: What is a persistent audio loop best driven by?
A: Durable replicated/local state with start/stop dedupe, not repeated transient fire events.
Category: Audio / Runtime
Priority: P2

Q: What makes MetaSounds different from simple playback?
A: Procedural graph execution and parameter behaviour can affect runtime cost.
Category: Audio / MetaSounds
Priority: P3

Q: What is the usual split between animation state machine and montage?
A: State machine for continuous pose selection; montage for discrete authored actions/layers.
Category: Animation / Runtime
Priority: P1

Q: Why are animation notifies weak gameplay authority alone?
A: Playback can vary by client, montage state, LOD, prediction or cancellation.
Category: Animation / Gameplay
Priority: P1

Q: What should a server-validated notify window check?
A: Montage/ability state, timing, authority, target validity and duplicate-hit prevention.
Category: Animation / Networking
Priority: P1

Q: Why can root motion foot-slide under networking?
A: Animation root motion, server movement authority, corrections and smoothing may not align.
Category: Animation / Root Motion
Priority: P2

Q: What should root-motion networking tests include?
A: Dedicated server, latency/loss, simulated proxy view, correction and montage/cancel cases.
Category: Animation / Networking
Priority: P2

Q: What is the collision triage first split?
A: Query versus simulation versus movement versus event notification.
Category: Physics / Collision
Priority: P0

Q: What does Collision Enabled separate?
A: Query participation, physics simulation participation, both or neither.
Category: Physics / Collision
Priority: P0

Q: What is the difference between Block and Overlap?
A: Block stops/impedes movement or query; overlap reports intersection without blocking.
Category: Physics / Collision
Priority: P0

Q: What can `StartPenetrating` indicate?
A: A sweep/query started already overlapping or inside blocking geometry.
Category: Physics / Queries
Priority: P1

Q: Why can multi sweeps confuse candidates?
A: They may return touches/overlaps plus a blocking hit with specific ordering/semantics.
Category: Physics / Queries
Priority: P1

Q: What does CCD help with?
A: Fast-moving body collision detection that might otherwise tunnel between frames.
Category: Physics / Simulation
Priority: P2

Q: What does substepping trade?
A: Better simulation stability for extra CPU cost and added complexity.
Category: Physics / Simulation
Priority: P2

Q: What commonly destabilises constraints?
A: Bad frames, extreme mass ratios, unrealistic drives, collisions, timestep or solver settings.
Category: Physics / Constraints
Priority: P2

Q: What should ragdoll recovery validate?
A: Physics Asset bodies, constraints, collision, authority, sleep state and animation blend handoff.
Category: Physics / Ragdoll
Priority: P2

Q: Why should UI not own gameplay truth?
A: Widgets can be destroyed/recreated and are presentation scoped; truth belongs in gameplay/server/model state.
Category: UI / Architecture
Priority: P1

Q: What should UI model projection define?
A: Data audience, lifetime, owner, update path and stale-reference handling.
Category: UI / Architecture
Priority: P1

Q: Why can UMG bindings hurt performance?
A: They can evaluate repeatedly and perform hidden work every frame.
Category: UI / Performance
Priority: P1

Q: Why are ListView entries dangerous with async icons?
A: Recycled widgets may receive stale async completion for a previous item.
Category: UI / ListView
Priority: P2

Q: What should virtualised list entries reset?
A: Text, images, selection, delegates, async requests, model refs and transient state.
Category: UI / ListView
Priority: P2

Q: What does CommonUI not remove the need for?
A: Clear input/focus ownership, activation policy and local-player scoping.
Category: UI / CommonUI
Priority: P2

Q: Why is `GetPlayerController(0)` risky in UI?
A: It breaks split-screen/local-player scoping assumptions.
Category: UI / Local Player
Priority: P2

Q: What should low scalability preserve?
A: Gameplay readability, critical UI, input affordances and accessibility requirements.
Category: UI / Scalability
Priority: P2

Q: What should a save schema version support?
A: Migration, missing/renamed definitions, stable IDs and backwards compatibility decisions.
Category: Gameplay / Saves
Priority: P1

Q: Why should saves avoid transient actor pointers?
A: Streamed/destroyed/renamed actors may not exist when restoring; stable IDs are safer.
Category: Gameplay / Saves
Priority: P1

Q: What is the split between item definition and item instance?
A: Definition is shared content; instance holds player-owned mutable state.
Category: Gameplay / Inventory
Priority: P1

Q: Why is stackable inventory not just a count?
A: Stacks need definition, owner, limits, split/merge policy, save/replication and UI identity.
Category: Gameplay / Inventory
Priority: P2

Q: What is a transactional gameplay operation?
A: A multi-step change with validation, authoritative commit, idempotency and rollback/compensation policy.
Category: Gameplay / Transactions
Priority: P2

Q: What prevents duplicate rewards?
A: Authoritative operation/completion IDs and idempotent reward application.
Category: Gameplay / Idempotency
Priority: P1

Q: What is a command in gameplay messaging?
A: A request for a system to perform an action.
Category: Gameplay / Messaging
Priority: P1

Q: What is an event in gameplay messaging?
A: A report that something happened.
Category: Gameplay / Messaging
Priority: P1

Q: Why can event-driven quests break on load?
A: Past events are gone unless durable quest state is restored.
Category: Gameplay / Quests
Priority: P1

Q: What should quest objective state use?
A: Stable IDs, durable progress, authority and reconstruction for UI/world presentation.
Category: Gameplay / Quests
Priority: P2

Q: Why are gameplay tags not a replacement for structured data?
A: They label state/meaning but can become ambiguous without taxonomy and validation.
Category: Gameplay / Tags
Priority: P1

Q: What makes tag hierarchies maintainable?
A: Naming rules, ownership, documentation, validation and limited overlapping semantics.
Category: Gameplay / Tags
Priority: P1

Q: When is direct call clearer than a delegate?
A: When there is a known required collaborator and one-to-one dependency.
Category: Architecture / Communication
Priority: P1

Q: When is a delegate appropriate?
A: For observation, fan-out or optional listeners with explicit lifetime/unbinding.
Category: Architecture / Communication
Priority: P1

Q: When is an interface appropriate?
A: For polymorphic capability requests without hard dependency on concrete class.
Category: Architecture / Communication
Priority: P1

Q: What is the risk of service locator overuse?
A: Hidden dependencies, scope, lifetime, authority and testing problems.
Category: Architecture / Dependencies
Priority: P1

Q: What should subsystem host selection match?
A: Engine/editor/game-instance/world/local-player lifetime and scope.
Category: Architecture / Subsystems
Priority: P1

Q: Why can GameInstanceSubsystem be wrong for local player state?
A: It is too global for split-screen/player-specific input/UI state.
Category: Architecture / Subsystems
Priority: P1

Q: When should you tick?
A: Only when continuous per-frame work is truly needed and cheaper/batched alternatives do not fit.
Category: Gameplay Framework / Tick
Priority: P0

Q: Why is "never tick" also a weak rule?
A: Some high-frequency changes are clearer or cheaper as controlled Tick/batched updates.
Category: Gameplay Framework / Tick
Priority: P1

Q: What can many actors cost besides Tick?
A: Components, rendering primitives, physics, replication, GC, construction and registration.
Category: Performance / Actors
Priority: P1

Q: What should you inspect in an Actor-count problem?
A: Ticks, component counts, primitives/materials, collision, replication, UObject count and spawn churn.
Category: Performance / Actors
Priority: P1

Q: What is deferred spawning useful for?
A: Supplying initial data before actor construction/activation-dependent work completes.
Category: Gameplay Framework / Spawning
Priority: P1

Q: What is the CDO?
A: The class default object that holds default reflected/object state for a UClass.
Category: UObject / CDO
Priority: P0

Q: Why can mutating defaults at runtime be dangerous?
A: You may change class/default-subobject state rather than per-instance state.
Category: UObject / CDO
Priority: P1

Q: What does `UPROPERTY` not automatically mean?
A: Editable in editor, Blueprint-visible, replicated, saved or always strong in every context.
Category: UObject / Reflection
Priority: P0

Q: Why are raw persistent UObject members risky?
A: GC/reflection may not see the reference and lifetime can become accidental.
Category: UObject / Pointers
Priority: P0

Q: When is `TWeakObjectPtr` useful?
A: Non-owning observation of a UObject that may be destroyed.
Category: UObject / Pointers
Priority: P1

Q: When is `TSoftObjectPtr` useful?
A: Referencing an asset/object by path for deferred loading without keeping it resident.
Category: UObject / Pointers
Priority: P1

Q: When is `TStrongObjectPtr` useful?
A: Explicit strong native ownership where a reflected UPROPERTY edge is unavailable.
Category: UObject / Pointers
Priority: P2

Q: Why is `TSharedPtr` usually not UObject ownership?
A: UObjects are managed by Unreal GC/reflection, not standard/shared reference counting.
Category: UObject / Pointers
Priority: P0

Q: What is a soft reference not?
A: A guarantee that the asset is cooked, loaded or valid at runtime.
Category: Assets / Soft References
Priority: P1

Q: What should async asset callbacks validate?
A: Request identity, owner/widget validity, expected item and stale completion.
Category: Assets / Async
Priority: P1

Q: What does Asset Registry provide?
A: Metadata and discovery of assets without necessarily loading full objects.
Category: Assets / Registry
Priority: P1

Q: What does Asset Manager add?
A: Primary Asset identity, loading policy, bundles, cook/chunk rules and project-level asset ownership.
Category: Assets / Asset Manager
Priority: P1

Q: Why can redirectors matter for releases?
A: Stale references or unresolved moves can create clean-cook/package surprises.
Category: Assets / Redirectors
Priority: P2

Q: What can `ConstructorHelpers` create?
A: Hardcoded constructor-time asset dependencies and early load/cook coupling.
Category: UE C++ / Assets
Priority: P2

Q: When is `FText` appropriate?
A: User-facing localised/culture-aware display text.
Category: UE C++ / Strings
Priority: P1

Q: When is `FName` appropriate?
A: Stable identifiers/keys where name-table semantics and comparison are desired.
Category: UE C++ / Strings
Priority: P1

Q: When is `FString` appropriate?
A: Mutable string manipulation/storage rather than localised display identity.
Category: UE C++ / Strings
Priority: P1

Q: What should Blueprint pure functions avoid?
A: Side effects, expensive work, hidden loads, randomness and mutable state changes.
Category: Blueprint / API Design
Priority: P1

Q: Why can Blueprint bindings be expensive?
A: They may reevaluate frequently and hide logic from obvious execution flow.
Category: Blueprint / Performance
Priority: P1

Q: What is a safe Blueprint API surface?
A: Narrow, validated, intention-revealing functions with correct edit/Blueprint/authority/lifetime specifiers.
Category: Blueprint / API Design
Priority: P1

Q: What should editor batch tools support?
A: Dry-run, source-control awareness, validation, logs, undo/transactions where relevant and partial-failure recovery.
Category: Tools / Editor
Priority: P1

Q: When should you write a commandlet?
A: For repeatable headless validation, migration, export/import or build pipeline work.
Category: Tools / Commandlets
Priority: P2

Q: When should you use an Editor Utility Widget?
A: For interactive editor workflows needing guided selection, preview or designer control.
Category: Tools / Editor Utility
Priority: P2

Q: Why share core logic between commandlet and EUW?
A: It keeps CI/headless and interactive tools consistent.
Category: Tools / Architecture
Priority: P2

Q: What is a spatial hash good for?
A: Average-case nearby queries when distribution, cell size and update cost fit the workload.
Category: Algorithms / Spatial
Priority: P1

Q: What can break spatial hash performance?
A: Bad cell size, clustering, large query radii and high update churn.
Category: Algorithms / Spatial
Priority: P1

Q: Why use a slow oracle in algorithm tests?
A: To prove fast implementation correctness on random and adversarial cases.
Category: Algorithms / Testing
Priority: P1

Q: What is A* admissibility?
A: The heuristic never overestimates true remaining cost, preserving optimality under assumptions.
Category: Algorithms / Pathfinding
Priority: P1

Q: What else matters besides A* heuristic?
A: Graph quality, costs, dynamic obstacles, smoothing, partial paths, avoidance and budget.
Category: Algorithms / Pathfinding
Priority: P1

Q: Why do normals need special transform handling under non-uniform scale?
A: They represent surface orientation and generally require inverse-transpose-style handling.
Category: Maths / Transforms
Priority: P2

Q: Why do quaternions not solve rotation policy automatically?
A: Interpolation, constraints, shortest path, designer control and networking still need decisions.
Category: Maths / Rotation
Priority: P1

Q: Why avoid exact float equality in gameplay?
A: Floating-point operations are approximate and order/platform sensitive.
Category: Maths / Numerics
Priority: P1

Q: What does server authority do for float divergence?
A: It corrects clients to authoritative state instead of trusting divergent local simulation.
Category: Maths / Networking
Priority: P2

Q: Why is Lua integration plugin-dependent in Unreal?
A: Unreal has no single built-in Lua runtime semantics; bridges define lifetime, reflection and packaging behaviour.
Category: Scripting / Lua
Priority: P3

Q: What is the main Lua/UObject lifetime risk?
A: Script references may outlive destroyed UObjects or hide references from GC.
Category: Scripting / Lua
Priority: P3

Q: What is a common C# Unreal-adjacent use?
A: AutomationTool, Gauntlet/build tooling or plugin-specific editor/runtime ecosystems.
Category: Scripting / C#
Priority: P3

Q: What should runtime C# claims in Unreal include?
A: Plugin, platform, packaging, UObject bridge, GC/lifetime and performance caveats.
Category: Scripting / C#
Priority: P3

Q: What should managed/native bridge tests cover?
A: Object destruction, world teardown, reload, async callbacks and delegate unbinding.
Category: Scripting / Interop
Priority: P3

Q: Why is simulator performance not target-device proof?
A: Hardware, graphics stack, OS services and memory pressure differ from physical devices.
Category: Platform / Apple
Priority: P2

Q: What does Apple/console profiling evidence need?
A: Build identity, target device/package, scenario markers, authorised tools and source-safe reporting.
Category: Platform / Profiling
Priority: P3

Q: Why not publish console profiler details casually?
A: Platform-holder documentation, tools and certification details are confidential.
Category: Platform / Console
Priority: P3

Q: What does AGI complement in Unreal Android profiling?
A: Device/GPU/render-pass evidence that aligns with Unreal markers and CSV/traces.
Category: Platform / Android
Priority: P3

Q: Why not compare raw Android GPU counters across vendors blindly?
A: Counter names and meanings are vendor-specific and need documentation.
Category: Platform / Android
Priority: P3

Q: What does Android LMK evidence show?
A: OS/process low-memory kill context that should be correlated with Unreal memory checkpoints.
Category: Platform / Android Memory
Priority: P2

Q: How can ADPF hide a regression?
A: It may drop quality or workload to keep frame time acceptable.
Category: Platform / Android Performance
Priority: P3

Q: What belongs in a device-lab run manifest?
A: Build ID, device ID, scenario ID, command line, profile/settings, artifacts, timestamps and gate policy.
Category: Automation / Device Lab
Priority: P2

Q: What are device-lab gate states?
A: Pass, product fail, lab infrastructure fail, flaky/quarantined, reporting-only and blocked by missing evidence.
Category: Automation / Device Lab
Priority: P2

Q: Why is blind rerun a bad automation policy?
A: It hides deterministic product failures and erodes trust in gates.
Category: Automation / Triage
Priority: P1

Q: What should a smoke scenario prove?
A: Package launches, representative content loads, basic path runs and pass/fail marker exits.
Category: Automation / Smoke
Priority: P1

Q: What should a failure injection prove?
A: Diagnostics and recurrence guards catch an expected broken state clearly.
Category: Debugging / Verification
Priority: P1

Q: What makes a regression guard useful?
A: It fails for the original root cause with the smallest reliable check and acceptable false-positive rate.
Category: Debugging / Regression
Priority: P1

Q: What should structured logs include?
A: Scenario, build ID, object/operation IDs, net mode/owner, state transition and error code.
Category: Debugging / Observability
Priority: P1

Q: When should high-frequency data use CSV instead of logs?
A: When repeated samples/counters would spam logs and obscure state transitions.
Category: Debugging / Observability
Priority: P1

Q: What is an actionable handoff report?
A: Repro, artifact identity, evidence, first cause, owner, fix/guard and residual risk.
Category: Team Workflow / Debugging
Priority: P1

Q: What is a good "hard bug" interview structure?
A: Symptom, scope, hypotheses, evidence, root cause, fix, guard and lesson.
Category: Interview / Debugging
Priority: P0

Q: What is a good optimisation story structure?
A: Target scenario, bottleneck evidence, change, before/after result, trade-off and recurrence guard.
Category: Interview / Performance
Priority: P0

Q: What makes a system design answer concrete?
A: State ownership, data flow, lifecycle, authority, failure modes, profiling/debugging and tests.
Category: Interview / System Design
Priority: P0

Q: What should you do when unsure about an Unreal API name?
A: State the stable concept and explain target branch source/API verification instead of bluffing.
Category: Interview / Version Sensitivity
Priority: P0

Q: What does "source-sensitive" mean in this curriculum?
A: The concept is useful but exact signatures/commands/behaviour must be verified against the target branch.
Category: Interview / Source Discipline
Priority: P1

Q: What should a rejection memo include?
A: Evidence unsupported now, fallback design, residual risk and trigger for future verification.
Category: Production Readiness / Scope
Priority: P1

Q: What is the best first answer to "it depends"?
A: Name exactly what it depends on and how you would verify each variable.
Category: Interview / Communication
Priority: P0

Q: What is the minimum portfolio proof packet?
A: Problem, scope, source/version notes, implementation, failure case, profiler/debug evidence and decision memo.
Category: Interview / Portfolio
Priority: P1

Q: Why should a five-minute deep dive include a failure case?
A: It proves debugging judgement and production learning, not just feature description.
Category: Interview / Communication
Priority: P0

Q: What should an "out of scope" section include?
A: Exclusions, rationale, risk, and triggers that would make the scope change.
Category: System Design / Scope
Priority: P1

Q: What is a production-readiness checklist trying to prevent?
A: Demo-only success that fails under packaging, scale, platforms, networking, saves or failure states.
Category: Production Readiness / General
Priority: P0

Q: What is a good answer when a feature is branch-unsupported?
A: Document evidence, fallback and future verification instead of silently omitting or overclaiming.
Category: Production Readiness / Versioning
Priority: P1

## Final Target Reinforcement Cards

Q: What should a packaged proof packet never rely on alone?
A: A developer's local editor state, warm cache or manual memory of commands.
Category: Packaged Proof / Evidence
Priority: P1

Q: What makes a package reproducible?
A: Exact source revisions, commands, target/config/platform, artifacts, symbols and validation results.
Category: Build / Reproducibility
Priority: P1

Q: Why should a build artifact path include build ID?
A: To prevent downstream jobs from accidentally consuming stale or adjacent outputs.
Category: Build / Artifacts
Priority: P1

Q: What is a good staged-file check?
A: Verify every expected runtime dependency exists in the packaged/staged layout from a clean build.
Category: Build / Staging
Priority: P2

Q: What should a package smoke failure report include first?
A: First causal log line, phase, package identity and scenario marker.
Category: Build / Smoke
Priority: P1

Q: What does a clean-agent package prove?
A: The project does not rely on undeclared local files, caches or developer-machine environment state.
Category: Build / CI
Priority: P1

Q: Why keep cook and stage manifests?
A: They let you trace whether expected content and runtime files entered the package.
Category: Build / Artifacts
Priority: P2

Q: What should a package-size regression start with?
A: Manifest and dependency diff by build ID.
Category: Assets / Package Size
Priority: P2

Q: What can shader library data affect?
A: Package size, startup, first-use hitches and platform-specific rendering behaviour.
Category: Rendering / Shaders
Priority: P2

Q: What is a good PSO regression guard?
A: A packaged scenario that exercises representative first-use states and records misses/hitches where supported.
Category: Rendering / PSO
Priority: P2

Q: What does a target branch audit prevent?
A: Copying commands or APIs from a different UE version before proving availability.
Category: Versioning / Branch Proof
Priority: P1

Q: What is a branch-sensitive command?
A: A command whose name, flags, output or support can change across engine versions or studio wrappers.
Category: Versioning / Branch Proof
Priority: P1

Q: What is a branch-sensitive API example?
A: NetworkPrediction model hooks, custom AI sense signatures, Iris fragments or builder commandlet classes.
Category: Versioning / Branch Proof
Priority: P1

Q: What should schematic code clearly mark?
A: Which names/signatures are conceptual and require target-branch verification.
Category: Writing / Source Discipline
Priority: P1

Q: Why is "works in PIE" weak multiplayer evidence?
A: PIE can hide process, package, ownership, timing and dedicated-server differences.
Category: Networking / Testing
Priority: P1

Q: What should a dedicated-server network test add?
A: Multi-process or packaged server/client execution, latency/loss settings and authority/ownership logs.
Category: Networking / Testing
Priority: P1

Q: What is a good packet-loss test for prediction?
A: Fixed movement/action scenario under configured lag/loss with correction, replay and side-effect logs.
Category: Networking / Prediction
Priority: P2

Q: Why can client-only movement changes create corrections?
A: The server cannot reproduce the client's movement, so it sends authoritative correction.
Category: Networking / Movement
Priority: P1

Q: What should a custom movement flag encode?
A: Compact deterministic input/state needed by server movement replay.
Category: Networking / Movement
Priority: P2

Q: What is a bad custom movement flag use?
A: Encoding arbitrary gameplay authority or one-off side effects in movement compression.
Category: Networking / Movement
Priority: P2

Q: What should replicated collection late-join proof test?
A: New client receives current collection state after previous adds/changes/removes.
Category: Networking / Fast Array
Priority: P1

Q: What should replicated subobject removal proof test?
A: Client cleanup, stale-reference rejection and GC-safe registered-list removal.
Category: Networking / Subobjects
Priority: P2

Q: What is a good Iris adoption status result?
A: Enabled/supported with branch proof, or explicitly not enabled with fallback replication path.
Category: Networking / Iris
Priority: P3

Q: Why should NetworkPrediction side effects use a ledger or cue dedupe?
A: Rollback/replay can otherwise repeat gameplay or presentation side effects.
Category: Networking / NetworkPrediction
Priority: P3

Q: What should a rollback invalid-command test prove?
A: Server rejection reconciles state without double-applying costs, cues or rewards.
Category: Networking / Rollback
Priority: P3

Q: What should a GameplayCue not do?
A: Own authoritative gameplay state such as damage, inventory or quest completion.
Category: GAS / Gameplay Cues
Priority: P2

Q: Why is ASC owner/avatar split important?
A: Owner defines long-lived controlling entity; avatar defines current physical actor body.
Category: GAS / ASC
Priority: P2

Q: What should ability cancellation clean up?
A: Tasks, delegates, timers, cues, target actors and provisional UI/predicted state.
Category: GAS / Cancellation
Priority: P2

Q: What should a server ability validation check?
A: Tags, cost, cooldown, target data, range, authority state and current activation rules.
Category: GAS / Validation
Priority: P2

Q: What is Smart Object release failure?
A: A claimed slot remains unavailable because use/abort cleanup did not release it.
Category: AI / Smart Objects
Priority: P2

Q: What should Smart Object race tests include?
A: Two or more agents attempting the same exclusive slot under timing variation.
Category: AI / Smart Objects
Priority: P2

Q: What is a Mass processor ordering issue?
A: Processors read stale or not-yet-produced fragment data because dependencies/phases are wrong.
Category: MassEntity / Processors
Priority: P3

Q: What should Mass debug views help inspect?
A: Entity archetypes, fragments/tags, processors, representation and LOD/promotion state.
Category: MassEntity / Debugging
Priority: P3

Q: What should PCG package proof include?
A: Generated output policy, cooked asset references and packaged runtime behaviour if runtime generation is enabled.
Category: PCG / Packaging
Priority: P2

Q: What is a PCG stale-output bug?
A: Regeneration leaves duplicate or obsolete generated actors/components/assets.
Category: PCG / Cleanup
Priority: P2

Q: What should an HLOD visual acceptance compare?
A: Far, transition and near ranges for missing geometry, material artefacts and popping.
Category: World Partition / HLOD
Priority: P2

Q: What should an HLOD performance acceptance compare?
A: Primitive/draw/material/shadow cost, memory/proxy cost and transition hitches.
Category: World Partition / HLOD
Priority: P2

Q: What should a Data Layer save/load test prove?
A: Durable state restores expected runtime layer presentation after reload.
Category: World Partition / Data Layers
Priority: P2

Q: Why can editor-only Data Layers fool testing?
A: They can appear correct in editor while not being runtime-loadable/controllable in package.
Category: World Partition / Data Layers
Priority: P2

Q: What should a packaged traversal failure report classify?
A: Streaming, Data Layer, HLOD, cook/package, collision/nav, gameplay readiness or performance.
Category: World Partition / Debugging
Priority: P2

Q: What should a platform profiler report avoid?
A: Overgeneralising one device/tool capture to all platforms or SKUs.
Category: Platform / Profiling
Priority: P2

Q: What should an Apple Metal capture be tied to?
A: Exact device/OS/Xcode/build/scenario and corresponding UE markers or captures.
Category: Platform / Apple
Priority: P3

Q: What should console profiling answers say about confidentiality?
A: Use authorised tools and docs, keep raw confidential artifacts out of public reports.
Category: Platform / Console
Priority: P3

Q: What is a thermal soak test for?
A: Proving sustained performance after the device reaches realistic thermal state.
Category: Platform / Mobile
Priority: P2

Q: What is a lifecycle memory test?
A: Exercise map load, background/foreground, memory pressure and return-to-menu checkpoints.
Category: Platform / Mobile
Priority: P2

Q: What should an automation rerun policy distinguish?
A: Deterministic product failures, flaky tests, flaky devices and infrastructure failures.
Category: Automation / Triage
Priority: P1

Q: What does "blocked by missing evidence" mean?
A: The gate cannot make a valid pass/fail decision because required artifacts or markers are absent.
Category: Automation / Triage
Priority: P1

Q: What should a debug overlay never assume?
A: That developer-only usage makes unbounded draw/text/allocation cost acceptable.
Category: Debugging / Tools
Priority: P2

Q: What is the safest capture strategy under time pressure?
A: Use a cheap classifier first, then a targeted causal capture for the suspected failing window.
Category: Profiling / Workflow
Priority: P1

Q: What is bottleneck migration?
A: An optimisation reduces one limit but exposes or worsens another thread/resource limit.
Category: Profiling / Method
Priority: P1

Q: What should before/after optimisation tables include?
A: Same scenario/settings, repeated metrics, visual/correctness checks and changed trade-offs.
Category: Profiling / Reporting
Priority: P1

Q: What makes a performance improvement not shippable?
A: It breaks gameplay readability, quality requirements, memory limits or another target scenario.
Category: Profiling / Reporting
Priority: P1

Q: Why is "more async" not a general fix?
A: Async work still has scheduling, ownership, merge, memory, thread-safety and latency costs.
Category: Performance / Async
Priority: P1

Q: What does task granularity affect?
A: Scheduling overhead, worker utilisation, cache locality and merge cost.
Category: Performance / Tasks
Priority: P2

Q: What should an async UObject task never do casually?
A: Mutate or dereference UObjects off-thread without a verified thread-safe contract.
Category: Performance / Async
Priority: P1

Q: What is false sharing?
A: Independent thread writes to data on the same cache line causing cache ownership contention.
Category: C++ / Performance
Priority: P2

Q: What is the safest default for multi-field shared invariants?
A: A mutex-protected critical section unless a proven atomic protocol exists.
Category: C++ / Concurrency
Priority: P1

Q: What does relaxed atomic ordering not provide?
A: Publication/visibility ordering for other non-atomic data.
Category: C++ / Atomics
Priority: P1

Q: What should condition-variable waits use?
A: A mutex-protected predicate loop, not notification as stored state.
Category: C++ / Concurrency
Priority: P1

Q: Why can lambda capture by reference be unsafe in async code?
A: The referenced object may be destroyed before the callback/task runs.
Category: C++ / Lifetime
Priority: P1

Q: What should `TFunctionRef` never do?
A: Escape or be stored beyond the synchronous call that received it.
Category: UE C++ / Callables
Priority: P2

Q: What should `TArray` selection consider?
A: Dense iteration, order, cache locality, insertion/removal pattern and invalidation.
Category: UE C++ / Containers
Priority: P0

Q: What should `TMap` selection consider?
A: Key lookup needs, hashing, memory overhead, deterministic order and mutation patterns.
Category: UE C++ / Containers
Priority: P1

Q: Why can Blueprint pure functions be called more than expected?
A: Graph evaluation can reevaluate pure nodes wherever their output is needed.
Category: Blueprint / Pure
Priority: P1

Q: What should an interview answer do when the exact API is unknown?
A: Avoid bluffing, explain stable design shape and name the target-source verification path.
Category: Interview / Accuracy
Priority: P0

Q: What makes an interview answer "specialist depth"?
A: Implementation trade-offs, edge cases, debugging workflow, proof artifacts and version caveats.
Category: Interview / Depth
Priority: P0

Q: What is a weak answer pattern for Unreal interviews?
A: Naming a system without ownership, lifecycle, failure modes or evidence.
Category: Interview / Answer Patterns
Priority: P0

Q: What is a strong answer pattern for Unreal interviews?
A: Define the system, explain why, describe implementation boundaries, debug failures and cite proof.
Category: Interview / Answer Patterns
Priority: P0

Q: What should every hands-on project extension produce?
A: Working behaviour or clear rejection, injected failure, evidence artifacts and acceptance criteria.
Category: Practice / Projects
Priority: P1

Q: What is the curriculum's bias for target-sensitive topics?
A: Teach the mental model, then require target-branch/source/capture proof before implementation claims.
Category: Study Strategy / Versioning
Priority: P1

## Systems C++, OS, Networking Security, and Anti-Cheat Cards

Q: What is the safest wording for references at assembly level?
A: References are language-level aliases that are often represented as addresses when storage or parameter passing requires it; they are not simply pointers in the C++ semantic model.
Category: Standard C++
Priority: P0

Q: What is the key semantic difference between pointer and reference?
A: A pointer is an object value that can be null and reseated; a reference is bound as an alias and does not imply ownership.
Category: Standard C++
Priority: P0

Q: What does `new T` do that `malloc` does not?
A: It obtains storage and constructs a `T`; `malloc` only provides raw storage.
Category: Standard C++
Priority: P0

Q: What does `delete` do that `free` does not?
A: It destroys the object before releasing matching storage; `free` releases raw storage without running C++ destructors.
Category: Standard C++
Priority: P0

Q: What is placement `new`?
A: Construction of an object in storage that has already been provided, with destruction and storage release handled separately.
Category: Standard C++
Priority: P1

Q: Why not `new UObject`?
A: UObjects need Unreal creation/lifetime systems such as `NewObject`, `CreateDefaultSubobject` or `SpawnActor`, plus reflection, Outer, flags and GC integration.
Category: UE C++
Priority: P0

Q: What is a vtable?
A: A common ABI mechanism for C++ virtual dispatch, not a layout mandated by the C++ standard.
Category: Standard C++
Priority: P1

Q: How does Unreal add to ordinary C++ OOP?
A: It adds generated reflection metadata, UObject lifetime/GC, serialisation/editor/Blueprint integration and reflection-based calls alongside native C++ virtual dispatch.
Category: UE C++
Priority: P0

Q: What are `brk`, `mmap` and `VirtualAlloc`?
A: OS-level virtual memory/address-range APIs below ordinary C++ allocators and object lifetimes.
Category: Systems Programming
Priority: P1

Q: What does reserve versus commit mean for virtual memory?
A: Reserve claims virtual address space; commit provides backing/charge so pages can be accessed, often with physical allocation on first touch.
Category: Systems Programming
Priority: P1

Q: What is a page fault?
A: A CPU/OS event when a virtual page access needs translation, backing, permission handling or faults due to invalid/protected access.
Category: Systems Programming
Priority: P1

Q: Why is a major page fault dangerous for frame time?
A: It often requires storage I/O to satisfy the page, which can hitch loading or gameplay.
Category: Systems Programming
Priority: P2

Q: What does `static` not automatically mean?
A: It does not always mean "global"; it can refer to storage duration, linkage, class membership or block-local persistence depending on context.
Category: Standard C++
Priority: P0

Q: What is `inline` mainly about in C++?
A: ODR/linkage rules allowing matching definitions across translation units, not a guarantee of machine-code inlining.
Category: Standard C++
Priority: P0

Q: Why are macros risky compared with inline functions?
A: Macros are token substitution before C++ type checking, scope, overload resolution and normal debugging boundaries.
Category: Standard C++
Priority: P1

Q: What is the difference between a process and a thread?
A: A process owns an isolated address space/resources; a thread is an OS-scheduled execution stream sharing its process memory.
Category: Systems Programming
Priority: P1

Q: What is a task in a game job system?
A: A scheduler-managed unit of work that runs on worker threads and should have explicit dependencies and safe captured data.
Category: Concurrency
Priority: P1

Q: Why can more threads make code slower?
A: Scheduling, contention, false sharing, memory bandwidth and serial merge/wait points can dominate useful parallel work.
Category: Concurrency
Priority: P1

Q: How does shared memory work at a high level?
A: The OS maps the same shared object/pages into multiple processes; the program defines layout and synchronisation.
Category: IPC
Priority: P2

Q: Why avoid raw pointers inside shared memory?
A: Each process may map the same pages at different virtual addresses, so offsets/indices are safer.
Category: IPC
Priority: P2

Q: What is deadlock?
A: A wait cycle where participants hold resources while waiting for resources held by each other.
Category: Concurrency
Priority: P1

Q: What should deadlock diagnosis capture?
A: Thread/task stacks, held locks, waited locks, owner IDs and the minimal wait cycle.
Category: Concurrency
Priority: P1

Q: What does TCP provide?
A: A reliable ordered byte stream with congestion control.
Category: Networking
Priority: P1

Q: What does UDP not provide?
A: Built-in delivery, ordering or duplicate protection.
Category: Networking
Priority: P1

Q: What is state synchronisation?
A: Sending selected current state or snapshots so clients converge, interpolate, predict and correct.
Category: Networking
Priority: P1

Q: What is frame/input synchronisation?
A: Exchanging inputs per tick/frame so deterministic simulations advance from the same inputs.
Category: Networking
Priority: P2

Q: What does rollback require?
A: Deterministic state capture, input logs, restore/resimulate support and side-effect deduplication.
Category: Networking
Priority: P2

Q: What does HTTPS not solve?
A: It does not make a malicious client honest, validate gameplay rules or prevent application-level replay by itself.
Category: Network Security
Priority: P1

Q: What is a replay attack?
A: Reuse of a previously valid message/action so it is accepted again outside its intended context.
Category: Network Security
Priority: P1

Q: What defends against replay in gameplay commands?
A: Identity binding, sequence or operation IDs, freshness windows, idempotency and server-side state validation.
Category: Network Security
Priority: P1

Q: What is the main defence against Cheat Engine-style health editing?
A: Make health server-authoritative and treat client health as prediction/presentation, not truth.
Category: Defensive Anti-Cheat
Priority: P1

Q: Why does encryption of a client variable not solve cheating?
A: A determined local attacker can inspect/tamper with the client; consequential state must be validated and owned server-side.
Category: Defensive Anti-Cheat
Priority: P1

Q: What is the first anti-ESP design question?
A: Does this client truly need this information now, at this precision and for this duration?
Category: Defensive Anti-Cheat
Priority: P2

Q: What is a realistic ESP defence goal?
A: Reduce unnecessary information exposure, validate actions server-side and collect reviewable evidence, not promise perfect hiding once rendering is required.
Category: Defensive Anti-Cheat
Priority: P2

Q: What should server-side aimbot defence validate first?
A: Hits, ammo, rate, cooldown, visibility, range, target state and weapon rules before applying damage.
Category: Defensive Anti-Cheat
Priority: P2

Q: Why are single-threshold aimbot detections risky?
A: False positives are costly and player hardware/accessibility/network conditions vary; use evidence over time and review.
Category: Defensive Anti-Cheat
Priority: P2

Q: What is a DLL in defensive anti-cheat discussion?
A: A dynamically loaded module; unexpected modules/load paths can become an integrity/tamper signal but do not replace server authority.
Category: Defensive Anti-Cheat
Priority: P2
