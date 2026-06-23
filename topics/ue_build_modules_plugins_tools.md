# Unreal Build Systems, Modules, Plugins, and Editor Tools

See also: [[ue_assets_loading_cooking]], [[ue_packaged_performance_build_worldpartition_proof]], [[ue_device_lab_automation]], [[ue_build_farm_failure_workbook]], [[ue_hands_on_projects]].

This chapter targets a three-year Unreal engineer who should be able to design clean runtime/editor boundaries, diagnose compile/link/load/package failures, and build safe content tooling. Specialist depth begins with custom build infrastructure, complex third-party/platform integration, BuildGraph/CI at scale, and deep editor framework extensions.

Version target: **UE5.3–UE5.6**. Build-rule properties, descriptor fields, module host types/loading phases, Live Coding, editor APIs, Python and utility features are version-sensitive.

## 1. The build pipeline: name the failing stage

Unreal's build is a chain of distinct systems:

1. **UBT (UnrealBuildTool)** reads targets/modules/descriptors and constructs the compile/link build graph.
2. **UHT (UnrealHeaderTool)** parses supported reflection declarations and generates glue/metadata code.
3. The platform **C++ compiler** compiles translation units.
4. The **linker** resolves symbols and produces binaries.
5. Unreal's **module/plugin loader** discovers compatible binaries/descriptors and loads modules at configured phases.
6. The **cooker/packager** selects target modules/content/configuration and assembles a distributable build.

UBT invokes and coordinates several stages, but it is not the C++ compiler, UHT, or linker. UHT is not a general C++ compiler. IDE solution/project files help editing and invoke UBT; UBT's `.Target.cs` and `.Build.cs` graph is authoritative. [SRC-BUILD-001] [SRC-BUILD-007]

### Failure classification

| Symptom | Likely stage | First evidence |
|---|---|---|
| reflection macro/generated-header parse error | UHT | first UHT diagnostic, reflected declaration/include order |
| unknown type/include error | compiler/dependency visibility | first compiler error, include/IWYU and Build.cs |
| unresolved external symbol | linker/export/dependency/definition | missing definition, API macro, linked module/library |
| module binary missing/incompatible | loader/build ID/descriptor | load log, target/module type, binary output/version |
| works Editor, fails Game/Server/Shipping compile | target/module boundary | Target.cs, editor dependency leakage, conditionals |
| works compiled but asset/tool absent packaged | cook/stage/plugin descriptor | package logs, plugin/module target support, content rules |
| crash after Live Coding/shutdown | reinstancing/stale cache/lifetime | reload delegates, old instance/static pointer, clean restart |

Read from the first causal diagnostic. Cascading errors after a generated header or missing include are often noise.

## 2. Targets versus modules

### Target

A target describes an application being built: Game, Client, Server, Editor, or Program, plus global rules/configuration. A `.Target.cs` class derives from `TargetRules`; UBT evaluates it to decide the target's overall composition and environment. [SRC-BUILD-004]

Examples:

- `MyGameTarget`: standalone/cooked game;
- `MyGameEditorTarget`: editor plus project/editor modules;
- `MyGameServerTarget`: dedicated server without client-only code;
- a Program target: standalone UE-based utility.

Target **configuration**—Debug, DebugGame, Development, Test, Shipping—is another axis. “Development Editor” combines configuration and target; do not treat it as one indivisible mode.

### Module

A module is a separately built functionality unit with a `.Build.cs` `ModuleRules` class, C++ sources, public API and private implementation. Modules form a directed dependency graph. They can encapsulate runtime features, editor tools, libraries, third-party integrations or programs. [SRC-BUILD-002] [SRC-BUILD-003]

A target contains/links/loads a set of modules. A plugin can contain multiple modules. A module is not a C++20 module.

### Practical split

For a reusable inventory-tools plugin:

- `InventoryRuntime`: runtime definitions/transactions/components; no UnrealEd/Slate editor dependencies.
- `InventoryEditor`: validators, asset actions, details customisation, Editor Utility bridge; depends on Runtime.
- optional `InventoryTests`: automation tests/fixtures according to project policy.

The dependency arrow should not point from Runtime to Editor.

## 3. `.Build.cs`: compile and link contract

`ModuleRules` configures module dependencies, include paths, definitions, libraries, platform conditions and build properties. It is C# executed by UBT at build-graph construction time, not code shipped as gameplay C#. [SRC-BUILD-003]

### Public versus private dependency

- **Public dependency:** types/includes from the dependency appear in this module's public headers/API, so consumers need that dependency transitively.
- **Private dependency:** used only in private headers/`.cpp` implementation; consumers should not inherit it.

Prefer private dependencies where the public contract does not expose the type. Forward declarations and pImpl/private implementation can reduce public include/dependency propagation.

Example reasoning:

