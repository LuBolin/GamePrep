# Android Platform Profiling Workbook For Unreal

See also: [[ue_platform_constraints]], [[ue_profiling_optimisation]], [[ue_device_lab_automation]], [[ue_packaged_performance_build_worldpartition_proof]].

Version target: **UE5.3-UE5.6 plus current Android tooling**. Android profiler support depends on device, GPU vendor, driver, OS version, capture tool release, graphics API and packaged build configuration. Treat this workbook as a workflow and interview-practice guide, not a guarantee that every capture pane exists on every device. [SRC-PLAT-010] [SRC-PLAT-011] [SRC-PLAT-012] [SRC-PLAT-013] [SRC-PLAT-014]

## Scope

This workbook covers Android target evidence for Unreal projects:

- correlating Unreal `stat`, CSV, Unreal Insights and Memory/LLM data with Android profiler captures;
- Android GPU Inspector system and frame profiling;
- CPU/GPU frame-time interpretation on Android;
- render-pass, binning, shading and GMEM bandwidth diagnosis;
- GPU counter caution across Mali, PowerVR and Adreno;
- Android low-memory-killer evidence and Play Console/vitals interpretation;
- Android Dynamic Performance Framework and the Unreal ADPF plugin;
- interview answer patterns and hands-on drills.

It is intentionally Android-specific. Apple/console profiler workflows require the target platform tools and, for consoles, authorised platform-holder documentation.

## Tool Ladder

Use the cheapest useful evidence first, then escalate:

| Layer | Tool | Best for | Caveat |
|---|---|---|---|
| Engine quick triage | `stat unit`, `stat gpu`, `stat memory`, logs | immediate thread/resource classification | editor and package can differ |
| Repeatable trend | CSV Profiler | frame percentiles, counters, gate history | low causal detail |
| Engine causal trace | Unreal Insights / Memory Insights / LLM | CPU scopes, loading, memory, task timing | trace overhead and channel availability |
| Android system trace | APA or AGI system profile | CPU/GPU scheduling, GPU queue/utilization, memory/device behavior | tool/device/GPU support varies |
| Android frame trace | AGI Frame Profiler | render passes, draw commands, GMEM/buffering, shader/pass cost | one or few frames, not whole scenario |
| Long-term field signal | Android vitals | LMK/crash/ANR/slow session trends | aggregate, delayed, not a replacement for lab reproduction |
| Sustained adaptation | ADPF / Unreal ADPF plugin | thermal headroom/status, performance hints, scalability adaptation | can hide regressions if baseline is not recorded |

## Evidence Packet

Every Android profiling report should include:

- engine commit and project build ID;
- device model, SoC/GPU, Android version, graphics API and driver if available;
- package name, build config, map/scenario and command line;
- active Device Profile, scalability level and important CVars;
- warm-up window, sample window and repeat count;
- battery, thermal and power mode notes;
- Unreal CSV/Insights artifact path;
- Android profiler capture path and tool version;
- first causal hypothesis and final owner.

If two captures disagree, do not average them. Explain what each tool measures.

## Workflow 1: CPU-Bound Or GPU-Bound On Android

**Prompt:** The frame time is 31 ms on target Android hardware. Unreal `stat unit` shows high Game and high Frame, but AGI shows GPU queue pressure.

**Key distinction:** AGI's CPU timing guidance distinguishes total CPU frame time from active CPU time. Total CPU time can include CPU waiting for GPU work, such as waits around `eglSwapBuffers`, `dequeueBuffer` or `vkQueuePresentKHR`; active CPU time represents when the CPU is running app code. [SRC-PLAT-012]

**Steps:**

1. Capture Unreal `stat unit`/CSV for the exact packaged scenario.
2. Capture Unreal Insights if CPU scopes look causal.
3. Capture AGI/APA system trace around the same window.
4. Compare active CPU time, total CPU time, GPU utilization/queue and present/swap waits.
5. If CPU is waiting on GPU, investigate render passes/materials/resolution/bandwidth before rewriting gameplay code.
6. If active CPU is high, map to Game/Render/RHI/worker scopes and Unreal callstacks.

