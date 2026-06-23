# Custom AI Senses and MassAI Specialist Pass

See also: [[ue_ai_navigation]], [[ue_smart_objects_statetree]], [[ue_massentity_ecs]], [[ue_ai_senses_massai_workbook]], [[ue_hands_on_projects]].

**Priority:** P2 for broad gameplay roles, P1 for AI/gameplay roles  
**Expected depth:** D4 specialist implementation awareness  
**Version scope:** UE5.3-UE5.6 preferred. AI Perception is core AIModule territory; custom sense APIs, StateTree, Smart Objects, MassEntity, ZoneGraph and Mass Crowd are version-, module- and plugin-sensitive.  
**Prerequisites:** [[ue_ai_navigation|AI and navigation]], [[ue_smart_objects_statetree|Smart Objects and StateTree]], [[ue_massentity_ecs|ECS and MassEntity]], gameplay framework, event-driven design, navigation/pathfinding, profiling.

This chapter deepens the two AI gaps left by the first pass:

1. how to design, implement, debug and profile custom AI senses without turning perception into a hidden gameplay state machine;
2. how to reason about MassAI-style crowds where decisions, lane movement, StateTree/Smart Objects, representation and Actor promotion are split across Mass fragments/processors and classic gameplay systems.

## The specialist mental model

Generalist AI answers often stop at Behaviour Tree, Blackboard, EQS and NavMesh. Specialist AI work separates **evidence** from **truth** and **crowd simulation** from **rich gameplay identity**.

```text
World fact or event
  -> sensing source/event
  -> sense-specific batching and filtering
  -> FAIStimulus-style record
  -> listener's perception component
  -> explicit memory projection
  -> decision model
  -> movement/query/interaction request
  -> gameplay authority validates durable outcome
```

For MassAI-style crowds:

```text
Spawner/template
  -> entity fragments/tags/shared data
  -> ZoneGraph lane/route state
  -> movement/avoidance processors
  -> StateTree/Smart Object tasks or signals where needed
  -> simulation LOD
  -> Actor/ISM/no representation LOD
  -> gameplay bridge only for important nearby interactions
```

The important interview distinction: custom senses and MassAI are not "more advanced AI" by themselves. They are tools for modelling input and scale. Durable gameplay rules, authority, animation, combat, inventory, save/load and replication still need explicit architecture. [SRC-AI-004] [SRC-AI-016] [SRC-AI-018] [SRC-MASS-001] [SRC-MASS-002]

## Custom AI senses

### What a custom sense is for

A custom sense is appropriate when the game has a recurring kind of perceptual evidence that should flow through the same listener/stimulus/memory/debug pipeline as sight or hearing, but whose production rules are game-specific.

Good candidates:

| Sense idea | Why custom sense can fit | Alternative that may be simpler |
|---|---|---|
| Scent/trail | Evidence persists in world space and decays over time | Simple investigation points if only one guard uses it |
| Radio chatter | Event source, strength/range/team filtering and memory matter | Gameplay message if no spatial perception is needed |
| Magic aura | Spatial + affiliation + strength rules need consistent perception | Tag query if only nearby player UI uses it |
| Footprint/noise proxy | Many low-level events need batching and stimulus memory | Built-in hearing if sound-like enough |
| Last-known squad sighting | Agent receives indirect evidence with confidence/age | Blackboard broadcast if it is not sensory evidence |

Poor candidates:

- one-off scripted trigger;
- purely authoritative gameplay state such as "door unlocked";
- UI discovery or interact prompt state;
- every combat event that would flood listeners;
- expensive per-listener traces where perception range/affiliation would have filtered cheaply.

### Contract boundaries