```csharp
PublicDependencyModuleNames.AddRange(new[] { "Core", "CoreUObject" });
PrivateDependencyModuleNames.AddRange(new[] { "Engine", "Slate", "SlateCore" });
```

This is correct only if public headers expose no `Engine`/Slate types requiring consumers to compile. Copying a dependency list from another module until compilation succeeds hides the real API graph.

### Public and Private folders

- `Public/`: headers intentionally consumable by dependent modules.
- `Private/`: implementation and private headers visible only inside the module.
- `.cpp` files belong in `Private/`.

These folders are module visibility conventions, unrelated to C++ `public/private/protected` member access. [SRC-BUILD-002]

### Include paths are not dependencies

An include path makes a file discoverable; a module dependency supplies compile environment and link relationship. Manually adding another module's private include directory is an encapsulation violation and often breaks installed builds/engine upgrades.

### API export macros

A public class/function/data symbol crossing a dynamic-module boundary may need the module's generated `MYMODULE_API` macro. If headers compile but consumers get unresolved externals, check:

- definition actually exists with exact signature;
- owning module is a dependency and linked for the target;
- symbol is exported/imported appropriately;
- template/inline implementation visibility;
- configuration/platform preprocessor conditions;
- stale binary is not being loaded.

Not every symbol needs export—private implementation should remain private. Monolithic builds can hide missing export mistakes that modular Editor builds reveal, or vice versa.

## 4. IWYU, forward declarations, and build performance

Include What You Use means each file includes what it directly needs rather than relying on transitive/precompiled/unity includes. Forward-declare when only pointer/reference declarations need an incomplete type; include the full definition where size, inheritance, inline member access, templates or destruction semantics require it.

### Why unity builds mislead

Unity builds combine source files to reduce overhead. An unrelated neighbouring include can make a translation unit compile accidentally. Non-unity, clean and different-order builds expose missing includes/ODR problems. Treat unity as optimisation, not dependency semantics.

### Public-header cost

A heavy include in a widely used public header fans out recompilation. Useful interventions:

- narrow public interfaces/value DTOs;
- forward declarations;
- move implementation to `.cpp`/private class;
- split a stable lightweight core module from volatile integrations;
- avoid inline implementation that requires huge engine/editor headers;
- measure clean/incremental build time before creating many tiny modules.

More modules can improve incremental isolation but add link/load/descriptors/API boundaries. Split by ownership/reuse/target dependency, not one module per folder.

## 5. UHT and generated code

UHT scans supported reflected declarations and generates code consumed by compilation. Reflected headers normally include their matching `.generated.h` last among includes. UHT errors commonly involve unsupported syntax around reflected members, missing macros, bad specifiers, generated-header placement or types unavailable to UHT. [SRC-BUILD-007]

`GENERATED_BODY` injects generated declarations; `.gen.cpp`/generated outputs are build artefacts and should not be hand-edited. Reflection changes can alter generated code, class layout, Blueprint exposure and serialisation; they are high-risk Live Coding changes even if a patch appears to succeed.

UHT does not replace the compiler's type checking and is not a full preprocessor/compiler. A declaration can pass UHT then fail C++, or fail UHT before the compiler sees useful code.

## 6. `.uproject` and `.uplugin` descriptors

Descriptors are JSON-like metadata declaring modules, plugin dependencies, target/platform compatibility, content support and other discovery/build/load settings.

### Plugin shape

```text
Plugins/InventoryTools/
  InventoryTools.uplugin
  Source/
    InventoryRuntime/
      InventoryRuntime.Build.cs
      Public/
      Private/
    InventoryEditor/
      InventoryEditor.Build.cs
      Private/
  Content/                 # only when CanContainContent is intended
  Config/
  Resources/
```

Engine plugins live under the engine and can serve many projects but complicate engine upgrades/distribution. Project plugins live with the project and are easier to version/reproduce with it. Promote to engine-level only when ownership/reuse/release policy justifies it.

### Code versus content plugin

A plugin may contain code modules, content, or both. `CanContainContent` is a descriptor contract, not merely a folder convention. Content introduces cook/reference/chunk/migration concerns. A runtime module must not quietly depend on editor-only plugin modules.

### Module type and loading phase

Module type controls which hosts/targets may load it—commonly Runtime versus Editor, with Developer/Program and other exact host types being version-sensitive. Loading phase controls *when* a compatible module starts. Default fits most modules. Earlier phases are for real startup dependencies, not a generic fix for missing build dependency or incorrect registration. [SRC-BUILD-002] [SRC-BUILD-005]

### Startup and shutdown symmetry

An editor module may register menus, tabs, detail customisations, asset actions, styles, commands, validators and delegates during startup. It must unregister/release them during shutdown where the owning module/system is available.

Watch for:

- stale delegates after module reload;
- duplicate registration after Live Coding;
- shutdown order when dependency module already unloaded;
- static shared pointers/styles surviving reload;
- callbacks running after editor/module teardown.

Use `IsModuleLoaded`/shutdown guards as appropriate to the API and target version; do not force-load modules merely to unregister during teardown.

## 7. Runtime versus editor separation

Editor modules may depend on runtime modules. Runtime modules must remain buildable without UnrealEd, PropertyEditor, AssetTools, Blutility and other editor-only modules.

### `WITH_EDITOR` is not the boundary

`#if WITH_EDITOR` can conditionally compile editor-only helpers inside otherwise runtime types where appropriate. It does not legitimise a runtime `.Build.cs` dependency on an editor module for Shipping. Also distinguish editor-only **code** from editor-only **data** (`WITH_EDITORONLY_DATA` where appropriate and supported). Exact macros/stripping semantics require target validation.

Strong proof:

1. build Game/Client/Server targets, not only Editor;
2. cook/package Development and Shipping;
3. inspect runtime module dependency graph;
4. run packaged application without editor plugins/content;
5. test dedicated server where applicable.

### Shared definitions

If editor tools and runtime need the same asset class/schema, place the class in Runtime (without editor dependencies) and put validation/custom UI/actions in Editor. Avoid placing runtime-loadable UClasses in an Editor module; packaged content referring to them will fail.

## 8. Third-party modules and libraries

An external module rule can describe headers, static/import libraries, delay-loaded DLLs, frameworks and runtime dependencies by platform/configuration. Specialist integration must address:

- ABI/compiler/runtime-library compatibility;
- architecture/platform/configuration selection;
- include isolation and macro/warning conflicts;
- link order/symbol export/name mangling;
- delay loading and staged runtime binaries;
- licence/security/version provenance;
- thread/lifetime/error translation at the UE boundary.

“Header found” proves only preprocessing. Package and launch on every target, including clean machine/runtime dependency tests.

## 9. Live Coding versus Hot Reload versus full build

Live Coding recompiles and patches binaries while the process runs. With object reinstancing enabled, reflected object instances may be replaced for structural changes. This is valuable for function-body iteration but not a substitute for clean process lifecycle. [SRC-BUILD-006]

### Safer iteration categories

| Change | Iteration judgement |
|---|---|
| ordinary function-body logic | good Live Coding candidate; test clean later |
| `.cpp` helper/private implementation | usually reasonable if lifetime/static state is controlled |
| UPROPERTY/UFUNCTION/USTRUCT/UCLASS layout | reinstancing may help, but restart/rebuild and asset/serialisation validation required |
| constructor/default subobject/default changes | existing instances/CDOs/Blueprint defaults may not reflect intended clean-start behaviour |
| module/descriptors/Build.cs/Target.cs/plugin dependency | regenerate/rebuild/restart as required; patch cannot redefine build graph safely |
| static globals/singletons/vtables/destructors | high stale-instance/shutdown risk; clean restart strongly preferred |

When objects are reinstanced, cached raw/static pointers may become stale. Relevant reload/reinstancing completion delegates allow caches to invalidate/rebuild. Live Coding docs specifically warn about destructor/version conflicts during shutdown. [SRC-BUILD-006]

Legacy Hot Reload remains available but is not the preferred baseline. Neither workflow proves clean UHT generation, startup registration, Blueprint asset integrity, config, cook, Server/Shipping, or cold process behaviour.

### “Delete Binaries/Intermediate” judgement

Generated outputs can become stale/corrupt, so removing regenerable build products is a valid targeted recovery after preserving logs. It is not the first answer to every compiler error and never fixes source dependency/design mistakes. Do not delete source/config/content or caches without understanding cost and scope.

## 10. Build and package debugging workflow

### Reproduce precisely

Record engine commit/version, host/target platform, target type, configuration, clean/incremental/unity, installed/source engine, plugin set, command line and first error.

### Work from stage and target

1. Identify UHT, compile, link, module load, cook, stage or runtime.
2. Rebuild the smallest failing target/module with useful verbosity.
3. Read the first causal error and exact command/rule context.
4. Confirm dependency direction and public-header exposure.
5. Compare Editor versus Game/Server/Shipping rule branches.
6. Check descriptors/module host type/loading phase/platform lists.
7. Check exported symbol and third-party runtime staging.
8. Clean only relevant generated artefacts if stale output is evidenced.
9. reproduce from a clean checkout/build agent.
10. add CI target that catches the class of failure earlier.

### Common diagnostic examples

**Header compiles in module A but not consumer B:** B may rely on a transitive/private include. Make A's public header self-sufficient and expose only required public dependencies.

**Unresolved external for a public class:** declaration visible, implementation/export/link dependency missing or configuration-excluded.

**Editor type appears, Shipping fails:** runtime content/code references a class in Editor module, or runtime module depends on editor-only APIs.

