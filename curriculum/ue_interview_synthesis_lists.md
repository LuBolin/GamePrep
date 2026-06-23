# Unreal Engine Interview Synthesis Lists

These lists compress the curriculum into high-yield interview retrieval prompts. They do not replace the source-backed topic chapters. Use them after reading the relevant chapters to rehearse concise, defensible answers.

Source anchors for the core lists include the C++ references, Epic reflection/UObject/pointer/specifier docs, gameplay framework docs, networking/AI/profiling/render/assets/build docs, and the specialist system sources already cited in the topic chapters [SRC-CPP-001] [SRC-CPP-020] [SRC-EPIC-001] [SRC-EPIC-009] [SRC-EPIC-035] [SRC-EPIC-039] [SRC-EPIC-010] [SRC-NET-001] [SRC-AI-002] [SRC-PERF-003] [SRC-RENDER-003] [SRC-ASSET-003] [SRC-BUILD-001] [SRC-MASS-001] [SRC-GAS-001].

## Top 75 UE and C++ Concepts

| # | Concept | What a strong interview answer must include |
|---:|---|---|
| 1 | C++ object lifetime | Lifetime is not scope, storage duration or ownership; name construction, valid use and destruction. |
| 2 | RAII | Tie resource acquisition/release to object lifetime; prefer Rule of Zero where members own correctly. |
| 3 | Rule of Zero/Five | Custom destructor/copy/move choices are a set; avoid accidental shallow ownership. |
| 4 | Move semantics | `std::move` casts to an xvalue; resource transfer depends on the selected move operation. |
| 5 | Moved-from state | Valid but usually unspecified; only rely on documented operations. |
| 6 | `noexcept` move | Containers may prefer copy over throwing move to preserve guarantees. |
| 7 | Unique ownership | `unique_ptr`/`TUniquePtr` express single owner for non-UObject resources. |
| 8 | Shared ownership | Reference counting is a cost and a lifetime policy, not a garbage collector. |
| 9 | Weak shared references | Use weak observation to break cycles and test liveness before use. |
| 10 | Iterator/reference invalidation | Every container operation has an invalidation contract; do not cache handles blindly. |
| 11 | `TArray` | Contiguous value storage with relocation/invalidation concerns; excellent default sequence. |
| 12 | `TSet` | Hash-based uniqueness; do not assume stable order or addresses. |
| 13 | `TMap` | Key/value hash storage; key hash/equality and mutation rules matter. |
| 14 | `FName` | Interned identifier; good for comparison/lookup, not display/localisation. |
| 15 | `FString` | Mutable owned string; useful for manipulation and debugging. |
| 16 | `FText` | Localised display text with history; do not downgrade to string casually. |
| 17 | Template deduction | Forwarding references and reference collapse rules explain many API surprises. |
| 18 | Concepts/SFINAE | Compile-time constraints improve overload selection and diagnostics, but do not prove runtime semantics. |
| 19 | Lambda capture lifetime | Captures can dangle; weak UObject capture and cancellation are often required. |
| 20 | Callable wrappers | Distinguish owning `TFunction`/`std::function` from non-owning `TFunctionRef`. |
| 21 | Atomics/memory order | A race-free program needs a happens-before proof, not `volatile`. |
| 22 | Mutex/condition variable | Protect predicates with the mutex; notifications are not stored state. |
| 23 | Task partitioning | Partition immutable input, merge results on the owning thread and avoid hidden UObject access. |
| 24 | ODR/linkage | Header definitions, module macros and build configs must produce one coherent program. |
| 25 | Sanitizers | ASan/TSan/UBSan are diagnostic tools with platform/instrumentation caveats. |
| 26 | UHT | Generates reflection glue from supported macro-marked declarations; it is not a full C++ compiler. |
| 27 | Reflection macros | `UCLASS`, `USTRUCT`, `UENUM`, `UFUNCTION`, `UPROPERTY` expose selected surfaces to engine systems. |
| 28 | CDO | Class defaults live on the Class Default Object; constructors set defaults, not per-instance world logic. |
| 29 | Default subobjects | Create in constructors with stable names; runtime object creation has different rules. |
| 30 | `NewObject` | Creates UObjects with explicit class, outer, name, flags and optional template; name/outer affect identity. |
| 31 | `SpawnActor` | World-aware Actor spawning with transform, collision and lifecycle concerns. |
| 32 | Outer/package/name/path | Object identity and lifetime context depend on ownership tree and package placement. |
| 33 | Object flags | Flags influence loading, transactions, duplication, transient behaviour and editor/runtime treatment. |
| 34 | UObject GC | Reachability is traced from roots and GC-visible references; ordinary C++ ownership is separate. |
| 35 | `TObjectPtr` | Reflected UObject member pointer vocabulary with GC/editor tooling implications. |
| 36 | `TWeakObjectPtr` | Non-owning UObject observation that can become stale/null; check before use. |
| 37 | Soft object/class pointers | Asset identity without immediate load; resolve/load deliberately and handle residency. |
| 38 | `TStrongObjectPtr` | Native strong reference for scoped/special cases; use carefully to avoid hidden retention. |
| 39 | `UPROPERTY` edit flags | Edit/visible flags describe authoring surface, not runtime authority. |
| 40 | Blueprint property/function flags | Expose the narrow API designers need while preserving invariants. |
| 41 | Config/SaveGame/Transient | Persistence flags are consumed by different systems; name the serialisation path. |
| 42 | DuplicateTransient | Duplication semantics differ from save/load and runtime mutation. |
| 43 | Metadata | Mostly editor/Blueprint tooling guidance; do not build runtime rules on metadata alone. |
| 44 | `UINTERFACE` | Reflected interface pair for UObject/Blueprint dispatch. |
| 45 | Native abstract interface | Ordinary C++ polymorphic contract; no reflection/Blueprint dispatch by itself. |
| 46 | Delegates | Choose static/native/dynamic/multicast by lifetime, reflection and serialisation needs. |
| 47 | Dynamic delegates | Reflection/Blueprint/serialisation capable but usually costlier and more constrained. |
| 48 | `UE_LOG`/categories | Categories and verbosity make logs filterable; logs are diagnostic evidence. |
| 49 | `UE_LOGFMT` | Structured formatting improves readability but needs the correct header and field rules. |
| 50 | `check`/`verify`/`ensure` | Use for programmer invariants and diagnostics; do not model recoverable game flow as asserts. |
| 51 | Actor | World object with transform, components, ticking/replication/lifecycle. |
| 52 | Component | Reusable behaviour/representation attached to an Actor with registration and ownership boundaries. |
| 53 | Pawn/Character | Controllable body; Character adds movement assumptions and networked movement support. |
| 54 | Controller | Decision/input owner; separate from the controlled body. |
| 55 | GameMode | Server-only rules and match flow. |
| 56 | GameState | Replicated match state visible to clients. |
| 57 | PlayerController | Connection/input/control state, often owner-only. |
| 58 | PlayerState | Replicated participant state that can outlive pawns. |
| 59 | Subsystems | Lifetime-scoped service hosts; choose Engine/GameInstance/World/LocalPlayer deliberately. |
| 60 | Blueprint API boundary | Expose stable authoring knobs and events; keep authority and invariants in C++/server-owned code. |
| 61 | Network authority | Server owns authoritative state unless a system explicitly predicts local presentation. |
| 62 | Owning connection | Determines client RPC routing and owner-only replication conditions. |
| 63 | Replicated properties/OnRep | State transport and client reaction; do not assume immediate or ordered relation to unrelated RPCs. |
| 64 | RPC reliability | Delivery guarantee is not gameplay-state correctness or late-join recovery. |
| 65 | Relevancy/dormancy | Per-connection update policy and bandwidth optimisation, with wake/change pitfalls. |
| 66 | Character movement prediction | Client predicts, server validates/replays/corrects; clients are not authoritative. |
| 67 | Behaviour Tree | Decision graph with tasks/decorators/services and Blackboard-driven reactivity. |
| 68 | StateTree | Hierarchical state/transition model; not a universal BT replacement. |
| 69 | Perception/EQS/Nav | Separate sensing, query/scoring, path planning and local movement/avoidance. |
| 70 | Unreal Insights/stat triage | Identify limiting thread/resource before fixing. |
| 71 | Render cost dimensions | Submission, geometry, material, pixel, shader, bandwidth and synchronization can bottleneck separately. |
| 72 | Asset Manager | Primary Asset IDs, bundles and rules provide intentional load/cook management. |
| 73 | UBT/modules/plugins | Build dependency, API export and runtime/editor boundaries shape every compile/package failure. |
| 74 | GAS | Ability/effect/tag/prediction framework; adopt only when its model pays for complexity. |
| 75 | MassEntity | Data-oriented ECS for scale; use when workload and representation fit, not as a fashion default. |