| Layer | Owns | Should not own |
|---|---|---|
| Event/source | Emits bounded evidence with location, strength, tag, target/source identity | AI memory policy or final decision |
| Sense | Batches, filters, ages/schedules, emits stimuli | Blackboard writes, attack decisions, ability activation |
| Perception component | Receives stimuli for a listener | Global gameplay truth |
| Memory projection | Turns stimuli into target, last-known, alert, confidence | Raw per-sense implementation details |
| Decision | Chooses investigate/chase/ignore | Low-level sense collection |
| Gameplay authority | Validates damage, rewards, interaction, replication | Trusting a stimulus as a final outcome |

This separation prevents a custom sense from becoming a second hidden Behaviour Tree. [SRC-AI-004] [SRC-AI-019]

### Stimulus semantics

A stimulus is not "the target". It is evidence about a target/source at a moment. Track at least:

- sense type or channel;
- source Actor or stable source ID;
- target Actor if different from source;
- stimulus location and receiver location;
- strength/confidence;
- success/loss state where applicable;
- tag/category;
- server/world time and age/expiry policy;
- source generation if Actors can be destroyed/reused;
- team/affiliation at the time of sensing if relevant.

The existing AI chapter covers the "currently perceived versus known versus last-known" distinction. Custom senses make that distinction more important because evidence can be indirect, delayed or persistent. [SRC-AI-004] [SRC-AI-018]

### Schematic C++ shape

The following is intentionally schematic. Exact method names, constructor signatures, listener iteration helpers, registration functions and module includes must be verified against the target UE5.3-UE5.6 branch. Use it as an interview design pattern, not paste-ready code. [SRC-AI-016] [SRC-AI-017] [SRC-AI-018] [SRC-AI-019]

```cpp
USTRUCT(BlueprintType)
struct FGamePrepScentEvent
{
    GENERATED_BODY()

    UPROPERTY()
    TObjectPtr<AActor> SourceActor = nullptr;

    UPROPERTY()
    FVector Location = FVector::ZeroVector;

    UPROPERTY()
    float Strength = 1.0f;

    UPROPERTY()
    FName Tag = NAME_None;

    UPROPERTY()
    double ServerWorldTime = 0.0;
};

UCLASS(EditInlineNew, DefaultToInstanced)
class UGamePrepScentSenseConfig : public UAISenseConfig
{
    GENERATED_BODY()

public:
    UPROPERTY(EditDefaultsOnly, Category="Sense")
    float MaxRange = 2500.0f;

    UPROPERTY(EditDefaultsOnly, Category="Sense")
    float DecaySeconds = 8.0f;

    UPROPERTY(EditDefaultsOnly, Category="Sense")
    bool bRequireLineOfSmell = false;
};

UCLASS()
class UGamePrepScentSense : public UAISense
{
    GENERATED_BODY()

public:
    void RegisterScentEvent(const FGamePrepScentEvent& Event)
    {
        PendingEvents.Add(Event);
        RequestImmediateUpdate(); // Schematic: verify target API.
    }

protected:
    virtual float Update() override
    {
        // Schematic only:
        // 1. discard expired or invalid events;
        // 2. batch by source/location/tag;
        // 3. iterate listeners in range/team filters;
        // 4. optionally perform bounded traces;
        // 5. report FAIStimulus-like records to listeners.
        PendingEvents.Reset();
        return SuspendNextUpdate; // Verify target constant/return contract.
    }

private:
    TArray<FGamePrepScentEvent> PendingEvents;
};
```

The design point is not the exact API. The design point is that the sense owns collection and bounded filtering, while AIController/Blackboard/StateTree owns interpretation.

### Memory projection example

```cpp
USTRUCT(BlueprintType)
struct FGamePrepKnownTargetMemory
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    TObjectPtr<AActor> Target = nullptr;

    UPROPERTY(BlueprintReadOnly)
    FVector LastEvidenceLocation = FVector::ZeroVector;

    UPROPERTY(BlueprintReadOnly)
    float Confidence = 0.0f;

    UPROPERTY(BlueprintReadOnly)
    double LastEvidenceTime = 0.0;

    UPROPERTY(BlueprintReadOnly)
    FName EvidenceTag = NAME_None;
};
```