**Plugin loads too late:** first prove a legitimate startup dependency and registration phase; do not use `PreDefault` to mask missing module dependency or static-init ordering.

## 11. Choosing an editor automation surface

| Surface | Best fit | Strength | Main limitation |
|---|---|---|---|
| Editor Utility Blueprint/Widget | designer-facing interactive project tools | fast UMG/Blueprint iteration | version/status/performance, weaker large-code maintenance |
| C++ editor module | durable integrated tools/custom asset/editor UX | full APIs, testable architecture | compile/maintenance/version coupling |
| Python editor script | pipeline glue, batch/inter-DCC automation | rapid and ecosystem-friendly | editor-only, experimental/version-sensitive API surface |
| Commandlet | headless repeatable CI/batch process | automation and build-agent fit | reduced/raw environment; no assumed World/UI |
| Data Validation | enforce content invariants on save/manual/CI | standard feedback/CI integration | validation is not automatic repair |
| Details customisation | improve editing of known class/struct properties | contextual UX with property handles | editor module registration/lifecycle and Slate complexity |

Editor Utility Widgets are UMG-based editor tabs and are explicitly version/status-sensitive. Python's Unreal environment is editor-only, not packaged runtime scripting. Commandlets run without a normal game/client/world environment unless explicitly built. [SRC-BUILD-008] [SRC-BUILD-011] [SRC-BUILD-012]

## 12. Safe batch tool architecture

A production batch tool is a transaction pipeline:

1. **Select/query** candidates through Asset Registry/selection/folder filters.
2. **Analyse** without mutation; report counts, dependency impact and invalid/unavailable packages.
3. **Preview/dry run** exact changes and outputs.
4. **Validate preconditions**: class/version, source-control checkout, package writable, naming collision, references.
5. **Transact/mutate** through supported editor APIs (`Modify`, property handles, asset tools), not raw filesystem edits.
6. **Mark dirty/save** deliberately; never silently save unrelated packages.
7. **Report** changed/skipped/failed with stable paths and reasons.
8. **Support cancellation/resume/idempotence** for large jobs.
9. **Validate postconditions** and optionally run Data Validation.
10. **Expose the same core operation** to UI and commandlet/CI when useful.

### Undo and transactions

Interactive changes should use editor transaction APIs where supported so users can undo. Not all operations are perfectly transactional—asset rename/save/source-control/external files need explicit rollback/recovery design. Test multi-select, partial failure and editor restart.

### Source control

Before mutation, account for checkout/add/delete/move, changelist scope, read-only files and multi-user conflicts. Do not auto-check out thousands of assets without preview/confirmation and audit. Source control integration should be an adapter around the operation, not hardwired into domain transformation logic.

## 13. Data Validation

Custom asset classes can override `IsDataValid`; broader validators derive from `UEditorValidatorBase` and decide whether they can validate an asset. Validation can run on save, from editor menus and through a commandlet for CI. [SRC-BUILD-009]

Good validation rules are:

- deterministic and side-effect-free;
- actionable: asset/property plus expected rule and repair hint;
- scoped/fast enough for their trigger;
- classified warning versus error consistently;
- robust to unloaded/missing dependencies and partial projects;
- versioned with schema/content policy;
- tested against valid and invalid fixtures.

Examples: stable ID uniqueness, missing soft target, invalid tag combination, prohibited runtime→editor reference, hard-reference budget, naming/path policy, impossible item values, circular content dependency.

Validation should not silently “fix” source assets during CI. Repair belongs in an explicit tool with preview/transaction/reporting.

## 14. Details customisations, asset actions, and factories

`IDetailCustomization` customises UObject/Actor details; `IPropertyTypeCustomization` customises a struct/property type using property handles and child/header builders. Register in an editor module and unregister on shutdown. Use property handles to preserve multi-edit, transactions, notifications and reset/default semantics rather than writing raw object memory. [SRC-BUILD-010]

Asset actions add type/category/context operations; factories define supported creation/import and initial objects; custom asset editors add deeper editing workflow. Keep the runtime asset class/schema independent from these editor modules.

Choose the lowest-cost intervention:

1. reflection specifiers/metadata/default Details panel;
2. Data Validation;
3. utility action/widget;
4. property/class customisation;
5. custom asset type/editor/factory.

Do not build a custom editor because a metadata specifier would solve the authoring problem.

## 15. Tool testing and performance

Test editor tools as carefully as runtime systems:

- zero/one/many selection and mixed classes;
- unloaded assets, redirectors and missing dependencies;
- read-only/source-control conflicts;
- undo/redo and partial failure;
- cancel/restart/idempotent rerun;
- PIE/editor-world distinctions;
- module reload/shutdown;
- commandlet/headless path;
- clean checkout and CI;
- package Runtime module with Editor module excluded.

