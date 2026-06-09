# RTCM WebSocket 实时流式输出学习文档

> 适用范围：当前 `rtk-parser-app` 的 RTCM 解析实时 JSON WebSocket 输出。  
> 目标读者：刚接触后端实时推送、WebSocket、限流批量推送的应届生。  
> 阅读目标：你看完后，要能说清楚“这个项目为什么要用 WebSocket、代码怎么串起来、前端怎么接、哪里可能有坑、自己怎么写一个最小 demo”。

## 1. 先用一句话讲清楚

这个项目里的 WebSocket，就是把原来后端控制台打印的 RTCM 卫星解析信息，换成前端也能实时收到的 JSON 消息。

以前的模式是：

```text
RTCM 数据进来 -> 后端解析 -> 控制台 log.info 打中文日志 -> 人眼看控制台
```

现在的模式是：

```text
RTCM 数据进来 -> 后端解析 -> 生成结构化 JSON -> WebSocket 推给前端 -> 页面实时展示
```

再加上这次做的限流以后，模式变成：

```text
RTCM 数据进来 -> 后端解析 -> 生成结构化 JSON -> 先进 WebSocket 队列 -> 每隔 1 秒合成一批 -> 前端收到 rtcm.batch
```

这样做的核心原因是：RTCM 解析速度可能非常快，如果后端解析到一条就通过 WebSocket 发一条，前端会被刷爆，本机 CPU 和风扇也会跟着起飞。

## 2. 项目是怎么设计 WebSocket 的

### 2.1 这个 WebSocket 解决什么问题

当前项目里，`rtk-parser-app` 会做这些事：

```text
接收 RTCM 原始字节
解析 RTCM frame
解码 MSM4 观测数据
解码 NAV 导航星历
写 RINEX OBS/NAV 文件
输出实时状态
```

其中“输出实时状态”以前主要靠控制台日志。

但是前端不能直接消费控制台日志，所以我们补了一条 WebSocket 通道：

```text
ws://localhost:18081/api/rtcm/realtime/ws
```

前端连接这个地址以后，可以实时收到 JSON。

### 2.2 这块代码分成几层

当前实现大概分成 5 层：

| 层级 | 代码 | 作用 |
| --- | --- | --- |
| 配置层 | `application.yml` | 控制哪些事件推送、是否限流、多久推一批、队列多大 |
| 属性绑定层 | `RealtimeOutputProperties` | 把 YAML 配置绑定成 Java 对象 |
| WebSocket 注册层 | `RtcmRealtimeWebSocketConfig` | 注册 `/api/rtcm/realtime/ws` 这个 WebSocket 地址 |
| 连接处理层 | `JsonWebSocketStreamHandler` | 处理连接建立、关闭、ping/pong |
| 推送工具层 | `JsonWebSocketStreamer` | 静态工具类，业务代码调用 `stream(...)` 就能推 JSON |
| 业务输出层 | `RtcmRealtimeAnalyzer` / `RtcmProductWriter` | RTCM 解析完成后，把卫星信息交给 WebSocket 工具类 |

你可以把它理解成一个“广播喇叭”：

```text
业务代码只负责喊一句：
JsonWebSocketStreamer.stream("rtcm.observation.detail", sessionId, sourceName, payload)

WebSocket 工具类负责：
1. 看有没有前端连接
2. 看这个事件是否要限流
3. 要限流就先进队列
4. 定时合成 rtcm.batch
5. 给所有在线前端连接发送 JSON
```

业务代码不用关心 WebSocketSession、连接锁、JSON 序列化、队列、定时器这些细节。

这就是把 WebSocket 做成工具类的好处。

## 3. 架构流程是怎么样的

### 3.1 总体流程

```text
前端页面
  |
  | 1. 建立 WebSocket 连接
  v
ws://localhost:18081/api/rtcm/realtime/ws
  |
  | 2. Spring 把连接交给 JsonWebSocketStreamHandler
  v
JsonWebSocketStreamHandler.afterConnectionEstablished(...)
  |
  | 3. 注册连接
  v
JsonWebSocketStreamer.register(session)
  |
  | 4. 后端立即返回 websocket.connected
  v
前端知道连接成功
```

然后 RTCM 解析链路开始推数据：

```text
RTCM 原始数据
  |
  v
RtcmRealtimeAnalyzer.accept(...)
  |
  v
Msm4Decoder / EphemerisDecoder
  |
  v
RtcmProductWriter.writeObservation(...) / writeNavigationRecord(...)
  |
  v
JsonWebSocketStreamer.stream("rtcm.observation.detail", ...)
  |
  v
进入 WebSocket 批量队列
  |
  v
定时线程每 1000ms flush 一批
  |
  v
前端收到 type = rtcm.batch
```

### 3.2 前端收到什么

连接成功时，前端会收到即时事件：

```json
{
  "type": "websocket.connected",
  "sessionId": null,
  "sourceName": null,
  "emittedAt": "2026-06-08T09:49:00Z",
  "payload": {
    "activeSessions": 1
  }
}
```

RTCM 高频业务事件会被合并成批量包：

```json
{
  "type": "rtcm.batch",
  "sessionId": null,
  "sourceName": null,
  "emittedAt": "2026-06-08T09:49:01Z",
  "payload": {
    "count": 2,
    "dropped": 0,
    "remaining": 0,
    "intervalMillis": 1000,
    "items": [
      {
        "type": "rtcm.observation.detail",
        "sessionId": "session-001",
        "sourceName": "manual",
        "emittedAt": "2026-06-08T09:49:00Z",
        "payload": {
          "title": "RTCM MSM4 观测数据详细字段",
          "text": "这里是原来控制台那段中文解析文本",
          "lines": ["这里是原来控制台那段中文解析文本"],
          "fields": [],
          "observations": []
        }
      },
      {
        "type": "rtcm.navigation.detail",
        "sessionId": "session-001",
        "sourceName": "manual",
        "emittedAt": "2026-06-08T09:49:00Z",
        "payload": {
          "title": "RINEX NAV 导航星历详细字段",
          "text": "这里是原来控制台那段中文星历文本",
          "lines": ["这里是原来控制台那段中文星历文本"],
          "fields": [],
          "parameterLines": []
        }
      }
    ]
  }
}
```

