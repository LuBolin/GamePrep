# Unreal Rendering and GPU Performance

This chapter targets a gameplay or engine programmer with roughly three years of experience. The expected standard is not “write a renderer from memory”. It is: explain how a frame reaches the GPU, separate different kinds of rendering cost, identify the expensive pass with evidence, and collaborate intelligently with rendering engineers and technical artists.

Version target: **UE5.3–UE5.6**. Nanite, Lumen, Virtual Shadow Maps (VSMs), Substrate, renderer feature support, and profiling UI are version- and platform-sensitive. Re-test on the exact engine branch and target hardware. [SRC-RENDER-001] [SRC-RENDER-005] [SRC-RENDER-006]

## 1. The mental model: data becomes commands, then pixels

A useful frame model has several lanes:

1. The **game thread** updates gameplay state and creates the scene changes that rendering must consume.
2. The **render thread** converts the renderer's scene representation and views into render work.
3. The **RHI thread**, where enabled, translates or submits lower-level command work for the platform graphics API.
4. The **GPU** executes graphics and compute work: geometry processing, rasterisation, pixel/compute shading, copies, resolves, and presentation.

These lanes overlap. Their displayed times are not additive. A 10 ms Game time, 8 ms Draw time, and 11 ms GPU time do not imply a 29 ms frame; the slowest dependency chain normally limits throughput. Synchronisation, queue starvation, buffered frames, VSync, frame caps, and async compute complicate the picture. Begin with the frame-budget method in `ue_profiling_optimisation.md`, then descend into GPU passes. [SRC-PERF-001] [SRC-PERF-003]

The **RHI** is Unreal's abstraction over APIs such as Direct3D, Vulkan, and Metal. It exposes resources, pipelines, command lists, transitions, and draw/dispatch operations without making high-level renderer code identical to a particular API. The RHI does not erase platform differences; capability checks and platform-specific paths still matter.

### Scene state is not a live mirror

The renderer generally works from a render-side representation, not by freely dereferencing mutable gameplay objects on the GPU. A gameplay-side change may be enqueued, propagated, uploaded, and become visible to later frames. This has three interview consequences:

- game-thread correctness and render-thread correctness are different questions;
- creating or mutating render state can have CPU and GPU costs beyond the setter call;
- an apparent one-frame delay can be a pipeline/lifetime issue rather than “the GPU being random”.

Do not cache or touch render-thread-only resources from arbitrary threads. Follow the owning API's threading and lifetime contract.

## 2. From views to passes

For each camera/view, the renderer determines visible primitives, selects material/shader variants, builds work, and executes a set of raster and compute passes. The exact graph varies with renderer, platform, features, view type, and engine version. A representative deferred desktop frame may include:

| Stage / pass | Purpose | Typical pressure | Useful first evidence |
|---|---|---|---|
| CPU visibility and draw preparation | Determine visible primitives and prepare commands | object count, state diversity, dynamic updates, command setup | Draw/RHI time, Insights render tracks, primitive/draw counts |
| Shadow depth / VSM page rendering | Render light-space depth needed for shadows | shadow casters, lights, invalidated pages, WPO | `stat gpu`, GPU Visualiser, VSM visualisations |
| Depth prepass | Establish scene depth early | triangle count, masked materials, extra geometry pass | GPU pass time, early-Z settings, depth view |
| Base pass / GBuffer | Write surface attributes used by deferred lighting | visible pixels, material shader cost, bandwidth, overdraw | BasePass time, Shader Complexity, material stats |
| Lighting | Evaluate direct lighting using scene/GBuffer data | light count and coverage, shadowing, screen resolution | lighting pass names, light complexity, isolation toggles |
| Lumen / reflections | Compute dynamic indirect lighting and reflections | scene representation, traces, update budget, resolution/quality | Lumen overview visualisations, named GPU passes |
| Translucency | Blend translucent surfaces after/beside opaque work | layers, coverage, shader cost, sorting | Translucency time, Shader Complexity/Quad Overdraw |
| Post-processing | Exposure, bloom, depth of field, motion blur, tonemapping, upscaling and more | screen pixels, quality, history, extra views | named post passes, resolution sensitivity |
| UI / composition / present | Composite UI and final targets | widget redraw, fill, target copies, platform present | Slate/UI stats, GPU events, present/VSync checks |

This is a reasoning map, not a guaranteed pass order. Features may merge, split, use async compute, or disappear. Trust the capture from the target build.

### Depth prepass trade-off

Rendering depth first adds geometry work, but can reject hidden pixels before expensive base-pass shading. It tends to help scenes with substantial opaque occlusion or expensive pixel shaders and can hurt where the extra vertex/draw cost buys little. Masked materials, Nanite, platform paths, and engine heuristics affect the result. Compare the actual pass timings; do not make “early Z always wins” a rule.

## 3. Deferred versus forward shading

### Deferred shading

In a conventional deferred path, the base pass writes surface data to multiple GBuffer targets, and lighting is evaluated later. This is Unreal's default desktop renderer because it supports a broad feature set and handles many dynamic lights flexibly. Its costs include GBuffer memory/bandwidth and limitations around hardware MSAA and transparent surfaces.

### Forward shading

The desktop forward renderer shades supported lights and reflection captures during the forward base pass. Unreal culls them into a frustum-space grid, and each pixel processes the relevant set. Forward shading can provide a faster baseline for suitable VR/content configurations and supports MSAA, but has a different feature set and material/light cost model. Epic explicitly recommends profiling the project rather than assuming the renderer switch is a universal win. [SRC-RENDER-002]

| Decision factor | Deferred tends to suit | Forward tends to suit |
|---|---|---|
| Feature breadth | projects needing the default desktop feature set | projects whose required features are supported by forward |
| Anti-aliasing | temporal techniques are acceptable | MSAA is important, often in VR |
| Dynamic light model | many lights benefit from deferred evaluation | bounded per-pixel light sets and simpler feature use |
| Bandwidth / pass shape | GBuffer cost is acceptable | a leaner baseline may fit the content/platform |
| Decision method | benchmark target scenes | benchmark the same target scenes |

**Interview answer:** choose from requirements, then profile representative content on target hardware. “Forward is faster” and “deferred is better” are both incomplete.

## 4. Do not conflate the cost dimensions

Rendering optimisation goes wrong when “too many polygons”, “too many draw calls”, and “expensive shader” are treated as synonyms.

### Submission and state cost