Profile Asset Registry query time, asset load count, peak memory, package save count/time, shader/DDC side effects, source-control requests and Slate responsiveness. Process in bounded batches and yield/cancel where interactive responsiveness matters.

## 16. Common misconceptions

| Misconception | Better model |
|---|---|
| “Visual Studio builds the project.” | IDE invokes workflows; UBT rules construct the authoritative Unreal build graph. |
| “UHT is Unreal's compiler.” | UHT generates reflection glue; the platform compiler/linker build C++. |
| “Public dependency means everyone can include it.” | It propagates a compile/link contract because your public API exposes it; keep it private otherwise. |
| “A header in Public exports symbols.” | Header visibility and binary symbol export/link dependency are separate. |
| “`WITH_EDITOR` keeps editor dependencies out of Shipping.” | Module dependency graph must also keep runtime free of editor modules. |
| “Live Coding means no restart.” | Patching/reinstancing accelerates iteration; clean startup/layout/default/asset/package proof still needs restart/rebuild. |
| “Loading phase fixes missing plugin class.” | First check build dependency, descriptor compatibility, binary and registration; phase only addresses legitimate timing. |
| “Editor Utility Widget is a runtime UI.” | It is editor automation UI, version/status-sensitive. |
| “Python can script packaged gameplay.” | UE's documented Python integration is editor-only. |
| “Validation should repair bad assets.” | Validation reports invariants; explicit transactional tools perform repairs. |

## 17. Strong interview answers

### UBT versus UHT

> UBT evaluates Target.cs, Build.cs and descriptors to construct the target/module compile and link graph and invoke tools. UHT parses Unreal's supported reflected declarations and emits generated code/metadata consumed by the compiler. Then the native compiler and linker build binaries. I classify the first error by stage because a UHT generated-header failure, compiler include error and linker unresolved external have different fixes.

### Public versus private dependencies

> A dependency is public when my module's public headers expose types/includes from it, so consumers need that compile contract. If it is only used in cpp/private implementation, it should be private. Public/private folders control module API visibility, not C++ member access. I make public headers self-contained, forward-declare where legal, avoid private include paths and verify non-unity consumer builds.

### Live Coding safety

> I use Live Coding for function-body iteration, with reinstancing enabled as recommended, but treat reflected layout, constructors/defaults, subobjects, static caches/destructors and build/descriptors as restart boundaries. Reinstancing can replace UObjects, so raw/static caches need invalidation on reload. Before committing or diagnosing asset corruption, I reproduce with a clean editor restart and normal target build.

### Tool architecture

> I separate a deterministic analyse/transform core from editor adapters. The interactive front end offers dry run, transaction/undo where supported, source-control preflight, cancellation and structured report. The same core can run through a commandlet for CI. Validation remains side-effect-free and catches bad content early; runtime assets live in a runtime module and all PropertyEditor/AssetTools/UnrealEd dependencies stay in the editor module.

## 18. Hands-on verification tasks

### Build graph lab

Create Runtime and Editor modules. Deliberately leak an editor type into a runtime public header, rely on a unity/transitive include, omit an API macro, and misclassify a dependency. Diagnose each in Editor, Game and Server/non-unity targets.

### Reinstancing lab

Live-code a function body, constructor default, reflected field and cached raw/static pointer. Record existing/new instance behaviour and shutdown. Add cache invalidation on reload, then compare with clean restart.

### Tool pipeline lab

Build one item-definition audit/repair operation with C++ core, Editor Utility Widget front end, Data Validator and commandlet path. Add dry run, transaction, source-control preflight, cancellation, idempotence and report. Package the project proving no Editor module/class enters Runtime.

## 19. Source map

- UBT, targets, modules and rules: [SRC-BUILD-001] [SRC-BUILD-002] [SRC-BUILD-003] [SRC-BUILD-004] [SRC-BUILD-013]
- Plugins/descriptors/module hosts: [SRC-BUILD-005]
- UHT/reflection generation: [SRC-BUILD-007]
- Live Coding/reinstancing: [SRC-BUILD-006]
- Utility UI/Python/commandlets: [SRC-BUILD-008] [SRC-BUILD-011] [SRC-BUILD-012]
- Validation/details customisation: [SRC-BUILD-009] [SRC-BUILD-010] [SRC-BUILD-014]

## 20. Specialist cluster: release automation, BuildGraph, artifacts, and crash proof

The first build/tools chapter teaches how to classify local compile/link/load/package failures. Release specialist depth is different: it is about making the same result reproducible on a clean machine or build farm, separating build/cook/stage/package/deploy/run operations, producing auditable artifacts, preserving symbols and crash evidence, and preventing editor-only success from reaching players. [SRC-BUILD-015] [SRC-BUILD-016] [SRC-BUILD-017]

### AutomationTool, BuildCookRun, and BuildGraph