前端写代码时要注意：现在不是只看顶层 `type`，还要遍历 `payload.items`。

伪代码：

```js
socket.onmessage = (event) => {
  const message = JSON.parse(event.data);

  if (message.type === "rtcm.batch") {
    message.payload.items.forEach(handleRtcmEvent);
    return;
  }

  handleRtcmEvent(message);
};
```

## 4. 这个设计有什么好处

### 4.1 好处一：业务代码很干净

业务代码只需要调用：

```java
JsonWebSocketStreamer.stream("rtcm.observation.detail", sessionId, sourceName, payload);
```

它不用管：

- 当前有几个前端连接。
- 连接断了怎么办。
- JSON 怎么序列化。
- 多线程同时发送会不会乱。
- 高频数据要不要限流。
- 队列满了怎么办。

这些都在 `JsonWebSocketStreamer` 里统一处理。

### 4.2 好处二：前端拿到的是 JSON，不是字符串日志

控制台日志适合人看，不适合前端程序处理。

比如一段中文日志：

```text
卫星ID=G03 伪距=22123123.123m 载噪比=45.2dB-Hz
```

人能看懂，但前端想画表格、筛选卫星、排序、画曲线会很麻烦。

所以当前设计把原中文文本也保留，同时补充结构化字段：

```json
{
  "text": "卫星ID=G03 伪距=22123123.123m 载噪比=45.2dB-Hz",
  "fields": [
    {
      "label": "卫星ID",
      "key": "satelliteId",
      "value": "G03",
      "displayValue": "G03"
    }
  ]
}
```

这样前端既可以按原日志样式展示，也可以用字段做表格和图表。

### 4.3 好处三：限流保护前端和机器

RTCM 数据流可能很快。

如果每解析一条就 WebSocket 推一条，会产生这些问题：

- 后端频繁 JSON 序列化。
- WebSocket 频繁发送小包。
- 前端频繁触发 `onmessage`。
- 浏览器频繁更新页面。
- CPU 占用上升，风扇一直转。

现在通过批量限流：

```yaml
websocket-batch-enabled: true
websocket-batch-interval-millis: 1000
websocket-batch-max-items: 200
websocket-batch-max-queue-size: 5000
websocket-batch-drop-oldest: true
```

意思是：

```text
最多每 1 秒推 1 个包
每个包最多 200 条事件
队列最多攒 5000 条
满了丢最旧的展示事件，保留最新状态
```

这属于比较常见的企业项目处理方式：核心业务继续跑，展示层限速。

### 4.4 好处四：开关都在 YAML

控制台中文日志、WebSocket 明细、批量限流都在 YAML 控制。

这比写死在代码里好，因为不同环境可以不同配置：

```text
本地调试：可以打开详细中文控制台日志
测试环境：可以打开 WebSocket 明细
生产环境：可以关闭高频日志，保留批量推送和错误事件
```

## 5. 潜在不足和要注意的坑

### 5.1 批量推送不是严格每条都实时

开启批量以后，前端看到的数据会有一点延迟。

默认延迟大概是：

```text
0 到 1000ms
```

这对实时展示通常可以接受，但如果以后做“毫秒级交易系统”或者“强实时控制系统”，就要重新评估。

RTCM 解析展示一般不需要毫秒级逐条刷新，所以这里用 1 秒批量比较合理。

### 5.2 队列满了会丢展示事件

当前默认：

```yaml
websocket-batch-drop-oldest: true
```

意思是队列满了以后，丢最旧的 WebSocket 展示事件。

注意，这里丢的是“前端展示消息”，不是丢 RTCM 原始数据，也不是丢 RINEX 文件写入。

核心解析、落盘还在跑。

前端可以看：

```json
"dropped": 10
```

如果 `dropped > 0`，说明前端看到的展示流不是完整逐条流，而是被限流保护过。

### 5.3 静态工具类简单好用，但也有边界

`JsonWebSocketStreamer` 做成静态工具类，好处是调用非常方便。

但是静态工具类也有潜在不足：

- 单元测试时全局状态要注意清理。
- 多个 Spring ApplicationContext 共存时可能互相影响。
- 如果以后要做多租户隔离，静态全局集合可能不够细。
- 如果以后要把 WebSocket 节点横向扩容到多台机器，需要引入 Redis/Kafka 等跨节点广播。

当前项目是单个 parser 应用内做实时展示，这个设计是够用的。

### 5.4 多台 parser 时不能天然互通

现在 WebSocket 连接在哪台 parser，哪台 parser 就推自己的事件。

如果以后部署多台 parser，前端连到了 A 节点，但 RTCM 数据被 B 节点解析，那前端可能收不到 B 的消息。

这时候可以考虑：

```text
RTCM 解析节点 -> Kafka/Redis 发布实时事件 -> WebSocket 网关统一订阅 -> 前端只连 WebSocket 网关
```

这就是更大型系统常见的“实时事件网关”设计。

## 6. 类似设计推荐

### 6.1 当前设计：静态工具类 + Spring WebSocket

适合：

- 单体应用或单个 parser 节点。
- 后端业务代码想简单调用。
- 前端需要实时 JSON。
- 数据量中等，但需要限流。

