# Unreal Physics and Collision for Interviews

See also: [[ue_runtime_presentation_target_proof]], [[game_math_for_interviews]], [[ue_animation_systems]], [[ue_hands_on_projects]].

Unreal physics work begins with collision semantics, not with “turn on Simulate Physics”. A strong engineer can predict filtering and query results, distinguish movement sweeps from rigid-body simulation, choose simple/complex shapes, reason about forces/timestep/network authority, and reconstruct failures from recorded scene-query/contact/constraint evidence.

Version target: **UE5.3–UE5.6**. Chaos is the UE5 physics engine. Collision fundamentals are target-documented, but networked physics modes and Chaos Visual Debugger capabilities evolve rapidly; maintained UE5.7 pages are specialist awareness unless confirmed in the target branch.

## 1. The layered model

```text
content geometry
  simple primitives/convex, complex triangle mesh, Physics Asset bodies
        ↓
filtering
  collision enabled + object type + per-channel response/profile
        ↓
scene query or simulation
  ray/shape sweep/overlap       rigid-body broadphase/contact/solver
        ↓
result/notification
  FHitResult / overlap set      contact, hit event, sleep/wake, constraint state
        ↓
gameplay contract
  validated interaction/damage/movement under authority
```

Keep these layers separate. A correct collision profile can still have bad geometry; a correct scene query can be interpreted incorrectly; a physically plausible local result can still violate server authority.

## 2. Query, collision and simulation

`UPrimitiveComponent` collision can participate in:

- **query:** line/shape traces, sweeps and overlaps;
- **physics simulation:** rigid-body contacts and solver response;
- **both** or neither, depending on collision-enabled mode.

A query answers a geometric/filtering question at a moment or along a path. Simulation advances bodies through time and solves contacts/constraints. A component can block a trace without simulating; a kinematic/non-simulating wall can collide with a simulated body; an overlap volume can generate events without blocking.

Do not use a simulated rigid body merely to detect proximity, and do not expect a trace result to apply impulse automatically.

## 3. Collision filtering

Each collidable object has an **Object Type** and response (`Block`, `Overlap`, `Ignore`) to object/trace channels. Collision profiles package Collision Enabled, Object Type and responses into named project policy. [SRC-PHYS-001]

### Channel versus object query

- **Trace by channel:** “What blocks/overlaps this semantic query?” Each target's response to that trace channel decides.
- **Query for object types:** “Which objects belong to these object categories?” Useful for collecting Pawns/PhysicsBodies/etc, with query filters.

Use semantic trace channels (`WeaponVisibility`, `Interaction`, `Camera`) rather than reusing one because it currently works. Use object types for intrinsic category. Centralise profiles in project settings/config and validate assets/components against them.

### Bilateral simulation response

For object–object interaction, both sides' configured responses matter. Epic's collision overview documents that mismatched Block/Overlap/Ignore resolves according to the pair and that overlap events require appropriate generation settings. [SRC-PHYS-001]

### Collision Enabled matrix

Interview answer should distinguish at least:

- no collision;
- query-only;
- physics-only;
- query and physics.

Exact enum labels/API can differ, but the architectural question is which pipeline needs the shape. Physics-only debris may not be targetable by gameplay traces; query-only trigger should not enter rigid-body solving.

### Profile discipline

Avoid dozens of per-instance response overrides. Define named profiles, document their matrix and change them transactionally. A response change can affect AI visibility, weapons, camera, movement and overlaps simultaneously; treat it like schema/config change with automated tests.

## 4. Block, overlap, hit event and overlap event

**Blocking response** prevents penetration through movement/simulation according to the active path. It does not automatically mean an Event Hit callback is generated; simulation hit notifications require the relevant setting. **Overlap response** allows passage but can maintain overlap state/events when generation is enabled. Ignore produces neither interaction for that pair/query. [SRC-PHYS-001]

Event details depend on whether movement was swept, body simulated, teleport used, and which component requested notifications. Do not write gameplay that assumes every block yields exactly one hit event.

### Event-stream cautions