Projection policy:

1. sight success can produce high-confidence current target;
2. hearing or scent can update last-known/investigation target;
3. failed sight should lower visual certainty, not erase all evidence;
4. old scent decays confidence rather than creating permanent psychic knowledge;
5. direct damage can override weak indirect evidence;
6. server authority validates combat outcomes separately.

This keeps Blackboard/StateTree keys human-readable: `CurrentTarget`, `LastEvidenceLocation`, `AlertConfidence`, `EvidenceAge`, `InvestigationReason`.

### Edge cases that distinguish specialist answers

| Edge case | Symptom | Strong response |
|---|---|---|
| Source destroyed before listener processes event | callback references invalid Actor | use weak/object validity checks and stable IDs/generation where durable identity matters |
| Stimulus never expires | AI always knows target | explicit Max Age/decay policy and tests for forget/investigate/reacquire |
| Loss stimulus clears all knowledge | guard ignores noise/damage memory | per-sense memory merge policy |
| Per-listener traces inside hot loop | frame spikes with many listeners/events | prefilter by range/team/tag, batch events, cap traces, stagger work |
| Client emits authoritative sensory truth | cheating/desync | server owns durable gameplay perception for AI; clients may predict cosmetics |
| World Partition unload removes sources | stale memories or null refs | source generation/world lifetime policy; decay or mark evidence stale |
| Team changes while stimuli live | old friendly/hostile assumptions | re-evaluate team at decision time or store labelled evidence intentionally |
| Async work touches UObjects off-thread | rare crashes | copy POD query input, return results to game thread, validate objects |
| Blueprint custom sense floods events | VM and allocation spikes | native batching for hot senses, data-only event payloads, counters |

### Debugging workflow: custom sense never fires

1. Confirm the listener has a `UAIPerceptionComponent` and the custom sense config is present.
2. Confirm module/plugin dependencies include AIModule and the custom class is loadable in the build target.
3. Confirm the source event is emitted in the expected World, on the authoritative side if relevant.
4. Log event location, tag, strength, source, team and time before registration.
5. Log sense update count, pending event count, discarded event reasons and listener count.
6. Inspect Perception debug output and Visual Logger snapshots.
7. Reduce to one listener and one source; then scale listeners/events back up.
8. Add an automated scenario: emit one event at known location and assert memory key updates within a bounded time.

### Debugging workflow: AI remembers forever

1. Inspect sense config age/decay and whether zero means "never forget" for the configured path.
2. Log per-sense last update rather than a single target bool.
3. Separate `CurrentTarget` from `LastEvidenceLocation`.
4. Verify failed/lost stimulus handling does not accidentally refresh memory.
5. Check whether another sense keeps updating the same target.
6. Force time advance with no stimuli and assert confidence falls below decision threshold.
7. Confirm save/load or streaming does not rehydrate stale evidence as fresh.

### Profiling workflow

Instrument:

- events emitted per second by tag/source;
- events accepted, merged, discarded and expired;
- listeners considered after range/team filters;
- traces/queries performed;
- stimuli reported;
- perception callbacks invoked;
- memory projections updated;
- game-thread time per sense update;
- allocations per update;
- Visual Logger/log volume when debugging is enabled;
- decision changes caused by the sense.

Optimisation order:

1. reduce event production at source;
2. batch/merge similar events;
3. filter by coarse range/team/tag before expensive tests;
4. cap traces and spread expensive work;
5. lower update frequency for low-priority evidence;
6. move ambient background agents to Mass/aggregate simulation when full perception is unnecessary.

Do not first micro-optimise the callback if the actual issue is 500 agents emitting 30 scent points per second.

## MassAI and crowd-scale AI

### What "MassAI" means in this package

