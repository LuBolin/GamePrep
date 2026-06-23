# Unreal Engine AI and Navigation

See also: [[ue_ai_custom_senses_massai]], [[ue_smart_objects_statetree]], [[game_algorithms_and_data_structures]], [[ue_hands_on_projects]], [[ue_interview_question_bank]].

## Cluster 1 — Decision Architecture, Perception, EQS, Navigation, and Debugging

**Priority:** P1 (P0 for AI/gameplay roles)  
**Expected depth:** D3–D4  
**Version scope:** UE5.3–UE5.6 preferred. StateTree is UE5-era and may require plugin/module configuration; core Behaviour Tree, Perception, EQS, and Navigation concepts are broadly stable.  
**Prerequisites:** Gameplay Framework, Components, event-driven design, vector/distance maths, graph/pathfinding fundamentals.

### Decompose the AI pipeline

An AI agent is not a Behaviour Tree. A useful production model separates:

```text
World stimuli
  -> perception/sensing
  -> memory/knowledge (Blackboard or state data)
  -> decision policy (BT, StateTree, FSM, utility, custom code)
  -> environmental query (EQS or bespoke query)
  -> path request (Navigation System)
  -> path following/movement
  -> local avoidance/steering
  -> animation/combat execution
  -> feedback events update knowledge
```

When an enemy “stands still”, ask which layer failed. It may have selected Chase correctly but have no path, a mismatched nav agent radius, an already-running task that never finished, an avoidance deadlock, or movement disabled by animation—not a decision problem.

### AIController and Pawn

`AIController` represents decision-making/control; the Pawn/Character represents the physical agent and reusable capabilities. The controller observes the world, owns or coordinates perception/decision components, possesses a Pawn, and issues movement/action requests. This mirrors PlayerController/Pawn separation and allows respawn, possession changes, or one controller policy to drive compatible Pawns. [SRC-AI-001]

Keep physical capabilities on Pawn/components (movement, weapon, health) and decision policy on Controller/decision assets. Avoid making Behaviour Tree Tasks directly manipulate arbitrary animation/UI/network state; call a stable Pawn/component capability so decision technology can change without rewriting mechanics.

### Behaviour Tree and Blackboard

Unreal Behaviour Trees execute branches visually while a Blackboard stores decision-relevant keys. Typical keys are current target, last known location, patrol point, alert state, or booleans/tags derived from perception/game state. [SRC-AI-002]

Node roles:

| Node | Responsibility | Common misuse |
|---|---|---|
| Selector | Try children by priority until one succeeds/runs | Treating left-to-right order as incidental |
| Sequence | Run ordered children until one fails/runs | Huge procedural scripts hidden in one branch |
| Task | Perform an action and finish success/failure/abort | Never calling FinishLatentTask; polling conditions as tasks |
| Decorator | Gate a branch and optionally observe/abort | Duplicating condition logic in tasks/services |
| Service | Periodic update while its branch is active | Every-frame expensive world scans |
| Blackboard | Shared decision memory/state keys | Global dumping ground with unclear writers/ownership |

UE Behaviour Trees are event-driven: decorators can observe Blackboard changes and abort/replan branches instead of reevaluating the entire tree every frame. Unreal uses Services, Simple Parallel, and Observer Aborts rather than general parallel complexity for common concurrent behaviour. [SRC-AI-002]

### Observer Aborts and priority

Observer Abort modes answer two questions:

- Should this branch stop when its own condition becomes invalid? (`Self`)
- Should a newly valid higher-priority branch interrupt lower-priority work? (`Lower Priority`)

`Both` combines them where appropriate. This creates responsive transitions without services polling every frame. A classic pattern is a high-priority Combat branch whose target-valid decorator observes the Blackboard: when a target appears it aborts lower-priority Patrol; when the target becomes invalid it aborts its own combat tasks.

Abort correctness requires Tasks to implement cleanup/cancellation. A Move task or montage ability that ignores abort leaves gameplay running after the tree changed branches. Treat task start/finish/abort as a lifecycle contract.