A **draw call** submits a batch with compatible geometry, material/pipeline state, resources, and parameters. Many small independently configured objects can make the CPU render/RHI lanes expensive even when the GPU shades few pixels.

Likely levers include:

- culling objects that do not matter;
- instancing repeated compatible meshes with ISM/HISM or a suitable system;
- reducing unnecessary material slots and state variation;
- HLOD/proxying for distant groups;
- avoiding needless per-frame render-state mutation;
- choosing Nanite where its supported GPU-driven path fits.

Merging is not free: it can worsen culling granularity, transforms, authoring, streaming, and material flexibility. One giant merged city is not the enlightened end state of draw-call optimisation.

### Geometry and vertex cost

Geometry work scales with visible/submitted primitives, vertices, instances, passes that redraw geometry, skinning/deformation, and vertex shader work. Shadow and depth passes may process geometry without paying the full material pixel cost. Tiny triangles can be inefficient because rasterisation still works in pixel quads and because detail may be sub-pixel.

### Pixel, fill-rate, and overdraw cost

Pixel cost grows with render resolution, covered screen area, samples, shader work, and the number of times pixels are shaded. Opaque depth rejection can avoid some hidden pixel work. Translucent layers generally still blend in order and are a classic source of overdraw. Particle fog covering the screen may be expensive with almost no conventional geometry pressure.

### Shader and material cost

Runtime shader cost includes instructions, texture samples, control flow, register pressure/occupancy, and the frequency at which work runs. A node count is not a GPU time. Compiler optimisation, platform shader model, static branches, precision, texture behaviour, and pass context all matter.

### Bandwidth and memory cost

Render targets, GBuffer traffic, depth, shadow pages, texture reads, history buffers, transient resources, and copies consume bandwidth and memory. A shader with modest arithmetic can be bandwidth-bound. High resolution and multiple views amplify these costs.

### The scaling tests

Use controlled changes to sharpen a hypothesis:

- lower internal resolution: large improvement suggests pixel/post/lighting pressure;
- reduce object count without changing screen coverage: improvement in Draw/RHI suggests submission pressure;
- simplify material while preserving geometry: improvement localises shading/texture pressure;
- disable shadow casting on a controlled set: improvement in shadow passes suggests caster/light pressure;
- reduce translucent layers/coverage: improvement localises overdraw;
- freeze camera or feature updates: improvement can expose invalidation/update work.

A scaling test is evidence, not proof by itself. Record the named pass and corroborate it.

## 5. Materials, permutations, and expensive edge cases

### Material instances and parameters

A material instance reuses a parent material and changes exposed parameters. **Dynamic** parameters change values at runtime; **static** switches change compiled shader permutations. Static switches can remove runtime branches/work, but combinations multiply compile time, shader library size, memory, PSO diversity, and cache pressure.

Prefer a bounded family of purposeful parent materials. Avoid both extremes: one universal mega-material with an uncontrolled permutation space, and thousands of unique materials that prevent reuse and batching.

### World Position Offset (WPO)

WPO changes vertex positions in the material. It can add vertex work, affect bounds/culling, invalidate dynamic shadow data, and require the deformation in multiple relevant passes. Incorrect bounds can cause disappearing geometry or shadows; oversized bounds can make culling ineffective. Profile the actual content and inspect VSM invalidation when applicable. [SRC-RENDER-006]

### Masked versus translucent

- **Opaque** surfaces usually get the best depth rejection and broad renderer support.
- **Masked** surfaces discard pixels by an opacity mask. They can be expensive with dense cut-outs and may add depth/shadow cost, but retain many opaque-path advantages.
- **Translucent** surfaces require blending and ordering approximations, have feature limitations, and become especially expensive with large overlapping layers. Epic's material guidance calls out stacked translucency as high per-pixel cost. [SRC-RENDER-004]

Choose the cheapest model that preserves the visual requirement. A fence or leaf card may need masked edges; smoke may need translucency; neither conclusion means the cost is free.

### Diagnosing a material

1. Find the expensive pass and objects first.
2. Inspect Material Editor platform statistics and generated shader information where available.
3. Use Shader Complexity and Quad Overdraw as comparative views, not literal final frame cost.
4. Create controlled parameter/simplified variants.
5. Check visual equivalence and all relevant passes: base, depth, shadows, velocity, translucency.
6. Re-measure in the target build.

## 6. Visibility, instancing, LOD, and HLOD

Unreal combines multiple visibility mechanisms. View-frustum culling removes objects outside the camera volume; occlusion methods reject objects hidden by nearer geometry; distance and precomputed/platform-specific methods may also contribute. Incorrectly small bounds cause popping or missing objects; excessively large bounds reduce culling effectiveness. [SRC-RENDER-008]

### LOD

Traditional mesh LODs substitute simpler geometry/materials with distance or screen size. They reduce geometry, shading, shadow, and sometimes skinning cost, but transitions, memory, and authoring must be managed. Skeletal mesh LOD can also reduce bones and animation work; that is not the same problem as static mesh triangle LOD.

### ISM and HISM

Instanced Static Mesh components represent repeated compatible mesh instances more efficiently than many independent components/draws. HISM adds a hierarchy intended to improve culling/LOD behaviour for large instance sets. Trade-offs include per-instance feature limits, update patterns, bounds/tree maintenance, and reduced independence. Measure the relevant UE version; Nanite changes some historical draw-call reasoning but does not erase CPU scene-management cost.

### HLOD

HLOD replaces groups of distant Actors with proxies, reducing object/draw/material complexity and helping large-world streaming/rendering. It complements rather than means the same thing as per-object LOD. A useful distinction:

- LOD: simplify one asset;
- instancing: submit repeated compatible assets efficiently;
- HLOD: simplify a distant spatial group;
- culling: do not render work that cannot contribute;
- Nanite: virtualise supported geometry detail and GPU-driven visibility.

## 7. Nanite: what it solves and what it does not

Nanite analyses supported meshes into hierarchical clusters, selects detail according to the view, streams needed data, and uses its own GPU-driven rendering path rather than traditional draw submission for that geometry. It is especially suited to high triangle counts, very small triangles, many instances, major occluders, and geometry casting VSM shadows. [SRC-RENDER-005]

### Strong working model

Nanite primarily attacks **geometry detail, LOD selection, visibility, and draw submission for supported geometry**. It does not make these free:

- material/pixel shading;
- translucent layers;
- lighting, reflections, or post-processing;
- shadow page invalidation and rasterisation;
- WPO/deformation;
- scene-object management, transforms, or streaming;
- unsupported platform/rendering configurations.

