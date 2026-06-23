# C++ Interview Question Bank

See also: [[cpp_for_unreal_interviews]], [[ue_cpp_idioms]], [[ue_flashcards]], [[ue_hands_on_projects]].

## Lifetime, Value Semantics, and Ownership

### Question: What is the difference between storage duration and object lifetime?

- **Category:** Standard C++ / Lifetime
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Storage provides suitably sized/aligned bytes; object lifetime is the interval during which an object of a type exists in that storage and may be accessed as that object.
- **Strong 3-Year-Engineer Answer:** An address can remain available after the object's lifetime ends, so a non-null pointer can dangle. Lifetime normally begins after suitable storage is obtained and initialisation completes, and ends when destruction starts or storage is released/reused under the relevant rules. This distinction matters in containers, arenas, placement construction, async callbacks, and references to locals. I debug by tracing the owner/destruction event, not merely checking for null.
- **Common Weak Answer:** “Stack objects die at the brace and heap objects live until delete.”
- **Follow-up Questions:** Can storage outlive an object? What is a dangling reference? How does this differ for UObjects?
- **Hands-on Verification Task:** Use AddressSanitizer to catch a pointer to a destroyed local and a vector element invalidated by reallocation.
- **Sources:** [SRC-CPP-002]
- **Version Notes:** Core C++ concept; examples target C++17+.

### Question: What is RAII, and why is it broader than smart pointers?

- **Category:** Standard C++ / Resource Management
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** RAII binds acquisition and release of any resource to an object's construction and destruction.
- **Strong 3-Year-Engineer Answer:** A completed constructor establishes an invariant that the object owns a resource; the destructor releases it on every scope-exit path, including early returns and stack unwinding where enabled. The resource can be memory, a lock, file, GPU handle, delegate registration, or transaction. I prefer composing existing RAII members so the containing type follows Rule of Zero. In Unreal code I still use RAII for ordinary resources, while UObject lifetime remains GC-managed.
- **Common Weak Answer:** “RAII means use `unique_ptr` instead of `new`.”
- **Follow-up Questions:** How would you make a registration RAII-safe? What if construction fails? RAII versus UObject GC?
- **Hands-on Verification Task:** Implement a scope-bound registration token and prove cleanup on every return path.
- **Sources:** [SRC-CPP-001]
- **Version Notes:** Stable modern C++ guidance.

### Question: Explain the Rule of Zero and Rule of Five.

- **Category:** Standard C++ / Class Design
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Prefer members that manage themselves so no special operations are needed; if one destruction/copy/move operation is declared, deliberately define, default, or delete the coherent set.
- **Strong 3-Year-Engineer Answer:** Rule of Zero is the design goal because standard containers and ownership wrappers already encode correct destruction/copy/move. If I directly manage a resource, I define its copy policy—deep copy or deleted—and move policy, plus destruction. Declaring a destructor or copy operation can suppress implicit moves, so I make every special member intentional. “Rule of Five” does not mean hand-writing boilerplate five times.
- **Common Weak Answer:** “If you write a destructor, write the other four.”
- **Follow-up Questions:** Why can a destructor disable implicit move? When should copy be deleted? What about polymorphic bases?
- **Hands-on Verification Task:** Add a destructor to a movable instrumented type and observe which operations the compiler generates.
- **Sources:** [SRC-CPP-003], [SRC-CPP-011]
- **Version Notes:** C++11+; Rule of Zero preferred.

### Question: What does `std::move` actually do?

- **Category:** Standard C++ / Move Semantics
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** It casts its argument to an xvalue, allowing move-aware overloads to be selected; it does not transfer resources itself.
- **Strong 3-Year-Engineer Answer:** The selected move constructor or assignment determines the transfer. After moving, standard-library objects are generally valid but unspecified, so I destroy, reassign, or use only operations whose preconditions I establish. Moving from `const` often copies because destructive moves normally require non-const access. I avoid `return std::move(Local)` because normal return semantics support NRVO or implicit move.
- **Common Weak Answer:** “It moves memory and clears the source.”
- **Follow-up Questions:** What is an xvalue? Why can moving const copy? What state is the source in?
- **Hands-on Verification Task:** Instrument copy/move overloads for const and non-const values and record which overload wins.
- **Sources:** [SRC-CPP-004]
- **Version Notes:** C++11+.

