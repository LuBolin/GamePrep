# Unreal Engine Hands-On Projects

See also: [[ue_interview_question_bank]], [[ue_flashcards]], [[cpp_for_unreal_interviews]], [[systems_programming_for_game_interviews]], [[ue_networking_and_replication]], [[game_security_anticheat]], [[ue_profiling_optimisation]].

## Project 7A: UObject Lifetime and Reference Graph Lab

**Goal:** Build a minimal, repeatable experiment that turns UObject lifetime and pointer choice from memorised rules into observed behaviour.  
**Target Roles:** Gameplay, engine generalist, tools, AI, networking.  
**Topics Covered:** UHT/reflection, CDO awareness, `NewObject`, `Outer`, mark-and-sweep GC, `TObjectPtr`, raw/weak/soft/strong references, async capture safety, GC profiling.  
**Minimum Features:** A UObject owner; five child/reference cases; forced collection; structured logs; one asset loaded through a soft reference; one accidental-retention case; one missing-edge case.  

**Implementation Steps:**

1. Create `ULifetimeLabOwner` and `ULifetimeProbe`. Give every probe a unique label and log creation plus destruction-phase callbacks.
2. Store probes using (a) `UPROPERTY() TObjectPtr`, (b) raw member pointer, (c) `TWeakObjectPtr`, and (d) `TStrongObjectPtr` in a non-UObject helper. Keep the variables otherwise comparable.
3. Create probes with `NewObject` using the same Outer. Predict which survive collection before running.
4. Trigger collection from a development-only command. Record validity after each pass; never ship forced per-frame GC.
5. Remove the reflected strong edge while keeping Outer unchanged. Use the result to explain why containment is not the same contract as retention.
6. Add a `TSoftObjectPtr` asset selected in defaults. Show null/pending/valid states and load it asynchronously; do not conflate asset loading with probe retention.
7. Capture a probe weakly in a delayed callback and resolve it at execution. Contrast with a deliberately unsafe raw capture in a controlled test.
8. Create accidental retention, identify its strong chain, then change the appropriate edge to weak.

**Key Unreal Systems:** CoreUObject, reflection, GC, object factories, soft object paths/streaming, timers or async completion.  
**Debugging Exercises:** Diagnose one early collection, one stale observer, one unexpected retained object, and one world-dependent action incorrectly placed in a constructor.  
**Performance Checks:** Capture collection in Unreal Insights; compare a burst of probe allocation/destruction with a modest reuse pool; report both frame time and retained memory rather than declaring a universal winner.  
**Interview Questions Unlocked:** What does `UPROPERTY` do? Why `TObjectPtr`? Weak versus soft? What does Outer mean? Why is `TSharedPtr<UObject>` normally wrong? How do you debug GC hitches?  
**Stretch Goals:** Add an `FGCObject` native owner; test experimental incremental reachability in a separate UE5.6 configuration; use verification console variables; inspect reference chains with engine tooling/source available to the project.  
**Version / Plugin Caveats:** UE5.3–UE5.6 target. Incremental reachability is experimental in UE5.6. `TLazyObjectPtr` is deprecated and excluded. [SRC-EPIC-005] [SRC-EPIC-009]

### Evidence sheet

| Case | Prediction | Observed before GC | Observed after GC | Explanation |
|---|---|---|---|---|
| Reflected `TObjectPtr` |  |  |  |  |
| Raw member only |  |  |  |  |
| Weak observer only |  |  |  |  |
| Native strong owner |  |  |  |  |
| Same Outer, no retaining edge |  |  |  |  |

### Project 7A UE C++ Specialist Contract Extension

Add a focused contract lab to the UObject lifetime project. The goal is to prove which system consumes each specifier, which lifetime path it affects, and which assumptions are false.

**Minimum Features:**

1. One `UObject` settings class with `Config` values and an explicit load/save policy.
2. One `USaveGame` value record with stable IDs, schema version and at least one intentionally missing/renamed definition case.
3. One Actor or Component with default-tuned, instance-authored, runtime-only, duplicated and transient fields.
4. One reflected `UINTERFACE` implemented by a native Actor and a Blueprint child.
5. One ordinary native abstract C++ interface used only inside a non-reflected helper boundary.
6. One `NewObject` UObject graph with explicit Outer/name/flags plus a duplicate test.
7. One sync utility using `TOptional`, `TVariant`, a non-owning callable parameter and a contiguous view.
8. One async Blueprint proxy around a timer, async load or task with cancellation and stale-generation rejection.
9. One `FGCObject` or explicitly rejected `FGCObject` design memo, with the simpler reflected-owner alternative shown.
10. One task or async computation that snapshots plain data and applies results back on the Game Thread.

**Implementation Steps:**

1. Write a table before coding: declaration, specifier/metadata, consuming system, expected editor/Blueprint/runtime/save/duplicate behaviour and verification method.
2. Add authoring fields using `EditDefaultsOnly`, `EditInstanceOnly`, visible-only fields and Blueprint read/write variants. Observe class defaults, placed instances and Blueprint access separately.
3. Add `Config`, `SaveGame`, `Transient` and `DuplicateTransient` cases. Test config reload, duplicate/PIE copy, save/load and packaged build where possible.
4. Implement `IInteractableContract` as a reflected interface. Call both a native implementation and a Blueprint implementation from C++ without assuming a direct native virtual call covers Blueprint-only behaviour.
5. Implement a pure native interface for a small algorithm service. Record why reflection is unnecessary there.
6. Create child UObjects with distinct Outer/name/flags. Duplicate the graph and log name, class, Outer, package, flags and whether references point to source or duplicate.
7. Add useful log categories and `UE_LOGFMT` correlation fields. Add one `check`, one `verify` and one `ensure` with written rationale.
8. Wrap a query result in `TOptional` instead of a sentinel; use `TVariant` for a closed result type; use a non-owning callback/view only synchronously.
9. Build the async proxy. Test success, failure, owner destruction, manual cancel, travel/PIE stop and success-after-cancel. Broadcast only on the Game Thread and only once.
10. Add one native owner that needs to retain UObjects. Prefer reflected ownership; if using `FGCObject`, prove retained and released cases under forced GC and travel.
11. Snapshot target data on the Game Thread, run a worker computation, then revalidate weak owner/World/generation before applying the result.
12. Produce a two-page evidence memo: surprises, corrected mental models, red flags for interview answers and the minimal safe pattern you would use in production.

**Debugging Exercises:** UHT unsupported type; wrong generated header order; missing public module dependency; metadata mistaken for runtime rule; saved live Actor pointer; duplicate copies cache state; async proxy collected early; proxy leaked by delegate; worker touches UObject; stale completion after travel; wrong assertion for recoverable failure.  
**Performance Checks:** log volume by category/verbosity, duplicate/load/save operation cost, async proxy count/leaks, UObject count/GC before and after native owner teardown, task granularity and Game Thread apply time.  
**Interview Questions Unlocked:** property specifiers by consuming system; metadata versus runtime authority; config/save/transient/duplicate boundaries; `Outer`; reflected versus native interfaces; object APIs; logs/assertions; `TOptional`/`TVariant`/`TFunctionRef`; async Blueprint proxies; `FGCObject`; task/UObject boundaries.  
**Version / Plugin Caveats:** UE5.3-UE5.6 target. `UE_LOGFMT` is UE5.2+. Rare specifiers, Core vocabulary APIs, async proxy signatures and `FGCObject` details must be compiled against the exact target branch. [SRC-EPIC-032] [SRC-EPIC-035] [SRC-EPIC-037] [SRC-EPIC-038] [SRC-EPIC-039] [SRC-EPIC-040] [SRC-EPIC-041] [SRC-EPIC-042] [SRC-EPIC-043] [SRC-EPIC-044]

## Project 7B: C++ Value and Ownership Lab

**Goal:** Make ordinary C++ copy, move, destruction, ownership, invalidation, and locality behaviour visible and measurable beside the UObject lab.  
**Target Roles:** All C++-based Unreal roles.  
**Topics Covered:** RAII, Rule of Zero/Five, unique/shared/weak pointers, moved-from state, `noexcept`, copy elision, iterator invalidation, object pools, cache locality.  
**Minimum Features:** Instrumented value/resource types; unique ownership transfer; shared cycle and weak repair; vector invalidation tests; AoS/SoA or value/pointer locality comparison; simple pool with reset contract.  

**Implementation Steps:**

1. Build an instrumented `FTrackedValue` that logs construction, copy, move, assignment, destruction, and a stable debug ID.
2. Return it and a vector of it from factories. Compare prvalue return, named-local return, and `return std::move(Local)`.
3. Add an RAII resource wrapper. Delete copy, implement a genuinely non-throwing move, and prove exactly one release.
4. Store throwing-move and `noexcept`-move variants in `std::vector`; record relocation copies/moves.
5. Transfer a `unique_ptr<FJob>` through factory, queue, and consumer APIs. Mark every borrower explicitly.
6. Create two `shared_ptr` nodes with a strong cycle, observe missing destruction, then change one back-edge to `weak_ptr` and use `lock()`.
7. Save iterator/pointer/reference handles across vector `reserve`, `push_back`, `insert`, and `erase`; predict validity before each run. Use AddressSanitizer/debug iterators where available.
8. Compare contiguous values with separately allocated pointer-owned objects in a hot iteration benchmark. Report hardware/build and avoid universal claims from one microbenchmark.
9. Implement a small object pool with acquire/release and explicit reset. Test stale external handles and double release; document why pooling trades churn for retained memory and reset complexity.

**Key Unreal Systems:** Unreal module/toolchain integration, `TUniquePtr`/`TSharedPtr` equivalents, Unreal Insights or platform profiler for allocation/cache experiments.  
**Debugging Exercises:** Double-delete from shallow copy; dangling observer after owner destruction; shared cycle leak; use-after-vector-reallocation; use-after-move assumption; pool state leakage.  
**Performance Checks:** Allocation count, copy/move count, vector growth, reference-count traffic, iteration time, peak memory, pool warm/cold behaviour.  
**Interview Questions Unlocked:** RAII, Rule of Zero/Five, move semantics, `noexcept`, unique/shared/weak ownership, iterator invalidation, locality, pooling trade-offs, C++ versus UObject lifetime.  
**Stretch Goals:** Custom deleter, pImpl with incomplete type, allocator-aware container, sanitiser CI target, lock-free reclamation reading notes.  
**Version / Plugin Caveats:** Standard C++17 baseline; verify compiler, exception, sanitiser, and standard-library support in the project toolchain. [SRC-CPP-001] [SRC-CPP-005] [SRC-CPP-009]


## Project 7C: Game Algorithms Verification Harness

**Goal:** Build a standalone C++ or Unreal automation harness that proves algorithms against simple references and measures realistic/adversarial game workloads.  
**Target Roles:** Gameplay, AI, engine generalist, networking, optimisation.  
**Topics Covered:** complexity, heaps, binary search, graph search, DAGs, DSU, weighted random, ring buffers, spatial hashing and differential testing.  
**Minimum Features:** deterministic generated tests; slow correctness oracles; invariant checks; reproducible failure seeds; separate benchmark mode; path visualisation; spatial comparison; written complexity notes.  

**Implementation Steps:**

1. Create deterministic cases recording seed, size, distribution and parameters; keep assertion and timing runs separate.
2. Implement half-open lower-bound and compare every insertion point with the standard algorithm, including empty/duplicates/extremes.
3. Implement a min-heap with invariant checker; compare top-K against full sort and inject reversed-comparator/stale-priority failures.
4. Implement BFS, DFS, Dijkstra and A* over one graph interface. Compare A* cost with Dijkstra on non-negative graphs and visualise expansions.
5. Exercise weighted/unreachable grids, invalid heuristics, duplicate frontier entries, deterministic ties and frame-budget cancellation.
6. Implement deterministic Kahn topological sort with an exact cycle report.
7. Implement DSU with path compression/union-by-size and compare random connectivity against BFS.
8. Implement linear and prefix/binary weighted sampling; validate negative/NaN/zero-total policy, boundaries and distribution.
9. Implement a bounded frame-history ring buffer with wrap, overwrite/reject, stable frame IDs and restore-window tests.
10. Keep all-pairs neighbour search as oracle; implement a 2D spatial hash using floor for signed cells, multi-cell AABBs, deduplication and exact filtering.
11. Generate uniform, clustered, formation, negative-coordinate and mixed-size moving populations; compare exact sets every frame.
12. Record build/update/query, allocations, candidates/hits, maximum bucket/frontier, expansions and P50/P95/P99.
13. Inject one bug per system and require a minimal reproducible seed, not only a mismatch count.
14. Present requirement → baseline → invariant → complexity → edge case → production trade-off → evidence.

**Debugging Exercises:** mutable hash key; invalid comparator; binary off-by-one; BFS visited-on-pop; negative Dijkstra edge; invalid heuristic; topo cycle; DSU metadata corruption; weighted boundary gap; negative-cell truncation; duplicate candidates.  
**Performance Checks:** phase-separated costs, memory/slack, frontier/bucket maxima, uniform versus clustered P95/P99, resumable work.  
**Interview Questions Unlocked:** data selection; expected/amortised complexity; heaps; search; BFS/DFS/Dijkstra/A*; topo/DSU; sampling; spatial structures; broad/narrowphase; proof.  
**Stretch Goals:** flow field, hierarchical pathfinding, Fenwick tree updates, BVH comparison, rollback resimulation and counterexample shrinking.  
**Version / Plugin Caveats:** Concepts are engine-independent. For UE containers/tests, document UE5.3–UE5.6 APIs, invalidation and determinism separately from standard-library claims. [SRC-ALG-002] [SRC-ALG-007] [SRC-ALG-010] [SRC-ALG-011] [SRC-ALG-012]

### Project 7C visual game-maths extension

Add a UE visual playground that draws vector/dot-cone/projection/reflection/cross-basis results; point versus direction transforms through nested non-uniform parents; Euler lerp versus quaternion Slerp; fixed-alpha versus exponential versus constant-speed movement; closest segment point; ray/plane/sphere/AABB parameters and normals; and local/world/view/clip/NDC/screen coordinates. Generate zero, parallel, tangent, inside, behind, negative-scale and large-coordinate cases. Differential-test custom primitives against engine helpers/traces where semantics match, and record space, units, tolerance, frame rate and UE version for every result. Add a material panel demonstrating tangent/world normal, accidental sRGB data texture and non-linear depth. [SRC-MATH-001] [SRC-MATH-004] [SRC-MATH-007] [SRC-MATH-009] [SRC-MATH-010]

## Project 1: Core Gameplay Sandbox

**Goal:** Demonstrate deliberate placement of behaviour and state across the core gameplay framework, including one respawn and one two-client test.  
**Target Roles:** Gameplay, networking, AI/gameplay, engine generalist, technical design.  
**Topics Covered:** Actor/Component composition, Character/Controller, GameMode/GameState/PlayerState, C++/Blueprint boundary, lifecycle, interfaces, delegates, soft references, UI, SaveGame.  
**Minimum Features:** C++ Character with Blueprint child; reusable Health Component; damage; C++ interactable interface; DataAsset-driven item; event-driven health UI; SaveGame record; respawn preserving public player score; replicated shared match score; one soft asset reference.  

**Implementation Steps:**

1. Draw a one-page responsibility map before coding: rules in GameMode, shared match state in GameState, public participant score in PlayerState, avatar state/capabilities on Character components, local UI coordination on the owning client.
2. Create a C++ Character base and Blueprint child. Create default collision/visual components in the constructor; perform runtime bindings in the appropriate play hook.
3. Implement `UHealthComponent` with server-authoritative mutation, explicit death event, no direct UI dependency, and balanced delegate cleanup.
4. Implement an interaction interface and one pickup Actor whose definition is a DataAsset. Keep display asset references soft where eager loading is unnecessary.
5. Drive the health widget from events, not per-frame property binding/polling.
6. Add match score to GameState and scoring rules to GameMode. Add player score to PlayerState and prove it survives Pawn destruction/respawn.
7. Save stable identifiers/data, not live Actor pointers. Reload in a fresh world.
8. Test standalone, listen server plus one client, respawn, map reload, and PIE shutdown.

