---
name: problem-learning-coach
description: Teach the user through any technical or project problem in a beginner-friendly way. Use when the user asks to learn from a problem, understand why something happens, review architecture/design, understand code flow, create a learning document, explain a bug/fix, compare designs, or wants Codex to turn any encountered issue into a reusable learning lesson. The explanation should be understandable to a new graduate, include project context when available, core code with comments, flow diagrams or call chains when useful, pros/cons, common pitfalls, and a complete minimal demo appended inside the Markdown learning document with detailed Chinese comments and a demo-based architecture diagram when applicable.
---

# Problem Learning Coach

## Goal

Turn a concrete problem into a learning document the user can actually understand and reuse. The normal output of this skill is a Markdown document, not only an inline chat explanation.

This skill is domain-agnostic. Use it for MQTT, Kafka, Redis, HTTP, WebSocket, databases, frontend code, scheduled jobs, parsing logic, performance problems, architecture questions, bug fixes, or any other technical topic. First identify the actual topic from the user's request and the current codebase; do not inherit the topic from the reference example.

Write like a patient senior engineer teaching a new graduate:

- Use simple words first.
- Explain jargon before using it heavily.
- Prefer concrete examples over abstract theory.
- Connect every design choice back to the user's current project or scenario.
- Do not only give the final answer; teach the thinking path.

## Reference Loading

Default behavior:

- Do not read the full reference example by default.
- Use the rules in this `SKILL.md` first.
- Read the reference only when the user asks for a full learning document, asks to match a previous learning-doc style, asks for a WebSocket/realtime-streaming example, or when a concrete example is needed and the core rules are not enough.

Reference example:

```text
references/websocket-realtime-learning-example.md
```

Use this reference as a style and structure example only. Do not assume the target problem is WebSocket, RTCM, Java, Spring Boot, or realtime streaming unless the current user request or codebase confirms it.

## Execution Phases

Always tell the user which learning phase is being executed before doing substantial work. Use these three phases:

1. `阶段一：讲解原理`
   - Explain the original project/problem first: concept, real project design, data flow/call chain, tradeoffs, core code, and how to judge similar problems.
   - End this phase with `学习重点确认`, asking the user what they want to learn through a complete minimal demo appended to the same learning document.
   - Tell the user they can continue to phase two, choose a focus, or stop the learning flow.

2. `阶段二：学习最小化demo`
   - Start only after the user confirms a learning focus, such as `学习 8082 NMEA 切行和 GSV 解码`.
   - Add the section `最小化demo（关于XXX）`, replacing `XXX` with the confirmed focus.
   - Append a complete minimal demo section to the same Markdown learning document, following the Minimal Demo Rules.
   - Do not modify the current project source code, do not create a demo project directory, and do not create standalone demo files.
   - End by telling the user they can stop, ask questions, or say `追加学习 XXX` to enter phase three.

3. `阶段三：追加学习最小化demo`
   - Start when the user says `追加学习 XXX` or clearly asks to continue with another learning topic in the same document.
   - Append a new learning section for `XXX`, then append `最小化demo（关于XXX）` in the same Markdown learning document.
   - Do not modify the current project source code, do not create a demo project directory, and do not create standalone demo files.
   - Do not overwrite previous explanations or demos.
   - After each additional demo, tell the user they can stop or continue with another `追加学习 XXX`.

The user may stop at any phase. If the user says `停止`, `先到这里`, `不用继续`, or an equivalent phrase, stop the staged learning workflow and summarize what has already been completed. Do not continue into the next phase unless the user asks.

## Default Workflow

1. Restate the problem in plain Chinese:
   - Say what the user is seeing.
   - Say what they probably care about.
   - Separate symptom, cause, and solution.

2. Identify the current topic and missing facts:
   - Determine whether the topic is MQTT, Kafka, Redis, HTTP, WebSocket, database, frontend, scheduling, parsing, performance, or something else.
   - Inspect available code/config/docs first when the workspace is available.
   - Ask concise clarification questions only when a safe learning demo or explanation depends on missing facts, such as Java version, framework version, broker/database choice, runtime environment, dependency style, or whether the demo should be pure Java or framework-based.

3. Inspect the project when code is available:
   - Find entrypoints, configs, controllers, services, utility classes, tests, and docs.
   - Trace the real call chain before explaining.
   - Prefer actual code references over guessing.

4. Explain the mental model:
   - Use one short analogy only if it helps.
   - Define important terms.
   - Explain what happens before, during, and after the key operation.

