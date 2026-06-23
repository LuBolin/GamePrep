# Unreal Engine Networking and Replication

## Cluster 1 — Authority, Ownership, Properties, RPCs, Relevancy, and Dormancy

**Priority:** P0  
**Expected depth:** D4 for gameplay/networking/generalist roles; D3 broad awareness for adjacent roles  
**Version scope:** UE5.3–UE5.6 preferred. Generic replication is the baseline in this chapter; Replication Graph and Iris are identified as alternative/advanced systems rather than assumed defaults.  
**Prerequisites:** Gameplay Framework roles, UObject references, delegates/events, and C++/Blueprint authority boundaries.

### Networking changes the programming model

A multiplayer game is not one world with delayed function calls. It is several process-local worlds with separate object instances, connected by selective, delayed, bandwidth-limited, sometimes lossy communication. The authoritative server owns the true replicated gameplay state; clients send intent, receive selected state, predict selected local actions, and present/interpolate results. [SRC-NET-001]

Start every networking decision with five questions:

1. Which machine is authoritative for this state transition?
2. Which connection(s) need the result?
3. Is this durable state or a transient event?
4. What happens under loss, delay, duplication assumptions, or reordering?
5. How much bandwidth/CPU does the update consume at expected player and Actor counts?

If these are unanswered, adding `Replicated` or `Server` specifiers is decoration, not design.

### Network modes and testing bias

| Mode | Meaning | Interview/production implication |
|---|---|---|
| Standalone | One process acts with server functionality and local players | Hides many ownership/remote-instance bugs |
| Listen server | Server plus a local host player | Host has zero network leg and local server access; can hide host-only success |
| Dedicated server | Headless authoritative server, no local player | Best representation of server/client separation |
| Client | Connects to remote server; no server authority | Must request state changes through valid owning pathways |

Standalone and listen-host tests are insufficient because host code often sees both authority and local presentation. Test at least dedicated/listen plus a remote client, and test each relevant player perspective. [SRC-NET-001]

### Authority, local role, ownership, and owning connection

These are related but not synonyms:

- **Authority:** the server instance of a replicated Actor owns authoritative state.
- **Local role:** how this machine participates for that Actor (`Authority`, `AutonomousProxy`, `SimulatedProxy`, or none).
- **Actor Owner:** gameplay/network ownership chain set through Actor ownership.
- **Owning connection:** the client connection reached through an owning PlayerController.
- **Possession:** Controller-to-Pawn control relationship.
- **Relevancy:** whether this Actor should replicate to a specific connection now.

Epic explicitly warns that role/remote role are not ownership. The server has authority over replicated Actors. The owning client's controlled Pawn commonly appears as an autonomous proxy there; other clients see simulated proxies. [SRC-NET-002] [SRC-NET-003]

Ownership matters because it determines:

- whether a client can successfully call a Server RPC on an Actor;
- which connection receives a Client RPC;
- owner-only or skip-owner property conditions;
- owner-based relevancy and priority.

The common “Server RPC silently does nothing” bug is usually: the Actor does not replicate, the caller does not own it through a valid owning connection, the call is made too early, or the RPC contract/signature is wrong.

### Model intent, validation, and result separately

A secure gameplay flow is:

```text
Owning client gathers input
    -> optional local prediction/presentation
    -> Server RPC requests an action (intent, not trusted outcome)
    -> server validates authority, rate, state, target, distance, resources
    -> server mutates authoritative state
    -> replicated properties / targeted RPCs communicate result
    -> clients update presentation through OnRep/events
```

Do not send “I dealt 100 damage” when the server can derive whether a shot hit and how much damage applies. Send compact intent/evidence appropriate to the game's prediction and anti-cheat model. Server authority does not mean accepting client parameters because the function executed on the server.

### Actor and component replication setup

Replication is opt-in for most Actors. `bReplicates = true` enables an Actor for replication; movement replication is a separate concern (`bReplicateMovement` or a specialised movement system). A server-spawned replicated Actor can create client proxies when relevant. A client-spawned Actor is local and does not become an authoritative replicated Actor merely because its class normally replicates. [SRC-NET-001]

Actor Components replicate as part of their owning Actor when both the Actor and Component are configured appropriately. A Component is not an independent network channel/ownership island; reason from its owning Actor. Replicated non-Actor UObjects require replicated-subobject support and are specialist depth, not automatic because they derive from UObject.

### Replicated properties are state convergence

Properties replicate from server to clients when the Actor is replicated/relevant and the replication system detects state that should be sent. The normal C++ declaration needs both the property specifier and lifetime registration:

```cpp
UPROPERTY(ReplicatedUsing=OnRep_Health)
float Health = 100.0f;

void ACombatant::GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(ACombatant, Health);
}

UFUNCTION()
void ACombatant::OnRep_Health(float OldHealth)
{
    HealthChanged.Broadcast(OldHealth, Health);
}
```

`ReplicatedUsing` associates a RepNotify/OnRep callback on receiving clients. Treat OnRep as a local reaction to newly applied state: update visuals, UI, caches, or derived presentation. The server commonly invokes the same reaction explicitly after server mutation if shared local behaviour is needed; do not assume the server's property assignment triggers the client-style notification in every path. [SRC-NET-004]

Properties are good for durable “what is true now” state because a later update can converge a client even if an earlier property bunch was lost. RPCs are events and do not automatically reconstruct current truth for late joiners.

### OnRep architecture and ordering

Good OnRep functions are:

- idempotent or tolerant of repeated/combined state transitions;
- independent of arbitrary ordering between different properties;
- safe when referenced Actors/Components have not resolved yet;
- focused on local reaction, not authoritative mutation;
- able to derive old/new transition where the signature supports old value or a cached value.

There is no deterministic ordering guarantee between OnRep callbacks for different replicated variables. If several values form one invariant or transition, replicate one struct/state machine value, or coordinate reactions after all required fields are available. RPC ordering across different Actors is also not globally preserved. [SRC-NET-005]

This is why separate `bIsDead`, `Health`, `DeathLocation`, and `Killer` fields can produce fragile presentation if each OnRep assumes all others already updated. A single replicated death-state struct or explicit state transition ID can make the contract coherent.

### Replication conditions are audience policy

Common conditions include:

| Condition | Use |
|---|---|
| `COND_OwnerOnly` | Private state for owning connection |
| `COND_SkipOwner` | State the owner predicts locally, sent to others |
| `COND_InitialOnly` | Initial configuration that should not update continuously |
| `COND_SimulatedOnly` | Data relevant to simulated proxies, not autonomous owner |
| `COND_Custom` / dynamic controls | Advanced actor-wide condition; beware per-connection limitations/cost |

Conditions reduce audience/bandwidth and clarify privacy, but are not security by obscurity. Never put secrets in broadly replicated state then rely on UI not displaying them. Epic notes custom active overrides are per Actor rather than safe for arbitrary per-connection state. [SRC-NET-004]

### RPC types and routing

| RPC | Called from | Executes on | Typical purpose |
|---|---|---|---|
| Server | Owning client or server | Server | Submit player intent/request authoritative action |
| Client | Server | Actor's owning client | Private response/correction/presentation for one connection |
| NetMulticast | Server (for network fan-out) | Server and relevant clients | Transient event for currently relevant recipients |

