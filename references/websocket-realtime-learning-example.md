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

下面这个 demo 是一个可以新建工程后复制粘贴运行的最小 Spring Boot WebSocket 项目。

它不依赖 RTK/RTCM 业务，只保留这次要学习的核心能力：

所属位置：学习说明，不需要创建文件。

```text
1. 后端注册一个 WebSocket 地址。
2. 前端连接后，后端保存这个连接。
3. HTTP 接口模拟业务产生很多条实时消息。
4. 高频消息先进入队列。
5. 定时线程每 1 秒把队列消息合成 demo.batch 推给前端。
6. 前端收到 demo.batch 后拆开 payload.items。
```

### 9.1 第一步：新建工程目录

先新建一个独立工程，名字叫 `websocket-demo`。

所属位置：命令行，不需要创建项目文件。

```text
websocket-demo
  ├─ pom.xml
  ├─ src
  │  └─ main
  │     ├─ java
  │     │  └─ com
  │     │     └─ example
  │     │        └─ websocketdemo
  │     │           ├─ WebSocketDemoApplication.java
  │     │           ├─ api
  │     │           │  └─ DemoController.java
  │     │           └─ websocket
  │     │              ├─ DemoWebSocketConfig.java
  │     │              ├─ DemoWebSocketHandler.java
  │     │              ├─ DemoWebSocketMessage.java
  │     │              ├─ DemoWebSocketProperties.java
  │     │              └─ DemoWebSocketStreamer.java
  │     └─ resources
  │        └─ application.yml
  └─ demo-websocket.html
```

这就是开发顺序：先有工程骨架，再一层一层把后端和前端补进去。

### 9.2 第二步：写 Maven 依赖

所属路径：`websocket-demo/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!--
      使用 Spring Boot parent。
      为什么需要 parent：
      它会帮你统一管理 Spring Boot 依赖版本，减少自己手动配版本的麻烦。
    -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.12</version>
        <relativePath/>
    </parent>

    <!--
      下面三个坐标是这个 demo 工程自己的身份。
      groupId 通常写公司或组织名，这里用 com.example。
      artifactId 是工程名，这里用 websocket-demo。
      version 是当前版本。
    -->
    <groupId>com.example</groupId>
    <artifactId>websocket-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <!--
      Java 版本使用 17。
      Spring Boot 3.x 最常见的学习配置就是 Java 17。
    -->
    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!--
          提供普通 HTTP REST 接口能力。
          为什么需要它：
          后面要写 POST /demo/push，用它模拟“业务系统产生实时数据”。
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!--
          提供 WebSocket 能力。
          为什么需要它：
          后面要注册 /demo/ws，让浏览器可以通过 WebSocket 连接后端。
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-websocket</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!--
              Spring Boot 打包和运行插件。
              有了它，可以用 mvn spring-boot:run 启动这个 demo。
            -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### 9.3 第三步：写 application.yml

所属路径：`websocket-demo/src/main/resources/application.yml`

```yaml
server:
  # 后端服务端口。
  # 前端连接 ws://localhost:18081/demo/ws 时，18081 就来自这里。
  port: 18081

demo:
  websocket:
    # 是否开启批量限流。
    # true：高频消息先进入队列，再由定时线程批量推送。
    # false：高频消息来一条发一条，学习时能看到原始效果，但数据多时容易刷屏。
    batch-enabled: true

    # 每隔多少毫秒推送一次批量包。
    # 1000 表示大约每 1 秒推一次 demo.batch。
    batch-interval-millis: 1000

    # 每个批量包最多放多少条原始消息。
    # 这个值用于控制单个 JSON 包大小。
    batch-max-items: 100
```

### 9.4 第四步：写启动类

所属路径：`websocket-demo/src/main/java/com/example/websocketdemo/WebSocketDemoApplication.java`

```java
package com.example.websocketdemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

// @SpringBootApplication 是 Spring Boot 应用启动入口。
// 它会自动扫描 com.example.websocketdemo 以及它下面的子包。
// 所以后面的 api 包、websocket 包都要放在 com.example.websocketdemo 下面。
@SpringBootApplication
public class WebSocketDemoApplication {