- resting/sliding contacts can produce repeated notifications;
- high-speed/substepped motion can produce multiple callbacks in one game frame;
- Begin and End Overlap may occur in the same frame;
- destroying/disabling/moving components changes overlap lifetime;
- initial overlaps differ from impact along a sweep;
- both a hit and overlap route can duplicate gameplay if enabled together;
- server and clients may observe different local cosmetic contacts.

For durable gameplay, maintain state/transaction IDs and deduplicate rather than counting raw callbacks blindly.

## 5. Collision geometry: simple versus complex

Simple collision uses primitives/convex hulls; complex collision uses render triangle geometry. Project Default/Simple-and-Complex and “use one as the other” modes control which representation answers simple/complex requests. Maintained docs note that `UseComplexAsSimple` has simulation restrictions; confirm the target asset/branch. [SRC-PHYS-003]

### Simple collision

Strengths: cheaper broad/narrowphase, robust simulation, intentional gameplay silhouette. Use boxes/spheres/capsules/convex decomposition. Costs: approximation, authoring and possible gaps/steps.

### Complex collision

Strengths: detailed scene-query hit/face/material data. Costs: memory/query complexity, thin/degenerate triangles, unstable snagging and simulation limitations. Prefer for precise static-world traces where product needs it, not as a universal fix for poor simple hulls.

### Collision authoring questions

- Is this for movement, rigid simulation, weapon trace, camera, nav or interaction?
- What maximum speed/size and contact fidelity?
- Does the shape need to move/simulate?
- Is per-face physical material/UV/face index required?
- Which LOD/asset variants share collision?
- How is collision cooked on each platform?

Visualise collision in the Static Mesh/Physics Asset editor and runtime; never infer it from the render mesh.

## 6. Scene queries: line, sweep and overlap

### Line trace

Tests a line segment from start to end. Good for visibility, hitscan and probing a surface. It has no volume and can miss thin/off-axis targets that a real character/projectile would contact.

### Shape sweep

Moves a sphere/capsule/box/convex shape from start to end and returns contacts along the path. Use the shape matching the gameplay volume: capsule for character clearance, sphere for forgiving melee/interaction, box for oriented region. A “sphere trace” is a swept sphere, not merely a sphere overlap. [SRC-PHYS-002]

### Overlap query

Tests which shapes overlap a volume at one pose; it does not provide time of impact along movement. Use for spawn validation, area effects and occupancy.

### Single versus multi

Epic's trace overview distinguishes:

- channel multi-trace: overlaps up to and including first blocking hit;
- object-type multi-query: returns matching object types according to query support.

Do not assume result order for every query/API without checking contract; explicitly sort by distance/time if product depends on it. [SRC-PHYS-002]

### Query parameters

Define deliberately:

- channel or object types;
- simple versus complex;
- ignored Actors/Components/self;
- return physical material/face index where needed;
- initial overlap policy;
- trace tag/owner for diagnostics;
- single/multi and blocking/overlap expectation;
- shape orientation and units;
- synchronous/asynchronous/batched path when relevant.

## 7. Reading `FHitResult`

Key distinctions from Epic's trace overview: [SRC-PHYS-002]

- `bBlockingHit`: filtering resolved a blocking result;
- `bStartPenetrating`/initial overlap: query began overlapping;
- `Time`: normalised along sweep segment, commonly `[0,1]`;
- `Distance`: world distance from trace start;
- `Location`: safe swept-shape location at contact;
- `ImpactPoint`: contact point on hit geometry;
- `Normal`: normal related to swept shape contact;
- `ImpactNormal`: hit surface normal;
- Actor versus Component: Component is the actual primitive hit;
- bone/item/face/material: only valid under relevant geometry/query settings.

Do not use `Hit.Actor` alone when one Actor has shield/body/weak-point Components. Validate weak references/lifetime before deferred use.

### Initial penetration

If a sweep starts inside geometry, distance/time may be zero and depenetration normal/depth semantics matter. Spawn/teleport code should run an overlap/adjust policy rather than pretending this is a normal time-of-impact collision.

## 8. Movement sweeps versus rigid-body simulation

