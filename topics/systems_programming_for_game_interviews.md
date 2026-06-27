# Systems Programming for Game Interviews

See also: [[cpp_for_unreal_interviews]], [[ue_networking_and_replication]], [[game_security_anticheat]], [[ue_profiling_optimisation]], [[ue_hands_on_projects]].

**Priority:** P1 for gameplay/generalist/tools/networking roles; P0 when the role touches engine, platform, anti-cheat, networking infrastructure or performance.  
**Expected depth:** D2-D3 for most three-year Unreal engineers; D4 when you claim engine/platform depth.  
**Scope:** C++/OS fundamentals that explain what allocators, tasks, IPC, virtual memory and module loading are doing underneath game code.

## 1. Process, Thread, Task

A process is an executing program with its own virtual address space, handles/resources, loaded modules and security context. A thread is the OS-scheduled execution unit inside a process; threads in the same process share address space and can execute any process code. Microsoft documents a thread as the basic unit to which the OS allocates processor time. [SRC-SYS-003]

| Concept | What it owns | What it shares | Interview trap |
|---|---|---|---|
| Process | Address space, loaded modules, handles, security boundary | Usually nothing directly; communicates through OS IPC | "Two processes can just read the same pointer." |
| Thread | Stack, registers, instruction pointer, thread-local state | Process heap/globals/code/handles | "More threads means faster." |
| Task/job | Work item scheduled onto worker threads | Depends on captured data and scheduler | "Destroying a task handle cancels running work." |

Multithreading distributes tasks by splitting independent or staged work across worker threads, then synchronising where dependencies meet. Good game workloads are partitionable, bounded, and mostly data-parallel: animation evaluation, particle updates, path queries, asset processing, cooking, compression, or per-chunk simulation. Bad workloads capture mutable UObject state, block on the Game Thread, fight over one lock, or create work so tiny that scheduling dominates. [SRC-CPP-026]

Strong interview answer:

> A thread is not the same thing as a task. I design the dependency graph first: snapshot inputs, split independent chunks, avoid shared writes, merge deterministically, and profile queueing, execution and wait time. If work is memory-bandwidth-bound or serial by design, adding threads can move the bottleneck or increase contention rather than improve frame time.

## 2. IPC and Shared Memory

Inter-process communication exists because processes do not normally share ordinary pointers. Use the smallest mechanism that matches the data flow, lifetime, latency and trust boundary.

| IPC mechanism | Good for | Costs and risks |
|---|---|---|
| Command line/environment | launch-time configuration | one-shot, not live communication |
| Exit codes/stdout/stderr | build tools, commandlets, automation | stream parsing, buffering, process lifetime |
| Files | durable handoff, cook artifacts, logs | latency, locking, cleanup, partial writes |
| Pipes | parent/child streaming, tool protocols | byte stream framing, back-pressure |
| Sockets | local/remote services | protocol design, security, ordering/loss depending on transport |
| OS messages/events/semaphores | signalling | not bulk data; lifetime/security handles matter |
| Shared memory | low-latency bulk exchange | needs explicit layout, synchronisation, versioning and permissions |

POSIX shared memory creates or opens a named object, sizes it, maps it into each process's virtual address space, and normally requires separate synchronisation. The same physical pages can appear at different virtual addresses in each process, so the data layout must avoid raw process-local pointers unless they are offsets or handles. [SRC-SYS-008] [SRC-SYS-005]

Windows named shared memory follows the same high-level shape: create a file-mapping object, map a view in one process, open the mapping in another, and map a corresponding view there. Names, security descriptors, handle lifetime and view lifetime are part of the contract. [SRC-SYS-002]

Shared-memory design checklist:

1. Fixed header with magic, version, endian/ABI assumptions, capacity and writer state.
2. Offsets or indices instead of raw pointers.
3. One clear ownership rule for each slot/range.
4. Synchronisation: mutex/semaphore/event/atomics with a written memory-order proof.
5. Crash/abandon policy for half-written data.
6. Permissions and namespace policy.
7. Observability: counters for overruns, missed frames, wait time and stale readers.