## Top 50 Patterns, System Prompts and Workflows

| # | Prompt | Expected answer shape |
|---:|---|---|
| 1 | Design health and damage. | Authority, data model, damage pipeline, prediction/cosmetics, UI projection, save/replication, tests. |
| 2 | Design inventory/equipment. | Item definitions versus instances, replication audience, equip authority, asset refs, persistence, UI. |
| 3 | Design interaction. | Trace/overlap source, candidate filtering, server validation, prompt UI, latency and accessibility. |
| 4 | Design a weapon system. | Fire input, ammo/state, prediction, traces/projectiles, effects, replication and tuning data. |
| 5 | Design abilities. | Bespoke component versus GAS, costs/cooldowns/tags, prediction, cancellation and cues. |
| 6 | Design quests/objectives. | Data definitions, runtime progress, events, save schema, network visibility and editor tooling. |
| 7 | Design save/load. | Stable IDs, schema migration, authoritative state, transient exclusions and async IO/cook concerns. |
| 8 | Design UI projection. | Gameplay owns truth; UI binds to view models/events and handles lifetime/local-player scope. |
| 9 | Choose Actor/Component/subsystem. | Lifetime, world ownership, per-instance state, reuse, ticking, replication and editor workflow. |
| 10 | Choose C++ or Blueprint. | Invariants/hot paths/authority in C++; authoring/orchestration in Blueprint; profile before rewriting. |
| 11 | Choose hard or soft asset references. | Dependency closure, load time, residency, cook inclusion, async handle and failure UX. |
| 12 | Choose Tick/timer/event. | Frequency, ordering, burst cost, lifetime cancellation, latency and measurement. |
| 13 | Choose delegate type. | Reflection need, serialisation, binding lifetime, multicast semantics and performance. |
| 14 | Choose container. | Operation pattern, order, lookup, invalidation, memory locality and replication/serialisation needs. |
| 15 | Choose pointer wrapper. | UObject versus non-UObject, ownership versus observation, GC visibility, load path and async safety. |
| 16 | Choose BT/StateTree/FSM/utility. | Decision shape, reactivity, designer workflow, debugging, scale and state explosion. |
| 17 | Choose Actor crowd or Mass. | Entity count, interaction richness, representation, LOD, authority, debugging and content pipeline. |
| 18 | Choose GAS or custom. | Tags/effects/prediction/stacking need versus complexity, team familiarity and debugging cost. |
| 19 | Diagnose UObject crash. | Validity, root/reference chain, GC visibility, async callback lifetime, world teardown. |
| 20 | Diagnose missing property in Blueprint. | UHT parse, macro specifiers, module dependency, generated files, editor reload and metadata. |
| 21 | Diagnose CDO/default issue. | Constructor defaults, Blueprint class defaults, instance overrides, construction script and load. |
| 22 | Diagnose packaging-only asset failure. | Hard/soft refs, cook rules, Primary Asset IDs, redirectors, DDC and platform paths. |
| 23 | Diagnose linker/module failure. | Public/private dependencies, API macros, exports, unity/IWYU masking and target rules. |
| 24 | Diagnose Live Coding weirdness. | Header/layout/reflection changes, reinstancing limits, stale objects and restart boundary. |
| 25 | Diagnose client RPC not firing. | Authority, owning connection, actor channel/relevancy, reliable saturation and call site. |
| 26 | Diagnose `OnRep` not called. | `GetLifetimeReplicatedProps`, actual value change, condition, relevancy, dormancy and owner. |
| 27 | Diagnose bandwidth spike. | Actor/property/RPC attribution, frequency, relevancy, dormancy, payload size and compression. |
| 28 | Diagnose AI not moving. | Controller possession, Blackboard, BT task status, navmesh, path result, movement component and collision. |
| 29 | Diagnose bad EQS result. | Generator, context, test weights/filtering, run mode, item count and debug visualisation. |
| 30 | Diagnose collision miss. | Object type, trace channel, response, simple/complex, query shape, transforms and ignored actors. |
| 31 | Diagnose physics instability. | Timestep, mass/inertia, constraints, collision shapes, forces, substeps and network authority. |
| 32 | Diagnose animation glitch. | Gameplay source state, AnimBP snapshot, state transition, montage slots, notifies and sync groups. |
| 33 | Diagnose UI stale data. | Widget/Slate/activation lifetime, binding/event ownership, LocalPlayer, async asset request and list recycling. |
| 34 | Diagnose audio not playing. | Request, component lifetime, concurrency, attenuation/listener, routing/submix, stream cache and volume. |
| 35 | Diagnose Niagara missing effect. | Spawn path, component pooling, pre-cull, bounds, Effect Type cull/scalability and asset load. |
| 36 | Diagnose CPU frame spike. | `stat unit`, Insights Timing, game thread tasks, ticks, allocation, GC and contention. |
| 37 | Diagnose GPU frame spike. | GPU visualizer/profile, pass cost, resolution, material/pixel/overdraw, shadows, Lumen/VSM and bandwidth. |
| 38 | Diagnose memory growth. | Object counts, references, assets/residency, allocator churn, caches, pools and leak repro. |
| 39 | Diagnose loading hitch. | Sync load call sites, hard refs, async handle lifetime, streaming source, DDC and IO. |
| 40 | Use Component pattern well. | Component owns cohesive behaviour but not unrelated global truth. |
| 41 | Use Observer/delegate well. | Decouple notification from ownership; unbind or use weak binding. |
| 42 | Use Command pattern well. | Represent input/actions as data for buffering, undo, replay or networking. |
| 43 | Use State/Strategy well. | Separate policy variation from object identity and avoid giant switches. |
| 44 | Use Factory/Prototype well. | Spawn configured instances from data while preserving lifecycle/ownership rules. |
| 45 | Use Object Pool well. | Pool expensive reusable presentation objects; reset state and avoid pooling wrong identities. |
| 46 | Use Service Locator/subsystem well. | Scope service lifetime explicitly and keep dependencies testable. |
| 47 | Use Event Queue well. | Buffer cross-system events with ordering/lifetime rules; do not hide authority. |
| 48 | Use Dirty Flag well. | Recompute derived state lazily with invalidation discipline. |
| 49 | Use Double Buffer well. | Separate read/write phases for rendering/simulation/threaded handoff. |
| 50 | Use data-driven design well. | Data tunes policy and content; code still enforces invariants and versioning. |

