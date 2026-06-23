# Master Execution Roadmap

This is the long-horizon control plan for turning `research.md` into a complete interview curriculum. It does not replace `research.md`; the specification remains the master scope. `STATUS.md` remains the continuation pointer, while this file records dependency order, wave boundaries, and completion tests.

## Operating rules

Every topic cluster follows the same production loop:

1. Define scope, prerequisites, dependent topics, priority and expected depth.
2. Collect a bounded evidence pack, preferring official/API/source material.
3. Write substantive learning content before broadening the source crawl.
4. Add realistic interview questions, strong answer patterns, flashcards and hands-on verification.
5. Add only enough source metadata for claim traceability.
6. Update the graph when the dependency model changes materially.
7. Validate source IDs, JSON, graph edges, duplicate IDs and artefact counts.
8. Update `STATUS.md` with exact continuation point.

No wave is complete when it contains only an outline, source list or empty file.

## Current baseline

Substantive first passes now span standard C++ lifetime/ownership; UE object/C++ idioms; gameplay framework; architecture/patterns/system design; networking; AI/navigation; profiling; rendering; assets/loading/cooking; build/tools; MassEntity; GAS; algorithms; game maths; animation; physics/collision; UI; audio; Niagara/VFX; and Lua/C# integration. The next dependency frontier is the Wave 8 specialist deepening ledger, beginning with standard C++.

## Dependency waves

| Wave | Cluster | Main outcome | Exit evidence |
|---|---|---|---|
| 1 | C++ lifetime and UE object model | Safe ownership/reflection/GC reasoning | Chapters, labs, questions/cards, graph and source pack |
| 2 | Gameplay framework and C++/Blueprint | Correct responsibility/lifecycle placement | Core sandbox and framework interview practice |
| 3 | Networking, AI, profiling, rendering | Diagnose common runtime systems from evidence | Network/AI/render labs and trace-led workflows |
| 4A | Build, modules, plugins, editor tools | Design compile/package-safe runtime/editor boundaries | Tools Plugin Project, module graph and packaging proof |
| 4B | MassEntity/ECS | Choose Actor versus Mass and implement a data-oriented crowd slice | Project 5 with fragments/processors/traits/representation/LOD/debugging |
| 4C | GAS and modern UE5 gameplay systems | Explain and build a bounded ability/effect/tag/prediction slice | GAS chapter, failure matrix and network exercise |
| 5A | Algorithms and data structures | Solve practical game problems and explain complexity/data shape | Implementations, test cases and interview bank |
| 5B | Game maths | Apply vectors/transforms/quaternions/intersections/render spaces | Worked problems, visual tests and flashcards |
| 6A | Animation | Connect gameplay state to animation safely and efficiently | Montage/state machine/root motion/notifies/debugging lab |
| 6B | Physics and collision | Design channels/queries/sweeps/Chaos interactions and debug misses | Collision matrix and reproducible trace/physics failures |
| 6C | UI/CommonUI | Build event-driven LocalPlayer-aware UI without owning gameplay truth | UI profiling and input/focus/lifetime exercises |
| 6D | Audio/Niagara/MetaSounds | Trigger scalable cosmetics with pooling/concurrency/replication judgement | Effects/audio budget and debugging exercises |
| 7 | Lua and C# integration | Explain plugin-specific runtime/tooling interop and lifetime boundaries | Project 10, comparison tables, plugin/version caveats |
| 8 | Specialist deepening | Fill Fast Array/Iris/RepGraph, C++ templates/concurrency/toolchain, shaders/PSOs, advanced assets | Topic-specific evidence and required question counts |
| 9 | Interview synthesis | Turn chapters into schedules, mock loops, role overlays and summary lists | All required curriculum/gap/synthesis artefacts |
| 10 | Final acceptance | Verify every `research.md` deliverable and quantitative minimum | Automated counts plus manual scope/source/version audit |

## Wave 4A: build, modules, plugins, and editor tools

