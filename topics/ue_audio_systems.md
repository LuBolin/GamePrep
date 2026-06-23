# Unreal Audio Systems for Interviews

See also: [[ue_runtime_presentation_target_proof]], [[ue_niagara_vfx]], [[ue_ui_systems]], [[ue_hands_on_projects]].

Game audio is a real-time resource-allocation and signal-routing system, not a collection of `Play Sound` calls. A strong Unreal engineer can choose the right source abstraction, predict source lifetime, spatialise for the correct listener, control concurrency and voice stealing, route categories through mixes/submix DSP, budget compressed/decompressed data, and prove whether a missing sound was never requested, rejected, virtualised, stolen, inaudible or starved.

Version target: **UE5.3–UE5.6**. Sound Waves, Sound Cues, Audio Components, attenuation, concurrency, classes and submix concepts are core. MetaSounds is UE5 and its graph/runtime details evolve. Quartz, maintained Audio Stream Cache and console-command pages currently surface as UE5.7; they are labelled specialist/version-sensitive unless target APIs are verified.

## 1. End-to-end mental model

```text
gameplay/presentation event or continuous state
        ↓ request + identity + parameters
USoundBase source design
  SoundWave / SoundCue / MetaSound Source
        ↓ source instance / AudioComponent lifetime
attenuation + spatialisation + occlusion + focus
        ↓ admission
concurrency group + priority + voice/virtualisation policy
        ↓ routing
SoundClass/Mix → source effects → Submix sends/effects → endpoint
        ↓
audio render thread buffers → platform backend → output device
```

Each stage can suppress or transform the result. “The asset plays in the editor” verifies only a narrow slice.

## 2. Source assets: Wave, Cue and MetaSound

### Sound Wave

A Sound Wave wraps imported/sample audio plus playback, compression/loading, looping, priority, class, concurrency and attenuation-related defaults. It is the underlying recorded content, not automatically a complete product-level behaviour. [SRC-AUDIO-001]

Use a Wave directly for a simple stable one-shot or music/stem whose behaviour belongs elsewhere. Avoid cloning the same Wave merely to vary runtime volume/pitch when parameters or a higher-level source design suffice.

### Sound Cue

A Sound Cue is a Sound Node graph that references Waves and adds randomisation, modulation, branching/concatenation and named parameter-controlled behaviour. [SRC-AUDIO-001]

It remains useful for authored sample-based variation. Random selection is not concurrency policy, and a Cue graph is not a gameplay state machine. Keep semantic decisions—surface, weapon state, ability validation—in gameplay/presentation code and pass bounded parameters/asset selections.

### MetaSound Source

MetaSounds are DSP rendering graphs with procedural generation, graph reuse, parameters and sample-accurate control at audio-buffer/sample scale. Their Wave Player supports timing features such as seeking, loop points and sample-accurate concatenation. Parameters can be supplied through the Audio Component parameter interface. [SRC-AUDIO-007]

Use MetaSound when sound design needs procedural signal flow, sample-accurate internal sequencing or rich parameter-driven synthesis. Do not migrate every Cue because MetaSound is newer. Each MetaSound instance performs DSP work; polyphony, graph complexity and platform budget still matter.

### Separation rule

- gameplay decides **whether and why** an audible event/state exists;
- source graph decides **how that event sounds**;
- attenuation decides **listener-relative perception**;
- concurrency/priority decides **which instances consume scarce voices**;
- routing/mix decides **category balance and shared processing**.

## 3. One-shot versus controlled source lifetime

Choose an API by required lifetime/control:

- a fire-and-forget location one-shot suits an event needing no later stop/fade/parameter/follow control;
- a spawned attached sound creates an Audio Component that follows a Scene Component and can expose control;
- a persistent owned Audio Component suits engine loops, ambience emitters, charge sounds, dialogue or state-driven machines.

`SpawnSoundAttached` exposes attachment, stop-when-owner-destroyed and auto-destroy behaviour. [SRC-AUDIO-002] A fire-and-forget call does not give product code a durable source handle.

### One source owner

For a loop, define one owner and transition it explicitly:

```text
Stopped → Starting → Playing → FadingOut → Stopped
                    ↘ Interrupted ↗
```