**Key Unreal Systems:** Gameplay Framework, Components, reflection, replication foundations, DataAsset, interfaces/delegates, UMG, SaveGame.  
**Debugging Exercises:** A client incorrectly reading GameMode; score placed on Pawn and lost at respawn; runtime Component not registered; duplicate delegate binding after reconstruction; stale timer after EndPlay.  
**Performance Checks:** Compare event-driven UI with per-frame polling; record enabled Actor/Component ticks; ensure interaction does not scan all Actors every frame.  
**Interview Questions Unlocked:** Actor versus Component, Pawn versus Character, Controller versus Pawn, GameMode versus GameState, PlayerController versus PlayerState, constructor versus BeginPlay, subsystem placement.  
**Stretch Goals:** Seamless-travel PlayerState copy, inventory/equipment component, local-player subsystem for UI flow, automation test for respawn state.  
**Version / Plugin Caveats:** UE5.3–UE5.6. Core systems are baseline; exact seamless-travel and lifecycle internals require version-specific testing. [SRC-EPIC-010] [SRC-EPIC-013] [SRC-EPIC-015]

### Project 1/9 Enhanced Input Architecture Extension

**Goal:** Replace device-level input handling with a semantic, local-player-aware input architecture that survives remapping, UI modal screens, possession changes, respawn, optional GAS activation and multiplayer authority checks.

**Minimum Features:** semantic Input Actions; common/on-foot/menu/vehicle/radial Mapping Contexts; LocalPlayer input coordinator; C++ action bindings; modifiers for WASD/axis/inversion; press/hold/release triggers; remapping conflict policy; UI modal input isolation; ability or mock-ability adapter; network-safe intent route.

**Implementation Steps:**

1. Define semantic actions: Move, Look, Jump, Interact, Fire, FireRelease, AbilityPrimary, Confirm, Cancel, OpenMenu and VehicleExit. Do not name actions after physical keys/buttons.
2. Author `IMC_Common`, `IMC_OnFoot`, `IMC_Menu`, `IMC_Vehicle` and `IMC_RadialMenu`. Assign explicit priority constants and document who owns each add/remove.
3. Implement a LocalPlayer or PlayerController input coordinator that adds/removes contexts idempotently and logs local player, context, priority, requester and reason.
4. Bind actions in C++ through `UEnhancedInputComponent`. Use `Started` for one-shot edges, `Triggered` for continuous movement/look and `Completed`/`Canceled` for release/cancel semantics.
5. Use modifiers to combine WASD/arrow keys into one 2D Move action, apply inversion/sensitivity settings to Look and verify value types.
6. Add triggers for hold-to-charge, tap-to-interact and one chorded or blocker case. Record which trigger event should commit gameplay.
7. Add a remapping UI with per-local-player persistence, conflict detection and restore defaults. Update UI glyphs/prompts after remap.
8. Open a menu/modal/radial wheel and prove gameplay actions do not leak through while Confirm/Cancel still route to UI.
9. Integrate with a mock ability adapter or GAS slice: input press/release maps to ability spec handle/tag, rebuilds after grants/equipment/respawn and never stores physical key names inside the ability.
10. For networked actions, send semantic intent through the owning route and validate on server. Do not send every key/tick as reliable RPC.
11. Inject failures: duplicate binding after respawn, leaked Vehicle context, wrong value type, `Triggered` spam on one-shot action, UI focus intercept and conflicting remap.
12. Produce an input routing diagram and log packet for gameplay, menu, vehicle, respawn and split-screen/local-player case if supported.

**Debugging Exercises:** action asset missing; wrong value type; mapping context on wrong LocalPlayer; priority collision; trigger event fires every tick; duplicate binding after possession; context not removed; CommonUI/direct input-mode fight; remap conflict; client-only authoritative mutation.  
**Performance Checks:** handler call count per frame, context add/remove frequency, duplicate binding count, custom modifier/trigger cost, UI prompt rebuild cost, network request frequency and trace of expensive downstream work.  
**Interview Questions Unlocked:** Enhanced Input mental model, Input Action value types, Mapping Context lifecycle, modifiers/triggers, UI/CommonUI boundary, remapping policy, GAS adapter, network-safe intent, split-screen local-player design.  
**Version / Plugin Caveats:** Enhanced Input APIs, CommonUI integration, remapping settings and platform glyph support are branch/project sensitive. Verify exact UE5.3–UE5.6 target behaviour and do not treat local input as gameplay authority. [SRC-INPUT-001] [SRC-INPUT-002] [SRC-INPUT-003] [SRC-INPUT-004] [SRC-INPUT-005] [SRC-UI-010] [SRC-UI-011] [SRC-GAS-010]

## Project 3: Networked Mini Game

**Goal:** Build a small authoritative objective game that proves property replication, RPC ownership, relevancy, dormancy, and loss-aware debugging rather than merely showing two editor windows.  
**Target Roles:** Gameplay, networking, engine generalist, technical design.  
**Topics Covered:** Listen/dedicated modes, authority, roles, ownership, replicated state, OnRep, Server/Client/Multicast RPCs, reliability, relevancy, dormancy, profiling, prediction awareness.  
**Minimum Features:** Server-owned health and score; replicated GameState match phase/team score; PlayerState score; Server RPC interaction/fire request; targeted Client RPC feedback; unreliable multicast cosmetic; OnRep UI; owner-only data; one dormant replicated objective; one relevance-limited pickup; ownership bug exercise.  

**Implementation Steps:**

1. Write an authority/audience table for every variable and event before implementation.
2. Run dedicated server plus two clients. Add structured logs containing world, net mode, local role, owner, controller, and Actor name.
3. Spawn all replicated gameplay Actors on the server. Prove that a client-spawned “replicated” pickup remains local.
4. Implement server-authoritative health and PlayerState score. Replicate Health with OnRep and drive UI/events locally.
5. Route an interaction request through the owning Pawn/Controller. Validate target, distance, state, cooldown/rate, and authority on server.
6. Add a reliable targeted Client RPC for an infrequent private rejection message and an unreliable multicast for replaceable cosmetics.
7. Add a replicated match-state struct so phase/time/objective data forms one coherent transition rather than relying on OnRep order.
8. Make a placed objective initially dormant. Wake/flush before a state change; intentionally reverse the order and observe/diagnose the failure.
9. Limit pickup relevancy and observe client destruction/recreation as it leaves/enters relevance. Contrast with dormant persistence.
10. Add owner-only private state and verify another client cannot receive it.
11. Simulate latency, jitter, and loss. Record which unreliable events may disappear and ensure durable state still converges.
12. Capture network traffic and rank Actor/property/RPC contributors before optimising.

**Key Unreal Systems:** Gameplay Framework, generic replication, `GetLifetimeReplicatedProps`, RepNotify, RPCs, Actor roles/ownership, relevancy, dormancy, CharacterMovement awareness, Network Profiler/Insights.  
**Debugging Exercises:** Server RPC on unowned world Actor; missing `DOREPLIFETIME`; client-side authoritative mutation; multicast called from client; reliable RPC spam; separate OnRep ordering assumption; dormant mutation before flush; always-relevant bandwidth spike.  
**Performance Checks:** Bandwidth per connection, RPC/property frequency, relevant Actor count, dormancy savings, update frequency, server frame/network processing cost.  
**Interview Questions Unlocked:** Authority versus ownership, why RPC fails, property versus RPC, OnRep ordering, reliability, relevancy versus dormancy, autonomous versus simulated proxy, prediction/correction, Replication Graph/Iris awareness.  
**Stretch Goals:** Fast Array inventory, replicated UObject subobject, custom CharacterMovement saved move, Replication Graph node, Iris comparison branch where supported.  
**Version / Plugin Caveats:** UE5.3–UE5.6. Generic replication baseline. Replication Graph and Iris are project/version/configuration-dependent; verify exact target support. [SRC-NET-001] [SRC-NET-004] [SRC-NET-008]

### Project 3 Specialist Replication Extension

**Goal:** Extend the mini game with component replication, dynamic UObject subobjects, Fast Array delta replication and scale-aware replication policy, while proving every branch with dedicated-server evidence.  
**Minimum Features:** Static replicated component; dynamic server-created replicated component; dynamic replicated UObject subobject; safe registered-list add/remove; Fast Array inventory/status list; target-branch network debug CVar evidence; Replication Graph adoption/rejection memo; Iris config/status check.

**Implementation Steps:**

1. Add a static `UActorComponent` to the Player/Pawn with one replicated property and one component RPC. Prove the owning Actor and component both need replication enabled.
2. Add a server-created dynamic component. Log server creation, client creation, property replication and destruction convergence. Prove a locally client-created component does not become authoritative replicated state.
3. Create a dynamic `UObject` subobject owned by a replicated Actor or Component. Implement networking support, lifetime props and one replicated field.
4. Replicate a pointer/reference to the subobject only if clients need it. Add `OnRep`/validity handling that tolerates null before mapping.
5. Use the target branch's registered subobject path if available. Remove the subobject from the registered list before deletion; in a throwaway branch, skip removal and capture the failure or validation.
6. Add a small owner-only inventory or status collection. Implement a simple replicated array first, then a Fast Array version with stable IDs and centralised authoritative mutation.
7. Test add, change, remove, reorder, reused ID, late join, packet loss and client UI stale selection. Compare bytes/frequency and complexity between simple and Fast Array variants.
8. If push model is enabled/supported in the target project, intentionally miss a dirty mark and use validation to catch it.
9. Classify every replicated Actor into policy buckets: spatial, owner-only, always relevant, team relevant, dormant/static or carried/attached. If scale justifies it, prototype a tiny Replication Graph node; if not, write a rejection memo with profiler evidence.
10. Inspect Iris status/configuration. Do not implement Iris-specific code unless the target branch/project actually enables it; if enabled, document registered subobject and replication-fragment requirements.

**Debugging Exercises:** dynamic subobject pointer null before mapping; stale registered subobject after GC; component condition hides component subobject; forgotten Fast Array dirty mark; reused item ID; stale Replication Graph spatial bucket; always-relevant bucket explosion; copied Iris assumption in Generic replication branch.  
**Performance Checks:** bytes per item mutation, whole-array versus Fast Array traffic, component/subobject count overhead, per-connection relevant Actor count, server net update/list-building time, dormant/static bucket savings, debug-CVar/log overhead.  
**Interview Questions Unlocked:** component replication, UObject subobjects, registered subobjects list, Fast Array identity/dirty marking, push model, Replication Graph adoption, Iris honesty, network debug CVars.  
**Version / Plugin Caveats:** Actor Component replication is UE5.6 source-backed. Object Replication, Iris and network debug CVar pages are maintained beyond the target and must be pinned in UE5.3-UE5.6 source before exact implementation claims. [SRC-NET-011] [SRC-NET-012] [SRC-NET-013] [SRC-NET-014] [SRC-NET-015] [SRC-NET-016]

### Project 3 Movement Prediction and Lag Compensation Extension

**Goal:** Extend the mini game with one custom predicted movement feature and one bounded server-rewind weapon so the project proves responsiveness, server authority, correction handling, fairness policy and profiling evidence.

**Minimum Features:** predicted sprint saved-move flag; predicted dash/slide or similar custom movement edge; correction instrumentation; dedicated-server packet lag/loss test; server-side hitbox/history ring buffer; lag-compensated hitscan request; timestamp/aim/fire-rate validation; bounded rewind/fairness memo.

**Implementation Steps:**

1. Branch from the working Project 3 baseline. Record the exact UE version and source/header locations used for `UCharacterMovementComponent`, `FSavedMove_Character` and client prediction data.
2. Implement predicted sprint first. Capture the wants-to-sprint state in a saved move, encode it compactly, restore it for replay and prevent move combining across sprint transitions.
3. Add server validation for sprint resources/tags/state. Log when the server accepts, rejects or corrects the prediction.
4. Add a predicted dash, slide or short wall-run. Store the input edge and compact movement-affecting state; validate cooldown, direction, resources, movement mode, floor/wall evidence and collision.
5. Deliberately implement one broken version: delayed replicated bool only, missing `CanCombineWith`, or direct transform dash. Capture the correction symptoms before fixing it.
6. Add correction metrics: feature name, correction count, distance, local role, movement mode, compressed/custom flags and rejection reason.
7. Run dedicated server plus two remote clients under clean LAN, 100 ms lag, jitter and packet loss. Compare before/after correction counts and subjective feel.
8. Build a bounded server-side history buffer for target hitboxes/bounds. Store value snapshots with server time/sequence, root transform, compact hitboxes and target state/generation.
9. Add a hitscan weapon where the client predicts presentation and sends intent/timing/aim evidence through an owned Server RPC. The server validates ownership, weapon state, ammo, fire rate, timestamp window, aim plausibility and target state.
10. Implement server rewind query and authoritative trace. Reject unbounded/old timestamps, impossible aim deltas and stale respawn generations.
11. Define the occlusion policy: current blockers, timestamped blockers, or no blocker rewind. Document why the policy fits the game.
12. Compare three modes: no lag compensation, bounded rewind and deliberately excessive rewind. Record one case that feels fair to shooter, one that feels unfair to target and one exploit-style request that is rejected.
13. Profile snapshot memory/allocation, rewind query/trace cost, movement custom payload, fire request traffic and replicated damage/death traffic.
14. Produce a final evidence packet: timelines, correction table, network traffic capture, memory/CPU table, fairness memo and branch-sensitive API notes.

**Debugging Exercises:** sprint jitter from delayed replicated bool; dash edge merged by move combining; client predicts dash rejected by cooldown; root motion and CharacterMovement both move capsule; rewind buffer stores mutable current transforms; old timestamp hit; stale target after respawn; client-supplied hit result ignored by authority.  
**Performance Checks:** correction count/distance per feature, saved-move allocation/combine/replay, custom movement payload bytes, snapshot memory, steady-state allocations, rewind query/trace cost, fire request bandwidth, false local hit marker rejection rate.  
**Interview Questions Unlocked:** CharacterMovement prediction pipeline, saved-move extension, custom sprint/dash debugging, prediction versus reconciliation, server rewind, lag-compensated hitscan, rewind security, rollback terminology, profiling prediction and history buffers.  
**Version / Plugin Caveats:** CharacterMovement saved-move and prediction data APIs are branch-sensitive. Epic API pages are used as anchors; exact signatures must be verified in UE5.3-UE5.6 target source. Server rewind design here is a game-specific pattern, not a single generic UE subsystem. [SRC-NET-009] [SRC-NET-017] [SRC-NET-018] [SRC-NET-019] [SRC-NET-010]

### Project 3 NetworkPrediction Rollback Proof Extension

**Goal:** Prove whether the target branch's `NetworkPrediction` plugin is suitable for one bounded rollback/resimulation feature, with source-backed APIs, forced reconciles, cue dedupe and measured buffer costs.  
**Minimum Features:** target-branch plugin evidence packet; minimal counter model; rollback dash model; input/sync/aux/cue state; forced reconcile; invalid command rejection; side-effect ledger; cue dedupe; smoothing on/off comparison; CPU/memory/bandwidth report.

**Implementation Steps:**