## 3. Virtual Memory, Pages and Faults

Virtual memory gives each process a virtual address space. The OS maps virtual pages to physical memory, files, zero pages, device memory or "not present" states. A pointer value is an address in a process's virtual address space, not a universal physical location.

VirtualAlloc exposes the reserve/commit split on Windows: reserving address space does not allocate physical storage, while committing charges backing store and guarantees zero-fill when the page is first accessed. Physical pages are not necessarily allocated until access. [SRC-SYS-001]

On Linux, `mmap` creates a mapping in the virtual address space. It can map files, anonymous memory, shared mappings and private copy-on-write mappings. The mapping operation defines address range, protection and sharing behaviour, but page contents and physical residency are still demand-driven. [SRC-SYS-005]

`brk`/`sbrk` move the program break, the end of the process data segment. They are historically important because allocators may use them for heap growth, but modern code should normally call allocator APIs rather than `brk` directly. The Linux man page explicitly says to avoid `brk`/`sbrk` and use `malloc` as the portable allocation interface. [SRC-SYS-006]

A page fault is not automatically a crash. It is the CPU/OS mechanism that handles a virtual page access whose translation or permission is not immediately satisfied. Common cases:

| Fault type | What it often means | Game implication |
|---|---|---|
| Minor/soft fault | page can be satisfied without disk I/O, such as zero-fill or already-resident page table update | can still hitch if many pages are touched at once |
| Major/hard fault | backing data must be read from storage | high hitch risk during streaming/loading |
| Protection fault | access violates page permissions | often crash/security guard/use-after-free clue |

Linux `getrusage` exposes `ru_minflt` and `ru_majflt` as minor and major page fault counts, which makes them useful in memory experiments and load-time diagnostics. [SRC-SYS-007]

Interview pattern:

> I separate virtual address reservation, committed backing, physical residency and object lifetime. Allocators manage objects and sub-allocations; OS page APIs manage address ranges and protection. A hitch after loading may be page touching, decompression, shader compilation, IO, allocator work or asset initialisation, so I measure faults, IO, CPU and allocation traces instead of saying "the heap is slow."

## 4. `new`/`delete` Versus `malloc`/`free`

`new` is a C++ expression: it obtains storage, then constructs an object. `delete` destroys the object, then releases storage through the matching deallocation function. Arrays use `new[]`/`delete[]`, and placement `new` constructs an object in supplied storage without allocating by itself. [SRC-CPP-034] [SRC-CPP-035]

`malloc` allocates suitably aligned raw storage and returns `void*`; it does not construct a C++ object. `free` releases storage allocated by the C allocation family; it does not run destructors. [SRC-CPP-036]

| Pair | Constructs/destructs? | Correct use |
|---|---:|---|
| `new T` / `delete` | yes | ordinary C++ object allocation when dynamic lifetime is needed |
| `new T[n]` / `delete[]` | yes for each element | dynamic arrays, rarely preferred over containers |
| `malloc` / `free` | no | C ABI, raw buffers, custom allocator internals |
| placement `new` / explicit destructor | construction only | arenas/pools where storage is managed separately |

Never mix allocation families. `delete` memory from `new`, `free` memory from `malloc`, and pair custom allocators with their matching release. In Unreal, also distinguish ordinary C++ allocation from engine object factories: use `NewObject`, `CreateDefaultSubobject` and `SpawnActor` for UObject/Actor lifetimes rather than `new UObject`. [SRC-EPIC-003] [SRC-EPIC-004]

## 5. Alignment, Padding and Cache

Alignment is the address multiple required or preferred for a type. The C++ object model treats object size, alignment, padding and representation as core properties; some bit patterns may be padding or trap/invalid for a type, and exact layout can depend on ABI/compiler. [SRC-CPP-019]

