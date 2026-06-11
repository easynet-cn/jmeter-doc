# 19-Plugins核心插件实战 🔌

> **难度：★★★★☆** | Stepping Thread Group + Throughput Shaping Timer + PerfMon + Dummy Sampler + Inter-Thread Communication

---

## 1. 场景概述

本文聚焦 **JMeter Plugins** 中最常用、最有价值的插件实战，帮助你在实际压测中高效利用这些工具。

### 本文覆盖的核心插件

| 插件 | 类别 | 重要性 | 一句话说明 |
|------|------|:---:|------|
| **Stepping Thread Group** | Thread Group | ⭐⭐⭐⭐⭐ | 阶梯加压找性能拐点 |
| **Concurrency Thread Group** | Thread Group | ⭐⭐⭐⭐⭐ | 精准并发控制 |
| **Throughput Shaping Timer** | Timer | ⭐⭐⭐⭐⭐ | 时间序列吞吐塑形 |
| **Dummy Sampler** | Sampler | ⭐⭐⭐⭐ | 调试和测试原型 |
| **PerfMon Metrics Collector** | Listener | ⭐⭐⭐⭐⭐ | 服务器资源监控 |
| **Response Times vs Threads** | Graph | ⭐⭐⭐⭐ | 性能拐点可视化 |
| **Transactions per Second** | Graph | ⭐⭐⭐⭐ | 吞吐量可视化 |
| **AutoStop Trigger** | Listener | ⭐⭐⭐⭐ | 自动停止条件 |
| **Flexible File Writer** | Listener | ⭐⭐⭐ | 自定义结果输出 |
| **Inter-Thread Communication** | Tool | ⭐⭐⭐ | 线程间通信 |
| **Parameterized Controller** | Logic | ⭐⭐⭐ | 片段参数化复用 |

---

## 2. 阶梯加压实战

### 2.1 Stepping Thread Group 配置

**场景**：从 0 并发逐步加压到 200，每次增加 20 线程，观察系统在多少并发时出现性能拐点。

```
Test Plan
├── Stepping Thread Group
│   ├── This group will start: 20 threads
│   ├── First, wait for: 30 seconds
│   ├── Then start: 20 threads
│   ├── Next, add: 20 threads
│   ├── every: 60 seconds
│   ├── using ramp-up: 10 seconds
│   ├── Then hold load for: 60 seconds
│   ├── Finally, stop: 20 threads
│   └── every: 10 seconds
├── HTTP Request (目标接口)
├── jp@gc - Active Threads Over Time
├── jp@gc - Response Times Over Time
├── jp@gc - Transactions per Second
├── jp@gc - Response Times vs Threads
└── jp@gc - PerfMon Metrics Collector
```

**加压时间线**：

```
时间(s)   并发数    说明
0-10      0→20    首次加压
10-30     20       预热等待
30-40     20→40   第二次加压
40-100    40       保持
100-110   40→60   第三次加压
110-170   60       保持
...       ...
370-380   180→200 第N次加压
380-440   200      峰值保持
440-450   200→180 开始减压
...       ...
```

### 2.2 拐点分析方法

通过 **Response Times vs Threads** 图观察：

```
Response Time (ms)
 5000│                              ●──●──●  ← 指数增长！
 4000│                         ●───●
 3000│                    ●───●
 2000│               ●───●            ← 拐点：150并发
 1000│      ●──●──●──●
  500│  ●──●
     └────────────────────────────────────────
      20  40  60  80 100 120 140 160 180 200  Threads
```

**判断标准**：

| 现象 | 结论 |
|------|------|
| 响应时间线性增长 | 系统正常运行中 |
| 响应时间增速加快 | 接近瓶颈 |
| 响应时间指数增长 | 已到瓶颈 |
| 吞吐量不再增加 | 已达最大吞吐 |
| 吞吐量开始下降 | 过饱和，需立即减压 |

---

## 3. 吞吐量塑形实战

### 3.1 Throughput Shaping Timer

**场景**：模拟双11流量模型——预热→峰值→保持→衰减。

```
Thread Group (Concurrency Thread Group)
│   └── Target Concurrency: 100
├── Throughput Shaping Timer
│   ├── Start RPS: 10, End RPS: 10, Duration: 60      # 预热 60s, 10 QPS
│   ├── Start RPS: 10, End RPS: 100, Duration: 120    # 加压 120s, 10→100 QPS
│   ├── Start RPS: 100, End RPS: 100, Duration: 600   # 保持 600s, 100 QPS
│   ├── Start RPS: 100, End RPS: 200, Duration: 60    # 冲顶 60s, 100→200 QPS
│   └── Start RPS: 200, End RPS: 10, Duration: 300    # 衰减 300s, 200→10 QPS
└── HTTP Request
```