The exact support matrix evolves. The maintained Epic page can describe features later than UE5.6, so verify static/skeletal mesh, foliage, spline, forward, VR, MSAA, ray-tracing, material, and deformation support against the target branch. Do not import a current UE5.7 support statement backwards into UE5.3–5.6. [SRC-RENDER-005]

### Debugging Nanite

Use Nanite visualisation modes to inspect clusters, overdraw, material ranges, instances, and streaming behaviour available in the target version. Compare Nanite and fallback/traditional configurations in the same camera path. If GPU time remains high, name the pass: the bottleneck may be material shading, VSM, Lumen, translucency, or post-processing rather than Nanite rasterisation.

## 8. Lumen: dynamic GI and reflections with a budget

Lumen provides fully dynamic diffuse global illumination and reflections. Its pipeline combines screen traces with a more reliable scene-based method; software ray tracing uses signed distance fields, while hardware ray tracing uses supported ray-tracing hardware and has different quality/performance trade-offs. [SRC-RENDER-007]

Lumen enables dynamic lighting changes and integrates with systems such as Nanite, VSMs, and World Partition. It does not mean “physically perfect lighting at no setup cost”. Scene representation, mesh distance fields, hardware support, view dependence, surface cache coverage, trace distance, resolution, fast camera changes, emissive size/brightness, translucency, foliage, and scalability can affect quality and cost.

### A practical artefact workflow

1. Classify the error: missing indirect light, leaking, noise, slow update, incorrect reflection, or excessive time.
2. Determine software versus hardware tracing and the active scalability level.
3. Compare the main view with Lumen overview/scene/surface-cache visualisations available in the target version.
4. Check whether the problematic geometry is represented appropriately and whether screen traces are masking a scene-representation issue.
5. Isolate GI and reflections; inspect their named GPU passes.
6. Change one quality, trace, geometry, or material variable and retest the same view.

Epic documents explicit console-oriented performance targets rather than an unlimited quality promise. Treat its published millisecond examples as context for those platforms/settings, not a budget guaranteed for your project. [SRC-RENDER-007]

## 9. Virtual Shadow Maps: virtual pages and invalidation

VSMs provide high-resolution dynamic shadows designed to work with highly detailed Nanite geometry and large worlds. Conceptually, they divide extremely high-resolution shadow maps into pages, allocate/render pages needed by visible pixels, and cache them between frames. Movement or changes to lights and shadow-casting geometry invalidate relevant cached pages. [SRC-RENDER-006]

This produces a better interview diagnosis than “VSMs are expensive”:

- many visible new pages can cost raster work;
- moving lights can invalidate broad regions;
- moving/deforming casters, including WPO, can invalidate pages;
- non-Nanite geometry can produce conventional draw-call pressure in shadow work;
- coarse pages and large light influence can matter;
- excessive bounds make invalidation/culling less precise.

Use VSM page/cache/invalidation visualisations and correlate them with shadow pass time. Freeze or isolate moving casters/lights to test the hypothesis. Do not disable all shadows and call that diagnosis; it only proves shadows contributed.

## 10. Texture streaming and virtual texturing

Texture streaming aims to keep useful mip levels resident within a memory budget according to view demand and other heuristics. Symptoms span two different failure types:

- **quality/residency:** blurry textures, late mip transitions, missing detail;
- **performance/capacity:** pool pressure, I/O/upload hitches, memory overcommit, churn from rapidly changing views.

Investigate texture group/LOD bias, asset dimensions, mip generation, material use, bounds/streaming data, pool state, requested versus resident mips, camera path, and target storage. Increasing the pool can hide bad content or exceed the target memory envelope; lowering all textures can hide a scheduling/streaming defect.

Virtual Texturing and Runtime Virtual Texturing tile texture data and can reduce residency for large sparse content, but add page-table, cache, feedback, upload, authoring, and compatibility trade-offs. They solve a texture residency/access pattern, not arbitrary GPU memory or draw-call problems. Exact platform and feature support is version-sensitive.

## 11. RDG: rendering-engine awareness

The Render Dependency Graph (RDG) records render commands into a graph that is compiled and executed. Pass parameter declarations expose resource reads/writes and dependencies, allowing Unreal to manage transitions, transient resource lifetimes/aliasing, pass culling, parallel command recording, and asynchronous-compute fences. [SRC-RENDER-003]

For an interview at generalist depth, explain:

- `FRDGBuilder` creates graph resources and records passes;
- creating an RDG texture/buffer descriptor does not necessarily allocate its underlying RHI resource immediately;
- pass parameters declare resources and let RDG derive dependencies/lifetimes;
- the pass lambda records commands during graph execution and must respect deferred lifetime/threading rules;
- external resources cross the graph boundary; transient resources belong to graph lifetimes;
- undeclared access, captured short-lived memory, side effects, and immediate-command-list use can defeat correctness or parallelism.

RDG is not merely a convenient list of callbacks. The dependency/resource declarations are the contract that makes scheduling, barriers, validation, culling, and memory reuse possible.

### RDG debugging awareness

Development builds have validation support. RDG Insights can inspect pass/resource relationships, lifetimes, transient allocation overlap, async compute, pass culling, merging, and parallel ranges. Immediate mode and feature-disable CVars are diagnostic tools but change optimisation and timing, so they are not representative benchmarks. [SRC-RENDER-003]

## 12. The pass-led GPU investigation workflow

### Phase A: establish the experiment

Record engine commit/version, hardware, driver, API, packaged configuration, resolution/internal screen percentage, scalability, ray-tracing path, map, camera path, warm-up, frame cap/VSync/dynamic resolution, and capture duration. Use GPU timings only when the GPU is actually fed and unconstrained by unrelated caps.

### Phase B: classify and name

1. Use `stat unit`/a trace to establish whether GPU throughput is the current limiter.
2. Capture `stat gpu`, the GPU Visualiser (`ProfileGPU`), or target platform tooling.
3. Identify one or two dominant passes. A total GPU number without a pass is not yet a diagnosis.
4. Inspect event hierarchy and correlate the pass with views, lights, primitives, or materials.

### Phase C: form a mechanism hypothesis

State it so it can fail:

> “Translucency is expensive because four full-screen particle layers repeatedly shade most pixels; halving internal resolution or coverage should reduce the Translucency pass materially, while reducing object count outside the effect should not.”

