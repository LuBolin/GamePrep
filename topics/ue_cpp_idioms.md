# UE C++ Idioms

See also: [[cpp_for_unreal_interviews]], [[game_architecture_patterns]], [[ue_assets_loading_cooking]], [[ue_hands_on_projects]], [[ue_interview_question_bank]].

## Cluster 1 — Reflection, UObject Lifetime, and Pointer Selection

**Priority:** P0  
**Expected depth:** D4 for gameplay/generalist roles; D3 for adjacent roles  
**Version scope:** Stable UE4/UE5 foundations with UE5 pointer guidance. Incremental reachability is **Experimental in UE5.6**.  
**Prerequisites:** C++ object lifetime, pointers/references, RAII, and the distinction between owning and observing references.

### The useful mental model

Unreal adds an engine-visible type and property system because ordinary compiled C++ does not retain enough structured runtime information for editor exposure, Blueprint calls, serialisation, replication, reference discovery, and other engine services. `UCLASS`, `USTRUCT`, `UENUM`, `UFUNCTION`, and `UPROPERTY` opt declarations into parts of that system; `GENERATED_BODY()` connects generated support code. UHT parses Unreal-aware headers and generates code before the ordinary C++ compiler runs. It is code generation, not a replacement compiler and not runtime reflection magically inferred from arbitrary C++. [SRC-EPIC-001] [SRC-EPIC-002]

A `UObject` is therefore both a C++ object and an entry in Unreal's managed object universe. That universe provides identity, runtime type data, serialisation hooks, editor integration, replication support where applicable, and garbage-collected lifetime. Not every C++ type should become a UObject: plain values, hot data, short-lived helpers, and RAII resources are often better represented by ordinary C++ types or reflected `USTRUCT` values. A `USTRUCT` can expose fields to reflection and serialisation, but the struct value itself is not independently garbage-collected. [SRC-EPIC-003] [SRC-EPIC-008]

### What UHT and the macros do

Compilation has two relevant stages: UBT invokes UHT to parse supported Unreal declarations and emit generated code, then invokes the configured C++ compiler. This explains several familiar rules: the `.generated.h` include must be last among includes, reflected declarations need supported syntax, and errors mentioning generated code often originate in the declaration immediately before the macro rather than in the generated file. [SRC-EPIC-002] [SRC-EPIC-003]

`UPROPERTY` is not synonymous with “show this in the editor”. Its specifiers can participate in editor visibility, Blueprint access, serialisation, configuration, save-game tagging, replication declarations, instanced subobjects, and transient behaviour. For UObject references, a reflection-visible strong property also gives the collector a discoverable edge. These capabilities are separate: for example, `VisibleAnywhere` concerns editor editing, `BlueprintReadOnly` concerns Blueprint access, and `Transient` concerns persistence. State the specific capability you need instead of sprinkling macros as superstition. [SRC-EPIC-001] [SRC-EPIC-007]

### CDOs, constructors, and instances

Every `UCLASS` has a Class Default Object (CDO), which acts as the class's default template. Constructors run in contexts that create defaults and subobjects, not merely when a gameplay instance appears. Keep a UObject constructor focused on default values and `CreateDefaultSubobject`; do world-dependent work in an appropriate lifecycle phase such as `BeginPlay` for Actors. Runtime UObjects are created with `NewObject`, Actors through `SpawnActor`, and constructor-time default subobjects through `CreateDefaultSubobject`; do not allocate UObjects with ordinary `new` or destroy them with `delete`. [SRC-EPIC-003] [SRC-EPIC-004]

This is a frequent interview discriminator. “The constructor runs before BeginPlay” is true but shallow. A stronger answer mentions the CDO and explains why world lookup, player access, dynamic binding, or mutable gameplay state in a constructor can affect templates, run before a valid world exists, or behave surprisingly in the editor.

### Garbage collection: reachability, not C++ scope

Unreal's UObject collector uses mark-and-sweep reachability. Conceptually, it begins from roots and follows engine-visible strong references; unreachable UObjects become eligible for reclamation. A local raw pointer going out of C++ scope does not itself delete a UObject, and a raw pointer remaining in memory does not necessarily keep one alive. Conversely, a reflected strong reference can keep a target alive even when no gameplay system still needs it. [SRC-EPIC-005] [SRC-EPIC-006]

This is different from RAII. Ordinary C++ owners such as `std::unique_ptr` have deterministic destruction tied to their owner's lifetime. UObject reclamation is deferred to collector work, and destruction should not be treated as an exact-scope event. Use RAII for non-UObject resources and the UObject reference model for UObjects; do not blur the two into “all smart pointers own things”.

UE5.6 documents incremental reachability analysis as experimental. It spreads reachability work over frames and relies on write barriers supplied by `TObjectPtr` properties. That feature strengthens the reason to use `UPROPERTY() TObjectPtr<T>` for persistent reflected references, but it must not be described as universally enabled production behaviour in UE5.6. [SRC-EPIC-005]

### `Outer` is not a general ownership guarantee

`Outer` establishes object containment/identity: it participates in an object's path and package/context relationships. It is supplied to `NewObject`, and an outer chain is useful for organisation and lookup. Do not infer that merely passing an Outer creates the strong reference edge your lifetime design needs. Store a discoverable strong reference where the object must be retained. This distinction—identity/containment versus GC reachability—is safer than the common but misleading phrase “the Outer owns the UObject”. [SRC-EPIC-004] [SRC-EPIC-006]

### Pointer choice by intent

| Intent | Typical type | Keeps target alive? | Loads on demand? | Key caveat |
|---|---|---:|---:|---|
| Persistent UObject field that must remain alive | `UPROPERTY() TObjectPtr<T>` | Yes | No | `TObjectPtr` needs reflection visibility for the normal member-field GC edge. |
| Short-lived local/parameter on the game thread | `T*` | No | No | Valid only while another mechanism guarantees lifetime. |
| Cache/observer that must not retain target | `TWeakObjectPtr<T>` | No | No | Resolve and validate at use time. |
| Asset/class reference without a hard load dependency | `TSoftObjectPtr<T>` / `TSoftClassPtr<T>` | No | Yes, explicitly | The path is not a loaded object; handle pending/null states and async completion. |
| Strong UObject reference from a non-UObject owner | `TStrongObjectPtr<T>` or explicit `FGCObject` integration | Yes | No | Use deliberately; hidden retention and churn make it unsuitable as a default. |
| Ordinary non-UObject exclusive ownership | `TUniquePtr<T>` / `std::unique_ptr<T>` | Deterministic C++ ownership | No | Not a UObject GC mechanism. |
| Ordinary non-UObject shared lifetime | `TSharedPtr<T>` / `std::shared_ptr<T>` | Reference-counted C++ lifetime | No | Do not use as the normal UObject ownership model. |

