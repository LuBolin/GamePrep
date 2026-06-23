# Unreal Smart Objects and StateTree Gameplay Integration

See also: [[ue_ai_navigation]], [[ue_ai_custom_senses_massai]], [[ue_massentity_ecs]], [[ue_hands_on_projects]].

## Cluster scope

**Priority:** P2 for general gameplay engineers; P1 for AI, open-world, systemic gameplay, crowd, and activity-system roles.  
**Expected depth:** D3-D4.  
**Version scope:** UE5.3-UE5.6 preferred. Smart Objects and StateTree are UE5-era, plugin/module/project-configuration sensitive systems. Exact API names, debugger categories, replication behavior, and Mass integration must be verified in the target branch. [SRC-AI-003] [SRC-AI-010] [SRC-AI-011] [SRC-AI-012] [SRC-AI-013]

This chapter is about environment affordances and structured stateful behavior. The goal is not "AI can click things." The goal is to let world objects advertise usable opportunities, let agents find and reserve those opportunities safely, execute behavior through a clear state machine, and clean up correctly when movement, animation, combat, authority, or lifetime changes interrupt the flow.

## Why Smart Objects exist

Without Smart Objects, interactions often become scattered:

- the bench knows how every NPC should sit;
- the NPC hardcodes bench, chair, cover, terminal, door, and vending-machine logic;
- designers duplicate trigger volumes and custom Blueprint scripts;
- two agents race for one position;
- behavior fails halfway and leaves the object or agent permanently "busy";
- Mass/crowd systems need a different implementation from Actor AI.

Smart Objects give the world a way to expose affordances. A station, chair, cover point, door panel, pickup shelf, workbench, charging port, or crowd waiting slot can be represented as an object with slots and behavior metadata. Agents can query for matching opportunities, claim a slot, move to it, run behavior, and release it.

StateTree is a natural companion because many Smart Object interactions are stateful sequences: find slot, claim slot, reach slot, align, play use behavior, wait, react to interruption, release. Epic's StateTree overview explicitly shows Smart Object task grouping, including reaching and using a Smart Object, claiming a wait slot, moving to it, and using the claimed slot location as shared task data. [SRC-AI-010]

## Core mental model

```text
World Actor / Component
  -> Smart Object Definition
  -> one or more Slots with transforms/tags/data/behavior
  -> registered with Smart Object subsystem
  -> agent request / filter / search
  -> claim one slot
  -> navigate / align / validate
  -> execute behavior, often StateTree-driven
  -> complete / fail / abort
  -> release claim and update object state
```

The interview signal is boundaries. Smart Objects do not solve pathfinding, animation, authority, or gameplay validation by themselves. They coordinate an available opportunity. Movement, animation, ability/cost rules, inventory mutation, damage, replication, save/load, and presentation still need their own contracts.

## Smart Object concepts

### Smart Object component and definition

At a practical architecture level, a placed world actor usually owns a Smart Object component, and that component references a Smart Object Definition asset. The API anchors for `USmartObjectComponent` and Smart Object definitions are branch-sensitive, but the stable design idea is: the component gives the instance a world presence; the definition describes available slots and behavior/configuration. [SRC-AI-011] [SRC-AI-013] [SRC-AI-014]

Treat the definition as authoring data, not the runtime owner of every interaction's mutable truth. Runtime state such as claim, cooldown, disabled state, damage state, or "currently occupied by Actor X" needs an explicit owner and lifetime policy.

### Slots

A Smart Object slot is an opportunity for one user or one role in an interaction: sit position, cover edge, terminal standing point, door use point, queue position, repair station mount, or vendor counter spot. Slot definitions are official API concepts and are where branch details need verification. [SRC-AI-015]

A slot design should answer:

- who can use it: tags, class, team, faction, role, size, required ability;
- where to go: transform, approach point, alignment policy, nav projection rule;
- what to do: behavior asset, animation tag, ability tag, interaction verb;
- when it is valid: enabled, cooldown, object health, powered state, occupied state;
- how to release it: success, failure, abort, user destroyed, object disabled, timeout.

Do not confuse a found slot with an owned slot. Querying says "this may be usable." Claiming says "this user has reserved it under current rules." Using says "the behavior has begun." Completion says "the interaction ended." Each phase can fail.

### Query, claim, use, release