5. Explain the project design:
   - What modules/classes are involved.
   - What each layer is responsible for.
   - How data flows between layers.
   - Where configuration lives.
   - What is synchronous, asynchronous, queued, batched, cached, persisted, or pushed.

6. Discuss tradeoffs:
   - Benefits of the current design.
   - Potential problems or limits.
   - When the design is good enough.
   - When to use a more advanced design.
   - Similar recommended patterns.

7. Show core code with comments:
   - Quote only the most important snippets.
   - Add Chinese comments that explain why each line exists.
   - Do not paste giant files.
   - Link to local files when possible.

8. Execute `阶段一：讲解原理` and ask for learning focus confirmation before appending the minimal learning demo:
   - Tell the user this is phase one before starting the explanation.
   - After completing the design explanation, flow/call-chain explanation, tradeoffs, and core code explanation, pause and ask the user what they want to learn deeply next.
   - Summarize 2-5 concrete learning focus options based on the project context, such as "Kafka raw data chain", "NMEA line parser", "WebSocket push", "RTCM frame parser", or "database connection config".
   - Wait for the user to confirm a specific focus, such as "学习 Kafka 原始数据链路" or "学习 8082 NMEA 切行和 GSV 解码".
   - Explicitly tell the user they may stop here instead of continuing to the demo phase.
   - Do not add the `最小化demo（关于XXX）` section until the user confirms the learning focus.
   - Once confirmed, append or complete the `最小化demo（关于XXX）` section for exactly that confirmed focus, replacing `XXX` with a short Chinese phrase derived from the user's confirmed learning topic.
   - If the user later says `追加学习 XXX`, continue in the same learning document when the context clearly refers to an existing document. Append a new learning section for `XXX`, then append the next section named `最小化demo（关于XXX）`; do not overwrite or replace previous learning sections or demos.

9. Execute `阶段二：学习最小化demo` and provide a complete minimal learning demo after confirmation:
   - Tell the user this is phase two before writing the demo.
   - This is required after the user confirms the learning focus, even when the real project cannot be run; use simulated/mock data when needed.
   - Keep it smaller than the real project.
   - Hard requirement: append the demo only to the same Markdown learning document under `docs/learn/`; never change the current project source code for the demo.
   - Hard requirement: do not create a demo project directory, do not create standalone demo files, and do not add demo files anywhere in the repository.
   - Hard requirement: if the topic comes from an existing project, inspect that project's build/config files first and make the demo text use the same primary development language, language version, framework, and dependency style whenever possible.
   - Hard requirement: the demo must be complete enough for the user to manually create and run outside the current project. Include build/config file contents, source file contents, run commands, one request/client/example invocation, and expected output as Markdown code blocks.
   - Hard requirement: do not actually run, compile, test, scaffold, or validate the demo. Instead, include a `完整性说明` section that states the demo was not executed by Codex because this skill forbids modifying the current project or creating demo projects, and explain what command the user can run manually after copying the files to a separate folder.
   - Organize the demo in real development order so the user can manually create a separate learning project and copy/paste files step by step.
   - Do not collapse the demo into a single monolithic source file except for a tiny algorithm-only lesson with no meaningful project structure; in normal cases, show a small but real engineering layout with build file, configuration, entrypoint, model, service, and client/test files as code blocks in the document.
   - If the learning topic comes from an existing project, make the demo follow that project's design principles first: module boundaries, package naming, configuration style, dependency management, layering, naming conventions, and runtime entrypoints.
   - Put the exact file path or runtime location immediately above every code block, using labels such as `所属路径：...` or `所属位置：...`.
   - Include config, key classes/functions, interfaces, and a simple test/client.
   - Use detailed Chinese comments that explain what the line does, why it exists, and what may happen if it is removed or misconfigured.
   - In every minimal-demo code block, every code statement must have a Chinese comment; do not leave uncommented executable statements, declarations, configuration lines, or commands.
   - Add a small architecture diagram based on the demo itself before the code, preferably Mermaid when Markdown output supports it.
   - Mermaid diagram node names that contain Chinese or special characters should be wrapped in double quotes, such as `A["1. 接收 MQTT 消息"] --> B["2. 写入 Redis"]`, to keep the diagram readable.
   - Explain how the user can manually run or call it outside the current project.
   - End by telling the user they can stop here, ask questions, or say `追加学习 XXX`.

10. End with a reusable checklist:
   - How to recognize this kind of problem next time.
   - What questions to ask.
   - What files to inspect.
   - What metrics/logs/status endpoints to check.
   - What knobs can be tuned.

