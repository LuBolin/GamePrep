# ECS and MassEntity for Unreal Interviews

See also: [[ue_ai_custom_senses_massai]], [[ue_smart_objects_statetree]], [[game_programming_patterns]], [[ue_hands_on_projects]].

MassEntity is Unreal Engine 5's data-oriented entity framework. A broad three-year engineer should understand why it exists, its data/execution model, when an Actor is better, and how to profile/debug a hybrid system. Exact API fluency is specialist and highly branch-sensitive.

Version target: **UE5.3–UE5.6**. MassEntity/MassGameplay features are plugin- and version-sensitive; maintained Epic pages often display UE5.7 APIs.

## 1. The problem Mass is trying to solve

Actors and UActorComponents are excellent identity-rich world objects: reflection, lifecycle, replication, collision, rendering, Blueprint authoring and independent polymorphic behaviour. Those conveniences carry per-object memory, registration, indirection, update scheduling and cache costs.

Mass targets workloads with many entities that share a small number of data shapes and can be processed in batches: crowds, ambient agents, traffic, projectiles, foliage-like simulation, background world populations or large sets of lightweight gameplay records.

The core optimisation is not “remove virtual functions”. It is:

- group entities by **composition**;
- store the same fragment type contiguously in **chunks**;
- query matching archetypes once;
- run processors over fragment arrays in batches;
- separate simulation fidelity from visual Actor representation;
- reduce work/frequency by significance/LOD.

Data-oriented design starts from access patterns and budgets, not from translating every OOP class into fragments.

## 2. Actor architecture versus Mass

| Dimension | Actor/Component | MassEntity |
|---|---|---|
| Identity | UObject/Actor identity, references, reflection | lightweight entity handle in an entity manager |
| Composition | Components/subclasses/UObjects | fragment/tag composition |
| Logic | methods, Components, Tick/events | processors/query batches |
| Memory/access | object-oriented, potentially scattered | archetype/chunk-oriented contiguous fragment columns |
| World integration | native transform/render/physics/replication/Blueprint lifecycle | supplied through MassGameplay integrations or hybrid bridge |
| Per-instance variation | easy independent behaviour | best when variations map to bounded compositions/data |
| Structural change | add/remove Components can be heavy | add/remove fragments moves entity between archetypes; defer/batch |
| Best fit | dozens/hundreds of rich interactive objects | large populations with batchable behaviour and fidelity tiers |

Use Actors for a boss, player avatar, interactable quest object or unique physics object unless measured scale/processing says otherwise. Use Mass for thousands of agents whose movement/state can be expressed as data passes. Hybrid systems commonly promote a nearby/selected entity to an Actor representation while Mass remains simulation identity. [SRC-MASS-001] [SRC-MASS-002]

## 3. Entity, handle, fragment, and tag

### Entity

A Mass entity is an identity/handle associated with a fragment composition in an `FMassEntityManager` hosted per World. It is not a UObject and does not contain methods. Handles can become stale after destruction and possible index reuse; validate them with the owning manager and never treat them as persistent save/network IDs.

### Fragment

A fragment is one atomic data aspect used by processors—for example transform, velocity, health or target. Fragments should be small/cohesive enough to support useful query/access patterns, but not split every scalar into a separate type.

```cpp
USTRUCT()
struct FGamePrepVelocityFragment : public FMassFragment
{
    GENERATED_BODY()
    FVector Value = FVector::ZeroVector;
};
```

This is conceptual UE5 code; includes/base requirements/API details must be verified in the target branch.

### Tag

A Mass tag has no per-entity payload; presence/absence is state used in composition/query matching. Tags fit categorical conditions such as `NeedsTarget` or `Dead`, but frequently toggling tags causes structural/archetype changes. A Boolean field may be cheaper when the state changes very often and selective query grouping brings little benefit. [SRC-MASS-001]

### Entity handle is not gameplay identity