1. Record the exact engine version, plugin status, module list, header/source locations, build configuration and debug CVars available in the target branch.
2. Build a toy counter model first: compact input, integer sync state, versioned aux state and one threshold cue. Force divergence and prove replay.
3. Build a rollback dash model with quantised move input, dash press, location/velocity/phase sync state, movement tuning aux state and dash-start cue.
4. Run dedicated server plus at least one autonomous client under clean LAN, 100 ms RTT, jitter and packet loss.
5. Inject client-only acceleration bias and server-only blocker. Use reconcile logs to identify the restored frame, field diff, replay window and final convergence.
6. Reject invalid dash commands for cooldown/resource/state/range/timestamp reasons. Log structured rejection reasons and prove client presentation recovers.
7. Create a side-effect ledger. Keep damage/rewards/analytics outside the resimulated tick and prove VFX/audio/camera cues dedupe under forced replay.
8. Add aux-state versioning for equipment or movement tuning. Swap equipment during a pending prediction window and prove old-frame replay uses correct aux data or rejects pending inputs.
9. Disable/enable smoothing or interpolation where the target build supports it. Compare visible correction with reconcile frequency so smoothing does not hide correctness failure.
10. Measure input/sync/aux/cue buffer memory, replay CPU burst, cue history cost and network traffic at 1/16/64/128 model instances.
11. Compare against server-authoritative command, CharacterMovement saved moves, server rewind and a custom lightweight prediction path. Recommend the smallest architecture that satisfies the product requirement.

**Debugging Exercises:** reconcile every frame; duplicated dash cue; replay with new weapon tunings; stale model instance after respawn; Actor replicated transform fighting model sync state; hidden correction spam under smoothing; client-provided target accepted as truth.  
**Performance Checks:** command payload size/rate, correction payload size, replay frames per correction, per-instance buffer memory, cue dedupe table size, model tick/replay CPU, correction-storm worst case, network capture.  
**Interview Questions Unlocked:** what full rollback means, what NetworkPrediction proves/does not prove, input/sync/aux/cue state boundaries, cue replay safety, aux-state versioning, reconcile debugging, rollback rejection criteria.  
**Version / Plugin Caveats:** The maintained Epic API page is used as public vocabulary evidence only; target UE5.3-UE5.6 source must prove exact plugin support and signatures before implementation. [SRC-NET-020] [SRC-NET-001] [SRC-NET-003] [SRC-NET-006] [SRC-NET-009] [SRC-NET-010]

### Project 3 Unified Networking Target-Proof Extension

**Goal:** Produce one branch-pinned networking evidence packet covering saved moves, dynamic subobjects, Fast Array, RepGraph adoption/rejection, Iris status and NetworkPrediction compile proof or rejection.  
**Minimum Features:** source/config audit; saved-move sprint and dash; dynamic replicated subobject lifecycle; simple array versus Fast Array comparison; RepGraph scale profile; Iris enabled/disabled proof; NetworkPrediction toy/dash compile proof or rejection memo; packet-condition matrix.

**Implementation Steps:**

1. Create `networking_source_audit.md` with target file paths/signatures for every copied API and plugin/config status for RepGraph, Iris and NetworkPrediction.
2. Run predicted sprint and dash saved-move labs with broken-before/fixed-after correction evidence.
3. Create, replicate, map, remove and GC-test a dynamic UObject subobject; include null-before-mapped client behaviour.
4. Compare simple replicated array and Fast Array for 10/100/1000 item cases, including add/change/remove/reorder/reused-ID failures.
5. Profile generic replication first; adopt RepGraph only if relevant Actor/list-construction cost is proven. Otherwise write a rejection memo.
6. Prove Iris status. If disabled, stop at status proof; if enabled, test registered subobjects and any target-required UObject fragment registration.
7. Compile and run a NetworkPrediction toy/dash proof or write a target-source rejection memo.
8. Run all implemented paths under a shared 0 ms, 100 ms jitter and 200 ms/loss packet matrix with join-in-progress.

**Acceptance Criteria:** every implemented feature has source proof, a failing injected case, a fixed run, dedicated-server evidence and a profiler/log artefact. Any unsupported feature is labelled as rejected/not available with target-branch evidence rather than omitted silently.  
**Version / Plugin Caveats:** This extension is source-sensitive and should be run only in a real target branch. The curriculum provides proof structure, not target-engine signatures. [SRC-NET-009] [SRC-NET-012] [SRC-NET-013] [SRC-NET-014] [SRC-NET-015] [SRC-NET-020]

## Project 2: AI Combat Sandbox

**Goal:** Build an authoritative combat AI whose sensing, memory, decisions, spatial queries, navigation, and local avoidance can each be inspected and deliberately broken.  
**Target Roles:** AI/gameplay, gameplay, engine generalist, technical design.  
**Topics Covered:** AIController/Pawn, Behaviour Tree/Blackboard, tasks/services/decorators/observer aborts, AI Perception, EQS, Recast NavMesh, MoveTo/path following, StateTree comparison, avoidance, AI Debugger, Visual Logger, performance LOD.  
**Minimum Features:** Patrol, perceive, investigate last known location, chase, attack, lose/forget target; native or Blueprint BT Tasks with correct finish/abort; EQS attack position; NavMesh movement; one nav link/modifier; RVO or Detour Crowd comparison; server-authoritative operation.  

**Implementation Steps:**

1. Draw the pipeline from stimuli to movement and define every Blackboard key's writer, reader, type, and clearing policy.
2. Create an AIController that possesses the enemy Pawn and starts the Behaviour Tree on authority.
3. Configure sight/hearing perception and a stimuli source. Convert perception events into current target, last-known location, stimulus age, and alert state.
4. Build priority branches: Combat, Investigate, Patrol/Idle. Use Decorators and Observer Aborts rather than high-frequency condition Services.
5. Implement a latent attack/move task that completes and aborts cleanly. Deliberately omit completion once and diagnose the stuck branch.
6. Build EQS attack-position selection with cheap distance/filter tests before line-of-sight/path tests. Inspect scores in the EQS debugger.
7. Add a Nav Modifier and Nav Link representing a dangerous zone and discontinuity. Verify cost/path changes.
8. Create a MoveTo failure matrix: no NavMesh, invalid goal, wrong agent radius, blocked link, movement disabled, overlapping request, authority mismatch.
9. Compare no avoidance with either RVO or Detour Crowd; document why both should not be enabled together.
10. Record AI Debugger and Visual Logger evidence for one transient perception/path bug.
11. Spawn increasing agent counts. Stagger services/EQS, reduce perception, and add distance/significance update tiers; measure game-thread impact.
12. Sketch the equivalent high-level behaviour as StateTree and a simple FSM; defend which model best fits this agent.

**Key Unreal Systems:** AIModule, NavigationSystem, Behaviour Trees, Blackboard, AI Perception, EQS, StateTree awareness, CharacterMovement/crowd avoidance, Visual Logger.  
**Debugging Exercises:** Missing stimuli registration; lost-sight memory cleared too soon; incorrect observer abort; latent task never finishes; EQS scores without hard reachability filter; wrong nav agent; MoveTo replaced; RVO out-of-nav-bounds.  
**Performance Checks:** Agent count, service frequency, perception pair/range, EQS candidates/traces/path tests, nav rebuilds, path requests, game-thread time, significance tiers.  
**Interview Questions Unlocked:** BT node roles, observer aborts, StateTree versus BT/FSM, perception memory, EQS architecture, MoveTo failure workflow, pathfinding versus avoidance, A*, AI profiling.  
**Stretch Goals:** Custom sense, Smart Object interaction, squad coordinator, utility scoring, custom StateTree task/evaluator, MassEntity crowd comparison.  
**Version / Plugin Caveats:** UE5.3–UE5.6. Verify StateTree and EQS plugin/configuration status in the target project; maintained docs may show later UE5 UI/APIs. [SRC-AI-002] [SRC-AI-003] [SRC-AI-008]

### Project 2 Smart Objects and StateTree Extension

**Goal:** Turn the AI sandbox into an affordance-driven activity system where agents discover, claim, use, abort and release world interactions with traceable StateTree behavior.  
**Minimum Features:** three Smart Object affordances, at least five total slots, activity/user tags, StateTree `Acquire -> Reach -> Use -> Exit` lifecycle, explicit claim/use/release logging, server-authoritative validation option, two-agent race test, failure injection and 1/20/100-agent query profiling.  

**Implementation Steps:**

1. Create rest chair, cover point and repair terminal affordances. Give each a definition, slot transforms, activity tags, user requirements and behavior reference.
2. Add an AI need/activity selector that chooses `Rest`, `TakeCover` or `Repair` from state rather than directly naming a world actor.
3. Query matching Smart Objects with an explicit origin, radius, filters and user data. Log candidate count and rejection categories.
4. Claim a slot before movement. Log claimant, slot identity and failure reason. Prove "found" and "claimed" are different states.
5. Build a StateTree interaction flow: `AcquireClaim`, `ReachSlot`, `UseBehavior`, `ReleaseAndExit`.
6. Pass slot/object/user identity through StateTree context/parameters/task outputs with clear names.
7. Validate reachability with NavMesh projection/path result before committing to use.
8. Execute a visible behavior: sit/rest, crouch cover, or repair loop. Keep animation/ability/component work outside the Smart Object query code.
9. Release the claim on success, MoveTo fail, StateTree failure, combat interruption, agent death/despawn, object disabled and object destroyed.
10. Run two agents racing for one slot. The loser must retry, choose another slot, or fail cleanly without overlapping use.
11. In a networked variant, let client UI propose interaction but let the server validate claim and durable use outcome.
12. Profile synchronous versus staggered queries at 1, 20 and 100 agents. Record query frequency, candidate count, path checks, StateTree task count and game-thread time.

**Debugging Exercises:** candidate found but claim fails; claim succeeds but path is unreachable; helper StateTree task finishes early and transitions out of Reach; object disabled mid-use; claim leaked after agent death; client-only use appears locally but server rejects; two agents animate into the same slot because alignment is wrong.  
**Performance Checks:** query radius/candidate count, failed-query backoff, path validation count, monitored transition frequency, evaluator/task allocation, staggered activity update cost, Actor AI versus Mass/crowd suitability.  
**Interview Questions Unlocked:** Smart Object lifecycle, slot/definition distinction, query versus claim, release-on-abort policy, StateTree data flow, StateTree versus BT/FSM, multiplayer authority for interactions, Smart Object debugging and scaling.  
**Version / Plugin Caveats:** Smart Objects, StateTree, Mass integration and debugger categories are plugin/project/branch-sensitive. Verify exact UE5.3-UE5.6 API signatures and behavior in target headers before implementation. [SRC-AI-010] [SRC-AI-011] [SRC-AI-012] [SRC-AI-013] [SRC-AI-014] [SRC-AI-015] [SRC-NET-004]

### Project 2 / Project 5 Custom AI Senses and MassAI Extension

**Goal:** Add one custom perception channel and one MassAI crowd activity so the AI sandbox proves evidence collection, memory decay, Mass simulation, representation LOD and Smart Object/StateTree bridge correctness.

**Minimum Features:** custom scent/noise/radio/magic sense contract; explicit stimulus memory merge/decay policy; listener/source registration failure tests; 1/20/100-agent profile; Mass or Mass-like crowd lane/activity prototype; Smart Object claim/use/release from crowd activity; Actor promotion/demotion dual-writer test; evidence packet using the workbook template.

**Implementation Steps:**

1. Choose one sense that genuinely belongs in perception rather than a direct gameplay event. Define event payload, config values and memory keys.
2. Implement or sketch the custom `UAISense`/`UAISenseConfig` path with target-branch API verification notes. Keep decision and Blackboard writes outside the sense.
3. Add deterministic tests for source registration, listener config, event acceptance, discarded reasons and stimulus memory projection.
4. Build the "remembers forever" scenario: see, lose sight, hear or smell, investigate, decay, forget and reacquire.
5. Profile event production, listener filters, trace counts, callbacks, memory updates and game-thread time at 1, 20 and 100 agents.
6. Extend Project 5 with a lane/activity crowd: route/lane state, movement/avoidance processor, simulation LOD and Actor/ISM/no representation counts.
7. Add one shared activity such as kiosk queue, bench rest or hazard flee. Store durable activity/claim state in fragments or equivalent data, not transient task locals.
8. Bridge the crowd activity to a Smart Object lifecycle: candidate, claim, reach, use, release. Capture a 20-agent/5-slot race.
9. Promote one nearby entity to an Actor for rich interaction. Inject and then fix a dual-writer transform bug.
10. Produce the final evidence packet from [[ue_ai_senses_massai_workbook]].

**Debugging Exercises:** sense never fires; source destroyed before update; wrong World; affiliation filter rejects everything; failed sight clears all evidence; event flood causes traces spike; entity found candidate but failed claim; Mass entity stuck in lane; StateTree waits forever after movement; Actor promotion jitter from dual transform ownership.  
**Performance Checks:** custom sense events/s, discarded/merged events, listeners considered, traces/s, callbacks/s, memory projections/s, Mass processor time, structural commands, lane density, representation counts, Actor promotions/demotions, Game/Draw/GPU split.  
**Interview Questions Unlocked:** custom AI sense design, perception memory policy, stimulus expiry, custom sense debugging, MassAI versus Actor AI, ZoneGraph/lane reasoning, Mass StateTree activity, Smart Object claim from Mass, Actor promotion contract, crowd profiling.  
**Version / Plugin Caveats:** Custom AI sense APIs, ZoneGraph, Mass Crowd, Mass StateTree, Smart Objects and Mass representation/debugging are branch- and plugin-sensitive. Treat code as schematic until verified in target UE5.3-UE5.6 source. [SRC-AI-004] [SRC-AI-016] [SRC-AI-017] [SRC-AI-018] [SRC-AI-019] [SRC-MASS-001] [SRC-MASS-002] [SRC-MASS-009] [SRC-MASS-010]

## Project 4: Rendering and Optimisation Lab

**Goal:** Build a reproducible scene and use evidence to diagnose/fix one game-thread bottleneck, one hitch/memory issue, and one GPU-pass bottleneck.  
**Target Roles:** All UE roles; deeper extension for rendering/engine/technical art.  
**Topics Covered:** Frame budgets, `stat unit/game/gpu`, Timing/Memory Insights, custom stats, Tick/events, allocations/GC, instancing/LOD/Nanite comparison, material cost, translucency/overdraw, scalability/device profiles.  
**Minimum Features:** Fixed benchmark camera/path; many meshes/materials; Actor-versus-ISM/HISM setup; expensive material; translucent overlap; LOD/Nanite comparison; high-frequency gameplay workload; allocation/GC spike; at least three scalability levels.  

**Implementation Steps:**

1. Define target hardware, 60 FPS budget, packaged build configuration, resolution, quality, camera path, warm-up, and sample duration.
2. Capture baseline `stat unit/unitgraph`, Timing Insights, Memory Insights, `stat gpu`, and a GPU Visualiser snapshot.
3. Create a game-thread issue using many polling Actor/Component ticks. Prove call count/time, replace with event/dirty batching or manager updates, and recapture.
4. Create short-lived allocation/UObject churn and a bulk-destruction GC spike. Query short-lived/growth allocations and correlate GC timing. Compare bounded reuse/pooling including retained-memory cost.
5. Compare individual StaticMesh Actors/Components with ISM/HISM and appropriate Nanite/LOD configurations. Record Game/Draw/GPU/memory, not only FPS.
6. Build a material with controllable instruction/texture/WPO cost and a large translucent overdraw case. Identify the actual GPU pass and test resolution sensitivity.
7. Change one variable per experiment. Include at least one plausible hypothesis that the trace disproves.
8. Add custom scoped stats/counters around project work and use callers/callees to find duplicate scheduling.
9. Define scalability/device-profile overrides and test lowest/highest tiers for quality, memory, and performance.
10. Produce a one-page regression benchmark with thresholds and capture instructions.

### Rendering experiment matrix

Run these as separate captures so one bottleneck does not hide another:

| Experiment | Controlled change | Prediction to record before capture | Required evidence |
|---|---|---|---|
| Submission | Independent Components → ISM/HISM, same mesh count/view | Draw/RHI should fall more than pixel-scaled passes | Insights render lanes, draw/primitive counts, GPU passes |
| Geometry | Traditional LOD → Nanite where supported, same material/view | Geometry/visibility path changes; material/post cost should not disappear | Nanite views, Draw/RHI, depth/base/shadow timings, memory |
| Pixel shading | Cheap → expensive opaque full-screen material | Base pass should scale with resolution/material, not object count | BasePass timing, resolution sweep, material stats |
| Overdraw | One → multiple translucent full-screen layers | Translucency should rise with coverage/layers | Translucency timing, Shader Complexity/Quad Overdraw |
| Shadows | Static → moving → WPO casters under VSM | Cache invalidation and shadow work should rise with affected pages | VSM cached-page/invalidation views, shadow pass timing |
| Lumen | Valid → deliberately poor scene representation | Visual artefact should correlate with a Lumen representation view | Lumen overview views, GI/reflection pass timings |
| HLOD/culling | Fine objects → distant HLOD proxy | distant object/submission cost should fall; culling granularity changes | draw/primitive counts, pop/quality notes, memory/streaming |

For every row, include one “feature off” localisation result and one requirement-preserving candidate fix. Feature-off is not accepted as the final optimisation unless the feature itself is outside the product requirements.

### Required GPU diagnosis report

1. State build, engine commit/version, hardware/driver/RHI, resolution/internal percentage, scalability, VSync/frame cap/dynamic resolution, and warm-up/camera path.
2. Show that GPU rather than Game/Draw/RHI/display pacing limits the selected case.
3. Name the dominant/regressed pass and its before value/distribution.
4. Write the causal hypothesis as **pass + mechanism + controlled prediction**.
5. Show one isolation test and explain what it rules in/out.
6. Apply the smallest visual-requirement-preserving change.
7. Re-run the same capture and report P50/P95/P99, named pass, Game/Draw/RHI, memory, and quality outcome.
8. Record whether the result generalises to at least one different view and hardware/quality tier.

**Key Unreal Systems:** Stats System, Unreal Insights Timing/Memory, GPU Visualiser, RenderDoc awareness, scalability, device profiles, ISM/HISM, LOD/HLOD, Nanite, materials, GC.  
**Debugging Exercises:** VSync misclassification; editor-only overhead; async completion hitch; Tick replaced by equal-frequency timer; pool memory regression; Draw-bound versus GPU-bound confusion; dynamic resolution hiding cost.  
**Performance Checks:** Median/P95/P99 frame time, worst hitch, Game/Draw/RHI/GPU, allocations/live memory, UObject count/GC, draw/primitive counts, resolution sensitivity, quality regression.  
**Interview Questions Unlocked:** CPU versus GPU bound, stat unit interpretation, Insights workflow, memory leaks/churn, GC hitch, Tick optimisation, draw calls versus shader/fill/bandwidth cost, deferred versus forward, Nanite/LOD/ISM/HLOD selection, Lumen/VSM diagnosis, RDG awareness, scalability strategy.  
**Stretch Goals:** RenderDoc capture; PSO hitch exercise; automated trace capture; platform profiler; thermal/power run; minimal named RDG compute/full-screen pass inspected in RDG Insights; forward/deferred comparison branch.  
**Version / Plugin Caveats:** UE5.3–UE5.6. Tool UI, pass names, and Nanite/Lumen/VSM/RDG behaviour evolve; record exact engine commit, RHI, platform and active tracing paths. Current maintained documentation may expose post-5.6 support. [SRC-PERF-001] [SRC-PERF-003] [SRC-PERF-006] [SRC-RENDER-003] [SRC-RENDER-005] [SRC-RENDER-006] [SRC-RENDER-007]

### Project 4 Specialist Rendering Extension

**Goal:** Convert the broad optimisation lab into an implementation-facing renderer investigation covering shader permutations, plugin/global shaders, PSO first-use hitches, mesh draw command pressure, render targets, scene captures and external GPU captures.

**Minimum Features:** material permutation audit; minimal source-branch global shader or RDG pass; packaged PSO first-use hitch reproduction and cache/precache proof; mesh draw stress comparison; render target/scene capture budget matrix; one RenderDoc or platform GPU capture.

**Implementation Steps:**

1. Record engine version/commit, RHI/API, platform, packaged configuration, scalability, resolution, DDC state and shader/PSO project settings.
2. Build two material families: one bounded parent-material set and one intentionally unbounded static-switch variant set. Record shader compile count/time where available, cook/shader library impact, material stats, runtime pass time and PSO diversity.
3. In a source-capable branch, create a minimal plugin/global shader or RDG pass that writes a constant/debug output to a render target. Verify shader directory mapping, module load phase, permutation predicate, parameter struct dependencies and packaged output.
4. Deliberately break the shader path once: wrong entry point/path, missing permutation, missing RDG dependency or invalid lifetime. Capture the validation/log symptom and fix.
5. Create a first-use hitch path using a rare VFX/UI/scene-capture material. Reproduce in a packaged build with cold caches, collect/log the missing PSO through target workflow, package/precache it, and verify improvement on a fresh run.
6. Compare 1000 independent mesh Components, ISM, HISM, Nanite-supported geometry and a version that calls render-state-dirty updates every frame. Record Draw/RHI, render-thread tracks, primitive/draw counts, GPU pass timing, culling quality and memory.
7. Build three render target use cases: minimap, inventory thumbnail and security camera/mirror. Vary resolution, format, mip generation, update frequency, show flags/show-only actors and active count.
8. Add one SceneCaptureCube variant and document why its six-view cost model is different from SceneCapture2D.
9. Include scene capture/VFX/UI paths in PSO collection. Prove that the main camera's cache coverage did or did not include them.
10. Take one RenderDoc/PIX/Nsight/Xcode/console capture, depending on target support. Mark the custom pass/effect, inspect pipeline state/resources/shaders and translate one capture finding into a normal Unreal profiler before/after.
11. Produce a final table: shader compile/permutation data, PSO hitch evidence, mesh draw comparison, render-target memory/cost, scene-capture budget and capture-tool screenshots/notes.

**Debugging Exercises:** shader compiles in editor but fails packaged; global shader outputs black; RDG missing dependency/lifetime violation; PSO hitch hidden by warm DDC; first-use scene-capture material not in PSO cache; one mesh produces many pass draws; per-frame render-state invalidation; many per-actor render targets; cube capture budget blow-up.  
**Performance Checks:** shader compile count/time, DDC/shader library evidence, PSO miss/hitch timing, Draw/RHI/render-thread time, primitive/draw counts, GPU pass timing, render target memory, scene capture update cost, capture-tool overhead.  
**Interview Questions Unlocked:** shader permutation versus runtime shader cost, PSO cache workflow, shader plugin/global shader extension, mesh draw pipeline, render target budgeting, scene capture cost, RenderDoc/platform capture escalation.  
**Version / Plugin Caveats:** Shader plugin/global shader APIs, PSO cache workflows, mesh draw internals and capture tooling are source/RHI/platform-sensitive. Verify all commands/signatures in the target branch and packaged platform. [SRC-RENDER-012] [SRC-RENDER-013] [SRC-RENDER-014] [SRC-RENDER-015] [SRC-RENDER-016] [SRC-RENDER-017] [SRC-RENDER-018] [SRC-RENDER-019]

### Project 4 Target Performance Gate Extension

**Goal:** Convert the optimisation lab into a repeatable packaged-build performance gate with CSV telemetry, short causal traces, LLM/memory checkpoints, device-profile proof and actionable thresholds.

**Minimum Features:** three deterministic scenarios; packaged target run; two device profiles; CSV capture; one Timing/Memory Insights investigation; LLM/tag budget checkpoints; applied-CVar proof; performance gate proposal.

**Implementation Steps:**

1. Define three scenarios: UI/front-end churn, combat/VFX peak and streaming/travel. Each scenario needs a fixed seed, camera/input path, warm-up period and sample window.
2. Record build metadata: engine/project revision, branch, target/config/platform, device, power/thermal state, resolution, scalability and active device profile.
3. Run each scenario in a packaged build under low and high quality/device profiles. Dump or otherwise prove the applied CVars/profile inheritance.
4. Collect CSV output for every run. Track frame percentiles, hitch counts, Game/Draw/GPU, memory/LLM totals and scenario-specific counts.
5. Capture a short Unreal Insights trace only around one worst CPU hitch, not the whole run.
6. Capture Memory Insights and LLM checkpoints across a repeated open/close or travel cycle. Identify one capacity issue, one growth/retention issue and one churn issue.
7. Create three deliberate regressions: duplicate CPU update, retained UI/content memory and GPU/render-target or VFX cost. Prove the CSV/device matrix catches them.
8. For each failed gate, escalate to the smallest causal capture: Timing Insights, Memory Insights, GPU Visualiser/external GPU capture or platform profiler.
9. Write threshold rules with tolerance bands and owners, for example P95 frame time, hitch count, LLM tag growth or first-interactive time.
10. Produce an evidence packet: CSVs, trace files, screenshots/CVar dumps, before/after table, false-positive notes and the final gate policy.

**Debugging Exercises:** average FPS hides hitches; device profile not selected; VSync/dynamic resolution hides GPU regression; CSV threshold too noisy; Insights trace overhead changes result; LLM tag too broad; low tier removes gameplay readability; one-run CI false failure.  
**Performance Checks:** P50/P95/P99 frame time, hitch count, Game/Draw/GPU, first-interactive time, LLM tag totals, allocation growth/churn, applied CVars, thermal/power notes, repeated-run variance and artifact size.  
**Interview Questions Unlocked:** reproducible benchmark design, CSV versus Insights, Memory Insights versus LLM, device-profile proof, packaged hitch workflow, performance CI thresholds.  
**Version / Plugin Caveats:** CSV commands/categories, trace channels, LLM tag coverage, automation hooks and device profile selectors are branch/platform sensitive. Verify exact target commands and acceptances before using the gate as a release blocker. [SRC-PERF-001] [SRC-PERF-003] [SRC-PERF-006] [SRC-PERF-007] [SRC-PERF-008] [SRC-PERF-009] [SRC-PERF-010] [SRC-PERF-011]

### Project 4/6 Platform Readiness Extension

**Goal:** Convert packaged performance/release work into a platform-readiness packet that covers device tiers, mobile constraints, certification-adjacent failure states and crash/symbol evidence.

**Minimum Features:** two target platform/device tiers; packaged clean install; device profile/scalability proof; cold/warm launch evidence; 5-minute and stress performance run; memory/storage check; suspend/resume or closest available lifecycle test; input/device/network/storage failure matrix; forced crash with symbols; artifact manifest.

**Implementation Steps:**

1. Choose two tiers: for example low/mobile and target/high, or desktop and closest available mobile device. Record device/OS/SDK/toolchain versions.
2. Define budgets for FPS/frame percentiles, memory, texture pool, package size, startup time, hitch count and acceptable thermal-duration behavior.
3. Set device profile/scalability rules and prove the applied CVars in a packaged run.
4. Package from a clean path and archive package, logs, cook/stage manifests, symbols, build metadata and exact command parameters.
5. Run cold boot, warm boot, 5-minute gameplay and stress scenario. Record frame percentiles, hitch count, memory, thermal/power notes where available and first interactive time.
6. Test platform-state failures available on the target: suspend/resume, controller disconnect, app background/foreground, storage full/write failure simulation, network loss, account/entitlement unavailable or permissions denied.
7. Verify UI safe area/readability/input mode on each tier; do not let low scalability remove gameplay-significant information.
8. Force a packaged crash and prove symbolicated callstack matches the archived build.
9. Create a certification-adjacent matrix: event, expected behavior, observed result, evidence path, owner and remaining risk.
10. Write the release decision: ship, hold, reduce scope, device-tier exclusion or require platform-holder documentation/test pass.

**Debugging Exercises:** SDK/toolchain mismatch; package launches only on dev machine; wrong device profile selected; low tier disables gameplay object; suspend resumes with stale delegates; storage write fails silently; controller/input state lost; network loss loops; crash symbols from wrong build.  
**Performance Checks:** cold/warm first interactive time, P50/P95/P99 frame time, hitch count, Game/Draw/GPU, memory/LLM/texture pool, package/install size, battery/thermal notes, repeated-run variance, crash symbolication time.  
**Interview Questions Unlocked:** mobile readiness, certification readiness, device profiles versus scalability, device-only crash workflow, packaged platform performance, release artifact contract, source-sensitive console requirements.  
**Version / Platform Caveats:** Android/iOS setup, SDK/NDK/Xcode/provisioning, mobile rendering, package signing and store/cert requirements are platform/version sensitive. Console certification docs are confidential; use platform-holder checklists in real projects and do not invent public TRC/TCR details. [SRC-PLAT-001] [SRC-PLAT-002] [SRC-PLAT-003] [SRC-PLAT-004] [SRC-PLAT-005] [SRC-PLAT-006] [SRC-PLAT-007] [SRC-PLAT-008] [SRC-BUILD-018] [SRC-PERF-007] [SRC-PERF-008]

### Project 4 Android Platform Profiler Extension

**Goal:** Produce a target Android profiling packet that correlates Unreal engine-side evidence with Android device/GPU evidence instead of relying on editor stats or one profiler screenshot.  
**Minimum Features:** packaged Android build; deterministic scenario; active Device Profile/scalability proof; Unreal CSV; one Unreal Insights or Memory/LLM capture; Android system profile through APA or AGI; AGI frame profile for one GPU-bound frame if supported; LMK/lifecycle memory check; ADPF baseline/adaptation comparison if supported.

**Implementation Steps:**

1. Pick one Android target device and record model, SoC/GPU, Android version, graphics API, driver/tool version where available and battery/thermal state.
2. Package a build from a clean path. Archive package, symbols/logs if available, command line, Device Profile/CVar dump and scenario version.
3. Define one deterministic scene: static GPU scene, traversal scene, combat stress scene or UI/render-target stress scene. Include warm-up and sample windows.
4. Capture Unreal CSV for three repetitions and classify P50/P95/P99 frame time, hitch count, Game/Draw/GPU and memory.
5. Capture Unreal Insights or Memory/LLM for the exact failing window if engine-side CPU/loading/memory ownership is still unclear.
6. Capture Android system profiling through Android Performance Analyzer or AGI. Distinguish active CPU work from CPU waiting around present/swap/GPU completion.
7. For a GPU-bound representative frame, capture AGI Frame Profiler where supported. Identify the longest render pass and classify binning, rendering or GMEM load/store pressure.
8. If GPU counters are used, record GPU vendor and counter names; do not compare raw counters across vendors without vendor documentation.
9. Run a lifecycle memory test: heavy map, background, memory pressure or another app, foreground. Correlate Android LMK/process evidence with Unreal memory checkpoints.
10. If ADPF is available, run baseline without adaptation and then with the Unreal ADPF plugin. Record thermal/scalability decisions and whether quality level changed.
11. Make one content/performance change and repeat the same scenario. The final report must explain whether the fix changed active CPU, GPU pass cost, memory, thermal behavior or only quality settings.
12. Produce a profiler evidence packet with artifacts, before/after table, residual risk, and owner.

**Debugging Exercises:** high CPU total caused by GPU wait; expensive GMEM store from unnecessary render target/depth store; AGI frame profile unsupported on device; GPU counter names misunderstood across vendors; LMK occurs only after background/foreground; ADPF hides regression by dropping quality; fixed-performance benchmark mixed with normal thermal run; active Device Profile differs from expected.  
**Performance Checks:** active versus total CPU frame time, GPU utilization/queue, longest render pass, binning/rendering/GMEM load-store cost, draw-command count where available, CSV percentiles, Unreal memory/LLM tags, Android LMK evidence, thermal headroom/status, scalability-quality transitions and repeated-run variance.  
**Interview Questions Unlocked:** Unreal Insights versus Android profilers, Android CPU/GPU frame-time diagnosis, AGI render-pass workflow, GPU counter caution, LMK memory workflow, ADPF adoption, fixed-performance benchmarking and target-evidence reporting.  
**Version / Platform Caveats:** Android profiler availability, AGI/APA UI, GPU counters, Frame Profiler support, ADPF plugin version, fixed-performance mode and graphics API behavior are device/driver/toolchain sensitive. Verify target hardware and do not generalise one Android capture to every SKU. [SRC-PLAT-010] [SRC-PLAT-011] [SRC-PLAT-012] [SRC-PLAT-013] [SRC-PLAT-014] [SRC-PLAT-015] [SRC-PLAT-016] [SRC-PLAT-017] [SRC-PERF-003] [SRC-PERF-009]