Good hypotheses name **pass + mechanism + controlled prediction**.

### Phase D: isolate

Use view modes, show flags, scalability controls, carefully chosen CVars, material overrides, distance/caster restrictions, resolution scaling, and scene variants. Change one meaningful variable at a time. Feature-off tests locate cost but are not automatically shippable fixes.

### Phase E: optimise and verify

Retain visual/correctness requirements, measure repeated before/after samples, examine P50/P95/P99 and hitches, test multiple views and content loads, and check whether cost moved to Game/Draw/memory. Validate on every target class that matters.

### GPU capture escalation

Use RenderDoc, PIX, Nsight Graphics, Xcode GPU tools, or console vendor tools when Unreal's pass timings cannot explain pipeline state, shader/resource behaviour, occupancy, bandwidth, or a driver/GPU-specific issue. Captures can perturb execution and may require special build/platform setup.

## 13. Common bugs and misconceptions

| Claim or symptom | Better model / first check |
|---|---|
| “We are GPU-bound, so reduce draw calls.” | Draw calls often pressure CPU submission; inspect Draw/RHI and the named GPU pass. |
| “Nanite removes polygon budgets.” | It changes supported geometry detail/visibility economics; pixels, materials, shadows, memory, deformation, and platforms remain bounded. |
| “Shader Complexity is the frame time.” | It is a diagnostic approximation; verify named pass timing on target. |
| “Forward is always faster.” | It has a different feature/cost model; benchmark required features and content. |
| “Lower resolution fixed it, so the shader is bad.” | It localises pixel-scaled work; lighting, post, translucency, bandwidth, or upscaling may be responsible. |
| “One material instance means one shader.” | Static switches, vertex factories, passes, platforms, quality levels, and features create permutations. |
| “VSM cache should make shadows free.” | Visibility changes, lights, moving/WPO casters, and new pages cause work and invalidation. |
| “Make bounds huge to fix popping.” | It may mask bad bounds while harming culling, shadows, and invalidation. Fix the deformation/bounds contract deliberately. |
| “Merge everything to reduce draws.” | Coarse culling, streaming, transforms, authoring, and material variation can regress. |
| “Disable the feature; we found the fix.” | You found a cost contributor. Find a requirement-preserving intervention and re-measure. |

## 14. What depth is expected?

### Strong three-year engineer (D3–D4)

- distinguishes Game, Draw/RHI, and GPU bottlenecks;
- explains deferred versus forward at a practical level;
- can describe depth, base/GBuffer, shadows, lighting, translucency, and post passes;
- separates draw/geometry/pixel/shader/bandwidth costs;
- uses pass timings, view modes, scaling tests, and controlled captures;
- chooses among culling, instancing, LOD/HLOD, and Nanite deliberately;
- understands Lumen/VSM purpose, trade-offs, and visualisation-led debugging;
- understands material instances, static permutations, WPO, masked, and translucent costs;
- can explain RDG's dependency/lifetime purpose without pretending to be a renderer specialist.

### Specialist depth (D5)

- reads renderer/RHI/RDG source and GPU captures;
- authors global/material shaders and RDG passes safely;
- reasons about barriers, render-pass merging, async compute, wave occupancy, cache/bandwidth, and transient aliasing;
- understands PSO creation/caching and platform shader pipelines;
- tunes Nanite/Lumen/VSM internals for a target renderer branch;
- validates changes across vendors, APIs, consoles, and representative workloads.

## 15. Strong interview answer patterns

### “How would you investigate a GPU regression?”

> I would first reproduce it in a controlled target build and record resolution, scalability, VSync, dynamic resolution, hardware, and camera path. I would confirm GPU throughput is limiting rather than relying on FPS, then use `stat gpu`/GPU Visualiser or platform tools to name the pass that regressed. I would compare a known-good capture, form a mechanism hypothesis—such as increased translucent coverage or VSM invalidation—and run a controlled scaling/isolation test. I would make the smallest requirement-preserving change, then recapture repeated samples and check visual quality plus Game/Draw/memory on all target tiers.

### “Nanite, LOD, instancing, or HLOD?”

> They address overlapping but different costs. Nanite virtualises supported geometry detail and GPU visibility; conventional LOD simplifies one asset; instancing efficiently represents repeated compatible meshes; HLOD replaces distant spatial groups with proxies. I choose based on platform/feature support, object repetition, distance, material and deformation needs, culling/streaming granularity, then measure CPU submission, GPU passes, and memory.

### “Why is translucency expensive?”

> It often combines per-pixel shading with large coverage and repeated overdraw because layers must blend and cannot use the opaque depth path in the same way. Sorting and feature limitations add constraints. I would inspect the Translucency pass and overdraw views, test coverage/resolution/material simplification, then reduce layers, screen coverage, shader work, or use masked/opaque/impostor alternatives where the look permits.

### “What is RDG?”

> RDG is Unreal's render graph: high-level renderer code records passes and declares resources/dependencies, then the graph is compiled and executed. That contract lets the engine manage transitions and lifetimes, cull unused passes, alias transient memory, and schedule/parallelise suitable work. The common implementation danger is violating the deferred lifetime or using resources that were not declared through pass parameters.

## 16. Hands-on verification tasks

### Task A: cost-dimension matrix

Create four controlled scenes: many tiny opaque objects, one full-screen expensive material, stacked translucent particles, and moving WPO shadow casters. Predict Game/Draw/RHI/GPU and pass changes, then capture. Keep one disproved prediction in the report.

### Task B: representation comparison

Render the same repeated asset population as independent components, ISM/HISM, traditional LOD, Nanite where supported, and an HLOD proxy at distance. Hold camera and visual coverage constant. Record CPU scene setup/update, Draw/RHI, GPU pass times, memory, pop/culling behaviour, and authoring constraints.

### Task C: Lumen/VSM diagnosis

Create one Lumen artefact and one VSM invalidation spike deliberately. Use their visualisations to explain the internal representation/cache failure, then fix it without globally disabling the feature.

### Task D: minimal RDG pass (specialist extension)

In a source-capable branch, add a small named compute or full-screen RDG pass using declared input/output resources and a scoped event. Inspect it in GPU Visualiser/RDG Insights. Deliberately capture stack memory with insufficient lifetime or omit a resource dependency only in a safe experiment, observe validation, then repair it.

## 17. Source map