Epic's object-pointer guide distinguishes raw, strong, weak, and soft UObject references and marks `TLazyObjectPtr` deprecated in favour of soft references. The crucial question is not “which pointer is safest?” but “should this edge retain, observe, locate/load, or deterministically own?” [SRC-EPIC-009]

### Creation and lifetime example

```cpp
UCLASS()
class UTargetingSession : public UObject
{
    GENERATED_BODY()

public:
    UPROPERTY()
    TObjectPtr<AActor> CurrentTarget;

    TWeakObjectPtr<AActor> LastSeenTarget;

    UPROPERTY(EditDefaultsOnly)
    TSoftObjectPtr<UCurveFloat> AimAssistCurve;
};

// In a UObject-derived owner:
UPROPERTY()
TObjectPtr<UTargetingSession> Session;

Session = NewObject<UTargetingSession>(this);
```

`Session` is retained by a reflected strong edge. `CurrentTarget` is also retained; if retaining the target is undesirable, make it weak. `LastSeenTarget` never promises survival and must be checked when resolved. `AimAssistCurve` avoids forcing the asset to load merely because the class defaults refer to it. The `Outer` gives the session a containing context, while the `Session` property supplies the explicit retaining edge.

### Common bugs and misconceptions

1. **“`UPROPERTY` is only for the editor.”** It is a gateway to several independent engine systems; editor exposure is only one.
2. **“`TObjectPtr` alone guarantees lifetime.”** For a normal UObject member, the persistent strong pattern is `UPROPERTY() TObjectPtr<T>`.
3. **“A non-null raw UObject pointer is valid.”** A stale address or an object already entering destruction defeats a null-only check; use the appropriate validity contract and avoid caching unsupported raw edges.
4. **“Outer means GC owner.”** Containment and identity are not a substitute for a reference edge.
5. **“`TSharedPtr<UObject>` is safer.”** It belongs to a separate reference-counted system and is not the normal way the UObject collector discovers reachability.
6. **“Soft means weak but faster.”** A soft reference carries a path and supports deferred loading; a weak pointer observes a loaded object without retaining it.
7. **“Constructors are instance initialisation hooks.”** They also initialise class defaults. Runtime/world-dependent work belongs later.
8. **“GC fixes all leaks.”** Strong edges, roots, delegates/closures, and non-UObject bridges can retain objects unintentionally; GC pressure can also hitch.

### Debugging workflow: object disappears

1. Reproduce with a deliberate GC (`gc.CollectGarbageEveryFrame 1` is a stress tactic, not a shipping setting) or explicit collection in a controlled test.
2. Identify who is supposed to retain the object. Write the intended strong-reference chain on paper.
3. Check every member edge is visible to GC and uses the intended strong/weak semantics.
4. Check async callbacks: a raw capture may outlive its object; a weak capture must be resolved at execution time.
5. Check creation: correct factory, correct context, and no assumption that `Outer` alone retains the child.
6. Log names, paths, classes, and validity at creation, hand-off, callback, and failure—not every tick.
7. Reduce to a tiny owner/child test and force repeated GC. If experimental incremental GC is involved, use its verification/stress console variables described by Epic. [SRC-EPIC-005]

### Debugging workflow: object never disappears or GC hitches

1. Confirm the symptom in Unreal Insights and correlate frame spikes with GC rather than guessing.
2. Search for reflected strong references, root-set additions, `TStrongObjectPtr`, global/static holders, and native owners that report references.
3. Inspect whether a cache should be weak, an asset reference should be soft, or a large graph is retained by one accidental edge.
4. Measure object count and allocation churn; pooling can reduce churn but increases retained memory and reset complexity.
5. Remove causes before tuning GC intervals. Longer intervals can turn small pauses into rarer, larger pauses.

### What a three-year engineer should know

You should confidently explain UHT's place in the build, what reflection enables, why the CDO changes constructor judgement, the reachability model, and the pointer table above. You should be able to diagnose one missing-edge bug and one accidental-retention bug. You do not need to recite collector implementation files or every property flag.

**Specialist depth:** custom reference collection through `FGCObject`, clusters, collector internals, incremental write barriers, low-level object arrays/serial numbers, async UObject constraints, and engine-source lifecycle hooks.

### Strong interview answer pattern

For a pointer/lifetime question:

1. Name the lifetime domain: ordinary C++ or UObject.
2. State the intended relationship: retain, observe, locate/load, or own deterministically.
3. Choose the type and explain how the relevant system discovers it.
4. Mention the failure mode of the nearest alternative.
5. Add version context only when it changes the answer.

Example: “This is a persistent member on a UObject owner and the target must stay alive, so I would use `UPROPERTY() TObjectPtr<T>`. The reflected property gives GC a strong edge and supports engine services such as serialisation where configured. A raw member would not express that contract; a weak pointer would allow collection; a soft pointer would be appropriate if this were an asset dependency I wanted to load on demand. In UE5, `TObjectPtr` also supports newer GC and cook-time machinery.” [SRC-EPIC-005] [SRC-EPIC-009]

### Hands-on verification

Complete Project 7A in [[ue_hands_on_projects]]. The proof is not that it compiles; it is a table of predicted and observed outcomes under forced collection, including one prediction you initially got wrong and the corrected model.

### Conflict and uncertainty notes

- Epic's current object documentation can use broad wording such as “smart pointers are not intended for UObjects”, while the dedicated pointer guide documents UObject-specific `TStrongObjectPtr`, `TWeakObjectPtr`, and `TSoftObjectPtr`. Resolve this as: ordinary shared-pointer ownership is not the UObject model; purpose-built UObject pointer wrappers are part of that model. [SRC-EPIC-003] [SRC-EPIC-009]
- Historical documentation often treats raw `UPROPERTY` UObject pointers as the standard strong form. For UE5.3–UE5.6 curriculum code, prefer reflected `TObjectPtr`; older raw reflected declarations remain important to recognise. [SRC-EPIC-005] [SRC-EPIC-009]
- Incremental reachability is experimental in UE5.6 and should not be presented as baseline shipping configuration. [SRC-EPIC-005]

## Cluster 2 — Core Types, Containers, Delegates, Casts, and the C++/Blueprint Boundary

**Priority:** P0–P1  
**Expected depth:** D3–D4  
**Version scope:** UE5.3–UE5.6 preferred; most core-type concepts are stable UE4/UE5.  
**Prerequisites:** Standard C++ value/lifetime chapter and Cluster 1 above.

### Unreal idioms are contracts, not cosmetic aliases

UE C++ is ordinary C++ plus engine-specific types, generated metadata, build rules, and subsystem contracts. `FString`, `TArray`, delegates, `Cast`, and Blueprint specifiers are not spelling exercises. Each carries semantics about identity, localisation, allocation, reflection, serialisation, lifetime, or designer-facing API shape.

A strong engineer asks:

- Is this data an identifier, mutable string, or localisable display text?
- Does this collection need order, uniqueness, or key lookup?
- Does this callback need one listener, many listeners, reflection/serialisation, or only native speed?
- Am I checking a UObject runtime type, converting an ordinary C++ hierarchy, or masking poor architecture with repeated casts?
- Is Blueprint the caller, implementation extension, data authoring layer, or owner of volatile business logic?

