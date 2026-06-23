# Unreal Engine Interview Gap Analysis

This gap analysis compares what the current source pack covers well against what Unreal interviews and production work usually demand. It is a curriculum control document, not a claim that every job-ad or interview-anecdote source has already been exhaustively crawled. The current repository is strongest on official Epic/API documentation and established technical references; a later Wave 9/10 pass should add a labelled job-description and interview-anecdote corpus if final quantitative coverage requires it.

## Evidence Classes

| Evidence class | Current repository strength | What it is good for | What it misses |
|---|---|---|---|
| Official Epic docs | Strong | Concepts, tool names, supported workflows, API entry points, target-version anchors [SRC-EPIC-001] [SRC-NET-001] [SRC-PERF-001] [SRC-BUILD-001] | Production trade-offs, failure frequency, studio conventions, exact branch divergences when maintained docs resolve to later UE5 |
| Official API docs | Strong | Function/class existence, signatures to verify, lifecycle hints [SRC-EPIC-004] [SRC-EPIC-043] [SRC-EPIC-044] | Full behavioural contracts, call-order guarantees, performance costs, source-level edge cases |
| Engine source | Partial | Exact implementation truth for UE5.3-UE5.6, generated code, subsystem order, branch-specific changes | Not yet systematically indexed; source claims must still be branch-checked before final acceptance |
| Official talks/conference material | Light | Architectural intent, historical context, performance heuristics | Needs a targeted pass for renderer, networking, Mass/GAS and tooling talks |
| Studio engineering blogs | Light | Production constraints, large-scale failures, optimisation narratives | Needs source collection; current docs do not yet represent a broad studio sample |
| Job descriptions | Not yet a corpus | Role weighting and common screening terms | Current role overlays are synthesis from curriculum scope, not a sourced job-ad frequency study |
| Interview blogs/anecdotes | Supplementary only | Common phrasing and traps | Often stale or shallow; should never override official/API/source evidence |
| Practical project needs | Strong within this repo | Hands-on proof, failure matrices, debugging/profiling practice | Implementation evidence still has to be produced by a learner in an engine project |

## Topics Official Docs Cover Well

| Topic | Why docs are strong | Remaining interview risk |
|---|---|---|
| Reflection/UHT/UObject basics | Epic docs explain reflection macros, UHT and object handling clearly enough to build the mental model [SRC-EPIC-001] [SRC-EPIC-002] [SRC-EPIC-003] | Candidates still need to connect this to generated-code failures, CDO/default-subobject rules and GC reachability. |
| UObject pointer selection | Dedicated pointer documentation gives a useful decision table for `TObjectPtr`, weak, soft and strong object pointers [SRC-EPIC-009] | Many interviews probe misuse: `TSharedPtr` for UObjects, raw pointer lifetime, async callbacks and containers not visible to GC. |
| Gameplay framework roles | Gameplay Quick Reference, Actor/Component docs and GameMode/GameState docs cover core responsibilities [SRC-EPIC-010] [SRC-EPIC-011] [SRC-EPIC-012] [SRC-EPIC-013] | The hard part is state placement under multiplayer, persistence, UI projection and respawn/lifecycle changes. |
| Networking baseline | Networking overview, property replication, RPCs, ownership and execution-order docs give strong P0/P1 coverage [SRC-NET-001] [SRC-NET-003] [SRC-NET-004] [SRC-NET-005] [SRC-NET-006] | Specialist interviews quickly move into Fast Array, subobjects, RepGraph, Iris, prediction and bandwidth evidence. |
| AI tools | Behaviour Tree, Perception, EQS, NavMesh and AI debugging docs are practical and tool-oriented [SRC-AI-002] [SRC-AI-004] [SRC-AI-005] [SRC-AI-006] [SRC-AI-008] | Production questions ask about architecture, update frequency, scalable perception, dynamic nav and when not to use each tool. |
| Profiling workflow | Stat commands, Insights and performance intro material support evidence-first diagnosis [SRC-PERF-001] [SRC-PERF-002] [SRC-PERF-003] | Interviews expect the candidate to choose the next capture and avoid FPS-only reasoning. |
| Build/modules/plugins | UBT, modules, plugin docs and Live Coding docs are good anchors [SRC-BUILD-001] [SRC-BUILD-002] [SRC-BUILD-005] [SRC-BUILD-006] | Real failures often involve dependency leakage, editor-only code, package differences and local build settings. |
| Assets/loading/cooking | Hard/soft reference, Asset Manager, packaging, chunks, DDC and World Partition docs are strong enough for a first pass [SRC-ASSET-001] [SRC-ASSET-003] [SRC-ASSET-005] [SRC-ASSET-008] [SRC-ASSET-010] | Candidates need packaged-build evidence and dependency-closure thinking, not only editor success. |

