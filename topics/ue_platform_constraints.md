# Unreal Platform Constraints, Mobile, and Certification-Ready Release Thinking

## Cluster scope

**Priority:** P2 for engine generalists, build/release, performance, mobile/console and senior gameplay roles; P3 for narrow gameplay roles unless the team ships on constrained platforms.  
**Expected depth:** D3-D4.  
**Version scope:** UE5.3-UE5.6 preferred. Android/iOS setup, packaging, SDK requirements, mobile renderer support, device profiles, scalability groups and crash/reporting pipelines are platform/version-sensitive. Console certification requirements are usually platform-holder confidential; this chapter discusses public engineering patterns and labels certification-specific detail as source-sensitive. [SRC-PLAT-001] [SRC-PLAT-002] [SRC-PLAT-003] [SRC-PLAT-004] [SRC-PLAT-005] [SRC-PERF-007] [SRC-PERF-008]

The goal is to think like a shipping engineer. "It runs in editor" is not a platform answer. A platform-ready build must survive target hardware, cold start, suspend/resume, memory pressure, storage failures, network loss, controller/input changes, entitlement/sign-in edge cases, crash reporting, symbolication, packaging/signing and long-session thermal behavior.

## Platform constraint model

For every target platform, define a contract:

```text
Target SKU / OS / store / hardware tier
  -> build configuration, SDK, signing, packaging
  -> frame, memory, storage, I/O, thermal, battery budgets
  -> renderer/feature/scalability/device profile policy
  -> input, safe area, accessibility and platform UI requirements
  -> suspend/resume, disconnect, storage, network and entitlement failures
  -> crash/log/symbol/artifact workflow
  -> certification or submission checklist
```

Good platform work is evidence-driven. You need target-device captures, not guesses from a developer desktop.

## Mobile constraints

Mobile introduces several hard constraints:

- SoC/GPU tiers vary widely.
- Thermal throttling can change performance after minutes, not seconds.
- Battery and power use matter.
- Memory limits can be lower and less forgiving.
- OS lifecycle events such as backgrounding, foregrounding and interruption are common.
- Input includes touch, gestures, virtual controls, gamepads and platform UI.
- Store packaging, signing, SDK/NDK/Xcode/provisioning requirements can block release before code runs.

Epic provides public Android and iOS/tvOS development documentation and separate packaging/setup pages; treat these as setup anchors, not a substitute for target-device proof. [SRC-PLAT-001] [SRC-PLAT-002] [SRC-PLAT-003] [SRC-PLAT-004] [SRC-PLAT-007] [SRC-PLAT-008]

Mobile performance needs device-profile tiers, not one "Low" preset. The device profile page and scalability reference provide the public mechanism: profiles and scalability settings adjust CVars and quality groups by hardware/platform. [SRC-PERF-007] [SRC-PERF-008]

## Renderer and content policy

A platform plan should classify expensive features:

| Area | Platform decision |
|---|---|
| Resolution | native, fixed percentage, dynamic, TSR/upscale support |
| Shadows | static/baked, cascades, VSM/CSM support, contact shadows |
| Materials | quality switches, shader permutations, texture samples, WPO/translucency restrictions |
| Textures | pool, mip bias, streaming, compression format |
| Geometry | Nanite/LOD/HLOD/ISM/HISM/foliage policy by platform |
| Post/effects | bloom, DOF, AO, SSR, translucency, particles, Niagara budgets |
| UI | not blindly scaled by render resolution; safe area and text readability |

The scalability reference explicitly covers resolution scale, view distance, AA, post process, shadows, textures, effects, detail mode, material quality, skeletal LOD and foliage density; it also warns that gameplay-affecting objects should not be disabled by detail mode or scalability. [SRC-PERF-007]

## Memory, storage, and I/O

Platform readiness requires separate budgets:

- resident memory and peak transient memory;
- texture pool and streaming behavior;
- audio stream/cache memory;
- shader/PSO cache and warmup impact;
- package size and patch delta size;
- save data size and failure policy;
- install/free-space assumptions;
- cold boot, first launch, level load and streaming latency.

Memory failures are rarely fixed by "optimise assets" as a blanket instruction. Use Memory Insights, LLM tags, asset audits, cook/stage manifests and platform memory tools where available. [SRC-PERF-006] [SRC-PERF-010] [SRC-ASSET-005] [SRC-ASSET-006]

## Certification-adjacent failure matrix

