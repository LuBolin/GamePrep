# Lua and C# Integration for Unreal-Adjacent Interviews

See also: [[cpp_for_unreal_interviews]], [[ue_cpp_idioms]], [[ue_build_modules_plugins_tools]], [[ue_hands_on_projects]], [[cpp_interview_question_bank]].

Lua and C# are not first-class gameplay languages built into Unreal Engine in the way C++ and Blueprint are. They appear through third-party runtime plugins, studio-owned embedders, editor/build tools, services and cross-engine teams. The interview skill is therefore not memorising one plugin API: it is designing and debugging a boundary among two runtimes, two object models, two garbage collectors, generated/reflected bindings, platform packaging and gameplay authority.

Version target: **UE5.3–UE5.6**. Every runtime path in this chapter is **plugin-dependent** and must be pinned to an exact engine/plugin/runtime commit. UnLua and sluaunreal describe UE4/UE5 Lua integration; UnrealCLR describes a .NET 6/C# 10 integration snapshot. Puerts' Unreal path is **JavaScript/TypeScript**, not Lua or C#; it is included only as an adjacent generated/reflection binding comparison. No third-party README is treated as an Epic support guarantee.

## 1. Why add a scripting/managed runtime?

Potential benefits:

- faster iteration and reload for product logic;
- data-driven missions/UI/live-event logic owned by a broader team;
- safer or more constrained mod/plugin surface;
- reuse of existing language skills/libraries/tooling;
- remote content or patching where platform policy permits;
- C# editor/build/content tools or services around Unreal;
- a stable high-level API that isolates game logic from native engine churn.

Costs:

- another runtime, allocator/GC, debugger, build and crash surface;
- wrapper generation/reflection coverage gaps;
- cross-boundary marshalling and callback overhead;
- native object invalidation hidden behind apparently live script objects;
- platform/JIT/AOT/code-signing and package constraints;
- hot-reload state/schema migration;
- weaker source-level access to non-reflected engine functionality;
- hiring, ownership and long-term plugin maintenance risk.

Adopt from a concrete iteration/ownership/modding/tooling requirement. “The language is easier” is not enough to justify a second runtime in a performance-sensitive shipped product.

## 2. A language-neutral integration model

```text
UE native world and authoritative gameplay
  UObject / reflection / delegates / Tasks / packaging
                ↕ binding layer
  generated wrappers / dynamic reflection / manual native API
                ↕ marshalled values + object handles + callbacks
script or managed runtime
  VM/CLR, modules/assemblies, heap/GC, exceptions, debugger
                ↕
script gameplay, UI/presentation, tools or content logic
```

Every exposed call should answer:

1. Who owns the native object?
2. Who keeps the wrapper/managed object alive?
3. How is native destruction observed?
4. What thread may call it?
5. Are arguments copied, converted, pinned or borrowed?
6. Can it yield/throw/callback across the boundary?
7. What happens on reload, travel, shutdown and package?
8. What is the cost per call/allocation?

## 3. API surface design

Prefer a narrow, semantic façade over exposing all engine internals.

Good boundary calls resemble:

- `RequestActivateAbility(StableId, TargetData)`;
- `QueryInventorySnapshot()` plus change subscriptions;
- `SetObjectiveState(ObjectiveId, State)` through server validation;
- `SpawnCosmeticEffect(EffectId, Transform, Params)`;
- editor tool operations over `FAssetData`, validation reports and explicit transactions.

Poor boundaries expose raw pointers, mutable Components, thread-affine internals or chatty per-element calls. Batch data, use stable IDs and keep native invariants in native code.

### Reflection versus generated/manual bindings

- **dynamic reflection:** broad access to `UFUNCTION`/`UPROPERTY`-visible APIs with low setup; runtime lookup/conversion and incomplete C++-only access;
- **generated/static wrappers:** type-aware fast calls and IDE declarations; generation/build/version maintenance and code size;
- **manual bindings:** precise stable façade and best control; implementation effort and risk of inconsistent coverage.

Many production integrations combine them: reflected baseline, generated hot paths and manual semantic APIs.