## Topics Official Docs Explain Poorly or Incompletely

| Topic | Gap | How this curriculum compensates |
|---|---|---|
| "Specifier means what?" | Specifier pages list flags, but the learning problem is mapping each flag to the consuming system: UHT, editor, Blueprint, serialisation, config, duplication, SaveGame, GC, networking or cook [SRC-EPIC-032] [SRC-EPIC-035] [SRC-EPIC-036] | `topics/ue_cpp_idioms.md` organises specifiers by consumer and asks the learner to name the lifecycle. |
| Generated-code failures | UHT docs explain tooling, but not every practical failure mode: missing generated header, stale intermediate files, unsupported reflection type, module dependency leakage [SRC-EPIC-002] [SRC-BUILD-007] | Build/tools and UE C++ chapters include generated-code debugging workflows and module-boundary diagnosis. |
| UObject threads | Docs establish object and GC concepts, but thread-safe UObject access is mostly a production discipline built from object lifetime, task and animation-thread boundaries [SRC-EPIC-003] [SRC-EPIC-005] [SRC-CPP-026] [SRC-ANIM-015] | The curriculum uses snapshot/apply-back patterns and marks branch/source checks for exact thread APIs. |
| Runtime versus editor metadata | Metadata docs are useful but can encourage overreliance on editor-only metadata for runtime rules [SRC-EPIC-036] | Chapters label metadata as tooling/authoring guidance and require runtime invariants in code/data. |
| Replication ordering and protocols | Execution-order docs are unusually valuable, but designing robust protocols still requires synthesis [SRC-NET-005] | Networking questions require idempotent state design, version counters and avoiding order-dependent RPC/property coupling. |
| Renderer internals | RDG docs are strong but not enough for shader compilation, PSO caches, mesh draw commands and platform-capture interpretation [SRC-RENDER-003] | Rendering chapter gives first-pass cost dimensions and flags shaders/PSOs as specialist follow-ons. |
| UI lifetime | UMG/Slate/CommonUI docs are spread across architecture, API and plugin pages [SRC-UI-001] [SRC-UI-002] [SRC-UI-009] [SRC-UI-010] | UI chapter separates UObject widget, Slate widget and activation/input lifetime. |
| Animation thread boundaries | Animation docs mention worker-thread and property-access concerns, but implementation safety is graph/project-specific [SRC-ANIM-004] [SRC-ANIM-015] | Animation chapter frames runtime data as snapshots, not arbitrary Actor reads from animation evaluation. |

## Topics Commonly Asked in Interviews but Hard to Learn From Docs Alone

| Interview topic | Why docs are insufficient alone | Strong answer needs |
|---|---|---|
| "Why did this UObject get destroyed?" | Docs explain GC reachability, but interviews combine GC, raw pointers, timers, async delegates, latent actions and container visibility [SRC-EPIC-005] [SRC-EPIC-009] | Root/reference chain, pointer-wrapper selection, async cancellation and an inspection/debug plan. |
| "Where should this gameplay state live?" | Framework docs list classes, but production state placement depends on authority, visibility, lifetime, respawn, save/load and UI needs [SRC-EPIC-010] [SRC-EPIC-013] | A responsibility matrix and a reasoned rejection of tempting wrong owners. |
| "Debug a replication bug." | Networking docs define mechanisms, but the diagnosis path spans authority, ownership, relevance, dormancy, property registration, RPC route and packet evidence [SRC-NET-003] [SRC-NET-004] [SRC-NET-008] [SRC-NET-010] | Ordered triage with logs, Net Profiler/Insights and a minimal repro. |
| "Optimise this frame spike." | Performance docs name tools; interviews test whether the candidate identifies the limiting thread/resource before changing content [SRC-PERF-001] [SRC-PERF-003] | A capture plan, a hypothesis, one variable changed, and before/after frame-time evidence. |
| "Blueprint or C++?" | Blueprint docs explain exposure; production answers require API design, designer iteration, perf, authority, testing and invariant placement [SRC-EPIC-030] [SRC-EPIC-031] | Blueprint for orchestration/authoring, C++ for stable invariants/hot paths/authority, with profiling caveats. |
| "Actor crowd or MassEntity crowd?" | Mass docs describe entities/fragments/processors, but the choice depends on representation, ownership, content workflow, interaction richness and debugging cost [SRC-MASS-001] [SRC-MASS-002] | A workload threshold, hybrid bridge plan and "do not use Mass when Actors are enough" answer. |
| "GAS or custom ability system?" | GAS docs are extensive, but interviews want adoption judgment and lifecycle traps [SRC-GAS-001] [SRC-GAS-002] [SRC-GAS-003] | Compare tag/effect/prediction requirements to complexity, team skill and debugging burden. |