特点：

```text
简单
代码少
容易理解
适合当前项目
```

### 6.2 推荐设计一：Spring WebSocket + STOMP

STOMP 可以理解成 WebSocket 上的一层消息协议。

它适合：

- 前端要订阅不同 topic。
- 后端要按 topic 广播。
- 需要更标准的订阅模型。

例子：

```text
/topic/rtcm/observation
/topic/rtcm/navigation
/topic/rtcm/error
```

好处是 topic 模型清晰。

不足是比当前原生 WebSocket 复杂。

### 6.3 推荐设计二：SSE

SSE 全称 Server-Sent Events。

它适合：

- 只需要后端推给前端。
- 前端不需要频繁发消息给后端。
- 希望实现比 WebSocket 简单。

RTCM 实时展示其实也可以用 SSE。

但是如果以后前端要发 ping、订阅命令、切换会话，WebSocket 更灵活。

### 6.4 推荐设计三：Kafka/Redis + WebSocket 网关

这适合更大型的企业部署：

```text
parser-app 只负责解析
parser-app 把实时事件发布到 Kafka/Redis
websocket-gateway 只负责维护前端连接
websocket-gateway 订阅 Kafka/Redis 后广播给前端
```

好处：

- parser 和 WebSocket 解耦。
- 多台 parser 可以统一推送。
- WebSocket 网关可以独立扩容。
- 更适合生产集群。

不足：

- 架构复杂。
- 运维成本更高。
- 本项目当前阶段可能有点过度设计。

## 7. 核心代码以及注释

### 7.1 YAML 配置

文件：

```text
rtk-parser-app/src/main/resources/application.yml
```

核心配置：

```yaml
rtk:
  realtime:
    output:
      # 是否开启 WebSocket 批量限流。
      websocket-batch-enabled: true

      # 每隔多少毫秒推送一次批量包。
      websocket-batch-interval-millis: 1000

      # 每个批量包最多放多少条原始事件。
      websocket-batch-max-items: 200

      # 队列最多缓存多少条待推送事件。
      websocket-batch-max-queue-size: 5000

      # 队列满时是否丢弃最旧事件。
      websocket-batch-drop-oldest: true

      # 是否把原控制台 MSM4 观测明细转成 WebSocket JSON。
      websocket-detailed-observation-enabled: true

      # 是否把原控制台 NAV 星历明细转成 WebSocket JSON。
      websocket-detailed-navigation-enabled: true

      # 是否打印控制台中文观测明细。
      detailed-observation-log-enabled: false

      # 是否打印控制台中文星历明细。
      detailed-navigation-log-enabled: false
```

### 7.2 配置绑定类

文件：

```text
rtk-parser-app/src/main/java/com/pnt/rtk/config/RealtimeOutputProperties.java
```

核心思想：

```java
@ConfigurationProperties(prefix = "rtk.realtime.output")
public class RealtimeOutputProperties {

    // 控制是否开启 WebSocket 批量限流，默认开启。
    private boolean websocketBatchEnabled = true;

    // 控制 WebSocket 批量推送间隔，默认 1000 毫秒。
    private long websocketBatchIntervalMillis = 1000;

    // 控制每个批量包最多包含多少条原始事件。
    private int websocketBatchMaxItems = 200;

    // 控制队列最多缓存多少条待推送事件。
    private int websocketBatchMaxQueueSize = 5000;

    // 控制队列满了以后是否丢弃最旧事件。
    private boolean websocketBatchDropOldest = true;
}
```

应届生理解方式：

```text
YAML 是配置文件。
RealtimeOutputProperties 是 Java 里的配置对象。
Spring Boot 启动时会把 YAML 里的值塞进这个对象。
业务代码不要直接读 YAML，读这个对象就行。
```

### 7.3 WebSocket 注册类

文件：

```text
rtk-parser-app/src/main/java/com/pnt/rtk/websocket/RtcmRealtimeWebSocketConfig.java
```

核心代码：

```java
@Configuration
@EnableWebSocket
public class RtcmRealtimeWebSocketConfig implements WebSocketConfigurer {

    // Spring Boot 自动配置好的 JSON 序列化器。
    private final ObjectMapper objectMapper;

    // YAML 绑定出来的实时输出配置。
    private final RealtimeOutputProperties outputProperties;

    public RtcmRealtimeWebSocketConfig(ObjectMapper objectMapper,
                                       RealtimeOutputProperties outputProperties) {
        // 保存 ObjectMapper，后面交给静态工具类。
        this.objectMapper = objectMapper;
        // 保存限流配置，后面交给静态工具类。
        this.outputProperties = outputProperties;
    }

    @PostConstruct
    public void configureStreamer() {
        // 应用启动后，把 JSON 配置和限流配置交给 WebSocket 工具类。
        JsonWebSocketStreamer.configure(objectMapper, outputProperties);
    }

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        // 注册前端可以连接的 WebSocket 地址。
        registry.addHandler(new JsonWebSocketStreamHandler(), "/api/rtcm/realtime/ws")
                // 允许前端不同端口连接，方便本地前后端分离调试。
                .setAllowedOriginPatterns("*");
    }
}
```

应届生理解方式：

```text
这个类就像“开门的人”。
它告诉 Spring：/api/rtcm/realtime/ws 这个地址不是普通 HTTP，而是 WebSocket。
前端连接上来以后，交给 JsonWebSocketStreamHandler 处理。
```

### 7.4 WebSocket 连接处理类

文件：

```text
rtk-parser-app/src/main/java/com/pnt/rtk/websocket/JsonWebSocketStreamHandler.java
```

核心代码：