Starting a new loop each Tick is not continuous audio; it is uncontrolled polyphony. Cache the Component, update parameters at a sensible rate, and stop/fade once. Handle owner destruction, component deactivation, level travel and pooled-Actor reuse.

### Auto-destroy and pooling

Auto-destroy is convenient for true one-shots. Do not retain a pointer and assume it remains valid after completion. Conversely, disabling auto-destroy means the owner/pool must reset sound, parameters, delegates, transform, attachment and playback state before reuse.

Pool only after measuring creation/churn and proving reset semantics. Audio concurrency often limits real voices without requiring a bespoke Component pool.

## 4. Parameters and update cadence

Use stable named audio parameters to communicate presentation state: RPM, speed, surface intensity, charge, weather or health tension. Establish parameter units/range/default and ownership. Smooth discontinuous values in the appropriate domain; do not send noisy updates every frame when 10–20 Hz with interpolation is perceptually equivalent.

Parameter writes cross from gameplay to audio processing. Batch/coalesce where supported and avoid creating strings/looking up names repeatedly. A MetaSound graph cannot safely query arbitrary gameplay UObjects on the audio render thread; push a bounded value snapshot through supported interfaces.

## 5. Listener, spatialisation and attenuation

Attenuation defines how a source is perceived relative to one or more listeners. Its settings can include distance-volume curves/shapes, spatialisation, air absorption, focus, occlusion, reverb/submix sends and priority scaling. [SRC-AUDIO-003]

### Volume attenuation

The inner shape/area is full level; distance beyond it follows the chosen curve until the outer boundary/minimum. Choose distances from world scale and audibility goals, not one convenient default. Log source-listener distance and effective gain when debugging.

### Spatialisation

Spatialisation maps source direction to the output format/headphone/speaker model. Mono source content generally provides clean point-source positioning; stereo assets may have different spatial behaviour. Platform plugin/HRTF support and channel format are target-sensitive.

### Occlusion and obstruction

Occlusion tests geometry between listener and source and can apply volume/low-pass changes. It is an approximation, not physical acoustic simulation. Queries cost CPU and can flicker at doorways/thin geometry, so choose channel, interval, interpolation and source significance. Do not run expensive traces for thousands of inaudible sources.

### Listener focus and priority

Focus can treat sounds in front of the listener differently from off-axis sounds, affecting volume, distance or priority with interpolation. Priority attenuation can lower far/off-focus sources before voice admission. [SRC-AUDIO-003]

In split-screen/local multiplayer, listener selection/multi-listener behaviour needs explicit product tests. Dedicated servers have no human listener and should not run client-only audio rendering logic.

## 6. Concurrency, priority, stealing and virtualisation

Hardware/software source voices are finite. Sound Concurrency groups define how many active instances a semantic family can have and how a new request resolves when the limit is reached. UE5.6 settings include maximum count, optional owner scoping, retrigger time, resolution policy, ducking/scaling and voice-steal release behaviour. [SRC-AUDIO-004]

### Concurrency is semantic

Useful groups might be:

- one weapon mechanical loop per weapon owner;
- bounded footsteps per character and globally;
- small UI click group;
- dialogue category that protects intelligibility;
- bounded explosion/debris group with distance/priority policy.

Do not put every sound in one global group. Do not give each asset its own group if the product constraint spans many variants.

### Owner scoping

Per-owner concurrency works only when the playback request supplies meaningful ownership. If no owner is available it can fall back to global behaviour. [SRC-AUDIO-004] A fire-and-forget call with no semantic owner may therefore behave differently from an owned Component.

### Resolution and voice stealing

When full, policy can reject the new request or stop/replace an existing instance according to age, distance, priority or quietness, depending on configured options. A stolen sound was successfully requested but lost admission later. Use a release fade where abrupt cuts are audible.

Priority should encode product importance, then attenuation/focus may modify it. Dialogue/critical warnings can outrank debris, but “highest priority everywhere” only destroys prioritisation.

### Virtualisation