### Project 4 Apple/Console Platform Profiler Extension

**Goal:** Produce a source-safe Apple or console profiling packet that correlates Unreal evidence with platform-specific evidence and avoids overclaiming across devices.

**Minimum Features:** packaged target build; scenario markers; Unreal CSV/Insights or equivalent; one Apple Metal/Instruments/MetricKit artifact or one authorised console profiling artifact; capture metadata; before/after validation; source-safe report.

**Implementation Steps:**

1. Choose one Apple device/macOS target or authorised console devkit. Record device/SKU, OS/firmware, Xcode/SDK/tool version where allowed, UE branch, build config, RHI, package ID and scenario version.
2. Add shared UE markers for boot, menu, map load, peak scene, background/foreground or platform event.
3. Capture UE CSV and one deeper UE trace for the same failing window.
4. On Apple, capture one relevant artifact: Xcode Metal GPU capture, Instruments memory/responsiveness trace, Xcode memory graph, Organizer/MetricKit report or Thread Performance Checker issue.
5. On console, capture only authorised platform evidence and keep confidential raw artifacts outside the public repo.
6. Map the platform finding back to a UE pass, subsystem, content asset, native wrapper, package/cook setting or release artifact.
7. Make one change and rerun the same scenario. The report must say whether the improvement is visible in UE evidence, platform evidence or both.
8. Add a section titled "What this evidence does not prove" to prevent overgeneralising one device/tool result.

**Debugging Exercises:** simulator memory accepted as device proof; Metal trace replayed on different GPU/OS; capture option left enabled during max-performance test; MetricKit field signal lacks scenario markers; console counter is shared publicly by mistake; devkit package has wrong symbols; platform finding cannot be mapped back to UE ownership.  
**Performance Checks:** CPU responsiveness, memory high-water/retained, GPU pass/pipeline/counter evidence, frame percentiles, hitch count, field metric trend, capture overhead, replay compatibility, symbol/build ID match and repeated-run variance.  
**Interview Questions Unlocked:** Apple Instruments/Xcode/MetricKit workflow, Metal capture-to-UE mapping, simulator versus device memory, source-safe console profiler answer, console certification without confidential leakage, platform evidence memo.  
**Version / Platform Caveats:** Apple tooling depends on Xcode, OS, device/GPU family, capture settings and package configuration. Console profiling/certification details are confidential and must come from authorised platform-holder documentation. [SRC-PLAT-018] [SRC-PLAT-019] [SRC-PLAT-020] [SRC-PLAT-021] [SRC-PLAT-022] [SRC-PLAT-023] [SRC-PLAT-024] [SRC-PERF-003] [SRC-PERF-009] [SRC-RENDER-019]

### Project 4/6 Device Lab Automation Extension

**Goal:** Convert the platform-readiness packet into an automated target-evidence lane that can package, install, launch, measure, archive, triage and report without manual clicking.

**Minimum Features:** one reproducible packaged artifact; two devices or two device-profile tiers; one smoke scenario; one performance scenario; one lifecycle or failure-state scenario; Gauntlet/RunUAT or equivalent documented runner; CSV artifact capture; active profile/CVar proof; forced packaged crash; run manifest; flake/triage policy.

**Implementation Steps:**

1. Define the device matrix: device ID, platform, OS/toolchain, install mode, input device, power/thermal state and whether it is a blocking or reporting lane.
2. Package from a clean path and archive package, symbols, cook/stage manifests, BuildCookRun/BuildGraph command line, engine/project revision and build ID.
3. Create a run manifest that binds build ID, device ID, scenario ID, command line and output paths.
4. Add two project scenario contracts: `frontend_smoke` and `combat_peak` or equivalents. Each needs warm-up, sample window, readiness marker, exit marker and owner.
5. Use Gauntlet/RunUnreal, AutomationTool, platform scripts or a documented equivalent to install, launch and pull artifacts. If Gauntlet is used, verify target branch command classes and device adapters.
6. Capture target logs, CSV, screenshot/video where useful, and active Device Profile/scalability evidence for each run.
7. Add one short Unreal Insights or Memory Insights capture only for a selected failed or sampled run.
8. Force a packaged crash and prove the callstack symbolicates against the archived symbols.
9. Inject one missing-asset, wrong-profile or missing-staged-file failure and prove the lab classifies it without a broad workaround.
10. Define gate states: pass, product fail, lab infrastructure fail, flaky device quarantine, reporting-only metric and blocked by missing evidence.
11. Produce a device-lab report with run manifest, pass/fail table, CSV summary, evidence links, rerun policy and owner for every failure.

**Debugging Exercises:** runner reports generic failure but first causal log is cook error; device install succeeds but launch hangs behind OS permission dialog; CSV missing because app crashed before flush; active profile differs from expected profile; one low-tier device thermal-throttles into false failure; crash symbols from wrong artifact; automated rerun hides deterministic crash; lab infrastructure failure is assigned to gameplay.  
**Performance Checks:** P50/P95/P99 frame time, hitch count, run-to-run variance, active profile/CVars, first-interactive time, install/launch duration, artifact pull duration, LLM/memory checkpoints, trace overhead, device temperature/power notes where available.  
**Interview Questions Unlocked:** device-lab design, Gauntlet/RunUnreal fit, CSV versus Insights in automation, flaky-device policy, artifact identity, target evidence packet, promotion from reporting metric to blocking gate.  
**Version / Platform Caveats:** Gauntlet command classes, C# APIs, device deployment interfaces, RunUAT flags, telemetry commands, platform logs/profilers and lifecycle hooks are branch/platform sensitive. Console certification requirements must come from authorised platform-holder documentation. [SRC-PLAT-009] [SRC-BUILD-015] [SRC-BUILD-016] [SRC-BUILD-017] [SRC-BUILD-018] [SRC-PERF-003] [SRC-PERF-009] [SRC-PERF-011] [SRC-PLAT-005]

## Project 8: Game Patterns Sandbox

**Goal:** Implement recurring patterns as observable alternatives, including their lifetime, timing and performance costs, rather than presenting a pattern-name showcase.  
**Target Roles:** Gameplay, engine generalist, technical design, tools.  
**Topics Covered:** Component, Observer/delegate, Command, State, Strategy, Factory, Object Pool, Service Locator/subsystems, Event Queue/message bus, Dirty Flag, Flyweight/Type Object, data-driven definitions.  
**Minimum Features:** Component-based combat capability; direct/delegate/queued message comparison; serialisable input Command; explicit weapon State; two targeting/fire Strategies; Data Asset definitions; bounded effect pool; scoped subsystem with an explicit-dependency alternative; queue telemetry and data validation.  

**Implementation Steps:**

1. Define the sandbox's forces and budgets: 100 actors, 10 listeners each, synchronous versus end-of-frame feedback, one authoritative local/server mode, and a fixed benchmark action sequence.
2. Implement a coherent Health Component with an owner contract, validated mutation, committed result and balanced delegate lifecycle.
3. Route the same `HealthChanged` transition through direct interface call, multicast delegate and queued domain message. Record listener order, re-entrancy, latency, payload copies, stale-listener behaviour and trace visibility.
4. Represent fire input as a compact Command carrying issuer/sequence/aim parameters. Execute immediately, queue it, serialise/replay it, then inject duplicate/out-of-order commands and define handling.
5. Implement weapon idle/equip/fire/reload states with allowed transitions and explicit enter/exit cancellation. Deliberately leak a timer/delegate on exit and diagnose it.
6. Implement hitscan and projectile as Strategies behind one contract. Compare a two-case branch with Strategy objects and document the threshold where the abstraction pays.
7. Create immutable `UPrimaryDataAsset` weapon definitions and separate runtime state. Validate missing IDs/assets, invalid ammo/intervals, contradictory tags and hard-reference budget.
8. Add a bounded cosmetic/projectile pool with acquire/release ownership, generation IDs, full reset, exhaustion policy and retained-memory metrics. Compare fresh spawn, pool and batched representation.
9. Put one world-scoped coordination service in a `UWorldSubsystem`. Show the hidden-dependency version where every object looks it up, then retrieve it at orchestration and pass a narrow interface/handle into core logic.
10. Add an end-of-frame event queue with bounded capacity, coalescing for replaceable updates, ordering key, queue-depth/latency counters and dead-target policy.
11. Add a Dirty Flag aggregate (for weapon modifiers or UI summary) and prove multiple writes cause one recomputation. Deliberately omit invalidation once and catch the stale read with an assertion/test.
12. Write a two-page report: force → simplest baseline → pattern → UE mechanism → cost/failure → keep/remove decision.

**Key Unreal Systems:** ActorComponents, native/dynamic delegates, UObjects/structs, Data Assets, Asset Manager awareness, Subsystems, timers, replication awareness, Data Validation, Insights/custom stats.  
**Debugging Exercises:** duplicate delegate binding; raw lambda use-after-owner; re-entrant mutation; State exit leak; queued message after target destruction; duplicate Command; pool double release/stale state; missing dirty invalidation; subsystem lookup in wrong World.  
**Performance Checks:** broadcast count/time, queue depth/P95 latency, payload allocations/copies, component/Tick count, pool hit/miss/peak memory, dirty recompute ratio, definition hard-reference closure.  
**Interview Questions Unlocked:** Component versus inheritance, direct call versus Observer/queue, Command value, State versus Strategy, subsystem/service-locator risk, pool trade-offs, data-driven definition versus runtime state.  
**Stretch Goals:** deterministic fixed-step command replay; undoable editor Command; lock-free queue research (not presumed necessary); compare one hot population with MassEntity later.  
**Version / Plugin Caveats:** UE5.3–UE5.6. Data Asset/Subsystem/delegate fundamentals are stable; exact Asset Manager, validation and network replay APIs are target-sensitive. [SRC-ARCH-001] [SRC-ARCH-009] [SRC-ARCH-012] [SRC-ARCH-013]

## Project 9: System Design Portfolio

**Goal:** Produce implementation-backed, interview-ready designs for inventory, health/damage, weapon/ability, interaction, quest/objective, save/load, AI enemy integration and UI notifications.  
**Target Roles:** Gameplay, engine generalist, networking, technical design, AI/gameplay.  
**Topics Covered:** requirement discovery, definitions/state/presentation, identity, transactions, authority/audience, prediction/reconciliation, persistence/versioning, soft loading, data validation, testability and observability.  
**Minimum Features:** One integrated vertical slice with authoritative health; inventory/equipment; interaction; one weapon/ability; two-step quest; save/load; AI enemy; event-driven UI; Data Asset definitions; stable IDs and schema migration; two-client test.  

**Required design canvas for each system:**

1. Assumptions, maximum scale and explicit non-goals.
2. Definition, runtime state and presentation schemas.
3. Lifetime, owner, authority and replication audience.
4. Stable identity and missing/stale-target policy.
5. Commands, read-only queries, committed events/deltas and timing.
6. Validation/invariants and structured rejection reasons.
7. Save schema/version/migration and content-change fallback.
8. Hard/soft reference and async loading policy.
9. Debug correlation fields and one failure reconstruction.
10. Performance budget/escalation threshold, tests and nearest simpler alternative.

**Implementation Steps:**

1. Create validated Data Asset definitions for items, weapon and quest. Keep mutable quantities/ammo/progress outside definitions.
2. Implement authoritative Health Component and committed damage result. Drive AI/UI/quest effects from committed state without coupling Health to their concrete classes.
3. Implement inventory transactions with operation ID and container revision. Support add/move/split/merge/equip and structured rejection; expose no mutable internal container.
4. Make backpack contents owner-private and equipped appearance public. Simulate stale client drag/drop response and rebuild from authoritative revision.
5. Implement interaction discovery locally and execution as a server-validated request with target/action ID, distance/LOS/state/rate checks and targeted rejection.
6. Implement one weapon with explicit idle/fire/reload state and one swappable firing Strategy. Keep authoritative rules independent of animation/UI.
7. Implement a quest definition/instance with stable objective IDs. On activation/load, initialise “own N items” from current inventory, then consume committed deltas.
8. Save stable IDs plus mutable state and schema version. Add one v1→v2 migration and one missing-definition fallback. Restore without re-awarding quest/reward side effects.
9. Connect an AI enemy as a normal Health/interaction/damage participant through interfaces/events; no quest or UI dependency in the enemy class.
10. Build event-driven UI/view models that take initial snapshots, bind/unbind safely, and display pending/confirmed/rejected operations.
11. Run standalone, dedicated server + two clients, respawn, travel/save/load, disconnect and content-version tests. Capture one end-to-end transaction trace.
12. Present each system in five minutes: requirements, model, data flow, authority/persistence, failure/performance trade-off.

**Key Unreal Systems:** Gameplay Framework, Components, interfaces/delegates, Data/Primary Assets, Asset Manager awareness, replication/OnRep/RPCs, SaveGame, UMG/view-model pattern, AI Controller/Perception/BT integration, Data Validation.  
**Debugging Exercises:** definition mutated at runtime; duplicate/lost inventory transaction; stale revision; client-authoritative pickup; UI owns truth; quest misses pre-subscription state; save replays reward; renamed objective ID; soft presentation asset missing; callback after travel.  
**Performance Checks:** inventory delta size/frequency, hard-reference closure/load, UI update count, event cascade depth, Actor/UObject count, save size/time, validation/cook failures, AI/quest listener count.  
**Interview Questions Unlocked:** design health/inventory/interactions/weapons/quests/save; data-driven architecture; multiplayer-ready and save-friendly design; events versus durable state; testing and refactoring seams.  
**Stretch Goals:** Fast Array inventory; GAS implementation branch; party-shared quest policy; trade transaction with two-container atomicity; encrypted/cloud profile awareness; automated migration fixtures.  
**Version / Plugin Caveats:** UE5.3–UE5.6. Fast Array, GAS, Asset Manager and save/platform details require their dedicated version-specific clusters; this project establishes contracts without claiming every specialist implementation. [SRC-ARCH-009] [SRC-ARCH-011] [SRC-NET-001] [SRC-NET-004]

### Project 9 GAS vertical-slice extension

Replace only the weapon/ability branch with an evidence-backed GAS implementation; keep inventory, quest, save and UI contracts independent of GAS internals.

1. Decide Pawn-owned versus PlayerState-owned ASC from respawn/cooldown requirements; diagram server, owner and simulated-proxy lifetime.
2. Add Health/MaxHealth/Stamina and Combat attributes with tested base/current/clamp/replication behaviour.
3. Grant one startup ability and one equipment ability by source/spec handle; prove idempotent initialisation and exact removal.
4. Implement server-only damage, local-predicted dash and duration/periodic regeneration with explicit commit and termination paths.
5. Add a two-source stacking buff and stun that blocks/cancels appropriate abilities through a documented tag taxonomy.
6. Use target data for an aimed action, server-derive final damage and record accepted/rejected evidence.
7. Add one custom or well-wrapped Ability Task and Gameplay Cues whose failure cannot alter gameplay truth.
8. Test Full/Mixed/Minimal GE replication visibility for owner and simulated proxy; record bandwidth and product trade-off.
9. Inject wrong actor info, duplicate grants, missing set-by-caller, leaked end path, tag-count leak, stale avatar binding and task callback after cancel.
10. Run dedicated server plus two clients with lag/loss; correlate spec handle, prediction key, target data, effect handle, failure tags and end reason.

**Exit evidence:** architecture decision, tag dictionary, effect/stacking matrix, failure trace, replication-audience table, Insights/network capture and a five-minute GAS-versus-bespoke defence. [SRC-GAS-001] [SRC-GAS-002] [SRC-GAS-005] [SRC-GAS-012] [SRC-GAS-013]

