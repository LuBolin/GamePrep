# Unreal Animation Systems for Interviews

See also: [[ue_runtime_presentation_target_proof]], [[ue_physics_collision]], [[ue_gameplay_ability_system]], [[ue_hands_on_projects]].

Animation in Unreal is a runtime pose pipeline layered over imported skeletal assets and driven by gameplay/movement state. A strong engineer can separate gameplay truth from visual pose, choose state machine versus montage versus procedural layer, diagnose timing/space/thread failures, and scale many characters without turning the Animation Blueprint into a second gameplay framework.

Version target: **UE5.3–UE5.6**. Core concepts are stable, but maintained Epic pages often display UE5.7. IK Rig, Control Rig, Motion Warping, Pose Search/Motion Matching and budget/debug tooling evolve quickly and must be checked in the target branch.

## 1. The end-to-end mental model

```text
authored/imported data
  Skeletal Mesh + Skeleton + Animation Sequences/curves/notifies/root motion
        ↓
gameplay/movement snapshot
  velocity, acceleration, movement mode, aim, action/state, equipment
        ↓
Animation Blueprint update
  derive animation parameters with a safe thread/lifetime contract
        ↓
AnimGraph evaluation
  state machine + asset players + blend spaces + montages/slots + layers + IK
        ↓
final component-space bone pose + curves/attributes/root-motion contribution
        ↓
Skeletal Mesh skinning/deformation and presentation
```

The skeletal animation system starts from a Skeletal Mesh and Skeleton; Animation Sequences contain keyed bone transforms and can be blended/evaluated by Animation Blueprints. [SRC-ANIM-001] [SRC-ANIM-002]

### The three ownership rules

1. **Gameplay owns durable truth.** Health, ammo, attack permission and authoritative state do not live only in an animation state or notify.
2. **Movement owns authoritative displacement by default.** Root motion is an explicit alternative/ingredient coordinated with CharacterMovement and networking.
3. **Animation owns pose and presentation policy.** It derives locomotion/action presentation from a bounded snapshot and can request cosmetic/gameplay windows through explicit contracts.

Violating these boundaries creates the classic bugs: attack damage exists only if a notify fires; animation decides whether reload is legal; CharacterMovement and root motion fight over location; replicated montage playback is mistaken for replicated gameplay state.

## 2. Asset and skeleton layer

### Skeletal Mesh

A Skeletal Mesh combines skinned geometry, material sections, LODs, skin weights/morph targets and a bone hierarchy association. The mesh is presentation data; it should not become gameplay identity.

### Skeleton asset

The Skeleton provides the bone hierarchy/reference relationship used by animation assets and sharing/compatibility workflows. Bone names, parent relationships, reference pose, retarget setup, curves, slots and notify metadata all affect downstream assets. Changing hierarchy or reference pose is a pipeline/schema change, not a harmless mesh edit.

### Animation Sequence

An Animation Sequence contains time-sampled/keyed bone translation, rotation and scale for a Skeleton, plus optional curves, notifies and root-motion data. Compression reduces memory/storage at a quality/cost trade-off. [SRC-ANIM-002]

For each sequence, establish:

- source Skeleton/reference pose and intended mesh proportions;
- sample rate/range, looping and additive settings;
- root bone and root-motion policy;
- curve names/semantics and notify ownership;
- compression settings and visual error budget;
- retarget/export/import version and source file;
- sync markers/contact phases where required.

### Additive animation

An additive pose represents a delta relative to a base/reference pose, then combines with an input pose. Local-space additive suits many overlays; mesh-space additive is commonly used for Aim Offsets because orientation deltas are evaluated in mesh space. A wrong additive base or space produces exploding/leaning limbs even when the source sequence looks correct.

### Compression

Animation compression trades memory/streaming and decode quality/cost. Test end effectors, fingers/facial curves and high-frequency motion at representative LOD/camera distances. “Small asset” does not prove acceptable visual error; “looks fine in editor” does not prove runtime memory/decode budget.

## 3. Retargeting and compatibility

Retargeting transfers animation between characters/skeletons/proportions. UE5 IK Rig/IK Retargeter uses source/target IK Rigs, retarget chains, roots and retarget poses, and can preserve contacts with IK. UE5.6 documentation explicitly describes transfer across differing bone counts/names/orientations. [SRC-ANIM-013]