Virtualisation can preserve logical playback/timeline while a source is not consuming an audible hardware voice, then resume when relevant, subject to settings. It is useful for persistent loops/music-like timelines, not automatically for every transient. Verify whether the source was stopped, rejected, stolen or virtualised—these are different diagnoses.

## 7. Sound Classes, Sound Mixes and Submixes

### Sound Classes

Sound Classes group sources hierarchically for shared properties such as volume/pitch and routing/loading policy. Sound Mixes temporarily adjust classes during gameplay; passive mix modifiers can activate based on class playback/volume thresholds. [SRC-AUDIO-005]

Use classes for semantic categories and user settings: Master, Music, SFX, Dialogue, UI, Ambience. Store user sliders as settings, then apply them through the category system. Avoid multiplying volume independently in every widget/Actor.

### Submixes

A Submix is an always-running DSP/mixing graph. Multiple source signals flow into a submix, shared effects process the mixed buffer, and submix output flows to one downstream output/master endpoint. Sources can also send portions to additional submixes, including distance-driven sends. [SRC-AUDIO-006]

Use submixes for shared signal processing/routing: reverb buses, radio/dialogue processing, dynamics, recording/capture or platform endpoints. An always-running deep effect graph has cost even during silence depending on effects; profile it.

### Source effects versus submix effects

- source effect: per source instance before shared mixing; cost scales with voices;
- submix effect: processes the combined bus; cost scales with active graph/channel/buffer complexity rather than one copy per source.

Do not use a Submix volume as the only user category-mix strategy without understanding sends and dry paths. Signals can reach multiple buses; draw the route and check pre/post-attenuation send stage.

### Ducking

For dialogue ducking music/SFX, define attack, release, minimum attenuation and ownership of the duck request. Stack/reference-count overlapping requests or use a mix/envelope-driven policy so one dialogue ending does not release ducking while another remains active.

## 8. Timing: game frame, audio buffer, Quartz and MetaSound

Ordinary play commands cross from game timing to audio render buffers. “Called on this frame” does not imply sample-accurate output; command consumption and platform buffers introduce latency/jitter.

Quartz schedules against seconds or musical bars/beats ahead of rendering to provide sample-accurate playback independent of game-frame boundaries. It is useful for music transitions and timing-sensitive repeated effects. [SRC-AUDIO-008]

Quartz does not make gameplay deterministic or remove output-device latency. Gameplay authority still decides an action; Quartz schedules local presentation. Networked music synchronisation additionally needs a shared clock/epoch, drift correction and late-join policy.

MetaSound provides sample-accurate behaviour inside its DSP graph. Use Quartz for scheduling/transport across sources and MetaSound for source-internal signal timing, while verifying target API/status.

## 9. Loading, compression, streaming and residency

Audio has multiple relevant memory forms:

- source/import data in editor;
- cooked compressed chunks on storage/cache;
- decoded PCM buffers;
- streaming/cache metadata and active decoder state;
- source/effect/submix processing buffers.

Audio stream caching separates most compressed data from the loaded `USoundWave` and loads/releases chunks on demand. Therefore a loaded Wave does not guarantee immediate audible readiness. A small cache can evict aggressively or run out of available chunks; retaining too much trades latency for memory. [SRC-AUDIO-009]

### Policy examples

- tiny latency-critical UI/weapon transients: retain/prime as budget allows;
- long music/dialogue/ambience: stream, with startup/next-chunk strategy;
- optional catalogue audio: soft/load near need, then release by residency owner;
- repeated combat set: prime before encounter/loading boundary.

Compression quality and codec are platform/content dependent. Long stereo music and short mono effects have different CPU, memory and quality trade-offs. Test cooked target hardware; editor playback is not representative.

Cache trimming can lock/work and cause underruns if used carelessly. Tune with concurrent long streams, bursts of short sounds and actual chunk sizes—not just total megabytes.

## 10. Networking and authority

Audio is usually local cosmetic presentation. Do not replicate Audio Components or rely on “multicast play sound” as durable state.

Choose by semantic type:

- **durable state:** replicate the gameplay state; clients start/stop the matching loop and late joiners reconstruct it;
- **important transient:** replicate/derive an event with stable identity/sequence and relevancy, then play locally;
- **owner prediction:** play responsive local feedback, correlate with server acceptance and correct/cancel unobtrusively;
- **pure local feedback:** UI hover/click, accessibility cues and local settings need no server traffic.