    public static void main(String[] args) {
        // 启动 Spring Boot 应用。
        // 启动后，HTTP 接口和 WebSocket 地址才会真正监听 18081 端口。
        SpringApplication.run(WebSocketDemoApplication.class, args);
    }
}
```

### 9.5 第五步：写配置绑定类

所属路径：`websocket-demo/src/main/java/com/example/websocketdemo/websocket/DemoWebSocketProperties.java`

```java
package com.example.websocketdemo.websocket;

import org.springframework.boot.context.properties.ConfigurationProperties;

// @ConfigurationProperties 的作用：
// 把 application.yml 里 demo.websocket 开头的配置绑定到这个 Java 类。
// 比如 batch-enabled 会绑定到 batchEnabled。
@ConfigurationProperties(prefix = "demo.websocket")
public class DemoWebSocketProperties {

    // 是否开启批量限流。
    // 默认 true 是为了保护浏览器和后端，避免高频消息来一条发一条。
    private boolean batchEnabled = true;

    // 批量推送间隔，单位是毫秒。
    // 这个值控制 WebSocket 展示层推送频率，不控制业务生产速度。
    private long batchIntervalMillis = 1000;

    // 每个批量包最多包含多少条消息。
    // 如果这个值太大，单个 JSON 包可能很大；太小，积压消息会分很多批慢慢发。
    private int batchMaxItems = 100;

    public boolean isBatchEnabled() {
        // 让其他代码可以读取 batchEnabled。
        return batchEnabled;
    }

    public void setBatchEnabled(boolean batchEnabled) {
        // Spring Boot 绑定 YAML 时会调用这个 setter。
        this.batchEnabled = batchEnabled;
    }

    public long getBatchIntervalMillis() {
        // 让定时任务读取批量推送间隔。
        return batchIntervalMillis;
    }

    public void setBatchIntervalMillis(long batchIntervalMillis) {
        // Spring Boot 绑定 YAML 时会调用这个 setter。
        this.batchIntervalMillis = batchIntervalMillis;
    }

    public int getBatchMaxItems() {
        // 让 flush 逻辑知道一次最多取多少条消息。
        return batchMaxItems;
    }

    public void setBatchMaxItems(int batchMaxItems) {
        // Spring Boot 绑定 YAML 时会调用这个 setter。
        this.batchMaxItems = batchMaxItems;
    }
}
```

### 9.6 第六步：写 WebSocket 统一消息结构

所属路径：`websocket-demo/src/main/java/com/example/websocketdemo/websocket/DemoWebSocketMessage.java`

```java
package com.example.websocketdemo.websocket;

import java.time.Instant;

// record 适合表示这种“只装数据”的对象。
// WebSocket 统一消息结构包括：
// type：事件类型，前端靠它判断怎么处理。
// emittedAt：后端产生消息的时间。
// payload：真正的业务数据。
public record DemoWebSocketMessage(
        String type,
        Instant emittedAt,
        Object payload
) {
}
```

### 9.7 第七步：写 WebSocket 推送工具类

所属路径：`websocket-demo/src/main/java/com/example/websocketdemo/websocket/DemoWebSocketStreamer.java`

```java
package com.example.websocketdemo.websocket;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;

import java.time.Instant;
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ScheduledFuture;
import java.util.concurrent.TimeUnit;

public final class DemoWebSocketStreamer {

    // 保存当前所有在线 WebSocket 连接。
    // 使用线程安全 Set，是因为连接建立、关闭、HTTP 推送、定时 flush 可能发生在不同线程。
    private static final Set<WebSocketSession> SESSIONS = ConcurrentHashMap.newKeySet();

    // 保存待批量推送的高频消息。
    // ArrayDeque 在这里当 FIFO 队列使用：addLast 入队，removeFirst 出队。
    private static final ArrayDeque<DemoWebSocketMessage> QUEUE = new ArrayDeque<>();

    // 队列锁。
    // ArrayDeque 不是线程安全的，所以入队和出队时要加锁。
    private static final Object QUEUE_LOCK = new Object();

    // JSON 序列化器。
    // 先给一个默认值，避免 configure 之前被调用时出现空指针。
    private static ObjectMapper objectMapper = new ObjectMapper().findAndRegisterModules();