### Retarget debugging order

1. Confirm source/target preview meshes and exact Skeleton assets.
2. Check root and pelvis/retarget root selection.
3. Check chain start/end bones and chain mapping.
4. Match source/target retarget poses (A/T-pose mismatch is not cosmetic).
5. Check forward axis, scale/proportion and root translation settings.
6. Disable root/FK/IK passes independently to localise error.
7. Inspect output log and compare one simple reference pose/sequence.
8. Check end-effector goals, stride warping and LOD operation thresholds.

Do not repair a systematic retarget-pose mismatch with per-animation offsets. Keep source, retarget setup and exported results versioned/reproducible.

## 4. Animation Blueprint lifecycle and threads

An Animation Blueprint (`UAnimInstance` class at runtime) controls a Skeletal Mesh's animation behaviour. Its AnimGraph evaluates pose nodes; its EventGraph and functions update data used by that graph. [SRC-ANIM-003] [SRC-ANIM-004]

### Update versus evaluate

Conceptually separate:

- **initialisation:** establish owner/component references and defaults;
- **update:** advance node state and derive parameters for the frame;
- **evaluate:** produce pose/curve/attribute/root-motion output;
- **post-evaluate/application:** final pose becomes available to the component and dependent systems.

Exact scheduling is engine/version-sensitive, and update/evaluation may occur on different threads. Never use an AnimGraph node as an arbitrary game-thread UObject callback surface.

### Thread model

Epic's graphing documentation states that the EventGraph runs on the game thread, while AnimGraph work can run on worker threads. `Blueprint Thread Safe Update Animation` and property access provide supported routes to worker-thread-safe data derivation; unsafe operations can force/warn/fail optimisation. [SRC-ANIM-004] [SRC-ANIM-015]

Worker-thread animation code must not casually:

- mutate Actors/Components/UObjects;
- perform world queries or spawn objects;
- call arbitrary Blueprint/gameplay functions;
- traverse mutable containers without a synchronisation/snapshot contract;
- retain transient pointers/views across evaluation;
- dispatch gameplay events whose listeners assume the game thread.

### Snapshot boundary

A robust architecture gives animation a compact read-only snapshot:

```text
movement/gameplay source → stable frame data → thread-safe Anim update → pose graph
```

Examples: velocity, acceleration, movement mode, ground state, aim rotation, stance/equipment tag/enum, action phase and explicit montage request. Prefer property access or project-owned proxy/snapshot patterns over repeated `TryGetPawnOwner → Cast → Component queries` in a hot EventGraph.

### Push versus pull nuance

“Animation should always pull” and “gameplay should always push” are both too broad:

- animation can safely derive presentation parameters from a stable owner snapshot/property access;
- gameplay can issue discrete requests such as play/cancel action montage through a narrow animation interface;
- gameplay should not micromanage every blend variable;
- animation should not query half the world each frame.

The contract is one-way truth plus explicit requests/events, not dogma about one call direction.

## 5. AnimGraph composition

The AnimGraph is a dataflow graph of poses plus curves, attributes, sync data, root motion and inertialisation information. It is evaluated according to relevance and blend weights, not like ordinary Blueprint event execution. [SRC-ANIM-004]

Typical order:

1. base locomotion state machine/pose selection;
2. cached base pose for multiple consumers;
3. upper-body/equipment layer through slots or linked layers;
4. additive aim/look/recoil;
5. IK/warping and targeted bone controls;
6. final output pose.

Order is part of behaviour. Applying Aim Offset before versus after an upper-body override, or IK before versus after a montage, can erase or double an effect.

### Save Cached Pose

Cache a pose when the same expensive upstream graph is consumed multiple times. It is not a generic memoisation guarantee across arbitrary frames; it shares the evaluated pose within graph semantics. Over-caching adds memory/complexity and can obscure relevance—profile.

### Blend nodes

Blend nodes combine pose sources by alpha, bone masks/layers, additive rules, blend profiles or inertialisation. [SRC-ANIM-007]

Common failures:

- alpha outside intended range;
- branch still evaluated unexpectedly due node/settings;
- wrong additive space/base;
- Layered Blend per Bone begins at wrong bone or blend depth;
- mesh-space rotation blend produces different result from local-space;
- curve/root-motion handling differs between branches;
- abrupt source change without suitable transition/inertialisation.