The Smart Object subsystem is the natural runtime service for finding and coordinating Smart Objects. `USmartObjectSubsystem` is the API anchor, while the exact request and handle signatures should be verified in the target branch. [SRC-AI-012]

Use this four-phase lifecycle in design reviews:

1. **Query:** search for candidates that satisfy location, tags, user requirements, and activity need.
2. **Claim:** reserve a specific slot so another user cannot take it concurrently.
3. **Use:** execute the behavior, often through StateTree, gameplay behavior, ability, animation, or component code.
4. **Release:** clear the claim and restore/update object state on success, failure, abort, destruction, timeout, or mode change.

If a system has no explicit release path, it is not production-ready. The release path must run when:

- the agent dies or is despawned;
- the Smart Object actor/component is disabled or destroyed;
- the MoveTo request fails;
- the StateTree transitions to failure or interruption;
- animation notify windows miss;
- combat/alert logic preempts the activity;
- server authority rejects the interaction.

## StateTree integration

Epic describes StateTree as a hierarchical state machine that combines Behavior Tree-style selectors with state-machine states and transitions. Selection starts at the root, checks Enter Conditions, descends into child states, and activates the selected root-to-leaf path. Tasks execute for all active states from root to leaf. Transitions can be triggered by task completion, success/failure, or monitored conditions, and transition evaluation proceeds from leaf upward. [SRC-AI-010]

For Smart Objects, that maps cleanly to interaction lifecycles:

```text
Use Smart Object
  Acquire
    Find candidates
    Claim slot
  Reach
    Move to slot / approach point
    Align / face target
  Use
    Play animation / ability / component action
    Monitor interruption
  Exit
    Release claim
    Restore locomotion / tags / cooldown
```

### Parameters, context data, evaluators, and global tasks

StateTree data flow matters in interviews because many bugs are "wrong data in the right-looking tree." The StateTree overview distinguishes parameters, context data, evaluators, and global tasks:

- **Parameters** customise a tree instance from the outside, such as desired activity or animation asset.
- **Context data** depends on where the tree runs. The overview explicitly notes that a StateTree used with Smart Object behavior can expose the Smart Object used and the Actor using it as context.
- **Evaluators** expose runtime data to conditions and tasks, and can tick/start/stop.
- **Global tasks** run for the tree lifetime and can provide data needed before root state selection. [SRC-AI-010]

Use parameters for explicit configuration, context data for runtime owner/user/object identity, evaluators for observed facts, and tasks for actions. Do not hide an interaction's ownership state in a random task output if release logic in another branch needs it.

### Task concurrency and transition policy

StateTree tasks in active states run concurrently. The first task that completes can trigger transition evaluation. That is powerful but dangerous:

- If `LookAt` completes before `MoveTo`, a naive transition can leave Reach early.
- If a monitor task fails, failure may need to bubble to a parent state that performs cleanup.
- If a child state loops to itself on success/failure, make sure parent interruption still releases the claim.
- If a global task owns claim state, its Stop path must release or hand off deliberately.

Write state names and transition reasons as evidence: `AcquireClaim`, `ReachSlot`, `UseLoop`, `InterruptedRelease`, `FailedRelease`. A tree that looks elegant but does not explain failure handling will be hard to debug in production.

## Choosing StateTree, Behavior Tree, or code

Use the smallest model that matches the semantics.

| Need | Good fit | Why |
|---|---|---|
| Reactive combat priority, perception-driven action selection | Behavior Tree | mature AI action-selection/debug pattern |
| Stateful interaction sequence with enter/exit/abort behavior | StateTree | explicit states, hierarchical transitions, shared context |
| Three local modes on one component | C++/Blueprint enum or small FSM | avoids asset/tool overhead |
| Designer-authored ambient world activities | Smart Object + StateTree | data-driven affordance plus structured lifecycle |
| High-volume crowd slots | Smart Object + Mass/StateTree only after target verification | plugin/version/performance sensitive |

Smart Objects should usually call stable capabilities rather than embed all mechanics. A "UseWorkbench" state can call an inventory/component/ability contract. It should not mutate every possible crafting rule inside a StateTree task.

## Multiplayer and authority

For networked gameplay, treat Smart Object use as gameplay state, not only local animation. The usual policy is:

- clients may discover or preview possible interactions for responsiveness;
- the server validates and owns durable claim/use outcomes;
- client UI can show "Press Use" only as a proposal;
- server validates distance, line of sight, object enabled state, permissions, cooldown, tags, inventory, team, and claim availability;
- replicated state drives other clients' presentation;
- cosmetic prediction is allowed only if rollback/correction is acceptable.

