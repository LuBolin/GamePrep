# Unreal Engine Profiling and Optimisation

See also: [[ue_rendering_graphics_performance]], [[ue_platform_constraints]], [[ue_device_lab_automation]], [[ue_packaged_performance_build_worldpartition_proof]], [[ue_hands_on_projects]].

## Cluster 1 — Frame Budgets, Bottleneck Classification, Insights, Memory, and Regression

**Priority:** P1 (P0 as a cross-cutting interview skill)  
**Expected depth:** D4  
**Version scope:** UE5.3–UE5.6 preferred. Tool UI and trace channels evolve; concepts and target-device method are stable.  
**Prerequisites:** Gameplay framework, C++ lifetime/containers, UObject GC, networking and AI performance foundations.

### Optimisation is controlled investigation

The professional loop is:

```text
Define target and budget
  -> reproduce a representative problem
  -> establish baseline distribution and variance
  -> classify limiting resource/thread
  -> capture detailed evidence
  -> form one falsifiable hypothesis
  -> change one meaningful variable
  -> rerun identical scenario
  -> verify performance, correctness, quality, memory, and other platforms
  -> keep/revert and prevent regression
```

“Remove Tick”, “use pooling”, “reduce draw calls”, and “move to C++” are candidate interventions, not diagnoses.

### Frame time, not only FPS

FPS is reciprocal and awkward for budgets. Frame time is additive:

| Target | Frame budget |
|---:|---:|
| 30 FPS | 33.33 ms |
| 60 FPS | 16.67 ms |
| 90 FPS | 11.11 ms |
| 120 FPS | 8.33 ms |

Budget below the theoretical maximum to absorb OS work, content variance, network bursts, streaming, GC, and thermal changes. Consistency matters: a game averaging 60 FPS with regular 50 ms hitches feels worse than its average suggests. Track percentiles and worst representative spikes, not just mean FPS. Epic's UE5.6 overview explicitly frames performance in frame time and target hardware/resource bottlenecks. [SRC-PERF-001]

### Threads and pipeline overlap

At interview depth, know these lanes:

- **Game thread:** gameplay, Actor/Component ticks, much Blueprint, AI orchestration, UObject work, submission/coordination.
- **Render thread (“Draw” in stat unit):** prepares scene/render commands and rendering-side CPU work.
- **RHI thread:** translates/submits lower-level rendering work where enabled/used.
- **GPU:** executes rendering/compute passes.
- **Task/worker threads:** async/task graph work that can still block a critical dependency.

They overlap. Frame time is often near the slowest synchronised lane, not the sum of Game + Draw + GPU. A worker task can be the true cause while the game thread appears to wait. GPU timings can be influenced by synchronisation, VSync, frame caps, dynamic resolution, or measurement mode.

### First-pass classification with `stat unit`

`stat unit` displays Frame, Game, Draw, GPU, RHIT and dynamic-resolution information. If Frame tracks Game, investigate game-thread/CPU work; if it tracks Draw, render-thread CPU; if it tracks GPU, investigate GPU passes/content. Use a non-debug representative build, disable confounding frame caps/VSync where appropriate, and profile target hardware. [SRC-PERF-002]

Useful triage commands:

| Command | Question |
|---|---|
| `stat unit` / `stat unitgraph` | Which high-level lane limits frame time and where are spikes? |
| `stat game` | Which gameplay stat groups dominate? |
| `stat gpu` | Which broad GPU categories dominate? |
| `stat scenerendering` | What scene/render counts suggest pressure? |
| `stat rhi` | Draw/primitive/RHI resource indicators |
| `stat memory` / platform memory stats | Is memory pressure or allocation behaviour suspicious? |
| `stat uobjects` | UObject counts/context for GC investigation |
| `stat slate` / UI stats where available | Is UI traversal/painting/layout material? |

Stats are fast orientation. They are not enough to establish call relationships, async dependencies, allocation lifetime, or one-frame causality; capture Insights next.

