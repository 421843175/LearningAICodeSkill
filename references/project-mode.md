---
name: system-call-chain-analysis
description: Analyze an unfamiliar software system or module in two gated passes: first produce L1-L2 macro topology plus L3 domain modules together, then after confirmation produce L4 business-scenario-driven call chains. Includes an optional Inspection Mode for architecture-interview-style questions and answers. Produces a Markdown handoff document under doc/project/PyyyyMMdd(<project-or-module>分析).md. Use when the user asks to understand a project, backend, architecture, modules, service interactions, call chains, deployment flow, wants a reusable way to ask AI to梳理系统, or says 进入考察模式, 退出考察. For ambiguous 你考我一下 / 你问我一下 triggers, obey the main SKILL.md Ambiguous Trigger Arbitration Rule first.
---

# System Call Chain Analysis

## Goal

Analyze a codebase or subsystem in three stages with one gate: stages 1 and 2 are produced together, then stage 3 waits for confirmation. Do not dump scenario-level details before the gate. Produce a handoff-quality Markdown document that grows stage by stage and answers:

- Who uses this system?
- What problem does the system solve?
- What are the macro service, process, DB, MQ, gateway, and infrastructure relationships?
- What are the module/package responsibilities?
- How do core DTO/VO/Entity objects move between modules?
- What key storage and state nodes does the system depend on, and which modules read/write them?
- In key scenarios, which class/function/API calls which next component?
- In key scenarios, what does each step do and what important state/data changes happen?
- What is already implemented versus only planned or missing?
- For every scenario, how the implementation fits the project description or product goal, what design choices were made, what highlights are worth learning, what possible shortcomings exist, and what improvement direction can address them.

## Stage Gate Rule

Hard requirement: work in exactly these three stages, but stages 1 and 2 run continuously in the first pass. Pause only after stage 2.

- First pass output must include both stage 1 and stage 2 in the same document: stage 1 covers L1-L2 macro topology, then stage 2 covers L3 domain modules.
- Do not pause between stage 1 and stage 2.
- After writing stage 2, stop and ask the user to reply `继续` before doing stage 3.
- If the user interrupts the stage 2/3 pause with a question or request other than proceeding, answer that interruption normally within the allowed stage-2 granularity, but end the response with an explicit reminder that stage 3 is still pending and the user can reply `继续` to trigger it.
- Stage 3 output covers L4 business-scenario-driven understanding. Only stage 3 may output `Class.method()`-level chains and scenario design analysis.
- If stage 2 has not been completed and confirmed, do not output stage 3 details, even if the code has already been inspected.
- If explain mode or any reader-friendly writing style is fused into project mode, this gate remains absolute. Narrative clarity may change wording only; it cannot justify early scenario maps, scenario design analysis, or method-level call-chain leakage before the `继续` checkpoint.

### 考察模式（Inspection Mode）- 可选分支机制

Enter Inspection Mode whenever the user says `学习SKILL考察模式`, `考察模式`, `项目考察模式`, `进入考察模式`, or an equivalent explicit request to be quizzed/interviewed about the analyzed system. For ambiguous `你考我一下` / `你问我一下` triggers, first obey the main `SKILL.md` `Ambiguous Trigger Arbitration Rule`; enter project Inspection Mode only when that arbitration routes to `doc/project/` or default project-inspection scope. After Stage 3 is fully completed, proactively ask whether the user wants to enter Inspection Mode.

On trigger:

1. Read and strictly follow `references/rules/inspection-mode.md`.
2. Inspection questions must target the whole project's core logic and core principles, not isolated local code trivia.
3. First response must output only question 1, then stop and wait.
4. Do not simulate user answers, scores, templates, reference answers, or later questions in the same turn.
5. During the active Inspection Mode round, do not write any file. After the user exits the round or explicitly asks to persist after exit, append only new Inspection Mode records under `# 考察模式`; if a question score is below 8, persist only the question and reference answer, not the user answer, score, or strengths/weaknesses evaluation. Do not rewrite `阶段一` through `阶段三`.
6. When the user exits Inspection Mode, provide the 100-point overall review described in `references/rules/inspection-mode.md`; do not write that overall review to the document.

## Document Path

Always create or update one Markdown document in:

```text
doc/project/PyyyyMMdd(<项目或模块名>分析).md
```

Rules:

- `yyyyMMdd` is the current local date.
- `<项目或模块名>` should come from the repo name, README, package name, main module, or the user's requested scope.
- Example: `doc/project/P20260610(订单模块分析).md`.
- Create `doc/project/` if it does not exist.
- If the project/module name contains filesystem-invalid characters, replace them with short safe Chinese or ASCII words.
- Each stage appends or updates the same document; do not create separate files for each stage.

## Resource Loading

Load these bundled resources only when needed:

- Inspection Mode rules: read and strictly follow `references/rules/inspection-mode.md` whenever Inspection Mode is triggered.
- Compact stage 3 call-chain template: read `references/templates/call-chain-template.md` when a short method-node or scenario-format example is needed.
- No full legacy RTK example is bundled. If the user asks for the old RTK example, say it is not available in this skill because outdated examples were intentionally removed.

## Workflow

Before stage 1, inspect enough to identify the project scope, document path, macro topology, and L3 domain/module structure: README, build files, package manifests, top-level directories, configs, entrypoints, deployment files, important source packages, core data objects, and storage/state nodes. Then write stages 1 and 2 in one pass.

## Diagram/Text-Graph Stability Rule

For stage 1 and stage 2 diagrams, prefer fenced `text` code blocks for topology and data/object-flow graphs. If Mermaid is used instead, strictly adhere to the Mermaid Diagram Stability Rules from the main `SKILL.md`: avoid complex syntax, quote text nodes safely, avoid unsupported arrows or nested constructs, and fall back to a fenced `text` diagram whenever the diagram is large, contains Chinese punctuation/parentheses, or may render unstably.

### Stage 1: 宏观拓扑（L1-L2）

Goal: explain the system from the outside and at infrastructure/service level.

Do:

1. 梳理系统一句话总结与核心业务边界：
   - 谁使用这个系统。
   - 解决什么业务/系统问题。
   - 当前能力边界和不做什么。
2. 分析技术栈与基础设施拓扑：
   - 网关、前端、后端服务进程、定时任务、消费者、DB、MQ、缓存、文件/对象存储、外部系统之间的关系。
   - 入口端口、主要配置、启动方式、部署触点。
   - 用文本图表达 L1-L2 拓扑。默认使用 fenced `text` 代码块；如改用 Mermaid，必须严格遵守主 `SKILL.md` 的 Mermaid Diagram Stability Rules，避免复杂语法并确保中文节点安全加引号。
3. 梳理项目运行与启动：
   - 启动入口、启动命令、运行模式、本地/容器/部署启动差异。
   - 必需依赖服务，如 DB、MQ、Redis、对象存储、外部 API、硬件设备或模拟器。
   - 关键配置文件、环境变量、端口、健康检查或启动后验证方式。
   - 如果代码或 README 没有明确启动方式，标记为未确认，不要编造。

Do not:

- Do not output module/package responsibility tables.
- Do not output DTO/VO/Entity flow.
- Do not output `Class.method()` call chains.

After stage 1:

- Write or update the document through the stage 1 sections.
- Continue directly to stage 2 in the same pass.

### Stage 2: 领域模块（L3）

Run immediately after stage 1 in the same first pass. Do not wait for user confirmation between stage 1 and stage 2.

Goal: explain the internal domain/module structure without dropping into method-level scenario chains.

Do:

1. 输出模块拆分与类/文件拆分两层结构，不要混在一张表里：
   - `2.1.1 模块`: first list module/service/app-level responsibilities, boundaries, upstream/downstream dependencies, and storage/external resources touched. For backend systems, treat each microservice or independently deployable backend application as the primary module unit; packages inside a service should usually be listed later in `2.1.2` unless the project is a single-service monolith. Keep this at module granularity; do not stuff detailed classes/files into this table.
   - `2.1.2 具体模块下的类/文件`: then list important classes/files under each module, including each class/file's role, key entrypoints or functions, upstream caller, downstream dependency, and related data/state objects.
   - If the project is small and has no formal modules, still separate by logical directory/package in `2.1.1`, then list concrete files/classes in `2.1.2`.