Keep a project-stable ID when an entity must survive save/load, replication remapping, World recreation or Actor promotion. Maintain an explicit mapping and generation/validity policy rather than serialising `FMassEntityHandle` as durable identity.

## 4. Archetypes and chunks

An **archetype** represents one exact composition of fragment/tag/shared/chunk-fragment types. All entities in that archetype have that data shape. Entities are stored in memory chunks; each fragment type forms a contiguous stream/column for entities in the chunk. [SRC-MASS-001]

If 20,000 entities share Transform + Velocity + AgentRadius, a movement processor can read/write arrays sequentially instead of chasing 20,000 object pointers and virtual calls.

### Archetype explosion

Every independent optional fragment/tag combination can produce more archetypes. Excessive composition variants reduce batch sizes, expand query matching/scheduling and increase structural movement. Avoid encoding every transient Boolean as a tag without measuring.

### Chunk capacity and fragment size

Larger/hot fragments reduce entities per chunk and pull unused data through caches. Split by access frequency: hot movement state separate from cold biography/presentation metadata. But dozens of microscopic fragments create query and authoring complexity. Design from processor access sets.

## 5. Shared and chunk fragments

### Shared fragment

A shared fragment stores data shared by a group/archetype partition rather than repeated per entity—configuration such as movement parameters or model type. Entities with different shared values may be separated into different groups/archetypes, so high-cardinality “shared” values can fragment batches.

Const/shared mutation semantics and exact APIs are version-sensitive. Treat shared configuration as immutable where possible and create/reuse canonical shared values.

### Chunk fragment

A chunk fragment stores one value per chunk, used for managerial/batch metadata such as LOD grouping. It is not per-entity state. [SRC-MASS-001]

Use the correct cardinality:

- per entity: position, velocity, health;
- per group/shared value: movement configuration, representation descriptor;
- per chunk: batch visibility/LOD/status metadata;
- global/World: subsystem/service data declared as query dependency.

## 6. Queries and access declarations

`FMassEntityQuery` declares required fragment/tag/subsystem composition, presence (`All`, `Any`, `None`, `Optional`) and access (`ReadOnly`, `ReadWrite`). It caches matching archetypes and passes chunk views to processor execution. [SRC-MASS-003]

Access declarations are not documentation fluff. They support scheduling conflict analysis, parallelism and validation. Declaring ReadWrite when only reading reduces possible overlap; writing through undeclared access breaks the model.

Conceptual processor query:

```cpp
void UGamePrepMoveProcessor::ConfigureQueries()
{
    EntityQuery.AddRequirement<FTransformFragment>(EMassFragmentAccess::ReadWrite);
    EntityQuery.AddRequirement<FGamePrepVelocityFragment>(EMassFragmentAccess::ReadOnly);
    EntityQuery.AddTagRequirement<FGamePrepMovingTag>(EMassFragmentPresence::All);
}
```

Exact query construction/registration changed across UE5 versions. In maintained APIs, queries may be constructed/registered with their owning processor; verify UE5.6 patterns in headers/samples. [SRC-MASS-003]

### Chunk iteration

Inside `ForEachEntityChunk`, retrieve fragment views once and loop contiguous indices. Avoid per-entity UObject lookup, allocations, global map searches, logging or virtual interface calls in the hot loop.

### Optional fragments

Optional requirements can simplify processors but may create branches and less clear contracts. Separate processors/queries can be faster/clearer for materially different work. Measure composition count and branch behaviour.

## 7. Processors, phases, and dependency order

A processor is a typically stateless UObject that configures queries and executes logic over matching entities. Processor metadata/configuration controls execution phase/group, automatic registration, threading requirements and ordering relative to other processors. [SRC-MASS-004]

Example movement pipeline:

1. acquire/update target;
2. compute desired velocity;
3. apply avoidance force;
4. integrate movement/transform;
5. update representation.