```java
public class JsonWebSocketStreamHandler extends TextWebSocketHandler {

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        // 前端连接成功后，把这个连接注册到工具类里。
        JsonWebSocketStreamer.register(session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        // 前端发 ping，后端回 pong，用来判断连接还活着。
        if ("ping".equalsIgnoreCase(message.getPayload())) {
            JsonWebSocketStreamer.streamTo(session, "websocket.pong", Map.of("session", session.getId()));
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        // 前端断开后，清理连接，避免后端继续给失效连接发消息。
        JsonWebSocketStreamer.unregister(session);
    }
}
```

应届生理解方式：

```text
连接建立：放进在线连接池。
收到 ping：回一个 pong。
连接关闭：从在线连接池移除。
```

### 7.5 WebSocket 静态工具类

文件：

```text
rtk-parser-app/src/main/java/com/pnt/rtk/websocket/JsonWebSocketStreamer.java
```

这个类是当前设计的核心。

它主要做 6 件事：

```text
1. 保存所有在线 WebSocket 连接。
2. 提供静态 stream(...) 方法给业务代码调用。
3. 把 Java 对象序列化成 JSON。
4. 给所有在线连接广播。
5. 对高频 rtcm.* 事件做批量限流。
6. 队列满时丢弃展示事件，保护内存。
```

简化后的核心代码：

```java
public final class JsonWebSocketStreamer {

    // 保存当前在线的 WebSocket 连接。
    private static final Set<WebSocketSession> SESSIONS = ConcurrentHashMap.newKeySet();

    // 保存待批量推送的 RTCM 事件。
    private static final ArrayDeque<JsonWebSocketMessage> BATCH_QUEUE = new ArrayDeque<>();

    // 保护批量队列的锁，避免多线程同时操作队列出问题。
    private static final Object BATCH_LOCK = new Object();

    // 记录因为队列满而丢弃了多少展示事件。
    private static final AtomicLong DROPPED_BATCH_MESSAGES = new AtomicLong();

    // 单线程定时器，负责定时把队列里的事件合成一批推给前端。
    private static final ScheduledExecutorService BATCH_EXECUTOR =
            Executors.newSingleThreadScheduledExecutor();

    // 是否开启批量限流。
    private static volatile boolean batchEnabled = true;

    // 批量推送间隔，默认 1000ms。
    private static volatile long batchIntervalMillis = 1000;

    // 每个批量包最多多少条事件。
    private static volatile int batchMaxItems = 200;

    public static void stream(String type, String sessionId, String sourceName, Object payload) {
        // 没有前端连接时直接返回，避免无意义的 JSON 序列化。
        if (SESSIONS.isEmpty()) {
            return;
        }

        // 把业务数据包装成统一消息格式。
        JsonWebSocketMessage message =
                new JsonWebSocketMessage(type, sessionId, sourceName, Instant.now(), payload);

        // 如果是高频 RTCM 事件，就进入批量队列。
        if (shouldBatch(type)) {
            enqueueBatch(message);
            return;
        }

        // 低频事件直接广播。
        streamToAll(message);
    }
}
```

### 7.6 为什么只批量 `rtcm.*`

当前判断逻辑大概是：

```java
private static boolean shouldBatch(String type) {
    // 关闭限流时，不批量。
    if (!batchEnabled) {
        return false;
    }

    // 空类型不批量。
    if (type == null) {
        return false;
    }

    // 解码失败是关键错误，立即推给前端。
    if ("rtcm.decode.failed".equals(type)) {
        return false;
    }

    // 只有 RTCM 业务事件才批量。
    return type.startsWith("rtcm.");
}
```

为什么这样设计？

```text
websocket.connected 要立即发，不然前端不知道连接是否成功。
websocket.pong 要立即发，不然心跳检测不准。
rtcm.decode.failed 要立即发，不然错误提示会慢。
rtcm.observation.detail 这种高频展示事件可以批量。
```

### 7.7 业务代码怎么推送

文件：

```text
rtk-parser-app/src/main/java/com/pnt/rtk/product/RtcmProductWriter.java
```

观测明细推送：

```java
private void streamDetailedObservationSummary(RtcmSession session, Msm4Summary summary) {
    // 把原控制台中文观测详情变成 JSON，通过 WebSocket 发给前端。
    JsonWebSocketStreamer.stream(
            "rtcm.observation.detail",
            session.sessionId(),
            session.sourceName(),
            detailedObservationPayload(session, summary)
    );
}
```

导航星历推送：

```java
private void streamDetailedNavigationRecord(RtcmSession session, RinexNavRecord record, Path navFile) {
    // 把原控制台中文导航星历详情变成 JSON，通过 WebSocket 发给前端。
    JsonWebSocketStreamer.stream(
            "rtcm.navigation.detail",
            session.sessionId(),
            session.sourceName(),
            detailedNavigationPayload(record, navFile)
    );
}
```

应届生理解方式：

```text
业务代码只负责告诉 WebSocket：
我这里有一条 observation/detail 或 navigation/detail。

至于这条消息是立即发，还是先进队列，业务代码不用管。
```

## 8. 接口说明

### 8.1 WebSocket 实时接口

```text
ws://localhost:18081/api/rtcm/realtime/ws
```

用途：

```text
前端连接后，实时接收 RTCM 解析事件。
```

连接成功事件：

```json
{
  "type": "websocket.connected",
  "payload": {
    "activeSessions": 1
  }
}
```

高频业务事件：

```json
{
  "type": "rtcm.batch",
  "payload": {
    "count": 200,
    "dropped": 0,
    "remaining": 0,
    "intervalMillis": 1000,
    "items": []
  }
}
```

### 8.2 WebSocket 状态接口