### Question: Why should a move constructor often be `noexcept`?

- **Category:** Standard C++ / Move Semantics
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Containers can prefer a non-throwing move during relocation while preserving strong exception guarantees; a potentially throwing move may cause copying instead.
- **Strong 3-Year-Engineer Answer:** During vector reallocation, the old range must remain recoverable if element construction fails. `move_if_noexcept` selects move when it is non-throwing or copy is unavailable, otherwise copy. I mark a move `noexcept` only if every operation it performs supports that promise; a false promise terminates on throw. This matters even in game code that generally avoids exceptions because traits and library strategy still observe the specification.
- **Common Weak Answer:** “`noexcept` makes move faster.”
- **Follow-up Questions:** What if the type is move-only and move can throw? How do member moves affect the implicit exception specification?
- **Hands-on Verification Task:** Reallocate vectors of instrumented throwing-move and non-throwing-move types.
- **Sources:** [SRC-CPP-005]
- **Version Notes:** C++11+.

### Question: When should ownership be `unique_ptr` rather than `shared_ptr`?

- **Category:** Standard C++ / Ownership
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Use unique ownership by default; use shared ownership only when multiple independent parties must extend lifetime.
- **Strong 3-Year-Engineer Answer:** `unique_ptr` makes the owner and destruction point clear, avoids reference-count traffic, and transfers explicitly through move. Borrowers take references or pointers without ownership. I choose `shared_ptr` only when no single owner can express the true lifetime, not to avoid deciding. In a hot game system, clear manager ownership plus borrowed access often improves both reasoning and locality.
- **Common Weak Answer:** “Use shared_ptr whenever more than one object accesses it.”
- **Follow-up Questions:** How do you pass ownership into a function? How do borrowers avoid dangling? UE equivalent?
- **Hands-on Verification Task:** Model a job queue first with shared ownership, then with unique queue ownership and borrowers; compare operations and shutdown reasoning.
- **Sources:** [SRC-CPP-001], [SRC-CPP-006]
- **Version Notes:** C++11+.

### Question: How do `shared_ptr` cycles happen, and how do you fix them?

- **Category:** Standard C++ / Ownership
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** A cycle of strong shared owners keeps every strong count above zero after external owners disappear; make non-owning back/observer edges weak or redesign ownership.
- **Strong 3-Year-Engineer Answer:** Reference counting cannot infer that an isolated cycle is unreachable. I draw the ownership graph and identify which edge is observational—often child-to-parent, listener-to-subject, or cache-to-entry—then use `weak_ptr` there. At use time I call `lock()` and keep the resulting local shared owner for the complete operation. Sometimes the better repair is one explicit manager owner rather than a web of shared ownership.
- **Common Weak Answer:** “Call reset on both pointers.”
- **Follow-up Questions:** Does garbage collection have the same cycle problem? Why lock instead of expired-then-access?
- **Hands-on Verification Task:** Create a two-node cycle, observe missing destructors, then break the back edge with `weak_ptr`.
- **Sources:** [SRC-CPP-008]
- **Version Notes:** C++11+.

### Question: Is `shared_ptr` thread-safe?

- **Category:** Standard C++ / Concurrency / Ownership
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** Its control block supports concurrent operations through distinct smart-pointer objects, but the pointee and the same mutable smart-pointer object are not thereby safe.
- **Strong 3-Year-Engineer Answer:** Copies sharing a control block can update reference counts safely under the library guarantee. That does not serialise access to the managed object. Concurrent mutation of one `shared_ptr` variable needs atomic smart-pointer facilities or external synchronisation. I treat memory ownership and execution/data-race ownership as separate designs.
- **Common Weak Answer:** “Yes, shared_ptr uses atomics.”
- **Follow-up Questions:** What does `atomic<shared_ptr>` solve? Can destruction run on an unexpected thread? How does `TSharedPtr` differ?
- **Hands-on Verification Task:** Build a test with safe distinct pointer copies and an intentionally raced pointee under ThreadSanitizer where available.
- **Sources:** [SRC-CPP-007]
- **Version Notes:** C++11+; `atomic<shared_ptr>` specialisation is C++20.