### `FName`, `FString`, and `FText`

| Type | Primary purpose | Mutability / identity | Typical use | Common mistake |
|---|---|---|---|---|
| `FName` | Fast repeated identifiers and comparisons | Immutable name-table identity; case-insensitive semantics | Object/property/socket/bone/tag-like names, lookup keys | Treating it as user-facing localisable text or repeatedly converting dynamic strings into names |
| `FString` | Mutable character data and general string manipulation | Owns mutable character array | Parsing, paths, logs, transient string building, external textual data | Using it for every stable identifier or user-facing localisation |
| `FText` | Culture-aware display text | Carries localisation/display history | UI labels, dialogue, formatted numbers/dates, localisable content | Converting through FString and losing localisation history |

Epic describes `FName` as a lightweight reference into a global unique-string table, useful for frequent comparisons; `FText` is the localisation type for user-facing text; `FString` is mutable and supports search/manipulation at higher cost. [SRC-EPIC-022]

Decision rule:

- If humans see it and it may be localised or culture-formatted, start with `FText`.
- If code repeatedly compares it as an identifier and does not edit it, start with `FName`.
- If code must build, parse, edit, or transmit character sequences, use `FString`.

Conversions are not free or always lossless. `FString` to `FName` loses case distinctions under FName semantics. `FText` to `FString` discards localisation history/context. `FString` to `FText` creates display text but does not retroactively create a stable localisation key. Convert at boundaries, not back and forth in hot code. [SRC-EPIC-022] [SRC-EPIC-023]

Use `TEXT("...")` for TCHAR literals in target-era UE code. Use `LOCTEXT`/`NSLOCTEXT` for localisable literals and `INVTEXT`/culture-invariant construction only when the content is intentionally not translated, such as a user-supplied external name.

### UE containers are value-owning types

`TArray`, `TSet`, and `TMap` own their elements and follow value semantics. Copying produces an independent collection; destruction destroys contained values. Choose from access semantics:

| Need | Container | Core model |
|---|---|---|
| Ordered contiguous sequence, index access | `TArray<T>` | Dynamic contiguous array |
| Unique values, order irrelevant, hashed lookup | `TSet<T>` | Hash set backed by sparse storage |
| Unique key to value lookup | `TMap<K,V>` | Hashed key/value pairs backed by set-like storage |
| Multiple values per key | `TMultiMap<K,V>` | Multi-key map; use only when this relationship is intentional |

`TArray` is the default general sequence: contiguous, cache-friendly, and deep-copying as a value. Growth can relocate storage and invalidate pointers/references. Epic's target-era documentation also states that `TArray` assumes its element type is trivially relocatable in Unreal's sense, which is an important difference from simply substituting arbitrary self-referential C++ types. [SRC-EPIC-024]

`TSet` and `TMap` rely on equality and hashing (`GetTypeHash` by default). Iteration order is not a stable contract. Sparse backing storage can contain gaps, and reallocation can invalidate element pointers even though removal does not compact exactly like an array. Never serialise or replicate gameplay meaning by incidental hash iteration order. [SRC-EPIC-025] [SRC-EPIC-026]

#### `std::vector` versus `TArray`

| Dimension | `std::vector<T>` | `TArray<T>` |
|---|---|---|
| Ecosystem | ISO C++ standard library | Unreal Core and engine APIs |
| Storage | Contiguous dynamic sequence | Contiguous dynamic sequence |
| Ownership | Owns elements; value semantics | Owns elements; value semantics |
| Reflection/UPROPERTY | Not supported as a reflected UE property | Supported for reflected element types |
| Serialisation/Blueprint/replication integration | Manual/project-specific | Engine integrations where property/type supports them |
| Relocation model | Uses C++ construction/move requirements | UE assumes types are safely relocatable according to UE traits/model |
| APIs | `push_back`, `emplace_back`, `reserve`, `erase` | `Add`, `Emplace`, `Reserve`, `RemoveAt`, `RemoveAtSwap`, etc. |
| Invalidations | Standard operation-specific rules | Similar risk from allocation/mutation, but verify UE-specific APIs rather than assuming identity |
| Best default in UE-facing interfaces | Useful in pure/internal standard C++ boundaries | Usually preferred when crossing UE reflection/engine APIs |

Do not choose `TArray` because it is “always faster” or `std::vector` because it is “more standard”. Choose the ecosystem contract. Keep standard types at library boundaries when appropriate, and convert once at explicit boundaries rather than churning between representations.

### Removal and order are design decisions

`RemoveAt` preserves the relative sequence by shifting elements; `RemoveAtSwap` fills the gap from the end and changes order more cheaply. The second is not a magic optimisation if callers rely on ordering or stable indices. Likewise, using `TSet` for uniqueness communicates that order is not meaningful. Container choice should expose invariants to readers.

For hot loops:

1. Prefer contiguous values when practical.
2. Reserve when size can be estimated, but do not use reserve as a lifetime guarantee.
3. Avoid repeated `Find` plus insertion when an API can combine the operation.
4. Store stable IDs rather than long-lived element pointers when mutation is common.
5. Profile allocator churn and cache behaviour before building a custom allocator.

### Casting and type relationships

Use `Cast<T>(Object)` for checked runtime conversion among UObject-aware types; it returns null when the object is not compatible. Use `CastChecked` only when incompatibility is a programming invariant violation and a hard diagnostic is appropriate. `IsA<T>()` asks the type question without needing the converted pointer. [SRC-EPIC-027]

Do not use C-style casts. For ordinary non-UObject C++ hierarchies, use the appropriate standard cast and respect project RTTI configuration; `dynamic_cast` is not a drop-in assumption in UE projects. `ExactCast` is specialist/rare: it asks for exact class rather than accepting subclasses, which often violates polymorphic expectations.

```cpp
if (const AMyInteractable* Interactable = Cast<AMyInteractable>(Candidate))
{
    Interactable->Preview();
}
```

Repeated casting is often an architecture smell, not a CPU crisis. If a system repeatedly casts many unrelated Actors to discover capabilities, consider an interface, component, gameplay tag/query, registered service, or event. A single cast at a boundary can be perfectly clear; eliminating every cast with elaborate abstraction can be worse.

### Creation APIs encode lifecycle

| Need | API | Essential contract |
|---|---|---|
| Runtime UObject | `NewObject<T>(Outer, ...)` | Managed UObject creation; retain through an appropriate reference edge |
| Runtime Actor | `World->SpawnActor<T>()` | World Actor spawn path, collision/spawn parameters, Actor lifecycle |
| Configure before construction finishes | `SpawnActorDeferred` + `FinishSpawning` | Set exposed/dependency state before construction and BeginPlay path completes |
| Constructor-time component/subobject | `CreateDefaultSubobject<T>()` | Defines class/default subobject structure; constructor only |
| Load known object synchronously | `LoadObject<T>()` | Blocking load; avoid casual hot-path/editor-only assumptions |
| Asset dependency loaded later | `TSoftObjectPtr` + streamable/asset manager flow | Path-based dependency with explicit async ownership/completion |