## 4. Lua language model for engine interviews

Lua 5.4 is a small embeddable dynamically typed language implemented as a C library. Its official manual defines values including nil, booleans, numbers, strings, functions, userdata, threads/coroutines and tables; tables are the central associative container and metatables customise operations. [SRC-SCRIPT-001]

The exact Lua version/runtime used by a plugin matters. Lua 5.1, LuaJIT and Lua 5.4 differ in language, libraries, bytecode and GC behaviour. Never assume the latest reference manual describes a plugin's embedded VM.

### Tables and metatables

Tables implement arrays, maps, objects, namespaces and modules by convention. Key interview points:

- only `nil` and `false` are false; `0` and empty string are true;
- assigning `nil` removes a table key;
- array-like length is only straightforward for sequences without holes;
- table keys use equality/identity semantics; mutable object wrappers need stable policy;
- `__index` can delegate lookup to a table/function, enabling prototypes/inheritance-like patterns;
- `__newindex`, arithmetic/call/string/equality metamethods can hide expensive/native work;
- iteration order should not be treated as deterministic gameplay order.

Avoid clever metatable chains at the engine boundary. Explicit modules and façade objects are easier to profile and invalidate.

### Functions, closures and modules

Functions are first-class values. Closures capture upvalues, which can retain large tables/native wrappers after the visible owner is gone. Modules loaded by `require` are cached in `package.loaded`; reload must decide whether to replace the module table, mutate it in place or migrate existing instances.

### Errors

Lua errors unwind to a protected boundary. Host code should use protected calls (`pcall`/equivalent plugin wrapper), capture traceback/context and convert failure into a product-safe result. An unprotected C API error can reach Lua's panic path and abort. [SRC-SCRIPT-001]

Do not let Lua errors cross arbitrary C++ frames with live RAII assumptions unless the embedding/plugin explicitly preserves them. Establish one protected call gateway.

### Coroutines

Lua coroutines are cooperative, not OS threads. `yield` suspends a Lua execution stack; Unreal work still runs on whatever thread resumes it. A coroutine is not permission to touch UObjects from a worker thread.

Yielding across C API calls has restrictions; Lua's continuation-aware API exists because a yield may remove C stack frames. [SRC-SCRIPT-001] Plugin async wrappers must define cancellation on owner destruction, World travel and VM shutdown.

## 5. Lua C API awareness

The Lua C API passes values through a virtual stack associated with `lua_State`. A native function reads arguments at indices and pushes return values. [SRC-SCRIPT-001]

### Stack discipline

At each boundary:

- validate argument count/type;
- use absolute indices when pushes may shift negative indices;
- restore/check stack top on every return/error path;
- avoid retaining borrowed string pointers after their stack value/lifetime ends;
- use registry references for Lua values retained by native code;
- release registry references symmetrically;
- make error/yield behaviour explicit.

A stack leak is not C++ heap leakage; it is unbalanced VM state that may corrupt later calls.

### Userdata and native handles

Full userdata is Lua-managed storage with metatables/finalisers; light userdata is essentially an unmanaged pointer identity. Neither makes the pointed UObject safe.

A robust wrapper usually stores an indirection/handle that can be marked invalid when native destruction occurs. Every call checks validity and World/thread context. A Lua `__gc` finaliser releases wrapper-side registrations; it does not own or destroy arbitrary UObjects unless the API explicitly transferred ownership.

## 6. Lua GC versus Unreal GC

Lua GC traces Lua objects. Unreal GC traces UObject references known through reflected/native reference mechanisms. Neither collector automatically understands every edge in the other heap.

### Failure patterns

- Lua wrapper remains reachable after UObject destruction → stale native call;
- UObject is kept alive by binding-layer reference while Lua code believes it released it → native retention;
- native delegate stores Lua closure, closure captures wrapper, wrapper/binding roots native object → cross-runtime cycle;
- `__gc` timing is nondeterministic and too late for delegate/timer teardown;
- reload creates new wrappers while old callbacks remain registered;
- Lua table caches world objects across seamless/non-seamless travel.

### Correct pattern