### Question: When are vector iterators, pointers, and references invalidated?

- **Category:** Standard C++ / Containers
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Reallocation invalidates all element handles; without reallocation, insertions invalidate at/after the insertion point and erasure invalidates the erased element and following elements.
- **Strong 3-Year-Engineer Answer:** I reason per operation, not from whether the container variable still exists. `push_back` may reallocate when size exceeds capacity. `reserve` can prevent specific reallocations but does not protect against erase/reorder and can itself invalidate if capacity changes. For long-lived references I choose a stable ownership/ID design, reacquire handles after mutation, and use sanitiser or debug-iterator builds to catch mistakes.
- **Common Weak Answer:** “Vector pointers are safe if you reserve.”
- **Follow-up Questions:** What does erase invalidate? Are indices stable? Do TArray rules exactly match vector?
- **Hands-on Verification Task:** Keep handles across push, insert, erase, and reserve; predict and verify each case.
- **Sources:** [SRC-CPP-009]
- **Version Notes:** Standard-library rules; check UE containers separately.

### Question: Is returning a large object by value inefficient?

- **Category:** Standard C++ / Value Semantics
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Not inherently; NRVO, guaranteed C++17 prvalue construction, and move semantics make value return a normal efficient API.
- **Strong 3-Year-Engineer Answer:** A same-type prvalue can be constructed directly in the destination in C++17, and a named local is an NRVO candidate with implicit move fallback. I write the clear value-return API, avoid `return std::move(Local)`, and measure if the type or compiler situation is exceptional. Output parameters often make ownership and partial failure less clear without delivering the imagined performance benefit.
- **Common Weak Answer:** “Use an output reference to avoid a copy.”
- **Follow-up Questions:** NRVO versus guaranteed elision? What if move is deleted? Why can `std::move` hurt NRVO?
- **Hands-on Verification Task:** Instrument a factory return under optimised and non-optimised builds.
- **Sources:** [SRC-CPP-010]
- **Version Notes:** Guaranteed prvalue construction is C++17; NRVO remains permitted rather than mandatory.

### Question: Raw pointer or reference—does either imply ownership?

- **Category:** Standard C++ / API Design
- **Priority:** P0
- **Expected Depth:** D3
- **Short Answer:** Neither should imply ownership by default; references express required borrowing, pointers commonly express nullable borrowing, and ownership should be explicit.
- **Strong 3-Year-Engineer Answer:** I use `const T&` for required read-only access, `T&` for required mutation, and `T*` for optional/nullable observation. Transfer uses `unique_ptr` by value; co-ownership uses `shared_ptr` only when intended. Documentation must state lifetime constraints because reference syntax prevents null but not dangling. In UObject code, raw locals can still be borrows while persistent fields use UObject-aware wrappers.
- **Common Weak Answer:** “References are safe pointers.”
- **Follow-up Questions:** Can references dangle? When should a smart pointer parameter be by value? Span versus pointer/length?
- **Hands-on Verification Task:** Redesign an API that passes `shared_ptr` everywhere into owner, borrower, and optional-borrow contracts.
- **Sources:** [SRC-CPP-001], [SRC-CPP-002]
- **Version Notes:** Stable modern C++ guidance.

### Question: Compare `std::shared_ptr`, `TSharedPtr`, and UObject pointers.

