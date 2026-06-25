# UE 面试备考学习指南

适用对象：有编程背景、正在系统准备 Unreal Engine 工程师面试的人。

目标：把 GamePrep 作为可直接执行的面试训练库，按优先级完成“读 → 复述 → 实操/推演 → 自测 → 补弱”的循环。

---

## 1. 知识库结构与用途

### 1.1 根目录文件

- [[README]]
  - 仓库总入口。
  - 用于回看整体结构、Start here、主要章节列表。

- STATUS
  - 当前知识库进度、覆盖范围、最近新增内容、下一步建议。
  - 用于判断当前学习重点、已覆盖内容和薄弱区。

- research
  - 研究范围和边界定义。
  - 用于检查知识库是否缺主题或范围是否偏移。

- AGENTS / command
  - 生成与维护流程说明。
  - 日常学习时无需重点阅读。

### 1.2 `curriculum/`：学习路径层

- 学习优先级和知识依赖关系
  - 定义 P0 / P1 / P2 / P3 优先级和 D1–D5 深度层级。
  - 见本文件「1.7 学习优先级和深度」小节。

- [[ue_study_schedule]]
  - daily loop、mock ladder、evidence packet、stop conditions。
  - 用于执行学习循环和 mock 训练。

- [[ue_gap_analysis]]
  - 列出真空区和薄弱区。
  - 用于按岗位方向补缺。

- [[ue_interview_synthesis_lists]]
  - 高频考点和答题抓手的压缩索引。
  - 用于冲刺阶段复习。

### 1.3 `topics/`：知识正文层

常见主题包括：

- [[cpp_for_unreal_interviews]]
- [[ue_cpp_idioms]]
- [[game_architecture_patterns]]
- [[game_programming_patterns]]
- [[ue_networking_and_replication]]
- [[ue_profiling_optimisation]]
- [[ue_rendering_graphics_performance]]
- [[ue_assets_loading_cooking]]
- [[ue_build_modules_plugins_tools]]
- [[ue_world_partition_large_world_pipeline]]
- [[ue_gameplay_ability_system]]
- [[ue_gas_specialist_depth]]
- [[ue_ai_custom_senses_massai]]
- [[ue_apple_console_profiling]]

学习目标：

1. 解释概念：这是什么、为什么存在。
2. 判断边界：什么时候用、什么时候不用。
3. 定位问题：出现 bug / 卡顿 / 不同步时先查什么。
4. 说明 trade-off：为什么选 A 不选 B。

### 1.4 `practice/`：练习层

- [[ue_flashcards]]
  - 高频记忆材料。
  - 适合碎片化复习。

- [[ue_interview_question_bank]]
  - 正式面试题集合。
  - 用于口头自测。

- [[cpp_interview_question_bank]]
  - C++ 基础与 Unreal 结合题。
  - 适合补 C++ / UE 基础。

- [[ue_hands_on_projects]]
  - 项目练习、调试实验、性能验证和证据包。
  - 用于形成可讲述的项目经历。

- workbook
  - 例如：
    - [[ue_world_partition_debugging_workbook]]
    - [[ue_build_farm_failure_workbook]]
    - [[ue_ai_senses_massai_workbook]]
    - [[ue_gas_specialist_workbook]]
    - [[ue_apple_console_profiling_workbook]]
    - [[ue_networking_target_branch_proof_workbook]]
    - [[ue_packaged_performance_build_worldpartition_workbook]]
    - [[ue_runtime_presentation_target_proof_workbook]]
  - 用于练习排查、验证和 debug 过程表达。

### 1.5 `sources/`：证据来源层

- [[ue_topic_source_index]]
  - 结论来源索引。
  - 用于追溯版本、官方原文和证据出处。

### 1.6 `data/`：机器可读层

- `data/ue_interview_knowledge_graph.json`
- [[ue_interview_knowledge_graph]]
- `data/ue_knowledge_graph.canvas`

用途：
- 人类学习主入口：[[README]] + `curriculum/` + `topics/` + `practice/`
- 机器和结构分析入口：`data/`

### 1.7 学习优先级和深度

#### 1.7.1 Priority markers

| Marker | Meaning |
|---|---|
| P0 | Essential in a broad UE interview |
| P1 | Very common; strong working knowledge expected |
| P2 | Common follow-up; medium depth expected |
| P3 | Role-dependent specialist topic |
| P4 | Advanced differentiation |

#### 1.7.2 Depth markers

| Depth | Meaning |
|---|---|
| D1–D2 | Recognise, explain, and use normally |
| D3 | Diagnose common failures |
| D4 | Design and defend trade-offs |
| D5 | Internals or advanced extensibility |

