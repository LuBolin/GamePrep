# C++ for Unreal Interviews

See also: [[ue_cpp_idioms]], [[systems_programming_for_game_interviews]], [[game_architecture_patterns]], [[cpp_interview_question_bank]], [[ue_flashcards]], [[ue_hands_on_projects]].

## Cluster 1 — Object Lifetime, Value Semantics, RAII, and Ownership

**Priority:** P0  
**Expected depth:** D4 for gameplay, engine, tools, networking, AI, and rendering engineers  
**Language scope:** C++17 fundamentals with selected C++20 awareness; Unreal projects may configure a newer standard, but do not assume it without checking the target.  
**UE connection:** Ordinary C++ and UObject lifetime are separate domains. This chapter covers ordinary C++ first, then compares the domains explicitly.

### The mental model: object, storage, lifetime, and handle

An object is not merely bytes at an address. It has a type, value, alignment, storage duration, and a lifetime with a definite beginning and end. Storage may exist before an object's lifetime begins or after it ends; dereferencing a pointer into such storage as though the object were alive can be undefined behaviour. A pointer or reference can therefore remain non-null and still dangle. [SRC-CPP-002]

Keep four ideas separate:

| Idea | Question |
|---|---|
| Storage | Where are the bytes and how long is the storage available? |
| Object lifetime | When is a value of this type actually alive in that storage? |
| Ownership | Which object or scope is responsible for ending the resource lifetime? |
| Access | Which pointers, references, iterators, or views may observe it, and for how long? |

“Stack versus heap” is too crude for strong interview reasoning. Automatic objects usually have scope-bound lifetime and dynamic allocations require an owning mechanism, but storage duration does not by itself define a semantic owner. Static and thread-local storage introduce further lifetimes, while containers own dynamically allocated buffers without requiring their callers to manage the allocation.

### Value semantics first

A value type owns its state and behaves predictably when copied, moved, assigned, compared, and destroyed. `FVector`, `std::string`, `TArray`, and `std::vector` are value-like even though some own dynamic storage internally. Prefer direct values when size, optionality, polymorphism, stable address, or sharing does not demand indirection.

Value semantics reduce questions:

- Lifetime follows the containing object.
- Copying has a defined independent-result meaning.
- Moving can transfer expensive representation.
- Containers can own elements directly.
- Fewer allocations and pointer chases often improve locality.

Do not turn “avoid allocations” into “never use pointers”. Runtime polymorphism, large stable-address objects, pImpl boundaries, graph structures, and shared external resources can justify indirection. The point is to make indirection earn its complexity.

### RAII is resource ownership, not merely memory cleanup

Resource Acquisition Is Initialisation binds a resource to an object's lifetime: acquire during construction/factory creation, release in the destructor. The resource might be memory, a lock, file, socket, GPU handle, scope guard, transaction, or temporary registration. Destruction runs on normal scope exit and stack unwinding, making cleanup structural instead of dependent on every control-flow path. The C++ Core Guidelines make RAII the default resource-management technique. [SRC-CPP-001]

```cpp
class FScopedRegistration
{
public:
    explicit FScopedRegistration(FRegistry& InRegistry)
        : Registry(InRegistry), Token(Registry.Register()) {}

    ~FScopedRegistration()
    {
        Registry.Unregister(Token);
    }

    FScopedRegistration(const FScopedRegistration&) = delete;
    FScopedRegistration& operator=(const FScopedRegistration&) = delete;

private:
    FRegistry& Registry;
    FToken Token;
};
```

The destructor makes release unavoidable for every completed construction. Copy is deleted because duplicating the token would create two apparent owners. Depending on requirements, move could transfer the token and invalidate the source.

RAII and Unreal are compatible. Use RAII for ordinary C++ resources inside UE code. UObject destruction is not scope-bound, so a `UObject*` is not made RAII-owned by wrapping it casually in a standard smart pointer. Conversely, not every type in an Unreal module should derive from UObject.

### Rule of Zero, Rule of Five, and ownership types

The preferred design is Rule of Zero: compose members that already manage themselves, and let the compiler generate destructor/copy/move operations. A class containing `std::vector`, `FString`, and `TUniquePtr` often needs no manual destructor. [SRC-CPP-003] [SRC-CPP-011]

If a class directly manages a resource through a raw handle and must declare one copy, move, or destructor operation, consider the full set deliberately:

1. Destructor
2. Copy constructor
3. Copy assignment
4. Move constructor
5. Move assignment

This is not a demand to hand-write all five. Some should be `= default`; some should be `= delete`; many classes should define none. The rule means the operations form one semantic contract. A user-declared destructor can suppress implicit move generation, silently turning expected moves into copies or making container use fail. [SRC-CPP-003] [SRC-CPP-011]

Examples:

- **Value owner:** deep copy plus cheap move may be meaningful.
- **Unique resource:** delete copy, support move.
- **Non-owning view:** copying the view is fine, but its documentation must state the observed lifetime.
- **Polymorphic base:** commonly use a virtual destructor and make copy policy explicit to avoid slicing.

### Copy and move semantics

Copy creates another object with the source's logical value. Move is an overload-selected opportunity to transfer representation from an expiring or explicitly relinquished source. `std::move` itself moves nothing: it casts an expression to an xvalue so move-aware overloads may be selected. The selected constructor or assignment performs the work. [SRC-CPP-004]

After moving from a standard-library object, it is normally valid but in an unspecified state. You may destroy it, assign a new value, or call operations whose preconditions you can establish; you may not assume it is empty unless that type's contract says so. [SRC-CPP-004]

```cpp
std::vector<FCommand> Commands = BuildCommands();
Submit(std::move(Commands)); // ownership/content transfer is now explicit

Commands.clear();            // valid: clear has no state precondition
// Do not assume Commands was empty before clear.
```

