# UE Interview Knowledge Graph

The machine-readable counterpart is `ue_interview_knowledge_graph.json`. This human view shows the currently evidenced cluster only; later clusters will connect to it without changing IDs.

```mermaid
flowchart TD
    CPP["C++ object lifetime"] --> RAII["RAII and Rule of Zero/Five"]
    RAII --> MOVE["Copy, move, and value return"]
    MOVE --> OWN["Unique/shared/weak ownership"]
    CPP --> INV["Iterator and handle invalidation"]
    CL["C++ object lifetime"] --> UHT["UHT and generated code"]
    UHT --> REF["Reflection macros"]
    REF --> UOBJ["UObject model"]
    UOBJ --> CDO["CDO and construction"]
    UOBJ --> GC["GC reachability"]
    GC --> PTR["UObject pointer selection"]
    CDO --> PTR
    PTR --> ASSET["Hard vs soft asset references"]
    PTR --> ASYNC["Async lifetime safety"]
    GC --> PERF["GC profiling"]
    UOBJ --> ACT["Actor and Component model"]
    CDO --> LIFE["Actor lifecycle"]
    ACT --> PC["Pawn, Character, Controller"]
    PC --> STATE["GameMode/State and PlayerController/State"]
    STATE --> SUB["Subsystem lifetimes"]
    OWN --> DOM["C++ versus UObject lifetime domains"]
    PTR --> DOM
    REF --> STR["FName, FString, FText"]
    INV --> CONT["TArray, TSet, TMap"]
    LIFE --> CREATE["Creation APIs and Cast"]
    OWN --> DEL["Delegate lifetime"]
    DEL --> BP["C++/Blueprint API boundary"]
    ACT --> BP
    STATE --> AUTH["Authority, roles, owning connection"]
    AUTH --> PROP["Properties, OnRep, conditions"]
    PROP --> RPC["RPC routing and reliability"]
    PROP --> REL["Relevancy and dormancy"]
    RPC --> PRED["Movement prediction and correction"]
    PRED --> SAVEDMOVE["Custom CharacterMovement saved moves"]
    AUTH --> SAVEDMOVE
    RPC --> REWIND["Lag compensation and server rewind"]
    AUTH --> REWIND
    ALG --> REWIND
    SAVEDMOVE --> NETTERMS["Prediction, reconciliation, rewind, and rollback terms"]
    REWIND --> NETTERMS
    NETTERMS --> NPBRANCH["NetworkPrediction plugin branch proof"]
    PRED --> NPBRANCH
    NPBRANCH --> NPSTATE["Rollback input, sync, and aux state model"]
    AUTH --> NPSTATE
    NETTERMS --> NPCUE["Rollback cues and side-effect safety"]
    NPSTATE --> NPRECON["Rollback reconcile proof and profiling"]
    NPCUE --> NPRECON
    REL --> NPROF["Network profiling and scalability"]
    RPC --> NPROF
    REWIND --> NPROF
    NPRECON --> NPROF
    AUTH --> NETSUB["Replicated Components and UObject subobjects"]
    PROP --> NETSUB
    UOBJ --> NETSUB
    ACT --> NETSUB
    PROP --> FASTARR["Fast Array and push-model replication"]
    CONT --> FASTARR
    NETSUB --> FASTARR
    REL --> REPGRAPH["Replication Graph scaling and relevance lists"]
    NPROF --> REPGRAPH
    ALG --> REPGRAPH
    NETSUB --> IRIS["Iris replication awareness"]
    AUTH --> IRIS
    REL --> NETCVARS["Network debug CVars and evidence workflow"]
    NETSUB --> NETCVARS
    FASTARR --> NETCVARS
    INS --> NETCVARS
    PC --> AIC["AIController decision pipeline"]
    AIC --> BT["Behaviour Tree and Blackboard"]
    BT --> ST["StateTree / FSM comparison"]
    AIC --> PER["Perception and memory"]
    PER --> CUSTSENSE["Custom AI sense pipeline"]
    CUSTSENSE --> AIPROJ["Evidence decay and memory projection"]
    PER --> EQS["EQS candidates and scoring"]
    AIPROJ -.-> EQS
    BT --> EQS
    AIC --> NAV["NavMesh, path following, avoidance"]
    EQS -.-> NAV
    ST --> STFLOW["StateTree execution and data flow"]
    BP --> STFLOW
    AIC --> SOAFF["Smart Object affordances and slots"]
    DATA --> SOAFF
    SOAFF --> SOLIFE["Smart Object query, claim, use, release"]
    STFLOW --> SOLIFE
    NAV --> SOLIFE
    SOLIFE --> SOSCALE["Smart Object authority, debugging, scaling"]
    AUTH --> SOSCALE
    INS --> SOSCALE
    SOSCALE -.-> MASSPROC
    MOVE --> FRAME["Frame budget and bottleneck classification"]
    FRAME --> INS["Timing Insights and instrumentation"]
    INS --> MEM["Memory, churn, and GC"]
    INS --> CPUOPT["Gameplay CPU optimisation"]
    FRAME --> SCALE["Scalability and device profiles"]
    FRAME --> BENCH["Target benchmark capture contract"]
    INS --> BENCH
    SCALE --> BENCH
    MEM --> MEMBUD["Memory Insights and LLM budgets"]
    BENCH --> MEMBUD
    BENCH --> CSVGATE["CSV performance regression gates"]
    MEMBUD --> CSVGATE
    RELAUTO -.-> CSVGATE
    CSVGATE -.-> SCALE
    FRAME --> RPIPE["Render pipeline, RHI, and passes"]
    INS --> RPIPE
    RPIPE --> RCOST["Submission, geometry, pixel, shader, bandwidth cost"]
    RCOST --> MAT["Materials, permutations, WPO, translucency"]
    MAT --> PSO["Shader permutations and PSO cache workflow"]
    RCOST --> PSO
    RCOST --> GEO["Culling, ISM/HISM, LOD/HLOD, Nanite"]
    GEO --> MESHDRAW["Mesh drawing pipeline and draw command pressure"]
    RPIPE --> MESHDRAW
    RPIPE --> LUMEN["Lumen tracing and scene representations"]
    RCOST --> LUMEN
    GEO --> VSM["VSM pages and invalidation"]
    MAT -.-> VSM
    RPIPE --> RDG["RDG resources and dependencies"]
    CPP --> RDG
    RDG --> GSHADER["Global shader and plugin shader workflow"]
    PSO --> GSHADER
    RPIPE --> RTCAP["Render targets and scene capture budgets"]
    RCOST --> RTCAP
    PSO --> RTCAP
    RDG --> GPUCAP["RenderDoc and platform GPU capture workflow"]
    RCOST --> GPUCAP
    GSHADER -.-> GPUCAP
    RTCAP -.-> GPUCAP
    ACT --> COMP["Component contracts and composition"]
    LIFE --> COMP
    DEL --> COMM["Calls, Observers, Commands, and queues"]
    RPC --> COMM
    COMP --> POLICY["State and Strategy policy"]
    PTR --> DATA["Definitions, runtime state, presentation"]
    BP --> DATA
    SUB --> SERVICE["Subsystem and service boundaries"]
    COMP --> SERVICE
    DATA --> TX["Transactions, revisions, and persistence"]
    COMM --> TX
    AUTH --> TX
    PTR --> AREF["Hard/soft references and dependency closure"]
    DATA --> AREF
    AREF --> ALOAD["Async request and residency lifecycle"]
    LIFE --> ALOAD
    AREF --> AMGR["Registry, Asset Manager, Primary IDs/bundles"]
    AMGR --> COOK["Cook, package, chunks, redirectors, DDC"]
    ALOAD --> WORLDSTREAM["Level and World Partition streaming"]
    LIFE --> WORLDSTREAM
    STATE --> WORLDSTREAM
    AREF --> PCGDATA["PCG graphs, points, density, attributes"]
    DATA --> PCGDATA
    PCGDATA --> PCGMODE["PCG generation modes and output ownership"]
    WORLDSTREAM --> PCGMODE
    COOK --> PCGMODE
    PCGDATA --> PCGDEBUG["PCG debugging and reproducibility"]
    PCGMODE --> PCGPERF["PCG performance and output cost"]
    PCGDEBUG --> PCGPERF
    INS --> PCGPERF
    RCOST --> PCGPERF
    SOLIFE -.-> PCGMODE
    WORLDSTREAM --> WPGRID["World Partition grid/source policy"]
    ALOAD --> WPGRID
    STATE --> WPGRID
    WPGRID --> WPDATA["Runtime Data Layers and durable state"]
    TX --> WPDATA
    WPGRID --> WPOFPA["OFPA and Level Instance collaboration"]
    AREF --> WPOFPA
    WPGRID --> WPHLOD["World Partition HLOD build validation"]
    GEO --> WPHLOD
    RCOST --> WPHLOD
    COOK --> WPHLOD
    WPDATA -.-> PCGMODE
    WPHLOD -.-> PCGMODE
    WPHLOD -.-> CIMATRIX
    UHT --> BUILD["UBT/UHT/compiler/linker/load pipeline"]
    CPP --> BUILD
    BUILD --> MODULE["Module dependency and API boundaries"]
    MODULE --> PLUGIN["Plugin Runtime/Editor boundaries"]
    COOK --> PLUGIN
    BUILD --> LIVE["Live Coding and reinstancing"]
    LIFE --> LIVE
    REF --> LIVE
    PLUGIN --> ETOOLS["Editor automation and safe batch tools"]
    AMGR --> ETOOLS
    ETOOLS --> VALID["Asset validation and explicit repair"]
    DATA --> VALID
    BUILD --> RELAUTO["AutomationTool, BuildGraph, and release gates"]
    COOK --> RELAUTO
    PLUGIN --> RELAUTO
    VALID --> RELAUTO
    RELAUTO --> RELART["Release artifacts, symbols, and crash proof"]
    RELAUTO --> CIMATRIX["CI matrix and clean-machine packaging"]
    RELART --> CIMATRIX
    VALID --> CIMATRIX
    MODULE -.-> CIMATRIX
    RELART -.-> COOK
    SCALE --> PLATMOBILE["Mobile constraints and device profiles"]
    BENCH --> PLATMOBILE
    CIMATRIX --> PLATART["Platform release artifact contract"]
    RELART --> PLATART
    PLATART --> PLATCERT["Lifecycle and certification-adjacent matrix"]
    TX --> PLATCERT
    UILAYOUT --> PLATCERT
    PLATMOBILE --> PLATGATE["Platform target profiling gate"]
    PLATART --> PLATGATE
    CSVGATE --> PLATGATE
    PLATGATE --> ANDROIDPROF["Android profiler correlation"]
    CSVGATE --> ANDROIDPROF
    INS --> ANDROIDPROF
    ANDROIDPROF --> ANDROIDGPU["Android GPU render-pass/counter diagnosis"]
    RCOST --> ANDROIDGPU
    RTCAP --> ANDROIDGPU
    ANDROIDPROF --> ANDROIDLMK["Android LMK, thermal, and ADPF evidence"]
    MEM --> ANDROIDLMK
    SCALE --> ANDROIDLMK
    PLATGATE --> DLABRUN["Device lab matrix and run manifests"]
    CIMATRIX --> DLABRUN
    RELAUTO --> DLABRUN
    DLABRUN --> GAUNTLET["Gauntlet / RunUnreal target orchestration"]
    GAUNTLET --> DLABTEL["Automated telemetry and artifact harvesting"]
    CSVGATE --> DLABTEL
    ANDROIDPROF -.-> DLABTEL
    GPUCAP -.-> DLABTEL
    PLATCERT --> DLABTRIAGE["Flaky device triage and gate promotion"]
    DLABTEL --> DLABTRIAGE
    COMP --> MASSDECIDE["Actor versus Mass trade-off"]
    CPUOPT --> MASSDECIDE
    MASSDECIDE --> MASSDATA["Fragments, archetypes, and chunks"]
    CONT --> MASSDATA
    MASSDATA --> MASSPROC["Queries and processors"]
    MASSPROC --> MASSTRUCT["Deferred structure, Traits, and templates"]
    MASSTRUCT --> MASSLOD["Representation, simulation LOD, and hybrid Actors"]
    GEO --> MASSLOD
    MASSPROC --> MASSROUTE["MassAI ZoneGraph and crowd routes"]
    NAV --> MASSROUTE
    MASSROUTE --> MASSBRIDGE["Mass StateTree and Smart Object bridge"]
    SOLIFE --> MASSBRIDGE
    STFLOW --> MASSBRIDGE
    MASSBRIDGE --> MASSPROMO["Mass Actor promotion and dual-writer contract"]
    MASSLOD --> MASSPROMO
    ACT --> MASSPROMO
    STATE --> GASASC["GAS adoption and ASC owner/avatar"]
    COMP --> GASASC
    AUTH --> GASASC
    GASASC --> GASABILITY["Ability specs, lifecycle, commit, and ending"]
    GASABILITY --> GASTAGS["Tags, attributes, and invariants"]
    GASTAGS --> GASEFFECT["Effects, specs, captures, and stacking"]
    GASABILITY --> GASTASK["Tasks, target data, and cues"]
    GASEFFECT --> GASTASK
    GASTASK --> GASPRED["Prediction, reconciliation, and GE replication"]
    RPC --> GASPRED
    GASPRED --> GASEVID["GAS specialist evidence contract"]
    GASABILITY --> GASGRANT["GAS grant source ledger"]
    GASTAGS --> GASATTRINV["GAS attribute invariant matrix"]
    GASEFFECT --> GASATTRINV
    GASPRED --> GASSIDEFX["GAS prediction side-effect ledger"]
    NETTERMS --> GASSIDEFX
    GASTASK --> GASTARGETSAFE["GAS target/task/cue safety"]
    GASSIDEFX --> GASTARGETSAFE
    CPP --> ALG["Constraints, complexity, and selection"]
    CONT --> ALG
    ALG --> DSTRUCT["Sequences, hashes, heaps, and queues"]
    DSTRUCT --> SORT["Sorting, binary search, and sequence patterns"]
    DSTRUCT --> GRAPH["Traversal, shortest paths, DAGs, and DSU"]
    GRAPH --> NAV
    ALG --> SPATIAL["Spatial partitions, BVHs, and broadphase"]
    SORT --> VERIFY["Differential verification and benchmarks"]
    GRAPH --> VERIFY
    SPATIAL --> VERIFY
    ALG --> VECMATH["Vectors, projection, and orientation"]
    VECMATH --> XFORM["Spaces, transforms, matrices, and normals"]
    ACT --> XFORM
    XFORM --> ROTMATH["Rotators, quaternions, and interpolation"]
    VECMATH --> NUMMATH["Kinematics and numerical robustness"]
    VECMATH --> HITMATH["Closest points, intersections, and sweeps"]
    SPATIAL --> HITMATH
    NUMMATH --> HITMATH
    XFORM --> RENDERMATH["Rendering spaces, tangent basis, depth, and colour"]
    RPIPE --> RENDERMATH
    XFORM --> ANIMASSET["Animation assets, Skeletons, and retargeting"]
    ASSET --> ANIMASSET
    ANIMASSET --> ANIMRUN["Anim runtime, threads, and gameplay snapshot"]
    ACT --> ANIMRUN
    ANIMRUN --> ANIMPOSE["State machines, Blend Spaces, layers, and sync"]
    ROTMATH --> ANIMPOSE
    ANIMPOSE --> ANIMACTION["Montages, slots, sections, and notify contracts"]
    ANIMACTION --> ROOTMOTION["Root motion, CharacterMovement, and networking"]
    ANIMPOSE --> ANIMPROC["Linked layers, IK, Control Rig, and warping"]
    ANIMRUN --> ANIMPERF["Animation debugging, profiling, and budgeting"]
    ANIMPROC --> ANIMPERF
    ACT --> PHYSFILTER["Collision filtering, profiles, and geometry"]
    SPATIAL --> PHYSFILTER
    PHYSFILTER --> PHYSQUERY["Scene queries, Hit Results, and events"]
    HITMATH --> PHYSQUERY
    PHYSQUERY --> PHYSBODY["CharacterMovement, sweeps, and rigid bodies"]
    PHYSBODY --> PHYSSTEP["Timestep, contacts, and constraints"]
    PHYSSTEP --> RAGDOLL["Physical Materials, Physics Assets, and ragdoll"]
    ROOTMOTION --> RAGDOLL
    PHYSQUERY --> PHYNET["Networked physics, debugging, and scalability"]
    PHYSSTEP --> PHYNET
    RAGDOLL --> PHYNET
    GC --> UILIFE["UMG, Slate, and widget lifecycle"]
    DEL --> UILIFE
    SUB --> UILIFE
    UILIFE --> UIMODEL["UI projection, events, and MVVM"]
    COMM --> UIMODEL
    AUTH --> UIMODEL
    UILIFE --> UILAYOUT["Responsive layout, localisation, and accessibility"]
    UILAYOUT --> UIINPUT["Hit testing, focus, navigation, and CommonUI"]
    UIMODEL --> UILIST["Virtual lists, async assets, and world widgets"]
    ALOAD --> UILIST
    UILAYOUT --> UIPERF["Invalidation, rendering, and UI performance"]
    UILIST --> UIPERF
    INS --> UIPERF
    RCOST --> UIPERF
    PC --> EINACT["Enhanced Input actions and values"]
    BP --> EINACT
    EINACT --> EINCTX["Mapping contexts and LocalPlayer lifecycle"]
    SUB --> EINCTX
    EINACT --> EINTRIG["Modifiers and triggers"]
    EINCTX --> EINROUTE["Input routing: UI, GAS, networking"]
    EINTRIG --> EINROUTE
    UIINPUT --> EINROUTE
    GASABILITY -.-> EINROUTE
    AUTH --> EINROUTE
    SAVEDMOVE -.-> EINROUTE
    ACT --> AUDSRC["Audio sources, parameters, and Component lifetime"]
    GC --> AUDSRC
    AUDSRC --> AUDVOICE["Spatialisation, concurrency, and voice admission"]
    XFORM --> AUDVOICE
    AUDSRC --> AUDMIX["Classes, mixes, Submix routing, and DSP"]
    AUDSRC --> AUDSTREAM["Quartz timing, streaming, and memory"]
    AREF --> AUDSTREAM
    AUDVOICE --> AUDPROF["Networked audio, debugging, and profiling"]
    AUDMIX --> AUDPROF
    AUDSTREAM --> AUDPROF
    AUTH --> AUDPROF
    INS --> AUDPROF
    COMM --> FXSTACK["Niagara stack, execution states, and parameters"]
    XFORM --> FXSTACK
    FXSTACK --> FXDATA["CPU/GPU simulation and gameplay data flow"]
    RPIPE --> FXDATA
    FXSTACK --> FXLIFE["Component lifetime, pre-cull, and pooling"]
    GC --> FXLIFE
    ALOAD --> FXLIFE
    FXLIFE --> FXSCALE["Bounds, Effect Types, and significance"]
    SCALE --> FXSCALE
    FXDATA --> FXRENDER["Collision, renderers, materials, and overdraw"]
    RCOST --> FXRENDER
    PHYSQUERY --> FXRENDER
    FXSCALE --> FXPROF["Niagara networking, debugging, and profiling"]
    FXRENDER --> FXPROF
    AUTH --> FXPROF
    INS --> FXPROF
    COMM --> SCRIPTAPI["Scripting adoption and native API boundaries"]
    MODULE --> SCRIPTAPI
    AUTH --> SCRIPTAPI
    SCRIPTAPI --> LUACAPI["Lua language, coroutines, errors, and C API"]
    CPP --> LUACAPI
    LUACAPI --> LUAGC["Lua–UObject GC, lifetime, reload, and plugins"]
    GC --> LUAGC
    DEL --> LUAGC
    SCRIPTAPI --> CSINTEROP["C# managed lifetime, marshalling, callbacks, and Tasks"]
    GC --> CSINTEROP
    CSINTEROP --> CSHOST[".NET hosting, UnrealCLR, AOT, and packaging"]
    PLUGIN --> CSHOST
    COOK --> CSHOST
    LUAGC --> SCRIPTPROF["Reload, security, networking, debugging, and profiling"]
    CSHOST --> SCRIPTPROF
    AUTH --> SCRIPTPROF
    INS --> SCRIPTPROF
    MOVE --> CPPTMPL["Template deduction, forwarding, SFINAE, and concepts"]
    CPP --> CPPTMPL
    CPPTMPL --> CPPCALL["Lambda lifetime and callable type erasure"]
    OWN --> CPPCALL
    DEL --> CPPCALL
    CPP --> CPPLAYOUT["Object layout, cache locality, arenas, and pools"]
    ALG --> CPPLAYOUT
    CPPLAYOUT --> CPPSYNC["Data races, mutexes, condition variables, and atomics"]
    RAII --> CPPSYNC
    CPPSYNC --> CPPTASK["Task DAGs, parallel partitioning, cancellation, and merge"]
    CPPCALL --> CPPTASK
    INS --> CPPTASK
    BUILD --> CPPODR["ODR, linkage, static initialisation, and sanitizers"]
    MODULE --> CPPODR
    CPPSYNC --> CPPODR
    REF --> UESPEC["UE specifiers and metadata by consuming system"]
    BP --> UESPEC
    BUILD --> UESPEC
    UESPEC --> UEPERSIST["Config, SaveGame, transient, duplication, and serialisation flags"]
    AREF --> UEPERSIST
    UOBJ --> UEIDENT["Outer, package, name, path, flags, and object helper APIs"]
    PTR --> UEIDENT
    LIFE --> UEIDENT
    ALOAD --> UEIDENT
    REF --> UEIFACE["Reflected UINTERFACE versus native abstract interfaces"]
    BP --> UEIFACE
    COMM --> UEIFACE
    CPPODR --> UEDIAG["Logging, UE_LOGFMT, check, verify, and ensure"]
    UESPEC --> UEDIAG
    CPPCALL --> UEVOCAB["TOptional, TVariant, TFunctionRef, views, and lifetime"]
    CPP --> UEVOCAB
    CONT --> UEVOCAB
    BP --> UEASYNC["Async Blueprint proxies and cancellation lifetime"]
    GC --> UEASYNC
    DEL --> UEASYNC
    ALOAD --> UEASYNC
    CPPTASK --> UEASYNC
    GC --> UEGCBRIDGE["FGCObject, root-set awareness, and thread-safe UObject boundaries"]
    PTR --> UEGCBRIDGE
    CPPTASK --> UEGCBRIDGE
    CPPSYNC --> UEGCBRIDGE
    CPP --> CPPABI["C++ ABI, references, OOP, and allocation families"]
    CPPLAYOUT --> CPPABI
    CPPODR --> CPPABI
    CPPABI --> SYSMEM["OS virtual memory, processes, IPC, and shared memory"]
    CPPSYNC --> SYSMEM
    CPPTASK --> SYSMEM
    AUTH --> NETSYNC["TCP/UDP and state/frame/rollback sync models"]
    NETTERMS --> NETSYNC
    NETSYNC --> NETSEC["TLS, HTTPS, MITM, and replay resistance"]
    TX --> NETSEC
    NETSEC --> ANTICHEAT["Defensive anti-cheat and client trust boundaries"]
    REL --> ANTICHEAT
    SYSMEM -.-> ANTICHEAT
    ALG --> RBTREE["Ordered trees, RB vs AVL, and STL ordered containers"]
    CONT -.-> RBTREE
```