### Unreal Insights: answer “where and why?”

Unreal Insights records trace events and visualises CPU/GPU tracks, frames, timers, counters, logs, callers/callees, task/context-switch, memory, asset loading, network, and Slate channels depending on capture configuration/version. Trace sessions are stored in `.utrace` and are self-describing. [SRC-PERF-003]

Timing workflow:

1. Capture a short, labelled, reproducible window including normal frames and the bad event.
2. Select slow frames and compare against nearby healthy frames.
3. Inspect Game/Render/RHI/GPU and worker lanes at the same timestamp.
4. Expand expensive scopes; use callers/callees to find who scheduled/called them.
5. Check whether a long scope works or waits on another lane/task/fence.
6. Group/aggregate across many frames to distinguish one spike from steady cost.
7. Add a narrow custom trace/stat scope around ambiguous project code, then recapture.

Do not capture every trace channel for minutes by default. Tracing consumes CPU, memory, and disk and can perturb the workload. Capture the minimum evidence window/channel set that answers the question, then validate without heavy instrumentation.

### Instrument project code deliberately

UE's Stats System supports cycle counters, per-frame counters, accumulators, and memory stats. `QUICK_SCOPE_CYCLE_COUNTER` is useful during investigation; stable subsystem stats deserve named groups and consistent scope. [SRC-PERF-004]

Instrumentation should answer a question:

- How many inventory recomputations occur per frame?
- Which caller triggers duplicate path queries?
- How long does save serialisation block?
- How many entities are active versus visible/significant?
- Did the optimisation reduce total work or merely move it to a worker/thread?

Count and time. A 2 microsecond call made 50,000 times costs more than a scary-looking 1 ms call made once per loading screen.

### CPU-bound workflow

After `stat unit` suggests Game/Draw/CPU limitation:

1. Capture Timing Insights on target-like build/hardware.
2. Find top steady scopes and worst spikes separately.
3. Check call count, inclusive/exclusive time, callers, and worker dependencies.
4. Classify waste: frequency, algorithm, allocation, cache locality, synchronisation, virtual/Blueprint boundary, or duplicate work.
5. Change the largest verified cost while preserving semantics.

Common gameplay patterns:

- **Tick:** disable while idle, use events/timers/manager batching where cadence permits. Tick is correct for genuine frame integration.
- **Polling:** replace known state-change checks with callbacks/delegates; if events fire repeatedly and overwrite results, batch/dirty-flag once per frame.
- **Actor/Component count:** each UObject/Actor carries memory/lifecycle overhead; use lighter values/ISM/Mass/manager representations where identity is unnecessary.
- **Blueprint:** reduce high-frequency VM boundary calls, repeated pure-node work, and broad searches; move measured hotspots behind narrow native APIs.
- **Containers:** avoid per-frame allocation/re-hash/copy; reserve from evidence; use contiguous iteration and stable-ID strategies.
- **AI:** stagger services/EQS/perception, significance-LOD, reduce candidates/queries.
- **UI:** event-driven state, invalidation/retainer usage where appropriate, avoid unnecessary widget construction/layout/paint and binding polling.

Epic's common-performance guide explicitly contrasts Tick polling with callbacks/timers/scheduled intervals and notes that event-driven is not automatically cheaper when events fire many times per frame. [SRC-PERF-005]

### GPU-bound workflow

If GPU time limits the frame:

1. Confirm by changing resolution/screen percentage. A strong GPU-time change suggests pixel/fill/shader work; little change points towards geometry/submission/fixed-cost passes or another bottleneck.
2. Use `stat gpu`, GPU Visualiser/profile GPU, and Timing Insights GPU tracks to rank passes.
3. Inspect scene counts/overdraw/shader complexity/light/shadow/translucency/material/texture pressure.
4. Use RenderDoc for single-frame pipeline/resource/shader debugging when a pass or artefact needs deeper inspection. [SRC-PERF-001]
5. Change the content/system driving the expensive pass, not merely the pass label.

