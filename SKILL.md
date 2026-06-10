---
name: problem-learning-coach
description: Teach the user through any technical or project problem in a beginner-friendly way. Use when the user asks to learn from a problem, understand why something happens, review architecture/design, understand code flow, create a learning document, explain a bug/fix, compare designs, or wants Codex to turn any encountered issue into a reusable learning lesson. The explanation should be understandable to a new graduate, include project context when available, core code with comments, flow diagrams or call chains when useful, pros/cons, common pitfalls, and a minimal runnable demo with detailed Chinese comments and a demo-based architecture diagram when applicable.
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

8. Provide a minimal learning demo in every learning document:
   - This is required, even when the real project cannot be run; use simulated/mock data when needed.
   - Keep it smaller than the real project.
   - Organize the demo in real development order so the user can create a new project and copy/paste files step by step.
   - Do not collapse the demo into a single monolithic source file except for a tiny algorithm-only lesson with no meaningful project structure; in normal cases, split it into a small but real engineering layout with build file, configuration, entrypoint, model, service, and client/test files.
   - If the learning topic comes from an existing project, make the demo follow that project's design principles first: module boundaries, package naming, configuration style, dependency management, layering, naming conventions, and runtime entrypoints.
   - Put the exact file path or runtime location immediately above every code block, using labels such as `所属路径：...` or `所属位置：...`.
   - Include config, key classes/functions, interfaces, and a simple test/client.
   - Use detailed Chinese comments that explain what the line does, why it exists, and what may happen if it is removed or misconfigured.
   - In every minimal-demo code block, every code statement must have a Chinese comment; do not leave uncommented executable statements, declarations, configuration lines, or commands.
   - Add a small architecture diagram based on the demo itself before the code, preferably Mermaid when Markdown output supports it.
   - Explain how to run or call it.

9. End with a reusable checklist:
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
## 9. 最小化学习 demo
## 9.1 基于 demo 的架构图
## 9.2 按开发顺序新建工程并写代码
## 10. 下次遇到怎么判断
```

Always create or update a Markdown learning document under `docs/` unless the user gives another path. Use the filename format `Lyyyymmdd(文档学习什么).md`, where `yyyymmdd` is the current date and the parentheses contain a short Chinese learning topic, for example `L20260610(学习8080和8082数据获取).md`. A brief chat summary may accompany it, but it does not replace the document. The document must focus on the exact topic the user asked to learn, not on a canned example or prior reference topic.

Every learning document must include a `最小化学习 demo` section. The demo may use simulated/mock data when real infrastructure, devices, brokers, databases, or third-party services are unavailable. The demo should preserve the core idea of the requested topic while staying small enough to read and run mentally.

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

## Minimal Demo Rules

A minimal demo is required for every learning document and should:

- Remove unrelated business complexity.
- Preserve the key idea.
- Start with a small architecture diagram based on the demo, showing actors, entrypoints, core classes/functions, queues/storage/network boundaries, and output.
- If the demo is based on a real project, first summarize that project's relevant conventions and then mirror them in the demo:
  - module layout, such as parent module plus app module,
  - package layout, such as `api`, `config`, `service`, `websocket`, `parserapp`, or the project's actual package names,
  - configuration style, such as YAML prefix and properties classes,
  - dependency style, such as root parent POM and module POM,
  - framework entrypoint style, such as Spring Boot scan base packages or explicit property binding,
  - naming style, such as project-specific suffixes or domain names.
- Do not use generic packages like `com.example.demo` when the user's learning goal is to understand code written for a specific existing project. Use project-like package names in the demo unless the user explicitly asks for a completely neutral standalone demo.
- Present files in the order a developer would actually create them, such as project skeleton, build file, configuration, application entrypoint, model/config classes, core utility/service, transport adapter, API/controller, client/test page, and run/test commands.
- Prefer a small engineering project layout over a one-file script; the demo should feel like a miniature real project, not a scratchpad.
- Do not use a separate file placement table unless the user explicitly asks for one.
- Put the exact path or location immediately above every code block:
  - Use `所属路径：relative/path/File.java` for files the user should create.
  - Use `所属位置：命令行` for commands.
  - Use `所属位置：运行现象说明，不需要创建文件` for expected output.
  - Use `对应配置路径：relative/path/application.yml` when showing a config excerpt.
- Mention framework scanning/registration details when relevant, such as Spring component scan, `@EnableConfigurationProperties`, route registration, module dependencies, or frontend asset location.
- Include enough code to run or mentally execute.
- Include detailed Chinese comments, not just labels. Comments should explain:
  - what this line/block does,
  - why it is needed,
  - what problem it prevents,
  - what could go wrong if it is removed or misconfigured.
- In every minimal-demo code block, every code statement must have a Chinese comment. This includes executable statements, class/function/variable declarations, configuration entries, shell commands, and test/client examples. Prefer concise inline or immediately preceding Chinese comments over large uncommented code blocks.
- If the demo includes heartbeat, polling, scheduled tasks, retries, queues, or any background behavior, make it visibly observable in logs or UI and document the expected output.
- Include one request/client example.
- Include expected output.

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
