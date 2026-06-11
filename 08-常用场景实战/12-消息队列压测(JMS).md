# 12-消息队列压测(JMS) 📨

> **难度：★★★★☆** | JMS Publisher / JMS Subscriber / JMS Point-to-Point 采样器

---

## 1. 场景概述

### 为什么需要消息队列压测？

- **吞吐验证**：验证 MQ 的生产/消费吞吐能力是否满足业务需求
- **积压恢复**：测试消费端在消息积压后的恢复速度
- **消息丢失**：高并发下验证消息可靠性（at-least-once / exactly-once）
- **延迟评估**：测量端到端消息延迟（produce → consume）

### 支持的 JMS 实现

| 消息中间件 | JMS 实现 | 所需 Jar |
|-----------|---------|----------|
| ActiveMQ | 原生 JMS 1.1 | `activemq-all-5.x.x.jar` |
| RabbitMQ (via JMS) | RabbitMQ JMS Client | `rabbitmq-jms-2.x.x.jar` + `amqp-client-5.x.x.jar` |
| IBM MQ | IBM MQ JMS | `com.ibm.mq.allclient-9.x.x.jar` |
| Apache Kafka | 非 JMS 协议，需用自定义 Sampler | 不适用 |

> ⚠️ **Kafka 注意**：Kafka 不走 JMS 协议，JMeter 官方不支持。可以使用 [kloadgen](https://github.com/corunet/kloadgen) 或 JSR223 Sampler + Kafka Client 库。

---

## 2. 环境准备

### 2.1 放置 JMS 客户端 Jar

```
$JMETER_HOME/lib/
├── activemq-all-5.17.0.jar        # ActiveMQ 客户端
├── geronimo-jms_1.1_spec-1.1.1.jar # JMS 1.1 规范
└── hawtbuf-1.11.jar               # ActiveMQ 依赖
```

### 2.2 ActiveMQ 快速启动（测试用）

```bash
# Docker 启动 ActiveMQ
docker run -d --name activemq \
  -p 61616:61616 \
  -p 8161:8161 \
  symptoma/activemq:5.17.0

# 管理控制台: http://localhost:8161 (admin/admin)
```

---

## 3. 测试计划构建

### 3.1 生产者压测（JMS Publisher）

```
Test Plan
├── User Defined Variables
│   └── brokerUrl = tcp://localhost:61616
├── Thread Group (消息生产者)
│   ├── JMS Publisher
│   │   ├── Initial Context Factory
│   │   ├── Provider URL
│   │   ├── Connection Factory
│   │   ├── Destination (Queue/Topic)
│   │   ├── Message Source (文本/文件/文件夹)
│   │   └── Number of samples to aggregate
│   └── View Results Tree
└── Aggregate Report
```

**JMS Publisher 配置详解**：

| 配置项 | 示例值 | 说明 |
|--------|--------|------|
| **Use JNDI properties** | ✅ 勾选 | 使用属性文件配置连接 |
| **Initial Context Factory** | `org.apache.activemq.jndi.ActiveMQInitialContextFactory` | JNDI 工厂类 |
| **Provider URL** | `tcp://localhost:61616` | Broker 地址 |
| **Connection Factory** | `ConnectionFactory` | JNDI 中连接工厂名称 |
| **Destination** | `dynamicQueues/test.queue` | 动态队列名 |
| **Message Type** | `Text Message` | 消息类型 |
| **Content** | `{"userId":"${__UUID}","timestamp":"${__time}"}` | 消息内容 |
| **Number of samples to aggregate** | `100` | 批量发送数量 |

**消息类型**：

| 类型 | 说明 | 使用场景 |
|------|------|----------|
| **Text Message** | 文本消息（JSON/XML/纯文本） | 最常用，业务数据 |
| **Map Message** | 键值对消息 | 结构化简单数据 |
| **Object Message** | 序列化对象 | Java 对象传输（不推荐） |
| **Bytes Message** | 二进制消息 | 文件/图片传输 |
| **Stream Message** | 流式消息 | 有序数据流 |

### 3.2 消费者压测（JMS Subscriber）

```
Thread Group (消息消费者)
├── JMS Subscriber
│   ├── Initial Context Factory (同上)
│   ├── Provider URL (同上)
│   ├── Connection Factory (同上)
│   ├── Destination (同上)
│   ├── Read response ✅
│   ├── Timeout (milliseconds): 5000
│   └── Sample aggregate: 100
└── JSR223 PostProcessor (处理消费的消息)
```

**JMS Subscriber 配置详解**：

| 配置项 | 值 | 说明 |
|--------|-----|------|
| **Read response** | ✅ | 必须勾选，否则不读取消息 |
| **Timeout** | `5000` | 等待消息超时(ms)，超时则请求失败 |
| **Stop between samples** | `100` | 批次间间隔(ms) |
| **Use non-persistent delivery** | ☐ | 非持久化模式（更快但可能丢失） |
| **Use Request/Reply** | ☐ | 请求-响应模式（P2P 专用） |
| **Use alternate fields** | ☐ | 自定义 JMS 属性 |

### 3.3 点对点压测（JMS Point-to-Point）

```
Thread Group (P2P 压测)
├── JSR223 PreProcessor (构造请求消息)
├── JMS Point-to-Point
│   ├── QueueConnectionFactory
│   ├── Request Queue: dynamicQueues/request.q
│   ├── Receive Queue: dynamicQueues/reply.q
│   ├── Communication Style: Request-Reply
│   ├── Timeout: 10000
│   └── Content: ${requestMessage}
└── JSON Extractor (从回复消息提取数据)
```

---

## 4. JNDI 配置方式

### 4.1 方式一：代码内配置（推荐测试用）

在 JMS Sampler 中直接填写 `Initial Context Factory` 和 `Provider URL`，无需外部配置文件。

### 4.2 方式二：JNDI 属性文件

创建 `jndi.properties` 放到 JMeter 的 `bin/` 目录：

```properties
java.naming.factory.initial=org.apache.activemq.jndi.ActiveMQInitialContextFactory
java.naming.provider.url=tcp://localhost:61616

# 队列定义
queue.testQueue=test.queue

# 主题定义
topic.testTopic=test.topic
```

### 4.3 方式三：JMeter 全局属性

在 `user.properties` 或命令行中：

```properties
# JMeter Properties
jmeter.jms.jndi.initialContextFactory=org.apache.activemq.jndi.ActiveMQInitialContextFactory
jmeter.jms.jndi.providerUrl=tcp://localhost:61616
```

---

## 5. 实战场景

### 5.1 消息吞吐测试

**目标**：测量在 100 并发下，ActiveMQ 的生产和消费吞吐量。

```
Test Plan
├── Thread Group - Producer (50 线程)
│   ├── JMS Publisher
│   │   └── Destination: dynamicQueues/perf.test
│   └── Constant Throughput Timer (目标 TPS: 1000/s)
│
├── Thread Group - Consumer (50 线程)
│   ├── JMS Subscriber
│   │   └── Destination: dynamicQueues/perf.test
│   └── JSR223 PostProcessor (消费计数)
│
└── Aggregate Report (分别查看生产/消费 TPS)
```

### 5.2 消息堆积恢复测试

```
Test Plan
├── Thread Group - Producer (持续发送 5 分钟)
│   ├── JMS Publisher
│   └── Runtime Controller (300s)
│
├── Thread Group - Consumer (延迟 2 分钟后启动)
│   ├── Test Action (暂停 120000ms)
│   ├── JMS Subscriber
│   └── Transaction Controller
│       ├── JSR223 PostProcessor (记录消费耗时)
│       └── Duration Assertion (每批 < 1000ms)
│
└── Backend Listener → InfluxDB (监控堆积量)
```

### 5.3 Topic 广播测试

```xml
<!-- JMS Publisher → Topic -->
<JMSSampler>
  <stringProp name="JMSSampler.jndiInitialContextFactory">
    org.apache.activemq.jndi.ActiveMQInitialContextFactory
  </stringProp>
  <stringProp name="JMSSampler.jndiProviderUrl">tcp://localhost:61616</stringProp>
  <stringProp name="JMSSampler.jndiCF">ConnectionFactory</stringProp>
  <stringProp name="JMSSampler.destination">dynamicTopics/broadcast.topic</stringProp>
  <boolProp name="JMSSampler.isTopic">true</boolProp>
  <boolProp name="JMSSampler.useNonPersistent">true</boolProp>
  <stringProp name="JMSSampler.textmessage">
    {"event":"price_update","symbol":"BTC","price":"${__Random(40000,50000)}"}
  </stringProp>
</JMSSampler>
```

### 5.4 JMS 属性传递

```groovy
// JSR223 PreProcessor - 设置 JMS 自定义属性
import javax.jms.Message;

// 通过 JMeter 变量传递 JMS 属性
// JMS Publisher 的 JMS Properties 配置：
// propertyName=JMSCorrelationID, propertyValue=${correlationId}
// propertyName=priority, propertyValue=HIGH

vars.put("correlationId", UUID.randomUUID().toString());
vars.put("messagePriority", "9");  // 0-9, 9最高
```

**JMS Publisher 属性配置**：

| Name | Value | Type |
|------|-------|------|
| `JMSCorrelationID` | `${correlationId}` | `java.lang.String` |
| `JMSPriority` | `9` | `java.lang.Integer` |
| `JMSExpiration` | `${__longSum(${__time},3600000)}` | `java.lang.Long` |
| `customHeader` | `${customValue}` | `java.lang.String` |

---

## 6. 消费端消息处理

### 6.1 消息反序列化

```groovy
// JSR223 PostProcessor - 处理 JMS Subscriber 接收的消息
import groovy.json.JsonSlurper;

// 获取 JMS Subscriber 响应数据
def responseData = prev.getResponseDataAsString();

if (responseData) {
    def json = new JsonSlurper().parseText(responseData);
    
    // 提取关键字段到 JMeter 变量
    vars.put("receivedUserId", json.userId);
    vars.put("receivedAmount", json.amount.toString());
    
    // 计算端到端延迟
    def sentTime = json.timestamp as Long;
    def recvTime = System.currentTimeMillis();
    def latency = recvTime - sentTime;
    
    vars.put("e2eLatency", latency.toString());
    
    log.info("Message received: userId=${json.userId}, latency=${latency}ms");
}
```

### 6.2 消息去重校验

```groovy
// JSR223 PostProcessor - 消息去重检测
def messageId = vars.get("JMSCorrelationID");

// 使用 JMeter 属性存储已处理消息（线程安全）
def processedSet = props.get("processedMessages");
if (processedSet == null) {
    processedSet = new java.util.concurrent.ConcurrentHashMap().newKeySet();
    props.put("processedMessages", processedSet);
}

if (!processedSet.add(messageId)) {
    log.warn("Duplicate message detected: " + messageId);
    prev.setSuccessful(false);
    prev.setResponseMessage("Duplicate: " + messageId);
}
```

---

## 7. 关键断言

### 7.1 生产者断言

```xml
<!-- Response Assertion - 验证消息发送成功 -->
<ResponseAssertion>
  <stringProp name="Assertion.test_field">JMSSampler.msgSent</stringProp>
  <stringProp name="Assertion.test_type">2</stringProp>  <!-- Contains -->
  <collectionProp name="Asserion.test_strings">
    <stringProp name="0">true</stringProp>
  </collectionProp>
</ResponseAssertion>
```

### 7.2 消费者断言

```xml
<!-- JSON Assertion - 验证消息格式正确 -->
<JSONPathAssertion>
  <stringProp name="JSON_PATH">$.userId</stringProp>
  <stringProp name="EXPECTED_VALUE">\d+</stringProp>
  <boolProp name="JSONVALIDATION">true</boolProp>
  <boolProp name="ISREGEX">true</boolProp>
</JSONPathAssertion>
```

---

## 8. 常见问题排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **ClassNotFoundException: ActiveMQInitialContextFactory** | 缺少 ActiveMQ jar | 放入 `activemq-all-5.x.x.jar` 到 `lib/` |
| **Connection refused** | Broker 未启动或端口错误 | 检查 Broker 状态，确认端口 |
| **Destination not found** | 队列/主题名不正确 | ActiveMQ 使用 `dynamicQueues/xxx` 或 `dynamicTopics/xxx` |
| **Consumer 收不到消息** | 未勾选 Read response | 勾选 JMS Subscriber 的 `Read response` |
| **OOM / 内存溢出** | 消息积压过多 | 限制 Subscriber 的 `Sample aggregate` 大小 |
| **非持久化消息丢失** | Broker 重启 | 改用持久化消息 `DeliveryMode.PERSISTENT` |

---

## 9. 性能优化建议

1. **批量发送**：增大 `Number of samples to aggregate` 减少网络开销
2. **非持久化**：测试吞吐时使用 `non-persistent` 模式
3. **异步发送**：Producer 使用异步模式 + 回调
4. **连接复用**：使用 JNDI 连接池，避免每次创建连接
5. **消费者预取**：调整 `prefetchSize` 提高消费效率
6. **消息压缩**：大消息启用压缩（ActiveMQ: `useCompression=true`）