## Topics Where Blogs Are Often Outdated

This section is intentionally conservative because the repository has not yet built a broad blog corpus. The risks below are based on version labels and official-source cautions already present in the source ledger.

| Topic | Common outdated blog pattern | Safer study rule |
|---|---|---|
| Hot Reload versus Live Coding | Older advice treats Hot Reload as normal C++ iteration or ignores object reinstancing limits | Prefer Live Coding docs for target-era iteration, but restart/rebuild after header/reflection/layout changes [SRC-BUILD-006]. |
| UE4 Actor lifecycle posts | UE4-era call-order diagrams are copied into UE5 answers without checking target source | Use lifecycle diagrams as conceptual maps and verify exact UE5.3-UE5.6 hooks in API/source [SRC-EPIC-015] [SRC-EPIC-016]. |
| `UProperty` terminology | Old posts use deprecated `UProperty` names or pre-`TObjectPtr` assumptions | Use UE5.6 specifier and pointer docs, with target branch checks [SRC-EPIC-007] [SRC-EPIC-009] [SRC-EPIC-035]. |
| Networking roles | Older tutorials conflate role with ownership or assume role values solve RPC routing | Keep role, authority and owning connection separate [SRC-NET-002] [SRC-NET-003]. |
| Rendering feature support | Posts about Nanite, Lumen, VSM and platform support age quickly | Treat support matrices and limits as version/platform-sensitive [SRC-RENDER-005] [SRC-RENDER-006] [SRC-RENDER-007]. |
| CommonUI/MVVM | Plugin examples are often version-specific and presented as mandatory UI architecture | Label CommonUI/MVVM plugin-dependent and choose them by project need [SRC-UI-009] [SRC-UI-012]. |
| Niagara lightweight/GPU collision | Maintained pages and posts can describe UE5.7 or experimental capabilities | Pin UE5.4+ lightweight emitters and experimental collision paths before claiming support [SRC-FX-010] [SRC-FX-011]. |
| Lua/C# plugins | README snippets are copied without target branch/platform/package proof | Treat UnLua, sluaunreal and UnrealCLR as third-party/commit-specific [SRC-SCRIPT-002] [SRC-SCRIPT-003] [SRC-SCRIPT-005]. |

## UE4/UE5 Differences That Confuse Learners

| Area | Confusion | Correct framing |
|---|---|---|
| Actor lifecycle | UE4 lifecycle docs are still useful, but exact UE5 hook order can differ | Label UE4-era sources and verify exact UE5.3-UE5.6 code paths before relying on ordering [SRC-EPIC-015]. |
| Object pointers | UE5 guidance around `TObjectPtr` changes how reflected UObject members are discussed | Use the UE5 pointer matrix and distinguish GC visibility from C++ ownership [SRC-EPIC-009]. |
| Large World Coordinates | "UE5 uses doubles, so precision is solved" | LWC changes coordinate precision but not every conversion, rendering path or local-space need [SRC-MATH-002]. |
| World Partition | "World Partition replaces all level streaming" | World Partition is central for large worlds; sublevel/level streaming remains a valid bounded design [SRC-ASSET-009] [SRC-ASSET-010]. |
| Nanite/Lumen/VSM | "UE5 feature exists" becomes "every platform/project should use it" | Treat feature status, content support and performance targets as platform/version-sensitive [SRC-RENDER-005] [SRC-RENDER-007]. |
| StateTree and modern AI | "StateTree replaces Behaviour Trees" | Treat BT, StateTree and FSMs as different models with project-specific fit [SRC-AI-002] [SRC-AI-003]. |
| Chaos/networked physics | "Networked physics modes are baseline UE5 interview knowledge" | Keep modern network physics as specialist and highly version-sensitive until target source confirms it [SRC-PHYS-010]. |
| Animation tooling | "Motion Warping/Control Rig/Pose Search are always runtime defaults" | Label feature/plugin/tool status and separate authoring/runtime/authority [SRC-ANIM-014] [SRC-ANIM-018]. |

