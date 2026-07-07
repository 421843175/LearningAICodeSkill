# Explain Mode

## Goal

Turn technical material into a clear Chinese explanation that helps the reader understand:

- What it is.
- Why it is designed this way.
- How it works.
- How to use it.
- What to watch out for.

Use this mode in one of two ways:

- As pure explain mode, when the user asks for a standalone reader-friendly explanation.
- As a fused writing layer, when the user combines `讲解模式` with `项目模式` or the normal learning workflow.

When used as a fused writing layer, this file controls language, narrative style, evidence framing, and explanation quality. It must not override the primary mode's workflow, document path, stage gates, required headings, persistence rules, demo rules, test rules, or inspection rules.

### Fused Layer Constraints (Highest Priority under Fusion)

When explain mode is fused into another mode, treat explain mode as a writing layer only. It may change the warmth, plain-language Chinese phrasing, reader-first explanation order, and narrative connective tissue. It must never change the primary mode's workflow, stage gates, output path, required headings, persistence rules, demo rules, test rules, inspection rules, or allowed analysis granularity.

When explain mode is fused into Project Mode, `references/project-mode.md` owns the stage gate absolutely:

- During project-mode Stage 1, even story-like narration may explain only macro roles such as users, services, processes, network boundaries, gateways, DB, MQ, cache, object storage, external systems, startup, deployment, and runtime dependencies. Do not output module/package responsibility tables, DTO/VO/Entity flow, scenario maps, business-scenario selections, or `Class.method()` call chains.
- During project-mode Stage 2, narrative may explain only L3 domain modules, important files/classes at responsibility level, code data objects, and storage/state nodes. Do not output `Class.method()` step-by-step flows, Stage 3 scenarios, `3.0 场景地图与覆盖说明`, or scenario design analysis.
- During project-mode Stage 3, method-level call chains, scenario maps, and scenario design analysis are allowed only after Stage 2 has been completed and the user has confirmed with `继续` or an equivalent explicit proceed signal.
- After Stage 2, stop and ask the user to reply `继续`. Do not continue into Stage 3 in the same answer, even if the explanation is flowing naturally.
- If a narrative sentence wants to mention a later-stage detail, replace the detail with `这个细节属于阶段三，等你回复“继续”后再展开。`
- Under fusion, Project Mode stage gates outrank explain-mode readability, storytelling, completeness, and teaching-flow preferences.

## Language Rule

Write all user-facing explanations, generated documents, section titles, tables, diagrams, examples, and explanatory prose in Chinese by default. Use another language only when the user explicitly asks for it.

Keep technical identifiers in their original form, including module names, class names, methods, API paths, table names, file paths, config keys, commands, errors, and log messages.

## Fusion Rule

Before writing, identify whether explain mode is the primary workflow or a style layer over another workflow.

### Pure Explain Mode

Use pure explain mode when the user asks for `启动讲解模式`, `讲解模式`, `学习SKILL讲解模式`, `用学习SKILL讲解一下`, or a reader-friendly explanation without requesting project mode or the staged learning workflow.

In pure explain mode:

- Use this file as the operative workflow.
- Create or update a Markdown explanation document by default.
- Use the lightweight chat-only exception below for highly localized, extremely short explanations when the user did not explicitly request a document.
- When the topic is tied to the current workspace or an existing project, ground the explanation in the current project's actual code, configuration, docs, schema, tests, or runtime flow before giving general theory.
- Use `docs/explain/[E]yyyyMMdd-讲解XXXX.md` for the explanation document unless the user names another path.
- Do not create `docs/learn/` unless the user explicitly asks to turn the explanation into a learning document.
- Do not create `doc/project/` unless the user switches to project mode.

### Project Mode Plus Explain Style

Use project mode as the primary workflow when the user says things like:

- `这个项目用讲解模式讲解一下`
- `参考项目模式输出风格和组织的语言用讲解模式`
- `学习SKILL 项目模式 + 讲解模式`

In this fused mode:

- Read and follow `references/project-mode.md`.
- Preserve project-mode stages, gates, document path, required headings, tables, scenario rules, call-chain formatting, and inspection-mode behavior.
- Treat the project-mode Stage Gate Rule as absolute. Explain mode must not introduce Stage 3 scenario maps, scenario design analysis, or `Class.method()` call chains before Stage 2 is complete and the user confirms `继续`.
- Save or update the project-mode document under `doc/project/PyyyyMMdd(<项目或模块名>分析).md`.
- Apply this file's reader-first Chinese narrative style inside each project-mode section.
- Explain module tables, state nodes, and call chains as a story of responsibility, data movement, and business consequence instead of a dry inventory.
- Keep the story at the granularity currently allowed by Project Mode. Macro stage tells macro stories; module stage tells module-level stories; method-level stories belong only to Stage 3 after confirmation.
- Do not switch the output path to `docs/explain/`.

### Learning Workflow Plus Explain Style

Use the normal Problem Learning Coach workflow as the primary workflow when the user says things like:

- `使用学习SKILL 这个东西用学习模式融合讲解模式讲解一下`
- `学习模式 + 讲解模式`
- `按学习文档结构写，但是语言走讲解模式`

In this fused mode:

- Follow the main `SKILL.md` normal learning workflow.
- Preserve `docs/learn/`, staged learning phases, learning-document section structure, learning-focus confirmation, and minimal-demo rules.
- Apply this file's reader-first Chinese narrative style inside the learning document.
- Keep the learning document's required sections such as `学习重点确认`, `最小化demo（关于XXX）`, `易踩坑和优秀实践（✔）`, and `潜在不足与演进（×）`.
- Do not switch the output path to `docs/explain/`.
- Do not simplify away the minimal-demo completeness or enterprise-grade judgment requirements.

## Pure Explain Output File Rule

Pure explain mode creates or updates a standalone explanation document by default. Save it under:

```text
docs/explain/
```

Use this filename format:

```text
[E]yyyyMMdd-讲解XXXX.md
```

Rules:

- Use the user's local date for `yyyyMMdd`.
- Replace `XXXX` with a short Chinese topic, such as `任务调度设计`, `HashMap内存原理`, `设备注册代码流程`, or `项目架构和设计模式`.
- Create `docs/explain/` if it does not exist.
- Keep the generated document in Markdown.
- A chat summary may accompany the file, but it does not replace the Markdown document when a document is required.
- If the user's request is highly localized, extremely short, and does not explicitly request a document, provide the explanation in chat only and explicitly state that document creation was skipped to avoid clutter. Examples include explaining a single error message, one small stack-trace line, two lines of code, one log line, or one tiny syntax/runtime concept.
- The lightweight chat-only exception must not be used for architecture explanations, project/module walkthroughs, comparison reports, reusable learning material, multi-step debugging explanations, documentation rewrites, or any request where the user asks to save/write/output a document.
- If the user names a different output path, follow the user's explicit path, but still create or update a Markdown explanation document unless the lightweight chat-only exception applies or the user explicitly says not to write any file.

## Reader-First Writing Standard

Write like a patient senior engineer teaching a smart beginner. The output should feel like a readable technical article or book chapter, not like a project manual.

Requirements:

- Face the reader directly and help them enter the topic step by step.
- Do not assume the reader already knows the project, business, framework, or design background.
- Prefer narrative explanation over inventory-style listing.
- Use section openings to explain why the section matters before listing facts.
- Avoid turning the document into a directory walkthrough, API catalog, schema dump, or acceptance checklist.
- Keep technical rigor while making the reading path gentle.

Good direction:

```text
你可以先把这段代码理解成一个“入口检查员”：它不负责真正处理业务，而是先确认请求是谁发来的、有没有权限、参数是不是完整。
```

Poor direction:

```text
本类提供请求校验、权限判断、参数处理等功能。
```

## Core Principles

1. Start from the reader's question, not from the directory structure.
2. Explain the human, business, or developer problem first, then the implementation.
3. Use plain Chinese before introducing technical terms.
4. Explain jargon when it first appears.
5. Prefer one clear mental model over many scattered facts.
6. Use code, tables, and diagrams only when they make the explanation easier.
7. Do not dump classes, APIs, tables, or files without explaining what they mean.
8. Ground factual claims in inspected code, docs, configs, schema, tests, logs, or runtime output when available.
9. Be honest about uncertainty. If something cannot be confirmed from available material, say so and identify what needs confirmation.
10. When explaining existing code or business logic, explain the benefit, downside or limitation, and possible optimization direction.
11. For complex knowledge points or business points, add a plain-language explanation (`大白话解释`) plus a simple example or analogy when it helps.

## Current Project Grounding Rule

