# Unreal Asset References, Loading, Cooking, and Streaming

Asset correctness is a four-part contract:

1. **Discovery/identity:** how code finds the asset without accidentally loading it.
2. **Cook/package inclusion:** why the target build contains it.
3. **Load/residency:** when bytes/objects become available and what keeps them resident.
4. **Use/lifetime:** how callers survive latency, cancellation, failure, unload, travel and platform constraints.

A reference type solves only part of that contract.

Version target: **UE5.3–UE5.6**. Asset Manager configuration, async APIs, IoStore, Zen/DDC, cook rules, World Partition and profiling panels are version/platform-sensitive.

## 1. Packages, assets, objects, and dependency closure

An Unreal asset is a serialised top-level object in a package, often with subobjects and references to other packages/assets. Loading one root can load a **dependency closure**: its hard-referenced class defaults, materials, textures, sounds, meshes and nested definitions.

The important budget is therefore not merely “this Data Asset is 2 KB” but “what becomes loaded because this reachable root is loaded?” A lightweight menu Blueprint can accidentally retain combat classes and their presentation trees through hard class/default references.

Use Reference Viewer, Size Map/Asset Audit tools available in the branch, Asset Manager audits, memory traces and packaged loading evidence to inspect closure. Folder organisation does not define runtime dependency; serialised references and cook management do.

## 2. Hard versus soft references

### Hard reference

A reflected object/class property or other serialised direct reference generally causes the target to load with its referencer. This is correct when the dependency is always needed together and eager availability is valuable. [SRC-ASSET-001]

Hard references are not “bad”. A character's required collision/animation class dependency or a small always-used icon may be appropriate. The danger is an uncontrolled transitive closure from high-level roots.

### Soft object reference

`TSoftObjectPtr<T>` stores typed indirect identity (backed by a soft object path) and can resolve if loaded or be loaded on demand. `FSoftObjectPath` is untyped path identity. A soft pointer does not keep an unloaded target resident like a hard GC reference and does not magically initiate loading when read. [SRC-ASSET-002]

### Soft class reference

`TSoftClassPtr<T>` identifies a class asset expected to derive from `T`; after load, it resolves to a class suitable for spawning/constructing where appropriate. It is often preferable to a hard `TSubclassOf` when optional classes should not join the root's eager dependency closure.

### Weak is not soft

- `TWeakObjectPtr`: observe an object instance that may be destroyed; no asset path for later load.
- `TSoftObjectPtr`: identify an asset/object by path across unloaded state and load it on demand.
- `TObjectPtr`/hard UPROPERTY: reflected in-memory reference that can retain a UObject and create an eager asset dependency when serialised.

### Path/string cautions

Prefer engine-supported soft references/asset IDs over arbitrary strings. They integrate with editor pickers, redirects and cooking rules better. Paths remain identity coupled to content organisation; Primary Asset IDs or project-stable IDs can provide a domain identity layer.

## 3. Synchronous versus asynchronous load

### Synchronous load

A synchronous load blocks the calling thread until dependencies are available. It is acceptable during controlled loading screens, startup phases with explicit budget, tools/commandlets, or rare non-interactive paths. It is dangerous in input, Tick, replication callbacks, UI hover, combat spawn or other latency-sensitive code.

“It is cached on my machine” is not evidence. Test cold DDC/storage/cache scenarios and packaged target hardware.

### Async load lifecycle

A robust async request has:

- requested soft paths/Primary IDs and bundle intent;
- an owning scope/request generation;
- progress/pending/failure/cancelled/ready state;
- a retained handle or other explicit residency owner;
- callback that uses weak/validated owner and World identity;
- cancellation/ignore policy for travel, screen close, replacement request and shutdown;
- fallback presentation/behaviour;
- instrumentation for queue, I/O, deserialisation and game-thread completion work.

`FStreamableManager`/Asset Manager async loading can retain loaded assets through a handle/request. Older documentation notes temporary retention until completion; if gameplay needs residency afterward, keep a documented strong reference or managed handle. Never assume the local soft pointer itself owns residency. [SRC-ASSET-002]