**Common trap:** Seeing high CPU total time and assuming gameplay is CPU-bound. On Android, CPU wait on GPU can inflate the apparent frame interval. [SRC-PLAT-012]

**Interview answer pattern:**

> I would separate active CPU work from CPU waiting on the GPU. Unreal stats tell me which engine threads look high; Android system profiling tells me whether the CPU is actually running or blocked around present/swap. I would not optimise gameplay code until I know whether the frame is GPU-limited.

## Workflow 2: Render-Pass Cost With AGI Frame Profiler

**Prompt:** `stat gpu` says mobile base/shadow/post work is expensive, but the team needs hardware-level proof.

AGI Frame Profiler can examine individual render passes that compose one frame. The Android docs recommend starting with the longest render pass and looking at what dominates within the pass. On Adreno, the docs call out binning, rendering and GMEM load/store as useful signals. [SRC-PLAT-013]

**Steps:**

1. Pick one representative frame after warm-up; avoid a transition/loading frame unless the transition is the target.
2. Capture AGI frame profile.
3. Find the longest render pass.
4. Classify likely cost:
   - binning high: vertex count/format, draw setup, over-detailed geometry;
   - rendering high: pixel shading, texture fetches, high-resolution framebuffer, overdraw;
   - GMEM load/store high: render-target/depth/stencil store/load bandwidth.
5. Map pass to Unreal content: shadows, base pass, translucency, post, UI, scene capture, custom render target.
6. Fix one content/system lever and recapture the same frame.

**Common trap:** "Render pass is long" is not a diagnosis. Name the pass, the hardware sub-cost and the Unreal content lever.

**Interview answer pattern:**

> I would use AGI Frame Profiler after Unreal's GPU triage to identify the expensive hardware pass. If rendering dominates, I examine shader complexity, texture fetches, resolution and overdraw. If GMEM stores dominate, I look for render targets or depth/stencil stores that the frame no longer needs.

## Workflow 3: GPU Counter Use Without Cargo-Culting

AGI can sample GPU counters from Mali, PowerVR and Adreno GPUs, but counter names and definitions are vendor-specific and can depend on driver/tool support. The Android docs explicitly point to GPU manufacturer guides for more details. [SRC-PLAT-014]

Use counters to support a hypothesis:

- "texture bandwidth is high" should correlate with texture fetch-heavy shaders, high-resolution targets, mip/streaming issues or cache-unfriendly sampling;
- "vertex/tiler pressure is high" should correlate with vertex count, vertex format, skinning, foliage or instance count;
- "fragment/shader pressure is high" should correlate with overdraw, translucent effects, expensive material paths or high-resolution buffers;
- "idle or wait" should correlate with CPU/GPU sync, present waits, insufficient work or thermal throttling.

Do not compare raw counter names across GPU vendors as if they were the same metric.

## Workflow 4: LMK And Memory Pressure

Android low-memory-killer failures are release-quality problems, not only profiler curiosities. Android vitals exposes a user-perceived LMK rate; the official page states that a user-perceived LMK rate above 1% indicates a critical need for action, while a lower rate still does not prove memory health because background kills can hurt warm start and multitasking. [SRC-PLAT-015]

**Prompt:** Testers report that the game sometimes restarts after switching apps or returning from camera/social apps.

**Steps:**

1. Confirm whether the process was killed, crashed or intentionally restarted.
2. Capture app lifecycle logs: background, foreground, map, memory checkpoints.
3. Capture Unreal memory data: LLM tags, Memory Insights where possible, texture/mesh/audio pools and loaded maps.
4. Use Android device logs/platform signals to confirm LMK or process termination class.
5. Compare foreground peak, background retained memory and warm-return memory.
6. Reproduce with a fixed sequence: launch, load heavy map, background, run another memory-heavy app, foreground.
7. Fix residency: unload optional content on background, reduce pools, lower texture/memory budgets, adjust streaming and test warm return.

**Common trap:** Passing a foreground five-minute run does not prove lifecycle memory health.

**Interview answer pattern:**