#### 1.7.3 Dependency-aware tracks

| Track | Default priority/depth | Core outcome |
|---|---|---|
| Standard C++ and UE C++ idioms | P0–P1 / D3–D4 | Reason correctly about value, lifetime, ownership, containers, APIs, and build boundaries. |
| UObject, reflection, lifetime | P0 / D4 | Select safe reference types and explain what reflection and GC actually do. |
| Gameplay framework and C++/Blueprint | P0–P1 / D3–D4 | Place responsibilities correctly across Actor, Component, Controller, state classes, subsystems, C++, and Blueprint. |
| Architecture, patterns, and system design | P0–P2 / D2–D4 | Design data-driven, testable, multiplayer-ready gameplay systems. |
| Networking | P0–P2 / D2–D4 | Reason from authority, ownership, relevance, and prediction; debug failed replication systematically. |
| AI and navigation | P1–P3 / D2–D4 | Build and debug decision-making, perception, pathfinding, avoidance, StateTree and Smart Object interactions. |
| Rendering and profiling | P1–P3 / D2–D4 | Identify the limiting thread/resource before changing content or code, then prove results on target platforms and automated device-lab gates. |
| Assets, build, editor, and tools | P1–P3 / D2–D4 | Make modular, cook-safe workflows and diagnose editor/package/procedural generation/world-building differences. |
| MassEntity/ECS and modern UE5 | P2–P3 / D1–D3 | Explain use cases and build a small data-oriented slice without forcing ECS everywhere. |
| Lua/C# integration | P3–P4 / D1–D3 | Explain plugin boundaries, interop, lifetime, debugging, and role relevance. |

#### 1.7.4 Role overlays

| Role | Highest-priority overlay | Breadth-only unless role demands it |
|---|---|---|
| Gameplay engineer | Object model, framework, architecture, C++/Blueprint, networking | RDG internals, advanced Mass |
| AI/gameplay engineer | Gameplay foundation, navigation, BT/StateTree, perception, profiling | shader internals, editor customisation |
| Rendering/graphics engineer | C++, maths, frame pipeline, GPU profiling, shaders/RDG | deep GAS, quest architecture |
| Networking engineer | Gameplay foundation, authority/ownership, replication, prediction, profiling | content authoring details |
| Engine/tools engineer | C++, object model, modules, assets/cooking, editor APIs | specialist animation authoring |
| Technical designer/artist | Blueprint architecture, data assets, UI/animation/VFX, profiling | C++ memory-model internals |
| Engine generalist | P0/P1 breadth plus debugging and profiling workflows | D5 depth chosen per job |

---

## 2. 推荐学习顺序

> 下面的时长都是 rough estimate，用 topics/ 与 practice/ 下 `.md` 文件的行数做粗估；它表示“学完一个主题”的总时长，默认包含：看正文 + 刷 flashcards + 口答 3~5 道 question + 记录薄弱点。实际耗时会明显受当天状态、已有基础、是否卡在某个概念或 debug 点影响。
>
> - **P0 总时长粗估**：约 **8.8 小时**（建议按 9~11 小时心里预期安排更稳）
> - **P0 → P3 全部列出主题都学一遍的粗略总时长**：约 **40.5 小时**（如果 P3 只选一条投递方向，实际会低于这个数）

### 2.1 P0：必问题基

顺序：

1. [[cpp_for_unreal_interviews]] — ~90 min
2. [[ue_cpp_idioms]] — ~90 min
3. [[game_architecture_patterns]] — ~90 min
4. [[game_programming_patterns]] — ~75 min
5. [[ue_networking_and_replication]] — ~90 min
6. [[ue_profiling_optimisation]] — ~90 min

覆盖重点：

- C++ 与 UE 对象模型
- framework / architecture / patterns
- networking
- profiling 与性能定位

每学完一个 topic，立即做：

- 去 [[ue_flashcards]] 刷对应卡片
- 去 [[ue_interview_question_bank]] 做对应主题口答
- 从 [[ue_hands_on_projects]] 里选一个最小可执行实验

### 2.2 P1：常见系统补全

建议顺序：

1. [[ue_ai_navigation]] — ~60 min
2. [[ue_assets_loading_cooking]] — ~60 min
3. [[ue_rendering_graphics_performance]] — ~90 min
4. [[ue_build_modules_plugins_tools]] — ~90 min
5. [[ue_animation_systems]] — ~90 min
6. [[ue_physics_collision]] — ~90 min
7. [[ue_ui_systems]] — ~75 min
8. [[ue_audio_systems]] — ~75 min
9. [[ue_niagara_vfx]] — ~90 min