## Overrepresented in Interviews Relative to Daily Work

| Topic | Why it is overrepresented | How to answer without bluffing |
|---|---|---|
| C++ move/forwarding/template minutiae | Easy to ask in isolation and reveals language precision | Explain the exact rule you know, then tie it to UE APIs, ownership and bugs. |
| Manual memory versus GC | Interviewers test whether candidates separate C++ RAII, smart pointers and UObject GC | Use separate ownership domains and examples. |
| RPC and replication semantics | Network bugs are expensive and common, even on non-network-specialist teams | Know authority/ownership/relevance/order basics; label advanced prediction/Iris as specialist. |
| Draw calls/shader complexity | Common shorthand for rendering performance | Expand into Game/Render/RHI/GPU, submission, material, pixel, bandwidth and capture evidence. |
| GAS | Many roles ask about it even if the studio uses a custom system | Give adoption criteria and lifecycle traps rather than pretending every game should use GAS. |
| MassEntity | Increasingly fashionable but not universal | Explain data-oriented benefits and debugging/content costs; do not force ECS into small Actor-rich features. |
| Lua/C# | Asked by studios with scripting layers or tools teams, but plugin-specific | State that Unreal core is C++/Blueprint and discuss integration boundaries plus plugin proof. |

## Underrepresented in Docs Relative to Interview Relevance

| Topic | Why it matters | Repository response |
|---|---|---|
| Debugging workflow | Interviews often ask "what do you check first?" not "define this feature" | Every topic chapter includes common bugs and debugging/profiling workflows. |
| Production state placement | Docs describe classes, not the full authority/lifetime/visibility/persistence matrix | Architecture chapter and project specs require state-placement justifications. |
| Packaged-build failures | Editor success hides cooking, dependency and platform failures | Asset/build chapters and Project 6 require package-safe validation. |
| Version-sensitive claims | Maintained docs can surface later UE5 features | Source ledger records version notes and flags branch-check items. |
| Cross-system bugs | Real issues span gameplay, UI, animation, networking and assets | Hands-on projects use failure matrices that cross subsystem boundaries. |
| Interview answer patterns | Docs do not teach concise answer structures | Question banks include strong answer pattern templates. |
| Evidence presentation | Interviews reward candidates who can show traces/logs/results | Study schedules require portfolio evidence packets. |

## Plugin-Dependent Areas Often Presented as UE Core

| Area | Plugin/status risk | Safer wording |
|---|---|---|
| Gameplay Ability System | Engine plugin; widely used but not mandatory | "GAS is a powerful plugin framework for tag/effect/prediction-heavy ability designs; a bespoke component can be better for small/simple teams." [SRC-GAS-001] |
| MassEntity/MassGameplay | UE5 plugin ecosystem; APIs and tools evolve | "Mass is data-oriented and useful for large homogeneous simulations; use Actors for rich object identity and low counts." [SRC-MASS-001] [SRC-MASS-002] |
| CommonUI and MVVM | Plugin-dependent; MVVM labelled Beta in current docs | "Use when multiplatform input/layering or binding architecture justifies it, not as mandatory UMG." [SRC-UI-009] [SRC-UI-012] |
| Animation Budget Allocator | Plugin-dependent | "A scalability tool that must be tested against animation quality and gameplay event requirements." [SRC-ANIM-017] |
| Control Rig/Motion Warping/Pose Search | Feature/plugin/tool version-sensitive | "Modern animation tooling; verify runtime target and authoring pipeline." [SRC-ANIM-014] [SRC-ANIM-018] |
| MetaSounds/Quartz | Audio feature/version/platform-sensitive | "Powerful procedural/timing systems; still profile polyphony, routing, stream cache and output latency." [SRC-AUDIO-007] [SRC-AUDIO-008] |
| Niagara lightweight emitters/GPU ray-tracing collision | UE5.4+ or experimental/current-doc-sensitive | "Specialist VFX performance/collision awareness; pin target support." [SRC-FX-010] [SRC-FX-011] |
| Lua/C# runtime integration | Third-party/studio-owned | "Evaluate plugin commit, platform, packaging, GC bridge, debugging and reload support." [SRC-SCRIPT-002] [SRC-SCRIPT-005] |