- GPU/frame profiling workflow: [SRC-PERF-001] [SRC-PERF-003] [SRC-RENDER-009]
- Forward versus deferred: [SRC-RENDER-001] [SRC-RENDER-002]
- RDG architecture/debugging: [SRC-RENDER-003]
- Materials, view modes, overdraw: [SRC-RENDER-004] [SRC-RENDER-010]
- Nanite: [SRC-RENDER-005]
- Virtual Shadow Maps: [SRC-RENDER-006]
- Lumen: [SRC-RENDER-001] [SRC-RENDER-007]
- Visibility and HLOD: [SRC-RENDER-008] [SRC-RENDER-011]

## 18. Specialist cluster: shaders, PSOs, mesh drawing, render targets, and captures

This section moves from pass-led diagnosis into implementation-facing rendering work. A gameplay engineer does not need to write Unreal's renderer from memory, but a strong engine/generalist candidate should know why shader changes can affect cook time and runtime hitches, why PSO caches exist, why "one mesh" can produce many draw commands, why a render target is often an extra camera, and when to escalate from Unreal pass timings to RenderDoc/PIX/Nsight/Xcode captures. [SRC-RENDER-012] [SRC-RENDER-015] [SRC-RENDER-019]

### Shader vocabulary: runtime cost versus build/runtime pipeline cost

Separate three ideas:

| Concept | What it means | Failure mode |
|---|---|---|
| Shader source | `.usf` / `.ush` HLSL-like code, material graph-generated code, engine shader code or plugin shader files | Bad include/mapping/module setup; compile errors hidden until target platform cook. |
| Shader permutation | One compiled variant for a platform, feature level, material/static switch, vertex factory, pass or quality path | Combinatorial explosion in compile time, DDC/shader library size and PSO diversity. |
| Runtime shader cost | Instructions, texture reads, bandwidth, register pressure, occupancy and frequency of execution | Optimising compile count but leaving the actual hot pass/pixels expensive. |

The mistake is to use one word, "shader", for all three. A static switch can reduce runtime instructions for one variant while multiplying the number of variants that need to compile and ship. A material simplification can reduce BasePass cost without meaning PSO hitching is fixed. A shader library cook can succeed while a runtime path still creates uncached PSOs. [SRC-RENDER-004] [SRC-RENDER-012] [SRC-RENDER-013]

### Shader compile pipeline mental model

A practical shader lifecycle:

```text
Author material/global/plugin shader code
    -> engine discovers shader types and source directories
    -> shader compiler creates platform/feature/permutation variants
    -> results are cached in DDC/shader libraries where applicable
    -> cooked build packages needed shader code/libraries
    -> runtime binds shaders through materials, global shader calls or renderer passes
    -> pipeline state may still need compatible PSO creation/cache coverage
```

When a build or runtime stalls, ask which stage is failing:

1. Did the module map its shader directory and load early enough for shader type registration?
2. Did the target platform compile the intended permutation?
3. Did a static switch, quality level, vertex factory, platform define or material domain create an unexpected variant?
4. Was DDC warm locally but cold in CI or for another developer?
5. Was the cook missing a shader library or runtime path?
6. Was the stall actually PSO creation rather than shader compilation?

Shader compile time is not the same as GPU frame time. Both matter, but they need different evidence.

### Plugin/global shader extension workflow

Global or plugin shaders are justified when a material graph or built-in renderer path is the wrong abstraction: custom full-screen processing, compute jobs, data conversion, debug visualisation, generated buffers, or a source-branch renderer extension. They are not justified for ordinary surface shading that materials already handle well.

High-level implementation shape:

```cpp
// Schematic only. Verify exact macros, includes, module loading phase,
// shader directory mapping and RDG APIs in the target branch.

class FMySimpleComputeCS : public FGlobalShader
{
    DECLARE_GLOBAL_SHADER(FMySimpleComputeCS);
    SHADER_USE_PARAMETER_STRUCT(FMySimpleComputeCS, FGlobalShader);

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_RDG_TEXTURE(Texture2D, InputTexture)
        SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float4>, OutputTexture)
        SHADER_PARAMETER(FVector4f, Params)
    END_SHADER_PARAMETER_STRUCT()

    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
    {
        return IsFeatureLevelSupported(Parameters.Platform, ERHIFeatureLevel::SM5);
    }
};

IMPLEMENT_GLOBAL_SHADER(
    FMySimpleComputeCS,
    "/Plugin/MyRenderPlugin/MySimpleCompute.usf",
    "MainCS",
    SF_Compute);

void AddMyComputePass(
    FRDGBuilder& GraphBuilder,
    FRDGTextureRef Input,
    FRDGTextureRef Output)
{
    FMySimpleComputeCS::FParameters* PassParameters =
        GraphBuilder.AllocParameters<FMySimpleComputeCS::FParameters>();

    PassParameters->InputTexture = Input;
    PassParameters->OutputTexture =
        GraphBuilder.CreateUAV(FRDGTextureUAVDesc(Output));

    const TShaderMapRef<FMySimpleComputeCS> ComputeShader(
        GetGlobalShaderMap(GMaxRHIFeatureLevel));

    FComputeShaderUtils::AddPass(
        GraphBuilder,
        RDG_EVENT_NAME("MySimpleCompute"),
        ComputeShader,
        PassParameters,
        FIntVector(GroupCountX, GroupCountY, 1));
}
```

The interview-relevant contract is more important than memorising syntax:

- shader type registration must happen before shader compilation expects it;
- shader source path mapping must be correct for plugin shaders;
- permutation predicates prevent compiling unsupported/unneeded variants;
- parameter structs declare the GPU resources and constants;
- RDG resources and pass parameters describe dependencies/lifetimes;
- the pass executes later, so captured memory and UObject access must obey render-thread/RDG lifetime rules;
- output validation needs a GPU capture or readback/debug visualisation, not just "it compiles." [SRC-RENDER-003] [SRC-RENDER-013] [SRC-RENDER-014]

### Debugging workflow: custom shader outputs black

1. Confirm the pass appears in GPU Visualiser/RDG Insights with the expected event name.
2. Confirm `ShouldCompilePermutation` returned true for the target platform/feature level.
3. Confirm shader file path mapping and entry point names match packaged/cooked paths.
4. Confirm all RDG inputs/outputs are declared in the parameter struct and are valid for the pass.
5. Confirm thread and lifetime rules: no expired stack references, no game-thread UObject dereference in pass execution.
6. Use a constant colour/value output first; remove algorithmic complexity.
7. Inspect resource format, UAV/render-target flags, viewport size, dispatch group counts and barriers via RDG validation/capture.
8. Capture in RenderDoc or platform GPU tools when Unreal event timing cannot explain the wrong output. [SRC-RENDER-003] [SRC-RENDER-019]

