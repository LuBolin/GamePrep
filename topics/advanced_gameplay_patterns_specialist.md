# Advanced Gameplay Patterns Specialist Pass

This chapter deepens the first-pass architecture/patterns material. It focuses on the practical contracts that make gameplay systems survive multiplayer, save/load, UI prediction, content iteration and production debugging.

Source scope: game-pattern references, UE framework/components/subsystems/delegates, Data Assets/Asset Manager/Data Validation, property specifiers and networking references already in the source index. [SRC-ARCH-001] [SRC-ARCH-003] [SRC-ARCH-004] [SRC-ARCH-006] [SRC-ARCH-007] [SRC-ARCH-009] [SRC-ARCH-011] [SRC-ARCH-012] [SRC-EPIC-012] [SRC-EPIC-017] [SRC-EPIC-028] [SRC-EPIC-035] [SRC-NET-001] [SRC-NET-004]

## 1. The specialist upgrade: every pattern needs an operation contract

The baseline pattern question is "which pattern fits?" The specialist question is "what does this operation guarantee under failure?"

For any gameplay system, define:

| Contract | Question |
|---|---|
| Identity | What stable ID identifies the player, item, objective, world object, transaction or command? |
| Authority | Who may commit state? Who may request, predict or observe? |
| Version | What revision/schema/content version proves the request matched the state it was based on? |
| Idempotency | What happens if the same request arrives twice or a response is replayed? |
| Atomicity | Which fields must change together or not at all? |
| Notification | Which observers receive local events, replicated state, queued messages or UI deltas? |
| Persistence | What is saved, migrated, ignored, rebuilt or invalidated? |
| Recovery | What happens after disconnect, travel, reload, missing asset, stale target or partial failure? |
| Evidence | Which operation ID/log fields let you reconstruct the bug? |

Patterns are vocabulary. These contracts are the production system.

## 2. Transactional gameplay operations

A transaction is a validated operation that moves a system from one coherent revision to another. It is not necessarily a database transaction; it is an engineering promise that partial side effects are controlled.

### Inventory move example

Bad shape:

```cpp
// Hidden partial states: removes first, then may fail to add.
Source.Remove(ItemId, Count);
Target.Add(ItemId, Count);
```

Better shape:

```cpp
// Schematic value-level transaction.
struct FInventoryMoveRequest
{
    FGuid OperationId;
    FInventoryId SourceContainer;
    FInventoryId TargetContainer;
    FItemInstanceId Item;
    int32 Count = 1;
    int32 ExpectedSourceRevision = INDEX_NONE;
    int32 ExpectedTargetRevision = INDEX_NONE;
};

struct FInventoryMoveResult
{
    FGuid OperationId;
    bool bCommitted = false;
    FName RejectionReason;
    int32 NewSourceRevision = INDEX_NONE;
    int32 NewTargetRevision = INDEX_NONE;
    TArray<FInventoryDelta> Deltas;
};

FInventoryMoveResult TryMoveItem(const FInventorySnapshot& Source,
                                 const FInventorySnapshot& Target,
                                 const FInventoryMoveRequest& Request)
{
    FInventoryMoveResult Result;
    Result.OperationId = Request.OperationId;

    if (Request.ExpectedSourceRevision != Source.Revision ||
        Request.ExpectedTargetRevision != Target.Revision)
    {
        Result.RejectionReason = "RevisionMismatch";
        return Result;
    }

    // Validate count, ownership, locks, capacity, stack compatibility and item state.
    // Build deltas first. Apply only if every invariant passes.
    Result.Deltas = BuildMoveDeltas(Source, Target, Request);
    if (!ValidateDeltas(Source, Target, Result.Deltas, Result.RejectionReason))
    {
        return Result;
    }

    Result.bCommitted = true;
    Result.NewSourceRevision = Source.Revision + 1;
    Result.NewTargetRevision = Target.Revision + 1;
    return Result;
}
```

The important part is not the exact container type. The important part is that validation happens before mutation, the commit result carries revisions/deltas, UI can reconcile by operation ID, and duplicate/reordered responses are harmless. [SRC-ARCH-004] [SRC-NET-001]

### When to use transactional design

Use it when operations can fail after several checks or when multiple systems must agree:

- inventory move/split/merge/equip/trade;
- shop purchase and currency spend;
- quest reward grant;
- crafting consume/produce;
- save/load restore;
- party invite/session join;
- world-state phase change;
- ability cost plus target application when not using GAS for that policy.

Do not overbuild tiny local cosmetic operations. Transaction machinery is justified by conflict, persistence, authority, rollback/retry or audit pressure.

## 3. Operation IDs, revisions and idempotency

### Operation ID

An operation ID identifies one user/system intention. It lets you correlate request, validation, commit, replication, UI prediction and telemetry.

Use it for:

- client drag/drop inventory prediction;
- quest reward grant;
- save migration step;
- async purchase flow;
- editor batch repair;
- multiplayer trade confirmation.

### Revision