Exact console TRC/TCR/XR requirements are confidential and platform-specific. In interviews, do not invent numbers or quote requirements you cannot source. Instead, describe the categories and the engineering workflow.

Common certification-adjacent categories:

- boot, splash, loading, first interactive frame;
- suspend/resume, quick resume, background/foreground;
- controller disconnect/reconnect and input device changes;
- user/profile sign-in changes, guest users and privileges;
- network offline/online transitions and service errors;
- storage full, save failure, corrupted save, cloud conflict;
- DLC/entitlement/package ownership changes;
- platform overlay, achievements/trophies, invites and rich presence;
- privacy, age/rating, parental controls and platform policy;
- crash handling, logs and symbolicated callstacks;
- localisation, safe area, accessibility and text overflow.

Treat each as a testable state machine. The product should either handle the event, block the action gracefully, or exit through a platform-compliant path. "We never tested that state" is the failure.

## Build, package, signing, and artifacts

Platform release needs a reproducible artifact contract:

- engine and project commits;
- target, platform, configuration and SDK/toolchain versions;
- plugins and platform feature flags;
- cook list, maps and asset management rules;
- package/sign/archive parameters;
- staged file manifest and container/chunk manifests;
- symbols and crash reporter configuration;
- smoke-test scenario and launch command;
- known platform caveats and certification checklist version.

Use AutomationTool/RunUAT or BuildGraph for repeatable build/cook/stage/package/archive flows, and include commandlet/data-validation gates before expensive packaging. [SRC-BUILD-015] [SRC-BUILD-016] [SRC-BUILD-017]

## Debugging workflow: "fails only on device"

1. **Identify the exact artifact:** commit, engine, platform, SDK, config, package ID, signing profile, device model, OS version.
2. **Reproduce cleanly:** install from archived package on a clean device; avoid editor-adjacent files.
3. **Classify phase:** install, launch, boot, asset load, shader/PSO, login, map travel, gameplay, suspend/resume, network, save, shutdown.
4. **Collect target logs:** platform logs, UE logs, crash dump/minidump, callstack, last successful milestone.
5. **Check cooked/staged content:** manifest, chunk/container, plugin files, certificates, dynamic libraries, config files.
6. **Check memory/perf:** target profiler, Memory Insights/LLM where available, CSV/Insights captures if the build supports them.
7. **Check platform state:** permissions, storage, network, account, entitlement, controller, background/resume.
8. **Minimise:** build a reduced map/feature flag branch that isolates the platform failure without changing build configuration.
9. **Archive evidence:** logs, symbols, package, repro steps and causal fix.

Do not "fix" device-only failures by testing a different build path.

## Profiling workflow

For platform performance:

1. Define target frame budget, resolution, quality tier, thermal duration and representative content.
2. Use packaged builds, not editor.
3. Capture cold/warm load, 5-minute, 20-minute and worst-case gameplay sessions where relevant.
4. Record FPS/frame time percentiles, hitches, Game/Render/RHI/GPU, memory, texture pool, I/O, battery/thermal signals if available.
5. Compare device profiles and scalability tiers with the same scenario.
6. Verify visual/gameplay acceptance at each tier.
7. Add CSV/performance gates only after noise and device variance are understood.
8. Re-test after packaging, signing, compression, shader/PSO and asset changes.

The target performance gate chapter already covers CSV, Insights, Memory Insights and LLM budget workflow; platform work adds SKU/device/thermal/package reality. [SRC-PERF-009] [SRC-PERF-010] [SRC-PERF-011]

## Android target-profiler escalation

For Android, pair Unreal evidence with Android device evidence:

1. Start with packaged Unreal stats/CSV to identify the failing scenario and frame window.
2. Use Unreal Insights or Memory Insights when engine-side CPU/loading/memory ownership is the question.
3. Escalate to Android system profiling when CPU/GPU scheduling, present waits, thermal behavior, GPU utilization or device memory pressure is the question.
4. Use AGI Frame Profiler when a single representative GPU frame needs render-pass, draw-command, binning, rendering or GMEM load/store analysis.
5. Use Android vitals/LMK evidence for field memory pressure, but reproduce locally with lifecycle and Unreal memory checkpoints.
6. Treat ADPF and the Unreal ADPF plugin as sustained-performance adaptation tools that require a no-adaptation baseline and visible quality/scalability logging.