RPC routing depends on replication and owning connection. A Server RPC on an arbitrary world Actor that the client does not own will not become valid because the client can obtain a pointer to it. Route the request through the owned PlayerController/Pawn/Component and validate the target on the server. [SRC-NET-003] [SRC-NET-006]

Use properties for durable state and RPCs for transient actions. Examples:

- Door open/closed: replicated state (late joiner needs truth).
- One-off cosmetic spark: possibly unreliable multicast if loss is acceptable.
- Inventory request: Server RPC intent, server state mutation, owner-only replicated inventory/result.
- Error message for requester: targeted Client RPC or replicated owner state, depending durability.

### Reliable does not mean immediate, free, or ordered with everything

Reliable RPCs are retransmitted until acknowledged and preserve ordering within their guaranteed stream/context, but they cost bandwidth and can block later reliable traffic while missing data is recovered. Unreliable RPCs can be dropped and do not promise order. Use reliable for infrequent essential events that cannot be reconstructed from state; use unreliable for frequent superseding/ephemeral data. [SRC-NET-006]

Never send a reliable RPC every Tick. A queue of reliable cosmetic or input updates under packet loss can create latency and overflow pressure. Prefer replicated state, unreliable updates with sequence/timestamp semantics, batching, or built-in movement systems.

Replication has nuanced execution order: OnReps for different properties have no deterministic relative order; RPC order across Actors is not preserved; property updates and RPCs have documented but loss-sensitive ordering interactions. Build protocols around explicit state/sequence dependencies rather than observed LAN order. [SRC-NET-005]

### Relevancy is per connection

The server only considers/sends an Actor to connections for which it is relevant. Base relevancy can account for ownership, attachment, always-relevant flags, owner relevancy, distance, view target, and class-specific overrides. Dynamically spawned replicated Actors can be destroyed on a client when they become irrelevant and recreated if relevant later. [SRC-NET-007]

Relevancy is not visibility culling. An invisible Actor may still matter to gameplay; a visible long-range event may need relevance even outside normal distance. Override `IsNetRelevantFor` cautiously because it runs in a performance-sensitive per-connection context and mistakes create bandwidth explosions or missing Actors.

Useful controls:

- `bAlwaysRelevant`: expensive if overused.
- `bOnlyRelevantToOwner`: private Actor to owning connection.
- `bNetUseOwnerRelevancy`: delegate relevance/priority to owner.
- `NetCullDistanceSquared`: distance threshold where applicable.

### Dormancy reduces consideration of infrequently changing Actors

Dormancy keeps an Actor existing on client and server while allowing the server to skip normal replication consideration for it. It is a strong optimisation for many replicated Actors that rarely change. It differs from irrelevancy: an irrelevant dynamic Actor may be destroyed on the client; a dormant Actor remains. [SRC-NET-008]

Rules:

1. Put only genuinely stable Actors to sleep.
2. Wake or flush dormancy **before** changing replicated state.
3. Wake the owning Actor for replicated Component/subobject changes.
4. Avoid rapid sleep/wake churn; it can cost more than staying awake.
5. Use dormancy logging/validation to diagnose lost updates.

Epic notes partial dormancy is not recommended and planned for deprecation in current maintained documentation; do not build new architecture around it. [SRC-NET-008]

### Roles and movement prediction awareness

For a player Character:

- Server instance: authority.
- Owning client's instance: autonomous proxy; predicts its local movement.
- Other clients' instances: simulated proxies; interpolate/simulate received movement.

`CharacterMovementComponent` records client moves, sends compact move data, lets the server reproduce/validate, acknowledges or corrects, and replays unacknowledged client moves after correction. Simulated proxies smooth remote updates. Directly setting transforms around this system can cause corrections because the movement was not captured in its prediction protocol. [SRC-NET-002] [SRC-NET-009]

At three-year breadth, explain prediction, authoritative validation, correction, replay, and smoothing. Extending `FSavedMove_Character`/packed move data is specialist depth.

### Generic replication, Replication Graph, and Iris

UE5.6 documentation presents generic replication, Replication Graph, and Iris as available replication systems. Generic replication remains the baseline for this chapter. Replication Graph is a scalable relevance-routing architecture for many Actors/connections. Iris is a newer replication framework with different descriptors/filtering/serialisation architecture and version/plugin/configuration considerations. Do not claim Iris is automatically used merely because the project is UE5. [SRC-NET-001]

Interview breadth:

- Know why Replication Graph exists: avoid naïve per-Actor/per-connection scaling through graph nodes/spatial routing.
- Know Iris exists and may be project/version dependent.
- Be honest if you have not shipped either; explain the baseline concepts they still solve—state serialisation, filtering, prioritisation, and bandwidth.

### Debugging workflow: “it works on server but not client”

1. Reproduce with a remote client, not only standalone/listen host.
2. Log net mode, local role, remote role, Actor owner, owning PlayerController/connection context, and whether the Actor replicates.
3. Confirm the Actor was spawned on the server and is relevant to that client.
4. For properties: confirm `UPROPERTY(Replicated...)`, `GetLifetimeReplicatedProps`, server mutation, condition, dormancy, and Actor/Component replication.
5. For RPCs: confirm call side, intended execution side, ownership, replication, timing, and implementation signature.
6. Check whether the “missing” object reference is not yet resolved or not legal/stably named for network reference.
7. Disable dormancy/relevancy optimisation selectively to isolate policy from base replication.
8. Add packet lag/loss simulation and retry; LAN success is weak evidence.
9. Capture Networking Insights/Network Profiler and inspect Actor, property, and RPC traffic rather than spamming reliable calls.

### Debugging workflow: RPC does not execute

Use this decision chain:

```text
Is the Actor replicated and present on both sides?
  -> Is this caller allowed by ownership/owning connection?
  -> Is the call made from the correct machine?
  -> Is the Actor channel/relevancy established yet?
  -> Is the RPC declaration/implementation correct?
  -> Is dormancy/timing involved?
  -> Is the symptom actually execution on a different instance than the log/view assumes?
```

Do not “repair” an invalid client call by making the target always relevant or reliable. Fix the authority/ownership route.

### Performance workflow

1. Establish target player count, Actor count, update rates, bandwidth, and server frame budget.
2. Capture real traffic; rank Actors, properties, and RPCs by bytes and frequency. Network Profiler can attribute traffic, while newer Insights tooling may suit target versions/workflows. [SRC-NET-010]
3. Remove redundant high-frequency reliable RPCs.
4. Tighten relevancy and ownership audiences; avoid global `bAlwaysRelevant`.
5. Lower update rate only with acceptable interpolation/latency.
6. Use dormancy for stable Actors and validate wake-before-change.
7. Batch/delta serialise large changing collections; investigate Fast Array when item-level delta semantics fit.
8. Consider Replication Graph/Iris only after profiling shows the baseline scaling problem and project constraints permit them.

### Common bugs and misconceptions