> I would treat LMK as a lifecycle/memory-residency failure. I would correlate Android termination evidence with Unreal LLM/Memory Insights and reproduce a background/foreground scenario, because a game can be acceptable in foreground but too heavy to survive warm return.

## Workflow 5: ADPF Without Hiding Regressions

ADPF lets Android games interact with dynamic thermal and CPU management. The Android docs describe APIs for thermal state, game mode/game state and fixed-performance benchmarking. [SRC-PLAT-017]

The official ADPF Unreal plugin page says the plugin provides stable performance and helps prevent thermal throttling, includes runtime CVars, reads thermal state, can adjust Unreal Scalability and reports target/actual frame duration through performance hint sessions for Game, Render and RHI threads. It also recommends establishing a baseline before using ADPF and tuning in-game settings for the game's content. [SRC-PLAT-016]

**Correct adoption pattern:**

1. Establish baseline without ADPF adaptation: FPS, frame time, thermals, clocks if available, quality level, battery state.
2. Enable ADPF plugin on supported Android devices.
3. Record plugin CVars and chosen thermal/scalability policy.
4. Run sustained scenario: cold/warm launch, 10-30 minute loop, thermal ramp.
5. Compare frame stability, quality changes and player-visible degradation.
6. Verify adaptation does not mask a newly introduced rendering/content regression.

**Bad adoption pattern:**

```text
Enable ADPF
  -> FPS looks stable
  -> ignore that quality silently dropped from level 3 to level 0
  -> call the build fixed
```

**Interview answer pattern:**

> I treat ADPF as a sustainability and scheduling/scalability tool, not as a replacement for optimisation. I keep a no-ADPF baseline, log quality/thermal decisions, and make sure dynamic scaling is acceptable to design and art.

## Workflow 6: Fixed Performance Mode For Benchmarks

ADPF docs list Fixed Performance Mode as a way to reduce dynamic CPU clock variation during benchmarking where available. [SRC-PLAT-017]

Use it carefully:

- good for repeatable micro/benchmark comparisons;
- not representative of normal thermal/battery behavior;
- may be unavailable on many devices;
- should be recorded in the evidence packet;
- should not be mixed silently with normal device-lab trend runs.

**Interview answer pattern:**

> I would use fixed-performance mode only when the question is benchmark reproducibility. For release-facing sustained performance, I also need normal thermal behavior because users do not play in a lab-only fixed mode.

## Scenario Matrix

| Scenario | Primary Android evidence | Unreal evidence | Likely owner |
|---|---|---|---|
| High total CPU, low active CPU | AGI CPU frame wait/present tracks | `stat unit`, Insights Render/RHI | rendering/performance |
| Expensive post pass | AGI frame render-pass timing | `stat gpu`, GPU Visualizer | rendering/content |
| GMEM store bandwidth | AGI Frame Profiler render pass | render target/depth/stencil usage | rendering/engine |
| Texture bandwidth high | AGI counters / APA analysis | texture streaming/mips/materials | rendering/content |
| Background return restarts app | Android LMK/vitals/logs | LLM/Memory Insights/lifecycle logs | memory/platform |
| Thermal throttling after 15 minutes | ADPF/thermal/device logs | CSV trend, scalability state | platform/performance |
| ADPF quality drop unacceptable | plugin CVar/quality logs | scalability/device profile | performance/design/art |
| Device-lab run is noisy | fixed mode comparison, thermal state | CSV variance, run manifest | test/platform |

## Hands-On Lab: Android Target Profiling Evidence

Build on Project 4 or the device-lab extension:

1. Package an Android Development build and a Shipping-like build if project policy allows.
2. Define one deterministic scene: static GPU scene, traversal scene or combat stress scene.
3. Record device identity, active Device Profile, scalability level, graphics API and important CVars.
4. Capture Unreal CSV for three repetitions.
5. Capture an Unreal Insights trace for one representative window.
6. Capture Android system profiling with APA or AGI around the same window.
7. If GPU-bound, capture AGI Frame Profiler for one representative frame and identify the longest render pass.
8. If memory-sensitive, run background/foreground LMK reproduction and capture Unreal memory checkpoints.
9. If thermal-sensitive, run with and without ADPF adaptation and log quality-level changes.
10. Produce a one-page report: bottleneck classification, evidence, change made, before/after, residual risk.