目标：

- 能说明基本原理
- 能举出常见 bug 场景
- 能说出排查方法
- 能说明与 gameplay 主线的边界

### 2.3 P2：现代 UE 常见专项

建议顺序：

1. [[ue_gameplay_ability_system]] — ~75 min
2. [[ue_enhanced_input]] — ~90 min
3. [[ue_smart_objects_statetree]] — ~60 min
4. [[ue_pcg_procedural_content]] — ~60 min
5. [[ue_world_partition_large_world_pipeline]] — ~90 min
6. [[ue_platform_constraints]] — ~60 min
7. [[ue_device_lab_automation]] — ~90 min

目标：补齐现代 UE 常见专项，提升候选人区分度。

### 2.4 P3：按投递方向选专项

如果投 **gameplay / generalist**，优先：

- [[ue_gas_specialist_depth]] — ~90 min
- [[advanced_gameplay_patterns_specialist]] — ~90 min
- [[ue_networking_target_branch_proof]] — ~90 min
- [[ue_runtime_presentation_target_proof]] — ~75 min

如果投 **AI / gameplay AI**，优先：

- [[ue_ai_custom_senses_massai]] — ~90 min
- [[ue_massentity_ecs]] — ~75 min

如果投 **performance / tools / engine-adjacent**，优先：

- [[ue_apple_console_profiling]] — ~60 min
- [[ue_packaged_performance_build_worldpartition_proof]] — ~90 min
- [[ue_build_modules_plugins_tools]] — ~90 min
- [[ue_rendering_graphics_performance]] — ~90 min

### 2.5 推荐循环方式

#### 单主题循环

1. 看 topic 正文
2. 刷同主题 flashcards
3. 做 5~10 道 interview questions 口答
4. 做一个 workbook / hands-on 小实验
5. 记录答不顺的点到错题清单
6. 48 小时内回刷一次
7. 状态好时继续，状态差时最低保：flashcards + 1 道 question

---

## 3. flashcards 使用方法

### 3.1 推荐工具

- **Anki**：适合长期 spaced repetition
- **Obsidian + 手动标记**：适合轻量复习

### 3.2 使用节奏

#### 每次学习：按状态选择刷卡量

只做三类卡：

1. **P0 高频错卡**
2. **最近 7 天新学主题的卡**
3. **面试时容易说混的对比型卡**
   - `GameMode vs GameState`
   - `TWeakObjectPtr vs TSoftObjectPtr`
   - `Actor vs Component`
   - `RPC vs replicated property`
   - `Dormancy vs relevancy`

### 3.3 刷卡要求

- 先口头回答，再看答案
- 对每张卡补一句：如果面试官追问，我会怎么展开

示例：`TSharedPtr<UObject>` 为什么不对，至少应覆盖：

- UObject 生命周期属于另一套系统
- retained / observed / soft-loading 对应不同工具
- 项目中如何选择 `TObjectPtr` / `TWeakObjectPtr` / `TSoftObjectPtr`

### 3.4 复习策略

手动分 4 档：

- A：闭眼会，能展开
- B：知道概念，但讲不顺
- C：经常混
- D：完全薄弱

复习优先级：

**D > C > B > A**

---

## 4. interview questions 使用方法

[[ue_interview_question_bank]] 的用途是口语输出训练。

### 4.1 一次自测流程

每次拿 5~8 题。

每题流程：

1. 先脱稿回答 1~3 分钟
2. 录音，或至少计时
3. 对照题里的：
   - Short Answer
   - Strong 3-Year-Engineer Answer
   - Follow-up Questions
4. 标出缺失项：
   - 概念没讲准
   - trade-off 没讲到
   - debug / verification 讲不出来

### 4.2 自测检查项

- 有没有先给结论
- 有没有讲清楚边界
- 有没有举实际 bug / 验证方式
- 有没有把术语说对
- 有没有把 gameplay truth、presentation、network authority 等边界说清楚

### 4.3 技术答题版 STAR

- **S（Situation）**：场景 / 需求 / 问题
- **T（Task）**：核心任务
- **A（Action）**：定位、设计、验证过程
- **R（Result）**：性能 / 稳定性 / 协作结果
- **L（Learning）**：后续沉淀出的规则

适用表达模板：

> 当时遇到 X，表现是 Y。我的任务是先确认真相层和表现层边界，再判断是 authority / lifetime / loading / profiling 哪一类问题。然后做了 A、B、C，最后确认根因是 D。修复后结果是 E，后续沉淀成 F 规则。

### 4.4 推荐训练顺序