- **Category:** Standard C++ / UE C++ / Lifetime
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Standard and Unreal shared pointers are reference-counted ownership systems for ordinary objects; UObject wrappers participate in Unreal's GC/asset-reference domain.
- **Strong 3-Year-Engineer Answer:** `std::shared_ptr` and `TSharedPtr` both represent non-intrusive co-ownership, with different APIs and Unreal's explicit `ESPMode` thread mode. `std::weak_ptr`/`TWeakPtr` observe those control blocks. `UPROPERTY() TObjectPtr` is instead a GC-visible strong UObject edge; `TWeakObjectPtr` observes a UObject; `TSoftObjectPtr` stores a path for deferred loading. I never choose between these by naming similarity—I identify the lifetime domain first.
- **Common Weak Answer:** “TSharedPtr is Unreal's faster shared_ptr and TObjectPtr is its replacement.”
- **Follow-up Questions:** Why not TSharedPtr for UObjects? TWeakPtr versus TWeakObjectPtr? What does ESPMode mean?
- **Hands-on Verification Task:** Implement one ordinary shared graph and one UObject graph, then document destruction/reclamation triggers.
- **Sources:** [SRC-CPP-007], [SRC-EPIC-009], [SRC-EPIC-019], [SRC-EPIC-020]
- **Version Notes:** UE5.3–UE5.6 target; UObject pointer idioms are UE5-sensitive.

## Generic Programming, Memory, Concurrency, and Toolchain

### Question: When is `T&&` a forwarding reference?

- **Category:** Standard C++ / Templates
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** When `T` is a cv-unqualified template parameter deduced for that function call; otherwise it is an ordinary rvalue reference.
- **Strong 3-Year-Engineer Answer:** An lvalue argument deduces `T` as an lvalue reference, and reference collapsing makes `T&&` an lvalue reference; an rvalue remains rvalue. `std::forward<T>` preserves that original category for one downstream consumer. I use forwarding in adaptors/factories, avoid double-forwarding, and do not return a forwarded reference whose temporary dies.
- **Common Weak Answer:** “`T&&` accepts both lvalues and rvalues and `std::forward` moves only rvalues.”
- **Follow-up Questions:** `Widget&&`? Class-template `T&&`? Reference collapsing? Forward twice?
- **Hands-on Verification Task:** Record overload selection for const/non-const lvalues/rvalues and a move-only type.
- **Sources:** [SRC-CPP-015]
- **Version Notes:** C++11+.

### Question: SFINAE versus concepts?

- **Category:** Standard C++ / Templates
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** SFINAE removes invalid candidates during substitution; C++20 concepts name requirements and participate directly in constrained overload ordering.
- **Strong 3-Year-Engineer Answer:** Detection/`enable_if`/`void_t` remains useful for compatibility and traits, but concepts put readable requirements in the interface and improve diagnostics. Neither proves semantic laws such as strict weak ordering, complexity or thread safety. I use small named concepts and ordinary overloads rather than one giant generic branch.
- **Common Weak Answer:** “Concepts are modern SFINAE and prevent template errors.”
- **Follow-up Questions:** Immediate context? Requires-expression? Runtime invariant? Partial ordering?
- **Hands-on Verification Task:** Implement one detection/SFINAE API and concept equivalent; compare bad-call diagnostics.
- **Sources:** [SRC-CPP-016], [SRC-CPP-017]
- **Version Notes:** Concepts require C++20/project support.

### Question: When should you use `if constexpr`?

- **Category:** Standard C++ / Templates
- **Priority:** P1
- **Expected Depth:** D3
- **Short Answer:** For small compile-time behaviour differences where a discarded dependent branch should not be instantiated.
- **Strong 3-Year-Engineer Answer:** It is useful after the interface requirement is clear—for example choosing a representation operation by trait. It does not replace overloads/policies when branches represent different concepts, and non-dependent invalid code can still fail. I avoid turning one template into a sprawling type switch.
- **Common Weak Answer:** “It removes the unused branch at compile time, so it is faster than a normal if.”
- **Follow-up Questions:** Dependent expression? Code size? Concepts plus `if constexpr`?
- **Hands-on Verification Task:** Build a two-type formatter, then refactor an intentionally overgrown compile-time switch.
- **Sources:** [SRC-CPP-017]
- **Version Notes:** C++17+.

### Question: What can go wrong with lambda captures in asynchronous work?