### Project 9 GAS specialist evidence extension

Use [[ue_gas_specialist_workbook]] as the acceptance suite for the GAS branch. The extension is complete only when the evidence proves production failure handling rather than happy-path ability use.

1. Add a trace row shared by activation, target validation, effect application, task lifecycle, cue execution and UI update.
2. Produce an ASC lifetime decision table for Pawn-owned versus PlayerState-owned state across death, respawn and possession.
3. Implement a grant-source ledger that supports duplicate ability classes from different sources and exact handle removal.
4. Prove Health/MaxHealth/Shield invariants across instant, duration, periodic, predicted and replicated mutation paths.
5. Implement one authoritative damage execution or equivalent calculation with capture timing, set-by-caller validation and zero-damage explanation.
6. Complete a stacking policy matrix for at least two effects with different source/refresh/overflow/expiration rules.
7. Reject a local-predicted dash under forced lag/loss and show resource, montage, cue, movement and UI convergence.
8. Validate target data on the server and explicitly distinguish target validation, server rewind and full rollback.
9. Cancel a custom Ability Task during every wait phase and prove no late callback mutates gameplay.
10. Compare Full/Mixed/Minimal active-effect replication from owner and simulated-proxy evidence, including bandwidth and UI consequences.

**Acceptance criteria:** all workbook labs have a pass/fail result, injected failures have traces, final report names which state is GAS-owned versus ordinary gameplay architecture, and the GAS-versus-bespoke defence includes one place where GAS is better and one place where bespoke code remains simpler. [SRC-GAS-002] [SRC-GAS-003] [SRC-GAS-004] [SRC-GAS-005] [SRC-GAS-009] [SRC-GAS-011] [SRC-GAS-012] [SRC-GAS-013]

## Project 6: Tools and Plugin Project

**Goal:** Build and package a reusable project plugin whose runtime asset/schema is cleanly separated from editor validation, batch modification and custom authoring UI.  
**Target Roles:** Engine/tools, gameplay, technical artist/designer, build/release, engine generalist.  
**Topics Covered:** UBT/UHT, Build.cs/Target.cs, plugin/module descriptors, Public/Private dependencies, runtime/editor modules, API exports, IWYU, Live Coding, Editor Utility Widgets, commandlets, Data Validation, details customisation, transactions, source control and packaging.  
**Minimum Features:** Project plugin; Runtime and Editor modules; custom item-definition Data Asset in Runtime; Data Validator; Editor Utility Widget; batch audit/repair operation; details/property customisation; commandlet/CI entry point; package test proving Editor exclusion.  

### Proposed plugin layout

```text
Plugins/GamePrepTools/
  GamePrepTools.uplugin
  Source/
    GamePrepToolsRuntime/
      GamePrepToolsRuntime.Build.cs
      Public/Definitions/GamePrepItemDefinition.h
      Private/Definitions/GamePrepItemDefinition.cpp
      Private/GamePrepToolsRuntimeModule.cpp
    GamePrepToolsEditor/
      GamePrepToolsEditor.Build.cs
      Private/Validation/
      Private/Batch/
      Private/Customisations/
      Private/Commandlets/
      Private/GamePrepToolsEditorModule.cpp
  Content/Editor/                 # Editor Utility Widget only
```

**Implementation Steps:**

1. Write the module dependency graph before code. `GamePrepToolsEditor → GamePrepToolsRuntime`; never the reverse. Define intended target/module types and default loading phases in the plugin descriptor.
2. Add a Runtime `UPrimaryDataAsset` item definition with stable ID, tags, stack limit, weight, and soft icon/world-class references. Keep it free of UnrealEd, AssetTools, PropertyEditor, Blutility and editor-only types.
3. Make the runtime public header self-contained. Add only dependencies exposed by public API to `PublicDependencyModuleNames`; keep implementation dependencies private. Verify a tiny consumer module can include it in non-unity mode.
4. Add the Editor module and depend privately on UnrealEd/AssetTools/PropertyEditor/DataValidation/Blutility (exact names verified for the target branch). Register all editor extensions in startup and unregister symmetrically in shutdown.
5. Implement `IsDataValid` or `UEditorValidatorBase` rules for duplicate/missing IDs, invalid stack/weight values, missing soft targets, prohibited path/tag combinations, and optional hard-reference budget.
6. Build a deterministic audit core returning planned changes and diagnostics without mutating assets. It accepts explicit `FAssetData`/paths and produces stable reports.
7. Build an Editor Utility Widget front end with filters, selection/folder mode, dry run, preview, confirmation, progress/cancel and exportable changed/skipped/failed report.
8. Add the mutation adapter using supported editor APIs: source-control/write preflight, transaction/`Modify` where supported, package dirtying, explicit save selection and per-asset failure isolation. Make rerunning idempotent.
9. Add a Details or property-type customisation that supplies one high-value authoring improvement—such as stable-ID status and “Generate/Validate” action—using property handles so multi-edit/undo/notifications remain valid.
10. Expose the same audit/validation core to a commandlet or CI-compatible command-line path. It must require no normal game World or Slate UI, return non-zero/failure on policy violations, and emit machine-readable-enough stable lines/report.
11. Add source-control conflict, read-only package, missing dependency, redirector, partial failure, cancellation and editor-shutdown exercises. Prove no tool silently saves unrelated packages.
12. Build deliberate failures: private transitive include, missing API export, wrong module type, runtime dependency on Editor, duplicate extension registration after reload and stale Live Coding pointer. Classify and repair each by stage.
13. Build Development Editor, Development Game, Server if relevant, and Shipping/package. Inspect package logs/binaries/content to prove Editor module and utility assets are excluded while Runtime definitions load.
14. Run the tool on 10, 1,000 and representative maximum assets. Record Asset Registry query, loaded asset count, peak memory, save time, source-control calls and editor responsiveness.
15. Produce a release checklist: supported engine versions/platforms, descriptor version, dependencies, migration, clean-build/package test, docs, validation fixtures and rollback instructions.

**Key Unreal Systems:** UBT, UHT, ModuleRules/TargetRules, IModuleInterface, plugin descriptors, Asset Registry/Manager, Data Assets, Data Validation, Editor Utility Widgets/Blutility, PropertyEditor, AssetTools, commandlets, transactions, source control, packaging.  
**Debugging Exercises:** UHT parse failure; missing include hidden by unity; unresolved external/API macro; missing module dependency; Editor leak into Shipping; incompatible/missing plugin binary; stale/duplicate registration after Live Coding; commandlet assumes World/UI; partial batch failure; non-idempotent repair.  
**Performance Checks:** clean/incremental and unity/non-unity compile time; public include fan-out; module load/startup; query/load/save counts; peak memory; UI frame responsiveness; validation and CI duration; packaged size/dependency audit.  
**Interview Questions Unlocked:** UBT versus UHT; target versus module; Public versus Private dependencies/folders; unresolved externals; Runtime versus Editor modules; plugin module type/loading phase; Live Coding risks; commandlet versus Editor Utility/Python; safe batch tool and validation design.  
**Stretch Goals:** third-party static/dynamic library in an External module; Python front end to the same audit core; custom asset factory/type editor; automation tests for fixtures; BuildGraph/CI matrix; Marketplace-style version compatibility review.  
**Version / Plugin Caveats:** UE5.3–UE5.6. Build-rule properties, host types, loading phases, editor APIs, Utility Widget status and Python APIs evolve. Verify exact target headers/docs and package every supported target; editor Python is not runtime scripting. [SRC-BUILD-001] [SRC-BUILD-002] [SRC-BUILD-005] [SRC-BUILD-006] [SRC-BUILD-009]

### Project 6 Release Automation Extension

**Goal:** Prove the plugin/tooling project can be built, cooked, staged, packaged, launched, symbolicated and diagnosed from a clean machine or build agent, not only from a warm editor workspace.

**Minimum Features:** documented RunUAT or BuildGraph pipeline; clean build matrix; cook/package/stage/archive proof; commandlet/data-validation gate; third-party runtime staging exercise; symbol/artifact archive; forced packaged crash report; tiered CI matrix.

**Implementation Steps:**

1. Define release parameters: engine commit, project commit, platform, target, configuration, archive root, version/build ID, maps, plugins and optional feature flags.
2. Write a simple BuildGraph file or documented RunUAT script that separates build, cook, stage/package/archive and smoke-test steps. Prefer explicit dependencies over one long shell command.
3. Run from a clean workspace with no local Binaries/Intermediate reliance. Record the first causal error if the pipeline fails.
4. Build Development and Shipping Game targets, Editor target and Server target where applicable. Include at least one non-unity/clean job if project policy allows.
5. Cook only intended maps/assets. Inject one missing Primary Asset/cook rule and verify the gate catches it without a blanket "cook everything" workaround.
6. Stage a mock third-party runtime file or DLL/shared library. Prove a clean machine launches without global installs or editor-adjacent files.
7. Archive build logs, cook/stage manifests, packaged output, exact metadata and symbols in one associated artifact set.
8. Add a commandlet/data-validation smoke gate that returns non-zero on policy failure and emits stable report lines.
9. Force a packaged crash. Verify crash metadata, log tail and symbolicated callstack match the exact archived build.
10. Define CI tiers: pre-submit quick, nightly clean, release-candidate and platform/certification jobs. Justify cost and feedback time.

**Debugging Exercises:** BuildGraph node dependency omitted; RunUAT wrapper hides first cook error; editor-only module in packaged content; missing staged runtime DLL; asset cooked locally through broad setting but missing in CI; symbols from wrong build; crash reporter endpoint/symbol path misconfigured; CI job too slow and bypassed.  
**Performance Checks:** clean build duration, cook duration, package/archive size, staged file count, cache dependency, commandlet duration, artifact upload size, smoke test duration and first-launch time.  
**Interview Questions Unlocked:** AutomationTool versus BuildGraph, release pipeline gates, clean-machine proof, packaged-only missing asset, third-party staging, symbols/crash reporting, CI matrix design.  
**Version / Plugin Caveats:** BuildGraph/RunUAT flags, platform packaging/signing, IoStore/Pak defaults, symbol formats and crash reporting endpoints are platform/project sensitive. Verify target engine help/output and studio policy. [SRC-BUILD-015] [SRC-BUILD-016] [SRC-BUILD-017] [SRC-BUILD-018] [SRC-ASSET-005] [SRC-ASSET-006]

### Project 6 PCG Biome Tool Extension

**Goal:** Build a reusable PCG content tool that produces deterministic, debuggable and scalable generated output instead of opaque random scatter.  
**Minimum Features:** PCG Graph and PCG Component/Volume; point generation from spatial input; density shaping; at least five attributes; metadata-domain checks; graph parameters and one graph instance; data-driven mesh/type selection; debug/sanity checkpoint; generated-output cleanup; 1x/10x/100x scale captures; editor versus runtime policy memo.  

**Implementation Steps:**

1. Define the product requirement: editor-only biome authoring, editor-generated set dressing, or runtime generation. Do not leave generation mode implicit.
2. Create a PCG Graph that samples a volume, landscape, spline or actor data. Record exact input sources and seed.
3. Generate point data and add attributes such as `Biome`, `SlopeClass`, `RoadDistance`, `WaterDistance`, `SpawnWeight`, `Species` and `bGameplayBlocking`.
4. Shape density deliberately and record input point count, density distribution and output point count separately.
5. Use filters for hard constraints such as forbidden slope/radius/road overlap; use scoring/weights only for preferences.
6. Use data-driven variation through attributes, Data Tables or Match And Set-style logic rather than hardcoded graph branches.
7. Expose graph parameters for seed, density multiplier, asset set, exclusion tag, debug mode and quality/output policy. Create at least one graph instance with overrides.
8. Add a debug/sanity checkpoint that validates a key assumption and inspect attributes at several graph phases.
9. Spawn output using an instance-friendly representation where possible. Justify any generated Actor.
10. Test cleanup/regeneration: change seed, change parameters, disable/re-enable component, and confirm no duplicate/stale output.
11. Verify World Partition/Data Layer/HLOD behavior if used; otherwise explicitly mark it out of scope.
12. Capture editor generation and packaged runtime behavior if runtime generation is enabled.
13. Compare 1x/10x/100x output scale and separate graph execution cost from generated render/collision/navigation/memory cost.
14. Produce a final PCG report with seed/input/graph version, parameters, point counts, output counts, screenshots, trace data and adoption limits.

**Debugging Exercises:** wrong attribute domain; density changed but final count unchanged; dead branch due to control flow; debug node assumed to run in shipping; duplicate output after regeneration; `Apply On Actor`-style mutation not reverted; generated Actor count creates game-thread/render overhead; runtime generation hitch.  
**Performance Checks:** graph execution time, point count, attribute count, temporary attributes, spawned Actor/Component/instance count, memory, collision/nav impact, draw/GPU cost, runtime hitch duration, cooked asset references and streaming/HLOD behavior.  
**Interview Questions Unlocked:** PCG mental model, points/density/attributes, metadata domains, graph parameters/instances, editor versus runtime generation, PCG debugging workflow, generated-output ownership, PCG performance.  
**Version / Plugin Caveats:** PCG is plugin-dependent and runtime/partition behavior is target-branch sensitive. Verify UE5.3-UE5.6 plugin status, node behavior, generation modes, API signatures and packaged output before promising runtime generation. [SRC-PCG-001] [SRC-PCG-002] [SRC-PCG-003] [SRC-PCG-004] [SRC-PCG-005] [SRC-PCG-006] [SRC-PCG-007]

### Project 6 Large-World Production Pipeline Extension

**Goal:** Prove a small World Partition production pipeline from authoring through packaged traversal, with explicit ownership for durable state, Data Layers, reusable POIs, generated content, HLOD and source-control validation.  
**Minimum Features:** World Partition map; OFPA enabled; three regions; one always-loaded world-state coordinator; two Runtime Data Layers; one Level Instance; one Packed Level Blueprint or equivalent static visual cluster; one generated/PCG or simulated generated region; one HLOD Layer; one fast-travel/streaming-source readiness path; clean cook/package traversal proof.

**Implementation Steps:**

1. Define the world contract: traversal speed, target platform, memory budget, fast-travel points, regions, streaming assumptions and out-of-scope features.
2. Create or select a World Partition map with `Town`, `Wilderness` and `Outpost` regions. Record grid/loading choices and why one grid is sufficient unless proven otherwise.
3. Add a world-state coordinator outside spatial cells. Store durable state such as `TownPhase`, collected IDs and schema version there, not only in unloadable Actors.
4. Create Runtime Data Layers such as `Town_Normal` and `Town_Burned`. Drive them from the coordinator and document initial state, save/load and multiplayer authority policy.
5. Enable/use OFPA and create a changelist validation note mapping changed external actor files to meaningful Actor labels. Inject one omitted external Actor file and document the failure.
6. Build one reusable POI as a Level Instance and one static dense arrangement as a Packed Level Blueprint or justified alternative. Record mode choice, OFPA compatibility and packaged behavior.
7. Add one PCG or simulated generated-output region with seed/input/version metadata. Verify Data Layer/HLOD assignment and cleanup/regeneration behavior.
8. Create an HLOD Layer for distant region visuals and run the target-branch HLOD builder. Capture far, transition and near screenshots plus primitive/draw/memory notes.
9. Implement a schematic fast-travel preloader or scripted streaming source. It must wait for streaming completion and project-specific collision/gameplay readiness before moving the player.
10. Cook/package the map from a clean workspace. Archive build log, cook/stage manifest where available, HLOD builder output, package path and command line.
11. Run packaged traversal and fast-travel scenarios on a target or constrained device profile. Capture cell/layer/readiness logs, CSV or Insights evidence, memory and hitch data.
12. Inject failures: Runtime Data Layer not activated, hard reference keeping a region loaded, stale HLOD, Level Instance mode mismatch, and teleport without readiness. Diagnose each from evidence.
13. Write a five-minute interview memo covering requirements, grid/source/layer policy, durable state, source-control validation, HLOD/PCG choices, package proof, profiler evidence and remaining risks.