1. **Authority means locally controlled.** Server authority and owning autonomous proxy are different.
2. **Ownership means attachment or Outer.** Network owning connection follows Actor ownership/PlayerController context.
3. **Multicast called by a client broadcasts.** Network fan-out requires the authoritative server path.
4. **Reliable means faster/safer.** It retransmits and can block; overuse increases latency under loss.
5. **Replicated property is an event log.** It communicates current state and may coalesce intermediate changes.
6. **OnRep order follows declaration.** Different OnReps have no deterministic ordering guarantee.
7. **Client can replicate a property by assigning it.** Server replication overwrites non-authoritative client state.
8. **Always relevant fixes networking.** It can hide relevancy design and explode bandwidth/CPU.
9. **Dormant means destroyed/not loaded.** The Actor remains; it is skipped for normal replication consideration.
10. **Standalone success proves multiplayer correctness.** It hides remote ownership and timing.
11. **Server RPC validation ends at `_Validate` syntax.** Real gameplay validation remains system-specific and mandatory.
12. **UE5 means Iris.** Replication system selection is project/version/config dependent.

### Strong interview answer pattern

For “how would you replicate this system?”:

1. Identify authoritative state and mutation side.
2. Identify owning connection and audiences.
3. Separate durable state from transient events.
4. Choose property conditions/RPC type and reliability.
5. Discuss relevancy, dormancy, update frequency, and late joiners.
6. Explain client prediction only where responsiveness requires it.
7. Give a packet-loss/multi-client test and profiler plan.

Example: “The server owns health. The owning client sends a reliable, rate-limited damage-request only for an action it owns, but the server derives hit/damage and mutates health. Health is replicated with OnRep to relevant clients; private details use owner-only state. Death is represented by durable replicated state so late joiners converge, while cosmetic impact effects can be unreliable multicast if loss is acceptable. I test dedicated server with two clients under lag/loss and inspect property/RPC traffic.”

### What a three-year engineer should know

You should reason correctly about authority versus ownership, know why Server RPCs fail, configure Actor/Component/property replication, use OnRep safely, choose RPC type/reliability, explain roles, relevancy and dormancy, and debug with multiple processes plus packet conditions. You should recognise CharacterMovement prediction and Replication Graph/Iris without claiming internals you have not used.

**Specialist depth:** custom movement saved moves, rollback/lag compensation, Fast Array delta serialisation, replicated subobjects, Replication Graph nodes, Iris descriptors/filters/serialisers, network physics, packet-level profiling, and large-scale backend/session architecture.

### Hands-on verification

Complete Project 3 in `practice/ue_hands_on_projects.md`. Every feature must include an authority/audience/state-event table and at least one intentionally broken ownership, relevancy, dormancy, or reliability case.

### Conflict and uncertainty notes

- Maintained Epic pages may surface UE5.7 content when searching target URLs. Baseline concepts are stable, but exact RPC execution tables, Iris status, and dormancy implementation details must be confirmed against UE5.6 docs/source for shipping decisions.
- Role documentation can phrase autonomous proxies as having authority to “change state and call remote functions”; the server remains authoritative for replicated game truth. Treat autonomous authority as local prediction/input privilege within the server-validated model. [SRC-NET-002]
- Dormancy documentation states changes should follow wake/flush-before-mutation; behaviour that appears to work after mutation is implementation-dependent and must not be relied upon. [SRC-NET-008]

## Cluster 2 - Specialist Replication: Components, Subobjects, Fast Arrays, Replication Graph, Iris, and Debug CVars

The first replication question is "who owns the truth?" The specialist follow-up is "how does that truth scale when it is nested, high-cardinality, or per-connection?" This section covers the common P2/P3 follow-ons: replicated Actor Components, replicated UObject subobjects, delta-replicated collections, Replication Graph, Iris awareness, push-model debugging and network CVars.

### Actor Components are replicated through their owning Actor

Actor Components do not replicate by default. A replicated component needs both sides of the contract:

1. The owning Actor replicates.
2. The component is set to replicate.
3. The component registers its replicated properties in its own `GetLifetimeReplicatedProps`.
4. Component RPCs follow normal Actor RPC concepts, but can carry more overhead than routing through the owning Actor in some designs. [SRC-NET-011]

Static and dynamic components are different:

| Component kind | Created where | Replication implication | Common bug |
|---|---|---|---|
| Static default component | Actor constructor or Blueprint component tree | Exists on both server and client as part of the owning Actor class/default construction; replicated fields/events still need component replication enabled | Component exists but properties never update because `SetIsReplicatedByDefault`/Replicates was missed. |
| Dynamic component | Runtime gameplay code, normally server-side for replicated creation | Server-created component can replicate creation/deletion to clients when component replication is enabled | Client creates a local component and expects server/client convergence. |
| Component subobject owner | Component owns additional replicated UObjects | Owning Actor and component must both replicate before the component subobject's conditions matter | Subobject uses `COND_OwnerOnly` but the component itself skips the owner. |

Bandwidth matters. Epic's component replication page calls out per-component NetGUID header/footer overhead plus property/RPC cost; one component is cheap, but a high number of components/subobjects can become a server and bandwidth problem. [SRC-NET-011]

### Replicated UObject subobjects are not lightweight Actors by magic

Replicated UObjects can be useful for state that belongs to an Actor or Actor Component but does not deserve full Actor identity. The owning Actor/channel still provides replication context. Treat a replicated UObject as a nested state object with explicit registration, lifetime and reference mapping. [SRC-NET-012]

There are two broad cases:

| Case | Server/client existence | Reference behaviour | Design implication |
|---|---|---|---|
| Default subobject | Constructed in owner constructor on both server and clients with stable naming | Network references can be stably resolved relative to the owner even before property replication starts | Good for fixed nested state known at class construction time. |
| Dynamic subobject | Created at runtime on the server | A replicated reference can be null on clients until the subobject bunch creates/maps the client object | Design UI/gameplay to tolerate an unresolved reference and react on `OnRep`/valid mapping. |

Required setup checklist:

1. UObject class supports networking, usually via `IsSupportedForNetworking`.
2. UObject class implements `GetLifetimeReplicatedProps`.
3. Owning Actor or Component replicates.
4. Dynamic subobject pointer is replicated if clients need a reference to it.
5. Subobject is actually registered or explicitly replicated.
6. Removal unregisters the subobject before it is deleted/collected.

The registered subobjects list is the modern path to know about. It avoids per-subobject virtual calls and is the only method compatible with Iris according to the maintained Object Replication page. The older `ReplicateSubobjects` override remains a backward-compatible Generic/Replication Graph path. [SRC-NET-012]

```cpp
// Schematic only: verify exact API/signature in the target UE branch.
AMyActor::AMyActor()
{
    bReplicates = true;
    bReplicateUsingRegisteredSubObjectList = true;
}

void AMyActor::CreateServerSubobject()
{
    if (!HasAuthority())
    {
        return;
    }

    if (IsValid(MySubobject))
    {
        RemoveReplicatedSubObject(MySubobject);
    }

    MySubobject = NewObject<UMyReplicatedState>(this);
    MySubobject->InitialiseFromAuthoritativeState(...);
    AddReplicatedSubObject(MySubobject);
}

void AMyActor::DestroyServerSubobject()
{
    if (IsValid(MySubobject))
    {
        RemoveReplicatedSubObject(MySubobject);
        MySubobject = nullptr;
    }
}
```

The dangerous bug is a stale raw pointer in the registered subobject list. The maintained Object Replication page explicitly warns that a subobject must be removed from the list before deletion; otherwise the list can retain a raw pointer after GC purges the object. [SRC-NET-012]

### Subobjects and RPCs need routing discipline