Common move mistakes:

- Moving from an object and later reading an assumed old/empty state.
- Applying `std::move` to a `const` object; most move constructors need non-const rvalues, so this often copies.
- Returning `std::move(Local)` and interfering with NRVO opportunities; return the local by value.
- Writing a move constructor that leaves the source violating its destructor's invariants.
- Forgetting `noexcept` where a truly non-throwing move can promise it.

Standard containers may prefer copying elements during reallocation if moving could throw and copying is available, preserving stronger exception guarantees. `std::move_if_noexcept` formalises this choice. Mark move operations `noexcept` only when the implementation genuinely cannot throw. [SRC-CPP-005]

### Return by value and copy elision

Modern C++ makes returning values normal. Named Return Value Optimisation may construct a named local directly in the caller's result storage; since C++17, same-type prvalue returns are constructed directly in the destination under guaranteed elision rules. Do not contort APIs into output parameters merely to avoid an imagined copy, and usually do not write `return std::move(Local);`. [SRC-CPP-010]

```cpp
std::vector<FSpawnPoint> BuildSpawnPoints()
{
    std::vector<FSpawnPoint> Result;
    // populate
    return Result; // NRVO candidate; otherwise implicit move is available
}
```

Measure actual copies before redesigning. Debug logging in copy/move constructors or compiler optimisation reports can verify behaviour; intuition based on pre-C++11 code is a poor profiler.

### Ownership vocabulary

| Relationship | Standard C++ expression | Meaning |
|---|---|---|
| Direct value | `T` member/local | Containing scope/object owns the value directly. |
| Unique dynamic owner | `std::unique_ptr<T>` / `TUniquePtr<T>` | Exactly one owner; movable, not copyable. |
| Shared dynamic owner | `std::shared_ptr<T>` / `TSharedPtr<T>` | Reference-counted co-ownership; last strong owner destroys target. |
| Weak observer of shared object | `std::weak_ptr<T>` / `TWeakPtr<T>` | Does not retain; lock/pin to obtain temporary strong access. |
| Borrow | `T&`, `const T&`, `T*`, span/view | No ownership; caller must guarantee observed lifetime. |
| UObject strong field | `UPROPERTY() TObjectPtr<T>` | GC-visible retaining edge in UObject domain. |
| UObject weak observer | `TWeakObjectPtr<T>` | Does not retain UObject; resolve/validate at use. |
| UObject soft locator | `TSoftObjectPtr<T>` | Path-based non-retaining reference supporting explicit loading. |

The Core Guidelines recommend unique ownership by default and shared ownership only when it is genuinely required. Unique ownership is simpler, cheaper, and makes destruction timing easier to reason about. [SRC-CPP-001]

### `unique_ptr`: transfer, not sharing

`std::unique_ptr` owns and disposes of an object when the pointer is destroyed, reset, or assigned another ownership. It is movable but not copyable. Prefer `make_unique`/`MakeUnique` for normal construction, and expose ownership transfer in function signatures. [SRC-CPP-006] [SRC-EPIC-019]

```cpp
std::unique_ptr<FJob> MakeJob();                   // returns ownership
void Enqueue(std::unique_ptr<FJob> Job);           // consumes ownership
void Inspect(const FJob& Job);                     // borrows, never owns
```

Calling `.release()` relinquishes ownership without deleting; it is therefore a sharp interop tool, not an ordinary getter. `.get()` observes while ownership remains with the smart pointer.

### `shared_ptr`: co-ownership with costs and ambiguity

`std::shared_ptr` maintains a control block with strong and weak counts, deleter/allocator state, and either the managed object or a pointer to it. Destruction occurs when the final strong owner releases. `make_shared` commonly combines object and control block allocation. [SRC-CPP-007]

Costs include reference-count operations, control-block storage/allocation, less predictable destruction site/thread, weaker ownership clarity, and the possibility of cycles. Thread-safe reference-count bookkeeping does **not** make the pointed-to object thread-safe, nor does it make concurrent mutation of the same non-atomic smart-pointer variable safe. [SRC-CPP-007]

Use shared ownership only when several independent parties truly extend lifetime. A manager with clear ownership and many raw/reference borrowers is often simpler than making every user a co-owner.

### `weak_ptr`: observe and break cycles

`std::weak_ptr` observes an object managed by `shared_ptr` without increasing its strong count. Call `lock()` to atomically obtain a temporary `shared_ptr` if the object is still alive. Weak edges break cycles where two shared owners would otherwise keep each other alive after external owners disappear. [SRC-CPP-008]

```cpp
if (std::shared_ptr<FSession> Session = WeakSession.lock())
{
    Session->Flush();
}
```

Do not check expiry and then access separately; lifetime can change between operations in concurrent code. Acquire the temporary strong owner and use that local.

### Standard versus Unreal pointer families

| Type | Domain | Ownership model | Thread note | UObject use |
|---|---|---|---|---|
| `std::unique_ptr` | Standard C++ | Exclusive, deterministic | Owner transfer itself needs normal synchronisation | Do not use as normal UObject owner |
| `TUniquePtr` | Unreal Core, non-UObject | Exclusive, deterministic | Same conceptual contract | Do not use as normal UObject owner |
| `std::shared_ptr` | Standard C++ | Reference-counted co-ownership | Control block supports operations on distinct pointer objects; pointee is not made safe | Wrong normal UObject model |
| `TSharedPtr` | Unreal Core, non-UObject | Non-intrusive reference counting | Thread safety depends on `ESPMode`; mode must match the use | Wrong normal UObject model |
| `std::weak_ptr` | Standard shared domain | Non-retaining observer | `lock()` creates strong access | Not for UObject GC observation |
| `TWeakPtr` | Unreal shared domain | Non-retaining observer | `Pin()` creates `TSharedPtr`; mode matters | Not `TWeakObjectPtr` |
| `TObjectPtr` / `TWeakObjectPtr` / `TSoftObjectPtr` | UObject domain | GC edge / weak observation / path locator | UObject thread access has separate constraints | Purpose-built UObject choices |