1. UObject / reflection / GC / pointers
2. Gameplay framework / architecture
3. Networking
4. Profiling / rendering
5. Assets / build / package
6. AI / navigation
7. GAS / modern systems

---

## 5. workbooks 和 hands-on projects 使用方法

### 5.1 基本要求

每做一个 workbook 或 hands-on，至少产出 4 项：

1. **一个最小实验 / 小 demo**
2. **一份结果记录**
3. **一个 bug / failure case**
4. **一段可复述的面试故事**

### 5.2 接入真实 UE 项目

建议流程：

1. 选一个当前学习主题
   - 例如 networking / GAS / World Partition
2. 去 `topics/` 看原理和边界
3. 去对应 `practice/` 找 hands-on 或 workbook
4. 在自己的项目里做最小落地版本
5. 记录：
   - 需求是什么
   - 如何设计
   - 踩了什么坑
   - 如何 debug
   - 如何验证通过
6. 整理成一页面试材料：
   - 现象
   - 根因
   - 方案
   - 证据
   - trade-off

### 5.3 暂时没有真实 UE 项目时

优先从 [[ue_hands_on_projects]] 里选：

1. **Project 7A**：UObject lifetime / reference graph
2. **Project 1**：Core Gameplay Sandbox
3. **Project 3**：Networked Mini Game
4. **Project 4**：Rendering and Optimisation Lab
5. **Project 7C**：Algorithms Verification Harness

选择原则：

- 覆盖高频面试主线
- 能产出可讲述的实践材料

### 5.4 workbook 使用方法

把 workbook 当成排障脚本。

例如：

- [[ue_build_farm_failure_workbook]]
- [[ue_networking_target_branch_proof_workbook]]
- [[ue_runtime_presentation_target_proof_workbook]]

执行顺序：

1. 先预测问题可能在哪
2. 按 workbook 跑检查
3. 写出定位顺序
4. 整理成可口述的 debug 过程

---

## 6. 学习节奏框架

目标：**2 个月内尽量完成 P0 → P1 → P2 → P3**。顺序不要乱，但推进速度要尽量快；状态好时主动多吃进度，状态差时至少不断线。

### 6.1 推进原则

- **级别优先**：把一个优先级层级推进到可口答、可举例、可排查，再进下一级。
- **闭环优先**：每次学习都要同时覆盖输入、记忆、输出，避免只读不练。
- **阻塞点优先**：如果某个薄弱点已经卡住后续 topics、mock 或 hands-on，优先修它，不要机械按目录往下翻。
- **能快就快**：一旦状态好、时间够、理解顺，直接连续推进多个 topic / question / 实操，不必等固定周期。

### 6.2 每次学习的最低操作单元

每次学习至少完成一个最小闭环：

1. **完成 1 个 topic 小节**，并能用自己的话复述核心概念、边界和典型 bug。
2. **刷对应 flashcards**，把刚学内容过一遍，至少确认哪些卡还不稳。
3. **口答 1 道对应 question**，强迫自己做输出。
4. **记录 1 个薄弱点**，例如讲不顺的概念、混淆项、不会展开的追问。

如果当天只能做最低量，也至少守住这个闭环，不要完全断掉。

### 6.3 状态好时的扩展方向

如果状态好，就在同一主题上继续扩展，优先级从高到低：

1. **再推进 1~2 个 topic 小节**，趁理解连续时拉进度。
2. **多答几道 question**，把同主题常见追问一起打掉。
3. **补一个 workbook / hands-on 小实验**，把知识转成可讲述证据。
4. **回刷前 48 小时或近一周的薄弱点**，防止新内容学完就掉。
5. **整理 1 个技术故事**，把问题、定位、方案、验证和 trade-off 讲顺。

### 6.4 状态差时的降档规则

状态差时不要硬拼时长，改为保连续性：

- 不开新大主题，只修当前主题的最小闭环。
- 优先做 **flashcards + 1 道 question + 1 个薄弱点修正**。
- 如果连完整闭环都吃力，至少回顾最近主题的关键结论，并把一个容易卡壳的问题重新答顺。

原则不是“今天学多久”，而是“今天有没有完成最小推进，明天能不能无缝接上”。

### 6.5 节奏检查点

用下面 3 个问题判断自己是否在正确节奏上：

- **推进够不够快**：P0 / P1 是否在持续清空，而不是长期停在同一块。
- **闭环够不够完整**：是不是每学一点都做了 flashcards 和口答，而不是只看不输出。
- **薄弱点有没有被处理**：错题、卡壳点、讲不顺的题，是否在后续 session 里被回收。

如果答案是否定的，就减少“看更多新内容”，优先恢复闭环和补弱。
---

## 7. 什么时候可以开始投递