## Similar But Different

| Pair | Similarity | Crucial difference | Interview trap |
|---|---|---|---|
| Hot Reload vs Live Coding | Both change code during editor iteration | Live Coding patches binaries at runtime and has documented reinstancing limits; neither replaces clean rebuild/restart for many header/reflection/layout changes [SRC-BUILD-006] | Saying "I can hot reload any C++ change safely." |
| `TSharedPtr` vs UObject GC ownership | Both can appear to "keep something alive" | `TSharedPtr` reference-counts ordinary C++ objects; UObject lifetime is traced by GC-visible references/rooting [SRC-CPP-007] [SRC-EPIC-009] | Wrapping a UObject in `TSharedPtr`. |
| GameMode vs GameState | Both are match-level concepts | GameMode is server-only rules; GameState is replicated match state [SRC-EPIC-013] | Putting client-visible scoreboard state only in GameMode. |
| PlayerController vs PlayerState | Both relate to a player | PlayerController is connection/input/control; PlayerState is replicated participant state that can outlive pawns [SRC-EPIC-010] [SRC-EPIC-014] | Putting public score/name in PlayerController for all clients. |
| Behaviour Tree vs StateTree | Both model decisions | BTs are tree/task/decorator/service and event-driven blackboard flows; StateTree models hierarchical state/transition selection [SRC-AI-002] [SRC-AI-003] | Claiming StateTree universally replaces BTs. |
| Pathfinding vs avoidance | Both influence movement | Pathfinding plans a route through nav data; avoidance resolves local dynamic crowd/obstacle conflicts [SRC-AI-006] [SRC-AI-007] | Tweaking avoidance to fix a bad path or vice versa. |
| Draw calls vs shader complexity | Both affect rendering cost | Draw/submission cost and pixel/material/shader cost can bottleneck different stages [SRC-RENDER-004] [SRC-RENDER-010] | Optimising draw count when GPU pixel cost is the issue. |
| Hard reference vs soft reference | Both point to assets/classes | Hard references load/include dependencies; soft references identify assets that can be loaded on demand [SRC-ASSET-001] [SRC-ASSET-002] | Believing a soft pointer means the asset is already resident. |
| Actor crowd vs MassEntity crowd | Both can represent many agents | Actors carry UObject identity/components/ticking; Mass uses fragments/archetypes/processors for data-oriented batch work [SRC-MASS-001] | Migrating small interactive agents to Mass without a workload reason. |
| Blueprint event vs delegate | Both can call script/user logic | Blueprint events are overridable/reflected API points; delegates are callable lists/events with static/dynamic/lifetime binding choices [SRC-EPIC-028] [SRC-EPIC-031] | Using dynamic delegates everywhere or events for internal multicast state. |
| Lua GC vs UE GC | Both collect unreachable objects | Lua collects Lua objects/closures/tables; Unreal traces UObjects through its own graph; bindings must bridge invalidation explicitly [SRC-SCRIPT-001] [SRC-EPIC-005] | Assuming one GC sees the other's references automatically. |
| C# managed lifetime vs native lifetime | Both can wrap resources | .NET GC controls managed objects; native/UObject resources need explicit wrappers, handles and deterministic cleanup where required [SRC-SCRIPT-006] [SRC-SCRIPT-008] | Letting a C# finalizer decide UObject lifetime. |

## Misconceptions and Red-Flag Advice