### Completion is not necessarily cheap

Moving file reads off the game thread does not make all work asynchronous. Package creation, UObject serialisation/fixups, resource creation, shader/texture work and callback-side spawning can still cause main/render-thread spikes. Correlate loading and CPU/GPU/memory traces.

### Race-resistant callback pattern

At completion:

1. verify request generation/token is still current;
2. verify owner/world has not ended or changed;
3. resolve each requested asset and handle partial failure;
4. establish required residency ownership before releasing temporary handle;
5. commit one state transition;
6. notify presentation/consumers;
7. release/cancel obsolete requests safely.

This prevents an earlier slow request overwriting a newer selection.

## 4. Asset Registry: query metadata without loading

The Asset Registry stores/searches metadata such as package path, asset name/class and tags and returns `FAssetData`. It enables editor/tools/runtime discovery without loading every candidate. `FARFilter` combines path/class/tag criteria. [SRC-ASSET-004]

Important traps:

- `FAssetData::GetAsset()` loads the asset if needed; a metadata query can silently become a load if converted indiscriminately;
- editor registry discovery can be asynchronous and incomplete until scan completion;
- searchable tags reflect saved asset headers and may require resave after schema/tag changes;
- editor-only metadata is not automatically a runtime gameplay database;
- querying “all assets” repeatedly is not a hot-path design.

Use Asset Registry for discovery/auditing; use Asset Manager for managed primary identity/loading/cook policy where appropriate.

## 5. Asset Manager, Primary Assets, and bundles

The Asset Manager distinguishes:

- **Primary Asset:** directly identified/managed by `FPrimaryAssetId` (`Type:Name`), with rules and load/unload control;
- **Secondary Asset:** included/loaded through references or management by a Primary Asset.

`UPrimaryDataAsset` supplies Primary Asset identity and bundle support. Projects configure Primary Asset types and scan paths/classes. [SRC-ASSET-003]

### Why Primary IDs matter

They give domain-addressable identity independent of holding a loaded UObject pointer. They support discovery, audit, cook/chunk rules and managed loading. They do not make arbitrary save compatibility automatic: renames/type changes still need project policy/migration.

### Bundles

Bundles group soft references within a Primary Asset by use phase, such as `UI`, `Game`, or `Actor`. Loading the definition alone can remain cheap; requesting `UI` may load icon/model preview, while `Game` loads runtime classes/effects.

Bundle names are project contracts. Define inclusion and release semantics, test mixed bundle requests, and avoid assuming recursive behaviour without verifying the target implementation/configuration.

### Primary Asset Labels and chunks

Labels/rules can assign management/cook/chunk policy to sets of assets. Chunks support separable distribution such as base content, maps, heroes, DLC or patches. Conflicting ownership/priorities and shared dependencies require audit; folder placement is not sufficient proof. [SRC-ASSET-006]

## 6. Cooking versus packaging

These terms are not synonyms:

- **Build:** compile code/plugins/binaries for the target.
- **Cook:** convert assets to target-platform runtime formats and select included content.
- **Stage:** copy binaries/cooked content/config/prerequisites into a staging layout.
- **Package:** bundle a distributable application/content containers.
- **Deploy:** transfer to target.
- **Run:** execute on target. [SRC-ASSET-005]

Cook inclusion can arise from reachable references, maps/config lists, Asset Manager rules/labels, directories or explicit project settings. Exact rules vary. An asset existing in Content Browser does not mean it is cooked.

### “Works in editor, missing in package” workflow

1. Reproduce in a clean packaged Development build and capture exact path/ID/log.
2. Verify spelling/case and object versus package/class path.
3. Determine whether discovery uses registry/Primary ID/config or arbitrary runtime string.
4. Inspect cook logs/manifests/container output and Asset Manager audit for inclusion/chunk.
5. Trace the intended management/reference chain.
6. Check editor-only flags/folders/modules/data stripping.
7. Check plugin enabled-for-target and content packaging settings.
8. Validate redirectors and case-sensitive target behaviour.
9. Fix the management/reference rule; avoid a blanket “cook all content” workaround unless product policy truly requires it.