### PSOs: why a shader can be compiled and still hitch

A Pipeline State Object packages a compatible set of GPU pipeline state: shader stages plus fixed-function state, render target/depth formats, vertex declaration/input layout, raster/depth/blend state and related RHI-specific details. Modern explicit APIs often make PSO creation expensive enough that creating a new one during gameplay can hitch. Unreal's PSO caching/precaching workflows exist to discover, compile and ship likely-needed PSOs before the player hits them. [SRC-RENDER-012]

Useful mental split:

```text
Shader library coverage: can the shader code be found/loaded?
PSO coverage: has this exact pipeline state combination been created/prepared?
Runtime hotness: is the draw/dispatch frequent or expensive once it runs?
```

These are related but not interchangeable. A material permutation can exist in the shader library but still produce a never-before-seen PSO when paired with a vertex factory, render target format, blend mode, depth state, pass and platform path.

### PSO cache workflow

A production-style workflow usually has these steps:

1. Enable target branch/project PSO logging or collection in a representative packaged build.
2. Play through real content paths: maps, scalability levels, camera views, weapons/VFX, UI, cutscenes, character cosmetics and scene captures.
3. Collect stable PSO data from multiple runs/hardware/settings where required.
4. Merge, validate and package the cache according to the target platform/RHI workflow.
5. Warm/precache at controlled moments if the branch supports it.
6. Re-run with hitch tracing to find uncached PSOs.
7. Treat every "first time I see this effect" hitch as a missing coverage hypothesis until proved otherwise.

Common missed paths:

- rare material static switch combinations;
- VFX/Niagara materials and mesh particles;
- UI materials and Retainer/render-target paths;
- scene captures, thumbnails, portals, mirrors and minimaps;
- skeletal mesh clothing/cosmetic variants;
- scalability toggles, resolution/upscaler paths and renderer settings;
- cutscenes and loading-screen-only materials;
- platform-specific shader model/RHI differences.

PSO caching is not a substitute for reducing permutation diversity. If the project has an uncontrolled material family, PSO caches become a symptom management system rather than a cure.

### Debugging workflow: first-use rendering hitch

1. Confirm the hitch in a packaged target build, not only editor PIE.
2. Correlate frame hitch timing with first appearance of material/effect/mesh/view.
3. Check whether shader compilation, shader library loading, PSO creation, texture streaming, async asset load or render-state creation is the culprit.
4. Run with target branch PSO logging/validation and collect the missed PSO if supported.
5. Reproduce after warming DDC and assets; if it disappears only on warm developer machines, the shipped path is still suspect.
6. Add the content path to PSO collection coverage or reduce the variant/state diversity.
7. Re-measure after packaging the cache and after clearing local caches.

Strong answer:

> A PSO hitch is not solved by saying "compile shaders earlier" unless the evidence says shader compilation is the stage. I separate shader library availability from PSO creation, collect representative PSOs in packaged builds, package/warm them through the target platform workflow, then verify first-use hitches with cold-cache runs.

### Mesh drawing pipeline: from component to draw commands

The mesh drawing pipeline is specialist internals, but the practical model is valuable. A gameplay component does not become a draw call by magic. It contributes render-side primitive data, material relevance, mesh batches and pass-specific draw commands. Unreal can cache mesh draw commands for suitable static/reusable work, while dynamic primitives or frequently changing state require more per-frame setup. [SRC-RENDER-015]

High-level flow:

```text
Game thread component/primitive state
    -> render-state creation/update
    -> FPrimitiveSceneProxy-like render-side representation
    -> view relevance and pass relevance
    -> mesh batches / material / vertex factory data
    -> pass processors build draw commands
    -> cached or dynamic mesh draw commands are submitted
    -> RHI/PSO/resources are bound and GPU executes
```

This explains why these gameplay choices affect renderer CPU:

- many components with unique materials create many primitive/material combinations;
- many material slots/sections split work even on one mesh;
- per-frame `MarkRenderStateDirty` or component recreation invalidates cached work;
- changing mobility, visibility, custom depth, stencil, material overrides or instance buffers can move cost from static cached paths to dynamic update paths;
- one Actor may generate work in depth, base, shadow, velocity, custom depth and other passes;
- scene captures duplicate view traversal and pass work for their own views.

### Mesh draw command debugging workflow

Use this when Draw/RHI/render-thread time rises while GPU pass time is not the main limiter:

1. Compare object/primitive/component counts before and after the regression.
2. Count material slots and unique material instances per mesh population.
3. Check for per-frame render-state invalidation, component creation/destruction or material override churn.
4. Compare independent components against ISM/HISM/Nanite/HLOD where content fits.
5. Inspect Insights render-thread/RHI tracks and draw/primitive counts.
6. Use GPU capture only after CPU-side draw setup has been separated from GPU execution.

Bad fixes include merging all geometry blindly, moving every object to Nanite without checking material/pixel/shadow cost, or converting to instancing when per-instance updates dominate the workload.

### Render targets are memory, bandwidth, and often extra views

A render target is a GPU resource that can be rendered into and then sampled or consumed elsewhere. It is useful for mirrors, portals, minimaps, security cameras, thumbnails, UI previews, painting/simulation buffers, post-process intermediates and debug visualisation. Its cost is not just "one texture". [SRC-RENDER-016]

Cost dimensions:

| Dimension | Why it matters |
|---|---|
| Resolution | Pixel work, memory footprint, bandwidth and copy cost scale with width x height. |
| Format | HDR/float/depth/stencil formats can multiply memory and bandwidth. |
| Update cadence | Every-frame updates are fundamentally different from event/manual updates. |
| View cost | Scene captures run their own visibility, pass and shading work. |
| Mips | Generated mips cost GPU work and memory but may be needed for sampling quality. |
| Clear/copy/readback | Clears, resolves, CPU readback and UI copies can add sync/cost. |
| Lifetime | Many transient per-actor targets can fragment memory or create spikes. |
| Consumers | Sampling in UI/materials/post can add dependent pass cost. |

Render target memory estimate:

```text
width * height * bytes_per_pixel * array_slices_or_faces * mip_factor * buffering

Example:
1024 * 1024 * 8 bytes RGBA16F ~= 8 MB per mip-0 surface
Cube capture = 6 faces ~= 48 MB before mips/history/extra buffers
20 active captures at that size is no longer a harmless UI feature
```