Replicated UObject subobjects do not automatically behave like Actors for RPCs. The maintained docs describe overriding callspace/remote-function routing for actor-owned objects when subobject RPCs are required, and point to engine test plugin examples. [SRC-NET-012]

Interview-safe rule:

- Prefer the owning Actor or replicated Component as the RPC surface unless a subobject RPC has a strong modelling reason.
- Keep subobject state replicated as state when possible.
- If you implement subobject RPC routing, write target-branch tests for ownership, callspace, teardown, dormancy, relevancy and object mapping.

### Fast Array is for item-level delta semantics, not every `TArray`

`FFastArraySerializer` is the Unreal vocabulary for efficient delta replication of arrays of replicated items. It is specialist depth because correctness depends on item identity, dirty marking, add/remove/change hooks, replication limits and target-branch API details. [SRC-NET-015]

Use Fast Array when:

- the collection is large enough or changes often enough that whole-array replication is wasteful;
- individual items have stable replication identity;
- add/remove/change events matter on clients;
- the server is authoritative over the collection;
- you can centralise all mutations through dirty-marking helpers.

Avoid it when:

- the array is tiny and rarely changes;
- full replacement is simpler and cheap;
- item identity is unstable or can be derived from other replicated state;
- designers mutate arrays in multiple places without a single authoritative path;
- you have not written tests for join-in-progress, removal, reorder and repeated modify cases.

Fast Array answer pattern:

1. Define the item identity.
2. Define authoritative mutation API.
3. Mark item/array dirty exactly when needed.
4. Handle add/change/remove callbacks idempotently.
5. Test late join, packet loss, reorder, removal and item reuse.
6. Measure bytes and CPU against simpler replication before committing.

### Push model changes how dirtiness is reported, not who owns truth

Push-model replication lets game code notify networking that properties changed instead of relying purely on polling/scraping. Network debug CVars include validation options for push-model properties and skipping undirtied replication/Fast Arrays. [SRC-NET-016]

Do not describe push model as "faster replication" in the abstract. The useful answer is:

- It can reduce unnecessary scanning/work when properties are marked dirty correctly.
- It can fail silently from a gameplay point of view if mutation paths forget to mark dirty.
- It needs validation CVars/tests in development.
- It does not change server authority, relevancy, dormancy, ownership or protocol design.

### Replication Graph is a relevance-list architecture

Generic replication can degrade when many Actors must evaluate many connections. Replication Graph moves relevance list construction into persistent graph nodes that can store shared data across frames and connections. It is designed for large numbers of players and replicated Actors, and the docs cite Fortnite-scale motivation. [SRC-NET-013]

Good Replication Graph candidates:

- many replicated Actors and connections;
- spatial or team/room/faction relevance;
- persistent buckets/lists can be reused across clients;
- profiling shows server CPU cost in replication list construction;
- the project can own custom graph node logic.

Bad candidates:

- low player/Actor count;
- mostly owner-only gameplay;
- no profiling evidence;
- team lacks networking/tooling ownership;
- splitscreen/target platform constraints conflict with the plugin/page caveats. [SRC-NET-013]

Graph-node design vocabulary:

| Node idea | Use case | Failure mode |
|---|---|---|
| Spatial grid/zone | World-space relevance by viewer location | Actors in wrong cell, stale movement updates, cell size mismatch |
| Always relevant | GameState-like global Actors | Global list grows until every client receives too much |
| Owner/team relevant | Private inventory, team reveal, squad data | Wrong owner/team membership leaks or hides data |
| Dormant/static bucket | Placed stable objects | Wake/change path not integrated |
| Carried/attached items | Items update with carrier | Item relevance not transferred when carrier changes |

### Iris is opt-in and experimental in the docs, not automatic UE5

Iris is a newer opt-in replication system, described as experimental in Epic docs and motivated by larger worlds, higher player counts and lower server costs. It works alongside the existing replication system; projects choose their replication system. [SRC-NET-014]

For interview breadth:

- Know that Iris exists.
- Know it changes lower-level replication architecture and requires branch/config awareness.
- Know registered subobjects are required for Iris subobject replication, and non-Actor/non-Component objects interfacing with Iris need additional replication-fragment registration in the maintained Object Replication docs. [SRC-NET-012]
- Do not claim "UE5 uses Iris" unless the project config/source confirms it.

For specialist answers, say what you would verify:

1. Is `net.Iris.UseIrisReplication` enabled in the target project/build?
2. Which Actors/classes use Iris-compatible descriptors/fragments?
3. Are subobjects registered through the registered list?
4. Do custom UObjects implement target-branch fragment registration where required?
5. Which debug tools/CVars exist in this exact branch?
6. What is the migration risk from Generic/Replication Graph assumptions?

### Network debug CVars are evidence tools, not production fixes

The network debug command page is broad and maintained beyond the target branch. Use it as a menu of possible diagnostics, then query the target build for actual availability. [SRC-NET-016]

High-value families to know:

| Diagnostic family | What it helps with | Caution |
|---|---|---|
| Dormancy validation/debug draw | State changes while dormant; relevancy/dormancy visibility | Can be noisy; some commands unavailable in Shipping |
| PackageMap/NetGUID logging | Unmapped references, stale object mapping, async load references | Can generate large logs and change timing |
| Push-model validation | Missing dirty marks or skipped updates | Only useful if push model is enabled and target command exists |
| Subobject comparison/debug | Registered-list versus legacy divergence, deprecated subobject paths | Version-sensitive; use in dev/test builds |
| Fast Array limits/debug | Large change/delete counts, delta behaviour, skipped undirtied arrays | API and CVar names must be checked |
| Packet lag/loss/saturation | Reliability, ordering, queue, timeout and channel behaviour under stress | Simulation is not a substitute for real network tests |

### Debugging workflow: replicated subobject is null on client

1. Confirm the owning Actor replicates and is relevant to the client.
2. Confirm the subobject is created on the server, not only locally.
3. Confirm the UObject supports networking and has replicated properties registered.
4. If dynamic, confirm the replicated pointer can legitimately be null until the subobject bunch maps.
5. Confirm the subobject is registered with `AddReplicatedSubObject` or the target legacy path.
6. Confirm default subobject names are stable if relying on stable references.
7. Confirm removal/unregistration did not race with GC or dormancy.
8. Check NetGUID/unmapped-reference logs and relevant subobject debug CVars in the target build.
9. Add an `OnRep` for pointer/reference changes and update client-side registered lists where required.
10. Test join-in-progress, dormancy wake, owner change, destruction and replay if supported.

### Debugging workflow: Fast Array item did not update

1. Confirm mutation happened on the authoritative server copy.
2. Confirm the item identity/key is stable.
3. Confirm the mutation path marks the item or array dirty in the target API.
4. Confirm the owning object/Actor is relevant and not dormant.
5. Confirm push model, if enabled, marks the property dirty as required.
6. Confirm add/change/remove callbacks are idempotent and not treating reorder as identity.
7. Check change/delete limits and whether many changes are coalesced.
8. Compare bytes/frequency against naive replication to prove the complexity pays for itself.

### Debugging workflow: Replication Graph scale issue