**关键注意**：

> Throughput Shaping Timer 需要配合足够的线程数。如果并发线程不足，实际吞吐量将无法达到目标 RPS。经验公式：`所需线程数 ≈ 目标RPS × 平均响应时间(秒) + 20%缓冲`

### 3.2 验证吞吐量是否达标

使用 **Transactions per Second** 图对比实际吞吐与目标吞吐：

```
TPS
200│                              ██████████
   │                        ██████
100│                  ██████
   │            ██████
 10│      ██████
   │██████
   └───────────────────────────────────────────> Time
   实际吞吐  ———   目标吞吐  ······
```

### 3.3 常见问题：实际 TPS 低于目标

```
排查步骤：
1. 检查并发线程数是否足够
   → 增大 Concurrency Thread Group 的 Target Concurrency
2. 检查平均响应时间
   → 如果 RT > 期望值，TPS 必然不足
3. 检查是否有线程阻塞
   → 查看 Active Threads Over Time 图
4. 检查网络带宽
   → 查看 Bytes Throughput Over Time 图
5. 检查是否有错误请求
   → 查看 Response Codes per Second 图
```

---

## 4. 服务端资源监控实战

### 4.1 PerfMon Agent 部署

```bash
# 1. 在目标服务器上下载并启动
wget https://github.com/undera/perfmon-agent/releases/download/2.2.3/ServerAgent-2.2.3.zip
unzip ServerAgent-2.2.3.zip
cd ServerAgent-2.2.3

# 2. 启动 Agent（后台运行）
nohup ./startAgent.sh --tcp-port 4444 --udp-port 0 > agent.log 2>&1 &

# 3. 验证
telnet localhost 4444
# 输入 test，应返回 Yep
```

### 4.2 JMeter 端配置

```
jp@gc - PerfMon Metrics Collector

添加行：
| Host           | Port | Metric       | Metric Parameter |
|----------------|------|--------------|------------------|
| 192.168.1.100  | 4444 | CPU          | combined         |
| 192.168.1.100  | 4444 | Memory       | used             |
| 192.168.1.100  | 4444 | Network I/O  | eth0             |
| 192.168.1.100  | 4444 | Disks I/O    | sda              |
| 192.168.1.101  | 4444 | CPU          | combined         |  # 多机监控
```

### 4.3 关键指标解读

```
CPU combined > 85% 持续  → CPU 瓶颈
Memory used > 90%       → 内存瓶颈（可能 OOM）
Network I/O 达到带宽上限 → 网络瓶颈
Disks I/O 持续高位       → 磁盘 I/O 瓶颈
CPU 高 + I/O Wait 高     → 磁盘瓶颈导致 CPU 等待
```

### 4.4 多指标关联分析

```
最佳实践：在 Dashboard 中同时查看：
┌─────────────────────────────────────┐
│  Response Times Over Time          │  ← 应用指标
├─────────────────────────────────────┤
│  Transactions per Second           │  ← 吞吐指标
├─────────────────────────────────────┤
│  PerfMon - CPU / Memory / Network  │  ← 系统指标
├─────────────────────────────────────┤
│  Response Codes per Second         │  ← 错误指标
└─────────────────────────────────────┘

关联分析：
RT 上升 + CPU 上升 + TPS 不降 → 正常压力
RT 上升 + CPU 不变 + TPS 下降 → 外部依赖瓶颈
RT 上升 + CPU 下降 + IO Wait 高 → 磁盘瓶颈
```

---

## 5. Dummy Sampler 调试实战

### 5.1 调试提取器

```
Thread Group (1 线程, 1 循环)
├── Dummy Sampler
│   ├── Response Code: 200
│   ├── Response Message: OK
│   └── Response Data:
│       {
│         "code": 0,
│         "data": {
│           "user": {
│             "id": 12345,
│             "name": "John",
│             "roles": ["admin", "editor"],
│             "profile": {
│               "age": 30,
│               "email": "john@example.com"
│             }
│           },
│           "token": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NSJ9.xxx"
│         }
│       }
├── JSON Extractor
│   ├── userId: $.data.user.id
│   ├── userName: $.data.user.name
│   ├── roles: $.data.user.roles
│   ├── age: $.data.user.profile.age
│   └── token: $.data.token
├── Debug Sampler (查看提取的变量)
└── View Results Tree
```