Unreal release automation usually layers concepts:

```text
UBT/UHT/compiler/linker
    -> build binaries for target/config/platform
Cooker
    -> produce platform-specific cooked content
Stage/package/archive
    -> assemble distributable layout/containers/prereqs
Deploy/run/test
    -> install or execute on target device/environment
AutomationTool / RunUAT
    -> command-line orchestration of those operations
BuildGraph
    -> declarative multi-step CI/build-farm graph using AutomationTool tasks
```

AutomationTool is the command-line automation surface. BuildGraph is a way to describe larger repeatable graphs: properties, nodes, dependencies, agents, labels and artifacts. It is useful when the release pipeline has multiple targets, platforms, cooks, tests, symbol/artifact steps and dependent jobs. It is overkill for a one-off local package command but valuable once the team needs repeatable release branches and build-farm visibility. [SRC-BUILD-015] [SRC-BUILD-016]

### BuildGraph mental model

Think of BuildGraph as a dataflow/control graph, not a shell script:

- **Properties** parameterise branch, project, platform, configuration, version, archive root, signing, maps and optional features.
- **Nodes** perform work such as compile, cook, stage, package, tests, symbol processing or artifact publication.
- **Dependencies** express required previous nodes rather than relying on command order.
- **Agents** describe where work can run and which machine capabilities are needed.
- **Artifacts/labels** expose outputs for later nodes, humans or CI.

Good graph design makes skipped work, reuse and failure isolation explicit. Bad graph design is a giant linear command blob embedded in XML.

### Release pipeline gates

A release candidate should pass gates that are stronger than "Development Editor opens":

| Gate | Evidence | Common failure caught |
|---|---|---|
| Clean sync | fresh checkout/workspace and deterministic dependency setup | local generated files hide missing source/config |
| Clean build | no stale Binaries/Intermediate/DDC dependency for source correctness | transitive include, missing export, stale generated header |
| Target matrix | Game/Client/Server/Editor as relevant, Development/Test/Shipping | editor-only dependency, server/client leak |
| Cook | selected maps/assets/Primary rules/chunks cook for platform | missing asset, redirector, unsupported asset format |
| Stage/package/archive | distributable layout/container/prereqs produced | unstaged DLL/data/plugin content |
| Clean-machine launch | run outside developer editor environment | PATH/runtime dependency, missing redistributable, config path |
| Smoke test | deterministic startup/map/menu/test commandlet/automation | fatal runtime assertion, missing module/class |
| Symbol/artifact upload | exact binaries, symbols, logs/manifests retained | unusable crash reports |
| Crash reporter proof | forced crash produces symbolicated report with build ID/context | broken endpoint/symbol/privacy pipeline |

The gate order matters. A crash pipeline test after packaging but before distribution can save days of unsymbolicated production investigation. [SRC-BUILD-018]

### Artifact contract

Every release pipeline should define what is produced and retained:

- packaged build/archive;
- exact build metadata: engine commit, project commit, branch, changelist, platform, target, config, build ID, version string;
- cooked/staged manifests and chunk/container list;
- UBT/UHT/cook/package logs;
- symbol files and map files where applicable;
- crash reporter configuration and endpoint policy;
- automation test reports;
- PSO/shader/DDC/cache artifacts where project policy requires them;
- signing/notarisation/upload receipts where platform policy requires them.

Artifacts must be associated with the binary that shipped. Symbols from "a similar build" are weak evidence. Logs without the command line and environment often fail to reproduce the build.

### Build/cook/stage/package debugging workflow

Use this when a build works locally but fails in CI or only packaged:

1. Identify the failed operation: build, cook, stage, package/archive, deploy, run, smoke test, symbol upload or crash report.
2. Compare the exact command line, target, config, platform, plugin set, maps, cook mode and archive settings.
3. Reproduce from a clean workspace without local Intermediate/Binaries/DerivedDataCache reliance.
4. Check whether the missing file is source, generated code, cooked asset, staged runtime dependency, platform prerequisite or symbol artifact.
5. Read the first causal UAT/cook/stage error, not the final AutomationTool failure wrapper.
6. Inspect cook/stage manifests and container/chunk output.
7. Verify runtime/editor module boundaries and plugin descriptor target/platform filters.
8. Run a clean-machine launch or target-device smoke test.
9. Convert the failure into a CI gate: commandlet validation, package smoke, dependency audit or artifact check.

Example weak fix: "add the whole Content folder to always cook." That can hide the missing management rule, balloon package size and break chunking. A strong fix identifies the Primary Asset rule, map list, explicit dependency, plugin content policy or staging rule that should include the asset. [SRC-ASSET-005] [SRC-ASSET-006] [SRC-BUILD-017]

### Third-party runtime dependency release proof

External libraries are not release-ready when the compiler finds headers. Prove:

1. static/import library is selected for platform/config/architecture;
2. exported symbols link in modular and monolithic targets as required;
3. delay-loaded DLL/shared object/framework is staged into the package;
4. clean machine has no hidden PATH/install dependency;
5. licence and security provenance are recorded;
6. symbols/debug files are retained as policy permits;
7. crash stack crosses the third-party boundary intelligibly;
8. dedicated server and client targets do not carry unwanted client/server-only dependencies.

The most common failure is "works on developer machine" because the DLL was globally installed or next to the editor, not staged into the build.

### Crash reporting as part of build quality

A crash report is useful only if it can be matched to exact symbols and context. Crash reporting should capture:

- executable/build ID/version/branch/changelist;
- callstack with symbols;
- log tail and error message;
- platform/hardware/RHI/thread;
- gameplay/context tags that do not violate privacy;
- reproduction metadata such as map, mode, command line, net mode or content version;
- whether the crash was editor, client, server, commandlet or packaged.

Release preparation should include a forced-crash smoke test in a non-editor packaged build. Verify the report arrives, is symbolicated, preserves build metadata and routes to the right triage location. If privacy or platform policy restricts uploads, document the approved local/first-party crash flow. [SRC-BUILD-018]

### CI matrix design

Do not run every expensive job on every change. Use a tiered matrix:

| Tier | Trigger | Jobs | Purpose |
|---|---|---|---|
| Pre-submit quick | every PR/change | compile key modules, UHT, unit/automation smoke, validators | catch fast source/content regressions |
| Nightly clean | scheduled/mainline | clean build, non-unity/sample configs, cook/package major platforms | catch hidden generated/cache and target-matrix failures |
| Release candidate | branch/tag | full BuildGraph, packaging, symbols, PSO/cache, smoke tests, crash proof | produce distributable artifacts |
| Platform certification | release window | platform-specific packaging/signing/performance/saves/network policies | meet store/console/mobile requirements |

The answer should mention cost control: split jobs by ownership and feedback time. A five-hour full cook on every tiny C++ change causes teams to bypass CI; a too-thin matrix lets release-only defects pile up.

### Interview answer pattern: build works in editor but packaged build crashes

> I would classify the failed stage first: cook, stage/package, launch or runtime crash. Then I would reproduce from a clean packaged build and inspect logs/manifests. Editor success can hide loose uncooked content, editor-only modules/classes, globally installed DLLs, warm DDC and loaded assets. I would check module/plugin descriptors, runtime/editor dependencies, Asset Manager/cook rules, staged runtime dependencies and crash symbols. The fix should become a CI gate, not just a local cleanup step.

### Interview answer pattern: designing a release pipeline

> I would model it as BuildGraph/AutomationTool jobs: clean sync/build, cook selected platforms, stage/package/archive, run smoke/commandlet tests, publish artifacts and symbols, then verify crash reporting. The graph should expose parameters, dependencies, agents and artifacts instead of a long shell blob. Every artifact is tied to engine/project commit, target/config/platform and build ID. The release is not ready until a clean-machine launch and symbolicated forced-crash proof pass.

### Specialist common bugs

1. Local Binaries/Intermediate/DDC hide a missing source or cook rule.
2. Editor target passes while Game/Server/Shipping fails.
3. Runtime content references classes in an Editor module.
4. Plugin content exists in editor but `CanContainContent`/cook policy/staging excludes it.
5. BuildGraph node order is implicit rather than dependency-driven.
6. RunUAT failure wrapper is read instead of the first cook/stage error.
7. Third-party DLL exists on the developer machine but is not staged.
8. Symbols are not archived with the exact shipped binary.
9. Crash reporting is configured after release instead of smoke-tested before release.
10. CI runs either too much to be used or too little to catch target-only failures.

### Hands-on release automation extension

Extend Project 6:

1. Write a simple BuildGraph or documented RunUAT pipeline with parameters for platform, config, archive root and version.
2. Build Development and Shipping Game targets plus a dedicated Server target if the project supports networking.
3. Cook only the intended maps/assets and verify a deliberately missing Primary Asset is caught.
4. Stage a mock third-party runtime file and prove a clean machine can launch without global installs.
5. Archive build metadata, logs, manifests and symbols together.
6. Add one commandlet/data-validation smoke gate.
7. Force a packaged crash and verify symbolicated crash report with build metadata.
8. Document which steps are pre-submit, nightly and release-candidate jobs and why.

For failure-drill practice, use [[ue_build_farm_failure_workbook]]. It covers RunUAT first-cause analysis, BuildGraph dependency mistakes, clean-agent failures, editor-only Shipping leaks, third-party staging, missing cooked content, symbol mismatch, cache masking, smoke-map coverage, Android toolchain drift and telemetry artifact loss.

### Specialist source caveats