**Debugging Exercises:** missing runtime Data Layer activation; editor-only layer used for gameplay; accidental always-loaded reference; partial OFPA submit; Level Instance content missing after embedding; stale HLOD proxy; PCG output missing Data Layer/HLOD assignment; fast travel into collisionless space; packaged cook excludes expected content.  
**Performance Checks:** loaded cell count, always-loaded Actor count, Actor/Component count per region, cell activation time, fast-travel time-to-ready, package/I/O bytes, peak memory, primitive/draw/material-section count, HLOD proxy memory, transition hitch P95/P99 and worst-frame trace.  
**Interview Questions Unlocked:** open-world pipeline design, World Partition versus Level Streaming, Data Layer progression, OFPA source control, Level Instance versus Packed Level Blueprint, HLOD adoption, fast-travel streaming readiness, packaged large-world validation.  
**Version / Plugin Caveats:** World Partition, Data Layer APIs, Level Instance modes, HLOD builder commandlets, PCG generation and packaged runtime behavior are target-branch sensitive. Verify UE5.3-UE5.6 docs/source and run packaged proof before making production claims. [SRC-ASSET-010] [SRC-WORLD-001] [SRC-WORLD-002] [SRC-WORLD-003] [SRC-WORLD-004] [SRC-PCG-001] [SRC-BUILD-016] [SRC-BUILD-017]

### Project 4/6 Unified Target-Proof Packet Extension

**Goal:** Combine packaged performance gates, BuildGraph/RunUAT release identity, World Partition/HLOD builder output, packaged traversal, symbols and crash proof into one evidence packet another engineer can reproduce.

**Minimum Features:** release manifest; target automation audit; clean packaged build; BuildGraph or documented RunUAT flow; staged/cooked manifest evidence; active Device Profile/CVar proof; three-run CSV scenario; one causal trace; HLOD builder output or explicit rejection; packaged traversal/readiness logs; forced packaged crash with matching symbols; injected failure report.

**Implementation Steps:**

1. Fill a `manifest.yaml` with engine/project revision, branch, target, config, platform, build ID, archive root, map list, scenario IDs, symbol location and crash route.
2. Capture target branch help/output for RunUAT or the studio wrapper, BuildGraph if used, World Partition builder commandlets, CSV commands and trace/memory options.
3. Package from a clean workspace or clean agent. Archive command line, UBT/cook/package logs, package path, symbols and cooked/staged manifests where available.
4. Run a packaged smoke scenario outside the editor. Prove active Device Profile, scalability and important CVars from runtime logs.
5. Define one deterministic performance scenario with warm-up, readiness marker and sample window. Run three CSV captures and summarise P50/P95/P99, hitch count, Game/Draw/GPU if available and memory/LLM if available.
6. Capture one short causal trace for the worst frame or an injected regression. Classify CPU, Draw/RHI, GPU, memory, I/O, shader/PSO, streaming or HLOD handoff.
7. Run the World Partition/HLOD builder for the target map or document branch/project rejection. Archive builder command, log and output freshness evidence.
8. Run packaged traversal and fast-travel readiness. Log cell/layer/readiness/HLOD state and record memory, loaded-cell count and transition hitches.
9. Force a packaged crash and prove the callstack symbolicates against the exact archived symbols/build ID.
10. Inject failures: missing cook/stage asset, wrong Device Profile, stale/missing HLOD output or CSV not flushed. Prove the gate catches each with owner and recurrence guard.
11. Produce a final report with artifact links, pass/fail table, unsupported features, residual risks and one five-minute interview answer.

**Debugging Exercises:** BuildGraph green but package path stale; RunUAT wrapper hides first cook error; editor capture passes but packaged CSV fails; wrong profile masks GPU cost; HLOD builder runs for wrong map; stale HLOD hides map changes; teleport commits before streaming readiness; crash symbols from adjacent build; noisy CSV threshold blocks unrelated changes.  
**Performance Checks:** P50/P95/P99 frame time, hitch count, Game/Draw/GPU, first-interactive time, active profile/CVars, memory/LLM checkpoints, loaded World Partition cells, HLOD transition hitch, package size, cook/package duration, artifact completeness and crash symbolication status.  
**Interview Questions Unlocked:** target proof design, packaged performance gate, BuildGraph versus RunUAT, release artifact manifest, World Partition/HLOD builder proof, signal versus causal capture, crash/symbol proof, noisy gate triage.  
**Version / Plugin Caveats:** command flags, BuildGraph tasks, CSV categories, trace channels, LLM coverage, World Partition builder names, HLOD generation requirements and crash pipelines are branch/platform/project sensitive. Verify target help/source and document unsupported features rather than copying commands blindly. [SRC-PERF-003] [SRC-PERF-009] [SRC-PERF-010] [SRC-BUILD-015] [SRC-BUILD-016] [SRC-BUILD-017] [SRC-BUILD-018] [SRC-ASSET-010] [SRC-WORLD-003]

### Project 1/4/9 Runtime Presentation Target-Proof Extension

**Goal:** Prove one gameplay action across animation, collision/physics, UI, audio and VFX under packaged runtime, multiplayer/lifecycle and profiler conditions.

**Minimum Features:** montage or animation timing; server-validated collision or hit query; UI model projection with recycled entry or HUD update; local/predicted plus server-confirmed audio dedupe; pooled Niagara/VFX with reset and bounds policy; one physics reaction or ragdoll variant; packaged standalone and dedicated-server/two-client scenario where feasible; CSV plus one targeted trace; five injected failures.

**Implementation Steps:**

1. Define one action such as melee hit, projectile impact, ability cast or interactable use. Record operation ID, authority, presentation events and expected artifacts.
2. Add animation timing: montage section, notify window or state-machine transition. Server-side gameplay must validate timing/state rather than trusting the notify alone.
3. Add collision proof: trace/sweep/overlap matrix, object channels, expected hit/overlap/block result and `FHitResult` logging.
4. Add UI proof: health/damage feed/inventory row driven from model state. If using a list, include recycled-entry reset and stale async icon rejection.
5. Add audio proof: predicted/local sound plus confirmed/replicated sound with event key dedupe and concurrency policy.
6. Add VFX proof: pooled Niagara or equivalent effect with owner, transform, user parameter, delegate and event-key reset. Test bounds/significance culling.
7. Add physics proof: impulse, knockback or ragdoll with authority policy, collision profile restore and recovery path.
8. Package and run standalone plus dedicated-server/two-client or nearest feasible target mode. Use latency/loss if networking is in scope.
9. Inject failures: cancel montage while window open, wrong collision profile, stale UI row, duplicate audio, pooled VFX stale target, too-small bounds or ragdoll never recovers.
10. Capture CSV and one targeted trace for the worst frame. Classify Animation, Physics, UI, Audio, VFX, Draw/RHI, GPU or Networking owner.
11. Write a presentation-proof report with artifact links, pass/fail table, owners, fixed failures and residual risks.

**Debugging Exercises:** notify applies damage after cancel; simulated proxy foot-slide; start-penetrating sweep misread; stale virtualised list entry; UI focus remains on closed modal; hit sound plays twice; pooled effect keeps old colour/target; VFX bounds cull impact early; ragdoll collision profile not restored; packaged audio/effect missing due to cook rules.  
**Performance Checks:** animation update/evaluate cost, physics query/constraint time, widget rebuild/invalidation/list count, audio voice/concurrency pressure, Niagara system/emitter/render cost, render overdraw, memory, P95/P99 frame time and worst-frame trace.  
**Interview Questions Unlocked:** animation notify authority, root-motion networking, collision matrix, recycled UI entry, audio dedupe, VFX pooling/bounds, cross-system failure injection, presentation versus gameplay truth.  
**Version / Plugin Caveats:** montage/notify APIs, Chaos physics details, CommonUI/MVVM/ListView behaviour, audio concurrency/MetaSound, Niagara debug/profiling tools and target package support are branch/platform/plugin sensitive. Verify exact UE5.3-UE5.6 behaviour and use [[ue_runtime_presentation_target_proof_workbook]]. [SRC-ANIM-003] [SRC-ANIM-006] [SRC-PHYS-003] [SRC-UI-005] [SRC-AUDIO-004] [SRC-FX-001] [SRC-PERF-003] [SRC-NET-006]

## Project 5: MassEntity Crowd Prototype

**Goal:** Compare a conventional Actor crowd with a MassEntity implementation under equivalent simulation and representation requirements, then defend a hybrid production architecture from measured evidence.  
**Target Roles:** Gameplay/AI, engine generalist, optimisation, tools, technical design.  
**Topics Covered:** ECS/data-oriented design, entities/fragments/tags, archetypes/chunks, queries/processors, deferred commands, traits/configs/spawners, representation, simulation LOD, hybrid Actor promotion, debugging/profiling and replication awareness.  
**Minimum Features:** Actor baseline; Mass entities with Transform/Velocity/Target/state fragments; at least three processors; custom Trait and entity config/spawner; ISM representation; High/Medium/Low/Off simulation/visual tiers; Actor promotion; structural-change exercise; Mass debugging; measured comparison.  

**Implementation Steps:**

1. Define equal workload/fidelity: agent counts (100/1,000/10,000 as hardware permits), fixed camera path, movement/target behaviour, visual mesh/animation approximation, collision/avoidance scope, update budget and 60 FPS target.
2. Build an Actor/Component baseline with manager/significance options. Capture Game/Draw/GPU, memory, Actor/Component/Tick counts and behaviour correctness.
3. Enable only required Mass/MassGameplay plugins for the exact UE version. Record plugin/module/version status and do not copy later API signatures blindly.
4. Define small access-oriented fragments: transform (engine fragment where appropriate), velocity, target/goal, stable domain ID and simple state/LOD. Add tags only for query-selective stable states.
5. Implement processors for target/desired velocity, avoidance or steering, and movement integration. Declare precise read/write requirements and explicit phase/order dependencies.
6. Create a custom Trait that adds/configures required fragments/shared parameters and validates conflicting/missing composition. Build Mass Entity Config/Template and spawn through a Mass Spawner or supported programmatic path.
7. Use deferred commands for death/state composition changes. Deliberately mutate structure during iteration or toggle a tag every frame in a safe branch; compare correctness and migration/command cost.
8. Add visual representation through the target-version MassRepresentation path: ISM for normal crowd and Actor representation for high significance/selected agents. Define entity↔Actor stable mapping and pool reset.
9. Add simulation LOD with different frequencies/fidelity and visual LOD with Actor/ISM/none. Add hysteresis and maximum high-detail counts. Prove gameplay invariants remain valid at Low/Off.
10. Implement player selection/interaction against an Actor representation and resolve it to authoritative Mass identity/state. Demote the Actor during/after interaction and prove no stale pointer or duplicate ownership.
11. Add debug counters/views for entity count, archetypes, chunks, query matches, processor time, deferred commands, structural migrations, LOD populations, representation types and promotion rate.
12. Create failure matrix: wrong query presence; query not registered/configured; inactive/wrong-phase processor; stale entity handle; deferred command not flushed; archetype explosion; high-cardinality shared fragment; dual Actor/Mass transform writer; representation thrash.
13. Compare Actor and Mass at each count with equivalent visible output. Report simulation versus representation separately; include one result where a simpler Actor manager is preferable.
14. Add multiplayer design notes: server authority, stable network ID, viewer LOD/relevancy, which fragments are local/replicated, and why Mass replication status/API must be verified before commitment.
15. Present a production decision memo: keep Actors, adopt Mass, or hybrid; team/tooling/version risk; migration plan; profiler thresholds and fallback.

**Key Unreal Systems:** MassEntity, MassGameplay, MassSpawner, MassRepresentation, MassLOD, Mass Debug tools available in target branch, StateTree/Signals awareness, ISM, Actor pooling/Components, Insights/stats.  
**Debugging Exercises:** missing query match; stale handle; structural mutation/churn; processor order; undeclared write/thread access; wrong World manager; fragmented archetypes; Actor promotion state loss; ISM transform lag; LOD oscillation.  
**Performance Checks:** total/per-processor time, time per entity, entity/chunk/query counts, chunk occupancy, archetypes/shared variants, migrations/commands, memory/entity, LOD/representation counts, Actor/Component/Tick count, Game/Draw/GPU and network estimate.  
**Interview Questions Unlocked:** why ECS/Mass; Actor versus Mass; fragment/archetype/chunk; query/processor; structural changes; Trait/template; shared/chunk fragment; representation/simulation LOD; hybrid architecture; Mass debugging/profiling; when not to use Mass.  
**Stretch Goals:** Mass StateTree/Signals; Smart Objects; ZoneGraph or Mass Avoidance; deterministic replay experiment; custom Mass replication prototype only if target plugin status/source supports it; automated performance scenario.  
**Version / Plugin Caveats:** UE5-specific and plugin-dependent. Exact UE5.3–UE5.6 query registration, processors, traits, representation, debugger, StateTree and replication differ from maintained UE5.7 docs. Pin the engine branch and cite source/API used by the prototype. [SRC-MASS-001] [SRC-MASS-002] [SRC-MASS-003] [SRC-MASS-005] [SRC-MASS-007]

## Project 1/9 Local-Player UI Extension

Build a source-traceable UI vertical slice with a local-player-owned HUD root, virtualised inventory, settings screen, modal confirmation and scalable nameplates. Keep gameplay state authoritative and expose a presenter snapshot plus event deltas; prove clean rebinding through reopen, possession change and travel. Support mouse, keyboard, gamepad and touch with deterministic per-user focus and restoration. CommonUI and MVVM are optional comparisons, not hidden assumptions.

Populate 10/100/1000 inventory items with asynchronously loaded soft icons. Treat items as durable identity and rows as recycled presentation; inject delayed out-of-order completion, stale selection and entry-release bugs, then reject them with stable IDs/request generations and symmetric teardown. Test 16:9, ultrawide, narrow, DPI/device-safe profiles, pseudo-localisation, long/RTL text where required, contrast, scalable text and non-colour cues.

Compare projected screen-space and world-space nameplates for 1/20/100 actors with relevance, redraw and occlusion policies. Capture Widget Reflector hierarchy/hit/focus evidence and Slate Insights update/invalidation/paint traces. Compare binding/Tick versus event updates, virtualised versus rebuilt lists, invalidation and measured Retainer experiments; report Game/Render/GPU, draw/batch/render-target and asset-memory effects. On dedicated server plus two clients/split-screen, prove public/private audience boundaries and server rejection of invalid purchase/equip intent. [SRC-UI-002] [SRC-UI-003] [SRC-UI-004] [SRC-UI-005] [SRC-UI-010] [SRC-UI-014] [SRC-UI-015] [SRC-UI-016]

## Project 1/4 Audio Runtime Extension

Build a Wave/Cue/MetaSound comparison plus one-shots, attached sources and persistent parameter-driven loops with explicit ownership. Author listener-relative attenuation/focus/occlusion, per-owner/global concurrency with forced reject/steal/virtualise cases, Master/Music/SFX/Dialogue/UI classes, overlapping-safe duck mixes and a documented Submix effect/send graph. Compare game-timer and target-verified Quartz cadence under hitches.

In a cooked target build, stress retain/prime/on-demand/stream policies using long streams plus a 100-impact burst. Inject destroyed attachment, pooled stale Component, wrong listener, zero class/mix gain, full concurrency, stolen voice, missing chunk and disconnected route. On dedicated server/two clients, prove predicted/confirmed event deduplication and late-join reconstruction of loops. Profile 1/20/100 emitters across request/Component/logical/virtual/rendered voice counts, rejects/steals, occlusion, DSP, cache/I/O, audio callback and memory; deliver source-lifetime, concurrency, routing and residency diagrams plus audible before/after evidence. [SRC-AUDIO-002] [SRC-AUDIO-003] [SRC-AUDIO-004] [SRC-AUDIO-006] [SRC-AUDIO-007] [SRC-AUDIO-009] [SRC-AUDIO-010]

