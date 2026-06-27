# Systems C++, OS, Networking, and Game-Security Gap Plan

This audit checks the project against a user-requested non-exhaustive list of low-level C++, OS, networking, cryptography, anti-cheat, and data-structure topics. It is a planning checkpoint, not a substitute for the final sourced chapters. Existing coverage is strongest in standard C++ ownership/lifetime, UE C++ lifetime, task-based concurrency, replication, rollback, and algorithm selection. The main gaps are OS-level implementation depth, applied IPC/shared-memory/process knowledge, defensive game-security coverage, cryptography/TLS/MITM fundamentals, and balanced-tree/STL internals.

## Coverage Verdict

| Requested area | Current coverage | Verdict | Update target |
|---|---|---|---|
| Reference vs pointer, implementation/assembly level | Borrowing, dangling, ownership, invalidation are covered in `topics/cpp_for_unreal_interviews.md` and practice. ABI/register/stack representation is not explained deeply. | Partial | Expand C++ chapter with ABI-aware pointer/reference lowering, calling conventions, optimiser caveats, and a Compiler Explorer-style lab. |
| How OOP is achieved in C++ and UE C++ | UE reflection, interfaces, components, UCLASS/UObject, vptr awareness and polymorphism are covered. Standard C++ object model/vtable/multiple-inheritance ABI depth is thin. | Partial | Add C++ object-model/OOP internals section; contrast virtual dispatch/RTTI with UE reflection/UClass/Cast. |
| RAII in C++ and UE C++ | Covered well, including UObject GC distinction and RAII for locks/files/handles/delegates. | Covered | Add only a short OS-handle RAII example when OS chapter is written. |
| `new`/`delete` vs `malloc`/`free` | Covered indirectly through lifetime/storage/allocators; direct constructor/destructor/alignment/operator-new comparison is missing. | Partial | Add direct comparison, placement new, aligned allocation, matching allocation families, `FMemory`/`FMalloc` awareness, and why UObjects use engine factories. |
| Low-level allocation: `brk`, `mmap`, platform virtual allocation | Not substantively covered. | Missing | Add OS virtual-memory and allocator-backend section with Linux `brk`/`mmap`, Windows `VirtualAlloc`, page granularity, commit/reserve, and allocator layering. |
| Memory alignment: how and why | Covered well for object layout, padding, cache lines, false sharing, and raw serialisation risk. | Covered | Add platform/page/cache/SIMD alignment examples and aligned allocation pitfalls. |
| `static` purpose, lifetime, scope/linkage | Covered well in storage duration, linkage, ODR, and static initialisation order. | Covered | Add a compact comparison table: block static, class static, namespace static, `static` function, `inline` variable, `thread_local`. |
| `inline` vs `#define`, complications and build process | ODR, inline semantics, UBT/UHT/build stages are covered; macro hazards are not deep enough. | Partial | Add macro-vs-inline comparison, header-only definitions, templates, generated code, unity-build masking, and preprocessor-debug workflow. |
| Lvalues/rvalues and move | Covered well, including xvalues, forwarding references, moved-from state, `noexcept`, elision. | Covered | Add a diagram tying value categories to overload resolution and UE `MoveTemp`. |
| Thread vs process, multithreading task distribution | Tasks, jobs, mutex/atomics, worker pools and Unreal task boundaries are covered. Process isolation is thin. | Partial | Add OS process/thread model, address spaces, context switches, scheduler, stack/TLS, and how game jobs map onto worker pools. |
| IPC methods | Not substantively covered. | Missing | Add pipes, sockets, named pipes, shared memory, memory-mapped files, RPC, message queues, files, and platform caveats. |
| Shared memory implementation | Not substantively covered. | Missing | Add mmap/file-mapping model, page mapping, synchronisation, lifetime/security, ABI/layout/versioning, and a safe producer/consumer lab. |
| Deadlock | Covered in mutex/scoped_lock and practice, but needs a named Coffman-conditions/debug workflow. | Partial | Add deadlock taxonomy, lock ordering, wait graphs, timeouts, contention profiling, and Unreal/GameThread deadlock examples. |
| Page fault and virtual memory | Only platform memory pressure/profiling is covered. Implementation depth is missing. | Missing | Add address translation, page tables, TLB, demand paging, minor/major faults, copy-on-write, guard pages, stack growth, and page-fault profiling interpretation. |
| DLL injection | Not covered. | Missing, defensive only | Add high-level defensive overview: module loading, remote thread/hook concepts, detection surfaces, signing/integrity, and why client trust remains unsafe. Avoid procedural exploit instructions. |
| Cheat Engine: how and how to fight | Not covered directly. Some server-authority and anti-cheat validation appears in networking. | Missing, defensive only | Add memory editing/scanning concept, runtime value tampering, client-side limits, server validation, telemetry, authoritative simulation, obfuscation limits, and privacy/legal cautions. |
| Replay attack | Partly covered via idempotency, operation IDs, rollback, validation and command semantics; not named as a security topic. | Partial | Add replay-threat section for network commands, purchases/rewards, auth/session tokens, sequence numbers, timestamps, nonces, idempotency keys. |
| ESP, aimbot, etc. and mitigation | Not covered directly. | Missing, defensive only | Add threat taxonomy, information minimisation/relevancy, server-side LOS/range/timing checks, input plausibility, kill-cam/replay review, telemetry, and limits of detection. |
| TCP and UDP, state sync and frame sync | UE replication, reliability, ordering, rollback and server-authority are covered. Generic TCP/UDP and state-sync/frame-sync taxonomy is not deep enough. | Partial | Add transport primer, reliable-UDP concepts, snapshot/state sync, command/input sync, deterministic lockstep, rollback, host authority and late join trade-offs. |
| Symmetric/asymmetric encryption, how and why | Not covered. | Missing | Add crypto primer: symmetric confidentiality/integrity, public-key key exchange/signatures, hashing/MACs, nonces, random generation, do-not-roll-your-own. |
| HTTPS and man-in-the-middle | Not covered. | Missing | Add TLS/HTTPS handshake mental model, certificate validation, trust stores, pinning trade-offs, MITM proxy/dev environments, and what TLS does not solve for game cheats. |
| Red-black tree vs AVL tree | Only generic balanced tree awareness exists. | Partial | Expand algorithms chapter with RB vs AVL invariants, rotation/rebalance trade-offs, lookup/update workloads, and interview answer pattern. |
| Where C++ STL uses red-black tree | Not covered directly. | Missing/implementation-sensitive | Add `std::map`, `std::set`, `std::multimap`, `std::multiset` as ordered associative containers commonly implemented as red-black trees, while noting the standard specifies behaviour/complexity rather than mandating RB tree internals. |