```text
GET http://localhost:18081/api/rtcm/realtime/ws/status
```

用途：

```text
查看当前 WebSocket 连接数和批量限流状态。
```

返回示例：

```json
{
  "path": "/api/rtcm/realtime/ws",
  "activeSessions": 1,
  "batch": {
    "enabled": true,
    "intervalMillis": 1000,
    "maxItems": 200,
    "maxQueueSize": 5000,
    "dropOldest": true,
    "queueSize": 0,
    "droppedSinceLastBatch": 0
  }
}
```

### 8.3 手动测试消息接口

```text
POST http://localhost:18081/api/rtcm/realtime/ws/messages
Content-Type: application/json
```

请求示例：

```json
{
  "type": "manual.test",
  "sessionId": "demo-session",
  "sourceName": "apifox",
  "payload": {
    "message": "hello websocket"
  }
}
```

用途：

```text
不用真的喂 RTCM 数据，也能测试 WebSocket 是否能推消息。
```

注意：

```text
manual.test 不是 rtcm.*，所以它不会进入批量限流，会立即推给前端。
```

## 9. 最小化学习 demo

这个 demo 是一个可以新建工程后复制粘贴运行的最小 Spring Boot WebSocket 项目。因为它是基于当前 `rtk` 工程的 WebSocket 设计来学习，所以它不使用随意的 `com.example.websocketdemo` 分包，而是刻意模仿当前工程的模块和包设计：

所属位置：学习说明，不需要创建文件。

```text
1. 保留 Maven parent + app module 的结构，像当前 rtk-parent + rtk-parser-app。
2. 启动类放 parserapp 包，像当前 RtkParserApplication。
3. 配置类放 config 包，像当前 RealtimeOutputProperties。
4. REST 测试接口放 api 包，像当前 RtcmRealtimeController。
5. WebSocket 连接、配置、推送工具放 websocket 包，像当前 JsonWebSocketStreamer 这一组类。
6. YAML 使用分层配置，像当前 rtk.realtime.output。
```

### 9.1 第一步：按当前工程风格新建 demo 目录

所属位置：命令行，不需要创建项目文件。

```text
rtk-websocket-learning-demo
  ├─ pom.xml
  ├─ rtk-parser-app
  │  ├─ pom.xml
  │  └─ src
  │     └─ main
  │        ├─ java
  │        │  └─ com
  │        │     └─ pnt
  │        │        └─ rtk
  │        │           ├─ api
  │        │           │  └─ DemoRealtimeController.java
  │        │           ├─ config
  │        │           │  └─ DemoRealtimeOutputProperties.java
  │        │           ├─ parserapp
  │        │           │  └─ RtkParserLearningApplication.java
  │        │           └─ websocket
  │        │              ├─ DemoJsonWebSocketMessage.java
  │        │              ├─ DemoJsonWebSocketStreamHandler.java
  │        │              ├─ DemoJsonWebSocketStreamer.java
  │        │              └─ DemoRealtimeWebSocketConfig.java
  │        └─ resources
  │           └─ application.yml
  └─ demo-websocket.html
```

这套结构是故意贴着当前项目来的。你练完这个 demo，再回到当前项目看 `rtk-parser-app/src/main/java/com/pnt/rtk/websocket`，会更容易对应上。

### 9.2 第二步：写父工程 pom

所属路径：`rtk-websocket-learning-demo/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!--
      这里沿用当前工程的 Spring Boot parent 风格。
      这样 demo 的依赖管理方式和 rtk 工程接近。
    -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.12</version>
        <relativePath/>
    </parent>

    <groupId>com.pnt</groupId>
    <artifactId>rtk-websocket-learning-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <!--
      当前真实工程是多模块结构。
      demo 虽然只需要 parser app，但仍然保留 module 思维，方便你适应当前项目。
    -->
    <modules>
        <module>rtk-parser-app</module>
    </modules>

    <properties>
        <!-- 当前 rtk 工程使用 Java 17，所以学习 demo 也按 Java 17 写。 -->
        <java.version>17</java.version>
    </properties>
</project>
```

### 9.3 第三步：写 parser app 模块 pom

所属路径：`rtk-websocket-learning-demo/rtk-parser-app/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!--
      子模块继承父工程。
      这和当前 rtk-parser-app 继承 rtk-parent 的方式一致。
    -->
    <parent>
        <groupId>com.pnt</groupId>
        <artifactId>rtk-websocket-learning-demo</artifactId>
        <version>0.0.1-SNAPSHOT</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <artifactId>rtk-parser-app</artifactId>
    <name>rtk-parser-app</name>

    <dependencies>
        <!-- REST 接口需要它，例如 DemoRealtimeController。 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- WebSocket 注册、Handler、TextMessage 都需要它。 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-websocket</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- 用于启动 Spring Boot app。 -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### 9.4 第四步：写 YAML 配置

所属路径：`rtk-websocket-learning-demo/rtk-parser-app/src/main/resources/application.yml`

```yaml
spring:
  application:
    # 命名方式贴近当前 rtk-parser-app。
    name: rtk-parser-app

server:
  # 和当前 parser 应用端口保持一致，方便你迁移理解。
  port: 18081

demo:
  realtime:
    output:
      # 是否开启 WebSocket 批量限流。
      batch-enabled: true

      # 每隔多少毫秒推送一次批量包。
      batch-interval-millis: 1000

      # 每个批量包最多放多少条原始消息。
      batch-max-items: 100
```

### 9.5 第五步：写启动类

所属路径：`rtk-websocket-learning-demo/rtk-parser-app/src/main/java/com/pnt/rtk/parserapp/RtkParserLearningApplication.java`

```java
package com.pnt.rtk.parserapp;