1. First prove the bottleneck is server replication CPU/list construction or bandwidth, not gameplay Tick, physics, rendering or packet loss.
2. Capture Actor/property/RPC frequency and per-connection audiences.
3. Classify Actors by relevance policy: spatial, owner, team, always, dormant/static, carried/attached, temporary.
4. Build the simplest node policy and instrument actor counts per node and per connection.
5. Add movement/reclassification tests so Actors leave stale buckets.
6. Test worst-case hotspots, teleports, high-speed projectiles, spectators and team reveal/expiry.
7. Compare against generic replication with update-rate/dormancy/relevancy tuning.

### Interview answer patterns

**How would you replicate an inventory?**

"The server owns item instances and stable item IDs. If the inventory is small, I replicate a compact state struct or owner-only array and handle UI through OnRep. If item count/change rate is high, I consider Fast Array so add/change/remove deltas are explicit. I keep mutations behind server methods that mark dirty. I do not send every equip/use as a reliable RPC log; the server changes durable state and the client reacts. If items have nested state, I decide whether it belongs in the item struct, a replicated subobject, or a separate Actor. I test join-in-progress, reorder, item removal, stale UI selection and packet loss."

**How would you replicate modular equipment with subobjects?**

"I start with the owning Character/PlayerState/Inventory Actor that provides network context. Static equipment slots can be default subobjects or fixed structs. Runtime-created equipment state can be a dynamic replicated UObject subobject if it needs nested replicated fields but not Actor identity. I register it, replicate a pointer only if clients need a direct reference, and remove it from the registered list before deletion. I prefer Actor/Component RPC routing unless subobject RPC routing has a strong reason and target-branch tests."

**When would you use Replication Graph?**

"Only after profiling shows generic replication relevance/list construction is a server CPU problem at the target Actor/connection scale. I would group Actors by spatial zones, always-relevant globals, team/owner policy and dormant/static classes, then instrument per-node/per-connection list sizes. I would not use it just because the project is multiplayer or because one Actor replicates too much."

**How do you talk about Iris honestly?**

"Iris is an opt-in newer replication system, experimental in the docs I checked, and it changes implementation details. I know it requires registered subobjects for subobject replication and additional fragment registration for some custom UObjects. I would verify project config, target branch APIs and migration constraints before giving implementation details."

### Specialist common bugs

1. Replicated component exists but its component replication flag is off.
2. Dynamic component is created on the client, so it never becomes the authoritative replicated component.
3. Component subobject condition is valid, but the component itself does not replicate to that connection.
4. Dynamic subobject pointer is assumed valid before the subobject is mapped on the client.
5. Registered subobject is GC'd after deletion but not removed from the raw registered list.
6. Subobject RPC is declared but no callspace/remote routing path exists.
7. Fast Array item identity changes or is reused, so client callbacks apply to the wrong presentation.
8. Mutation path forgets dirty marking under Fast Array or push model.
9. Replication Graph node keeps Actors in stale spatial/team lists.
10. Always-relevant graph node becomes a dumping ground and destroys scalability.
11. Iris-specific requirement is copied into a Generic project or vice versa.
12. Debug CVar from maintained docs is unavailable or different in the target branch.

### Hands-on extension for Project 3

Add a specialist replication branch to Project 3:

1. Replicate a static Actor Component with one replicated property and one component RPC; measure/log the route.
2. Create a server-only dynamic component and prove client creation/destruction convergence.
3. Add one dynamic replicated UObject subobject with a replicated pointer and field; test null-before-mapped behaviour.
4. Implement safe registration/removal and force GC after removal to catch stale-list mistakes.
5. Implement one small Fast Array inventory or status-list and compare bytes/behaviour against whole-array replication.
6. If project branch supports it, enable push-model validation in a dev build and deliberately miss a dirty mark.
7. Prototype a tiny Replication Graph only if Actor/connection count is high enough to make list construction visible; otherwise write a rejection memo with profiler evidence.
8. Treat Iris as an awareness/config check unless the target branch/project is actually using it.

Evidence required:

- dedicated server plus two remote clients;
- join-in-progress;
- packet lag/loss;
- dormancy wake-before-change;
- NetGUID/unmapped-reference or equivalent logs for subobject mapping;
- Network Profiler/Insights byte comparison for Fast Array versus simple replication;
- source/version note for every CVar/API copied from docs.

### Source and version caveats

- `SRC-NET-012` and `SRC-NET-016` are maintained pages accessed beyond the target version navigation. Their conceptual warnings are valuable, but signatures/CVars must be pinned in UE5.3-UE5.6 source.
- `SRC-NET-014` labels Iris experimental/opt-in. Do not present Iris internals as baseline UE5.6 knowledge unless the interview role/project asks for it.
- `SRC-NET-015` is an API anchor with limited rendered content through the docs site. Use target headers/source before writing implementation code.

## Cluster 3 - CharacterMovement Saved Moves, Prediction Extension, and Lag Compensation

This cluster is specialist networking depth. The goal is not to memorise private engine internals; it is to reason correctly about the prediction contract and to know where custom gameplay state must enter that contract. CharacterMovement already implements a sophisticated client prediction, server reproduction, correction, replay and smoothing pipeline. Custom sprint, dash, wall-run, mantle, slide or stamina-limited acceleration bugs usually come from changing movement state outside that pipeline and then being surprised when the server corrects the client. [SRC-NET-009] [SRC-NET-017]

### Prediction is a contract, not permission to own truth

For an owning player Character, responsiveness comes from local prediction:

```text
Owning client samples input and local movement state
    -> packages a move
    -> simulates immediately for presentation
    -> stores the move for possible replay
    -> sends compact move data to the server
Server receives the move
    -> reconstructs movement-relevant state
    -> simulates authoritatively
    -> accepts or corrects
Owning client receives result
    -> discards acknowledged moves
    -> applies correction when needed
    -> replays unacknowledged moves
Remote clients
    -> see simulated proxies and smooth/interpolate replicated movement
```

That design gives local feel while keeping the server authoritative. The client can predict "I am sprinting now" or "my dash movement mode is active" only if the server receives enough move data to reproduce the same rule. It cannot predict "I hit the enemy and dealt damage" as final truth. Damage, pickups, ammo spend, cooldown consumption and score remain server decisions or server-confirmed results. [SRC-NET-001] [SRC-NET-009]

Interview-safe language:

> CharacterMovement prediction lets the autonomous proxy simulate local movement immediately, but the server still validates and reproduces the move. If I add custom movement-affecting state, I must save, compress, restore and replay it with the move so the server and client simulate the same ability. If I only replicate a bool or set the transform directly, I have built a correction generator, not prediction.

### What belongs in a saved move

A saved move should contain any small piece of state that changes the movement simulation for that move. Examples:

| State | Saved-move candidate? | Reason |
|---|---|---|
| Wants to sprint | Yes, often one compressed flag | Changes max speed/acceleration for this input sample. |
| Wants to crouch/jump | Usually built-in path first | CharacterMovement already has supported pathways; reuse before extending. |
| Dash requested this frame | Yes, with sequence/cooldown validation | A one-frame impulse or custom mode must replay exactly. |
| Wall-run side / surface normal | Maybe, but be careful | If derived from collision consistently, derive on both sides; if not, compact enough evidence must be validated. |
| Stamina display | Usually no | Presentation/UI can follow replicated or predicted state; it should not affect movement unless stamina gates speed. |
| Weapon cosmetic state | No | Not part of movement simulation. |
| Final hit result | No | Server should validate hit/damage separately. |