- **Category:** Standard C++ / Lambdas / Lifetime
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** References and `this` can dangle; value/ownership captures can retain too much; callbacks may execute after owner, World or subsystem shutdown.
- **Strong 3-Year-Engineer Answer:** `[this]` captures a pointer, not lifetime. I capture explicit copied immutable inputs or weak/stable handles, then revalidate at execution and dispatch UObject work to Game Thread. Init-capture can transfer unique ownership deliberately. Every task/callback has cancellation or generation policy; default captures hide retention and are avoided in delayed code.
- **Common Weak Answer:** “Capture by value for safety and by reference for performance.”
- **Follow-up Questions:** `[*this]`? Move capture? Stack local? Fire-and-forget?
- **Hands-on Verification Task:** Delay four capture variants past scope/owner destruction and catch the unsafe forms.
- **Sources:** [SRC-CPP-002], [SRC-CPP-026]
- **Version Notes:** Core rules stable; UE task API version-sensitive.

### Question: Template callable versus function pointer versus `std::function` versus Unreal delegate?

- **Category:** Standard C++ / Callable Design
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Choose compile-time/inlinable generic call, stateless ABI, owning runtime type erasure, or engine binding/multicast/reflection semantics respectively.
- **Strong 3-Year-Engineer Answer:** A template exposes callable type and enables optimisation but expands compile/binary surface. Function pointer is simple and stateless. `std::function` owns a copyable type-erased target with indirect/possible allocation cost and empty-call exception. Unreal delegates fit UObject weak/dynamic/multicast integration. I choose semantics first and benchmark hot runtime substitution.
- **Common Weak Answer:** “Templates are fastest; std::function is flexible; delegates are for Blueprints.”
- **Follow-up Questions:** Capturing lambda to function pointer? Allocation? Move-only callable? Empty invocation?
- **Hands-on Verification Task:** Benchmark/store equivalent callbacks and inspect allocation/copy/invocation behaviour.
- **Sources:** [SRC-CPP-018], [SRC-EPIC-028]
- **Version Notes:** Implementation allocation details vary; use target Unreal delegate docs.

### Question: Explain padding, alignment and why raw struct serialisation is dangerous.

- **Category:** Standard C++ / Object Model
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Members/objects need aligned addresses, so object representation may contain indeterminate padding and compiler/ABI-dependent layout.
- **Strong 3-Year-Engineer Answer:** I inspect `sizeof`, `alignof` and member offsets on target. Reordering can reduce size but ABI/semantic/access considerations matter. Raw bytes can include padding, endianness, pointer/vptr and version differences, so network/save formats serialise fields explicitly. Only supported trivially-copyable/relocation contracts permit byte operations.
- **Common Weak Answer:** “Put largest fields first to remove padding and use packed structs for network data.”
- **Follow-up Questions:** `alignas`? Packed unaligned access? Vptr? Trivially copyable?
- **Hands-on Verification Task:** Compare layouts across member orders and repair a raw-byte save mismatch with field schema.
- **Sources:** [SRC-CPP-019]
- **Version Notes:** ABI/layout is compiler/platform-specific.

### Question: AoS versus SoA in a game hot loop?

- **Category:** Standard C++ / Data Layout
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** AoS keeps whole records together; SoA makes individual fields contiguous. Choose from actual access patterns, working set, writes and all consumers.
- **Strong 3-Year-Engineer Answer:** Position-only iteration across thousands may benefit from SoA/SIMD/cache bandwidth; per-entity logic using most fields may favour AoS. Chunked hybrid layouts often work well. I profile time/cache misses/bandwidth and include authoring, indirection, write/merge and other systems before transforming data.
- **Common Weak Answer:** “SoA is faster because it is cache-friendly.”
- **Follow-up Questions:** SIMD? Chunking? Sparse fields? Structural changes?
- **Hands-on Verification Task:** Benchmark equivalent AoS/SoA/chunked integration with one-field and all-field workloads.
- **Sources:** [SRC-CPP-019], [SRC-MASS-001]
- **Version Notes:** Performance result is workload/hardware-dependent.

### Question: Arena versus pool versus general allocator?