Unreal does not make every AI problem a Mass problem. Here, MassAI means an architecture where many NPC-like agents use MassEntity data/processor pipelines for simulation and only bridge to classic AI/gameplay where needed. The exact plugins/classes are target-sensitive; the stable interview model is the separation of identity, route, simulation LOD, representation and rich interaction. [SRC-MASS-001] [SRC-MASS-002] [SRC-MASS-009] [SRC-MASS-010]

### Actor AI versus Mass crowd

| Requirement | Actor AI likely better | MassAI likely better |
|---|---|---|
| Player, boss, quest NPC | rich identity, animation, replication, bespoke logic | only for distant/offscreen proxy if design supports it |
| 20 guards with BT combat | debuggability and direct components matter | optional for distant LOD, not mandatory |
| 2,000 pedestrians | Actor Tick/animation/collision too expensive | batched movement, avoidance, representation LOD |
| Traffic lanes | route/lane data and batch movement fit | yes, often ZoneGraph/Mass style |
| MMO-scale replicated crowds | needs custom relevance/identity design | possible but specialist; not automatic |
| Cinematic authored extras | Actor/Sequencer may fit | Mass if many and behaviour is simple/batchable |

### ZoneGraph and lane reasoning

ZoneGraph-style systems represent authored movement space as zones/lanes rather than a general combat NavMesh query per agent. That fits crowds and traffic because the agent often needs a lane, next segment, local steering and occupancy/avoidance, not a full arbitrary path query every decision tick. [SRC-MASS-009] [SRC-MASS-010]

Strong mental model:

- NavMesh answers broad traversability for general agents.
- ZoneGraph/lane graphs answer structured movement through authored spaces.
- Avoidance keeps local agents from colliding.
- Smart Objects reserve specific interaction slots.
- StateTree can express explicit activity sequences.
- Mass processors apply the batch simulation and representation changes.

Confusing these layers creates systems where every pedestrian runs a full BT/EQS/Nav query every frame and then still collides at a doorway.

### Mass crowd data flow

```text
Entity template
  -> movement fragments
  -> lane/route fragments
  -> target/activity fragments
  -> LOD fragments
  -> representation fragments
  -> processors:
      choose activity
      request route/lane
      follow lane
      avoid neighbours
      update transform
      update representation
      bridge important interactions
```

Fragments carry state. Processors update state. Traits configure templates. StateTree or signals may be used for selected behaviour flow, but durable state still belongs in fragments, not hidden task members. [SRC-MASS-001] [SRC-MASS-002]

### StateTree in MassAI

Mass StateTree-style integration is useful when many entities share a state machine and the state/tasks can be expressed through fragments/context rather than arbitrary Actor calls. Typical use:

- choose high-level activity (`Wander`, `Queue`, `UseObject`, `Flee`);
- set route/activity fragments;
- wait for movement/interaction completion;
- react to signals/events;
- bridge selected agents to Smart Objects.

Common mistake: running Actor-style StateTrees that call arbitrary UObjects per entity, every tick, for thousands of agents. That preserves the authoring shape but loses Mass's data-oriented benefit. [SRC-AI-010] [SRC-MASS-002]

### Smart Objects and Mass crowds

Smart Objects can be the concurrency boundary for rich world interactions: a bench slot, kiosk, repair terminal or cover point. In a Mass crowd, not every entity should become an Actor to "sit". A scalable design often has:

1. Mass activity selection chooses a need;
2. entity queries or receives candidate Smart Object data;
3. a claim/reservation is attempted through the authoritative object system;
4. nearby/important entity may promote to Actor or use a lightweight representation;
5. StateTree/processor tracks `Acquire -> Reach -> Use -> Release`;
6. every abort path releases the claim and clears fragments.

The claim/release contract from the Smart Object chapter remains the same; Mass adds scale and representation constraints. [SRC-AI-011] [SRC-AI-012] [SRC-MASS-002]

### Actor promotion contract