2. 梳理核心数据对象流转：
   - 本节写代码里的数据结构，例如 DTO/VO/Entity/message/event/config/request/response 对象。
   - 即使某个 `Entity` 映射数据库表，也先把 `OrderEntity` 这类代码对象放在本节；真实表名如 `t_order` 放到下一节的状态节点清单。
   - 对每个对象说明：它代表什么、由哪个模块创建、被哪个模块读取/转换/持久化/发布/返回，以及它携带的关键字段或状态值。
   - 用文本图表达对象在模块间的流转。默认使用 fenced `text` 代码块；如改用 Mermaid，必须严格遵守主 `SKILL.md` 的 Mermaid Diagram Stability Rules，避免复杂语法并确保中文节点安全加引号。
3. 输出核心存储与状态节点清单：
   - 本节写系统状态承载点，不写 Java/TypeScript/Python 类名。判断标准：这个东西是否在单次本地函数调用之外保存、持有、控制或传递状态，并且会被一个或多个模块读写。
   - 包括持久化存储和运行时状态介质，例如 MySQL/PostgreSQL 表、Redis Key、MQ 队列/Topic、文件/对象存储路径、硬件寄存器、前端全局状态 Store、内存 Map、Session、Socket、锁、定时器或定时任务状态。
   - 对每个节点说明：节点类型、精确名称或匹配 Pattern、核心作用、状态定义/生命周期含义、主责读写模块。
   - 使用中粒度节点，不要展开成方法级场景步骤。例如写 `t_order`、`lock:order:{orderId}`、`TOPIC_ORDER_PAY_NOTIFY`、`runningConnections`、`useAuthStore.hasLogined`，不要写 `OrderService.createOrder()`。
   - 不要在两个小节重复同一个东西：代码对象放 `核心数据对象总览`，存储/状态承载点放 `核心存储与状态节点清单`。

Do not:

- Do not output complete scenario call chains.
- Do not output `Class.method()`-level step-by-step flows except as short examples inside the module table when necessary.
- Do not select business scenarios, output `3.0 场景地图`, or apply scenario coverage rules during stage 2. Scenario selection and business-scenario-driven analysis are stage 3 only.

After stage 2:

- Update the same document through the stage 2 sections.
- Ask the user to reply `继续` before stage 3.
- Stop.

### Stage 3: 业务场景驱动理解（L4）

Only run after the user confirms stage 2 by replying `继续` or clearly asking to proceed.

Goal: use a carefully selected set of business scenarios to help the reader understand the whole project: business purpose, system boundary, state nodes, module collaboration, method-level implementation, and design tradeoffs.

Before writing individual scenarios, output `3.0 场景地图与覆盖说明`. Use it to prove why these scenarios were selected and how they cover the project.

Scenario selection iron law:

- Do not choose low-density single-table CRUD scenarios or pure pass-through/forwarding interfaces.
- Each selected scenario must satisfy the boundary-closure formula: it has a clear originating action and a clear physical terminal state.
- Originating actions include user operation, received network packet, device callback, MQ message, timer trigger, scheduler execution, application startup, external webhook, or admin command.
- Physical terminal states include database persistence, cache/session mutation, in-memory state mutation, MQ publication/consumption result, file/object-storage output, socket/session lifecycle change, hardware register bit change, frontend global state change, or an observable status shown to users. For embedded/IoT systems, terminal states also include hardware peripheral state changes, PWM duty-cycle adjustments, interrupt triggering/clearing, and raw control bytes sent to devices. For client/mobile/frontend systems, terminal states also include global state-machine mutations such as BLoC/Provider/Vuex/Redux/Zustand state changes and raw byte streams sent to peripherals.
- Scenario titles must be business actions or business closed loops, not raw endpoint names or method names.

The selected scenarios should cover the project comprehensively. Prefer a small set of high-value scenarios that jointly cover:

- 用户/管理员入口
- 核心业务对象生命周期：创建、推进、完成、取消、失败或恢复
- 数据进入系统：用户输入、外部 API、设备报文、MQ 消息、文件导入、第三方回调
- 状态推进：业务状态、运行态、连接态、任务态、前端全局状态如何变化
- 查询与反馈：用户/API/控制台如何看到系统价值和当前结果
- 异步/后台链路：MQ 消费、定时任务、重试、补偿、批处理、后台同步
- 管理与配置：管理员如何改变系统运行规则、连接参数、策略或开关
- 异常/失败/关闭/恢复路径
- 外部系统或设备接入