## 6. State machines

Animation state machines select persistent pose regimes such as grounded locomotion, falling, swimming or incapacitated. A state contains a pose graph; transition rules determine when and how blending moves between states. [SRC-ANIM-005]

### Good state boundaries

Create a state when it has a distinct persistent pose source and transition lifecycle—not for every gameplay Boolean. Keep gameplay's authoritative state machine separate; map its stable facts into presentation states.

Examples:

- Grounded / InAir is a useful locomotion partition.
- Idle/Walk/Run often belong in one Blend Space rather than three transition-heavy states.
- Fire/reload/melee reactions are frequently montages layered over locomotion, not locomotion states.

### Transition rule design

Rules should be cheap, deterministic from the animation snapshot and explainable. Use remaining time/automatic rules only when the asset player and relevance assumptions are clear. Define:

- source and destination condition;
- blend duration/profile/inertialisation;
- interruption/priority;
- re-entry/reset behaviour;
- whether transition can be taken from multiple sources;
- what happens when state becomes irrelevant/skipped by update throttling.

### Conduits

A conduit centralises a shared branch decision among several destinations (for example death type or locomotion landing variant). It reduces duplicated transition conditions; it should not become a hidden gameplay state machine.

### State-machine debugging

Capture current state, elapsed/remaining time, transition rule values, blend weight and source asset time in Animation Blueprint Debugger/Rewind Debugger. If a state is never entered, distinguish “rule false” from “state machine not relevant” and “snapshot stale”.

## 7. Blend Spaces and Aim Offsets

A Blend Space interpolates Animation Sequence samples over one or two input axes; locomotion commonly uses speed/direction or local velocity. Aim Offsets are mesh-space additive Blend Spaces layered over an input pose. [SRC-ANIM-006]

### Axis design

Use variables with stable semantics and calibrated ranges:

- speed magnitude, not input magnitude, if feet should track actual motion;
- local-space velocity direction, not raw world yaw, for strafing;
- acceleration/has-acceleration separately when start/stop behaviour matters;
- aim yaw/pitch normalised relative to actor/control basis.

### Common blend-space bugs

- world velocity fed where local direction expected;
- direction wraps at ±180 and causes sample flipping;
- ranges do not match actual movement speeds;
- smoothing duplicates CharacterMovement acceleration and creates lag;
- sample animations have mismatched stride/contact phase;
- triangulation/sample placement causes diagonal artefact;
- one sample contains root motion/additive settings inconsistent with others;
- evaluator teleports normalised time, suppressing notify/root-motion evaluation as documented. [SRC-ANIM-006]

## 8. Sync Groups and markers

Sync Groups synchronise related animations of different lengths during blending. A leader supplies timing; followers adjust. Leaders also have priority for triggering notifies according to Epic's documentation. [SRC-ANIM-011]

Use locomotion sync markers for semantic contacts such as left/right foot rather than assuming equal normalised time corresponds to equal gait phase. Define missing-marker and leader-change behaviour. A notify disappearing during a blend may be a sync leadership/relevance issue, not a broken notify asset.

## 9. Montages, slots, sections, and action playback

Montages combine sequences into controllable runtime action assets with sections, branching/control, slots, notifies and root motion. Slots are insertion points in the AnimGraph; slot groups affect concurrency/override behaviour. [SRC-ANIM-008]

### Montage versus state machine

Use a state machine for persistent mutually exclusive pose regimes whose transition graph is continuously evaluated. Use a montage for discrete directed actions—attack, reload, hit reaction, interaction—especially when gameplay starts/stops/jumps sections and the action layers through a slot.

Not every one-shot needs a montage, and not every action belongs outside the state machine. Ask:

- must gameplay explicitly trigger/cancel it?
- does it need sections/branching windows?
- must locomotion continue underneath?
- does it carry root motion?
- does it need network correlation/prediction?
- what wins if two actions use the same slot/group?

### Sections and branching

Sections support loops/combos and runtime jumps, but gameplay must own the legal action transition. Montage sections are presentation/action timing, not a secure combo state by themselves. Networked section changes need an explicit authoritative/prediction contract.

### Slot placement