## Top 30 Algorithms and Maths Prompts

| # | Prompt | Must include |
|---:|---|---|
| 1 | Big-O for a game system | State operation, input size, expected/worst/amortised case and distribution. |
| 2 | Array versus linked list | Cache locality and iteration usually dominate; linked lists rarely win by default. |
| 3 | Hash table failure | Hash/equality, load factor, rehash invalidation and adversarial keys. |
| 4 | Heap/priority queue | Use for repeated min/max extraction; mention decrease-key workaround if absent. |
| 5 | Ring buffer | Fixed-capacity producer/consumer history with wrap and overwrite policy. |
| 6 | BFS versus DFS | BFS shortest unweighted paths; DFS traversal/backtracking/cycle detection. |
| 7 | Dijkstra | Non-negative weighted shortest path with priority queue and relaxation. |
| 8 | A* | Dijkstra plus admissible/consistent heuristic; path optimality depends on heuristic. |
| 9 | Weighted A* | Faster but can sacrifice optimality; useful when bounded suboptimality is acceptable. |
| 10 | Topological sort | DAG ordering; detects cycles in dependency graphs. |
| 11 | Union-find | Fast dynamic connectivity without deletion. |
| 12 | Spatial hash | Uniform grid bucket acceleration; distribution/cell size are critical. |
| 13 | Quadtree/octree/k-d tree | Hierarchical spatial partition; update/query distribution determines fit. |
| 14 | BVH | Bounds hierarchy for broadphase/raycast/collision; rebuild/refit trade-off. |
| 15 | Broadphase/narrowphase | First find candidate pairs, then exact tests. |
| 16 | Weighted random | Prefix sums/alias method; verify probabilities with tests. |
| 17 | Deterministic replay | Seed, input log, fixed/update policy and nondeterminism audit. |
| 18 | Floating-point equality | Use tolerances appropriate to scale and avoid unstable branch thresholds. |
| 19 | Vector dot product | Projection, angle, facing, signed/unsigned decisions. |
| 20 | Vector cross product | Perpendicular axis, orientation and handedness; Unreal is left-handed Z-up. |
| 21 | Projection/rejection | Split motion or influence along/away from a normal. |
| 22 | Coordinate spaces | Always name local/world/view/clip/screen/tangent and point/vector semantics. |
| 23 | Matrix/transform composition | Order and non-uniform scale can change results; test known basis vectors. |
| 24 | Quaternion versus Rotator | Quaternions avoid gimbal interpolation issues; Rotators are useful authoring vocabulary. |
| 25 | Slerp versus Lerp | Constant angular interpolation versus linear blend/normalise trade-offs. |
| 26 | Exponential smoothing | Frame-rate-independent approach to target with time constant, not fixed per-frame alpha. |
| 27 | Ray-plane intersection | Denominator near zero, signed distance and behind-origin cases. |
| 28 | Ray/AABB slab test | Per-axis intervals, parallel axes and near/far ordering. |
| 29 | Sweep reasoning | Shape over motion interval, initial overlap and depenetration policy. |
| 30 | Rendering spaces and colour | Tangent/world/view/clip, depth linearisation and sRGB/linear conversions. |