- **Category:** Standard C++ / Allocation
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Arena bulk-releases one lifetime region; pool reuses similar slots individually; general allocation handles arbitrary lifetimes/sizes. Each trades memory, reset and complexity.
- **Strong 3-Year-Engineer Answer:** A monotonic arena fits frame/request scratch that dies together; individual deallocate is a no-op and it is not inherently thread-safe. Pools fit repeated similar objects but need generation, reset and retained-memory policy. Escaping references and semantic destructors are the main traps. I profile allocation distribution/lifetimes/contention before introducing either.
- **Common Weak Answer:** “Arenas are fastest for temporaries and pools eliminate allocations.”
- **Follow-up Questions:** Destructor? Exhaustion? Fragmentation? UObject pool? PMR propagation?
- **Hands-on Verification Task:** Build both with intentional escape/reset/destructor bugs and compare peak/retained memory.
- **Sources:** [SRC-CPP-024]
- **Version Notes:** C++17 PMR; Unreal allocator policy must be checked.

### Question: What is a C++ data race?

- **Category:** Standard C++ / Concurrency
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Conflicting unsynchronised accesses to one memory location from different threads, with at least one write, produce undefined behaviour.
- **Strong 3-Year-Engineer Answer:** I identify exact location/accesses and intended happens-before edge. `volatile` is not synchronisation. I protect multi-field invariants with ownership partitioning or mutex/snapshot first, use TSan where supported, and treat lifetime separately: an atomic pointer can still reference a destroyed object.
- **Common Weak Answer:** “A race is when thread execution order gives inconsistent results.”
- **Follow-up Questions:** Conflicting? Happens-before? Two reads? Atomic pointer lifetime?
- **Hands-on Verification Task:** Reproduce and repair a raced counter and a raced multi-field invariant under TSan.
- **Sources:** [SRC-CPP-020], [SRC-CPP-031]
- **Version Notes:** C++11 memory model; TSan platform-sensitive.

### Question: Mutex or atomic?

- **Category:** Standard C++ / Concurrency
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Use a mutex for compound invariants/critical sections; use atomic for a carefully specified single-object/protocol operation.
- **Strong 3-Year-Engineer Answer:** A mutex is easier to prove and its unlock/lock publishes protected state. RAII prevents forgotten unlock and `scoped_lock` handles multiple mutexes. An atomic counter/flag may fit, but several atomics do not make a transactional invariant. I start correct, measure contention, then redesign partitioning/granularity before reaching for lock-free code.
- **Common Weak Answer:** “Atomics for simple values, mutexes for objects.”
- **Follow-up Questions:** Lock-free? False sharing? Destruction? Memory order?
- **Hands-on Verification Task:** Implement one invariant with atomics incorrectly, then repair with mutex and a proven publication variant.
- **Sources:** [SRC-CPP-020], [SRC-CPP-021]
- **Version Notes:** Stable C++11+ principles.

### Question: Explain release/acquire publication and relaxed ordering.

- **Category:** Standard C++ / Atomics
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** A release operation synchronising with an acquire that observes it makes preceding writes visible; relaxed provides atomicity/order for that atomic but no publication of other data.
- **Strong 3-Year-Engineer Answer:** Producer fully writes immutable payload then release-stores ready; consumer acquire-loads ready and may read that payload. No unsynchronised post-publication mutation is allowed. Relaxed suits independent statistics where only atomic count matters. I begin seq_cst/mutex, weaken only from a written proof and test across architectures.
- **Common Weak Answer:** “Acquire prevents later operations moving before the load and release flushes caches.”
- **Follow-up Questions:** Synchronizes-with? CAS order? Atomicity versus visibility? Volatile?
- **Hands-on Verification Task:** Document a publication litmus test and deliberately break it with relaxed ready flag.
- **Sources:** [SRC-CPP-020]
- **Version Notes:** Avoid hardware-only explanations; language memory model governs.

### Question: How should a condition variable be used?

