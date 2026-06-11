# 10-JMeter Plugins 插件参考 🔌

> **插件官网**：https://jmeter-plugins.org/  
> **安装方式**：通过 Plugins Manager 或手动下载 jar 放入 `$JMETER_HOME/lib/ext/`

---

## 1. 概述

**JMeter Plugins** 是由 Andrey Pokhilko 发起的 JMeter 第三方插件生态项目，提供大量官方 JMeter 缺失的高级功能，包括高级线程组、专业图形报告、协议扩展等。

### 1.1 Plugins Manager 安装

**方式一：独立安装**

```bash
# 下载 plugins-manager.jar 放入 lib/ext/
wget https://jmeter-plugins.org/get/ -O $JMETER_HOME/lib/ext/jmeter-plugins-manager.jar
```

**方式二：Docker**

```bash
docker run -v $(pwd):/test justb4/jmeter:latest
```

安装后重启 JMeter，菜单栏会出现 `Options → Plugins Manager`。

### 1.2 Plugins Manager 使用

```
Options → Plugins Manager
├── Installed Plugins      # 已安装
├── Available Plugins      # 可安装
└── Upgrades               # 可升级
```

> 推荐安装时勾选依赖项，Plugins Manager 会自动处理。

---

## 2. 插件分类总览

| 分类 | 插件数 | 核心价值 |
|------|:---:|------|
| **Thread Groups** | 5 | 高级加压策略、并发控制 |
| **Samplers** | 10 | 扩展协议（UDP、HDFS、HBase） |
| **Timers** | 1 | 精准吞吐量塑形 |
| **Listeners** | 5 | 自动停止、灵活输出 |
| **Graphs** | 17 | 专业性能可视化 |
| **Tools** | 7 | 进程间通信、命令行工具 |
| **Post-Processors** | 4 | JSON/YAML 路径提取 |
| **Config Items** | 5 | Redis、目录列表、变量配置 |
| **Assertions** | 1 | JSON/YAML 路径断言 |
| **Logic Controllers** | 1 | 参数化控制器 |
| **Pre-Processors** | 1 | 原始数据源 |
| **Functions** | 1 | 自定义函数 |

---

## 3. Thread Groups（线程组）

### 3.1 Stepping Thread Group（阶梯加压）

逐步增加并发用户数，适合找到系统性能拐点。

| 参数 | 说明 |
|------|------|
| **This group will start** | 初始线程数 |
| **First, wait for** | 首次启动等待时间(秒) |
| **Then start** | 每次增加的线程数 |
| **Next, add** | 增加的线程数 |
| **every** | 每 N 秒增加一次 |
| **using ramp-up** | 每次增加的启动时间(秒) |
| **Then hold load for** | 达到峰值后保持时间(秒) |
| **Finally, stop** | 每次减少线程数 |
| **every** | 每 N 秒减少一次 |

**加压曲线示例**：

```
Threads
100│                              ████████████████
 80│                        ██████
 60│                  ██████
 40│            ██████
 20│      ██████
  0│██████
   └───────────────────────────────────────────────> Time
    0   60   120  180  240  300  360  420
```

### 3.2 Ultimate Thread Group（高级线程调度）

支持多段线程调度，模拟复杂负载模型。

```
线程调度表：
| Start Threads Count | Initial Delay | Startup Time | Hold Load For | Shutdown Time |
|                   0 |             0 |           30 |           300 |            10 |
|                 100 |            60 |           30 |           300 |            10 |
|                 200 |           120 |           30 |           300 |            10 |
```

### 3.3 Concurrency Thread Group（并发线程组）

基于目标并发数的线程管理，适合精确控制并发。

| 参数 | 说明 |
|------|------|
| **Target Concurrency** | 目标并发数 |
| **Ramp Up Time** | 加压时间(分钟) |
| **Ramp-Up Steps Count** | 加压步数 |
| **Hold Target Rate Time** | 峰值保持时间(分钟) |

### 3.4 Arrivals Thread Group（到达率线程组）

基于请求到达率（TPS）控制，更贴近生产流量模型。

| 参数 | 说明 |
|------|------|
| **Target Rate** | 目标到达率（请求/分钟） |
| **Ramp Up Time** | 加压时间 |
| **Ramp-Up Steps Count** | 加压步数 |
| **Hold Target Rate Time** | 峰值保持时间 |
| **Concurrency Limit** | 最大并发限制 |

### 3.5 Free-Form Arrivals Thread Group

自由定义到达率曲线，通过时间序列数组灵活控制流量。