The server validates gunfire/damage, but it does not need to render the muzzle sound. A dedicated server must remain correct with audio disabled. Avoid double-play when an owning client predicts a sound and later receives the replicated event; use prediction/event IDs or owner suppression.

## 11. Debugging: why did I not hear the sound?

Follow the signal path:

1. **Request:** did the gameplay/presentation event fire once on this machine? Log source, owner, location, parameters and event ID.
2. **Asset:** is the expected Wave/Cue/MetaSound loaded/cooked and valid? Does the exact runtime source graph reach output?
3. **Component:** was it created/playing, auto-destroyed, stopped, detached, deactivated or reused?
4. **Listener:** correct World/audio device/listener? Source distance/orientation and attenuation shape?
5. **Admission:** concurrency group full, retrigger rejected, priority lost, voice stolen or virtualised?
6. **Gain:** effective Wave/Component/Class/Mix/attenuation/modulation volume; muted/paused/UI/background flags?
7. **Routing:** base submix and sends reach the intended endpoint; effect not filtering/silencing it?
8. **Data/render:** stream chunk ready, decoder/cache healthy, audio render not underrunning?
9. **Platform:** output device/channel/plugin and packaged settings match target?

Audio console commands provide runtime debugging toggles for mixes, devices and audio systems; query each target CVar with `?` because names/behaviour evolve. [SRC-AUDIO-010]

### Common symptom splits

- plays in asset preview, not game: request/World/listener/concurrency/routing/cook;
- first play hitches or misses: asset/chunk not primed, decoder/setup or cache pressure;
- cuts out under battle load: concurrency/priority/voice stealing/cache, not necessarily attenuation;
- sounds doubled: predicted + replicated event, duplicate notify/callback, two Components or loop restarted;
- attached sound stays behind: used location one-shot, wrong attachment/location type, or pooled owner reset;
- mix remains ducked: unbalanced push/pop/reference count or passive mix still active.

## 12. Profiling and optimisation

Separate these cost domains:

- gameplay event/parameter update CPU;
- active logical sources versus audible/rendered voices;
- source graph/DSP cost, especially procedural MetaSounds;
- spatialisation/HRTF, occlusion queries and environment sends;
- decoder/streaming I/O and cache hits/evictions/underruns;
- source effects multiplied by voices;
- always-running submix graph/effects and channel count;
- audio render callback time/buffer headroom;
- compressed, decoded and cache memory;
- platform output/backend latency.

Scale with representative stress: quiet scene, 100 simultaneous impacts, many occluded emitters, dialogue + music + combat, cache churn and low-end platform. Count requests, rejections, steals, virtual voices and active Components. Optimise semantic population before micro-optimising graphs.

### Optimisation order

1. stop duplicate/restarted sources and unnecessary Components;
2. define concurrency/priority/virtualisation by perceptual importance;
3. reduce insignificant occlusion/parameter update frequency;
4. simplify high-polyphony source graphs/effects;
5. move common compatible processing to a shared submix;
6. tune loading/cache/compression against measured latency/memory/I/O;
7. quality-scale spatialisation, effects and voice budgets by platform;
8. recapture audible quality and performance together.

## 13. What a three-year engineer should know

Expected D3–D4:

- Wave versus Cue versus MetaSound responsibility;
- one-shot versus controlled/attached Audio Component lifetime;
- parameter push and loop ownership;
- attenuation, listener, spatialisation and occlusion concepts;
- concurrency groups, owner scope, priority, stealing and virtualisation;
- Sound Class/Mix versus Submix/source-effect routing;
- game-frame versus audio-buffer timing and Quartz awareness;
- stream/cache/residency and packaged-platform testing;
- cosmetic/network authority and predicted duplicate prevention;
- signal-path debugging and population/voice/cache/DSP profiling.

Specialist D4–D5:

- audio render-thread/source manager internals;
- custom MetaSound nodes/DSP and SIMD;
- HRTF/ambisonics/soundfield/plugin integration;
- acoustic propagation/reverb systems;
- dynamic music transport/network clocking;
- codec/platform backend and underrun diagnosis.