## Project 1/4 Niagara Combat VFX Extension

Build one impact System from flash, CPU/GPU sparks, smoke, optional ribbon and one bounded light. Document stack order, namespaces, attribute writers, Data Interfaces and renderer bindings. Feed typed impact/surface/normal/intensity data before activation, then compare individual pooled Components with a UE5.6 Niagara Data Channel under 10/100/1000 impacts. Add an attached persistent trail and prove complete pooled reset.

Inject too-small/too-large local bounds, wrong simulation space, stale parameter/attachment, pre-cull null return, Effect Type distance/instance/budget culling, GPU readback delay, event fan-out and expensive full-screen translucent materials. Design Combat/Environment/UI Effect Types with significance, local-player policy and per-platform graceful degradation. Compare Sprite/Mesh/Ribbon/Light/Component Renderers and evaluate a lightweight emitter only on UE5.4+ target support.

Use Niagara Debugger/FX Outliner, Insights, GPU capture, Shader Complexity and Quad Overdraw for 1/20/100 Systems. On dedicated server/two clients prove gameplay collision/damage remains authoritative, owner-predicted/confirmed FX do not duplicate and durable effect state reconstructs for late clients. Deliver data/lifetime diagrams, bounds/cull matrix, Effect Type budget table and visual/performance acceptance evidence. [SRC-FX-002] [SRC-FX-003] [SRC-FX-004] [SRC-FX-005] [SRC-FX-007] [SRC-FX-008] [SRC-FX-009] [SRC-FX-010]

## Project 10: Scripting Integration Lab

Create one C++ semantic façade exposing stable IDs, read-only inventory/objective snapshots, validated commands, change events and one cancellable async request. Prototype one pinned Lua path (UnLua or sluaunreal) and one pinned C# path (UnrealCLR snapshot or, if runtime feasibility fails, an explicitly scoped out-of-process/editor tool). Do not expose raw ownership or authoritative mutation.

For Lua, exercise modules/tables/metatables, closure callback, coroutine cancellation, C API/binding errors, UObject invalidation, forced Lua/UE GC, reload generation/schema migration and restricted libraries/quotas. For C#, exercise explicit-layout values, handle/wrapper invalidation, rooted/unregistered callback, IDisposable/SafeHandle-style teardown, off-thread Task result dispatch, managed exception capture and clean-machine runtime deployment/AOT feasibility.

Benchmark native and feasible integrations at 1/1,000/100,000 empty/value/batched calls plus snapshots/callback fan-out, allocations/GC, wrapper/root counts, startup/reload and package size. Inject stale handle, wrong thread, ABI bool/layout mismatch, old-generation callback and missing staged runtime. Deliver API schema, lifetime diagrams, plugin/version/platform/source matrix, mixed-stack evidence, security/package checklists and an adopt/reject/limited-use memo. A well-evidenced rejection is successful. [SRC-SCRIPT-001] [SRC-SCRIPT-002] [SRC-SCRIPT-003] [SRC-SCRIPT-005] [SRC-SCRIPT-006] [SRC-SCRIPT-007] [SRC-SCRIPT-009] [SRC-SCRIPT-010]

## Project 7B Standard C++ Systems Extension

Extend the lifetime lab with a forwarding factory tested against lvalue/rvalue/const/move-only inputs; SFINAE and C++20 concept versions of one API; callable comparison across template/function pointer/`std::function`/Unreal delegate; and measured object-layout/AoS/SoA/chunked alternatives. Build monotonic arena and pool variants with escaped-handle, destructor and reset failures.

Add concurrency cases for data race, deadlock, missed/spurious wake, false sharing and release/acquire publication. Implement snapshot→partitioned tasks→deterministic merge→Game Thread apply with owner-generation cancellation and Insights evidence. Across two translation units/modules, inject unresolved/duplicate/template visibility/export and static-initialisation failures. Run ASan/TSan/UBSan where supported, documenting configuration, overhead, first root report and blind spots. Deliver language/toolchain matrix, happens-before diagrams, compiler layout/symbol evidence and the simplest-correct-design memo. [SRC-CPP-015] [SRC-CPP-017] [SRC-CPP-019] [SRC-CPP-020] [SRC-CPP-022] [SRC-CPP-024] [SRC-CPP-026] [SRC-CPP-027] [SRC-CPP-030] [SRC-CPP-031] [SRC-CPP-032]

## Project 1 Animation Systems Extension

Extend the Core Gameplay Sandbox with a source-traceable character animation vertical slice. Build a thread-safe movement/presentation snapshot, grounded/in-air state machine, calibrated speed/direction Blend Space, Aim Offset, sync markers, upper-body fire/reload montages and a linked equipment layer. Add a server-validated melee action whose Notify State only reports a correlated trace window and whose authoritative owner closes it on cancellation/timeout. Compare in-place and root-motion dash under dedicated-server lag/collision. Retarget the complete set to a differently proportioned mesh, then add foot IK and one validated Motion Warping interaction.

Inject and diagnose: wrong Anim Class/Skeleton, stale owner snapshot, missing/irrelevant Slot, slot-group interruption, missed/duplicated notify, wrong additive base, world/local IK mismatch, bad retarget root/pose, root-motion double movement and significance throttling of a critical character. Capture Rewind Debugger, Pose Watch and Animation Insights for 1/20/100 characters. Deliver the gameplay-animation contract, asset/layer dependency graph, thread map, montage/notify matrix, root-motion network evidence and measured quality/performance decision. [SRC-ANIM-004] [SRC-ANIM-008] [SRC-ANIM-009] [SRC-ANIM-013] [SRC-ANIM-016] [SRC-ANIM-017]

## Project 1/3 Physics and Collision Extension

Create a documented collision matrix for Pawn, World, Interactable, Projectile, Melee and Trigger profiles, then automate all expected pair/query responses. Author simple/complex variants and implement line, sphere and capsule sweeps plus overlaps with full `FHitResult`, trace-tag and debug-draw evidence. Compare CharacterMovement push, kinematic sweep and simulated-body impulse. Add mass/inertia/off-centre impulse, sleep/CCD and substep experiments; a constrained door with deliberate frame/mass/stiffness failures; Physical Material surface feedback; and an authoritative animation→ragdoll→collision-safe recovery flow.

On a dedicated server, validate throw/push intent and compare replicated corrections under lag/loss; investigate modern Network Physics modes only after confirming the target UE branch. Record CVD scene queries, contacts, joint frames and solver/game/network timelines where available. Scale 10/100/1000 props and report query, awake-body, contact, solver, callback, substep and network costs. Inject wrong enabled mode/channel/response, missing overlap generation, ignored self, initial penetration, duplicate Components/callbacks, complex-collision misuse, double transform owner, asleep body, tunnelling and unstable constraint. [SRC-PHYS-001] [SRC-PHYS-002] [SRC-PHYS-005] [SRC-PHYS-008] [SRC-PHYS-009] [SRC-PHYS-010]

## Project 11: Production Bug Triage Capstone

**Goal:** Build one small but deliberately failure-rich Unreal vertical slice, then prove you can diagnose it like a production engineer instead of only describing systems in isolation.  
**Target Roles:** gameplay, networking, engine generalist, tools, optimisation, platform, technical design.  
**Topics Covered:** UObject lifetime, gameplay framework, networking, GAS or custom abilities, UI, animation, physics, audio/VFX, asset loading/cooking, profiling, packaging, automation and platform evidence.

**Minimum Features:**

- one replicated player action with local prediction or responsive presentation;
- one interactable/objective with save/load identity;
- one UI list or HUD projection;
- one animation/audio/VFX feedback chain;
- one async-loaded asset;
- one packaged build and one automated smoke run.

**Injected Failures:**

1. Server RPC sent from an unowned Actor.
2. Replicated state split across two OnRep callbacks with ordering assumption.
3. Async icon load updates a recycled UI row.
4. Animation notify opens a damage window that fails to close on interruption.
5. Collision profile blocks visually but gameplay trace uses the wrong channel.
6. Audio warning is rejected by concurrency with no fallback feedback.
7. Niagara effect disappears because fixed bounds are too small.
8. Soft-referenced asset works in editor but is missing in package.
9. Plugin/runtime dependency works locally but fails on clean-agent package.
10. Packaged target has a performance regression hidden by editor testing.

**Required Evidence:**

- bug matrix with symptom, subsystem owner, expected invariant, reproduction steps, first failing evidence and fix;
- dedicated server or multi-process networking evidence for the RPC/replication failures;
- Widget Reflector or Slate Insights evidence for UI row/focus/performance issues;
- Rewind Debugger/Pose Watch/Animation Insights or equivalent animation evidence for notify interruption;
- collision trace/debug-draw or CVD evidence for collision mismatch;
- audio log/console/concurrency evidence for rejected critical cue;
- Niagara Debugger/FX Outliner evidence for bounds/scalability issue;
- cook/package logs and asset reference proof for missing packaged asset;
- clean-agent or clean-environment build log for missing dependency;
- packaged performance capture using Unreal Insights/stat/CSV and platform tool where available.

**Acceptance Criteria:**

1. Every injected failure is reproduced before fixing.
2. Each fix includes a before/after evidence pair, not only a code diff.
3. At least three bugs are verified in packaged build, not only PIE.
4. At least two bugs are verified under dedicated-server or multi-client conditions.
5. At least one performance issue includes a before/after timing table.
6. At least one asset/package issue is reproduced on a clean machine/agent or clean cache.
7. The final report labels version-sensitive APIs and plugin/platform caveats.
8. The capstone ends with a prevention plan: validation rule, automated test, checklist or instrumentation for each bug class.

**Interview Questions Unlocked:** "Tell me about a hard Unreal bug", "How do you debug across systems?", "How do you know the fix worked?", "How do you separate editor-only success from packaged proof?", "How do you triage multiplayer versus presentation bugs?", "What evidence belongs in a portfolio packet?"  
**Version / Plugin Caveats:** Exact tooling varies by UE5.3-UE5.6 branch and platform. The capstone is successful when it records what was available, what was not, and what evidence replaced missing tooling. [SRC-NET-003] [SRC-NET-005] [SRC-UI-003] [SRC-UI-005] [SRC-ANIM-010] [SRC-PHYS-002] [SRC-AUDIO-004] [SRC-FX-003] [SRC-ASSET-006] [SRC-BUILD-017] [SRC-PERF-003] [SRC-PERF-009]

## Project 7D: Systems C++ and OS Memory Lab

**Goal:** Prove the C++/OS boundary concepts behind allocation, virtual memory, threads, IPC, shared memory and deadlock.  
**Target Roles:** Engine/generalist/tools/networking/performance.  
**Topics Covered:** pointer/reference ABI awareness, `new`/`delete` versus `malloc`/`free`, placement `new`, alignment, `static`/inline/macro build behaviour, process/thread/task distinction, `VirtualAlloc`/`mmap`/`brk` awareness, page faults, IPC and shared memory.  
**Minimum Features:** one command-line native test app or Unreal Program target; allocation/lifetime counters; virtual-memory first-touch experiment; two-process IPC sample; shared-memory ring buffer; deadlock reproducer/fix; pointer/reference assembly notes.  

**Implementation Steps:**

1. Build constructor/destructor counter types and compare `new`, `new[]`, `malloc`, placement `new`, arena and pool paths.
2. Print `sizeof`, `alignof`, member offsets and cache-line-adjacent fields for two hot structs; explain the layout decision.
3. Create pointer and reference parameter examples, compile with debug and optimised settings, and document observed assembly/IR differences without overclaiming.
4. Add `static` local, namespace `static`, inline variable and macro examples across multiple translation units; include a non-unity build check.
5. Reserve/commit/touch a large virtual address range on the current OS and record first-touch timing plus available page-fault counters.
6. Launch two processes and prove that an ordinary pointer value from one process is meaningless in the other.
7. Implement one pipe/socket IPC sample for messages and one shared-memory ring buffer for bulk data.
8. Use offsets/indices in shared memory, add a versioned header, capacity, producer/consumer state, overrun count and explicit synchronisation.
9. Reproduce a two-lock deadlock, capture the wait graph, then fix with lock ordering or a multi-lock helper.
10. Deliver a short evidence memo: what is language-level, what is ABI/toolchain-level and what is OS-level.

**Debugging Exercises:** mismatched allocation family, missing destructor after placement `new`, stale pointer after container growth, ODR/macro mismatch across TUs, page-fault hitch during first touch, raw pointer stored in shared memory, deadlock cycle, task capture lifetime bug.  
**Performance Checks:** allocation counts, first-touch latency, page fault counters where available, shared-memory throughput, lock wait time and false-sharing comparison.  
**Interview Questions Unlocked:** references versus pointers at ABI level; `new` versus `malloc`; `brk`/`mmap`/`VirtualAlloc`; process versus thread; IPC choices; shared-memory layout; deadlock diagnosis.  
**Version / Platform Caveats:** OS APIs differ; keep Windows and POSIX experiments separated or mark unsupported cases. [SRC-CPP-033] [SRC-CPP-034] [SRC-CPP-036] [SRC-SYS-001] [SRC-SYS-005] [SRC-SYS-008]

## Project 3 Security and Anti-Cheat Extension

**Goal:** Turn Project 3's networked mini game into a defensive security lab for replay resistance, server authority, information exposure and evidence-driven anti-cheat.  
**Target Roles:** Gameplay/networking/backend/security-aware generalist.  
**Topics Covered:** TCP/UDP semantics, state versus input/frame sync, replay attacks, TLS/HTTPS boundaries, Cheat Engine-style memory tamper, ESP/aimbot defences, relevancy, server validation and telemetry.  
**Minimum Features:** sequence/operation IDs, duplicate/stale command tests, server-side hit validation, relevancy exposure logs, replay-review packet, backend HTTPS audit and anti-cheat design memo.  

**Implementation Steps:**

1. Classify every replicated property/RPC/backend call as durable state, reliable event, unreliable/superseded update or transaction.
2. Add operation IDs or monotonic sequences to fire, reward and inventory commands.
3. Inject duplicate, stale, out-of-order and wrong-owner messages and prove the server rejects or idempotently handles them.
4. Build a hit-validation matrix covering range, line of sight, ammo, fire rate, cooldown, team, target alive/invulnerable and weapon mode.
5. Add a deliberately client-only health/ammo display mutation and prove authoritative server state corrects or rejects it.
6. Log what Actor/target data each client receives before and after relevancy/interest-management changes.
7. Create suspicious-event telemetry for aim acquisition, impossible visibility and repeated near-threshold hits while minimising personal data.
8. Produce a replay-review packet for one accepted and one rejected shot: inputs, timestamps, server validation results, target state and packet/network conditions.
9. Audit backend/mock service calls for HTTPS/TLS use, certificate-validation bypasses and missing idempotency keys.
10. Write a defensive anti-cheat memo: what is prevented, what is only detected, what remains visible to the client and what needs platform/commercial anti-cheat support.

**Debugging Exercises:** trusted client damage, duplicated reward, replayed inventory command, stale fire timestamp, ESP leak through relevancy, false-positive aim threshold, cert validation disabled in a debug helper, telemetry with excessive personal data.  
**Performance Checks:** bandwidth before/after relevancy, CPU cost of validation, telemetry volume, replay-packet size and storage retention.  
**Interview Questions Unlocked:** TCP versus UDP; state sync versus rollback; replay attacks; HTTPS/MITM; Cheat Engine-style tamper; ESP and aimbot defences; privacy-aware anti-cheat.  
**Version / Platform Caveats:** Anti-cheat internals and platform policy require authorised sources. This extension is defensive and does not include exploit or bypass procedures. [SRC-SEC-001] [SRC-SEC-002] [SRC-SEC-003] [SRC-SEC-004] [SRC-SEC-006] [SRC-SEC-007]