### Pak and IoStore awareness

UE builds may use Pak archives and/or IoStore containers (`.utoc`/`.ucas`) depending on version/platform/settings. These affect layout, patching, chunking, I/O and inspection tools, but the interview baseline is conceptual: cook creates platform data, staging assembles it, containers distribute it, and manifest/chunk rules decide availability. Do not hard-code one archive assumption for every UE5 target.

## 7. Redirectors, moves, migration, and source control

Moving/renaming an asset leaves a redirector at the old location so unloaded packages can still resolve it. Fix-up resaves referencing packages to the new target and removes the redirector when possible. [SRC-ASSET-007]

Failure patterns:

- redirector chains accumulate;
- package cannot be checked out/resaved;
- asset name collides with an old redirector;
- arbitrary string/config paths bypass expected handling;
- redirect works in editor but cook/package policy exposes a stale path;
- cross-project migration omits dependencies or brings unwanted closure.

Move through the editor, commit moved asset + redirector/reference resaves coherently, run project validation/fix-up in source-control-aware workflows, and test the cooked target. Do not delete redirectors blindly from disk.

## 8. Derived Data Cache is not source content

Many assets produce platform/settings-derived data—compiled shaders, texture/mesh derived forms and similar outputs. DDC stores regenerable results locally/shared/remotely so teams avoid repeating expensive work. It should generally not be treated as authoritative source content or checked into source control. [SRC-ASSET-008]

A cold/misconfigured DDC can cause long editor startup, shader compilation, cook time or runtime-development hitches; deleting DDC may diagnose corruption but forces regeneration and is not a routine performance fix. Record backend configuration, key/version invalidation and shared-cache health. Exact Zen/DDC topology is UE-version-specific.

## 9. World and level streaming

Level streaming loads/unloads level packages and controls visibility to bound memory and support seamless worlds. World Partition automates spatial cell streaming around streaming sources for large UE5 worlds and integrates with Data Layers, OFPA and HLOD. [SRC-ASSET-009] [SRC-ASSET-010]

### Architectural questions

- What is the streaming source and preload horizon?
- Which Actors are spatially loaded versus always loaded?
- Which cross-cell Actor references force bundling/co-loading?
- Who owns persistent gameplay state when cell Actors unload?
- How are async tasks, delegates, timers and weak references cancelled on unload?
- How are teleport/fast travel destinations preloaded and readiness confirmed?
- What are memory, I/O bandwidth and concurrent-cell budgets?
- How do server and each client choose relevant loaded regions?

Do not store durable quest/inventory/session truth only in spatial Actors that may unload. Persist stable records in the correct authority/lifetime, and reconstruct world representation as cells load.

### Streaming bugs

- teleport before destination collision/gameplay is ready;
- hard cross-cell references defeat streaming granularity;
- async callback writes to unloaded Actor/old World;
- “always loaded” manager references the whole map;
- rapid source movement causes thrash;
- HLOD visual proxy appears while required gameplay Actor is absent;
- server/client loading assumptions are conflated.

## 10. Texture streaming and virtual texture relation

Texture streaming controls mip residency; asset loading controls UObject/package availability. A loaded texture asset can still have only selected mips resident. Virtual textures tile texture data with their own page/cache behaviour. Diagnose blurry texture, pool over-budget, I/O hitch and missing asset as different problems.

The rendering chapter covers GPU/residency symptoms; this chapter's architecture point is to budget definition/object load separately from bulk resource streaming and avoid hard-loading optional high-resolution presentation through lightweight gameplay definitions.

## 11. Debugging and profiling loading

### Evidence ladder

1. Reproduce cold/warm and editor/packaged differences.
2. Place correlation markers around request, async begin, package/object ready, resource ready, callback commit and first visible use.
3. Use Insights loading/file/CPU/memory channels available in the branch; inspect game-thread completion work and I/O gaps. [SRC-ASSET-012]
4. Audit dependency closure and retained owners.
5. Inspect cook inclusion/chunk/container logs for missing packaged assets.
6. Run on target storage/memory, not only an NVMe developer machine.