`SetActorLocation(..., bSweep=true)`/movement components perform kinematic scene queries and move to a safe position according to their logic. A simulated rigid body is advanced by the physics solver. Teleporting a simulated body or setting transform every frame fights simulation, discards physical continuity and can create tunnelling/energy.

### CharacterMovement

`ACharacter` normally uses an upright capsule and `UCharacterMovementComponent`, not full rigid-body locomotion. CharacterMovement performs sweeps, floor tests, step/slide, movement modes and network prediction. The Skeletal Mesh may animate or ragdoll separately.

Choose CharacterMovement when responsive authored character control, walkable-floor semantics and established network prediction matter. Choose a physics Pawn/rigid body when force/contact-driven motion is the actual mechanic and the team accepts custom control/network complexity.

### One transform owner

At any moment define the authoritative writer:

- CharacterMovement capsule;
- root motion through CharacterMovement integration;
- Chaos rigid body;
- server replicated correction;
- cinematic/teleport transition.

Multiple writers produce jitter and non-reproducible contacts.

## 9. Rigid-body foundations

A rigid body has transform, linear/angular velocity, mass and inertia, collision shapes, damping, gravity/sleep/CCD and material/contact properties. Chaos integrates forces and solves collision/constraint impulses over physics steps. [SRC-PHYS-004]

### Force, impulse and direct velocity

- **Force** acts over time; use for continuous thrust and apply consistently with timestep/physics API.
- **Impulse** changes momentum immediately; useful for hits/explosions/jumps.
- **Force/impulse at location** also produces torque according to lever arm.
- **Direct velocity set** imposes state and can erase solver/contact response; reserve for explicit control/initialisation/correction.

Know whether an API scales by mass (`Accel Change`/`Vel Change`-style options) and whether call occurs once, per game frame or per physics step. A force repeatedly applied in Tick and also maintained across substeps can be mis-scaled if custom logic misunderstands the API.

### Mass and inertia

Mass controls linear response; inertia tensor controls angular response and depends on shape/mass distribution. Two equal-mass bodies can rotate very differently. Centre of mass and inertia matter for vehicles, doors and held objects. Scaling collision or overriding mass can produce implausible inertia; visualise centre of mass and test off-centre impulses.

### Damping, friction and restitution

Damping is not physical surface friction; it attenuates velocity. Friction and restitution arise at contact and combine according to physical-material policies. High restitution plus unstable timestep/penetration can inject visible energy. Do not tune solver defects by adding extreme damping everywhere.

### Sleep/wake

Sleeping removes quiet bodies from active simulation. A body that ignores force may be asleep, kinematic, collision-disabled, welded, constrained or called on the wrong component—not necessarily a Chaos bug. Define wake-on-interaction and avoid waking thousands of props through broad events.

## 10. Continuous collision detection and tunnelling

Discrete simulation tests positions over steps; a fast/small object can cross thin geometry between them. Options:

- scene-query sweep from previous to new position;
- CCD on selected simulated bodies;
- smaller physics timestep/substeps;
- thicker/simpler collision;
- analytical projectile/hitscan where appropriate.

CCD costs more and does not fix wrong filtering/geometry. Enable selectively and test worst speed, frame hitch and target thickness.

## 11. Physics timestep, substepping and async

Physics prefers sufficiently small, stable timesteps. Substepping divides a long game-frame delta into multiple physics steps, improving ragdoll/constraint stability at CPU/bookkeeping cost. Epic documents maximum substep delta/count and delayed callback queues that can yield multiple callbacks in one game frame. [SRC-PHYS-005]

### What substepping does not guarantee

- perfect determinism across platforms/builds;
- identical gameplay callback count;
- stability with impossible constraints/extreme mass ratios;
- no tunnelling without suitable CCD/geometry;
- that game-thread Tick is a physics-substep callback;
- free quality.

Async physics can run solver frames distinct from game-thread frames. Cross-thread data must use supported physics callbacks/handles/snapshots; do not read/write body internals casually. Chaos Visual Debugger exposes separate game and solver timelines in maintained tooling. [SRC-PHYS-009]

## 12. Constraints