`TSharedPtr` and `TWeakPtr` mirror the ordinary reference-counted domain for non-UObjects; they are not aliases for `TObjectPtr` and `TWeakObjectPtr`. Epic's API describes `TSharedPtr` as a non-intrusive reference-counted authoritative pointer and `TWeakPtr` as its weak counterpart; `TUniquePtr` is non-copyable and movable. [SRC-EPIC-019] [SRC-EPIC-020] [SRC-EPIC-021]

### Borrowing and parameter contracts

Prefer signatures that reveal intent:

```cpp
void Render(const FMesh& Mesh);                 // required read-only borrow
void Update(FState& State);                     // required mutable borrow
void TryTarget(AActor* Candidate);              // nullable borrow, not ownership
void Consume(std::unique_ptr<FJob> Job);        // ownership transfer
void Share(std::shared_ptr<FService> Service);  // shared ownership by design
```

For small trivially copied types, pass by value. For larger input-only objects, `const T&` is a common default. For sink parameters that will be stored, pass-by-value then move can be effective when both lvalues and rvalues matter, but evaluate copies and API clarity. Avoid passing smart pointers by value when the function merely borrows the pointee; doing so performs ownership operations and falsely advertises lifetime participation.

### Iterator, pointer, and reference invalidation

Container operations can invalidate handles even though the container remains alive. For `std::vector`, reallocation invalidates all iterators, pointers, and references to elements. Without reallocation, insertion invalidates handles at or after the insertion point; erase invalidates the erased element and those after it. `reserve` invalidates only if capacity changes. [SRC-CPP-009]

```cpp
auto* First = &Items[0];
Items.push_back(NewItem); // may reallocate
Use(*First);              // possibly dangling: undefined behaviour
```

Debug workflow:

1. Identify the exact container mutation between handle creation and use.
2. Check capacity/reallocation and the container's operation-specific invalidation rules.
3. Prefer indices or stable IDs when they match semantics, but remember indices can change after erase/reorder.
4. Reacquire handles after mutation; do not “fix” the symptom by reserving an arbitrary giant capacity.
5. Enable iterator diagnostics/sanitisers in suitable non-shipping builds and minimise the reproduction.

The same principle applies to Unreal containers, but their exact invalidation rules must be checked independently rather than assumed identical to STL.

### Common bugs and misconceptions

1. **Non-null means alive.** Dangling pointers and references can retain an old address.
2. **RAII means smart-pointer memory.** RAII applies to every resource with acquire/release semantics.
3. **`std::move` performs a move.** It enables move overload resolution; the selected operation decides.
4. **Moved-from means empty.** Usually only valid-but-unspecified is guaranteed.
5. **`shared_ptr` is the safest default.** It obscures ownership, costs more, and permits cycles.
6. **Atomic reference count means thread-safe object.** It does not protect pointee state.
7. **Raw pointer means owner.** Prefer raw/reference types as borrows; make ownership explicit elsewhere.
8. **Returning by value is slow.** Elision and move semantics often make it the cleanest efficient API.
9. **Reserve fixes invalidation.** It addresses some reallocation cases, not erase/reorder or lifetime design.
10. **UE smart pointers own UObjects.** Non-UObject reference counting and UObject GC are separate domains.

### Debugging workflow: lifetime corruption

1. Classify the symptom: use-after-free, double delete, leak, stale iterator, or data race.
2. Name the intended owner and destruction event. If nobody can answer, the design is already suspect.
3. Trace copies, moves, resets, container reallocations, async captures, callbacks, and shutdown order.
4. Reduce to one owner and one observer; add complexity back only after the lifetime contract works.
5. Use AddressSanitizer for use-after-free/out-of-bounds, UndefinedBehaviourSanitizer where supported, debugger data breakpoints, and iterator diagnostics in development configurations.
6. For leaks, inspect ownership graphs and cycles rather than only allocation sites.
7. For cross-thread cases, establish synchronisation and execution ownership separately from memory ownership.

### Performance workflow

1. Measure allocations, copies, moves, cache misses, and contention in a representative workload.
2. Verify whether a type actually allocates; value semantics do not imply “stored entirely on stack”.
3. Prefer contiguous values for hot iteration when polymorphism/stability do not require indirection.
4. Remove unnecessary shared-count traffic from hot paths; do not replace it blindly with unsafe borrowing.
5. Mark moves `noexcept` when true and verify container behaviour with counters/traces.
6. Separate peak-frame latency from aggregate throughput and memory usage.

### Strong interview answer pattern

For a lifetime question:

1. Identify the lifetime domain and resource.
2. Name the owner and deterministic/non-deterministic destruction rule.
3. Distinguish owning from borrowing handles.
4. State copy and move policy.
5. Name invalidation/concurrency risks and how you would test them.

Example: “The job has one queue owner, so I would return and consume `unique_ptr`, borrowing `FJob&` during execution. Copy is disabled and move transfers ownership. I would not use shared ownership unless independent systems must extend lifetime. Any callback captures a weak/ID-based handle or is cancelled before queue destruction. I would verify shutdown and cancellation with AddressSanitizer and a stress test.”

### What a three-year engineer should know

You should explain object lifetime versus storage, implement ordinary RAII, prefer Rule of Zero, deliberately define copy/move policy, use unique/shared/weak ownership correctly, reason about moved-from state, understand value-return/elision, and diagnose iterator invalidation. You should also compare standard/UE reference counting with UObject GC without mixing them.

