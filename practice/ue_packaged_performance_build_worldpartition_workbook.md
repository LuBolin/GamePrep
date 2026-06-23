# Packaged Performance, BuildGraph, and World Partition Proof Workbook

See also: [[ue_packaged_performance_build_worldpartition_proof]], [[ue_profiling_optimisation]], [[ue_build_modules_plugins_tools]], [[ue_world_partition_large_world_pipeline]].

Use this workbook to turn the proof chapter into hands-on evidence. Each lab should produce a small artifact. The goal is not to complete a massive release pipeline; it is to learn how to prove target behaviour with clean, reproducible evidence.

Sources: [SRC-PERF-003] [SRC-PERF-009] [SRC-PERF-010] [SRC-BUILD-015] [SRC-BUILD-016] [SRC-BUILD-017] [SRC-BUILD-018] [SRC-ASSET-010] [SRC-WORLD-003]

## Lab 1 - Target Automation Audit

**Goal:** establish what the target branch can actually run.

**Tasks:**

1. Capture `RunUAT` or internal wrapper help/output for build, cook, stage, package, deploy and run.
2. Capture BuildGraph documentation or a dry-run/small-run log if BuildGraph is used.
3. Capture World Partition builder commandlet availability for the target map.
4. Capture CSV/trace/memory command availability in the packaged target config.
5. Record unsupported or replaced commands explicitly.

**Acceptance criteria:**

- A one-page audit table exists.
- Every command used later points back to this audit.
- Unsupported items are labelled with evidence rather than omitted.

**Injected failure:** use a command copied from another UE version and show how the audit catches it before a long build.

## Lab 2 - Release Manifest

**Goal:** create a manifest that can identify one package without relying on memory.

**Tasks:**

1. Fill `manifest.yaml` with engine/project revision, target, platform, config, build ID and archive root.
2. Add map/cook scope and plugin list.
3. Add symbol/crash route.
4. Add scenario IDs and threshold file path.
5. Store the manifest next to logs and package output.

**Acceptance criteria:**

- Another engineer can identify the package, symbols, command line and scenarios from the manifest alone.
- Manifest fields match actual logs.

**Injected failure:** change the build ID in only one artifact and write the detection rule.

## Lab 3 - Minimal RunUAT Package Proof

**Goal:** prove that the project packages from a clean workspace or clean agent.

**Tasks:**

1. Run a documented BuildCookRun or equivalent wrapper command.
2. Archive the command line and logs.
3. Launch the package outside the editor.
4. Record first causal error if packaging fails.
5. Keep cook/stage/archive manifests where available.

**Acceptance criteria:**

- Package exists at the manifest path.
- Package launches outside editor.
- Logs show build/cook/stage/package phases or equivalent wrapper phases.

**Injected failure:** remove one required cooked asset rule and prove the package gate fails for the right reason instead of silently cooking everything.

## Lab 4 - BuildGraph Artifact Flow

**Goal:** model release operations as dependencies instead of one hidden script.

**Tasks:**

1. Create or sketch a graph with nodes for build, cook, package, smoke, performance, symbols and crash proof.
2. Identify inputs and outputs for each node.
3. Add dependencies where downstream nodes require artifacts.
4. Add artifact existence checks after each key node.
5. Write how the graph would be split across pre-submit, nightly and release-candidate lanes.

**Acceptance criteria:**

- `PerfGate` cannot run before `Package`.
- `ForcedCrashProof` depends on both package and matching symbols.
- Artifact paths are stable and include build ID.

**Injected failure:** remove a dependency between package and performance gate; explain how stale artifacts could pass and how the graph should prevent it.

## Lab 5 - Packaged Performance Scenario

**Goal:** turn a gameplay path into a repeatable performance gate.

**Tasks:**

1. Define scenario ID, owner, map, warm-up, readiness markers and sample window.
2. Prove active Device Profile and key CVars in packaged runtime.
3. Run three CSV captures.
4. Summarise P50/P95/P99 frame time, hitch count, Game/Draw/GPU if available and memory/LLM totals if available.
5. Write thresholds and owner.

**Acceptance criteria:**

- Gate report has repeated runs and no editor-only evidence.
- The threshold is actionable and has a tolerance/noise policy.
- CSV files are linked from the report.

**Injected failure:** enable dynamic resolution, VSync or wrong profile so the gate is misleading; record how runtime CVar proof catches it.

## Lab 6 - Causal Trace Escalation

**Goal:** use expensive traces only when needed.

**Tasks:**

1. Pick one failing metric from Lab 5 or inject a regression.
2. Capture a short Insights, Memory Insights, GPU or platform trace for the failing window.
3. Align the trace with scenario markers.
4. Classify root cause: CPU, Draw/RHI, GPU, memory, I/O, shader/PSO, streaming, HLOD transition or platform wait.
5. Make one fix or write a justified owner handoff.

**Acceptance criteria:**

- The trace window is bounded and relevant.
- The report explains why this tool was selected.
- The finding maps to a subsystem/content owner.

**Injected failure:** capture all trace channels for a long run and compare artifact size/overhead with the targeted capture.

## Lab 7 - HLOD Builder Proof

**Goal:** prove that HLOD output is built, packaged and fresh.

**Tasks:**

1. Run the target branch HLOD builder command for the map or document why it is unavailable.
2. Archive the command, log and output report.
3. Change one source actor/material relevant to HLOD.
4. Prove the builder detects or regenerates expected output.
5. Confirm output is included in packaged build.

**Acceptance criteria:**

- Builder log names the correct map and completes with a clear status.
- Output freshness is tied to source revision or timestamp.
- Packaged build uses the expected output.

**Injected failure:** ship stale HLOD output and write a gate that detects it.

## Lab 8 - World Partition Traversal Proof

**Goal:** prove large-world runtime behaviour in a package.

**Tasks:**

1. Define a traversal route and one fast-travel destination.
2. Log cell/layer/readiness state at scenario boundaries.
3. Wait for streaming and project-specific collision/gameplay readiness before fast travel.
4. Capture CSV or trace around the route.
5. Record memory, loaded cell count, HLOD transition and worst hitch.

**Acceptance criteria:**

- Packaged logs show traversal start/end and readiness decisions.
- Teleport does not commit before destination readiness.
- HLOD transition has visual and metric evidence.

**Injected failure:** teleport immediately without readiness and diagnose from logs/traces.

## Lab 9 - Forced Crash And Symbol Proof

**Goal:** prove that release artifacts can diagnose a packaged crash.

**Tasks:**

1. Trigger a controlled packaged crash in a non-editor build.
2. Verify the report includes build ID, platform/config, log tail and callstack.
3. Verify symbols match the exact archived package.
4. Record the crash route or privacy-safe local process.
5. Add a recurrence check to the graph or release checklist.

**Acceptance criteria:**

- Crash can be matched to the package manifest.
- Callstack is symbolicated or the reason it cannot be is documented.
- Symbols from another build are rejected.

**Injected failure:** point the crash process at the wrong symbols and show the mismatch.

## Lab 10 - Final Evidence Review

**Goal:** turn labs into a five-minute interview answer.

**Tasks:**

1. Build a pass/fail table for all gates.
2. Link every pass/fail decision to an artifact.
3. List unsupported branch features and accepted risks.
4. List recurrence guards added.
5. Write a concise answer for: "How did you prove the branch was release-ready?"

**Acceptance criteria:**

- The answer mentions build identity, package, automation graph, performance gate, World Partition/HLOD and crash proof.
- The answer distinguishes signal from causal traces.
- The answer does not overclaim beyond the evidence.