- **Category:** Standard C++ / Concurrency
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Store the condition as mutex-protected state and wait with a `unique_lock` plus predicate loop; notification only prompts rechecking.
- **Strong 3-Year-Engineer Answer:** The producer locks, updates queue/stop predicate and notifies. Consumer's predicate wait handles spurious wakes and missed timing because state—not notification—is truth. It moves work out under lock, unlocks, then executes. Shutdown joins/cancels all waiters before destroying synchronisation objects.
- **Common Weak Answer:** “Wait releases the mutex and notify wakes one waiting thread.”
- **Follow-up Questions:** Spurious wake? Notify before wait? Atomic predicate? Shutdown?
- **Hands-on Verification Task:** Inject missed-notification/spurious-wake/shutdown bugs into a worker queue and fix them.
- **Sources:** [SRC-CPP-022]
- **Version Notes:** C++11+.

### Question: What makes a good task/job-system workload?

- **Category:** Standard C++ / UE Tasks
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** Independent or dependency-explicit work over safe snapshots/partitions, coarse enough to amortise scheduling, with owned outputs, cancellation and thread-affine merge.
- **Strong 3-Year-Engineer Answer:** I snapshot on Game Thread, fork chunks with disjoint/local outputs, express DAG prerequisites rather than blocking workers, merge deterministically and apply on Game Thread. Captures outlive actual execution; releasing a task handle does not imply cancellation. Insights task traces reveal launch/schedule/run/wait and granularity.
- **Common Weak Answer:** “Parallelise expensive loops using one task per core.”
- **Follow-up Questions:** Nested task? Prerequisite? False sharing? Worker deadlock? Owner travel?
- **Hands-on Verification Task:** Build a fork–join pipeline and compare granularity, contention and cancellation under Insights.
- **Sources:** [SRC-CPP-026], [SRC-CPP-023]
- **Version Notes:** UE Tasks APIs/wait behaviour are target-version-sensitive.

### Question: Diagnose unresolved external versus duplicate symbol versus static-init failure.

- **Category:** Standard C++ / Linking
- **Priority:** P0
- **Expected Depth:** D4
- **Short Answer:** Missing definition/export/module/template instantiation causes unresolved; multiple forbidden definitions cause duplicate; cross-TU dynamic global dependencies can compile/link then fail at startup.
- **Strong 3-Year-Engineer Answer:** I read the full symbol signature and compare declaration/definition namespace, cv/ref/template args and API export, then module/Build.cs linkage. Template definitions need visibility or explicit exported instantiation. Duplicate header definitions need inline/internal/source ownership. Startup crashes require examining non-local constructors and replacing cross-TU dependencies with constant/on-first-use or explicit module startup.
- **Common Weak Answer:** “Clean rebuild, add the module dependency and move globals into a cpp.”
- **Follow-up Questions:** ODR? Inline? Header static? Mangling? Unity builds?
- **Hands-on Verification Task:** Create and repair all three failures across two translation units/modules.
- **Sources:** [SRC-CPP-027], [SRC-CPP-028], [SRC-CPP-029]
- **Version Notes:** ABI/export details are toolchain/platform/Unreal-module specific.

### Question: ASan, TSan and UBSan—what does each prove?

- **Category:** Standard C++ / Tooling
- **Priority:** P1
- **Expected Depth:** D4
- **Short Answer:** They dynamically detect classes of memory, race and undefined-behaviour faults in instrumented executed code; none proves total correctness.
- **Strong 3-Year-Engineer Answer:** ASan targets use-after-free/bounds, TSan data races with substantial overhead/restrictions, and UBSan selectable null/alignment/shift/overflow/vptr checks. I preserve symbols, reproduce the same seed/input, fix the first report and avoid blanket suppressions. Partial instrumentation/custom allocators/platform gaps and unexecuted paths remain blind spots.
- **Common Weak Answer:** “ASan finds memory leaks, TSan finds thread bugs, UBSan finds all undefined behaviour.”
- **Follow-up Questions:** Deadlocks? False positives? CI? Shipping target? Partial instrumentation?
- **Hands-on Verification Task:** Trigger one authentic finding for each supported sanitizer and document coverage/overhead/blind spot.
- **Sources:** [SRC-CPP-030], [SRC-CPP-031], [SRC-CPP-032]
- **Version Notes:** Clang/platform/Unreal integration varies.
