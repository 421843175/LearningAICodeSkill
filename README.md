# Problem Learning Coach

`problem-learning-coach` 是一个通用学习型 Codex Skill。

它不是 WebSocket 专用，也不是某一个项目专用。它的目标是：当你遇到任何技术问题、代码看不懂、架构不理解、Bug 不知道为什么、AI 写了一段代码你想学明白时，让 Codex 按“应届生也能听懂”的方式带你学习。

## 适用场景

你可以在这些场景使用它：

- 看不懂一段 Java、前端、后端、配置或脚本代码
- 想理解 MQTT、Kafka、Redis、HTTP、WebSocket、数据库、定时任务等技术
- 想知道某个 Bug 为什么发生，以及应该怎么修
- 想让 AI 不只是修代码，而是讲清楚背后的设计
- 想生成一份学习文档
- 想要一个可以新建工程复制粘贴运行的最小化 demo

## 它会怎么回答

使用这个 Skill 时，Codex 会尽量按下面的方式讲：

1. 先用人话说明问题是什么
2. 识别当前技术主题，比如 MQTT、Kafka、WebSocket、数据库等
3. 如果有项目代码，就先读项目代码和配置
4. 如果缺关键信息，会先问你，比如 Java 版本、框架版本、Broker 或数据库选择
5. 画架构图或流程图
6. 拆核心代码，并加中文注释
7. 讲当前设计的好处和不足
8. 给一份最小化学习 demo
9. demo 按真实开发顺序写，每个代码块上方标明文件路径或运行位置

## 安装方式

把整个目录复制到你的 Codex skills 目录：

```text
~/.codex/skills/problem-learning-coach
```

Windows 一般是：

```text
C:/Users/<你的用户名>/.codex/skills/problem-learning-coach
```

目录结构应该类似这样：

```text
problem-learning-coach
  ├─ SKILL.md
  ├─ README.md
  └─ references
     └─ websocket-realtime-learning-example.md
```

## 使用方式

你可以这样说：

```text
用 problem-learning-coach 给我讲一下这段 MQTT 代码，我看不懂
```

也可以这样说：

```text
这个 Kafka 消费者为什么这么写？用学习模式讲给我
```

或者：

```text
不要只修 Bug，用 problem-learning-coach 带我学明白原因，并给一个最小 demo
```

再比如：

```text
这段 Redis 缓存代码我看不懂，按应届生能听懂的方式解释
```

## 最小化 demo 的要求

这个 Skill 生成 demo 时，默认会按“可以新建工程复制粘贴运行”的方式来写。

也就是说，它会尽量按开发顺序组织：

```text
1. 新建工程目录
2. 写构建文件，比如 pom.xml 或 package.json
3. 写配置文件
4. 写启动类或入口文件
5. 写核心模型、配置类、工具类、服务类
6. 写接口或客户端
7. 写测试页面或测试命令
8. 写运行方式和预期输出
```

每个代码块上方都会写清楚：

```text
所属路径：xxx/xxx/File.java
```

或者：

```text
所属位置：命令行
```

这样你复制的时候不用猜这个文件应该建在哪里。

## Reference 案例

`references/websocket-realtime-learning-example.md` 是一个完整学习文档案例。

它只是参考案例，不会默认读取，避免浪费 token。

Codex 只有在这些情况才应该读取它：

- 你要求生成完整学习文档
- 你要求参考之前的学习文档风格
- 你问 WebSocket 或实时流式输出相关问题
- `SKILL.md` 里的规则不够，需要看一个完整例子

注意：这个 reference 是 WebSocket 案例，但 Skill 本身是通用的。问 MQTT 就按 MQTT 讲，问 Kafka 就按 Kafka 讲，不能把所有问题都套成 WebSocket。

## 推荐提示词

```text
用 problem-learning-coach 讲一下这个问题，我要知道为什么，不只是要答案。
```

```text
这段代码我看不懂，先结合项目讲流程，再给一个可以复制运行的最小 demo。
```

```text
如果你需要 Java 版本、框架版本、运行环境这些信息，先问我。
```

## 适合谁

这个 Skill 很适合：

- 应届生
- 刚接手项目的人
- 想从 AI 生成代码里学东西的人
- 不想只拿答案、还想知道原理的人

**没有用AI直接生成最小化项目而是在文档中体现 是因为生成文档在复制的过程中看代码是学习的关键要素**

## 维护说明

核心规则写在：

```text
SKILL.md
```

完整参考案例放在：

```text
references/websocket-realtime-learning-example.md
```

如果以后你有新的优秀学习文档案例，可以继续放到 `references/`，并在 `SKILL.md` 里写清楚什么时候才允许读取，避免默认浪费上下文。