### Scene captures: "another camera" is the safe default assumption

SceneCapture2D renders a view into a render target. SceneCaptureCube renders six directions, so its worst-case view cost can be much larger. The exact feature set and shortcuts depend on settings, show flags and renderer path, but the safe starting assumption is: a scene capture is an additional camera/view with its own visibility and rendering work. [SRC-RENDER-017] [SRC-RENDER-018]

Design rules:

1. Avoid `CaptureEveryFrame` unless the visual requirement truly needs it.
2. Prefer manual or low-frequency capture for thumbnails, inventory previews, minimaps and security screens.
3. Reduce resolution and format aggressively before reducing main-scene quality.
4. Use show-only/hidden actor lists, show flags and max view distance to reduce content.
5. Avoid recursive captures and many active portal/mirror views.
6. Share render targets only when lifetime/consumer rules are explicit.
7. Budget cube captures as six views unless the target capture proves cheaper.
8. Include scene captures in PSO collection and GPU capture paths; they can exercise material/pass combinations the main camera misses.

### Debugging workflow: scene capture tanks frame rate

1. Disable only the capture update, not the consuming UI/material, to isolate capture cost from sampling/composition cost.
2. Record capture resolution, format, update cadence and number of active captures.
3. Compare `CaptureEveryFrame`, movement-triggered and manual capture.
4. Restrict show flags and show-only actors; measure pass reductions.
5. Lower render-target resolution/format and check visual acceptance.
6. Inspect whether the capture causes unique PSOs/shader paths on first use.
7. Profile memory for render target allocation and mip generation.
8. Add a per-feature capture budget: max active captures, update frequency, resolution tier and platform fallback.

### GPU capture escalation workflow

Unreal's `stat gpu`, GPU Visualiser, RDG Insights and view modes answer "which pass and likely mechanism?" External GPU captures answer "what exact pipeline state/resources/shaders did the GPU execute?" Use both. [SRC-RENDER-003] [SRC-RENDER-019]

Escalate when:

- pass timings identify the pass but not the resource/shader/pipeline cause;
- a platform/RHI-specific bug appears;
- shader output is wrong despite engine-level logic seeming correct;
- bandwidth/occupancy/register pressure/resource hazards need inspection;
- PSO/resource binding questions cannot be answered from Unreal logs;
- vendor/console performance differs from desktop expectations.

Capture checklist:

1. Use a reproducible packaged or controlled development build.
2. Disable frame caps/VSync/dynamic resolution unless testing them.
3. Add named GPU events/scopes around custom passes where allowed.
4. Capture the minimal bad frame after warm-up.
5. Record engine commit, RHI/API, GPU/driver, scalability, resolution and content path.
6. Inspect event hierarchy, render targets, pipeline state, bound shaders/resources, draw/dispatch dimensions and expensive draws.
7. Translate findings back into an Unreal-level fix and re-measure in normal profiling tools.

RenderDoc is a strong PC/Vulkan/D3D/OpenGL-style capture tool where supported, but platform/vendor tools such as PIX, Nsight Graphics, Xcode GPU tools and console SDK tools may be more appropriate for a specific target. Do not assume a RenderDoc-only diagnosis generalises to consoles or mobile.

### Specialist common bugs

1. Treating shader compile time, runtime shader cost and PSO hitching as one problem.
2. Adding static switches until permutation count explodes.
3. Testing a shader in editor with warm DDC, then shipping a cold packaged hitch.
4. Failing to map a plugin shader directory or loading the module too late.
5. Capturing stack/local data in an RDG pass that executes later.
6. Omitting an RDG resource dependency and relying on incidental barriers.
7. Assuming one Static Mesh means one draw in every pass.
8. Recreating components/material overrides every frame and invalidating render-state caches.
9. Giving every actor its own 1024 or 2048 render target for UI/world previews.
10. Leaving many SceneCapture2D components on `CaptureEveryFrame`.
11. Using cube captures like cheap 2D thumbnails.
12. Missing scene-capture-only PSOs in cache collection.
13. Fixing a PSO hitch by lowering material quality without verifying cache coverage.
14. Making a GPU capture without recording RHI/platform/build settings.

### Strong interview answer patterns

**Shader permutations and PSO hitches**

> I separate shader permutation management from runtime GPU cost and PSO creation. Static switches and material features can multiply compiled variants and PSO combinations. A packaged first-use hitch may be a missing PSO even if shader code exists. I would collect representative PSOs in target packaged builds, include rare VFX/UI/scene capture paths, package/precache through the platform workflow and verify cold-cache first-use frames.

**Mesh draw pipeline**

> A component contributes render-side primitive data and mesh batches; pass processors produce draw commands for depth, base, shadow, velocity and other passes. Cached mesh draw commands help suitable static work, but dynamic state changes, many material slots, material overrides and component churn can create render-thread cost. I would inspect Draw/RHI, primitive/draw counts, material diversity and render-state invalidation before changing geometry representation.

**Render target / scene capture cost**

> A render target is memory and bandwidth; a scene capture is usually another view. I budget resolution, format, update frequency, view flags and active count. For minimaps/thumbnails/security cameras, I prefer manual or low-rate captures with show-only lists and reduced formats. Cube captures are especially costly because they can render six views. I include capture paths in PSO collection and GPU profiling.

**When to use RenderDoc or platform GPU tools**

> I use Unreal's GPU Visualiser/stat/RDG tools first to name the pass. I escalate to RenderDoc/PIX/Nsight/Xcode/console tools when I need exact pipeline state, bound resources, shader assembly/occupancy clues or platform-specific behaviour. The capture must record engine/RHI/platform/settings and lead back to a normal profiling before/after result.

### Hands-on specialist rendering extension

Extend Project 4 with five specialist cases:

1. **Shader permutation audit:** create a material family with bounded and unbounded static switch variants. Record compile count/time, shader library/DDC impact where visible, runtime material pass cost and PSO diversity.
2. **Global shader/RDG pass:** in a source-capable branch, add a minimal named compute or full-screen global shader plugin pass. Validate shader directory mapping, parameter struct dependencies, output correctness and GPU capture visibility.
3. **PSO first-use hitch lab:** create a rare VFX/UI/scene-capture material path, reproduce first-use hitch in packaged build, collect/cache/precache the PSO through target workflow and verify cold-cache improvement.
4. **Mesh draw command stress:** compare 1000 independent components, ISM/HISM, Nanite-supported meshes and a deliberately render-state-dirty version. Record Draw/RHI, GPU pass times, primitive/draw counts and visual/culling consequences.
5. **Scene capture budget:** build minimap, thumbnail and mirror/security-camera variants. Compare `CaptureEveryFrame`, manual capture, low-frequency capture, show-only lists, resolution/format tiers and cube capture cost. Include memory and PSO coverage.