The rule is: if it affects velocity, acceleration, movement mode, root-motion consumption, collision response or input interpretation during movement simulation, it probably needs to be represented in the prediction path or derived identically on client and server. [SRC-NET-009] [SRC-NET-018]

### Custom movement extension checklist

Before extending saved moves, answer these design questions:

1. Is there an existing CharacterMovement feature that already models the behaviour?
2. Is the movement deterministic enough between client and server for prediction/replay?
3. Which fields change movement calculation for one move?
4. Which fields are derivable from authoritative world queries and which must be sent?
5. Which fields are cosmetic and should not enter the saved move?
6. Can adjacent moves with the same state combine safely?
7. What server validation rejects impossible speed, timing, surface, cooldown or stamina claims?
8. What correction metrics prove the implementation improved rather than hid the problem?

Good custom movement avoids bloating every move with rich gameplay state. Use compressed flags and compact custom data for small inputs; use authoritative server state for durable gameplay facts; use replicated properties for long-lived state that late joiners need. Saved moves are a high-frequency protocol surface, so every bit and branch matters.

### Schematic saved-move extension

The exact signatures, packed move pathways and helper APIs are target-branch sensitive. Treat this as shape, not copy-paste code. Pin against `CharacterMovementComponent.h/.cpp` in the project branch before implementing. [SRC-NET-017] [SRC-NET-018] [SRC-NET-019]

```cpp
// Schematic only: verify exact signatures and compressed flag availability
// in the target UE5.3-UE5.6 branch before copying.

class UMyCharacterMovementComponent : public UCharacterMovementComponent
{
    GENERATED_BODY()

public:
    bool bWantsToSprint = false;

    virtual void UpdateFromCompressedFlags(uint8 Flags) override
    {
        Super::UpdateFromCompressedFlags(Flags);
        bWantsToSprint = (Flags & FSavedMove_Character::FLAG_Custom_0) != 0;
    }

    virtual float GetMaxSpeed() const override
    {
        const float Base = Super::GetMaxSpeed();
        return bWantsToSprint ? Base * SprintMultiplier : Base;
    }

    virtual FNetworkPredictionData_Client* GetPredictionData_Client() const override;

private:
    float SprintMultiplier = 1.35f;
};

class FSavedMove_MyCharacter final : public FSavedMove_Character
{
public:
    uint8 bSavedWantsToSprint : 1;

    virtual void Clear() override
    {
        Super::Clear();
        bSavedWantsToSprint = false;
    }

    virtual uint8 GetCompressedFlags() const override
    {
        uint8 Result = Super::GetCompressedFlags();
        if (bSavedWantsToSprint)
        {
            Result |= FLAG_Custom_0;
        }
        return Result;
    }

    virtual bool CanCombineWith(
        const FSavedMovePtr& NewMove,
        ACharacter* Character,
        float MaxDelta) const override
    {
        const FSavedMove_MyCharacter* Other =
            static_cast<const FSavedMove_MyCharacter*>(NewMove.Get());

        if (bSavedWantsToSprint != Other->bSavedWantsToSprint)
        {
            return false;
        }

        return Super::CanCombineWith(NewMove, Character, MaxDelta);
    }

    virtual void SetMoveFor(
        ACharacter* Character,
        float InDeltaTime,
        FVector const& NewAccel,
        FNetworkPredictionData_Client_Character& ClientData) override
    {
        Super::SetMoveFor(Character, InDeltaTime, NewAccel, ClientData);

        const UMyCharacterMovementComponent* MoveComp =
            Cast<UMyCharacterMovementComponent>(Character->GetCharacterMovement());
        bSavedWantsToSprint = MoveComp && MoveComp->bWantsToSprint;
    }

    virtual void PrepMoveFor(ACharacter* Character) override
    {
        Super::PrepMoveFor(Character);

        UMyCharacterMovementComponent* MoveComp =
            Cast<UMyCharacterMovementComponent>(Character->GetCharacterMovement());
        if (MoveComp)
        {
            MoveComp->bWantsToSprint = bSavedWantsToSprint;
        }
    }
};

class FNetworkPredictionData_Client_MyCharacter
    : public FNetworkPredictionData_Client_Character
{
public:
    explicit FNetworkPredictionData_Client_MyCharacter(
        const UCharacterMovementComponent& ClientMovement)
        : FNetworkPredictionData_Client_Character(ClientMovement)
    {
    }

    virtual FSavedMovePtr AllocateNewMove() override
    {
        return FSavedMovePtr(new FSavedMove_MyCharacter());
    }
};
```

The important ideas in the sample:

- `SetMoveFor` captures the movement-affecting state when the input sample is made.
- `GetCompressedFlags` sends compact state with the move.
- `UpdateFromCompressedFlags` reconstructs that state for simulation.
- `PrepMoveFor` restores the saved state before replay.
- `CanCombineWith` prevents move compression from erasing a state transition.
- `GetMaxSpeed` uses the same state on client and server so prediction can match authority.

Common missing line: `CanCombineWith`. If a sprint press/release or dash start is merged into a neighbouring move with different state, the client and server disagree at the exact transition point. The result can look like random micro-corrections instead of an obvious state bug.

### Custom dash, slide, mantle and wall-run patterns

**Predicted sprint**

Sprint is a good first extension because the movement-affecting state is small. The client predicts the input flag; the server validates stamina, gameplay tags, weapon state, debuffs and movement mode before accepting the speed. If stamina can run out mid-sprint, the server must be able to reproduce the same stamina gate or correct the client quickly. Do not drive sprint only from a replicated `bIsSprinting` that arrives later; that makes responsiveness depend on round trip time.

**Predicted dash**

Dash is harder because it is often a one-shot impulse, montage or custom movement mode. Use an input edge/sequence, cooldown window, direction and compact movement state. Server validation should reject dash while stunned, on cooldown, without charge, with impossible direction, or from impossible floor/wall context. If the dash depends on an animation montage/root motion, define whether animation drives movement, CharacterMovement drives movement, or a root-motion source is integrated into the prediction path. Mixing independent transform changes with CharacterMovement correction is a common jitter source. [SRC-ANIM-014] [SRC-NET-009]

**Slide and wall-run**

Slide and wall-run often depend on floor/wall queries. Prefer deriving the surface on client and server from the same movement/collision state. If a wall normal, ledge ID or surface handle cannot be reproduced reliably, send compact evidence and validate it on the server. Do not let the client choose arbitrary geometry facts. Store only what the move needs for simulation and validation; leave camera tilt, sound, VFX and UI outside the saved move.

**Mantle**

Mantle usually crosses movement, animation, collision and authority boundaries. A safe design separates:

- input intent: client requested mantle;
- evidence: ledge candidate, start transform, timestamp/sequence, movement mode;
- validation: server confirms reachability, collision clearance and state;
- movement: predicted root-motion source, custom mode or server-driven correction;
- presentation: montage/VFX driven by prediction key or replicated state.

If the project cannot make mantle prediction stable, an acceptable tradeoff is local anticipation plus server-confirmed movement, but the interview answer should name the latency/feel cost.

### Direct transform writes and root motion fights