    // 是否开启批量。
    private static boolean batchEnabled = true;

    // 批量推送间隔。
    private static long intervalMillis = 1000;

    // 每批最大消息数。
    private static int maxItems = 100;

    // 单线程定时器，用来定时执行 flush。
    private static final ScheduledExecutorService EXECUTOR =
            Executors.newSingleThreadScheduledExecutor(runnable -> {
                // 给线程起名，后面排查线程问题时更容易看懂。
                Thread thread = new Thread(runnable, "demo-websocket-flusher");
                // 设置为守护线程，避免应用退出时被这个学习 demo 的线程卡住。
                thread.setDaemon(true);
                return thread;
            });

    // 保存当前定时任务。
    // 这样 configure 被重复调用时，可以先取消旧任务，避免启动多个 flush 定时器。
    private static ScheduledFuture<?> flushTask;

    private DemoWebSocketStreamer() {
        // 私有构造方法表示这个类不应该被 new。
        // 外部直接调用 DemoWebSocketStreamer.send(...)。
    }

    public static synchronized void configure(ObjectMapper mapper, DemoWebSocketProperties properties) {
        // 使用 Spring Boot 配好的 ObjectMapper。
        // copy() 是为了避免静态工具类误改 Spring 容器里的全局 ObjectMapper。
        objectMapper = mapper.copy().findAndRegisterModules();

        // 从 YAML 配置读取是否开启批量。
        batchEnabled = properties.isBatchEnabled();

        // 防呆：最小间隔限制为 100ms。
        // 如果误配成 0ms，定时任务会疯狂运行，CPU 可能被打满。
        intervalMillis = Math.max(100, properties.getBatchIntervalMillis());

        // 防呆：每批至少 1 条。
        // 如果误配成 0，flush 永远取不到消息，前端就看不到批量数据。
        maxItems = Math.max(1, properties.getBatchMaxItems());

        // 如果已经启动过定时任务，先取消旧任务。
        // 这是为了防止重复 configure 时出现多个定时器同时 flush。
        if (flushTask != null) {
            flushTask.cancel(false);
        }

        // 启动定时 flush。
        // 第一个 intervalMillis：启动后等多久第一次执行。
        // 第二个 intervalMillis：每次 flush 完成后，再等多久执行下一次。
        flushTask = EXECUTOR.scheduleWithFixedDelay(
                DemoWebSocketStreamer::flushSafely,
                intervalMillis,
                intervalMillis,
                TimeUnit.MILLISECONDS
        );
    }

    public static void register(WebSocketSession session) {
        // 保存前端连接。
        // 保存以后，broadcast 才能把消息发给这个前端。
        SESSIONS.add(session);

        // 连接成功消息立即发，不进入批量队列。
        // 因为前端需要马上知道连接已经成功。
        sendTo(session, "websocket.connected", Map.of("activeSessions", SESSIONS.size()));
    }

    public static void unregister(WebSocketSession session) {
        // 连接断开后移除，避免后面继续给失效连接发消息。
        SESSIONS.remove(session);

        // 如果已经没有前端连接，就清空队列。
        // 因为没人在线时，展示消息继续积压没有意义。
        if (SESSIONS.isEmpty()) {
            synchronized (QUEUE_LOCK) {
                QUEUE.clear();
            }
        }
    }

    public static void send(String type, Object payload) {
        // 如果没有前端连接，直接返回。
        // 这样可以避免没人看页面时还浪费 CPU 做 JSON 序列化和网络发送。
        if (SESSIONS.isEmpty()) {
            return;
        }

        // 创建统一消息对象。
        DemoWebSocketMessage message = new DemoWebSocketMessage(type, Instant.now(), payload);

        // demo.data.* 表示学习 demo 里的高频业务事件。
        // 这些事件进入队列，等待定时批量推送。
        if (batchEnabled && type.startsWith("demo.data.")) {
            synchronized (QUEUE_LOCK) {
                QUEUE.addLast(message);
            }
            return;
        }

        // 其他低频事件立即广播。
        // 例如 websocket.connected、websocket.pong、error 等。
        broadcast(message);
    }