1. Define native ownership independently of Lua reachability.
2. Binding creates a wrapper/handle and registers destruction invalidation where supported.
3. Calls validate the native handle and expected World/thread.
4. Explicit `Dispose`/`Unbind`/scope shutdown removes delegates, timers and registry references.
5. Lua finalisation is a safety net for wrapper resources, not product lifecycle.
6. VM shutdown invalidates all wrappers before native services disappear.

The Lua manual notes GC only sees Lua reachability and never collects values retained in the registry; tuning is non-portable across versions/platforms. [SRC-SCRIPT-001]

## 7. Unreal Lua ecosystem snapshots

### UnLua

Tencent's UnLua repository describes a Lua scripting plugin supporting Unreal 4.17.x through 5.x. [SRC-SCRIPT-002] It aims to expose reflected Unreal APIs and support Lua implementation/extension of gameplay classes.

Treat exact UE5.3–UE5.6 compatibility, embedded Lua version/backend, binding mode, hot reload, GC hooks, delegate support and packaging as commit-specific. Inspect release tags, CI/build files and target platform code before adoption.

### sluaunreal

Tencent's sluaunreal repository describes a Lua development plugin for Unreal 4 or 5 with gameplay/hot-fix goals. [SRC-SCRIPT-003] It is a distinct architecture and API from UnLua despite shared organisation history; examples from one are not transferable.

### Puerts correction

Puerts' official Unreal manual describes JavaScript/TypeScript integration: reflected APIs are broadly exposed, non-reflected C++ can use manual/static binding, and VM/class-generation/hot-reload workflows are provided. [SRC-SCRIPT-004]

Puerts is relevant to compare generated declarations, reflection and VM lifetime, but its Unreal path is **not a Lua or C# integration**. Current primary project material says Unreal supports JavaScript/TypeScript while newer Lua/Python backends are for Unity. Do not list Puerts as an Unreal Lua/C# plugin. [SRC-SCRIPT-004]

### Plugin evaluation matrix

For any candidate record:

- exact UE branches/tags and last tested commit;
- Lua/VM version and licence;
- reflected/generated/manual coverage;
- UObject/delegate/container/latent/coroutine semantics;
- wrapper invalidation and cross-GC rooting;
- game-thread guarantees;
- editor/package/server/console/mobile support;
- debugger/profiler/source maps;
- hot reload/state migration;
- cook/stage content and bytecode/source policy;
- maintainer/community/upgrade evidence;
- benchmark and fallback/removal strategy.

## 8. C# roles around Unreal

C# can matter even when runtime gameplay remains C++/Blueprint:

- build/release/CI and content-pipeline tools;
- external import/export/process automation;
- backend/services, telemetry and test harnesses;
- companion launchers or dedicated operational tooling;
- teams moving between Unity and Unreal;
- runtime gameplay through a third-party .NET host plugin.

Ask whether the requirement is **inside the game process**. An out-of-process C# tool/service avoids embedding CLR, native wrapper and platform shipping risk.

## 9. C# and .NET fundamentals for integration

### Value and reference types

C# structs are value types; classes, arrays, delegates and strings are reference types. This is not simply “stack versus heap”: boxing, captures, arrays and containing objects affect storage. Mutable large structs copied across interop boundaries are risky; use small explicit-layout/blittable records where practical.

### Managed GC

.NET GC traces roots such as statics, thread stacks/registers, GC handles and finalisation queues, and can compact live objects. Allocation/survival rate drives collection behaviour. [SRC-SCRIPT-006]

A native pointer to managed memory may become invalid if GC moves the object. Pinning prevents movement but can fragment/impair GC; minimise duration. A `GCHandle` used as callback context must be freed explicitly. [SRC-SCRIPT-007]

### Deterministic native-resource release

Managed GC does not know how to release arbitrary native resources. Use `IDisposable`/`using` and preferably `SafeHandle` for native handles; finalisation is a fallback, not deterministic teardown. [SRC-SCRIPT-008]

A managed wrapper for a UObject should not call `delete`; it releases its binding handle/delegate registrations and becomes invalid when Unreal destroys the object.