When a Mass entity becomes a rich Actor representation, define:

- stable entity ID and Actor mapping;
- which side owns transform during promotion;
- state copied into the Actor;
- state copied back into fragments;
- animation/collision readiness;
- external reference invalidation on demotion;
- server authority and replication audience;
- transition hysteresis and budget.

Never let Mass movement and Actor CharacterMovement both authoritatively integrate the same capsule. If an Actor is driving physical gameplay, Mass should either mirror it or suspend conflicting processors for that entity.

### Debugging workflow: Mass crowd jams in a lane

1. Confirm this is route selection, lane occupancy, local avoidance, animation/collision, or representation.
2. Inspect lane/route fragments for agents in the jam.
3. Count agents sharing the lane and desired direction.
4. Verify avoidance processors run and use expected neighbour data.
5. Check whether Smart Object or door/slot reservations are blocking exit.
6. Compare no-avoidance, avoidance and reduced-density runs.
7. Visualise routes/lane links and record a frame-by-frame Visual Logger or Mass debugger capture if available.
8. If Actor representations are involved, ensure collision capsules are not fighting the Mass avoidance result.

### Debugging workflow: Mass entity will not use Smart Object

1. Confirm entity has activity/need state and the right query/filter tags.
2. Confirm candidate Smart Objects are registered in the active streamed world cell.
3. Distinguish "candidate found" from "slot claimed".
4. Check lane/path reachability to slot approach transform.
5. Verify claim handle is stored in durable fragment/state, not a transient task local.
6. Confirm StateTree/task/signal wakes after movement or claim result.
7. Confirm every failure path clears claim and activity fragments.
8. Check whether representation tier lacks the Actor/animation capability required by the object.

### Debugging workflow: Mass performance regresses after adding behaviour

1. Record baseline entity count, archetypes, chunk occupancy, processors, representation counts and frame lanes.
2. Add the behaviour under a feature flag.
3. Compare processor time and per-entity time, not only FPS.
4. Inspect structural command volume and archetype count.
5. Inspect per-entity UObject calls and Actor promotions.
6. Count StateTree instances/tasks/transitions and Smart Object query/claim attempts.
7. Separate simulation regression from representation/animation/collision regression.
8. Add LOD/frequency tiers only after identifying which layer regressed.

### Production design patterns

| Pattern | Use | Caution |
|---|---|---|
| Perception aggregation | One squad/crowd collector emits shared evidence | Avoid losing individual stealth/facing rules if design needs them |
| Evidence tiering | Full perception near player, coarse events far away | Must preserve gameplay invariants |
| Activity fragments | Mass entities store durable current activity | Avoid arbitrary strings/tags without validation |
| Reservation fragments | Claim handles/state live in fragments | Release on every abort/despawn/demotion path |
| Actor promotion | Rich nearby interaction and animation | Prevent dual ownership and transition thrash |
| Route lane fragments | Structured movement through authored spaces | Do not replace combat NavMesh needs blindly |
| Signal-driven wakeup | Avoid polling inactive StateTrees/processors | Signal has little value if durable state is missing |
| Representation LOD | Actor/ISM/none selected by significance | Visual LOD is not simulation correctness |

## Strong interview answer patterns

### "How would you implement a custom AI sense?"

> I would first prove it belongs in perception rather than a simple event or Blackboard update. Then I define the evidence payload: source, location, strength, tag, time, team and expiry. The sense batches and filters events, reports stimuli to listeners and stays out of decisions. The controller or memory component merges stimuli into current target, last-known location and confidence. I instrument event counts, discarded reasons, listener counts, traces and memory updates. Exact `UAISense` and config API details are branch-sensitive, so I verify headers before writing production code.

### "Why does a guard remember a player forever?"

