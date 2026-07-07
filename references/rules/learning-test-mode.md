# 学习测试模式（Learning Test Mode）

Use this file only when the user triggers the learning skill's test mode by saying `测试模式`, `学习SKILL测试模式`, `学习测试模式`, `测一下我刚学的`, or when the main `SKILL.md` `Ambiguous Trigger Arbitration Rule` routes `你考我一下` / `你问我一下` to a `docs/learn/` learning scope.

For `你考我一下` / `你问我一下`, never self-select this file directly. Always obey the main `SKILL.md` arbitration rule first, because project inspection mode uses the same ambiguous trigger words.

### Read-Only Sandbox Assertion

Learning Test Mode is an absolute read-only sandbox while the current test round is active. Until the user exits the current test round, do not call any tool that creates, rewrites, patches, modifies, formats, moves, deletes, stages, commits, or otherwise changes physical files.

This read-only assertion outranks normal Problem Learning Coach behavior, including requests about bugs, fixes, refactors, demos, scripts, tests, configs, or documentation edits. If the user pastes code and asks whether a bug or reference answer should be fixed, discuss the issue in chat only. Do not modify source code, config, tests, scripts, project docs, learning docs, demo files, or any other file while the mode is still active.

If the user wants a real file edit, ask them to exit test mode first. After the mode has ended, treat the edit as a separate implementation request under the normal tool and validation rules.

## Goal

Run a focused learning check based on what the user has just learned through the normal Problem Learning Coach workflow. The scope is the current learning topic, the latest learning document, or the specific concept/demo the user just studied. Do not expand the scope to the whole project unless the user explicitly switches to `考察模式` or `项目模式`.

This mode tests whether the user understands:

- 当前学习文档的一句话主题
- 问题在项目中的真实背景
- 关键数据流、调用链或状态变化
- 核心代码片段为什么这样写
- 易踩坑和当前项目已经做对的地方
- 潜在不足、边界和演进方向
- 最小化 demo 与原项目之间的对应关系
- demo 里哪些地方可以参考，哪些地方不能直接照抄

## Scope Selection

1. If the conversation clearly points to a current `docs/learn/LyyyyMMdd(...).md` document, use that document as the primary source.
2. If multiple learning documents may match, inspect `docs/learn/` and choose the most recent document that matches the user's current topic.
3. If no learning document exists, test only the learning content already discussed in chat. Do not invent a `docs/learn/` path.
4. If the learning target is ambiguous, ask one concise clarification question before asking test questions.
5. Do not read or use `doc/project/` project-analysis documents unless the user explicitly says `考察模式` or asks to test project-mode analysis.

## Interaction Rules

1. A learning-test round contains exactly 5 questions.
2. On entering test mode, output only question 1, then stop and wait for the user's answer.
3. Do not simulate the user answer.
4. Do not output scoring, evaluation, reference-answer placeholders, or later questions in the same turn as the first question.
5. Only after receiving the user answer may you score it, provide feedback/reference answer, then ask the next single question and stop again.
6. If the user replies `跳过`, `跳过这题`, `skip`, or an equivalent request, skip that question immediately: do not score it, do not provide a reference answer, and ask the next single question.
7. Questions should be beginner-friendly but not superficial. Prefer "explain why/data flow/boundary" questions over syntax recall.
8. The 5 questions should jointly cover the learning document's main idea, core flow, code reason, pitfall/boundary, and demo-to-project mapping.
9. If the user interrupts the round with a bug/fix/code-change request, stay in read-only test mode. Give only chat-level reasoning or say that file edits require exiting test mode first; do not switch modes or touch files implicitly.

## Feedback Rules

For each answered question, immediately provide:

- `评分`: score on a 1-10 scale.
- `评价（足和不足）`: explain what the user understood and what they missed.
- `参考答案`: provide a clear beginner-friendly answer grounded in the learning document or current learning topic.

If the answer is weak, keep feedback direct and teach the missing mental model. Do not shame the user.

## Persistence Rules

1. During the active test round, do not persist anything to disk. Keep answered question records in chat-visible content only.
2. Default persistence after exit must not modify the source learning document. Save test records to a separate Markdown file under:

```text
docs/learn/tests/[T]yyyyMMdd-XXXX-测试记录.md
```

3. Use the user's local date for `yyyyMMdd`. Derive `XXXX` from the source learning document topic or current learning topic, such as `Kafka原始数据链路`, `8082NMEA切行`, or `RTCM帧解析`. If the source file is `docs/learn/L20260707(学习Kafka原始数据链路).md`, a matching test record path can be `docs/learn/tests/[T]20260707-Kafka原始数据链路-测试记录.md`.
4. If the same test record file already exists for the same date/topic, append a new round section to that test record file. Do not create duplicate same-day same-topic files unless the user explicitly asks for separate files.
5. In the separate test record file, include a source reference back to the learning document path when one exists. If no learning document exists, use the chat topic as the source reference and state that no source learning document was found.
6. Only append test records to the original learning document if the user explicitly asks to keep test records inside the learning document. In that case, wrap the entire test area in a Markdown `<details>` block so the learning document remains clean for knowledge-base sharing.
7. Never append raw scoring and personal answers directly to the end of a learning document without a `<details>` wrapper.
8. Never modify source code, config, tests, scripts, project analysis documents, demo files, or any non-learning-document file from Learning Test Mode.
9. Do not rewrite, reorder, or polish existing learning sections when persisting test records after exit.

## Separate Test Record File Format

Use this format in `docs/learn/tests/[T]yyyyMMdd-XXXX-测试记录.md`. Append each new test round as a new `## 测试轮次 N` section.

```markdown
# XXXX - 测试记录

> 来源学习文档：docs/learn/LyyyyMMdd(XXXX).md
> 测试日期：yyyy-MM-dd
> 记录说明：本文件保存学员测试过程，避免污染可分享的学习正文。

## 测试轮次 1

### 问题 1：[问题题目]
#### 用户回答
[填入用户的实际回答]
#### 评分
[X / 10 分]
#### 评价（足和不足）
- **足（亮点）：**
- **不足（盲点）：**
#### 参考答案
[基于当前学习文档或学习主题的参考答案]

---
```

## Optional Folded Append Format

```markdown
<details>
<summary>点击展开学员测试与通关记录</summary>

# 测试模式

## 问题 1：[问题题目]
### 用户回答
[填入用户的实际回答]
### 评分
[X / 10 分]
### 评价（足和不足）
- **足（亮点）：**
- **不足（盲点）：**
### 参考答案
[基于当前学习文档或学习主题的参考答案]

---

</details>
```

## Exit Rule

When the round ends or the user exits by saying `退出测试`, `退出测试模式`, or an equivalent phrase, provide a chat-only summary:

1. `总评分`: score the answered questions out of 100.
2. `掌握较好的地方`: name the concepts the user understands.
3. `需要回看`: name the exact learning document sections, code snippets, diagrams, or demo parts to review.
4. `下一步建议`: recommend whether to reread, continue another test round, or return to normal learning.

Then say:

```text
测试模式已结束，正在返回学习流程...
```