Examples:

- Base pass high: material complexity, pixels, geometry, early-Z/config.
- Shadows high: light count/type, shadowed primitives, resolution, VSM page pressure.
- Translucency high: overdraw, screen coverage, particle layers.
- Post-process high: resolution, enabled effects, quality.
- Draw/RHI CPU high but GPU moderate: draw submission/state changes/scene traversal rather than shader arithmetic.

Deep rendering is a follow-on cluster; the profiling skill is to identify the pass and evidence before prescribing Nanite/LOD/instancing/material changes.

### Memory is capacity, churn, locality, and lifetime

Memory problems appear as:

- peak/resident footprint and out-of-memory;
- short-lived allocation churn causing CPU/fragmentation;
- retained/leaked allocations across transitions;
- asset/texture streaming pressure and I/O churn;
- poor locality/cache misses despite acceptable byte count;
- GC object count/reference scanning and destruction spikes.

Memory Insights tracks allocation/reallocation/free events, callstacks and LLM tags; its queries compare live, growth, decline, short-lived, long-lived, and leak-like allocations across timeline markers. [SRC-PERF-006]

Workflow:

1. Mark before/after a repeatable event (open inventory, stream level, complete match, travel).
2. Query growth and expected release.
3. Group by callstack/tag, not only allocation size.
4. Separate intended cache growth from leak/retention.
5. Repeat the cycle several times: unbounded staircase growth is stronger evidence than one warm-up increase.
6. Verify GC/streaming timing before concluding raw memory leaked.

### GC workflow

GC spikes correlate with UObject population/reference graph and destroyed-object bursts. Confirm the spike in Insights and object/stat data. Then ask:

- Are high-churn short-lived concepts unnecessarily UObjects/Actors?
- Are large reference graphs retained accidentally?
- Are many objects destroyed in one frame/travel boundary?
- Is pooling appropriate, or would it retain too much memory/state?
- Is manual GC during an acceptable loading boundary justified by measured UX?

Do not begin by increasing GC intervals; that can accumulate a larger eventual scan/purge. Do not pool everything; pools trade allocation/destruction churn for memory, reset bugs, stale references, and worst-case capacity.

### Loading and hitches

A hitch may be CPU computation, synchronous asset I/O, shader/PSO creation, object construction/registration, GC, or thread synchronisation. Align timing, loading, file activity, and memory tracks around the spike. Common interventions include soft/async loading, preloading at known boundaries, PSO caching, bounded construction, and spreading work—after the source is proven.

An async API does not guarantee hitch-free behaviour: completion may construct/register assets on the game thread, callers may synchronously wait, or dependencies may still hard-load.

### Scalability and device profiles

Optimisation is not one quality level that barely runs on a developer PC. Scalability groups adjust quality features; device profiles apply target-family/device CVars and memory/quality overrides. [SRC-PERF-007] [SRC-PERF-008]

Design scalable dimensions:

- resolution/dynamic resolution;
- shadows, lighting, post-processing, effects;
- texture quality/streaming pools;
- view distance and LOD;
- foliage/crowd/significance counts;
- animation/AI update rates where gameplay permits.

Never make gameplay authority or deterministic simulation depend accidentally on a client graphics profile. Test lowest and highest tiers for correctness, visual readability, memory, and transition behaviour.

### Benchmark discipline

A credible result records:

- hardware, OS, driver, power/thermal mode;
- engine/project revision and build configuration;
- resolution, scalability, VSync/frame cap/dynamic resolution;
- map, camera path, player/AI count, network mode;
- warm-up period and sample duration;
- median/percentiles/worst spikes and memory;
- profiler/trace overhead;
- before/after artefacts and correctness/quality trade-off.

Use fixed camera paths or automation where possible. Compare identical scenarios. One editor viewport frame is not a benchmark.

### Common misconceptions