For `3.0 场景地图与覆盖说明`:

- Include each chosen scenario, business goal, triggering role/source, covered modules, related core objects from stage 2, related storage/state nodes from stage 2, and why the scenario is selected.
- If an obvious core module or state node from stage 2 is not covered by any scenario, mention it in the coverage notes and either add a scenario or explain why it is outside the current analysis scope.

For each selected scenario:

- Start from the actor and trigger.
- Use the exact Markdown scenario skeleton from Output Format. Do not omit numbered headings such as `#### 3.1.1 什么时候发生` or `#### 3.1.2 调用链文本图`.
- Add `场景解释` immediately under the scenario heading. It must include these three elements:
  - `触发源与角色`: who or what event triggers the scenario, and in what context.
  - `核心技术挑战`: why this scenario is worth analyzing, such as distributed transactions, high-concurrency contention, idempotency, hardware async callbacks, state-machine transitions, protocol parsing, data consistency, permission boundaries, retry/compensation, or observability.
  - `涉及的状态节点`: which `核心存储与状态节点` from stage 2 this scenario reads or writes.
- Use `Class.method()`-level text call chains.
- Show every important API/class/function hop.
- Include request path, message topic, table, file, queue, cache, socket, or external system when relevant.
- Under `#### 3.1.2 调用链文本图`, the call chain must be a fenced `text` code block. Do not use a Markdown indented block, bullet list, or plain paragraph for the call chain.
- The call chain must preserve visible arrows using `|`, `v`, and `->` or equivalent text-flow arrows. Do not output a flat `A -> B -> C` paragraph and do not drop arrows between nodes.
- Annotate each function/method node with square brackets `【】` for the method's role in the chain, not internal details.
- If the node maps to a local project file, add a separate source-location line between the method node and the parentheses: `// module>file>methodName`.
- After the optional source-location line, add parentheses `()` explaining internal handling, data transformation, state changes, and next destination.
- Prefer this node style inside the fenced text block: `Class.method(args) 【method role】`, then `// module>file>methodName`, then `(internal handling; data transformation; next destination)`.
- Explain what happens, why it happens, how it is implemented, what data object is transformed, and what state is stored in memory, DB, file, queue, cache, or socket.
- Add three separate scenario design analysis sections after the files/key points section:
  - `结合这个场景说目前这个设计是怎样的`: explain how the current design completes the business closed loop, how it fits the project description, README, product goal, or business context, and how it uses the stage 2 core objects/state nodes.
  - `结合这个场景说目前这个设计的技术亮点`: explain the technical highlights in expanded prose. Do not only list short bullet names.
  - `结合这个场景说目前这个设计潜在的不足之处`: explain possible risks, boundaries, gaps, or tradeoffs in expanded prose, plus practical improvement directions. Do not only list short bullet names.
- Hard requirement: every scenario must explicitly include both technical highlights and potential shortcomings. Do not omit either one. If the code is too small to expose an obvious weakness, still write a cautious tradeoff such as observability, validation, error handling, coupling, scalability, maintainability, test coverage, or operational risk.
- Expansion requirement for `技术亮点`: write at least two substantive points. For each point, explain the design choice, why it fits this scenario, what problem it solves, and what benefit/tradeoff it creates for the project.
- Expansion requirement for `潜在的不足之处`: write at least two substantive points. For each point, explain the risk or limitation, when it may be triggered, what impact it has, and a concrete improvement direction. Potential shortcomings must focus on architectural or engineering tradeoffs such as concurrency safety, single points of failure, protocol robustness, memory leak risk, state inconsistency, hardware blocking, lifecycle leaks, backpressure, retry/idempotency, or operational observability. Do not pad this section with generic complaints such as insufficient comments or missing unit tests unless they directly affect the scenario's architecture-level risk. If the issue is inferred rather than code-confirmed, mark it as inference.

Always separate facts from inference:

- Mark code-confirmed behavior as current implementation.
- Mark guessed intent or future work as inference/recommendation.
- Do not overclaim features that are not present in code.

## Output Format

Use this Markdown structure in the single document at `doc/project/PyyyyMMdd(<项目或模块名>分析).md`:

Important:

- In the first pass, include the title metadata, `阶段一`, and `阶段二` sections together. Do not add an empty `阶段三` heading.
- Do not stop after stage 1; continue directly into stage 2.
- When writing stage 3, append or update `阶段三` sections.

```markdown
# <系统名或模块名>分阶段系统分析文档

> 文档路径：doc/project/PyyyyMMdd(<项目或模块名>分析).md
> 分析方式：阶段一 L1-L2 宏观拓扑 -> 阶段二 L3 领域模块 -> 阶段三 L4 业务场景驱动理解

## 阶段一：宏观拓扑（L1-L2）
### 1.1 系统一句话
### 1.2 核心业务边界
### 1.3 技术栈与基础设施
### 1.4 L1-L2 总体拓扑文本图
使用 fenced `text` 代码块；如使用 Mermaid，必须严格遵守主 `SKILL.md` 的 Mermaid Diagram Stability Rules。
### 1.5 当前阶段结论与待确认点
### 1.6 项目运行与启动

## 阶段二：领域模块（L3）
### 2.1 模块/包结构职责拆分
#### 2.1.1 模块
| 模块/微服务/应用 | 模块职责 | 业务边界 | 上游调用方 | 下游依赖 | 存储/外部资源 |
| --- | --- | --- | --- | --- | --- |
#### 2.1.2 具体模块下的类/文件
| 所属模块 | 类/文件 | 类型/层次 | 核心职责 | 关键入口/函数 | 上游调用方 | 下游依赖 | 关联数据对象/状态节点 |
| --- | --- | --- | --- | --- | --- | --- | --- |
### 2.2 核心数据对象总览（代码里的数据结构）
本节只写代码里的数据结构，例如 DTO/VO/Entity/Message/Event/Config/Request/Response。说明对象含义、创建方、主要字段/状态、流转去向。注意：`OrderEntity` 这类 Entity 类属于本节；它映射的真实表 `t_order` 属于下一节。不要把 MySQL 表、Redis Key、MQ Topic、文件路径、全局 Store、内存 Map 等状态承载点写到这里。
### 2.3 核心存储与状态节点清单（系统状态承载点）
本节只写系统状态承载点：凡是跨函数调用、跨请求、跨线程、跨进程或跨前后端保存/传递/控制状态的介质，都放在这里。不要写普通 DTO/VO/Entity 类名。
| 节点类型 | 标识/名称/Pattern | 核心作用与状态定义 | 主责/读写模块 |
| --- | --- | --- | --- |
| **持久化表** | `t_order` / `t_order_item` | 存储订单主档及明细，承载订单生命周期的终态 | `order-module` (读写) |
| **缓存/内存Key** | `lock:order:{orderId}` | 保证分布式下单排他性的分布式锁，TTL 5秒 | `order-module` (写/删) |
| **消息队列** | `TOPIC_ORDER_PAY_NOTIFY` | 接收支付成功异步通知，用于驱动订单状态机向后流转 | `pay` (写) -> `order` (读) |
| **硬件寄存器** | `REG_ALARM_CTRL (0x40001004)` | 控制警报器开关。Bit[0]: 蜂鸣器; Bit[1]: LED闪烁 | `driver-layer` (读写) |
| **前端状态** | `useAuthStore.hasLogined` | 全局响应式布尔值，控制全局路由守卫与鉴权 UI 显示 | `auth-views` (读) |
| **运行时状态** | `runningConnections` | 进程内运行连接表，记录设备连接对象和当前运行态；服务重启后会丢失 | `gateway-ingest` (读写) |
### 2.4 DTO/VO/Entity/Message 流转文本图
使用 fenced `text` 代码块；如使用 Mermaid，必须严格遵守主 `SKILL.md` 的 Mermaid Diagram Stability Rules。
### 2.5 当前阶段结论与待确认点

## 阶段三：业务场景驱动理解（L4）
### 3.0 场景地图与覆盖说明
| 场景 | 业务目标 | 触发源/角色 | 覆盖模块 | 关联核心对象 | 关联状态节点 | 为什么选它 |
| --- | --- | --- | --- | --- | --- | --- |
覆盖说明：说明这些场景如何共同覆盖项目的入口、核心对象生命周期、数据进入、状态推进、查询反馈、异步/后台链路、管理配置、异常/失败/恢复路径；如果阶段二的核心模块或状态节点没有被覆盖，要说明原因。
### 3.1 场景一：<场景名>
场景解释：
- 触发源与角色：说明谁/什么事件在什么上下文下触发。
- 核心技术挑战：说明该场景为什么值得分析，例如分布式事务、高并发竞争、幂等、硬件异步回调、状态机流转、协议解析、数据一致性、权限边界、重试/补偿或可观测性。
- 涉及的状态节点：列出本场景会读写阶段二盘点出的哪些【核心存储与状态节点】。
#### 3.1.1 什么时候发生
说明触发时机、前置条件、完成后的业务结果或物理终态。
#### 3.1.2 调用链文本图
```text
触发源/角色
  |
  | 触发动作（上下文/请求/报文/消息/定时器）
  v