    public static void sendTo(WebSocketSession session, String type, Object payload) {
        // 只给某一个连接发消息。
        // 适合 pong 或连接欢迎语，不适合全局广播。
        sendJson(session, new DemoWebSocketMessage(type, Instant.now(), payload));
    }

    private static void flushSafely() {
        try {
            // 包一层 try/catch，避免 flush 抛异常后定时任务彻底停止。
            flush();
        } catch (RuntimeException e) {
            // 学习 demo 里简单打印异常。
            // 真实项目里建议使用日志框架记录。
            e.printStackTrace();
        }
    }

    private static void flush() {
        // 没有连接时不发送。
        if (SESSIONS.isEmpty()) {
            return;
        }

        // 从队列取一批消息到局部变量。
        // 这样后面 JSON 序列化和网络发送时，不会一直占着 QUEUE_LOCK。
        List<DemoWebSocketMessage> items = new ArrayList<>();
        synchronized (QUEUE_LOCK) {
            while (items.size() < maxItems && !QUEUE.isEmpty()) {
                items.add(QUEUE.removeFirst());
            }
        }

        // 没有待发送消息时，不发送空包。
        if (items.isEmpty()) {
            return;
        }

        // 合成批量包。
        // 前端收到的顶层 type 是 demo.batch。
        // 原始消息在 payload.items 里。
        DemoWebSocketMessage batch = new DemoWebSocketMessage(
                "demo.batch",
                Instant.now(),
                Map.of(
                        "count", items.size(),
                        "intervalMillis", intervalMillis,
                        "items", items
                )
        );

        // 把批量包发给所有在线连接。
        broadcast(batch);
    }

    private static void broadcast(DemoWebSocketMessage message) {
        // 遍历所有在线连接。
        for (WebSocketSession session : SESSIONS) {
            sendJson(session, message);
        }
    }

    private static void sendJson(WebSocketSession session, DemoWebSocketMessage message) {
        try {
            // 连接已经关闭时，先移除再返回。
            if (!session.isOpen()) {
                SESSIONS.remove(session);
                return;
            }

            // Java 对象转 JSON 字符串。
            String json = objectMapper.writeValueAsString(message);

            // 通过 WebSocket 发送文本消息。
            session.sendMessage(new TextMessage(json));
        } catch (Exception e) {
            // 发送失败通常说明连接不可用。
            // 移除后，下次广播不会再尝试这个 session。
            SESSIONS.remove(session);
        }
    }
}
```

### 9.8 第八步：写 WebSocket 连接处理器

所属路径：`websocket-demo/src/main/java/com/example/websocketdemo/websocket/DemoWebSocketHandler.java`

```java
package com.example.websocketdemo.websocket;

import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;

import java.util.Map;

// TextWebSocketHandler 是 Spring 提供的文本 WebSocket 处理器。
// 当前 demo 只收发 JSON 字符串，所以继承它就够了。
public class DemoWebSocketHandler extends TextWebSocketHandler {

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        // 前端连接成功后，保存这个 session。
        // 不保存的话，后端后面就不知道该把消息发给谁。
        DemoWebSocketStreamer.register(session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        // 前端发 ping，后端回 pong。
        // 这是最小心跳机制，用来确认连接还活着。
        if ("ping".equalsIgnoreCase(message.getPayload())) {
            DemoWebSocketStreamer.sendTo(session, "websocket.pong", Map.of("sessionId", session.getId()));
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        // 前端断开后清理 session。
        // 否则后端集合里会残留无效连接。
        DemoWebSocketStreamer.unregister(session);
    }

    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) {
        // 传输异常时也清理 session。
        // 这样可以避免异常连接一直留在内存里。
        DemoWebSocketStreamer.unregister(session);
    }
}
```

### 9.9 第九步：写 WebSocket 注册配置

所属路径：`websocket-demo/src/main/java/com/example/websocketdemo/websocket/DemoWebSocketConfig.java`

```java
package com.example.websocketdemo.websocket;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.annotation.PostConstruct;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

// @Configuration 表示这是 Spring 配置类。
// Spring Boot 启动时会读取这里的 WebSocket 注册逻辑。
@Configuration

// @EnableWebSocket 表示开启 Spring 原生 WebSocket 支持。
@EnableWebSocket