CharacterMovement owns a Character capsule through sweep-based movement, floor logic, movement modes and network prediction. Repeated `SetActorLocation`, physics impulses on the wrong body, or animation root motion that also displaces the capsule can create a three-way disagreement:

```text
Client local presentation says: the capsule moved here.
Saved move says: movement input would have moved it there.
Server reproduction says: the legal authoritative result is somewhere else.
```

The correction system is then doing its job. The bug is not "network jitter"; it is an ownership conflict. Pick one movement owner for the interval, integrate that owner into prediction, and make other systems consume that result. [SRC-ANIM-009] [SRC-NET-009]

### Debugging workflow: predicted sprint jitters

1. Reproduce on dedicated server with a remote client and artificial lag/loss.
2. Log sprint input edge, local role, movement mode, max speed, acceleration and compressed flags on client and server.
3. Count corrections while sprint toggles; compare against idle/walk baseline.
4. Confirm the sprint state is captured in the saved move and restored before replay.
5. Confirm `CanCombineWith` prevents combining moves with different sprint state.
6. Confirm server validation and client prediction use the same stamina/tag/cooldown gates or a deliberately documented correction policy.
7. Remove cosmetic camera/FOV/VFX changes from the diagnosis path; they can hide the actual capsule correction.
8. Use movement visualisation/logging and Network Profiler/Insights to correlate input, move send, correction and replay. [SRC-NET-010] [SRC-PERF-003]

Do not start by making movement replication more reliable or increasing update rate. A saved-move mismatch is a deterministic protocol bug; bandwidth changes may only make it less visible.

### Debugging workflow: custom dash rubber-bands

Use this hypothesis chain:

```text
Was the dash initiated by the owning client?
  -> Did the saved move capture the dash edge and direction?
  -> Did the server reconstruct exactly one dash, not zero or two?
  -> Was dash state restored for client replay?
  -> Was the move allowed to combine across the dash edge?
  -> Did server validation reject the dash for a real gameplay reason?
  -> Did animation/root motion/physics move the capsule outside CharacterMovement?
  -> Did the client predict through geometry the server considered blocked?
```

Record at least one replay trace showing the unacknowledged move list before/after correction. In an interview, mention the difference between fixing a bad prediction input and masking a valid correction. If the server rejects an illegal dash, the correction is expected and the UX should communicate the rejection cleanly.

### Lag compensation and server rewind

Lag compensation solves a different problem from movement prediction. Prediction is about making the local player feel responsive before the server responds. Lag compensation is about evaluating an action, often a shot or melee query, against the world state that the player reasonably saw when they acted.

A common authoritative hitscan flow:

```text
Client fires locally for responsiveness
    -> records local fire sequence, camera transform, aim direction, local timestamp
    -> sends fire request/evidence to server through an owned RPC
Server receives request
    -> validates fire rate, ammo, weapon state, player state, timestamp window and aim plausibility
    -> finds historical snapshots around the claimed shot time
    -> rewinds or queries hit-relevant target bounds/poses
    -> performs authoritative trace/test
    -> applies damage or rejection
    -> replicates durable result and plays presentation
```

The server is not trusting the client's hit result. It is using client-provided timing/aim evidence within bounds, then running its own hit test against a server-maintained history. [SRC-NET-001] [SRC-NET-006]

### Snapshot history design

A practical server rewind buffer is a bounded, per-Actor or central ring buffer of hit-relevant snapshots:

```cpp
struct FHitboxSnapshot
{
    double ServerTimeSeconds = 0.0;
    uint32 Sequence = 0;
    FTransform RootTransform;
    TArray<FTransform> HitboxLocalToWorld; // Or compact named boxes/bones.
    FVector LinearVelocity;
    uint8 MovementMode = 0;
};

class FRewindHistory
{
public:
    void Push(const FHitboxSnapshot& Snapshot);

    bool Sample(double QueryTime, FHitboxSnapshot& OutA, FHitboxSnapshot& OutB) const;

private:
    TArray<FHitboxSnapshot> Ring;
    int32 WriteIndex = 0;
    int32 NumValid = 0;
};
```

The useful algorithmic property is predictable bounded storage and `O(1)` steady-state append. Query can be linear for small buffers, binary search if snapshots are time-ordered in a contiguous view, or indexed by sequence/time if scale demands it. Choose from measured target player count, tick rate, retained milliseconds, hitbox count and query frequency. [SRC-ALG-002] [SRC-PERF-006]

Memory budget example:

```text
64 players
20 snapshots per second
250 ms history = 5 snapshots/player minimum
round up to 16 snapshots/player for jitter and interpolation
32 hitboxes * 48 bytes compact transform ~= 1536 bytes/snapshot
64 * 16 * 1536 ~= 1.5 MB before arrays/metadata/alignment
```

That is acceptable for many games; full skeletal pose, long windows, high tick rates or multiple rewind layers can become expensive. The design should store exactly what hit validation needs, not entire Actors.

### Rewind validation boundaries

Lag compensation can improve fairness for the shooter but harm fairness for the target if the window is too generous. Define policy:

| Policy | Recommended guard |
|---|---|
| Maximum rewind window | Clamp to a small, design-approved latency plus jitter budget. |
| Timestamp trust | Convert client timing into server time through connection timing/sequence evidence; reject impossible values. |
| Fire rate | Validate on server using authoritative weapon state. |
| Ammo/cooldown | Server owns consumption; client prediction is provisional presentation. |
| Aim evidence | Validate view direction, muzzle/camera relation and angle deltas. |
| Geometry | Consider current/past occlusion policy; avoid letting rewind shoot through a door that closed before the server processed the request unless the game deliberately permits it. |
| Target state | Invulnerability, death, team changes and respawn need timestamped policy too. |
| Abuse detection | Log repeated extreme-window hits, impossible aim deltas and sequence gaps. |

The interview answer should be honest: server rewind is a game-design tradeoff, not a universal good. Tactical shooters, melee games, fighting games and co-op PvE often choose different windows and validation strictness.

### Rollback, rewind, interpolation and reconciliation are different words

Use these definitions carefully:

| Term | Meaning in a UE interview |
|---|---|
| Client prediction | Local client simulates likely outcome before server confirmation. |
| Reconciliation | Client applies server correction and replays unacknowledged inputs/moves. |
| Interpolation / smoothing | Remote proxies present received state smoothly between updates. |
| Server rewind / lag compensation | Server evaluates an action against stored past hit-relevant state. |
| Rollback | A broader model where simulation returns to an earlier state and resimulates forward, common in deterministic fighting/gameplay models or specialised network prediction systems. |

Do not say "rollback" when you mean "CharacterMovement correction" or "hitscan rewind." They share history and replay vocabulary, but the engineering constraints differ. Full rollback needs deterministic or controlled simulation state, input logs, state snapshots, side-effect management, resimulation windows and presentation correction. CharacterMovement prediction is a specialised, mature movement pipeline; generic game-state rollback is a separate architecture.

### Lag-compensated hitscan workflow

1. Client predicts muzzle flash, recoil and local tracer with a local sequence ID.
2. Client sends owned Server RPC with weapon ID, sequence, local fire timestamp or tick, view transform/aim evidence and random seed if spread is predicted.
3. Server validates ownership, weapon state, fire rate, ammo, tags, alive state and timestamp window.
4. Server maps the client timestamp to a bounded server query time.
5. Server samples/interpolates target snapshots around that time.
6. Server performs trace/intersection against rewind hitboxes, not client-reported hit result.
7. Server applies authoritative damage and consumes ammo/cooldown.
8. Server replicates damage/death and sends any private correction/rejection to the shooter.
9. Clients reconcile presentation: remove false local hit marker, confirm hit marker, or play impact from server result.