import com.pnt.rtk.config.DemoRealtimeOutputProperties;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.EnableConfigurationProperties;

// 当前真实工程的 RtkParserApplication 使用 scanBasePackages = "com.pnt.rtk"。
// demo 也这么写，是为了让 api/config/websocket/parserapp 这些平级包都能被扫描到。
@SpringBootApplication(scanBasePackages = "com.pnt.rtk")

// 把 YAML 中 demo.realtime.output 绑定到 DemoRealtimeOutputProperties。
@EnableConfigurationProperties(DemoRealtimeOutputProperties.class)
public class RtkParserLearningApplication {

    public static void main(String[] args) {
        // 启动 demo parser app。
        SpringApplication.run(RtkParserLearningApplication.class, args);
    }
}
```

### 9.6 第六步：写配置绑定类

所属路径：`rtk-websocket-learning-demo/rtk-parser-app/src/main/java/com/pnt/rtk/config/DemoRealtimeOutputProperties.java`

```java
package com.pnt.rtk.config;

import org.springframework.boot.context.properties.ConfigurationProperties;

// 当前真实工程把实时输出配置放在 config 包下。
// demo 也放在 config 包，方便你回头对应 RealtimeOutputProperties。
@ConfigurationProperties(prefix = "demo.realtime.output")
public class DemoRealtimeOutputProperties {

    // 是否开启 WebSocket 批量限流。
    private boolean batchEnabled = true;

    // 批量推送间隔，单位毫秒。
    private long batchIntervalMillis = 1000;

    // 每批最多消息数。
    private int batchMaxItems = 100;

    public boolean isBatchEnabled() {
        return batchEnabled;
    }

    public void setBatchEnabled(boolean batchEnabled) {
        this.batchEnabled = batchEnabled;
    }

    public long getBatchIntervalMillis() {
        return batchIntervalMillis;
    }

    public void setBatchIntervalMillis(long batchIntervalMillis) {
        this.batchIntervalMillis = batchIntervalMillis;
    }

    public int getBatchMaxItems() {
        return batchMaxItems;
    }

    public void setBatchMaxItems(int batchMaxItems) {
        this.batchMaxItems = batchMaxItems;
    }
}
```

### 9.7 第七步：写 WebSocket 统一消息结构

所属路径：`rtk-websocket-learning-demo/rtk-parser-app/src/main/java/com/pnt/rtk/websocket/DemoJsonWebSocketMessage.java`

```java
package com.pnt.rtk.websocket;

import java.time.Instant;

// 当前 rtk 工程使用 Java 17，所以 demo 可以使用 record。
// 如果你的目标是 Java 8，需要改成普通 class。
public record DemoJsonWebSocketMessage(
        String type,
        Instant emittedAt,
        Object payload
) {
}
```

### 9.8 第八步：写 WebSocket 推送工具类

所属路径：`rtk-websocket-learning-demo/rtk-parser-app/src/main/java/com/pnt/rtk/websocket/DemoJsonWebSocketStreamer.java`

```java
package com.pnt.rtk.websocket;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.pnt.rtk.config.DemoRealtimeOutputProperties;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;

import java.time.Instant;
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ScheduledFuture;
import java.util.concurrent.TimeUnit;

// 当前真实工程里 JsonWebSocketStreamer 是静态工具类。
// demo 也按这个设计写，让业务代码只调用 stream(...)。
public final class DemoJsonWebSocketStreamer {

    private static final Set<WebSocketSession> SESSIONS = ConcurrentHashMap.newKeySet();
    private static final ArrayDeque<DemoJsonWebSocketMessage> QUEUE = new ArrayDeque<>();
    private static final Object QUEUE_LOCK = new Object();
    private static ObjectMapper objectMapper = new ObjectMapper().findAndRegisterModules();
    private static boolean batchEnabled = true;
    private static long intervalMillis = 1000;
    private static int maxItems = 100;
    private static ScheduledFuture<?> flushTask;

    private static final ScheduledExecutorService EXECUTOR =
            Executors.newSingleThreadScheduledExecutor(runnable -> {
                Thread thread = new Thread(runnable, "demo-websocket-flusher");
                thread.setDaemon(true);
                return thread;
            });

    private DemoJsonWebSocketStreamer() {
    }

    public static synchronized void configure(ObjectMapper mapper, DemoRealtimeOutputProperties properties) {
        objectMapper = mapper.copy().findAndRegisterModules();
        batchEnabled = properties.isBatchEnabled();
        intervalMillis = Math.max(100, properties.getBatchIntervalMillis());
        maxItems = Math.max(1, properties.getBatchMaxItems());

        if (flushTask != null) {
            flushTask.cancel(false);
        }
        flushTask = EXECUTOR.scheduleWithFixedDelay(
                DemoJsonWebSocketStreamer::flushSafely,
                intervalMillis,
                intervalMillis,
                TimeUnit.MILLISECONDS
        );
    }

    public static void register(WebSocketSession session) {
        SESSIONS.add(session);
        Map<String, Object> payload = new LinkedHashMap<>();
        payload.put("activeSessions", SESSIONS.size());
        streamTo(session, "websocket.connected", payload);
    }

    public static void unregister(WebSocketSession session) {
        SESSIONS.remove(session);
        if (SESSIONS.isEmpty()) {
            synchronized (QUEUE_LOCK) {
                QUEUE.clear();
            }
        }
    }

    public static void stream(String type, Object payload) {
        if (SESSIONS.isEmpty()) {
            return;
        }

        DemoJsonWebSocketMessage message = new DemoJsonWebSocketMessage(type, Instant.now(), payload);
        if (batchEnabled && type.startsWith("demo.data.")) {
            synchronized (QUEUE_LOCK) {
                QUEUE.addLast(message);
            }
            return;
        }
        broadcast(message);
    }