// 启用 DemoWebSocketProperties 配置绑定。
// 如果不加它，DemoWebSocketProperties 可能无法注入到构造方法。
@EnableConfigurationProperties(DemoWebSocketProperties.class)
public class DemoWebSocketConfig implements WebSocketConfigurer {

    // Spring Boot 自动配置好的 JSON 工具。
    private final ObjectMapper objectMapper;

    // application.yml 绑定出来的配置。
    private final DemoWebSocketProperties properties;

    public DemoWebSocketConfig(ObjectMapper objectMapper, DemoWebSocketProperties properties) {
        // 保存 JSON 工具。
        this.objectMapper = objectMapper;
        // 保存批量限流配置。
        this.properties = properties;
    }

    @PostConstruct
    public void init() {
        // 应用启动后，把 ObjectMapper 和 YAML 配置交给静态工具类。
        // 因为 DemoWebSocketStreamer 是静态工具类，它自己不是 Spring Bean。
        DemoWebSocketStreamer.configure(objectMapper, properties);
    }

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        // 注册 WebSocket 地址 /demo/ws。
        // 前端连接 ws://localhost:18081/demo/ws 时，会进入 DemoWebSocketHandler。
        registry.addHandler(new DemoWebSocketHandler(), "/demo/ws")
                // 学习 demo 允许任意来源连接，方便直接打开本地 HTML 测试。
                // 生产环境建议改成你的前端域名。
                .setAllowedOriginPatterns("*");
    }
}
```

### 9.10 第十步：写 HTTP 接口模拟高频数据

所属路径：`websocket-demo/src/main/java/com/example/websocketdemo/api/DemoController.java`

```java
package com.example.websocketdemo.api;

import com.example.websocketdemo.websocket.DemoWebSocketStreamer;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

// @RestController 表示这是 HTTP 接口类。
// 这个类不负责 WebSocket 连接，只负责模拟业务产生实时数据。
@RestController
@RequestMapping("/demo")
public class DemoController {