| Red flag | Better answer |
|---|---|
| "Use `EditAnywhere BlueprintReadWrite` by default." | Choose edit, visibility, Blueprint, persistence, duplication and metadata flags by consuming system and invariant risk. |
| "Raw UObject pointers are fine if I know they exist." | Raw pointers are not ownership; use GC-visible properties, weak/soft wrappers, validity checks and lifetime cancellation as appropriate. |
| "Blueprint is slow; rewrite it in C++." | First identify bottleneck and call frequency. C++ is for invariants, hot paths and stable APIs; Blueprint is excellent for authoring/orchestration when profiled. |
| "Reliable RPC means reliable gameplay state." | Reliable only affects delivery/order on that channel; robust gameplay state still needs authority, idempotence and late-join/state recovery. |
| "OnRep is called immediately when the server changes a property." | Replication is network/update driven; design for delay/order and use `OnRep` as a client-side reaction to received state. |
| "Use Tick less, use timers/events always." | Event-driven is not universally cheaper; measure scheduling, allocation, delegate and burst costs [SRC-PERF-005]. |
| "Nanite/Lumen solve rendering performance." | They shift constraints and have content/platform/version limits; profile actual passes and target hardware. |
| "Use Mass for performance." | Use Mass when data shape and scale justify it; conversion, representation, debugging and gameplay interaction are real costs. |
| "Metadata controls runtime behaviour." | Metadata is primarily editor/Blueprint-node/tooling guidance. Runtime rules belong in code/data consumed at runtime [SRC-EPIC-036]. |
| "A plugin doc means the feature is core." | Record plugin status, target-version support and project adoption requirements. |

## Role-Specific Gaps

| Role | Most dangerous gap | Repair action |
|---|---|---|
| Gameplay engineer | Weak state placement and UObject lifetime | Build Project 1 plus Project 7A; rehearse framework, GC and Blueprint API questions. |
| AI/gameplay engineer | Treating BT/EQS/Nav/avoidance as one black box | Implement Project 2 and explain decision, query, path and local steering separately. |
| Networking engineer | Knowing terms but not diagnosis order | Run Project 3 failure matrix and produce Net Profiler/Insights evidence. |
| Rendering/graphics engineer | Saying "GPU bound" without pass/cost evidence | Run Project 4 render matrix and learn RDG/render-pass vocabulary. |
| Engine/tools engineer | Ignoring module/editor/runtime/package boundaries | Build Project 6 with runtime/editor modules, commandlet and packaged proof. |
| Technical artist/designer | Owning gameplay truth in presentation systems | Build UI/animation/Niagara/audio extensions with authority and lifetime notes. |
| Generalist | Broad shallow definitions but no project stories | Use the 8-week schedule and portfolio packet; prepare one bug and one optimisation story. |

## Remaining Research Gaps for Final Acceptance

| Gap | Why it matters | Next evidence pass |
|---|---|---|
| Job-description corpus | Needed to validate role weighting and common screening terms | Collect current UE gameplay/network/tools/rendering job ads, tag required systems and compare to role overlays. |
| Interview anecdote corpus | Needed to compare public interview traps with source-backed curriculum | Add only as supplementary S8 evidence and label stale/low-confidence claims. |
| Engine-source branch checks | Needed for exact UE5.3-UE5.6 API and order claims | Pick a target branch and verify UHT/generated code, UObject threading, replication internals, Live Coding and specialist plugin APIs. |
| Official talks | Useful for renderer/network/Mass/GAS architectural intent | Add source records for current Epic talks and avoid treating talk demos as universal production defaults. |
| Specialist networking | Fast Array, replicated subobjects, RepGraph, Iris and lag compensation need deeper treatment | Create a bounded networking-specialist cluster with source/API evidence and Project 3 extensions. |
| Rendering specialist | Shader compilation, PSO caches, mesh draw commands and platform captures need deeper treatment | Create a rendering-specialist cluster after final P0/P1 synthesis. |
| Final quantitative banks | Research target requires many more questions by area | Expand banks by distinct models and failure workflows, not trivia padding. |

## Practical Gap Repair Loop

For any weak topic:

1. Name the subsystem and the consuming lifecycle.
2. Read one chapter section and one primary source.
3. Implement or inspect one minimal example.
4. Break it deliberately.
5. Capture the log, trace, profiler output or package failure.
6. Write the interview answer in four parts: what it is, why it exists, common bug, how to debug.
7. Add a flashcard only if the answer exposed a reusable misconception.

This loop keeps gap repair from becoming passive reading or metadata work.