These are not interchangeable allocation functions. `NewObject<AActor>` skips the world spawn contract; `SpawnActor<UObject>` is nonsensical; `CreateDefaultSubobject` at runtime misunderstands CDO/class construction. Deferred Actor spawning is useful when required data must be set before Blueprint construction or final spawn, but every deferred path must call `FinishSpawning` or deliberately abort/clean up. [SRC-EPIC-004] [SRC-EPIC-011] [SRC-EPIC-015]

### Delegates: choose the weakest feature set that meets the boundary

Delegates are type-safe callback objects. The major axes are:

- **Single-cast versus multicast:** one target/result versus notifying multiple listeners.
- **Native versus dynamic:** native C++ speed/flexibility versus reflection, serialisation, and Blueprint binding.
- **Binding lifetime:** raw, UObject-aware, shared-pointer-aware, weak lambda, or explicit handle management.

| Requirement | Typical choice |
|---|---|
| One native callback, possibly with return | Native single-cast delegate |
| Notify several native systems | Native multicast delegate |
| Designer binds an event in Blueprint | Dynamic multicast property with `BlueprintAssignable` |
| Blueprint supplies one overridable implementation | `BlueprintImplementableEvent` or `BlueprintNativeEvent`, not necessarily a delegate |

Dynamic delegates can be serialised and locate functions through reflection, and are slower than native delegates. Use them where that capability is required, not throughout internal C++ call chains. [SRC-EPIC-028] [SRC-EPIC-029]

Binding choice matters more than the declaration macro:

- `AddRaw` does not track raw object lifetime; unbind before destruction or prove the source outlives the delegate.
- `AddUObject`/UObject-aware binding avoids invoking a destroyed UObject under the delegate's documented weak binding behaviour.
- `AddSP`/`AddSPLambda` connects to shared-pointer lifetime.
- `AddWeakLambda` guards a UObject capture.
- `AddLambda` with raw captures is your lifetime responsibility.

Store `FDelegateHandle` when targeted unbinding is needed. Bind/unbind symmetrically in lifecycle hooks; avoid `RemoveAll(this)` as a substitute for knowing what was registered when precise handles are practical. Never broadcast a public delegate while mutating the same listener collection recursively without understanding the delegate implementation and re-entrancy contract.

### Blueprint exposure is API design

Blueprint is a complete visual scripting environment and an intended designer extension layer, not merely “slow C++”. Epic explicitly supports C++ baseline systems extended by Blueprint classes and graphs. [SRC-EPIC-030] [SRC-EPIC-031]

A robust division often looks like:

- **C++:** invariants, replication/authority, performance-sensitive loops, reusable algorithms, platform integration, complex lifetime, automated-test seams.
- **Blueprint/DataAssets:** asset composition, tuning, presentation, level-specific orchestration, designer-authored variants, event responses.
- **Either, by evidence:** moderate gameplay logic with clear ownership and profiling results.

The decision is about change ownership, iteration, invariants, debugging, reuse, and performance—not ideology.

### Exposure specifiers communicate different capabilities

| Specifier | Contract |
|---|---|
| `BlueprintType` | Instances/references of the type can be variables/pins in Blueprint |
| `Blueprintable` | Blueprint classes may derive from the class |
| `BlueprintCallable` | Blueprint may invoke an impure/scheduled native function |
| `BlueprintPure` | Blueprint may evaluate a side-effect-free function without exec pins |
| `BlueprintReadOnly` / `BlueprintReadWrite` | Blueprint property access; independent from editor editability |
| `BlueprintAssignable` | Blueprint may bind listeners to a dynamic multicast delegate property |
| `BlueprintImplementableEvent` | Blueprint supplies implementation; no native fallback |
| `BlueprintNativeEvent` | Native default `_Implementation` exists and Blueprint may override |

Epic's exposure guide distinguishes these class, property, callable, and override paths. [SRC-EPIC-031]

Prefer narrow functions over exposing writable state. A public `SetHealth(float)` Blueprint call can validate authority, clamp values, fire events, and preserve invariants; `BlueprintReadWrite` health lets every graph bypass policy. Use `EditDefaultsOnly`/`EditInstanceOnly`/`Visible*` according to authoring intent rather than `EditAnywhere` by reflex.

### Pure nodes and events have hidden execution implications

`BlueprintPure` means no observable mutation, not “small” or “fast”. Pure nodes can be reevaluated where their outputs are needed, potentially multiple times. Do not mark expensive searches pure merely to remove an exec wire. Cache deliberately or expose an impure query when scheduling/cost visibility matters.

Use `BlueprintImplementableEvent` when an absent Blueprint implementation is acceptable. Use `BlueprintNativeEvent` when a stable native default is required and designers may override it. In Blueprint overrides, forgetting “Call to Parent” can skip native behaviour; design APIs so mandatory invariants are enforced in non-overridable wrappers:

```cpp
void ApplyDamage(float Amount)
{
    // mandatory validation and state mutation
    OnDamageApplied(Amount); // extension hook
}

UFUNCTION(BlueprintImplementableEvent)
void OnDamageApplied(float Amount);
```

This “template method” shape keeps invariants native while giving presentation/extension control to Blueprint.

### Blueprint architecture and performance workflow

Common red flags:

1. Tick-driven polling for UI/state that could be event-driven.
2. Repeated `GetAllActorsOfClass`, broad casts, and object discovery in hot graphs.
3. Huge Event Graphs with implicit dependencies and no ownership boundary.
4. Long chains of hard asset/class references causing load coupling.
5. Writable public variables bypassing authority and validation.
6. Pure nodes that perform expensive work repeatedly.
7. Level Blueprint holding reusable gameplay logic.
8. Dynamic delegate binding duplicated during reconstruction/BeginPlay.

Debug/profiling loop:

1. Reproduce with Blueprint debugger, breakpoints, watch values, and execution trace.
2. Identify event frequency and call ownership; eliminate accidental repeated binding first.
3. Use Unreal Insights/stat tooling to establish whether Blueprint/game-thread work is material.
4. Move a hotspot to C++ only after identifying the expensive boundary; retain a small Blueprint-facing API.
5. Verify cooked/packaged behaviour, especially asset references and editor-only assumptions.

### Common bugs and misconceptions