## Anti-Patterns to Recognise Quickly

| Anti-pattern | Why it is bad | Better direction |
|---|---|---|
| "God Actor" | Owns unrelated state, UI, networking and effects; hard to test/replicate | Split truth, presentation and services by lifetime/authority. |
| Blueprint as database | Runtime truth hidden in graphs/defaults without schema/versioning | Use data assets/config/save models with validation. |
| `EditAnywhere BlueprintReadWrite` everywhere | Destroys invariants and unclear authoring authority | Expose narrow edit/Blueprint surface. |
| Raw UObject async capture | Callback can run after destruction/world teardown | Weak pointer, cancellation, game-thread apply and lifecycle owner. |
| Dynamic delegates everywhere | Reflection cost/constraints when native delegate would do | Choose delegate by reflection/lifetime need. |
| Reliable RPC spam | Can saturate channels and still not model durable state | Replicate state, batch, rate-limit and design idempotently. |
| Cosmetic multicast as truth | Late joiners and relevancy lose state | Replicate durable state; multicast only presentation events. |
| UI owning gameplay state | Breaks authority, save/load and multiplayer | UI projects model state and sends requests. |
| Animation notify as authority | Notifies can be skipped/timing-dependent | Use validated gameplay windows and server authority. |
| Pooling identity-heavy Actors | Reset bugs and stale references outweigh spawn cost | Pool presentation or expensive stateless pieces. |
| Asset hard-reference chains | Hidden load/cook bloat | Use soft references/Asset Manager rules deliberately. |
| Profiling by FPS | FPS hides frame-time scale and bottleneck domain | Use ms, traces and limiting-resource classification. |
| Optimising without a repro | Changes cannot be trusted | Build deterministic scenario and measure before/after. |
| Rendering by folklore | Draw-call/shader myths applied blindly | Capture GPU passes and content cost dimensions. |
| Plugin-as-core thinking | Version/support/platform assumptions leak into design | Label plugin status and verify target branch. |
| ECS for every object | Loses rich object workflow and increases complexity | Use Mass for scale/data shape; Actors for interactive identity. |
| GAS for trivial actions | Complexity without payoff | Use custom component unless GAS features are needed. |
| Ignoring packaged builds | Editor-only success masks cook/path/module errors | Test cooked/package targets early. |
| Threading by hope | UObject access/data races appear under load | Snapshot immutable data, synchronize and apply on owner thread. |
| Source-less version claims | Maintained docs evolve across UE5 | Pin source/API/version before final claim. |