Class.method() 【方法在业务链路中的作用】
  // module>file>methodName
  (内部处理、对象转换、状态读写、下一跳)
  |
  | 关键数据/状态传递
  v
NextClass.nextMethod() 【下一步作用】
  // module>file>methodName
  (继续处理；直到明确物理终态)
```
#### 3.1.3 涉及文件/关键点
#### 3.1.4 结合这个场景说目前这个设计是怎样的
说明当前设计如何完成这个业务闭环、如何服务项目目标/业务定位/README 里的系统说明，以及它如何使用阶段二盘点出的核心对象和状态节点。
#### 3.1.5 结合这个场景说目前这个设计的技术亮点
必须展开写，不能只列短 bullet。至少写两个实质性亮点；每个亮点都说明：设计点是什么、为什么适合这个场景、解决了什么问题、给项目带来什么收益或取舍。
#### 3.1.6 结合这个场景说目前这个设计潜在的不足之处
必须展开写，不能只列短 bullet。至少写两个实质性不足；每个不足都说明：风险/限制是什么、什么情况下会触发、会造成什么影响、可以怎么具体改进。若属于推测或建议，必须标明是推测/建议。
### 3.2 场景二：<场景名>

<!-- 如果用户触发了考察模式，活跃考察轮次内保持只读；退出后或用户明确要求持久化后，才按 `references/rules/inspection-mode.md` 的 Document Append Format 在文档最末尾增量追加。否则不生成。 -->
```

## Stage 3 Template Resource

For compact method-node and scenario examples, read `references/templates/call-chain-template.md` only when needed. Do not use method-level call-chain formatting during stage 1 or stage 2.

## Prompt Template For Users

Copy this when asking an AI to analyze a project:

```text
请使用“system-call-chain-analysis”方式分析这个项目。

总体要求：
1. 分三个阶段分析该项目，但阶段一和阶段二必须连续输出；阶段二完成并获得我确认前，不要输出阶段三细节。
2. 输出文档统一放在 `doc/project/` 下，命名为 `PyyyyMMdd(<项目或模块名>分析).md`，例如 `doc/project/P20260610(订单模块分析).md`。
3. 三个阶段都写入同一份 Markdown 文档，不要每个阶段新建一个文件。
4. 区分“代码已经实现的能力”和“推测/后续可能要做的能力”。

【阶段一：宏观拓扑（L1-L2）】
1. 梳理系统一句话总结与核心业务边界（谁用，解决什么问题，当前能力边界）。
2. 分析技术栈与基础设施拓扑（网关、服务进程、DB、MQ、缓存、文件/对象存储、外部系统关系）。
3. 输出 `1.6 项目运行与启动`：说明启动入口、启动命令、运行模式、本地/容器/部署启动差异、依赖服务、关键配置、端口、健康检查或启动后验证方式；不明确的地方标记为未确认。
4. 只输出宏观拓扑和运行启动信息，不要输出模块职责表、数据对象流转或 Class.method() 调用链。
5. 写入阶段一后不要暂停，直接继续执行阶段二。

【阶段二：领域模块（L3）】
1. 输出 `2.1 模块/包结构职责拆分`，并拆成两张表：`2.1.1 模块` 只写模块/微服务/应用层职责、边界、上下游依赖和存储/外部资源；对于后端项目，`2.1.1 模块` 优先指每个微服务或可独立启动/部署的后端应用，不要把微服务内部普通 package 当成这一层的主粒度；`2.1.2 具体模块下的类/文件` 再写每个模块下的重要类/文件、类型/层次、核心职责、关键入口/函数、上下游和关联数据对象/状态节点。不要把模块级职责和具体类/文件混在一张表里。
2. 梳理核心数据对象（DTO/VO/Entity/Message/Event/Config/Request/Response）在模块间的流转关系，并放在 `核心数据对象总览（代码里的数据结构）`。这里写代码对象，不写表、Key、Topic、文件路径或全局状态。
3. 输出 `核心存储与状态节点清单（系统状态承载点）`：列出系统依赖的关键状态介质。判断标准是：它是否跨函数调用、跨请求、跨线程、跨进程或跨前后端保存/传递/控制状态，并且会被一个或多个模块读写。例子包括 MySQL 表、Redis Key、MQ 队列/Topic、文件/对象存储路径、硬件寄存器、前端全局 State、内存 Map、Session、Socket 状态、锁、定时任务状态等。
4. `核心存储与状态节点清单` 必须明确节点类型、标识/名称/Pattern、核心作用与状态定义、主责/读写模块；不要写成方法步骤，也不要和 `核心数据对象总览` 重复。
5. 只输出领域模块、代码数据结构、系统状态承载点和数据对象流转层面，不要输出重点业务场景的精细调用链。
6. 写入阶段二后暂停，等待我回复“继续”再执行阶段三。

【阶段三：业务场景驱动理解（L4）】
1. 这是阶段三的工作。只有在阶段二完成并获得确认后，才允许选取业务场景、输出 `3.0 场景地图与覆盖说明`、展开 `Class.method()` 级别调用链。
2. 先输出 `3.0 场景地图与覆盖说明`：用一张表说明所选场景、业务目标、触发源/角色、覆盖模块、关联核心对象、关联状态节点、为什么选它。场景组合要尽量覆盖项目的入口、核心对象生命周期、数据进入、状态推进、查询反馈、异步/后台链路、管理配置、异常/失败/恢复路径。
3. 场景选取铁律：禁止选择无技术密度的单表简单 CRUD 或纯转发接口。所选场景必须满足“边界闭环公式”：有明确的始发动作（如用户操作、收到网络报文、设备回调、MQ 消息、定时器触发、应用启动、外部 webhook、管理命令），以及明确的物理终态（如数据库落库、缓存/Session 改变、内存状态突变、MQ 发布/消费结果、文件/对象存储产物、Socket/连接生命周期变化、硬件寄存器变位、前端全局状态变化、用户可见状态变化）。对于嵌入式/IoT 系统，物理终态还包括硬件外设状态改变、PWM 占空比调整、中断触发/清除、向设备发送的原始控制字节流；对于客户端/移动端/前端，物理终态还包括 BLoC/Provider/Vuex/Redux/Zustand 等全局状态机突变，以及向蓝牙、串口、USB、传感器或其他外设发送的原始字节流。
4. 每个场景标题必须是业务动作或业务闭环，不要用裸接口名或方法名当标题。
5. 每个场景标题下面先写“场景解释”，且必须严格包含：
   - `触发源与角色`：谁/什么事件在什么上下文下触发。
   - `核心技术挑战`：该场景为什么值得分析，例如分布式事务、高并发竞争、幂等、硬件异步回调、状态机流转、协议解析、数据一致性、权限边界、重试/补偿或可观测性。
   - `涉及的状态节点`：本场景会读写阶段二盘点出的哪些【核心存储与状态节点】。
6. 每个场景必须严格保留编号小标题：`#### 3.x.1 什么时候发生`、`#### 3.x.2 调用链文本图`、`#### 3.x.3 涉及文件/关键点`、`#### 3.x.4 结合这个场景说目前这个设计是怎样的`、`#### 3.x.5 结合这个场景说目前这个设计的技术亮点`、`#### 3.x.6 结合这个场景说目前这个设计潜在的不足之处`。不要把这些标题改成普通段落，也不要省略编号。
7. 针对每个选中的场景，使用 `Class.method()` 级别输出精细调用链。`#### 3.x.2 调用链文本图` 下必须使用 fenced `text` 代码块，并用 `|`、`v`、`->` 等可见箭头表达流向；不要输出缩进代码块、项目符号列表或没有箭头的平铺列表。
8. 调用链里的每个方法节点都要加 `【】` 标注方法作用，例如 `ConsoleController.login() 【登录页入口】`。
9. 如果这个方法节点来自本地项目文件，要在下一行补源码定位：`// 模块>文件>方法名`，例如 `// console-app>ConsoleController.java>login`。
10. 源码定位下一行再用小括号 `()` 说明方法内部怎么处理、数据怎么变化或传到哪里。
11. 每个场景都说明：什么时候被调用、谁调用谁、怎么做的、数据怎么传。
12. 每个场景都要额外结合项目说明/README/产品目标拆成三个独立小节写清楚：
   - `结合这个场景说目前这个设计是怎样的`
   - `结合这个场景说目前这个设计的技术亮点`
   - `结合这个场景说目前这个设计潜在的不足之处`