1. **FName is a cheaper FString.** It is identifier semantics, not mutable text.
2. **FText is FString with localisation later.** Localisation history/context can be lost through conversion.
3. **Hash container iteration is stable.** It is not an ordering contract.
4. **Reserve makes TArray pointers safe.** Later growth/removal/reorder still invalidates assumptions.
5. **Cast is always bad.** Boundary casts are fine; repeated discovery casts indicate coupling.
6. **Dynamic delegates are better because Blueprint can see them.** Reflection features have cost; choose them only at the Blueprint/serialisation boundary.
7. **BlueprintPure means optimised.** It means side-effect-free evaluation semantics.
8. **C++ should contain all logic.** That can destroy iteration and designer ownership.
9. **Blueprint should own all gameplay.** That can weaken invariants, diffability, testability, and hot-path performance.
10. **BlueprintReadWrite is convenient.** It often leaks mutation authority.

### Strong interview answer pattern

For “C++ or Blueprint?”:

1. Identify who changes the behaviour and how often.
2. State invariants, authority, lifetime, performance, and test needs.
3. Put stable policy/mechanics in C++ and expose a narrow extension/data surface.
4. Explain how events/delegates/data assets connect layers.
5. Name one profiling or packaged-build verification step.

Example: “Damage authority, clamping, replication, and death state stay in C++ because they are invariants and need tests. Designers tune damage definitions through DataAssets and implement hit presentation through a Blueprint event. Health is read-only to Blueprint; mutation goes through validated calls. UI listens to a health-changed delegate rather than polling. If the event layer becomes hot, I measure it in Insights before changing the boundary.”

### What a three-year engineer should know

Choose string and container types by semantics; understand value ownership and invalidation; use UObject-aware casts; select the right creation path; choose delegate features and bindings by listener/lifetime/reflection needs; expose narrow Blueprint APIs; and profile graphs before migrating them.

**Specialist depth:** custom allocators and `TInlineAllocator`, `TStructOpsTypeTraits` relocation/serialisation hooks, delegate implementation details, custom K2/async nodes, Blueprint compiler/VM internals, nativisation history, and property-system custom thunks.

### Hands-on verification

Extend Project 1 with a type/container audit and a C++/Blueprint boundary review. Record every exposed property/function, its intended caller, mutation authority, asset dependency, and event frequency. Replace one polling graph with an event, one public writable property with a validated API, and one hard reference with a soft reference where load timing justifies it.

### Conflict and uncertainty notes

- Current Epic pages may resolve to UE5.7 even with target-era navigation. Concepts here are stable, but exact overloads/traits should be checked against UE5.6 headers for version-sensitive code.
- Epic's TArray documentation guarantees an empty source after its documented `MoveTemp` example, which is stronger than the generic standard-library valid-but-unspecified rule shown earlier; do not generalise either contract to unrelated types. [SRC-EPIC-024] [SRC-CPP-004]
- Blueprint performance depends on call frequency, node work, platform, and engine version. Avoid unsupported universal multipliers; profile the project.

## Cluster 3 - Specialist UE C++ Contracts, Diagnostics, and Runtime Boundaries

**Priority:** P0-P1 for gameplay/generalist/tools roles  
**Expected depth:** D3-D4, with selected D5 awareness for engine/tools interviews  
**Version scope:** UE5.3-UE5.6 preferred. Maintained Epic API pages can resolve beyond the target branch; exact signatures and generated-code behaviour must be checked against the project engine.  
**Prerequisites:** Clusters 1 and 2, gameplay framework lifecycle, networking authority, C++ callable lifetime, standard C++ concurrency, and build/module basics.

### The key upgrade: name the consuming system

Weak UE C++ answers often list macros. Strong answers name which engine system consumes the declaration and when. `UCLASS(Abstract)` affects editor/class behaviour; `UFUNCTION(BlueprintCallable)` affects Blueprint-callable API; `UPROPERTY(SaveGame)` only matters when the save/archive path actually serialises save-game fields; `meta=(WorldContext=...)` changes generated Blueprint node behaviour but should not become game authority. [SRC-EPIC-032] [SRC-EPIC-034] [SRC-EPIC-035] [SRC-EPIC-036]

Use this sentence pattern in interviews:

> "This specifier expresses X to system Y at phase Z. It does not by itself guarantee A or B."

Examples:

- `BlueprintReadOnly` exposes read access to Blueprint; it does not make the value immutable in C++ or authoritative on the server. [SRC-EPIC-031] [SRC-EPIC-035]
- `Config` marks a property for config serialisation when the class has an appropriate config class specifier and the config load/save path is used; it is not a replicated settings system. [SRC-EPIC-032] [SRC-EPIC-035]
- `SaveGame` marks fields for save-game archives that honour the flag; it does not automatically persist all references, version schemas, async assets, or runtime Actors. [SRC-EPIC-035]
- `Transient` blocks normal persistence for that property or class; it does not mean "temporary lifetime" in C++ or GC terms. [SRC-EPIC-032] [SRC-EPIC-035]
- Metadata such as `DisplayName`, `ToolTip`, `WorldContext`, `BlueprintThreadSafe`, or `ExposedAsyncProxy` describes editor/node/tooling behaviour; Epic explicitly warns that metadata should not be treated as runtime game logic. [SRC-EPIC-032] [SRC-EPIC-036]

### Class, struct, enum, function, property: choose the narrowest reflection surface

| Declaration | What it is good at | Interview-grade judgement |
|---|---|---|
| `UCLASS` | UObject identity, inheritance, reflection, Blueprint class use, editor/object lifecycle | Use when independent identity, polymorphic UObject behaviour or engine services are needed. Do not make every data record a UObject. |
| `USTRUCT` | Reflected value type, details panels, serialisation, `UPROPERTY` fields, DataTable rows | Use for value state and data records. Remember the struct value is not a standalone GC object. [SRC-EPIC-008] [SRC-EPIC-033] |
| `UENUM` | Reflected enum values and Blueprint pins | Use `enum class` and a narrow underlying type where Blueprint requires it; design invalid/default states deliberately. [SRC-CPP-025] |
| `UFUNCTION` | Reflected callable entry point for Blueprint, RPC, console `exec`, events and generated thunks | Keep callable surfaces narrow. Separate validation/invariants from optional extension events. [SRC-EPIC-034] |
| `UPROPERTY` | Reflected field metadata for editor, Blueprint, serialisation, config, save, replication declarations and GC reference discovery | State which capability is needed. Avoid `EditAnywhere, BlueprintReadWrite` as a default. [SRC-EPIC-007] [SRC-EPIC-035] |

Specifier misuse usually comes from mixing authoring, runtime state and persistence. A tuning value might be `EditDefaultsOnly` because designers set it on class defaults. A runtime health value might be `VisibleInstanceOnly` or not editor-visible at all. A replicated value needs networking declarations and server authority, not merely Blueprint access. A save value needs a stable schema and archive path, not merely a field flag.

### Practical specifier buckets