### Common causes of “async load hitch”

- request starts too late;
- too many dependencies joined by one hard root;
- completion callback spawns/registers many objects;
- resource initialisation/texture/shader work lands near first use;
- DDC/PSO/shader cache is cold;
- GC follows an unload burst;
- queue prioritisation/starvation;
- load is async but immediate `GetAsset`/sync fallback blocks elsewhere.

## 12. Strong interview answers

### Hard versus soft reference

> A hard reference makes the dependency part of the referencer's load closure and is correct for always-needed content. A `TSoftObjectPtr` stores typed indirect identity and allows deferred loading, but does not alone guarantee cook inclusion or post-load residency. I would choose from co-residency requirements, loading phase and memory budget, manage the async handle/strong owner, validate cancellation/failure, and inspect the packaged dependency/cook result.

### Missing only in packaged build

> I would reproduce in a clean packaged Development build, capture the exact Primary ID/path, and check the cook manifest/Asset Manager audit and logs rather than adding a delay. Then I would verify management/reference rules, scan paths, chunk/label policy, editor-only/plugin flags, case and redirectors. The editor can see loose assets and cached loaded objects, so editor success does not prove cook inclusion.

### Asset Manager versus Asset Registry

> The Registry indexes metadata and lets me discover/query `FAssetData` without loading assets—unless I call a loading conversion such as `GetAsset`. The Asset Manager builds managed identity and load/cook/chunk policy around Primary Assets and their Secondary dependencies/bundles. I often use Registry-style discovery underneath a project-specific Asset Manager domain.

## 13. Hands-on asset loading lab

1. Create one small Primary Item definition with `UI` and `Game` bundles and soft references.
2. Create an accidental hard root that pulls in all item classes/presentation; measure closure and startup memory.
3. Replace optional dependencies with managed soft loading; implement pending/failure/cancelled/ready states and generation tokens.
4. Close the screen/travel while loading and prove no stale callback/use-after-world.
5. Release retained handle/strong owner and observe residency/unload/GC according to the target setup.
6. Package with one intentionally unmanaged soft-by-string asset; diagnose missing cook, then repair with a Primary Asset rule/explicit management rather than Cook Everything.
7. Move/rename content, fix redirectors through source-control-aware resave, and retest save IDs and package.
8. Record cold/warm DDC and target-storage loading captures.
9. Add a World Partition teleport preload source and wait for readiness before moving the player.
10. Produce an asset budget sheet: root, closure size, load phase, bundle, resident owner, cook rule, failure fallback and trace marker.

## 14. What a three-year engineer should know

- hard/soft/weak reference semantics and dependency closure;
- typed soft object versus soft class versus raw soft path;
- synchronous versus async loading and callback/residency ownership;
- Asset Registry versus Asset Manager and Primary/Secondary identity;
- definition bundles and basic labels/chunks;
- build/cook/stage/package/deploy distinctions;
- packaged-only missing-asset workflow;
- redirector/DDC purpose;
- level streaming/World Partition ownership and unload safety;
- loading trace and target-device verification.

Specialist depth includes package store/IoDispatcher internals, custom Asset Manager rules, cooker graphs, install bundles/patch pipelines, IoStore tooling, async loading thread internals, Zen/DDC operations, platform certification and large-world streaming-grid tuning.

## 15. Source map

- References and async lifetime: [SRC-ASSET-001] [SRC-ASSET-002]
- Asset Manager/Registry: [SRC-ASSET-003] [SRC-ASSET-004]
- Cook/package/chunks: [SRC-ASSET-005] [SRC-ASSET-006]
- Redirectors and DDC: [SRC-ASSET-007] [SRC-ASSET-008]
- Level/World Partition streaming: [SRC-ASSET-009] [SRC-ASSET-010]
- Validation/profiling: [SRC-ASSET-011] [SRC-ASSET-012]