A Physics Constraint joins bodies/world with linear/angular locked, limited or free degrees; motors/drives can target position/velocity and break thresholds can trigger separation. [SRC-PHYS-006]

### Constraint design checklist

- body/component names and local frames/axes;
- which body is parent/reference;
- free/limited/locked degrees and angular representation;
- limits, softness, restitution/contact distance;
- drive target/strength/damping/max force;
- collision enabled/disabled between constrained bodies;
- mass/inertia ratio and solver iterations;
- break force/torque and authoritative event;
- projection/stabilisation policy;
- replication and rebuild/lifetime.

Common hinge bug: constraint frame axis is wrong, so “swing” and “twist” settings act on unexpected motion. Draw both local frames before tuning strengths.

### Stability

Extreme mass ratios, tiny bodies, long chains, contradictory hard limits, very stiff drives and large timesteps cause jitter/explosion. Fix geometry/frames/mass/timestep and simplify constraints before increasing solver settings blindly.

## 13. Physical Materials and surface types

Physical Materials configure contact properties such as friction/restitution and project Surface Type classification. They may be assigned through mesh/material/Physics Asset paths; the returned material depends on simple/complex query and asset setup. [SRC-PHYS-007]

Separate:

- **physics response:** friction/restitution/density-like properties;
- **gameplay/presentation classification:** surface type for footsteps, impacts or damage response.

Do not make surface type the sole authority for gameplay material durability without a content/data contract. Request physical material in queries only when needed; per-face complex data has cost/setup requirements.

## 14. Physics Assets and ragdoll

A Physics Asset defines bodies and constraints for one Skeletal Mesh and supports collision/ragdoll/physical animation. [SRC-PHYS-008]

### Asset authoring

- body coverage follows gameplay/collision needs, not one body per bone;
- shapes avoid initial interpenetration;
- adjacent body collision disabled where appropriate;
- mass distribution/constraint frames/limits reflect anatomy;
- root/pelvis body and blend hierarchy deliberate;
- profiles capture reusable ragdoll/physical-animation settings;
- skeletal LOD/bone availability considered.

### Animated to ragdoll transition

1. authoritative gameplay enters ragdoll/death state;
2. capture current pose/velocities where supported;
3. disable or reconfigure CharacterMovement/capsule collision deliberately;
4. enable body simulation and collision profile;
5. transfer/seed momentum without double impulse;
6. replicate/reconstruct required presentation;
7. manage camera, attachments and recovery/despawn.

### Recovery

Choose a stable reference body (often pelvis), detect rest/ground safely, select get-up orientation/animation, align capsule to a collision-safe location, stop/blend physics and restore movement in one ordered transition. Never snap capsule into obstruction or let capsule and ragdoll both push each other.

### Partial physical animation

Blending simulated bodies with animation can produce hit reactions or active ragdolls. Define which bones simulate, blend weight, drive strength, collision and network audience. It adds solver and pose cost and is specialist/version-sensitive.

## 15. Network authority and physics replication

The server normally owns authoritative physics outcomes. Replicated movement for a simulated root body can smooth/correct clients, but local contact differences and latency make arbitrary multiplayer physics difficult.

### Durable state versus physical presentation

Replicate pickup ownership, broken/intact state, score/damage and important final transform/state. Do not infer durable gameplay solely from transient local collision. Cosmetic debris can be client-local if it cannot affect gameplay.

### Client interaction request

For throwing/pushing/grabbing:

1. client sends intent/target/input, not final transform/velocity;
2. server validates ownership, distance, line of sight, rate and object state;
3. server applies force/constraint/state;
4. clients present predicted or replicated motion;
5. correction/rollback policy handles divergence.

### Modern modes awareness

Maintained Epic docs describe Default, Predictive Interpolation and Resimulation physics-replication modes plus a low-level Network Physics Component/history/input path. Resimulation caches state and reruns physics after divergence, consuming CPU/memory; this is **highly version-sensitive** and must not be assumed available/stable identically in UE5.3–UE5.6. [SRC-PHYS-010]

At broad interview depth: explain server authority, correction versus prediction, interaction divergence and why speed/latency/contacts are hard. Specialist depth requires target-source implementation and CVD multi-source/resimulation evidence.