**Specialist depth:** allocator-aware types, placement lifetime rules, strict aliasing/provenance, exception guarantees in generic containers, intrusive reference counting, ABI/pImpl constraints, lock-free reclamation, and custom allocators.

### Hands-on verification

Complete Project 7B in [[ue_hands_on_projects]]. Instrument constructors, copies, moves, destructors, allocations, and invalidated handles. Predictions must be written before execution.

### Conflict and uncertainty notes

- “Rule of Five” is often taught as “always implement five functions”. The stronger guidance is Rule of Zero first; once one special operation is declared, consider and explicitly default/delete the coherent set. [SRC-CPP-003] [SRC-CPP-011]
- Standard and Unreal smart pointers have similar shapes but are not freely interchangeable and have different thread-mode/API details. UObject pointers form a third domain. [SRC-EPIC-009] [SRC-EPIC-019] [SRC-EPIC-020]
- Exception usage policy varies across game/UE build configurations. `noexcept` still affects type traits and container strategy even in codebases that avoid throwing; verify project/compiler settings before discussing runtime exception handling.

## Specialist deepening: generic programming, memory, concurrency, and binaries

The following topics move beyond ordinary ownership into the areas where a three-year engineer is expected to reason accurately even if they have not built a standard library or lock-free queue. The goal is not template wizardry. It is readable constraints, correct lifetime/concurrency, predictable data layout and evidence-led debugging.

## Templates: one algorithm, a family of concrete programs

A template is a compile-time recipe. Instantiating `TArray<FEnemy>` and `TArray<FProjectile>` produces type-specific code/data layouts; it is not runtime polymorphism. Templates enable zero-overhead generic code, but can increase compile time, binary size and diagnostic complexity.

### Function-template deduction

For a function call, the compiler deduces template arguments from parameter pattern `P` and argument type `A`, substitutes them, then performs overload resolution. Top-level references/cv and array/function decay depend on the parameter form. [SRC-CPP-015]

```cpp
template <typename T>
void Read(const T& Value);  // accepts lvalue/rvalue; T usually excludes top-level cv/ref

template <typename T>
void Store(T Value);        // copies/moves a value; arrays/functions decay

template <typename T>
void Forward(T&& Value);    // forwarding reference only when T is deduced here
```

Do not say every `T&&` is a forwarding reference. `Widget&&` is an rvalue reference. `T&&` in a class template member where `T` was fixed by the class can also be an ordinary rvalue reference. The special deduction rule applies to an rvalue reference to a cv-unqualified deduced template parameter.

### Reference collapsing and perfect forwarding

When a forwarding reference receives an lvalue, `T` deduces as an lvalue reference. Reference collapsing makes `T&&` an lvalue reference; for an rvalue it remains rvalue reference. `std::forward<T>` preserves that category.

```cpp
template <typename T, typename... TArgs>
T& Emplace(std::vector<T>& Items, TArgs&&... Args)
{
    return Items.emplace_back(std::forward<TArgs>(Args)...);
}
```

Forward each argument at most once. After forwarding, treat its source state as potentially consumed. Perfect forwarding is appropriate for adaptors/factories; ordinary domain APIs are often clearer with explicit overloads or values.

Common mistakes:

- using `std::move(Arg)` instead of `std::forward<T>(Arg)` in a forwarding wrapper, moving from lvalues;
- returning a forwarded reference to a temporary whose lifetime ends;
- forwarding the same argument to two consumers;
- adding a universal-reference overload that steals calls from copy/other overloads;
- accepting everything and producing unreadable substitution diagnostics.

### Specialisation and overloads

Prefer function overloads for behavioural alternatives. Function templates cannot be partially specialised; class templates can. Full specialisation is exact and must obey ODR/visibility rules. Traits commonly use a primary class template plus partial specialisations.

Specialise standard templates only where the standard explicitly permits it and the user-defined type meets requirements. Do not add overloads to namespace `std`.

### `if constexpr`

`if constexpr` selects a branch at compile time in a templated context; the discarded branch is not instantiated when dependent. It is excellent for small local variation after a readable requirement is established.

```cpp
template <typename T>
FString Describe(const T& Value)
{
    if constexpr (std::is_integral_v<T>)
    {
        return FString::FromInt(static_cast<int32>(Value));
    }
    else
    {
        return Value.ToString(); // only required for the selected non-integral T
    }
}
```

Do not build a 200-line type switch in one template. Split concepts/overloads/policies.

## SFINAE and concepts

SFINAE means substitution failure in an immediate template context removes a candidate instead of producing a hard compilation error. It powered `enable_if`, expression detection and `void_t` traits. [SRC-CPP-016]

```cpp
template <typename T, typename = std::void_t<decltype(std::declval<T>().Reset())>>
void TryReset(T& Value)
{
    Value.Reset();
}
```

C++20 concepts name requirements and make them part of the interface/overload ordering. [SRC-CPP-017]

```cpp
template <typename T>
concept Resettable = requires(T& Value)
{
    { Value.Reset() } -> std::same_as<void>;
};

template <Resettable T>
void TryReset(T& Value)
{
    Value.Reset();
}
```

Concepts improve diagnostics and intent; they do not validate runtime semantics, complexity or thread safety. A `Sortable` requirement can prove expressions/types exist, not that a comparator is a strict weak order.

Use the project's actual C++ standard and Unreal branch. Epic's maintained coding standard currently states a C++20 baseline, but the curriculum targets UE5.3–UE5.6 and toolchain/project settings must be verified rather than inferred from the maintained page. [SRC-CPP-025]

### Strong interview distinction

> SFINAE is a candidate-removal mechanism during substitution; concepts express named compile-time requirements and participate directly in constraint ordering. I use concepts for new C++20 interfaces, detection/SFINAE when supporting older/tool-specific code, and keep requirements small and semantic.

