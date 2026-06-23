# Networking Target-Branch Proof Workbook

See also: [[ue_networking_target_branch_proof]], [[ue_networking_and_replication]], [[ue_network_prediction_rollback_proof]], [[ue_hands_on_projects]].

Use this workbook inside a real UE5.3-UE5.6 project branch. If the branch does not expose a feature, the correct output is a sourced rejection/status memo, not invented code.

## Lab 1 - Source and Config Audit

**Goal:** Prove the exact networking surface available in the target branch.

**Build:**

- record engine version, changelist/source type, project net driver config and build targets;
- locate headers/source for `UCharacterMovementComponent`, `FSavedMove_Character`, `FNetworkPredictionData_Client_Character`, `FFastArraySerializer`, replicated subobjects, Replication Graph, Iris and NetworkPrediction;
- query plugin status and target CVars for RepGraph/Iris/NetworkPrediction/debug commands;
- create `networking_source_audit.md` with copied signatures limited to what the branch actually has.

**Acceptance Criteria:**

- every API used later has a file path and line/section note;
- missing/renamed APIs are listed;
- enabled/disabled plugin status is proven by config/source/runtime output;
- no later maintained-doc signature is copied without target proof.

## Lab 2 - Saved-Move Sprint Proof

**Goal:** Prove a small movement-affecting flag belongs in saved moves.

**Build:**

- custom movement component with `bWantsToSprint`;
- saved move capture/flags/restore/combine policy;
- server validation for stamina/tags/state;
- correction counter by feature.

**Broken Branch:**

- implement sprint only as a replicated bool.

**Acceptance Criteria:**

- broken branch shows delayed state or corrections under 100 ms latency;
- fixed branch records sprint state in moves and prevents combine across transitions;
- dedicated-server run with packet lag/loss records lower correction count;
- logs show client/server flags and max speed for the same move ID.

## Lab 3 - Saved-Move Dash Edge Proof

**Goal:** Prove one-frame edges and compact direction data survive prediction/replay.

**Build:**

- dash edge and quantised direction in target saved-move path;
- cooldown/resource/server-state validation;
- explicit rejection UX path;
- movement-owner policy: CharacterMovement/root-motion/physics cannot fight each other.

**Broken Branches:**

- omit dash edge from saved move;
- allow move combining across dash edge;
- move capsule with direct transform.

**Acceptance Criteria:**

- broken branches produce captured correction/rubber-band evidence;
- fixed branch shows exactly one server-reconstructed dash;
- illegal dash rejection corrects local presentation without durable side effects;
- root-motion/physics/capsule ownership is documented.

## Lab 4 - Dynamic Subobject Lifecycle Proof

**Goal:** Prove nested replicated UObject state is safe across creation, mapping, teardown and GC.

**Build:**

- server-created dynamic UObject subobject owned by a replicated Actor or Component;
- replicated pointer/reference if clients need it;
- replicated field inside the subobject;
- registered-list path if available in target branch;
- `OnRep`/client handling for null-before-mapped.

**Broken Branches:**

- create subobject on client only;
- delete/GC subobject without unregistering;
- assume replicated pointer is valid immediately.

**Acceptance Criteria:**

- join-in-progress receives correct current state;
- client tolerates null before mapping;
- forced teardown removes from registered list before GC;
- logs include creation, registration, pointer mapping, property update, removal and GC test.

## Lab 5 - Fast Array Versus Simple Array Proof

**Goal:** Prove whether Fast Array complexity is worth it.

**Build:**

- simple replicated owner-only array version;
- Fast Array version with stable IDs, revision and centralised server mutation;
- UI projection keyed by stable ID, not index;
- byte/callback capture for 10/100/1000 item cases.

**Broken Branches:**

- mutate without dirty mark;
- reuse item ID;
- handle callbacks by index.

**Acceptance Criteria:**

- missed dirty mark is caught by validation/log evidence where available;
- reused ID causes captured stale UI before fix;
- late join, add, change, remove, reorder and packet loss all pass;
- final memo adopts or rejects Fast Array with measured bytes/CPU/callback complexity.

## Lab 6 - RepGraph Adoption or Rejection

**Goal:** Make a scale decision from evidence.

**Build:**

- synthetic scale scenario: replicated pickups/NPC proxies/projectiles across multiple clients;
- baseline generic replication profile;
- Actor audience classification: spatial, owner-only, team, always relevant, dormant/static, carried;
- tiny RepGraph node only if baseline profile justifies it.

**Broken Branches:**

- stale spatial bucket after movement/teleport;
- always-relevant node grows without policy;
- team visibility not updated after team switch.

**Acceptance Criteria:**

- baseline bottleneck is proven before graph work;
- per-node/per-connection list sizes are logged;
- edge cases pass or adoption is rejected;
- memo includes scale threshold where RepGraph starts paying for itself.

## Lab 7 - Iris Status and Subobject Proof

**Goal:** Avoid false Iris claims and prove behaviour only if enabled.

**Build:**

- config/runtime proof of Iris enabled or disabled;
- if disabled, write status memo and stop;
- if enabled, test registered subobject path and any target-required custom UObject fragments/descriptors;
- compare assumptions with Generic path.

**Broken Branches if Enabled:**

- Generic-only subobject path;
- missing fragment registration for custom UObject where target source requires it;
- stale registered subobject.

**Acceptance Criteria:**

- enabled/disabled state is proven;
- Iris-specific code exists only in an Iris-enabled proof branch;
- dynamic subobject create/update/remove/GC passes under Iris if enabled;
- target branch caveats are documented.

## Lab 8 - NetworkPrediction Compile Proof or Rejection

**Goal:** Convert public-doc NetworkPrediction knowledge into target-branch evidence.

**Build one of:**

- toy counter model using target plugin APIs;
- rollback dash model;
- explicit rejection memo proving API/plugin/status mismatch.

**Required Tests:**

- input/sync/aux/cue state is serialised/logged;
- forced reconcile from client/server divergence;
- cue dedupe under replay;
- side-effect ledger keeps damage/rewards/analytics outside replayed tick;
- smoothing off/on comparison;
- buffer CPU/memory/bandwidth table.

**Acceptance Criteria:**

- compile/run proof or explicit rejection memo;
- logs show restored frame, replay window and final convergence;
- no duplicate irreversible side effects;
- target-source API notes are included.

## Lab 9 - Unified Packet Matrix

**Goal:** Run all proven networking paths under the same packet-condition matrix.

| Feature | 0 ms | 100 ms + jitter | 200 ms + 2% loss | Join-in-progress | Packaged/dev build | Result |
|---|---|---|---|---|---|---|
| saved sprint | | | | | | |
| saved dash | | | | | | |
| dynamic subobject | | | | | | |
| Fast Array | | | | | | |
| RepGraph or rejection | | | | | | |
| Iris status/proof | | | | | | |
| NetworkPrediction proof/rejection | | | | | | |

**Acceptance Criteria:**

- every cell has pass/fail/not-applicable with artefact links;
- at least one broken-before/fixed-after case per implemented feature;
- no feature is marked production-ready without source, packet and profiling evidence.