- `SRC-BUILD-015` BuildGraph is the official specialist anchor, but studios often wrap it in internal build-farm conventions.
- `SRC-BUILD-016` and `SRC-BUILD-017` should be checked with target `RunUAT` help/output because flags and platform behaviours change.
- `SRC-BUILD-018` crash reporting is operationally sensitive: symbol upload, privacy, endpoint and platform crash flows are project/studio policy decisions.

## 21. Reproducible package and build-farm proof playbook

This section answers the question many teams fail late: not “can we package once?”, but “can another clean machine reproduce, launch, symbolicate and diagnose the exact build we think we shipped?”. [SRC-BUILD-015] [SRC-BUILD-016] [SRC-BUILD-017] [SRC-BUILD-018]

### Release-manifest minimum contract

Every archived packaged build should carry one compact manifest containing at least:

- engine/project commit or changelist;
- platform, configuration, target type and package/build ID;
- command lines or graph parameters used for build/cook/stage/package;
- SDK/toolchain version and branch-sensitive toggles where relevant;
- cooked map/content scope;
- symbol archive location and crash-reporting route;
- logs/manifests/artifact paths;
- smoke/performance/validation result summary.

A package without this manifest is not really reproducible; it is just a folder of files someone hopes are the right ones.

### Schematic BuildGraph shape for artifact-aware release work

```xml
<!-- Schematic only. Verify node/task names for the exact branch/tooling. -->
<BuildGraph>
  <Node Name="BuildGame">
    <Produces>$(Artifacts)/Binaries</Produces>
  </Node>

  <Node Name="CookContent" Requires="BuildGame">
    <Produces>$(Artifacts)/Cooked</Produces>
  </Node>

  <Node Name="PackageGame" Requires="CookContent">
    <Produces>$(Artifacts)/Package</Produces>
  </Node>

  <Node Name="ArchiveSymbols" Requires="BuildGame">
    <Produces>$(Artifacts)/Symbols</Produces>
  </Node>

  <Node Name="CrashProof" Requires="PackageGame;ArchiveSymbols">
    <Produces>$(Artifacts)/CrashProofReport</Produces>
  </Node>
</BuildGraph>
```

The point is not XML syntax memorisation. The point is that artifact flow must be explicit. If `CrashProof` needs packaged binaries and matching symbols, that dependency should exist in the graph, not in an engineer's memory.

### Cache-busting order for build/package failures

Do not jump straight to deleting everything. Use a bounded order:

1. identify the failing phase and first causal log line;
2. rerun only that phase if the tooling supports it;
3. compare local versus clean-agent inputs: generated files, plugin binaries, staging rules, cook roots, DDC, environment variables and SDK/toolchain;
4. invalidate the smallest plausible cache or output set;
5. only do a full clean rebuild when the evidence says state contamination is broad or the smaller reset failed.

This matters in interviews because “delete Intermediate and try again” is not a debugging method; it is a temporary symptom reset.

### Failure-triage matrix for packaged-build incidents

| Incident | First question | Strong next action |
|---|---|---|
| Packaged build crashes on startup | Is this before engine init, native/plugin load, cooked asset load, map load or gameplay startup? | inspect package manifest, staged runtime files, logs and matching symbols |
| Works locally but not on clean agent | Which dependency existed only on the developer machine? | compare command line, toolchain, local binaries, DDC and staged runtime files |
| Shipping fails, Editor passes | Did runtime code/content depend on Editor modules/classes? | audit Build.cs/module types/runtime assets and prove Game/Server/Shipping package |
| CI red with generic RunUAT wrapper | Which concrete sub-phase failed first? | search upward to first cause, isolate phase and assign owner |
| Crash report exists but is unusable | Do symbols/build IDs/log tails match the exact package? | verify symbol archive, upload route and forced-crash proof |
| Package launches but missing feature/data | Was the content cooked/staged and discoverable, or only present as loose editor data? | inspect cook scope, Asset Manager rules, stage manifests and clean-machine repro |

### Build-farm evidence packet

A useful build-farm or release-failure report should include:

1. failing phase and first causal error;
2. exact artifact identity (package/build ID, branch/commit, platform/config);
3. log/manifests/symbol paths;
4. whether the failure reproduced on a clean rerun;
5. root-cause classification: source, cook content, staging, runtime dependency, crash routing, infrastructure or lab/device issue;
6. smallest recurrence guard: Build.cs fix, cook rule, staging rule, graph dependency, smoke test or crash-proof gate.

This makes the report actionable for another engineer without replaying your whole investigation.

For a combined target-branch packet that ties BuildGraph/RunUAT artifacts to packaged performance gates, World Partition/HLOD builder output and forced-crash evidence, use [[ue_packaged_performance_build_worldpartition_proof|Packaged Performance, BuildGraph, and World Partition Proof]] and [[ue_packaged_performance_build_worldpartition_workbook]].