Why it matters:

- Correctness: misaligned access can be slower, faulting, or undefined for some operations/types.
- ABI: structs passed across DLL/plugin/process/network/file boundaries need an explicit layout contract.
- SIMD/GPU/IO: many APIs prefer or require specific alignment.
- Cache: packing can reduce footprint, but poor field order can increase false sharing or hurt hot-loop access.
- Serialisation: copying raw padded bytes can leak uninitialised data and break across compilers/platforms.

Practical rule: optimise layout from the access pattern, not from a "sort fields smallest to largest" mnemonic. Measure size, alignment, hot-field locality, false sharing and serialisation needs separately.

## 6. `static`: Storage Duration, Linkage and Scope

The keyword `static` is overloaded. Explain which meaning you mean.

| Use | Meaning | Pitfall |
|---|---|---|
| block-scope `static` local | static storage duration, initialised once when control first passes | shutdown order and hidden state in tests/hot reload |
| namespace-scope `static` | internal linkage | duplicate state per translation unit if placed in headers |
| class `static` data member | one member associated with the class, not each instance | needs correct definition or inline variable rules |
| class `static` member function | no `this` pointer | cannot access instance state without an object |

Scope is where a name is visible; lifetime is when an object exists; linkage is whether declarations denote the same entity across translation units; storage duration is where/how long storage lasts. These are related, not synonyms. [SRC-CPP-028] [SRC-CPP-027]

## 7. `inline` Versus `#define`

`inline` is mainly a language/ODR facility, not a promise that the optimiser will inline machine code. Inline functions/variables may have definitions in multiple translation units when the definitions match and are reachable; compilers may inline functions that are not marked `inline`, or call functions that are. [SRC-CPP-038]

`#define` performs preprocessor token substitution before normal C++ compilation. It does not obey type checking, scope, overload resolution or ordinary debugging boundaries the way a function/constant/template does. [SRC-CPP-039]

Prefer:

- `constexpr`/`constinit`/typed constants over value macros;
- inline/template functions over expression macros;
- generated/reflection macros only where Unreal's build toolchain requires them;
- macros with namespacing, parentheses and tiny surface area when unavoidable.

Build-process complication: macros can make two translation units see different definitions for "the same" inline function or template, creating ODR problems that are hard to diagnose. Unity builds can hide missing includes or macro-order bugs that fail in non-unity builds. [SRC-CPP-027] [SRC-CPP-038]

## 8. References, Pointers and Assembly-Level Reality

A C++ reference is a language-level alias. It must be initialised to refer to a valid object/function, references are not objects, and the implementation may allocate storage only when needed to preserve semantics. A pointer is an object value that stores an address-like value and can be reseated, null, copied, stored in arrays and manipulated explicitly. [SRC-CPP-033]

At assembly/ABI level, a by-reference parameter is often implemented like a hidden pointer: the callee receives an address and loads/stores through it. But that is an implementation strategy, not the full semantic contract. Optimisation may keep the referred value in a register, remove the indirection, devirtualise, or prove aliasing constraints differently. Do not claim "references are just pointers" without the qualifier "often represented as addresses when storage/passing is required." [SRC-CPP-033] [SRC-CPP-019]

Interview distinction:

| Question | Pointer answer | Reference answer |
|---|---|---|
| Can represent "no object"? | yes, with `nullptr` or sentinel | normally no; use pointer/optional/reference wrapper |
| Can reseat? | yes | no for the reference itself |
| Owns? | no by default | no by default |
| ABI representation? | address-sized object in many ABIs | often address-like for parameters/members, but not an object in the standard model |

## 9. OOP in C++ and UE C++

