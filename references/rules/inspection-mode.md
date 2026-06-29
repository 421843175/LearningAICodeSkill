# 考察模式（Inspection Mode）

Use this file only when the user triggers project Inspection Mode by saying `学习SKILL考察模式`, `考察模式`, `项目考察模式`, `进入考察模式`, or says `你考我一下` / `你问我一下` while the current context is project mode or a project-analysis document.

## Goal

Run like a technical interview or architecture review over the whole project. Do not test line numbers, syntax trivia, rote API names, or isolated local implementation trivia. Test the user's understanding of the project's core logic and core principles:

- 整个项目的业务目标、系统边界与核心问题
- 核心业务生命周期与端到端闭环
- 核心架构原理与模块协作方式
- 模块拆分原理
- 模块间依赖与耦合
- 核心对象/数据流转
- 状态机设计与状态流转边界
- 关键状态节点的并发/一致性隐患
- 外部系统、异步链路、设备/客户端接入等关键集成路径
- 核心链路痛点
- 选型折中（Trade-offs）

## Interaction Rules

1. Each Inspection Mode round must ask exactly 5 deep technical questions across the full round.
2. On entering Inspection Mode, the first response must output only question 1, then immediately stop and wait for the user answer.
3. Do not simulate the user answer.
4. Do not output scoring, evaluation, reference-answer placeholders, or later questions in the same turn as the first question.
5. Only after receiving the user answer may you score it, provide feedback/reference answer, then ask the next single question and stop again.
6. If the user replies `跳过`, `跳过这题`, `skip`, or an equivalent request to skip the current question, skip that question immediately: do not score it, do not provide feedback, do not provide a reference answer, and do not persist it as a completed question-and-feedback record. Ask the next single question and stop again. Skipped questions do not contribute to the 100-point overall review except as a note about coverage.
7. Each question should be grounded in the current codebase and the already written Stage 1-3 analysis, but its scope should target whole-project core logic, core principles, or a high-value slice that reveals them. Do not ask generic textbook questions or isolated local code-trivia questions unless the system analysis proves they are relevant to the project's core design.
8. The 5 questions must jointly cover the whole project's core logic and core principles: business goals and system boundaries, core business closed loops, module dependency/coupling, core object/data flow, state transitions, concurrency or consistency risks around key state nodes, external/device/client/async integration where relevant, and design tradeoffs.
9. If the user performs consistently well across answered questions, especially with strong business-closure understanding, state-node awareness, and tradeoff reasoning, you may recommend that the user exit Inspection Mode early instead of continuing to ask more questions. Phrase this as a recommendation, not a forced exit.

## Feedback Rules

For each answered question, immediately provide deep feedback in chat:

- `评分`: score strictly but politely on a 1-10 scale.
- `评价（足和不足）`: state what the answer got right and what hidden boundary, state risk, coupling issue, protocol risk, or tradeoff it missed.
- `参考答案`: provide a senior-architect-level reference answer grounded in this codebase, not a generic textbook answer.
- If the score is below 8, still show the score, strengths, weaknesses, and reference answer to the user in chat, but treat the user's answer and the strengths/weaknesses as chat-only material for persistence purposes.

## Persistence Rules

1. Persist only answered, non-skipped questions. Skipped questions are not persisted.
2. Incrementally append the new record to the end of the current document `doc/project/PyyyyMMdd(<项目或模块名>分析).md`.
3. Apply score-based persistence:
   - If the question score is 8 or above, persist the full record: question, user answer, score, strengths/weaknesses evaluation, and reference answer.
   - If the question score is below 8, persist only the question heading/title and the reference answer. Do not write the user's answer, score, or strengths/weaknesses evaluation into the document; those remain chat-only feedback.
4. A document may contain only one top-level `# 考察模式` section.
5. If Inspection Mode is entered more than once, append new `## 问题 N` entries inside the existing `# 考察模式` section instead of creating a second top-level heading.
6. Do not rewrite, reorganize, polish, or alter the already finalized `阶段一` through `阶段三` architecture-analysis content when appending Inspection Mode records.
7. If the current analysis document has not been created yet, ask the questions in chat first and persist them once the document path is known. Do not invent a document path that violates the main skill's Document Path rules.

## Exit Rule

When the round ends or the user exits by saying `退出考察` / `退出考察模式`, first provide a user-facing overall review. This overall review is chat-only and must not be written to the analysis document.

The overall review must include:

1. `总评分`: score the user's Inspection Mode performance out of 100 points, based on the answers actually given so far. Skipped questions are not scored; mention them only as coverage notes if relevant.
2. `总体评价`: summarize the user's grasp of business closure, module decomposition, state nodes, concurrency/consistency risks, and design tradeoffs.
3. `理解较强的地方`: identify what the user understood well.
4. `可能薄弱的地方`: identify likely weak areas, blind spots, or concepts that need another pass.
5. `建议`: give practical next steps, such as revisiting specific Stage 2 state nodes, re-reading one Stage 3 scenario, or practicing another question round.

After the overall review, say:

```text
考察模式已结束，正在返回正常阶段流程...
```

Then resume the prior analysis context and clearly state which normal stage or follow-up context has been restored.

## Document Append Format

Use this format at the end of the current analysis document. Generate exactly one record format per answered question according to the score-based persistence rule.

```markdown
# 考察模式

<!-- 当该题评分 >= 8 时，使用完整记录格式 -->
## 问题 1：[问题题目]
### 用户回答
[填入用户的实际回答]
### 评分
[X / 10 分]
### 评价（足和不足）
- **足（亮点）：**
- **不足（盲点）：**
### 参考答案
[结合项目实际代码与架构的高质量参考答案]

---

<!-- 当该题评分 < 8 时，使用精简记录格式；不要写用户回答、评分、足和不足 -->
## 问题 2：[问题题目]
### 参考答案
[结合项目实际代码与架构的高质量参考答案]

---
## 问题 3：[问题题目]
... (按分数选择完整记录或精简记录格式，以此类推)
```

## Prompt Template Snippet

```text
【考察模式（可选切换）】
1. 无论在任何时候，只要我输入“进入考察模式”“你考我一下”“你问我一下”，或者在阶段三完全结束后由你主动邀请并获得我确认，系统应立刻暂停当前的常规文档输出，切入“考察面试状态”。
2. 考察模式的硬性要求：
   - 一轮考察总共提出 5 个关于该项目的硬核技术问题。考察范围是整个项目的核心逻辑和核心原理，不考局部代码记忆、语法细节或 API 名字。
   - 问题要围绕业务目标、系统边界、核心闭环、模块协作、对象/数据流、状态流转、关键状态节点、异步/外部/设备/客户端接入链路和设计取舍展开。
   - 切入考察模式后的首轮回复必须只输出第 1 个问题，然后立刻 Stop and wait，等待我回答；严禁在同一轮内自行模拟我的回答，严禁一次性把后续问题、评分格式、评价模板或参考答案都打出来。
   - 只有收到我的回答后，你才能针对该题给出评分、评价和参考答案，然后再抛出下一个单独的问题，并再次停止等待。
   - 如果我回复“跳过”“跳过这题”或 `skip`，你必须跳过当前问题：不评分、不复盘、不写参考答案、不把该题作为已完成反馈记录写入文档，直接提出下一道单独的问题并再次停止等待。
   - 我每回答一个问题，你都要针对该问题进行即时反馈：给出 1-10 的评分，详细拆解我回答的“足（亮点）”与“不足（盲点/技术风险漏掉的点）”，并附带一份基于当前代码的高质量“参考答案”。如果该题低于 8 分，评分、用户回答、足和不足只在聊天中输出，文档里只保留该题题目和参考答案。
   - 问题维度必须覆盖核心业务闭环、模块间依赖耦合、关键状态节点的并发/一致性隐患、选型折中（Trade-offs）。
   - 将已完成的新一轮考察记录严格按照分数持久化规则增量追加到当前分析文档的末尾（`# 考察模式` 章节）：8 分及以上写【问题、用户回答、评分、评价、参考答案】；8 分以下只写【问题、参考答案】。整篇文档只能存在一个 `# 考察模式`，二次进入时在原章节继续追加。
   - 追加考察记录时，只能增量追加新问题与反馈；请勿重写、重排、润色或篡改文档前半部分的【阶段一】至【阶段三】已定稿系统架构分析内容。
3. 如果我在考察模式中一直表现比较优异，你可以建议我退出考察模式，但不能强制退出。
4. 考察结束或我输入“退出考察”“退出考察模式”后，你必须先给出一个满分 100 分的总评，只输出给我看，不写入文档；总评要指出我理解较强的地方、可能薄弱的地方和后续建议。然后提示“考察模式已结束，正在返回正常阶段流程...”，并清晰说明当前回到了哪个正常阶段或分析上下文。
```