## 16. Debugging collision and traces

### Trace returns nothing

1. Draw start/end/shape/orientation and confirm World/context.
2. Check Collision Enabled includes query.
3. Check query channel/object types and target response/object type.
4. Check simple/complex geometry actually exists and target query asks for the right kind.
5. Check ignored Actors/self/components.
6. Check start inside/zero length/shape scale/orientation.
7. Check target component, not only Actor root.
8. Add trace tag/owner and inspect scene query in CVD if available.

### Unexpected object blocks

Inspect the exact hit Component's profile, object type and response to the query channel—not the Actor class or intended profile. Audit runtime overrides and child Components. Record `FHitResult` component/item/bone/face and collision shape.

### Overlap event missing

Check both components' collision enabled, Object Type/response pair, Generate Overlap Events, registration/initial overlap/update movement and whether teleport/profile change recomputed overlaps. Bind before the expected event and query current overlaps to establish initial state.

### Duplicate hits

Determine whether results are multiple Components/bodies, multi-cell/shape contacts, substep callbacks, resting contacts, client+server cosmetic paths or both hit/overlap handlers. Deduplicate by action/query plus component/body and time/window as the product requires.

## 17. Debugging unstable simulation

1. Record with Chaos Visual Debugger/Insights before tuning.
2. Inspect body state, shape, mass/inertia/centre, velocity and sleep.
3. Inspect contact pair normals/penetration/materials.
4. Inspect constraint local frames, errors, drives/limits and break state.
5. Reduce to two bodies/one constraint/simple collision.
6. Test fixed controlled timestep, then hitches/substeps.
7. Remove extreme mass ratios/stiffness/restitution.
8. Find external transform/velocity writers.
9. Compare server/client solver frames for network divergence.

CVD can record particles, collision geometry/channels, contacts, joints and scene queries, and supports game/solver timelines in maintained versions. UE5.6 release notes are cited by the maintained page, but exact 5.3–5.6 availability requires verification. [SRC-PHYS-009]

## 18. Profiling and scalability

Measure independently:

- active/awake/sleeping body counts;
- broadphase candidate and narrowphase/contact count;
- solver/island/constraint time and iterations;
- scene query count/type/shape/candidates/time;
- overlap/hit callback volume and downstream work;
- substeps and max physics delta/clamping;
- Physics Asset body/constraint count;
- async/game sync/wait time;
- network state/history/bandwidth/corrections;
- physical VFX/audio cascades;
- memory for collision geometry/history.

### Optimisation order

- remove unnecessary collision/query/simulation participation;
- simplify shapes and profiles; avoid complex queries unless needed;
- batch/cache/static-query results only with valid lifetime/invalidation;
- reduce query frequency/extent and use semantic filters;
- let quiet bodies sleep and avoid broad wake-ups;
- simplify ragdoll bodies/constraints and significance/LOD;
- apply CCD/substeps only to workloads that need them;
- reduce callback spam and move expensive consequences out of per-contact paths;
- bound replicated physics population/rate/history by relevance.

Profile the downstream response. A cheap overlap that spawns Blueprint/VFX/audio every frame can cost more than the query.

## 19. What a three-year engineer should know

Expected D3–D4:

- Object Type/channel/response/profile matrix;
- Query/Physics enabled distinction;
- simple versus complex collision;
- line/sweep/overlap and single/multi semantics;
- `FHitResult` locations/normals/component/initial overlap;
- CharacterMovement sweep versus rigid simulation;
- force versus impulse, mass/inertia/damping/sleep/CCD awareness;
- constraint frames/limits/drives and Physical Materials;
- Physics Asset/ragdoll transition;
- server authority and default replication/correction awareness;
- collision visualisation/CVD/trace-led diagnosis and profiling.

Specialist D4–D5:

- Chaos solver/contact/island internals;
- async physics callback/thread interfaces;
- custom shapes/queries and low-level handles;
- Network Physics Component, Predictive Interpolation/Resimulation;
- vehicle/destruction/cloth/fluid systems;
- physical animation/active ragdoll;
- deterministic/fixed-step architecture and platform solver tuning.

