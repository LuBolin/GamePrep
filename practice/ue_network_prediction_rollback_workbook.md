# NetworkPrediction Rollback Proof Workbook

See also: [[ue_network_prediction_rollback_proof]], [[ue_networking_and_replication]], [[ue_networking_target_branch_proof]], [[ue_hands_on_projects]].

Use this workbook only with a pinned UE branch. The public API page gives enough vocabulary for study, but exact plugin integration must be proven in the target engine source. [SRC-NET-020]

## Lab 1 - Target Branch Evidence Packet

**Goal:** Prove what the target branch actually supports before writing feature code.

**Build:**

- record UE version, changelist if available, platform and build configuration;
- confirm `NetworkPrediction` plugin enablement and module compilation;
- list headers/classes used for component, model definition, driver, state types, services and CVars;
- record which debug CVars exist in Development, Test and Shipping-like builds;
- note whether fixed tick, independent tick, rollback, interpolation and smoothing services are usable for the chosen model.

**Acceptance Criteria:**

- no implementation claim relies only on maintained public docs;
- every copied signature has a source/header location;
- unsupported or changed APIs are listed as blockers or adaptation tasks;
- the evidence packet includes a minimal compile target.

## Lab 2 - Minimal Deterministic Counter Model

**Goal:** Build the smallest rollback-capable model before attempting movement.

**Build:**

- input command: increment/decrement/none plus local frame;
- sync state: integer value and state frame;
- aux state: multiplier/version;
- cue: threshold crossed;
- local prediction, authority correction and replay.

**Injected Failures:**

- client-only multiplier mismatch;
- dropped input;
- duplicated threshold cue on replay;
- aux version change between predicted frames.

**Acceptance Criteria:**

- forced reconcile restores the authoritative frame and replays pending input;
- cue dedupe prevents duplicate presentation;
- logs include input hash, aux version, pre/post sync hash and replay window.

## Lab 3 - Rollback Dash Model

**Goal:** Move from toy state to compact gameplay movement without using full CharacterMovement as the proof.

**Build:**

- input command: quantised move vector, dash press, local frame;
- sync state: location, velocity, movement phase, dash ticks remaining, state frame;
- aux state: speed, dash duration, dash impulse, cooldown table version;
- cue: dash started and dash rejected presentation.

**Injected Failures:**

- client-only acceleration bias;
- server-only blocker;
- late aux-state update;
- invalid dash while cooldown is active.

**Acceptance Criteria:**

- server rejection converges cleanly without leaving local dash phase active;
- forced reconcile identifies the exact divergent field;
- smoothing off reveals correctness, smoothing on improves UX without hiding reconcile spam;
- damage/reward/inventory side effects are absent from the replayed tick.

## Lab 4 - Cue Trait and Dedupe Matrix

**Goal:** Prove which side effects are safe under prediction and resimulation.

**Build a matrix for:**

- local dash VFX;
- camera shake;
- footstep;
- hit marker;
- damage number;
- item reward;
- analytics event.

For each, decide:

- predicted, replicated, authority-only or local-only;
- replay-safe or not;
- key fields for dedupe;
- late-join reconstruction policy;
- whether it belongs outside the model.

**Acceptance Criteria:**

- replay-safe cues do not duplicate under forced reconcile;
- irreversible effects occur exactly once on the authority;
- the matrix references the target branch cue trait API rather than generic guesses.

## Lab 5 - Aux State Versioning Drill

**Goal:** Catch the common bug where old frames replay with new tunables.

**Build:**

- a weapon/equipment modifier that changes dash acceleration;
- aux-state version numbers;
- a command sequence where the player swaps equipment during a prediction window.

**Injected Failures:**

- model reads the current weapon component during replay;
- aux version is not logged;
- pending inputs are replayed with the wrong modifier.

**Acceptance Criteria:**

- replay uses the aux value intended for each frame, or pending inputs are explicitly rejected;
- logs show old/new aux version and policy;
- the system has a deterministic answer for "swap at the same frame as dash".

## Lab 6 - Server Authority and Anti-Cheat Validation

**Goal:** Keep rollback responsive without trusting client authority.

**Build:**

- server-side validation for cooldown, resource, movement phase, max distance, target identity and timestamp bounds;
- structured reject reasons;
- client UX reconciliation for rejection.

**Injected Failures:**

- client claims a target behind a wall;
- client sends dash press during cooldown;
- client sends impossible local frame/timestamp;
- client sends a stale target ID after respawn.

**Acceptance Criteria:**

- each invalid command is rejected by server-owned state;
- reject reason is logged and visible to debugging UI;
- client presentation returns to a valid state without permanent ghost effects.

## Lab 7 - Buffer Budget and Correction Storm

**Goal:** Measure memory and CPU cost of rollback instead of assuming it is cheap.

**Build:**

- input/sync/aux/cue buffer size counters;
- replay-frame count and max replay burst;
- per-instance memory estimate;
- scenario for 1, 16, 64 and 128 model instances.

**Injected Failures:**

- force reconcile every N frames;
- increase rollback window;
- enlarge sync state with unnecessary fields;
- emit too many cues per frame.

**Acceptance Criteria:**

- report memory per instance and total memory at target player/model count;
- report replay CPU cost and worst-case correction burst;
- identify one field removed from sync/aux after measurement;
- define pass/fail thresholds for automated tests.

## Lab 8 - Actor Bridge and Replication Boundary

**Goal:** Prevent Actor replication from fighting the rollback model.

**Build:**

- clear mapping between model-owned state, Actor replicated durable state and presentation state;
- owning connection validation;
- respawn/destruction invalidation of model instance IDs;
- late-join reconstruction from durable authoritative state.

**Injected Failures:**

- Actor replicated transform overwrites model correction;
- model instance remains after Actor destroy;
- cue history survives respawn;
- late joiner sees old predicted presentation.

**Acceptance Criteria:**

- one writer owns each field;
- destruction/respawn invalidates stale model and cue keys;
- late joiner reconstructs durable state without replaying historical predicted cues.

## Lab 9 - Compare Against Alternatives

**Goal:** Avoid adopting rollback by fashion.

Compare the same feature using:

1. server-authoritative command only;
2. CharacterMovement saved move where applicable;
3. server rewind validation;
4. NetworkPrediction rollback model;
5. custom lightweight prediction/reconcile.

**Acceptance Criteria:**

- comparison includes responsiveness, exploit surface, implementation complexity, debug tooling, bandwidth, CPU, memory and platform risk;
- at least one alternative is honestly rejected with evidence;
- the recommendation names the smallest system that satisfies the product requirement.

