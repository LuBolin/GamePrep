# Runtime Presentation Target-Proof Workbook

See also: [[ue_runtime_presentation_target_proof]], [[ue_animation_systems]], [[ue_physics_collision]], [[ue_ui_systems]], [[ue_audio_systems]], [[ue_niagara_vfx]].

This workbook turns animation, physics, UI, audio and VFX first-pass knowledge into target-proof practice. Use a small arena map and one attack/interact/action scenario.

Sources: [SRC-ANIM-003] [SRC-ANIM-006] [SRC-PHYS-003] [SRC-UI-005] [SRC-AUDIO-004] [SRC-FX-001] [SRC-PERF-003] [SRC-NET-006]

## Lab 1 - Scenario Contract

**Goal:** define one cross-system action with clear authority and presentation.

**Tasks:**

1. Name the action, operation ID and owner.
2. Define server-authoritative state changes.
3. Define predicted/local presentation.
4. Define target package, net mode, latency/loss and sample window.
5. Define pass/fail evidence.

**Acceptance criteria:** the scenario distinguishes gameplay truth from animation/audio/VFX/UI presentation.

## Lab 2 - Animation Notify Window Proof

**Goal:** prove a montage/notify does not apply gameplay after cancel or wrong state.

**Tasks:**

1. Open and close a hit window from animation timing.
2. Server-validate claims against montage/window state.
3. Cancel montage mid-window and after window.
4. Test under dedicated-server latency.

**Acceptance criteria:** cancelled or stale claims are rejected; cosmetic trails can still end cleanly.

## Lab 3 - Collision Matrix Proof

**Goal:** classify and verify collision expectations.

**Tasks:**

1. Create a matrix for player, target, shield, trigger and world static.
2. Run representative traces/sweeps/overlaps.
3. Log object type, response, hit/overlap/block and authority.
4. Inject one wrong profile and diagnose it.

**Acceptance criteria:** every row has expected and observed result with evidence.

## Lab 4 - UI Recycled Entry Proof

**Goal:** prove UI entries reject stale model and async asset results.

**Tasks:**

1. Build a damage feed or inventory list with recycled entries.
2. Add async icons or names.
3. Rapid scroll/filter/sort/delete.
4. Add request IDs or expected item IDs.

**Acceptance criteria:** stale async completions do not change recycled widgets.

## Lab 5 - Local Player And Focus Proof

**Goal:** prove UI/input scope is correct.

**Tasks:**

1. Run two local players if project supports it, or simulate local-player scoped services.
2. Open a menu for one player.
3. Verify focus, input contexts and owner-only data do not leak.
4. Close and restore gameplay input.

**Acceptance criteria:** player-specific UI/input state remains scoped.

## Lab 6 - Audio Dedupe And Concurrency Proof

**Goal:** prevent duplicate predicted/server audio and test voice priority.

**Tasks:**

1. Play local predicted action sound.
2. Play server-confirmed impact sound.
3. Add event keys to dedupe.
4. Stress concurrency with many emitters.

**Acceptance criteria:** no double-play for one logical event; important sounds survive concurrency rules.

## Lab 7 - VFX Pool And Bounds Proof

**Goal:** make pooled VFX safe under reuse and culling.

**Tasks:**

1. Pool one Niagara component/effect.
2. Reset owner, attachment, parameters, delegates and event ID.
3. Reuse across different targets.
4. Test camera edge/bounds and significance culling.

**Acceptance criteria:** no stale parameter/target; bounds and culling behave as intended.

## Lab 8 - Physics Reaction Or Ragdoll Proof

**Goal:** validate physics presentation and recovery.

**Tasks:**

1. Add impulse, knockback or ragdoll reaction.
2. Record authority policy.
3. Verify collision profile changes and recovery.
4. Test dedicated server or target net mode.

**Acceptance criteria:** reaction does not create stuck physics state or authority divergence.

## Lab 9 - Performance Capture

**Goal:** prove the action stack fits budget.

**Tasks:**

1. Run packaged 1/20/100 actor or effect scale where feasible.
2. Capture CSV.
3. Capture one targeted trace for worst frame.
4. Classify Animation, Physics, UI, Audio, VFX, Draw/RHI or GPU owner.

**Acceptance criteria:** performance report has target package identity, scenario and owner.

## Lab 10 - Failure Injection Report

**Goal:** prove diagnostics catch broken states.

Inject at least five:

- notify remains open after cancel;
- wrong collision profile;
- stale list entry icon;
- duplicate sound from prediction plus server;
- VFX pooled component keeps old target colour;
- VFX bounds too small;
- ragdoll never recovers;
- UI focus remains on closed menu.

**Acceptance criteria:** each injected failure has expected symptom, evidence, fix and recurrence guard.