```properties
# 示例：0-60秒 10/s, 60-120秒 50/s, 120-180秒 30/s
Start: 0,60,120
End: 60,120,180
Duration: 60,60,60
Target Rate: 10,50,30
```

---

## 4. Samplers（采样器）

### 4.1 Dummy Sampler（调试采样器）⭐⭐⭐⭐⭐

**最常用的调试工具**，不发起真实网络请求。

| 参数 | 说明 |
|------|------|
| **Response Data** | 自定义返回内容（支持变量） |
| **Response Code** | 自定义状态码（如 200/500） |
| **Response Message** | 自定义响应消息 |
| **Latency** | 模拟延迟(ms) |
| **Response Time** | 模拟响应时间(ms) |
| **Request Data** | 请求数据（Log Viewer 中查看） |
| **Successful** | 是否标记为成功 |

**典型用法**：

```
场景1：调试提取器和断言
├── Dummy Sampler
│   ├── Response Data: {"userId": 123, "token": "abc"}
│   └── Response Code: 200
├── JSON Extractor → $..token
└── Response Assertion

场景2：验证逻辑控制器
├── Dummy Sampler (Response Code: 200)
├── Dummy Sampler (Response Code: 500)
└── If Controller → 根据状态码分支
```

### 4.2 UDP Sampler（UDP 协议）

| 参数 | 说明 |
|------|------|
| **Hostname** | 目标主机 |
| **Port** | 目标端口 |
| **Data Encode/Decode** | 编码/解码类 |
| **Request Data** | 发送数据 |

### 4.3 HTTP Raw Request

发送原始 HTTP 报文，支持非标准 HTTP 请求。

```
Request Data:
GET /api/health HTTP/1.1
Host: localhost
X-Custom-Header: value


```

### 4.4 HDFS Operations / Hadoop Job Tracker / HBase 系列

大数据生态集成，支持 HDFS 文件操作、Hadoop Job 监控、HBase CRUD 操作。

### 4.5 Set Variables Action

在测试执行过程中动态设置变量。

| 参数 | 说明 |
|------|------|
| **Variable Name** | 变量名 |
| **Variable Value** | 变量值 |

---

## 5. Timers（定时器）

### 5.1 Throughput Shaping Timer ⭐⭐⭐⭐⭐

**最重要的插件定时器**，通过时间序列定义吞吐量曲线。

| 参数 | 说明 |
|------|------|
| **Start RPS** | 起始每秒请求数 |
| **End RPS** | 结束每秒请求数 |
| **Duration** | 该阶段持续时间(秒) |

```
吞吐量塑形调度表：
| Start RPS | End RPS | Duration(sec) |
|        10 |      10 |            60 |  # 预热 60s，10 RPS
|        10 |     100 |           120 |  # 加压 120s，10→100 RPS
|       100 |     100 |           300 |  # 保持 300s，100 RPS
|       100 |       0 |            60 |  # 减压 60s
```

---

## 6. Listeners（监听器）

### 6.1 Flexible File Writer ⭐⭐⭐⭐

高度可定制的文件写入器，支持自定义字段和分隔符。

```properties
# 写入字段配置（Tab 分隔）
elapsedTime|timeStamp|responseCode|responseMessage|threadName|dataType|success|bytes|grpThreads|allThreads|Latency|IdleTime|Connect|URL|label
```

### 6.2 AutoStop Trigger

根据条件自动停止测试。

| 参数 | 说明 |
|------|------|
| **Average Response Time** | 平均响应时间阈值 |
| **Error Rate** | 错误率阈值 |
| **Check interval** | 检查间隔(秒) |

### 6.3 Synthesis Report

可过滤的聚合报告，支持正则过滤和偏移量。

### 6.4 Graphs Generator

从 JTL 结果文件生成图表，适合命令行环境。

```bash
# 命令行生成图表
JMeterPluginsCMD --generate-png response_times.png \
  --input-jtl results.jtl \
  --plugin-type ResponseTimesOverTime \
  --width 1920 --height 1080
```

---

## 7. Graphs（图形插件）⭐⭐⭐⭐⭐

### 7.1 核心图形列表