### Tasks, Services, and hidden instance state

Tasks should start bounded work, report Running when asynchronous, and finish exactly once. Blueprint Tasks often create per-instance behaviour conveniently but add VM/event overhead. Native Tasks may share node objects depending on instancing settings; storing mutable per-agent state in a shared node is a specialist trap. Prefer Blackboard/task memory or explicit instancing according to the node contract.

Services are for periodic context updates, not a replacement for event-driven perception or system events. Frequency and random deviation should reflect gameplay latency needs. One service performing `GetAllActorsOfClass` for hundreds of agents is a scaling failure even if its interval is “only” 0.2 seconds.

### Behaviour Tree versus StateTree versus FSM

| Model | Strength | Weakness / caution | Good fit |
|---|---|---|---|
| Finite State Machine | Explicit states/transitions, easy small-system reasoning | Transition explosion and duplicated hierarchy as complexity grows | Small bounded modes, animation/gameplay state |
| Hierarchical FSM | Groups shared transitions/behaviour | Custom tooling/debugging burden if hand-built | Medium systems with clear modal hierarchy |
| Behaviour Tree | Priority selection, reactive tasks, readable action decomposition, mature AI tooling | Blackboard coupling and large-tree complexity; action abort lifecycle | Reactive NPC decisions and action selection |
| StateTree | Hierarchical states/transitions plus selector-style state selection | UE5-era system, data binding/lifecycle learning, plugin/version sensitivity | Structured stateful logic for AI/gameplay/Smart Objects |

Epic describes StateTree as a general-purpose hierarchical state machine combining Behaviour Tree selectors with state-machine states and transitions. [SRC-AI-003]

Choose from semantics, team tooling, and debugging—not novelty. Behaviour Tree excels when priority/reactivity selects actions. StateTree excels when being in a state with explicit enter/exit/transitions is central. A tiny enum switch can beat both for three stable states. Systems can compose: StateTree controls high-level modes while a task invokes specialised movement/combat logic.

### AI Perception: stimuli, not omniscience

AI Perception components listen for configured senses and collect registered stimuli sources. Sight, hearing, damage, touch, team, and custom senses have different generation/configuration. Perception events provide the sensed Actor and stimulus data such as success, location, receiver location, age/expiration, strength, and tag. [SRC-AI-004]

Important distinctions:

- **Currently perceived:** sensed now.
- **Known perceived:** sensed and not yet forgotten.
- **Last known location:** memory derived from a prior stimulus, not current truth.
- **Successfully sensed false:** a transition/loss event, not necessarily “forget immediately”.

Design memory policy explicitly. A stealth guard may investigate last known location for five seconds, downgrade alert, then forget. Do not clear target immediately on any failed sight update if hearing/damage still supplies valid knowledge. Conversely, Max Age 0 can mean never forgotten for a sense configuration, creating “psychic AI” if misunderstood. [SRC-AI-004]

Affiliation/team filtering and stimuli registration must be tested. UE5.6 docs note Blueprint affiliation limitations and common neutral/tag workarounds; production teams should prefer coherent team-attitude logic rather than scattered Actor tags where C++ control is available. [SRC-AI-004]

### EQS: generate, filter, score

Environment Query System answers “which candidate best satisfies spatial/contextual criteria?” It is not the decision tree itself.

- **Generator:** produces candidate Actors/locations.
- **Context:** supplies reference frames such as querier, target, squad leader, objective.
- **Tests:** filter invalid candidates and score valid ones.
- **Run mode:** chooses best item, random among best percentage, all matches, etc., depending API/asset setup.

Examples: cover with line of sight to target but hidden from another threat; reachable healing pickup; attack position within preferred range; patrol point away from recent danger. [SRC-AI-005]

Order cheap/high-rejection filters before expensive traces/path tests where the query planner/settings permit. Limit candidate count and query frequency. Cache results only for a validity window appropriate to a changing world. An EQS query every frame per NPC with hundreds of trace-tested points is a content-authored denial of service.