1. **Higher FPS proves optimisation.** Frame cap, VSync, resolution, dynamic resolution, or different workload can fake it.
2. **Frame time equals Game + Draw + GPU.** Pipelines overlap and synchronise.
3. **GPU time high means too many draw calls.** Draw calls can be CPU submission; GPU pass cost may be pixels/shaders/bandwidth.
4. **Tick is bad.** Wasteful work/frequency is bad; Tick is correct for frame-dependent integration.
5. **Timers are faster than Tick.** Same frequency/work rarely changes fundamentals.
6. **Async means off-thread and hitch-free.** Callers/completion/dependencies can block.
7. **Pooling always improves performance.** It can worsen memory/reset complexity and locality.
8. **Move work to a worker and frame is fixed.** Critical-path waits or contention may remain.
9. **Average FPS is enough.** Hitches and percentiles matter.
10. **Editor profiling predicts shipping.** Profile representative packaged builds on target hardware.
11. **Optimise code first.** Content/config/system architecture often dominates.
12. **One fix fits every platform.** Bottlenecks move across hardware and quality tiers.

### Strong interview answer pattern

For “the game drops from 60 to 40 FPS”:

1. Confirm target, build, hardware, scenario, frame cap/VSync/dynamic resolution.
2. Use frame time and `stat unit` to classify Game/Draw/GPU/other.
3. Capture Insights/GPU profile around representative slow frames.
4. Identify top scope/pass plus frequency/callers and form a hypothesis.
5. Change one cause, rerun identical capture, and verify quality/correctness/memory.
6. Test target devices/tiers and add regression telemetry/test.

### What a three-year engineer should know

Convert FPS to budget, distinguish steady cost from hitches, interpret `stat unit`, navigate Timing Insights, add scoped instrumentation, investigate memory growth/churn, diagnose GC/Tick/UI/Actor-count patterns, profile GPU passes at a high level, and apply scalability/device profiles. Most importantly, defend an evidence chain rather than reciting optimisations.

**Specialist depth:** platform profilers, context-switch analysis, cache counters, allocator internals, custom trace channels, RDG/GPU event internals, PSO pipelines, automated performance CI, thermal/power optimisation, and platform certification budgets.

### Hands-on verification

Complete Project 4's baseline profiling lab before the deep rendering expansion. Submit two before/after traces: one CPU/game-thread issue and one GPU/content issue, each with a falsified alternative hypothesis.

## Cluster 2 — Specialist target capture, memory budgets, CSV gates, and device matrices

**Priority:** P2/P3 specialist depth  
**Expected depth:** D4 for engine/generalist/tools roles  
**Version scope:** UE5.3–UE5.6 preferred. Tool commands, trace channels, CSV categories and LLM tag coverage are target-branch sensitive.

The first profiling cluster teaches how to diagnose a local performance problem. Specialist depth is proving that the result survives real target conditions: packaged builds, device profiles, cold caches, long-running sessions, memory pressure, thermals, streaming, automated benchmark noise and CI cost. The output is not just a faster frame; it is a repeatable evidence packet a team can trust. [SRC-PERF-001] [SRC-PERF-003] [SRC-PERF-009]

### The target capture contract

A performance report should begin with a capture contract. Without it, before/after numbers are usually theatre.

```text
Build identity:
  engine revision, project revision, branch, target, config, platform, build ID
Runtime environment:
  device model, OS/driver, power mode, thermals, storage, network mode
Display and quality:
  resolution, screen percentage, VSync/cap, dynamic resolution, scalability, device profile
Scenario:
  map, spawn counts, camera path, input script, warm-up, sample length, seed
Capture tools:
  stat commands, trace channels, CSV categories, platform profiler, GPU capture
Metrics:
  median/P95/P99/worst frame time, hitch count, Game/Draw/GPU, memory, I/O, correctness
Decision rule:
  pass/fail threshold, quality acceptance, regression budget, owner
```