| 图形插件 | 展示内容 | 适用场景 |
|----------|----------|----------|
| **Active Threads Over Time** | 活跃线程数随时间变化 | 验证加压策略执行情况 |
| **Response Times Over Time** | 响应时间趋势 | 发现性能劣化 |
| **Transactions per Second** | 每秒事务数 | 验证吞吐量 |
| **Response Times vs Threads** | 响应时间-线程数关系 | 找性能拐点 |
| **Transaction Throughput vs Threads** | 吞吐-线程数关系 | 验证可扩展性 |
| **Response Times Distribution** | 响应时间分布 | 查看长尾分布 |
| **Response Times Percentiles** | 百分位分析 | p90/p95/p99 可视化 |
| **Response Codes per Second** | 每秒响应状态码 | 监控错误率变化 |
| **Bytes Throughput Over Time** | 带宽使用 | 网络瓶颈分析 |
| **Response Latencies Over Time** | 延迟时间趋势 | 区分网络延迟和处理延迟 |
| **Connect Times Over Time** | 连接建立时间 | 分析连接池/SSL 问题 |
| **Composite Timeline Graph** | 综合时间线 | 多指标叠加对比 |
| **Server Hits per Seconds** | 每秒请求数 | 服务端视角的请求量 |
| **PerfMon Metrics Collector** | 服务器资源监控 | CPU/内存/磁盘/网络 |
| **DbMon Sample Collector** | 数据库性能指标 | SQL 层面监控 |
| **JMX Monitoring Collector** | JVM 指标 | 堆内存/GC/线程 |
| **Extracted Data Over Time** | 提取数据趋势 | 自定义指标可视化 |

### 7.2 图形插件通用配置

```
右键 Thread Group → Add → Listener → jp@gc - [图形名称]

通用设置：
├── Interval (ms): 1000        # 采样间隔
├── Graph Title                 # 图形标题
├── Line Width                  # 线条宽度
├── Dynamic Graph               # 动态更新 Y 轴范围
└── Time Format                 # 时间格式
```

---

## 8. Post-Processors（后置处理器）

### 8.1 JSON/YAML Path Extractor ⭐⭐⭐⭐

支持 JSONPath 和 YAMLPath 提取，比官方的 JSON Extractor 功能更丰富。

| 参数 | 说明 |
|------|------|
| **Destination Variable Name** | 变量名 |
| **JSONPath / YAMLPath Expression** | 路径表达式 |
| **Default Value** | 默认值 |
| **Compute concatenation var** | 计算拼接变量 |
| **Fragment Name** | 从哪个 Fragment 提取 |

**高级表达式示例**：

```
$.store.book[?(@.price < 10)].title     # 过滤价格<10的书名
$.store.book[0:3].title                  # 前3本书名
$.store.book[-1].title                   # 最后一本书名
$.store.book.length()                    # 书本数量
```

### 8.2 JSON Formatter PostProcessor

格式化 JSON 响应，便于在 View Results Tree 中查看。

### 8.3 JSON To XML Converter

将 JSON 响应转换为 XML 格式。

### 8.4 XML Format PostProcessor

格式化 XML 响应。

---

## 9. Config Items（配置元素）

### 9.1 Variables from CSV

增强版 CSV 配置，支持更多格式。

### 9.2 Redis Data Set

从 Redis 读取测试数据集。

| 参数 | 说明 |
|------|------|
| **Redis Key** | 数据集 Key |
| **Variable Names** | 变量名列表 |
| **Delimiter** | 分隔符 |
| **Redis Server** | Redis 地址 |
| **Redis Port** | Redis 端口 |

### 9.3 Directory Listing Config

读取目录文件列表作为测试数据。

### 9.4 Lock Files

基于锁文件的并发控制，防止测试意外重复启动。

---

## 10. Assertions（断言）

### 10.1 JSON/YAML Path Assertion

基于 JSONPath/YAMLPath 的断言。

```jsonpath
# 验证响应中 userId 存在且为数字
$.userId =~ /\d+/

# 验证 books 数组长度 >= 1
$.books.length() >= 1
```

---

## 11. Logic Controllers（逻辑控制器）

### 11.1 Parameterized Controller ⭐⭐⭐

实现测试片段的参数化复用，类似于函数调用。

```
Test Plan
├── Parameterized Controller (调用者)
│   ├── Param: userId=123
│   └── Param: action=view
│   └── → 调用 "UserAction" Test Fragment
│
└── Test Fragment "UserAction" (被调用者)
    ├── HTTP Request
    │   └── GET /api/users/${userId}/${action}
    └── JSON Extractor
```

---

## 12. Pre-Processors（前置处理器）

### 12.1 Raw Data Source

从原始数据源读取请求体，支持大文件流式读取。

---

## 13. Tools（工具）