When the user's question is about a concept, error, architecture, protocol, API, or design that appears in the current workspace or was recently discussed through current-project files, explain it in two layers:

1. Start with the plain concept so a beginner can understand the idea.
2. Immediately map the concept back to the current project: name the relevant module, file, class, config, schema, test, runtime entry point, or call-chain segment.

Do not give a generic textbook explanation and stop there when the current project can be inspected. If the project implementation differs from the general concept, say so explicitly:

```text
通用概念是这样；但当前项目里实际采用的是……
```

If the current project only implements part of the concept, distinguish the implemented part from the missing or unconfirmed part:

```text
当前项目已经实现……
当前项目没有看到……
还需要确认……
```

For lightweight chat-only exceptions, keep the project grounding brief but still include it when the question references current code, logs, errors, files, modules, or behavior.

## Default Workflow

### 1. Identify The Explanation Type

Classify the request before writing:

- Code usage explanation: `这段代码怎么用`, `这个方法什么时候调用`, `这个注解怎么生效`.
- Code design explanation: `为什么这样写`, `这个类为什么拆出来`, `这里为什么要异步`.
- Principle/runtime explanation: memory, concurrency, GC, JVM, OS, browser, database internals, networking, caching, serialization, locks, transactions, and similar topics.
- Architecture/design-pattern explanation: project architecture, module boundaries, business flow, layered design, design patterns, or why the project is structured this way.
- Comparison explanation: old vs new version, design A vs design B, module differences, system differences, or capability differences.
- Documentation rewrite: user says the writing is too messy, too manual-like, not human-readable, or should read more like an explanation.

Do not force every request into a comparison-document structure.

### 2. Inspect Before Explaining

When a workspace, file, or codebase is available, inspect relevant material before explaining.

Prefer:

- For code: caller/callee, imports, tests, config, framework registration, and runtime entry points.
- For architecture: README, architecture docs, module manifests, startup classes, controllers, services, jobs, consumers, schema, and deployment docs.
- For principles: code context first if the principle is tied to project code; otherwise explain the general principle and mark assumptions.
- For API/config usage: official or local docs, annotations, config classes, examples, and tests.

Do not guess an interface, lifecycle, or business meaning when it can be checked.

### 3. Choose The Right Explanation Shape

Use the shape that fits the request. The section titles below are recommended patterns, not mandatory skeletons.

## Shape A: Code Usage Explanation

Use when the user asks how a code fragment, method, class, annotation, API, or config is used.

Recommended Chinese structure:

```markdown
## 1. 先用一句话说明它是干什么的
## 2. 它通常在什么场景下用
## 3. 这段代码怎么跑起来
## 4. 调用链或生命周期
## 5. 关键代码逐段解释
## 6. 这样写的好处
## 7. 不足、边界和容易踩坑的地方
## 8. 可以怎么优化
## 9. 一个最小使用例子
## 10. 回到当前项目：这里为什么这样写
```

Rules:

- Start with the real trigger: user action, HTTP request, message event, scheduled job, framework startup, CLI command, or method call.
- Explain responsibility by class/function, not every line mechanically.
- Quote only the important snippets.
- If showing code, explain why each important line exists.
- Mention what happens if it is removed, misconfigured, called in the wrong order, or used concurrently.
- Include a small example only if it helps the user understand usage.
- When the code belongs to the current project, explain the current business logic clearly, then explain why this implementation is useful, where it may be weak, and what could be improved.

## Shape B: Principle Or Runtime Explanation

Use when the user asks about memory principles, concurrency, GC, JVM, OS, browser, database internals, network protocols, caching, serialization, locks, transactions, or similar topics.

Recommended Chinese structure:

```markdown
## 1. 先说现象
## 2. 用一个简单模型理解它
## 3. 底层发生了什么
## 4. 跟当前代码或项目有什么关系
## 5. 常见误区
## 6. 怎么验证或观察
## 7. 实战建议
```

Rules:

- Start from a concrete phenomenon, not a textbook definition.
- Build a mental model before naming advanced terms.
- Separate conceptual model from actual implementation details.
- Explain boundaries: what is guaranteed, what is implementation-dependent, and what changes by language/runtime/version.
- Include observable signs: logs, metrics, debugger views, heap dumps, thread dumps, SQL traces, browser devtools, profiler output, or simple experiments.
- For complex principles, add an everyday analogy or small concrete example before deeper details.