## Proposed New Structure

Add two bounded chapters rather than bloating every existing file:

| New or changed file | Purpose |
|---|---|
| `topics/systems_programming_for_game_interviews.md` | OS/process/thread/virtual-memory/allocation/IPC/shared-memory/security-sensitive loader concepts for game interviews. |
| `topics/game_security_anticheat.md` | Defensive multiplayer and client-security curriculum: memory tampering, replay, ESP/aimbot, DLL/module injection concepts, telemetry, server authority, trust boundaries, and mitigation limits. |
| `topics/cpp_for_unreal_interviews.md` | Expand specific C++ internals: reference/pointer lowering, object model/vtables, `new`/`delete` vs `malloc`/`free`, macro/inline complications, and small ABI-aware labs. |
| `topics/ue_networking_and_replication.md` | Add transport and sync-model primer before UE replication: TCP/UDP, state sync, frame/input sync, lockstep, rollback, replay resistance. |
| `topics/game_algorithms_and_data_structures.md` | Add RB tree vs AVL tree and STL ordered container internals caveat. |
| `practice/cpp_interview_question_bank.md` and `practice/ue_interview_question_bank.md` | Add full-format questions for OS memory, allocation families, vtables, IPC, anti-cheat, replay, TLS, and sync models. |
| `practice/ue_flashcards.md` | Add compact reinforcement cards for the new systems/security topics. |
| `practice/ue_hands_on_projects.md` | Add Project 7D Systems Programming Lab and Project 3 Security/Replay Threat-Model Extension. |
| `data/ue_interview_knowledge_graph.json` and `.md` | Add nodes after the chapters exist: OS memory, IPC/shared memory, transport protocols, crypto/TLS, anti-cheat threat model, balanced trees. |

## Source Collection Plan

Collect sources only for the bounded update, in this priority order:

| Source family | Why |
|---|---|
| ISO/C++ reference and cppreference pages | Object lifetime, references, new/delete, alignment, ordered associative containers, value categories, storage duration, ODR. |
| Microsoft Learn platform docs | Windows virtual memory, file mapping/shared memory, process/thread APIs, DLL loading/signing/security notes. |
| Linux man-pages and kernel docs | `brk`, `mmap`, page faults, shared mappings, process address space, copy-on-write. |
| RFCs and official TLS references | TCP, UDP, TLS 1.3, HTTPS/certificate model, replay-relevant nonce/sequence concepts. |
| Unreal/Epic networking docs and existing project sources | Tie generic sync models back to UE replication, CharacterMovement prediction, NetworkPrediction, Iris/RepGraph caveats. |
| Security engineering references | Threat modelling, anti-cheat limitations, telemetry/privacy, client trust boundaries, OWASP-style replay/MITM concepts. |
| Supplementary tool/community sources | Cheat Engine and DLL-injection concepts only as labelled supplementary evidence; do not include step-by-step offensive procedures. |

## Defensive-Safety Boundary

The anti-cheat and DLL-injection material should explain concepts, detection surfaces, and defences. It should not include operational exploit recipes, bypass instructions, copy-paste injector code, kernel-driver guidance, or instructions for defeating commercial anti-cheat. The goal is interview-ready defensive reasoning:

1. assume the client can be inspected and modified;
2. keep game truth server-authoritative where feasible;
3. minimise sensitive replicated information;
4. validate timing, range, line of sight, cooldowns, resources and sequence;
5. use telemetry, replay review and ban policy carefully;
6. treat obfuscation and integrity checks as layers, not trust foundations;
7. preserve player privacy, legal compliance and platform requirements.

## Update Sequence

1. **Systems C++ expansion:** update `cpp_for_unreal_interviews.md` with pointer/reference ABI awareness, OOP/vtable/UE reflection contrast, direct allocation-family comparison, macro/inline build hazards, and OS-handle RAII.
2. **OS systems chapter:** create `topics/systems_programming_for_game_interviews.md` with virtual memory, page faults, `brk`/`mmap`/`VirtualAlloc`, process/thread model, IPC and shared memory.
3. **Networking/security primer:** expand `ue_networking_and_replication.md` with TCP/UDP and sync-model taxonomy; create `topics/game_security_anticheat.md` for replay, MITM/TLS, Cheat Engine-style memory tampering, ESP/aimbot and defensive design.
4. **Algorithms patch:** expand `game_algorithms_and_data_structures.md` with RB vs AVL and STL ordered-container implementation caveat.
5. **Practice integration:** add 25-35 full-format questions, 80-120 flashcards, and two hands-on labs across C++ systems, OS, networking security and anti-cheat.
6. **Graph/status integration:** add graph nodes/edges after the written content is stable; update source index with actual collected sources and then update `.codex/STATUS.md`.

## Suggested Priority and Depth

| Topic cluster | Priority | Expected depth | Rationale |
|---|---|---|---|
| Pointer/reference, object model, RAII, allocation families | P0-P1 | D3-D4 | Common C++ interview foundation for UE/game roles. |
| OS virtual memory, page faults, process/thread, IPC | P1-P2 | D2-D3 | Common systems follow-up; important for debugging memory/performance/tooling. |
| Shared memory and DLL/module loading | P2-P3 | D1-D3 | Useful for tools, platform, anti-cheat and engine roles; defensive framing required. |
| TCP/UDP and sync models | P0-P2 | D3-D4 | Core multiplayer interview knowledge; links directly to UE replication/rollback. |
| Cryptography/TLS/MITM/replay | P2-P3 | D2-D3 | Important for online games/services/security judgement, not usually UE-specific. |
| Anti-cheat threat modelling | P2-P3 | D2-D4 for networking/online roles | Helps distinguish server authority from client-side wishful thinking. |
| RB vs AVL and STL ordered containers | P1-P2 | D2-D3 | Common data-structure interview topic; useful contrast with hash/dense game structures. |

## Acceptance Criteria for the Update

- Every requested topic above has at least one substantive explanation, one interview-answer pattern, and one practice artefact.
- Defensive security content clearly separates high-level threat models from procedural attack instructions.
- OS/platform topics label Windows/Linux differences and do not pretend `brk`/`mmap` are Windows APIs.
- C++ internals explain implementation tendencies without claiming the standard mandates a specific assembly, vtable layout, or STL tree implementation.
- Networking content distinguishes TCP/UDP, reliable/unreliable, state sync, input/frame sync, rollback, replay attacks and TLS/MITM.
- The source index records actual collected sources; this plan intentionally does not invent source IDs before collection.