    public static void streamTo(WebSocketSession session, String type, Object payload) {
        sendJson(session, new DemoJsonWebSocketMessage(type, Instant.now(), payload));
    }

    private static void flushSafely() {
        try {
            flush();
        } catch (RuntimeException e) {
            e.printStackTrace();
        }
    }

    private static void flush() {
        if (SESSIONS.isEmpty()) {
            return;
        }

        List<DemoJsonWebSocketMessage> items = new ArrayList<>();
        synchronized (QUEUE_LOCK) {
            while (items.size() < maxItems && !QUEUE.isEmpty()) {
                items.add(QUEUE.removeFirst());
            }
        }
        if (items.isEmpty()) {
            return;
        }

        Map<String, Object> payload = new LinkedHashMap<>();
        payload.put("count", items.size());
        payload.put("intervalMillis", intervalMillis);
        payload.put("items", items);
        broadcast(new DemoJsonWebSocketMessage("demo.batch", Instant.now(), payload));
    }

    private static void broadcast(DemoJsonWebSocketMessage message) {
        for (WebSocketSession session : SESSIONS) {
            sendJson(session, message);
        }
    }

    private static void sendJson(WebSocketSession session, DemoJsonWebSocketMessage message) {
        try {
            if (!session.isOpen()) {
                unregister(session);
                return;
            }
            session.sendMessage(new TextMessage(objectMapper.writeValueAsString(message)));
        } catch (Exception e) {
            unregister(session);
        }
    }
}
```

### 9.9 第九步：写 WebSocket 连接处理器

所属路径：`rtk-websocket-learning-demo/rtk-parser-app/src/main/java/com/pnt/rtk/websocket/DemoJsonWebSocketStreamHandler.java`

```java
package com.pnt.rtk.websocket;

import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;

import java.util.LinkedHashMap;
import java.util.Map;

// 当前真实工程的 JsonWebSocketStreamHandler 只负责连接生命周期。
// demo 也保持这个边界：连接注册、ping/pong、关闭清理。
public class DemoJsonWebSocketStreamHandler extends TextWebSocketHandler {

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        DemoJsonWebSocketStreamer.register(session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        if ("ping".equalsIgnoreCase(message.getPayload())) {
            Map<String, Object> payload = new LinkedHashMap<>();
            payload.put("sessionId", session.getId());
            DemoJsonWebSocketStreamer.streamTo(session, "websocket.pong", payload);
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        DemoJsonWebSocketStreamer.unregister(session);
    }

    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) {
        DemoJsonWebSocketStreamer.unregister(session);
    }
}
```

### 9.10 第十步：写 WebSocket 注册配置

所属路径：`rtk-websocket-learning-demo/rtk-parser-app/src/main/java/com/pnt/rtk/websocket/DemoRealtimeWebSocketConfig.java`

```java
package com.pnt.rtk.websocket;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.pnt.rtk.config.DemoRealtimeOutputProperties;
import jakarta.annotation.PostConstruct;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

import java.util.Objects;

@Configuration
@EnableWebSocket
public class DemoRealtimeWebSocketConfig implements WebSocketConfigurer {

    private final ObjectMapper objectMapper;
    private final DemoRealtimeOutputProperties properties;

    public DemoRealtimeWebSocketConfig(ObjectMapper objectMapper, DemoRealtimeOutputProperties properties) {
        // 构造器注入：如果 Spring 找不到这两个 Bean，应用会启动失败。
        // requireNonNull 让“不能为 null”这件事对学习者更直观。
        this.objectMapper = Objects.requireNonNull(objectMapper, "ObjectMapper 注入失败");
        this.properties = Objects.requireNonNull(properties, "DemoRealtimeOutputProperties 注入失败");
    }

    @PostConstruct
    public void init() {
        DemoJsonWebSocketStreamer.configure(objectMapper, properties);
    }

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new DemoJsonWebSocketStreamHandler(), "/api/demo/realtime/ws")
                .setAllowedOriginPatterns("*");
    }
}
```

### 9.11 第十一步：写 HTTP 接口模拟高频数据

所属路径：`rtk-websocket-learning-demo/rtk-parser-app/src/main/java/com/pnt/rtk/api/DemoRealtimeController.java`

```java
package com.pnt.rtk.api;

import com.pnt.rtk.websocket.DemoJsonWebSocketStreamer;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.LinkedHashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/demo/realtime")
public class DemoRealtimeController {