Common bugs:

- Client sends hit Actor and damage as truth.
- Rewind history stores current transform references instead of value snapshots.
- Query window is unbounded, so high-latency or manipulated clients gain unfair power.
- Hitboxes are rewound but world blockers are tested in current time with no policy.
- Dead/respawned targets are hit because snapshot lifetime crosses identity boundaries.
- Server-side random spread does not match client-predicted presentation.
- History arrays allocate every frame.
- Snapshot time uses mixed client/server clocks without conversion.

### Melee and projectile lag compensation

Hitscan rewind is the clean teaching case. Melee and projectiles need stricter design:

- **Melee:** query volumes often persist across animation windows. Store attack start/end sequence and server-authoritative montage/window timing. Rewind target hitboxes only within a small window, and validate attacker position/range at the relevant time. Be careful with root motion and capsule ownership.
- **Fast projectile:** can be represented as server-authoritative projectile simulation with client presentation, or as a hitscan/segmented sweep with lag-compensated origin and target state. Decide whether projectile travel is part of fairness or just visual.
- **Slow projectile:** usually should be a server Actor or server simulation state. Rewinding targets to when the client clicked can be unfair if the projectile visibly travels for everyone else.

The design question is what the player was expected to dodge: the click, the swing window, the projectile body, or the authoritative server event.

### Profiling workflow for movement prediction and rewind

Measure separately:

1. Corrections per minute per movement feature.
2. Correction distance and correction cause tags.
3. Saved moves allocated, combined and replayed.
4. Bytes per movement update and custom flag/data payload.
5. Server time spent in movement simulation for characters.
6. Rewind snapshot memory and allocation rate.
7. Rewind query count, trace count and time per weapon fire.
8. Network traffic for fire requests, confirmations and replicated damage/death.
9. UX metrics: false local hit markers, rejected predicted actions and visible correction frequency.

Use Network Profiler/Insights for traffic, Unreal Insights/Stats for CPU/memory, and targeted logs or debug CVars for hypothesis proof. Do not rely on "felt smoother on LAN." [SRC-NET-010] [SRC-NET-016] [SRC-PERF-003] [SRC-PERF-004]

### Strong interview answer patterns

**How does CharacterMovement prediction work?**

"The owning client creates and stores moves from input, simulates immediately, sends compact move data to the server, and keeps unacknowledged moves for replay. The server reproduces and validates movement, then acknowledges or corrects. On correction, the client moves to the server state and replays pending moves; simulated proxies smooth replicated movement. Custom movement state must be saved and restored in that pipeline, otherwise the server and client simulate different inputs."

**How would you implement predicted sprint?**

"I would add a movement-component state such as wants-to-sprint, capture it in a saved move, encode it in a custom compressed flag, restore it before replay and update the component from compressed flags on the server path. I would prevent move combining across sprint transitions and validate stamina/tags/server state. I would measure correction rate under packet lag/loss instead of only checking that a bool replicated eventually."

**How would you implement lag-compensated hitscan?**

"The client can predict presentation and send fire intent with sequence, timing and aim evidence through an owned Server RPC. The server validates weapon state, rate, ammo, tags and a bounded timestamp, samples server-stored target hitbox snapshots around the query time, runs the authoritative trace, then applies damage and replicates the result. I would clamp rewind windows, store compact value snapshots, define occlusion and respawn policy, and log extreme-window hits for abuse detection."

**Prediction versus rollback?**

"Prediction is local speculation for responsiveness. Reconciliation applies authoritative correction and replays pending inputs. Interpolation smooths remote state. Server rewind evaluates an action against stored past state. Rollback is a broader architecture that restores old simulation state and resimulates forward; it needs deterministic state, input logs and side-effect control. I would not call CharacterMovement correction generic rollback."

### Hands-on extension for Project 3

Add a movement prediction and lag-compensation branch:

1. Implement predicted sprint with one custom saved-move flag.
2. Add a predicted dash or slide with an input edge, compact direction/state, server validation and a clear rejection path.
3. Instrument corrections by feature, distance and reason.
4. Add artificial lag/loss and compare correction rate before/after saved-move integration.
5. Implement a server-side snapshot ring buffer for hit-relevant target boxes.
6. Add a hitscan weapon request that sends intent/timing/aim evidence, not damage truth.
7. Validate fire rate, ammo, timestamp window, aim plausibility, target state and occlusion policy.
8. Compare no lag compensation, bounded rewind and overly generous rewind with recorded fairness examples.
9. Profile snapshot memory, allocation rate, trace cost and network traffic.
10. Write a one-page design memo explaining why your rewind window is fair for this game's weapon speeds and player expectations.

Evidence required:

- dedicated server plus two remote clients;
- packet lag/loss/jitter;
- before/after correction counts for custom movement;
- target-branch saved-move signature notes;
- one rejected predicted action with clean UX;
- one local predicted hit marker that is later rejected by authority;
- Network Profiler/Insights traffic capture;
- memory/CPU measurement for history buffers and rewind traces.

### Specialist common bugs

1. Custom movement-affecting state is replicated as a delayed bool instead of saved with the move.
2. `CanCombineWith` allows a sprint/dash transition to be merged away.
3. Replay restores only built-in state; custom flags remain at current presentation value.
4. Server validates different stamina/cooldown/tag rules from the client prediction path.
5. Dash changes transform directly instead of going through movement simulation.
6. Root motion, physics and CharacterMovement all move the capsule in the same interval.
7. Correction logs are treated as noise instead of evidence.
8. Rewind history stores pointers/references to mutable current transforms.
9. Lag compensation trusts client hit results or unbounded client timestamps.
10. Rewind hitboxes ignore respawn/team/invulnerability state transitions.
11. Projectile fairness model is copied from hitscan without considering travel time.
12. Snapshot buffers allocate or copy large pose data every frame without a memory budget.

### Source and version caveats

- `SRC-NET-009` is the core conceptual source for CharacterMovement prediction/correction/replay/smoothing.
- `SRC-NET-017`, `SRC-NET-018` and `SRC-NET-019` are API anchors. The Epic API site can be dynamically rendered and exact signatures/packed move internals must be checked in target source.
- Lag compensation/history-buffer design in this section is synthesis from UE networking authority/RPC principles plus standard data-structure/performance reasoning. It is not presented as a one-size-fits-all Epic subsystem.
- Full rollback networking is specialist architecture beyond baseline UE generic replication. Label any project-specific Network Prediction plugin or custom rollback implementation separately before making implementation claims.
- For plugin-specific rollback proof, use [NetworkPrediction plugin and full rollback proof](ue_network_prediction_rollback_proof.md) plus the workbook in `practice/ue_network_prediction_rollback_workbook.md`.
- For a branch-pinned proof packet that ties saved moves, subobjects, Fast Array, RepGraph, Iris and NetworkPrediction together, use [Networking target-branch implementation proof](ue_networking_target_branch_proof.md) and `practice/ue_networking_target_branch_proof_workbook.md`.
