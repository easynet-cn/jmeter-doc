# 14-TCP协议压测 🔌

> **难度：★★★★☆** | TCP Sampler 采样器 + TCP Sampler Config 配置元素

---

## 1. 场景概述

### 适用场景

- **自定义 TCP 协议**：基于 TCP 的私有协议、游戏协议、物联网协议
- **Socket 服务**：WebSocket、Socket.IO 等长连接服务
- **负载均衡器**：测试 TCP 代理/L4 负载均衡的吞吐
- **网络层性能**：评估纯 TCP 层面的网络延迟和吞吐

### TCP Sampler 支持的实现类

| 实现类 | 说明 | 适用场景 |
|--------|------|----------|
| **TCPClientImpl** | 默认实现，文本/二进制 | 通用 TCP 请求-响应 |
| **BinaryTCPClientImpl** | 二进制模式 | 十六进制报文 |
| **LengthPrefixedBinaryTCPClientImpl** | 长度前缀二进制 | 固定头长度协议 |

---

## 2. 测试计划构建

### 2.1 基础 TCP 请求-响应

```
Test Plan
├── TCP Sampler Config
│   ├── TCPClient classname: TCPClientImpl
│   ├── Server: 192.168.1.100
│   ├── Port: 9000
│   ├── Connect Timeout: 5000
│   ├── Response Timeout: 10000
│   └── Close Connection: ✅
├── Thread Group (TCP 压测)
│   ├── TCP Sampler
│   │   ├── Server/Port (继承自 Config)
│   │   └── Text to send: PING\r\n
│   ├── Response Assertion
│   │   └── Contains: PONG
│   └── View Results Tree
└── Aggregate Report
```

### 2.2 TCP Sampler Config 配置详解

| 配置项 | 值 | 说明 |
|--------|-----|------|
| **TCPClient classname** | `TCPClientImpl` | 客户端实现类 |
| **Server** | `192.168.1.100` | 目标服务器地址 |
| **Port** | `9000` | 端口号 |
| **Connect Timeout** | `5000` | 连接超时(ms) |
| **Response Timeout** | `10000` | 响应等待超时(ms) |
| **Re-use Connection** | ✅ | 连接复用（长连接模式） |
| **Close Connection** | ☐ | 每次请求后关闭连接（短连接） |
| **Set NoDelay** | ✅ | 禁用 Nagle 算法，减少延迟 |
| **SO_LINGER** | `0` | Socket 关闭行为 |
| **End of line(EOL) byte value** | `10` | 行结束符 (\n = 10) |

### 2.3 文本协议请求

```
TCP Sampler
├── TCPClient classname: TCPClientImpl
└── Text to send:
    GET /health HTTP/1.1
    Host: localhost
    Connection: keep-alive
    
    
```

> 文本协议中，`\n` 用回车表示，`\r\n` 用 `\r\n` 表示。

### 2.4 二进制协议请求（十六进制）

```
TCP Sampler
├── TCPClient classname: BinaryTCPClientImpl
└── Text to send: 0x00 0x01 0x02 0x03 0x04
```

**BinaryTCPClientImpl** 使用十六进制格式，JMeter 会自动转换为字节数组发送。

### 2.5 长度前缀协议

```
TCP Sampler
├── TCPClient classname: LengthPrefixedBinaryTCPClientImpl
│   └── 自动在消息体前加 2 字节长度头
├── Binary Prefix Length: 2 (默认)
└── Text to send: {"method":"echo","params":["hello"]}
```

| 配置项 | 说明 |
|--------|------|
| **Binary Prefix Length** | 前缀长度字节数（2 或 4） |
| **Prefix includes length of itself** | 前缀是否包含自身长度 |

---

## 3. 实战场景

### 3.1 Redis 协议压测

Redis 使用 RESP（REdis Serialization Protocol），可以用 TCP Sampler 直接发送：

```
TCP Sampler (PING 命令)
├── TCPClient classname: TCPClientImpl
├── Server: 127.0.0.1
├── Port: 6379
└── Text to send:
    *1\r\n$4\r\nPING\r\n
```

```
TCP Sampler (SET 命令)
├── TCPClient classname: TCPClientImpl
└── Text to send:
    *3\r\n$3\r\nSET\r\n$4\r\nkey1\r\n$10\r\nhello world\r\n
```

```
TCP Sampler (GET 命令)
├── TCPClient classname: TCPClientImpl
└── Text to send:
    *2\r\n$3\r\nGET\r\n$4\r\nkey1\r\n
```

**RESP 协议格式说明**：
- `*N` — 数组长度 N
- `$L` — 字符串长度 L
- `\r\n` — CRLF 分隔符

### 3.2 自定义二进制协议