Exit evidence:

- target branch/source notes for shader/PSO APIs;
- packaged build captures, not only editor;
- `stat unit`, `stat gpu` or GPU Visualiser before/after;
- Unreal Insights/RDG Insights where relevant;
- at least one RenderDoc or platform capture with event names;
- PSO cache coverage note including scene captures/VFX/UI;
- a platform fallback table for low-end/mobile/console targets.

### Specialist source caveats

- `SRC-RENDER-012` PSO cache workflow is RHI/platform/project-setting sensitive. Confirm commands, file formats and precache behaviour in the exact branch and packaged target.
- `SRC-RENDER-013` and `SRC-RENDER-014` are implementation guides, but shader plugin/global shader symbols and module loading details require target compile proof.
- `SRC-RENDER-015` is renderer-internal and exact mesh draw command classes/caches evolve. Use it for mental model and branch-source routing, not memorised private names.
- `SRC-RENDER-016` through `SRC-RENDER-018` establish render target and scene capture concepts; production cost comes from target captures and content settings.
- `SRC-RENDER-019` is a RenderDoc integration anchor. Final performance conclusions for console/mobile/vendor-specific paths need the platform's own tools where applicable.

## 19. Target-branch packaged proof playbook

This playbook turns the specialist cluster into something you can actually execute on a real branch. The goal is not “I read the docs”, but “I can prove which stage failed, what the shipped evidence says, and whether the fix holds in a cold packaged run”. [SRC-RENDER-012] [SRC-RENDER-013] [SRC-RENDER-014] [SRC-RENDER-019]

### Branch-proof checklist before touching rendering code

Before changing shader/plugin/RDG/capture code, record:

1. exact engine branch/commit and whether it is source-capable;
2. target platform/RHI/feature level (`D3D12`, `Vulkan`, `SM5`, mobile renderer, etc.);
3. whether the path must work in Editor only, Development packaged, Shipping, dedicated server builds, or all of them;
4. where the evidence must come from: Unreal Insights, RDG Insights, GPU Visualiser, RenderDoc, PIX, Nsight, Xcode or platform tools;
5. what “success” means: visual correctness, hitch removal, memory reduction, stable cold-start behaviour, or all of these.

If you skip this metadata, later captures become difficult to compare and packaged-only failures are easy to mislabel as “random renderer instability”.

### Failure-triage matrix: black output, hitch, or memory spike?

| Symptom | First classification question | Likely next evidence |
|---|---|---|
| Custom pass is black/wrong | Is the pass present with the expected event name in packaged build? | RDG Insights, GPU capture, constant-colour output test |
| First use of effect hitches | Is this shader compile, PSO creation, asset streaming or render-target allocation? | packaged hitch trace, PSO logs, cold-cache rerun |
| GPU pass is expensive every frame | Which pass is hot, and does cost follow resolution/view count/material complexity? | GPU Visualiser, resolution scaling, show-flag isolation |
| Scene capture tanks frame rate | Is cost from capture rendering, target format/resolution, or downstream UI/material sampling? | disable capture update only, memory/cadence table, capture-specific pass timings |
| Render-thread cost rises but GPU does not | Are draw-command churn, material diversity or component invalidation dominating? | Insights render tracks, primitive/draw counts, `MarkRenderStateDirty` audit |
| Memory spikes after adding captures/targets | Is the spike transient lifetime, mip generation, cube faces, history buffers or many always-live targets? | allocation table, target inventory, update cadence and lifetime review |

This is the habit worth showing in interview answers: classify the failure mode before changing features.

### Schematic branch-proof code shape for plugin shader startup

```cpp
// Schematic only. Verify names/macros/APIs in the exact target branch.

class FMyRenderPluginModule : public IModuleInterface
{
public:
    virtual void StartupModule() override
    {
        const FString ShaderDir = FPaths::Combine(IPluginManager::Get().FindPlugin(TEXT("MyRenderPlugin"))->GetBaseDir(), TEXT("Shaders"));
        AddShaderSourceDirectoryMapping(TEXT("/Plugin/MyRenderPlugin"), ShaderDir);
    }

    virtual void ShutdownModule() override
    {
        // Keep shutdown symmetrical with startup if the branch requires cleanup.
    }
};
```

Why this matters:

- shader-directory mapping is part of the build/runtime contract, not cosmetic setup;
- module load timing can decide whether shader registration happens before compilation/cook expects it;
- “works in editor” is weak evidence if the packaged branch loads modules differently;
- a render feature should have one named proof point in GPU tools so you can show it exists and binds what you think it binds.

### Packaged-proof loop for rendering changes

Use this loop whenever the feature must survive outside editor:

1. make the smallest bounded rendering change;
2. verify visual correctness in editor with a deliberately simple output;
3. package a target build with cold caches where practical;
4. reproduce the feature path in a deterministic map/camera/content setup;
5. capture one short before/after evidence packet: build metadata, map/camera path, pass timings, RDG/GPU capture if needed and whether the effect hit a new PSO path;
6. clear the most relevant local cache and rerun once more to avoid proving only a warm-developer-machine success;
7. write down the fallback if the path is too expensive on low-end/mobile/console targets.

A strong specialist answer does not stop at “RenderDoc said this draw was expensive”. It finishes with “here is the target build, here is the pass, here is the classified mechanism, and here is the shipping-safe intervention plus fallback policy”.

### Boundary cases worth calling out explicitly

- A shader pass can be correct in editor but absent in packaged build because the module or shader directory mapping was not present in the packaged startup path.
- A fix that lowers GPU time can still regress first-use hitching by multiplying permutations or PSO diversity.
- A scene capture optimisation can reduce GPU time but increase correctness risk if UI depends on cadence or stale target lifetime.
- A RenderDoc capture can explain one PC frame and still be insufficient for mobile/console submission, bandwidth or tile-memory questions.
- A render-target pool that looks fine in a short run can fail after menu/world traversal because lifetime and “only sometimes active” assumptions were wrong.