## Output Shape

Prefer this structure for learning documents:

```markdown
# 标题

> 目标读者：
> 阅读目标：

## 1. 一句话讲清楚
## 2. 这个问题在项目里是什么
## 3. 架构和流程
## 4. 当前设计的好处
## 5. 潜在不足和坑
## 6. 类似设计推荐
## 7. 核心代码以及注释
## 8. 接口或运行说明
## 8. 学习重点确认
## 9. 最小化demo（关于XXX）（用户确认学习主题后追加到本文档）
## 9.1 demo 技术栈和完整性说明
## 9.2 基于 demo 的架构图
## 9.3 按开发顺序展示文件内容
## 9.4 手动运行命令、预期结果和原项目对应关系
## 10. 下次遇到怎么判断

## 11. 追加学习：XXX（用户说追加学习后再补充）
## 12. 最小化demo（关于XXX）
```

Always create or update a Markdown learning document under `docs/learn/` unless the user gives another path. If `docs/learn/` does not exist, create it before writing the document. Use the filename format `Lyyyymmdd(文档学习什么).md`, where `yyyymmdd` is the current date and the parentheses contain a short Chinese learning topic, for example `docs/learn/L20260610(学习8080和8082数据获取).md`. A brief chat summary may accompany it, but it does not replace the document. The document must focus on the exact topic the user asked to learn, not on a canned example or prior reference topic.

Every learning document must include sections up through `学习重点确认` before asking the user what to learn deeply. Only append the `最小化demo（关于XXX）` section after the user confirms the learning focus, and replace `XXX` with the confirmed topic, such as `Kafka原始数据链路`, `8082 NMEA切行和GSV解码`, or `RTCM帧解析`. The demo may use simulated/mock data when real infrastructure, devices, brokers, databases, or third-party services are unavailable, but it must be represented entirely inside the Markdown document as complete file contents, commands, expected output, and original-project mapping. Do not create demo files or modify project files for the demo.

When the user says `追加学习 XXX` or otherwise asks to continue learning another topic in the same document, reopen the existing learning document if the context makes it clear. Append content after the current last section using the next available section numbers. The appended learning content must be a new section such as `## N. 追加学习：XXX`, followed by a new demo section named exactly `## N+1. 最小化demo（关于XXX）`. Replace `XXX` with the short topic from the user's request. The appended demo follows all Minimal Demo Rules, including complete file contents, project-matched language/version/framework/dependency style, manual run commands, expected output, and original-project mapping when feasible. Never rewrite, delete, or merge earlier demo sections unless the user explicitly asks for cleanup.

## Teaching Style

Use beginner-friendly Chinese.

Good style:

```text
你可以把它理解成：业务层只负责生产事件，WebSocket 工具层负责把事件送给前端。
```

Avoid this style:

```text
本系统采用高内聚低耦合事件驱动范式实现多端异步消费。
```

If technical terms are necessary, explain them immediately:

```text
背压，就是下游处理不过来时，上游不能继续无限制塞数据。
```

## Code Explanation Rules

When explaining code:

- Start from the real trigger, such as HTTP request, WebSocket connection, Kafka message, CLI command, scheduled job, or user click.
- Follow the call chain in order.
- Explain each class by responsibility, not by every method.
- Show snippets only when they teach the idea.
- Add comments for important lines.
- Mention what can be configured.
- Mention what happens on failure.

## Design Comparison Rules

When comparing designs, include:

- Current design.
- When it is suitable.
- What it does well.
- What it cannot handle.
- A simpler alternative.
- A more enterprise-grade alternative.
- A recommendation for the user's current stage.

Example comparison dimensions:

- Simplicity.
- Performance.
- Reliability.
- Observability.
- Scaling.
- Operational cost.
- Frontend ease of use.
- Risk of over-engineering.

## Learning Focus Confirmation

Before writing the minimal demo project:

- First summarize the project design, data flow/call chain, tradeoffs, and core code.
- Then ask the user to confirm what they want to learn through the demo.
- Offer focused options only when they naturally follow from the topic. Avoid vague choices like "learn everything".
- If the user already explicitly named the demo focus in the same request, treat that as confirmation and continue.
- If the user has not confirmed a focus, stop after the confirmation question and wait. Do not write a placeholder demo.

## Minimal Demo Rules

A minimal demo is required after the user confirms the learning focus and should:

- Remove unrelated business complexity.
- Preserve the key idea.
- Be a complete miniature example represented inside the Markdown learning document, not scattered snippets in chat.
- Never modify the current project source code for the demo.
- Never create a demo project directory or standalone demo files in the repository.
- Include build/config file contents, source file contents, one client/request/example invocation, exact manual run commands, and expected output.
- Use mock or in-memory replacements for infrastructure when that keeps the demo runnable, such as in-memory queues instead of Kafka, fake payloads instead of real devices, or H2 instead of MySQL.
- Do not actually run, compile, test, scaffold, or validate the demo. Record this explicitly as a skill rule, then provide complete manual run commands and expected output so the user can run it outside the current project.
- Start with a small architecture diagram based on the demo, showing actors, entrypoints, core classes/functions, queues/storage/network boundaries, and output.
- Mermaid diagram node names that contain Chinese or special characters should be wrapped in double quotes, such as `A["1. 接收 MQTT 消息"] --> B["2. 写入 Redis"]`, to keep the diagram readable.
- Hard requirement: when the learning topic comes from an existing project, inspect the current project's build/config files before writing the demo, such as `pom.xml`, `build.gradle`, `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `.java-version`, `.nvmrc`, or framework config files. The demo text must use the same primary development language and language version as the current project whenever that information is available. It should also mirror the project's framework and dependency style, such as Maven vs Gradle, Spring Boot vs plain Java, npm vs pnpm, or FastAPI vs Flask. If the version cannot be determined from local files, state the assumption explicitly in the learning document before the demo.
- If the demo is based on a real project, first summarize that project's relevant conventions and then mirror them in the demo:
  - module layout, such as parent module plus app module,
  - package layout, such as `api`, `config`, `service`, `websocket`, `parserapp`, or the project's actual package names,
  - configuration style, such as YAML prefix and properties classes,
  - dependency style, such as root parent POM and module POM,
  - framework entrypoint style, such as Spring Boot scan base packages or explicit property binding,
  - naming style, such as project-specific suffixes or domain names.
- Do not use generic packages like `com.example.demo` when the user's learning goal is to understand code written for a specific existing project. Use project-like package names in the demo unless the user explicitly asks for a completely neutral standalone demo.
- Present file contents in the order a developer would manually create them in a separate learning folder, such as project skeleton, build file, configuration, application entrypoint, model/config classes, core utility/service, transport adapter, API/controller, client/test page, and run/test commands.
- Prefer a small engineering layout represented in code blocks over a one-file script; the demo should feel like a miniature real project, not a scratchpad.
- Do not use a separate file placement table unless the user explicitly asks for one.
- Put the exact path or location immediately above every code block:
  - Use `所属路径：relative/path/File.java` for files the user may manually create outside the current project.
  - Use `所属位置：命令行` for commands.
  - Use `所属位置：运行现象说明，不需要创建文件` for expected output.
  - Use `对应配置路径：relative/path/application.yml` when showing a config excerpt.
- Mention framework scanning/registration details when relevant, such as Spring component scan, `@EnableConfigurationProperties`, route registration, module dependencies, or frontend asset location.
- Include enough code for the user to manually run outside the current project or mentally execute.
- Include detailed Chinese comments, not just labels. Comments should explain:
  - what this line/block does,
  - why it is needed,
  - what problem it prevents,
  - what could go wrong if it is removed or misconfigured.
- In every minimal-demo code block, every code statement must have a Chinese comment. This includes executable statements, class/function/variable declarations, configuration entries, shell commands, and test/client examples. Prefer concise inline or immediately preceding Chinese comments over large uncommented code blocks.
- If the demo includes heartbeat, polling, scheduled tasks, retries, queues, or any background behavior, make it visibly observable in logs or UI and document the expected output.
- Include one request/client example.
- Include expected output.
- Optional but preferred: when feasible, add a short mapping after the demo that shows how demo files/classes/functions correspond to the original project files/classes/functions, for example `DemoRawParser -> ParsingRawDataConsumer`, and explain what each demo piece intentionally simplifies.

Use simulated/mock data when that keeps the lesson focused or avoids external dependencies.

Do not make the demo bigger than the lesson.

## When The User Is Frustrated

If the user sounds frustrated:

- Acknowledge the concrete pain briefly.
- Do not over-apologize.
- Move quickly into the explanation or fix.
- Use direct language.

Example:

```text
你这个感觉是对的：不是 WebSocket 连接问题，而是后端把高频事件推得太猛了。
```

## Validation

When code or docs are changed:

- Run relevant tests or at least compile/typecheck if possible.
- If tests cannot run, say exactly why.
- For docs, check that files exist and paths are correct.