### Delegates and callbacks

When native code retains a managed callback:

- use an exact calling convention/signature and stable function pointer/delegate;
- root the delegate/managed target for as long as native may call;
- unregister natively before freeing the GC handle;
- marshal callback to Game Thread before touching UObjects;
- catch exceptions before they cross into unmanaged frames;
- make shutdown/reload reject late callbacks.

### Exceptions

Do not allow managed exceptions to escape an unmanaged callback boundary. Catch, attach script/system/object context, log managed stack and return a native error result. Native failures should become explicit managed exceptions/results at a controlled façade, not arbitrary HRESULT/boolean ambiguity.

## 10. Native interop and marshalling

### Blittable and copied data

Blittable types have compatible bit representation and avoid conversion. Microsoft recommends fixed-layout blittable structures where possible and highlights traps such as C# `bool` not matching C/C++ `bool` by default. [SRC-SCRIPT-007]

Define ABI explicitly:

- integer width/signedness;
- struct layout/alignment/packing;
- boolean width;
- enum underlying type;
- UTF-8/UTF-16 and string ownership;
- array pointer/count and who frees;
- Unreal `FString`/`FName`/`FText` conversion semantics;
- vector/transform precision and coordinate conventions;
- UObject handle identity, never raw unmanaged lifetime assumptions.

### Call granularity

Cross-runtime calls add lookup, type checks, wrapper, conversion and exception/callback machinery. Avoid per-entity/per-particle/per-property ping-pong. Prefer snapshots, batched commands and event deltas.

Benchmark four layers separately:

1. pure native baseline;
2. empty boundary call;
3. realistic value marshalling;
4. full gameplay workload and allocations/GC.

### Runtime hosting

Embedding modern .NET means native code hosts a runtime, loads runtime configuration/assemblies and resolves managed entry points. Microsoft's hosting APIs use `nethost`/`hostfxr`; a plugin may abstract this but inherits deployment/version concerns. [SRC-SCRIPT-010]

## 11. UnrealCLR snapshot

The UnrealCLR repository describes a third-party Unreal integration embedding .NET 6 with C# 10/F# 6 and a managed runtime build/publish step. [SRC-SCRIPT-005]

This is evidence of one architecture, not proof of current support for every UE5.3–UE5.6 platform. Before adoption verify:

- exact Unreal fork/plugin commit and maintained activity;
- .NET/runtime version/security servicing;
- exposed API coverage and generated/manual wrappers;
- UObject identity/invalidation and delegate lifetime;
- assembly load/unload/reload semantics;
- Game Thread/async Task policy;
- editor, packaged game and dedicated-server deployment;
- Windows/Linux/console/mobile restrictions;
- crash dump and mixed managed/native debugging;
- JIT/AOT/code-signing/platform-store policy;
- source distribution/licensing and fallback plan.

Do not say “Unreal supports C#” without the qualifier “through a particular third-party or studio integration”.

## 12. JIT, AOT and packaging

A desktop development runtime may use JIT and dynamic assembly loading. Consoles/mobile/store policies may restrict executable memory/runtime code generation or require AOT. Native AOT is not a magic plugin switch: Microsoft's current documentation lists restrictions including no dynamic assembly loading, no runtime code generation/Reflection.Emit, required trimming and platform-specific builds/debug limitations. [SRC-SCRIPT-009]

An integration designed around dynamic reflection/hot reload may conflict fundamentally with AOT/trimming. Decide platform strategy before building the gameplay architecture.

Package checklist:

- runtime native libraries for each architecture/configuration;
- managed assemblies/dependencies/runtimeconfig/deps files;
- script source or compiled bytecode policy;
- staging/cook rules and case-sensitive paths;
- symbols/source maps and crash-report privacy;
- platform signing/entitlements and executable-memory policy;
- dedicated server/editor exclusion/inclusion;
- licence notices and third-party vulnerability updates;
- clean-machine offline install test.

## 13. Hot reload and state migration

Code reload and state reload are different.

### Safe reload transaction