When the contract is missing, candidates tend to argue from averages, editor viewport captures or "felt smoother" claims. A strong engineer makes the scenario reproducible first, then optimises inside it.

### Tool choice: stats, Insights, CSV, platform profilers

Different tools answer different questions:

| Tool | Best question | Weakness |
|---|---|---|
| `stat unit`, `stat game`, `stat gpu`, `stat memory` | quick local classification | not enough call/lifetime/causal detail |
| Unreal Insights | why a frame, task, allocation, load, network event or wait happened | trace overhead and large files if overused |
| Memory Insights | allocation lifetime/growth/churn with callstacks/LLM tags | requires suitable memory trace setup and platform support |
| CSV Profiler | long-run lightweight metrics, automated scenarios and regression comparison | less causal detail than Insights |
| Low-Level Memory Tracker | tag-level memory budgets and category ownership | tag granularity/coverage can hide attribution gaps |
| GPU Visualiser/GPU capture | pass/resource/shader/pipeline facts | single-frame or narrow-window and RHI/vendor-specific |
| Platform profilers | driver, CPU counter, thermal, memory and platform-specific truth | require target hardware, symbols and platform expertise |

Use the cheapest tool that can answer the current question. Escalate when the answer requires more detail. Do not use a multi-minute all-channel Insights trace as routine telemetry when CSV counters would catch the regression with less overhead. [SRC-PERF-003] [SRC-PERF-009] [SRC-PERF-010]

### Command and capture patterns

Exact command-line syntax changes by engine branch and platform, so verify with target help/output. These are patterns, not copy-paste promises:

```text
# Packaged or command-line run with targeted trace channels.
<GameOrEditorCmd> <ProjectOrMap> -game -trace=cpu,frame,gpu,bookmark,log,loadtime,file,memory

# Mark scenario boundaries in logs/traces from code or console where available.
TraceBookmark("PerfCase_InventoryOpen_Start")
TraceBookmark("PerfCase_InventoryOpen_End")

# CSV capture from console or automation wrapper.
csvprofile start
csvprofile stop

# Memory/LLM investigation pattern.
Enable target memory tracing or LLM options for the branch/platform,
capture repeated lifecycle cycles,
then compare live/growth/churn by tag and callstack.
```

The important part is not the command spelling. The important part is a named scenario, bounded window, known channels/categories, and artifacts saved with build metadata.

### Packaged hitch workflow

Use this when the editor is smooth but a packaged or target build hitches:

1. Confirm the hitch in the exact target build/config/device profile.
2. Disable or document frame caps/VSync/dynamic resolution so they do not hide the pattern.
3. Record a lightweight CSV run first if the hitch is intermittent or long-session dependent.
4. Capture a short Insights window around one reproduced hitch.
5. Align frame time with Game/Draw/GPU, file/load activity, memory allocation, GC, shader/PSO and task waits.
6. Classify the hitch:
   - synchronous asset or file load;
   - first-use shader/PSO creation;
   - UObject construction/registration/GC;
   - streaming texture/mesh/audio pressure;
   - device-profile quality setting;
   - CPU task wait or lock contention;
   - GPU pass spike or capture-only view.
7. Make one causal change and rerun the same packaged scenario.
8. Add a regression signal: CSV threshold, automation scenario, load gate, PSO coverage test or memory budget check.

Do not conclude "packaged is slower" from one capture. Packaged builds can be faster in CPU code but worse in cold caches, target RHI, shipping content, cooked asset layout, PSO coverage, memory limits, platform storage or device-profile quality.

### Memory budget workflow: Memory Insights plus LLM

Memory Insights and LLM answer related but distinct questions. Memory Insights is strong for allocation lifetime, callstack and churn. LLM is strong for budget ownership by engine/project tags. [SRC-PERF-006] [SRC-PERF-010]

Use both when a target device exceeds budget:

1. Define the memory budget by platform, map, mode and quality tier. Include headroom for OS/platform services, streaming bursts and long sessions.
2. Record LLM/tag totals at stable checkpoints: boot, front end, map loaded, peak combat, travel, post-match, return to menu.
3. Capture Memory Insights around the highest growth or churn interval.
4. Split the finding:
   - **capacity:** resident total too high at stable point;
   - **growth:** repeated cycle does not return near baseline;
   - **churn:** short-lived allocation rate causes CPU/fragmentation/hitches;
   - **streaming pressure:** pool/budget too low or content too large;
   - **UObject/GC:** object count and reference graph drive scan/destruction cost;
   - **locality:** bytes acceptable but access pattern/cache poor.
5. Assign ownership by tag, asset class, subsystem and callstack.
6. Fix the owning system/content, not just the global pool number.
7. Rerun the same checkpoints after a cold start and after long-session soak.

Weak answer: "Increase texture pool size" or "pool objects." Strong answer: "This tier exceeds the peak combat memory envelope by 220 MB. LLM shows Textures and VFX are the owners; Memory Insights shows an additional UI icon cache staircase across menu open/close. I would fix UI cache lifetime separately from texture LOD/profile policy."

### CSV performance regression gates

CSV Profiler is useful where Insights is too heavy: long runs, automated scenarios, soak tests and historical regression comparison. It should not replace causal profiling, but it can tell you when causal profiling is needed. [SRC-PERF-009]

A practical CSV gate tracks a small number of stable metrics:

- frame time percentiles and hitch counts;
- Game/Draw/GPU high-level timings;
- map load time and first-interactive time;
- memory/LLM totals by high-level tag;
- object/Actor/widget/System counts;
- network, streaming or shader/PSO counters if relevant;
- scenario-specific counters, such as enemy count, projectile count or inventory rows.

Gate design:

1. Choose deterministic or tightly bounded scenarios. Fixed camera paths and seeds beat "play for a minute."
2. Warm up intentionally, then measure a fixed window.
3. Store CSV output with build metadata and scenario name.
4. Compare against a rolling baseline or release budget, not a single lucky run.
5. Use tolerance bands to avoid noisy false failures.
6. Fail only on actionable signals; warn on early drift.
7. When the CSV gate fails, capture Insights/GPU/platform details for that scenario.

An interview-worthy answer should acknowledge noise. A 1 percent change on one desktop run is not a regression. A repeated P95 frame-time increase with unchanged scenario and matching memory/count drift is actionable.

### Device profile and scalability proof matrix

Scalability and device profiles are product design, not a late panic. [SRC-PERF-007] [SRC-PERF-008]

Build a matrix:

| Dimension | Low tier proof | High tier proof |
|---|---|---|
| Resolution/dynamic resolution | UI readable, aim/interaction still fair | no artificial cap hides GPU overload |
| Shadows/lights/Lumen/VSM | gameplay visibility preserved | GPU budget and memory acceptable |
| Texture/VT/streaming pool | no unreadable critical assets | no hidden over-budget pool |
| FX/audio/crowd/animation significance | important feedback retained | peak counts do not exceed frame budget |
| Device profile inheritance | expected CVars applied | profile overrides do not conflict |
| Network/gameplay authority | simulation independent of graphics tier | cosmetic quality does not change rules |

Verification must include an applied-CVar dump or equivalent proof. A profile file existing in source is not proof it selected at runtime. Test the lowest tier for correctness, not only performance; low settings that hide enemy telegraphs, UI affordances or navigation cues are product bugs.

### Performance CI without making CI unusable

Performance gates fail when they are either too expensive or too vague. Use tiers:

- **Pre-submit:** compile, focused tests, cheap counters for obviously bad changes.
- **Nightly:** clean packaged benchmark, CSV scenario suite, memory checkpoints, representative device profiles.
- **Weekly/release:** long-session soak, platform hardware, GPU captures, thermal/power, first-launch/cold-cache and certification-style cases.
- **Investigation on failure:** Insights/platform capture for the failing scenario only.