## Lambdas, captures, and callable erasure

A lambda expression creates a unique closure type with captured state as data members. Capturing does not extend arbitrary referent lifetime.

### Capture rules

- `[x]` copies `x` into the closure when the lambda is created;
- `[&x]` stores reference-like access; caller must outlive every invocation;
- `[this]` captures the pointer, not a copy/lifetime guarantee of the object;
- `[*this]` copies the object where supported, which can be expensive or semantically wrong;
- init-capture (`[Owner = MoveTemp(Owner)]`) can transfer ownership into the closure;
- default capture `[&]`/`[=]` hides exactly what asynchronous code retains.

For delayed/tasks/delegates, capture explicit values, stable IDs or weak handles and revalidate at execution. Never capture a stack reference into work that can outlive the scope.

```cpp
TWeakObjectPtr<AActor> WeakActor = Actor;
LaunchWork([WeakActor, RequestId]
{
    // Compute only safe copied data here; UObject use still needs Game Thread.
    AsyncTask(ENamedThreads::GameThread, [WeakActor, RequestId]
    {
        if (AActor* ActorNow = WeakActor.Get())
        {
            ActorNow->ApplyResult(RequestId);
        }
    });
});
```

### `std::function` and type erasure

`std::function<R(Args...)>` stores any compatible copy-constructible callable behind a uniform runtime interface. [SRC-CPP-018] This enables heterogenous callbacks without exposing closure types, but may add indirect calls, storage and allocation/copy overhead. Empty invocation throws `std::bad_function_call`, which is especially relevant in exception-disabled policy—check before call or use project callback types.

Use a template parameter for compile-time/inlinable callable when the callable type can be part of the interface; use function pointer for stateless C-style calls; use an owning type-erased wrapper when runtime storage/substitution is required; use Unreal delegates when engine binding, weak UObject/dynamic reflection or multicast semantics require them. These are semantic choices, not merely syntax.

Beware a `std::function` returning a reference when the target returns a temporary; standard-version rules differ and older forms can dangle. [SRC-CPP-018]

## Object representation, alignment, and cache behaviour

Every object has size, alignment, storage duration and lifetime. Its object representation includes value bytes plus possible padding. [SRC-CPP-019]

```cpp
struct FExample
{
    uint8  Flags;   // 1 byte
    double Time;    // likely requires 8-byte alignment, causing padding before it
    uint16 Count;
};

static_assert(alignof(FExample) >= alignof(double));
```

Member order can change size because each member and complete object must satisfy alignment. Inspect `sizeof`, `alignof`, compiler layout reports and the target ABI; never serialise/network/hash raw struct bytes unless padding, endianness, layout/version and initialisation are explicitly controlled.

### Polymorphic layout awareness

A polymorphic class commonly contains implementation ABI state such as a virtual-table pointer, but the C++ standard does not mandate one exact representation. Virtual calls can inhibit inlining and add indirection; the larger performance issue is often pointer-chasing scattered objects rather than one indirect branch.

Avoid `memcpy`/`memset`/raw relocation of non-trivially copyable types. Unreal containers/types may have engine-specific relocation traits; use supported APIs/traits, not assumptions from a visually simple class.

### Array of structs versus structure of arrays

```text
AoS: [Pos Vel Health][Pos Vel Health][Pos Vel Health]
SoA: [Pos Pos Pos...] [Vel Vel Vel...] [Health Health Health...]
```

AoS is cohesive for operations needing the whole entity. SoA improves bandwidth/SIMD when a hot loop reads only one/few fields over many elements. Hybrid/chunked layouts often balance access patterns. Transforming layout without workload evidence can make other systems slower and authoring harder.

### Cache locality

Contiguous iteration helps because hardware fetches cache lines, not C++ objects semantically. Cost depends on working set, stride, branch predictability, prefetch, write traffic and contention. Measure cache misses/bandwidth and time, not just container type.

False sharing occurs when independent frequently-written values used by different cores share a cache line, forcing coherence traffic. C++17 exposes implementation constants intended to help separate or group data, but availability/value and Unreal toolchain use must be checked. [SRC-CPP-023]

Padding atomics to cache lines can waste memory and worsen locality when contention is absent. Benchmark realistic core/thread placement.

## Allocators, arenas, and pools

Allocation strategy solves lifetime, size-distribution, locality, contention and observability problems—not “new is always slow”.

### Region/arena allocation

An arena allocates many objects cheaply and releases them together. `std::pmr::monotonic_buffer_resource` releases allocations when the resource is released/destroyed and individual deallocation is a no-op; it is not thread-safe. [SRC-CPP-024]

Good fits: frame scratch, parser/import job, temporary graph build, bounded request. Bad fits: individually long-lived objects, unknown unbounded growth, references escaping the arena, destructors/resources requiring individual semantic cleanup.

Raw arena release does not automatically call destructors for arbitrary manually placed objects. Container/resource ownership must ensure required destruction before memory release.

### Pools

A pool reuses fixed/similar allocation slots. It can reduce allocation churn and improve locality, but introduces generation/stale-handle, reset, fragmentation and retained-memory risks. Pooling a UObject/Actor/Component also requires complete engine-state reset; ordinary C++ slot reuse is not sufficient.

### PMR awareness

`std::pmr::polymorphic_allocator` delegates to a runtime `memory_resource`; nested PMR containers can share a resource. Allocator propagation/swap/move rules matter, and unequal resources can invalidate naive assumptions. [SRC-CPP-024]

Unreal uses its own allocators/containers extensively. Use standard PMR only where module/toolchain/project policy supports it and interop is clear; do not mix allocation/deallocation families.

### Allocator profiling questions