C++ achieves runtime polymorphism with classes, inheritance and virtual functions. A final overrider is selected dynamically for virtual calls through base pointers/references. The standard specifies the behaviour, not a required vtable layout. Typical ABIs implement virtual dispatch with a hidden vptr in each polymorphic object and one or more vtables per dynamic type, but multiple inheritance, virtual inheritance, devirtualisation and link-time optimisation complicate the exact layout. [SRC-CPP-037] [SRC-CPP-019]

Unreal adds a second type system for reflected UObjects: `UCLASS`, `USTRUCT`, `UFUNCTION`, metadata, `Cast<>`, Blueprint exposure, serialisation, GC reachability and editor tooling. UE C++ still uses ordinary C++ virtual dispatch where appropriate, but Blueprint-implementable events/reflection calls are not the same mechanism as a normal vtable call. [SRC-EPIC-001] [SRC-EPIC-002] [SRC-EPIC-009]

Strong answer:

> In ordinary C++, OOP is classes plus encapsulation and dynamic dispatch through virtual functions. A vtable is the usual ABI implementation but not the language guarantee. In Unreal, UObject types additionally participate in generated reflection metadata, GC, serialisation and Blueprint/event dispatch. I choose native virtuals for ordinary C++ extension points, reflected interfaces/events for Blueprint/editor/network integration, and composition when inheritance would mix unrelated lifetimes or responsibilities.

## 10. Deadlock

A deadlock needs four conditions: mutual exclusion, hold-and-wait, no preemption and circular wait. Break any one deliberately.

Practical prevention:

- keep lock ordering documented and enforce it;
- prefer one lock for one invariant;
- acquire multiple mutexes with deadlock-aware helpers such as `std::scoped_lock`;
- do not call arbitrary delegates/Blueprint/log sinks while holding hot locks;
- use timeouts only as diagnostics unless the data invariant can recover;
- make ownership transfer and cancellation paths explicit.

Diagnosis workflow: capture thread stacks, owned/waited locks, task IDs, owner generations and the last progress marker. Then reduce to the smallest wait cycle. "It hangs sometimes" becomes actionable only when you can name thread A waiting for lock X while holding Y, and thread B waiting for Y while holding X. [SRC-CPP-021]

## 11. Hands-On Verification

Extend Project 7B with a Systems Lab:

1. Print process ID/thread IDs and show two processes cannot share an ordinary pointer.
2. Build a tiny producer/consumer over a pipe or local socket.
3. Build a shared-memory ring buffer with offsets, versioned header, semaphore/event signalling and overrun counters.
4. Reserve/commit/touch a large range and measure first-touch page faults and zero-fill cost on the target OS.
5. Compare `new`, `malloc`, placement `new`, arena and pool cases with destructor counters.
6. Reproduce a deadlock, fix it with lock ordering or `std::scoped_lock`, and write the wait graph.
7. Inspect assembly for pass-by-pointer and pass-by-reference at `-O0` and optimised builds, then explain why it changed.
8. Build one pure C++ virtual hierarchy and one UObject/reflected event example; compare dispatch, tooling and lifetime implications.

## Source Map

- C++ object/lifetime/allocation/reference/OOP/static/inline/macro: [SRC-CPP-001] [SRC-CPP-019] [SRC-CPP-027] [SRC-CPP-028] [SRC-CPP-033] [SRC-CPP-034] [SRC-CPP-035] [SRC-CPP-036] [SRC-CPP-037] [SRC-CPP-038] [SRC-CPP-039]
- UE object/reflection factories: [SRC-EPIC-001] [SRC-EPIC-002] [SRC-EPIC-003] [SRC-EPIC-004] [SRC-EPIC-009]
- OS memory/process/IPC: [SRC-SYS-001] [SRC-SYS-002] [SRC-SYS-003] [SRC-SYS-005] [SRC-SYS-006] [SRC-SYS-007] [SRC-SYS-008]
- C++ synchronisation/tasks: [SRC-CPP-021] [SRC-CPP-022] [SRC-CPP-026]
