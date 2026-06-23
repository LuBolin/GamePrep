# GamePrep

GamePrep is a source-traceable, interview-oriented curriculum for a roughly three-year Unreal Engine engineer. It targets UE5.3–UE5.6, uses en-GB, and labels older, experimental, deprecated, and plugin-dependent material explicitly.

## Start here

1. Read [the learning plan](curriculum/ue_interview_learning_plan.md) for priorities and learning order.
2. Use [the master execution roadmap](curriculum/master_execution_roadmap.md) for long-horizon waves and completion gates.
3. Choose a cadence from [the study schedules](curriculum/ue_study_schedule.md).
4. Use [the gap analysis](curriculum/ue_gap_analysis.md) to pick repairs and role-specific deep dives.
5. Rehearse with [the synthesis lists](curriculum/ue_interview_synthesis_lists.md) after each topic block.
6. Study the chapters in `topics/`.
7. Retrieve and rehearse with `practice/` rather than merely rereading.
8. Use `sources/ue_topic_source_index.md` to audit claims and continue research.
9. Use `data/ue_interview_knowledge_graph.json` for machine analysis of dependencies and coverage.

`research.md` remains the master scope. `STATUS.md` is the living checkpoint and coverage map.

## Current substantive chapters

- [Standard C++ for Unreal interviews](topics/cpp_for_unreal_interviews.md)
- [UE C++ idioms, reflection, UObject lifetime, and Blueprint boundary](topics/ue_cpp_idioms.md)
- [Gameplay framework and system design](topics/game_architecture_patterns.md)
- [Game programming patterns](topics/game_programming_patterns.md)
- [Advanced gameplay patterns specialist pass](topics/advanced_gameplay_patterns_specialist.md)
- [Networking and replication](topics/ue_networking_and_replication.md)
- [Networking target-branch implementation proof](topics/ue_networking_target_branch_proof.md)
- [NetworkPrediction plugin and full rollback proof](topics/ue_network_prediction_rollback_proof.md)
- [AI and navigation](topics/ue_ai_navigation.md)
- [Smart Objects and StateTree gameplay integration](topics/ue_smart_objects_statetree.md)
- [Custom AI senses and MassAI specialist pass](topics/ue_ai_custom_senses_massai.md)
- [PCG and procedural content workflows](topics/ue_pcg_procedural_content.md)
- [Platform constraints and certification-ready release thinking](topics/ue_platform_constraints.md)
- [Apple and console profiling](topics/ue_apple_console_profiling.md)
- [Device lab automation and target evidence](topics/ue_device_lab_automation.md)
- [Profiling and optimisation](topics/ue_profiling_optimisation.md)
- [Packaged performance, BuildGraph, and World Partition proof](topics/ue_packaged_performance_build_worldpartition_proof.md)
- [Rendering and GPU performance](topics/ue_rendering_graphics_performance.md)
- [Asset references, loading, cooking, and streaming](topics/ue_assets_loading_cooking.md)
- [World Partition and large-world production pipeline](topics/ue_world_partition_large_world_pipeline.md)
- [Build systems, modules, plugins, and editor tools](topics/ue_build_modules_plugins_tools.md)
- [ECS and MassEntity](topics/ue_massentity_ecs.md)
- [Gameplay Ability System](topics/ue_gameplay_ability_system.md)
- [GAS specialist depth](topics/ue_gas_specialist_depth.md)
- [Enhanced Input architecture](topics/ue_enhanced_input.md)
- [Algorithms and data structures for game interviews](topics/game_algorithms_and_data_structures.md)
- [Game maths for Unreal interviews](topics/game_math_for_interviews.md)
- [Unreal animation systems](topics/ue_animation_systems.md)
- [Unreal physics and collision](topics/ue_physics_collision.md)
- [Unreal UI systems](topics/ue_ui_systems.md)
- [Unreal audio systems](topics/ue_audio_systems.md)
- [Unreal Niagara and VFX systems](topics/ue_niagara_vfx.md)
- [Runtime presentation target-proof](topics/ue_runtime_presentation_target_proof.md)
- [Lua and C# integration](topics/scripting_integration_lua_csharp.md)

## Repository map

| Area | Purpose | Planned required deliverables |
|---|---|---|
| `curriculum/` | Read-through curriculum, schedules, gap analysis, and synthesis lists | `ue_interview_learning_plan.md`, `ue_study_schedule.md`, `ue_gap_analysis.md`, `ue_interview_synthesis_lists.md` |
| `topics/` | Substantive reference chapters | C++, UE idioms, patterns, architecture, algorithms, maths, Lua/C# |
| `practice/` | Questions, flashcards, design prompts, and projects | Four banks, flashcards, projects |
| `sources/` | Human-readable, canonical source ledger | `ue_topic_source_index.md` |
| `data/` | Machine-readable graph and source/crawl data | knowledge graph, source index, manifest JSON |

Files are created when they contain useful material; grouped chapters preserve the master scope without keeping empty scaffolds.

## Research method

Queries are generated per bounded topic cluster, first against official documentation and API references, then engine/sample evidence, official talks, production sources, and finally supplementary community or interview material. URLs are canonicalised by removing cosmetic locale/tracking parameters while preserving engine-version selectors. Mirrors and alternate presentations of the same source share a deduplication group.

Sources are ranked from S1 (official Epic documentation) to S8 (anecdotal material). Within a class, the working quality score is: authority 30%, target-version relevance 20%, technical depth 15%, practical/debugging value 15%, clarity 10%, and interview relevance 10%. A lower-ranked source can supersede an older one only when the change is explicit and evidenced. Unresolved disagreement is retained as a conflict note.

Human chapters cite stable source IDs such as `[SRC-EPIC-001]`. The Markdown source index is currently canonical; JSON source records will be generated when automation benefits outweigh duplicate maintenance. Graph JSON is maintained from the outset because dependency traversal is already useful.