### 7.1 基础准备完成线

满足以下条件即可开始投递和面试：

1. **P0 / P1 常见题能稳定答出来**
2. **能连续做 30~45 分钟技术 mock**
3. **能讲清 3~5 个技术故事**
   - 一个架构设计
   - 一个网络 / 状态同步问题
   - 一个性能 / profiling 问题
   - 一个生命周期 / 指针 / UObject 问题
   - 一个项目权衡 / debug 过程
4. **能做基本的 debug reasoning**
   - 先分类，再给定位顺序
5. **至少有 1~2 个可展示的 UE 小项目或实验**
   - 要有明确问题、实现、验证、结果

### 7.2 更强的竞争力标准

- 能把 `Project 1 / 3 / 4 / 7A` 讲成熟
- 能说清 Gameplay truth、network authority、presentation、profiling evidence 的边界
- 能对 GAS / StateTree / World Partition / BuildGraph 给出使用判断
- 能在没有标准答案的题里给出结构化分析

### 7.3 投递策略

- 先把 P0 / P1 做稳
- 准备 1~2 个角色方向专项
- 先投一批，用真实面试校准差距
- 再回头补短板

---

## 8. 什么时候继续扩充知识库

原则：只有在存在明确知识缺口、投递方向变化、或真实面试暴露新弱点时再扩。

### 8.1 值得扩充的情况

#### 1）开始投更明确的岗位方向

例如：

- gameplay engineer
- AI gameplay
- tools / pipeline
- rendering / performance

处理方式：按岗位方向做专项加深，不做全领域同时扩张。

#### 2）真实面试暴露重复弱点

例如连续被追问：

- CharacterMovement prediction
- build / package / cook failures
- GAS prediction / cues / attribute model
- profiling evidence and benchmark design

处理方式：定向补一轮。

#### 3）有真实 UE 项目在做

优先扩充：

- 具体实现 proof
- 常见 failure matrix
- debug checklist
- 项目复盘模板

#### 4）版本目标变化

目标岗位明确要求某个 UE 版本、平台或插件生态时，做定向更新。

### 8.2 不建议继续扩充的情况

出现以下情况时，先吸收再扩：

- 看得多，说得少
- mock 一做就卡
- workbook 基本没跑过
- 没有形成可讲的项目故事
- 同类基础题反复答不稳

### 8.3 维护节奏

- **错题或 mock 记录开始堆积时**：回顾并合并同类弱点。
- **投递方向变化或重复弱点出现时**：决定要不要做一次定向补库。
- **每次真实面试后**：立即记录新暴露的弱点，必要时再补内容。

---

## 9. 从今天开始的执行建议

### 起步顺序

按下面顺序推进，不绑定固定周数；状态好时连续推进，状态差时只保最小闭环。

#### A. C++ / UObject 基础

- 读：
  - [[cpp_for_unreal_interviews]]
  - [[ue_cpp_idioms]]
- 刷：对应 flashcards
- 练：UObject / pointer / GC / reflection 相关 interview questions
- 做：`Project 7A` 相关最小实验

#### B. Gameplay framework / architecture

- 读：
  - [[game_architecture_patterns]]
  - [[game_programming_patterns]]
- 练：framework / architecture / Actor vs Component / GameMode vs GameState 相关 question
- 做：`Project 1` 最小版设计和职责划分

#### C. Networking

- 读：
  - [[ue_networking_and_replication]]
- 练：authority / ownership / RPC / OnRep / dormancy / relevancy
- 做：`Project 3` 最小网络验证

#### D. Profiling / rendering

- 读：
  - [[ue_profiling_optimisation]]
  - [[ue_rendering_graphics_performance]]
- 练：性能定位题
- 做：`Project 4` 最小实验

完成这组 P0 主线后，再逐步补齐 P1 主题。

---

## 10. 当前阶段的重点

当前重点：

- 把 P0 / P1 主题讲顺
- 把 1~2 个 hands-on 做成自己的故事
- 把 flashcards 和 question bank 整合进复习流程
- 用少量但持续的节奏，把知识变成可输出内容

目标状态：

> 我知道核心系统怎么分层；我能讲清常见问题怎么定位；我做过几个最小但真实的实验；我知道哪些地方要看证据，而非主观臆断。

---

## 11. 执行口令

下次学习按这个顺序：

1. 打开 STATUS 看当前重点
2. 打开一个主题学一节
3. 刷对应 [[ue_flashcards]]
4. 口答 3~5 题 [[ue_interview_question_bank]]
5. 做一个最小 workbook / hands-on 动作
6. 记下今天最卡的 1 个点
7. 48 小时内回刷
