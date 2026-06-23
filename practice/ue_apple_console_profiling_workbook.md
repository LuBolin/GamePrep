# Apple and Console Profiling Workbook

See also: [[ue_apple_console_profiling]], [[ue_platform_constraints]], [[ue_device_lab_automation]], [[ue_profiling_optimisation]].

Use this workbook only with public Apple tooling or authorised console tooling. Do not place confidential console counters, screenshots, commands, SDK names or certification text into public study artifacts.

Source anchors: [SRC-PLAT-018] [SRC-PLAT-019] [SRC-PLAT-020] [SRC-PLAT-021] [SRC-PLAT-022] [SRC-PLAT-023] [SRC-PLAT-024] [SRC-PERF-003] [SRC-PERF-006] [SRC-PERF-009] [SRC-RENDER-019]

## Lab 1: Apple capture packet

**Goal:** produce a repeatable Apple target capture packet that can be compared before/after.

**Build:**

1. Package a UE build for an Apple device or macOS target.
2. Record device model, OS, Xcode, UE branch, RHI, build config, package ID, map, scalability and scenario.
3. Add UE CSV/Insights/log markers around the scenario.
4. Capture one Unreal trace and one Apple/Xcode/Instruments/Metal artifact for the same scenario.

**Acceptance criteria:**

- evidence names the exact frame/phase;
- Unreal and Apple captures share marker vocabulary;
- result includes one finding, one change and one before/after comparison;
- simulator-only evidence is explicitly labelled non-final.

## Lab 2: Simulator versus device memory

**Goal:** prove why simulator memory is not iOS memory proof.

**Build:**

1. Run the same map/load/transition scenario in simulator and on device.
2. Record Xcode memory gauge, UE LLM/Memory Insights and peak asset/render target pools.
3. Force one memory spike with a controlled asset load or render target allocation.

**Injected failures:**

- accept simulator green memory gauge as pass;
- forget to record high-water memory;
- optimise asset pool without checking native/platform allocations.

**Acceptance criteria:**

- report explains simulator/device difference;
- high-water and retained memory are separated;
- one memory reduction is validated on device;
- UE and Xcode memory evidence are reconciled.

## Lab 3: Metal GPU capture map-back

**Goal:** map one Metal debugger finding back to a UE/content hypothesis.

**Build:**

1. Use Unreal tools to identify a GPU-bound frame.
2. Capture that frame with Xcode GPU Frame Capture.
3. Inspect performance timeline, pipeline grouping, counters or shader profiling.
4. Map the finding to a UE pass, material, render target, scene capture, VFX or geometry system.

**Injected failures:**

- capture a different frame than the UE spike;
- ignore capture overhead/performance state;
- replay on a different GPU/OS and call it target proof;
- fix a shader in the debugger but forget the source/content change.

**Acceptance criteria:**

- capture metadata is complete;
- finding is expressed in both Metal and UE terms;
- change is validated by normal UE GPU timing after capture;
- replay compatibility caveat is documented.

## Lab 4: Thread Performance Checker triage

**Goal:** identify platform/native responsiveness problems without misusing the tool as a UE GameThread profiler.

**Build:**

1. Add or mock a native wrapper/plugin call used during login, platform service init or menu open.
2. Intentionally block on main thread or create a priority inversion in the wrapper path.
3. Run with Xcode Thread Performance Checker.
4. Repair the wrapper by making work asynchronous or matching QoS/lifetime correctly.

**Acceptance criteria:**

- issue is classified as native/platform responsiveness, not generic UE gameplay cost;
- before/after shows the checker warning or profiler wait removed;
- UE marker shows the gameplay/menu phase affected;
- tests fail or warn when the runtime issue is reintroduced.

## Lab 5: MetricKit field signal to local reproduction

**Goal:** turn aggregated field metrics into a reproducible profiling task.

**Build:**

1. Use a MetricKit or Organizer-style report category such as hitch, launch, memory, GPU/display, termination or hang.
2. Break it down by app version, OS/device class and scenario marker if available.
3. Convert it into a local reproduction plan using UE/device lab tools.

**Acceptance criteria:**

- report distinguishes fleet prioritisation from frame-level root cause;
- reproduction task states device/OS/app/content version;
- local capture either explains the issue or records why field-only evidence is insufficient;
- signposts/UE markers are proposed for the next release if attribution is weak.

## Lab 6: Source-safe console GPU investigation

**Goal:** practise console GPU investigation without recording confidential details.

**Build:**

1. Use an authorised console environment if available; otherwise create a source-safe mock based on categories only.
2. Record package/build ID, symbols, scenario, UE GPU evidence and a redacted platform-tool finding category.
3. Map the finding to a UE/content hypothesis.

**Acceptance criteria:**

- no confidential tool names, commands, counters or platform requirements appear in public files;
- workflow still names pass/shader/bandwidth/render-target/content categories generically;
- before/after gate is recorded with source-safe metrics;
- confidential artifacts have access-controlled storage outside the public curriculum.

## Lab 7: Console certification-adjacent lifecycle matrix

**Goal:** answer certification readiness questions without inventing requirements.

**Build:**

1. Create a matrix of lifecycle events: suspend/resume, controller disconnect, account change, network loss, storage full, package update, invite/session change and crash/report path.
2. For each event, list source of truth: authorised platform docs, project policy or public engineering convention.
3. Record owner, expected behaviour, evidence path and automation possibility.

**Acceptance criteria:**

- exact cert text is not quoted in public material;
- every real requirement points to authorised docs in private workflow;
- source-safe matrix still helps engineering triage;
- one lifecycle event gets an automated or semi-automated test plan.

## Lab 8: Unified platform evidence memo

**Goal:** write a senior-level profiling memo from multiple evidence streams.

**Build:**

1. Choose one issue: Apple memory spike, Metal GPU bottleneck, console-only hitch or platform lifecycle failure.
2. Include UE evidence, platform evidence, metadata, hypothesis, change and validation.
3. Add one section titled "What this evidence does not prove."

**Acceptance criteria:**

- finding is not overgeneralised across devices/platforms;
- metadata is enough for another engineer to rerun;
- conclusion maps back to a UE/content owner;
- residual risk and next capture are explicit.

