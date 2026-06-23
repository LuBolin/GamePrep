# GAS Specialist Workbook

See also: [[ue_gas_specialist_depth]], [[ue_gameplay_ability_system]], [[ue_networking_and_replication]], [[ue_hands_on_projects]].

These drills extend Project 9 and the GAS specialist chapter. Each lab should run on a dedicated server with one owning client and one simulated proxy unless the lab explicitly says otherwise.

Source anchors: [SRC-GAS-001] [SRC-GAS-002] [SRC-GAS-003] [SRC-GAS-004] [SRC-GAS-005] [SRC-GAS-006] [SRC-GAS-009] [SRC-GAS-011] [SRC-GAS-012] [SRC-GAS-013]

## Lab 1: ASC lifetime and respawn proof

**Goal:** prove whether the project should use a Pawn-owned or PlayerState-owned ASC for the sample character.

**Build:**

1. Implement a minimal character with Health, MaxHealth, Stamina and a predicted Dash.
2. Run the same slice with Pawn-owned ASC and PlayerState-owned ASC.
3. Log owner, avatar, controller, owning connection, actor info initialisation and current grants.
4. Kill and respawn the player twice.

**Injected failures:**

- forget to refresh actor info after respawn;
- leave old-avatar input delegates bound;
- grant startup abilities from both possession and replicated loadout update;
- preserve cooldown when design expected reset, then reset when design expected preserve.

**Acceptance criteria:**

- a table states which state persists and why;
- duplicate grant count remains zero after three respawns;
- old avatar never receives input or task callbacks after death;
- owner client can activate after respawn under lag;
- simulated proxy sees only the intended replicated state.

## Lab 2: Grant source ledger

**Goal:** make startup, equipment and temporary-effect grants removable by source without class-name assumptions.

**Build:**

1. Create three grant sources: startup, weapon, temporary buff.
2. Let two sources grant the same ability class at different levels.
3. Store source ID, ability class, level and spec handle.
4. Remove each source independently.

**Injected failures:**

- remove by class instead of handle;
- unequip during active ability;
- re-equip during prediction;
- reload loadout after possession.

**Acceptance criteria:**

- each source removes only its own grant;
- active ability removal policy is explicit and tested;
- grant rebuild is idempotent;
- log output makes duplicate or stale handles obvious.

## Lab 3: Attribute invariant matrix

**Goal:** prove Health/MaxHealth rules across every mutation path.

**Build:**

1. Add Health, MaxHealth, Shield and IncomingDamage.
2. Apply instant damage, instant heal, MaxHealth buff, MaxHealth debuff and periodic regeneration.
3. Bind a UI/view-model layer to attribute change events.

**Injected failures:**

- clamp only in one hook;
- reduce MaxHealth below current Health;
- apply damage to client UI only;
- forget to reset IncomingDamage;
- bind UI to old avatar after respawn.

**Acceptance criteria:**

- Health is always within `[0, MaxHealth]`;
- MaxHealth changes do not generate fake damage attribution;
- server, owner client, simulated proxy and UI converge;
- repeated buff/remove cycles produce no delegate growth;
- final report distinguishes base/current, predicted/final and displayed values.

## Lab 4: Execution calculation evidence

**Goal:** implement authoritative damage routing with transparent calculation evidence.

**Build:**

1. Create an aimed attack that sends target data.
2. Server derives final damage from attack power, target defence, shield, tags and set-by-caller charge.
3. Use an execution-style calculation or a well-contained authoritative equivalent.

**Injected failures:**

- missing set-by-caller tag;
- client sends final damage instead of charge/evidence;
- buff added between shot launch and impact;
- target gains immunity tag during travel;
- shield absorbs damage but death still triggers.

**Acceptance criteria:**

- log records captured values, tag gates, set-by-caller inputs and output modifiers;
- snapshot versus live capture timing is documented and tested;
- invalid or missing set-by-caller values fail safely;
- final damage is server-derived;
- designers can explain why a zero-damage hit happened.

## Lab 5: Full stacking policy

**Goal:** replace "it stacks to five" with a complete stack lifecycle.

**Build:**

1. Implement poison from two weapons and haste from one ability.
2. Poison stacks to five, tracks source, has a duration, period and overflow result.
3. Haste does not stack; stronger source replaces weaker source.
4. UI displays count, strongest source and remaining duration.

**Injected failures:**

- one source clears another source's tag;
- period resets unexpectedly;
- expiry removes all stacks instead of one;
- overflow creates a hidden sixth stack;
- UI stores stale source after effect removal.

**Acceptance criteria:**

