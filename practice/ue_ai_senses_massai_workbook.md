# AI Custom Senses and MassAI Workbook

See also: [[ue_ai_custom_senses_massai]], [[ue_ai_navigation]], [[ue_smart_objects_statetree]], [[ue_massentity_ecs]], [[ue_hands_on_projects]].

This workbook turns the specialist AI chapter into repeatable evidence. The goal is not to memorise APIs; it is to prove that perception, memory, Mass simulation, representation and rich interactions are separable and measurable.

**Target projects:** Project 2 AI Combat Sandbox and Project 5 MassEntity Crowd Prototype  
**Expected depth:** D4 specialist practice  
**Source anchors:** [SRC-AI-004] [SRC-AI-016] [SRC-AI-017] [SRC-AI-018] [SRC-AI-019] [SRC-MASS-001] [SRC-MASS-002] [SRC-MASS-009] [SRC-MASS-010]

## Workbook Rules

1. Record exact engine version, enabled plugins/modules and target platform.
2. Keep all custom sense C++ code labelled branch-verified or schematic.
3. Every scenario must record before/after evidence, not only a fix.
4. Separate "source event emitted", "stimulus delivered", "memory changed" and "decision changed".
5. For Mass work, separate simulation, structural, bridge and representation cost.
6. Prefer deterministic fixtures before random crowd scenes.

## Lab 1: Custom Scent Sense Contract

**Goal:** Design the smallest useful custom sense without hiding decisions inside it.

**Build:**

1. Define `ScentEvent` fields: source, stable ID/generation, location, strength, tag, server time and optional team.
2. Define sense config values: max range, decay seconds, update interval, trace requirement, team filter and debug category.
3. Define memory keys: `ScentTarget`, `ScentLastLocation`, `ScentConfidence`, `ScentAge`, `EvidenceTag`.
4. Define merge policy with sight/hearing/damage.

**Required evidence:**

| Evidence | Pass condition |
|---|---|
| Event schema | Every field has a producer and consumer |
| Memory table | Sight/hearing/scent interactions are explicit |
| Failure policy | Destroyed source, expired scent and team change are covered |
| Debug logs | Event emitted, accepted, discarded and projected are separate |

**Interview prompt unlocked:** "How would you implement a custom AI sense?"

## Lab 2: Listener and Source Registration Failure

**Goal:** Diagnose "custom sense never fires" without changing decision logic first.

**Inject failures:**

- listener lacks perception component;
- custom sense config missing;
- event emitted in wrong World;
- source Actor destroyed before update;
- range/team filter rejects everything;
- module dependency missing in non-editor target.

**Debug sequence:**

1. One listener, one source, known location.
2. Confirm event producer fires.
3. Confirm sense receives pending event.
4. Confirm listener count after filters.
5. Confirm stimulus/memory projection.
6. Re-enable full AI decision only after perception evidence is correct.

**Deliverable:** a table with failure, first contradictory observation, tool/log used and fix.

## Lab 3: The "Remembers Forever" Bug

**Goal:** Prove the difference between current perception and memory.

**Scenario:**

1. AI sees player.
2. Player breaks line of sight.
3. AI hears one noise.
4. Scent evidence decays.
5. AI investigates last-known location.
6. AI forgets and returns to patrol.

**Required assertions:**

- lost sight lowers visual certainty but does not erase hearing/scent memory immediately;
- scent confidence decays and eventually falls below decision threshold;
- a new hearing event refreshes investigation memory;
- Max Age/decay settings are recorded and tested;
- Blackboard/StateTree keys visibly change over time.

**Common weak fix:** clearing target on every failed sight event.

## Lab 4: Custom Sense Scale Profile

**Goal:** Catch the first real bottleneck before optimising code style.

Run 1, 20 and 100 agents. For each, emit 1, 10 and 50 evidence events per second.