## 14. Common misconceptions

| Misconception | Better model |
|---|---|
| “Play Sound is cheap because it is fire-and-forget.” | It still creates/admit/processes a source and can cause uncontrolled polyphony. |
| “Sound Cue is old MetaSound.” | Cue is an authored Sound Node graph; MetaSound is a DSP graph. Choose by need. |
| “Concurrency is the platform voice limit.” | It is semantic admission policy layered within wider source/voice budgets. |
| “Low volume means low cost.” | A quiet source can still decode, spatialise, run DSP and occupy a voice unless culled/virtualised. |
| “Occlusion makes audio realistic.” | It is a configurable query/filter approximation with CPU and flicker trade-offs. |
| “Sound Class and Submix are interchangeable buses.” | Class groups/control source properties; submix is signal/DSP routing. |
| “Quartz removes latency.” | It schedules accurately by accounting ahead; device/buffer/end-to-end latency still exists. |
| “Loaded SoundWave is ready immediately.” | Streamed compressed chunks may still need loading/decoding. |
| “Multicast the sound.” | Replicate durable state or bounded events; clients render local presentation. |
| “Dedicated server should call the same audio path.” | Gameplay must be independent of client audio devices/rendering. |

## 15. Strong interview answer patterns

### Diagnose a missing combat sound

> I trace request → source/component → listener/attenuation → concurrency/priority/virtualisation → class/mix/submix gain → stream/cache/render → output device. I log semantic owner and event ID, inspect active/rejected/stolen sources, and reproduce under the battle voice load. That distinguishes “never requested” from “requested but inaudible”, which need completely different fixes.

### Design scalable footsteps

> Gameplay/animation reports a correlated foot-contact presentation event; surface selection is data-driven. The client plays a varied Wave/Cue/MetaSound through per-owner and global concurrency, distance/focus priority and suitable attenuation. Distant insignificant footsteps are rejected/virtualised rather than traced and rendered. I profile request/voice/steal/occlusion/cache counts with many characters and keep gameplay damage/movement independent of audio.

### Build a dynamic mix

> I classify sources with Sound Classes for user/category control and route signals into a documented Submix graph for shared DSP. Dialogue ducking uses balanced/reference-counted mix state with attack/release; sends and dry paths are drawn explicitly. I test overlapping dialogue, pause/menu/background, device changes and ensure silence does not leave expensive unintended DSP or stuck mix state.

## 16. Hands-on verification project

Extend Project 1/4 with an Audio Runtime Lab:

1. implement one-shots, attached sources and persistent parameter-driven loops with explicit ownership;
2. build Wave, Sound Cue and MetaSound variants of a weapon/engine and document why each exists;
3. author attenuation/focus/occlusion and visualise source/listener/query states;
4. create per-owner/global concurrency groups and force reject/steal/virtualise cases;
5. build Master/Music/SFX/Dialogue/UI classes, temporary mixes and a Submix graph with shared effects/sends;
6. implement overlapping dialogue duck requests without stuck early release;
7. schedule a music transition/repeated weapon cadence with Quartz only after target verification;
8. test retain/prime/on-demand/stream policies in a cooked build under cache pressure;
9. run dedicated server/two clients with predicted and replicated events, proving no double-play and late-loop reconstruction;
10. inject destroyed attachment, pooled stale Component, wrong listener, zero gain, full concurrency, stolen voice, missing chunk and disconnected route;
11. profile 1/20/100 emitters and a 100-impact burst on target hardware;
12. deliver source-lifetime state diagrams, concurrency matrix, routing graph, memory/cache policy and before/after audible performance report.

## 17. Source map

- source assets/components/parameters: [SRC-AUDIO-001] [SRC-AUDIO-002] [SRC-AUDIO-007]
- spatialisation/admission: [SRC-AUDIO-003] [SRC-AUDIO-004]
- classes/mixes/submix signal routing: [SRC-AUDIO-005] [SRC-AUDIO-006]
- timing/loading/debugging: [SRC-AUDIO-008] [SRC-AUDIO-009] [SRC-AUDIO-010]