A montage only affects the final pose if its Slot node is present and relevant in the graph. Place upper-body slot relative to cached locomotion, layered blend and Aim Offset deliberately. Wrong slot/group or branch relevance explains “Montage_Play succeeded but nothing appears”.

### Delegates and termination

Handle blend-out, interruption and end separately where the API distinguishes them. Bind/unbind with owner lifetime; correlate by montage/ability/action ID. Do not assume a notify or montage completion fires on every cancellation, mesh swap, relevancy or teardown path.

## 10. Animation Notifies and Notify States

A Notify marks an instant timeline point. A Notify State owns a begin–tick–end window. Skeleton notifies are named events; notify objects/classes can carry reusable behaviour/data. [SRC-ANIM-010]

Good uses:

- cosmetic sound/VFX/camera feedback;
- footstep presentation with surface query on the correct machine;
- opening/closing an authoritative attack trace window through a validated action owner;
- attachment visibility or temporary presentation;
- combo-input window notification to an existing gameplay action.

Bad uses:

- authoritative damage with no server/action validation;
- deducting ammo only when a client notify occurs;
- durable state that must survive skipped animation updates;
- assuming exactly-once delivery across blending, looping, dedicated server or LOD throttling.

### Notify contract

Specify:

- branching point versus queued/tick behaviour where applicable;
- server/owning client/simulated proxy audience;
- weight/relevance threshold;
- whether sync leader/follower can fire it;
- repeat/loop semantics;
- cancellation/end fallback;
- deduplication/correlation ID;
- whether it is cosmetic, request or authoritative commit.

## 11. Root motion and CharacterMovement

Root motion extracts root-bone displacement from animation and applies it to the character/movement path rather than leaving the mesh to drift away from its collision capsule. [SRC-ANIM-009]

### In-place locomotion

CharacterMovement/physics moves the capsule; animation visualises velocity. Advantages: responsive control, straightforward network movement prediction/correction and easy speed tuning. Foot sliding must be managed through animation matching, stride/pose warping or speed/play-rate design.

### Root-motion actions

Animation supplies displacement for authored attacks/vaults/interactions. Advantages: grounded authored trajectory/contact. Costs: collision/path adaptation, authority/prediction, interruption and target alignment become harder.

### Root-motion modes

Know the design distinction between root motion from everything and montage-focused modes, but verify exact enum/behaviour in UE5.3–UE5.6. Epic's maintained documentation notes that some root-motion modes affect worker-thread evaluation and that network games need deliberate montage/RPC handling. [SRC-ANIM-008] [SRC-ANIM-009]

### Network contract

Replicating a montage command or root-motion correction does not replicate gameplay outcome. Server owns action permission, target/collision/result and authoritative position. Owning client prediction, if used, requires correlation and correction; simulated proxies need enough action/pose state for presentation. Test dedicated server, latency/loss, interruption and collision blocking.

### Root-motion failure matrix

- mesh moves then snaps/capsule stays: extraction/application mode or asset root setup;
- no movement: root locked/extraction disabled/wrong montage mode/graph branch;
- double movement: CharacterMovement and animation both apply displacement;
- rotates/translates wrong axis: import/root/reference transform;
- network jitter: authority/prediction/correction mismatch;
- action misses target: fixed authored displacement without warp/target validation;
- worker-thread performance changes: root-motion mode or unsafe graph path.

## 12. Motion Warping, IK Rig, and Control Rig awareness

### Motion Warping

Motion Warping adjusts root-motion windows to align an animation with a runtime warp target. It can help a vault/finisher reach a ledge/enemy, but gameplay must validate target and collision. Define window, translation/rotation warp policy, target lifetime and failure fallback. A warp does not make unreachable geometry valid. [SRC-ANIM-014]

### IK

Inverse kinematics solves joint poses from goals/constraints: foot placement, hand contact, look/aim, reach. Inputs need explicit space, solver order and limits. Apply IK after the pose layers it is intended to correct; otherwise later layers overwrite it. [SRC-ANIM-013]

Common failures:

- trace/goal in world space fed to component/local solver input;
- pelvis offset and feet both compensate twice;
- no smoothing/hysteresis on contact normal;
- leg overextends because chain/limits/effector wrong;
- IK evaluated on LOD where required bones removed;
- traces run per foot per character without significance/budget;
- contact is visual only but gameplay assumes physical support.

### Control Rig