### 5.2 调试逻辑控制器

```
Thread Group (1 线程)
├── Dummy Sampler - 成功 (Response Code: 200)
├── Dummy Sampler - 失败 (Response Code: 500)
├── If Controller
│   └── Condition: ${JMeterThread.last_sample_ok} == "false"
│   └── Dummy Sampler - 重试 (Latency: 1000)
├── While Controller
│   └── Condition: ${__javaScript("${retryCount}" != "0")}
│   └── Dummy Sampler
└── Debug Sampler
```

### 5.3 模拟慢接口

```
Dummy Sampler - 模拟慢接口
├── Response Time: ${__Random(500,3000)}   # 500ms-3000ms 随机
├── Latency: ${__Random(100,500)}          # 100ms-500ms 延迟
└── Response Data: {"status": "ok"}
```

### 5.4 模拟故障注入

```
Throughput Controller (10%)
├── Dummy Sampler - 超时
│   ├── Response Code: 503
│   ├── Response Message: Service Unavailable
│   └── Successful: false ✅

Throughput Controller (5%)
├── Dummy Sampler - 慢响应
│   ├── Response Time: 5000
│   └── Response Code: 200

Throughput Controller (85%)
├── Dummy Sampler - 正常
│   ├── Response Time: 100
│   └── Response Code: 200
```

---

## 6. AutoStop 自动停止实战

### 6.1 基于错误率自动停止

```
jp@gc - AutoStop Trigger
├── Average Response Time: 0        # 不检查
├── Error Rate: 10                  # 错误率 > 10%
├── Latency: 0                      # 不检查
├── Check interval: 10              # 每 10s 检查一次
└── Stop Test on: failed            # 条件满足即停止
```

### 6.2 基于响应时间自动停止

```
jp@gc - AutoStop Trigger
├── Average Response Time: 5000     # 平均 RT > 5s
├── Error Rate: 0                   # 不检查
├── Check interval: 30              # 每 30s 检查
└── Stop Test on: failed
```

### 6.3 组合条件自动停止

```
Test Plan
├── jp@gc - AutoStop Trigger (RT 阈值)
│   └── Average Response Time: 3000
├── jp@gc - AutoStop Trigger (错误率阈值)
│   └── Error Rate: 5
└── jp@gc - AutoStop Trigger (延迟阈值)
    └── Latency: 2000
```

> 多个 AutoStop 独立判断，任一满足即停止。

---

## 7. Flexible File Writer 实战

### 7.1 自定义输出格式

```
jp@gc - Flexible File Writer
├── Filename: custom_results.csv
├── Write File Header: ✅
├── Record each sample: ✅
└── Fields:
    timeStamp|elapsed|responseCode|responseMessage|threadName|
    success|bytes|sentBytes|grpThreads|allThreads|Latency|
    IdleTime|Connect|URL|label|dataType|Hostname
```

### 7.2 仅记录失败请求

```
Fields:
timeStamp|elapsed|responseCode|responseMessage|threadName|URL|label|failureMessage

配合 JSR223 PostProcessor 过滤：
```

```groovy
// 仅记录失败的请求
if (!prev.isSuccessful()) {
    // Flexible File Writer 会自动处理，无需额外代码
    // 但可以通过 Sample Variables 添加自定义字段
    prev.setSampleLabel("FAILED:" + prev.getSampleLabel());
}
```

### 7.3 自定义字段输出

```groovy
// JSR223 PostProcessor - 添加自定义数据
def customMetric = vars.get("businessMetric") ?: "N/A";
prev.setSamplerData("custom_metric=" + customMetric);
```

---

## 8. Inter-Thread Communication 实战

### 8.1 生产者-消费者模式

```
setUp Thread Group (1 线程 - 数据准备)
├── JSR223 Sampler (生成测试数据)
│   └── props.put("testDataReady", "true")
└── JSR223 PostProcessor
    └── 将数据写入 FIFO 队列

Thread Group (100 线程 - 业务执行)
├── jp@gc - Inter-Thread Communication PreProcessor
│   ├── Queue Name: testDataQueue
│   ├── Timeout: 10000
│   └── Default Value: DEFAULT
├── HTTP Request (使用队列中的数据)
└── ...
```