| Bucket | Examples | Main question |
|---|---|---|
| Editor authoring | `EditDefaultsOnly`, `EditInstanceOnly`, `VisibleAnywhere`, `Category`, `AdvancedDisplay` | Who is allowed to author or inspect this, and on defaults or instances? |
| Blueprint API | `BlueprintType`, `Blueprintable`, `BlueprintCallable`, `BlueprintPure`, `BlueprintReadOnly`, `BlueprintAssignable` | Is Blueprint a caller, implementer, data owner or listener? |
| Persistence/config | `Config`, `GlobalConfig`, `SaveGame`, `Transient`, `DuplicateTransient`, `NonPIEDuplicateTransient` | Which serialisation path consumes this flag, and what is intentionally excluded? |
| Object/reference semantics | `Instanced`, `EditInline`, `DefaultToInstanced`, `Export` | Is this an embedded subobject instance or an asset/object reference? |
| Networking | `Replicated`, `ReplicatedUsing`, RPC function specifiers | What changes on the server, what is sent, and what local callback/order assumptions are valid? |
| Tooling/metadata | `DisplayName`, `ToolTip`, `WorldContext`, `DeterminesOutputType`, `AutoCreateRefTerm`, `BlueprintThreadSafe` | Does this improve authoring/node shape without becoming runtime authority? |

The red flag is an answer that says "add UPROPERTY so Unreal sees it" without naming what "sees" means. The editor, GC, serialiser, Blueprint compiler and replication layout are separate consumers.

### Config, defaults, save, duplication and transient state

Persistent state in Unreal has several overlapping but different lifetimes:

| State kind | Typical home | UE C++ guidance |
|---|---|---|
| Class default tuning | CDO/defaults, Blueprint defaults, Data Assets | `EditDefaultsOnly` or DataAsset fields are usually safer than per-instance ad hoc mutation. |
| Project/user config | `.ini` backed config classes/properties | Use class `Config=...` plus property `Config`; define reload/save policy and avoid using config as replicated gameplay state. [SRC-EPIC-032] [SRC-EPIC-035] |
| Runtime state | Actors, Components, subsystems, plain structs | Keep mutation behind methods/events and do not overexpose writable Blueprint fields. |
| Save-game state | explicit SaveGame object/archive path | Store stable IDs and schema versions; `SaveGame` flags help selected archives but do not design the save system for you. [SRC-EPIC-035] |
| Duplicated/editor-copied state | PIE, copy/paste, duplicate, archetype propagation | Use duplicate/transient flags deliberately for caches, handles and generated runtime-only data. [SRC-EPIC-035] |
| Ephemeral cache | non-persistent fields, weak refs, derived data | Mark transient where appropriate and rebuild from durable truth. |

Common bugs:

1. A property is `Config` but the class lacks the relevant class-level config setup.
2. A field is `SaveGame` but it stores a live Actor pointer, dynamic object name or asset loaded only in editor.
3. A runtime cache is duplicated into PIE or copied asset instances because it was not transient/duplicate-transient.
4. A designer mutates an instance field that should have been a class default or Data Asset definition.
5. A value is saved, replicated and config-driven at the same time without a clear authority order.

Interview answer pattern for state:

1. Classify the state lifetime: default, config, runtime, replicated, saved, duplicated, cache.
2. Pick one source of truth.
3. Name conversion/rebuild points between lifetimes.
4. Add version/migration and invalid-reference policy where persistence is involved.
5. Test editor, PIE, cooked build and dedicated server if those paths matter.

### Object identity: package, outer, name, path and flags

UObjects have identity beyond their C++ address. They have a class, object name, outer chain, package context and object flags. `NewObject` takes an Outer, optional name, flags and template; those inputs affect identity/defaulting/visibility, but they do not replace an explicit lifetime policy. [SRC-EPIC-003] [SRC-EPIC-004]

Think about object APIs by intent:

| Intent | API family | Failure mode |
|---|---|---|
| Create a runtime UObject | `NewObject` | Wrong Outer/name/flags, no retaining edge, doing Actor work outside spawn path. |
| Create an Actor in a World | `SpawnActor`, deferred spawn | Missing World, wrong collision/spawn params, failing to finish deferred spawn. |
| Duplicate a UObject graph | `DuplicateObject` or editor duplication path | Runtime-only fields copied, instanced subobjects shared unexpectedly, references still point to source graph. |
| Find already-loaded object | `FindObject` style lookup | Treating lookup as load, path/name mismatch, relying on brittle global names. |
| Load known asset/object synchronously | `LoadObject` | Blocking load on hot path, missing cook inclusion, editor-only path works but package fails. |
| Reference asset/class for later load | soft object/class path or pointer | Assuming loaded object exists, missing async cancellation, stale callback after owner destruction. |

`Outer` is best described as containment and identity context. `Within=...` can constrain required outer class at declaration level, but neither `Outer` nor `Within` should be explained as "ordinary C++ ownership". If a child UObject must survive, make the retaining edge visible to GC or implement a native reference collector bridge. [SRC-EPIC-004] [SRC-EPIC-032] [SRC-EPIC-043]

For debugging identity bugs, log:

```cpp
UE_LOGFMT(LogTemp, Warning,
    "Object {Name} Class={Class} Outer={Outer} Package={Package} Flags={Flags}",
    ("Name", GetNameSafe(Object)),
    ("Class", GetNameSafe(Object ? Object->GetClass() : nullptr)),
    ("Outer", GetNameSafe(Object ? Object->GetOuter() : nullptr)),
    ("Package", GetNameSafe(Object ? Object->GetPackage() : nullptr)),
    ("Flags", Object ? static_cast<uint32>(Object->GetFlags()) : 0));
```

Use names and paths for diagnosis, not as a substitute for stable gameplay IDs. Object paths can change through rename, redirector, duplication, PIE prefixing, cooked packaging and asset migration.

### Reflected interfaces versus native C++ interfaces

Unreal interfaces use a two-part model: a reflected `UINTERFACE` type for the UObject/reflection layer and an `I...` C++ interface type for callable behaviour. This lets classes implement a reflected interface and allows Blueprint integration where configured. [SRC-EPIC-037]

| Need | Prefer | Why |
|---|---|---|
| Cross-UObject capability visible to Blueprint/reflection | `UINTERFACE` + `IInterface` | Can be queried on UObjects and exposed to Blueprint. |
| Pure native helper polymorphism inside one module/library | ordinary abstract C++ interface | Less reflection overhead and no UObject requirement. |
| Reusable state plus behaviour | `UActorComponent` or UObject subobject | Interfaces do not store per-instance state by themselves. |
| Data-defined behaviour selection | strategy object/class/DataAsset plus interface | Keeps authored choice explicit and testable. |

Rules of thumb:

1. Use an interface to express "this object can do X", not "this object is secretly one of these concrete classes".
2. Do not use a `UINTERFACE` as a data container. Put state in the implementer or a Component.
3. In C++, prefer interface execution helpers or generated calls when Blueprint implementers are possible; direct C++ virtual calls only cover native implementations.
4. For networked gameplay, an interface call is still just a local call. It does not add authority, ownership or RPC routing.
5. Interfaces reduce casting to concrete classes; they do not remove lifecycle, null, authority or performance design.