### 13.1 Inter-Thread Communication ⭐⭐⭐⭐

线程间通信机制，实现线程间的数据传递和同步。

```
线程1 (Producer)
├── JSR223 Sampler → props.put("queue", data)

线程2 (Consumer)
├── Inter-Thread Communication PostProcessor
│   └── 从 FIFO 队列读取数据
```

### 13.2 Command-Line Graph Plotting Tool

命令行生成性能图表。

```bash
# 生成响应时间趋势图
JMeterPluginsCMD --generate-png ./report/rt_trend.png \
  --input-jtl results.jtl \
  --plugin-type ResponseTimesOverTime

# 生成吞吐量图
JMeterPluginsCMD --generate-png ./report/tps.png \
  --input-jtl results.jtl \
  --plugin-type TransactionsPerSecond

# 生成所有图表
for type in ResponseTimesOverTime TransactionsPerSecond \
            ResponseTimesPercentiles ActiveThreadsOverTime \
            ResponseTimesDistribution BytesThroughputOverTime; do
  JMeterPluginsCMD --generate-png ./report/${type}.png \
    --input-jtl results.jtl \
    --plugin-type $type
done
```

### 13.3 Test Plan Consistency Checker

检查测试计划的一致性问题（如变量未定义、循环引用等）。

### 13.4 HTTP Simple Table Server

内嵌 HTTP 服务器，用于管理 CSV 数据集。

```bash
# 启动 Table Server
jmeter -n -t table_server_test.jmx -JtableServer.port=9191

# 通过 HTTP 操作数据集
curl http://localhost:9191/read?filename=users.csv
curl "http://localhost:9191/write?filename=users.csv&line=user1,pass1"
```

### 13.5 Merge Results

合并多次压测的 JTL 结果文件，便于对比分析。

```bash
# 合并结果
JMeterPluginsCMD --tool MergeResults \
  --input-file-pattern "results_*.jtl" \
  --output-file merged_results.jtl
```

---

## 14. 推荐安装组合

### 14.1 最小安装（入门）

```
- jmeter-plugins-manager
- jpgc-dummy
- jpgc-graphs-basic
- jpgc-standard
```

### 14.2 标准安装（常规压测）

```
- jmeter-plugins-manager
- jpgc-dummy
- jpgc-graphs-basic
- jpgc-graphs-additional
- jpgc-standard
- jpgc-casutg
- jpgc-ffw
- jpgc-fifo
- jpgc-perfmon
```

### 14.3 完整安装（高级场景）

```
- jmeter-plugins-manager
- jpgc-dummy
- jpgc-graphs-composite
- jpgc-graphs-dist
- jpgc-graphs-vs
- jpgc-casutg
- jpgc-ffw
- jpgc-fifo
- jpgc-perfmon
- jpgc-dbmon
- jpgc-jmxmon
- jpgc-udp
- jpgc-hadoop
- jpgc-hbase
- jpgc-redis
- jpgc-json
- jpgc-autostop
- jpgc-cmd
- jpgc-synthesis
- jpgc-tst
- jpgc-prmctl
- jpgc-lockfile
```

---

## 15. PerfMon Metrics Collector 配置

### 15.1 服务端部署

```bash
# 在目标服务器上启动 PerfMon Agent
wget https://jmeter-plugins.org/get/ServerAgent/
unzip ServerAgent-2.2.3.zip
cd ServerAgent-2.2.3
./startAgent.sh --udp-port 0 --tcp-port 4444
```

### 15.2 JMeter 端配置

```
右键 Thread Group → Add → Listener → jp@gc - PerfMon Metrics Collector

添加指标：
├── Host: 192.168.1.100
├── Port: 4444
├── Metric: CPU
│   └── Metric Parameter: combined
├── Metric: Memory
│   └── Metric Parameter: used
├── Metric: Network I/O
│   └── Metric Parameter: eth0
└── Metric: Disks I/O
    └── Metric Parameter: sda
```

---

## 16. 常见问题

| 问题 | 解决方案 |
|------|----------|
| **Plugins Manager 不显示** | 确认 `jmeter-plugins-manager.jar` 在 `lib/ext/` 下 |
| **图形空白** | 确保使用 GUI 模式运行，检查采样间隔 |
| **PerfMon 无法连接** | 检查防火墙 4444 端口，确认 ServerAgent 已启动 |
| **插件冲突** | 使用 Plugins Manager 升级到最新版本 |
| **OutOfMemory** | 图形插件消耗内存，减少图形数量或增大 `-Xmx` |