If ordering is implicit or cyclic, behaviour can flicker or consume previous-frame data unexpectedly. Express prerequisites/groups through supported processor dependency mechanisms and inspect the actual pipeline in the target debugger/log.

### Processor statelessness

Per-entity mutable state belongs in fragments. Processor members can hold query/config/cache data with clear World/lifecycle/thread contracts, but not arbitrary entity state indexed implicitly by iteration order.

### Parallel execution

Mass can schedule processors/chunks based on declared access, but “Mass means multithreaded” is too broad. UObject/world mutation, subsystems, processor flags, query requirements and target version constrain threading. Do not touch arbitrary Actors/UObjects from worker execution. Stage data and perform game-thread bridges in explicit processors where required.

## 8. Structural changes and deferred commands

Adding/removing a fragment or tag changes composition and moves the entity to another archetype. Doing this while iterating can invalidate assumptions/storage. Mass command buffers defer structural operations and flush them in a safe batch. [SRC-MASS-001]

Typical deferred operations:

- add/remove tag or fragment;
- create/destroy entity;
- set composition/state requiring archetype migration;
- enqueue a later bridge/promotion operation.

### Structural churn

If every entity toggles a tag every frame, Mass spends time moving entities/batching commands rather than processing stable chunks. Alternatives:

- retain a field for high-frequency state and branch/filter cheaply;
- process state transitions less often;
- batch transitions;
- design fewer stable simulation tiers.

Structural change is a tool, not a free event system.

## 9. Traits, templates, configs, and spawners

A Trait configures a capability at entity-template build time: it adds required fragments/tags/shared data and can validate dependencies. `MassEntityConfig`-style assets combine/inherit Traits to produce entity templates; spawners instantiate entities from templates/distributions. [SRC-MASS-001] [SRC-MASS-002]

Traits are authoring/composition recipes, not per-frame behaviour. A movement Trait may add transform/velocity/config and ensure relevant processors exist. Processors remain the logic.

### Trait validation

Validate:

- required fragments/traits present;
- conflicting representation/movement traits absent;
- shared config values valid/canonical;
- target plugins/modules enabled;
- fragment defaults and template identity stable;
- authoring asset does not create uncontrolled hard dependency closure.

## 10. Observers and signals

Mass Observer processors react to composition events such as fragment/tag addition/removal, depending on exact target APIs. They are useful for initialisation/cleanup associated with structural changes, not arbitrary field-value change detection.

Mass Signals provide a lightweight named wake-up indication used by integrations such as Mass StateTree; the maintained overview describes a signal as event-like without payload. Signals do not replace durable fragment state. [SRC-MASS-002]

Avoid broadcasting a UObject-style delegate from every entity. Batchable fragment state and processors should carry the main simulation; bridge only important aggregate/presentation events out to Actor/UI systems.

## 11. Representation and simulation are separate

MassGameplay representation can select, by LOD, among high-resolution Actor, low-resolution Actor, instanced static mesh (ISM), or no representation, with subsystem-managed transitions/pooling. [SRC-MASS-002] [SRC-MASS-005]

This is the central hybrid insight:

- entity simulation identity can persist;
- nearby/important entity obtains an Actor for rich interaction/animation/collision;
- medium/far entity uses ISM/cheap representation;
- off-screen/distant entity has no visual representation and lower simulation frequency.

### Promotion/demotion contract

Define one owner of truth for every field during transitions:

- stable entity ↔ Actor mapping;
- position/velocity/state transfer direction;
- event/interaction routing;
- Actor pool reset and external reference invalidation;
- network authority;
- animation/collision readiness;
- transition hysteresis to prevent thrash.

Do not let both Actor Tick and Mass processor authoritatively integrate the same transform.

## 12. LOD: visual, simulation, and replication

MassGameplay distinguishes representation/visual LOD, simulation LOD and replication LOD. Visual LOD chooses presentation; simulation LOD changes update frequency/fidelity; replication LOD can vary per viewer. [SRC-MASS-002]