    @PostMapping("/push")
    public Map<String, Object> push(@RequestParam(defaultValue = "10") int count) {
        // 模拟 count 条业务消息。
        // 真实项目里，这些消息可能来自 RTCM 解码、设备上报、订单状态变化等。
        for (int i = 0; i < count; i++) {
            // type 使用 demo.data.item。
            // 因为 DemoWebSocketStreamer 里判断 type.startsWith("demo.data.")，
            // 所以这些消息会进入批量队列，而不是立即一条一条发。
            DemoWebSocketStreamer.send("demo.data.item", Map.of(
                    "index", i,
                    "message", "这是第 " + i + " 条模拟实时数据"
            ));
        }

        // HTTP 返回只表示“消息已经交给 WebSocket 工具类”。
        // 它不表示前端已经收到，因为前端要等下一次定时 flush。
        return Map.of("accepted", count);
    }
}
```

### 9.11 第十一步：写前端测试页面

所属路径：`websocket-demo/demo-websocket.html`

```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>WebSocket Demo</title>
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
      // 统一追加页面日志。
      log.textContent += text + "\n";
    }

    function sendPing(reason) {
      // 发送 ping 前先判断连接状态。
      // WebSocket.OPEN 表示连接已经建立，可以发送消息。
      if (!ws || ws.readyState !== WebSocket.OPEN) {
        append("心跳未发送：WebSocket 还没有连接成功");
        return;
      }

      // 前端主动发 ping。
      // 后端 DemoWebSocketHandler 收到 "ping" 后，会返回 websocket.pong。
      ws.send("ping");

      // 把“前端已经发出 ping”也打印出来。
      // 这样学习时能清楚看到：不是没心跳，而是先有 ping，后有 pong。
      append(`发送 ping：${reason}`);
    }

    function startHeartbeat() {
      // 如果之前已经启动过心跳，先清掉旧定时器。
      // 这样重复点击“连接 WebSocket”时，不会启动多个心跳定时器。
      if (heartbeatTimer) {
        clearInterval(heartbeatTimer);
      }

      // 连接成功后先立刻发一次 ping。
      // 这样不用等 5 秒，页面马上能看到心跳效果。
      sendPing("连接成功后立即发送");

      // 每 5 秒自动发一次 ping。
      // 这就是最小 demo 里的“自动心跳”。
      heartbeatTimer = setInterval(() => {
        sendPing("自动心跳");
      }, 5000);
    }

    function stopHeartbeat() {
      // 连接关闭时停止心跳定时器。
      // 如果不停止，页面会继续尝试向已关闭连接发送 ping。
      if (heartbeatTimer) {
        clearInterval(heartbeatTimer);
        heartbeatTimer = null;
      }
    }

    function handleMessage(message) {
      // 如果后端发来的是批量包，就拆开 items。
      // 拆开以后继续交给 handleMessage，这样前端不用写两套业务处理逻辑。
      if (message.type === "demo.batch") {
        append(`收到批量包：${message.payload.count} 条`);
        message.payload.items.forEach(handleMessage);
        return;
      }

      // 普通业务数据。
      if (message.type === "demo.data.item") {
        append(`业务数据：${JSON.stringify(message.payload)}`);
        return;
      }

      // 心跳响应。
      // 后端收到前端 "ping" 文本后，会返回 type = websocket.pong。
      if (message.type === "websocket.pong") {
        append(`收到 pong：${JSON.stringify(message.payload)}`);
        return;
      }

      // connected、pong 等低频事件走这里。
      append(`其他事件：${message.type} ${JSON.stringify(message.payload)}`);
    }

    document.querySelector("#connect").onclick = () => {
      // 如果已经有旧连接，先关闭旧连接。
      // 这样重复点击连接按钮时，不会同时存在多个 WebSocket。
      if (ws) {
        ws.close();
      }

      // 连接后端 WebSocket。
      // 地址必须和后端 registry.addHandler(..., "/demo/ws") 对上。
      ws = new WebSocket("ws://localhost:18081/demo/ws");

      ws.onopen = () => {
        // onopen 表示浏览器和后端 WebSocket 握手成功。
        append("浏览器 WebSocket 握手成功");

        // 握手成功后启动自动心跳。
        startHeartbeat();
      };

      ws.onmessage = (event) => {
        // 后端发送的是 JSON 字符串，前端先 JSON.parse。
        const message = JSON.parse(event.data);
        handleMessage(message);
      };

      ws.onclose = () => {
        // 连接关闭时提示。
        append("WebSocket 已关闭");

        // 连接关闭后停止自动心跳。
        stopHeartbeat();
      };
    };

    document.querySelector("#ping").onclick = () => {
      // 手动发送一次 ping。
      // 这和自动心跳共用同一个 sendPing 方法，避免写两套逻辑。
      sendPing("手动按钮");
    };
  </script>
</body>
</html>
```

### 9.12 第十二步：运行和测试

所属位置：命令行，在 `websocket-demo` 目录下执行。

```text
mvn spring-boot:run
```

所属位置：浏览器地址栏，直接打开本地 HTML 文件。

```text
websocket-demo/demo-websocket.html
```

打开页面后，先点 `连接 WebSocket`。

所属位置：页面日志，运行现象说明，不需要创建文件。

```text
浏览器 WebSocket 握手成功
发送 ping：连接成功后立即发送
其他事件：websocket.connected {"activeSessions":1}
收到 pong：{"sessionId":"..."}
发送 ping：自动心跳
收到 pong：{"sessionId":"..."}
```

如果你看不到 `收到 pong`，优先检查：

所属位置：排查说明，不需要创建文件。

```text
1. 后端是否已经启动在 18081 端口。
2. 前端是否真的点击了“连接 WebSocket”。
3. 浏览器控制台 Network 里 /demo/ws 是否连接成功。
4. DemoWebSocketHandler.handleTextMessage 是否判断了 "ping"。
5. 后端返回的 type 是否是 websocket.pong，前端 handleMessage 是否单独处理了这个 type。
```

所属位置：Apifox、Postman、curl 或浏览器接口测试工具。

```text
POST http://localhost:18081/demo/push?count=250
```

如果配置是：

对应配置路径：`websocket-demo/src/main/resources/application.yml`

```yaml
batch-interval-millis: 1000
batch-max-items: 100
```

那么前端大概会分几次收到：

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