Common bug: using a reflected interface for every dependency because "casts are bad". A one-time boundary cast to a known Component can be clearer than an interface with no stable contract. Use the abstraction when multiple implementers are real or expected.

### Logging, `UE_LOGFMT`, `check`, `verify`, and `ensure`

Logging and assertions are different tools:

| Tool | Use | Do not use for |
|---|---|---|
| `UE_LOG` | human-readable chronological diagnostic lines | high-volume structured telemetry without categories/verbosity. |
| `UE_LOGFMT` | structured fields, named/positional formatting, better object/correlation logging | mixing named and positional styles or building hot-path spam. |
| `check` | impossible programmer invariant; crash is acceptable because continuing is unsafe | recoverable runtime conditions or user/content errors. |
| `verify` | expression must execute even when check-style assertion may be compiled out | control flow that should differ by build config. |
| `ensure` | report unexpected but survivable condition and continue | silently tolerating an invariant breach forever. |

Epic documents log categories, verbosity filters, command-line/console verbosity control, `UE_LOGFMT` field rules and on-screen messages. `UE_LOGFMT` is UE5.2+ and needs the structured logging include. [SRC-EPIC-038]

The practical pattern is:

```cpp
DECLARE_LOG_CATEGORY_EXTERN(LogInventory, Log, All);

bool UInventoryComponent::ServerTryMoveItem(const FInventoryMoveRequest& Request)
{
    if (!ensureMsgf(OwnerState.IsValid(), TEXT("Inventory has no owner state")))
    {
        UE_LOGFMT(LogInventory, Error,
            "Rejected move {MoveId}: missing owner state on {Owner}",
            ("MoveId", Request.MoveId),
            ("Owner", GetNameSafe(GetOwner())));
        return false;
    }

    checkf(Request.SourceSlot != Request.TargetSlot,
        TEXT("Move request should have been normalised before validation"));

    // Validate player/content state and return structured rejection on normal failure.
    return ApplyMoveIfValid(Request);
}
```

Use categories per system so command-line filters can increase one subsystem without drowning the whole log. Include stable correlation fields: object ID, player/controller, World/net mode, request ID, prediction key, save schema, async generation or asset path. Avoid logging every tick unless the category is very verbose and disabled by default.

Assertion workflow:

1. Decide whether the condition is impossible programmer error, suspicious but survivable state, or normal runtime/content failure.
2. Use `check` only for the first category.
3. Use `ensure` to capture stack/report while keeping the session alive for investigation.
4. Use `verify` when the expression has required side effects.
5. Add a normal error path for content, network, save, input or user-driven failure.

The interview trap is claiming "`ensure` is a softer check". Better: "`ensure` is diagnostic evidence for unexpected state I may be able to survive; it is not validation policy." [SRC-EPIC-039]

### Core vocabulary: optional, variant, callable refs and views

UE Core has vocabulary types that express intent similarly to modern standard C++ but fit engine conventions. Exact APIs must be checked in the target branch, especially when maintained API pages move. [SRC-EPIC-040] [SRC-EPIC-041] [SRC-EPIC-042] [SRC-CPP-025]

| Intent | UE vocabulary | Good use | Misuse |
|---|---|---|---|
| Value may be absent | `TOptional<T>` | "No result" distinct from a default value | Using magic sentinel values or storing optional UObject ownership. |
| One of a closed set of value alternatives | `TVariant<...>` | Parser result, editor command payload, small state value | Replacing polymorphism when behaviour/lifetime differs deeply. |
| Non-owning callback parameter | `TFunctionRef<Signature>` | immediate algorithm callback that does not escape | Storing it for later or binding async work. |
| Owning callable | `TFunction<Signature>` or delegate | callback stored beyond call scope | Capturing raw UObject pointers without weak/generation checks. |
| Read-only contiguous view | array/span view type available in branch | API accepts many contiguous sources without copying | Returning a view to temporary storage or mutating through unclear aliases. |

For a three-year engineer, the important distinction is ownership and escape. A `TFunctionRef` is attractive for low-allocation synchronous utilities, but it is dangerous if copied into a timer/task/delegate for later. A view is attractive for allocation-free APIs, but it is only as valid as the underlying storage. A `TOptional` says absence is part of the type contract; it does not make a lifetime problem disappear.

Example:

```cpp
void ForEachAliveTarget(
    TConstArrayView<TWeakObjectPtr<AActor>> Candidates,
    TFunctionRef<void(AActor&)> Visit)
{
    for (const TWeakObjectPtr<AActor>& Candidate : Candidates)
    {
        if (AActor* Actor = Candidate.Get())
        {
            Visit(*Actor);
        }
    }
}
```

This function is synchronous. It does not retain `Visit`, and it resolves weak references at use time. If the work becomes async, change the API shape instead of quietly storing the function reference.

### Async Blueprint nodes and proxy lifetime

Async Blueprint APIs often use a UObject proxy such as a `UBlueprintAsyncActionBase` subclass. The proxy represents an operation, exposes dynamic multicast completion delegates, and needs explicit activation, cancellation and lifetime policy. [SRC-EPIC-036] [SRC-EPIC-044]

A safe async proxy design states:

1. Who owns or retains the proxy while the operation is in flight.
2. Which World/game instance/player context the request belongs to.
3. Which request generation or handle invalidates stale completion.
4. What cancellation means on owner destruction, travel, timeout and manual cancel.
5. Which thread invokes native completion and where Blueprint delegates are broadcast.
6. Whether completion delegates are one-shot and whether they clear native handles.
7. What happens when the asset/network/task succeeds after the proxy is already stale.

Common async Blueprint bugs:

- Proxy object is collected before completion because no retained reference exists.
- Proxy is retained forever by a delegate, timer, streamable handle or subsystem array.
- Completion broadcasts after World teardown or after the owning widget/Actor is gone.
- Worker thread touches UObjects or broadcasts Blueprint delegates off the Game Thread.
- Repeated activation registers duplicate native callbacks.
- Failure path never clears handles or marks the proxy complete.

Minimal mental model:

```cpp
UCLASS()
class UAsyncFindTargetProxy : public UBlueprintAsyncActionBase
{
    GENERATED_BODY()

public:
    UPROPERTY(BlueprintAssignable)
    FTargetFoundSignature OnFound;

    UPROPERTY(BlueprintAssignable)
    FTargetFailedSignature OnFailed;

    virtual void Activate() override;

private:
    TWeakObjectPtr<UWorld> World;
    int32 Generation = 0;
    FDelegateHandle NativeHandle;

    void CompleteIfCurrent(int32 CompletedGeneration, TWeakObjectPtr<AActor> Result);
    void Cleanup();
};
```

The interesting part is not the macro. The interesting part is cancellation, generation checks, Game Thread handoff, delegate cleanup and proving the proxy survives exactly as long as the operation should.

### `FGCObject`, root set, and native owners