## 20. Common misconceptions

| Misconception | Better model |
|---|---|
| “Block means Hit event fires.” | Blocking and notification generation are separate settings/paths. |
| “Overlap is a non-blocking collision.” | It is a filtering/query state whose events require generation and bilateral compatibility. |
| “Sphere trace checks a sphere.” | It sweeps a sphere from start to end; overlap checks one pose. |
| “Trace by channel finds that object type.” | Channel asks targets how they respond; object query selects categories. |
| “Complex collision is more accurate, so better.” | It is detailed but costlier/restricted; gameplay often needs robust simple shapes. |
| “SetActorLocation Sweep is physics simulation.” | It is kinematic movement/query; solver-driven bodies follow physics steps. |
| “Force and impulse differ only by magnitude.” | Force integrates over time; impulse changes momentum at an instant. |
| “Substepping makes physics deterministic.” | It improves temporal resolution/stability at cost; determinism has broader constraints. |
| “Ragdoll means enable simulation on the mesh.” | It is an ordered ownership/collision/movement/network transition using a Physics Asset. |
| “Replicate movement solves multiplayer physics.” | Contacts diverge under latency; authority, prediction/correction and relevance still need design. |

## 21. Strong interview answers

### Debug a failed trace

> I draw the exact start/end/shape/orientation, then inspect the target PrimitiveComponent—not just Actor—for query-enabled collision, object type, response to the trace channel and actual simple/complex geometry. I verify ignored actors and initial overlap. I log the full hit component/item/bone/face and use a trace tag or CVD scene-query recording to see candidates/filtering. I do not switch to Visibility or complex collision blindly.

### CharacterMovement versus physics

> CharacterMovement is kinematic/capsule-based gameplay movement with floor/step/slide modes and mature network prediction; it is the default for responsive humanoids. Rigid-body movement is force/contact driven and needs control, timestep and network-physics design. I choose from the mechanic, define one transform owner, and transition explicitly for root motion or ragdoll rather than running capsule movement and body simulation together.

### Physics performance

> I split scene queries, active bodies/broadphase contacts, solver/constraints, callbacks, substeps, Physics Assets and network history/corrections. I capture CVD/Insights with body/contact/query counts, then remove unnecessary participation, simplify geometry/filtering and reduce query/callback frequency. I reserve CCD/substeps and high-detail ragdolls for significant objects and validate quality under hitches, clustering and multiplayer latency.

## 22. Hands-on verification project

Extend Project 1/3 with a Physics and Collision Lab:

1. create a documented channel/object/profile matrix for Pawn, World, Interactable, Projectile, Melee and Trigger;
2. author simple/complex collision variants and automated expected-response tests;
3. implement line/sphere/capsule sweeps and overlaps with full hit logging/debug draw;
4. inject missing query enabled, wrong response, no overlap generation, ignored self, initial penetration and duplicate Components;
5. compare CharacterMovement push, kinematic sweep and simulated-body impulse;
6. build a mass/inertia/off-centre impulse/damping/sleep experiment;
7. create a constrained door and unstable mass/stiffness/substep failure matrix;
8. implement Physical Material surface footsteps/impact without making cosmetics authoritative;
9. transition animated character to ragdoll and collision-safe recovery;
10. replicate a thrown/pushed object on dedicated server under lag and compare default correction with target-supported modern modes only after source verification;
11. record CVD scene queries/contacts/joints and scale 10/100/1000 props with sleep/shape/query changes.

Deliver the collision matrix, query acceptance tests, ragdoll state diagram, network authority flow and evidence-based performance report.

## 23. Source map

- collision/filtering/query/hit geometry: [SRC-PHYS-001] [SRC-PHYS-002] [SRC-PHYS-003]
- Chaos/rigid bodies/substeps: [SRC-PHYS-004] [SRC-PHYS-005]
- constraints/materials/Physics Assets: [SRC-PHYS-006] [SRC-PHYS-007] [SRC-PHYS-008]
- debugging and network physics: [SRC-PHYS-009] [SRC-PHYS-010]