The pass/fail signal should be specific:

```text
Scenario: FrontEnd_Inventory_1000Items
Build: Win64 Development client, profile LowDesktop
Window: frames 600-1800 after warm-up
Fail if:
  P95 Frame > 18.0 ms for 3 consecutive runs, or
  Game thread P95 regression > 10 percent from rolling 20-build baseline, or
  LLM/UI grows > 64 MB after five open/close cycles
Artifacts:
  CSV, stat dump, applied CVars, screenshot, build metadata
Owner:
  UI Systems
```

This is more useful than "FPS below 60" because it includes scenario, window, metric, threshold, artifact and owner.

### Specialist answer patterns

**Question: how do you prove an optimisation is real?**

> I define the target build, hardware, scenario, quality profile, warm-up and metrics first. Then I capture a baseline distribution, not just one average. I use stats/CSV for broad repeatability and Insights or GPU/platform tools for causal detail. After one change I rerun the identical packaged scenario, verify correctness and visual quality, and test at least the low/high device tiers. If it matters long term, I add a CSV or automation gate with tolerance bands and artifacts.

**Question: Memory Insights versus LLM?**

> Memory Insights tells me allocation lifetime, callstacks, growth and churn. LLM gives tag/category ownership and budget views. For a budget overrun I use LLM to identify which subsystem owns the bytes and Memory Insights to find why those bytes are allocated, retained or churned. If the finding is texture streaming, UObject retention or native churn, the fix path is different.

**Question: How do you profile under thermal or mobile constraints?**

> I avoid short desktop/editor runs. I use target hardware, power mode, realistic duration, device profile, resolution and thermal state. I track frame-time percentiles over time, not only the first cool minute. I separate CPU, GPU, memory and I/O, then decide whether the fix is optimisation, scalability, workload scheduling or content budget.

### Hands-on specialist extension

Extend Project 4:

1. Build three deterministic scenarios: front-end/UI churn, combat/VFX peak and streaming/travel.
2. Run each in a packaged target build under two device profiles and two scalability tiers.
3. Collect CSV runs for each scenario with warm-up and fixed measurement windows.
4. Capture one short Insights trace for the worst CPU hitch, one Memory Insights capture for a repeated growth case and one GPU/platform capture for a GPU-bound case.
5. Add LLM or equivalent memory-budget checkpoints for boot, map loaded, peak and return-to-menu.
6. Create one intentional regression in CPU, memory and GPU; prove the CSV/device matrix catches each.
7. Write a performance gate proposal with metric, threshold, artifact, owner and false-positive mitigation.

For a deeper target-branch proof that combines this gate with release automation and World Partition/HLOD builder evidence, use [[ue_packaged_performance_build_worldpartition_proof|Packaged Performance, BuildGraph, and World Partition Proof]] and [[ue_packaged_performance_build_worldpartition_workbook]].

### Specialist common bugs

1. Comparing editor viewport before with packaged build after.
2. Reporting average FPS without frame-time percentiles or hitch count.
3. Capturing all trace channels for minutes and changing the workload.
4. Treating a wait scope as the work source.
5. Raising memory pools without proving content/subsystem ownership.
6. Assuming device-profile source file means runtime profile selection.
7. Letting low scalability remove required gameplay readability.
8. Failing CI on noisy one-run deltas.
9. Using CSV as causal proof instead of a regression signal.
10. Ignoring thermals, power mode, storage, shader/PSO warm-up or cold-cache behaviour.

### Conflict and uncertainty notes

- Maintained tool pages may surface UE5.7 UI. Trace channels, commands, and panels must be checked in UE5.6.
- Stats/timing include measurement overhead and synchronisation effects; cross-check with target platform tools for high-stakes platform decisions.
- “Common” optimisation guidance is conditional. This chapter preserves Epic's caveat that event-driven updates can lose to batched Tick when changes are excessively frequent. [SRC-PERF-005]