Most gameplay UObject references should be on reflected UObject fields using the pointer choices from Cluster 1. Specialist code sometimes has a native manager, Slate object, task helper or non-UObject cache that must report UObject references to GC. `FGCObject` is a bridge for native objects to add references during GC traversal. [SRC-EPIC-043]

Use it only when the owner genuinely cannot be a UObject with reflected properties. Document:

- owner lifetime and destruction order,
- which UObjects are reported,
- when references are added/removed,
- whether the bridge crosses Worlds/PIE sessions,
- how leaked native owners are detected,
- why a weaker reference or explicit loaded asset handle is insufficient.

`AddToRoot` is a sharper tool. It can retain a UObject globally until explicitly removed, which makes it useful for narrow engine/bootstrap cases and risky for gameplay. If an interview answer reaches for `AddToRoot` before `UPROPERTY`, `TStrongObjectPtr`, `FGCObject` or a clearer owner, challenge it.

Debugging retained/missing references:

1. Write the intended retention graph.
2. Check reflected properties first.
3. Search native strong holders: `TStrongObjectPtr`, root additions, singleton arrays, delegates, async handles and `FGCObject` implementers.
4. Force travel/PIE shutdown and collection, not just a local function exit.
5. Prefer removing accidental strong edges over tuning GC.

### UObject and thread boundaries

Standard C++ and UE tasks can run work off the Game Thread, but that does not make arbitrary UObject access safe. A robust off-thread workflow snapshots immutable/plain data on the Game Thread, processes that data in tasks, then applies results back on the Game Thread after validating owner/World/generation. [SRC-CPP-026] [SRC-CPP-025]

Safe shape:

```cpp
struct FTargetingSnapshot
{
    FVector Origin;
    TArray<FVector> CandidateLocations;
    int32 OwnerGeneration = 0;
};

// Game Thread: copy plain data out of UObjects.
FTargetingSnapshot Snapshot = BuildSnapshotFromWorld();

// Worker: no UObject dereference.
auto Result = ComputeBestCandidate(Snapshot);

// Game Thread: revalidate owner/generation before touching UObjects.
ApplyResultIfCurrent(Result.OwnerGeneration, Result);
```

Common thread-boundary bugs:

1. Lambda captures `this` or raw UObject pointer for deferred work.
2. Worker reads `TArray` owned by a UObject while Game Thread mutates it.
3. Completion assumes the same World after travel.
4. Task handle release is treated as cancellation.
5. Async operation owns a UObject strongly and hides retention until shutdown.
6. Assertion/logging path touches unsafe object state from the wrong thread.

The answer pattern:

1. State that UObjects and gameplay state are Game Thread owned unless a documented subsystem says otherwise.
2. Snapshot plain data.
3. Use tasks for computation or IO boundary work, not live object mutation.
4. Return through a Game Thread dispatcher or subsystem callback.
5. Revalidate weak owner, World, generation and cancellation token.
6. Record task IDs and generation in logs.

### Debugging workflow: specifier or generated-code bug

1. Read the first UHT/compiler error above the generated-file cascade.
2. Inspect the declaration immediately before the macro or generated header include.
3. Reduce to one reflected declaration and rebuild.
4. Confirm class/type prefixes and supported reflected types. Epic's coding standard notes UHT relies on correct prefixes in many cases. [SRC-CPP-025]
5. Separate UHT parse errors from C++ compile errors, link errors and Blueprint compiler errors.
6. Confirm module dependencies: a reflected type in a public header needs the right public dependency and include/export boundary.
7. If a specifier compiles but behaviour is wrong, identify the consuming system: details panel, Blueprint node, config archive, save archive, duplication, replication, GC or cook.

### Debugging workflow: async callback after destruction

1. Add a request ID, owner generation, World name and net mode to every start/cancel/complete log.
2. Cancel in `EndPlay`, subsystem deinitialisation, widget release or proxy cleanup as appropriate.
3. Resolve weak owners at completion time; do not keep raw captures.
4. Marshal completion to the Game Thread before any UObject/Blueprint broadcast.
5. Reject completion if generation/World/owner no longer matches.
6. Clear native delegate handles/streaming handles/timers exactly once.
7. Test travel, PIE stop, owner destroy, manual cancel, failure and success racing with cancel.

### What a three-year engineer should know

You should be able to explain common specifier buckets by consuming system, design reflected interfaces without confusing them with Components or native abstract classes, separate config/save/transient/duplicate lifetimes, choose `NewObject`/`SpawnActor`/load/find/duplicate APIs by intent, use log categories and assertions appropriately, recognise non-owning Core vocabulary, and design async Blueprint proxies around lifetime and cancellation.

**Specialist depth:** custom thunks, Blueprint compiler/K2 nodes, property flags in engine source, custom serialisation archives, object cluster internals, package/save linker internals, exact async loading handles, custom GC reference collectors, low-level object flags and thread-safe engine subsystems.

### Strong interview answer pattern

For a UE C++ macro/specifier question:

1. Name the declaration kind.
2. Name the consuming system.
3. Name the lifecycle phase.
4. State what it does not guarantee.
5. Give one debugging or packaged-build verification step.

Example: "`SaveGame` is a property flag consumed by save-game style archives that opt into it. It does not automatically make live Actor pointers, soft assets, schema migration or multiplayer state persistence correct. I would save stable IDs and versioned value structs, rebuild runtime objects on load, and test a clean packaged build with missing/renamed definitions." [SRC-EPIC-035]

For an async/lambda/thread question:

1. Identify ownership and thread affinity.
2. Snapshot plain data or retain through an explicit handle.
3. Use weak/generation checks for UObject owners.
4. Marshal completion to Game Thread.
5. Clean up handles and prove cancel/race cases.

### Hands-on verification

Extend Project 7A with the "UE C++ Specialist Contract Lab" in [[ue_hands_on_projects]]. The expected output is not a giant code sample; it is a table that predicts, observes and explains how specifier choices change editor visibility, Blueprint API, config/save persistence, duplication, async proxy lifetime, log/assert diagnostics and GC retention.

### Conflict and uncertainty notes

- Epic specifier pages are authoritative for target-era intent, but some pages are dynamically rendered or maintained beyond the target. Compile a minimal UE5.3-UE5.6 sample before relying on a rare specifier, metadata key or generated signature. [SRC-EPIC-032] [SRC-EPIC-036]
- Metadata can be read in editor tooling, but it should not be treated as shipped gameplay authority. This is especially important for validation, security, networking and save/load rules. [SRC-EPIC-032] [SRC-EPIC-036]
- `TOptional`, `TVariant`, `TFunctionRef` and view types are useful vocabulary, but exact APIs and reflection support differ from standard C++ types and can move across engine versions. Pin target headers before implementation. [SRC-EPIC-040] [SRC-EPIC-041] [SRC-EPIC-042] [SRC-CPP-025]
- Async Blueprint node examples in the wild often omit cancellation, World ownership and stale completion handling. Treat those as incomplete teaching examples unless they prove teardown and race behaviour. [SRC-EPIC-044]