## Debugging Prompts

### Prompt A: "CPU is 30 ms on Android"

Required answer:

- What does Unreal `stat unit` show?
- Is Android active CPU high or is CPU waiting on GPU/present?
- Which exact frame/window?
- What is the next profiler escalation?

Weak answer:

- "Optimise Tick."

### Prompt B: "GPU is slow but AGI says GMEM store is high"

Required answer:

- Which render pass stores/loads?
- Which Unreal target, depth/stencil, post, UI or scene-capture path maps to it?
- Can the buffer be transient, smaller, lower frequency or avoided?
- What visual/regression test proves the change?

Weak answer:

- "Reduce polygons."

### Prompt C: "LMK rate is below 1%, so memory is fine"

Required answer:

- Android docs warn lower LMK does not prove memory health.
- Check foreground peak and background warm-return behavior.
- Correlate Android termination evidence with Unreal LLM/Memory Insights.
- Define a memory budget and lifecycle test.

Weak answer:

- "Vitals is green, ignore it."

### Prompt D: "ADPF fixed FPS after our patch"

Required answer:

- Did quality/scalability drop?
- What was the baseline without ADPF?
- Is thermal headroom stable?
- Did the underlying content cost improve?

Weak answer:

- "ADPF solved performance."

## Interview Answer Patterns

### "How do you profile Android GPU performance for a UE game?"

> I start with a packaged build and Unreal stats/CSV to classify the scenario. If Unreal suggests GPU/render cost, I capture an Android system profile to check CPU/GPU scheduling and then an AGI frame profile for a representative frame. I map the longest render passes and hardware signals like binning, rendering or GMEM load/store back to Unreal passes, materials, render targets, shadows, UI or scene captures. I only compare before/after on the same device, settings and capture window.

### "How do Unreal Insights and Android GPU Inspector differ?"

> Unreal Insights tells me engine-side scopes, tasks, loading and memory from Unreal's instrumentation. AGI/APA tells me how the Android device and GPU executed the frame: queueing, present waits, GPU counters or render-pass details. I use both because engine scopes alone can miss driver/GPU/OS behavior, and Android tools alone do not know my gameplay/content ownership.

### "How do you handle Android LMK reports?"

> I treat LMK as lifecycle memory pressure. I confirm termination type, correlate with Android vitals/logs, then inspect Unreal memory with LLM/Memory Insights and reproduce background/foreground return. A low LMK rate is not proof that memory is healthy, and user-perceived LMK above 1% is a critical signal.

### "Should we enable ADPF?"

> I would evaluate it, especially for sustained Android performance, but not as a substitute for optimisation. I establish a baseline, enable the Unreal ADPF plugin on supported devices, log thermal/scalability decisions and verify quality changes are acceptable. Dynamic adaptation should improve sustained experience without hiding regressions.

## Report Template

```text
Android Profiling Report

Build:
Device:
GPU / API:
Scenario:
Warm-up / sample:
Unreal artifacts:
Android artifacts:

Classification:
  CPU active:
  CPU wait/present:
  GPU:
  Memory:
  Thermal:

Evidence:
  1.
  2.
  3.

Change:
Before:
After:
Residual risk:
Owner:
Next gate:
```

## Source Notes

- `SRC-PLAT-010` is the Android GPU Inspector hub.
- `SRC-PLAT-011` and `SRC-PLAT-012` anchor Android system profiling and CPU/GPU frame-time interpretation.
- `SRC-PLAT-013` and `SRC-PLAT-014` anchor AGI frame/render-pass and GPU-counter analysis.
- `SRC-PLAT-015` anchors Android LMK/vitals interpretation.
- `SRC-PLAT-016` and `SRC-PLAT-017` anchor ADPF and the Unreal ADPF plugin.
- `SRC-PERF-003`, `SRC-PERF-006`, `SRC-PERF-009`, `SRC-PLAT-005` and `SRC-PLAT-006` connect Android evidence back to Unreal profiling, memory, mobile renderer and target-device policy.