1. stop accepting new script work;
2. cancel/suspend coroutines/Tasks and unregister callbacks;
3. serialise only explicit reloadable state with schema/version;
4. invalidate wrappers/old function handles;
5. load/validate new modules/assembly in isolation;
6. migrate/rebind state to stable native IDs;
7. restore subscriptions and resume;
8. rollback or restart cleanly on failure.

Replacing a Lua module table does not update closures/instances that captured the old table. Reloading a .NET assembly may be impossible in the same load context or leave native callbacks to old code. A production “hotfix” also faces platform content/code policies and multiplayer version compatibility.

Never patch authoritative logic on only some peers/servers without protocol/content version coordination.

## 14. Threading and async

Default rule: script/managed gameplay that touches UObjects executes on Game Thread unless an API explicitly guarantees otherwise.

- Lua coroutine is cooperative scheduling, not parallelism.
- C# `Task` continuation may resume on a pool thread without a custom/game-thread synchronisation context.
- native async callbacks may arrive after World/VM/assembly shutdown;
- captured UObject wrappers require invalidation and thread hop;
- do not block Game Thread waiting synchronously for script/managed work that waits back on Game Thread.

Use immutable/copied data for worker computation, then enqueue a bounded result with owner/world/generation checks.

## 15. Sandboxing and security

Embedding code is not automatically sandboxing.

For untrusted/mod/remote scripts:

- expose an allow-list façade rather than full reflection;
- omit dangerous Lua `io`, `os`, `package.loadlib` and debug access unless required;
- constrain filesystem/network/process/native-library APIs;
- enforce CPU/instruction/time and memory quotas;
- prevent infinite coroutine/task/event fan-out;
- validate every server-side gameplay request independently;
- sign/version content and audit update provenance;
- treat native plugin/interop access as escaping the sandbox;
- separate trusted studio scripts from untrusted mods.

Lua debug hooks/instruction budgets can help but have overhead and bypass considerations. CLR code with broad framework/native interop access is not a secure mod sandbox merely because it is managed.

## 16. Networking and determinism

Scripted gameplay follows the same authority model:

- server validates/owns durable state;
- clients send intents, never trusted outcomes;
- stable network IDs replace VM pointers/wrappers;
- replicated schemas are native/versioned contracts;
- prediction must correlate and reconcile;
- iteration/hash/GC/reload/timing must not be assumed deterministic;
- dedicated server runs without editor, UI or client-only script runtime where configured.

If scripts define replicated content, all peers need compatible IDs/schema/version. Replicate semantic state, not a serialised VM heap.

## 17. Debugging workflow

### “Object is valid in script, crash in native”

1. Log wrapper ID, native path/name/serial or stable handle, World and runtime generation.
2. Verify native destruction invalidated the wrapper.
3. Inspect cross-runtime roots and delegate/timer registrations.
4. Check callback thread and travel/reload/shutdown timing.
5. Reproduce with delayed callback and forced GC in both runtimes.
6. Fail wrapper calls safely after invalidation; fix owner/unbind lifecycle.

### Binding mismatch

1. Pin plugin/engine/runtime commit and regenerate wrappers/declarations.
2. Inspect reflected signature, export metadata and generated code.
3. Verify ABI layout, bool/string/enum/container direction and ownership.
4. Reduce to one value/return call.
5. Compare editor versus packaged symbols/staged generated files.

### Reload-only bug

1. Log VM/assembly generation on every object/callback.
2. Find old closures/delegates/Tasks and module/assembly contexts.
3. Test explicit unbind/serialise/invalidate/rebind transaction.
4. Reject callbacks whose generation does not match current runtime.

### Mixed crash evidence

Capture native call stack/minidump plus Lua traceback or managed exception/stack and runtime/plugin versions. Neither stack alone explains the boundary. Include last boundary call IDs and marshalling schema.

## 18. Profiling and optimisation

Measure:

- VM/CLR startup, module/assembly load and wrapper generation;
- boundary calls by direction/type and time;
- marshalling bytes/allocations/copies;
- script/managed CPU by function/module;
- Lua heap/GC steps/pauses or managed allocation rate/generations/GC pause;
- wrapper/handle/delegate/registry-reference counts;
- native UObjects retained by the binding layer;
- coroutine/Task/timer count and cancellation;
- reload time, residual old-generation objects and package size;
- dedicated-server and low-end-platform impact.

Optimise architecture first:

- move hot inner loops and invariant enforcement native;
- batch queries/commands and cache stable binding metadata;
- reduce temporary tables, strings, boxing and managed allocations;
- avoid reflection on every call when generated/manual binding fits;
- pool only with complete lifetime/reset semantics;
- schedule GC based on measured allocation/latency, not folklore;
- keep script API coarse and event-driven;
- compare iteration-time benefit against runtime/shipping cost.

## 19. Decision framework

Score each candidate 1–5 with evidence:

| Dimension | Key question |
|---|---|
| product need | What workflow/content/modding problem cannot C++/Blueprint/tools solve adequately? |
| engine coverage | Which reflected and C++-only APIs are reachable and safe? |
| lifetime | Are wrapper invalidation, delegates, travel/reload/shutdown proven? |
| performance | Are representative calls, GC and workload within target budgets? |
| platforms | Do every shipping architecture/store/server/configuration build and run? |
| debugging | Can mixed stacks, source maps/symbols and packaged errors be investigated? |
| iteration | Is reload fast and state migration trustworthy? |
| security | Is the trust/sandbox/update model explicit? |
| maintenance | Who owns engine/runtime upgrades, CVEs and abandoned-plugin risk? |
| exit strategy | Can logic/API be migrated if the integration fails? |

Good uses often include high-level missions/UI/events, tools and constrained mod APIs. Poor uses include physics/rendering inner loops, arbitrary UObject access from workers, or replacing native foundations solely to avoid learning C++.

## 20. What a three-year engineer should know

Expected D1–D3:

- Unreal C++/Blueprint versus third-party scripting status;
- Lua tables/metatables/closures/coroutines/errors/C API stack awareness;
- Lua GC versus UObject GC and wrapper invalidation;
- reflection/generated/manual binding trade-offs;
- C# value/reference types, managed GC, `IDisposable`/handles/delegates;
- blittable/marshalling/call-granularity awareness;
- Game Thread and async callback rules;
- hot reload as state migration, not file replacement;
- platform/JIT/AOT/package constraints;
- debugging mixed stacks and profiling boundary/GC costs;
- plugin evaluation rather than brand memorisation.

Specialist D4–D5:

- custom Lua allocator/C API binding and coroutine scheduler;
- generated wrapper/compiler tooling and ABI verification;
- CLR hosting/load contexts/AOT/trimming and mixed-mode diagnostics;
- secure mod sandbox and resource quotas;
- cross-runtime memory/cycle leak tooling;
- network-compatible live update/schema migration.

## 21. Common misconceptions

| Misconception | Better model |
|---|---|
| “Lua/C# objects keep UObjects alive safely.” | Separate collectors need explicit rooting/invalidation policy. |
| “Lua coroutine is a background thread.” | It is cooperative execution resumed on a host thread. |
| “`__gc`/finalizer is destructor.” | Collection timing is nondeterministic; use explicit teardown. |
| “Managed means no memory bugs.” | Handles, native resources, callbacks, pinning and stale wrappers still fail. |
| “Reflection exposes all Unreal C++.” | Only supported reflected surface; C++-only API needs wrappers. |
| “Hot reload swaps code.” | Existing state, closures, handles and callbacks require migration. |
| “One interop call is cheap.” | Measure realistic call frequency, conversion and allocation. |
| “AOT solves console/mobile support.” | AOT/trimming remove dynamic features and remain integration/platform specific. |
| “Puerts is an Unreal Lua/C# plugin.” | Its Unreal integration is JavaScript/TypeScript. |
| “Unreal supports C#.” | A particular third-party/studio integration may host .NET; Epic baseline remains C++/Blueprint. |

## 22. Strong interview answer patterns

### Lua GC versus Unreal GC