Control Rig is a procedural rigging/animation framework for editor and runtime workflows, including Full-Body IK. It is powerful but can be expensive and version-sensitive. Use for a clearly needed procedural/deformation layer; profile VM/solver/bone scope and set LOD/quality policy. [SRC-ANIM-018]

### Motion Matching/Pose Search awareness

Motion Matching selects poses from a database using trajectory/pose features rather than a hand-authored locomotion transition graph. It changes authoring/data/debugging costs; it does not remove gameplay movement or animation layering. Status, databases, schemas and APIs changed substantially across UE5, so treat it as specialist/version-sensitive until the dedicated cluster. [SRC-ANIM-001]

## 13. Linked Anim Graphs and Linked Anim Layers

Linked Animation Blueprints/Layers modularise graph logic. An Animation Layer Interface defines overridable pose-layer contracts; equipment/vehicle-specific classes can replace layers and can be loaded/unloaded with those features. [SRC-ANIM-012]

Good boundary:

- base class owns universal locomotion and final composition;
- interface names semantic extension points (`WeaponUpperBody`, `VehiclePose`);
- equipment layer owns only weapon-specific pose logic/assets;
- gameplay/equipment system selects the layer class;
- layer change has explicit transition, montage cancellation and asset-lifetime policy.

Failure modes:

- circular asset/class references defeat streaming;
- interface pose/curve/root-motion expectations undocumented;
- linked instance reads wrong owner or stale equipment;
- rapid layer swap leaves montage/delegate state;
- layers duplicate expensive locomotion evaluation;
- weapon asset remains hard-referenced by base AnimBP.

## 14. Gameplay–animation interface

### Data flowing into animation

Prefer a small semantic snapshot:

```text
FAnimPresentationState
  VelocityWorld / AccelerationWorld
  MovementMode / bGrounded
  AimRelative
  Stance / Gait / EquipmentPresentation
  ActionPresentationState + Revision
```

Do not mirror every gameplay variable. Include only values needed to choose/shape pose.

### Requests flowing to animation

Examples:

- play action montage with action ID, section and predicted/confirmed context;
- cancel action presentation by action ID/reason;
- link equipment layer;
- set one-shot cosmetic emphasis.

Return structured completion/interruption callbacks, but gameplay must have a timeout/cancellation path and remain authoritative.

### Events flowing out

Notifies can emit cosmetic events or report timing windows to the action owner. The receiver validates that the action is still current and that the event has not duplicated. Animation does not reach directly into Inventory, Quest, Damage and Network systems.

## 15. CharacterMovement integration

Derive locomotion from actual movement state:

- velocity and acceleration from the movement component;
- movement mode for grounded/falling/swimming/flying;
- actor/control/velocity orientation policy;
- braking, rotation mode and network smoothing awareness;
- floor/landing data through a stable project contract.

Animation can lag or lead movement due to tick order, network smoothing, interpolation, thread snapshots and blend smoothing. Before adding arbitrary delay, record movement values and animation-consumed values on the same frame/timeline.

### Foot sliding diagnosis

1. Is capsule speed equal to assumed clip speed?
2. Are Blend Space axes/sample speeds calibrated?
3. Are gait phases synchronised?
4. Is play rate adjusted deliberately?
5. Is network smoothing moving mesh relative to capsule?
6. Is root motion accidentally mixed with in-place motion?
7. Is retarget proportion changing stride?
8. Is IK pinning feet while root/capsule continues?

Fix the responsible layer; IK alone is often cosmetic plaster.

## 16. Networking and cosmetic replication

Replicate durable gameplay state or an authoritative action record, then derive presentation. Use multicast/cosmetic events sparingly for transient effects that do not need late-join recovery.

For an attack:

1. owning client sends/ predicts an action request;
2. server validates and records authoritative action phase/revision/start time;
3. clients play/correct the montage from that state;
4. server opens validated hit window or processes server trace;
5. notify/cue presents swing/contact;
6. cancellation/death updates durable state; animation exits even if an event was missed.

Questions:

- Can a late joiner reconstruct the pose/action?
- Does relevancy loss/resume need current section/time?
- Does a dedicated server evaluate notifies/bones at all required rates?
- Are montage sections deterministic across content versions?
- What is predicted versus confirmed presentation?

## 17. Debugging workflow

### Wrong or frozen pose