- stack source, duration, period, overflow and expiration policy are documented;
- tag count remains correct across all removals;
- UI and gameplay agree after every stack event;
- save/travel persistence policy is explicitly "none" or implemented.

## Lab 6: Prediction rejection and cleanup

**Goal:** make a local-predicted dash reject cleanly.

**Build:**

1. Implement local-predicted Dash with stamina cost, cooldown, cue, montage and movement.
2. Record spec handle, prediction key, target/direction data, commit result and end reason.
3. Force server rejection through blocked tag, insufficient stamina, invalid direction and stale avatar.

**Injected failures:**

- cost is applied manually and by commit;
- cue plays locally and again from confirmation;
- montage continues after rejection;
- custom movement side effect is not reversible;
- ability has no terminal end path on rejection.

**Acceptance criteria:**

- owner client and server traces can be correlated by prediction key;
- every rejection returns resources/UI/montage/cue to a coherent state;
- irreversible gameplay effects never happen on the client first;
- listen-server success is not accepted without remote-client proof.

## Lab 7: Target data validation

**Goal:** treat target data as evidence rather than truth.

**Build:**

1. Add a hitscan or lock-on attack that sends target data.
2. Server validates range, line of sight, target state, faction, cadence, resources and relevant tags.
3. Optionally connect to a server-rewind history buffer if the networking project already has one.

**Injected failures:**

- client reports final damage;
- target is behind a wall on server;
- target dies before server validation;
- client fires faster than cadence;
- target actor is stale or belongs to another world.

**Acceptance criteria:**

- server derives final result from bounded evidence;
- rejection reason is visible to developer logs and owner UX where appropriate;
- validation can be replayed from a saved trace;
- terminology distinguishes target validation, server rewind and full rollback.

## Lab 8: Custom Ability Task cancellation

**Goal:** prove a custom task cannot mutate gameplay after cancellation.

**Build:**

1. Write or wrap a task that waits for release, montage event or target confirmation.
2. Bind external delegates/timers in `Activate`.
3. Unbind/cancel in task destruction.
4. Add trace rows for activation, broadcast, end and late-event rejection.

**Injected failures:**

- event arrives before delegate binding;
- ability is cancelled before event;
- owner is destroyed while task waits;
- task broadcasts twice;
- external delegate remains bound after task end.

**Acceptance criteria:**

- cancel at every wait phase;
- no callback mutates gameplay after cancel/end;
- all delegate/timer handles are released;
- completion/cancel path executes exactly once.

## Lab 9: Cue duplication and cosmetic authority

**Goal:** keep Gameplay Cues cosmetic, idempotent and resettable.

**Build:**

1. Add predicted dash cue and persistent poison cue.
2. Use cue parameters for presentation context.
3. Pool one expensive cue effect.
4. Disable cue assets in one test package.

**Injected failures:**

- cue calculates damage;
- predicted cue double-plays after server confirmation;
- persistent cue survives effect removal;
- pooled cue keeps previous colour/owner/attachment;
- missing cue blocks gameplay result.

**Acceptance criteria:**

- disabling cues does not affect health, tags, effects or win conditions;
- cue start/remove paths tolerate prediction/relevancy timing;
- pooled cue reset contract is documented;
- duplicate cue trace identifies source path.

## Lab 10: Replication audience comparison

**Goal:** choose Full, Mixed or Minimal active-effect replication from product evidence.

**Build:**

1. Run owner client and simulated proxy with three effect types: cooldown, poison and hidden self-buff.
2. Capture visible UI/cues/tags/effects for Full, Mixed and Minimal modes.
3. Record network bandwidth and gameplay observability.

**Injected failures:**

- owner setup is wrong for Mixed;
- simulated proxy depends on hidden owner-only details;
- UI reads active-effect detail unavailable in Minimal;
- cue expected by spectators never appears.

**Acceptance criteria:**

- table lists what owner and simulated proxy can observe in each mode;
- product requirement drives mode selection;
- attributes/tags are not misdescribed as "gone" because GE detail is minimal;
- bandwidth result is measured, not guessed.

## Lab 11: GAS versus bespoke ability defence

**Goal:** prove GAS earns its complexity for the slice or admit it does not.

**Build:**

1. Implement a bespoke Component version of sprint, dash, cooldown and one damage effect.
2. Implement the GAS version.
3. Compare lifecycle, data authoring, prediction, tags, stacking, debugging, test cost and team readability.

**Acceptance criteria:**

- five-minute defence names interaction density, not genre;
- report names state that remains outside GAS;
- report names the first point where GAS removes duplicated policy;
- report names the first point where bespoke code is still simpler.