> They trace different heaps. A reachable Lua wrapper can point at a destroyed UObject, and a binding-layer native reference can retain a UObject after Lua drops the wrapper. I define native ownership separately, invalidate wrappers on native destruction, unbind delegates/timers explicitly, use Lua finalisation only as a fallback and test travel/reload with forced GC on both sides.

### Evaluate a C# runtime integration

> I start with the product need and exact UE/platform matrix, then audit runtime/plugin activity, API coverage, generated/manual bindings, UObject invalidation, Game Thread policy, assembly reload, JIT/AOT, packaging, mixed debugging and CVE/upgrade ownership. I prototype a vertical slice and measure boundary calls, allocations/GC and package deployment. C++ still owns hot/invariant/authoritative foundations, and I require an exit strategy.

### Design the scripting boundary

> I expose coarse semantic commands and snapshots with stable IDs, typed units and explicit errors rather than raw mutable engine objects. Every wrapper has ownership, invalidation, thread and reload-generation rules. Calls are protected, callbacks cancellable, exceptions cannot cross native frames, and package/server/platform tests are acceptance criteria.

## 23. Project 10: Scripting Integration Lab

Research and, where feasible, prototype one Lua and one C# path without committing the production project to either.

### Shared native façade

Implement a small C++ module exposing:

- stable `FItemId`/`FObjectiveId`-style IDs;
- read-only inventory/objective snapshots;
- validated command requests and change events;
- one async operation with cancellation/generation;
- explicit bind/unbind/runtime-shutdown hooks;
- no raw ownership transfer or direct authoritative mutation.

### Lua track

1. Pin Lua version and one candidate plugin commit (UnLua or sluaunreal) with licence/platform table.
2. Implement module, table/metatable object, closure callback and coroutine delay.
3. Bind façade snapshots/commands/events; measure reflection versus generated/manual hot call if supported.
4. Destroy native objects, travel, force Lua/UE GC and verify wrappers fail safely.
5. Reload module with schema change; detect old closure/generation and migrate or clean restart.
6. Restrict dangerous libraries and impose a simple instruction/time/memory policy for a mod-style mode.
7. Package Development/Shipping and dedicated server on one clean target.

### C# track

1. Pin one runtime option—e.g. UnrealCLR snapshot—or explicitly choose an out-of-process/editor-tool path if runtime support is infeasible.
2. Document .NET/runtime/plugin/UE/platform matrix and deployment files.
3. Bind façade values, native handle wrapper and callback with explicit layout/signature.
4. Implement `IDisposable`/safe handle or plugin-equivalent teardown; force managed/UE GC and native invalidation.
5. Run a Task off-thread over copied data, then marshal the result to Game Thread with cancellation/generation.
6. Catch managed exceptions at boundary and capture mixed managed/native stack context.
7. Package on a clean machine; evaluate JIT/AOT/trimming/store constraints rather than assuming support.

### Comparative evidence

Benchmark native baseline and each feasible integration for:

- 1, 1,000 and 100,000 empty/value/batched calls;
- snapshot marshalling and callback fan-out;
- allocation/GC pressure and retained wrappers/handles;
- startup/reload and package size;
- editor versus packaged/dedicated-server behaviour;
- debugging time for stale handle, wrong thread, binding mismatch and old-generation callback.

Deliver the API schema, two lifetime diagrams, plugin/version/source matrix, security/package checklists, mixed-stack debugging evidence and an adopt/reject/limited-use decision memo. A well-evidenced rejection is a successful result.

## 24. Source map

- Lua language/C API/GC/coroutines: [SRC-SCRIPT-001]
- Unreal Lua ecosystems: [SRC-SCRIPT-002] [SRC-SCRIPT-003]
- adjacent Puerts binding comparison: [SRC-SCRIPT-004]
- UnrealCLR ecosystem snapshot: [SRC-SCRIPT-005]
- .NET GC/native resources/interop: [SRC-SCRIPT-006] [SRC-SCRIPT-007] [SRC-SCRIPT-008]
- .NET hosting/AOT/deployment: [SRC-SCRIPT-009] [SRC-SCRIPT-010]