1. Confirm correct Skeletal Mesh, Skeleton and Anim Class at runtime.
2. Confirm component animation mode and AnimInstance exists.
3. Inspect AnimBP debug object; record with Rewind Debugger.
4. Check snapshot values/thread-safe property access.
5. Check state/transition/relevance and asset player time.
6. Pose Watch upstream nodes to find first wrong pose.
7. Check montage slot/group/weight and linked layer class.
8. Check LOD, update-rate/budget and visibility tick policy.
9. Check retarget/additive/root-motion settings.

### Montage does not play visibly

1. Check call result and correct mesh/AnimInstance.
2. Check montage Skeleton compatibility.
3. Check required Slot node is in the active final graph branch.
4. Check slot group conflict/another montage override.
5. Check state/layer later overwrites the bones.
6. Check play rate, section, blend and interruption callback.
7. On network, check which machine requested/replicated it.

### Notify missing or duplicated

Inspect asset time crossing, loop, blend weight/relevance, sync-group leader, montage section jumps, update throttling/LOD, branching/queued type and network audiences. Add action/sequence ID deduplication and a durable fallback instead of moving the notify randomly.

### Retarget deformation

Inspect hierarchy/chain/root/retarget poses before per-animation correction. Disable passes, compare reference pose and inspect end-effector/root output.

Epic's Rewind Debugger, Pose Watch and Animation Insights preserve state transitions, variables, pose-node influence and trace cost for diagnosis. [SRC-ANIM-016] [SRC-ANIM-019]

## 18. Profiling and optimisation

Separate costs:

1. game-thread AnimBP EventGraph/data gathering;
2. worker-thread update/evaluate and node/solver cost;
3. skeletal mesh component finalisation/bone transforms;
4. skinning/deformation/render cost;
5. traces/queries supporting IK;
6. asset memory/compression/streaming;
7. notifies/cosmetic spawn cascades.

Measure with Unreal Insights/Animation Insights, relevant `stat anim`/component stats for the target branch, Animation Blueprint debugger and platform captures. Record character count by LOD/significance and per-node/system timing. [SRC-ANIM-016] [SRC-ANIM-019]

### Optimisation order

- remove game-thread EventGraph polling/casts and unsafe Blueprint functions;
- use thread-safe update/property access/Fast Path-compatible graphs where applicable;
- avoid evaluating irrelevant duplicate pose branches; cache deliberate shared bases;
- reduce bones/solver/layer complexity by skeletal LOD;
- lower update rate and interpolate for less significant characters;
- disable IK/Control Rig/curves/morphs/notifies where product permits;
- share Leader/Copy Pose architectures for modular meshes after validating constraints;
- tune compression/memory and async asset boundaries;
- budget traces and cosmetic effects separately.

The Animation Budget Allocator is a plugin that throttles Skeletal Mesh Component ticking/update rates/interpolation against a game-thread budget using significance. It trades temporal fidelity for bounded work and requires product-specific essential/always-tick policy. [SRC-ANIM-017]

### Budget failure tests

- important off-screen local player arms incorrectly throttled;
- combat notify/hit window depends on skipped visual update;
- ragdoll/cloth/attachment needs current bones;
- low-rate interpolation visibly penetrates ground;
- significance oscillates and creates update popping;
- server gameplay depends on bone evaluation that budget disables.

## 19. What a three-year engineer should know

Expected D3–D4:

- asset/Skeleton/AnimSequence/AnimBP pipeline;
- EventGraph versus AnimGraph and thread-safe snapshot boundary;
- locomotion state machine/Blend Space/Aim Offset;
- montage/slot/section/notify lifecycle;
- in-place versus root-motion trade-off with CharacterMovement/networking;
- layering/additives/sync groups/linked layers awareness;
- retarget/IK diagnosis at working depth;
- Pose Watch/Rewind/Insights and performance triage.

Specialist D4–D5:

- custom anim nodes/proxies/attributes;
- engine update/evaluate/task scheduling internals;
- Motion Matching/Pose Search schema/database internals;
- advanced Control Rig/FBIK/deformer runtime;
- compression codecs and SIMD pose evaluation;
- network root-motion source/prediction internals;
- platform skinning/memory optimisation.

## 20. Common misconceptions