> I separate current sensing from memory. I inspect every sense's last stimulus, age, success/loss state and Max Age/decay. Then I check whether failed sight clears only visual certainty or all target memory, whether another sense is still refreshing evidence, and whether old streamed/saved evidence rehydrates as fresh. I would add a deterministic test where the agent sees, loses, investigates, hears, and finally forgets after the configured interval.

### "When would you use MassAI instead of Actor AI?"

> I would use Mass when many agents share stable data shapes and behaviour can run in batches with simulation and representation LOD. Pedestrians, traffic and ambient crowds fit. Bosses, players and rich combat NPCs usually remain Actors. A hybrid is common: Mass owns large-scale state, and nearby important entities promote to Actors. I profile simulation, structural churn, bridge cost and representation separately, and I verify plugin/source status for ZoneGraph, Mass Crowd, Mass StateTree and replication.

### "How would you debug a Mass pedestrian not reaching a kiosk?"

> I would split candidate, claim, route, movement, representation and use. First check the activity fragment and candidate query. Then prove whether a Smart Object slot was claimed. Then inspect lane/path reachability and movement/avoidance fragments. Then inspect StateTree/task/signal completion and release paths. If the entity is in a low representation tier, I check whether the required Actor/animation bridge exists. I avoid fixing this by just making it an Actor until I know which layer failed.

## What a three-year engineer should know

A strong three-year UE engineer should know:

- custom senses are specialist, not mandatory interview baseline;
- perception emits evidence, not gameplay truth;
- memory projection should be explicit and debuggable;
- event batching/filtering matters before expensive traces;
- AI Debugger and Visual Logger should be used before changing decisions;
- Mass is for batchable populations with stable data shapes;
- Mass simulation, representation, and Actor promotion are separate;
- ZoneGraph/crowd systems are plugin/version-sensitive;
- Smart Object claim/release is a concurrency boundary;
- StateTree/Mass integration should operate on fragments/context, not arbitrary per-entity Actor calls.

Specialist depth includes exact `UAISense` API implementation, custom listener/source registration, native Mass StateTree tasks/evaluators, ZoneGraph runtime query APIs, crowd lane reservation, Mass replication, custom representation actors, target-branch debugger commands and production-scale telemetry.

## Hands-on verification

Use the companion [[ue_ai_senses_massai_workbook|AI custom senses and MassAI workbook]].

Minimum evidence packet:

1. custom scent/noise sense event schema and memory policy;
2. one "never forgets" bug fixed with before/after logs;
3. one listener/source registration bug diagnosed;
4. 1/20/100-agent event-count and trace-count profile;
5. Mass crowd lane/route prototype with LOD/representation counts;
6. Smart Object claim/use/release path for one crowd activity;
7. actor promotion/demotion trace with one prevented dual-writer bug;
8. interview explanation comparing Actor AI, BT/StateTree AI and MassAI.

## Conflict and uncertainty notes

- Epic docs are maintained and can resolve to later UE5 behaviour even with a UE5.6 selector. Treat exact APIs, debugger UI, plugin status, default processor groups, Mass StateTree behaviour and Mass Crowd examples as target-branch proof work.
- This chapter uses official API URLs as terminology anchors. The pages are dynamically rendered in this environment, so implementation samples are explicitly schematic.
- "MassAI" is used as an architectural shorthand for Mass-based AI/crowd workflows. It should not be treated as one stable monolithic subsystem across all UE5 versions.

## Source map

- AI Perception concepts and debugging: [SRC-AI-004] [SRC-AI-008] [SRC-AI-009]
- Custom sense/API anchors: [SRC-AI-016] [SRC-AI-017] [SRC-AI-018] [SRC-AI-019]
- StateTree and Smart Objects: [SRC-AI-010] [SRC-AI-011] [SRC-AI-012]
- MassEntity/MassGameplay: [SRC-MASS-001] [SRC-MASS-002]
- Mass debugging/avoidance/crowd: [SRC-MASS-007] [SRC-MASS-008] [SRC-MASS-009] [SRC-MASS-010]