Do not send "I used slot 3 and got reward" as a trusted client result. Send intent: object, slot/claim candidate, verb, timing, target evidence if needed. The authoritative system decides whether it succeeds. [SRC-NET-004] [SRC-AI-012]

For AI, run authoritative claim and behavior decisions on the server unless the interaction is purely cosmetic. For local single-player code paths, keep the same lifecycle boundaries so the system can survive later multiplayer or replay requirements.

## Navigation, animation, and ability boundaries

A claimed slot is not a reached slot. Always validate movement separately:

- project approach point to the correct NavMesh;
- request path with the right agent properties;
- handle partial path, blocked path, movement mode changes, and acceptance radius;
- handle root-motion/physics/animation ownership conflicts;
- abort or retry if the object moves or becomes disabled before arrival.

Animation and ability code should also stay separate. A Smart Object can select an animation tag or behavior asset, but the animation system owns montage/notify/root-motion correctness, and GAS or gameplay components own costs, cooldowns, tags and effects. This keeps Smart Object code from becoming a second ability system.

## Data design pattern

For a reusable interaction, define a small authored contract:

| Field | Purpose | Example |
|---|---|---|
| Activity tag | semantic use | `Activity.Rest`, `Activity.Cover.Low`, `Activity.Repair` |
| User requirement tags | who can use | `NPC.Humanoid`, `HasTool.Wrench`, `Faction.Civilian` |
| Slot role | multi-user meaning | `SeatLeft`, `CounterCustomer`, `RepairOperator` |
| Approach policy | movement setup | stand at slot, approach offset, nav link, require line of sight |
| Behavior reference | execution | StateTree, gameplay behavior, ability tag, animation set |
| Cooldown / lock | reuse policy | 10 seconds after use, disabled while powered off |
| Abort policy | cleanup | release claim, stop montage, clear tags, notify object |

The goal is not to generalise every object into one mega-definition. A generic contract should remove duplication while preserving strong invariants.

## Debugging workflow

For "NPC will not use the chair":

1. **Registration:** confirm the actor exists, the Smart Object component is enabled, and its definition is assigned.
2. **Slot authoring:** inspect slot transforms, tags, requirements, behavior reference, and object enabled/cooldown state.
3. **Query:** log request origin, radius, filters/tags, user data and candidate count.
4. **Claim:** separate "found candidate" from "claim succeeded"; log claim owner and failure reason if available.
5. **Navigation:** project start/goal, inspect NavMesh, path result, partial-path policy and movement component state.
6. **StateTree:** inspect active state path, task outputs, transition reason, context data, evaluator values and global-task lifecycle.
7. **Use behavior:** verify animation/ability/component call started, completed and handled abort.
8. **Release:** confirm release runs on success, fail, abort, user destruction and object destruction.
9. **Authority:** in multiplayer, repeat on server and owning/client proxy; look for local-only claim or stale replicated state.
10. **Evidence:** use AI Debugger/Visual Logger for spatial failures and StateTree debugging/logging for transition/data-flow failures. [SRC-AI-008] [SRC-AI-009] [SRC-AI-010]

Do not start by changing selector priority. First prove which lifecycle phase failed.

## Profiling workflow

Smart Object and StateTree performance problems are usually authored scale problems:

1. Count agents, active StateTrees, active tasks, query frequency, query radius, candidate count and slot count.
2. Back off failed queries; do not let every NPC search every frame for the same unavailable slot.
3. Prefer event-driven requery when activities become available, perception changes, or an agent changes intent.
4. Filter cheaply by tags/activity before expensive path, trace or line-of-sight validation.
5. Stagger ambient activity decisions across frames.
6. Avoid running high-frequency monitored transitions for conditions that could be event-driven.
7. Keep task outputs small and explicit; avoid per-tick allocation in evaluators/tasks.
8. For crowds, prove whether Actor AI is still appropriate or whether Mass/representation LOD should own distant agents.
9. Profile in a representative level. A tiny test map can hide spatial-query and path-contention costs.

## Common bugs and misconceptions