```groovy
// JSR223 PreProcessor - 动态构造二进制报文
def protocolVersion = (byte) 0x01;
def messageType = (byte) 0x02;  // 0x01=PING, 0x02=DATA
def sequenceId = vars.get("__jm__Thread Group__idx") as int;
def payload = vars.get("payload") ?: "default";

def body = payload.getBytes("UTF-8");
def length = 8 + body.length;  // 头8字节 + body

def buffer = ByteBuffer.allocate(length);
buffer.put(protocolVersion);
buffer.put(messageType);
buffer.putShort((short) length);    // 总长度
buffer.putInt(sequenceId);           // 序列号
buffer.put(body);

// 转为十六进制字符串给 BinaryTCPClientImpl
def hexStr = buffer.array().encodeHex().toString();
vars.put("binaryMessage", hexStr);
```

### 3.3 长连接压测（连接复用）

```
Test Plan
├── TCP Sampler Config
│   ├── Re-use Connection: ✅      # 关键配置
│   ├── Close Connection: ☐
│   └── Set NoDelay: ✅
├── setUp Thread Group (建立连接)
│   └── TCP Sampler - 登录/握手
├── Thread Group (业务请求)
│   ├── TCP Sampler - 心跳
│   ├── TCP Sampler - 业务请求 1
│   └── TCP Sampler - 业务请求 2
└── tearDown Thread Group (释放连接)
    └── TCP Sampler - 登出/关闭
```

### 3.4 短连接压测（每次新建连接）

```
TCP Sampler Config
├── Re-use Connection: ☐
├── Close Connection: ✅
└── Connect Timeout: 2000          # 短连接需要更小的超时
```

> 短连接模式用于测试服务器处理大量新连接的能力，常用于验证 TCP 连接数上限。

### 3.5 粘包/拆包处理

```groovy
// JSR223 PreProcessor - 自定义 TCP 客户端处理粘包
// 使用自定义 TCPClient 实现类
// 在 user.properties 中配置:
// jmeter.tcpclient.binarylength.prefix.length=4
// jmeter.tcpclient.binarylength.prefix.include.length=false

// 或者在 TCP Sampler 中指定:
// TCPClient classname: org.apache.jmeter.protocol.tcp.sampler.LengthPrefixedBinaryTCPClientImpl
```

---

## 4. 响应处理与断言

### 4.1 十六进制响应解析

```groovy
// JSR223 PostProcessor - 解析二进制响应
def responseData = prev.getResponseData();

if (responseData.length > 0) {
    // 解析长度前缀协议
    def length = ((responseData[0] & 0xFF) << 8) | (responseData[1] & 0xFF);
    def messageType = responseData[2] & 0xFF;
    def statusCode = responseData[3] & 0xFF;
    
    vars.put("responseLength", length.toString());
    vars.put("messageType", messageType.toString());
    vars.put("statusCode", statusCode.toString());
    
    // 提取 payload
    def payload = new String(responseData, 4, length - 4, "UTF-8");
    vars.put("responsePayload", payload);
    
    log.info("Response: type=${messageType}, status=${statusCode}, payload=${payload}");
}
```

### 4.2 关键断言

```xml
<!-- Response Assertion - 检查响应包含关键字 -->
<ResponseAssertion>
  <stringProp name="Assertion.test_field">JMSSampler.msgSent</stringProp>
  <stringProp name="Assertion.test_type">2</stringProp>
  <collectionProp name="Asserion.test_strings">
    <stringProp name="0">+OK</stringProp>
    <stringProp name="0">+PONG</stringProp>
  </collectionProp>
</ResponseAssertion>
```

```groovy
// JSR223 Assertion - 二进制协议断言
def responseData = prev.getResponseData();
if (responseData.length < 4) {
    AssertionResult.setFailure(true);
    AssertionResult.setFailureMessage("Response too short: " + responseData.length);
    return;
}

def statusCode = responseData[3] & 0xFF;
if (statusCode != 0) {
    AssertionResult.setFailure(true);
    AssertionResult.setFailureMessage("Error status code: " + statusCode);
}
```

---

## 5. 常见问题排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **Connection refused** | 端口未监听 | 确认服务已启动，端口正确 |
| **Read timed out** | 响应超时 | 增大 Response Timeout 或检查服务端处理时间 |
| **Connection reset** | 服务端主动断开 | 检查协议格式是否正确，是否需要先握手 |
| **No response data** | 未收到完整响应 | 检查 EOL byte 配置，确认行结束符 |
| **Binary data garbled** | 使用了文本模式 | 切换为 BinaryTCPClientImpl |
| **Sticky packet** | TCP 粘包 | 使用 LengthPrefixedBinaryTCPClientImpl |
| **Out of memory** | 连接数过多 | 使用短连接模式 + 合理设置线程数 |

---

## 6. 性能优化建议

1. **NoDelay = true**：禁用 Nagle 算法，减少小包延迟
2. **连接复用**：长连接场景下复用 TCP 连接，避免三次握手开销
3. **批量发送**：将多个小请求合并为一个 TCP 包发送
4. **合理的 Timeout**：Connect Timeout 和 Response Timeout 要分开设置
5. **TCP KeepAlive**：长连接场景启用 TCP KeepAlive 防止连接断开
6. **分布式压测**：TCP 压测容易受单机端口数限制（65535），多机分布式执行