- allocation count/bytes/size distribution and call sites;
- peak live versus cumulative bytes;
- lifetime histogram and thread contention;
- fragmentation and retained pool/arena capacity;
- constructor/destructor/reset cost;
- cache locality of the complete consumer;
- failure/fallback when a bounded arena exhausts.

## C++ memory model and data races

A data race occurs when two threads perform conflicting accesses to the same memory location, at least one is a write, and the accesses are not ordered by happens-before (with no applicable atomic treatment). A C++ data race is undefined behaviour—not merely a stale value.

`volatile` does not make shared data atomic or provide inter-thread ordering. It is for specialised observable access (for example, some memory-mapped I/O), not a mutex substitute. [SRC-CPP-020]

### Mutex first

Use a mutex to protect an invariant spanning multiple fields/steps. Manage it with RAII (`lock_guard`, `unique_lock`, `scoped_lock`) so every path unlocks. `scoped_lock` can lock multiple mutexes with deadlock avoidance; an unnamed temporary unlocks immediately, a subtle bug. [SRC-CPP-021]

Keep critical sections small but correct. Moving work outside requires copying/snapshotting data while locked and proving the invariant still holds.

### Condition variables

A condition variable coordinates waiting for a predicate. Wait with a `unique_lock` and predicate loop because wake-ups may be spurious and notifications are not queued state. The shared predicate is changed under the associated mutex. [SRC-CPP-022]

```cpp
std::unique_lock Lock(Mutex);
Ready.wait(Lock, [&] { return bStopping || !Queue.empty(); });
if (bStopping && Queue.empty())
{
    return;
}
FJob Job = std::move(Queue.front());
Queue.pop();
Lock.unlock();
Execute(Job);
```

Do not wait without a predicate, hold the lock during expensive execution, or destroy the condition/mutex while waiters exist.

## Atomics and memory order

An atomic makes operations on that atomic object race-free and provides specified ordering; it does not make a multi-field invariant atomic.

- `relaxed`: atomicity/modification order only; good for independent counters where no other data is published;
- `release` store + `acquire` load that observes it: publishes preceding writes to the acquiring thread;
- `acq_rel`: read-modify-write both consumes and publishes;
- `seq_cst`: strongest default common global ordering model, simplest to reason about but potentially costlier;
- `consume`: avoid in practical portable design; implementations historically treat it conservatively.

Memory order controls how regular accesses are ordered around atomics. [SRC-CPP-020]

```cpp
// Producer
Payload = BuildPayload(); // ordinary writes
bReady.store(true, std::memory_order_release);

// Consumer
if (bReady.load(std::memory_order_acquire))
{
    Consume(Payload); // publication makes preceding writes visible
}
```

This only works with one clear publication protocol and no concurrent mutation after publish. Start with mutexes or sequential consistency. Weaken ordering only with a proof, litmus/stress tests, cross-architecture testing and profiler evidence.

### Compare-exchange loop awareness

CAS can fail because the observed value changed; weak CAS may also fail spuriously and belongs in a loop. The `expected` argument is updated on failure. Lock-free does not mean wait-free, faster, fair or safe reclamation. ABA and object reclamation make pointer algorithms specialist territory.

## Tasks and job-system design

Threads are expensive ownership/scheduling units; games usually submit many small jobs to a worker pool. A task graph represents dependencies as a DAG so workers execute ready tasks rather than blocking for prerequisites.

Epic's maintained Tasks System documentation describes callable launch, prerequisites, nested tasks, pipes/events and Insights tracing; Tasks and legacy TaskGraph share scheduler/backend. Maintained docs also note UE5.5 changes such as busy-wait deprecation, so exact UE5.3–UE5.6 semantics need target checks. [SRC-CPP-026]

### Parallel work contract

For every task define:

- immutable/copied inputs or partitioned exclusive writes;
- output ownership and merge/reduction point;
- prerequisites/completion dependencies;
- cancellation/owner/World generation;
- thread-affine operations moved back to Game Thread;
- task granularity large enough to amortise scheduling;
- no worker blocking on work that requires the blocked execution resource.

### Fork–join example

```text
snapshot on Game Thread
       ↓
[chunk 0] [chunk 1] [chunk 2] [chunk 3]
       \      |       |      /
         deterministic merge
                 ↓
       apply on Game Thread
```

Avoid many workers pushing into one locked container. Give each chunk local output, then merge. Preserve deterministic order if product/network/replay requires it.

### Task lifetime trap

Releasing a task handle need not cancel execution; the scheduler may retain its own reference until completion. [SRC-CPP-026] Captures must outlive actual execution, not merely the local handle. Fire-and-forget still needs shutdown and owner-invalidated behaviour.

## ODR, linkage, symbols, and static initialisation

### Declaration, definition, and ODR

A declaration introduces an entity; a definition supplies it. The One Definition Rule permits one program definition for odr-used non-inline functions/variables, with controlled exceptions for inline/templates across translation units when definitions satisfy equivalence requirements. [SRC-CPP-027]

Common failures:

- non-inline function/global defined in a header → duplicate symbol;
- declaration exists but implementation/module not linked/exported → unresolved external;
- signature/namespace/cv/calling convention mismatch → different mangled symbol;
- template definition hidden in `.cpp` without explicit instantiation → unresolved specialisation;
- different macros/build flags produce non-equivalent inline/template class definitions → ODR undefined behaviour;
- missing module API export across DLL boundary.

### Internal/external linkage

Namespace-scope names may have internal/external/module/no linkage based on declaration. [SRC-CPP-028] Use unnamed namespace or suitable `static` for `.cpp` implementation details, but avoid header `static` state that unintentionally creates one copy per translation unit.

`inline` primarily permits ODR-compliant multiple definitions; it is not a command that the optimiser must inline. `constexpr`/templates often imply header definitions because compile-time/instantiation visibility is required.

### Static initialisation order