| Misconception | Better model |
|---|---|
| “AnimBP is the character state machine.” | It maps authoritative gameplay/movement state to pose presentation. |
| “AnimGraph is just Blueprint Tick.” | It is pose dataflow with relevance and worker-thread evaluation constraints. |
| “Notifies are reliable gameplay events.” | They are timeline events affected by evaluation/blending/network; validate and provide durable state paths. |
| “Montages replicate automatically.” | Root-motion support exists, but play/action state and gameplay outcomes need deliberate replication. |
| “Root motion is more realistic.” | It trades control/network/collision flexibility for authored displacement; choose per action. |
| “IK fixes foot sliding.” | Sliding often originates in capsule speed, phase, retarget, root-motion or smoothing mismatch. |
| “Blend Space interpolates anything correctly.” | Inputs, ranges, sample phase/additive/root settings and triangulation determine quality. |
| “Thread safe means pure Blueprint function.” | Access and called operations must obey the worker-thread contract; use supported property access/snapshots. |
| “Animation Budget Allocator is free optimisation.” | It trades update frequency/quality and can skip timing-dependent visual events. |
| “Linked layers are only graph organisation.” | They are runtime class/asset/lifetime boundaries with transition and dependency implications. |

## 21. Strong interview answers

### Montage versus state machine

> I use a state machine for persistent pose regimes whose transitions are continuously derived, such as grounded and in-air locomotion. I use a montage for a discrete gameplay-directed action with sections, cancellation, notifies, root motion or slot layering, such as reload or melee. Gameplay owns action legality and durable phase; the montage presents it and reports correlated timing. I verify slot placement, interruption and network/late-join behaviour.

### Gameplay to animation

> Gameplay and movement expose a compact stable presentation snapshot; the AnimBP derives pose parameters through thread-safe update/property access. Gameplay can request discrete montage/layer changes through a narrow interface, and notifies can report cosmetic or validated timing windows back with an action ID. Animation never owns health, ammo or attack authority, and gameplay has cancellation/timeouts if presentation is skipped.

### Animation performance

> I split game-thread data gathering, worker update/evaluate, skeletal finalisation, skinning/rendering, IK queries and asset memory. I capture Animation Insights/Insights with character/LOD counts, remove EventGraph polling and unsafe graph calls, then reduce duplicated pose evaluation and expensive IK/layers. Finally I apply skeletal LOD/update-rate/significance or the plugin budget allocator, while testing which characters and gameplay-adjacent events must never be throttled.

## 22. Hands-on verification project

Extend Project 1 or 9 with an animation vertical slice:

1. import/document one Skeleton, two meshes and locomotion/action sequences;
2. implement thread-safe movement snapshot and AnimBP locomotion state machine;
3. add calibrated speed/direction Blend Space, Aim Offset and sync markers;
4. layer upper-body reload/fire montage through slots and linked weapon layer;
5. implement server-validated melee trace window with notify-state correlation and cancellation fallback;
6. compare in-place dash with root-motion montage under dedicated-server lag;
7. retarget to a differently proportioned mesh and debug root/chain/pose/contact;
8. add foot IK and one Motion Warping target with explicit failure policy;
9. inject wrong slot, stale owner, missing notify, wrong additive base, world/local IK error, root-motion double movement and budget-throttle failure;
10. capture Rewind Debugger/Pose Watch/Animation Insights and scale 1/20/100 characters by LOD/significance.

Deliver an asset dependency diagram, gameplay-animation contract, montage/notify failure matrix, thread map, root-motion network test and measured optimisation report.

## 23. Source map

- system/assets/sequences: [SRC-ANIM-001] [SRC-ANIM-002]
- Animation Blueprints/threads/state/blending: [SRC-ANIM-003] [SRC-ANIM-004] [SRC-ANIM-005] [SRC-ANIM-006] [SRC-ANIM-007]
- montages/root motion/notifies/sync: [SRC-ANIM-008] [SRC-ANIM-009] [SRC-ANIM-010] [SRC-ANIM-011]
- modular layers/retarget/IK/warping/Control Rig: [SRC-ANIM-012] [SRC-ANIM-013] [SRC-ANIM-014] [SRC-ANIM-018]
- optimisation/debugging/profiling/budget: [SRC-ANIM-015] [SRC-ANIM-016] [SRC-ANIM-017] [SRC-ANIM-019]
