# Unreal Engine Interview Study Operating Plan

This operating plan turns the topic chapters, practice banks and projects into a dependency-aware, flexible study flow. It assumes the learner already programs in C++ but needs interview-ready Unreal Engine fluency across UE5.3-UE5.6. If a topic is labelled maintained UE5.7, UE4-era, experimental or plugin-dependent in the chapter/source index, treat it as awareness until checked against the target branch.

## How to Use These Plans

Use this file as the operating layer, not a fixed calendar. Keep the order dependency-aware: P0 foundations first, then P1 breadth, then P2/P3 role-aligned depth. Every cycle should produce evidence, not only notes.

Use these source families as anchors:

| Area                             | Primary source anchors                                                                                                                                                                                                                               |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Standard C++                     | Core Guidelines and C++ references for lifetime, ownership, templates, memory model and sanitizers [SRC-CPP-001] [SRC-CPP-002] [SRC-CPP-020] [SRC-CPP-030]                                                                                           |
| UE object model and C++          | Reflection, UHT, UObject, pointers, specifiers, interfaces, logging/asserts and async proxy APIs [SRC-EPIC-001] [SRC-EPIC-002] [SRC-EPIC-003] [SRC-EPIC-009] [SRC-EPIC-032] [SRC-EPIC-037] [SRC-EPIC-038] [SRC-EPIC-039] [SRC-EPIC-044]              |
| Gameplay framework               | Actor/Component, GameMode/GameState/PlayerState and subsystem roles [SRC-EPIC-010] [SRC-EPIC-011] [SRC-EPIC-012] [SRC-EPIC-013] [SRC-EPIC-017]                                                                                                       |
| Networking and AI                | Replication authority, RPC/property ordering, movement prediction, Behaviour Trees, StateTree, Perception, EQS and nav debugging [SRC-NET-001] [SRC-NET-005] [SRC-NET-006] [SRC-NET-009] [SRC-AI-002] [SRC-AI-003] [SRC-AI-004] [SRC-AI-008]         |
| Profiling/rendering/assets/build | Insights/stat tools, RDG, Nanite/Lumen/VSM, hard/soft references, Asset Manager, UBT/modules/plugins and Live Coding [SRC-PERF-001] [SRC-PERF-003] [SRC-RENDER-003] [SRC-RENDER-005] [SRC-ASSET-001] [SRC-ASSET-003] [SRC-BUILD-001] [SRC-BUILD-006] |
| Specialist systems               | MassEntity, GAS, animation, physics, UI, audio, Niagara and scripting sources are used after the P0/P1 foundation is stable [SRC-MASS-001] [SRC-GAS-001] [SRC-ANIM-015] [SRC-PHYS-001] [SRC-UI-005] [SRC-AUDIO-007]                                  |

## Flexible Operating Loop

| Block | Output |
|---|---|
| Recall | Review flashcards and answer old questions without notes. |
| Source-grounded reading | Read one chapter section and inspect the cited official/API source. |
| Implementation or trace | Build, instrument, or deliberately break the relevant project slice. |
| Interview conversion | Turn the session into one strong answer pattern plus one "what I would check next" note. |
| Error log | Record misconceptions, version caveats and missing evidence. |

## Global Dependency Timeline

Recommended dependency order:

1. Standard C++ lifetime and ownership
2. UE C++ object model and reflection
3. Gameplay framework and architecture
4. Networking and AI
5. Profiling, rendering and assets
6. GAS, Mass, animation, physics, UI, audio and VFX breadth
7. Build/tools and scripting integration
8. Mock loops, portfolio stories and gap repair

## Role Overlay Graphic

Role overlays should be applied on top of the same P0 base: C++, UObject, gameplay framework and debugging.

- Gameplay role: architecture, Blueprint API, networking, GAS
- AI role: BT/StateTree, Perception, EQS, navigation, profiling
- Network role: authority, ownership, RPC/property ordering, prediction, bandwidth
- Tools role: modules, plugins, assets/cooking, editor automation, validation
- Rendering role: maths, GPU pipeline, RDG, profiling, asset cost
- Technical art/design: Blueprint, UI, animation, Niagara, audio, data assets
- All overlays should eventually convert into portfolio evidence

## Mock Interview Ladder

| Stage | Trigger | Format | Pass Standard |
|---|---|---|---|
| Recall drill | During regular study sessions | Flashcards + short questions | Correct without reading, or marked for repair. |
| Narrow topic mock | After a topic reaches basic recall | Focused mock on the current topic | Answer includes what/why/how, common bug, debug workflow and version caveat. |
| Cross-system mock | After several related systems are covered | Combine gameplay, networking, performance and build/package | Candidate identifies ownership, authority, lifetime and evidence before proposing fixes. |
| Portfolio mock | After a project artefact exists | Use a real project artefact | Candidate can explain design constraints, implementation, bug, trace/evidence and trade-off. |
| Full loop mock | Before active interview batches | C++, UE core, system design, runtime bug, performance bug | Candidate can recover after a weak answer by naming a verification plan. |

## Portfolio Evidence Packet

By the end of the main preparation pass, collect a folder or document with:

- system diagrams for one gameplay feature and one runtime subsystem;
- snippets showing UObject lifetime, specifier choices and async cancellation;
- a replication failure and fix with logs or Network Profiler/Insights evidence;
- a performance issue before/after with frame-time units rather than FPS alone;
- a packaged-build or cook/load failure and fix;
- a role-specific specialist slice, labelled plugin/version-sensitive where appropriate;
- a one-page "things I would verify in the target branch" list.

## Choosing Optional Deep Dives

| Target role | First optional deep dive | Second optional deep dive |
|---|---|---|
| Gameplay | GAS or system design portfolio | Networking prediction/lag compensation awareness |
| AI/gameplay | StateTree/EQS/Smart Objects | MassAI or scalable perception |
| Networking | Fast Array/subobjects/RepGraph/Iris | Character movement prediction and lag compensation |
| Engine/tools | Build modules/plugins/editor commandlets | Asset Manager/cook/package/validation pipeline |
| Rendering/graphics | RDG/shaders/PSOs/platform captures | Nanite/Lumen/VSM limits and profiling |
| Technical art/design | Blueprint API, UI, animation and Niagara | Asset pipeline, data validation and performance budgets |
| Generalist | One gameplay vertical slice | One trace-led optimisation story |

## Stop Conditions Before Interview Week

Do not keep reading if any of these are still failing:

- You cannot explain why `TSharedPtr` does not keep a UObject alive.
- You cannot place GameMode/GameState/PlayerController/PlayerState responsibilities.
- You cannot debug a client RPC that never arrives.
- You cannot classify a frame spike as Game Thread, Render Thread, GPU, loading, GC or network evidence.
- You cannot describe one packaged-only asset/build failure.
- You cannot give one concrete bug story with logs/traces and a prevention rule.

When a stop condition fails, spend the next session repairing that item with one source, one chapter section, one implemented or inspected example and one mock answer.