The Android profiling workbook in `practice/ue_android_platform_profiling_workbook.md` contains the detailed drills. Keep the scope Android-specific unless Apple or console tooling is available in the target environment. [SRC-PLAT-010] [SRC-PLAT-012] [SRC-PLAT-013] [SRC-PLAT-015] [SRC-PLAT-016] [SRC-PLAT-017]

The companion chapter `topics/ue_apple_console_profiling.md` covers public Apple/Xcode/Metal profiling workflows and source-sensitive console profiler answer patterns. Use it when the target is iOS, iPadOS, tvOS, macOS or an authorised console environment. [SRC-PLAT-018] [SRC-PLAT-021] [SRC-PLAT-022] [SRC-PLAT-024]

## Common bugs and misconceptions

1. **Editor performance predicts device performance.** It does not.
2. **Certification is QA's problem.** Engineering owns state handling and evidence.
3. **Low scalability can disable anything.** Do not disable gameplay-significant objects through detail/scalability.
4. **Mobile is just lower settings.** Thermal, power, lifecycle, input and store requirements are different.
5. **One Android/iOS device is enough.** Device tiers and OS/toolchain combinations matter.
6. **Runtime feature supported means product-appropriate.** Support, quality and budget are separate.
7. **Crash report without symbols is enough.** Release artifacts must match exact symbols.
8. **Storage/network/account failures are rare.** They are common certification and live-ops edge cases.
9. **Platform docs are stable forever.** SDK, signing, store and engine requirements change.
10. **Console cert details can be guessed publicly.** Use platform-holder documentation and keep confidential details out of public notes.

## Strong interview answer patterns

### "How would you prepare a UE game for mobile?"

> I would define device tiers, frame and memory budgets, then set device profiles and scalability rules per tier. I would use packaged builds on target devices, capture thermal-length sessions, control resolution, texture pool, material quality, shadows, effects and foliage, and verify UI safe areas/input. I would also lock SDK/signing/package requirements and keep crash/log/symbol artifacts for every build.

### "What is certification readiness?"

> Certification readiness means the build handles platform-required state transitions and failure cases, not just that gameplay works. I would build a confidential checklist from platform-holder docs, then test suspend/resume, input disconnect, account changes, storage errors, network loss, overlays, localisation/safe area, crash handling, save corruption and package entitlement paths. Each case needs expected behavior, evidence and an owner.

### "A packaged Android build crashes on launch; what do you do?"

> I identify the exact package, device, OS, SDK/NDK, config and commit; reproduce from a clean install; collect platform and UE logs; classify whether it fails before engine init, plugin load, asset load, shader/PSO, map load or gameplay startup; check staged native libraries/config/assets; symbolicate if it crashes; then produce a reduced repro without changing the build path.

### "How do device profiles and scalability fit together?"

> Device profiles select platform/device-specific CVars and settings, while scalability groups expose quality tiers for features such as resolution, shadows, textures, effects, view distance, material quality and foliage. I use profiles to choose sane defaults per hardware tier and scalability to give controlled quality/performance trade-offs, then validate both on target hardware.

## What a three-year engineer should know

Know why editor success is insufficient; how device profiles, scalability groups and packaged target profiling fit together; how to reason about mobile renderer/memory/thermal constraints; what kinds of platform state transitions certification checks; how release artifacts, symbols, logs and packaging manifests support diagnosis; and how to answer without inventing confidential console requirements.

**Specialist depth:** platform-holder certification docs, console SDK/RHI/profiler tools, store submission pipelines, patch/DLC systems, crash reporter endpoints, privacy/age/commerce compliance, platform-specific online subsystem behavior, render feature matrices and automated device-lab infrastructure.

## Hands-on verification task

Build the Project 4/6 Platform Release Gate extension:

1. Pick two device/platform tiers and define frame, memory, startup and package-size budgets.
2. Create device profile/scalability settings for the tiers.
3. Package a clean build and archive logs, symbols, manifests and metadata.
4. Run a cold-start, 5-minute and stress scenario on target device or closest available hardware.
5. Inject storage/network/account/input/suspend-style failures where the platform allows.
6. Force a packaged crash and verify symbolication.
7. Produce a certification-adjacent matrix with event, expected behavior, evidence and owner.

## Source caveats

- Public Epic docs cover Android/iOS setup, packaging, mobile rendering/performance, device profiles and scalability, but do not replace platform-holder certification documentation.
- Console certification details are intentionally not enumerated here. Use confidential platform docs in a real project.
- Mobile SDK, NDK, Xcode, provisioning, store and device-profile behavior change over time; verify the exact target versions before implementation.