A revision identifies the state version the operation expected. It solves "user acted on stale UI" and "two requests raced."

Examples:

- inventory container revision;
- quest objective revision;
- equipment loadout revision;
- save schema version;
- world phase revision;
- replicated Fast Array item replication ID/change key where using that pattern.

### Idempotency

A request is idempotent if repeating it does not double-apply the outcome. In gameplay systems, this often means:

- server remembers recent committed operation IDs per player/session;
- duplicate request returns the same result or a safe "already committed";
- UI discards stale responses if its local pending operation was superseded;
- save migration records completed migration version;
- reward grant checks a durable entitlement/claim record.

Interview trap: reliable RPC delivery is not idempotency. Reliability says transport tries to deliver; idempotency says duplicate or replayed intent cannot double-spend/double-reward.

## 4. Events are not durable truth

Events tell consumers something happened. Durable state answers what is true now. Confusing them causes late-join, save/load and subscription-order bugs. [SRC-ARCH-003] [SRC-ARCH-006]

### Event-only quest bug

Bug:

1. Player picks up ore before accepting quest.
2. Quest subscribes to `OnItemAdded`.
3. Quest requires owning 10 ore.
4. Player already owns 10 ore but receives no event after subscription.

Fix:

- quest activation queries current inventory snapshot;
- then subscribes to committed inventory deltas;
- every delta carries operation ID, item ID, old/new quantity and revision;
- save stores quest progress/state, not the event stream unless event sourcing is explicitly designed.

### Event payload strength

A weak event says:

```text
OnInventoryChanged()
```

A useful event says:

```text
OperationId, ContainerId, BeforeRevision, AfterRevision,
Deltas[{ItemInstanceId, DefinitionId, OldQuantity, NewQuantity, Reason}]
```

If every listener immediately queries five fields from the publisher, the event probably lacks a complete payload or snapshot.

## 5. Queues and buses: design delivery semantics

Event queues and buses are dangerous when the schema is vague and the processing phase is implicit. Design:

- delivery phase: immediate, end-of-frame, fixed-step, async worker, next server tick;
- order scope: global, per actor, per player, per container, per topic;
- capacity: unbounded, bounded drop-old, bounded reject-new, coalesce, back-pressure;
- stale target handling: weak object check, stable ID resolve, rejection event;
- duplicate handling: operation ID or sequence;
- thread rules: payload ownership and UObject access restrictions;
- observability: topic, sequence, queue depth, process time, dropped/coalesced count.

### Good use cases

- batching inventory/UI deltas at end of frame;
- collecting many AI stimuli into a bounded perception update;
- editor validation result streams;
- telemetry/logging that must not block gameplay;
- decoupling content event producers from non-critical analytics.

### Bad use cases

- hiding a required synchronous answer;
- global cross-system bag with unversioned payloads;
- authoritative state mutation from arbitrary subscribers;
- pretending event order is durable save data.

## 6. State machines with side-effect ownership

A production state machine owns not only state names but also side effects.

For each state:

- enter actions;
- exit actions;
- timers/tasks/delegates started in the state;
- cancellation paths;
- allowed transitions and guards;
- authority side;
- replicated/public projection;
- save/load restoration policy;
- debug fields.

### Reload state example

`Reloading` might own:

- montage/task binding;
- ammo reservation or not;
- input buffering policy;
- interruption by sprint/swap/stun/death;
- predicted local presentation;
- authoritative ammo commit at a defined moment;
- cleanup of montage delegate and UI progress.

The bug is not "state machine missing." The bug is "the reload side effects are not owned by a terminal path."

## 7. Data definitions, runtime state and migrations

Data Assets are definitions; runtime instances carry mutable state. [SRC-ARCH-009] [SRC-ARCH-011]

Specialist rules:

- Definition IDs must be stable across rename/move where saves refer to them.
- Runtime state stores definition ID plus mutable fields, not direct loaded asset assumptions.
- Asset soft references need cook/load/residency policy.
- Validation rejects impossible content before runtime.
- Migrations are tested with old saves and old content IDs.
- Missing content has policy: fail load, substitute placeholder, refund, hide, preserve opaque record or migrate.

### Migration fixture pattern

Keep small historical fixture saves:

```text
Save_v1_item_renamed.json
Save_v2_objective_split.json
Save_v3_removed_definition.json
Save_v4_party_quest_schema.json
```

Each fixture has an expected post-load state. Add one whenever a designer-facing identity or schema changes. This is cheaper than discovering broken player saves late.

## 8. UI prediction and reconciliation

UI may predict for responsiveness; it must not become the owner of durable truth.

Pattern:

1. User drags item from slot A to B.
2. UI creates operation ID and visually marks pending state.
3. Server validates against container revisions.
4. Server commits or rejects with operation ID and new revisions.
5. UI reconciles pending state or shows rejection.
6. Full authoritative snapshot/delta can rebuild the UI at any time.

Failure cases:

- response for operation 1 arrives after operation 2;
- target slot changed on server;
- item was consumed by another system;
- UI widget was recycled/destroyed;
- player disconnected or travelled;
- server rejected but UI kept local state.

Answer pattern:

> UI can predict presentation, but every pending operation has an ID and expected revision. The server commits a delta or rejection. The view model reconciles by operation ID and can rebuild from authoritative state. Widgets are disposable projections.

## 9. Save/load as controlled restore, not normal gameplay replay

Applying normal setters during load can fire achievements, quests, replication, audio and analytics as if the player just performed actions. Use a restore mode.

Restore path:

1. load header and schema version;
2. validate file/account/platform ownership;
3. migrate records;
4. load/resolve required definitions;
5. create runtime containers/instances;
6. apply state with notifications suppressed or tagged as restore;
7. rebuild derived caches;
8. emit one "restore ready" event/snapshot;
9. validate invariants and report recoverable omissions.

Do not store live Actor/UObject pointers as durable identity. Store stable IDs and define resolve/failure policy. [SRC-EPIC-035]

## 10. Debugging workflows

### Duplicate item

1. Find operation ID and container revisions.
2. Check duplicate request/retry path.
3. Check whether validation mutated before final failure.
4. Inspect server commit result and replicated delta.
5. Verify UI did not apply local prediction twice.
6. Check save/load or migration replay path.
7. Add idempotency test for duplicate operation ID.

### Quest missed progress

1. Determine whether objective depends on current state, event delta or both.
2. Query durable state at quest activation/load.
3. Check event subscription timing.
4. Validate event payload includes ID/revision/source tags.
5. Confirm save stores objective state and migration handles definition changes.

### UI shows stale state after rejection

1. Find pending operation ID.
2. Check expected revision versus committed revision.
3. Check stale response ordering.
4. Verify view model rebuilt from authoritative state after rejection.
5. Check widget recycling/delegate cleanup.

### Save replayed rewards

1. Identify reward claim operation ID or durable claimed flag.
2. Check restore path versus normal quest completion path.
3. Confirm load applies state without firing normal grant events.
4. Add historical fixture for the schema that regressed.

## 11. Profiling advanced patterns

Measure architecture costs:

- operation count per second and commit latency;
- transaction validation cost;
- queue depth, drops, coalesces and processing time;
- delegate listener count and broadcast cascade duration;
- dirty-cache invalidation frequency and recompute cost;
- save/load migration time and allocation spikes;
- UI pending operation count and rebuild cost;
- data validation time and false positive/negative rate;
- replicated delta size and frequency.

Do not optimise away the contract first. If the queue is expensive, batch/coalesce or narrow payloads. If transactions allocate heavily, move validation to value structs or pooled temp buffers. If UI rebuilds are costly, use precise deltas and virtualised lists. Keep the operation ID/revision evidence.

## 12. Strong interview patterns

### "Design an inventory"

> I separate item definitions, item instances/stacks and containers. Mutations go through validated transaction commands with operation IDs and container revisions. Server owns truth; clients can predict UI only. Replication sends compact owner-relevant deltas; public equipped appearance is separate. Saves store stable IDs and schema version, not UObjects. Debugging starts from the operation log, not screenshots.

### "When would you use an event queue?"

> I use a queue when delayed/batched processing is part of the semantics: end-of-frame UI deltas, telemetry, perception batching or editor validation streams. I define order, capacity, stale target handling, duplicate policy, thread access and tracing. If the caller needs an immediate answer, I use a direct command/query instead.

### "How do you make save/load safe?"

> Save/load is a restore transaction. I store stable IDs and versioned state, migrate records, resolve definitions, apply state through a restore path that suppresses normal reward/combat notifications, rebuild derived caches and emit one ready snapshot. I keep old fixture saves to test migrations.

## 13. Hands-on extension

Extend Project 8 or Project 9:

1. Implement inventory move/split/equip as value-level transactions with operation IDs and revisions.
2. Add UI drag/drop prediction and reconciliation.
3. Add quest objective activation that queries current state and then observes deltas.
4. Add a bounded event queue for non-critical UI/telemetry updates and record drops/coalesces.
5. Add save fixture migration for one renamed item definition and one split objective.
6. Inject duplicate request, stale revision, missing definition, late UI response and save replay reward bugs.
7. Produce an operation log reconstruction for each injected bug.

## 14. Source map

- pattern vocabulary: [SRC-ARCH-001] [SRC-ARCH-003] [SRC-ARCH-004] [SRC-ARCH-006] [SRC-ARCH-007] [SRC-ARCH-008]
- UE components, subsystems and delegates: [SRC-EPIC-012] [SRC-EPIC-017] [SRC-EPIC-028] [SRC-ARCH-013] [SRC-ARCH-014]
- data definitions, validation and asset identity: [SRC-ARCH-009] [SRC-ARCH-010] [SRC-ARCH-011] [SRC-ARCH-012]
- persistence/specifier and networking authority: [SRC-EPIC-035] [SRC-NET-001] [SRC-NET-004]