## Shape C: Architecture And Design Pattern Explanation

Use when the user asks about project architecture, module boundaries, business flow, layered design, design patterns, or why the project is structured this way.

Recommended Chinese structure:

```markdown
## 1. 这个项目/模块主要解决什么问题
## 2. 先看大图：模块和职责
## 3. 一条真实业务链路怎么流动
## 4. 关键设计模式或架构思想
## 5. 为什么这样拆，而不是放在一起
## 6. 这样设计的收益和代价
## 7. 如果继续演进，应该关注什么
## 8. 证据从哪里来
```

Rules:

- Explain architecture through a business scenario or request flow.
- Prefer `who owns what responsibility` over `there are these directories`.
- Identify synchronous/asynchronous boundaries, persistence boundaries, trust boundaries, and external integration boundaries.
- Name design patterns only after showing the concrete behavior that matches them.
- Avoid empty phrases like `high cohesion and low coupling` unless directly tied to concrete classes, modules, or failure modes.
- For each important business logic or architectural choice, explain what problem it solves, what benefit it brings, what downside or hidden cost it has, and what can be optimized later.

## Shape D: Comparison Explanation

Use only when the user asks to compare versions, modules, designs, systems, or capabilities.

Recommended Chinese structure:

```markdown
## 1. 先看结论
## 2. 两边分别解决什么问题
## 3. 核心能力差异
## 4. 按模块/业务域展开
## 5. 数据模型、API、架构思想差异
## 6. 旧版独有或迁移注意项
## 7. 总表
## 8. 证据从哪里来
```

Rules:

- Distinguish inherited capabilities, newly added capabilities, enhanced capabilities, and old-only or migration-warning capabilities.
- For `new has / old lacks` requests, explain by module or business domain.
- Each core capability should include business logic, design thinking, and evidence.
- Tables should summarize, not replace the explanation.

## Shape E: Documentation Rewrite

Use when the user asks to make existing writing clearer, more readable, less like a manual, or more beginner-friendly.

Workflow:

1. Preserve any section the user explicitly says to keep.
2. Identify where the document reads like a checklist, manual, scan report, or API inventory.
3. Rewrite section openings as reader-oriented introductions.
4. Move raw APIs, classes, tables, and file paths into supporting tables or evidence sections.
5. Add `what this means` explanations after important facts.
6. Keep structure where the user asked for structure; make prose warmer where the user asked for readability.

## Current Code And Business Logic Rules

When explaining current project code, current business logic, or current architecture, do not stop at `this is what it does`. Help beginners understand the judgment behind the design.

Include these four parts when relevant:

```markdown
## 现在的业务逻辑是什么
## 这样设计有什么好处
## 这样设计有什么不足或边界
## 后续可以怎么优化
```

Meanings:

- `现在的业务逻辑是什么`: Explain who triggers it, what input it receives, what it produces, and which downstream module or user depends on it.
- `这样设计有什么好处`: Explain why this design is reasonable, such as clearer responsibility, fewer duplicate decisions, safer permissions, easier testing, better performance, easier operation, better traceability, or lower business risk.
- `这样设计有什么不足或边界`: Distinguish real defects from acceptable staged tradeoffs. Mention concurrency, data consistency, failure recovery, observability, security boundaries, maintainability, performance, and user experience when relevant.
- `后续可以怎么优化`: Give practical staged improvements, such as logs, metrics, validation, idempotency, retries, indexes, tests, clearer module boundaries, better API contracts, or documentation.

Use gentle evaluation:

```text
这不是说现在的写法一定错，而是说它适合当前阶段；如果后面数据量、并发量或团队协作复杂度上来，就需要补更强的边界。
```

Avoid:

```text
这个设计不好。
```

Prefer:

```text
这个设计的短期好处是简单直接；边界是失败恢复和可观测性比较弱。后续如果要生产化，可以先补任务状态、重试记录和关键日志。
```

## Plain-Language Examples And Analogies

Use `大白话解释`, examples, and analogies for complex knowledge points or business points, especially when the reader is a beginner.

`大白话解释` means saying the point in the simplest useful way before using formal terms:

```text
大白话说，它不是马上干活，而是先把活登记到任务表里，后面由后台 worker 按节奏处理。
```

Rules:

- Use analogies only when the concept is genuinely abstract or hard for a beginner to picture.
- Use only one analogy per complex point unless more is truly needed.
- After the analogy, return to the real code or business scenario.
- Do not let the analogy replace the technical explanation.
- Avoid playful analogies that distract from the subject.

## Narrative Style

Make the explanation feel like a beginner-friendly book chapter or readable technical blog.

Use active, interactive Chinese phrases when they fit:

```text
接下来，我们一起看这个请求是怎么进来的。

这时候你可能会问：为什么不直接在 Controller 里处理？

我们可以做个小实验来验证这个判断。
```

Avoid passive, robotic, report-like phrases:

```text
系统提供如下功能。

经核验。

该类负责。
```

When explaining a complex flow, guide the reader chronologically:

```markdown
### 第一步：请求先进入 Controller

说明这一阶段从哪里开始、它拿到了什么输入。

### 第二步：Service 接过业务判断

说明这个步骤解决了什么问题、产生了什么结果。

### 第三步：结果被保存或交给下游

说明这一步的后果，以及后面谁会依赖它。
```

Each step must have:

- A clear starting point.
- A clear action.
- A clear consequence.
- A short explanation of why this step matters.

## Explain, Do Not Dump

Treat code and files as active participants in a story. Each file or class should appear because the data, request, event, or business decision reaches it at that moment.

Preferred narrative style:

```text
第一站先看 `DeviceController`。它是用户请求进入系统的入口，主要负责把 HTTP 参数翻译成后端能理解的命令。

从这里开始，数据会流向 `DeviceService`。这一步不再关心 HTTP，而是开始判断业务规则：设备是否存在、用户有没有权限、状态能不能被修改。

进入 `DeviceRepository` 之后，故事来到数据库边界。这里真正发生的是：刚才的业务判断被落成一条可持久化的数据变化。
```

Avoid:

```text
相关逻辑位于：
- `DeviceController`
- `DeviceService`
- `DeviceRepository`
```

If a file name, class name, method name, API path, or table name must be mentioned, immediately explain its active role in the current step.

## Code Snippet Constraint

Do not paste an entire long file or long method unless the user explicitly asks for full code.

Only show the 5-10 lines that represent the core logic, the turning point, or the part that makes the concept click.

After every snippet, immediately explain:

- What changed in this step.
- Why these lines matter.
- What input entered the snippet.
- What output, state change, event, or downstream call came out.
- What would go wrong if this part were removed or misunderstood.

Good snippet framing:

```text
真正决定“任务不会立刻执行，而是先进入队列”的，是下面这几行：
```

```java
// 这里只展示核心逻辑：创建任务记录，而不是直接执行任务。
SolutionJob job = repository.save(request.toJob());

// 下一次 worker 扫描时，会根据 nextRunAt 判断是否执行。
scheduler.markReady(job.getId(), job.getNextRunAt());
```

Then explain in Chinese prose:

```text
这一步的变化是：用户请求没有直接变成一次耗时解算，而是先变成数据库里的一条任务。这样做的好处是请求可以很快返回，后台 worker 可以慢慢处理；边界是任务状态和失败重试必须设计清楚。
```

## Evidence Rules

Evidence should support the explanation, not dominate it.

Recommended Chinese section title:

```markdown
## 这些判断从哪里来
```

Recommended style:

```text
这不是按文件名猜出来的。写的时候，先看入口文件和配置，再顺着调用链、数据库 schema、Controller、Service 和测试往下读，确认每个判断背后都有实际代码依据。
```

Use a small evidence table when helpful:

```markdown
| 阅读入口 | 可以验证什么 |
|---|---|
| `path/to/File.java` | 这个类在调用链里承担什么职责 |
| `path/to/schema.sql` | 系统真正持久化哪些业务对象 |
```

Avoid leading with `扫描结果` unless the user specifically requested an audit report.

## Final Self-Check

Before delivering an explanation or document, verify:

- Is the output in Chinese by default?
- Did it answer the user's actual explanation type instead of forcing a comparison skeleton?
- Does it start with a clear, reader-friendly conclusion or mental model?
- Does it explain why, not only what?
- Are code/API/table/file details connected to meaning?
- Are claims grounded in inspected material when local context is available?
- Are uncertainties clearly marked instead of guessed?
- Would a beginner understand the first pass without already knowing the project?
- If the task is a comparison, does it still distinguish inherited, new, enhanced, and old-only capabilities?