### Knowledge units

- UBT versus compiler/linker versus UHT responsibilities.
- `.Build.cs` module rules and `.Target.cs` target composition/configuration.
- `.uproject`/`.uplugin` descriptors, module type and loading phase.
- Public/Private folders, public/private dependency propagation, API export macros, forward declarations and IWYU.
- Runtime, editor, developer, program and third-party module boundaries.
- Engine plugin versus project plugin versus content-only plugin.
- Module startup/shutdown, registration symmetry and unload safety.
- Generated headers and reflection/build ordering.
- Live Coding/object reinstancing versus legacy Hot Reload; full-restart boundaries.
- Compile/link/load/cook/package failure classification.
- `WITH_EDITOR` versus editor-only modules/data and Shipping proof.
- Editor Utility Widgets/Blueprints, Python editor scripting, commandlets, details/property customisation, asset actions/factories and validation.
- Transactions, dirty packages, save prompts, source control, cancellation, dry run and audit logging for batch tools.

### Deliverables

- `topics/ue_build_modules_plugins_tools.md`.
- Project 6 with runtime/editor modules, plugin descriptor, validation, Editor Utility UI, batch operation and package test.
- At least 12 initial full-format questions and 40 cards.
- Source entries and graph nodes for build pipeline, module boundaries, plugin/runtime-editor split, iteration/reinstancing and scalable editor automation.

### Exit test

The learner can explain why an include compiles locally but fails in another module, why editor code leaks into Shipping, why a plugin module is missing at load/package, what Live Coding can safely patch, and how to make a destructive batch tool undoable/dry-run/CI-capable.

## Wave 4B: MassEntity and ECS

### Knowledge units

- ECS versus Actor/component object composition; data-oriented motivations and costs.
- Entity handles, archetypes, chunks, fragments, tags, shared/chunk fragments.
- Queries, execution contexts, processors, phases/dependencies and deferred commands.
- Traits/templates/spawners, observers and entity lifecycle.
- MassRepresentation, Actor/ISM representation, significance/LOD and simulation tiers.
- Hybrid Actor–Mass ownership, conversion and authoritative state transfer.
- MassGameplay/AI/replication awareness and version/plugin status.
- Debugging query composition, processor order, stale handles and structural changes.

### Deliverables and exit test

Project 5 compares Actor and Mass crowds under the same behaviour/representation budget. The candidate can defend when *not* to use Mass, inspect fragment composition, explain structural changes, and profile simulation separately from representation.

## Wave 4C: GAS and modern UE5 gameplay systems

### Knowledge units

- ASC ownership/avatar model, abilities, specs, activation and cancellation.
- Gameplay Effects, attributes/sets, modifiers, executions and aggregators.
- Gameplay Tags and tag-driven requirements/blocking/state.
- Costs, cooldowns, Gameplay Cues and cosmetic separation.
- prediction keys/windows, local-predicted versus server-only abilities and reconciliation awareness.
- replication modes, attribute replication and common ownership/initialisation bugs.
- Enhanced Input integration and broad awareness of CommonUI, Smart Objects, PCG, Motion Matching/Pose Search, Control Rig and MetaSounds.

### Exit test

The learner can compare a simple bespoke ability component with GAS, trace a failed activation from input to authority/tags/cost/cooldown, and explain prediction without claiming that all ability logic is locally authoritative.

## Wave 5: algorithms, data structures, and maths

### Algorithms track

- complexity, arrays/hash tables/heaps/queues/graphs and operation trade-offs;
- BFS/DFS/Dijkstra/A*/weighted A*, admissible heuristics and path reconstruction;
- sorting/searching, topological order and union-find awareness;
- weighted random, interpolation/smoothing and noise awareness;
- grids/spatial hashes/quadtrees/octrees/BVH/k-d trees;
- broadphase/narrowphase and bounds;
- interest management, snapshots/interpolation, lag compensation and rollback awareness.

### Maths track