13. 硬性要求：每个场景都必须出现“技术亮点”和“潜在的不足之处”，不能省略；这两节必须展开讲，不能只写短 bullet 或一句话结论。
14. `技术亮点` 至少写两个实质性点；每个点都要说明设计点是什么、为什么适合这个场景、解决了什么问题、给项目带来什么收益或取舍。
15. `潜在的不足之处` 至少写两个实质性点；每个点都要说明风险/限制是什么、什么情况下会触发、会造成什么影响、可以怎么具体改进。潜在不足必须聚焦于并发安全、单点故障、协议鲁棒性、内存泄漏风险、状态不一致、硬件阻塞、生命周期泄漏、背压、重试/幂等或运维可观测性等架构/工程权衡层面；不要用“代码注释不够”“没有写单元测试”这类泛泛问题凑数，除非它们会直接放大该场景的架构级风险。如果代码里没有明显问题，也要写出谨慎的工程权衡或潜在风险；若属于推测或建议，必须标明是推测/建议。

【考察模式（可选切换）】
1. 当我输入“进入考察模式”，或者在阶段三完全结束后由你主动邀请并获得我确认，系统应立刻暂停当前的常规文档输出，读取并严格遵循 `references/rules/inspection-mode.md`。当我输入“你考我一下”“你问我一下”时，必须先遵循主 `SKILL.md` 的 `Ambiguous Trigger Arbitration Rule`：若当前是 `docs/learn/` 局部学习范围则进入学习测试模式；只有路由到 `doc/project/` 或默认项目考察范围时，才进入本项目考察模式。
2. 考察范围是整个项目的核心逻辑和核心原理，不考局部代码记忆；问题应围绕业务目标、系统边界、核心闭环、模块协作、对象/数据流、状态流转、关键状态节点、异步/外部/设备/客户端接入链路和设计取舍展开。
3. 切入考察模式后的首轮回复必须只输出第 1 个问题，然后立刻 Stop and wait，等待我回答；严禁在同一轮内自行模拟我的回答，严禁一次性把后续问题、评分格式、评价模板或参考答案都打出来。
4. 活跃考察轮次内属于只读沙箱，不写任何物理文件。退出考察模式后或用户明确要求持久化后，追加考察记录时只能增量追加新问题与反馈；如果某题低于 8 分，文档只保留该题题目和参考答案，不写用户回答、评分、足和不足；请勿重写、重排、润色或篡改文档前半部分的【阶段一】至【阶段三】已定稿系统架构分析内容。
5. 用户退出考察模式时，按 `references/rules/inspection-mode.md` 输出满分 100 分的总评；总评只展示给用户，不写入文档。
请先读代码和配置，然后连续输出阶段一和阶段二；写完阶段二后暂停，等待我回复“继续”再执行阶段三。
```

## Quality Bar

The analysis is good only if a new engineer can answer:

- “为什么选这些场景？它们合起来覆盖了项目哪些核心能力？”
- “每个场景的触发源、核心技术挑战和最终物理状态是什么？”
- “我点这个按钮后后端发生了什么？”
- “设备/外部系统发来数据后谁先收到？”
- “数据经过哪些对象、表、topic、文件？”
- “哪个模块负责接入，哪个模块负责处理？”
- “系统现在做到哪里，还缺什么？”