Examples:

- High: Actor, full avoidance/decision at high frequency;
- Medium: ISM, simplified steering every few frames;
- Low: ISM or no representation, coarse path/aggregate update;
- Off: no representation, sparse schedule or analytical state.

LOD needs hysteresis, maximum counts, transition budgets and gameplay invariants. A distant agent must not become able to walk through locked progression or duplicate rewards merely because simulation simplified.

## 13. Hybrid Actor–Mass architecture

### Good bridge

Mass owns large-scale population/simulation data. A representation subsystem owns Actor acquisition/release. Actor components present/handle rich nearby interaction but communicate through explicit fragments/commands. Stable game identity maps both.

### Bad bridge

Every Mass entity stores a UObject pointer and every processor calls Actor methods per entity. This regains object indirection/game-thread coupling and loses batching. If every entity requires full Actor collision/animation/replication continuously, Mass may be the wrong representation.

### Interaction example

Player targets an Actor representation; bridge resolves stable entity handle/ID; server validates request against authoritative Mass state; deferred command adds `Selected/Interacting` state; processors update simulation; committed state drives Actor/UI. If the Actor demotes, the entity remains valid and external Actor references are released.

## 14. Replication awareness

Mass replication is a specialised plugin system using viewer relevancy/LOD and custom fragment data support; its status and APIs have changed across UE5. The maintained MassGameplay overview preserves an old “experimental in 5.1” statement, which does not establish UE5.6 production status. Inspect target source/docs. [SRC-MASS-002]

At broad interview depth:

- server remains authoritative;
- not every local fragment should replicate;
- replication representation may be separate from simulation fragments;
- stable network identity/remapping matters more than local entity handle;
- per-viewer relevance/update frequency is essential at crowd scale;
- Actor replication and Mass replication must not both authoritatively duplicate the same entity state.

## 15. When not to use Mass

Do not choose Mass merely because:

- ECS is fashionable;
- there may someday be 500 entities;
- Tick looks suspicious without profiling;
- inheritance is disliked;
- one system has Components;
- you want multithreading automatically.

Avoid or bound Mass when entities are few, unique, UObject/Blueprint-heavy, physics-rich, independently replicated, constantly changing composition, require arbitrary cross-object interactions, or the team lacks tools/ownership for a plugin-sensitive architecture.

A manager updating structs/ISM, conventional Components with significance tiers, Niagara, or a specialised batched simulation may be simpler.

## 16. Debugging workflow

### Entity missing from processor

1. Verify correct World/entity manager and handle validity.
2. Inspect entity composition/archetype.
3. Compare query `All/Any/None/Optional` requirements exactly.
4. Confirm query is registered/configured for the target API.
5. Confirm processor active, auto-registered, phase/group and target execution mode.
6. Confirm structural command has flushed.
7. Inspect shared/chunk requirements and tags.
8. Use Mass debugger/console/source capabilities available in the target branch. [SRC-MASS-006] [SRC-MASS-007]

### Processor order bug

Log processor phase/group/prerequisites and marker fragments/counters. Determine whether it reads previous-frame data or races another writer. Correct dependency declarations/access modes rather than relying on class name or registration order.

### Stale handle

Validate with the manager before access; do not cache handles across uncertain structural/destruction lifetimes without stable ID/generation policy. Never store fragment views/pointers beyond the chunk execution callback.

### Representation bug

Separate entity state from visual representation state. Inspect LOD/significance, representation type, Actor mapping/pool lifecycle, ISM transform update, hysteresis and transition budget. “Entity exists” does not mean “Actor exists”.

## 17. Profiling workflow

Measure four layers independently:

1. **simulation:** processor total/self time, entities/chunks matched, frequency;
2. **structure:** archetype count, chunk occupancy, deferred command volume, migrations;
3. **bridges:** UObject/Actor lookups, game-thread processors, promotions/demotions;
4. **representation:** Actor count/Tick/animation/collision, ISM update/render cost, LOD transitions.