    @PostMapping("/push")
    public Map<String, Object> push(@RequestParam(defaultValue = "10") int count) {
        for (int i = 0; i < count; i++) {
            Map<String, Object> payload = new LinkedHashMap<>();
            payload.put("index", i);
            payload.put("message", "这是第 " + i + " 条模拟实时数据");
            DemoJsonWebSocketStreamer.stream("demo.data.item", payload);
        }

        Map<String, Object> result = new LinkedHashMap<>();
        result.put("accepted", count);
        return result;
    }
}
```

### 9.12 第十二步：写前端测试页面

所属路径：`rtk-websocket-learning-demo/demo-websocket.html`

```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>RTK Style WebSocket Demo</title>
</head>
<body>
  <button id="connect">连接 WebSocket</button>
  <button id="ping">发送 ping</button>
  <pre id="log"></pre>

  <script>
    const log = document.querySelector("#log");
    let ws;
    let heartbeatTimer;

    function append(text) {
      log.textContent += text + "\n";
    }

    function sendPing(reason) {
      if (!ws || ws.readyState !== WebSocket.OPEN) {
        append("心跳未发送：WebSocket 还没有连接成功");
        return;
      }
      ws.send("ping");
      append(`发送 ping：${reason}`);
    }

    function startHeartbeat() {
      if (heartbeatTimer) {
        clearInterval(heartbeatTimer);
      }
      sendPing("连接成功后立即发送");
      heartbeatTimer = setInterval(() => sendPing("自动心跳"), 5000);
    }

    function stopHeartbeat() {
      if (heartbeatTimer) {
        clearInterval(heartbeatTimer);
        heartbeatTimer = null;
      }
    }

    function handleMessage(message) {
      if (message.type === "demo.batch") {
        append(`收到批量包：${message.payload.count} 条`);
        message.payload.items.forEach(handleMessage);
        return;
      }
      if (message.type === "websocket.pong") {
        append(`收到 pong：${JSON.stringify(message.payload)}`);
        return;
      }
      append(`收到事件：${message.type} ${JSON.stringify(message.payload)}`);
    }

    document.querySelector("#connect").onclick = () => {
      if (ws) {
        ws.close();
      }
      ws = new WebSocket("ws://localhost:18081/api/demo/realtime/ws");
      ws.onopen = () => {
        append("浏览器 WebSocket 握手成功");
        startHeartbeat();
      };
      ws.onmessage = (event) => handleMessage(JSON.parse(event.data));
      ws.onclose = () => {
        append("WebSocket 已关闭");
        stopHeartbeat();
      };
    };

    document.querySelector("#ping").onclick = () => sendPing("手动按钮");
  </script>
</body>
</html>
```

### 9.13 第十三步：运行和测试

所属位置：命令行，在 `rtk-websocket-learning-demo` 目录下执行。

```text
mvn -pl rtk-parser-app spring-boot:run
```

所属位置：浏览器地址栏，直接打开本地 HTML 文件。

```text
rtk-websocket-learning-demo/demo-websocket.html
```

所属位置：页面日志，运行现象说明，不需要创建文件。

```text
浏览器 WebSocket 握手成功
发送 ping：连接成功后立即发送
收到事件：websocket.connected {"activeSessions":1}
收到 pong：{"sessionId":"..."}
发送 ping：自动心跳
收到 pong：{"sessionId":"..."}
```

所属位置：Apifox、Postman、curl 或浏览器接口测试工具。

```text
POST http://localhost:18081/api/demo/realtime/push?count=250
```

所属位置：运行现象说明，不需要创建文件。

```text
第 1 秒：demo.batch，里面 100 条
第 2 秒：demo.batch，里面 100 条
第 3 秒：demo.batch，里面 50 条
```

这就是批量限流：业务一下子产生 250 条，但 WebSocket 不会一瞬间推 250 次。

## 10. 学习时怎么自己验证

### 10.1 先连 WebSocket

前端或 Apifox 连接：

```text
ws://localhost:18081/api/rtcm/realtime/ws
```

你应该马上收到：

```text
websocket.connected
```

如果连上了但只收到 connected，没有收到 RTCM 事件，说明 WebSocket 是通的，但后端还没有解析出可推送的 RTCM 业务数据。

### 10.2 看状态接口

请求：

```text
GET http://localhost:18081/api/rtcm/realtime/ws/status
```

重点看：

```json
{
  "activeSessions": 1,
  "batch": {
    "enabled": true,
    "intervalMillis": 1000,
    "queueSize": 0
  }
}
```

如果 `activeSessions` 是 0，说明没有前端连着 WebSocket。

如果 `queueSize` 很大，说明解析速度比前端推送速度快，队列正在积压。

如果 `droppedSinceLastBatch` 或批量包里的 `dropped` 大于 0，说明队列满过，展示事件被限流丢弃过。

### 10.3 喂一段 RTCM 文件

连接 WebSocket 后，再用接口上传 RTCM 原始文件：

```text
POST http://localhost:18081/api/rtcm/sessions/chunks?sourceType=HTTP&sourceName=manual
Content-Type: application/octet-stream
Body: docs/tcp-202.127.200.33_26702-20260525-153914-11266f44.rtcm3
```

如果 RTCM 数据里解析出了 MSM4 或 NAV，你就会看到 `rtcm.batch`。

## 11. 记忆口诀

你可以这样记：

```text
Handler 管连接。
Streamer 管发送。
Properties 管开关。
Analyzer/ProductWriter 管业务事件。
YAML 管环境差异。
Batch 管风扇别起飞。
```

更像工程里的表达：

```text
业务层只生产事件。
WebSocket 工具层负责传输。
配置层负责调参。
批量队列负责削峰。
状态接口负责观测。
```

## 12. 以后遇到类似问题怎么判断方案

如果你以后再遇到“后端要实时把数据推给前端”的需求，可以按下面问自己：

| 问题 | 如果答案是这样 | 推荐 |
| --- | --- | --- |
| 前端只需要后端单向推送吗 | 是 | SSE 可以考虑 |
| 前端也要发命令/订阅/心跳吗 | 是 | WebSocket 更合适 |
| 是否需要订阅多个 topic | 是 | STOMP 或自定义 type |
| 是否是单应用单节点 | 是 | 当前静态工具类方案够用 |
| 是否多节点部署 | 是 | Kafka/Redis + WebSocket 网关 |
| 数据是否高频 | 是 | 必须考虑批量、限流、背压 |
| 是否每条都不能丢 | 是 | 不要丢队列，改成可靠消息或前端分页拉取 |
| 是否只是实时看最新状态 | 是 | 队列满丢旧保新通常合理 |

最重要的一句话：

```text
实时展示不等于无限制逐条推送。
企业项目里要考虑前端承受能力、后端 CPU、网络、内存、队列积压和可观测性。
```