Non-local dynamic initialisation order across translation units is not safely dependency-ordered. One global constructor using another TU's global can observe uninitialised state—the static initialisation order fiasco. [SRC-CPP-029]

Prefer constant initialisation, function-local statics for on-first-use where lifetime/shutdown is acceptable, or explicit engine/module startup/shutdown ownership. Function-local static initialisation is thread-safe since C++11, but destruction order and hot reload/module unload still deserve care.

Avoid substantial UObject/engine work in C++ global constructors: engine subsystems/config/allocators/logging may not be ready, and shutdown order is opaque.

### Link diagnosis workflow

1. Classify compile, unresolved external, duplicate symbol, load-time missing import or runtime ABI failure.
2. Read the full demangled/mangled symbol: namespace, class, parameter, cv/ref and template args.
3. Find declaration and exact definition; compare generated/export macros.
4. Verify owning module/Build.cs dependency and public/private/API boundary.
5. For templates, ensure definition visible or explicit instantiation exported.
6. Inspect binary symbols/imports/map file when source looks correct.
7. Reproduce without unity builds; unity can hide missing includes/internal collisions.

## Sanitizers and correctness tools

Sanitizers instrument compiled code; support and integration vary by compiler/platform/Unreal configuration.

- **AddressSanitizer (ASan):** use-after-free, buffer errors and related memory bugs; high overhead and requires instrumented/link runtime. [SRC-CPP-030]
- **ThreadSanitizer (TSan):** data races; large memory/runtime overhead and platform/link restrictions. [SRC-CPP-031]
- **UndefinedBehaviorSanitizer (UBSan):** selectable checks such as signed overflow, invalid shift, misalignment, null, bounds and vptr misuse. [SRC-CPP-032]

They are complementary. ASan does not prove race freedom; TSan does not catch every logical deadlock; UBSan does not define intentional UB. Partial instrumentation, custom allocators, inline assembly and unsupported platforms can reduce coverage.

Use:

- deterministic minimal reproducer and symbols/frame pointers;
- same failing input/seed in instrumented build;
- first reported root cause before downstream crashes;
- narrow justified suppressions with owner/expiry, not blanket ignore lists;
- CI smoke/stress lanes where supported;
- target hardware/testing in addition to desktop sanitizers.

## Unreal policy: standard guarantees versus project rules

Epic's maintained coding standard states C++20 default/minimum at the current documentation point and documents Unreal-specific style/portability choices such as constrained `auto` usage and `MoveTemp`. [SRC-CPP-025] The target UE5.3–UE5.6 branch may differ in compiler/library feature availability.

- Use engine types/containers/delegates/task APIs where engine integration/reflection/allocator/platform tooling requires them.
- Use standard C++ where supported and semantically appropriate; do not assume every STL facility is banned or ideal.
- Unreal commonly builds with exceptions and native RTTI disabled by default for engine modules, but exact target/module/third-party settings must be checked. Do not throw across module/toolchain boundaries or use `dynamic_cast` without policy proof.
- `noexcept`, destructors and RAII remain meaningful even when exceptions are disabled.
- Prefer Unreal reflection/cast mechanisms for UObjects and ordinary C++ virtual/type techniques for non-UObject domains according to architecture.

## Specialist common misconceptions

| Misconception | Better model |
|---|---|
| “`T&&` means rvalue.” | It is a forwarding reference only in a specific deduced context. |
| “Concepts replace templates/SFINAE.” | Concepts constrain templates; SFINAE remains a mechanism/compatibility tool. |
| “Lambda capture by reference is cheaper and safe until callback returns.” | Safety depends on invocation lifetime, often far beyond creation scope. |
| “`std::function` is just a function pointer.” | It owns type-erased callable state and can allocate/copy/indirect. |
| “Reorder members smallest to largest.” | Order from access/ABI/semantics and measured size/cache, not one mnemonic. |
| “Arena means no destructors.” | Storage release is bulk; semantic resource destruction may still be required. |
| “Atomic means thread-safe.” | It protects one atomic operation/object, not an invariant or lifetime. |
| “Relaxed is faster.” | It supplies weaker guarantees; only a proven protocol makes it correct. |
| “Lock-free is faster and cannot block.” | Contention/retry/reclamation may perform worse and lock-free is not wait-free. |
| “Task handle destruction cancels the task.” | Scheduler execution/lifetime is separate; captures must survive actual completion. |
| “Inline means compiler inlines it.” | It is chiefly an ODR property; optimisation is separate. |
| “Sanitizer-clean means correct.” | Coverage is configuration/workload/platform limited and logical bugs remain. |

## Specialist interview answer patterns

### Perfect forwarding

> A forwarding reference is `T&&` where `T` is deduced and cv-unqualified. Lvalues deduce `T` as an lvalue reference and reference collapsing preserves lvalueness; rvalues remain rvalues. `std::forward<T>` passes that original category to one downstream consumer. I use it in generic adaptors/factories, not as decoration on domain APIs, and avoid double forwarding or returning references to temporaries.

### Data race diagnosis

> First I identify the exact memory location, conflicting accesses and intended happens-before edge. I reproduce with TSan where supported and log task/thread/owner generations. Then I protect the whole invariant with one mutex/snapshot/ownership partition before considering atomics. For publication I can use release/acquire, but only after writing the protocol and proving no later unsynchronised mutation.

### Cache/layout optimisation

> I capture time plus cache/bandwidth/allocation data for the actual traversal. Then I inspect size/alignment/padding and hot-field access, comparing AoS, SoA or chunked layouts with equivalent semantics. I include writer contention/false sharing and all consumers. I do not serialise raw padded bytes or trade maintainability for an unmeasured size reduction.

### Unresolved external