Useful derived metrics:

- nanoseconds/time per entity per processor;
- entities and chunks per query;
- average chunk occupancy;
- archetypes and shared-fragment variants;
- structural commands/migrations per frame;
- high/medium/low/off population;
- Actor/ISM/no-representation counts;
- promotion/demotion rate;
- memory per entity and total;
- Game/Draw/GPU and network bandwidth.

Compare against an Actor baseline with the **same behaviour and visual fidelity**. A Mass simulation with no skeletal animation/physics cannot fairly “beat” fully featured Characters unless the difference is the intentional product design.

## 18. Common misconceptions

| Misconception | Better model |
|---|---|
| “Mass is Unreal's replacement for Actors.” | It is a data-oriented framework for suitable populations; Actors remain rich world objects/representations. |
| “Fragment is an ECS Component and can contain behaviour.” | Mass fragments are data; processors supply batch logic. |
| “Tags are free booleans.” | Tag changes alter composition/archetype and can create structural churn. |
| “Mass automatically multithreads everything.” | Scheduling depends on declared access, processor/query requirements and World/UObject constraints. |
| “Entity handle is a persistent ID.” | It is manager-local lifetime identity; use stable domain IDs for persistence/network mapping. |
| “Trait is a runtime Component.” | Trait is an authoring/template composition recipe. |
| “Mass entity has an Actor.” | Representation can be Actor, ISM or none and changes by LOD. |
| “More fragments means better data orientation.” | Fragment boundaries should match access patterns; over-fragmentation/archetype explosion hurts. |
| “Structural changes are events.” | They migrate composition; defer/batch and avoid high-frequency churn. |
| “Mass solves crowd rendering.” | Simulation and representation are separate; profile both. |

## 19. Strong interview answers

### Actor versus Mass

> I would start from population scale and access pattern. Actors are ideal for identity-rich independently behaving world objects with Blueprint, physics, replication and Components. Mass pays when thousands share stable compositions and processors can batch contiguous fragments, with simulation/representation LOD. I would prototype the same workload, measure simulation, structural churn and representation separately, and use a hybrid Actor promotion path for nearby rich agents. I would not convert a boss or a few hundred ordinary Actors without evidence.

### Fragment/archetype/chunk/processor

> A fragment is one data aspect; an entity is a handle plus fragment/tag composition. Entities with identical composition share an archetype and are stored in chunks with contiguous fragment columns. A processor declares an EntityQuery and reads/writes matching chunk arrays in batches. Adding/removing a fragment changes archetype, so structural commands are deferred and should not churn every frame.

### Missing processor execution

> I check the correct World/manager and valid entity, inspect its exact composition, then compare query presence/access requirements. I verify query registration, processor activation/phase/group and that deferred composition commands flushed. After that I use the target Mass debugger/log/source to inspect matched archetypes and processor pipeline. I do not add fragments randomly until it starts matching.

## 20. Hands-on verification

Build Project 5 with equal Actor and Mass crowds. Start with Transform/Velocity/Target/LOD fragments, movement/target processors, Traits/config/spawner and ISM representation. Add an Actor promotion tier, deferred dead/alive tag transition, simulation LOD and debugger counters. Deliberately create archetype explosion, structural churn, stale handle, wrong query and dual transform ownership; diagnose each from evidence.

## 21. Source map

- Concepts/traits/commands: [SRC-MASS-001]
- Gameplay representation/LOD/spawning/signals/replication awareness: [SRC-MASS-002]
- Queries/processors: [SRC-MASS-003] [SRC-MASS-004]
- Representation/debugging: [SRC-MASS-005] [SRC-MASS-006] [SRC-MASS-007]
- Avoidance pipeline example: [SRC-MASS-008]