| Metric | 1 agent | 20 agents | 100 agents |
|---|---:|---:|---:|
| Events emitted/s | | | |
| Events merged/discarded/s | | | |
| Listeners considered/s | | | |
| Traces/s | | | |
| Stimuli reported/s | | | |
| Perception callbacks/s | | | |
| Memory updates/s | | | |
| Game-thread ms | | | |
| Allocations/update | | | |

**Fix candidates to compare:**

- source-side event throttling;
- event merge by tag/location/source;
- range/team prefilter before traces;
- trace cap/staggering;
- lower update frequency for low-confidence evidence;
- significance tiers.

## Lab 5: ZoneGraph/Mass Crowd Route Prototype

**Goal:** Distinguish general NavMesh pathfinding from structured lane movement.

**Build:**

1. Create a small pedestrian area with at least three lane choices and one bottleneck.
2. Spawn equivalent Actor and Mass crowds.
3. Track lane/route state, desired velocity and local avoidance state.
4. Add one blocked lane or closed area.
5. Compare route/lane failure to local avoidance failure.

**Evidence:**

- lane or route fragment/state per agent;
- processor timing;
- crowd density at bottleneck;
- representation counts;
- before/after blocked-lane behaviour;
- explanation of why full NavMesh/EQS per agent is or is not needed.

## Lab 6: Mass StateTree Activity

**Goal:** Express one simple crowd activity with durable fragment state, not hidden task locals.

**Activity:** queue at a kiosk, sit on a bench, flee from hazard, or inspect an object.

**Requirements:**

1. Activity intent fragment.
2. Activity target/claim fragment.
3. StateTree or equivalent flow: `Find -> Reserve -> Reach -> Use -> Exit`.
4. Signal or state update wakes after movement/claim result.
5. Abort clears durable fragments and releases claims.

**Bad implementation to capture:** task stores claim handle only in local state, then loses it on abort/demotion.

## Lab 7: Smart Object Claim From Mass

**Goal:** Prove "candidate found" and "slot claimed" remain different under crowd load.

**Scenario:** 20 entities compete for 5 slots.

**Record:**

- candidate count per entity;
- claim attempts;
- claim success/failure reason;
- retry/backoff policy;
- release reasons;
- leaked claims after death/despawn/demotion;
- animation/slot alignment if promoted.

**Pass condition:** no two users own the same slot, and no slot remains claimed after all users are destroyed.

## Lab 8: Actor Promotion Dual-Writer Bug

**Goal:** Detect and fix Mass and Actor systems both writing authoritative transform/state.

**Inject bug:**

1. Nearby Mass entity promotes to Actor.
2. Mass movement processor continues writing transform.
3. Actor CharacterMovement also writes transform.
4. Observe jitter, collision mismatch or correction.

**Fix policy options:**

- suspend Mass movement for promoted entity and mirror Actor state back to fragments;
- keep Mass authoritative and make Actor presentation-only;
- split high-fidelity Actor agents out of Mass entirely for the interaction.

**Evidence:** promotion trace, transform writer log, representation tier, processor query exclusion, before/after movement/collision behaviour.

## Final Report Template

```markdown
## AI Senses and MassAI Evidence Packet

**Engine / branch / plugins:**  
**Target platform and build:**  
**Custom sense scope:**  
**Memory policy:**  
**Mass crowd scope:**  
**Representation tiers:**  

### Perception Evidence
| Case | Event | Stimulus | Memory | Decision | Tool |
|---|---|---|---|---|---|

### Performance Evidence
| Scenario | Agents | Events/s | Traces/s | Processor/sense ms | Actor count | ISM count | Notes |
|---|---:|---:|---:|---:|---:|---:|---|

### Bugs Found
| Bug | Layer | First contradictory observation | Fix | Regression guard |
|---|---|---|---|---|

### Interview Summary
Write 10 sentences explaining:
1. why the custom sense is evidence;
2. how memory decays;
3. why Mass was or was not justified;
4. how representation LOD differs from simulation LOD;
5. how Smart Object claim/release was kept correct.
```