## Tutorial-Only Differentiators

Use these to distinguish interview-ready understanding from tutorial recall.

| Tutorial answer | Interview-ready differentiator |
|---|---|
| "Create an Actor and add components." | Explain component registration, ownership, replication, ticking, construction defaults and destruction. |
| "Use `UPROPERTY` so GC sees it." | Explain which references are GC-visible, which are editor/Blueprint/serialisation/config flags and which containers are safe. |
| "Call a Server RPC." | Explain authority, owning connection, validation, relevancy, reliability and state recovery. |
| "Use Behaviour Tree for AI." | Explain Blackboard events, aborts, services, task lifetime, perception memory, EQS cost and path/avoidance split. |
| "Use Widget Binding." | Explain update frequency, invalidation, view-model/event alternatives and LocalPlayer/widget lifecycle. |
| "Use object pooling for performance." | Explain reset invariants, stale delegates, hidden references, memory residency and whether spawn cost was measured. |
| "Use soft references." | Explain async load handle lifetime, cook inclusion, bundles, failure UI and residency release policy. |
| "Use Nanite/Lumen." | Explain content/platform/version limits and measure actual pass costs. |
| "Use Mass for crowds." | Explain fragments/archetypes/processors, representation LOD, hybrid ownership and debugging trade-offs. |
| "Use GAS for abilities." | Explain ASC owner/avatar, grants/spec handles, tags, effects, prediction, cancellation and replication mode. |
| "Use Live Coding." | Explain restart boundaries for headers/reflection/layout/static state. |
| "Use a plugin for Lua/C#." | Explain VM/runtime hosting, binding generation, GC bridge, reload, thread rules, package and platform support. |

## Final Oral Drill

For each concept above, rehearse this 45-second shape:

1. One-sentence definition.
2. Why it exists.
3. One common bug.
4. How to debug or verify it.
5. One version/plugin/source caveat when relevant.

If an answer cannot include items 3 and 4, return to the chapter and project exercise before moving on.