### 8.2 同步栅栏实现

```groovy
// Thread Group 1 - 信号生产者
// JSR223 Sampler
import org.apache.jmeter.util.JMeterUtils;

// 等待所有线程就绪后广播信号
props.put("readyCount", "0");

// Thread Group 2-N - 等待信号
// jp@gc - Inter-Thread Communication PreProcessor
// 阻塞等待生产者信号
```

---

## 9. 完整压测方案：Plugins 组合

### 9.1 标准压测模板

```
Test Plan
├── User Defined Variables
│   ├── targetHost = api.example.com
│   └── targetPort = 443
├── HTTP Request Defaults
├── HTTP Cookie Manager
├── HTTP Header Manager
│
├── setUp Thread Group (数据准备)
│   ├── CSV Data Set Config
│   └── JSR223 Sampler (预热/初始化)
│
├── Concurrency Thread Group (主压测)
│   ├── Target Concurrency: 200
│   ├── Ramp Up Time: 5 (分钟)
│   ├── Ramp-Up Steps Count: 10
│   └── Hold Target Rate Time: 30 (分钟)
│   │
│   ├── Throughput Shaping Timer
│   │   └── Start: 50 → End: 500 RPS over 5min
│   │
│   ├── HTTP Request (业务接口 1)
│   ├── HTTP Request (业务接口 2)
│   ├── HTTP Request (业务接口 3)
│   │
│   ├── JSON Extractor
│   ├── Response Assertion
│   └── Duration Assertion (< 3000ms)
│
├── jp@gc - Active Threads Over Time
├── jp@gc - Response Times Over Time
├── jp@gc - Transactions per Second
├── jp@gc - Response Times vs Threads
├── jp@gc - Response Codes per Second
├── jp@gc - Response Times Percentiles
├── jp@gc - PerfMon Metrics Collector
├── jp@gc - AutoStop Trigger
│   ├── Error Rate: 5%
│   └── Avg RT: 5000ms
├── jp@gc - Flexible File Writer
├── View Results Tree (禁用)
└── Aggregate Report
│
└── tearDown Thread Group (清理)
    └── JSR223 Sampler (数据清理/通知)
```

### 9.2 CI/CD 集成中的图形生成

```bash
#!/bin/bash
# post_test_graphs.sh - 压测后生成图表

RESULTS_DIR="./results/$(date +%Y%m%d_%H%M%S)"
mkdir -p $RESULTS_DIR

# 合并所有 JTL 文件
JMeterPluginsCMD --tool MergeResults \
  --input-file-pattern "./jtl/*.jtl" \
  --output-file $RESULTS_DIR/merged.jtl

# 生成图表
declare -A GRAPHS=(
  ["rt_over_time"]="ResponseTimesOverTime"
  ["tps"]="TransactionsPerSecond"
  ["active_threads"]="ActiveThreadsOverTime"
  ["rt_vs_threads"]="ResponseTimesVsThreads"
  ["rt_percentiles"]="ResponseTimesPercentiles"
  ["rt_distribution"]="ResponseTimesDistribution"
  ["response_codes"]="ResponseCodesPerSecond"
  ["bytes_throughput"]="BytesThroughputOverTime"
)

for name in "${!GRAPHS[@]}"; do
  JMeterPluginsCMD --generate-png "$RESULTS_DIR/${name}.png" \
    --input-jtl $RESULTS_DIR/merged.jtl \
    --plugin-type ${GRAPHS[$name]} \
    --width 1920 --height 1080 \
    --forceY 0
done

echo "Charts generated in $RESULTS_DIR"
ls -la $RESULTS_DIR/
```

---

## 10. 常见问题排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **Plugins Manager 菜单不出现** | jar 未正确放置 | 确认 `plugins-manager.jar` 在 `lib/ext/` |
| **PerfMon 连接超时** | Agent 未启动或防火墙 | `telnet host 4444` 测试 |
| **Throughput Shaping Timer 不生效** | 线程数不足 | 增大 Concurrency Thread Group |
| **图形不显示数据** | 采样间隔问题 | 检查 Interval 设置 |
| **插件冲突/异常** | 版本不兼容 | 使用 Plugins Manager 升级 |
| **命令行生成图失败** | 依赖缺失 | 安装 `jpgc-cmd` 插件 |
| **内存溢出** | 图形监听器消耗大 | 非 GUI 模式禁用图形监听器 |