1. **"Found" means "reserved."** Query results can race; claim is the ownership boundary.
2. **No abort release path.** The object stays busy after movement fails, death, despawn, alert interruption or tree stop.
3. **Slot transform treated as reachable.** NavMesh/path following can still reject it.
4. **StateTree task completion transitions too early.** A helper task finishes and interrupts the real action.
5. **Smart Object owns all gameplay.** It should coordinate affordance; gameplay systems own durable rules.
6. **Client-local use is trusted.** Server must validate durable interactions.
7. **One mega Smart Object definition.** Over-generalisation hides requirements and makes debugging harder.
8. **Data binding is invisible magic.** Parameters, context data, evaluators and task outputs need naming and ownership.
9. **Tags are decoration.** Activity/user/slot tags are contract surfaces; inconsistent taxonomy breaks queries.
10. **Behavior Tree, StateTree and Smart Objects are interchangeable.** They solve related but different problems.

## Strong interview answer patterns

### "What are Smart Objects?"

> Smart Objects are a way for world objects to advertise usable affordances, usually through definitions and slots. An agent queries for a matching opportunity, claims a slot, reaches it, executes behavior, then releases it. The important part is the lifecycle: found is not claimed, claimed is not reached, use can fail, and cleanup must happen on every abort path. I keep gameplay authority, movement, animation and ability rules in their own systems.

### "How would you build NPCs using chairs, terminals and cover points?"

> I would model each as an affordance with activity tags, user requirements, slots, approach policy and behavior reference. The AI decision layer decides it wants Rest, Hack or TakeCover; the Smart Object query finds candidates; the server or authoritative AI claims a slot; navigation moves to an approach point; StateTree runs Reach/Use/Exit states; the interaction releases on success, failure, death, interruption or object disable. I would debug by separating query, claim, movement, behavior and release instead of treating it as one task.

### "Why use StateTree for Smart Object behavior?"

> Smart Object interactions are often explicit sequences with enter, active, fail and exit behavior. StateTree gives hierarchical states, transitions, shared context data and task grouping, so Acquire, Reach, Use and Cleanup can be visible and testable. I still keep mechanics behind components or abilities, and I would use a simpler FSM or Behavior Tree when that better matches the problem.

### "How do you debug two NPCs using the same slot?"

> First I prove whether they both only found the same candidate or both actually claimed it. Then I inspect claim ownership, release timing, authority, and whether one path is client-only or bypasses the subsystem. I add logs around query result, claim success/failure, user identity, claim handle lifetime, StateTree abort/stop and object destruction. If claim is correct but animation overlaps, I check slot transforms, approach offsets, animation alignment and crowd avoidance.

## What a three-year engineer should know

Know the Smart Object lifecycle, slot/definition/instance distinction, why claim/release is the concurrency boundary, how StateTree selection/tasks/transitions/data binding work, when StateTree is a good fit versus BT or a small FSM, how to separate movement/animation/ability/authority from affordance coordination, and how to debug query, claim, movement, behavior and release as separate phases.

**Specialist depth:** custom Smart Object filters, runtime registration/streaming behavior, Mass Smart Object integration, StateTree custom schemas/evaluators/tasks/conditions in C++, data-view internals, replicated interaction prediction, save/load of object state, large-world spatial query cost, and branch-specific debugger categories.

## Hands-on verification task

Build the Project 2 Smart Object and StateTree extension:

1. Create three Smart Object affordances: rest chair, cover point and repair terminal.
2. Give each at least one slot, activity tags, user requirements and behavior reference.
3. Implement an AI decision that chooses an activity from need/state, then queries matching Smart Objects.
4. Implement explicit claim/use/release logging.
5. Use StateTree for `Acquire -> Reach -> Use -> Exit`.
6. Inject failures: slot already claimed, unreachable slot, object disabled mid-use, agent killed mid-use, combat interruption.
7. Record evidence that release runs in every failure.
8. Repeat with two agents racing for one slot.
9. Add a server-authoritative validation path if networked.
10. Profile 1, 20 and 100 agents with staggered versus synchronous query frequency.

## Source caveats

- `SRC-AI-010` is the strongest official source for StateTree selection, task execution, data flow and Smart Object task grouping.
- `SRC-AI-011` through `SRC-AI-015` are official Smart Object overview/API anchors. The docs are useful for traceability, but implementation should verify headers and behavior in the target UE5.3-UE5.6 branch.
- Smart Object, StateTree, Mass, and Gameplay Behavior integration is plugin/project-sensitive. Do not copy feature availability or debugger UI claims between minor versions without target proof.