EQS scores express preference, not correctness. Hard constraints should filter. If “must be reachable” is merely a low score, the best remaining result may still be unreachable.

### Navigation Mesh and pathfinding

UE's Navigation System generates a tiled polygon mesh from collision. Polygons form a graph with traversal costs; pathfinding finds a low-cost route for an agent configuration. Static, Dynamic, and Dynamic Modifiers Only generation modes trade runtime adaptability against build/update cost. Nav modifiers change area/cost; nav links connect discontinuities such as jumps or doors. [SRC-AI-006]

The NavMesh answers global traversability around static/nav-represented obstacles. It does not guarantee:

- the Pawn's movement component can execute every segment;
- the target remains reachable/moving;
- local crowds will not collide;
- animation/root motion follows the path;
- network authority is correct;
- agent radius/height matches generated data.

### Why `MoveTo` fails

Use a layered checklist:

1. **Possession:** Does the Pawn have the expected AIController, and did OnPossess/startup run?
2. **Nav data:** Is start and goal projected onto the correct NavMesh? Is a NavMeshBoundsVolume present and built?
3. **Agent configuration:** Radius, height, step height, movement capabilities, and supported agent match?
4. **Goal:** Valid Actor/location? Reachable projection? Acceptance radius and stop-on-overlap reasonable?
5. **Path:** Nav area blocked, missing link, dynamic obstacle, partial paths disallowed?
6. **Path following:** Another request aborted/replaced it? Task never finished? Controller lacks path-following component?
7. **Movement:** Component active, speed > 0, movement mode valid, root motion/physics not overriding?
8. **Authority/network:** Is movement being requested on the authoritative AI instance?

Always inspect the Move request/result/status rather than treating “Failed” as one cause.

### Pathfinding versus avoidance

Pathfinding chooses a route through mostly static navigable space. Avoidance adjusts local velocity to avoid moving agents/obstacles while following that route. Rebuilding/replanning the global path every time two agents approach is expensive and unstable; local avoidance exists for that problem. [SRC-AI-007]

UE provides RVO on CharacterMovement and Detour Crowd through crowd management/DetourCrowd AIController. Epic recommends using them exclusively rather than enabling both. RVO does not use NavMesh for avoidance and can steer out of bounds; Detour Crowd adds corridor/topology optimisation and has capacity/configuration constraints. [SRC-AI-007]

Avoidance is not crowd strategy. Doorway deadlocks, formations, lane selection, reservation, and group priorities may require Smart Objects, slots, flow fields, local rules, or Mass-based crowd systems beyond individual velocity avoidance.

### A* interview depth

NavMesh pathfinding is graph search over polygons/costs. At interview depth, explain:

- Dijkstra explores by accumulated cost and finds optimal paths with non-negative edges.
- A* prioritises `f(n)=g(n)+h(n)`.
- An admissible heuristic does not overestimate, preserving optimality under the relevant assumptions.
- Better heuristics reduce explored nodes; weighted/non-admissible heuristics trade optimality for speed.
- Pathfinding computes a route; path following executes it; avoidance handles local dynamic interactions.

Do not claim UE's exact Recast/Detour implementation is “just A*” at every layer without source confirmation. The transferable interview model is graph, costs, heuristic, path corridor, and local steering.

### Debugging tools

UE5.6 AI Debugging centralises NavMesh, general AI/controller/path status, Behaviour Tree/Blackboard, EQS candidates/scores, and Perception stimuli. The apostrophe key enables it; default numpad categories expose navigation, general AI, BT, EQS, and Perception. [SRC-AI-008]

Visual Logger records spatial and textual snapshots over time, allowing a rare one-frame Blackboard/perception/path change to be inspected after the event. This is superior to flooding logs without position/context. [SRC-AI-009]

Recommended workflow:

1. Select the failing Pawn and confirm Controller/Pawn/active task/path-follow status.
2. Inspect Blackboard values and active BT branch or StateTree state.
3. Inspect Perception's currently sensed/known stimulus and last location.
4. Visualise NavMesh, start/goal projection, current path, and agent settings.
5. Inspect EQS generated candidates, failed tests, normalised scores, and winner.
6. Record Visual Logger around the failure.
7. Only then add targeted logs/breakpoints to the layer with contradictory state.

### Performance workflow

1. Measure game-thread AI cost and count active agents, controllers, BT tasks/services, perception listeners/sources, EQS queries/candidates/tests, nav rebuilds, and path requests.
2. Replace polling with events/observer aborts where state changes are already known.
3. Stagger service/query updates; avoid synchronised spikes across hundreds of agents.
4. Reduce perception range/senses/affiliation pairs to design needs.
5. Reduce EQS candidate count before optimising individual tests; filter cheaply before traces/path tests.
6. Avoid dynamic NavMesh rebuild over large areas when modifiers/links or bounded dynamic tiles suffice.
7. Cache paths/queries only while valid; uncontrolled stale cache creates behaviour bugs.
8. Apply significance/LOD: distant agents can think, sense, query, animate, and update less frequently or use Mass/aggregate simulation.

### Common bugs and misconceptions

1. **Blackboard is the AI brain.** It is decision memory; policy lives elsewhere.
2. **Service means background thread.** It is periodic behaviour while a branch is active, generally game-thread work.
3. **Simple Parallel means multithreaded.** It models concurrent behaviour, not necessarily parallel CPU execution.
4. **Task returns Running and engine finishes it.** Latent tasks must signal completion/abort correctly.
5. **Lost sight means target never existed.** Memory/other senses can remain valid.
6. **EQS chooses behaviour.** It ranks environmental candidates for a decision.
7. **Highest score means valid.** Constraints must filter; scores rank survivors.
8. **Pathfinding avoids moving agents.** Local avoidance solves a different layer.
9. **Green NavMesh means MoveTo must work.** Agent, projection, path following, movement, and authority can still fail.
10. **More frequent sensing/services is more responsive.** Events and sensible intervals can be both faster and responsive.
11. **StateTree replaces Behaviour Trees.** They model different strengths; selection is project-specific.
12. **AI must run on every client.** Authoritative gameplay AI normally runs server-side; clients receive replicated outcomes/presentation.

### Strong interview answer pattern

For “AI does not chase the player”:

1. Confirm authoritative AIController possession and decision system running.
2. Confirm perception stimulus/current memory and Blackboard/state update.
3. Confirm branch/state selection and decorator/transition conditions.
4. Confirm MoveTo request/result, start/goal NavMesh projection, agent config, and path.
5. Confirm path following/movement and avoidance.
6. Use AI Debugger and Visual Logger before changing logic.
7. Profile frequency/count if the issue occurs only at scale.

### What a three-year engineer should know

Explain Controller/Pawn separation, BT/Blackboard node roles, observer aborts and latent task lifecycle, StateTree/FSM/BT trade-offs, Perception memory, EQS generation/filter/scoring, NavMesh/cost/agent basics, MoveTo failure diagnosis, pathfinding versus avoidance, and AI Debugger/Visual Logger use.

**Specialist depth:** custom senses, native BT task memory/instancing, StateTree custom schemas/data views, asynchronous EQS internals, Recast generation/source areas, custom path following/nav links, crowd algorithms, utility/GOAP/HTN planners, MassAI, and networked deterministic AI.

### Hands-on verification

Complete Project 2. The deliverable must include a recorded failure catalogue: at least one bug in perception, Blackboard/decision, EQS, pathfinding, path following, and avoidance, each diagnosed with the appropriate tool.

### Conflict and uncertainty notes

- Maintained BT, StateTree, and avoidance pages may surface UE5.7. The core concepts are stable, but exact node properties, plugin status, and APIs should be checked in UE5.6.
- EQS has historically been presented with experimental/plugin caveats in some engine eras. Verify target project/plugin status rather than copying an old label globally.
- StateTree is not universally superior to Behaviour Tree; the chapter's comparison is architectural synthesis based on their documented models, not an Epic mandate.