> I read the complete missing symbol and compare declaration/definition namespace, signature, cv/ref/template arguments and export macro. Then I verify owning module linkage and Build.cs dependency. Template definitions must be visible or explicitly instantiated/exported. I test non-unity and inspect binary symbols/imports rather than randomly adding includes or modules.

## Specialist hands-on verification

Extend Project 7B with a C++ Systems Lab:

1. write a forwarding factory and tests for lvalue/rvalue/const/move-only inputs, including a deliberate double-forward bug;
2. implement one detection/SFINAE API and its C++20 concept equivalent; compare diagnostics;
3. benchmark direct templated callable, function pointer, `std::function` and an Unreal delegate without claiming universal results;
4. print/verify size/alignment/member offsets for layouts, then compare AoS/SoA/chunked hot loops;
5. build bounded monotonic arena and pool experiments with escaped-handle/destructor/reset failures;
6. reproduce a race, deadlock, missed wake-up and false-sharing case; fix with mutex/predicate/partitioning before a justified atomic variant;
7. implement release/acquire publication and document its happens-before proof;
8. build a fork–join task pipeline with copied snapshot, per-chunk output, deterministic merge and owner-generation cancellation;
9. create unresolved, duplicate, hidden-template and static-initialisation failures across two modules/TUs;
10. run ASan/TSan/UBSan where supported and record configuration, overhead, first root report and blind spots;
11. deliver compiler/engine standard matrix, benchmarks with variance, lifetime/thread diagrams and a “simplest correct design” decision log.

## Specialist source map

- deduction/forwarding/SFINAE/concepts: [SRC-CPP-015] [SRC-CPP-016] [SRC-CPP-017]
- lambdas/type-erased callable: [SRC-CPP-018]
- objects/layout/storage/linkage/ODR/static initialisation: [SRC-CPP-019] [SRC-CPP-027] [SRC-CPP-028] [SRC-CPP-029]
- memory model/locks/condition variables/false sharing: [SRC-CPP-020] [SRC-CPP-021] [SRC-CPP-022] [SRC-CPP-023]
- allocators/PMR: [SRC-CPP-024]
- Unreal C++/tasks policy: [SRC-CPP-025] [SRC-CPP-026]
- sanitizers: [SRC-CPP-030] [SRC-CPP-031] [SRC-CPP-032]

## Cluster 3 - ABI, Allocation Families, and OS Boundary Awareness

The systems chapter owns the full OS/process/security treatment, but a C++ interview answer should connect language rules to the generated program.

### Pointer versus reference at implementation level

A pointer is an object value that can hold an address-like value, be null, be reseated and be stored independently. A reference is a language-level alias that is initialised to refer to an existing object or function; references themselves are not objects in the standard model. [SRC-CPP-033]

In many ABIs a by-reference parameter is passed as an address, so unoptimised assembly for `void f(int&)` can look similar to `void f(int*)`. That is an implementation strategy, not the whole contract. Optimised code may remove the indirection, keep the value in a register, or materialise storage only when the observable semantics require it. The safe interview phrasing is: "references are often implemented with addresses when they need representation, but their C++ semantics are aliasing and binding rules, not pointer syntax." [SRC-CPP-033] [SRC-CPP-019]

### OOP in C++ and UE C++

C++ dynamic polymorphism is specified through virtual functions and final overriders, commonly implemented by compiler ABIs with hidden vptr/vtable structures. The standard does not require one exact vtable layout. [SRC-CPP-037]

Unreal adds reflection metadata and generated code around `UCLASS`, `USTRUCT`, `UFUNCTION` and related macros. Native virtual dispatch, UObject reflection, Blueprint events, `Cast<>`, serialisation and GC are related parts of UE C++ programming but not the same mechanism. Use native virtuals for ordinary C++ extension points, reflected interfaces/events where Blueprint/editor/serialisation integration matters, and composition where object lifetime or responsibilities do not fit inheritance. [SRC-EPIC-001] [SRC-EPIC-002] [SRC-EPIC-009]

### `new`/`delete` versus `malloc`/`free`

`new` obtains storage and constructs an object; `delete` destroys the object and releases matching storage. `malloc` returns raw storage and `free` releases it; constructors and destructors are not involved. Placement `new` constructs an object in already-provided storage, so explicit destruction and storage-owner policy become the programmer's responsibility. [SRC-CPP-034] [SRC-CPP-035] [SRC-CPP-036]

In Unreal, do not use ordinary `new` for UObjects or Actors. `NewObject`, `CreateDefaultSubobject` and `SpawnActor` connect the object to Unreal's class, Outer, flags, construction, reflection, GC and Actor lifecycle systems. [SRC-EPIC-003] [SRC-EPIC-004]

### Static, inline and macros in build reality

`static` can mean static storage duration, internal linkage, class-level data/function membership or block-local persistent state depending on context. Scope, lifetime, linkage and storage duration are different axes; a strong answer names the axis being discussed. [SRC-CPP-028]

`inline` is chiefly an ODR/linkage facility that permits matching definitions across translation units; it is not an optimisation command. `#define` is preprocessor token substitution before C++ type checking and scope rules. Macro-dependent inline/template definitions can therefore create configuration-sensitive ODR bugs across modules or unity/non-unity builds. [SRC-CPP-027] [SRC-CPP-038] [SRC-CPP-039]

### Low-level allocation awareness

OS APIs such as `VirtualAlloc`, `mmap` and `brk` operate on virtual address ranges/pages. Allocators sit above them and carve those ranges into program allocations; containers and objects sit above allocators and manage C++ lifetimes. Do not blur page mapping, heap allocation and object lifetime into one "heap memory" bucket. [SRC-SYS-001] [SRC-SYS-005] [SRC-SYS-006]

See [[systems_programming_for_game_interviews]] for the full process/thread/IPC/shared-memory/page-fault treatment.