- vector operations, projection/reflection, basis and coordinate spaces;
- transforms, matrices, rotators/quaternions, gimbal lock, lerp/slerp;
- ray/plane/sphere/AABB intersections and sweep reasoning;
- velocity/acceleration/impulse/gravity/friction/torque awareness;
- view/clip/NDC/screen spaces, UV/tangent/normal maps, depth and linear/gamma.

### Exit test

Every core concept includes a worked game problem, complexity or numerical caveat, implementation/visual verification, common interview trap and mapping to at least one UE subsystem.

## Wave 6: animation, physics, UI, audio, and VFX

This wave is split because each domain has independent failure tools and performance models. Each subcluster receives its own source pack, chapter section, question/cards and hands-on failure matrix. Integration boundaries receive special focus: gameplay owns truth; animation/UI/effects present and request, physics supplies/query state under authority rules, and replicated durable state is not replaced by cosmetic events.

## Wave 7: Lua and C#

Lua and C# must appear in the graph, schedules, source index, flashcards, question banks, role overlays, gap analysis and Project 10—not only one appendix. Sources must distinguish language facts from UnLua/sluaunreal/Puerts/UnrealCLR or other plugin behaviour. The implementation lab must document native/managed/script ownership, wrapper invalidation, GC interaction, thread rules, reflection/binding generation, debugging, performance and package/deployment caveats.

## Wave 8: specialist deepening ledger

| Existing area | Required follow-ons |
|---|---|
| Standard C++ | templates/SFINAE/concepts, lambdas/type erasure, layout/cache/allocators, concurrency/atomics/job systems, linking/ODR/symbols/sanitisers |
| UE C++ | optional/variant/function refs, interfaces, specifiers/config/save/transient, logging/check/assert, duplicate/find/load helpers, async Blueprint nodes |
| Networking | components/subobjects, Fast Array, RepGraph, Iris, prediction/lag compensation/rollback |
| AI | Smart Objects, Gameplay Tasks, dynamic nav/invokers, utility/GOAP/HTN awareness, MassAI |
| Rendering | render targets/scene capture, shader compilation/PSO caches, mesh draw commands, texture/VT depth, platform captures |
| Profiling | task graph/ParallelFor/async, significance, Networking/Asset Insights, widget/NPC/Actor scale |
| Assets | compression, OFPA, migration, patch/install bundles, cooker/IoStore specialist depth |

## Quantitative completion strategy

The final question minima sum far beyond a normal single bank. Counts will be grown by cluster with no trivia padding. Each addition must test a distinct model, debugging path or design trade-off. The final flashcard target is at least 500 overall *and* the category minima; cards are tracked by `Category:` rather than inferred from file location.

Before final acceptance, automated checks will report:

- questions by required area and template-field completeness;
- cards by required category and duplicate question/answer detection;
- all ten projects and required fields/features;
- graph schema, unique nodes/edges and resolvable source IDs;
- required filenames or documented grouped equivalents;
- source records, dedupe groups and version/conflict/plugin flags;
- broken local links, empty files and unresolved placeholders.

## Wave 9 synthesis artefacts

- 4-, 8-, and 12-week dependency-aware schedules with Mermaid timelines.
- Gap analysis across official docs, source/talks, production reality, jobs and interview anecdotes.
- Role overlays for gameplay, AI, rendering, networking, tools, technical art/design and generalist.
- Mock interview loops and portfolio explanation frameworks.
- Top 75 UE and C++ concepts; Top 50 patterns/system prompts/debug/performance workflows; Top 30 algorithms/maths; anti-pattern and tutorial-only differentiator lists.
- Similar-but-different, plugin-dependent and deprecated/UE4-only tables.

## Completion discipline

“First pass complete” means the topic has a useful evidence-backed chapter, debugging/design workflow and practice, not that every required final question exists. “Project complete” is reserved for the entire acceptance list in `research.md`; until then, `STATUS.md` must expose partial and untouched areas plainly.
